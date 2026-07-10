# Shared Inference Cluster — Analysis & Implementation Design

How the `ocp_4_20_ai_maas` role serves both MaaS tenant clusters and inference-only shared clusters through parameterization — component analysis, parameter flow, gating design, and use case matrix.

## 1. Design Overview

The role has five feature flags plus two OIDC parameters in `defaults/main.yaml`:

```yaml
ocp_4_20_ai_maas_enable_maas: true           # MaaS gateway + auth + rate limiting
ocp_4_20_ai_maas_enable_keycloak: true        # Per-cluster Keycloak (Phase 1 default)
ocp_4_20_ai_maas_oidc_issuer: ""             # External OIDC issuer URL
ocp_4_20_ai_maas_oidc_client_id: ""          # External OIDC client ID
ocp_4_20_ai_maas_enable_observability: true   # COO + OTel + Perses dashboards
ocp_4_20_ai_maas_enable_dashboard: true       # RHOAI web console
ocp_4_20_ai_maas_run_verify: true            # Smoke tests after deployment
```

`post_install.yaml` orchestrates with `configure_cluster.yaml` (always) before `configure_maas.yaml` (gated):

```yaml
- name: Configure cluster (gateway, observability, dashboard)
  ansible.builtin.include_tasks: configure_cluster.yaml

- name: Configure MaaS platform (Phases 1-5)
  ansible.builtin.include_tasks: configure_maas.yaml
  when: ocp_4_20_ai_maas_enable_maas | default(true)
```

## 2. Component Analysis

### Operators (install_operators.yaml — 10 operators)

| # | Operator | Purpose | Inference-Only? |
|---|----------|---------|-----------------|
| 1 | **cert-manager** (1.18) | TLS certificates, RHOAI dependency | YES |
| 2 | **Service Mesh 3** | Gateway API controller. MaaS: pinned to 3.2 (`stable-3.2` channel) for RHCL compatibility. Inference-only: auto-installed by RHOAI via `stable` channel (no pinning) | Conditional — MaaS: manual install; Inference: auto via RHOAI |
| 3 | **Authorino** (1.3.1) | Auth enforcement (MaaS Gate 1) | NO — MaaS only |
| 4 | **Limitador** (1.3.1) | Rate limiting (MaaS Gate 2) | NO — MaaS only |
| 5 | **DNS Operator** (1.3.1) | Kuadrant DNS management | NO — MaaS only |
| 6 | **RHCL** (1.3.4) | Kuadrant API gateway policy engine | NO — MaaS only |
| 7 | **LWS** (1.0) | Distributed inference (llm-d) | YES |
| 8 | **RHOAI** (3.4) | KServe + dashboard + model serving | YES |
| 9 | **OTel** | Metrics collection | YES (default ON) |
| 10 | **COO** (1.4) | Perses dashboards | YES (default ON) |

### MaaS Configuration (configure_maas.yaml — 5 phases)

| Phase | What | File | Inference-Only? |
|-------|------|------|-----------------|
| Phase 1: Keycloak | RHBK + realm + users | configure_maas.yaml | NO — MaaS identity |
| Phase 2: MaaS Prerequisites | Kuadrant, Authorino, gateway, PostgreSQL | configure_maas.yaml | NO — MaaS gateway |
| Phase 3: Enable MaaS | DSC MaaS component, Tenant CR, MaaS dashboard flags (`modelAsService`, `maasAuthPolicies`) | configure_maas.yaml | NO — MaaS platform |
| Cluster: `openshift-ai-inference` gateway | Gateway for LLMInferenceService routing | configure_cluster.yaml | YES — always runs |
| Cluster: Dashboard flags | `vLLMDeploymentOnMaaS`, `llmGatewayField`, `deploymentWizardYAMLViewer`, `observabilityDashboard` | configure_cluster.yaml | YES — always runs |
| Cluster: Observability verify | Monitoring CR, Perses, datasources, dashboards | configure_cluster.yaml | YES — always runs |
| Phase 4: Deploy Model | LLMInferenceService + MaaSModelRef | configure_maas.yaml | NO — MaaS model |
| Phase 5: Auth + Subscriptions | AuthPolicy, Subscriptions, rate limiting | configure_maas.yaml | NO — MaaS governance |
### DSC Components

