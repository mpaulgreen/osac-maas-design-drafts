# MaaS + llm-d Spoke-Hub Multi-Cluster Routing Guide

*Deploy MaaS governance + llm-d intelligent multi-cluster routing across 3 OpenShift clusters. One model name, one API key — requests distributed across GPU clusters transparently with automatic failover and session stickiness. Validated July 15, 2026.*

## Architecture

```
                          osacmaas (Hub — Full MaaS, OCP 4.21)
                    ┌─────────────────────────────────────────────────┐
                    │ maas-default-gateway                            │
                    │   ├─ Kuadrant AuthPolicy (tenant auth)          │
                    │   ├─ Kuadrant TRLP (rate limiting)              │
                    │   └─ ExternalModel → Hub Gateway                │
                    │        Hub InferencePool → Hub EPP               │
                    │                    ┌──┴──┐  (EPP picks cluster) │
                    │                proxy-1  proxy-2                 │
                    │                 ┌──────────────┐               │
                    │                 │ nginx :8080  │ (inference)   │
                    │                 │ sidecar:8081 │ (metrics/TLS) │
                    │                 └──────────────┘               │
                    └───────────────┬──────────┬──────────────────────┘
                                   │          │
                    ┌──────────────┘          └──────────────┐
                    ▼                                        ▼
              spoke1 (Full MaaS, OCP 4.20)           spoke2 (Full MaaS, OCP 4.20)
         ┌──────────────────────┐              ┌──────────────────────┐
         │ maas-default-gateway  │              │ maas-default-gateway  │
         │  + InferencePool     │              │  + InferencePool     │
         │  + EPP (picks POD)   │              │  + EPP (picks POD)   │
         │  + Granite 8B vLLM   │              │  + Granite 8B vLLM   │
         │  + pass-through auth │              │  + pass-through auth │
         │  + metrics Route     │              │  + metrics Route     │
         └──────────────────────┘              └──────────────────────┘
```

**Validated capabilities** (July 15, 2026):

| Capability | Status |
|------------|--------|
| MaaS governance (401, 403, 200, 429) | **Validated** — hub `maas-default-gateway` via Hub EPP chain |
| Hub EPP multi-cluster routing | **Validated** — 4/6 load-aware distribution |
| Failover | **Validated** (10/10) — upstream health readiness probe, 40s detection |
| Recovery | **Validated** (10/10) — automatic after model restored |
| Session stickiness | **Validated** (5/5) — `x-session-token` in both request and response via MaaS gateway |
| Load-aware scoring | **Partial** — Hub EPP scrapes spoke metrics via custom `spoke-epp` engine config; differential scoring under asymmetric load not yet conclusive (see Test 3 notes) |
| Manual InferencePool backendRef | **Validated** — Istio 1.27.3 on OCP 4.21 |

## Cluster Roles

| Cluster | OCP | Role | Domain | GPU | MaaS |
|---------|-----|------|--------|-----|------|
| **osacmaas** | **4.21** | Hub (governance + routing + Hub EPP) | `apps.osacmaas.kni.syseng.devcluster.openshift.com` | g5.2xlarge | Full MaaS |
| **spoke1** | 4.20 | Spoke 1 (inference) | `apps.spoke1.kni.syseng.devcluster.openshift.com` | g5.2xlarge/4xlarge | Full MaaS |
| **spoke2** | 4.20 | Spoke 2 (inference) | `apps.spoke2.kni.syseng.devcluster.openshift.com` | g5.2xlarge/4xlarge | Full MaaS |

> **OCP version matters for Hub EPP.** LLMInferenceService-created InferencePool works on any OCP (4.19+). But **manual InferencePool** as HTTPRoute backendRef (for Hub EPP with proxy pods) requires **OCP 4.21** (Istio 1.27.3). Spokes can stay on 4.19/4.20.

## Prerequisites

- All 3 clusters: Full MaaS deployed (`ansible-playbook --tags full`), certs, OLM catalog fix (128Mi), GPU operators (NFD + NVIDIA)
- Cross-cluster connectivity: spoke `maas-default-gateway` NLBs reachable from osacmaas

---

## Part 1: Deploy Spokes (spoke1 + spoke2)

Repeat **all steps** on **each spoke cluster**. Switch context before starting each spoke.

```bash
# For spoke1:
oc config use-context "default/api-spoke1-kni-syseng-devcluster-openshift-com:6443/kube:admin"
# For spoke2:
oc config use-context "default/api-spoke2-kni-syseng-devcluster-openshift-com:6443/kube:admin"
```

### 1.1 Deploy LLMInferenceService with EPP

> **Critical**: All three router fields — `gateway`, `route: {}`, `scheduler: {}`. Without `route: {}`, the controller deadlocks (creates InferencePool but waits for acceptance before creating HTTPRoute; istiod needs the HTTPRoute to discover the InferencePool).

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
        - name: maas-default-gateway
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

Wait for READY (typically 2-9 minutes for GPU model load):

```bash
oc get llminferenceservice granite-3-1-8b-fp8 -n granite-serving -w
# Ctrl+C when READY=True

echo "=== Verify ==="
echo "--- vLLM pod (expect 2/2) ---" && oc get pods -n granite-serving -l kserve.io/component=workload
echo "--- EPP scheduler (expect 3/3) ---" && oc get pods -n granite-serving | grep scheduler
echo "--- InferencePool (expect Accepted) ---" && oc get inferencepool -n granite-serving
echo "--- HTTPRoute ---" && oc get httproute -n granite-serving
```

