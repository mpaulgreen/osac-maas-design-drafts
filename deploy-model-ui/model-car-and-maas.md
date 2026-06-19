# Deploy Granite 8B on RHOAI 3.4 via OpenShift AI Dashboard

Deploy IBM Granite 3.1 8B Instruct (FP8) on a single NVIDIA A10G GPU using the OpenShift AI dashboard wizard, then publish to Models-as-a-Service for governed access.

**Model**: `ibm-granite/granite-3.1-8b-instruct` (FP8 quantized)
**Modelcar**: `oci://quay.io/mpaulgreen/granite-3.1-8b-instruct-fp8:gori-1.0`
**GPU**: NVIDIA A10G (24GB VRAM) — FP8 model uses ~8GB, leaving ~16GB for KV-cache
**Runtime**: vLLM NVIDIA CUDA GPU LLMInferenceServiceConfig (RHOAI 3.4 provided)

## Before You Begin

This guide assumes you have completed:

1. **Operator installation** from the BOM ([`bom_rhoai_3.4.md`](../bom_rhoai_3.4.md)) — NFD, NVIDIA GPU Operator, cert-manager, SM 3.2, RHCL 1.3, LWS, RHOAI 3.4, and DataScienceCluster
2. **MaaS deployment Phases 1-3** from the [MaaS deployment guide](../maas/README.md):
   - **Phase 1**: Keycloak MaaS instance with realm, OIDC client, tier groups (`tier-free-users`, `tier-premium-users`, `tier-enterprise-users`), and test users
   - **Phase 2**: MaaS prerequisites — User Workload Monitoring, Kuadrant instance, `maas-default-gateway`, PostgreSQL for maas-api, `maas-db-config` secret
   - **Phase 3**: MaaS enabled in DSC (`modelsAsService: Managed`), Tenant configured with Keycloak OIDC

## Prerequisites

### RHOAI 3.4 with MaaS Enabled

```bash
# Verify RHOAI version
oc get csv -n redhat-ods-operator | grep rhods

# Verify MaaS is enabled in DSC
oc get dsc default-dsc -o jsonpath='{.spec.components.kserve.modelsAsService.managementState}'
# Expected: Managed

# Verify KServe is enabled
oc get dsc default-dsc -o jsonpath='{.spec.components.kserve.managementState}'
# Expected: Managed
```

### NVIDIA GPU Operator Functional

The GPU Operator must be installed with a ClusterPolicy so GPUs are allocatable. Verify:

```bash
# Check GPU nodes report allocatable GPUs
oc get nodes -l nvidia.com/gpu.present=true \
  -o custom-columns='NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'
# Expected: GPU column shows "1" (not "0")

# Check GPU operator pods are running
oc get pods -n nvidia-gpu-operator | grep -E 'driver|device-plugin|validator'
# Expected: All Running

# Verify GPU product label
oc get nodes -l nvidia.com/gpu.present=true \
  -o jsonpath='{.items[0].metadata.labels.nvidia\.com/gpu\.product}'
# Expected: NVIDIA-A10G (or similar)
```

If GPUs show `0` allocatable, install the NVIDIA GPU Operator from OperatorHub and create a ClusterPolicy before proceeding.

### Modelcar Image Accessible

```bash
# Verify the modelcar image is pullable (from a node or build pod)
oc run test-pull --rm -i --restart=Never \
  --image=quay.io/mpaulgreen/granite-3.1-8b-instruct-fp8:gori-1.0 \
  --command -- echo "Image pullable" 2>/dev/null || echo "Check image access"
```

### MaaS Infrastructure Deployed

The MaaS prerequisites from the [MaaS deployment guide](../maas/README.md) must be complete:
- Kuadrant instance ready in `kuadrant-system`
- Gateway `maas-default-gateway` in `openshift-ingress` programmed
- PostgreSQL database for maas-api with `maas-db-config` secret
- Tenant CR in `models-as-a-service` namespace

```bash
# Quick verification
oc get kuadrant kuadrant -n kuadrant-system -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
# Expected: True

oc get gateway maas-default-gateway -n openshift-ingress -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}'
# Expected: True

oc get tenants.maas.opendatahub.io -n models-as-a-service
# Expected: At least one tenant with READY=True
```

---

## Step 1: Configure GPU Hardware Profile

The default hardware profile has no GPU resources. Create a profile for A10G workloads.

