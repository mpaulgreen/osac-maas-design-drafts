# External Model Architecture — Three Paths Compared

*Three validated approaches for cross-cluster model access from a MaaS tenant. Each differs in upstream protection but all enforce tenant-side governance (auth + rate limiting). All validated on live clusters July 3, 2026.*

## Overview

```
                    Upstream Protection
                    ───────────────────►
         None              K8s RBAC            MaaS Pass-through
    ┌──────────┐      ┌──────────────┐      ┌──────────────────┐
    │  Path A  │      │   Path C     │      │     Path B       │
    │  Manual  │      │  openshift-  │      │ maas-default-    │
    │  Route   │      │ ai-inference │      │    gateway       │
    │ (open)   │      │ (SA token)   │      │ (API key)        │
    └──────────┘      └──────────────┘      └──────────────────┘
         │                   │                       │
         └───────────────────┴───────────────────────┘
                             │
                    Tenant MaaS Gateway
                    Auth ✅  Rate Limit ✅
                    (all 3 paths enforce)
```

**Key insight:** Tenant-side MaaS governance (AuthPolicy + TokenRateLimitPolicy) is enforced **before** the request leaves the tenant cluster — regardless of which path is used. The 3 paths differ only in how the upstream is protected.

---

## 1. Architecture Diagrams

### Path A: Manual Route (No Upstream Protection)

```
Tenant Cluster                             Upstream Cluster
┌──────────────────────────────┐   ┌──────────────────────────────┐
│  User → MaaS Gateway          │   │                              │
│    ├── AuthPolicy ✅          │   │  openshift-ai-inference       │
│    ├── TRLP ✅                │   │    (bypassed)                │
│    └── ExternalModel          │   │                              │
│         └── Direct HTTPS ─────────▶ Manual Route (reencrypt)     │
│            (dummy credential) │   │   → workload Service:8000   │
│                               │   │     → vLLM (GPU)            │
└──────────────────────────────┘   └──────────────────────────────┘
```

- **No proxy** — ExternalModel points directly to Route FQDN
- **No upstream auth** — Route is publicly accessible
- **Dummy credential** — MaaS requires non-empty `api-key` but upstream doesn't check it

### Path B: MaaS Gateway (MaaS Pass-through)

```
Tenant Cluster                             Upstream Cluster
┌──────────────────────────────┐   ┌──────────────────────────────┐
│  User → MaaS Gateway          │   │  maas-default-gateway         │
│    ├── AuthPolicy ✅          │   │    ├── AuthPolicy (pass-thru) │
│    ├── TRLP ✅                │   │    ├── TRLP (999999999/min)   │
│    └── ExternalModel          │   │    └── LLMInferenceService    │
│         │                    │   └──────────────────────────────┘
│         ▼                    │              ▲
│  maas-proxy Pod              │    HTTPS     │
│    └── /<ns>/<model>/v1/...  │──────────────┘
│         (MaaS API key)       │
└──────────────────────────────┘
```

- **Proxy required** — adds `/<namespace>/<model>/` path prefix
- **MaaS pass-through auth** — `system:authenticated` + 999999999 tokens/min
- **Real MaaS API key** — created on upstream, shared with tenant

### Path C: Inference Gateway (K8s RBAC)

```
Tenant Cluster                             Upstream Cluster
┌──────────────────────────────┐   ┌──────────────────────────────┐
│  User → MaaS Gateway          │   │  openshift-ai-inference       │
│    ├── AuthPolicy ✅          │   │    ├── K8s RBAC AuthPolicy    │
│    ├── TRLP ✅                │   │    └── KServe HTTPRoute       │
│    └── ExternalModel          │   │         ├── URLRewrite        │
│         │                    │   │         └── workload Service   │
│         ▼                    │   │              → vLLM (GPU)     │
│  maas-proxy Pod              │   └──────────────────────────────┘
│    └── /<ns>/<model>/v1/...  │              ▲
│         (K8s SA token)       │──────────────┘
└──────────────────────────────┘        (NLB + SA token)
```

- **Proxy required** — adds `/<namespace>/<model>/` path prefix
- **K8s RBAC auth** — `kubernetesTokenReview` validates SA token
- **SA token credential** — ServiceAccount token with `get` on `llminferenceservices`

---

## 2. Request Flows

### Path A Flow

```
1. User → https://maas.<tenant>/external-models/granite-external-route/v1/chat/completions
2. Tenant MaaS Gateway (Envoy)
   → Kuadrant AuthPolicy: validates tenant API key
   → Kuadrant TRLP: checks token budget
   → HTTPRoute matches → IPP strips path → /v1/chat/completions
   → ExternalName Service → granite-3-1-8b-fp8.apps.upstream.com
3. OpenShift Router (upstream)
   → Route (reencrypt) → KServe workload Service:8000
4. vLLM serves request
```

