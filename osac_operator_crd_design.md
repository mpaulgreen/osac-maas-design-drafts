# ModelServiceOrder CRD — Detailed Operator Design

*Comprehensive reference for the `ModelServiceOrder` CRD, dual-controller architecture, and provisioning lifecycle in the OSAC Operator.[Feedback Reconciler design is aspirational at this point, till software stack using APP provisioning is completed]*

## 1. Overview

### What Is It?

`ModelServiceOrder` is a **Kubernetes Custom Resource** on the OSAC hub cluster that represents a request to provision a MaaS tenant entitlement. It is the operator-side counterpart to the `ModelServices` fulfillment API resource — the CRD that the OSAC Operator watches, processes, and uses to trigger AAP provisioning.

### Where It Sits in the Gap II Path

```
Cloud Provider API
    │ POST /api/fulfillment/v1/model_services
    ▼
Fulfillment Service (PostgreSQL)
    │ Reconciler watches DB records
    ▼
ModelServiceOrder CRD (Hub Cluster)    ← THIS DOCUMENT
    │ Operator watches CRDs
    ▼
AAP / Ansible (Provisioning)
    │ Creates AITenant, Keycloak realm, MaaS CRs
    ▼
MaaS Cluster (Tenant Ready)
```

### Who Creates It?

The `ModelServiceOrder` is **NOT created by a human operator**. It is created programmatically by the fulfillment service reconciler (`internal/controllers/modelservice/model_service_reconciler_function.go`) when a `ModelService` record appears in the fulfillment database. The fulfillment reconciler translates the API-layer `ModelServiceSpec` into the CRD-layer `ModelServiceOrderSpec` and creates the CR on the hub cluster.

### Who Processes It?

The OSAC Operator runs two controllers per resource (the dual-controller pattern):

| Controller | Purpose | Finalizer |
|-----------|---------|-----------|
| **Resource controller** | Provisions/deprovisions via AAP | `osac.openshift.io/modelserviceorder` |
| **Feedback controller** | Syncs CRD status back to fulfillment service via gRPC | `osac.openshift.io/modelserviceorder-feedback` |

---

## 2. CRD Metadata

| Attribute | Value |
|-----------|-------|
| API Group | `osac.openshift.io` |
| Version | `v1alpha1` |
| Kind | `ModelServiceOrder` |
| Plural | `modelserviceorders` |
| Short Name | `mso` |
| Scope | Namespaced |
| Subresource | `status` |
| Default Namespace | `osac-model-service` (configurable via `OSAC_MODEL_SERVICE_ORDER_NAMESPACE`) |

### Print Columns (`oc get mso`)

| Column | JSONPath | Type |
|--------|----------|------|
| Template | `.spec.templateID` | string |
| Organization | `.spec.organizationRef` | string |
| Phase | `.status.phase` | string |