### Option A: Via Dashboard UI

1. Navigate to **Settings** → **Hardware profiles**
2. Click **Create hardware profile**
3. Enter:
   - **Name**: `NVIDIA A10G (24GB)`
   - **Description**: `Single NVIDIA A10G GPU with 24GB VRAM — suitable for 7-8B parameter models`
4. Click **Add resource** for each:
   - CPU: identifier `cpu`, default 8, min 4, max 16
   - Memory: identifier `memory`, default 24Gi, min 16Gi, max 32Gi
   - NVIDIA GPU: identifier `nvidia.com/gpu`, default 1, min 1, max 1
5. Add node selector: `nvidia.com/gpu.present: "true"`
6. Add toleration: key `nvidia.com/gpu`, operator `Exists`, effect `NoSchedule`
7. Click **Create**

### Option B: Via CLI

```bash
oc apply -f - <<'EOF'
apiVersion: infrastructure.opendatahub.io/v1
kind: HardwareProfile
metadata:
  name: nvidia-a10g-24gb
  namespace: redhat-ods-applications
spec:
  displayName: "NVIDIA A10G (24GB)"
  description: "Single NVIDIA A10G GPU with 24GB VRAM — suitable for 7-8B parameter models"
  enabled: true
  identifiers:
    - displayName: CPU
      identifier: cpu
      resourceType: CPU
      defaultCount: 8
      minCount: 4
      maxCount: 16
    - displayName: Memory
      identifier: memory
      resourceType: Memory
      defaultCount: 24Gi
      minCount: 16Gi
      maxCount: 32Gi
    - displayName: NVIDIA GPU
      identifier: nvidia.com/gpu
      resourceType: GPU
      defaultCount: 1
      minCount: 1
      maxCount: 1
  nodeSelector:
    nvidia.com/gpu.present: "true"
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
EOF
```

Verify:

```bash
oc get hardwareprofile nvidia-a10g-24gb -n redhat-ods-applications
```

## Step 2: Enable Dashboard Features

Patch the OdhDashboardConfig to enable the MaaS and GenAI Studio dashboard UI:

```bash
oc patch odhdashboardconfig odh-dashboard-config \
  -n redhat-ods-applications \
  --type merge \
  -p '{
    "spec": {
      "dashboardConfig": {
        "modelAsService": true,
        "genAiStudio": true,
        "vLLMDeploymentOnMaaS": true,
        "maasAuthPolicies": true
      }
    }
  }'
```

Enable LlamaStack operator in DSC (required for GenAI Studio):

```bash
oc patch dsc default-dsc --type merge \
  -p '{"spec":{"components":{"llamastackoperator":{"managementState":"Managed"}}}}'
```

Verify the dashboard flags and their corresponding DSC backend components are aligned:

```bash
# Dashboard flags (UI visibility — opt-in, must be true)
oc get odhdashboardconfig odh-dashboard-config -n redhat-ods-applications \
  -o jsonpath='{.spec.dashboardConfig}' | python3 -m json.tool

# DSC backend components (must be Managed for the UI path to be functional)
echo "MaaS backend:      $(oc get dsc default-dsc -o jsonpath='{.spec.components.kserve.modelsAsService.managementState}')"
echo "KServe backend:    $(oc get dsc default-dsc -o jsonpath='{.spec.components.kserve.managementState}')"
echo "LlamaStack backend: $(oc get dsc default-dsc -o jsonpath='{.spec.components.llamastackoperator.managementState}')"
```

Dashboard flags control **UI visibility only**. A mismatch (flag enabled, DSC component `Removed`) creates a visible-but-broken path. Ensure each enabled flag has its backend in `Managed`:

| Dashboard Flag | DSC Component | Required State |
|----------------|---------------|----------------|
| `modelAsService: true` | `kserve.modelsAsService` | Managed |
| `genAiStudio: true` | `llamastackoperator` | Managed |
| `vLLMDeploymentOnMaaS: true` | `kserve` | Managed |

After patching, refresh the OpenShift AI dashboard in your browser. The left navigation should now show **Models-as-a-Service** and **GenAI Studio** sections.

## Step 3: Create a Data Science Project

1. Open the OpenShift AI dashboard: `https://rhods-dashboard-redhat-ods-applications.<cluster-domain>`
2. In the left navigation, click **Projects**
3. Click **Create project**
4. Enter:
   - **Name**: `granite-serving` (or your preferred name)
   - **Description**: "Granite 8B model serving with MaaS governance"