| Component | Current | Inference-Only |
|-----------|---------|----------------|
| dashboard | Managed | Managed (default ON) |
| kserve | Managed | Managed |
| All others | Removed | Removed |

## 3. Parameter Flow: Fulfillment API → AAP → Role

The OSAC platform has a complete parameter pipeline — **no changes needed in fulfillment-service or osac-operator:**

```
1. meta/osac.yaml defines `parameters` (TemplateParameterDefinition)
     ↓ (discovery pipeline reads at publish time)
2. Fulfillment API stores ClusterTemplate with parameter definitions
     ↓ (user creates cluster with template_parameters)
3. ClusterOrder CRD spec.templateParameters = JSON string
     ↓ (operator wraps in ansible_eda.event.payload)
4. AAP playbook receives cluster_order
     ↓ (cluster_settings role extracts templateParameters)
5. Template role receives as Ansible variables
```

**Key files in the flow:**

| Layer | File | What it does |
|-------|------|-------------|
| Discovery | `find_template_roles.py` `TemplateParameterDefinition` | Parses `parameters` from `meta/osac.yaml`, publishes to fulfillment API |
| Fulfillment | `cluster_template_type.proto` `ClusterTemplateParameterDefinition` | Stores parameter name, type, required, default |
| Fulfillment | `cluster_type.proto` `ClusterSpec.template_parameters` | User-provided values per cluster |
| Fulfillment | `template_parameters.go` | Validates, applies defaults, converts to JSON |
| Operator | `clusterorder_types.go` `ClusterOrderSpec.TemplateParameters` | JSON string in CRD |
| Operator | `aap_provider.go` `extractExtraVars()` | Wraps ClusterOrder in `ansible_eda.event.payload` |
| AAP | `cluster_settings/tasks/main.yml` | `templateParameters \| from_json` → dict |
| AAP | `extract_template_info/tasks/main.yaml` | Extracts `template_parameters` as variable |

## 4. Role Parameterization

### 4a. `meta/osac.yaml` — Parameter definitions

```yaml
---
title: OpenShift AI 4.20 Cluster with MaaS
description: >
  Builds an OpenShift 4.20 cluster with RHOAI 3.4, Models-as-a-Service (GA),
  Keycloak SSO, GPU support, and observability (COO + OTel). When MaaS is
  enabled, includes RHCL 1.3 version pinning and SM 3.2 for auth enforcement
  compatibility. Single-tenant deployment — one organization per cluster.
template_type: cluster
implementation_strategy: ocp_4_20_ai_maas
default_node_request:
  - resourceClass: g5
    numberOfNodes: 2
allowed_resource_classes: []
parameters:
  - name: enable_maas
    title: Enable MaaS Gateway
    description: >
      Install MaaS governance layer (auth, rate limiting, API keys).
      Set to false for inference-only shared clusters.
    type: boolean
    required: false
    default: true
  - name: enable_keycloak
    title: Enable Managed Keycloak
    description: >
      Deploy per-cluster Keycloak for MaaS identity management.
      Set to false when using shared OSAC Keycloak or BYOIDP.
      When false and enable_maas is true, oidc_issuer must be provided.
    type: boolean
    required: false
    default: true
  - name: oidc_issuer
    title: External OIDC Issuer URL
    description: >
      OIDC issuer URL for external identity provider (e.g., shared OSAC Keycloak
      or customer IdP). Kuadrant AuthPolicy auto-discovers JWKS via OIDC discovery
      ({issuerUrl}/.well-known/openid-configuration -> jwks_uri).
      Required when enable_maas is true and enable_keycloak is false.
    type: string
    required: false
  - name: oidc_client_id
    title: External OIDC Client ID
    description: >
      OIDC client ID registered in the external identity provider.
      Required when enable_maas is true and enable_keycloak is false.
    type: string
    required: false
  - name: enable_observability
    title: Enable Observability Stack
    description: >
      Install COO v1.4, OTel, and Perses dashboards for model monitoring.
    type: boolean
    required: false
    default: true
  - name: enable_dashboard
    title: Enable OpenShift AI Dashboard
    description: >
      Deploy the RHOAI web console for model management and monitoring.
      Set to false for headless inference clusters managed via API only.
    type: boolean
    required: false
    default: true
```