### Kubebuilder Markers

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:shortName=mso
// +kubebuilder:printcolumn:name="Template",type=string,JSONPath=`.spec.templateID`
// +kubebuilder:printcolumn:name="Organization",type=string,JSONPath=`.spec.organizationRef`
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
```

### Labels

| Label | Purpose | Set By |
|-------|---------|--------|
| `osac.openshift.io/modelserviceorder-uuid` | Links CRD to fulfillment service DB record by ModelService ID | Fulfillment reconciler |

### Annotations

| Annotation | Purpose | Set By |
|-----------|---------|--------|
| `osac.openshift.io/tenant` | OSAC Organization ID (redundant with `spec.organizationRef`, used for quick filtering) | Fulfillment reconciler |
| `osac.openshift.io/management-state` | If set to `unmanaged`, controller skips this resource | Human operator (manual override) |

### Finalizers

| Finalizer | Owner | Purpose |
|-----------|-------|---------|
| `osac.openshift.io/modelserviceorder` | Resource controller | Ensures AAP deprovisioning completes before CRD deletion |
| `osac.openshift.io/modelserviceorder-feedback` | Feedback controller | Ensures deletion state is synced to fulfillment service before CRD deletion |

---

## 3. Spec Fields (Field-by-Field)

All spec fields are **set by the fulfillment service reconciler**, not by a human. Field names must match what the reconciler creates (see `buildSpec()` in the reconciler).

### `templateID` (required)

| Attribute | Value |
|-----------|-------|
| JSON | `templateID` |
| Go type | `string` |
| Validation | Required, Pattern: `^[a-zA-Z_][a-zA-Z0-9._]*$` |
| Source | `ModelServiceSpec.template` (fulfillment API) |

Determines the AAP template role to execute. The operator auto-derives the AAP job template name as `{prefix}-create-model-service-order` / `{prefix}-delete-model-service-order` where prefix defaults to `osac`.

| Value | Deployment Model |
|-------|-----------------|
| `maas_shared` | Option 2 — namespace-per-tenant on shared cluster |
| `maas_dedicated` | Option 3 — dedicated cluster per tenant |

### `organizationRef` (required)

| Attribute | Value |
|-----------|-------|
| JSON | `organizationRef` |
| Go type | `string` |
| Validation | Required |
| Source | `ModelService.metadata.tenant` (fulfillment API, auto-assigned from auth) |

The OSAC Organization ID that owns this entitlement. Used by AAP to look up org configuration (users, groups, IdP settings) and scope all provisioned resources.

### `gatewayType` (optional)

| Attribute | Value |
|-----------|-------|
| JSON | `gatewayType` |
| Go type | `string` |
| Validation | Optional |
| Source | `ModelServiceSpec.gateway.type` → mapped by reconciler |
| Values | `"shared"`, `"per-tenant"`, or empty |

Mapped from the proto enum by the fulfillment reconciler:
- `MODEL_SERVICE_GATEWAY_TYPE_SHARED` → `"shared"`
- `MODEL_SERVICE_GATEWAY_TYPE_PER_TENANT` → `"per-tenant"`
- `MODEL_SERVICE_GATEWAY_TYPE_UNSPECIFIED` → empty (omitted)

> **Current scope:** Gateway provisioning is out of scope for the OSAC fulfillment layer. This field is retained for future extensibility.

### `modelCatalog` (optional)

| Attribute | Value |
|-----------|-------|
| JSON | `modelCatalog` |
| Go type | `[]string` |
| Validation | Optional |
| Source | `ModelServiceSpec.model_catalog` (fulfillment API) |

List of model identifiers to make available to the tenant at onboarding. The AAP template uses this to configure `MaaSModelRef` tenant bindings on shared models. Can be empty — tenant admin adds models post-provisioning.

### `identityType` (required)

| Attribute | Value |
|-----------|-------|
| JSON | `identityType` |
| Go type | `string` |
| Validation | Required |
| Source | `ModelServiceSpec.identity.type` → mapped by reconciler |
| Values | `"managed"`, `"byoidp"` |

Mapped from the proto enum:
- `MODEL_SERVICE_IDENTITY_TYPE_MANAGED` → `"managed"` (platform provisions Keycloak realm)
- `MODEL_SERVICE_IDENTITY_TYPE_BYOIDP` → `"byoidp"` (tenant provides OIDC issuer)

### `oidcIssuer` (optional)

| Attribute | Value |
|-----------|-------|
| JSON | `oidcIssuer` |
| Go type | `string` |
| Validation | Optional |
| Source | `ModelServiceSpec.identity.oidc_issuer` (fulfillment API) |
| Relevant when | `identityType = "byoidp"` |

The tenant's OIDC issuer URL. JWKS endpoint is auto-discovered via the OIDC standard (`{issuerUrl}/.well-known/openid-configuration` → `jwks_uri`). Passed to the MaaS Tenant CR's `spec.externalOIDC.issuerUrl`.

---

## 4. Status Fields (Field-by-Field)

All status fields are **set by the system** (operator controllers or AAP callbacks). The status subresource is enabled, meaning spec and status are updated independently.

### `phase`

| Attribute | Value |
|-----------|-------|
| JSON | `phase` |
| Go type | `ModelServiceOrderPhaseType` (string enum) |
| Validation | Enum: Progressing, Ready, Failed, Deleting, Suspended |
| Set by | Resource controller |

The single headline answer to "what's the state of this order?"

| Phase | Meaning | Set When |
|-------|---------|----------|
| `Progressing` | AAP provisioning in progress | First reconcile, or spec changed while Ready |
| `Ready` | Provisioning succeeded, tenant can use MaaS | AAP job succeeds |
| `Failed` | Provisioning failed | AAP job fails |
| `Deleting` | Deprovisioning in progress | Deletion timestamp set |
| `Suspended` | Tenant access frozen | Organization suspended (future) |

### `conditions`

| Attribute | Value |
|-----------|-------|
| JSON | `conditions` |
| Go type | `[]metav1.Condition` |
| Merge strategy | `patchStrategy:"merge" patchMergeKey:"type"` |
| Set by | Resource controller |

Standard Kubernetes conditions with `type`, `status` (True/False/Unknown), `lastTransitionTime`, `reason`, and `message`. Condition types defined:

| Type | Meaning |
|------|---------|
| `Progressing` | Whether provisioning is actively running |
| `Ready` | Whether the tenant entitlement is operational |
| `Failed` | Whether an irrecoverable error occurred |

### `jobs`

| Attribute | Value |
|-----------|-------|
| JSON | `jobs` |
| Go type | `[]JobStatus` |
| Set by | Provisioning lifecycle |

Tracks AAP job history for provisioning and deprovisioning. Each entry contains:

| Field | Type | Purpose |
|-------|------|---------|
| `jobID` | string (required) | AAP job template launch ID |
| `type` | JobType (required) | `provision` or `deprovision` |
| `timestamp` | string (required, date-time) | When the job was triggered |
| `state` | JobState (required) | Current state of the job |
| `message` | string | Human-readable status from AAP |
| `blockDeletionOnFailure` | bool | Prevent CR deletion on deprovision failure |
| `configVersion` | string | Spec hash at time of trigger |

**Job states:**

| State | Terminal? | Meaning |
|-------|----------|---------|
| `Pending` | No | Job submitted, not yet started |
| `Waiting` | No | Waiting for resources |
| `Running` | No | Actively executing |
| `Succeeded` | Yes | Completed successfully |
| `Failed` | Yes | Completed with error |
| `Canceled` | Yes | Cancelled before completion |
| `Unknown` | No | State cannot be determined |

The `maxJobHistory` setting (default from provisioning package) limits the jobs array size. Oldest entries are pruned.

### `desiredConfigVersion`

| Attribute | Value |
|-----------|-------|
| JSON | `desiredConfigVersion` |
| Go type | `string` |
| Set by | Resource controller |

SHA hash of the spec. Enables **idempotent re-provisioning**: if the spec changes, the hash changes, triggering a new provisioning job. If the spec reverts to a previously-provisioned state, the matching hash prevents duplicate work.

### `gatewayUrl`

| Attribute | Value |
|-----------|-------|
| JSON | `gatewayUrl` |
| Go type | `string` |
| Set by | AAP (via status update) |
| Populated when | Phase = Ready |

The inference gateway endpoint URL for this tenant. Synced back to the fulfillment service's `ModelServiceStatus.gateway_url` by the feedback controller.

### `apiKeyEndpoint`

| Attribute | Value |
|-----------|-------|
| JSON | `apiKeyEndpoint` |
| Go type | `string` |
| Set by | AAP (via status update) |
| Populated when | Phase = Ready |

The maas-api endpoint URL where the tenant mints API keys. Synced back to `ModelServiceStatus.api_key_endpoint`.

### `tenantNamespace`

| Attribute | Value |
|-----------|-------|
| JSON | `tenantNamespace` |
| Go type | `string` |
| Set by | AAP (via status update) |
| Populated when | Phase = Ready |

The Kubernetes namespace created on the MaaS cluster for this tenant (e.g., `acme-maas`). Not exposed in the fulfillment API — internal tracking only.

---

## 5. Phase Lifecycle

```
                  create (reconciler)
  (none) ─────────────────────────► Progressing
                                        │
                          AAP succeeds  ▼
                                      Ready ◄─────── Reactivate (future)
                                        │                 ▲
                            spec change ▼                 │
                                   Progressing            │
                                        │                 │
                            suspend     ▼                 │
                                    Suspended ────────────┘
                                        │
                          AAP fails     ▼
                                      Failed
                                        │
                          delete        ▼
                                    Deleting ──────► (deleted)
