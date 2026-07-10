# Deploy Granite 2B from HuggingFace via Dashboard + GenAI Studio Playground

Deploy IBM Granite 3.1 2B Instruct from HuggingFace using the OpenShift AI dashboard, then test inference interactively via GenAI Studio's Playground.

**Model**: `ibm-granite/granite-3.1-2b-instruct` (2B parameters, FP16)
**Source**: HuggingFace Hub → downloaded to PVC → deployed from Cluster storage
**GPU**: NVIDIA A10G (24GB VRAM) — 2B model uses ~4GB, leaving ~20GB for KV-cache
**Runtime**: Auto-selected by RHOAI based on hardware profile

 **Why PVC instead of direct `hf://`?** The direct `hf://` URI download path has a known issue on OpenShift: the KServe storage initializer can't finalize downloaded files due to restricted SCC permissions (`chmod` fails on `/mnt/` temp files). The recommended workaround is to download the model to a PVC via a workbench (which runs with proper permissions), then deploy from Cluster storage. See [KServe issue #5225](https://github.com/KServe/KServe/issues/5225) and [Red Hat AI Validated Models](https://docs.redhat.com/en/documentation/red_hat_ai/3/html-single/validated_models/index) which recommends ModelCar images for RHOAI deployments.

## Before You Begin