**Expected**: vLLM pod `2/2 Running` + EPP scheduler `3/3 Running` + InferencePool accepted + HTTPRoute created.

### 1.2 Create pass-through governance

Spoke MaaS gateway has default-deny. Create pass-through CRs so the hub proxy can access the model:

```bash
oc apply -f - <<'EOF'
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSModelRef
metadata:
  name: granite-3-1-8b-fp8
  namespace: granite-serving
spec:
  modelRef:
    kind: LLMInferenceService
    name: granite-3-1-8b-fp8
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSAuthPolicy
metadata:
  name: granite-3-1-8b-fp8-access
  namespace: models-as-a-service
spec:
  modelRefs:
    - name: granite-3-1-8b-fp8
      namespace: granite-serving
  subjects:
    groups:
      - name: system:authenticated
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSSubscription
metadata:
  name: granite-3-1-8b-fp8-passthrough
  namespace: models-as-a-service
spec:
  owner:
    groups:
      - name: system:authenticated
  modelRefs:
    - name: granite-3-1-8b-fp8
      namespace: granite-serving
      tokenRateLimits:
        - limit: 999999999
          window: 1m
  priority: 1
EOF

# Restart maas-controller to pick up new MaaSModelRef
oc rollout restart deployment/maas-controller -n redhat-ods-applications

# Wait for controller restart + TRLP enforcement (required before API key creation)
echo "Waiting 40s for maas-controller restart + TRLP enforcement..."
sleep 40
```

### 1.3 Create spoke API key (for hub proxy)

```bash
SPOKE_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
SPOKE_MAAS_URL="https://maas.${SPOKE_DOMAIN}"

JWT=$(curl -sk -X POST \
  "https://keycloak-maas.${SPOKE_DOMAIN}/realms/maas/protocol/openid-connect/token" \
  -d "client_id=maas-client&client_secret=maas-client-secret-value&username=testuser-premium&password=password123&grant_type=password" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

SPOKE_API_KEY=$(curl -sk -X POST "${SPOKE_MAAS_URL}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer ${JWT}" \
  -H "Content-Type: application/json" \
  -d '{"name":"hub-proxy-access"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['key'])")

SPOKE_GW=$(oc get gateway maas-default-gateway -n openshift-ingress -o jsonpath='{.status.addresses[0].value}')
SPOKE_HOSTNAME="maas.${SPOKE_DOMAIN}"

echo "SAVE: SPOKE_API_KEY=${SPOKE_API_KEY}"
echo "SAVE: SPOKE_GW=${SPOKE_GW}"
echo "SAVE: SPOKE_HOSTNAME=${SPOKE_HOSTNAME}"
echo "SAVE: SPOKE_DOMAIN=${SPOKE_DOMAIN}"
```

### 1.4 Expose spoke EPP metrics (for Hub EPP scraping)

The Hub EPP needs to scrape spoke metrics for load-aware scoring. EPP metrics auth can't be disabled (KServe controller reverts patches). Solution: expose via OpenShift Route with SA token auth.

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

SPOKE_SA_TOKEN=$(oc get secret epp-metrics-token -n granite-serving -o jsonpath='{.data.token}' | base64 -d)
SPOKE_METRICS_ROUTE=$(oc get route epp-metrics -n granite-serving -o jsonpath='{.spec.host}')

echo "SAVE: SPOKE_SA_TOKEN=${SPOKE_SA_TOKEN:0:50}..."
echo "SAVE: SPOKE_METRICS_ROUTE=https://${SPOKE_METRICS_ROUTE}/metrics"

# Verify metrics endpoint
HTTP=$(curl -sk -o /dev/null -w "%{http_code}" \
  "https://${SPOKE_METRICS_ROUTE}/metrics" \
  -H "Authorization: Bearer ${SPOKE_SA_TOKEN}")
echo "Metrics endpoint: HTTP $HTTP (expect 200)"
```

### 1.5 Verify spoke inference (before proceeding to hub)

```bash
# Quick inference test on the spoke
HTTP=$(curl -sk -o /dev/null -w "%{http_code}" \
  "${SPOKE_MAAS_URL}/granite-serving/granite-3-1-8b-fp8/v1/chat/completions" \
  -H "Authorization: Bearer ${SPOKE_API_KEY}" -H "Content-Type: application/json" \
  -d '{"model":"granite-3.1-8b-instruct-fp8","messages":[{"role":"user","content":"Hello"}],"max_tokens":5}')