5. Click **Create**

The project creates an OpenShift namespace where the model will be deployed.

## Step 4: Deploy Model via Dashboard Wizard

The wizard has 4 steps: Model details → Model deployment → Advanced settings → Review.

1. From the project page (`granite-serving`), click the **Deployments** tab
2. Click **Deploy model** to open the wizard

### Step 1 of 4: Model Details

3. **Model location**: Select **OCI compliant registry**
4. **Registry host**: `quay.io`
5. **Model URI**: `quay.io/mpaulgreen/granite-3.1-8b-instruct-fp8:gori-1.0` (the `oci://` prefix is auto-added)
6. **Create a connection to this location**: Leave checked
7. **Name**: `granite-3-1-8b-fp8`
8. **Description**: `IBM Granite 3.1 8B Instruct FP8 modelcar`
9. **Model type**: Select **Generative AI model (Example, LLM)**
10. Click **Next**

> For public OCI registries, leave Access type and Secret details empty — no auth needed.

### Step 2 of 4: Model Deployment

11. **Model deployment name**: `granite-3-1-8b-fp8`
12. **Hardware profile**: Select **NVIDIA A10G (24GB)** (the profile created in Step 1)
13. **Deployment resource**: Select **Automatic selection** — the wizard picks the best runtime based on model type and hardware profile (selects vLLM NVIDIA CUDA GPU LLMInferenceServiceConfig for NVIDIA GPUs)
14. **Number of replicas**: `1`
15. Click **Next**

### Step 3 of 4: Advanced Settings

16. **Publish as MaaS**: **Check this** — creates the MaaSModelRef automatically and publishes the model to the MaaS gateway. Note: this automatically disables "Require token authentication" since MaaS handles auth through its own gateway (Authorino + API keys).
17. **Publish as AI asset endpoint**: Optional — check if you want playground access
18. **Add custom runtime arguments**: Leave unchecked — vLLM auto-detects FP8 from the model config. Add `--max-model-len 4096` later if you encounter GPU OOM.
19. Click **Next**

### Step 4 of 4: Review

Verify the summary matches:

| Field | Expected Value |
|-------|---------------|
| Model type | Generative AI model (Example, LLM) |
| Model location | OCI compliant registry |
| Model URI | `oci://quay.io/mpaulgreen/granite-3.1-8b-instruct-fp8:gori-1.0` |
| Project | granite-serving |
| Hardware profile | nvidia-a10g-24gb |
| Deployment resource | vLLM NVIDIA CUDA GPU LLMInferenceServiceConfig |
| Replicas | 1 |
| MaaS endpoint | Yes |
| Token authentication | No |

20. Click **Deploy**

### Wait for Model Ready

The dashboard shows deployment progress. The model pod will:
1. Pull the modelcar OCI image (~8GB, may take several minutes on first pull)
2. Load the model weights into GPU memory
3. Start the vLLM inference server
4. Health checks pass → status becomes **Ready**

Monitor progress:

```bash
# Watch pod status
oc get pods -n granite-serving -w

# Check LLMInferenceService status
oc get llminferenceservice -n granite-serving
# Expected: READY = True
```

Typical startup time: 3-5 minutes (dominated by image pull and model loading).

## Step 5: Verify MaaS Publishing

The **Publish as MaaS** checkbox in Step 4 automatically creates the MaaSModelRef.

1. In the dashboard, navigate to **Projects** → **granite-serving** → **Deployments**
2. The model should show status **Ready** with an inference endpoint
3. Verify via CLI:

```bash
oc get maasmodelref -n granite-serving
# Expected: PHASE = Ready, ENDPOINT shows the gateway URL
```

## Step 6: Configure MaaS Governance

Create authorization policies and subscriptions so Keycloak-authenticated users can access the model through the MaaS gateway with tier-based rate limiting.

> **Prerequisite**: Keycloak MaaS instance must be deployed with the `maas` realm containing tier groups (`tier-free-users`, `tier-premium-users`, `tier-enterprise-users`) and test users. See [MaaS deployment guide Phase 1](../maas/README.md).

> **Note: External OIDC is Technology Preview in RHOAI 3.4.** The Keycloak groups are external OIDC groups — they exist in the JWT `groups` claim, not as OpenShift Group objects. The dashboard Groups dropdown only shows OpenShift groups by default, but you can **type** external group names manually in the field. MaaS validates groups directly from the OIDC token without requiring OpenShift group creation. API key creation for external OIDC users must use `curl` — the dashboard does not support it.

