# MaaS Observability Dashboard Setup — RHOAI 3.4

Set up the Models-as-a-Service observability dashboard to monitor subscription-level usage metrics — token consumption, request counts, rate-limit violations, GPU utilization, and cost attribution.

**Technology Preview**: The MaaS observability dashboard is a Technology Preview feature in RHOAI 3.4. It is designed for showback reporting, not billing-grade metering. For production chargeback workflows, access the Limitador metrics endpoint directly via Prometheus.

**Source**: [RHOAI 3.4 — Monitor Models-as-a-Service usage with observability](../maas/Red_Hat_OpenShift_AI_Self-Managed-3.4-Govern_LLM_access_with_Models-as-a-Service-en-US.pdf) (Section 1.12)

**Slack thread**: [RHOAI Observability Investigation](https://redhat-internal.slack.com/archives/C05SMJ09DD2/p1781889176436489)

## Architecture

```
Limitador (rate limiting) → PodMonitor → Prometheus → Thanos Querier
                                                            ↓
Authorino (auth)          → ServiceMonitor → Prometheus → Thanos Querier
                                                            ↓
Gateway (Istio)           → TelemetryPolicy → Prometheus → Thanos Querier
                                                            ↓
                                              RHOAI Dashboard (Perses via COO)
```

The observability dashboard is embedded in the OpenShift AI console using **Perses** (provided by the Cluster Observability Operator) and queries metrics from **Thanos Querier**. The RHOAI Monitoring Service Controller creates the Perses instance, datasources, and dashboards in `redhat-ods-monitoring`.

## Before You Begin

This guide assumes you have completed in order:

1. **BOM deployment** from [`bom_rhoai_3.4.md`](../bom_rhoai_3.4.md) — NFD, GPU, cert-manager, SM 3.2, RHCL 1.3, LWS, RHOAI 3.4, DSC
2. **MaaS deployment Phases 1-3** from [MaaS deployment guide](../maas/README.md) — Keycloak, prerequisites, MaaS enabled
3. **Dashboard config** from [model-car-and-maas.md](model-car-and-maas.md) Step 2 — `observabilityDashboard: true` included

> **Version pinning**: RHCL must remain at 1.3.x, sub-operators at 1.3.1, SM at 3.2.x. The monitoring operators (COO, OTel) must be installed in **isolated namespaces** — NOT in `openshift-operators` — to prevent OLM from bundling their install plans with pending v1.4.0 RHCL upgrade plans.

---

## Step 1: Install Red Hat OpenTelemetry Operator

Required by the RHOAI Monitoring controller — it fails without the OpenTelemetryCollector CRD.

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-opentelemetry-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: opentelemetry-operator
  namespace: openshift-opentelemetry-operator
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: opentelemetry-product
  namespace: openshift-opentelemetry-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: opentelemetry-product
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Verify:

```bash
oc get csv -n openshift-opentelemetry-operator | grep telemetry
# Expected: opentelemetry-operator.vX.X.X   Succeeded
```

## Step 2: Install Cluster Observability Operator (COO) v1.4

> **Critical: Use COO v1.4, NOT v1.5.** COO 1.5 introduced breaking changes:
> - CEL validation requiring `namespace` on `caCert` fields ([perses-operator#290](https://github.com/perses/perses-operator/commit/2fa3828630187c4e5866426741b8bd6ae20a79ae)) — RHOAI 3.4 templates don't include this field
> - Perses image incompatibility — RHOAI passes `--web.tls-min-version` which COO 1.5's Perses binary doesn't support
> - Fixes pending: [opendatahub-operator PR #3659](https://github.com/opendatahub-io/opendatahub-operator/pull/3659), [PR #3590](https://github.com/opendatahub-io/opendatahub-operator/pull/3590)

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cluster-observability-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cluster-observability-operator
  namespace: openshift-cluster-observability-operator
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-observability-operator
  namespace: openshift-cluster-observability-operator
spec:
  channel: stable
  installPlanApproval: Manual
  name: cluster-observability-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: cluster-observability-operator.v1.4.0
EOF
```

Approve the install plan:

```bash
# Wait for install plan
sleep 30
oc get installplan -n openshift-cluster-observability-operator --no-headers

# Approve (replace <plan-name>)
oc patch installplan <plan-name> -n openshift-cluster-observability-operator \
  --type merge -p '{"spec":{"approved":true}}'
```

Verify:

```bash
oc get csv -n openshift-cluster-observability-operator | grep observability
# Expected: cluster-observability-operator.v1.4.0   Succeeded

oc get pods -n openshift-cluster-observability-operator
# Expected: observability-operator, perses-operator, obo-prometheus-operator — all Running
```

## Step 3: Configure DSCI Metrics

The RHOAI Monitoring controller only creates Perses/datasource resources when `spec.monitoring.metrics` is configured in the DSCInitialization CR. Without this, it skips silently.

```bash
oc patch dsci default-dsci --type merge -p '{
  "spec": {
    "monitoring": {
      "metrics": {
        "replicas": 1,
        "storage": {
          "retention": "7d",
          "size": "10Gi"
        }
      }
    }
  }
}'
```

## Step 4: Restart RHOAI Operator

The operator needs to re-reconcile to detect the new COO CRDs and DSCI metrics config.

```bash
oc rollout restart deployment/rhods-operator -n redhat-ods-operator
```

Wait 90 seconds for reconciliation.

> **Note: UIPlugin is NOT required.** The RHOAI dashboard has a built-in Perses client that renders the Cluster/Model/Usage tabs directly. The `UIPlugin` CR (type `Dashboards`) is only needed if you want Perses dashboards in the OpenShift native console's `Observe → Dashboards` menu — a separate entry point. The RHOAI dashboard's observability feature is enabled by COO + OTel + DSCI metrics config + `observabilityDashboard: true` in OdhDashboardConfig. Validated on RHOAI 3.4 without UIPlugin — all three dashboard tabs work correctly.

## Step 5: Verify

```bash
# Monitoring CR
oc get monitoring default-monitoring -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
# Expected: True

# Perses server running
oc get perses -n redhat-ods-monitoring
# Expected: data-science-perses

oc get pods -n redhat-ods-monitoring | grep perses
# Expected: data-science-perses-0   1/1   Running

# Datasources created
oc get persesdatasource -n redhat-ods-monitoring
# Expected: cluster-prometheus-datasource, data-science-prometheus-datasource

# Dashboards created
oc get persesdashboard -n redhat-ods-monitoring
# Expected: dashboard-0-cluster-admin, dashboard-1-model

# Version pinning held
oc get csv -n openshift-operators | grep -E 'rhcl|authorino|limitador|dns|servicemesh'
# Expected: all v1.3.x, SM v3.2.x — zero v1.4.0
```

Then navigate to **Observe & monitor → Dashboard** in the RHOAI dashboard.

## Step 6: Generate Traffic for Dashboard Metrics

The Models tab shows "No data" until inference requests flow through the MaaS gateway.

> **Prerequisite**: A model deployed with MaaS governance (auth policy + subscriptions) via the dashboard UI. See [model-car-and-maas.md](model-car-and-maas.md) Steps 3-6.

### Generate Traffic

```bash
export DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
export HOST="https://maas.${DOMAIN}"
export KEYCLOAK_URL="https://keycloak-maas.${DOMAIN}"

# Get Keycloak JWT and create API key
ACCESS_TOKEN=$(curl -sk -X POST "${KEYCLOAK_URL}/realms/maas/protocol/openid-connect/token" -d "client_id=maas-client" -d "client_secret=maas-client-secret-value" -d "username=testuser-premium" -d "password=password123" -d "grant_type=password" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

API_KEY=$(curl -sk -X POST "${HOST}/maas-api/v1/api-keys" -H "Authorization: Bearer ${ACCESS_TOKEN}" -H "Content-Type: application/json" -d '{"name":"observability-test","description":"generate dashboard metrics"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['key'])")

# Generate 20 inference requests
for i in $(seq 1 20); do
  HTTP_CODE=$(curl -sk -o /dev/null -w "%{http_code}" \
    "${HOST}/granite-serving/granite-3-1-8b-fp8/v1/chat/completions" \
    -H "Authorization: Bearer ${API_KEY}" \
    -H "Content-Type: application/json" \
    -d '{"model":"granite-3-1-8b-fp8","messages":[{"role":"user","content":"What is OpenShift?"}],"max_tokens":30}')
  echo "  Request $i: HTTP ${HTTP_CODE}"
done
# Expected: all HTTP 200
```

Wait 1-2 minutes for Prometheus to scrape the new metrics, then refresh the dashboard:

- **Cluster tab**: System health, GPU utilization, Memory allocated
- **Models tab**: Model deployments, P90 latency, token throughput, request success rate
- **Usage tab**: Shows zeros for external OIDC users (see [Known Issues](#usage-tab-shows-zeros--external-oidc-users-not-tracked))

---

## What the RHOAI Operator Creates (for reference)

When all prerequisites are in place, the RHOAI Monitoring Service Controller creates in `redhat-ods-monitoring`:

| Resource | Name | Purpose |
|----------|------|---------|
| Perses CR | `data-science-perses` | COO creates the Perses server StatefulSet |
| PersesDatasource | `data-science-prometheus-datasource` | Connects to namespace-scoped Prometheus |
| PersesDatasource | `cluster-prometheus-datasource` | Connects to cluster Thanos Querier |
| Secret | `cluster-prometheus-datasource-secret` | SA token for Thanos Querier auth |
| PersesDashboard | `dashboard-0-cluster-admin` | Cluster overview dashboard |
| PersesDashboard | `dashboard-1-model` | Model performance dashboard |
| Prometheus | `data-science-monitoringstack` | Dedicated metrics collection |
| Thanos Querier | `data-science-thanos-querier` | Federated metric queries |
| OTel Collector | `data-science-collector` | Telemetry pipeline |

The MaaS component additionally creates in `redhat-ods-applications`:

| Resource | Name | Purpose |
|----------|------|---------|
| PersesDatasource | `kuadrant-prometheus-datasource` | Connects to Kuadrant/Limitador metrics |
| PersesDashboard | `dashboard-3-maas-usage-admin` | MaaS token consumption dashboard |

---

## Dashboard Metrics Reference

### Overview Metrics (Cluster tab)

| Metric | Description |
|--------|-------------|
| System Health | Overall cluster health percentage |
| Deployed Models | Number of active model deployments |
| GPU Utilization | Aggregate GPU usage across all nodes |
| Request Success Rate | Percentage of successful inference requests |

### MaaS Usage Metrics (after model deployment + traffic)

| Metric | Source | Description |
|--------|--------|-------------|
| `authorized_hits` | Limitador | Total tokens consumed by subscription, model |
| `authorized_calls` | Limitador | Successful API requests |
| `limited_calls` | Limitador | Rate-limited requests (HTTP 429) |
| `istio_request_duration_milliseconds_bucket` | Gateway | Request latency by subscription |
| `auth_server_authconfig_duration_seconds` | Authorino | Auth processing time |

---

## Known Issues

### COO 1.5 Incompatibility with RHOAI 3.4

COO v1.5.0 breaks the observability dashboard due to:
1. **CEL validation** — `namespace` required on `caCert` when `type: configmap`, but RHOAI template omits it
2. **Perses image** — `--web.tls-min-version` flag not supported

**Workaround**: Pin to COO v1.4.0 with `startingCSV` and `Manual` approval.

**Fixes**: [PR #3659](https://github.com/opendatahub-io/opendatahub-operator/pull/3659), [PR #3590](https://github.com/opendatahub-io/opendatahub-operator/pull/3590)

### Usage Tab Shows Zeros — Missing Tenant Telemetry Capture Flags

The Usage tab shows zeros when the Tenant CR's telemetry section does not include the `metrics.captureUser` flag. Without it, Limitador does not emit a `user` label in its metrics, and the dashboard's PromQL queries (which filter on `{user!=""}`) return zero results.

**Fix**: Add telemetry capture flags to the Tenant CR:

```bash
oc patch tenant default-tenant -n models-as-a-service --type merge -p '{
  "spec": {
    "telemetry": {
      "metrics": {
        "captureUser": true,
        "captureGroup": true,
        "captureOrganization": true,
        "captureModelUsage": true
      }
    }
  }
}'
```

After applying, send inference requests and wait 1-2 minutes for Prometheus to scrape. The Usage tab will show per-user token consumption, request counts, and rate limiting data.

**Validated**: With `captureUser: true`, Limitador emits `user="testuser-premium"` for external OIDC API key users. All Usage tab panels (Total Tokens, Total Requests, Active Users, Token Consumption table) populate correctly with the original RHOAI dashboard — no dashboard patching or workarounds needed.

> **Note**: The Ansible role (`ocp_4_20_ai_maas`) includes these flags in the Tenant CR by default. If deploying manually, ensure they are set.

### OLM Bundling with RHCL v1.4.0

If COO/OTel are installed in `openshift-operators` (same namespace as RHCL), OLM bundles their install plans with pending v1.4.0 RHCL upgrades, causing `ConstraintsNotSatisfiable`. 

**Workaround**: Install COO in `openshift-cluster-observability-operator` and OTel in `openshift-opentelemetry-operator` to isolate from RHCL dependency chain.

### Monitoring Controller Skips Silently

If `spec.monitoring.metrics` is not configured in DSCI (empty `{}`), the Monitoring controller runs `deployPerses` and `deployPersesPrometheusIntegration` without error but creates nothing. The `MonitoringStackAvailable` condition shows `MetricsNotConfigured`.

**Fix**: Configure `metrics.replicas` and `metrics.storage` in DSCI before creating the DSC.