### Path B Flow

```
1. User → https://maas.<tenant>/external-models/granite-external-maas/v1/chat/completions
2. Tenant MaaS Gateway (Envoy)
   → Kuadrant AuthPolicy: validates tenant API key
   → Kuadrant TRLP: checks token budget
   → HTTPRoute matches → IPP strips path → /v1/chat/completions
   → ExternalName Service → granite-serving--granite-3-1-8b-fp8.maas-proxies.svc
3. maas-proxy
   → parseHost() → ns=granite-serving, model=granite-3-1-8b-fp8
   → Prepends path → /granite-serving/granite-3-1-8b-fp8/v1/chat/completions
   → Injects MaaS API key from Secret
   → Forwards to https://maas.<upstream>/granite-serving/granite-3-1-8b-fp8/v1/...
4. Upstream MaaS Gateway
   → AuthPolicy: system:authenticated → pass
   → TRLP: 999999999/min → pass
   → IPP strips path → /v1/chat/completions
   → Routes to InferencePool → vLLM
```

### Path C Flow

```
1. User → https://maas.<tenant>/external-models/granite-external-oai/v1/chat/completions
2. Tenant MaaS Gateway (Envoy)
   → Kuadrant AuthPolicy: validates tenant API key
   → Kuadrant TRLP: checks token budget
   → HTTPRoute matches → IPP strips path → /v1/chat/completions
   → ExternalName Service → granite-serving--granite-3-1-8b-fp8.maas-proxies.svc
3. maas-proxy
   → parseHost() → ns=granite-serving, model=granite-3-1-8b-fp8
   → Prepends path → /granite-serving/granite-3-1-8b-fp8/v1/chat/completions
   → Injects SA token from Secret
   → Forwards to https://<NLB>/granite-serving/granite-3-1-8b-fp8/v1/...
4. openshift-ai-inference Gateway (Envoy)
   → AuthPolicy: kubernetesTokenReview → validates SA token → pass
   → KServe HTTPRoute matches → URLRewrite strips prefix → /v1/chat/completions
   → Routes to workload Service:8000 → vLLM
```

---

## 3. Governance Comparison

### Where Governance Is Enforced

| Governance | Path A | Path B | Path C |
|-----------|--------|--------|--------|
| **Tenant auth** (MaaS API key) | ✅ Tenant gateway | ✅ Tenant gateway | ✅ Tenant gateway |
| **Tenant rate limiting** (TRLP) | ✅ Tenant gateway | ✅ Tenant gateway | ✅ Tenant gateway |
| **Upstream auth** | ❌ None | ✅ MaaS pass-through | ✅ K8s RBAC |
| **Upstream rate limiting** | ❌ None | ✅ Pass-through (unlimited) | ❌ None |

### Upstream CRs Required

| CR | Path A | Path B | Path C |
|----|--------|--------|--------|
| LLMInferenceService | ✅ (`openshift-ai-inference`) | ✅ (`maas-default-gateway`) | ✅ (`openshift-ai-inference`) |
| MaaSModelRef | ❌ | ✅ | ❌ |
| MaaSAuthPolicy | ❌ | ✅ (`system:authenticated`) | ❌ (auto-created by odh-model-controller) |
| MaaSSubscription | ❌ | ✅ (999999999/min) | ❌ |
| Manual Route | ✅ (reencrypt) | ❌ | ❌ |
| RoleBinding | ❌ | ❌ | ✅ (SA → view) |

### Tenant CRs (Same for All Paths)

| CR | Namespace |
|----|-----------|
| ExternalModel | `external-models` |
| MaaSModelRef | `external-models` |
| MaaSAuthPolicy | `models-as-a-service` |
| MaaSSubscription | `models-as-a-service` |
| Secret (credential) | `external-models` |

---

## 4. Gateway Differences

### Why Each Path Works Differently

| Aspect | `maas-default-gateway` (Path B) | `openshift-ai-inference` (Path C) | Manual Route (Path A) |
|--------|-------------------------------|----------------------------------|----------------------|
| **Hostname** | `maas.<domain>` (fixed) | NLB (infra-assigned) | `<model>.apps.<domain>` |
| **Protocol** | HTTPS/443 | HTTPS/443 (our cluster) | HTTPS/443 (reencrypt) |
| **Auth** | Kuadrant AuthPolicy + TRLP (default-deny) | K8s RBAC (`kubernetesTokenReview`) | None |
| **Path handling** | IPP ext-proc strips prefix (no URLRewrite) | KServe URLRewrite in HTTPRoute | N/A (direct `/v1/...`) |
| **ExternalModel support** | Reconciler creates HTTPRoutes here | Never targeted by reconciler | Not a gateway |
| **Created by** | MaaS deployment | Our Ansible template (RHOAIENG-38214) | Manual `oc apply` |