These parameters flow through the fulfillment API pipeline automatically — no proto or server changes needed.

**Why all parameters are `required: false`:**

The fulfillment service enforces `required` as a flat boolean — if `required: true`, the user MUST provide a value on every ClusterOrder, regardless of other parameter values. There is no support for conditional requirements (e.g., "required only when X is false").

- **Boolean flags** (`enable_maas`, `enable_keycloak`, `enable_observability`, `enable_dashboard`): All have sensible `default` values. Setting `required: true` would force users to specify all four on every ClusterOrder even when defaults are fine. The fulfillment service applies the default when the user doesn't provide a value.

- **`oidc_issuer`**: Has no default (deployment-specific URL). It's `required: false` in the API because it's only needed when `enable_maas: true` AND `enable_keycloak: false`. The fulfillment service can't express this conditional requirement.

**Conditional validation happens in the Ansible role** (not the API). All three asserts run at the top of `install_operators.yaml` — before any operator installation begins. This ensures misconfiguration is caught in seconds, not after 20+ minutes of operator installation. A defense-in-depth copy of the BYOIDP assert also exists in `configure_maas.yaml` for `--tags maas` direct runs.

```yaml
# install_operators.yaml (top of file, before any operator work):

- name: Validate enable_keycloak requires enable_maas
  ansible.builtin.assert:
    that:
      - ocp_4_20_ai_maas_enable_maas | bool
    fail_msg: >-
      enable_keycloak=true requires enable_maas=true. Keycloak without MaaS
      has no gateway to authenticate against.
  when:
    - ocp_4_20_ai_maas_enable_keycloak | bool

- name: Validate BYOIDP parameters before operator installation
  ansible.builtin.assert:
    that:
      - ocp_4_20_ai_maas_oidc_issuer is defined
      - ocp_4_20_ai_maas_oidc_issuer | length > 0
      - ocp_4_20_ai_maas_oidc_client_id is defined
      - ocp_4_20_ai_maas_oidc_client_id | length > 0
    fail_msg: >-
      oidc_issuer and oidc_client_id are required when enable_maas=true and
      enable_keycloak=false. Provide the OIDC issuer URL and client ID of
      your external identity provider.
  when:
    - ocp_4_20_ai_maas_enable_maas | bool
    - not (ocp_4_20_ai_maas_enable_keycloak | bool)

- name: Validate shell-interpolated variables (prevent injection)
  ansible.builtin.assert:
    that:
      - ocp_4_20_ai_maas_reject_version is regex('^v[\d.]+$')
      # ... (CSV name validation)
```

This is the correct layering: the API validates parameter types and presence of required fields, the Ansible role validates cross-parameter business logic.

### 4b. Per-flag gating design

Each flag controls which tasks run across three task files (`install_operators.yaml`, `configure_maas.yaml`/`configure_cluster.yaml`, `verify.yaml`).

#### `enable_maas` (default: `true`)

**`install_operators.yaml`:**

| Task | Gate |
|------|------|
| Assert: `enable_keycloak` requires `enable_maas` | `when: enable_keycloak` |
| Assert: BYOIDP requires `oidc_issuer` + `oidc_client_id` | `when: enable_maas AND NOT enable_keycloak` |
| Assert: Shell variable injection prevention | Always |
| Steps 2-4: Ingress Operator scale-down, SM 3.2 (pinned), sub-operators (Authorino/Limitador/DNS v1.3.1), RHCL v1.3.4, hold-until-clean approval, Ingress Operator scale-up | `block: when: enable_maas` |

When `false`: SM 3 auto-installs via RHOAI DSC. No version pinning, no sub-operators, no Ingress Operator workaround. The entire SM/RHCL OLM complexity disappears.

**`configure_maas.yaml`** (file-level gate: `when: enable_maas`):

| Task | Gate |
|------|------|
| Keycloak (Phase 1) | `enable_keycloak` |
| Kuadrant, Authorino, gateway, PostgreSQL (Phase 2) | File-level |
| DSC MaaS enable, Tenant CR (Phase 3) | File-level |
| MaaS dashboard flags (`modelAsService`, `maasAuthPolicies`) | `enable_dashboard` |
| Model deploy + MaaSModelRef (Phase 4) | File-level |
| Auth + Subscriptions + rate limiting (Phase 5) | File-level |