### Create Authorization Policy

1. In the left navigation, scroll down to **Authorization policies**
2. Click **Create authorization policy**
3. Fill in:
   - **Name**: `keycloak-granite-access`
   - **Model**: Select `granite-3-1-8b-fp8` from namespace `granite-serving`
   - **Groups**: The dropdown only shows OpenShift groups. **Type** each Keycloak group name manually and press Enter to add:
     - `tier-free-users`
     - `tier-premium-users`
     - `tier-enterprise-users`
4. Click **Create**

### Create Subscriptions (one per tier)

Create 3 subscriptions matching the Keycloak tier groups:

**Free tier:**

1. In the left navigation, click **Subscriptions**
2. Click **Create subscription**
3. Fill in:
   - **Name**: `free-subscription`
   - **Model**: Select `granite-3-1-8b-fp8` from `granite-serving`
   - **Group**: `tier-free-users`
   - **Token limit**: `100` tokens per minute
   - **Priority**: `0`
4. Click **Create**

**Premium tier:**

1. Click **Create subscription** again
2. Fill in:
   - **Name**: `premium-subscription`
   - **Model**: Select `granite-3-1-8b-fp8` from `granite-serving`
   - **Group**: `tier-premium-users`
   - **Token limit**: `5000` tokens per minute
   - **Priority**: `1`
3. Click **Create**

**Enterprise tier:**

1. Click **Create subscription** again
2. Fill in:
   - **Name**: `enterprise-subscription`
   - **Model**: Select `granite-3-1-8b-fp8` from `granite-serving`
   - **Group**: `tier-enterprise-users`
   - **Token limit**: `10000` tokens per minute
   - **Priority**: `2`
3. Click **Create**

### Verify via CLI

```bash
oc get maasauthpolicy -A
# Expected: keycloak-granite-access in models-as-a-service

oc get maassubscription -A
# Expected: 3 subscriptions — free, premium, enterprise — all Active
```

## Step 7: Verify

### 7a. Check Model Status in Dashboard

1. Navigate to **Projects** → **granite-serving** → **Deployments**
2. The model should show a green checkmark with status **Ready**
3. Check **Subscriptions** → all 3 should show **Active**
4. Check **Authorization policies** → `keycloak-granite-access` should show **Active**

### 7b. Set Environment Variables

```bash
export DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
export HOST="https://maas.${DOMAIN}"
export KEYCLOAK_URL="https://keycloak-maas.${DOMAIN}"
```

### 7c. Verify MaaS Resources

```bash
echo "=== MaaSModelRef ===" && oc get maasmodelref -n granite-serving
echo "=== Auth Policy ===" && oc get maasauthpolicy -A
echo "=== Subscriptions ===" && oc get maassubscription -A
# Expected: MaaSModelRef Ready, 1 auth policy Active, 3 subscriptions Active
```

### 7d. Test Unauthenticated Access (expect 401)

```bash
curl -sk -o /dev/null -w "HTTP %{http_code}\n" \
  "${HOST}/granite-serving/granite-3-1-8b-fp8/v1/chat/completions"
# Expected: HTTP 401 — auth enforcement is working
```

### 7e. Get Keycloak JWT and Create API Key

External OIDC users create API keys via curl (dashboard does not support this for external OIDC — see TP note in Step 6).

```bash
# Get JWT for premium tier test user
ACCESS_TOKEN=$(curl -sk -X POST \
  "${KEYCLOAK_URL}/realms/maas/protocol/openid-connect/token" \
  -d "client_id=maas-client" \
  -d "client_secret=maas-client-secret-value" \
  -d "username=testuser-premium" \
  -d "password=password123" \
  -d "grant_type=password" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Create API key with JWT
API_KEY_RESPONSE=$(curl -sk -X POST "${HOST}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"name":"granite-test-premium","description":"Premium tier test"}')

echo "${API_KEY_RESPONSE}" | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Key: {d[\"keyPrefix\"]}...\nSubscription: {d[\"subscription\"]}')"
# Expected: Key starts with sk-oai-, Subscription: premium-subscription

API_KEY=$(echo "${API_KEY_RESPONSE}" | python3 -c "import sys,json; print(json.load(sys.stdin)['key'])")
```

