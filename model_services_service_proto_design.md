# ModelServices API — Detailed Proto Design (Very Early Draft)

*Comprehensive reference for every field, enum, and design decision in the ModelServices fulfillment API resource.*

## 1. Overview

### What Is It?

`ModelServices` is a **new resource type in the OSAC Fulfillment API** that provisions a MaaS (Models as a Service) tenant entitlement for an OSAC Organization. It is the fulfillment-layer primitive for Gap II — the "missing middle" between OSAC (infrastructure tenancy) and MaaS (model governance).

### Why Does It Exist?

Before `ModelServices`, OSAC could provision clusters and VMs for organizations, but had no concept of provisioning AI model access. MaaS could create subscriptions and rate limits, but had no connection to OSAC Organizations. `ModelServices` connects them: a cloud provider creates a `ModelService` for an organization, and the system provisions the tenant's MaaS entitlement (namespace, identity, gateway, model access).

### Who Interacts With It?

| Actor | Interaction |
|-------|-------------|
| **Cloud Provider Admin** | Creates/updates/deletes ModelServices via the Fulfillment API |
| **System (Reconciler)** | Watches DB records, creates ModelServiceOrder CRDs on hub cluster |
| **System (Operator)** | Watches CRDs, triggers AAP provisioning, updates status |
| **Tenant Admin** | Does NOT interact directly — manages models/quotas via MaaS CRs after provisioning |

---

## 2. API Surface

### gRPC Service

```protobuf
service ModelServices {
  rpc List(ModelServicesListRequest) returns (ModelServicesListResponse)
  rpc Get(ModelServicesGetRequest) returns (ModelServicesGetResponse)
  rpc Create(ModelServicesCreateRequest) returns (ModelServicesCreateResponse)
  rpc Update(ModelServicesUpdateRequest) returns (ModelServicesUpdateResponse)
  rpc Delete(ModelServicesDeleteRequest) returns (ModelServicesDeleteResponse)
}
```

### HTTP Mapping (via grpc-gateway)

| Operation | HTTP | Path | Body |
|-----------|------|------|------|
| List | `GET` | `/api/fulfillment/v1/model_services` | — |
| Get | `GET` | `/api/fulfillment/v1/model_services/{id}` | — |
| Create | `POST` | `/api/fulfillment/v1/model_services` | `object` |
| Update | `PATCH` | `/api/fulfillment/v1/model_services/{object.id}` | `object` |
| Delete | `DELETE` | `/api/fulfillment/v1/model_services/{id}` | — |

### Filtering & Pagination (List) [Needs Validation]

```
GET /api/fulfillment/v1/model_services?offset=0&limit=10&filter=this.metadata.name=='acme-maas'&order=metadata.creation_timestamp DESC
```

- `offset` / `limit` — pagination
- `filter` — CEL expression (e.g., `this.metadata.labels['env'] == 'prod'`)
- `order` — SQL-like ordering

---

## 3. Resource Structure

```protobuf
message ModelService {
  string id = 1;              // System-generated UUID, immutable
  Metadata metadata = 2;      // Common fields shared across ALL OSAC resources
  ModelServiceSpec spec = 3;   // Desired state — what the cloud provider configures
  ModelServiceStatus status = 4; // Current state — what the system reports
}
```

### `id` (field 1)

| Attribute | Value |
|-----------|-------|
| Type | `string` (UUID) |
| Set by | System (auto-generated on create) |
| Mutable | No |
| Example | `019eb38f-3ca0-7145-bc81-64cae79573ec` |

The unique identifier for this ModelService. Used in all subsequent GET/PATCH/DELETE operations and as the linkage key to the `ModelServiceOrder` CRD via label `osac.openshift.io/modelserviceorder-uuid`.

### `metadata` (field 2)

