# MaaS on OSAC — Design Documents

Design documents for delivering Models as a Service (MaaS) on the Open Sovereign AI Cloud (OSAC) platform. These cover architecture, deployment, and operational design for RHOAI 3.4 with MaaS governance on OpenShift.

## Documents

### Architecture & Design

| Document | Description |
|----------|-------------|
| [maas_cr.md](maas_cr.md) | MaaS Custom Resource reference — 7 CRDs + 5 consumed (dual-gate architecture) |
| [software_infra_maas.md](software_infra_maas.md) | Operators required for MaaS deployment — 14 total with per-use-case breakdown |
| [shared_inference_cluster.md](shared_inference_cluster.md) | Shared inference cluster parameterization — 5 deployment flavors from single Ansible role |
| [external_model_arch.md](external_model_arch.md) | Cross-cluster model access — 3 paths (manual Route, MaaS proxy, inference gateway) |
| [version_pinning.md](version_pinning.md) | RHCL 1.3.x / SM 3.2 version pinning strategy for MaaS auth enforcement |

### OSAC Integration

| Document | Description |
|----------|-------------|
| [model_services_service_proto_design.md](model_services_service_proto_design.md) | ModelServices fulfillment API resource — proto field-by-field design |
| [osac_operator_crd_design.md](osac_operator_crd_design.md) | ModelServiceOrder CRD + operator controller design |

### Deployment Guides

| Document | Description |
|----------|-------------|
| [bom_rhoai_3.4.md](bom_rhoai_3.4.md) | Bill of Materials — validated operator inventory with installation order |
| [maas/README.md](maas/README.md) | MaaS deployment guide — Keycloak SSO, auth enforcement, API keys, rate limiting |
| [maas/manifests/](maas/manifests/) | Kubernetes manifests for MaaS components (gateway, Keycloak, subscriptions, model) |

### Dashboard & UI

| Document | Description |
|----------|-------------|
| [deploy-model-ui/model-car-and-maas.md](deploy-model-ui/model-car-and-maas.md) | Deploy Granite 8B via modelcar + MaaS governance |
| [deploy-model-ui/hf-model.md](deploy-model-ui/hf-model.md) | Deploy models from HuggingFace via PVC/workbench |
| [deploy-model-ui/dashboard-config.md](deploy-model-ui/dashboard-config.md) | OdhDashboardConfig feature flag reference (50+ flags) |
| [deploy-model-ui/maas-observability.md](deploy-model-ui/maas-observability.md) | Monitoring stack: COO v1.4 + OTel + Perses dashboards |

### Planning

| Document | Description |
|----------|-------------|
| [maas-osac-jira-tasks.csv](maas-osac-jira-tasks.csv) | JIRA-importable tasks for Phase 1 architecture |

## Related Repositories

| Repo | Purpose |
|------|---------|
| [osac-project/osac-aap](https://github.com/osac-project/osac-aap) | Ansible automation — `ocp_4_20_ai_maas` template role |
| [opendatahub-io/models-as-a-service](https://github.com/opendatahub-io/models-as-a-service) | Upstream MaaS code (CRDs, controllers, maas-api) |
| [osac-project/fulfillment-service](https://github.com/osac-project/fulfillment-service) | OSAC Fulfillment API |
| [osac-project/osac-operator](https://github.com/osac-project/osac-operator) | OSAC hub-cluster operator |
