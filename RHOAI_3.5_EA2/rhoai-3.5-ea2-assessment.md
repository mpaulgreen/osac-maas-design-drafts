# RHOAI 3.5 EA2 Assessment — Impact on MaaS-on-OSAC

*Assessment date: July 17, 2026*
*Source: `opendatahub-io/opendatahub-operator` tag `odh-3.5.0-ea.2` (June 17, 2026) + `opendatahub-io/models-as-a-service` main branch*
*Audience: OSAC MaaS team, AI BU stakeholders*

---

## 1. Executive Summary

RHOAI 3.5 introduces a **fundamentally restructured tenancy and gateway architecture** that will require updates to our OSAC MaaS implementation. The key changes are: (1) a new `GatewayConfig` CRD that manages the platform gateway (separate from MaaS gateway), (2) a three-tier tenant model (`AITenant` → `MaasTenantConfig` → deprecated `Tenant`) that moves from single-tenant to per-tenant maas-api Deployments with dedicated gateways, (3) new gateway annotations required for operator validation, and (4) a dual ExternalModel API with the new `inference.opendatahub.io` group adding the `path` field and `ExternalProvider` abstraction.

> **Note:** If OSAC ships starting from RHOAI 3.5 (no 3.4 backward compatibility), no version-conditional logic is needed in the Ansible role. The changes below are direct replacements, not conditional branches. IPP (payload-processing) deployment is operator-managed in both 3.4 and 3.5 — our Ansible role does not deploy it. Similarly, our role does not patch DSCI `spec.serviceMesh` (SM version pinning is done via Subscription channel and install plan approval).

**Key numbers:**
- **4 breaking changes** requiring Ansible role updates
- **0 RHOAI-specific PRs** made the EA2 cut (tag was June 17; all significant PRs merged after)
- **4 ai-gateway PRs** already on `stable` branch — will ship in next build (EA3 or GA)
- **RHOAI 3.5 GA target: August 20, 2026** — 5 weeks from this assessment