### Path Prefix Handling

| Gateway | How prefix is stripped | Who creates the HTTPRoute |
|---------|----------------------|--------------------------|
| `maas-default-gateway` | IPP ext-proc (gateway-level) | ExternalModel reconciler (MaaS controller) |
| `openshift-ai-inference` | URLRewrite filter in HTTPRoute | KServe LLMInferenceService controller |
| Manual Route | No prefix — direct `/v1/...` | N/A |

---

## 5. Security Comparison

### Upstream Protection Levels

```
Path A (Manual Route)         Path C (openshift-ai-inference)     Path B (MaaS Gateway)
    OPEN                           K8s RBAC                         MaaS API Key
     │                              │                                │
     │  Anyone with URL             │  Valid SA token                 │  Valid MaaS API key
     │  can access                  │  with RBAC perms               │  + pass-through sub
     │                              │  required                      │  required
     ▼                              ▼                                ▼
  No protection               Medium protection              Strongest protection
```

| Threat | Path A | Path B | Path C |
|--------|--------|--------|--------|
| **Direct bypass** (attacker knows upstream URL) | ❌ Model exposed | ✅ Needs MaaS API key | ✅ Needs SA token |
| **Credential type** | None needed | `sk-oai-*` (long-lived, revocable) | K8s SA token (time-limited) |
| **Credential rotation** | N/A | Delete + recreate API key | `oc create token` with `--duration` |
| **Network-level bypass** | Route is public | NLB may be public | NLB may be public |

### Recommendation by Environment

| Environment | Recommended Path | Why |
|-------------|-----------------|-----|
| **Same VPC / private network** | Path A (Manual Route) | Simplest, no proxy. Network isolation provides protection. |
| **Cross-VPC / same cloud** | Path C (Inference Gateway) | K8s RBAC auth without requiring MaaS on upstream. SA token is time-limited. |
| **Cross-datacenter / public** | Path B (MaaS Gateway) | Strongest auth. API key is revocable. Pass-through CRs are explicit. |
| **Multi-tenant upstream** | Path B (MaaS Gateway) | Per-tenant API keys + per-tenant pass-through subs. MaaS tracks usage. |

---

## 6. Validated Test Results 
All tests run on live clusters: tenant cluster → upstream cluster.

| Test | Path A | Path B | Path C |
|------|--------|--------|--------|
| **Upstream direct (no tenant)** | 200 (open) | 403 (needs API key) | 401 (needs SA token) |
| **Tenant unauthenticated** | 401 ✅ | 401 ✅ | 401 ✅ |
| **Tenant invalid key** | 403 ✅ | 403 ✅ | 403 ✅ |
| **Tenant valid inference** | 200 ✅ | 200 ✅ | 200 ✅ |
| **Tenant rate limiting** | 429 (req 9) ✅ | 429 (req 9) ✅ | 429 (req 9) ✅ |

**All 3 paths enforce tenant MaaS governance identically.** The only difference is upstream-side protection.

---

## 7. Decision Matrix

```
                        Need upstream protection?
                        ┌─────────┐
                        │   No    │──────► Path A (Manual Route)
                        │         │        Simplest. No proxy. Private VPC.
                        └────┬────┘
                             │ Yes
                             ▼
                    Upstream has MaaS deployed?
                    ┌─────────┐
                    │   No    │──────► Path C (openshift-ai-inference)
                    │         │        K8s RBAC auth. No MaaS needed on upstream.
                    └────┬────┘
                         │ Yes
                         ▼
                    Path B (maas-default-gateway)
                    MaaS API key auth. Pass-through CRs.
                    Best for multi-tenant upstream.
```

---

## 8. Component Inventory

### Proxy (Paths B and C only)

| Component | Detail |
|-----------|--------|
| Image | `quay.io/mpaulgreen/maas-proxy:latest` (private, needs pull secret) |
| Function | Parses `<ns>--<model>` from Service DNS name, prepends path prefix |
| One per upstream | Multiple models from same upstream share one Deployment |
| Source | Built from `gitlab.cee.redhat.com/mavazque/ai-grid` |

### DNS Routing Convention

```
Service name:  granite-serving--granite-3-1-8b-fp8.maas-proxies.svc.cluster.local
               └── upstream ns ─┘  └── model name ──┘ └── proxy namespace ───────┘

Proxy parses:  namespace=granite-serving, model=granite-3-1-8b-fp8
Prepends:      /granite-serving/granite-3-1-8b-fp8/v1/chat/completions
```

### Required Labels (All Paths)

```yaml
# external-models Namespace
opendatahub.io/generated-namespace: "true"
maas.opendatahub.io/gateway-access: "true"

# Credential Secret
inference.networking.k8s.io/bbr-managed: "true"
```

