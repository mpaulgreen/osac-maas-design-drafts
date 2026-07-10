# MaaS Custom Resources — Complete Reference

*Source: `opendatahub-io/models-as-a-service` (main branch, latest). Includes new multi-tenancy CRs from RHAIRFE-1487.*

## Quick Reference

| # | CR | API Group | Scope | Defined By | Role |
|---|------|-----------|-------|-----------|------|
| 1 | Tenant | `maas.opendatahub.io/v1alpha1` | Namespaced | MaaS | Platform config (OIDC, gateway, API keys, telemetry) |
| 2 | MaaSModelRef | `maas.opendatahub.io/v1alpha1` | Namespaced | MaaS | Model reference + dual-gate attachment point |
| 3 | MaaSAuthPolicy | `maas.opendatahub.io/v1alpha1` | Namespaced | MaaS | Gate 1: auth/authz rules → Kuadrant AuthPolicy |
| 4 | MaaSSubscription | `maas.opendatahub.io/v1alpha1` | Namespaced | MaaS | Gate 2: token rate limits → Kuadrant TRLP |
| 5 | ExternalModel | `maas.opendatahub.io/v1alpha1` | Namespaced | MaaS | External AI provider endpoint (Tech Preview) |
| 6 | **AITenant** | `maas.opendatahub.io/v1alpha1` | Namespaced | MaaS | **Multi-tenant bootstrap** (NEW — RHAIRFE-1487) |
| 7 | Config | `maas.opendatahub.io/v1alpha1` | **Cluster** | MaaS | Platform anchor for cross-namespace ownership |
| 8 | LLMInferenceService | `serving.kserve.io/v1` | Namespaced | KServe | On-cluster model serving (consumed) |
| 9 | AuthPolicy | `kuadrant.io/v1` | Namespaced | Kuadrant | Gateway auth policy (auto-created by MaaS) |
| 10 | TokenRateLimitPolicy | `kuadrant.io/v1alpha1` | Namespaced | Kuadrant | Token rate limiter (auto-created by MaaS) |
| 11 | HTTPRoute | `gateway.networking.k8s.io/v1` | Namespaced | Gateway API | Traffic routing (auto-created or wrapped) |
| 12 | Gateway | `gateway.networking.k8s.io/v1` | Namespaced | Gateway API | API gateway (pre-created by network admin) |

---

## Part 1: MaaS-Defined CRs

### 1. Tenant

**Purpose**: Configures the MaaS platform for a namespace — OIDC identity, gateway, API key policy, telemetry. In 3.4 this is a singleton (`default-tenant`); in multi-tenancy it becomes per-tenant.

**Created by**: TenantReconciler (auto-creates `default-tenant` on startup), or AITenant controller (in multi-tenant mode)
**Where it lives**: Tenant namespace (e.g., `openshift-ai` in single-tenant, `acme-maas` in multi-tenant)

**Spec**:
| Field | Type | Purpose |
|-------|------|---------|
| `gatewayRef.name` | string (1-63, DNS label) | Gateway name |
| `gatewayRef.namespace` | string (1-63, DNS label) | Gateway namespace (default: `openshift-ingress`) |
| `externalOIDC.issuerUrl` | string (9-2048, HTTPS) | Per-tenant OIDC issuer URL |
| `externalOIDC.clientId` | string (1-256) | OIDC client ID |
| `externalOIDC.ttl` | int (min 30, default 300) | JWKS cache duration (seconds) |
| `apiKeys.maxExpirationDays` | int32 (min 1) | Maximum API key lifetime |
| `telemetry.enabled` | bool (default true) | Enable metrics collection |
| `telemetry.metrics.captureOrganization` | bool (default true) | Track org_id dimension |
| `telemetry.metrics.captureUser` | bool (default false) | Track userid (**GDPR warning**) |
| `telemetry.metrics.captureGroup` | bool (default false) | Track group dimension |
| `telemetry.metrics.captureModelUsage` | bool (default true) | Track model usage |

**Status**: `phase` (Pending/Active/Degraded/Failed), conditions (Ready, DependenciesAvailable, MaaSPrerequisitesAvailable, DeploymentsAvailable, Degraded)

**Multi-tenancy**: Singleton CEL validation (`name == "default-tenant"`) removed. One Tenant CR per tenant namespace, each with its own OIDC config. Created by AITenant controller.

---

### 2. MaaSModelRef