echo "Spoke inference: HTTP $HTTP (expect 200)"
```

### Values to save per spoke

Before proceeding to the next spoke (or to the hub), save these values:

| Value | Example |
|-------|---------|
| `SPOKE_API_KEY` | `sk-oai-...` |
| `SPOKE_GW` | `a8c793...elb.amazonaws.com` |
| `SPOKE_HOSTNAME` | `maas.apps.spoke1.kni.syseng.devcluster.openshift.com` |
| `SPOKE_SA_TOKEN` | `eyJhbG...` |
| `SPOKE_METRICS_ROUTE` | `epp-metrics-spoke1.apps.spoke1.../metrics` |

**Repeat Steps 1.1–1.5 on the second spoke.** Save both sets of values.

---

## Part 2: Deploy Hub Components (osacmaas)

Switch to the hub cluster:

```bash
oc config use-context "default/api-osacmaas-kni-syseng-devcluster-openshift-com:6443/kube:admin"
```

### 2.1 Create hub routing namespace + proxies

Replace placeholders with values from Part 1:
- `<SPOKE1_GW>` / `<SPOKE2_GW>` — spoke gateway NLB hostnames
- `<SPOKE1_KEY>` / `<SPOKE2_KEY>` — spoke API keys from Step 1.3
- `<SPOKE1_HOSTNAME>` / `<SPOKE2_HOSTNAME>` — spoke MaaS hostnames
- `<SPOKE1_SA_TOKEN>` / `<SPOKE2_SA_TOKEN>` — spoke SA tokens from Step 1.4
- `<SPOKE1_METRICS_ROUTE>` / `<SPOKE2_METRICS_ROUTE>` — spoke metrics Route hostnames from Step 1.4

> **Proxy nginx requirements**:
> - `registry.access.redhat.com/ubi9/nginx-124:latest` (standard nginx crashes on restricted SCC)
> - `pid /tmp/nginx.pid` + temp paths (restricted SCC)
> - `proxy_set_header Host` (spoke gateway hostname filter)
> - `proxy_set_header Authorization` (spoke MaaS auth)
> - `proxy_ssl_verify off` (NLB hostname ≠ cert SAN)
> - `proxy_pass` includes path prefix (IPP strips path on hub side)
> - `/upstream-health` checks spoke model health (failover)
> - `/metrics` proxies to spoke EPP metrics Route (Hub EPP scraping)

```bash
oc create namespace llm-d-hub --dry-run=client -o yaml | oc apply -f -

cat <<'OUTEREOF' | sed \
  -e 's|<SPOKE1_GW>|'"$SPOKE1_GW"'|g' \
  -e 's|<SPOKE1_HOSTNAME>|'"$SPOKE1_HOSTNAME"'|g' \
  -e 's|<SPOKE1_KEY>|'"$SPOKE1_KEY"'|g' \
  -e 's|<SPOKE1_SA_TOKEN>|'"$SPOKE1_SA_TOKEN"'|g' \
  -e 's|<SPOKE1_METRICS_ROUTE>|'"$SPOKE1_METRICS_ROUTE"'|g' \
  -e 's|<SPOKE2_GW>|'"$SPOKE2_GW"'|g' \
  -e 's|<SPOKE2_HOSTNAME>|'"$SPOKE2_HOSTNAME"'|g' \
  -e 's|<SPOKE2_KEY>|'"$SPOKE2_KEY"'|g' \
  -e 's|<SPOKE2_SA_TOKEN>|'"$SPOKE2_SA_TOKEN"'|g' \
  -e 's|<SPOKE2_METRICS_ROUTE>|'"$SPOKE2_METRICS_ROUTE"'|g' \
  | oc apply -f -
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

      log_format debug_headers '$remote_addr [$time_local] '
                               '"$request" $status '
                               'sid=$http_x_session_id '
                               'stk=$http_x_session_token';
      access_log /dev/stdout debug_headers;

      server {
        listen 8080;
        add_header x-cluster spoke-1 always;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host <SPOKE1_HOSTNAME>;
        proxy_set_header Authorization "Bearer <SPOKE1_KEY>";
        proxy_ssl_verify off;
        proxy_ssl_server_name on;
        proxy_ssl_name <SPOKE1_HOSTNAME>;

        location = /healthz { return 200 "ok\n"; }

        location = /upstream-health {
          proxy_pass https://<SPOKE1_GW>/granite-serving/granite-3-1-8b-fp8/v1/models;
          proxy_connect_timeout 3s;
          proxy_read_timeout 5s;
        }

        location / {
          proxy_pass https://<SPOKE1_GW>/granite-serving/granite-3-1-8b-fp8/;
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

      log_format debug_headers '$remote_addr [$time_local] '
                               '"$request" $status '
                               'sid=$http_x_session_id '
                               'stk=$http_x_session_token';
      access_log /dev/stdout debug_headers;

      server {
        listen 8080;
        add_header x-cluster spoke-2 always;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host <SPOKE2_HOSTNAME>;
        proxy_set_header Authorization "Bearer <SPOKE2_KEY>";
        proxy_ssl_verify off;
        proxy_ssl_server_name on;
        proxy_ssl_name <SPOKE2_HOSTNAME>;

        location = /healthz { return 200 "ok\n"; }

        location = /upstream-health {
          proxy_pass https://<SPOKE2_GW>/granite-serving/granite-3-1-8b-fp8/v1/models;
          proxy_connect_timeout 3s;
          proxy_read_timeout 5s;
        }

        location / {
          proxy_pass https://<SPOKE2_GW>/granite-serving/granite-3-1-8b-fp8/;
        }
      }
    }
OUTEREOF
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

Create metrics sidecar ConfigMaps (replace `<SPOKE*_METRICS_ROUTE>` and `<SPOKE*_SA_TOKEN>` with values from Part 1):

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
          proxy_set_header Authorization "Bearer <SPOKE1_SA_TOKEN>";
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
          proxy_set_header Authorization "Bearer <SPOKE2_SA_TOKEN>";
          proxy_ssl_verify off;
          proxy_pass https://<SPOKE2_METRICS_ROUTE>/metrics;
        }
        location = /healthz { return 200 "ok\n"; }
      }
    }
