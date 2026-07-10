# MaaS on RHOAI 3.4 with Keycloak SSO

*Deploy Models-as-a-Service (GA) on RHOAI 3.4 with Keycloak as the OIDC identity provider for authentication, tier-based token rate limiting, and API key management.*

## Prerequisites

- OpenShift 4.19+ cluster with BOM deployed ([`bom_rhoai_3.4.md`](../bom_rhoai_3.4.md)) — RHOAI 3.4, SM 3.2, RHCL 1.3, KServe, Dashboard
- Let's Encrypt certificates installed on the cluster
- RHBK operator installed on the cluster (from BOM or [`osac_platform_deployment.md`](../../execute/osac_platform_deployment.md))
- `oc`, `jq`, `curl`, `envsubst` installed
- GPU worker node (A10G) available (optional — simulator model runs on CPU)

```shell
export DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
echo "Cluster domain: ${DOMAIN}"
```

## Architecture

```
User (JWT or API key)
    │
    ▼
maas-default-gateway (HTTPS, Let's Encrypt)
    │
    ├── Gate 1: AuthPolicy (Authorino)
    │     ├── Validate Keycloak JWT (OIDC)
    │     ├── OR validate MaaS API key (sk-oai-*)
    │     ├── Extract: groups, userid
    │     └── OPA: check group membership
    │
    ├── Gate 2: TokenRateLimitPolicy (Limitador)
    │     ├── Per-subscription token limits
    │     └── Per-user token counting
    │
    └── Model backend (LLMInferenceService — simulator, CPU-only)
```

## Subscriptions (Token Rate Limits)

| Tier | Tokens/min | Groups | Priority |
|------|-----------|--------|----------|
| Free | 100 | `tier-free-users` | 0 |
| Premium | 50,000 | `tier-premium-users` | 1 |
| Enterprise | 100,000 | `tier-enterprise-users` | 2 |

## Manifests

| File | Purpose |
|------|---------|
| [`keycloak-maas-instance.yaml`](manifests/keycloak-maas-instance.yaml) | Separate Keycloak instance with edge TLS (requires `envsubst`) |
| [`keycloak-realm-maas.yaml`](manifests/keycloak-realm-maas.yaml) | Keycloak realm with OIDC client, tier groups, test users |
| [`user-workload-monitoring.yaml`](manifests/user-workload-monitoring.yaml) | Enable User Workload Monitoring for MaaS metrics |
| [`kuadrant-instance.yaml`](manifests/kuadrant-instance.yaml) | Kuadrant CR (Authorino + Limitador) |
| [`gateway-resources.yaml`](manifests/gateway-resources.yaml) | ConfigMap with gateway proxy memory limits (2Gi — prevents OOMKill) |
| [`maas-gateway.yaml`](manifests/maas-gateway.yaml) | maas-default-gateway with HTTP + HTTPS (requires `envsubst`) |
| [`postgres-maas.yaml`](manifests/postgres-maas.yaml) | Standalone PostgreSQL for maas-api |
| [`model.yaml`](manifests/model.yaml) | LLMInferenceService (llm-d simulator, CPU-only) |
| [`maas-model-ref.yaml`](manifests/maas-model-ref.yaml) | MaaSModelRef — register model with MaaS governance |
| [`maas-auth-policy.yaml`](manifests/maas-auth-policy.yaml) | MaaSAuthPolicy — Keycloak groups to model access |
| [`maas-subscriptions.yaml`](manifests/maas-subscriptions.yaml) | 3 MaaSSubscription CRs (free/premium/enterprise tiers) |

---

## Phase 1: Keycloak MaaS Instance

Deploy a **separate Keycloak instance** dedicated to MaaS — independent from the OSAC Keycloak. Uses edge TLS termination with the cluster's Let's Encrypt certificate so Authorino can verify OIDC discovery.

### 1a. Deploy Keycloak MaaS instance

