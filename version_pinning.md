# OLM Version Pinning Strategy

How the `ocp_4_20_ai_maas` Ansible role pins operator versions using OLM Manual approval, `startingCSV`, and install plan filtering — and what happens on degraded clusters.

## 1. Why Version Pinning Exists

RHCL v1.4.0's Kuadrant Wasm shim sets `allow_on_headers_stop_iteration: true` in Envoy's PluginConfig. This field doesn't exist in SM 3.2 (Istio 1.26) or SM 3.3 (Istio 1.27) — their Envoy protos predate it. Without it, the Wasm shim can't pause requests while waiting for Authorino's gRPC auth response. Envoy forwards the request immediately, the auth response arrives too late, and auth enforcement silently fails.

The failure is auth-method-agnostic (OIDC and K8s TokenReview both fail) and produces no errors — just `kuadrant.denied: 0` in metrics.

**Resolution**: Pin the entire RHCL stack (RHCL, Authorino, Limitador, DNS) to v1.3.x and SM to 3.2.x. Additionally, COO is pinned to v1.4.0 because v1.5.0 breaks RHOAI 3.4 observability (CEL validation + Perses image flag mismatch).

**Reference**: [Red Hat Solutions 7097633](https://access.redhat.com/solutions/7097633), [Supported Config Matrix](https://access.redhat.com/articles/7092611)

## 2. Pinned Versions

| Operator | startingCSV | Pinned Range | Channel | Approval | Namespace |
|----------|------------|-------------|---------|----------|-----------|
| Service Mesh 3 | v3.2.7 | v3.2.x | `stable-3.2` | Manual | `openshift-operators` |
| Authorino | v1.3.2 | v1.3.x | `stable` | Manual | `openshift-operators` |
| Limitador | v1.3.1 | v1.3.x | `stable` | Manual | `openshift-operators` |
| DNS | v1.3.1 | v1.3.x | `stable` | Manual | `openshift-operators` |
| RHCL | v1.3.5 | v1.3.x | `stable` | Manual | `openshift-operators` |
| COO | v1.4.0 | v1.4.x | `stable` | Manual | `openshift-cluster-observability-operator` |

**Not pinned** (Automatic approval): cert-manager (`stable-v1.18`), LWS (`stable-v1.0`), RHOAI (`stable-3.4`), OTel (`stable`).

**Version-range checks**: Verification uses `accept_version` (`v1.3`) pattern matching instead of exact CSV names. OLM may auto-upgrade within v1.3.x (e.g., v1.3.4→v1.3.5) — the `startingCSV` is the minimum version, and the reject filter (`v1.4.0`) blocks major upgrades. Checks match any `operator-name.*v1.3` CSV as Succeeded.

**Key constraint**: SM `startingCSV` must match the channel head. When `startingCSV` is behind the channel head, OLM installs it but sets subscription state to `UpgradePending` — never sets `installedCSV`, causing `ConstraintsNotSatisfiable` for all other operators in the namespace.

## 3. Service Mesh Pinning

SM pinning is the most complex due to the **Ingress Operator interference**. The OCP Ingress Operator's `gatewayclass_controller` **actively watches and overwrites** the SM subscription on every reconciliation (~3s): `channel: stable-3.2 → stable`, `startingCSV: v3.2.7 → v3.1.0`. This is not a one-time creation — it's a continuous loop triggered by GatewayClass events. CVO restores the Ingress Operator if scaled down, making separate scale-down/wait/create tasks unreliable.

### Solution: Atomic shell task with inner race loop

All SM steps are merged into a **single shell task** that races the Ingress Operator. The key insight: once the SM CSV is `Succeeded`, the Ingress Operator can overwrite the subscription freely — the CSV is already installed and OLM won't downgrade. We only need to win the race ONCE.

**Prerequisite**: Before the SM task runs, a global cleanup task deletes stale subscriptions, CSVs, and install plans for sub-operators and RHCL in `openshift-operators` (see Section 10, Step 1b). Cleanup uses **version-range matching**: if all 4 operators have a v1.3.x CSV `Succeeded`, cleanup is skipped — preserving correctly installed operators on re-runs (even after OLM auto-upgrades within v1.3.x). Install plans are deleted by operator name (not `--all`) to preserve SM install plans.

```
Single shell task ("Install SM with Ingress Operator race protection"):

1. Early exit if SM CSV already Succeeded (idempotent on re-runs)

2. Guard: only delete stale SM CSVs when the target CSV does NOT exist.
   If the CSV exists in any state (Installing, Pending), skip cleanup —
   prevents killing an in-progress installation on outer retry.

3. Scale down Ingress Operator (best effort — CVO may restore)

4. Inner race loop (10 attempts) with four-way check:
   - If channel EMPTY (subscription doesn't exist) → create it, continue
   - If channel WRONG (Ingress Operator overwrote) → delete+recreate, continue
   - If channel CORRECT but CSV MISSING AND `status.installedCSV` SET →
     stale state from prior run; delete+recreate subscription, continue
   - If channel CORRECT but CSV MISSING AND `status.installedCSV` EMPTY →
     fresh subscription, CSV not generated yet; fall through to plan approval
   - If channel CORRECT and CSV exists → approve unapproved plans not
     containing reject_version, check CSV Succeeded → exit 0

All variables passed via environment: block (no Jinja2 in shell body —
prevents "unbalanced jinja2 block" parse errors with jsonpath braces).

Outer Ansible retry: 50 × 10s = ~8 min budget
```

### Why this design

- **Atomic**: All steps in one shell task eliminates the race window between separate Ansible tasks where the Ingress Operator can overwrite the subscription
- **Inner retry**: 10 attempts within each outer retry handles the Ingress Operator overwrite race
- **Guarded CSV cleanup**: Only deletes SM CSVs when the target CSV is absent — prevents killing an Installing CSV on retry (latent race condition triggered by slower clusters or EE containers)
- **Stale installedCSV detection**: Fourth check handles the case where the subscription has the correct channel but the CSV was deleted (e.g., by cleanup between use cases) — `oc apply` preserves stale `status.installedCSV` which blocks OLM from creating a new CSV. Only triggers when `status.installedCSV` is actually set; if empty, the subscription is fresh (CSV not generated yet by OLM) and we fall through to plan approval instead of delete+recreate looping
- **Best-effort scale-down**: Scales Ingress Operator to 0 but doesn't wait for pod termination (CVO restores it anyway). The inner retry loop handles the race instead
- **Idempotent**: Early exit on CSV Succeeded means re-runs are safe
- **No SM-specific install plan cleanup**: Handled by the global cleanup at top of MaaS block

The Ingress Operator is scaled back up at the **end of the entire operator install phase** (after DSC Ready), not after SM alone — prevents it from interfering during sub-operator and RHCL installation.

### Why `startingCSV` must match channel head

When `startingCSV` is behind the channel head (e.g., v3.2.6 when v3.2.7 is head):
1. OLM installs v3.2.6 from approved plan
2. OLM detects v3.2.7 is available → generates unapproved upgrade plan
3. Subscription state = `UpgradePending` (not `AtLatestKnown`)
4. `status.installedCSV` never set → OLM ownership label never added
5. OLM resolver sees CSV "not referenced by a subscription" → `ConstraintsNotSatisfiable` for ALL operators in the namespace

## 4. Sub-Operator Pinning (Authorino, Limitador, DNS)

Sub-operators are pinned by **pre-creating** their subscriptions with Manual approval and `startingCSV` before RHCL is installed. Without this, RHCL's catalog `dependencies.yaml` triggers OLM to auto-create subscriptions on the `stable` channel with `Automatic` approval → v1.4.0 installs immediately.

### Sequence (install_operators.yaml Step 3)

```
1. Pre-create Authorino subscription (v1.3.2, Manual)
2. Pre-create Limitador subscription (v1.3.1, Manual)
3. Pre-create DNS subscription (v1.3.1, Manual)

4. Hold-until-clean install plan approval:
   a. Early exit if all three operators have a v1.3.x CSV Succeeded
      (version-range match, not exact startingCSV name)
   b. Wait until ALL three subs have installPlanRef (exit 1 if any missing)
   c. Check each referenced plan for reject_version (v1.4.0)
      └─ If ANY plan contains v1.4.0: delete it and exit 1 (retry)
      └─ OLM regenerates plans without v1.4.0 since no CSVs are installed yet
   d. All plans clean → approve all
   e. Wait for all three v1.3.x CSVs Succeeded (version-range match)
   └─ Retries: 50 × 10s = ~8 min budget
```

### Why hold-until-clean

OLM install plan generation is non-deterministic. On degraded clusters:
- OLM may generate a plan for 2 of 3 operators before the third subscription exists
- Approving a partial plan (2 of 3) installs v1.3.1 CSVs for those operators
- The third operator's plan then bundles v1.4.0 upgrades (OLM sees v1.3.1 as installed, resolves latest for the remaining)
- Once v1.3.1 CSVs have `Succeeded`, they enter `Replacing` state when v1.4.0 is installed — **Replacing is terminal**, no recovery path

The hold-until-clean approach: don't approve ANY plan until ALL three subs have plans AND none contain v1.4.0. Since no CSVs are installed yet, deleting v1.4.0-contaminated plans forces OLM to regenerate clean ones.

## 5. RHCL Pinning

RHCL is installed **after** SM and sub-operators are Succeeded. By this point, v1.3.x sub-operator CSVs are installed and stable.

### Sequence (install_operators.yaml Step 4)

```
1. Create RHCL subscription (v1.3.5, Manual)

2. Approve install plans and wait for RHCL CSV:
   └─ Same reject_version (v1.4.0) filter as SM approval
   └─ Approves plans not containing v1.4.0
   └─ Checks any RHCL v1.3.x CSV Succeeded (version-range match)
   └─ Retries: 50 × 10s = ~8 min budget
```

RHCL installation is simpler than sub-operators because the sub-operators are already installed — OLM won't try to bundle them into RHCL's install plan.

## 6. COO Pinning

COO uses a different strategy: **isolated namespace** + **positive-match approval** (approve only v1.4.0, not "reject v1.4.0").

### Why isolated namespace

If COO is installed in `openshift-operators` (where RHCL lives), OLM bundles COO's install plan with pending v1.4.0 RHCL upgrade plans → `ConstraintsNotSatisfiable`. Installing in `openshift-cluster-observability-operator` with its own OperatorGroup isolates OLM resolution.

### Sequence (install_operators.yaml Step 6b)

```
1. Create namespace: openshift-cluster-observability-operator
2. Create OperatorGroup (AllNamespaces: spec: {})
3. Delete stale COO subscription + CSVs + install plans
   └─ Early exit if COO v1.4.0 CSV already Succeeded (idempotent)
   └─ Prevents stale installedCSV and approved plans from prior runs
4. Create subscription (stable, Manual, startingCSV: v1.4.0)

5. Approve COO v1.4.0 install plan:
   └─ POSITIVE match: approve plans containing "v1.4.0" (not reject-based)
   └─ v1.5.0 plans left unapproved
   └─ Checks specific startingCSV by name
   └─ Retries: 50 × 10s = ~8 min budget
```

**Key difference from SM/RHCL**: COO uses positive matching (`grep -q "v1.4.0"` → approve) instead of negative matching (`grep -q "$REJECT"` → skip). This is because the COO namespace only has one operator — no risk of approving the wrong plan.

## 7. Degraded Cluster Behavior

On resource-constrained or slow clusters, several issues occur:

### OLM Install Timeout → CSV Failed

OLM has a fixed timeout for install strategy execution. When the cluster is slow (many operators installing simultaneously, API server under load), the deployment takes longer than the timeout. The CSV transitions to `Failed` with message `install strategy failed: Timeout: request did not complete within requested timeout`.

**Behavior**: The CSV shows `Failed` but pods are actually Running. OLM retries the install from the approved plan. The retry task (50 × 10s) waits for CSV to reach `Succeeded`. On the retry, OLM finds pods already running and transitions the CSV to `Succeeded`.

**Impact**: Adds ~2-5 minutes to the install time. No intervention needed — the retry budget absorbs it.

### Leader Election Timeout → CrashLoopBackOff

When the API server is overwhelmed, operator pods can't renew their leader election leases (`Client.Timeout exceeded while awaiting headers` on lease PUT/GET to `172.30.0.1:443`). The operator loses leadership and exits. Kubernetes restarts it, but the next renewal may also timeout.

**Affected operators**: Authorino, Limitador, Kuadrant operators (all in `openshift-operators`).

**Impact on Kuadrant Ready**: The `Wait for Kuadrant ready` task in `configure_maas.yaml` checks for `Ready=True` condition. With operators in CrashLoopBackOff, Kuadrant can't become Ready. The task has 50 × 10s = ~8 min budget. If the cluster stabilizes within that window (usually after the heavy operator install phase completes), the operators recover and Kuadrant becomes Ready.

### OLM Resolver Stale State

After cleanup and redeploy, the OLM resolver's in-memory state can be stale:
- **`catalog-operator`** — handles catalog resolution and install plan generation. Stale cache causes install plans not to be generated for subscriptions.
- **`olm-operator`** — handles subscription reconciliation, sets `installedCSV`, adds ownership labels. Stale state causes `UpgradePending` even when CSV is at channel head.

**Fix**: Restart both pods in `openshift-operator-lifecycle-manager` before every fresh deployment:
```bash
oc delete pod -n openshift-operator-lifecycle-manager -l app=catalog-operator
oc delete pod -n openshift-operator-lifecycle-manager -l app=olm-operator
```

### Stale sailoperator CRDs

Previous SM installs leave sailoperator CRDs with stale `storedVersions`. New SM install plans fail because OLM detects "risk of data loss" from stored version mismatch.

**Guard**: Step 1.5 (`Clean stale sailoperator CRDs`) runs before the MaaS block for both MaaS and inference-only clusters. It checks if ANY SM CSV exists (regardless of phase). If an SM CSV exists in any state (Succeeded, Pending, Installing, Replacing), the CRDs are actively needed and deletion is skipped. Only when no SM CSV exists at all — truly orphaned CRDs from a prior deployment — are they deleted. This prevents CRD loss on AAP self-healing retries where the Ingress Operator has overwritten the SM subscription, leaving a non-Succeeded CSV that still depends on the CRDs.

### Ingress Operator SM Subscription Race

The Ingress Operator continuously reconciles the SM subscription. Two race conditions exist:

**During cleanup**: Deleting the SM subscription triggers the Ingress Operator to recreate it on the `stable` channel within seconds. Creates an orphaned subscription → `ConstraintsNotSatisfiable`.

**Fix**: Cleanup scripts scale down the Ingress Operator before deleting SM resources, scale back up after.

**During install (retries)**: After Job N fails, the Ingress Operator is scaled back up (end of `install_operators.yaml`). During the backoff before Job N+1, it recreates the SM subscription on `stable` channel with `startingCSV: v3.1.0`. CVO restores the Ingress Operator if scaled down, making separate scale-down/wait/create tasks unreliable.

**Fix**: Atomic shell task with inner race loop (see Section 3). Best-effort scale-down, then four-way check: subscription missing → create; wrong channel → delete+recreate; correct channel but CSV missing → stale `installedCSV`, delete+recreate; correct channel and CSV exists → approve plans. Once CSV Succeeded, the Ingress Operator overwrite is harmless.

### Catalog Source Instability

Default catalog source pods (`memoryTarget: 30-40Mi`) OOM or fail startup probes when the operator index is large (1.8GB for `redhat-operator-index`). The gRPC server can't initialize within the startup probe window, causing pods to cycle every 2-10 minutes. During cycling, OLM can't resolve subscriptions — install plans aren't generated or CSV bundles can't be pulled.

**Symptoms**: Catalog pods cycling (frequent `Killing` events in `openshift-marketplace`), `TRANSIENT_FAILURE` connection state, operator installations timing out even with correct subscriptions.

**Fix**: Increase `memoryTarget` to 256Mi on all catalog sources:
```bash
for cs in redhat-operators certified-operators community-operators redhat-marketplace; do
  oc patch catalogsource "$cs" -n openshift-marketplace --type merge \
    -p '{"spec":{"grpcPodConfig":{"memoryTarget":"256Mi"}}}'
done
```

**Diagnosis**: Run `analysis/debug_olm_issue.sh` — checks catalog health, pod stability, and memory configuration.

### CRD Loss Recovery (inference-only retries)

On `enable_maas=false` clusters, SM is auto-installed via RHOAI DSC + Ingress Operator. Between AAP retry jobs, the Ingress Operator's `gatewayclass_controller` may overwrite the SM subscription (channel → `stable`). OLM detects the change, replaces the CSV, and — if CRDs had ownerRefs or the stale CRD guard ran before the fix — the sailoperator CRDs can be lost. The SM CSV then shows `Pending` (RequirementsNotMet: CRDs missing) while install plans show `Complete` (stale — OLM won't recreate resources from completed plans).