```

| Transition | Trigger | Controller Action |
|-----------|---------|-------------------|
| → Progressing | CRD created by fulfillment reconciler | Resource controller sets initial phase, triggers AAP |
| Progressing → Ready | AAP provisioning job succeeds | `OnSuccess` callback sets phase |
| Progressing → Failed | AAP provisioning job fails | `OnFailed` callback sets phase |
| Ready → Progressing | Spec changed (new `desiredConfigVersion`) | Controller detects hash mismatch, reverts phase |
| Ready → Suspended | Organization suspended | Future: org lifecycle coupling |
| Suspended → Ready | Organization reactivated | Future: org lifecycle coupling |
| Any → Deleting | `DeletionTimestamp` set (CRD deleted) | Controller sets phase, triggers AAP deprovisioning |
| Deleting → (deleted) | AAP deprovisioning completes | Controller removes finalizer, K8s garbage-collects |

---

## 6. Dual-Controller Architecture

```
                ModelServiceOrder CRD
                    ▲           │
                    │           │ watches
          ┌────────┘           ▼
          │            ┌──────────────────────┐
          │            │  Resource Controller  │
          │            │                      │
          │            │  • Adds finalizer    │
          │            │  • Sets phase        │
          │            │  • Computes config   │
          │            │    version           │
          │            │  • Triggers AAP      │──────► AAP REST API
          │            │  • Polls job status  │
          │            │  • Handles deletion  │
          │            └──────────────────────┘
          │
          │            ┌──────────────────────┐
          │            │  Feedback Controller  │
          │            │                      │
          └────────────│  • Watches phase     │
                       │  • Maps phase →      │
                       │    proto state       │──────► Fulfillment Service
                       │  • Syncs conditions  │        (gRPC private API)
                       │  • Syncs endpoints   │
                       │  • Handles deletion  │
                       └──────────────────────┘
