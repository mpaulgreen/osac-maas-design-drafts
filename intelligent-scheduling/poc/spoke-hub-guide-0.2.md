# Release 0.2: Spoke-Hub Multi-Cluster Routing — No MaaS on Inference Clusters

*Deploy MaaS governance + llm-d intelligent multi-cluster routing with simplified inference clusters. Spokes use `openshift-ai-inference` gateway (no MaaS, no Kuadrant, no API keys). Hub handles ALL governance centrally. One model name, one API key — transparent multi-cluster.*

**Release 0.2 vs 0.1**: Spokes drop from 10 operators to 3 + SM auto. No pass-through governance CRs. K8s SA token auth instead of MaaS API keys. Same 5 capabilities: routing, session stickiness, failover, recovery, load-aware scoring.

## Architecture

```
                          osacmaas (Hub — Full MaaS, OCP 4.21)
                    ┌─────────────────────────────────────────────────┐
                    │ maas-default-gateway (tenant auth + rate limit) │
                    │   └─ ExternalModel → Hub Gateway                │
                    │        Hub InferencePool → Hub EPP               │
                    │                    ┌──┴──┐                      │
                    │                proxy-1  proxy-2                 │
                    │                 ┌──────────────┐               │
                    │                 │ nginx :8080  │ (inference)   │
                    │                 │ sidecar:8081 │ (metrics/TLS) │
                    │                 └──────────────┘               │
                    │                (SA token auth, cert-manager TLS)│
                    └───────────────┬──────────┬──────────────────────┘
                                   │          │
                    ┌──────────────┘          └──────────────┐
                    ▼                                        ▼
              spoke1 (NO MaaS, OCP 4.20)             spoke2 (NO MaaS, OCP 4.20)
         ┌──────────────────────┐              ┌──────────────────────┐
         │ openshift-ai-inference│              │ openshift-ai-inference│
         │  + K8s RBAC auth     │              │  + K8s RBAC auth     │
         │  + InferencePool     │              │  + InferencePool     │
         │  + EPP (picks POD)   │              │  + EPP (picks POD)   │
         │  + Granite 8B vLLM   │              │  + Granite 8B vLLM   │
         │  + metrics Route     │              │  + metrics Route     │
         │                      │              │                      │
         │  NO Kuadrant         │              │  NO Kuadrant         │
         │  NO Authorino        │              │  NO Authorino        │
         │  NO MaaS API keys    │              │  NO MaaS API keys    │
         └──────────────────────┘              └──────────────────────┘
```

**What changed from Release 0.1**:

| Aspect | 0.1 (Full MaaS on spokes) | 0.2 (No MaaS on spokes) |
|--------|--------------------------|-------------------------|
| Spoke operators | 10 (RHOAI, RHCL, Authorino, Limitador, DNS, SM, cert-manager, Keycloak, COO, OTel) | 3 + SM auto (RHOAI, cert-manager, GPU) |
| Spoke gateway | `maas-default-gateway` | `openshift-ai-inference` |
| Spoke auth | MaaS API key pass-through | K8s SA token (RBAC) |
| Spoke governance CRs | MaaSModelRef + MaaSAuthPolicy + MaaSSubscription | None |
| Proxy credential | `sk-oai-...` | K8s SA token |
| Spoke deployment | `ansible-playbook --tags full` (683s) | `ansible-playbook --tags full -e enable_maas=false` (~300s) |
| Metrics path | nginx `/metrics` on proxy (port 8080) | Dedicated TLS sidecar (port 8081, cert-manager cert) — nginx not in metrics flow |
| Hub EPP metrics scraping | `http://<pod>:8080/metrics` | `https://<pod>:8081/metrics` (TLS, cert-manager signed) |

## Cluster Roles

| Cluster | OCP | Role | Gateway | MaaS? |
|---------|-----|------|---------|-------|
| **osacmaas** | **4.21** | Hub (governance + routing + Hub EPP) | `maas-default-gateway` | **Yes** — Full MaaS |
| **spoke1** | 4.20 | Spoke (inference only) | `openshift-ai-inference` | **No** |
| **spoke2** | 4.20 | Spoke (inference only) | `openshift-ai-inference` | **No** |

## Prerequisites

- **Hub (osacmaas)**: Full MaaS deployed (`--tags full`), OCP 4.21, GPU operators
- **Spokes (spoke1, spoke2)**: Inference + Dashboard deployed (`--tags full -e enable_maas=false`), OCP 4.19+, GPU operators
- Cross-cluster connectivity: spoke `openshift-ai-inference` NLBs reachable from hub

## Known Issues & Workarounds (inference-only spokes)

### Issue 1: AuthPolicy CRD required even without MaaS