**Recovery**: The SM auto-approve task (runs only when `enable_maas=false`) detects this corrupt state: SM CSV is `Pending` AND zero sailoperator CRDs exist. When detected, it deletes the SM subscription, CSV, and all SM install plans, then exits 1. On the next retry iteration, the Ingress Operator recreates the subscription, OLM generates fresh install plans with CRDs, and the auto-approve task approves them normally.

### osac-operator Retry Behavior

When a provision job fails, the osac-operator retries with **exponential backoff** — it is NOT terminal on first failure. The backoff schedule: 2min → 4min → 8min → 16min → 30min (cap), repeating indefinitely. The `ClusterOrder` phase shows `Failed` during the backoff window, but this is temporary — the phase returns to `Progressing` when the retry job launches.

The `OnFailed` callback sets phase to `Failed`, but `EvaluateAction()` returns `Backoff` (not `Skip`) when the latest job failed with matching `ConfigVersion`. `HandleBackoff()` computes delay from `BackoffBaseDelay=2min` doubling to `BackoffMaxDelay=30min`.

## 8. Verification

`verify.yaml` includes a version pinning assertion (Test 4):

```yaml
- name: Assert no RHCL v1.4.0 CSVs installed
  ansible.builtin.assert:
    that:
      - >-
        _all_csvs.resources
        | selectattr('metadata.name', 'match', '(rhcl|authorino|limitador|dns)-operator')
        | selectattr('metadata.name', 'search', 'v1.4.0')
        | list | length == 0
    fail_msg: "RHCL v1.4.0 detected — version pinning failed"
    success_msg: "All RHCL operators at v1.3.x — version pinning held"
  when: ocp_4_20_ai_maas_enable_maas | bool
```