```

**Why two controllers?**

1. **Separation of concerns**: Resource controller owns infrastructure lifecycle (AAP). Feedback controller owns fulfillment service sync (gRPC). Each has its own finalizer.
2. **Resilience**: Operator works even if the fulfillment service is down — AAP provisioning proceeds, feedback syncs later.
3. **Independent cleanup**: On deletion, resource controller ensures AAP deprovisioning completes; feedback controller ensures fulfillment service is notified. Neither blocks the other.

---

## 7. Resource Controller

**File**: `internal/controller/modelserviceorder_controller.go`

### Reconciler Structure

```go
type ModelServiceOrderReconciler struct {
    client.Client
    APIReader                  client.Reader
    Scheme                     *runtime.Scheme
    ModelServiceOrderNamespace string
    ProvisioningProvider       provisioning.ProvisioningProvider
    StatusPollInterval         time.Duration
    MaxJobHistory              int
}
```

### RBAC

```go
//+kubebuilder:rbac:groups=osac.openshift.io,resources=modelserviceorders,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=osac.openshift.io,resources=modelserviceorders/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=osac.openshift.io,resources=modelserviceorders/finalizers,verbs=update
```

### Reconciliation Flow

```
Reconcile(ctx, request)
  │
  ├── 1. Fetch ModelServiceOrder CR
  │     └── NotFound? → return (deleted before reconcile)
  │
  ├── 2. Check management-state annotation
  │     └── "unmanaged"? → skip silently
  │
  ├── 3. Save old status (for change detection)
  │
  ├── 4. Branch on DeletionTimestamp
  │     ├── Not set → handleUpdate()
  │     └── Set → handleDelete()
  │
  └── 5. Compare old vs new status
        └── Changed? → Status().Update() with retry
