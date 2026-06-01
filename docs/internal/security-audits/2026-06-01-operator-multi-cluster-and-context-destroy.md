# Security Audit — operator multi-cluster + context-aware destroy

**Scope**: Commits `8600f45`, `5c7993f`, `a7158a3`, `1552e95` on `main`.
Multi-cluster discovery in `kars operator` + explicit `--context`
threading in `kars destroy` + per-sandbox kubeContext propagation
from operator → destroy shell-out.

Capability paths touched:

- `cli/src/commands/operator.ts`
- `cli/src/commands/destroy.ts`
- `cli/src/commands/operator/dialogs/delete.ts`
- `cli/src/commands/operator/fetchers/cluster.ts`

All four changes are **UX correctness fixes**. They do not introduce,
remove, or weaken a security capability. This audit documents that
analysis.

## 1. What changed

### 1a. `kars destroy --context <name>` (new flag)

New optional flag, threaded into every `kubectl` invocation in the
command (5 sites). When absent, behaviour is identical to before the
change (kubectl falls back to current-context). Print which cluster
will be targeted in cyan up-front so an operator can `^C` if it's the
wrong one.

### 1b. Operator delete-dialog forwards kubeContext

`cli/src/commands/operator/dialogs/delete.ts` `deleteSelectedAgent()`
now adds `--context <name>` to the `kars destroy` shell-out args. The
context value comes from:

1. The sandbox's own `kubeContext` (cross-cluster aggregation), or
2. The operator's `--context` flag, or
3. (none — falls back to kubectl current-context)

This prevents the "I have both kind-kars-dev AND kars-aks in my
kubeconfig and pressed 'd' on an AKS sandbox but it tried to delete
from kind" foot-gun.

### 1c. Operator: multi-cluster mesh-health aggregation

`fetchMeshHealthMulti(devMode, contexts[])` probes the relay/registry
health endpoint on every context in parallel and OR-aggregates the
ready bits + sums the pod counts. Wrapped by `resolveMeshHealth()`
that picks single- vs multi-cluster automatically based on whether
`--context` is set.

### 1d. Operator: column widths

Cosmetic — `Name` 32→36, `Model` 14→18, `Cluster` 18→22,
`columnSpacing` 1→2. No security relevance.

## 2. Security posture comparison

| Dimension | Before | After | Δ |
|---|---|---|---|
| **Auth model** | uses kubectl's local kubeconfig | uses kubectl's local kubeconfig | unchanged |
| **Cluster access** | per-sandbox kubeContext or current-context | same — now with explicit override and better fallback chain | unchanged |
| **Privilege escalation** | none | none | unchanged |
| **Cross-cluster blast radius** | `kars operator` (no --context) silently fell back to whichever kubeconfig context was current → potential for "delete from wrong cluster" | **strictly improved** — destroy shell-out now carries the per-sandbox kubeContext, eliminating the silent-cross-cluster footgun | better |
| **kubectl context isolation** | implicit (depends on user discipline) | explicit per command, printed in cyan before any destructive action | better |
| **Read scope of mesh health probe** | one cluster only | union of clusters whose sandboxes appear in the view | wider, but same probe (read-only `kubectl exec` of `/health`), same RBAC requirements per cluster |

## 3. Threat-model deltas

None. The threat model for `kars operator` and `kars destroy`
remains: the operator is a trusted operator console; deleting an
agent reads/writes only the resources the user's kubeconfig already
permits. The fixes here make explicit what was previously implicit
(which cluster a command targets), reducing accident-driven blast
radius.

No new endpoints. No new env vars consumed. No new ServiceAccounts.
No new RBAC requirements. No new outbound network paths.

## 4. Reviewer checklist

- [x] No new ports or listeners
- [x] No new outbound network destinations
- [x] No new privileges granted
- [x] Same kubeconfig-driven auth model
- [x] Destructive actions print the target cluster up-front
- [x] LOC delta within §4.3 budgets (operator.ts: 940 → 931)
- [x] 786/786 CLI tests still pass

---

Signed-off-by: @pallakatos
Signed-off-by: @Copilot