This assertion scans ALL CSVs in `openshift-operators` and fails if any RHCL-related operator (rhcl, authorino, limitador, dns) has v1.4.0 in its name. It runs only when MaaS is enabled (inference-only clusters don't have these operators).

COO version is verified separately in the observability stack assertion (Test 4b) — checks that the specific `startingCSV` is `Succeeded`.

## 9. Cleanup Requirements

Cleanup scripts must handle version-pinned operators carefully:

| Requirement | Reason |
|-------------|--------|
| Delete stale subscriptions + CSVs + install plans in `openshift-operators` and COO namespace | `state:present` updates spec but NOT `status.installedCSV` — stale `installedCSV` from prior runs makes OLM think operators are installed → `UpgradePending`. Already-approved v1.4.0 plans cause immediate v1.4.0 install before hold-until-clean can reject. |
| Scale down Ingress Operator before deleting SM resources | Prevents SM subscription recreation race |
| Delete sailoperator CRDs after SM CSV is removed | Prevents `storedVersions` conflict on reinstall |
| Delete COO/OTel from isolated namespaces (not `openshift-operators`) | OLM resolves per-namespace — mixing causes bundling |
| Never patch CSV finalizers | Let OLM handle its own lifecycle |
| Never approve plans containing v1.4.0 | `Replacing` state is terminal — no recovery |
| Restart both `catalog-operator` AND `olm-operator` before fresh deploy | Clears stale resolver and subscription reconciliation state |
| Scale Ingress Operator back up after cleanup | Otherwise next deploy fails at Ingress-dependent steps |

## 10. Installation Order

The installation order is critical. Violations cause `ConstraintsNotSatisfiable` or silent auth failure.

```
1.   cert-manager (Automatic, any channel — RHCL dependency)
       ↓
1.5  Clean stale sailoperator CRDs (UNCONDITIONAL — runs for all use cases)
     └─ Skip if ANY SM CSV exists (any state — CRDs actively needed)
     └─ Delete only when no SM CSV at all (orphaned from prior deployment)
       ↓
     ┌─── MaaS only (enable_maas=true) ───────────────────────────┐
     │                                                             │
     │ 1b. Delete stale subs + CSVs + install plans               │
     │     └─ Targets sub-operators and RHCL only                 │
     │     └─ Skips if all target CSVs already Succeeded          │
     │       ↓                                                     │
     │ 2.  SM 3.2 (Manual, stable-3.2, atomic shell task)         │
     │     └─ Inner race loop vs Ingress Operator overwrite       │
     │       ↓                                                     │
     │ 3.  Sub-operators: Authorino, Limitador, DNS (v1.3.1)      │
     │     └─ Hold-until-clean: reject v1.4.0 plans               │
     │       ↓                                                     │
     │ 4.  RHCL v1.3.4 (Manual, stable)                           │
     │                                                             │
     └─────────────────────────────────────────────────────────────┘
       ↓
5.   LWS (Automatic — no version sensitivity)
       ↓
6.   RHOAI (Automatic — installs SM via DSC if not already present)
       ↓
     ┌─── Inference-only (enable_maas=false) ─────────────────────┐
     │                                                             │
     │ 6x. SM auto-approve + CRD loss recovery                   │
     │     └─ Approves Ingress Operator's Manual SM install plans │
     │     └─ If SM Pending + 0 sailoperator CRDs: cleans corrupt │
     │        OLM state (deletes sub/CSV/plans), retries           │
     │                                                             │
     └─────────────────────────────────────────────────────────────┘
       ↓
6a.  OTel (Automatic, isolated namespace — if enable_observability)
6b.  COO v1.4.0 (Manual, isolated namespace — if enable_observability)
     └─ Stale subscription + CSVs + install plans deleted before creation
       ↓
7.   DSCI metrics patch → DSC creation → DSC Ready
       ↓
8.   Ingress Operator scaled back up (MaaS only)

All version-pinned operators: retries 50 × delay 10s = ~8 min budget per step
```

## Sources

- `install_operators.yaml` — the implementation (single source of truth)
- `verify.yaml` — version pinning assertion
- `defaults/main.yaml` — pinned version defaults
- `cleanup_maas_deployment.sh` — cleanup ordering
- [RHCL v1.4.0 Wasm shim issue](https://access.redhat.com/solutions/7097633)
- [RHCL/SM supported config matrix](https://access.redhat.com/articles/7092611)
- [COO v1.5 + RHOAI 3.4 incompatibility](https://issues.redhat.com/browse/RHOAIENG-69180)