```

### handleUpdate Flow

```
handleUpdate()
  │
  ├── 1. Add finalizer (if not present)
  │     └── osac.openshift.io/modelserviceorder
  │
  ├── 2. Set initial phase
  │     └── If phase empty → Progressing
  │
  ├── 3. Compute desiredConfigVersion
  │     └── SHA hash of spec
  │
  ├── 4. Check for spec change while Ready
  │     └── If Ready AND hash changed → revert to Progressing
  │
  └── 5. handleProvisioning()
        │
        ├── No ProvisioningProvider? → skip (logged)
        │
        └── RunProvisioningLifecycle()
              ├── Evaluate action: Skip / Trigger / Requeue / Backoff / Poll
              ├── Trigger → provider.TriggerProvision() → AAP REST API
              │     └── POST /v2/job_templates/{name}/launch/
              │           extra_vars: CRD spec fields
              ├── Poll → provider.GetProvisionStatus() → AAP REST API
              │     └── GET /v2/jobs/{id}/
              ├── OnSuccess → phase = Ready
              └── OnFailed → phase = Failed
```

### handleDelete Flow

```
handleDelete()
  │
  ├── 1. Set phase → Deleting
  │
  ├── 2. Check finalizer present
  │     └── Not present? → return (already cleaned up)
  │
  ├── 3. handleDeprovisioning()
  │     └── RunDeprovisioningLifecycle()
  │           ├── Wait for active provision job to terminate
  │           ├── Trigger → provider.TriggerDeprovision() → AAP REST API
  │           │     └── POST /v2/job_templates/{name}/launch/ (delete template)
  │           └── Poll until complete
  │
  └── 4. Remove finalizer
        └── osac.openshift.io/modelserviceorder
```

### AAP Template Naming

The operator derives AAP job template names using the `{prefix}-{action}-{resource-kind-kebab}` convention:

| Action | Template Name |
|--------|--------------|
| Provision | `osac-create-model-service-order` |
| Deprovision | `osac-delete-model-service-order` |

The prefix is configurable via `OSAC_AAP_TEMPLATE_PREFIX` (default: `osac`).

### Config Version for Idempotency

The `desiredConfigVersion` is a SHA hash of the spec. The provisioning lifecycle uses it to:
1. Detect whether the current spec has been provisioned (`configVersion` on the last successful job matches `desiredConfigVersion`)
2. Skip provisioning if already applied (idempotent)
3. Re-trigger provisioning if spec changed (hash mismatch even when phase is Ready)

### Status Update with Retry

```go
func (r *ModelServiceOrderReconciler) updateStatusWithRetry(ctx context.Context, mso *v1alpha1.ModelServiceOrder) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        return r.Status().Update(ctx, mso)
    })
}
```

Uses optimistic concurrency — retries on conflict (another writer updated the resource version).

---

## 8. Feedback Controller

**File**: `internal/controller/modelserviceorder_feedback_controller.go`

### Current Status: Scaffolded, gRPC Integration TODO

The feedback controller is structurally complete but awaits the fulfillment service private API publication to the Buf Schema Registry. The gRPC client stub is marked with TODO comments.

### Reconciler Structure

```go
type ModelServiceOrderFeedbackReconciler struct {
    hubClient                   clnt.Client
    grpcConn                    *grpc.ClientConn
    modelServiceOrderNamespace  string
    // TODO: modelServicesClient privatev1.ModelServicesClient
}
```

### Reconciliation Flow (designed, partially implemented)

```
Reconcile(ctx, request)
  │
  ├── 1. Fetch ModelServiceOrder CR from hub cluster
  │
  ├── 2. Extract ModelService ID from label
  │     └── osac.openshift.io/modelserviceorder-uuid
  │
  ├── 3. Fetch ModelService record from fulfillment service (TODO)
  │     └── modelServicesClient.Get(ctx, id)
  │
  ├── 4. Map CRD phase → proto state
  │     ├── Progressing → MODEL_SERVICE_STATE_PROGRESSING
  │     ├── Ready       → MODEL_SERVICE_STATE_READY
  │     ├── Failed      → MODEL_SERVICE_STATE_FAILED
  │     └── Suspended   → MODEL_SERVICE_STATE_SUSPENDED
  │
  ├── 5. Sync additional status fields (TODO)
  │     ├── conditions → ModelServiceStatus.conditions
  │     ├── gatewayUrl → ModelServiceStatus.gateway_url
  │     └── apiKeyEndpoint → ModelServiceStatus.api_key_endpoint
  │
  ├── 6. Save if changed (TODO)
  │     └── modelServicesClient.Update(ctx, modelService)
  │
  └── 7. Handle deletion
        ├── Remove feedback finalizer
        └── Signal fulfillment service for re-reconciliation (TODO)
