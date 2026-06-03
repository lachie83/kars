# Security Audit — DX noise cleanup (preflight gating + orphan event suppression + reputation auth)

**Scope**: Three independent DX fixes shipped as one PR.

## Changes

### 1. `cli/src/commands/dev.ts` + `cli/src/commands/dev.test.ts`

Preflight tool check (`preflightTools`) was failing every `kars dev`
invocation when the AGT toolkit checkout was absent — even for
overlay-refresh runs that never touch the toolkit. The check is now
gated behind `opts.build === true && opts.noMesh === false`. The
genuine "must clone the toolkit" path is unchanged.

`preflightTools` was promoted from module-private to `export`ed so the
new unit test (`dev.test.ts`) can exercise the three branches:

  - `build=false` → skip AGT check (the bug we fixed).
  - `build=true, noMesh=true` → skip AGT check (mesh stack not deployed).
  - `build=true, noMesh=false` → still require AGT toolkit (genuine path).

### 2. `controller/src/status/router_confirmation.rs` + three reconcilers

New shared helper `should_publish_warning(state, degraded) -> bool`
that returns `true` only when the controller is genuinely waiting on
a bound sandbox to echo (`RouterEnforcementState::Awaiting`). The
previous logic also returned `true` for `NoSandboxesReferencing`,
which spammed the event log every reconcile for any chart-installed
default CR nobody happens to reference — `kars-default` ToolPolicy
is the worst offender (every cluster).

All three reconcilers now use the helper:

  - `controller/src/tool_policy_reconciler.rs`
  - `controller/src/inference_policy_reconciler.rs`
  - `controller/src/kars_memory_reconciler.rs`

Also: `tool_policy_reconciler` requeue cadence for
`NoSandboxesReferencing` slowed from 15 s to `REQUEUE_OK` (5 min).
(The other two were already at 5 min — only ToolPolicy had the
tight 15 s loop for orphans.)

Orphan condition is still surfaced via the CR's
`Ready=False / reason=NoSandboxesReferencing` condition so
`kubectl get` shows the state — just without periodic Warning
Event spam.

### 3. `mesh-plugin/src/agt-transport.ts` + `agt-transport.test.ts`

`submitReputation()` now sends the same `Authorization: Ed25519-Timestamp
<did> <iso8601> <base64url(sig)>` header that
`pingRegistryPresence()` already produces (line 839-861). AGT 4.0
registry (commit `66918631`, May 28 2026) requires this auth on every
mutating POST including `/v1/agents/{did}/reputation`. Without it
every submission silently 401s on the server side, leaving
`feedback_count` at 0 forever — the comment at
`inference-router/src/routes/mesh.rs:431` documents the symptom.

Two new tests:

  - `attaches Ed25519-Timestamp authorization header on success` —
    asserts wire format (`Ed25519-Timestamp <did> <ISO> <base64url>`).
  - `returns false (not throw) when registry rejects with 401` —
    callsite at `runtimes/openclaw/src/index.ts:1068` ignores the
    boolean return; the SDK must not bubble errors.

## Risk Assessment

- **All three changes are strictly loosening behaviour or adding
  auth headers** — no new attack surface introduced.
- **Preflight (1):** worst case is a user reaches the build step
  with no AGT toolkit cloned — same failure they'd get if we left
  the check up-front, just deferred. The genuine "I'm doing `--build`"
  path still fails loud.
- **Orphan suppression (2):** the underlying status condition is
  unchanged — `kubectl get` still shows `Ready=False
  reason=NoSandboxesReferencing`. Operators who scripted on the
  Warning Event get fewer log lines but the same diagnostic surface
  (recommend they read the CR's `.status.conditions`, which is the
  canonical surface anyway).
- **Reputation auth (3):** the signed payload is identical to
  `pingRegistryPresence`. The AGT registry's
  `verify_ed25519_timestamp_auth` (verified at
  `~/Private/Repos/agt/.../registry/app.py:97-140`) signs the
  timestamp only, with the same replay window as heartbeats. The
  callsite (`runtimes/openclaw/src/index.ts:1068`) is never self-
  reporting (registry rejects that at `app.py:449-453`); the
  sender is the parent receiving a reply from the sub-agent, so
  `did` and `toAmid` always differ.

## Platform safety

| Change | AKS | local-k8s | docker |
|---|---|---|---|
| Preflight | n/a (CLI not used in cluster) | unblocks overlay refresh | unblocks reuse-image path |
| Orphan suppression | stops `kars-default` spam | same | n/a (no controller) |
| Reputation auth | unbreaks `feedback_count` | same | same |

All three changes share code paths across platforms — no
platform-specific branching introduced.

## Testing

- `cargo test --workspace --lib --bins`: 1761 passed (controller 826,
  router 932, plus crate sub-tests).
- New tests:
  - `status::router_confirmation::tests::should_publish_warning_only_when_awaiting`
  - `cli::commands::dev::tests::skips_the_AGT_toolkit_existence_check_when_build_is_false`
  - `cli::commands::dev::tests::skips_the_AGT_toolkit_check_when_noMesh_is_true_even_if_build_is_true`
  - `cli::commands::dev::tests::DOES_require_the_AGT_toolkit_when_build_is_set_and_mesh_is_on`
  - `mesh-plugin::agt-transport::submitReputation::attaches_Ed25519_Timestamp_authorization_header_on_success`
  - `mesh-plugin::agt-transport::submitReputation::returns_false_not_throw_when_registry_rejects_with_401`
- `cargo clippy --all-targets -- -D warnings`: clean.
- `cargo fmt --all`: clean.
- `npm test` (cli): 789 passed, 2 skipped (added 3 new).
- `npm test` (mesh-plugin): 68 passed, 3 skipped (added 2 new).
- `npm test` (runtimes/openclaw): 244 passed.
- LOC budget: `ci/check-loc.sh` passes (no budgeted files grew above caps).

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: GitHub Copilot CLI <223556219+Copilot@users.noreply.github.com>