**`verify.yaml`:**

| Test | Gate |
|------|------|
| v1.4.0 RHCL CSV assertion | `enable_maas` |
| Auth 401 unauthenticated (with retries 6x, 10s for Wasm warmup) | `enable_maas` |

#### `enable_keycloak` (default: `true`)

No tasks in `install_operators.yaml` — Keycloak RHBK is a MaaS resource installed in `configure_maas.yaml` Phase 1.

**`configure_maas.yaml`:**

| Task | Gate |
|------|------|
| Phase 1: RHBK operator, PostgreSQL, Keycloak instance, realm, users | `enable_keycloak` |
| Tenant CR `externalOIDC` | Conditional: managed Keycloak URL when `enable_keycloak`, else `oidc_issuer` + `oidc_client_id` |

**`verify.yaml`:**

| Test | Gate |
|------|------|
| Get Keycloak JWT | `enable_keycloak` |
| Create API key via JWT | `enable_keycloak` |
| Inference via MaaS gateway | `enable_keycloak` |
| Rate limiting (free tier → 429) | `enable_keycloak` |

Auth/inference/rate-limiting tests require Keycloak JWT for API key creation. When `enable_maas: true` + `enable_keycloak: false` (BYOIDP), these tests are skipped — customers test with their own IdP.

#### `enable_observability` (default: `true`)

**`install_operators.yaml`:**

| Task | Gate |
|------|------|
| OTel operator (isolated namespace) | `enable_observability` |
| COO v1.4 operator (isolated namespace, Manual approval) | `enable_observability` |
| DSCI metrics patch (replicas, retention, storage) | `enable_observability` |

**`configure_cluster.yaml`:**

| Task | Gate |
|------|------|
| User Workload Monitoring ConfigMap | `enable_observability` |
| `observabilityDashboard` OdhDashboardConfig flag | `enable_observability AND enable_dashboard` |
| Monitoring CR, Perses, datasources | `enable_observability` |
| PersesDashboards wait | `enable_observability AND enable_dashboard` |

**`verify.yaml`:**

| Test | Gate |
|------|------|
| COO CSV Succeeded | `enable_observability` |
| OTel CSV Succeeded | `enable_observability` |
| Perses pod Running | `enable_observability` |
| Observability stack assertion | `enable_observability` |

#### `enable_dashboard` (default: `true`)

**`install_operators.yaml`:**

| Task | Gate |
|------|------|
| DSC `dashboard` component | `set_fact`: `Managed` when true, `Removed` when false |

**OdhDashboardConfig flags** are split across two files:

**`configure_cluster.yaml`** (always runs):
- `observabilityDashboard`: `when: enable_observability AND enable_dashboard`
- `deploymentWizardYAMLViewer`, `vLLMDeploymentOnMaaS`, `llmGatewayField`: `when: enable_dashboard`

**`configure_maas.yaml`** (runs when `enable_maas: true`):
- `modelAsService`, `maasAuthPolicies`: `when: enable_dashboard`

`vLLMDeploymentOnMaaS` and `llmGatewayField` are cluster-level because `vLLMDeploymentOnMaaS` only requires `kserve: Managed` in the DSC (not `modelsAsService: Managed`). This enables the dashboard Form wizard to create LLMInferenceService on inference-only clusters.

**`verify.yaml`:** `enable_dashboard` included in final report message.

### 4c. `post_install.yaml` — Task orchestration

`post_install.yaml` orchestrates four task files in sequence. Cluster-level configuration (`configure_cluster.yaml`) runs unconditionally; MaaS-specific configuration (`configure_maas.yaml`) is gated by `enable_maas`.

```yaml
- name: Install platform operators
  ansible.builtin.include_tasks: install_operators.yaml

- name: Configure cluster (gateway, observability, dashboard)
  ansible.builtin.include_tasks: configure_cluster.yaml

- name: Configure MaaS platform
  ansible.builtin.include_tasks: configure_maas.yaml
  when: ocp_4_20_ai_maas_enable_maas | default(true)

- name: Run verification smoke tests
  ansible.builtin.include_tasks: verify.yaml
  when: ocp_4_20_ai_maas_run_verify | default(true)
```