```

### Completion Requirements

1. Update `buf.gen.yaml` with fulfillment service private API module version
2. Run `buf generate` to generate `ModelServicesClient`
3. Implement steps 3-6 in the reconcile loop
4. Implement Signal RPC call on deletion

---

## 9. Registration & Configuration

**File**: `cmd/main.go`

### Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `OSAC_MODEL_SERVICE_ORDER_NAMESPACE` | `osac-model-service` | Namespace where ModelServiceOrder CRDs are created |
| `OSAC_ENABLE_MODEL_SERVICE_ORDER_CONTROLLER` | (none) | Set to `true` to enable only this controller |
| `OSAC_AAP_URL` | (required) | AAP server URL |
| `OSAC_AAP_TOKEN` | (required) | AAP authentication token |
| `OSAC_AAP_TEMPLATE_PREFIX` | `osac` | Prefix for AAP job template names |
| `OSAC_AAP_STATUS_POLL_INTERVAL` | `30s` | How often to poll AAP job status |
| `OSAC_AAP_INSECURE_SKIP_VERIFY` | `false` | Skip TLS verification for AAP |

### Controller Enable Flag

If **no** enable flags are set, **all** controllers run (including KubeVirt, HyperShift controllers that will fail without their CRDs). Set `OSAC_ENABLE_MODEL_SERVICE_ORDER_CONTROLLER=true` to run only this controller.

### Setup Function

```go
func setupModelServiceOrderControllers(mgr mcmanager.Manager, grpcConn *grpc.ClientConn, maxJobHistory int)
```

1. Creates AAP client with prefix-based provider
2. Sets up feedback controller (if gRPC connection available)
3. Sets up resource controller with AAP provider

---

## 10. Field Mapping: ModelService → ModelServiceOrder → AAP

```
Fulfillment API (ModelService)       CRD (ModelServiceOrder)          AAP extra_vars
─────────────────────────────        ─────────────────────────        ──────────────
spec.template                    →   spec.templateID              →   template_id
metadata.tenant                  →   spec.organizationRef         →   organization_ref
spec.gateway.type (enum→string)  →   spec.gatewayType             →   gateway_type
spec.model_catalog               →   spec.modelCatalog            →   model_catalog
spec.identity.type (enum→string) →   spec.identityType            →   identity_type
spec.identity.oidc_issuer        →   spec.oidcIssuer              →   oidc_issuer

metadata.labels (link)           →   labels["modelserviceorder-uuid"]
metadata.tenant (redundant)      →   annotations["tenant"]
```

The fulfillment reconciler (`buildSpec()` in `model_service_reconciler_function.go`) performs the enum-to-string mapping. The CRD stores simple strings, which AAP receives as `extra_vars` in the job template launch request.

---

## 11. Example CRs

### Shared Cluster, Managed Identity

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: ModelServiceOrder
metadata:
  name: acme-maas
  namespace: osac-model-service
  labels:
    osac.openshift.io/modelserviceorder-uuid: "019eb38f-3ca0-7145-bc81-64cae79573ec"
  annotations:
    osac.openshift.io/tenant: "shared"
spec:
  templateID: maas_shared
  organizationRef: org-acme-123
  gatewayType: shared
  modelCatalog:
  - llama3-8b
  - granite-3-8b
  identityType: managed
```

### Dedicated Cluster, BYOIDP

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: ModelServiceOrder
metadata:
  name: contoso-maas
  namespace: osac-model-service
  labels:
    osac.openshift.io/modelserviceorder-uuid: "019eb38f-4bc1-8256-cd92-75dbf80684fd"