**Bottom line:** If OSAC ships starting from RHOAI 3.5 (no 3.4 backward compatibility), no version-conditional logic is needed — direct replacements only. The Tenant CR must be replaced with AITenant + MaasTenantConfig, and gateway annotations must be added. IPP deployment and DSCI monitoring patches remain unchanged (our role doesn't manage IPP, and DSCI `spec.monitoring.metrics` still exists in v2). The multi-tenancy model aligns well with our OSAC Option 2B design but requires dedicated gateways per tenant — a departure from our current shared-gateway assumption.

---

## 2. AI Gateway Impact

### 2.1 New `GatewayConfig` CRD

RHOAI 3.5 introduces `GatewayConfig` (`services.platform.opendatahub.io/v1alpha1`), a **cluster-scoped singleton** named `default-gateway` that manages the **platform gateway** for RHOAI components (dashboard, model registry, etc.).

| Field | Description | Default |
|-------|-------------|---------|
| `spec.ingressMode` | How gateway is exposed | `OcpRoute` (new installs), `LoadBalancer` (upgrades) |
| `spec.oidc` | OIDC provider config (issuerURL, clientID, clientSecretRef) | — |
| `spec.certificate` | TLS management | `OpenshiftDefaultIngress` |
| `spec.domain` / `spec.subdomain` | Hostname config | subdomain: `rh-ai` |
| `spec.enableK8sTokenValidation` | K8s SA token auth | `true` |
| `status.domain` | Computed gateway FQDN | — |

**What it creates:**
- Gateway: `data-science-gateway` in `openshift-ingress` (NOT `maas-default-gateway`)
- GatewayClass: `data-science-gateway-class` (controller: `openshift.io/gateway-controller/v1`)
- Deploys `kube-auth-proxy` (OAuth2 Proxy) for component auth
- Creates EnvoyFilter, NetworkPolicy, OCP Routes (in OcpRoute mode)

### 2.2 Platform Gateway vs MaaS Gateway — Two Separate Resources

**Critical finding:** The platform gateway (`data-science-gateway`) and MaaS gateway (`maas-default-gateway`) are **completely separate** resources. The `GatewayConfig` CRD manages only the platform gateway. The MaaS gateway is still a manually-created resource.

```
┌─────────────────────────────────────────────────────────┐
│                    openshift-ingress                      │
│                                                           │
│  ┌─────────────────────┐   ┌──────────────────────────┐  │
│  │  data-science-gateway│   │  maas-default-gateway    │  │
│  │  (operator-managed)  │   │  (manually created)      │  │
│  │  GatewayConfig CR    │   │  Ansible role / admin    │  │
│  │                      │   │                          │  │
│  │  Dashboard, Registry │   │  MaaS governance         │  │
│  │  KServe inference    │   │  Auth + Rate limiting    │  │
│  └─────────────────────┘   └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 2.3 `maas-default-gateway` — Still Manual, New Annotations Required

The RHOAI 3.5 operator **does NOT create** `maas-default-gateway`. It only validates its existence and checks for two annotations:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: maas-default-gateway
  namespace: openshift-ingress
  annotations:
    opendatahub.io/managed: "false"                            # NEW — tells operator not to manage
    security.opendatahub.io/authorino-tls-bootstrap: "true"    # NEW — confirms Authorino TLS
```

**Our Ansible role must add these annotations** to `maas-default-gateway`. Without them, the operator's `MaaSPrerequisitesAvailable` condition will be false.

### 2.4 IngressMode: OcpRoute vs LoadBalancer

| Mode | How | When |
|------|-----|------|
| `OcpRoute` (default for new installs) | ClusterIP Service + OpenShift Route | Standard OpenShift ingress |
| `LoadBalancer` (preserved on upgrade) | NLB / cloud LB Service | Cloud environments, spoke-hub |

**Upgrade behavior:** If our existing gateway uses `LoadBalancer` (NLB), the operator's `MigrateGatewayConfigIngressMode` migration preserves it. For new deployments, `OcpRoute` is the default. Our Ansible role should set `ingressMode: LoadBalancer` explicitly on `GatewayConfig` if NLB access is needed.

### 2.5 DSCI v2 Drops `spec.serviceMesh`

**Breaking change.** DSCI v2 (the new storage version) removes `spec.serviceMesh` entirely. The v1-to-v2 conversion discards it. ServiceMesh configuration is now managed via `GatewayConfig` CR.

| RHOAI 3.4 | RHOAI 3.5 |
|------------|-----------|
| `DSCI.spec.serviceMesh.managementState: Managed` | Removed — RHOAI operator no longer manages SM lifecycle |
| Manual SM version pinning via Subscription channel | Ingress Operator manages istiod version (tied to OCP version, not RHOAI) |

`GatewayConfig` itself has no field that controls SM or istiod. It creates the `data-science-gateway` Gateway CR with `gatewayClassName: data-science-gateway-class`, and the **Ingress Operator** (OCP component) manages istiod — including which Istio version is deployed (1.26.2 on OCP 4.20, 1.27.3 on OCP 4.21).

**Impact on our Ansible role:** None directly. Our role does **not** patch DSCI `spec.serviceMesh` — it only patches `spec.monitoring.metrics` for observability. SM version pinning is done via Subscription channel (`stable-3.2`) and install plan approval, which is separate from DSCI. The `GatewayConfig` CR manages the platform gateway (`data-science-gateway`), not our MaaS gateway.

### 2.6 Payload-Processing (IPP) Auto-Deployed

In RHOAI 3.5, the operator deploys payload-processing as part of the MaaS component. Resources deployed to the **gateway namespace** (`openshift-ingress`):

| Resource | Name |
|----------|------|
| Deployment | `payload-processing` |
| Service | `payload-processing` |
| ServiceAccount | `payload-processing` |
| ConfigMap | `payload-processing-plugins` |
| ClusterRoleBinding | `payload-processing-reader` |

**Two-stage ext-proc architecture:**
1. **Stage 1 (`ipp-pre`)**: Runs BEFORE Kuadrant WasmPlugin. Sets `X-Gateway-Model-Name` header. Port 9004 → `payload-pre-processing` service.
2. **Stage 2 (`ipp`)**: Runs AFTER WasmPlugin. Full request/response processing (auth headers, path rewrite, metering). Port 9004 → `payload-processing` service.

The EnvoyFilter now uses **dual anchor support** — matches both `extensions.istio.io/wasmplugin` (RHCL 1.3.x/community Kuadrant) and `envoy.filters.http.wasm` (RHCL 1.4.x). This suggests RHOAI 3.5 may work with both RHCL versions.

**Impact on our Ansible role:** None. Our `ocp_4_20_ai_maas` role does **not** deploy IPP — the RHOAI operator handles it in both 3.4 and 3.5 as part of the MaaS component (`modelsAsService: Managed` in DSC). No changes needed.

### 2.7 Gateway Impact Summary

| Our Current Implementation (3.4) | RHOAI 3.5 Change | Action Required |
|----------------------------------|-------------------|-----------------|
| Manual `maas-default-gateway` creation | Still manual, operator validates only | **Keep** — add 2 new annotations |
| IPP deployment | Operator-managed in both 3.4 and 3.5 | **None** — role doesn't deploy IPP |
| DSCI `spec.monitoring.metrics` patching | Field still exists in DSCI v2 | **None** — only `spec.serviceMesh` was removed (we don't use it) |
| SM version pinning (`stable-3.2` channel) | RHOAI 3.5 auto-manages SM via DSC; Ingress Operator controls Istio version | **Verified: not needed once upstream IPP ext-proc bugs are fixed** (NetworkPolicy, DestinationRule TLS, route disable — see Section 8.7). Currently requires two manual workarounds. Remove SM pinning tasks from role only after upstream fixes ship. |
| `openshift-ai-inference` manual creation | Not found in 3.5 operator | **Keep** workaround — operator still does not auto-create this gateway |
| GatewayConfig CR | NEW — manages platform gateway | **Informational** — does not affect MaaS gateway |

---

## 3. AI Tenant Impact

### 3.1 Three-Tier CRD Hierarchy

RHOAI 3.5 introduces a three-tier tenant model that **replaces the singleton Tenant CR** with purpose-separated resources:

```
┌────────────────────────────────────────────────────────────┐
│                      AITenant (bootstrap)                   │
│  Namespace: ai-tenants                                      │
│  Purpose: Platform context (gateway ref, OIDC config)       │
│  Creates: tenant namespace, MaasTenantConfig, Roles         │
│                                                              │
│  spec.gateway.name: "maas-default-gateway"                   │
│  spec.oidc.issuerUrl: "https://keycloak.../realms/maas"     │
│  spec.oidc.clientId: "maas-client"                           │
│  spec.rbac: DEPRECATED — controller ignores this field       │
├────────────────────────────────────────────────────────────┤
│                  MaasTenantConfig (runtime)                   │
│  Namespace: ai-tenant-{name} (per-tenant)                    │
│  Singleton name: "default-tenant"                            │
│  Purpose: MaaS-specific runtime config only                  │
│                                                              │
│  spec.apiKeys.maxExpirationDays: 90                          │
│  spec.telemetry.enabled: true                                │
│  spec.telemetry.metrics.captureUser: false                   │
│  spec.telemetry.metrics.captureOrganization: true            │
├────────────────────────────────────────────────────────────┤
│                    Tenant (DEPRECATED)                        │
│  Purpose: Migration compatibility only                       │
│  Controller copies config to AITenant + MaasTenantConfig     │
│  Then marks with deprecation annotations and removes         │
│  its finalizer                                               │
└────────────────────────────────────────────────────────────┘
```

### 3.2 What AITenant Creates

When an `AITenant` CR is created, the controller reconciles:

1. **Tenant namespace** named `ai-tenant-{aitenant-name}` (exception: default tenant uses `models-as-a-service`)
2. **MaasTenantConfig CR** (`default-tenant`) in the tenant namespace
3. **`tenant-admin` Role** in the tenant namespace — grants CRUD on `maasauthpolicies`, `maassubscriptions`, read on `maasmodelrefs`, patch on `maastenantconfigs`
4. **`object-admin` Role** in the `ai-tenants` namespace — grants `get` on the AITenant object
5. **Gateway claim ConfigMap** — atomically claims a gateway (prevents race conditions)

**Does NOT create:** RoleBindings (admin must create manually), Gateway (must pre-exist).

### 3.3 Per-Tenant maas-api Deployment

**Fundamental architectural change.** maas-api is now deployed **per tenant**, not shared.

| Tenant | Deployment Name | Service Name | Namespace |
|--------|----------------|--------------|-----------|
| Default (`models-as-a-service`) | `maas-api` | `maas-api` | `odh-ai-gateway-infra` / `redhat-ai-gateway-infra` |
| Additional (e.g., `redteam`) | `maas-api-redteam` | `maas-api-redteam` | Same infra namespace |

Each maas-api instance receives:
- `TENANT_NAME` env var — for DB queries (`WHERE tenant = $TENANT_NAME`)
- `MAAS_SUBSCRIPTION_NAMESPACE` — the tenant's namespace
- `GATEWAY_NAMESPACE` / `GATEWAY_NAME` — the tenant's dedicated gateway
- Pod label `maas.opendatahub.io/tenant-instance` — unique Service selector

**Impact on resource planning:** Each additional tenant adds a maas-api Deployment + Service. For 10 tenants, that's 10 maas-api pods in the infra namespace. Factor into cluster sizing.

### 3.4 Dedicated Gateway Per Tenant

**Each AITenant requires a dedicated Gateway. Gateways CANNOT be shared.**

The AITenant webhook (`validateGatewayUniqueness()`) lists all AITenants and rejects creation if any other AITenant uses the same gateway. Error: *"each AITenant requires a dedicated Gateway for isolation"*.

The gateway must be **pre-created by a network admin** before the AITenant is created. The controller validates its existence but does not create it.

**Impact on our architecture:**
- Our current single `maas-default-gateway` serves only the default tenant
- Each additional OSAC Organization → OSAC creates a dedicated Gateway → creates AITenant pointing to it
- This multiplies gateway infrastructure (NLB/Route per tenant)
- Aligns with **Option 3** (cluster-per-tenant) more than Option 2B (shared cluster)

### 3.5 OIDC Configuration Moves to AITenant

| RHOAI 3.4 | RHOAI 3.5 |
|------------|-----------|
| `Tenant.spec.externalOIDC.issuerUrl` | `AITenant.spec.oidc.issuerUrl` |
| `Tenant.spec.externalOIDC.clientId` | `AITenant.spec.oidc.clientId` |
| — | `AITenant.spec.oidc.ttl` (default 300s, min 30s — JWKS cache) |

When `MaasTenantConfig` has label `maas.opendatahub.io/managed-by-aitenant=true`, the tenant reconciler reads gateway ref and OIDC config from the owning AITenant, NOT from MaasTenantConfig.

### 3.6 RBAC Deprecated

`AITenant.spec.rbac` is **explicitly deprecated and ignored** by the controller. The field exists only for schema compatibility.

**Impact:** Our Ansible role must create standard Kubernetes RoleBindings manually instead of relying on `spec.rbac`.

### 3.7 Namespace Changes

| Purpose | RHOAI 3.4 | RHOAI 3.5 |
|---------|-----------|-----------|
| AITenant CRs | N/A | `ai-tenants` (configurable via `--aitenant-namespace`) |
| Default tenant MaaS CRs | `redhat-ai-gateway-infra` | `models-as-a-service` |
| Additional tenant MaaS CRs | Manual | `ai-tenant-{aitenant-name}` (auto-created) |
| maas-api Deployments | `redhat-ai-gateway-infra` | `odh-ai-gateway-infra` / `redhat-ai-gateway-infra` |
| maas-controller | `redhat-ods-applications` | `opendatahub` / `redhat-ods-applications` |
| Gateways | `openshift-ingress` | `openshift-ingress` (unchanged) |

### 3.8 Self-Bootstrap and Upgrade Migration

On startup, the controller:

1. Creates `AITenant/models-as-a-service` in `ai-tenants` namespace
2. `LifecycleReconciler` creates `Config/default` (cluster-scoped anchor)
3. AITenant reconciler creates `MaasTenantConfig/default-tenant` in `models-as-a-service` namespace

**Legacy migration:** If an existing `Tenant/default-tenant` is found:
1. Copies `spec.externalOIDC` → `AITenant.spec.oidc`
2. Copies `spec.gatewayRef.name` → `AITenant.spec.gateway.name`
3. Copies `spec.apiKeys` + `spec.telemetry` → `MaasTenantConfig`
4. Annotates Tenant as deprecated: `maas.opendatahub.io/deprecated-by: MaasTenantConfig`
5. Removes Tenant's finalizer

### 3.9 Multi-Tenancy Discovery Flag

The controller flag `--enable-tenant-namespace-discovery` controls multi-tenancy:

| Value | Behavior |
|-------|----------|
| `false` | Single-tenant mode — only reconciles default `MaasTenantConfig` |
| `true` | Multi-tenant — watches namespaces with `ai-gateway.opendatahub.io/tenant` label |

**In RHOAI 3.5, this flag is enabled by default.** The upstream Deployment manifest (`deployment/base/maas-controller/manager/manager.yaml`) includes `--enable-tenant-namespace-discovery` as a bare boolean arg (Go interprets bare boolean flag as `true`). The RHOAI operator applies this manifest verbatim — no override, no DSC field, no ConfigMap. No user action is required to enable multi-tenancy. There is no supported DSC-level knob to disable it.

### 3.10 AI Tenant Impact Summary

| Our Current Implementation (3.4) | RHOAI 3.5 Change | Action Required |
|----------------------------------|-------------------|-----------------|
| `Tenant/default-tenant` in `redhat-ai-gateway-infra` | Deprecated → `AITenant` + `MaasTenantConfig` | **Rewrite** tenant bootstrap |
| Single shared maas-api | Per-tenant maas-api Deployment | **Update** resource planning |
| Single `maas-default-gateway` for all tenants | Dedicated gateway per tenant (webhook-enforced) | **Create** per-tenant gateways |
| `spec.rbac` for RoleBindings | RBAC deprecated, ignored | **Create** manual RoleBindings |
| OIDC on `Tenant.spec.externalOIDC` | OIDC on `AITenant.spec.oidc` | **Migrate** OIDC config |
| `captureUser` on Tenant CR | `captureUser` on MaasTenantConfig | **Migrate** telemetry config |
| Namespace `redhat-ai-gateway-infra` | Namespace `models-as-a-service` | **Update** namespace references |
| — | `--enable-tenant-namespace-discovery` flag | **None** — enabled by default in RHOAI 3.5 |

---

## 4. Tenancy Architecture

### 4.1 Overall Namespace Layout

```
Cluster
├── ai-tenants/                              # AITenant CRs live here
│   ├── AITenant/models-as-a-service         # Default tenant (auto-created on startup)
│   ├── AITenant/org-alpha                   # Tenant for Organization Alpha
│   └── AITenant/org-beta                    # Tenant for Organization Beta
│
├── models-as-a-service/                     # Default tenant subscription namespace
│   ├── MaasTenantConfig/default-tenant      # Runtime config
│   ├── MaaSModelRef/granite-8b              # Model references
│   ├── MaaSAuthPolicy/...                   # Auth policies
│   └── MaaSSubscription/...                 # Rate limit subscriptions
│
├── ai-tenant-org-alpha/                     # Org Alpha subscription namespace
│   ├── MaasTenantConfig/default-tenant
│   ├── MaaSModelRef/...
│   ├── MaaSAuthPolicy/...
│   └── MaaSSubscription/...
│
├── ai-tenant-org-beta/                      # Org Beta subscription namespace
│   └── (same structure)
│
├── odh-ai-gateway-infra/                    # Infrastructure namespace (all maas-api pods)
│   ├── Deployment/maas-api                  # Default tenant
│   ├── Deployment/maas-api-org-alpha        # Org Alpha
│   └── Deployment/maas-api-org-beta         # Org Beta
│
└── openshift-ingress/                       # Gateways (one per tenant)
    ├── Gateway/maas-default-gateway         # Default tenant
    ├── Gateway/maas-org-alpha-gateway       # Org Alpha (pre-created)
    ├── Gateway/maas-org-beta-gateway        # Org Beta (pre-created)
    ├── Deployment/payload-processing        # IPP (operator-managed, shared)
    └── data-science-gateway                 # Platform gateway (GatewayConfig-managed)
```

### 4.2 Label Taxonomy

Labels applied to tenant namespaces:

| Label | Value | Purpose |
|-------|-------|---------|
| `maas.opendatahub.io/managed-by-aitenant` | `"true"` | Identifies AITenant-managed namespaces |
| `ai-gateway.opendatahub.io/tenant` | `{aitenant-name}` | Tenant namespace discovery |
| `maas.opendatahub.io/tenant-name` | `{aitenant-name}` | Tenant identification |
| `maas.opendatahub.io/tenant-namespace` | `{tenant-namespace}` | Self-referential |
| `opendatahub.io/generated-namespace` | `"true"` | Marks as operator-generated |
| `app.kubernetes.io/managed-by` | `maas-controller` | Standard K8s label |
| `app.kubernetes.io/part-of` | `modelsasservice` | Component label |

### 4.3 Resource Naming Convention

Multi-tenant resource naming follows a strict pattern:

- **Default tenant** (empty tenantID): base name as-is (e.g., `maas-api`)
- **Named tenant** (e.g., `org-alpha`): `{base}-{tenantID}` (e.g., `maas-api-org-alpha`)
- **Name length constraint**: AITenant name max **41 characters** (longest base: `maas-api-auth-policy` = 21 chars + 1 separator + 41 = 63 K8s limit)
- **Name validation**: DNS-1123 label (lowercase alphanumeric + hyphens)

### 4.4 Database Isolation

Database isolation uses a **shared PostgreSQL with `tenant_id` column** (schema-level isolation, not database-level):

- Each maas-api instance receives `TENANT_NAME` env var
- All DB queries filter by `WHERE tenant = $TENANT_NAME`
- API keys table has `tenant` column — scopes keys to the correct tenant
- No cross-tenant data leakage at the application layer

### 4.5 RHOAI 3.4 vs 3.5 Comparison

| Aspect | RHOAI 3.4 (Our Current) | RHOAI 3.5 |
|--------|--------------------------|-----------|
| Tenant CRD | `Tenant` (singleton) | `AITenant` + `MaasTenantConfig` |
| maas-api | Single shared instance | Per-tenant Deployment |
| Gateway | Shared `maas-default-gateway` | Dedicated per tenant |
| Namespace | `redhat-ai-gateway-infra` | `ai-tenant-{name}` per tenant |
| OIDC | On Tenant CR | On AITenant CR |
| RBAC | `Tenant.spec.rbac` | Manual RoleBindings |
| Telemetry | On Tenant CR | On MaasTenantConfig CR |
| Multi-tenancy | Manual namespace/config per tenant | Controller-managed via AITenant |
| DB isolation | Shared DB, tenant column | Same (shared DB, tenant column) |
| Payload-processing | Operator-managed | Operator-managed (unchanged) |

### 4.6 Alignment with OSAC Options

| OSAC Option | RHOAI 3.5 Alignment |
|-------------|---------------------|
| **Option 2B** (namespace-per-tenant, shared cluster) | **Partial fit** — namespace isolation is controller-managed, but dedicated gateways add per-tenant overhead. Best for small tenant counts. |
| **Option 3** (cluster-per-tenant) | **Good fit** — each cluster gets one AITenant with a dedicated gateway. ExternalModel bridges shared models. |
| **Spoke-Hub** | **Best fit** — hub has multi-tenant AITenants, spokes are inference-only. Gateway-per-tenant on hub, single gateway on spokes. |

The dedicated-gateway-per-tenant constraint pushes toward **fewer tenants per cluster with larger workloads** rather than many tenants with small workloads. For high-density multi-tenancy, the gateway overhead may become significant.

### 4.7 End-to-End Example: Onboarding NovaTech

A walkthrough of onboarding a fictional tenant **NovaTech** (biotech company) with two personas.

#### CSP Admin (Cloud Service Provider — manages the platform)

**Step 1: Pre-create the Gateway**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: maas-novatech-gateway
  namespace: openshift-ingress
  annotations:
    opendatahub.io/managed: "false"
    security.opendatahub.io/authorino-tls-bootstrap: "true"
spec:
  gatewayClassName: data-science-gateway-class
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: novatech-gateway-tls
      allowedRoutes:
        namespaces:
          from: All
```

Each tenant gets a dedicated network entry point. NovaTech's users hit `novatech-ai.apps.cluster.example.com`, completely separate from other tenants.

**Step 2: Create the AITenant**

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: AITenant
metadata:
  name: novatech
  namespace: ai-tenants
spec:
  gateway:
    name: maas-novatech-gateway
  oidc:
    issuerUrl: https://sso.novatech.io/realms/ai-platform
    clientId: novatech-maas-client
    ttl: 300
```

The controller reconciles and auto-creates:
1. Namespace `ai-tenant-novatech` (with tenant labels)
2. `MaasTenantConfig/default-tenant` in `ai-tenant-novatech` (see below)
3. Role `tenant-admin` in `ai-tenant-novatech` (CRUD on auth policies, subscriptions)
4. Role `novatech-object-admin` in `ai-tenants` (read-only on this AITenant)
5. Gateway claim ConfigMap (prevents other tenants from claiming `maas-novatech-gateway`)
6. Deployment `maas-api-novatech` in `odh-ai-gateway-infra` (per-tenant maas-api with `TENANT_NAME=novatech`)

**Auto-created MaasTenantConfig** (controller-managed, Tenant Admin can patch):

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaasTenantConfig
metadata:
  name: default-tenant                    # CEL-enforced singleton — always this name
  namespace: ai-tenant-novatech
  finalizers:
    - maas.opendatahub.io/tenant-cleanup
spec:
  apiKeys:
    maxExpirationDays: 90                 # max lifetime for API keys (min 1)
  telemetry:
    enabled: true
    metrics:
      captureOrganization: true           # org_id label on metrics
      captureUser: false                  # GDPR-sensitive — opt-in only
      captureGroup: false                 # high-cardinality — opt-in only
      captureModelUsage: true             # model label on metrics
```

Gateway and OIDC are NOT in this CR — they live on the AITenant. MaasTenantConfig only owns runtime config (API keys + telemetry).

**What Tenant Admin can do with MaasTenantConfig:** The `tenant-admin` Role grants `get, update, patch` on `maastenantconfigs` (scoped to resourceName `default-tenant`). NovaTech's admin can:
- Increase/decrease `maxExpirationDays` (e.g., set to 30 for stricter key rotation)
- Enable `captureUser: true` to track per-user token usage on the observability dashboard (requires GDPR consent from users)
- Enable `captureGroup: true` to break down usage by team
- Disable telemetry entirely with `enabled: false`

They **cannot** change the gateway, OIDC config, or create additional MaasTenantConfig CRs (singleton enforced by CEL).

**Step 3: Create RoleBindings (grant access to NovaTech's admin)**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: novatech-admins-binding
  namespace: ai-tenant-novatech
subjects:
  - kind: Group
    name: novatech-platform-admins
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: tenant-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: novatech-object-admin-binding
  namespace: ai-tenants
subjects:
  - kind: Group
    name: novatech-platform-admins
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: novatech-object-admin
  apiGroup: rbac.authorization.k8s.io
```

The controller creates Roles but NOT RoleBindings — the CSP Admin controls who gets tenant-admin access.

---

#### NovaTech Tenant Admin (manages NovaTech's AI access)

From this point, NovaTech's admin (member of `novatech-platform-admins`) takes over.

**Step 4: Deploy a model**

```yaml
apiVersion: serving.kserve.io/v1
kind: LLMInferenceService
metadata:
  name: granite-3-1-8b-fp8
  namespace: ai-tenant-novatech
spec:
  modelName: granite-3-1-8b-fp8
  modelUri: oci://registry.redhat.io/rhoai/granite-3-1-8b-instruct-fp8:latest
  router:
    gateway:
      name: maas-novatech-gateway
      namespace: openshift-ingress
    route: {}
    scheduler: {}
  workerSpec:
    replicas: 1
    resources:
      requests:
        nvidia.com/gpu: "1"
      limits:
        nvidia.com/gpu: "1"
```

KServe deploys the vLLM pod, InferencePool, EPP, and HTTPRoute. The `gateway` field points to NovaTech's dedicated gateway.

**Step 5: Create MaaSModelRef (attach governance)**

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSModelRef
metadata:
  name: granite-8b
  namespace: ai-tenant-novatech
spec:
  modelRef:
    kind: LLMInferenceService
    name: granite-3-1-8b-fp8
```

Governance attachment point — tells MaaS "this model should be governed." Without it, no auth policies or subscriptions can reference the model.

**Step 6: Create MaaSAuthPolicy (Gate 1 — who can access)**

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSAuthPolicy
metadata:
  name: novatech-researchers
  namespace: ai-tenant-novatech
spec:
  modelRefs:
    - name: granite-8b
      namespace: ai-tenant-novatech
  subjects:
    groups:
      - name: novatech-research-team
      - name: novatech-engineering-team
    users:
      - dr.sarah.chen@novatech.io
  meteringMetadata:
    organizationId: novatech-biotech
    costCenter: research-ai-2026
```

Members of `novatech-research-team`, `novatech-engineering-team`, or user `dr.sarah.chen@novatech.io` are authorized. The `meteringMetadata` tags every request for billing. Without this → **401 Unauthorized**.

**Step 7: Create MaaSSubscription (Gate 2 — token budgets)**

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSSubscription
metadata:
  name: novatech-research-tier
  namespace: ai-tenant-novatech
spec:
  owner:
    groups:
      - name: novatech-research-team
  modelRefs:
    - name: granite-8b
      namespace: ai-tenant-novatech
      tokenRateLimits:
        - limit: 500000
          window: 1h
      billingRate:
        perToken: "0.000003"
  tokenMetadata:
    organizationId: novatech-biotech
    costCenter: research-ai-2026
  priority: 10
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSSubscription
metadata:
  name: novatech-engineering-tier
  namespace: ai-tenant-novatech
spec:
  owner:
    groups:
      - name: novatech-engineering-team
  modelRefs:
    - name: granite-8b
      namespace: ai-tenant-novatech
      tokenRateLimits:
        - limit: 100000
          window: 1h
      billingRate:
        perToken: "0.000003"
  tokenMetadata:
    organizationId: novatech-biotech
    costCenter: engineering-ops
  priority: 5
```

Two tiers: research gets 500K tokens/hour, engineering gets 100K. `priority` determines which subscription applies when a user belongs to both groups (higher wins). Without a subscription → **429 Too Many Requests**.

---

#### The Dual-Gate in Action

When a NovaTech researcher calls the model:

```
curl https://novatech-ai.apps.cluster.example.com/v1/chat/completions \
  -H "Authorization: Bearer sk-oai-abc123..."
```

```
Request hits maas-novatech-gateway
    │
    ▼
Gate 1: AuthPolicy (Authorino)
    ├── Validates API key sk-oai-abc123 via maas-api-novatech
    ├── Resolves user → dr.sarah.chen@novatech.io
    ├── Checks: is she in novatech-research-team? YES
    ├── Injects: subscription key, org ID, cost center
    └── Result: PASS (otherwise 401 or 403)
    │
    ▼
Gate 2: TokenRateLimitPolicy (Limitador)
    ├── Matches subscription: novatech-research-tier (priority 10)
    ├── Checks: tokens used this hour < 500,000? YES (used 12,340)
    └── Result: PASS (otherwise 429)
    │
    ▼
Granite 8B responds with completion
```

#### Persona Responsibility Summary

| Resource | CSP Admin | Tenant Admin |
|----------|-----------|-------------|
| Gateway | Creates and manages | Cannot see or modify |
| AITenant | Creates in `ai-tenants` | Can read (via `object-admin` role) |
| RoleBindings | Creates — grants tenant admin access | Cannot create |
| LLMInferenceService | — | Deploys models in tenant namespace |
| MaaSModelRef | — | Creates (governance attachment) |
| MaasTenantConfig | Controller creates; CSP can patch | Can patch (apiKeys, telemetry) |
| MaaSAuthPolicy | — | Creates and manages (who can access) |
| MaaSSubscription | — | Creates and manages (token budgets) |
| API Keys | — | Tenant users self-service via maas-api |

---

## 5. ExternalModel API Evolution

### 5.1 Dual CRD Coexistence

RHOAI 3.5 has **two separate ExternalModel CRDs** representing a major API evolution:

| API Group | CRD | Status |
|-----------|-----|--------|
| `maas.opendatahub.io/v1alpha1` | `ExternalModel` | **Legacy** — simple flat spec |
| `inference.opendatahub.io/v1alpha1` | `ExternalModel` | **Canonical** — rich multi-provider spec |

### 5.2 Legacy ExternalModel (`maas.opendatahub.io`)

Simple flat spec (what we use today):
```yaml
spec:
  provider: openai
  targetModel: granite-3-1-8b-fp8
  endpoint: spoke1-gateway.apps.spoke1.example.com    # FQDN only, no path
  credentialRef:
    name: spoke1-credential
```

### 5.3 Canonical ExternalModel (`inference.opendatahub.io`)

Rich multi-provider spec with the **`path` field** (PR #347):
```yaml
spec:
  modelName: granite-8b                                # Client-facing name (dots/colons allowed)
  externalProviderRefs:                                # 1-64 providers
    - ref:
        name: spoke1-provider                          # References ExternalProvider CR
      targetModel: granite-3-1-8b-fp8
      apiFormat: openai-chat
      path: /granite-serving/granite-3-1-8b-fp8/v1     # NEW — eliminates proxy path prefix
      weight: 50                                        # Weighted random selection
      auth:
        type: simple                                    # Override provider auth
      config:
        region: ca-central-1
    - ref:
        name: spoke2-provider
      targetModel: granite-3-1-8b-fp8
      apiFormat: openai-chat
      path: /granite-serving/granite-3-1-8b-fp8/v1
      weight: 50
```

### 5.4 New `ExternalProvider` CR

Shared provider connection definition referenced by multiple ExternalModel CRs:

```yaml
apiVersion: inference.opendatahub.io/v1alpha1
kind: ExternalProvider
metadata:
  name: spoke1-provider
  namespace: ai-tenant-org-alpha
spec:
  provider: openai
  endpoint: spoke1-gateway.apps.spoke1.example.com
  auth:
    type: simple
    simple:
      secretRef:
        name: spoke1-api-key
  config:
    cluster: spoke1
```

### 5.5 Controller Fallback Logic

The MaaS controller's provider resolver:
1. Tries `inference.opendatahub.io/ExternalModel` first (canonical)
2. Falls back to `maas.opendatahub.io/ExternalModel` (legacy)
3. Uses `status.httpRouteName` from canonical ExternalModel to find the HTTPRoute

Resource naming uses `maas-` prefix on canonical ExternalModel resources to avoid collision with KServe's ExternalModel controller.

### 5.6 Impact on Spoke-Hub Architecture

| Feature | Impact |
|---------|--------|
| **`path` field** | Eliminates proxy path prefix workaround. ExternalModel can specify full path directly. |
| **Multi-provider weights** | Built-in weighted routing across spokes — potential replacement for Hub EPP round-robin |
| **ExternalProvider** | Shared credential/endpoint per spoke — avoids duplicating connection config per model |
| **Dual HTTPRoute** | Path prefix match + header match (`X-Gateway-Model-Name`) — supports both routing patterns |

**When `path` field lands (next build):** Proxy pods only needed for spoke credential injection, not path prefix prepending. Combined with `destination-credential` plugin (upstream PoC branch), proxy pods could be eliminated entirely.

---

## 6. PR Inclusion Status

### 6.1 EA2 Reference Point

The `v3.5.0-ea.2` tag was created on **June 17, 2026 at 10:56 UTC**. Any PR merged after that date is NOT included in EA2.

### 6.2 Full Status Table

| PR | Repo | State | Merged | In EA2? | On stable? | Next Build? |
|----|------|-------|--------|---------|------------|-------------|
| **#347** | ai-gateway-payload-processing | MERGED | Jun 25 | **No** | Yes | **Yes** |
| **#332** | ai-gateway-payload-processing | MERGED | Jul 8 | **No** | Yes | **Yes** |
| **#389** | ai-gateway-payload-processing | MERGED | Jul 9 | **No** | Yes | **Yes** |
| **#393** | ai-gateway-payload-processing | MERGED | Jul 10 | **No** | Yes | **Yes** |
| **#386** | ai-gateway-payload-processing | OPEN | — | **No** | No | **No** |
| **#1858** | llm-d/llm-d-router | MERGED | Jul 16 | **No** | No (needs v0.10.0) | **Pending** |
| **#1919** | llm-d/llm-d-router | MERGED | Jul 16 | **No** | No (needs v0.10.0) | **Pending** |
| **#2045** | llm-d/llm-d-router | OPEN | — | **No** | No | **No** |
| **#3590** | opendatahub-operator | MERGED | Jun 17 14:09 | **No** (3h late) | Yes (main) | **Yes** |
| **#1245** | cluster-ingress-operator | MERGED | Jul 2025 | N/A | OCP 4.20+ | Ships with OCP |
| **#1257** | cluster-ingress-operator | MERGED | Aug 2025 | N/A | OCP 4.20+ | Ships with OCP |

### 6.3 Analysis by Category

**ExternalModel Path Routing (PR #347):**
- Merged June 25, on `stable` — will ship in next RHOAI build
- Once available, eliminates proxy path prefix workaround in spoke-hub architecture
- Proxy pods still needed for credential injection until `destination-credential` plugin lands

**Metering / Token Billing (#332, #389, #393, #386):**
- Streaming metering (#332), chunk recovery (#389), error metering (#393) all merged and on `stable`
- **Tenant attribution (#386) still OPEN** — critical for multi-tenant billing. Without `organization_id` in CloudEvents, hub-side billing cannot attribute costs to tenants
- Metering will work in next build but without tenant attribution

**EPP mTLS (#1858, #1919, #2045):**
- Client mTLS (#1858) and server mTLS (#1919) both merged July 16 to llm-d-router main
- Need a new llm-d-router release (v0.10.0) before RHOAI can pick them up
- Cert hot-reload (#2045) still open — needed for production (cert rotation without restart)
- Our metrics sidecar workaround remains necessary until all three land in a release

**Observability (#3590):**
- COO v1.5 compatibility (#3590) missed EA2 by 3 hours — will be in next build
- We continue using COO v1.4.0 in isolated namespace

### 6.4 Key Takeaway

**Zero RHOAI-specific PRs made the EA2 cut.** The tag was June 17 and every significant PR merged after. However, the outlook is positive: 4 ai-gateway PRs are already on `stable` and will ship in the next build (EA3 or GA). The llm-d mTLS PRs need a release cut. Tenant attribution (#386) is the only truly blocked item.

---

## 7. Impact on OSAC Ansible Role (`ocp_4_20_ai_maas`)

### 7.1 Detailed Change Matrix

| Category | Current (3.4) | 3.5 Change | Action | Priority |
|----------|---------------|------------|--------|----------|
| **Gateway annotations** | No annotations | 2 new annotations required for operator validation | Add annotations to `maas-default-gateway` | High |
| **Tenant CR** | `Tenant/default-tenant` | Deprecated → AITenant + MaasTenantConfig | Rewrite tenant bootstrap (`configure_maas.yaml:345-404`) | High |
| **OIDC** | `Tenant.spec.externalOIDC` | `AITenant.spec.oidc` | Migrate OIDC fields into AITenant | High |
| **Per-tenant gateway** | Single shared `maas-default-gateway` | Dedicated gateway per tenant (webhook-enforced) | Create per-OSAC-Organization gateways | High |
| **Telemetry** | `Tenant.spec.telemetry` | `MaasTenantConfig.spec.telemetry` | Migrate telemetry config | Medium |
| **RBAC** | Not used (manual RoleBindings already) | `spec.rbac` deprecated — manual RoleBindings | No change — already manual | None |
| **Namespace** | `models-as-a-service` (already correct) | Default tenant namespace unchanged | Verify — may already be correct | Low |
| **SM pinning** | `stable-3.2`, Manual approval | RHOAI 3.5 auto-manages SM; Ingress Operator controls Istio version | **Verified: not needed once upstream IPP ext-proc bugs are fixed** (Section 8.7). Auth works on SM 3.4.0 + SM 3.2.7, but requires two manual workarounds (additive NetworkPolicy + EnvoyFilter). Remove ~350 lines of SM pinning only after upstream fixes ship. | Medium |
| **RHCL pre-create** | Sub-operator subs before RHCL | Not needed with `skip_version_pinning: true` (RHCL 1.4.x auto-installs deps) | **Verified: not needed once upstream fixes ship.** RHCL 1.4.x with Automatic approval works for auth but TRLP token counting broken (Section 8.6) and ext-proc bugs require workarounds (Section 8.7). | Medium |
| **ExternalModel** | `maas.opendatahub.io` (legacy) | Canonical `inference.opendatahub.io` with `path` field | Migrate when available — removes proxy path prefix | Low |
| **IPP deployment** | Operator-managed (not in role) | Still operator-managed | **None** — no change needed | None |
| **DSCI** | Patch `spec.monitoring.metrics` only | `spec.serviceMesh` removed (we don't use it) | **None** — monitoring patch unchanged | None |
| **Multi-tenancy flag** | N/A | `--enable-tenant-namespace-discovery` | **None** — enabled by default in upstream manifest | None |

### 7.2 No Version-Conditional Logic Needed

If OSAC ships starting from RHOAI 3.5 (no 3.4 backward compatibility), the role changes are direct replacements — no version detection or conditional branching required.

### 7.3 New CRs to Create (3.5)

1. **`AITenant/{org-name}`** — in `ai-tenants` namespace, with gateway ref + OIDC
2. **`MaasTenantConfig/default-tenant`** — auto-created by AITenant controller, patch for telemetry config if needed
3. **RoleBindings** — manual, per tenant namespace (already our approach in 3.4)
4. **Per-tenant Gateway** — one dedicated gateway per OSAC Organization in `openshift-ingress`

### 7.4 CRs/Config to Replace (3.5)

1. **`Tenant/default-tenant`** → replaced by `AITenant` + `MaasTenantConfig` (controller handles migration for upgrades)
2. **Gateway annotations** — add `opendatahub.io/managed: "false"` + `security.opendatahub.io/authorino-tls-bootstrap: "true"`
3. **Sub-operator pre-creation** — verify if still needed with RHCL 1.4.x dual-anchor support

---

## 8. Known Limitations — Multi-Tenant Isolation with Shared OIDC

### 8.1 The `tenant-gateway-isolation` Stub

The MaaS AuthPolicy controller generates a `tenant-gateway-isolation` authorization rule that is **explicitly a stub** — it always returns `allow { true }` with no tenant checking:

```go
tenantGatewayIsolationRule := map[string]any{
    "priority": int64(0),
    "metrics":  false,
    "opa": map[string]any{
        "rego": `# Tenant hostname isolation stub.
# Replace with a real maas-api call to validate that the API key's tenant
# matches the gateway hostname (prevents Coke key on Pepsi gateway).
allow { true }`,
    },
}
```

**Source code:**
- Stub definition: [`maasauthpolicy_controller.go#L826-L838`](https://github.com/opendatahub-io/models-as-a-service/blob/main/maas-controller/pkg/controller/maas/maasauthpolicy_controller.go#L826-L838)
- Wired into authorization rules: [`maasauthpolicy_controller.go#L903`](https://github.com/opendatahub-io/models-as-a-service/blob/main/maas-controller/pkg/controller/maas/maasauthpolicy_controller.go#L903)
- Documented as known limitation: [`release-notes/index.md#L53`](https://github.com/opendatahub-io/models-as-a-service/blob/main/docs/content/release-notes/index.md#L53)

### 8.2 Impact on OSAC Multi-Tenancy

When multiple OSAC Organizations share a single Keycloak realm (`osac`), users from different Organizations can have the same realm-level group names (e.g., `tier-free-users`). Since MaaS only checks the flat `groups` JWT claim and does NOT check the `organization` claim, a user from Organization `abc` could potentially access Organization `def`'s models if they discover `def`'s gateway URL.

```
abc user JWT:                          def user JWT:
{                                      {
  "organization": {"abc": {...}},        "organization": {"def": {...}},
  "groups": ["tier-free-users"],         "groups": ["tier-free-users"],
  "azp": "osac-cli"                      "azp": "osac-cli"
}                                      }

abc user hits def's gateway:
  ✓ JWT signature valid (same issuer)
  ✓ azp matches (same client)
  ✓ groups match AuthPolicy (same group name)
  ✓ tenant-gateway-isolation: allow { true }  ← STUB
  → ACCESS GRANTED (should be denied)
```

### 8.3 What IS Isolated — API Keys (Data Plane)

For model inference (the hot path), tenants use **API keys** (`sk-oai-*`), not raw JWTs:

| Access Method | Cross-Tenant Safe? | Why |
|---|---|---|
| **API keys** | **Yes** | Each tenant has its own `maas-api-{tenant}` instance. An API key created on `abc`'s maas-api validates only on `abc`'s maas-api. `def`'s maas-api rejects it — different database scope (`WHERE tenant = 'abc'`). |
| **OIDC JWT** (shared issuer) | **No** | Same issuer → same JWKS → signature valid on all gateways. Same `groups` claim → AuthPolicy matches. `organization` claim not checked. |
| **OIDC JWT** (separate issuer) | **Yes** | Different JWKS → signature fails on wrong gateway. |

The practical flow for OSAC tenants:
1. User authenticates via `osac login` → gets JWT with `organization` claim
2. User creates an **API key** via their tenant's maas-api using that JWT
3. User calls models with API key — not the raw JWT
4. API key validates only on the correct tenant's maas-api → **isolated**

### 8.4 The Remaining Gap — Management Plane

The gap is on the **management plane**: can a user from `abc` create an API key on `def`'s gateway using their JWT? With the stub, yes. This is limited scope (key creation, not inference) but needs to be closed.

The `tenantName` variable is already available in `buildGatewayAuthPolicySpec()` — the plumbing exists. An OPA rule checking `input.auth.identity.organization` against the expected tenant name would close this gap.

### 8.5 Recommendation

- **Phase 1**: Accept API key isolation as sufficient for model inference. Document that OIDC JWT cross-tenant access is a known upstream limitation.
- **Phase 2**: Track upstream fix for `tenant-gateway-isolation` — the MaaS team has already identified this as work to be done (per the code comment and release notes).
- **Raise with MaaS team**: The `organization` claim in OSAC's JWT contains the tenant identity. When the stub is replaced, it should check this claim — not just the API key's tenant.

### 8.6 Token-Based Rate Limiting (TRLP) Not Functional in EA2

**Validated on**: osacmaas cluster, OCP 4.21, RHOAI 3.5.0-ea.2, RHCL 1.4.1 (Kuadrant operator v1.5.0)

RHOAI 3.5 changes the `TokenRateLimitPolicy` from **request-based** counting (3.4) to **token-based** counting. The Wasm shim uses a two-phase mechanism:

1. **Request phase** (`ratelimit-check-service`): Checks remaining budget with `hits_addend: 0` — does NOT consume tokens
2. **Response phase** (`ratelimit-report-service`): Reports actual tokens consumed via `hits_addend: responseBodyJSON("/usage/total_tokens")` — reads `usage.total_tokens` from the model's response body

**Finding**: The response-phase token reporting **does not work** in RHOAI 3.5 EA2. The Kuadrant Wasm shim is loaded and active (`wasm.kuadrant-wasm-shim` stats present, auth denials recorded in `wasmcustom.kuadrant.denied`), but Limitador receives zero token reports. All requests return HTTP 200 regardless of token budget — even with a 1 token/min limit and responses containing 3-5 tokens each.

**Evidence**:
- Wasm shim loaded: `wasm.envoy.wasm.runtime.v8.active: 4`, `wasmcustom.kuadrant.allowed: 80`, `wasmcustom.kuadrant.denied: 6`
- Auth enforcement works: HTTP 401 for unauthenticated requests (Authorino via `ext_authz`)
- TRLP status shows `Accepted: True`, `Enforced: True`
- Limitador logs show zero counter activity beyond startup
- Tested with token_limit: 100, 10, and 1 — all requests return 200

**Impact**:
- Auth enforcement (Gate 1): **Works** — Authorino validates API keys, JWTs, and groups correctly
- Token rate limiting (Gate 2): **Does not work** — all requests pass regardless of budget
- The dual-gate is degraded to single-gate (auth only, no rate limiting)

**Root cause hypothesis**: The Wasm shim's response-body parsing (`responseBodyJSON("/usage/total_tokens")`) may not be functional in RHCL 1.4.1 / Kuadrant v1.5.0 with Istio 1.27.3 (OCP 4.21). This is an EA2-specific issue that may be fixed in GA.

**Comparison with RHOAI 3.4**: In 3.4 with RHCL 1.3.x, rate limiting was request-based (1 count per request, not per token). This worked reliably. The transition to token-based counting is a 3.5 architectural change.

**Recommendation**: Track as an EA2 limitation. Verify again when RHOAI 3.5 GA drops (August 20). If token-based TRLP doesn't work in GA, file a bug against RHCL/Kuadrant.

### 8.7 RHCL 1.3.x on RHOAI 3.5 EA2 — IPP ext-proc Bugs and Authorino CEL Bug

**Validated on**: tenantai (OCP 4.20.26) and osacmaas (OCP 4.21.22), RHOAI 3.5.0-ea.2, RHCL 1.3.5 (Authorino 1.3.2, Limitador 1.3.1, SM 3.2.7)

RHCL 1.3.x version pinning (`skip_version_pinning: false`) with RHOAI 3.5 EA2 causes HTTP 500 on API key creation and model inference. The original investigation attributed this to the Authorino CEL metrics bug alone. Subsequent testing on both OCP 4.20 and OCP 4.21 revealed the HTTP 500 is caused by **three IPP ext-proc bugs** in the per-tenant gateway configuration. The Authorino CEL bug is also present but is non-fatal — it is logged as an error but does not cause HTTP 500. After applying the ext-proc fixes, RHCL 1.3.x works with RHOAI 3.5 EA2 (API key creation 201, inference 200).

Full details: [`rhoai3.5 EA2_4.20.md`](rhoai3.5%20EA2_4.20.md)

#### IPP ext-proc Bugs (root cause of HTTP 500)

RHOAI 3.5 introduces IPP (Intelligent Payload Processing) — two ext-proc filter stages on the gateway: `ipp-pre` (request body parsing, `failure_mode_allow: true`) and `ipp` (response processing/metering, `failure_mode_allow: false`). The `ipp` filter's strict failure mode means any ext-proc connection failure becomes HTTP 500.

**Bug 1 — NetworkPolicy blocks `maas-default-gateway`**: The `ModelsAsService` controller creates a NetworkPolicy `payload-processing` that only allows ingress from pods with label `gateway-name: data-science-gateway`. The per-tenant `maas-default-gateway` pod has label `gateway-name: maas-default-gateway` — blocked. Owner: `ModelsAsService/default-modelsasservice`.

**Bug 2 — DestinationRule adds TLS to plain gRPC**: The `Config` controller creates DestinationRules for `payload-processing` and `payload-pre-processing` with `tls.mode: SIMPLE`. The ext-proc pods serve plain gRPC on port 9004 — no TLS. Owner: `Config/default`.

**Bug 3 — `ipp` filter not disabled for maas-api routes**: The controller's EnvoyFilter disables `ipp-pre` for maas-api routes but does NOT disable `ipp`. The `ipp` filter has no purpose on `/maas-api/v1/api-keys`. Owner: `Config/default`.

#### Fixes Applied (controller-proof)

Both controllers (`ModelsAsService`, `Config`) continuously reconcile their managed resources — patches or deletions are reverted within seconds.

**Fix 1 — Additive NetworkPolicy**: Create `payload-processing-maas-gateway` NetworkPolicy (separate resource, not managed by any controller) allowing `maas-default-gateway` pods to reach ext-proc on port 9004. Kubernetes NetworkPolicies are union-based — the additive policy works alongside the controller-managed one.

**Fix 2 — EnvoyFilter to disable `ipp` for maas-api routes**: Create `disable-ipp-for-maas-api` EnvoyFilter that disables `envoy.filters.http.ext_proc.ipp` on `maas-api-route.0` and `maas-api-route.1`. Same pattern the controller already uses for `ipp-pre` on those routes. Bypasses both the TLS mismatch (Bug 2) and the missing route disable (Bug 3). The `ipp` filter remains active on model inference routes.

#### Authorino 1.3.2 CEL Metrics Bug (non-fatal)

RHOAI 3.5 AuthPolicy includes `response.success.filters.identity` with `metrics: true`. Authorino 1.3.2's `metricLabels` function (`auth_pipeline.go:538`) serializes compiled CEL programs to Go struct strings and attempts to re-parse them as CEL, which fails:

```
{"level":"error","msg":"failed to parse CEL expression",
 "expression":"Expression { expression: IdedExpr { id: 3, expr: Select(SelectExpr { ... }) } }",
 "error":"ERROR: <input>:1:12: undeclared reference to 'Expression' (in container '')"}
```

Three expressions fail: `auth.identity.group`, `auth.identity.userid`, `responseBodyJSON("/model")`. The error is logged but does not cause HTTP 500 — `metricLabels` runs after the auth decision is made and the error is caught without failing the auth response. Fixed in Authorino 1.4.x (RHCL 1.4.x). RHOAI 3.4 was unaffected because its AuthPolicy used `metrics: false`.

#### Verified Results (after ext-proc fixes, RHCL 1.3.x)

| Test | tenantai (OCP 4.20) | osacmaas (OCP 4.21) |
|------|---------------------|---------------------|
| API key creation | 201 | 201 |
| Inference with API key | 200 | 200 |
| Unauthenticated request | 401 | 401 |
| IPP on model inference routes | Active | Active |
| Authorino CEL error in logs | Present, non-fatal | Present, non-fatal |

**RHCL version requirements** ([RHCL Supported Configurations](https://access.redhat.com/articles/7092611)):
- RHCL 1.3 GA: Service Mesh 3.2, cert-manager 1.18, RHBK 26.4
- RHCL 1.4 GA: Service Mesh 3.3, cert-manager 1.19, RHBK 26.6

**Impact — dual-gate status on RHOAI 3.5 EA2**:
| RHCL Version | Auth (Gate 1) | Rate Limiting (Gate 2) | Result |
|---|---|---|---|
| RHCL 1.3.x + ext-proc fixes | Works (401/201/200) | Broken (Section 8.6 — TRLP token counting) | **Single-gate only** |
| RHCL 1.4.x | Works (401/200) | Broken (Section 8.6 — TRLP token counting) | **Single-gate only** |

Neither RHCL version achieves full dual-gate on RHOAI 3.5 EA2. Track for GA verification (August 20).

**Recommendation**: Either RHCL version works for auth on RHOAI 3.5 EA2. RHCL 1.3.x requires the two ext-proc workarounds (NetworkPolicy + EnvoyFilter). RHCL 1.4.x avoids the workarounds and eliminates the CEL log errors. Both are single-gate only (auth without token-based rate limiting) on EA2.

---

## 9. Recommendations & Next Steps

### 9.1 Immediate (This Sprint)

| # | Action | Owner | Status | Notes |
|---|--------|-------|--------|-------|
| 1 | **Deploy RHOAI 3.5 EA2 on test cluster** | OSAC MaaS | **Done** (July 20-21) | Deployed on osacmaas (OCP 4.21) and tenantai (OCP 4.20), `beta` channel. Gateway Programmed, IPP auto-deployed, `modelsAsService: Managed` confirmed. See Section 8.7 — IPP ext-proc bugs require two workarounds (NetworkPolicy + EnvoyFilter). Both RHCL 1.3.x and 1.4.x work for auth after fixes. |
| 2 | **Test RHCL compatibility** | OSAC MaaS | **Done** (July 20-21) | RHCL 1.3.x works after ext-proc fixes (CEL error logged but non-fatal). RHCL 1.4.x also works (no workarounds needed). Token-based TRLP does NOT work in EA2 with either version — see Section 8.6. Single-gate (auth only) on EA2. Track for GA. |
| 3 | **Verify Tenant → AITenant migration** | OSAC MaaS | **Partial** | Auto-created `ai-tenants` namespace confirmed. `MaasTenantConfig` CRD not present in EA2. Legacy `Tenant` CR accepted and migrated (Active). AITenant CRD exists but no auto-created AITenant CR found — controller may only auto-create on upgrade, not fresh install. Full migration testing deferred to GA. |

### 9.2 TODO: Intelligent Scheduling PRs and RFEs

> **TODO:** Add/consolidate all PRs and RFEs for Intelligent Scheduling. The requirements are evolving as late as last week — consolidate once stable.

---

## Appendix A: Key Source Files

### opendatahub-operator (odh-3.5.0-ea.2)

| File | Purpose |
|------|---------|
| `api/services/v1alpha1/gateway_types.go` | GatewayConfig CRD types |
| `internal/controller/services/gateway/gateway.go` | Gateway controller |
| `internal/controller/services/gateway/gateway_support.go` | Gateway defaults, resource creation |
| `api/components/v1alpha1/modelsasservice_types.go` | ModelsAsService CR types |
| `internal/controller/components/modelsasservice/modelsasservice.go` | MaaS component controller |
| `internal/controller/components/modelsasservice/modelsasservice_support.go` | MaaS defaults, IPP deployment |
| `api/dscinitialization/v2/dscinitialization_types.rhoai.go` | DSCI v2 (serviceMesh removed) |
| `pkg/upgrade/upgrade.go` | Upgrade migration (GatewayConfig IngressMode) |

### models-as-a-service (main branch)

| File | Purpose |
|------|---------|
| `maas-controller/api/maas/v1alpha1/aitenant_types.go` | AITenant CRD types |
| `maas-controller/api/maas/v1alpha1/maastenantconfig_types.go` | MaasTenantConfig CRD types |
| `maas-controller/api/maas/v1alpha1/tenant_types.go` | Legacy Tenant (deprecated) |
| `maas-controller/pkg/controller/maas/aitenant_controller.go` | AITenant reconciler (~1340 lines) |
| `maas-controller/pkg/controller/maas/maasauthpolicy_controller.go` | AuthPolicy generation — contains `tenant-gateway-isolation` stub ([L826-838](https://github.com/opendatahub-io/models-as-a-service/blob/main/maas-controller/pkg/controller/maas/maasauthpolicy_controller.go#L826-L838)) |
| `maas-controller/pkg/controller/maas/tenant_controller.go` | MaasTenantConfig reconciler |
| `maas-controller/pkg/platform/tenantreconcile/constants.go` | Labels, naming, resource names |
| `maas-controller/pkg/platform/tenantreconcile/platform_context.go` | Gateway/OIDC resolution |
| `maas-controller/pkg/platform/tenantreconcile/params.go` | Per-tenant maas-api patching |
| `maas-controller/pkg/webhook/aitenant_webhook.go` | Gateway uniqueness validation |
| `maas-controller/cmd/manager/main.go` | Self-bootstrap logic |
| `docs/content/install/multi-tenant-setup.md` | Multi-tenant setup guide |

### ai-gateway-payload-processing (main branch)

| File | Purpose |
|------|---------|
| `api/inference/v1alpha1/externalmodel_types.go` | Canonical ExternalModel CRD |
| `api/inference/v1alpha1/externalprovider_types.go` | ExternalProvider CRD |
| `deployment/base/payload-processing/instance/deployment.yaml` | IPP Deployment manifest |
| `deployment/base/payload-processing/manager/envoy-filter.yaml` | Dual-anchor EnvoyFilter |

## Appendix B: DSC Configuration (3.5)

```yaml
apiVersion: datasciencecluster.opendatahub.io/v2
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    dashboard:
      managementState: Managed
    kserve:
      managementState: Managed
      modelsAsService:
        managementState: Managed    # Explicitly enable MaaS
```

The `ModelsAsService` sub-component has its own management state (default: `Removed`). It must be explicitly set to `Managed`. The handler checks both KServe and MaaS management states before enabling.

## Appendix C: ModelsAsService CR (3.5)

```yaml
apiVersion: components.platform.opendatahub.io/v1alpha1
kind: ModelsAsService
metadata:
  name: default-modelsasservice
spec:
  gatewayRef:
    namespace: openshift-ingress
    name: maas-default-gateway
  apiKeys:
    maxExpirationDays: 90
  externalOIDC:
    issuerUrl: "https://keycloak.apps.example.com/realms/maas"
    clientId: "maas-client"
    ttl: 300
  telemetry:
    enabled: true
    metrics:
      captureOrganization: true
      captureUser: false           # Privacy-sensitive — opt-in
      captureGroup: false
      captureModelUsage: true
```
