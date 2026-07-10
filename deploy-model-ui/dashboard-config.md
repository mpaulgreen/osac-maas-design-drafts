# OdhDashboardConfig Feature Flags — RHOAI 3.4

Reference for all `spec.dashboardConfig` flags in the `OdhDashboardConfig` CR (`opendatahub.io/v1alpha`). Validated against the live CRD on OCP 4.19 with RHOAI 3.4.0.

## Polarity Rule

The schema uses two conventions:

- **`disable*` (negative) flags** — default to `false` when omitted, meaning the feature is **shown**. Set to `true` to **hide** the feature.
- **Plain positive flags** — default to `false` when omitted, meaning the feature is **hidden**. Set to `true` to **enable/show** the feature. These are **opt-in**.

## How to View and Edit

```bash
# View current settings
oc get odhdashboardconfig odh-dashboard-config -n redhat-ods-applications \
  -o jsonpath='{.spec.dashboardConfig}' | python3 -m json.tool

# View full schema from CRD
oc explain odhdashboardconfig.spec.dashboardConfig --recursive

# Patch (example: enable MaaS UI)
oc patch odhdashboardconfig odh-dashboard-config -n redhat-ods-applications \
  --type merge -p '{"spec":{"dashboardConfig":{"modelAsService": true}}}'
```

## Flag and DSC Component Pairing

Dashboard flags control **UI visibility only**. The backend capability is gated separately by the DataScienceCluster (DSC) component `managementState`. A mismatch (flag enabled, DSC component `Removed`) creates a visible-but-broken path.

| Dashboard Flag | DSC Component | Required State |
|----------------|---------------|----------------|
| `modelAsService: true` | `kserve.modelsAsService` | Managed |
| `genAiStudio: true` | `llamastackoperator` | Managed |
| `vLLMDeploymentOnMaaS: true` | `kserve` | Managed |
| `disablePipelines: false` | `aipipelines` | Managed |
| `disableDistributedWorkloads: false` | `kueue` | Managed (but kueue blocks Managed in 3.4 — use Removed) |
| `disableFeatureStore: false` | `feastoperator` | Managed |
| `mlflow: true` | `mlflowoperator` | Managed |

---

## MaaS and GenAI Flags (opt-in — hidden by default)

Set to `true` to enable.

| Flag | Purpose |
|------|---------|
| `modelAsService` | Shows the **Models-as-a-Service** UI — subscriptions, authorization policies, model publishing to MaaS gateway |
| `maasAuthPolicies` | Shows the **Authorization policies** management page for creating/editing MaaSAuthPolicy CRs from the dashboard |
| `vLLMDeploymentOnMaaS` | Adds the **vLLM runtime** option (Technology Preview) in the deploy wizard when deploying through the MaaS path. Without this, only llm-d is available |
| `genAiStudio` | Shows the **Gen AI studio** section — Playground (interactive chat), AI asset endpoints, API keys |
| `observabilityDashboard` | Shows the MaaS **observability dashboard** — token consumption metrics, request counts, rate-limit violations by user/subscription/model |
| `llmGatewayField` | Shows the LLM gateway configuration field in deployment settings |

## Model Deployment and Serving Flags (shown by default)

Set to `true` to **hide**.

| Flag | Default | Purpose |
|------|---------|---------|
| `disableKServe` | `false` | Hides KServe single-model serving platform |
| `disableKServeRaw` | `false` | Hides KServe RawDeployment mode specifically |
| `disableKServeAuth` | `false` | Hides KServe authentication configuration options |
| `disableKServeMetrics` | `false` | Hides KServe model performance metrics |
| `disableLLMd` | `false` | Hides **Distributed inference with llm-d** runtime option in the deploy wizard |
| `disableNIMModelServing` | `false` | Hides **NVIDIA NIM** model serving option |
| `disableModelMesh` | `false` | Hides legacy multi-model serving platform (ModelMesh) |
| `disableModelServing` | `false` | Hides model serving features entirely |
| `disableModelCatalog` | `false` | Hides the **Model catalog** (AI hub → Models) |
| `disableModelRegistry` | `false` | Hides model registry features |
| `disableModelRegistrySecureDB` | `false` | Hides secure database configuration for model registry |
| `disableServingRuntimeParams` | `false` | Hides serving runtime parameter configuration in deployment settings |
| `disableCustomServingRuntimes` | `false` | Hides custom serving runtime management |

## Development and Training Flags (shown by default)

Set to `true` to **hide**.

| Flag | Default | Purpose |
|------|---------|---------|
| `disablePipelines` | `false` | Hides AI pipelines (Kubeflow/Tekton) |
| `disableDistributedWorkloads` | `false` | Hides distributed workload features (Kueue/CodeFlare) |
| `disableFineTuning` | `false` | Hides fine-tuning and training job features |
| `disableLMEval` | `false` | Hides LM evaluation features |
| `disableFeatureStore` | `false` | Hides Feature Store section (Feast) |
| `disableKueue` | `false` | Hides Kueue workload management |

## Experimental / New Features (opt-in — hidden by default)

Set to `true` to enable. These are Technology Preview or unreleased features.

