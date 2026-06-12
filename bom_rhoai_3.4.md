# Bill of Materials — RHOAI 3.4 MaaS Stack

*Deployment guide for OpenShift AI 3.4 with KServe, MaaS (GA), and GPU support for LLM inference.*

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
| cert-manager for Red Hat OpenShift | **1.18.1** (upstream v1.18.4) | `stable-v1` | `redhat-operators` | [OCP 4.20 Docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift) |
| Red Hat Connectivity Link (RHCL) | **1.4.0** | `stable` | `redhat-operators` | [RHCL Docs](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/). Bundles: Authorino Operator 1.4.0, Limitador Operator 1.4.0, DNS Operator 1.4.0 |
| Leader Worker Set Operator | **1.0.x** (GA in OCP 4.20) | `stable-v1.0` | `redhat-operators` | [OCP 4.20 AI Workloads](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/ai_workloads/leader-worker-set-operator) |
| Red Hat OpenShift AI (RHOAI) | **3.4.x** | `stable-3.4` | `redhat-operators` | [Release Notes](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/release_notes/index) |
| Red Hat OpenShift Service Mesh 3 | **3.3.3** | platform-managed (Ingress Operator) | `redhat-operators` | Auto-installed on OCP 4.19; RHOAI uses it directly |

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

### 2.1 — cert-manager (skip if already present)

```shell
# Check if already installed
oc get csv -A | grep cert-manager && echo "Already installed — skip this step"

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
  channel: stable-v1
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

### 2.2 — Red Hat Connectivity Link (RHCL)

RHCL provides Authorino (auth/authz) and Limitador (rate limiting) for MaaS. Starting with RHOAI 3.4, RHCL is included in Red Hat AI SKUs for MaaS use cases.

```shell
oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhcl-operator
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: rhcl-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Wait for RHCL and its bundled operators:
```shell
sleep 60

# Approve Authorino install plan if pending
AUTH_IP=$(oc get installplan -n openshift-operators -o json | \
  jq -r '.items[] | select(.spec.clusterServiceVersionNames[] | contains("authorino")) | select(.spec.approved==false) | .metadata.name')
if [ -n "$AUTH_IP" ]; then
  oc patch installplan "$AUTH_IP" -n openshift-operators --type merge -p '{"spec":{"approved":true}}'
fi
```

Verify all RHCL components:
```shell
oc get csv -n openshift-operators | grep -E 'rhcl|authorino|limitador|dns'
# Expected: 4 operators — rhcl, authorino, limitador, dns — all Succeeded

oc get crd | grep -E 'authpolicies|dnspolicies|ratelimitpolicies|tokenratelimitpolicies'
# Expected: AuthPolicy, DNSPolicy, RateLimitPolicy, TokenRateLimitPolicy CRDs
```

### 2.3 — Leader Worker Set Operator

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

### 2.4 — Red Hat OpenShift AI 3.4

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

Wait for the operator and auto-installed Service Mesh 3:
```shell
sleep 180

# Service Mesh 3 operator install plan may need manual approval
SM_IP=$(oc get installplan -n openshift-operators -o json | \
  jq -r '.items[] | select(.spec.clusterServiceVersionNames[] | contains("servicemeshoperator3")) | select(.spec.approved==false) | .metadata.name')
if [ -n "$SM_IP" ]; then
  oc patch installplan "$SM_IP" -n openshift-operators --type merge -p '{"spec":{"approved":true}}'
fi

# Wait for Service Mesh operator
oc wait --for=condition=CatalogSourcesUnhealthy=False subscription/servicemeshoperator3 -n openshift-operators --timeout=300s 2>/dev/null || echo "Service Mesh subscription not yet created — RHOAI may still be initializing"
```

Verify:
```shell
oc get csv -n redhat-ods-operator | grep rhods
# Expected: rhods-operator.3.4.x   Succeeded

oc get csv -n openshift-operators | grep servicemesh
# Expected: servicemeshoperator3.v3.2.x   Succeeded
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
    codeflare:
      managementState: Removed
    dashboard:
      managementState: Managed
    datasciencepipelines:
      managementState: Removed
    kserve:
      managementState: Managed
    kueue:
      managementState: Removed
    modelmeshserving:
      managementState: Removed
    ray:
      managementState: Removed
    trainingoperator:
      managementState: Removed
    trustyai:
      managementState: Removed
    workbenches:
      managementState: Removed
    mlflowoperator:
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
# Expected: Succeeded

echo "=== 8. KServe API ==="
oc api-resources --api-group=serving.kserve.io --no-headers
# Expected: inferenceservices, llminferenceservices, etc.

echo "=== 9. MaaS CRDs (if MaaS enabled) ==="
oc api-resources --api-group=maas.opendatahub.io --no-headers 2>/dev/null
# Expected: MaaSModelRef, MaaSSubscription, MaaSAuthPolicy, Tenant, AITenant, etc.

echo "=== 10. RHCL / Kuadrant CRDs ==="
oc get crd | grep -E 'authpolicies|ratelimitpolicies|tokenratelimitpolicies' --no-headers
# Expected: AuthPolicy, RateLimitPolicy, TokenRateLimitPolicy
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
│  Service Mesh 3 (Istio 1.27) → gateway API   │
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
