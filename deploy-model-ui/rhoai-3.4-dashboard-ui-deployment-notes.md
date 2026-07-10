# RHOAI 3.4 Dashboard UI — Model Deployment Investigation Notes

Working notes from an investigation into Red Hat OpenShift AI 3.4 model-deployment paths
that are driven from the OpenShift AI **dashboard UI** (as opposed to raw `oc apply`).
Context: validating a runbook that deploys Granite 3.1 8B (FP8) from a ModelCar OCI image
on a single A10G and publishes to Models-as-a-Service.

Three questions were investigated:

1. What other ways exist to deploy a model from the dashboard UI?
2. Can GPU hardware profiles be configured from the UI?
3. Are more `OdhDashboardConfig` customizations needed for different deployment kinds?

Sources: RHOAI 3.4 Self-Managed docs (Deploying models, Managing resources / Customizing
the dashboard, Managing OpenShift AI, 3.4 Release notes) plus a live `oc explain` of the
shipped `OdhDashboardConfig` CRD. Where the docs and the live CRD disagreed, the CRD won.

---

## Q1 — Other ways to deploy a model from the dashboard UI

There are two independent axes: **where the model comes from** and **how you enter the deploy flow**.

### Model source (Model location list in the Deploy a model wizard)

The wizard supports four sources, not just ModelCar:

| Source | Notes |
|---|---|
| S3-compatible object storage | Connection type; access key / secret / endpoint |
| URI-based repository | Covers HTTP/HTTPS **and** Hugging Face (`hf://`) URIs |
| OCI-compliant registry | ModelCar; needs image pull secret for private registries |
| PVC / Cluster storage | Only appears when a PVC is attached to a workbench; select PVC + path |

- **Hugging Face is not a separate tile** — it rides on the URI connection type as an `hf://` URI.
- **PVC ("Cluster storage")** is conditional: the option only surfaces in the Model location
  list when the PVC is attached to a workbench in the project.
- For public OCI registries, `URI - v1` is the simplest path (no auth config);
  use `OCI compliant registry - v1` when you need private-registry credentials.

### Entry points into deployment (beyond the project Deploy model wizard)

- **Deploy from the model catalog card** — deploy directly from a model card; the wizard then
  configures serving runtime and hardware profile.
- **Register-and-convert-to-ModelCar job (new in 3.4)** — register a model from S3 or URI,
  transform it into an OCI ModelCar image, and store it in an OCI registry. Runs as a
  background Kubernetes Job tracked in the dashboard (retry/delete, auto-GC of ConfigMaps/Secrets).
- **MaaS via vLLM (Tech Preview)** — deploy models using the vLLM runtime through Models-as-a-Service.

### Runtime / platform choices (the "kinds" selectable in the same wizard)

vLLM (NVIDIA CUDA and others), OpenVINO Model Server, **MLServer (now GA** — for classical/structured ML),
NVIDIA NIM, the Spyre runtimes, and **Distributed Inference with llm-d** (LLMInferenceService).

UX notes for 3.4:
- Selecting the **llm-d** runtime hides the deployment-strategy options.
- The wizard adds a **Form/YAML toggle** (`deploymentWizardYAMLViewer`) with real-time
  LLMInferenceService YAML preview and manual editing.

---

## Q2 — Configuring GPU hardware profiles in the UI

**Yes — this is a first-class UI flow; the `oc apply` in the runbook is optional.**

- Location: **Settings → Hardware profiles**.
- Enable/disable via the toggle in the **Enabled** column; **Edit** via the action menu.
- Creation is form-driven via the **Add resource** dialog: resource label, resource identifier,
  resource type, and default / minimum / maximum request limits.
- Scope is controlled with **Visible everywhere** vs **Limited visibility** radio buttons.
- For a GPU profile, add an identifier such as `nvidia.com/gpu` with default/min/max counts
  (mirrors the runbook YAML).
- Profiles are selectable in the UI for workbenches, model serving, and AI pipelines.

### Caveats found

- **`apiVersion` discrepancy in the runbook.** The runbook YAML uses
  `dashboard.opendatahub.io/v1alpha1`, but the 3.x CRD is `infrastructure.opendatahub.io/v1`.
  Validate against the live cluster CRD before relying on the YAML; if created in the UI this is moot.
- **UI doesn't cover every accelerator binding.** For Spyre schedulers, you must manually edit
  the InferenceService YAML after deployment to add the scheduler name and tolerations — the UI
  does not expose that option. So "configurable in UI" is true for the profile itself but not
  for every scheduler binding.

