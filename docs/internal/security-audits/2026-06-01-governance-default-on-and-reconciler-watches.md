# Security Audit — Governance-on-by-default + reconciler watches

**Scope**: Same-day follow-up to first-public-day demo polish. Two
controller behavior changes + one CLI UX fix that touch capability
surfaces in `ci-gates`' "capability-introducing files" list:

- `controller/src/crd.rs` — `GovernanceConfig` default flip
- `controller/src/reconciler/mod.rs` — empty-toolPolicyRef fallback
- `controller/src/kars_memory_reconciler.rs` — `.watches(KarsSandbox)` added
- `controller/src/inference_policy_reconciler.rs` — `.watches(KarsSandbox)` added
- `deploy/helm/kars/templates/toolpolicy-default.yaml` — NEW system-default ToolPolicy
- `deploy/helm/kars/files/kars-default-agt-profile.yaml` — NEW (copy of `cli/profiles/agt/kars-default.yaml`)
- `cli/src/commands/operator.ts` — auto-discover kube context (UX only, no capability)

This audit confirms none of these changes weaken any existing
enforcement, and the new defaults FAIL-SAFE (more enforcement, not
less).

## 1. `GovernanceConfig.enabled` defaults to `true`

**Was:** `#[serde(default)]` on `pub enabled: bool` — Rust's `bool::default()`
returns `false`. A `KarsSandbox` with no `spec.governance` block (or
with a `governance:` block omitting `enabled`) would have governance
**OFF** by default: no AGT tool-policy enforcement, no per-sandbox
Service on `:8443`, no `InferencePolicy` data-plane verification.

**Now:** `#[serde(default = "default_governance_enabled")]` returns `true`,
and a manual `impl Default for GovernanceConfig` mirrors the same.

**Capability impact:** **strictly more enforcement** for any sandbox
that did not explicitly opt out. Specifically:

- AGT tool policy is now active (deny/approval/rate-limit gates fire on
  every tool call).
- Per-sandbox Service on `:8443` is now created (enables the
  `InferencePolicy` controller's `/internal/policy-status` echo loop —
  Prompt Shields, token caps, model preference enforcement).
- Sandbox `NetworkPolicy` now allows ingress on `:8443` from
  `kars-system` (matching CEL labels — see `controller/src/reconciler/mod.rs:1004`).

**Opt-out:** set `spec.governance.enabled: false` explicitly. The
schema honors the explicit value over the new default.

**Migration risk:** Existing sandboxes WITHOUT `spec.governance` will
flip to `enabled=true` on the next reconcile. The same-commit
empty-toolPolicyRef fallback (item 2) ensures they don't break:
they auto-pick up the `kars-default` ToolPolicy shipped by Helm.

## 2. Empty `governance.toolPolicyRef.name` falls back to `kars-default`

**Was:** `reconciler/mod.rs:597-603` hard-degraded with
`SpecInvalid / "spec.governance.toolPolicyRef.name is required when
governance.enabled=true"`. The bundled `/opt/kars-plugin/policies/`
fallback was removed in Slice 1e Phase 2.

**Now:** when `tool_policy_ref.name` is empty AND governance is
enabled, the controller resolves the ref to `"kars-default"` in the
same namespace as the `KarsSandbox` CR (typically `kars-system`,
where the Helm chart ships the default). If `kars-default` does not
exist in that namespace, the existing
`ToolPolicyNotFound`/`SpecInvalid` paths apply unchanged.

**Capability impact:** **strictly more enforcement** — without the
fallback, governance-enabled sandboxes without an explicit policy
would hard-fail (no policy enforced, no pod start). The fallback
ensures a policy IS enforced (the Helm-shipped `kars-default`, which
is the canonical kars profile that the CLI's `kars add --governance`
already injects for new agents).