The LLMInferenceService controller has a **hard dependency on the AuthPolicy CRD** (`authpolicies.kuadrant.io`). Without it, the controller refuses to create HTTPRoutes, InferencePool, or EPP — reporting `GatewayPreconditionNotMet: AuthPolicy CRD is not available, please install Red Hat Connectivity Link`.

This is a KServe/RHOAI controller requirement, NOT a runtime requirement. No AuthPolicy CRs are created — the CRD just needs to exist.

**Fix**: Install the AuthPolicy CRD stub from the hub cluster (which has RHCL installed):

```bash
# On hub (osacmaas) — extract the CRD
oc get crd authpolicies.kuadrant.io -o yaml > /tmp/authpolicy-crd.yaml

# Clean metadata for cross-cluster apply
python3 -c "
import yaml
with open('/tmp/authpolicy-crd.yaml') as f:
    crd = yaml.safe_load(f)
for k in ['resourceVersion', 'uid', 'creationTimestamp', 'generation', 'managedFields']:
    crd['metadata'].pop(k, None)
crd.pop('status', None)
with open('/tmp/authpolicy-crd-clean.yaml', 'w') as f:
    yaml.dump(crd, f, default_flow_style=False)
"

# On each spoke — install CRD stub (server-side apply required — CRD is too large for client-side)
oc apply --server-side -f /tmp/authpolicy-crd-clean.yaml --force-conflicts
```

After installing the CRD, restart the llmisvc controller to re-evaluate:
```bash
oc rollout restart deployment/llmisvc-controller-manager -n redhat-ods-applications
```

### Issue 2: `ENABLE_GATEWAY_API_INFERENCE_EXTENSION` env var on istiod