---

## Q3 — `OdhDashboardConfig` customization per deployment kind

Confirmed against the live CRD (`oc explain odhdashboardconfig.spec.dashboardConfig`).
The published 3.4 docs lag the shipped schema — the GenAI/MaaS flags exist in the CRD ahead
of the docs being updated. The runbook's Step 2 patch is therefore **honored, not ignored**.

### Per-deployment-kind gates exist and are granular

| Flag | Gates |
|---|---|
| `disableKServe` | KServe platform availability |
| `disableKServeRaw` | RawDeployment mode specifically |
| `disableLLMd` | Distributed Inference with llm-d |
| `disableNIMModelServing` | NVIDIA NIM |
| `disableModelMesh` | Legacy multi-model serving |
| `disableModelServing` | Model serving overall |
| `modelAsService` | MaaS UI |
| `vLLMDeploymentOnMaaS` | vLLM-on-MaaS deploy path |
| `maasAuthPolicies` | MaaS auth policies UI |
| `genAiStudio` | GenAI Studio |

### Polarity is the thing to get right

The schema mixes two conventions:

- **`disable*` (negative) flags** — default to `false` when omitted (feature **shown** unless hidden).
  Examples: `disableKServe`, `disableKServeRaw`, `disableLLMd`, `disableNIMModelServing`,
  `disableModelMesh`, `disableModelServing`.
- **Plain positive (enable) flags** — given the CRD note that the object is "intended to just
  contain overrides," these almost certainly default to `false` (feature **hidden**) when absent,
  so they are **opt-in**. Examples: `modelAsService`, `genAiStudio`, `vLLMDeploymentOnMaaS`,
  `maasAuthPolicies`, `automl`, `autorag`, `mcpCatalog`, `promptManagement`, `trainingJobs`,
  `deploymentWizardYAMLViewer`, `llmGatewayField`, `observabilityDashboard`.

This is why the runbook's Step 2 must set the four MaaS/GenAI flags to `true` — they are
opt-in, not on-by-default. The patch is **necessary**, not redundant.

> Confirm actual defaulting rather than trusting convention — `oc explain` gives field names
> and types but not per-field defaults.

### Flag layer vs component layer (the still-valid nuance)

Dashboard flags govern **UI visibility only**. The backend capability is gated separately by
DataScienceCluster (DSC) component `managementState`:

- `modelAsService: true` shows the MaaS UI, **but** the backend needs
  `spec.components.kserve.modelsAsService.managementState: Managed` in the DSC.
- `disableLLMd: false` shows the llm-d runtime, **but** the deploy still needs the Gateway,
  Red Hat Connectivity Link (RHCL), and the LeaderWorkerSet operator wired up.
- `genAiStudio` / GenAI features pair with the `llamastackoperator` DSC component.

A mismatch (flag on, component `Removed`) yields a **visible-but-broken** path.

**Runbook recommendation:** add an explicit pairing check so each enabled UI flag has its
corresponding DSC component in `Managed`.

---

## Verification commands

```bash
# Full live schema of dashboard feature flags
oc explain odhdashboardconfig.spec.dashboardConfig --recursive

# Currently set values on this cluster
oc get odhdashboardconfig odh-dashboard-config -n redhat-ods-applications \
  -o yaml | yq '.spec.dashboardConfig'

# Confirm the HardwareProfile CRD apiVersion actually present
oc get crd hardwareprofiles.infrastructure.opendatahub.io -o jsonpath='{.spec.versions[*].name}'

# MaaS backend component state (must be Managed for the MaaS UI flag to be functional)
oc get dsc default-dsc \
  -o jsonpath='{.spec.components.kserve.modelsAsService.managementState}'
```

---

## Summary of corrections to the runbook

1. **Hardware profile `apiVersion`** — likely `infrastructure.opendatahub.io/v1`, not
   `dashboard.opendatahub.io/v1alpha1`. Verify on cluster.
2. **Step 2 dashboard flags** — valid and correctly polarized (opt-in positives set to `true`).
   No change needed; the earlier doubt was wrong.
3. **Add a flag↔DSC-component pairing check** so an enabled UI path can't point at a
   `Removed` backend component.

> Note: 3.4 docs were being actively revised at the time of this investigation; re-validate
> against the live cluster before publishing anything customer-facing.