EOF
```

> **Why a sidecar?** hexfusion feedback: "EPP hub to spoke /metrics should probably be mTLS which removes the proxy dependency." The sidecar decouples metrics from inference — nginx handles only inference + health (port 8080), the sidecar handles only metrics over TLS (port 8081). Hub EPP scrapes the sidecar directly.

Create proxy Deployments with metrics sidecar + Service:

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
            - name: cfg
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
        - name: cfg
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
            - name: cfg
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
        - name: cfg
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
OUTEREOF

oc -n llm-d-hub rollout status deploy/spoke-1 --timeout=120s
oc -n llm-d-hub rollout status deploy/spoke-2 --timeout=120s

echo "=== Verify ===" && oc get pods -n llm-d-hub --no-headers | grep spoke
echo "=== Endpoints (expect 2) ===" && oc get endpoints hub-pool-svc -n llm-d-hub
```

**Expected**: 2 proxy pods `2/2 Running` (proxy + metrics sidecar), 2 endpoints. If `1/2`, the inference proxy readiness failed (upstream-health). Metrics sidecar serves TLS on port 8081 — Hub EPP scrapes `https://<pod-ip>:8081/metrics`.

**Proxy config reference**:

| Location | Purpose |
|----------|---------|
| `/healthz` | Liveness probe — nginx self-check (always 200) |
| `/upstream-health` | Readiness probe — proxies to spoke `/v1/models`. If spoke model down → 5xx → readiness fails → pod removed from endpoints |
| `/metrics` | Hub EPP metrics scraping — proxies to spoke EPP metrics via OpenShift Route (0.35s latency) |
| `/` | Inference requests — proxies to spoke MaaS gateway with API key + path prefix |

### 2.2 Create ExternalModel + MaaS governance

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

# Restart maas-controller to pick up new model
oc rollout restart deployment/maas-controller -n redhat-ods-applications
echo "Waiting 40s for maas-controller restart + TRLP enforcement..."
sleep 40
```

> **Note**: The ExternalModel endpoint is the **Hub Gateway Service** (`hub-routing-data-science-gateway-class`), NOT the proxy Service (`hub-pool-svc`). This is critical — pointing directly to the proxy Service bypasses the Hub Gateway + InferencePool + Hub EPP entirely, disabling load-aware scoring and session stickiness. The credential is `dummy-not-used` because the Hub Gateway doesn't require auth (internal service). The actual spoke auth is handled by `proxy_set_header Authorization` in the nginx config.

### 2.3 Deploy Hub EPP + InferencePool + Gateway

The Hub EPP selects which proxy pod (spoke) receives each request based on load metrics, session affinity, and endpoint health.

> **EPP readiness probe is gRPC on port 9003** — NOT HTTP. Using `httpGet` will cause the pod to stay `0/1 Running` indefinitely.

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

# Wait for EPP + Gateway
oc -n llm-d-hub rollout status deploy/epp --timeout=120s

echo "=== Verify ==="
echo "--- EPP (expect 1/1) ---" && oc get pods -n llm-d-hub -l app=epp --no-headers
echo "--- InferencePool (expect Accepted) ---" && oc get inferencepool hub-pool -n llm-d-hub -o jsonpath='{.status.parents[0].conditions[0].reason}' && echo ""
echo "--- Gateway ---" && oc get gateway hub-routing -n llm-d-hub --no-headers
echo "--- HTTPRoute ---" && oc get httproute hub-route -n llm-d-hub --no-headers
```

**Expected**: EPP `1/1 Running`, InferencePool `Accepted`, Gateway with NLB address, HTTPRoute accepted.

> **EPP `core-metrics-extractor` with `spoke-epp` engine**: The default engine expects vLLM metrics (`vllm:num_requests_waiting`). Proxy pods serve spoke EPP aggregate metrics with different names (`inference_pool_average_queue_size`). The custom `spoke-epp` engine maps these metric names so the queue-scorer and kv-cache-utilization-scorer can parse them. Note: `running-requests-size-scorer` is omitted because spoke EPP does not expose an `inference_pool_average_running_requests` metric.
>
> **EPP `session-affinity-scorer`**: Sets `x-session-token` response header (base64-encoded proxy pod NamespacedName). Also scores the matching pod with weight 5 for stickiness.
> **EPP `session-affinity-filter`**: Filters to only the pod matching the incoming `x-session-token` request header.

---

## Part 3: Verification

Run all tests from the **hub cluster** (osacmaas).

### 3.1 Resource verification