spec:
  templateID: maas_dedicated
  organizationRef: org-contoso-456
  gatewayType: per-tenant
  identityType: byoidp
  oidcIssuer: https://login.contoso.com/realms/ai
```

### Status After Successful Provisioning

```yaml
status:
  phase: Ready
  conditions:
  - type: Progressing
    status: "False"
    lastTransitionTime: "2026-06-10T22:05:30Z"
  - type: Ready
    status: "True"
    lastTransitionTime: "2026-06-10T22:05:30Z"
  - type: Failed
    status: "False"
  jobs:
  - jobID: "12345"
    type: provision
    timestamp: "2026-06-10T22:02:50Z"
    state: Succeeded
    configVersion: "a1b2c3d4"
  desiredConfigVersion: "a1b2c3d4"
  gatewayUrl: "https://maas-gateway.apps.provider.cloud"
  apiKeyEndpoint: "https://maas-api.apps.provider.cloud/v1/api-keys"
  tenantNamespace: "acme-maas"
```

### Status When Blocked (No Hub Available)

This status comes from the **fulfillment reconciler** (not the operator), when no hub cluster is registered:

```yaml
status:
  phase: Progressing
  conditions:
  - type: Progressing
    status: "False"
    reason: ResourcesUnavailable
    message: "The model service cannot be provisioned because there are no resources available."
```

---

## 12. Files Reference

| File | Purpose |
|------|---------|
| `api/v1alpha1/modelserviceorder_types.go` | CRD type definitions (Spec, Status, Phase, Conditions) |
| `api/v1alpha1/groupversion_info.go` | Scheme registration (ModelServiceOrder + List) |
| `api/v1alpha1/job_types.go` | JobStatus and JobState types (shared across all CRDs) |
| `internal/controller/modelserviceorder_controller.go` | Resource controller (provisioning lifecycle) |
| `internal/controller/modelserviceorder_feedback_controller.go` | Feedback controller (gRPC sync, TODO) |
| `internal/controller/modelserviceorder_names.go` | Constants: labels, finalizers, default namespace |
| `cmd/main.go` | Controller registration, env vars, enable flag |
| `pkg/provisioning/provider.go` | ProvisioningProvider interface (shared) |
| `pkg/provisioning/provision_lifecycle.go` | Provisioning state machine (shared) |
| `config/crd/bases/osac.openshift.io_modelserviceorders.yaml` | Generated CRD manifest |
| `config/crd/kustomization.yaml` | CRD inclusion for `make install` |

---

## 13. Validation Results

Tested on live OpenShift 4.19.17 cluster (ca-central-1, g5.4xlarge workers).

| Level | What Was Tested | Result |
|-------|----------------|--------|
| **Level 1 (CRD Schema)** | CRD applied, `oc get mso` works, short name works, print columns show, schema validation rejects invalid values, BYOIDP variant accepted | Pass |
| **Level 2 (Controller)** | Operator deployed (`make deploy`), controller starts, finalizer added on first reconcile, phase set to Progressing, desiredConfigVersion computed, "No provisioning provider" logged (expected — no AAP), delete lifecycle works (phase→Deleting, finalizer removed) | Pass |
| **Level 3 (Full Stack)** | Fulfillment API → PostgreSQL → reconciler creates ModelServiceOrder on hub → operator processes CRD → finalizer + phase + config version set → delete propagates end-to-end | Pass |

### Deployment Commands

```shell
# Build and push
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o manager-linux ./cmd/main.go
podman build --no-cache --platform linux/amd64 -f Containerfile.design -t quay.io/mpaulgreen/osac-operator:design-v2 .
podman push quay.io/mpaulgreen/osac-operator:design-v2

# Deploy
make deploy IMG=quay.io/mpaulgreen/osac-operator:design-v2

# Enable only ModelServiceOrder controller
oc set env deployment/osac-operator-controller-manager -n osac-operator-system \
  OSAC_ENABLE_MODEL_SERVICE_ORDER_CONTROLLER=true

# Verify
oc rollout status deployment osac-operator-controller-manager -n osac-operator-system
oc get mso -A
```