| Flag | Purpose |
|------|---------|
| `automl` | Enables AutoML features |
| `autorag` | Enables AutoRAG (Retrieval-Augmented Generation) features |
| `mcpCatalog` | Enables **MCP catalog** — browse and manage MCP servers in Gen AI studio |
| `promptManagement` | Enables prompt management features (undocumented in RHOAI 3.4 — likely an upcoming feature for saving/organizing/sharing prompt templates across the Playground) |
| `trainingJobs` | Enables training job UI |
| `mlflow` | Enables MLflow experiment tracking integration |
| `deploymentWizardYAMLViewer` | Enables **Form/YAML toggle** in the deploy wizard with real-time LLMInferenceService YAML preview and manual editing |
| `aiAssetCustomEndpoints` | Enables **custom AI asset endpoints** — see [detailed explanation](#aiassetcustomendpoints-details) below |
| `externalVectorStores` | Enables external vector store configuration for RAG |
| `projectRBAC` | Enables **project-level RBAC** — see [detailed explanation](#projectrbac-details) below |

### `aiAssetCustomEndpoints` Details

**Source**: [RHOAI 3.4 — Experimenting with models in the gen AI playground](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html-single/experimenting_with_models_in_the_gen_ai_playground/index) (Technology Preview)

When `aiAssetCustomEndpoints: true`, users can create custom model endpoints in **Gen AI studio → AI asset endpoints** from models deployed **in another namespace on the same cluster** (not just the current project). This is useful for multi-tenant scenarios where shared models live in one namespace but need to be accessible from other projects.

There is a second-level configuration for external providers:

```yaml
spec:
  dashboardConfig:
    aiAssetCustomEndpoints: true
  genAiStudioConfig:
    aiAssetCustomEndpoints:
      externalProviders: true
      clusterDomains:
        - ".my-internal-domain.com"
```

| Setting | Scope |
|---------|-------|
| `aiAssetCustomEndpoints: true` | Cross-namespace endpoints (same cluster only) |
| `+ externalProviders: true` | External third-party providers (OpenAI, Anthropic, AWS) — **data leaves the cluster** |
| `clusterDomains` | Additional domains treated as internal (`.svc.cluster.local` is always internal) |

**Security warning**: Enabling `externalProviders` allows data from the responses API — including RAG context, MCP tool results, and user input — to be sent to endpoints outside the cluster. Evaluate your organization's data security requirements before enabling.

**Prerequisites**: Requires `genAiStudio: true` and `llamastackoperator: Managed` in the DSC.

### `projectRBAC` Details

**Source**: [RHOAI 3.4 Release Notes](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/pdf/release_notes/Red_Hat_OpenShift_AI_Self-Managed-3.4-Release_notes-en-US.pdf)

When `projectRBAC: true`, a **Permissions tab** appears in each Data Science Project. Administrators can:

- Create custom Kubernetes Role objects with the `opendatahub.io/dashboard: 'true'` label
- Assign those roles to users and groups from the dashboard UI
- Assign multiple custom roles simultaneously
- Roles labeled for the dashboard are displayed with an "AI role" label in the Role type column

Without this flag, project permissions fall back to default OpenShift namespace-level RBAC (anyone with namespace access can do everything). With it, you get fine-grained, dashboard-managed role assignments per project — essential for multi-tenant production environments where different teams manage different projects.

**Example roles**: model deployer (can deploy/stop models), model viewer (read-only access to deployments), workbench admin (can create/manage workbenches).

## Platform and Admin Flags (shown by default)

Set to `true` to **hide**.

| Flag | Default | Purpose |
|------|---------|---------|
| `disableHome` | `false` | Hides the Home page |
| `disableProjects` | `false` | Hides Projects section |
| `disableProjectSharing` | `false` | Hides project sharing features |
| `disableProjectScoped` | `false` | Hides project-scoped settings |
| `disableHardwareProfiles` | `false` | Hides hardware profile management (Settings → Hardware profiles) |
| `disableAcceleratorProfiles` | `false` | Hides accelerator profile management |
| `disableStorageClasses` | `false` | Hides storage class configuration |
| `disableBYONImageStream` | `false` | Hides custom notebook image management (Bring Your Own Notebook) |
| `disableUserManagement` | `false` | Hides user management settings |
| `disableClusterManager` | `false` | Hides cluster manager features |
| `disableAdminConnectionTypes` | `false` | Hides admin connection type management |
| `disablePerformanceMetrics` | `false` | Hides performance metrics |
| `disableTracking` | `false` | Disables telemetry/usage tracking |
| `disableSupport` | `false` | Hides support links |
| `disableInfo` | `false` | Hides info/about section |
| `disableISVBadges` | `false` | Hides ISV (Independent Software Vendor) badges on components |
| `enablement` | — | Controls general feature enablement |
| `disableTrustyBiasMetrics` | `false` | Hides TrustyAI bias metrics |

---

## Example: MaaS + GenAI Studio Configuration

The patch used in [model-car-and-maas.md](model-car-and-maas.md) Step 2:

```bash
oc patch odhdashboardconfig odh-dashboard-config \
  -n redhat-ods-applications \
  --type merge \
  -p '{
    "spec": {
      "dashboardConfig": {
        "modelAsService": true,
        "genAiStudio": true,
        "vLLMDeploymentOnMaaS": true,
        "maasAuthPolicies": true
      }
    }
  }'
```

This enables:
- Models-as-a-Service UI (subscriptions, auth policies)
- Gen AI studio (Playground, AI asset endpoints, API keys)
- vLLM runtime option in MaaS deploy wizard
- Authorization policy management page

## Source

Validated against the live `OdhDashboardConfig` CRD (`oc explain odhdashboardconfig.spec.dashboardConfig`) on OCP 4.19.17 with RHOAI 3.4.0. The published RHOAI 3.4 docs lag the shipped CRD schema — the GenAI/MaaS flags exist in the CRD ahead of the docs being updated.