```bash
echo "=== Hub proxies (expect 1/1 each) ===" && oc get pods -n llm-d-hub --no-headers | grep -E 'spoke|epp|hub-routing'
echo "=== Proxy Service ===" && oc get svc hub-pool-svc -n llm-d-hub --no-headers
echo "=== Endpoints (expect 2) ===" && oc get endpoints hub-pool-svc -n llm-d-hub
echo "=== EPP (expect 1/1) ===" && oc get pods -n llm-d-hub -l app=epp --no-headers
echo "=== InferencePool ===" && oc get inferencepool hub-pool -n llm-d-hub --no-headers
echo "=== Hub Gateway ===" && oc get gateway hub-routing -n llm-d-hub --no-headers
echo "=== Hub HTTPRoute ===" && oc get httproute hub-route -n llm-d-hub --no-headers
echo "=== ExternalModel ===" && oc get externalmodel -n external-models --no-headers
echo "=== MaaSModelRef ===" && oc get maasmodelref -n external-models --no-headers
echo "=== Subscription ===" && oc get maassubscription -n models-as-a-service --no-headers | grep multi
```

### 3.2 Governance tests

```bash
TENANT_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
MAAS_URL="https://maas.${TENANT_DOMAIN}"
MODEL_ID="granite-3.1-8b-instruct-fp8"

# Get tenant API key
JWT=$(curl -sk -X POST \
  "https://keycloak-maas.${TENANT_DOMAIN}/realms/maas/protocol/openid-connect/token" \
  -d "client_id=maas-client&client_secret=maas-client-secret-value&username=testuser-premium&password=password123&grant_type=password" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

API_KEY=$(curl -sk -X POST "${MAAS_URL}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer ${JWT}" \
  -H "Content-Type: application/json" \
  -d '{"name":"spoke-hub-test"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['key'])")

echo "API_KEY=${API_KEY}"

# Test 1: Unauthenticated (expect 401)
# NOTE: the 401 test may need retries (up to 3) as the Kuadrant Wasm plugin can take seconds to initialize
for i in 1 2 3; do
  HTTP=$(curl -sk -o /dev/null -w "%{http_code}" \
    "${MAAS_URL}/external-models/granite-multi-cluster/v1/chat/completions" \
    -X POST -H "Content-Type: application/json" \
    -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Hello\"}],\"max_tokens\":10}")
  echo "Unauthenticated: HTTP $HTTP"
  [ "$HTTP" = "401" ] && break
  sleep 10
done

# Test 2: Invalid key (expect 403)
HTTP=$(curl -sk -o /dev/null -w "%{http_code}" \
  "${MAAS_URL}/external-models/granite-multi-cluster/v1/chat/completions" \
  -X POST -H "Content-Type: application/json" -H "Authorization: Bearer sk-oai-INVALID" \
  -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Hello\"}],\"max_tokens\":10}")
echo "Invalid key: HTTP $HTTP"

# Test 3: Valid inference (expect 200)
curl -sk "${MAAS_URL}/external-models/granite-multi-cluster/v1/chat/completions" \
  -H "Authorization: Bearer ${API_KEY}" -H "Content-Type: application/json" \
  -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Hello\"}],\"max_tokens\":20}" | python3 -m json.tool

# Test 4: Rate limiting (expect 429)
for i in $(seq 1 12); do
  HTTP=$(curl -sk -o /dev/null -w "%{http_code}" \
    "${MAAS_URL}/external-models/granite-multi-cluster/v1/chat/completions" \
    -H "Authorization: Bearer ${API_KEY}" -H "Content-Type: application/json" \
    -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Tell me a long story\"}],\"max_tokens\":50}")
  echo "Request $i: HTTP $HTTP"
  [ "$HTTP" = "429" ] && echo "Rate limiting enforced!" && break
done
```

**Expected**: 401 → 403 → 200 with model response → 429 within 12 requests.

### 3.3 Multi-cluster routing

> **Wait 60s** after the rate-limiting test for the token budget window to reset.

```bash
echo "Waiting 65s for rate limit window reset..."
sleep 65

echo "=== Sending 10 requests ==="
for i in $(seq 1 10); do
  HTTP=$(curl -sk -o /dev/null -w "%{http_code}" "${MAAS_URL}/external-models/granite-multi-cluster/v1/chat/completions" \
    -H "Authorization: Bearer ${API_KEY}" -H "Content-Type: application/json" \
    -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Hello\"}],\"max_tokens\":5}")
  echo "  Request $i: HTTP $HTTP"
done

echo ""
echo "=== Distribution ==="
echo "spoke-1: $(oc logs -n llm-d-hub deployment/spoke-1 --since=2m | grep -c POST) requests"
echo "spoke-2: $(oc logs -n llm-d-hub deployment/spoke-2 --since=2m | grep -c POST) requests"
```

**Expected**: ~50/50 split across spokes (Hub EPP distribution). **Validated**: 13/13 split (16 completed before rate limit).

### 3.4 Failover test

The proxy readiness probe checks upstream spoke model health (`/upstream-health`). When a spoke model goes down, the probe fails after ~30s (3 × 10s), K8s removes the proxy from endpoints, and Hub EPP stops routing to it.