**Purpose**: Central governance attachment point. References a backend model (LLMInferenceService or ExternalModel). A model becomes accessible **only** when both a MaaSAuthPolicy AND a MaaSSubscription reference it (dual-gate).

**Created by**: Model deployer / tenant admin
**Where it lives**: Model serving namespace (same namespace as the backend workload)

**Spec**:
| Field | Type | Purpose |
|-------|------|---------|
| `modelRef.kind` | enum: `LLMInferenceService`, `ExternalModel` | Backend type |
| `modelRef.name` | string (1-253) | Name of the referenced backend |
| `endpointOverride` | string (optional) | Override discovered endpoint URL |

**Status**: `phase` (Pending/Ready/Unhealthy/Failed/Invalid), `endpoint`, `httpRouteName`, `httpRouteNamespace`, `httpRouteGatewayName`, `httpRouteGatewayNamespace`, `httpRouteHostnames[]`, conditions (Ready, GovernanceAttached, RuntimeReady)

**Key behavior**: Phase transitions to `Ready` only when backend is healthy AND at least one MaaSAuthPolicy + one MaaSSubscription both reference this model. Creates/wraps an HTTPRoute per backend kind.

**Multi-tenancy**: Future `tenantBinding` field proposed for explicit model-to-tenant grants. Today, access controlled via per-tenant MaaSAuthPolicy/MaaSSubscription.

---

### 3. MaaSAuthPolicy

**Purpose**: **Gate 1 of the dual-gate.** Defines who may access which models. Multiple MaaSAuthPolicies referencing the same model are aggregated into a single Kuadrant `AuthPolicy` per model.

**Created by**: Tenant admin in tenant namespace
**Where it lives**: Tenant namespace

**Spec**:
| Field | Type | Purpose |
|-------|------|---------|
| `modelRefs[].name` | string (1-63) | Target model name |
| `modelRefs[].namespace` | string (1-63) | Target model namespace |
| `subjects.groups[]` | []GroupReference | Authorized K8s groups (OR logic) |
| `subjects.users[]` | []string | Authorized K8s users (OR logic) |
| `meteringMetadata.organizationId` | string | Billing org ID |
| `meteringMetadata.costCenter` | string | Cost center tag |
| `meteringMetadata.labels` | map[string]string | Custom billing labels |

**Status**: `phase` (Pending/Active/Degraded/Failed), `authPolicies[]` (per-model Kuadrant AuthPolicy status)

**What the controller generates** (Kuadrant AuthPolicy named `maas-auth-{modelName}`):
- Authentication: API keys (`sk-oai-*` prefix via maas-api validation), K8s TokenReview, OIDC JWT (if Tenant has `externalOIDC`)
- Authorization: OPA rules enforcing group/user membership + subscription validation
- Response: strips Authorization header, injects subscription metadata for rate limiting

**Multi-tenancy**: Per-tenant instances in tenant namespaces. Multiple tenants' policies aggregate into one Kuadrant AuthPolicy per model — subjects from different tenants are OR'd together.

---

### 4. MaaSSubscription

**Purpose**: **Gate 2 of the dual-gate.** Defines token-based rate limits per subscription tier. Multiple MaaSSubscriptions referencing the same model are aggregated into a single Kuadrant `TokenRateLimitPolicy` per model.

**Created by**: Tenant admin in tenant namespace
**Where it lives**: Tenant namespace

**Spec**:
| Field | Type | Purpose |
|-------|------|---------|
| `owner.groups[]` | []GroupReference | Subscription owner groups |
| `owner.users[]` | []string | Subscription owner users |
| `modelRefs[].name` | string (1-63) | Target model name |
| `modelRefs[].namespace` | string (1-63) | Target model namespace |
| `modelRefs[].tokenRateLimits[].limit` | int64 (1-1B) | Max tokens allowed |
| `modelRefs[].tokenRateLimits[].window` | string (pattern `^[1-9]\d{0,3}(s|m|h)$`) | Time window |
| `modelRefs[].billingRate.perToken` | string | Cost per token |
| `tokenMetadata.organizationId` | string | Billing org ID |
| `tokenMetadata.costCenter` | string | Cost center |
| `priority` | int32 (default 0) | Priority across multiple subs (higher wins) |

**Status**: `phase` (Pending/Active/Degraded/Failed), `modelRefStatuses[]`, `tokenRateLimitStatuses[]`

