# MaaS on RHOAI 3.4 with OpenShift Token Auth

*Deploy Models-as-a-Service (GA) on RHOAI 3.4 using OpenShift tokens (K8s TokenReview) for authentication — no external IdP required. Includes tier-based token rate limiting and API key management.*

> ### Auth Enforcement Does Not Work with RHCL 1.4.0 + SM 3.3.x
>
> **Tested June 16, 2026 on OCP 4.19.17 with RHCL 1.4.0 + SM 3.3.4.** All auth enforcement fails with this version combination. The root cause is auth-method-agnostic — it breaks at the Wasm shim layer before any identity provider is consulted.
>
> **What happens:**
> - Unauthenticated inference returns **HTTP 200** (should be 401)
> - API key creation returns **HTTP 500 AUTH_FAILURE** (should be 201)
>
> **Root cause:** RHCL 1.4.0's Kuadrant operator creates a Wasm EnvoyFilter with `allow_on_headers_stop_iteration: true` — a field in Envoy's `PluginConfig` proto that controls whether the Wasm plugin can return `StopIteration` from `on_http_request_headers`. The Kuadrant Wasm shim requires this to pause request processing while it dispatches an async gRPC call to Authorino and waits for the auth response. **Neither SM 3.2 (Istio 1.26) nor SM 3.3 (Istio 1.27) supports this field** — their Envoy binaries predate the proto version that introduced it. Without `StopIteration`, Envoy ignores the pause and immediately forwards the request to the backend. The auth response from Authorino arrives too late — the request has already been forwarded without `X-MaaS-Username` / `X-MaaS-Group` headers, causing maas-api to return `AUTH_FAILURE`.
>
> **Evidence from live cluster diagnostics:**
> - `kuadrant.hits` increments (Wasm shim matches action sets) but `kuadrant.denied` is always 0
> - `kuadrant-auth-service` cluster has `cx_total: 0` — zero gRPC connections to Authorino ever made
> - Authorino logs show zero incoming requests after gateway pod restart
> - Config churn is a secondary symptom (operator reconciles ~2/sec) but not the root cause — same behavior with operator scaled down and config stable
>
> **Why RHOAI forces RHCL 1.4.0:** RHOAI has zero OLM dependencies (`olm.package.required` is empty). However, the RHCL bundle declares catalog-level dependencies on `authorino-operator`, `limitador-operator`, and `dns-operator`. OLM auto-generates subscriptions for these on the `stable` channel with `Automatic` approval, which upgrades them to v1.4.0. The v1.4.0 sub-operators pull `rhcl-operator.v1.4.0` as a dependency.
>
> **Working configuration:** See the `rhcl-hack` branch — RHCL 1.3.4 + SM 3.2.5 with pre-created sub-operator subscriptions (Manual approval, `startingCSV` pinned to v1.3.1) installed BEFORE the RHCL operator. SM must be installed first (before any RHCL subscriptions) to avoid OLM bundling v1.4.0 upgrades into the SM install plan.
>
> For the full investigation, see [`maas_auth_enforcement_issue.md`](../maas_auth_enforcement_issue.md).

## Prerequisites

- OpenShift 4.19+ cluster with BOM deployed ([`bom_rhoai_3.4.md`](../bom_rhoai_3.4.md)) — RHOAI 3.4, RHCL, SM, KServe, Dashboard
- Let's Encrypt certificates installed on the cluster
- `oc`, `jq`, `curl`, `envsubst` installed
- GPU worker node (A10G) available (optional — simulator model runs on CPU)

```shell
export DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
echo "Cluster domain: ${DOMAIN}"
```

## Architecture

```
User (OpenShift token or API key)
    │
    ▼
maas-default-gateway (HTTPS, Let's Encrypt)
    │
    ├── Gate 1: AuthPolicy (Authorino)
    │     ├── Validate OpenShift token (K8s TokenReview)
    │     ├── OR validate MaaS API key (sk-oai-*)
    │     ├── Extract: groups, userid
    │     └── Check group membership
    │
    ├── Gate 2: TokenRateLimitPolicy (Limitador)
    │     ├── Per-subscription token limits
    │     └── Per-user token counting
    │
    └── Model backend (LLMInferenceService — simulator, CPU-only)
```