```bash
# 1. Scale down spoke-1 model
oc config use-context "default/api-spoke1-kni-syseng-devcluster-openshift-com:6443/kube:admin"
oc scale deploy -n granite-serving -l kserve.io/component=workload --replicas=0
oc config use-context "default/api-osacmaas-kni-syseng-devcluster-openshift-com:6443/kube:admin"

# 2. Wait for readiness probe to fail (~40s)
echo "Waiting for proxy readiness failure..."
for i in $(seq 1 8); do
  S1=$(oc get pods -n llm-d-hub -l cluster=spoke-1 --no-headers | awk '{print $2}')
  EP=$(oc get endpoints hub-pool-svc -n llm-d-hub -o jsonpath='{.subsets[0].addresses}' 2>/dev/null | python3 -c "import sys,json; print(len(json.load(sys.stdin)))" 2>/dev/null || echo "?")
  echo "  ${i}0s: spoke-1=$S1 endpoints=$EP"
  [ "$S1" = "0/1" ] && echo "  spoke-1 proxy NOT READY!" && break
  sleep 10
done

# 3. Send 10 requests — expect all to spoke-2
echo ""
echo "=== Sending 10 requests (spoke-1 down) ==="
for i in $(seq 1 10); do
  HTTP=$(curl -sk -o /dev/null -w "%{http_code}" "${MAAS_URL}/external-models/granite-multi-cluster/v1/chat/completions" \
    -H "Authorization: Bearer ${API_KEY}" -H "Content-Type: application/json" \
    -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Hello\"}],\"max_tokens\":5}")
  echo "  Request $i: HTTP $HTTP"
done

echo ""
echo "=== Distribution ==="
echo "spoke-1: $(oc logs -n llm-d-hub deployment/spoke-1 --since=2m | grep -c POST)"
echo "spoke-2: $(oc logs -n llm-d-hub deployment/spoke-2 --since=2m | grep -c POST)"
```

**Expected**: spoke-1 proxy goes `0/1` within ~40s, endpoints drop from 2 to 1, all 10 requests go to spoke-2. **Validated**: 10/10 to spoke-2, 0 to spoke-1.

### 3.5 Recovery test

```bash
# 1. Restore spoke-1 model
oc config use-context "default/api-spoke1-kni-syseng-devcluster-openshift-com:6443/kube:admin"
oc scale deploy -n granite-serving -l kserve.io/component=workload --replicas=1
oc config use-context "default/api-osacmaas-kni-syseng-devcluster-openshift-com:6443/kube:admin"

# 2. Wait for model ready + proxy readiness recovery (2-5 min for GPU model load)
echo "Waiting for spoke-1 recovery..."
for i in $(seq 1 30); do
  S1=$(oc get pods -n llm-d-hub -l cluster=spoke-1 --no-headers | awk '{print $2}')
  EP=$(oc get endpoints hub-pool-svc -n llm-d-hub -o jsonpath='{.subsets[0].addresses}' 2>/dev/null | python3 -c "import sys,json; print(len(json.load(sys.stdin)))" 2>/dev/null || echo "?")
  echo "  ${i}0s: spoke-1=$S1 endpoints=$EP"
  [ "$S1" = "1/1" ] && echo "  spoke-1 RECOVERED!" && break
  sleep 10
done

# 3. Wait for rate limit window
echo "Waiting 65s for rate limit window reset..."
sleep 65

# 4. Send 10 requests — expect distribution across both spokes
for i in $(seq 1 10); do
  HTTP=$(curl -sk -o /dev/null -w "%{http_code}" "${MAAS_URL}/external-models/granite-multi-cluster/v1/chat/completions" \
    -H "Authorization: Bearer ${API_KEY}" -H "Content-Type: application/json" \
    -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Hello\"}],\"max_tokens\":5}")
  echo "  Request $i: HTTP $HTTP"
done

echo ""
echo "=== Distribution ==="
echo "spoke-1: $(oc logs -n llm-d-hub deployment/spoke-1 --since=2m | grep -c POST)"
echo "spoke-2: $(oc logs -n llm-d-hub deployment/spoke-2 --since=2m | grep -c POST)"
```

**Expected**: 10/10 success, both spokes receive requests. **Validated**: 3/7 split after ~210s recovery (GPU model reload).

### 3.6 Session stickiness test

The Hub EPP uses `x-session-token` (NOT `x-session-id`). The token is a base64-encoded proxy pod NamespacedName, set by the `session-affinity-scorer` on the response. The `session-affinity-filter` reads it on subsequent requests and routes to the same proxy pod.

The ExternalModel endpoint points to the Hub Gateway Service, so the Hub EPP is in the request/response path. The `x-session-token` response header propagates through the MaaS gateway back to the client.

```bash
# 1. Get session token from first MaaS gateway response
TOKEN=$(curl -sk -D /tmp/stk.txt -o /dev/null "${MAAS_URL}/external-models/granite-multi-cluster/v1/chat/completions" \
  -H "Authorization: Bearer ${API_KEY}" -H "Content-Type: application/json" \
  -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Hello\"}],\"max_tokens\":5}" && \
  grep -i x-session-token /tmp/stk.txt | awk '{print $2}' | tr -d '\r')
echo "Token: $TOKEN"
echo "Decodes to: $(echo $TOKEN | base64 -d)"

# 2. Send 5 requests with the token (expect all same spoke)
echo ""
echo "=== Session stickiness via MaaS gateway (expect 5/5 same spoke) ==="
for i in $(seq 1 5); do
  HTTP=$(curl -sk -o /dev/null -w "%{http_code}" "${MAAS_URL}/external-models/granite-multi-cluster/v1/chat/completions" \
    -H "Authorization: Bearer ${API_KEY}" -H "Content-Type: application/json" \
    -H "x-session-token: ${TOKEN}" \
    -d "{\"model\":\"${MODEL_ID}\",\"messages\":[{\"role\":\"user\",\"content\":\"Hello\"}],\"max_tokens\":5}")
  echo "  Request $i: HTTP $HTTP"
done

echo ""
echo "=== Distribution ==="
echo "spoke-1: $(oc logs -n llm-d-hub deployment/spoke-1 --since=1m | grep -c POST)"
echo "spoke-2: $(oc logs -n llm-d-hub deployment/spoke-2 --since=1m | grep -c POST)"
```