**Why two configure files:** The `openshift-ai-inference` gateway and observability verification must run regardless of MaaS — LLMInferenceService needs the gateway for routing, and metrics collection is independent of auth/rate-limiting. Placing these in `configure_cluster.yaml` (always runs) keeps `configure_maas.yaml` focused on MaaS-only tasks (Keycloak, Kuadrant, Tenant CR, subscriptions).

### 4d. `defaults/main.yaml` — Role defaults and variable bridging

```yaml
# Feature flags
ocp_4_20_ai_maas_enable_maas: true
ocp_4_20_ai_maas_enable_keycloak: true
ocp_4_20_ai_maas_oidc_issuer: ""
ocp_4_20_ai_maas_oidc_client_id: ""
ocp_4_20_ai_maas_enable_observability: true
ocp_4_20_ai_maas_enable_dashboard: true
ocp_4_20_ai_maas_run_verify: true

# DSC components — dashboard controlled by enable_dashboard
ocp_4_20_ai_maas_dsc_components:
  codeflare: Removed
  dashboard: Managed      # overridden to Removed when enable_dashboard: false
  datasciencepipelines: Removed
  kserve: Managed
  kueue: Removed
  modelmeshserving: Removed
  ray: Removed
  trainingoperator: Removed
  trustyai: Removed
  workbenches: Removed
```

**Variable bridging** in `tasks/install.yaml` maps OSAC pipeline parameters (short names from `template_parameters`) to role variables (prefixed names):

```yaml
- name: Bridge templateParameters to role variables
  ansible.builtin.set_fact:
    ocp_4_20_ai_maas_enable_maas: "{{ template_parameters.enable_maas | default(true) | bool }}"
    ocp_4_20_ai_maas_enable_keycloak: "{{ (template_parameters.enable_keycloak | default(true) | bool) and (template_parameters.enable_maas | default(true) | bool) }}"
    ocp_4_20_ai_maas_oidc_issuer: "{{ template_parameters.oidc_issuer | default('') }}"
    ocp_4_20_ai_maas_oidc_client_id: "{{ template_parameters.oidc_client_id | default('') }}"
    ocp_4_20_ai_maas_enable_observability: "{{ template_parameters.enable_observability | default(true) | bool }}"
    ocp_4_20_ai_maas_enable_dashboard: "{{ template_parameters.enable_dashboard | default(true) | bool }}"
  when: template_parameters is defined
```

This task only runs in the OSAC pipeline (where `extract_template_info` parses `spec.templateParameters` into `template_parameters` dict). For standalone testing with `-e`, `template_parameters` is not defined — the bridging is skipped and role defaults apply normally.

**Type handling:** Values arrive in two types depending on their source. User-provided values arrive as strings (`"false"`) from the API's `StringValue` wrapper. Fulfillment-applied defaults arrive as booleans (`true`) from the template's `BoolValue` default. Ansible's `| bool` filter handles both correctly: `"false" | bool` → `False`, `true | bool` → `True`.

**Keycloak constraint in bridging:** `enable_keycloak` is AND'd with `enable_maas` because the fulfillment service applies the `enable_keycloak=true` default regardless of `enable_maas`. Without the `and` constraint, setting `enable_maas=false` would still leave `enable_keycloak=true`, triggering the early assert.

### 4e. Use case matrix

| Use Case | enable_maas | enable_keycloak | enable_observability | enable_dashboard | Operators |
|----------|-------------|-----------------|---------------------|-----------------|-----------|
| Full MaaS tenant cluster | true | true | true | true | 10 |
| MaaS with BYOIDP | true | false | true | true | 9 (no RHBK) |
| Inference cluster with dashboard + monitoring | false | false | true | true | 5 + SM auto |
| Headless inference cluster (API-managed) | false | false | true | false | 5 + SM auto |
| Bare inference cluster | false | false | false | false | 3 + SM auto |

> **BYOIDP testing note**: Validated with RHBK deployed separately from the tenant cluster deployment, with both RHBK and the MaaS tenant hosted on the same cluster. The external Keycloak provides the OIDC issuer URL and client ID passed via `oidc_issuer` and `oidc_client_id` parameters.

### 4f. Flag interaction constraints

