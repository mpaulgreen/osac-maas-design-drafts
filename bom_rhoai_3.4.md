# Bill of Materials — RHOAI 3.4 MaaS Stack

*Deployment guide for OpenShift AI 3.4 with KServe, MaaS (GA), and GPU support for LLM inference.*

## Validated Operator Inventory

The following operator versions have been validated end-to-end on OCP 4.19.17 with MaaS auth enforcement (401 for unauthenticated requests), API key creation, inference, and token rate limiting all confirmed working.

| Operator | Version | CSV | Channel | Approval |
|----------|---------|-----|---------|----------|
| Red Hat OpenShift AI | **3.4.0** | `rhods-operator.3.4.0` | `stable-3.4` | Automatic |
| Red Hat Connectivity Link | **1.3.4** | `rhcl-operator.v1.3.4` | `stable` | Manual |
| Authorino Operator | **1.3.1** | `authorino-operator.v1.3.1` | `stable` | Manual |
| Limitador Operator | **1.3.1** | `limitador-operator.v1.3.1` | `stable` | Manual |
| DNS Operator | **1.3.1** | `dns-operator.v1.3.1` | `stable` | Manual |
| Red Hat OpenShift Service Mesh 3 | **3.2.5** | `servicemeshoperator3.v3.2.5` | `stable-3.2` | Manual |
| cert-manager Operator | **1.18.1** | `cert-manager-operator.v1.18.1` | `stable-v1.18` | Automatic |
| NVIDIA GPU Operator | **26.3.2** | `gpu-operator-certified.v26.3.2` | `v26.3` | Automatic |
| Node Feature Discovery Operator | **4.19.0** | `nfd.4.19.0-202605290618` | `stable` | Automatic |
| Red Hat build of Leader Worker Set | **1.0.0** | `leader-worker-set.v1.0.0` | `stable-v1.0` | Automatic |
| Red Hat build of Keycloak Operator | **26.4.12** | `rhbk-operator.v26.4.12-opr.1` | `stable-v26.4` | Automatic |

> **Version pinning is critical.** RHCL, Authorino, Limitador, DNS, and SM must use `Manual` approval to prevent OLM from auto-upgrading to v1.4.0 / SM 3.3.x. RHCL 1.4.0's Kuadrant Wasm shim requires `allow_on_headers_stop_iteration` — a field not supported by any SM version on OCP 4.19. See the installation order notes in Phase 2 for details.

---

## Target Stack & Versions

### Platform

