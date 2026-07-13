# [WIP] MaaS Metering Analysis

*Technical analysis of token metering, tenant attribution, and infrastructure metrics for the OSAC MaaS platform. Covers metrics inventory, data flow, upstream PR status, and phased implementation.*

---

## 1. Metrics Inventory

Complete inventory of metrics available in the MaaS stack, classified by source and billing relevance. Verified against [PR #332](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/332) source code, [vLLM docs](https://docs.vllm.ai/en/latest/design/metrics/), RHOAI 3.4.2 live cluster CRDs, and [Red Hat Developer article on Limitador metrics](https://developers.redhat.com/articles/2026/07/06/track-model-usage-openshift-ai-usage-dashboard).

### 1.1 Token Metrics (Billing)

These are the **primary billing data** — what the tenant is charged for.

| Metric | Source | Type | Description | Rate card |
|--------|--------|------|-------------|-----------|
| `prompt_tokens` | IPP plugin CloudEvent | Per-request | Input tokens sent by tenant | $/1K input tokens |
| `completion_tokens` | IPP plugin CloudEvent | Per-request | Output tokens generated | $/1K output tokens (higher rate) |
| `total_tokens` | IPP plugin CloudEvent | Per-request | prompt + completion | Quick aggregation |
| `cached_input_tokens` | IPP plugin CloudEvent | Per-request | Tokens served from KV cache | $/1K cached tokens (discounted) |
| `cache_creation_tokens` | IPP plugin CloudEvent | Per-request | Tokens used creating cache (Anthropic) | $/1K cache write tokens |
| `reasoning_tokens` | IPP plugin CloudEvent | Per-request | Chain-of-thought tokens (OpenAI) | $/1K reasoning tokens (premium) |
| `model` | IPP plugin CloudEvent | Per-request | Which model was used | Rate card is per-model |
| `duration_ms` | IPP plugin CloudEvent | Per-request | Request duration | SLA tracking (not billed) |

**Source**: IPP external-metering-streaming plugin ([PR #332](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/332)) running on `maas-default-gateway`. Verified from `reportUsageEvent()` in `plugin.go`.

### 1.2 Token Cross-Check Metrics (Billing — verification)

Independent sources that should agree with IPP CloudEvent token counts. Used for data integrity, not invoicing.

| Metric | Source | Type | Description |
|--------|--------|------|-------------|
| `authorized_hits` | Limitador (Prometheus) | Counter | Tokens consumed per subscription/model — primary cross-check |
| `vllm:prompt_tokens_total` | vLLM (Prometheus) | Counter | Total prompt tokens processed by vLLM engine |
| `vllm:generation_tokens_total` | vLLM (Prometheus) | Counter | Total generation tokens produced by vLLM engine |
| `vllm:request_success_total` | vLLM (Prometheus) | Counter | Completed requests by finish reason (stop, length, abort) |

**Labels on Limitador metrics** (via `TelemetryPolicy` CR, requires `captureUser: true` in Tenant CR):

| Label | Source | Value |
|-------|--------|-------|
| `model` | `responseBodyJSON("/model")` | `granite-3.1-8b-instruct-fp8` |
| `user` | `auth.identity.userid` | `jdoe` |
| `subscription` | `auth.identity.selected_subscription` | `ai-tenant-acme/enterprise-sub` |
| `organization_id` | `auth.identity.subscription_info.organizationId` | `acme-corp` |
| `cost_center` | `auth.identity.subscription_info.costCenter` | `engineering` |

### 1.3 Gateway Operational Metrics (Not Billing)

| Metric | Source | Type | Description | Why not billing |
|--------|--------|------|-------------|----------------|
| `authorized_calls` | Limitador | Counter | Successful inference requests | Tokens are the billing unit, not request count |
| `rate_limited` | Limitador | Counter | Rate-limited requests (HTTP 429) | Informational — shows budget consumption |
| `auth_server_authconfig_duration_seconds` | Authorino | Histogram | Auth processing time | SLA monitoring |
| `istio_request_duration_milliseconds_bucket` | Istio sidecar | Histogram | Request latency | SLA monitoring |

### 1.4 Model Performance Metrics (Operational)

| Metric | Source | Type | Description | Why not billing |
|--------|--------|------|-------------|----------------|
| `vllm:time_to_first_token_seconds` | vLLM | Histogram | TTFT latency | SLA — unless CSP offers latency tiers |
| `vllm:e2e_request_latency_seconds` | vLLM | Histogram | End-to-end request latency | SLA monitoring |
| `vllm:inter_token_latency_seconds` | vLLM | Histogram | Token-to-token latency (TPOT) | SLA monitoring |
| `vllm:num_requests_running` | vLLM | Gauge | Requests in execution | Capacity planning |
| `vllm:num_requests_waiting` | vLLM | Gauge | Requests in queue | Capacity planning |
| `vllm:kv_cache_usage_perc` | vLLM | Gauge | KV cache utilization (0-1) | Model density optimization |
| `vllm:request_queue_time_seconds` | vLLM | Histogram | Queue wait time | Capacity planning |
| `vllm:prefix_cache_hits` / `prefix_cache_queries` | vLLM | Counter | Prefix cache hit rate | If CSP discounts cached tokens |
| `vllm:request_prompt_tokens` | vLLM | Histogram | Per-request prompt token distribution | Statistical analysis |
| `vllm:request_generation_tokens` | vLLM | Histogram | Per-request generation token distribution | Statistical analysis |

### 1.5 Infrastructure/Capacity Metrics (CSP Internal)

These are relevant only if the CSP bills tenants for infrastructure (GPU-hours, CPU-hours). In the MaaS model where **the CSP absorbs capacity cost and tenants pay only for tokens**, these metrics are purely operational — they stay in per-cluster Prometheus/Perses dashboards for CSP monitoring. No billing pipeline needed.

| Metric | Source | Type | Description | Feeds rate card (if billed) |
|--------|--------|------|-------------|-----------------------------|
| `DCGM_FI_DEV_GPU_UTIL` | DCGM exporter | Gauge | GPU utilization % | GPU-hours × $/hr |
| `DCGM_FI_DEV_FB_USED` / `FB_FREE` | DCGM exporter | Gauge | GPU memory used/free (MiB) | GPU tier pricing |
| `DCGM_FI_DEV_POWER_USAGE` | DCGM exporter | Gauge | GPU power consumption (W) | Energy billing (rare) |
| `DCGM_FI_DEV_GPU_TEMP` | DCGM exporter | Gauge | GPU temperature | Hardware health only |
| `container_cpu_usage_seconds_total` | kubelet/cAdvisor | Counter | CPU consumption per container | CPU core-hours × $/hr |
| `container_memory_working_set_bytes` | kubelet/cAdvisor | Gauge | Memory consumption per container | Memory GiB-hours × $/hr |
| `kube_node_status_condition` | kube-state-metrics | Gauge | Node health | Cluster health |
| `kube_pod_status_phase` | kube-state-metrics | Gauge | Pod lifecycle | Scheduling |

### 1.6 Where Metrics Are Today

All metrics listed above are **already collected** on each tenant cluster by our Ansible role deployment:

| Scraper | Metrics scraped | Visible in |
|---------|----------------|------------|
| User-workload Prometheus | vLLM, Limitador | Perses dashboards (Model, Usage tabs) |
| Cluster Prometheus | DCGM, cAdvisor, kube-state-metrics | Perses dashboards (Cluster tab) |
| Data-science MonitoringStack | Limitador, Authorino | Perses dashboards (Usage tab) |

**What's missing**: The IPP plugin (Section 2) — per-request CloudEvents don't exist yet. All other metrics are already flowing.

---

## 2. Usage/Token Metering Architecture

### 2.1 The Plugin

The IPP external-metering-streaming plugin ([PR #332](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/332), merged to `main` July 8) intercepts every inference response, extracts token usage from the response body, and emits a CloudEvents v1.0 event via HTTP POST.

**PR lineage**: [PR #320](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/320) (original, CLOSED July 8) → superseded by [PR #332](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/332) (streaming variant, MERGED). PR #332 handles both streaming and non-streaming responses via the ChunkProcessor interface.

Additional metering PRs merged:
- [PR #389](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/389) — recover usage from response payloads split across Envoy chunk boundaries (MERGED July 8)
- [PR #393](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/393) — report provider error responses as metering events (MERGED July 9)

### 2.2 How It Integrates

The PayloadProcessor is a **separate Deployment** (ext-proc gRPC service), not a Wasm filter injected into the gateway. The gateway calls it via Envoy's ext-proc protocol:

```
maas-default-gateway (Envoy)
  │
  ├─ 1. ext-authz → Authorino (auth, injects x-maas-* headers)
  ├─ 2. ext-ratelimit → Limitador (token budget check)
  └─ 3. ext-proc → PayloadProcessor Deployment
                    │
                    │  Plugin chain (PayloadProcessorConfig):
                    │  ┌─────────────────────────────────────────────┐
                    │  │ 1. model-provider-resolver                  │
                    │  │    (maps model name → backend)              │
                    │  │ 2. maas-headers-guard                       │
                    │  │    (captures x-maas-username, x-maas-group, │
                    │  │     x-maas-subscription into CycleState)    │
                    │  │ 3. external-metering-streaming              │
                    │  │    (reads CycleState + response tokens      │
                    │  │     → emits CloudEvent to hub)              │
                    │  └─────────────────────────────────────────────┘
                    │
                    └─ HTTP POST CloudEvent → meteringURL endpoint
```

### 2.4 CloudEvent Schema

Verified from `reportUsageEvent()` in PR #332's `plugin.go` source code:

```json
{
  "specversion": "1.0",
  "id": "evt-a3f7c2d1-9e4b-4a1f-b8c3-2d5e6f7a8b9c",
  "source": "maas-gateway",
  "type": "inference.tokens.used",
  "subject": "jdoe",
  "time": "2026-07-09T10:30:00Z",
  "datacontenttype": "application/json",
  "data": {
    "user": "jdoe",
    "group": "tier-enterprise-users",
    "subscription": "ai-tenant-acme/enterprise-sub@llm/granite-3-1-8b-fp8",
    "provider": "vllm",
    "model": "granite-3.1-8b-instruct-fp8",
    "prompt_tokens": 150,
    "completion_tokens": 80,
    "total_tokens": 230,
    "cached_input_tokens": 0,
    "cache_creation_tokens": 0,
    "reasoning_tokens": 0,
    "duration_ms": 450
  }
}
```

**Transport**: HTTP POST to `/api/v1/events` on the configured `meteringURL`. Accepts 200/204. Timeout: 5 seconds (configurable). Max response body: 1 MiB.

### 2.5 ExternalModel (Cross-Cluster) Scenario

Metering happens at the **tenant's** `maas-default-gateway`, not the upstream:

```
Tenant cluster:
  User → maas-default-gateway → Authorino headers injected
       → PayloadProcessor (IPP captures tenant identity + tokens)
       → maas-proxy → upstream cluster → model → response

  CloudEvent: user=jdoe, tenant=acme, model=granite, tokens=230 ✅

Upstream cluster:
  No separate metering needed — tenant-side CloudEvent is authoritative
```

### 2.6 Plugin Configuration

```yaml
plugins:
- type: external-metering-streaming
  name: metering
  parameters:
    meteringURL: http://<hub-endpoint>:8080
    failOpen: true
    source: maas-gateway
    timeoutSeconds: 5
```

`failOpen: true` — metering failures don't block inference.

### 2.7 MaaS CRD Billing Fields

The billing-relevant fields already exist on the MaaSSubscription CRD in RHOAI 3.4.2 (confirmed on live cluster):

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSSubscription
metadata:
  name: enterprise-subscription
  namespace: models-as-a-service
spec:
  owner:
    groups:
      - name: tier-enterprise-users
  modelRefs:
    - name: granite-3-1-8b-fp8
      namespace: granite-serving
      tokenRateLimits:
        - limit: 100000
          window: 1m
      billingRate:
        perToken: "0.003"            # rate card — exists on CRD, unused today
  tokenMetadata:                     # billing attribution — exists on CRD, unused today
    organizationId: acme-corp
    costCenter: engineering
    labels:
      department: ai-research
  priority: 2
```

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSAuthPolicy
metadata:
  name: enterprise-model-access
  namespace: models-as-a-service
spec:
  modelRefs:
    - name: granite-3-1-8b-fp8
      namespace: granite-serving
  subjects:
    groups:
      - name: tier-enterprise-users
  meteringMetadata:                  # exists on CRD, unused today
    organizationId: acme-corp
    costCenter: engineering
```

Source: [MaaSSubscription types](https://github.com/opendatahub-io/models-as-a-service/blob/main/maas-controller/api/maas/v1alpha1/maassubscription_types.go), [MaaSAuthPolicy types](https://github.com/opendatahub-io/models-as-a-service/blob/main/maas-controller/api/maas/v1alpha1/maasauthpolicy_types.go)

---

## 3. Tenant Attribution

### 3.1 Phase 1: 1 Cluster = 1 Tenant

In Phase 1, every request on a tenant cluster belongs to that tenant. The cluster identity IS the tenant. The IPP plugin's `source` field can be set per-cluster (e.g., `maas-gateway/tenant-acme`). No parsing needed.

### 3.2 Future: AITenant Multi-Tenancy (1 Cluster ... N Tenants)

With AITenant enabled, subscriptions live in `ai-tenant-{tenantName}` namespaces. The Authorino-resolved subscription key format is:

```
{subscriptionNamespace}/{subscriptionName}@{modelNamespace}/{modelName}
```

Example: `ai-tenant-acme/enterprise-sub@llm/granite-3-1-8b-fp8` → tenant = `acme`

### 3.3 Three Approaches (incremental)

| # | Approach | Change | Effort | Status |
|---|----------|--------|--------|--------|
| 1 | **Parse subscription key** | String split on `/`, strip `ai-tenant-` prefix | Zero code on plugin side — parsing in consumer or plugin config | Validate on live cluster |
| 2 | **Upstream PR: propagate TokenMetadata** | Wire `MaaSSubscription.tokenMetadata.organizationId` → `X-MaaS-OrgId` header → CloudEvent `organization_id` field | ~20 lines in Authorino + IPP plugin | [PR #386](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/386) (DRAFT, `work-in-progress`) |
| 3 | **Lookup table on hub** | Cost Management maintains org/project mapping | Config, not code | Future |

### 3.4 CRD Fields Already Exist

The `MaaSSubscription` CRD has `tokenMetadata.organizationId` and `tokenMetadata.costCenter` — confirmed on RHOAI 3.4.2 live cluster. These fields exist but are:
- Not populated by our Ansible role
- Not propagated to CloudEvents by the IPP plugin (no upstream wiring yet)

[PR #386](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/386) adds this wiring. Currently DRAFT.

---

## 4. Infrastructure/Capacity Metrics

### 4.1 Two Protocols, Each Native

Token metering and infrastructure metering have fundamentally different data shapes:

| | Token metering (IPP) | Infrastructure metering (OTel) |
|--|---------------------|-------------------------------|
| **Data shape** | Discrete event (user X used Y tokens at time T) | Continuous time series (GPU was 85% at T, 87% at T+1) |
| **Protocol** | CloudEvents (HTTP POST) | OTLP (gRPC) |
| **Consumer** | Billing platform (M360) | Cost Management (Cost Management) |
| **Billing unit** | Per token | Per core-hour / GPU-hour |

### 4.2 CSP Absorbs Capacity Cost → Infra Metrics Are Operational Only

In the agreed MaaS billing model, **tenants pay for tokens, CSP absorbs infrastructure cost**. This means:

- Infra metrics (GPU utilization, CPU/memory usage) are for **CSP internal cost tracking** (margin analysis)
- They do NOT need to flow to the billing pipeline
- No OTLP push to hub needed for billing purposes

**Only if** the CSP bills for infrastructure (GPU-hours, CPU-hours) would these metrics need to flow to the hub — and that's capacity billing, which is **CaaS scope** (MaaS sits on top of CaaS).

### 4.3 If OTLP Push Is Needed (Phase 2) [Owner: Who? CaaS Team / OSAC Infra]

The OTel Collector (already deployed by our Ansible role) can be configured to push billing-relevant metrics to a hub OTel Collector:

```
Tenant Cluster                              Hub Cluster
┌───────────────────────┐                  ┌───────────────────────────┐
│ Prometheus             │                  │  Hub OTel Collector       │
│       │                │   OTLP/gRPC      │  (receives from all       │
│       ▼                │─────────────────▶│   tenant clusters)        │
│ OTel Collector         │                  │          │                │
│ (filter: billing only, │                  │          ▼                │
│  add cluster_id label) │                  │  Cost Management (Cost Management)   │
└───────────────────────┘                  └───────────────────────────┘
```

Config change only — `filter/billing` processor + `otlp/hub` exporter. No new infrastructure on the tenant cluster. Hub OTel Collector endpoint is OSAC Infra's responsibility.

---

## 5. Deliverables & Implementation Plan

### 5.1 Phase 1: RHOAI 3.4.2 (1 Cluster = 1 Tenant)

**What we deliver**: Token metering via IPP plugin.
**Tenant attribution**: Cluster identity = tenant (no parsing needed).
**Image**: Use `noyitz` fork image (PR #332 is on `main` but not on `stable`/`odh-stable` yet).

#### Ansible Role Changes

**New task file**: `configure_metering.yaml` — gated by `enable_metering` (default: `false`)

**New Jinja2 templates** (deployed via `kubernetes.core.k8s`, no Helm):

| Template | Creates | Purpose |
|----------|---------|---------|
| `payload-processor.yaml.j2` | Deployment + Service + ConfigMap | PayloadProcessor pod with metering plugin chain |
| `envoyfilter-metering.yaml.j2` | Istio EnvoyFilter | Tells `maas-default-gateway` to call PayloadProcessor via ext-proc |

**Upstream Helm reference** (what our Jinja2 templates replicate):

| Helm chart resource | Our Jinja2 equivalent |
|--------------------|-----------------------|
| `upstreamIpp.payloadProcessor` (Deployment) | `payload-processor.yaml.j2` Deployment section |
| `upstreamIpp.customConfig` (ConfigMap) | `payload-processor.yaml.j2` ConfigMap section |
| Istio EnvoyFilter (auto-generated) | `envoyfilter-metering.yaml.j2` |

**New parameters** in `defaults/main.yaml`:

```yaml
ocp_4_20_ai_maas_enable_metering: false
ocp_4_20_ai_maas_metering_endpoint: ""
ocp_4_20_ai_maas_metering_image: "quay.io/opendatahub/odh-ai-gateway-payload-processing:odh-stable"
ocp_4_20_ai_maas_metering_fail_open: true
ocp_4_20_ai_maas_metering_timeout: 5
ocp_4_20_ai_maas_metering_source: "maas-gateway"
```

#### Implementation Steps

1. Add `configure_metering.yaml` task file to the role
2. Add `templates/payload-processor.yaml.j2` (Deployment + Service + ConfigMap)
3. Add `templates/envoyfilter-metering.yaml.j2` (ext-proc filter on `maas-default-gateway`)
4. Gate deployment behind `enable_metering` flag (default: `false`)
5. Only deploy when `enable_maas=true`
6. Image: fork image for Phase 1, `odh-stable` when promoted

#### Demo Setup

Self-contained demo using [noyitz/ai-gateway-metering-service](https://github.com/noyitz/ai-gateway-metering-service) (standalone with PostgreSQL + built-in dashboards):

```bash
# 1. Deploy metering service
oc create namespace maas-metering
oc apply -f deploy/postgres.yaml -n maas-metering
oc apply -f deploy/metering-service.yaml -n maas-metering
oc expose svc/metering-service -n maas-metering

# 2. Deploy MaaS with metering enabled
ansible-playbook playbook_osac_create_maas_deployment.yml --tags full -v \
  -e ocp_4_20_ai_maas_enable_metering=true \
  -e ocp_4_20_ai_maas_metering_endpoint=http://metering-service.maas-metering.svc:8080

# 3. Send inference requests → CloudEvents → dashboard
DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
echo "Dashboard: http://metering-service-maas-metering.${DOMAIN}/dashboard"
```

### 5.2 Phase 2: RHOAI 3.5+ (1 Cluster ... N Tenants)

**What we deliver**: Token metering + tenant attribution.
**Tenant attribution**: Parse from subscription key (`ai-tenant-{name}/sub@model`) or via [PR #386](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/386) `organization_id` propagation.
**Image**: Official `odh-stable` (if `main→stable` promoted before RHOAI 3.5 GA Aug 20).

#### Additional Implementation

1. Populate `tokenMetadata.organizationId` and `costCenter` on MaaSSubscription CRs (Ansible role template change)
2. Validate subscription key format with AITenant on live cluster
3. Switch from fork image to official `odh-stable` image
4. If [PR #386](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/386) merges: CloudEvent carries `organization_id` directly — no consumer-side parsing

---

## 6. Upstream PR Status & RHOAI Availability

### 6.1 PR Status (as of July 10, 2026)

| PR | What | State | On `main`? | On `stable`? |
|----|------|-------|-----------|-------------|
| [#320](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/320) | External metering plugin (original) | **Closed** | No | No |
| [#332](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/332) | Streaming token metering (successor) | **Merged** July 8 | Yes | **No** — last promotion July 2 |
| [#386](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/386) | Propagate organization_id/cost_center | **Draft** (`work-in-progress`) | No | No |
| [#389](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/389) | Recover usage from split chunks | **Merged** July 8 | Yes | No |
| [#393](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/393) | Report error responses as events | **Merged** July 9 | Yes | No |

JIRA: RHAISTRAT-1919 — Dev Preview tracking for external metering.

### 6.2 RHOAI 3.5 Availability

RHOAI builds from the `odh-stable` image tag (tracks `stable` branch). Promotion from `main` to `stable` uses `promote-main-to-stable.yml` (on-demand).

| PR | In RHOAI 3.5 GA (Aug 20)? | In EA2? | Dependency |
|----|--------------------------|---------|------------|
| #332 (metering) | **Possible** | **Possible** | Needs `main→stable` promotion |
| #386 (tenant attribution) | **???** | **???** | Draft, not on `main` |

---

## Appendix A: AI Grid Pipeline Reference

What AI Grid built for the July 7 Verizon demo:

| Component | Repo | What | Size |
|-----------|------|------|------|
| PayloadProcessor | [opendatahub-io/ai-gateway-payload-processing](https://github.com/opendatahub-io/ai-gateway-payload-processing) | Helm chart + IPP plugin (tenant-side) | Upstream |
| cloudevents-kafka-producer | `gitlab.cee.redhat.com/mavazque/ai-grid` | HTTP → Kafka bridge (hub-side `meteringURL` endpoint) | ~130 LOC Go |
| m360-forwarder | `gitlab.cee.redhat.com/mavazque/ai-grid` | Kafka → M360 REST API (Verizon-specific) | ~200 LOC Go |
| Dev metering service | [noyitz/ai-gateway-metering-service](https://github.com/noyitz/ai-gateway-metering-service) | Standalone test backend with PostgreSQL + dashboards | K8s manifests included |

---

## Appendix C: Data Integrity Cross-Checks

Three independent sources measure token usage. They should agree:

```
IPP plugin:     Σ (prompt_tokens + completion_tokens) per request
Limitador:      authorized_hits (from TelemetryPolicy)
vLLM:           prompt_tokens_total + generation_tokens_total

IPP ≈ Limitador ≈ vLLM  (within rounding)
```

Divergence indicates:
- IPP < Limitador: IPP plugin not intercepting all requests (EnvoyFilter misconfigured)
- IPP > Limitador: Limitador not counting all tokens (TelemetryPolicy labels mismatch)
- IPP ≠ vLLM: Model reports different count than response body (possible with streaming)

---

## References

- [IPP external-metering plugin PR #320](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/320) (closed, superseded)
- [Streaming metering PR #332](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/332) (merged, successor)
- [Tenant attribution PR #386](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/386) (draft)
- [Error event reporting PR #393](https://github.com/opendatahub-io/ai-gateway-payload-processing/pull/393) (merged)
- [vLLM Prometheus metrics](https://docs.vllm.ai/en/latest/design/metrics/)
- [Track model usage with RHOAI 3.4 usage dashboard](https://developers.redhat.com/articles/2026/07/06/track-model-usage-openshift-ai-usage-dashboard)
- [Kuadrant Limitador](https://github.com/Kuadrant/limitador)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [MaaSSubscription types](https://github.com/opendatahub-io/models-as-a-service/blob/main/maas-controller/api/maas/v1alpha1/maassubscription_types.go)
- [MaaSAuthPolicy types](https://github.com/opendatahub-io/models-as-a-service/blob/main/maas-controller/api/maas/v1alpha1/maasauthpolicy_types.go)
- [CloudEvents specification](https://cloudevents.io/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [AI Grid metering pipeline](https://gitlab.cee.redhat.com/mavazque/ai-grid) (internal)
- [Dev metering service](https://github.com/noyitz/ai-gateway-metering-service)