| Constraint | Reason | Enforced by | When |
|-----------|--------|-------------|------|
| `enable_keycloak: true` requires `enable_maas: true` | Keycloak without MaaS is not useful — no gateway to authenticate against | Two levels: (1) `and enable_maas` in bridging forces `enable_keycloak=false` when `enable_maas=false`; (2) `assert` in `install_operators.yaml` as defense-in-depth | Before any operator installation |
| `enable_maas: true` + `enable_keycloak: false` requires `oidc_issuer` AND `oidc_client_id` | MaaS needs an identity provider. Kuadrant AuthPolicy auto-discovers JWKS via `{oidc_issuer}/.well-known/openid-configuration` → `jwks_uri`. The `oidc_client_id` is set in the Tenant CR's `externalOIDC.clientId`. Without both, auth enforcement fails | `assert` in `install_operators.yaml` (early) + `configure_maas.yaml` (defense-in-depth) | Before any operator installation |
| Shell-interpolated variable injection | CSV names and version strings are used in shell commands | `assert` in `install_operators.yaml` | Before any operator installation |
| `observabilityDashboard` OdhDashboardConfig flag requires both `enable_observability` AND `enable_dashboard` | Dashboard must exist to show observability tabs | `when:` condition on task | |
| MaaS dashboard flags (`modelAsService`, `maasAuthPolicies`) require both `enable_maas` AND `enable_dashboard` | Dashboard must exist to show MaaS tabs | `when:` condition on task | |
| `enable_dashboard: false` with `enable_observability: true` | Metrics collected but no UI — accessible via Prometheus API/Thanos Querier only | Valid combination, no constraint | |

**Early fail design:** All three `assert` tasks run at the top of `install_operators.yaml` — before any operator installation begins. Misconfiguration is caught in seconds, not after 20+ minutes of operator work. This applies to both execution paths (OSAC pipeline via `post_install.yaml` and standalone playbook).

## 5. Behavior Comparison

| Aspect | Full MaaS (all true) | MaaS with BYOIDP | Inference + Dashboard | Headless Inference | Bare Inference |
|--------|---------------------|------------------|----------------------|--------------------|----------------|
| Operators installed | 10 | 9 (no RHBK) | 5 + SM auto | 5 + SM auto | 3 + SM auto |
| SM installation | Manual, pinned 3.2 | Manual, pinned 3.2 | Auto via RHOAI | Auto via RHOAI | Auto via RHOAI |
| Gateway | `maas-default-gateway` | `maas-default-gateway` | `openshift-ai-inference` | `openshift-ai-inference` | `openshift-ai-inference` |
| Identity | Managed Keycloak | External OIDC (`oidc_issuer`) | None | None | None |
| Database | PostgreSQL | PostgreSQL | None | None | None |
| Dashboard UI | RHOAI dashboard | RHOAI dashboard | RHOAI dashboard | None | None |
| Dashboard model deploy | LLMInferenceService | LLMInferenceService | LLMInferenceService | N/A | N/A |
| Metrics collection | COO + OTel + Perses | COO + OTel + Perses | COO + OTel + Perses | COO + OTel + Perses | None |
| Metrics UI | Dashboard Observe tab | Dashboard Observe tab | Dashboard Observe tab | Prometheus API only | None |
| Model tab | Shows data | Shows data | Shows data | CLI metrics only | None |
| OLM complexity | Hold-until-clean | Hold-until-clean | Automatic | Automatic | Automatic |
| Model access | MaaS gateway (auth + rate-limit) | MaaS gateway (auth + rate-limit) | Direct KServe | Direct KServe | Direct KServe |

## Sources

- [RHOAI 3.4 Deploying Models](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/pdf/deploying_models/)
- [RHOAI 3.4 Distributed Inference with llm-d](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/pdf/deploy_models_using_distributed_inference_with_llm-d/)
- [OpenShift AI + Service Mesh 3 coexistence](https://developers.redhat.com/articles/2025/07/16/how-deploy-openshift-ai-service-mesh-3-one-cluster)
- [SM 3.2 with Ambient Mode](https://www.redhat.com/en/blog/introducing-openshift-service-mesh-32-istios-ambient-mode)
- Live cluster verification (OCP 4.19, RHOAI 3.4.1, SM 3.2.6)