| Component | Version | Notes |
|-----------|---------|-------|
| OpenShift Container Platform | **4.19.9+** / 4.20 / 4.21 | [Supported Configs](https://access.redhat.com/articles/rhoai-supported-configs-3.x) |

### GPU Layer

| Component | Version | Channel | Catalog | Notes |
|-----------|---------|---------|---------|-------|
| Node Feature Discovery (NFD) Operator | Platform-aligned (4.19+) | `stable` | `redhat-operators` | [NFD Docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/specialized_hardware_and_driver_enablement/psap-node-feature-discovery-operator) |
| NVIDIA GPU Operator | **v26.3.x** | `v26.3` | `certified-operators` | [Release Notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/release-notes.html) |

### Platform Operators

| Component | Version | Channel | Catalog | Notes |
|-----------|---------|---------|---------|-------|
| cert-manager for Red Hat OpenShift | **1.18.x** | `stable-v1.18` | `redhat-operators` | Pinned to 1.18 per RHCL 1.3 [supported configs](https://access.redhat.com/articles/7092611) |
| Red Hat Connectivity Link (RHCL) | **1.3.x** | `stable` | `redhat-operators` | [RHCL Docs](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.3). Bundles: Authorino, Limitador, DNS operators. See version matrix note below. |
| Leader Worker Set Operator | **1.0.x** (GA in OCP 4.20) | `stable-v1.0` | `redhat-operators` | [OCP 4.20 AI Workloads](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/ai_workloads/leader-worker-set-operator) |
| Red Hat OpenShift AI (RHOAI) | **3.4.x** | `stable-3.4` | `redhat-operators` | [Release Notes](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/release_notes/index) |
| Red Hat OpenShift Service Mesh 3 | **3.2.5** | `stable-3.2` | `redhat-operators` | **Pre-install BEFORE RHOAI** to prevent SM 3.3.3 auto-install. See version matrix note below. |

> **RHCL / Service Mesh Version Matrix — Critical for MaaS Auth Enforcement**
>
> Per the [RHCL Supported Configurations](https://access.redhat.com/articles/7092611):
>
> | RHCL Version | Required Service Mesh | cert-manager |
> |---|---|---|
> | 1.3 | 3.2 | 1.18 |
> | 1.2 | 3.1 | 1.17 |
>
> RHOAI 3.4 MaaS requires [RHCL 1.2 or later](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/govern_llm_access_with_models-as-a-service/). For MaaS auth enforcement to work, the RHCL and Service Mesh versions **must** match the supported matrix. Mismatches cause the Kuadrant Wasm shim to fail silently, resulting in unauthenticated requests being allowed (HTTP 200 instead of 401).
>
> **Known issue:** OCP 4.19 Ingress Operator installs Service Mesh 3.1.x. RHOAI may auto-upgrade it to 3.3.3 via OLM install plans. The RHCL `stable` channel auto-upgrades to 1.4.0. Neither RHCL 1.3 + SM 3.3.3 nor RHCL 1.4.0 + SM 3.3.3 is in the supported matrix. Use `installPlanApproval: Manual` for RHCL to stay on 1.3.x, and approve Service Mesh install plans carefully to stay on 3.2.x. If your versions don't match the matrix, verify auth enforcement (`curl` without credentials should return 401) and file a Red Hat support case if it doesn't.

### MaaS & AI Components (via DataScienceCluster)

| Component | Version | Status in 3.4 | Previous Status |
|-----------|---------|--------------|-----------------|
| Models-as-a-Service (MaaS) | **0.1.1** | GA | TP in 3.3 |
| KServe | Managed | GA | GA |
| Red Hat AI Inference | **3.4.0** | GA | New |
| Distributed Inference (llm-d) | **0.7.1** | GA | GA |

### Tooling Prerequisites

`oc`, `kubectl`, `jq`, `kustomize` v5.7.0+


---

## Phase 1: GPU Prerequisites

> GPU support must be configured BEFORE installing model serving components.

### 1.1 — NFD Operator Install

```shell
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-nfd
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-nfd
  namespace: openshift-nfd
spec:
  targetNamespaces:
  - openshift-nfd
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: nfd
  namespace: openshift-nfd
spec:
  channel: stable
  installPlanApproval: Automatic
  name: nfd
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Verify:
```shell
oc wait --for=condition=CatalogSourcesUnhealthy=False subscription/nfd -n openshift-nfd --timeout=120s
oc get csv -n openshift-nfd | grep nfd
```

### 1.2 — NodeFeatureDiscovery Instance

```shell
oc apply -f - <<'EOF'
apiVersion: nfd.openshift.io/v1
kind: NodeFeatureDiscovery
metadata:
  name: nfd-instance
  namespace: openshift-nfd
spec:
  operand:
    servicePort: 12000
  topologyUpdater: false
  workerConfig:
    configData: |
      core:
        sleepInterval: 60s
      sources:
        pci:
          deviceClassWhitelist:
          - "0200"
          - "03"
          - "12"
          deviceLabelFields:
          - vendor
EOF
```

Verify GPU nodes are labeled:
```shell
sleep 60
oc get nodes -l feature.node.kubernetes.io/pci-10de.present=true
oc get pods -n openshift-nfd -o wide --no-headers
```

### 1.3 — NVIDIA GPU Operator Install

```shell
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: nvidia-gpu-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: nvidia-gpu-operator
  namespace: nvidia-gpu-operator
spec:
  targetNamespaces:
  - nvidia-gpu-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: gpu-operator-certified
  namespace: nvidia-gpu-operator
spec:
  channel: v26.3
  installPlanApproval: Automatic
  name: gpu-operator-certified
  source: certified-operators
  sourceNamespace: openshift-marketplace
EOF
```

Verify:
```shell
oc wait --for=condition=CatalogSourcesUnhealthy=False subscription/gpu-operator-certified -n nvidia-gpu-operator --timeout=120s
oc get csv -n nvidia-gpu-operator | grep gpu
```

### 1.4 — ClusterPolicy Instance

```shell
oc apply -f - <<'EOF'
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  operator:
    defaultRuntime: crio
    runtimeClass: nvidia
  daemonsets:
    labels: {}
    annotations: {}
    tolerations: []
    priorityClassName: system-node-critical
    updateStrategy: RollingUpdate
    rollingUpdate:
      maxUnavailable: "1"
  driver:
    enabled: true
    kernelModuleType: auto
    rdma:
      enabled: false
    licensingConfig:
      nlsEnabled: true
    upgradePolicy:
      autoUpgrade: true
      drain:
        deleteEmptyDir: false
        enable: false
        force: false
        timeoutSeconds: 300
      maxParallelUpgrades: 1
      maxUnavailable: 25%
      podDeletion:
        deleteEmptyDir: false
        force: false
        timeoutSeconds: 300
      waitForCompletion:
        timeoutSeconds: 0
    useOpenShiftDriverToolkit: true
  dcgm:
    enabled: true
  dcgmExporter:
    config:
      name: ""
    enabled: true
    serviceMonitor:
      enabled: true
  devicePlugin:
    config:
      name: ""
    enabled: true
    mps:
      root: /run/nvidia/mps
  gfd:
    enabled: true
  migManager:
    enabled: true
    config:
      name: default-mig-parted-config
    strategy: single
  nodeStatusExporter:
    enabled: true
  sandboxDevicePlugin:
    enabled: true
  sandboxWorkloads:
    defaultWorkload: container
    enabled: false
  toolkit:
    enabled: true
  validator:
    enabled: true
  vfioManager:
    enabled: true
  vgpuDeviceManager:
    enabled: true
  vgpuManager:
    enabled: false
  cdi:
    enabled: true
  gdrcopy:
    enabled: false
  gds:
    enabled: false
EOF
```

Verify (wait up to 10 minutes for driver compilation and pod startup):
```shell
echo "Waiting for GPU Operator pods to be ready (5-10 minutes)..."
while true; do
  NOT_READY=$(oc get pods -n nvidia-gpu-operator --no-headers 2>/dev/null | grep -v -E 'Running|Completed' | wc -l | tr -d ' ')
  TOTAL=$(oc get pods -n nvidia-gpu-operator --no-headers 2>/dev/null | wc -l | tr -d ' ')
  GPU_COUNT=$(oc get nodes -l nvidia.com/gpu.present=true -o jsonpath='{.items[0].status.allocatable.nvidia\.com/gpu}' 2>/dev/null)
  echo "  Pods: $((TOTAL - NOT_READY))/${TOTAL} ready, GPU allocatable: ${GPU_COUNT:-none}"
  if [ "$NOT_READY" = "0" ] && [ "$TOTAL" -gt "0" ] && [ -n "$GPU_COUNT" ] && [ "$GPU_COUNT" != "0" ]; then
    echo "ClusterPolicy fully ready — GPUs allocatable."
    break
  fi
  sleep 15
done
```

### 1.5 — GPU Verification

```shell
oc get nodes -l nvidia.com/gpu.present=true -o json | \
  jq '.items[] | {name: .metadata.name, gpus: .status.allocatable["nvidia.com/gpu"]}'

oc get pods -n nvidia-gpu-operator --no-headers
```

Expected: GPU count > 0, no pods in non-Ready state.

---

## Phase 2: Platform Operator Installation

> **Critical: Installation order matters.** SM must be installed BEFORE RHCL sub-operators. If RHCL subscriptions exist when SM is installed, OLM bundles pending v1.4.0 upgrade plans into the SM install plan — making it impossible to approve SM without also upgrading to v1.4.0. The correct order is: cert-manager → SM → RHCL sub-operators + RHCL → LWS → RHOAI.

### 2.1 — cert-manager (skip if already present)

```shell
# Check if already installed
if oc get csv -A 2>/dev/null | grep -q cert-manager; then
  echo "Already installed -- skip this step"
else
  echo "Not installed -- proceeding with install"
fi

# Install if not present
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cert-manager-operator
  namespace: cert-manager-operator
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-cert-manager-operator
  namespace: cert-manager-operator
spec:
  channel: stable-v1.18
  installPlanApproval: Automatic
  name: openshift-cert-manager-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Verify:
```shell
oc wait --for=condition=CatalogSourcesUnhealthy=False subscription/openshift-cert-manager-operator -n cert-manager-operator --timeout=120s
oc get csv -n cert-manager-operator | grep cert-manager
# Expected: cert-manager-operator.v1.18.1   Succeeded
```

### 2.2 — Service Mesh 3.2 (install FIRST — before RHCL)

SM must be the first operator installed in `openshift-operators` to ensure a clean install plan. If RHCL subscriptions already exist with pending v1.4.0 upgrade plans, OLM bundles them into the SM install plan — making it impossible to get SM 3.2.5 without also upgrading RHCL to v1.4.0.

> **Why `stable-3.2` and not `stable`?** On OCP 4.19, the `stable` channel resolves to SM 3.3.3. The `stable-3.2` channel pins to SM 3.2.x, matching the [RHCL 1.3 supported configuration](https://access.redhat.com/articles/7092611). Use `Manual` approval to prevent any auto-upgrade.

```shell
oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: servicemeshoperator3
  namespace: openshift-operators
spec:
  channel: stable-3.2
  installPlanApproval: Manual
  name: servicemeshoperator3
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: servicemeshoperator3.v3.2.5
EOF
```

Approve the install plan:
```shell
sleep 30

# Approve SM 3.2.x install plan (reject any that bundle v1.4.0)
for plan in $(oc get installplan -n openshift-operators --no-headers | grep -v Complete | awk '{print $1}'); do
  CSVS=$(oc get installplan $plan -n openshift-operators -o jsonpath='{.spec.clusterServiceVersionNames}')
  if echo "$CSVS" | grep -q "v1.4.0"; then
    echo "SKIP $plan (contains v1.4.0 — do NOT approve)"
  elif echo "$CSVS" | grep -q "servicemeshoperator3"; then
    oc patch installplan $plan -n openshift-operators --type merge --patch '{"spec":{"approved":true}}'
    echo "APPROVED $plan: $CSVS"
  fi
done
```

Verify:
```shell
oc wait csv servicemeshoperator3.v3.2.5 -n openshift-operators --for=jsonpath='{.status.phase}'=Succeeded --timeout=300s
oc get csv -n openshift-operators | grep servicemesh
# Expected: servicemeshoperator3.v3.2.5   Succeeded
```

### 2.3 — Red Hat Connectivity Link (RHCL) and Sub-Operators

RHCL provides Authorino (auth/authz) and Limitador (rate limiting) for MaaS. Starting with RHOAI 3.4, RHCL is included in Red Hat AI SKUs for MaaS use cases.

> **Critical: Pre-create sub-operator subscriptions BEFORE RHCL.**
>
> The RHCL bundle declares catalog-level dependencies on `authorino-operator`, `limitador-operator`, and `dns-operator`. Without pre-existing subscriptions, OLM auto-generates subscriptions for these packages on the `stable` channel with `Automatic` approval — which upgrades them to v1.4.0. RHCL 1.4.0's Kuadrant Wasm shim requires `allow_on_headers_stop_iteration`, a field not supported by any SM version on OCP 4.19 (SM 3.2 uses Istio 1.26, SM 3.3 uses Istio 1.27). This breaks ALL MaaS auth enforcement.
>
> The fix: pre-create subscriptions with `Manual` approval pinned to v1.3.x. OLM finds these existing subscriptions and does NOT auto-generate new ones.

#### Step 1: Pre-create sub-operator subscriptions (pinned to v1.3.x)

```shell
oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: authorino-operator
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Manual
  name: authorino-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: authorino-operator.v1.3.1
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: limitador-operator
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Manual
  name: limitador-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: limitador-operator.v1.3.1
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: dns-operator
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Manual
  name: dns-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: dns-operator.v1.3.1
EOF
```

Approve only v1.3.x install plans:
```shell
sleep 30

for plan in $(oc get installplan -n openshift-operators --no-headers | grep -v Complete | awk '{print $1}'); do
  CSVS=$(oc get installplan $plan -n openshift-operators -o jsonpath='{.spec.clusterServiceVersionNames}')
  if echo "$CSVS" | grep -q "v1.4.0"; then
    echo "SKIP $plan (contains v1.4.0 — do NOT approve)"
  else
    oc patch installplan $plan -n openshift-operators --type merge --patch '{"spec":{"approved":true}}'
    echo "APPROVED $plan: $CSVS"
  fi
done

# Wait for sub-operator CSVs
sleep 30
oc get csv -n openshift-operators | grep -E 'authorino|limitador|dns'
# Expected: v1.3.x versions — all Succeeded
```

#### Step 2: Install RHCL operator (pinned to v1.3.4)

```shell
oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhcl-operator
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Manual
  name: rhcl-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: rhcl-operator.v1.3.4
EOF
```

Approve only the v1.3.x install plan:
```shell
sleep 30

for plan in $(oc get installplan -n openshift-operators --no-headers | grep -v Complete | awk '{print $1}'); do
  CSVS=$(oc get installplan $plan -n openshift-operators -o jsonpath='{.spec.clusterServiceVersionNames}')
  if echo "$CSVS" | grep -q "v1.4.0"; then
    echo "SKIP $plan (contains v1.4.0 — do NOT approve)"
  else
    oc patch installplan $plan -n openshift-operators --type merge --patch '{"spec":{"approved":true}}'
    echo "APPROVED $plan: $CSVS"
  fi
done
```

Verify all RHCL components (should be 1.3.x, NOT 1.4.0):
```shell
oc get csv -n openshift-operators | grep -E 'rhcl|authorino|limitador|dns'
# Expected: 4 operators at 1.3.x versions — all Succeeded
# If any show 1.4.0, the auto-upgrade was not blocked — see version matrix note above

oc get sub -n openshift-operators
# Expected: our named subscriptions (authorino-operator, limitador-operator, dns-operator, rhcl-operator, servicemeshoperator3)
# NOT auto-generated ones like authorino-operator-stable-redhat-operators-openshift-marketplace

oc get crd | grep -E 'authpolicies|dnspolicies|ratelimitpolicies|tokenratelimitpolicies'
# Expected: AuthPolicy, DNSPolicy, RateLimitPolicy, TokenRateLimitPolicy CRDs
```

### 2.4 — Leader Worker Set Operator

Required for distributed inference workloads (llm-d).

```shell
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-lws-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-lws-operator
  namespace: openshift-lws-operator
spec:
  targetNamespaces:
  - openshift-lws-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: leader-worker-set
  namespace: openshift-lws-operator
spec:
  channel: stable-v1.0
  installPlanApproval: Automatic
  name: leader-worker-set
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Verify:
```shell
oc wait --for=condition=CatalogSourcesUnhealthy=False subscription/leader-worker-set -n openshift-lws-operator --timeout=120s
oc get csv -n openshift-lws-operator | grep leader-worker-set
```

### 2.5 — Red Hat OpenShift AI 3.4

```shell
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: redhat-ods-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: redhat-ods-operator
  namespace: redhat-ods-operator
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
spec:
  channel: stable-3.4
  installPlanApproval: Automatic
  name: rhods-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Wait for the RHOAI operator:
```shell
sleep 120

oc wait --for=condition=CatalogSourcesUnhealthy=False subscription/rhods-operator -n redhat-ods-operator --timeout=300s
```

> **Important:** The OCP Ingress Operator (`ingress.operator.openshift.io/owned` annotation) takes ownership of the SM subscription and changes the channel from `stable-3.2` to `stable`. This is expected — the `installPlanApproval: Manual` is preserved, so upgrade plans to SM 3.3.x will not auto-install. The SM CSV stays at 3.2.5.
>
> Additionally, RHOAI may auto-create a SM subscription on the `stable` channel. If it does, **delete it** — SM 3.2.5 is already installed from step 2.2.

Check the SM subscription channel:
```shell
oc get sub servicemeshoperator3 -n openshift-operators -o jsonpath='{.spec.channel}' && echo ""
# If "stable" — run the fix below. If "stable-3.2" — skip.
```

If the channel was changed to `stable`, delete and recreate on `stable-3.2`:
```shell
oc delete sub servicemeshoperator3 -n openshift-operators

oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: servicemeshoperator3
  namespace: openshift-operators
spec:
  channel: stable-3.2
  installPlanApproval: Manual
  name: servicemeshoperator3
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: servicemeshoperator3.v3.2.5
EOF
```

Verify:
```shell
oc get csv -n redhat-ods-operator | grep rhods
# Expected: rhods-operator.3.4.x   Succeeded

oc get csv -n openshift-operators | grep servicemesh
# Expected: servicemeshoperator3.v3.2.5   Succeeded (NOT 3.3.3)
```

---

## Phase 3: DataScienceCluster Creation

### 3.1 — DSCInitialization

Created automatically by the RHOAI operator. Wait for it:
```shell
oc wait --for=condition=ReconcileComplete dsci/default-dsci --timeout=600s
```

### 3.2 — DataScienceCluster CR

The DSC enables KServe for model serving and the OpenShift AI Dashboard for tenant self-service. The dashboard provides the web UI for model management, endpoint monitoring, and project navigation — essential on the tenant cluster where admins and users interact with MaaS.


```shell
oc apply -f - <<'EOF'
apiVersion: datasciencecluster.opendatahub.io/v1
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    aipipelines:
      managementState: Removed
    dashboard:
      managementState: Managed
    feastoperator:
      managementState: Removed
    kserve:
      managementState: Managed
    kueue:
      managementState: Removed
    llamastackoperator:
      managementState: Removed
    mlflowoperator:
      managementState: Removed
    modelregistry:
      managementState: Removed
    ray:
      managementState: Removed
    sparkoperator:
      managementState: Removed
    trainer:
      managementState: Removed
    trainingoperator:
      managementState: Removed
    trustyai:
      managementState: Removed
    workbenches:
      managementState: Removed
EOF
```

Wait for readiness:
```shell
oc wait --for=condition=Ready datasciencecluster/default-dsc --timeout=900s
```

### 3.3 — Verification Checklist

Run all verification steps to confirm the full stack:

```shell
echo "=== 1. DSC Phase ==="
oc get datasciencecluster default-dsc -o jsonpath='{.status.phase}' && echo ""
# Expected: Ready

echo "=== 2. KServe Management State ==="
oc get datasciencecluster default-dsc -o jsonpath='{.spec.components.kserve.managementState}' && echo ""
# Expected: Managed

echo "=== 3. Istio Resource ==="
oc get istio openshift-gateway -n openshift-ingress -o jsonpath='{.status.state}' 2>/dev/null && echo ""
# Expected: Healthy

echo "=== 4. GatewayClass ==="
oc get gatewayclass data-science-gateway-class -o jsonpath='{.status.conditions[?(@.type=="Accepted")].status}' && echo ""
# Expected: True

echo "=== 5. Gateway ==="
oc get gateway data-science-gateway -n openshift-ingress -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}' 2>/dev/null && echo ""
# Expected: True

echo "=== 6. KServe Controller ==="
oc get pods -n redhat-ods-applications -l control-plane=kserve-controller-manager --no-headers
# Expected: Running

echo "=== 6b. OpenShift AI Dashboard ==="
oc get pods -n redhat-ods-applications -l app=rhods-dashboard --no-headers
# Expected: Running (1 or more pods)
DASHBOARD_ROUTE=$(oc get route rhods-dashboard -n redhat-ods-applications -o jsonpath='{.spec.host}' 2>/dev/null)
echo "Dashboard URL: https://${DASHBOARD_ROUTE}"
# Expected: accessible via browser with OpenShift SSO

echo "=== 7. Service Mesh Operator ==="
oc get csv -n openshift-operators | grep servicemeshoperator3
# Expected: servicemeshoperator3.v3.2.5   Succeeded (NOT 3.3.3)

echo "=== 8. KServe API ==="
oc api-resources --api-group=serving.kserve.io --no-headers
# Expected: inferenceservices, llminferenceservices, etc.

echo "=== 9. MaaS CRDs (if MaaS enabled) ==="
oc api-resources --api-group=maas.opendatahub.io --no-headers 2>/dev/null
# Expected: MaaSModelRef, MaaSSubscription, MaaSAuthPolicy, Tenant, AITenant, etc.

echo "=== 10. RHCL / Kuadrant CRDs ==="
oc get crd --no-headers | grep -E 'authpolicies|ratelimitpolicies|tokenratelimitpolicies'
# Expected: AuthPolicy, RateLimitPolicy, TokenRateLimitPolicy

echo "=== 11. RHCL Version Pinning (critical) ==="
oc get csv -n openshift-operators | grep -E 'rhcl|authorino|limitador|dns'
# Expected: ALL at v1.3.x — zero v1.4.0
# If any show v1.4.0, the pre-create strategy in step 2.3 was not followed correctly

echo "=== 12. Subscription Names ==="
oc get sub -n openshift-operators --no-headers
# Expected: our named subscriptions only:
#   authorino-operator, limitador-operator, dns-operator, rhcl-operator, servicemeshoperator3
# NOT auto-generated names like authorino-operator-stable-redhat-operators-openshift-marketplace
```

---

## What This Validates

If all verification steps pass, the following stack is operational:

```
┌─────────────────────────────────────────────┐
│ GPU Layer                                    │
│  NFD Operator → labels GPU nodes             │
│  NVIDIA GPU Operator → drivers + toolkit     │
├─────────────────────────────────────────────┤
│ Platform Layer                               │
│  cert-manager → TLS certificates             │
│  Service Mesh 3.2 (Istio 1.26) → gateway API  │
│  RHCL → Authorino (auth) + Limitador (rate)  │
│  Leader Worker Set → distributed inference   │
├─────────────────────────────────────────────┤
│ AI Layer                                     │
│  RHOAI 3.4 → DataScienceCluster             │
│  KServe → model serving (LLMInferenceService)│
│  Dashboard → tenant self-service web UI      │
│  MaaS 0.1.1 (GA) → governance + API keys    │
└─────────────────────────────────────────────┘
```

This stack is the foundation for the OSAC AAP `maas_shared` template role (Section 4c of the integrated design). The AAP template assumes this stack is pre-installed on the shared MaaS cluster and provisions tenant entitlements (AITenant CRs, Keycloak realms, MaaS policies) on top of it.

---

## References

- [RHOAI 3.4 Release Notes](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/release_notes/index)
- [RHOAI 3.4 New Features](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/release_notes/new-features-and-enhancements_relnotes)
- [RHOAI Supported Configurations 3.x](https://access.redhat.com/articles/rhoai-supported-configs-3.x)
- [RHCL 1.3 Documentation](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.3)
- [NVIDIA GPU Operator on OpenShift](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/index.html)
- [NVIDIA GPU Operator Release Notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/release-notes.html)
- [cert-manager for OCP 4.20](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift)
- [OpenShift Service Mesh 3.2](https://www.redhat.com/en/blog/introducing-openshift-service-mesh-32-istios-ambient-mode)
- [Leader Worker Set (OCP 4.20)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/ai_workloads/leader-worker-set-operator)
- [NFD Operator (OCP 4.19)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/specialized_hardware_and_driver_enablement/psap-node-feature-discovery-operator)