### 7f. List Models

```bash
curl -sk "${HOST}/granite-serving/granite-3-1-8b-fp8/v1/models" \
  -H "Authorization: Bearer ${API_KEY}" | python3 -c "import sys,json; [print('Model ID:', m['id']) for m in json.load(sys.stdin)['data']]"
# Expected: Model ID: granite-3-1-8b-fp8
```

> **Important**: The model ID is the **deployment name** (`granite-3-1-8b-fp8`), not the HuggingFace model name. Use this ID in all inference requests.

### 7g. Test Inference (Chat Completions)

```bash
curl -sk "${HOST}/granite-serving/granite-3-1-8b-fp8/v1/chat/completions" \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "granite-3-1-8b-fp8",
    "messages": [{"role": "user", "content": "What is Kubernetes in one sentence?"}],
    "max_tokens": 50
  }' | python3 -c "
import sys,json
d=json.load(sys.stdin)
print('Model:', d.get('model'))
print('Response:', d['choices'][0]['message']['content'])
print('Tokens:', d.get('usage'))
"
# Expected: HTTP 200 with a real model-generated response about Kubernetes
```

### 7h. Test Inference (Text Completions)

```bash
curl -sk "${HOST}/granite-serving/granite-3-1-8b-fp8/v1/completions" \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "granite-3-1-8b-fp8",
    "prompt": "OpenShift is",
    "max_tokens": 50
  }' | python3 -c "
import sys,json
d=json.load(sys.stdin)
print('Response:', d['choices'][0]['text'][:200])
print('Tokens:', d.get('usage'))
"
# Expected: HTTP 200 with text completion
```

### 7i. Test Token Rate Limiting (Free Tier)

Create a free-tier API key and send minimal requests until rate limited (100 tokens/min). Use a short prompt and low `max_tokens` to avoid overloading the model.

```bash
FREE_TOKEN=$(curl -sk -X POST "${KEYCLOAK_URL}/realms/maas/protocol/openid-connect/token" \
  -d "client_id=maas-client" -d "client_secret=maas-client-secret-value" \
  -d "username=testuser-free" -d "password=password123" \
  -d "grant_type=password" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

FREE_KEY=$(curl -sk -X POST "${HOST}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer ${FREE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"name":"free-rate-test","description":"Rate limit test"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['key'])")

echo "Free tier (100 tokens/min) — sending small requests..."
for i in $(seq 1 5); do
  RESP=$(curl -sk -w "\n%{http_code}" \
    -H "Authorization: Bearer ${FREE_KEY}" \
    -H "Content-Type: application/json" \
    -d '{"model":"granite-3-1-8b-fp8","messages":[{"role":"user","content":"Hi"}],"max_tokens":30}' \
    "${HOST}/granite-serving/granite-3-1-8b-fp8/v1/chat/completions")
  HTTP_CODE=$(echo "$RESP" | tail -1)
  if [ "$HTTP_CODE" = "200" ]; then
    TOKENS=$(echo "$RESP" | head -1 | python3 -c "import sys,json; print(json.load(sys.stdin).get('usage',{}).get('total_tokens','?'))" 2>/dev/null)
    echo "  Request $i: HTTP ${HTTP_CODE} — ${TOKENS} tokens"
  elif [ "$HTTP_CODE" = "429" ]; then
    echo "  Request $i: HTTP 429 — RATE LIMITED"
    break
  else
    echo "  Request $i: HTTP ${HTTP_CODE}"
  fi
done
# Expected: first 1-2 requests succeed (200, ~70-90 tokens each),
# then 429 after cumulative tokens exceed 100/min.
# Validated result: Request 1: 72 tokens, Request 2: 90 tokens, Request 3: 429
```

### 7j. Check GPU Utilization

```bash
oc exec -n granite-serving \
  $(oc get pod -n granite-serving --no-headers | awk '{print $1}' | head -1) \
  -c main -- nvidia-smi | grep -E 'MiB|GPU Name'
# Expected: ~21000MiB / 23028MiB (model weights + KV cache + CUDA graphs)
```

### 7k. Verify Operator Version Pinning

```bash
oc get csv -n openshift-operators --no-headers | grep -E 'rhcl|authorino|limitador|dns|servicemesh'
# Expected: ALL at v1.3.x, SM at v3.2.x — zero v1.4.0
```