## Auth Flow

```
1. Admin creates OpenShift Groups + Service Accounts (Phase 1)
2. User creates a bound SA token: oc create token <sa> --audience=...
3. User calls POST /maas-api/v1/api-keys with the SA token
   → Authorino validates via K8s TokenReview
   → maas-api reads X-MaaS-Username + X-MaaS-Group headers
   → Creates sk-oai-* API key bound to user's subscription
4. User calls inference endpoints with the API key
   → Authorization: Bearer sk-oai-*
```

## Subscriptions (Token Rate Limits)

K8s TokenReview returns `system:serviceaccounts:<namespace>` as a group for service accounts — NOT OpenShift Groups. To map each tier to a unique group, we use one namespace per tier.

| Tier | Tokens/min | K8s Group (from TokenReview) | Service Account | Priority |
|------|-----------|------------------------------|-----------------|----------|
| Free | 100 | `system:serviceaccounts:maas-free` | `maas-free:user` | 0 |
| Premium | 50,000 | `system:serviceaccounts:maas-premium` | `maas-premium:user` | 1 |
| Enterprise | 100,000 | `system:serviceaccounts:maas-enterprise` | `maas-enterprise:user` | 2 |

## Manifests

| File | Purpose |
|------|---------|
| [`user-workload-monitoring.yaml`](manifests/user-workload-monitoring.yaml) | Enable User Workload Monitoring for MaaS metrics |
| [`kuadrant-instance.yaml`](manifests/kuadrant-instance.yaml) | Kuadrant CR (Authorino + Limitador) |
| [`gateway-resources.yaml`](manifests/gateway-resources.yaml) | ConfigMap with gateway proxy memory limits (2Gi — prevents OOMKill) |
| [`maas-gateway.yaml`](manifests/maas-gateway.yaml) | maas-default-gateway with HTTP + HTTPS (requires `envsubst`) |
| [`postgres-maas.yaml`](manifests/postgres-maas.yaml) | Standalone PostgreSQL for maas-api |
| [`model.yaml`](manifests/model.yaml) | LLMInferenceService (llm-d simulator, CPU-only) |
| [`maas-model-ref.yaml`](manifests/maas-model-ref.yaml) | MaaSModelRef — register model with MaaS governance |
| [`maas-auth-policy.yaml`](manifests/maas-auth-policy.yaml) | MaaSAuthPolicy — OpenShift groups to model access |
| [`maas-subscriptions.yaml`](manifests/maas-subscriptions.yaml) | 3 MaaSSubscription CRs (free/premium/enterprise tiers) |

---

## Phase 1: Service Accounts (one namespace per tier)

Create one namespace per subscription tier with a service account in each. K8s TokenReview returns `system:serviceaccounts:<namespace>` as a group — this is what MaaS uses for subscription matching. No external IdP or OpenShift Groups needed.

### 1a. Create tier namespaces and service accounts

```shell
for TIER in free premium enterprise; do
  oc create namespace "maas-${TIER}" || true
  oc create sa user -n "maas-${TIER}"
done
```

### 1b. Verify token and group membership

```shell
TOKEN=$(oc create token user -n maas-premium \
  --audience=https://kubernetes.default.svc \
  --audience=maas-default-gateway-sa \
  --duration=1h)

echo "Token length: ${#TOKEN}"

# Verify the token via TokenReview
oc create -o json -f - <<EOF | jq '{authenticated: .status.authenticated, user: .status.user.username, groups: .status.user.groups}'
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "TokenReview",
  "spec": {
    "token": "${TOKEN}",
    "audiences": ["https://kubernetes.default.svc", "maas-default-gateway-sa"]
  }
}
EOF
# Expected: authenticated=true, user=system:serviceaccount:maas-premium:user
# groups: ["system:serviceaccounts", "system:serviceaccounts:maas-premium", "system:authenticated"]
```

---

## Phase 2: MaaS Platform Prerequisites

> **These are prerequisites required by RHOAI 3.4** per the official MaaS documentation ([PDF](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/pdf/govern_llm_access_with_models-as-a-service/Red_Hat_OpenShift_AI_Self-Managed-3.4-Govern_LLM_access_with_Models-as-a-Service-en-US.pdf) | [HTML — requires login](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/govern_llm_access_with_models-as-a-service/)). They must be in place **before** enabling `modelsAsService` in the DataScienceCluster. RHOAI does not auto-provision these.