This guide assumes:
1. **Operators and BOM deployed** from [`bom_rhoai_3.4.md`](../bom_rhoai_3.4.md)
2. **Dashboard features enabled** — `genAiStudio: true` in OdhDashboardConfig, `llamastackoperator: Managed` in DSC
3. **GPU hardware profile created** — `NVIDIA A10G (24GB)` from [model-car-and-maas.md](model-car-and-maas.md) Step 1
4. **`openshift-ai-inference` gateway created** — see [Known Issue](#known-issue-missing-openshift-ai-inference-gateway) below

### Known Issue: Missing `openshift-ai-inference` Gateway

The RHOAI 3.4 operator sets `kserveIngressGateway: "openshift-ingress/openshift-ai-inference"` in the `inferenceservice-config` ConfigMap, but the GatewayConfig controller only creates a gateway named `data-science-gateway`. This naming mismatch causes LLMInferenceService to get stuck at `Ready: False` with `RefsInvalid` — the HTTPRoute references a gateway that doesn't exist.

**Reported**: [Slack thread](https://redhat-internal.slack.com/archives/C03UGJY6Z1A/p1781812481940349)

**Workaround**: Create the missing gateway manually before deploying models:

```bash
oc apply -f - <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: openshift-ai-inference
  namespace: openshift-ingress
  labels:
    istio.io/rev: openshift-gateway
spec:
  gatewayClassName: data-science-gateway-class
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    allowedRoutes:
      namespaces:
        from: All
    tls:
      mode: Terminate
      certificateRefs:
      - name: data-science-gateway-service-tls
EOF
```

Verify:

```bash
oc get gateway openshift-ai-inference -n openshift-ingress
# Expected: PROGRAMMED = True
```

---

## Step 1: Create a Data Science Project

1. In the OpenShift AI dashboard, click **Projects** in the left navigation
2. Click **Create project**
3. Enter:
   - **Name**: `hf-models`
   - **Description**: `HuggingFace model deployments`
4. Click **Create**

## Step 2: Create a Workbench with PVC

Create a workbench to download the model from HuggingFace. The workbench runs a Jupyter notebook environment with proper permissions to write to a PVC.

1. In the `hf-models` project, click the **Workbenches** tab
2. Click **Create workbench**
3. Fill in:
   - **Name**: `model-downloader`
   - **Image selection**: **Jupyter | Minimal | CPU | Python 3.12**
   - **Version selection**: `3.4`
   - **Hardware profile**: **default-profile** (2 CPU, 4Gi — no GPU needed for downloading)
   - **Cluster storage**: Leave the default PVC (auto-created, mounted at `/opt/app-root/src/`)
4. Click **Create workbench**
5. Wait for the workbench to show **Running** status

## Step 3: Download the Model from HuggingFace

1. Click **Open** on the `model-downloader` workbench to open JupyterLab
2. In JupyterLab, open a **Terminal** (File → New → Terminal)
3. Run the following commands to download the model:

```bash
# Install huggingface_hub and its CLI dependency
!pip install -q huggingface_hub click

# Download the model to a folder on the PVC
# /opt/app-root/src/ is the PVC mount point — files here persist
!hf download ibm-granite/granite-3.1-2b-instruct \
  --local-dir /opt/app-root/src/granite-3.1-2b-instruct

# Verify the download
!ls -lh /opt/app-root/src/granite-3.1-2b-instruct/
# Expected: 13 files — config.json, tokenizer.json, model-00001-of-00002.safetensors (4.7G),
#           model-00002-of-00002.safetensors (65M), etc.
!du -sh /opt/app-root/src/granite-3.1-2b-instruct/
# Expected: ~4.8GB (downloaded at ~150MB/s, ~43 seconds)
```

4. Wait for the download to complete (~2-5 minutes depending on network speed)
5. Close the terminal

 **For gated models** (e.g., Llama): run `huggingface-cli login --token hf_YOUR_TOKEN` before downloading. `ibm-granite/granite-3.1-2b-instruct` is public — no token needed.

## Step 4: Stop the Workbench

The PVC can't be shared between the workbench and the model server simultaneously in ReadWriteOnce mode. Stop the workbench before deploying.

1. Go back to the `hf-models` project → **Workbenches** tab
2. Click **Stop** on the `model-downloader` workbench
3. Wait for it to show **Stopped**

## Step 5: Deploy Model from PVC via Dashboard Wizard

1. In the `hf-models` project, click the **Deployments** tab
2. Click **Deploy model**

### Step 1 of 4: Model Details

3. **Model location**: Select **Cluster storage**
4. **Cluster storage**: Select `model-downloader-storage` (the PVC auto-created for the workbench)
5. **Model path**: Enter `granite-3.1-2b-instruct` after the `pvc://model-downloader-storage/` prefix (the folder name from Step 3)

 The "Storage access mode warning" about ReadWriteMany is expected — ReadWriteOnce works fine with 1 replica.
6. **Name**: `granite-2b-instruct`
7. **Description**: `IBM Granite 3.1 2B Instruct from HuggingFace (PVC)`
8. **Model type**: Select **Generative AI model (Example, LLM)**
9. Click **Next**

### Step 2 of 4: Model Deployment

10. **Model deployment name**: `granite-2b-instruct`
11. **Hardware profile**: Select **NVIDIA A10G (24GB)**
12. **Deployment resource**: Select **Automatic selection**
13. **Number of replicas**: `1`
14. Click **Next**

### Step 3 of 4: Advanced Settings

15. **Publish as AI asset endpoint**: **Check this** — makes the model available in GenAI Studio Playground
16. **Publish as MaaS**: Leave unchecked
17. **Add custom runtime arguments**: Leave unchecked (2B model fits easily)
18. Click **Next**

### Step 4 of 4: Review

Verify:

| Field | Expected Value |
|-------|---------------|
| Model type | Generative AI model (Example, LLM) |
| Model location | Cluster storage |
| Project | hf-models |
| Hardware profile | nvidia-a10g-24gb |
| AI asset endpoint | Yes |
| MaaS endpoint | No |

19. Click **Deploy**

### Wait for Model Ready

The model pod will:
1. Mount the PVC with the pre-downloaded model weights (instant — no download needed)
2. Load weights into GPU memory (~4GB)
3. Start the vLLM inference server
4. Health checks pass → status becomes **Ready**

Startup time: ~2-3 minutes (no download — model is already on disk).

Monitor via CLI:

```bash
oc get pods -n hf-models -w
oc get llminferenceservice -n hf-models
# Expected: READY = True
```

## Step 6: Test in GenAI Studio Playground

Once the model shows **Ready** in the Deployments tab:

1. In the left navigation, expand **Gen AI studio**
2. Click **AI asset endpoints**
3. Select the **hf-models** project from the **Project** dropdown at the top
4. `granite-2b-instruct` should appear with status **Ready**
5. Click **Add to playground**
6. In the **Configure playground** dialog:
   - Model is pre-selected with a green checkmark
   - **Type**: Inference
   - **Max tokens**: 1000 (default, adjust as needed)
7. Click **Create**
8. The Playground opens — type a prompt (e.g., `What is OpenShift in one sentence?`) and click **Send**

### Sample Prompts

| Prompt | Tests |
|--------|-------|
| `What is OpenShift in one sentence?` | Basic knowledge |
| `Write a Python function that adds two numbers` | Code generation |
| `Explain Kubernetes pods to a 5-year-old` | Simplification |

## Step 7: Verify via CLI (Optional)

```bash
# Check model is Ready
oc get llminferenceservice -n hf-models
# Expected: READY = True

# Check model ID (direct pod access — bypasses gateway auth)
POD=$(oc get pod -n hf-models --no-headers | grep granite-2b-instruct-kserve | awk '{print $1}')
oc exec -n hf-models $POD -c main -- \
  curl -sk https://localhost:8000/v1/models | \
  python3 -c "import sys,json; [print('Model ID:', m['id']) for m in json.load(sys.stdin)['data']]"
# Expected: Model ID: granite-2b-instruct

# Test inference (direct pod access — single line to avoid zsh quoting issues)
oc exec -n hf-models $POD -c main -- curl -sk https://localhost:8000/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"granite-2b-instruct","messages":[{"role":"user","content":"What is Red Hat OpenShift?"}],"max_tokens":100}' | python3 -c "import sys,json; d=json.load(sys.stdin); print('Model:', d['model']); print('Response:', d['choices'][0]['message']['content'][:300]); print('Tokens:', d['usage'])"
# Validated: "Red Hat OpenShift is a comprehensive, open-source platform-as-a-service (PaaS)..."
# Tokens: prompt=66, completion=100, total=166
```

 **Note**: The gateway URL requires the service account token (Token authentication is enabled). To access via the gateway:

 ```bash
 SA_TOKEN=$(oc get secret default-name-granite-2b-instruct-sa -n hf-models -o jsonpath='{.data.token}' | base64 -d)
 URL=$(oc get llminferenceservice granite-2b-instruct -n hf-models -o jsonpath='{.status.url}')
 curl -sk "${URL}/v1/chat/completions" -H "Authorization: Bearer ${SA_TOKEN}" -H "Content-Type: application/json" -d '{"model":"granite-2b-instruct","messages":[{"role":"user","content":"Hello"}],"max_tokens":50}'
 ```

---

## Known Issue: Playground Empty Responses

The GenAI Studio Playground's LlamaStack server generates `base_url: http://...` for the model service, but the KServe workload service uses **HTTPS** (`appProtocol: https`). The HTTP connection fails with `APIConnectionError: Connection error`, resulting in empty responses in the Playground chat.

This is a TP bug — the Playground should use `https://` when the LLMInferenceService has TLS enabled. Verify with:

```bash
PGPOD=$(oc get pods -n hf-models --no-headers | grep playground | awk '{print $1}')

# HTTP fails (connection refused)
oc exec -n hf-models $PGPOD -- curl -sk http://granite-2b-instruct-kserve-workload-svc.hf-models.svc.cluster.local:8000/v1/models

# HTTPS works (200)
oc exec -n hf-models $PGPOD -- curl -sk https://granite-2b-instruct-kserve-workload-svc.hf-models.svc.cluster.local:8000/v1/models
```

**Workaround**: Use CLI verification (Step 7) to confirm the model works. The model is fully functional — only the Playground UI is affected.