Shared `Metadata` type used by every OSAC resource. See [metadata detail](#metadata-fields).

### `spec` (field 3)

The desired configuration. See [spec fields](#4-spec-fields-field-by-field).

### `status` (field 4)

The system-reported state. See [status fields](#5-status-fields-field-by-field).

---

## Metadata Fields

| Field | Type | Set By | Mutable | Purpose |
|-------|------|--------|---------|---------|
| `creation_timestamp` | Timestamp | System | No | When created |
| `deletion_timestamp` | Timestamp | System | No | When deletion requested |
| `creator` | string | System | No | Who created it (e.g., `service-account-osac-admin`) |
| `name` | string | Cloud Provider | No (immutable) | Human-friendly name (DNS label, 1-63 chars) |
| `tenant` | string | System | No (immutable) | Owning OSAC Organization ID |
| `labels` | map | Cloud Provider | Yes | Key-value pairs for filtering (indexed) |
| `annotations` | map | Cloud Provider | Yes | Arbitrary metadata |
| `version` | int32 | System | No | Auto-incremented on each change (optimistic locking) |

**Key behaviors:**
- `name` must follow DNS label rules (lowercase, alphanumeric + hyphens, 1-63 chars). NOT unique — multiple ModelServices can have the same name.
- `tenant` is automatically assigned from the authenticated caller's organization. The cloud provider doesn't set it explicitly.
- `version` enables optimistic locking: include it in PATCH to detect concurrent modifications.
- PATCH requires `name` to be included (immutable fields must be present unchanged, or the server returns "field is immutable" error).

---

## 4. Spec Fields (Field-by-Field)

### `template` (field 1)

| Attribute | Value |
|-----------|-------|
| Type | `string` |
| Required | Yes (on create) |
| Mutable | No (immutable after create) |
| Set by | Cloud Provider |
| Valid values | `"maas_shared"`, `"maas_dedicated"` |

**What it does:** Determines the **deployment model** — how the MaaS infrastructure is provisioned for this tenant.

| Value | Meaning | What happens |
|-------|---------|-------------|
| `maas_shared` | Option 2 — shared multi-tenant cluster | Tenant gets a namespace + AITenant CR on an existing shared MaaS cluster. No new infrastructure provisioned. |
| `maas_dedicated` | Option 3 — dedicated cluster per tenant | Tenant gets a new cluster with MaaS pre-installed via the `ocp_4_17_maas` AAP template. New infrastructure provisioned. |

**Why immutable:** Changing from shared to dedicated (or vice versa) would require fundamentally different infrastructure — it's a different AAP template, different provisioning flow, different lifecycle. Create a new ModelService instead.

**Relationship:** Maps to `ModelServiceOrderSpec.TemplateID` on the operator CRD. The AAP template name is auto-derived as `osac-create-{template-kebab}` (e.g., `osac-create-model-service-order`).

---

### `template_parameters` (field 2)

| Attribute | Value |
|-----------|-------|
| Type | `map<string, google.protobuf.Any>` |
| Required | No |
| Mutable | No (immutable after create) |
| Set by | Cloud Provider |

**What it does:** Passes template-specific configuration without changing the proto schema. A generic key-value bag where each value is self-describing (carries its own `@type`).

**When to use:** For parameters that are specific to a particular template and don't belong as top-level spec fields. Currently unused — all configuration is covered by the structured spec fields below.

**Example (JSON):**
```json
"template_parameters": {
  "shared_model_endpoint": {
    "@type": "type.googleapis.com/google.protobuf.StringValue",
    "value": "https://shared-maas.provider.cloud"
  }
}
```

**Why it exists:** Future extensibility. New templates may need parameters we can't anticipate today. The server stores and forwards them without needing to understand them — only the AAP template role interprets them.

---

### `gateway` (field 3)

| Attribute | Value |
|-----------|-------|
| Type | `ModelServiceGatewayConfig` (nested message) |
| Required | No (defaults to UNSPECIFIED) |
| Mutable | Yes |
| Set by | Cloud Provider |

**What it does:** Determines the **gateway topology** — whether this tenant shares a gateway with other tenants or gets a dedicated one.

```protobuf
message ModelServiceGatewayConfig {
  ModelServiceGatewayType type = 1;
}

enum ModelServiceGatewayType {
  MODEL_SERVICE_GATEWAY_TYPE_UNSPECIFIED = 0;  // System decides (defaults to SHARED)
  MODEL_SERVICE_GATEWAY_TYPE_SHARED = 1;       // Shared gateway for all tenants
  MODEL_SERVICE_GATEWAY_TYPE_PER_TENANT = 2;   // Dedicated gateway for this tenant
}
```

| Value | Meaning | Trade-off |
|-------|---------|-----------|
| `MODEL_SERVICE_GATEWAY_TYPE_UNSPECIFIED` | System decides (defaults to SHARED) | — |
| `MODEL_SERVICE_GATEWAY_TYPE_SHARED` | All tenants on one gateway | Lower cost, shared fault domain. Tenant discrimination via auth (JWT issuer). |
| `MODEL_SERVICE_GATEWAY_TYPE_PER_TENANT` | Dedicated gateway for this tenant | Higher isolation, separate fault domain, own hostname. Higher infra cost. |

**Independent of `template`:** Gateway type and deployment model are separate axes. A shared cluster can have per-tenant gateways, and a dedicated cluster could use a shared gateway entry point (though unlikely).

**Why mutable:** A tenant might start with a shared gateway and upgrade to per-tenant as their traffic/trust requirements grow.

**Relationship:** Maps to `ModelServiceOrderSpec.GatewayType` (as `"shared"` or `"per-tenant"` string). The AAP workflow creates a Gateway CR in step 2b if `PER_TENANT`.

> **Note on current scope:** RHAIRFE-1487 implements a **single shared gateway** for multi-tenant MaaS (confirmed in alignment notes, June 1). Meanwhile, the OSAC AI grid adopts a **one gateway per tenant** model as the default for dedicated clusters. Gateway provisioning is currently marked as **out of scope for OSAC fulfillment** in the OSAC AI grid — the gateway is provisioned as part of the MaaS cluster setup (by RHOAI), not by the ModelServices API. Given this, `MODEL_SERVICE_GATEWAY_TYPE_UNSPECIFIED` should default to no-op (the gateway is handled outside this API), and the `gateway` field is retained for **future extensibility** — if OSAC later needs to control gateway topology per tenant through the fulfillment layer, the API surface is already in place.

---

### `model_catalog` (field 4)

| Attribute | Value |
|-----------|-------|
| Type | `repeated string` |
| Required | No (can be empty) |
| Mutable | Yes |
| Set by | Cloud Provider |

**What it does:** Lists the **model identifiers** the cloud provider makes available to this tenant at onboarding time.

**Example:**
```json
"model_catalog": ["llama3-8b", "granite-3-8b"]
```

**What this is NOT:**
- NOT the only way to grant model access — the tenant admin can also create `MaaSAuthPolicy` + `MaaSSubscription` CRs to access shared models directly
- NOT a restriction — the tenant admin can deploy additional models in their namespace

**Can be empty:** Yes. If empty, the tenant is provisioned with infrastructure (namespace, identity, RBAC) but no initial model access. The tenant admin adds models themselves.

**Relationship:** Maps to `ModelServiceOrderSpec.ModelCatalog`. The AAP workflow uses this to configure `MaaSModelRef` tenant bindings on shared models.

---

### `identity` (field 5)

| Attribute | Value |
|-----------|-------|
| Type | `ModelServiceIdentityConfig` (nested message) |
| Required | Yes (`type` must be `MANAGED` or `BYOIDP`) |
| Mutable | No (changing IDP after provisioning is destructive) |
| Set by | Cloud Provider |

**What it does:** Configures how this tenant's users authenticate to MaaS.

```protobuf
message ModelServiceIdentityConfig {
  ModelServiceIdentityType type = 1;
  string oidc_issuer = 2;    // Required when type = BYOIDP
}

enum ModelServiceIdentityType {
  MODEL_SERVICE_IDENTITY_TYPE_MANAGED = 0;      // Platform manages a Keycloak realm
  MODEL_SERVICE_IDENTITY_TYPE_BYOIDP = 1;       // Tenant provides their own IDP
}
```

| Value | Meaning | What happens |
|-------|---------|-------------|
| `MODEL_SERVICE_IDENTITY_TYPE_MANAGED` | Platform provisions identity | AAP creates a Keycloak realm for this tenant, syncs users from OSAC org, configures OIDC client |
| `MODEL_SERVICE_IDENTITY_TYPE_BYOIDP` | Tenant brings their own IDP | AAP skips Keycloak realm creation; passes `oidc_issuer` directly to the Tenant CR |

**When `BYOIDP`:** `oidc_issuer` is required. Kuadrant's AuthPolicy auto-discovers the JWKS endpoint via OIDC discovery (`{issuerUrl}/.well-known/openid-configuration` → `jwks_uri`). Example:
```json
"identity": {
  "type": "MODEL_SERVICE_IDENTITY_TYPE_BYOIDP",
  "oidc_issuer": "https://login.contoso.com/realms/ai"
}
```

**When `MANAGED`:** Both fields are ignored — the platform generates the OIDC configuration from the Keycloak realm it creates.

**Why not mutable:** Changing from MANAGED to BYOIDP (or vice versa) would invalidate all existing API keys and break authentication for active users. Create a new ModelService instead.

**Relationship:** Maps to `ModelServiceOrderSpec.IdentityType`, `.OIDCIssuer`. The Tenant CR's `externalOIDC.issuerUrl` field is populated from `oidc_issuer`. JWKS discovery is automatic via the OIDC standard.

---

## 5. Status Fields (Field-by-Field)

All status fields are **read-only** — set by the system, not the cloud provider.

### `state` (field 1)

| Attribute | Value |
|-----------|-------|
| Type | `ModelServiceState` enum |
| Set by | System (reconciler + operator) |

The **single headline answer** to "what's the overall status?"

```protobuf
enum ModelServiceState {
  MODEL_SERVICE_STATE_UNSPECIFIED = 0;   // Protobuf default — never shown
  MODEL_SERVICE_STATE_PROGRESSING = 1;   // Being provisioned
  MODEL_SERVICE_STATE_READY = 2;         // Tenant can use it
  MODEL_SERVICE_STATE_SUSPENDED = 3;     // Access frozen (org suspended)
  MODEL_SERVICE_STATE_FAILED = 4;        // Provisioning failed
}
```

**Lifecycle:**
```
Create → PROGRESSING → READY ⇄ SUSPENDED
                    ↘ FAILED
Delete → (from any state) → removed
```

| Transition | Trigger |
|-----------|---------|
| → PROGRESSING | ModelService created; reconciler picks it up |
| PROGRESSING → READY | AAP provisioning succeeds (namespace, identity, gateway all created) |
| PROGRESSING → FAILED | AAP provisioning fails (check conditions for reason) |
| READY → SUSPENDED | Organization suspended (payment, compliance) |
| SUSPENDED → READY | Organization reactivated |
| Any → Deleted | Cloud provider calls DELETE |

### `conditions` (field 2)

| Attribute | Value |
|-----------|-------|
| Type | `repeated ModelServiceCondition` |
| Set by | System |

**The detailed breakdown** — multiple conditions coexist. Each has a type, status (TRUE/FALSE), and optional reason + message.

```protobuf
message ModelServiceCondition {
  ModelServiceConditionType type = 1;   // Which condition
  ConditionStatus status = 2;           // TRUE, FALSE, or UNSPECIFIED
  Timestamp last_transition_time = 3;   // When it last changed
  optional string reason = 4;           // Machine-readable code
  optional string message = 5;          // Human-readable detail
}

enum ConditionStatus {
  CONDITION_STATUS_UNSPECIFIED = 0;     // Unknown
  CONDITION_STATUS_TRUE = 1;           // Condition is met
  CONDITION_STATUS_FALSE = 2;          // Condition is not met
}
```

**Condition types and their meanings:**

| Type | When TRUE | When FALSE |
|------|-----------|------------|
| `PROGRESSING` | Actively provisioning | Not progressing (done, or blocked — check `reason`) |
| `READY` | Fully operational | Not ready |
| `FAILED` | Irrecoverable error | Not failed |
| `SUSPENDED` | Access frozen | Not suspended |

**State vs Conditions — why both?**

`State` answers: **"what's the headline?"** (one value, for dashboards and polling)

`Conditions` answer: **"why?"** (multiple simultaneous signals, with reasons, for debugging and automation)

Example — provisioning blocked because no hubs available:
```json
{
  "state": "MODEL_SERVICE_STATE_PROGRESSING",
  "conditions": [
    {"type": "PROGRESSING", "status": "FALSE", "reason": "ResourcesUnavailable", 
     "message": "The model service cannot be provisioned because there are no resources available."},
    {"type": "READY", "status": "FALSE"},
    {"type": "FAILED", "status": "FALSE"},
    {"type": "SUSPENDED", "status": "FALSE"}
  ]
}
```

State says "progressing." Condition PROGRESSING=FALSE with reason says "actually blocked, specifically because resources are unavailable."

### `gateway_url` (field 3)

| Attribute | Value |
|-----------|-------|
| Type | `string` |
| Set by | System |
| Populated when | `state = READY` |
| Example | `https://maas-gateway.apps.provider.cloud` |

The URL where the tenant sends inference requests. For shared gateways, all tenants use the same URL (discrimination via auth). For per-tenant gateways, each tenant gets a unique hostname.

**Empty when not ready.** Cloud provider polls until this is populated.

### `api_key_endpoint` (field 4)

| Attribute | Value |
|-----------|-------|
| Type | `string` |
| Set by | System |
| Populated when | `state = READY` |
| Example | `https://maas-api.apps.provider.cloud/v1/api-keys` |

The URL where the tenant admin mints API keys. The tenant authenticates with their OIDC token and receives long-lived API keys scoped to their identity.

### `hub` (field 5 — private API only)

| Attribute | Value |
|-----------|-------|
| Type | `string` |
| Set by | System (reconciler) |
| Visible in | Private API only (`/api/private/v1/model_services/`) |

The ID of the hub cluster where the `ModelServiceOrder` CRD was created. Assigned on first reconcile (random selection), sticky thereafter. Not exposed in the public API — it's an internal implementation detail.

---

## 6. Persona Mapping

### What each actor can do

| Field | Cloud Provider (Create) | Cloud Provider (Update) | Tenant Admin | System |
|-------|------------------------|------------------------|--------------|--------|
| `id` | — | — | — | Auto-generates |
| `metadata.name` | Sets | Cannot change | — | — |
| `metadata.tenant` | — | — | — | Auto-assigns from auth |
| `metadata.labels` | Sets | Updates | — | — |
| `spec.template` | Sets | Cannot change | — | — |
| `spec.gateway` | Sets | Updates | — | — |
| `spec.model_catalog` | Sets | Updates | — | — |
| `spec.identity` | Sets | Cannot change | — | — |
| `status.*` | — | — | — | Updates |

### Post-provisioning: what the tenant admin manages (via MaaS CRs, not this API)

| What | MaaS CR | Set by |
|------|---------|--------|
| Per-model rate limits | `MaaSSubscription` | Tenant Admin |
| Per-model access rules | `MaaSAuthPolicy` | Tenant Admin |
| Deploy tenant-specific models | `LLMInferenceService` | Tenant Admin |
| Access shared models | `MaaSModelRef` + `MaaSAuthPolicy` | Tenant Admin |
| Mint API keys | `POST /v1/api-keys` (maas-api) | Tenant Admin / End User |

---

## 7. Request/Response Examples

### Create (shared cluster, managed identity)

```
POST /api/fulfillment/v1/model_services
```

```json
{
  "metadata": {
    "name": "acme-maas"
  },
  "spec": {
    "template": "maas_shared",
    "gateway": {
      "type": "MODEL_SERVICE_GATEWAY_TYPE_SHARED"
    },
    "model_catalog": ["llama3-8b", "granite-3-8b"],
    "identity": {
      "type": "MODEL_SERVICE_IDENTITY_TYPE_MANAGED"
    }
  }
}
```

**Response (201):**
```json
{
  "id": "019eb38f-3ca0-7145-bc81-64cae79573ec",
  "metadata": {
    "creation_timestamp": "2026-06-10T22:02:49.376415Z",
    "creator": "service-account-osac-admin",
    "name": "acme-maas",
    "tenant": "shared",
    "version": 1
  },
  "spec": {
    "template": "maas_shared",
    "gateway": {"type": "MODEL_SERVICE_GATEWAY_TYPE_SHARED"},
    "model_catalog": ["llama3-8b", "granite-3-8b"],
    "identity": {"type": "MODEL_SERVICE_IDENTITY_TYPE_MANAGED"}
  }
}
```

### Create (dedicated cluster, BYOIDP)

```json
{
  "metadata": {"name": "contoso-maas"},
  "spec": {
    "template": "maas_dedicated",
    "gateway": {"type": "MODEL_SERVICE_GATEWAY_TYPE_PER_TENANT"},
    "identity": {
      "type": "MODEL_SERVICE_IDENTITY_TYPE_BYOIDP",
      "oidc_issuer": "https://login.contoso.com/realms/ai"
    }
  }
}
```

### Create (minimal — empty catalog)

```json
{
  "metadata": {"name": "startup-maas"},
  "spec": {
    "template": "maas_shared",
    "identity": {"type": "MODEL_SERVICE_IDENTITY_TYPE_MANAGED"}
  }
}
```

### Get (after provisioning — status populated)

```
GET /api/fulfillment/v1/model_services/019eb38f-3ca0-7145-bc81-64cae79573ec
```

```json
{
  "id": "019eb38f-3ca0-7145-bc81-64cae79573ec",
  "metadata": {
    "creation_timestamp": "2026-06-10T22:02:49.376415Z",
    "creator": "service-account-osac-admin",
    "name": "acme-maas",
    "tenant": "shared",
    "version": 3
  },
  "spec": {
    "template": "maas_shared",
    "gateway": {"type": "MODEL_SERVICE_GATEWAY_TYPE_SHARED"},
    "model_catalog": ["llama3-8b", "granite-3-8b"],
    "identity": {"type": "MODEL_SERVICE_IDENTITY_TYPE_MANAGED"}
  },
  "status": {
    "state": "MODEL_SERVICE_STATE_PROGRESSING",
    "conditions": [
      {"type": "MODEL_SERVICE_CONDITION_TYPE_PROGRESSING", "status": "CONDITION_STATUS_TRUE"},
      {"type": "MODEL_SERVICE_CONDITION_TYPE_READY", "status": "CONDITION_STATUS_FALSE"},
      {"type": "MODEL_SERVICE_CONDITION_TYPE_FAILED", "status": "CONDITION_STATUS_FALSE"},
      {"type": "MODEL_SERVICE_CONDITION_TYPE_SUSPENDED", "status": "CONDITION_STATUS_FALSE"}
    ]
  }
}
```

### List with filter

```
GET /api/fulfillment/v1/model_services?filter=this.spec.template=='maas_shared'
```

### Delete

```
DELETE /api/fulfillment/v1/model_services/019eb38f-3ca0-7145-bc81-64cae79573ec
```

Response: `200 {}`

---

## 8. Downstream Field Mapping

How each spec field flows through the system:

```
ModelService (Fulfillment API)
    │ spec fields
    ▼
ModelServiceOrder (K8s CRD on hub cluster)
    │ AAP provisions
    ▼
AITenant + Tenant CR + MaaS CRs
```

| ModelService spec | ModelServiceOrder spec | AAP creates | Final destination |
|---|---|---|---|
| `template` | `templateID` | Selects AAP template role | AAP workflow selection |
| `gateway.type` | `gatewayType` | Gateway CR (if PER_TENANT) | Tenant CR `spec.gatewayRef` |
| `model_catalog` | `modelCatalog` | MaaSModelRef tenant bindings | MaaSAuthPolicy + MaaSSubscription |
| `identity.type` | `identityType` | Keycloak realm (if MANAGED) | Tenant CR `spec.externalOIDC` |
| `identity.oidc_issuer` | `oidcIssuer` | Passed to Tenant CR (if BYOIDP) | Tenant CR `spec.externalOIDC.issuerUrl` |
| `metadata.tenant` | `organizationRef` | org ID for Keycloak/RBAC | AITenant `spec.tenantNamespace` |

### Label linkage through the stack

```
ModelService.id (fulfillment DB)
    ↕ linked via
ModelServiceOrder.labels["osac.openshift.io/modelserviceorder-uuid"] (K8s CRD)
    ↕ feedback controller syncs status back
ModelService.status (fulfillment DB)
```

---

## 9. Design Decisions Summary

| Decision | Rationale |
|----------|-----------|
| `template` is immutable | Changing deployment model requires different infrastructure — create a new ModelService |
| `identity` is immutable | Changing IDP invalidates all API keys and breaks auth for active users |
| `gateway` is mutable | Tenant can upgrade from shared to per-tenant as needs grow |
| `model_catalog` is optional | Tenant admin manages models post-provisioning; cloud provider can seed initial access |
| `template_parameters` exists but unused | Future extensibility without proto schema changes |
| State + Conditions both exist | State = headline for dashboards; Conditions = detail for debugging/automation (Kubernetes convention) |
| `hub` is private-only | Internal infrastructure detail, not relevant to cloud provider |
| Status fields empty until READY | `gateway_url` and `api_key_endpoint` only populated after successful provisioning |