### 2a. Enable User Workload Monitoring

Required for MaaS to collect token usage metrics.

```shell
oc apply -f manifests/user-workload-monitoring.yaml
```

### 2b. Create Kuadrant instance in `kuadrant-system`

Required by RHOAI — Kuadrant provides Authorino (auth) and Limitador (rate limiting) for the MaaS gateway. Must be in `kuadrant-system` namespace.

```shell
oc apply -f manifests/kuadrant-instance.yaml
sleep 30
oc get kuadrant kuadrant -n kuadrant-system
```

### 2c. Create maas-default-gateway

Required by RHOAI — the `maas-default-gateway` must exist in `openshift-ingress` before enabling MaaS. The gateway resources ConfigMap increases the Envoy proxy memory to 2Gi (prevents OOMKill when Kuadrant WasmPlugins are loaded).

```shell
oc apply -f manifests/gateway-resources.yaml

export CERT_NAME=$(oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.spec.defaultCertificate.name}' 2>/dev/null || true)
export CERT_NAME="${CERT_NAME:-router-certs}"

envsubst < manifests/maas-gateway.yaml | oc apply -f -
oc wait --for=condition=Programmed gateway/maas-default-gateway -n openshift-ingress --timeout=60s
```

### 2d. Deploy PostgreSQL for maas-api

Required by RHOAI — MaaS needs a PostgreSQL database for API key lifecycle management. RHOAI does not provision the database; you must provide it and create the `maas-db-config` secret.

```shell
oc apply -f manifests/postgres-maas.yaml
oc wait --for=condition=Available deployment/maas-postgres -n maas-db --timeout=120s
```

### 2e. Create maas-api database secret

```shell
oc create secret generic maas-db-config \
  -n redhat-ods-applications \
  --from-literal=DB_CONNECTION_URL='postgresql://maas:maas-db-password@maas-postgres.maas-db.svc.cluster.local:5432/maas?sslmode=disable'
```

---

## Phase 3: Enable MaaS in DataScienceCluster

> When MaaS is enabled, RHOAI **auto-creates**: `maas-api` deployment, `maas-controller` deployment, `default-tenant` Tenant CR, `models-as-a-service` namespace, and EnvoyFilters for gateway→Authorino TLS.

### 3a. Patch DSC to enable MaaS

```shell
oc patch datasciencecluster default-dsc --type=merge -p '{
  "spec": {
    "components": {
      "kserve": {
        "managementState": "Managed",
        "modelsAsService": {
          "managementState": "Managed"
        }
      }
    }
  }
}'
```

### 3b. Wait for MaaS components

```shell
oc wait --for=jsonpath='{.status.conditions[?(@.type=="ModelsAsServiceReady")].status}'=True \
  datasciencecluster/default-dsc --timeout=300s

oc get deployment maas-api -n redhat-ods-applications
oc rollout status deployment/maas-api -n redhat-ods-applications --timeout=120s

oc get tenant default-tenant -n models-as-a-service
```

### 3c. Configure Tenant CR (no external OIDC)

Without `externalOIDC`, the maas-api AuthPolicy uses only `openshift-identities` (K8s TokenReview) for non-API-key authentication. No IdP setup required.

```shell
oc patch tenant default-tenant -n models-as-a-service --type=merge -p '{
  "spec": {
    "gatewayRef": {
      "namespace": "openshift-ingress",
      "name": "maas-default-gateway"
    },
    "apiKeys": {
      "maxExpirationDays": 90
    },
    "telemetry": {
      "enabled": true
    }
  }
}'
```

### 3d. Verify Tenant is Active

```shell
oc get tenant default-tenant -n models-as-a-service -o jsonpath='{.status.phase}' && echo ""
# Expected: Active
```

---

## Phase 4: Deploy Model

### 4a. Create model namespace

```shell
oc create namespace llm || true
```

### 4b. Deploy LLMInferenceService (Simulator)

Uses `llm-d-inference-sim` — CPU-only, no GPU required, returns random tokens. Proves the full MaaS pipeline instantly without model download.

