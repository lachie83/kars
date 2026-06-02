# Security Audit — relaxed dev mode + Foundry tool gating fix

**Scope**: One commit. Adds `KARS_DEV_PROFILE=true` signal across the dev
stack (docker + local-k8s) so spawned sub-agent CRDs land with relaxed
network-policy defaults; also fixes a missing env-var push that left
`KARS_PROVIDER` invisible to the OpenClaw plugin in local-k8s mode.

## Changes

1. **`controller/src/reconciler/dev_env.rs`** (new) — Single-purpose
   helper that pushes `KARS_PROVIDER` + `KARS_DEV_PROFILE` env vars
   onto the per-sandbox container envs (router + openclaw). Extracted
   from `reconciler/mod.rs` to respect the §4.2 LOC budget.

2. **`controller/src/reconciler/mod.rs`** — New `dev_profile` field on
   `Context`, populated at startup from `KARS_DEV_PROFILE`. Calls
   `dev_env::apply(...)` to push KARS_PROVIDER onto BOTH the router
   AND the openclaw container (was: router only — the plugin reads
   it on openclaw, so the Foundry-tool gate never fired for local-k8s
   sub-agents in github-copilot / github-models mode).

3. **`inference-router/src/spawn/mod.rs`** + **`spawn/docker.rs`** —
   `build_sub_agent_crd_with_labels` reads `KARS_DEV_PROFILE` at
   call-time. When true: `approvalRequired=false` and
   `egressMode=Learn` on every spawned sub-agent CRD. Docker spawn
   helper also pushes `KARS_DEV_PROFILE=true` onto every child it
   creates (docker spawn is dev-only by definition). New
   `dev_profile_test.rs` pins both branches.

4. **`runtimes/openclaw/src/core/agt-tools/agt.ts`** — `kars_spawn`
   tool passes `learn_egress: true` in the spawn POST body when its
   own env has `KARS_DEV_PROFILE=true`. Defense in depth: gives the
   relaxed default even if the parent's router binary is a stale
   pre-relaxation build.

5. **`cli/src/commands/dev.ts`** — Docker dev parent gets
   `-e KARS_DEV_PROFILE=true` next to the existing
   `-e KARS_DEV_MODE=true`.

6. **`cli/src/commands/dev/local-k8s.ts`** — local-k8s controller
   deployment overlay adds `KARS_DEV_PROFILE=true` to
   `controller.extraEnv` so the controller reads it on boot and
   propagates it onto every reconciled sandbox.

## Risk Assessment

- **`KARS_DEV_PROFILE` is opt-in via env.** Production AKS clusters
  never receive this env var. `kars push --apply` writes no helm
  value for it. Strict defaults (`approvalRequired=true`,
  `egressMode=Strict`) remain on AKS.
- **The Foundry-tool gating fix is strictly tightening, not
  loosening.** It causes the OpenClaw plugin to register FEWER tools
  in copilot/models modes (which it was already supposed to do per
  prior security review).
- **Three governance suppressors** propagated to local-k8s sandbox
  routers (`KARS_SUPPRESS_EXFIL_URL=1`,
  `KARS_SUPPRESS_CONTENT_FLAGS=violence`,
  `KARS_CONTENT_FLAG_MIN_SEVERITY=medium`) are the exact same set
  the docker dev parent already had wired since 2026-05-21 (no new
  capability, just consistency).
- **Egress in Learn mode still logs every novel destination**;
  operators get a complete diff before promoting to Strict via
  `kars policy allow`. The forward-proxy blocklist (OISD + URLhaus)
  is enforced regardless of mode — Learn does NOT bypass threat
  intelligence.
- **`approvalRequired=false` only affects dev sub-agents** spawned
  via `POST /sandbox/spawn` under `KARS_DEV_PROFILE=true`. The
  parent sandbox's `approvalRequired` flag is unchanged. Operators
  still see every egress destination in audit logs even when
  approval isn't gated.

## Testing

- `cargo test --workspace --lib --bins`: 1761 passed (controller 825,
  router 932, others 4).
- New tests:
  - `controller::reconciler::dev_env::tests::no_provider_no_profile_pushes_nothing`
  - `controller::reconciler::dev_env::tests::provider_pushed_to_both_containers_when_openclaw`
  - `controller::reconciler::dev_env::tests::provider_not_pushed_to_openclaw_when_not_openclaw_runtime`
  - `controller::reconciler::dev_env::tests::dev_profile_pushes_relaxations_to_router_and_marker_to_openclaw`
  - `inference-router::spawn::dev_profile_test::sub_agent_crd_relaxes_network_policy_under_dev_profile`
- `cargo clippy --all-targets -- -D warnings`: clean.
- `cargo fmt --all`: clean.
- `npm test` (cli): 786 passed, 2 skipped.
- `npm test` (runtimes/openclaw): 244 passed.

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: GitHub Copilot CLI <223556219+Copilot@users.noreply.github.com>