Deploys a self-contained Keycloak with its own PostgreSQL database and edge-terminated Route (trusted TLS via Let's Encrypt).

```shell
envsubst < manifests/keycloak-maas-instance.yaml | oc apply -f -
```

Wait for PostgreSQL, then the RHBK operator, then Keycloak:
```shell
oc wait --for=condition=Available deployment/keycloak-postgres -n keycloak-maas --timeout=120s
sleep 30
oc wait --for=jsonpath='{.status.conditions[0].status}'=True keycloak/keycloak-maas -n keycloak-maas --timeout=300s
oc get pods -n keycloak-maas --no-headers
```

### 1b. Create MaaS realm and client

```shell
export KEYCLOAK_URL="https://keycloak-maas.${DOMAIN}"
oc apply -f manifests/keycloak-realm-maas.yaml
```

### 1c. Wait for realm import

```shell
oc wait keycloakrealmimport -n keycloak-maas maas-realm --for=condition=Done --timeout=300s
```

### 1d. Verify OIDC discovery (should work WITHOUT `-k` — trusted cert)

```shell
curl -s "${KEYCLOAK_URL}/realms/maas/.well-known/openid-configuration" | jq '{issuer, token_endpoint, jwks_uri}'
```

### 1d. Test token acquisition

```shell
ACCESS_TOKEN=$(curl -sk -X POST "${KEYCLOAK_URL}/realms/maas/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=maas-client" \
  -d "client_secret=maas-client-secret-value" \
  -d "username=testuser-premium" \
  -d "password=password123" \
  -d "grant_type=password" | jq -r '.access_token')

echo $ACCESS_TOKEN | cut -d'.' -f2 | python3 -c "import sys,base64,json; print(json.dumps(json.loads(base64.urlsafe_b64decode(sys.stdin.read().strip()+'=='))))" | jq '{groups, preferred_username}'
```

Expected: `groups: ["tier-premium-users"]`

---

## Phase 2: MaaS Platform Prerequisites

> ***These are prerequisites required by RHOAI 3.4** per the official MaaS documentation ([PDF](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/pdf/govern_llm_access_with_models-as-a-service/Red_Hat_OpenShift_AI_Self-Managed-3.4-Govern_LLM_access_with_Models-as-a-Service-en-US.pdf) | [HTML — requires login](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/govern_llm_access_with_models-as-a-service/)). They must be in place **before** enabling `modelsAsService` in the DataScienceCluster. RHOAI does not auto-provision these.

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

### 3c. Configure Tenant CR with Keycloak OIDC

```shell
oc patch tenant default-tenant -n models-as-a-service --type=merge -p "{
  \"spec\": {
    \"gatewayRef\": {
      \"namespace\": \"openshift-ingress\",
      \"name\": \"maas-default-gateway\"
    },
    \"externalOIDC\": {
      \"issuerUrl\": \"https://keycloak-maas.${DOMAIN}/realms/maas\",
      \"clientId\": \"maas-client\"
    },
    \"apiKeys\": {
      \"maxExpirationDays\": 90
    },
    \"telemetry\": {
      \"enabled\": true
    }
  }
}"
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

> **Tested with:** SM 3.2.5 (Istio 1.26) + RHCL 1.3.4 — the [supported configuration](https://access.redhat.com/articles/7092611). Auth enforcement requires this version pairing. RHCL 1.4.0 + SM 3.2/3.3 does NOT work — see [`maas_auth_enforcement_issue.md`](../maas_auth_enforcement_issue.md).

### 6a. Set variables

```shell
export DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
export KEYCLOAK_URL="https://keycloak-maas.${DOMAIN}"
export MAAS_API_URL="https://maas.${DOMAIN}"
```

### 6b. Test unauthenticated inference (expect 401)

This is the critical test — verifies that the Kuadrant Wasm shim is properly calling Authorino and enforcing auth.

```shell
curl -sk -w "\nHTTP_CODE: %{http_code}\n" \
  -H "Content-Type: application/json" \
  -d '{"model":"facebook/opt-125m","messages":[{"role":"user","content":"Hello"}],"max_tokens":50}' \
  "${MAAS_API_URL}/llm/facebook-opt-125m-simulated/v1/chat/completions"
# Expected: HTTP 401
# If you get HTTP 200, auth enforcement is broken — check operator versions (RHCL must be 1.3.x)
```

### 6c. Create API key (premium tier)

Authenticate with a Keycloak JWT to create an `sk-oai-*` API key. The key inherits the user's group membership for subscription resolution.

```shell
ACCESS_TOKEN=$(curl -s -X POST "${KEYCLOAK_URL}/realms/maas/protocol/openid-connect/token" \
  -d "client_id=maas-client" -d "client_secret=maas-client-secret-value" \
  -d "username=testuser-premium" -d "password=password123" -d "grant_type=password" | jq -r '.access_token')

API_KEY_RESPONSE=$(curl -sk -X POST "${MAAS_API_URL}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"name": "test-key-premium", "description": "Premium tier test key"}')

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

Create a free-tier API key (100 tokens/min) and send requests until rate limited.

```shell
FREE_TOKEN=$(curl -s -X POST "${KEYCLOAK_URL}/realms/maas/protocol/openid-connect/token" \
  -d "client_id=maas-client" -d "client_secret=maas-client-secret-value" \
  -d "username=testuser-free" -d "password=password123" -d "grant_type=password" | jq -r '.access_token')

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

### 6f. Verify Wasm shim and operator versions

```shell
GATEWAY_POD=$(oc get pods -n openshift-ingress --no-headers | grep "maas-default-gateway-data" | awk '{print $1}')

echo "=== Wasm warnings (should be 0) ==="
oc logs -n openshift-ingress "${GATEWAY_POD}" -c istio-proxy --tail=100 | grep -c "allow_on_headers_stop_iteration"

echo "=== Kuadrant stats ==="
oc exec -n openshift-ingress "${GATEWAY_POD}" -c istio-proxy -- pilot-agent request GET /stats 2>/dev/null | grep "kuadrant\."
# Expected: kuadrant.denied > 0 (from the unauthenticated test in 6b)
# Expected: kuadrant.allowed > 0 (from the API key tests in 6d)

echo "=== Operator version pinning ==="
oc get csv -n openshift-operators --no-headers | grep -E 'rhcl|authorino|limitador|dns|servicemesh'
# Expected: ALL at v1.3.x, SM at v3.2.5 — zero v1.4.0

echo "=== Subscription names (no auto-generated) ==="
oc get sub -n openshift-operators --no-headers
# Expected: authorino-operator, limitador-operator, dns-operator, rhcl-operator, servicemeshoperator3
# NOT auto-generated names like authorino-operator-stable-redhat-operators-openshift-marketplace
```

### 6g. Verify MaaS pipeline status

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
| API key creation (6c) | HTTP 201, `sk-oai-*` key | maas-api auth headers (X-MaaS-Username/Group) |
| GET /v1/models (6d) | HTTP 200, model list | API key auth + model catalog |
| POST /v1/chat/completions (6d) | HTTP 200, chat response | API key auth + inference |
| POST /v1/completions (6d) | HTTP 200, text completion | API key auth + inference |
| Free tier rate limit (6e) | HTTP 429 after ~100 tokens | Token rate limiting via Limitador |
| Wasm warnings (6f) | 0 warnings | RHCL 1.3.x compatibility with SM 3.2 |
| Operator versions (6f) | All v1.3.x, SM v3.2.5 | Version pinning held through deployment |

---

## Cleanup

```shell
# Remove MaaS resources
oc delete -f manifests/maas-subscriptions.yaml
oc delete -f manifests/maas-auth-policy.yaml
oc delete -f manifests/maas-model-ref.yaml
oc delete -f manifests/model.yaml
oc delete namespace llm

# Remove Tenant OIDC config
oc patch tenant default-tenant -n models-as-a-service --type=json \
  -p '[{"op":"remove","path":"/spec/externalOIDC"}]'

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

# Remove Keycloak MaaS instance
oc delete namespace keycloak-maas

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