```shell
oc apply -f manifests/model.yaml

# Simulator starts in seconds (no model download)
oc get llminferenceservice facebook-opt-125m-simulated -n llm -w
# Wait for READY=True
```

### 4c. Create MaaSModelRef

```shell
oc apply -f manifests/maas-model-ref.yaml

oc get maasmodelref facebook-opt-125m-simulated -n llm -w
# Wait for READY
```

---

## Phase 5: Create Subscriptions and Auth

### 5a. Create MaaSAuthPolicy

```shell
oc apply -f manifests/maas-auth-policy.yaml
```

### 5b. Create MaaSSubscriptions

```shell
oc apply -f manifests/maas-subscriptions.yaml
```

### 5c. Verify auto-generated policies

```shell
echo "=== MaaS Auth Policies ==="
oc get maasauthpolicy -n models-as-a-service

echo "=== MaaS Subscriptions ==="
oc get maassubscription -n models-as-a-service

echo "=== Auto-generated AuthPolicy ==="
oc get authpolicy -A -o wide

echo "=== Auto-generated TokenRateLimitPolicy ==="
oc get tokenratelimitpolicy -A -o wide
```

---

## Phase 6: Test End-to-End

> **Testing with:** RHCL 1.4.0 + SM 3.3.4, OpenShift token auth (no external IdP).

### 6a. Set variables

```shell
export DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
export MAAS_API_URL="https://maas.${DOMAIN}"
```

### 6b. Test unauthenticated inference (expect 401)

```shell
curl -sk -w "\nHTTP_CODE: %{http_code}\n" \
  -H "Content-Type: application/json" \
  -d '{"model":"facebook/opt-125m","messages":[{"role":"user","content":"Hello"}],"max_tokens":50}' \
  "${MAAS_API_URL}/llm/facebook-opt-125m-simulated/v1/chat/completions"
# Expected: HTTP 401
```

### 6c. Create API key with OpenShift service account token

```shell
# Create a bound token for the premium service account
TOKEN=$(oc create token user -n maas-premium \
  --audience=https://kubernetes.default.svc \
  --audience=maas-default-gateway-sa \
  --duration=1h)

# Create API key
API_KEY_RESPONSE=$(curl -sk -X POST "${MAAS_API_URL}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"name": "premium-sa-key", "description": "Premium tier via OpenShift SA token"}')

echo "${API_KEY_RESPONSE}" | jq '{id, key: .key[:30], keyPrefix, subscription, expiresAt}'
# Expected: HTTP 201, subscription=premium-subscription, key starts with sk-oai-

API_KEY=$(echo "${API_KEY_RESPONSE}" | jq -r '.key')
```

### 6d. Test model endpoints with API key

```shell
echo "=== GET /v1/models ==="
curl -sk -H "Authorization: Bearer ${API_KEY}" \
  "${MAAS_API_URL}/llm/facebook-opt-125m-simulated/v1/models" | jq '.data[] | {id}'
# Expected: {"id": "facebook/opt-125m"}

echo "=== POST /v1/chat/completions ==="
curl -sk -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model":"facebook/opt-125m","messages":[{"role":"user","content":"Hello, how are you?"}],"max_tokens":50}' \
  "${MAAS_API_URL}/llm/facebook-opt-125m-simulated/v1/chat/completions" | jq '{model: .model, content: .choices[0].message.content[:80], usage: .usage}'
# Expected: HTTP 200 with chat response and token usage

echo "=== POST /v1/completions ==="
curl -sk -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model":"facebook/opt-125m","prompt":"Once upon a time","max_tokens":50}' \
  "${MAAS_API_URL}/llm/facebook-opt-125m-simulated/v1/completions" | jq '{model: .model, text: .choices[0].text[:80], usage: .usage}'
# Expected: HTTP 200 with text completion and token usage
```

### 6e. Test token rate limiting (free tier)