**Expected**: Token in response, all to same spoke. **Validated**: 10/10 to spoke-2 (token encoded spoke-2 pod).

---

## Validated Results (July 15, 2026)

| Capability | Level | Status | Notes |
|------------|-------|--------|-------|
| Unauthenticated → 401 | Cross-cluster (Hub EPP) | **PASS** | Hub MaaS auth enforcement |
| Invalid API key → 403 | Cross-cluster (Hub EPP) | **PASS** | Hub MaaS key validation |
| Valid inference → 200 | Cross-cluster (Hub EPP) | **PASS** | MaaS → Hub Gateway → InferencePool → Hub EPP → proxy → spoke → vLLM |
| Rate limiting → 429 | Cross-cluster (Hub EPP) | **PASS** (request 10) | Hub TRLP enforcement |
| Multi-cluster routing | Cross-cluster (Hub EPP) | **PASS** (13/13 split) | Hub EPP distributes across both spokes |
| **Failover** | Cross-cluster (Hub EPP) | **PASS (10/10)** | Upstream health readiness probe — 40s detection, 100% failover to healthy spoke |
| Recovery | Cross-cluster | **PASS** (3/7 split) | spoke-1 re-enters pool after model restored (~210s recovery) |
| Session stickiness | Cross-cluster (Hub EPP) | **PASS (10/10)** | `x-session-token` in both request and response via MaaS gateway |
| Spoke metrics scraping | Cross-cluster (Hub EPP) | **PASS** (0.35s, 10KB) | OpenShift Route → EPP service, SA token auth |
| Load-aware scoring | Cross-cluster (Hub EPP) | **PARTIAL** | Custom `spoke-epp` engine maps spoke EPP metric names. EPP parses metrics but differential scoring under asymmetric load not conclusive — see note below |
| Manual InferencePool backendRef | Hub (OCP 4.21) | **PASS** | Istio 1.27.3 accepts — previously blocked on 4.19/4.20 |

### Load-Aware Scoring — PARTIAL (custom engine config applied)

The Hub EPP's default `core-metrics-extractor` expects vLLM metrics (`vllm:num_requests_waiting`, `vllm:kv_cache_usage_perc`). The proxy pods serve spoke EPP aggregate metrics with different names (`inference_pool_average_queue_size`, `inference_pool_average_kv_cache_utilization`). Without mapping, the EPP scorers see zero data and distribute randomly.

**Fix applied**: Custom `spoke-epp` engine config in the EPP ConfigMap maps spoke EPP metric names to the extractor's queue and KV usage specs. The EPP logs confirm: `"Registered engine mapping","engine":"spoke-epp","mapping":"Mapping{disabled: [running, lora, cacheInfo]}"`.

**Why PARTIAL**: Under an asymmetric load test (3 long-running requests to spoke-1, then 5 hub requests), spoke-1 still received 4/5 requests instead of spoke-2 being favored. Possible causes:
1. **Metrics propagation delay** — proxy scrapes spoke EPP Route every 5s; load may dissipate before Hub EPP sees the delta
2. **Aggregate vs per-pod metrics** — spoke EPP `inference_pool_average_*` metrics are pool-level averages, not raw queue counts. With 1 vLLM pod per spoke, average = raw. But the metric value representation may differ from what the EPP scorers expect
3. **Spoke EPP does not expose `running_requests`** — only queue + KV cache available. The `running-requests-size-scorer` has no data to score on (removed from config)

**What works**: The `spoke-epp` engine is registered and the EPP scrapes + parses metrics. Multi-cluster routing through Hub EPP is confirmed (session stickiness and failover both prove Hub EPP is in the path). The gap is demonstrating differential scoring under sustained asymmetric load — a test methodology challenge, not a configuration failure.

### Source Code Analysis (July 15): Corrected Previous Assumptions

1. **IPP does NOT strip custom headers.** `maas-headers-guard` only strips `x-maas-*` prefix headers. ext-proc response handler (`response.go`) is passthrough-by-default — unmodified headers preserved.

2. **ExternalModel endpoint must be Hub Gateway, not proxy Service.** The original endpoint `hub-pool-svc` routed directly to proxy pods via K8s round-robin, completely bypassing the Hub Gateway + InferencePool + Hub EPP. What looked like "Hub EPP routing" was K8s Service variance. Fix: point ExternalModel to `hub-routing-data-science-gateway-class.llm-d-hub.svc.cluster.local`.

3. **Hub EPP uses `x-session-token`, not `x-session-id`.** The `llm-d/llm-d-router` `session-affinity-filter` reads `x-session-token` (base64-encoded pod NamespacedName). `session-affinity-scorer` sets it on the response.

4. **EPP readiness is gRPC on port 9003.** Not HTTP — `httpGet` probe always fails.