**What the controller generates** (Kuadrant TRLP named `maas-trlp-{modelName}`):
- Per-subscription rate limits with predicate matching on `auth.identity.selected_subscription_key`
- Per-user token counting via `auth.identity.userid` counter
- `/v1/models` endpoint exempt from token budget
- Default: 100 tokens/min if unspecified

**Multi-tenancy**: Per-tenant instances. `X-MaaS-Tenant` header added as discriminator for tenant-scoped budgets on shared gateway. Conflict detection flags duplicate priorities for same model.

---

### 5. ExternalModel

**Purpose**: References an external AI model provider (OpenAI, Anthropic, etc.). Enables MaaS governance over external endpoints without on-cluster serving.

**Created by**: Tenant admin
**Where it lives**: Model serving namespace
**Status**: Tech Preview

**Spec**:
| Field | Type | Purpose |
|-------|------|---------|
| `provider` | string (1-63) | API format identifier (e.g., `"openai"`, `"anthropic"`) |
| `targetModel` | string (1-253) | Upstream model name (e.g., `"gpt-4o"`) |
| `endpoint` | string (1-253, hostname) | Provider FQDN without scheme (e.g., `"api.openai.com"`) |
| `credentialRef.name` | string (1-253) | Secret name containing `data.api-key` |

**Status**: `phase` (Pending/Ready/Failed), conditions

**Multi-tenancy**: Can be shared via MaaSModelRef or per-tenant with tenant-specific credentials.

---

### 6. AITenant (NEW — Multi-Tenancy Bootstrap)

**Purpose**: Bootstraps one tenant slice on a shared MaaS cluster. **The central multi-tenancy primitive for RHAIRFE-1487.** Creates labeled namespace, Tenant CR, and RBAC in one declarative step.

**Created by**: Platform admin, or OSAC controller (in Gap II integration)
**Where it lives**: Infrastructure namespace (`redhat-ai-gateway-infra`), NOT the tenant namespace
**Short name**: `ait`

**Spec**:
| Field | Type | Purpose |
|-------|------|---------|
| `tenantNamespace.name` | string (1-63, DNS label, **immutable**) | Target tenant namespace name |
| `tenantNamespace.create` | bool (default true) | Whether controller creates the namespace |
| `gateway.name` | string (max 63, optional) | Pre-existing Gateway reference (defaults to AITenant name) |
| `oidc.issuerUrl` | string (HTTPS) | OIDC issuer for the tenant |
| `oidc.clientId` | string | OIDC client ID |
| `oidc.ttl` | int | JWKS cache TTL |
| `rbac.admins[]` | []AITenantRBACSubject (max 128) | Tenant admin subjects |
| `rbac.admins[].kind` | enum: User, Group, ServiceAccount | Subject type |
| `rbac.admins[].name` | string (1-253) | Subject name |
| `rbac.admins[].namespace` | string (optional, required for SA) | Subject namespace |

**Status**: `phase` (Pending/Active/Failed), `tenantNamespace`, `gatewayRef`, conditions (Ready)

**What the controller creates**:
1. **Tenant namespace** with label `maas.opendatahub.io/tenant: <name>`
2. **Tenant CR** in tenant namespace (inherits OIDC, gateway config from AITenant)
3. **tenant-admin Role** in tenant namespace: CRUD on MaaSAuthPolicy, MaaSSubscription; get/update Tenant; read MaaSModelRef
4. **tenant-admin RoleBinding** for specified admins
5. **object-admin Role** in infrastructure namespace: read-only on the AITenant object
6. **object-admin RoleBinding**

**What the controller does NOT create**: Gateway (validates existence only), models, subscriptions, auth policies

**Critical for OSAC integration**: Instead of AAP creating namespace + Tenant CR + RBAC manually, AAP creates a single AITenant CR and the MaaS AITenant controller handles everything else. This dramatically simplifies the Option 2 integration path.

---

### 7. Config

**Purpose**: Cluster-scoped anchor for MaaS platform resources. Enables cross-namespace ownership (K8s prohibits cross-namespace owner refs). Spec and status are empty in v1alpha1.

**Created by**: LifecycleReconciler (maas-controller initialization)
**Where it lives**: Platform namespace (cluster-scoped)
**Spec/Status**: Empty (placeholder for future platform-wide settings)

**Referenced as owner by**: Deployments, ConfigMaps, Services, Secrets (via cross-namespace SSA)

---

## Part 2: Consumed External CRs