```shell
FREE_TOKEN=$(oc create token user -n maas-free \
  --audience=https://kubernetes.default.svc \
  --audience=maas-default-gateway-sa \
  --duration=1h)

FREE_KEY=$(curl -sk -X POST "${MAAS_API_URL}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer ${FREE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"name": "free-tier-test", "description": "Rate limit test"}' | jq -r '.key')

echo "Free key: ${FREE_KEY:0:25}..."
echo "Sending requests until rate limited (free tier = 100 tokens/min)..."

for i in $(seq 1 10); do
  RESP=$(curl -sk -w "\n%{http_code}" -H "Authorization: Bearer ${FREE_KEY}" \
    -H "Content-Type: application/json" \
    -d '{"model":"facebook/opt-125m","messages":[{"role":"user","content":"Tell me a long story about dragons and knights in a magical kingdom far away"}],"max_tokens":50}' \
    "${MAAS_API_URL}/llm/facebook-opt-125m-simulated/v1/chat/completions")
  HTTP_CODE=$(echo "$RESP" | tail -1)
  if [ "$HTTP_CODE" = "200" ]; then
    TOKENS=$(echo "$RESP" | head -1 | jq -r '.usage.total_tokens // "?"')
    echo "  Request $i: HTTP ${HTTP_CODE} — ${TOKENS} tokens"
  else
    echo "  Request $i: HTTP ${HTTP_CODE} — RATE LIMITED (429 Too Many Requests)"
    break
  fi
done
# Expected: first 1-3 requests succeed (200), then 429 after exceeding 100 tokens/min
```

### 6f. Verify MaaS pipeline status

```shell
echo "=== Tenant ===" && oc get tenant default-tenant -n models-as-a-service -o jsonpath='{.status.phase}' && echo ""
echo "=== MaaSAuthPolicy ===" && oc get maasauthpolicy -n models-as-a-service --no-headers
echo "=== MaaSSubscriptions ===" && oc get maassubscription -n models-as-a-service --no-headers
echo "=== AuthPolicy ===" && oc get authpolicy -A -o wide --no-headers
echo "=== TokenRateLimitPolicy ===" && oc get tokenratelimitpolicy -A -o wide --no-headers
echo "=== MaaSModelRef ===" && oc get maasmodelref -n llm --no-headers
```

### Expected Results Summary

| Test | Expected | Validates |
|------|----------|-----------|
| Unauthenticated inference (6b) | HTTP 401 | Wasm shim + Authorino auth enforcement |
| API key creation with SA token (6c) | HTTP 201, `sk-oai-*` key | K8s TokenReview + maas-api auth headers |
| GET /v1/models (6d) | HTTP 200, model list | API key auth + model catalog |
| POST /v1/chat/completions (6d) | HTTP 200, chat response | API key auth + inference |
| POST /v1/completions (6d) | HTTP 200, text completion | API key auth + inference |
| Free tier rate limit (6e) | HTTP 429 after ~100 tokens | Token rate limiting via Limitador |

---

## Cleanup

```shell
# Remove MaaS resources
oc delete -f manifests/maas-subscriptions.yaml
oc delete -f manifests/maas-auth-policy.yaml
oc delete -f manifests/maas-model-ref.yaml
oc delete -f manifests/model.yaml
oc delete namespace llm

# Remove tier namespaces (service accounts are deleted with the namespace)
oc delete namespace maas-free maas-premium maas-enterprise --ignore-not-found

# Remove maas-api database secret
oc delete secret maas-db-config -n redhat-ods-applications

# Remove PostgreSQL
oc delete -f manifests/postgres-maas.yaml

# Remove Gateway
oc delete gateway maas-default-gateway -n openshift-ingress

# Remove Kuadrant
oc delete -f manifests/kuadrant-instance.yaml

# Remove monitoring config
oc delete -f manifests/user-workload-monitoring.yaml

# Disable MaaS in DSC
oc patch datasciencecluster default-dsc --type=merge -p '{
  "spec": {
    "components": {
      "kserve": {
        "modelsAsService": {
          "managementState": "Removed"
        }
      }
    }
  }
}'
```

---

## References

- [RHOAI 3.4 MaaS Official Documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/govern_llm_access_with_models-as-a-service/)
- [Upstream MaaS Documentation](https://opendatahub-io.github.io/models-as-a-service/latest/)
- [rh-aiservices-bu/rhoai-maas-guide](https://github.com/rh-aiservices-bu/rhoai-maas-guide)
- [MaaS CR Reference](../../analysis/maas_cr.md)
- [MaaS Architecture](https://opendatahub-io.github.io/models-as-a-service/latest/concepts/architecture/)
