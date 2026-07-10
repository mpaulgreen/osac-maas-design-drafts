# Software Infrastructure for MaaS Deployment

*Operators required on an OpenShift cluster for a full MaaS deployment. All installed and validated on OCP 4.19.17 with RHOAI 3.4.2.*

## Prerequisites (expected to be pre-installed)

These operators are infrastructure-level GPU enablement — expected to be available on any GPU-capable OpenShift cluster before MaaS deployment begins.

| Operator | Version | Purpose |
|----------|---------|---------|
| **Node Feature Discovery (NFD)** | 4.19.0 | Detects hardware features on nodes (PCI devices, CPU flags). Labels GPU nodes with `nvidia.com/gpu.present=true` so workloads can target them via nodeSelector. |
| **NVIDIA GPU Operator** | 26.3.3 | Installs GPU drivers, device plugin, DCGM monitoring, and container toolkit on labeled nodes. Creates the `nvidia` RuntimeClass so containers can access GPUs. Managed via `ClusterPolicy` CR. |

## Operators Deployed by the MaaS Ansible Role

Our `ocp_4_20_ai_maas` Ansible role installs and configures the following operators. Installation order matters — RHCL sub-operators must be pinned before RHCL itself to prevent OLM auto-upgrading to v1.4.0.

### Platform Operators

| Operator | Version | Purpose |
|----------|---------|---------|
| **cert-manager** | 1.18.1 | TLS certificate lifecycle management. Issues certificates for MaaS gateway, Keycloak, and internal service communication via `ClusterIssuer`. |
| **Red Hat OpenShift Service Mesh 3** | 3.2.7 | Provides the Istio-based gateway infrastructure (`data-science-gateway-class`). The MaaS gateway (`maas-default-gateway`) and inference gateway (`openshift-ai-inference`) run on this. Pinned to 3.2.x for RHCL 1.3 compatibility. |
| **Red Hat Connectivity Link (RHCL)** | 1.3.5 | API gateway policy engine (Kuadrant). Manages `AuthPolicy` (Gate 1 — authentication) and `TokenRateLimitPolicy` (Gate 2 — rate limiting) on the MaaS gateway. Bundles Authorino, Limitador, and DNS operators. |
| **Authorino Operator** | 1.3.2 | External authorization service for Envoy. Validates OIDC tokens and MaaS API keys on every inference request. Injects `x-maas-username`, `x-maas-group`, `x-maas-subscription` headers into the request. |
| **Limitador Operator** | 1.3.1 | Distributed rate limiter. Enforces per-subscription token budgets (e.g., 100K tokens/min for enterprise tier). Emits `authorized_calls`, `authorized_hits`, and `rate_limited` Prometheus metrics used by the Usage dashboard. |
| **DNS Operator** | 1.3.1 | DNS management for Kuadrant. Handles DNS records for gateway endpoints. |
| **Leader Worker Set (LWS)** | 1.0.0 | Manages distributed inference workloads where a leader coordinates multiple worker replicas. Required by RHOAI for KServe model serving. |

### AI / Model Serving

| Operator | Version | Purpose |
|----------|---------|---------|
| **Red Hat OpenShift AI (RHOAI)** | 3.4.2 | The AI platform operator. Deploys KServe (model serving), MaaS controller (governance), maas-api (API key management), and the RHOAI dashboard. Configured via `DataScienceCluster` and `DSCInitialization` CRs. |

### Observability

| Operator | Version | Purpose |
|----------|---------|---------|
| **Cluster Observability Operator (COO)** | 1.4.0 | Deploys the Perses-based monitoring stack (MonitoringStack CR). Provides the Cluster, Model, and Usage dashboard tabs in the RHOAI console. Must be v1.4.0 — v1.5 is incompatible with RHOAI 3.4. |
| **Red Hat OpenTelemetry** | latest | Deploys OpenTelemetry Collectors for metrics and trace collection. Scrapes vLLM, Limitador, and DCGM metrics. Feeds the Perses dashboards and can push to external endpoints. |

### Identity (conditional)

| Operator | Version | When deployed | Purpose |
|----------|---------|---------------|---------|
| **Red Hat build of Keycloak (RHBK)** | 26.4.13 | `enable_keycloak=true` (default) | OIDC identity provider for MaaS. Creates the `maas` realm with tier groups (`tier-free-users`, `tier-premium-users`, `tier-enterprise-users`) and test users. Skipped in BYOIDP mode (`enable_keycloak=false`). |

## Deployment Flags

Not all operators are deployed in every use case. The Ansible role's feature flags control what gets installed:

| Flag | Default | What it controls |
|------|---------|-----------------|
| `enable_maas` | `true` | SM 3.2 (pinned), RHCL, Authorino, Limitador, DNS, MaaS gateway, Tenant, subscriptions |
| `enable_keycloak` | `true` | RHBK Keycloak operator + MaaS realm (auto-forced to `false` when `enable_maas=false`) |
| `enable_observability` | `true` | COO + OTel operators + Perses dashboards |
| `enable_dashboard` | `true` | RHOAI dashboard feature flags (`vLLMDeploymentOnMaaS`, `observabilityDashboard`, etc.) |
| `enable_metering` | `false` | PayloadProcessor for token metering (future — IPP external-metering plugin) |

## Operator Count by Use Case

| Use Case | Operators installed | Notes |
|----------|-------------------|-------|
| Full MaaS + Keycloak | 12 (+ 2 prereqs) | All operators |
| MaaS with BYOIDP | 11 (+ 2 prereqs) | No RHBK |
| Inference + Dashboard | 5 + SM auto | No RHCL/sub-operators, no Keycloak, no MaaS |
| Headless Inference | 5 + SM auto | Same as above, dashboard disabled |
| Bare Inference | 3 + SM auto | cert-manager, LWS, RHOAI only |