### 8. LLMInferenceService (KServe)

**API**: `serving.kserve.io/v1` | **Defined by**: KServe (OpenDataHub fork)

MaaS does NOT create this — RHOAI operator/KServe provisions it. MaaS references it via MaaSModelRef (`modelRef.kind: LLMInferenceService`).

**Fields MaaS uses**: `Status.Ready` (health), `Status.Addresses` (endpoint discovery, specifically `"gateway-external"` address), `Status.URL` (endpoint URL)

**Multi-tenancy**: Shared model = one LIS, multiple tenants access via separate MaaSAuthPolicy/MaaSSubscription. Dedicated model = per-tenant LIS (avoids KV-cache leak).

### 9. AuthPolicy (Kuadrant)

**API**: `kuadrant.io/v1` | **Defined by**: Kuadrant | **Created by**: MaaSAuthPolicy controller (auto-generated)

One AuthPolicy per model (`maas-auth-{modelName}`), aggregating subjects from ALL MaaSAuthPolicies that reference that model. Targets the model's HTTPRoute.

Contains: API key validation (via maas-api metadata call), K8s TokenReview, OIDC JWT, OPA authorization rules, header stripping, subscription metadata injection.

**Manual edits are overwritten** — the controller owns the full spec. Opt out by removing the `app.kubernetes.io/managed-by: maas-controller` label.

### 10. TokenRateLimitPolicy (Kuadrant)

**API**: `kuadrant.io/v1alpha1` | **Defined by**: Kuadrant | **Created by**: MaaSSubscription controller (auto-generated)

One TRLP per model (`maas-trlp-{modelName}`), aggregating limits from ALL MaaSSubscriptions referencing that model. Per-user counting via `auth.identity.userid`. `/v1/models` exempt.

**Manual edits are overwritten** — controller-owned.

### 11. HTTPRoute (Gateway API)

**API**: `gateway.networking.k8s.io/v1` | **Created by**: KServe (for LLMInferenceService) or MaaS (for ExternalModel)