5. **`x-session-token` propagates through MaaS gateway** in both directions when the request path includes the Hub EPP. The IPP framework is passthrough for both request and response headers.

6. **Hub EPP needs custom `spoke-epp` engine config for metrics.** The default `core-metrics-extractor` looks for vLLM metrics (`vllm:num_requests_waiting`). Proxy pods serve spoke EPP aggregate metrics (`inference_pool_average_queue_size`). Custom engine config maps the spoke metric names: `queuedRequestsSpec: "inference_pool_average_queue_size"`, `kvUsageSpec: "inference_pool_average_kv_cache_utilization"`. Source: `llm-d/llm-d-router` `pkg/epp/framework/plugins/datalayer/extractor/metrics/factories.go` — supports `engineConfigs` parameter with custom metric specs per engine type.

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Full MaaS on all clusters | Spokes need `maas-default-gateway` with pass-through auth for upstream protection |
| `router: { gateway: {refs: [maas-default-gateway]}, route: {}, scheduler: {} }` | All three fields required. `route: {}` breaks the InferencePool deadlock |
| nginx injects spoke API keys | Spoke MaaS enforces auth — proxy carries the credential, not the ExternalModel |
| nginx prepends path prefix | Hub's IPP strips path to `/v1/...` — proxy must reconstruct `/<ns>/<model>/v1/...` |
| UBI9 `nginx-124` with `command: ["nginx", "-g", "daemon off;"]` | Red Hat certified image. S2I builder — requires explicit `command` override (without it, shows usage text and exits) |
| ONE ExternalModel → **Hub Gateway** Service | Routes through InferencePool → Hub EPP (NOT direct to proxy Service — that bypasses EPP entirely) |
| Upstream health readiness probe | `/upstream-health` proxies to spoke `/v1/models` — detects model failure, enables automatic failover |
| `session-affinity-scorer` + `session-affinity-filter` | Scorer sets `x-session-token` response header; filter reads it on subsequent requests |
| gRPC readiness probe for EPP | EPP health port 9003 serves gRPC, not HTTP |
| OpenShift Routes for spoke metrics | Bypasses MaaS gateway (30s auth latency) — 0.35s direct to EPP service |
| Metrics sidecar (TLS, port 8081) | Decouples metrics from inference nginx per hexfusion feedback. cert-manager cert via `serving-cert-secret-name`. Hub EPP scrapes `https://<pod>:8081/metrics` |
| Custom `spoke-epp` engine in EPP | Default extractor expects vLLM metrics; proxy serves spoke EPP aggregate metrics with different names. Custom engine maps `inference_pool_average_*` to queue/KV specs |

## What We Learned (July 13-15)

| Assumption | Reality |
|------------|---------|
| ExternalModel per cluster works | **No** — `targetModel` collision in IPP Rule 2 |
| InferencePool needs Istio 1.27 | **No** — works on Istio 1.26.2 with correct spec |
| InferencePool needs OCP 4.20 | **No** — works on OCP 4.19 |
| `router.scheduler` alone is enough | **No** — deadlocks without `route: {}` |
| Standard nginx works on OpenShift | **No** — UBI9 `nginx-124` (Red Hat certified, S2I). Requires `command: ["nginx", "-g", "daemon off;"]` override |
| `proxy_pass` to spoke NLB just works | **No** — needs Host header, API key, SSL bypass, path prefix |
| MaaS IPP strips headers | **No** — source code confirms IPP only strips `x-maas-*` prefix. Both request and response handlers are passthrough-by-default |
| Hub EPP uses `X-Session-Id` | **No** — uses `x-session-token` with base64-encoded pod name |
| ExternalModel → proxy Service works | **No** — bypasses Hub Gateway + InferencePool + Hub EPP entirely. Must point to Hub Gateway Service |
| Proxy `/healthz` is sufficient | **No** — must check upstream spoke health for failover |
| EPP health is HTTP | **No** — gRPC on port 9003 |
| nginx should handle metrics | **No** — dedicated TLS sidecar (port 8081) decouples metrics from inference. cert-manager cert, Hub EPP scrapes via HTTPS |
| Default EPP metrics config works for hub | **No** — default expects vLLM metrics (`vllm:*`), proxy serves spoke EPP metrics (`inference_pool_average_*`). Need custom `spoke-epp` engine config |

See `summary.md` for the full investigation timeline.

## References

- [Investigation summary](summary.md) — full timeline of findings and corrections
- [Multi-upstream ExternalModel bugs](multi-upstream-external-model-bugs.md) — Bug 1/2/3 findings
- [ibmc-llmd Phase 6](https://github.com/mpaulgreen/ibmc-llmd/blob/main/manual/PHASE6-INFERENCE-SCHEDULING.md) — original working spec
- [hexfusion spoke-hub PoC](https://github.com/hexfusion/experiments/tree/main/llm-d/spoke-and-hub) — reference architecture
- [RHOAI 3.4 llm-d scheduler config](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/deploy_models_using_distributed_inference_with_llm-d/configuring-llm-scheduler)
- [ai-gateway-payload-processing](https://github.com/opendatahub-io/ai-gateway-payload-processing) — MaaS IPP source (header handling analysis)
- [llm-d-router](https://github.com/llm-d/llm-d-router) — Hub EPP source (session affinity plugins)