**Same-namespace constraint preserved:** the fallback resolves in
`sandbox_self_ns` (the KarsSandbox CR's own namespace). Cross-namespace
ToolPolicy references remain disallowed (principles.md §3). Sandboxes
in custom namespaces must ship their own `kars-default` ToolPolicy
locally OR set `toolPolicyRef.name` explicitly.

## 3. `kars-default` ToolPolicy shipped via Helm

**New file:** `deploy/helm/kars/templates/toolpolicy-default.yaml`
mounts `deploy/helm/kars/files/kars-default-agt-profile.yaml` (an
identical copy of `cli/profiles/agt/kars-default.yaml`) into a
`ToolPolicy` CR named `kars-default` in the chart's release namespace
(`kars-system`).

**Body provenance:** `cli/profiles/agt/kars-default.yaml` — the
canonical kars governance profile shipped with every `kars` CLI
install. It is the same body that `kars add --governance` already
inlines into per-sandbox ToolPolicy CRs today. No new policy authored
for this PR.

**Hook timing:** annotated with
`helm.sh/hook: post-install,post-upgrade` and
`helm.sh/hook-weight: "-5"` so it lands BEFORE any sandbox reconcile
can race on the lookup.

**Capability impact:** none — this profile is already in production
use everywhere `kars add --governance` is invoked. We're just making
it cluster-wide-default so users who didn't run that command still get
the same protection.

## 4. KarsMemory + InferencePolicy reconcilers watch KarsSandbox

**Was:** Both reconcilers only watched their own CR type (`Api<KarsMemory>` /
`Api<InferencePolicy>`). When a `KarsSandbox` was created that
referenced one of these policies, the policy reconciler did NOT
re-run immediately — it waited up to `REQUEUE_OK = 300s` (5 min) for
the next periodic resweep before flipping
`NoSandboxesReferencing` → `RouterEnforcing`. Demo + dev experience
was poor.

**Now:** both reconcilers add
`.watches(sandboxes, watcher::Config::default(), mapper)` where the
mapper returns:

- **KarsMemory:** `sb.spec.memory_ref` → `ObjectRef::<KarsMemory>::new(name).within(ns)`
- **InferencePolicy:** `sb.spec.inference_ref.name` → `ObjectRef::<InferencePolicy>::new(name).within(ns)`

**Capability impact:** none — this is a latency reduction only.
Policy enforcement state was always eventually consistent; this just
makes it converge in seconds instead of minutes. No new code path,
no new permission boundary, no new RBAC requirement (both reconcilers
already had `get/list/watch` on `KarsSandbox` via the cluster role).

**Label-selector-only InferencePolicies** (no `spec.appliesTo.sandboxName`
and never explicitly referenced from a sandbox) still rely on the
periodic 5-min resync. Out of scope for this change; would require a
reflector-backed mapper to enumerate candidates synchronously.

## 5. `kars operator` auto-discovers kube context

**Was:** `cli/src/commands/operator.ts` accepted `--context` and
honored it, otherwise fell through to `kubectl config current-context`.
When neither was set, kubectl defaulted to `http://localhost:8080`
and every fetcher silently failed.

**Now:** when `--context` is not passed AND `kubectl config
current-context` returns empty, the operator probes every kubeconfig
context and uses the first reachable one (mirrors `kars list` and
the recently-fixed `kars connect`).

**Capability impact:** UX only, no security impact. The fetchers
already passed `--context <ctx>` via the `kctl()` helper when given
one; this change just guarantees they always get one.

## RBAC & permission boundaries

No RBAC change. The controller's existing ClusterRole already has:
- `get/list/watch` on `KarsSandbox`, `KarsMemory`, `InferencePolicy`, `ToolPolicy`
- `create/update/patch` on `Service`, `NetworkPolicy`, `ConfigMap`
  inside per-sandbox namespaces

The new `.watches(KarsSandbox)` additions in the policy reconcilers
re-use the existing watch permission. The empty-toolPolicyRef
fallback re-uses the existing `Api::namespaced::<ToolPolicy>` get
permission in `sandbox_self_ns`.

## Testing

- `cargo build --release --package kars-controller` → clean (5 min cold).
- `cli npm run build` → clean.
- `cli npm test` → 786 passed | 2 skipped (no regressions).
- Live-applied the new `kars-default` ToolPolicy on the user's
  `kars-aks` cluster via `helm template ... | kubectl apply` —
  validates against live CRD schema. Existing demo sandbox
  unaffected (it already had an explicit `toolPolicyRef`).

## Conclusion

Safe to merge. All three capability-touching changes are
fail-safe-by-default — they either add enforcement (governance on by
default + Helm-shipped policy) or reduce latency of existing
enforcement (the two reconciler watches). The CLI fix is UX only.

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: GitHub Copilot CLI <223556219+Copilot@users.noreply.github.com>