InferencePool as HTTPRoute backendRef requires istiod to have `ENABLE_GATEWAY_API_INFERENCE_EXTENSION=true`. On OCP 4.20, the Ingress Operator (PR #1245) sets this flag when it detects the InferencePool CRD. However:

- The Ingress Operator checks **at startup** — if the InferencePool CRD is installed after the Ingress Operator starts, the flag may not be set
- SM 3.4.0 (RHOAI auto-managed) may have a `ReconcileError` that prevents the env var from propagating to istiod

**Verify**: Check if istiod has the flag:
```bash
oc get pods -n openshift-ingress -l app=istiod -o jsonpath='{.items[0].spec.containers[0].env}' | \
  python3 -c "import sys,json; [print(e['name']+'='+e.get('value','')) for e in json.load(sys.stdin) if 'INFERENCE' in e.get('name','')]"
```

**Fix if missing**: Restart the Ingress Operator (so it re-detects the InferencePool CRD):
```bash
oc delete pod -n openshift-ingress-operator -l name=ingress-operator
```

If still missing after Ingress Operator restart, patch the istiod Deployment directly:
```bash
oc patch deployment istiod-openshift-gateway -n openshift-ingress --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"ENABLE_GATEWAY_API_INFERENCE_EXTENSION","value":"true"}}]'
```

### Issue 3: Sailoperator CRD `storedVersions` conflict

If SM was previously installed at a different version (e.g., SM 3.4.0 then downgraded to 3.1.0), the sailoperator CRDs may have stale `storedVersions` entries that block the install plan: `"risk of data loss updating ztunnels.sailoperator.io: new CRD removes version v1"`.

**Fix**: Delete stale sailoperator CRDs before SM install:
```bash
for crd in $(oc get crd -o name | grep sailoperator); do
  oc delete "$crd" --timeout=15s 2>/dev/null || true
done
```

---

## Part 1: Deploy Spokes (spoke1 + spoke2)

Repeat **all steps** on **each spoke cluster**. No MaaS CRs needed — only model + auth + metrics.

```bash
# For spoke1:
oc config use-context "default/api-spoke1-kni-syseng-devcluster-openshift-com:6443/kube:admin"
# For spoke2:
oc config use-context "default/api-spoke2-kni-syseng-devcluster-openshift-com:6443/kube:admin"
```

### 1.0 Install AuthPolicy CRD stub (prerequisite)

> **Must be done BEFORE deploying LLMInferenceService.** Without this CRD, the controller won't create HTTPRoutes, InferencePool, or EPP.

```bash
# Copy from hub (or use a saved copy)
oc apply --server-side -f /tmp/authpolicy-crd-clean.yaml --force-conflicts

# Verify
oc get crd authpolicies.kuadrant.io --no-headers

# Restart llmisvc controller to detect the CRD
oc rollout restart deployment/llmisvc-controller-manager -n redhat-ods-applications
sleep 15
```

Also verify the GIE flag on istiod:
```bash
oc get pods -n openshift-ingress -l app=istiod -o jsonpath='{.items[0].spec.containers[0].env}' | \
  python3 -c "import sys,json; found=False; [print(e['name']+'='+e.get('value','')) or setattr(type('',(),{'v':True}),'v',True) for e in json.load(sys.stdin) if 'INFERENCE' in e.get('name','')]; found or print('MISSING — apply fix from Known Issues section')"
```

### 1.1 Deploy LLMInferenceService with EPP

> **Gateway ref**: `openshift-ai-inference` (NOT `maas-default-gateway`). All three router fields required.

```bash
oc create namespace granite-serving --dry-run=client -o yaml | oc apply -f -

oc apply -n granite-serving -f - <<'EOF'
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: granite-3-1-8b-fp8
  namespace: granite-serving
spec:
  model:
    name: granite-3.1-8b-instruct-fp8
    uri: oci://quay.io/mpaulgreen/granite-3.1-8b-instruct-fp8:gori-1.0
  replicas: 1
  router:
    gateway:
      refs:
        - name: openshift-ai-inference
          namespace: openshift-ingress
    route: {}
    scheduler: {}
  template:
    nodeSelector:
      nvidia.com/gpu.present: "true"
    tolerations:
      - effect: NoSchedule
        key: nvidia.com/gpu
        operator: Exists
    containers:
      - name: main
        args:
          - --max-model-len=4096
          - --enable-prefix-caching
        resources:
          requests:
            cpu: "4"
            memory: 16Gi
            nvidia.com/gpu: "1"
          limits:
            cpu: "8"
            memory: 32Gi
            nvidia.com/gpu: "1"
EOF
```

Wait for READY (2-9 min):

```bash
oc get llminferenceservice granite-3-1-8b-fp8 -n granite-serving -w
# Ctrl+C when READY=True

echo "=== Verify ==="
echo "--- vLLM pod (expect 2/2) ---" && oc get pods -n granite-serving -l kserve.io/component=workload
echo "--- EPP scheduler (expect 3/3) ---" && oc get pods -n granite-serving | grep scheduler
echo "--- InferencePool ---" && oc get inferencepool -n granite-serving
echo "--- HTTPRoute ---" && oc get httproute -n granite-serving
```

### 1.2 Create ServiceAccount + RBAC for hub proxy access

The `openshift-ai-inference` gateway uses K8s RBAC auth (`kubernetesTokenReview`). The hub proxy needs a ServiceAccount with permission to access the model.

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: hub-proxy-access
  namespace: granite-serving
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: hub-proxy-inference-access
  namespace: granite-serving
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    name: hub-proxy-access
    namespace: granite-serving
EOF
```

### 1.3 Create long-lived SA token + gather connection info

```bash
# Create token (720h = 30 days)
SPOKE_SA_TOKEN=$(oc create token hub-proxy-access -n granite-serving --duration=720h)

# Get gateway NLB
SPOKE_NLB=$(oc get gateway openshift-ai-inference -n openshift-ingress \
  -o jsonpath='{.status.addresses[0].value}')

SPOKE_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')

echo "SAVE: SPOKE_SA_TOKEN=${SPOKE_SA_TOKEN:0:50}..."
echo "SAVE: SPOKE_NLB=${SPOKE_NLB}"
echo "SAVE: SPOKE_DOMAIN=${SPOKE_DOMAIN}"

# Quick test — direct access via NLB (expect 200)
HTTP=$(curl -sk -o /dev/null -w "%{http_code}" \
  "https://${SPOKE_NLB}/granite-serving/granite-3-1-8b-fp8/v1/chat/completions" \
  -H "Authorization: Bearer ${SPOKE_SA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"model":"granite-3.1-8b-instruct-fp8","messages":[{"role":"user","content":"Hello"}],"max_tokens":5}')
echo "Direct spoke access: HTTP $HTTP (expect 200)"
```

### 1.4 Expose spoke EPP metrics (for Hub EPP scraping)

Same pattern as 0.1 — OpenShift Route with SA token auth:

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: epp-metrics-reader
  namespace: granite-serving
---
apiVersion: v1
kind: Secret
metadata:
  name: epp-metrics-token
  namespace: granite-serving
  annotations:
    kubernetes.io/service-account.name: epp-metrics-reader
type: kubernetes.io/service-account-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: epp-metrics-reader-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kserve-metrics-reader-cluster-role
subjects:
  - kind: ServiceAccount
    name: epp-metrics-reader
    namespace: granite-serving
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: epp-metrics
  namespace: granite-serving
spec:
  host: epp-metrics-$(echo ${SPOKE_DOMAIN} | cut -d. -f1).${SPOKE_DOMAIN}
  to:
    kind: Service
    name: granite-3-1-8b-fp8-epp-service
  port:
    targetPort: metrics
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
EOF

SPOKE_METRICS_TOKEN=$(oc get secret epp-metrics-token -n granite-serving -o jsonpath='{.data.token}' | base64 -d)
SPOKE_METRICS_ROUTE=$(oc get route epp-metrics -n granite-serving -o jsonpath='{.spec.host}')

echo "SAVE: SPOKE_METRICS_TOKEN=${SPOKE_METRICS_TOKEN:0:50}..."
echo "SAVE: SPOKE_METRICS_ROUTE=https://${SPOKE_METRICS_ROUTE}/metrics"

# Verify
HTTP=$(curl -sk -o /dev/null -w "%{http_code}" "https://${SPOKE_METRICS_ROUTE}/metrics" \
  -H "Authorization: Bearer ${SPOKE_METRICS_TOKEN}")
echo "Metrics endpoint: HTTP $HTTP (expect 200)"
```

### 1.5 Verify spoke inference

```bash
HTTP=$(curl -sk -o /dev/null -w "%{http_code}" \
  "https://${SPOKE_NLB}/granite-serving/granite-3-1-8b-fp8/v1/chat/completions" \
  -H "Authorization: Bearer ${SPOKE_SA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"model":"granite-3.1-8b-instruct-fp8","messages":[{"role":"user","content":"Hello"}],"max_tokens":5}')
echo "Spoke inference: HTTP $HTTP (expect 200)"
```

### Values to save per spoke

| Value | What it is |
|-------|-----------|
| `SPOKE_SA_TOKEN` | K8s SA token (720h) for hub proxy auth to `openshift-ai-inference` |
| `SPOKE_NLB` | `openshift-ai-inference` gateway NLB hostname |
| `SPOKE_METRICS_TOKEN` | SA token for EPP metrics scraping |
| `SPOKE_METRICS_ROUTE` | EPP metrics OpenShift Route hostname |

> **No MaaS CRs needed.** Unlike 0.1, spokes have no MaaSModelRef, MaaSAuthPolicy, MaaSSubscription, or Tenant CR. The `openshift-ai-inference` gateway's K8s RBAC handles auth via the SA token.

**Repeat Steps 1.1–1.5 on the second spoke.** Save both sets of values.

---

## Part 2: Deploy Hub Components (osacmaas)

Switch to the hub cluster:

```bash
oc config use-context "default/api-osacmaas-kni-syseng-devcluster-openshift-com:6443/kube:admin"
```

### 2.1 Create hub routing namespace + proxies

Replace placeholders with values from Part 1. The proxy uses **SA tokens** (not MaaS API keys) and the **NLB hostname** for both Host header and SNI (no MaaS hostname like `maas.apps.spoke1...`).

```bash
oc create namespace llm-d-hub --dry-run=client -o yaml | oc apply -f -
```

Create proxy ConfigMaps (replace `<SPOKE1_*>` and `<SPOKE2_*>` with actual values):

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: proxy-spoke-1
  namespace: llm-d-hub
data:
  nginx.conf: |
    pid /tmp/nginx.pid;
    events {}
    http {
      client_body_temp_path /tmp/client_temp;
      proxy_temp_path /tmp/proxy_temp;
      fastcgi_temp_path /tmp/fastcgi_temp;
      uwsgi_temp_path /tmp/uwsgi_temp;
      scgi_temp_path /tmp/scgi_temp;

      log_format debug_headers '\$remote_addr [\$time_local] '
                               '"\$request" \$status '
                               'stk=\$http_x_session_token';
      access_log /dev/stdout debug_headers;

      server {
        listen 8080;
        add_header x-cluster spoke-1 always;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host <SPOKE1_NLB>;
        proxy_set_header Authorization "Bearer <SPOKE1_SA_TOKEN>";
        proxy_ssl_verify off;
        proxy_ssl_server_name on;
        proxy_ssl_name <SPOKE1_NLB>;

        location = /healthz { return 200 "ok\n"; }

        location = /upstream-health {
          proxy_pass https://<SPOKE1_NLB>/granite-serving/granite-3-1-8b-fp8/v1/models;
          proxy_connect_timeout 3s;
          proxy_read_timeout 5s;
        }

        location / {
          proxy_pass https://<SPOKE1_NLB>/granite-serving/granite-3-1-8b-fp8/;
        }
      }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: proxy-spoke-2
  namespace: llm-d-hub
data:
  nginx.conf: |
    pid /tmp/nginx.pid;
    events {}
    http {
      client_body_temp_path /tmp/client_temp;
      proxy_temp_path /tmp/proxy_temp;
      fastcgi_temp_path /tmp/fastcgi_temp;
      uwsgi_temp_path /tmp/uwsgi_temp;
      scgi_temp_path /tmp/scgi_temp;

      log_format debug_headers '\$remote_addr [\$time_local] '
                               '"\$request" \$status '
                               'stk=\$http_x_session_token';
      access_log /dev/stdout debug_headers;

      server {
        listen 8080;
        add_header x-cluster spoke-2 always;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host <SPOKE2_NLB>;
        proxy_set_header Authorization "Bearer <SPOKE2_SA_TOKEN>";
        proxy_ssl_verify off;
        proxy_ssl_server_name on;
        proxy_ssl_name <SPOKE2_NLB>;

        location = /healthz { return 200 "ok\n"; }

        location = /upstream-health {
          proxy_pass https://<SPOKE2_NLB>/granite-serving/granite-3-1-8b-fp8/v1/models;
          proxy_connect_timeout 3s;
          proxy_read_timeout 5s;
        }

        location / {
          proxy_pass https://<SPOKE2_NLB>/granite-serving/granite-3-1-8b-fp8/;
        }
      }
    }
EOF
```

### 2.1a Create metrics sidecar infrastructure

The metrics path is **decoupled from nginx** — a dedicated sidecar container serves spoke EPP metrics over TLS on port 8081. OpenShift service-ca auto-generates the TLS certificate.

```bash
# Metrics Service with serving-cert annotation (auto-creates TLS Secret)
oc apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: hub-pool-metrics
  namespace: llm-d-hub
  annotations:
    service.beta.openshift.io/serving-cert-secret-name: metrics-sidecar-tls
spec:
  selector:
    app: hub-pool
  ports:
    - name: metrics
      port: 8081
      targetPort: 8081
EOF

# Wait for cert auto-generation
sleep 10
oc get secret metrics-sidecar-tls -n llm-d-hub --no-headers
```

Create metrics sidecar ConfigMaps (replace `<SPOKE*_METRICS_ROUTE>` and `<SPOKE*_METRICS_TOKEN>` with values from Part 1):

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: metrics-spoke-1
  namespace: llm-d-hub
data:
  nginx.conf: |
    pid /tmp/nginx-metrics.pid;
    events {}
    http {
      client_body_temp_path /tmp/m_client_temp;
      proxy_temp_path /tmp/m_proxy_temp;
      fastcgi_temp_path /tmp/m_fastcgi_temp;
      uwsgi_temp_path /tmp/m_uwsgi_temp;
      scgi_temp_path /tmp/m_scgi_temp;
      server {
        listen 8081 ssl;
        ssl_certificate /etc/tls/tls.crt;
        ssl_certificate_key /etc/tls/tls.key;
        location = /metrics {
          proxy_set_header Host <SPOKE1_METRICS_ROUTE>;
          proxy_set_header Authorization "Bearer <SPOKE1_METRICS_TOKEN>";
          proxy_ssl_verify off;
          proxy_pass https://<SPOKE1_METRICS_ROUTE>/metrics;
        }
        location = /healthz { return 200 "ok\n"; }
      }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: metrics-spoke-2
  namespace: llm-d-hub
data:
  nginx.conf: |
    pid /tmp/nginx-metrics.pid;
    events {}
    http {
      client_body_temp_path /tmp/m_client_temp;
      proxy_temp_path /tmp/m_proxy_temp;
      fastcgi_temp_path /tmp/m_fastcgi_temp;
      uwsgi_temp_path /tmp/m_uwsgi_temp;
      scgi_temp_path /tmp/m_scgi_temp;
      server {
        listen 8081 ssl;
        ssl_certificate /etc/tls/tls.crt;
        ssl_certificate_key /etc/tls/tls.key;
        location = /metrics {
          proxy_set_header Host <SPOKE2_METRICS_ROUTE>;
          proxy_set_header Authorization "Bearer <SPOKE2_METRICS_TOKEN>";
          proxy_ssl_verify off;
          proxy_pass https://<SPOKE2_METRICS_ROUTE>/metrics;
        }
        location = /healthz { return 200 "ok\n"; }
      }
    }
EOF
```

> **Why a sidecar?** hexfusion feedback: "EPP hub to spoke /metrics should probably be mTLS which removes the proxy dependency." The sidecar decouples metrics from inference — nginx handles only inference + health (port 8080), the sidecar handles only metrics over TLS (port 8081). Hub EPP scrapes the sidecar directly.

Create proxy Deployments + Service with metrics sidecar:

```bash
oc apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spoke-1
  namespace: llm-d-hub
  labels:
    app: hub-pool
    cluster: spoke-1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hub-pool
      cluster: spoke-1
  template:
    metadata:
      labels:
        app: hub-pool
        cluster: spoke-1
    spec:
      containers:
        - name: proxy
          image: registry.access.redhat.com/ubi9/nginx-124:latest
          command: ["nginx", "-g", "daemon off;"]
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: inference-cfg
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
          readinessProbe:
            httpGet:
              path: /upstream-health
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 6
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 10
        - name: metrics
          image: registry.access.redhat.com/ubi9/nginx-124:latest
          command: ["nginx", "-g", "daemon off;"]
          ports:
            - containerPort: 8081
          volumeMounts:
            - name: metrics-cfg
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
            - name: tls-cert
              mountPath: /etc/tls
              readOnly: true
      volumes:
        - name: inference-cfg
          configMap:
            name: proxy-spoke-1
        - name: metrics-cfg
          configMap:
            name: metrics-spoke-1
        - name: tls-cert
          secret:
            secretName: metrics-sidecar-tls
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spoke-2
  namespace: llm-d-hub
  labels:
    app: hub-pool
    cluster: spoke-2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hub-pool
      cluster: spoke-2
  template:
    metadata:
      labels:
        app: hub-pool
        cluster: spoke-2
    spec:
      containers:
        - name: proxy
          image: registry.access.redhat.com/ubi9/nginx-124:latest
          command: ["nginx", "-g", "daemon off;"]
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: inference-cfg
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
          readinessProbe:
            httpGet:
              path: /upstream-health
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 6
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 10
        - name: metrics
          image: registry.access.redhat.com/ubi9/nginx-124:latest
          command: ["nginx", "-g", "daemon off;"]
          ports:
            - containerPort: 8081
          volumeMounts:
            - name: metrics-cfg
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
            - name: tls-cert
              mountPath: /etc/tls
              readOnly: true
      volumes:
        - name: inference-cfg
          configMap:
            name: proxy-spoke-2
        - name: metrics-cfg
          configMap:
            name: metrics-spoke-2
        - name: tls-cert
          secret:
            secretName: metrics-sidecar-tls
---
apiVersion: v1
kind: Service
metadata:
  name: hub-pool-svc
  namespace: llm-d-hub
spec:
  selector:
    app: hub-pool
  ports:
    - port: 80
      targetPort: 8080
EOF

oc -n llm-d-hub rollout status deploy/spoke-1 --timeout=120s
oc -n llm-d-hub rollout status deploy/spoke-2 --timeout=120s

echo "=== Verify ===" && oc get pods -n llm-d-hub --no-headers | grep spoke
echo "=== Endpoints (expect 2) ===" && oc get endpoints hub-pool-svc -n llm-d-hub
```

**Expected**: 2 proxy pods `2/2 Running` (proxy + metrics sidecar), 2 endpoints. If `1/2`, the inference proxy readiness failed (upstream-health). If `0/2`, both containers failed.

> **Key differences from 0.1**:
> - Proxy uses `Host: <NLB-hostname>` and `Authorization: Bearer <SA-token>` (not MaaS key)
> - Each proxy pod has 2 containers: `proxy` (inference:8080) + `metrics` (TLS:8081)
> - nginx does NOT handle `/metrics` — the sidecar serves it over TLS with cert-manager cert
> - Hub EPP scrapes `https://<pod-ip>:8081/metrics` (not `http://<pod-ip>:8080/metrics`)

### 2.2 Create ExternalModel + MaaS governance (same as 0.1)

```bash
oc create namespace external-models --dry-run=client -o yaml | oc apply -f -
oc label namespace external-models \
  opendatahub.io/generated-namespace=true \
  maas.opendatahub.io/gateway-access=true --overwrite

oc apply -f - <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: hub-pool-credential
  namespace: external-models
  labels:
    inference.networking.k8s.io/bbr-managed: "true"
type: Opaque
stringData:
  api-key: "dummy-not-used"
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: ExternalModel
metadata:
  name: granite-multi-cluster
  namespace: external-models
  annotations:
    maas.opendatahub.io/tls: "false"
    maas.opendatahub.io/port: "80"
spec:
  provider: openai
  endpoint: hub-routing-data-science-gateway-class.llm-d-hub.svc.cluster.local
  targetModel: granite-3.1-8b-instruct-fp8
  credentialRef:
    name: hub-pool-credential
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSModelRef
metadata:
  name: granite-multi-cluster
  namespace: external-models
spec:
  modelRef:
    kind: ExternalModel
    name: granite-multi-cluster
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSAuthPolicy
metadata:
  name: granite-multi-cluster-access
  namespace: models-as-a-service
spec:
  modelRefs:
    - name: granite-multi-cluster
      namespace: external-models
  subjects:
    groups:
      - name: system:authenticated
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSSubscription
metadata:
  name: granite-multi-cluster-sub
  namespace: models-as-a-service
spec:
  owner:
    groups:
      - name: tier-premium-users
  modelRefs:
    - name: granite-multi-cluster
      namespace: external-models
      tokenRateLimits:
        - limit: 1000
          window: 1m
  priority: 10
EOF

oc rollout restart deployment/maas-controller -n redhat-ods-applications
echo "Waiting 40s for maas-controller restart + TRLP enforcement..."
sleep 40
```

> **ExternalModel endpoint**: `hub-routing-data-science-gateway-class.llm-d-hub.svc.cluster.local` — the Hub Gateway Service, NOT the proxy Service. This routes through InferencePool → Hub EPP.

### 2.3 Deploy Hub EPP + InferencePool + Gateway (same as 0.1)

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: epp
  namespace: llm-d-hub
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: hub-epp
rules:
  - apiGroups: ["inference.networking.k8s.io", "inference.networking.x-k8s.io"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints", "configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: hub-epp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: hub-epp
subjects:
  - kind: ServiceAccount
    name: epp
    namespace: llm-d-hub
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: epp-config
  namespace: llm-d-hub
data:
  config.yaml: |
    apiVersion: inference.networking.x-k8s.io/v1alpha1
    kind: EndpointPickerConfig
    plugins:
      - type: queue-scorer
      - type: kv-cache-utilization-scorer
      - type: session-affinity-scorer
      - type: session-affinity-filter
      - type: max-score-picker
      - type: core-metrics-extractor
        parameters:
          defaultEngine: spoke-epp
          engineConfigs:
            - name: spoke-epp
              queuedRequestsSpec: "inference_pool_average_queue_size"
              kvUsageSpec: "inference_pool_average_kv_cache_utilization"
    schedulingProfiles:
      - name: hub-pool
        plugins:
          - pluginRef: session-affinity-filter
          - pluginRef: session-affinity-scorer
            weight: 5
          - pluginRef: queue-scorer
            weight: 2
          - pluginRef: kv-cache-utilization-scorer
            weight: 2
          - pluginRef: max-score-picker
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: epp
  namespace: llm-d-hub
  labels:
    app: epp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: epp
  template:
    metadata:
      labels:
        app: epp
    spec:
      serviceAccountName: epp
      containers:
        - name: epp
          image: ghcr.io/llm-d/llm-d-router-endpoint-picker:v0.9.0
          args:
            - --pool-name=hub-pool
            - --pool-namespace=llm-d-hub
            - --config-file=/etc/epp/config.yaml
            - --secure-serving=false
            - --metrics-endpoint-auth=false
            - --health-checking=true
            - --refresh-metrics-interval=5s
            - --model-server-metrics-port=8081
            - --model-server-metrics-path=/metrics
            - --model-server-metrics-scheme=https
            - --metrics-staleness-threshold=30s
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - containerPort: 9002
              name: extproc
            - containerPort: 9090
              name: metrics
            - containerPort: 9003
              name: health
          readinessProbe:
            grpc:
              port: 9003
            periodSeconds: 5
            failureThreshold: 3
            timeoutSeconds: 2
          volumeMounts:
            - name: config
              mountPath: /etc/epp
      volumes:
        - name: config
          configMap:
            name: epp-config
---
apiVersion: v1
kind: Service
metadata:
  name: epp
  namespace: llm-d-hub
spec:
  selector:
    app: epp
  ports:
    - name: extproc
      port: 9002
      targetPort: 9002
      appProtocol: kubernetes.io/h2c
    - name: metrics
      port: 9090
      targetPort: 9090
    - name: health
      port: 9003
      targetPort: 9003
---
apiVersion: inference.networking.k8s.io/v1
kind: InferencePool
metadata:
  name: hub-pool
  namespace: llm-d-hub
spec:
  selector:
    matchLabels:
      app: hub-pool
  targetPorts:
    - number: 8080
  endpointPickerRef:
    group: ""
    kind: Service
    name: epp
    port:
      number: 9002
    failureMode: FailOpen
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: hub-routing
  namespace: llm-d-hub
spec:
  gatewayClassName: data-science-gateway-class
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hub-route
  namespace: llm-d-hub
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: hub-routing
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - group: inference.networking.k8s.io
          kind: InferencePool
          name: hub-pool
          port: 8080
EOF

oc -n llm-d-hub rollout status deploy/epp --timeout=120s

echo "=== Verify ==="
echo "--- EPP (expect 1/1) ---" && oc get pods -n llm-d-hub -l app=epp --no-headers
echo "--- InferencePool ---" && oc get inferencepool hub-pool -n llm-d-hub -o jsonpath='{.status.parents[0].conditions[0].reason}' && echo ""
echo "--- Gateway ---" && oc get gateway hub-routing -n llm-d-hub --no-headers
echo "--- HTTPRoute ---" && oc get httproute hub-route -n llm-d-hub --no-headers
```

---

## Part 3: Verification

Run all tests from the **hub cluster** (osacmaas). Same 9-test suite as 0.1.

### 3.1 Setup test variables

```bash
TENANT_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
MAAS_URL="https://maas.${TENANT_DOMAIN}"
MODEL_ID="granite-3.1-8b-instruct-fp8"

# Get tenant API key
JWT=$(curl -sk -X POST \
  "https://keycloak-maas.${TENANT_DOMAIN}/realms/maas/protocol/openid-connect/token" \
  -d "client_id=maas-client&client_secret=maas-client-secret-value&username=testuser-premium&password=password123&grant_type=password" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Restart maas-controller if not done already
# oc rollout restart deployment/maas-controller -n redhat-ods-applications && sleep 40

API_KEY=$(curl -sk -X POST "${MAAS_URL}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer ${JWT}" \
  -H "Content-Type: application/json" \
  -d '{"name":"spoke-hub-0.2-test"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['key'])")

echo "API_KEY=${API_KEY}"
EP="${MAAS_URL}/external-models/granite-multi-cluster/v1/chat/completions"
```

### 3.2 Governance tests (401, 403, 200, 429)

```bash
# Test 1: Unauthenticated (expect 401)
for i in 1 2 3; do
  HTTP=$(curl -sk -o /dev/null -w "%{http_code}" "$EP" -X POST -H "Content-Type: application/json" \
    -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Hi\"}],\"max_tokens\":3}")
  echo "Unauthenticated: HTTP $HTTP"
  [ "$HTTP" = "401" ] && break; sleep 10
done

# Test 2: Invalid key (expect 403)
HTTP=$(curl -sk -o /dev/null -w "%{http_code}" "$EP" -X POST -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-oai-INVALID" \
  -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Hi\"}],\"max_tokens\":3}")
echo "Invalid key: HTTP $HTTP"

# Test 3: Valid inference (expect 200)
curl -sk "$EP" -H "Authorization: Bearer ${API_KEY}" -H "Content-Type: application/json" \
  -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Hi\"}],\"max_tokens\":10}" | python3 -m json.tool

# Test 4: Rate limiting (expect 429)
for i in $(seq 1 12); do
  HTTP=$(curl -sk -o /dev/null -w "%{http_code}" "$EP" -H "Authorization: Bearer ${API_KEY}" -H "Content-Type: application/json" \
    -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Tell me a long story\"}],\"max_tokens\":50}")
  echo "Request $i: HTTP $HTTP"
  [ "$HTTP" = "429" ] && echo "Rate limiting enforced!" && break
done
```

### 3.3-3.6 Capability tests

Same scripts as Release 0.1 guide (`maas-spoke-hub-guide.md` sections 3.3-3.6). The only difference is spoke context names for failover/recovery:

```bash
# Failover: scale down spoke-1
oc config use-context "default/api-spoke1-kni-syseng-devcluster-openshift-com:6443/kube:admin"
oc scale deploy -n granite-serving -l kserve.io/component=workload --replicas=0
oc config use-context "default/api-osacmaas-kni-syseng-devcluster-openshift-com:6443/kube:admin"
# Wait 40s for proxy readiness failure → send 10 requests → expect 10/10 to spoke-2

# Recovery: restore spoke-1
oc config use-context "default/api-spoke1-kni-syseng-devcluster-openshift-com:6443/kube:admin"
oc scale deploy -n granite-serving -l kserve.io/component=workload --replicas=1
oc config use-context "default/api-osacmaas-kni-syseng-devcluster-openshift-com:6443/kube:admin"
# Wait ~210s for model reload → send 10 requests → expect both spokes
```

---

## What This Proves (Release 0.2)

| Capability | Expected |
|------------|----------|
| MaaS governance (401, 403, 200, 429) | Hub enforces ALL auth + rate limiting. Spokes have NO governance layer. |
| Multi-cluster routing | Hub EPP distributes across spokes via `openshift-ai-inference` NLBs |
| Session stickiness | `x-session-token` works same as 0.1 (Hub EPP sets/reads it) |
| Failover | Proxy readiness checks spoke `/v1/models` via SA token → same mechanism |
| Recovery | Automatic when spoke model reloads |
| **Simplified spokes** | 3 operators instead of 10. No Kuadrant, no Authorino, no MaaS API keys, no pass-through CRs |
| **Unified API key** | Single hub API key. No spoke API key management needed |
| **K8s SA token auth** | Standard K8s mechanism. 720h token duration, RBAC-scoped |
| **Decoupled metrics** | Dedicated TLS sidecar (port 8081) serves metrics — nginx NOT in metrics flow. cert-manager cert, no InsecureSkipVerify |
| **Hub EPP scrapes via HTTPS** | `--model-server-metrics-scheme=https --model-server-metrics-port=8081`. Foundation for full mTLS |

## Key Design Decisions (vs 0.1)

| Decision | Rationale |
|----------|-----------|
| `openshift-ai-inference` on spokes | Removes 7 operators. K8s RBAC is sufficient for hub proxy auth |
| K8s SA token (720h) | Long-lived, RBAC-scoped. No MaaS API key management on spokes |
| SA `view` ClusterRole | Minimal permission for inference access. Not cluster-admin |
| Proxy + metrics sidecar | nginx handles inference + health (8080). Dedicated sidecar handles metrics over TLS (8081). Decoupled per hexfusion feedback |
| cert-manager TLS for metrics | OpenShift service-ca auto-generates cert via `serving-cert-secret-name` annotation. Same pattern as Authorino TLS. No InsecureSkipVerify |
| Same Hub EPP config | spoke-epp engine, session-affinity, max-score-picker — gateway-independent |