MaaS wraps KServe-created routes (doesn't own them) or creates new routes for external models. AuthPolicy and TRLP target HTTPRoutes via `targetRef`.

### 12. Gateway (Gateway API)

**API**: `gateway.networking.k8s.io/v1` | **Created by**: Network admin (outside MaaS)

MaaS does NOT create gateways. Referenced by Tenant CR and AITenant CR. AITenant controller validates existence. Default: `openshift-ingress/maas-default-gateway`.

---

## Part 3: Design Patterns

### The Dual-Gate Security Model

```
User request (API key or JWT)
       │
       ▼
  Gate 1: AuthPolicy (Authorino)
  ├── Validate API key (sk-oai-* → maas-api /internal/v1/api-keys/validate)
  ├── OR K8s TokenReview
  ├── OR OIDC JWT (per-tenant issuer from Tenant CR)
  ├── OPA: is user in authorized group?
  └── Inject: subscription key, tenant context
       │
       ▼ (401/403 if fails)
  Gate 2: TokenRateLimitPolicy (Limitador)
  ├── Predicate: auth.identity.selected_subscription_key matches
  ├── Check: tokens remaining in window?
  └── Counter: per-user via auth.identity.userid
       │
       ▼ (429 if over budget)
  Model backend (LLMInferenceService / ExternalModel)
```

Both gates must pass. Absence of either = default-deny.

### The Aggregation Pattern

```
Multiple MaaS CRs (per tenant/tier)     →     One Kuadrant policy (per model)

MaaSAuthPolicy "acme-users"   ─┐
MaaSAuthPolicy "contoso-users" ─┼──► AuthPolicy "maas-auth-llama3"
MaaSAuthPolicy "globex-admins" ─┘    (subjects aggregated, OR logic)

MaaSSubscription "acme-free"     ─┐
MaaSSubscription "acme-premium"   ─┼──► TRLP "maas-trlp-llama3"
MaaSSubscription "contoso-ent"    ─┘    (limits aggregated, predicate-matched)
```

This means one Kuadrant policy per model regardless of how many tenants or tiers exist. Changes to any MaaS CR trigger re-aggregation.

### Cross-Namespace Tracking (Labels)

K8s prohibits cross-namespace owner refs. MaaS uses labels:

| Label | On | Purpose |
|-------|------|---------|
| `maas.opendatahub.io/model` | AuthPolicy, TRLP | Links to MaaSModelRef |
| `maas.opendatahub.io/model-namespace` | TRLP | Model namespace |
| `maas.opendatahub.io/tenant` | Namespace | Identifies tenant namespace |
| `app.kubernetes.io/managed-by: maas-controller` | AuthPolicy, TRLP | Ownership marker |
| `maas.opendatahub.io/subscriptions` (annotation) | TRLP | Comma-separated subscription list |

---

## Part 4: Multi-Tenancy Architecture

### Namespace Layout (Option 2 — Shared Cluster)

```
openshift-ai (platform namespace, shared)
  ├── Config CR (cluster-scoped)
  ├── maas-api Deployment (shared, tenant column in PostgreSQL)
  ├── maas-controller Deployment (shared, reconciles all tenant namespaces)
  └── PostgreSQL (api_keys table with tenant column)

redhat-ai-gateway-infra (infrastructure namespace)
  ├── AITenant "acme"     → creates acme-maas namespace + Tenant CR + RBAC
  ├── AITenant "contoso"  → creates contoso-maas namespace + Tenant CR + RBAC
  └── AITenant "globex"   → creates globex-maas namespace + Tenant CR + RBAC

acme-maas (tenant namespace, created by AITenant controller)
  ├── Tenant CR (acme's OIDC config, gateway ref)
  ├── MaaSAuthPolicy (acme's access rules)
  └── MaaSSubscription (acme's rate-limit tiers)

contoso-maas (tenant namespace)
  ├── Tenant CR (contoso's OIDC config)
  ├── MaaSAuthPolicy
  └── MaaSSubscription

model-serving-ns (model namespace)
  ├── LLMInferenceService (backend model)
  ├── MaaSModelRef (governance attachment, future tenantBinding field)
  ├── HTTPRoute (auto-created/wrapped)
  ├── AuthPolicy (aggregates acme + contoso subjects)
  └── TokenRateLimitPolicy (aggregates acme + contoso limits)

openshift-ingress (gateway namespace)
  └── Gateway (shared, or per-tenant if AIGateway approach)
```

### Tenant Onboarding via AITenant (Option 2)

```
Platform admin creates:
  AITenant "acme" in redhat-ai-gateway-infra
    spec:
      tenantNamespace:
        name: acme-maas
        create: true
      gateway:
        name: shared-gateway
      oidc:
        issuerUrl: https://keycloak.example.com/realms/acme
        clientId: acme-maas-client
      rbac:
        admins:
          - kind: Group
            name: acme-admins

AITenant controller automatically creates:
  1. Namespace "acme-maas" (labeled maas.opendatahub.io/tenant: acme)
  2. Tenant CR in acme-maas (OIDC + gateway from AITenant spec)
  3. Role "tenant-admin" in acme-maas
  4. RoleBinding for acme-admins → tenant-admin

Tenant admin then creates:
  5. MaaSSubscription(s) in acme-maas (rate-limit tiers)
  6. MaaSAuthPolicy(s) in acme-maas (access rules)

Model deployer grants access:
  7. MaaSModelRef in model-ns (future: tenantBinding includes "acme")
```

### OSAC Integration Impact (Gap II)

The discovery of AITenant **simplifies the OSAC integration**. Instead of the AAP template creating namespace + Tenant CR + RBAC manually (7-step workflow), it creates a single AITenant CR:

```
OSAC AAP maas_shared template (simplified):

1. Configure identity (Keycloak realm or BYOIDP)
2. Create AITenant CR in redhat-ai-gateway-infra
   → AITenant controller handles: namespace, Tenant CR, RBAC
3. Create MaaSSubscription + MaaSAuthPolicy in tenant namespace
4. Grant model access via MaaSModelRef
5. Report status
```

### Option 3 (Dedicated Cluster) — CR Differences

| CR | Option 2 (shared) | Option 3 (dedicated) |
|----|-------------------|---------------------|
| AITenant | Yes (one per tenant in infra ns) | No (single-tenant cluster) |
| Tenant | Multiple (one per tenant ns) | Single (default-tenant) |
| MaaSAuthPolicy | Per-tenant in tenant ns | Single in platform ns |
| MaaSSubscription | Per-tenant in tenant ns | Single in platform ns |
| Gateway | Shared (or per-tenant if AIGateway) | Per-cluster |
| AuthPolicy | Aggregates multiple tenants | Single tenant |
| TRLP | Aggregates multiple tenants | Single tenant |
