# Security Audit — Hermes runtime Act 1 (A1.1 + A1.2 + A1.4 scaffold)

**Scope**: Phase A1.1 (sandbox image), A1.2 (controller CRD enum + image selection), and A1.4 plugin scaffold for the new Hermes Agent runtime on kars. Foundational shipment — runtime works in single-agent mode on AKS, local-k8s, and docker dev; mesh tools ship as clear-error stubs (Act 2 ships the Python AgentMesh SDK at TypeScript parity).

## Changes

### 1. `sandbox-images/hermes/Dockerfile` + `entrypoint.sh` (new)

- Base image: `mcr.microsoft.com/azurelinux/base/python:3.12` pinned by SHA (`@sha256:485299b016fe5ae745ffee27f0b8a850576841205ed1d420c9a84b126198e320`).
- Install: tdnf (git, jq, curl, ripgrep), AGT-Python wheels from `runtimes/wheels/`, `hermes-agent` pinned to a version via `ARG HERMES_VERSION`, the kars-runtime-hermes Python package.
- Stages the kars plugin tree at `/opt/kars-hermes-stage/plugins/kars/` (read-only). The entrypoint mirrors it to `$HERMES_HOME/plugins/kars/` on every boot so plugin updates ship with the image, not via writable state.
- Pre-creates `/etc/kars/policies`, `/etc/kars/blocklist`, `/etc/kars/mcp` for router-side mounts.
- Runs as UID 1000 (matches AKS pod spec). Docker dev mode runs as root briefly to set up iptables egress guard, then drops via runuser.
- HEALTHCHECK probes `hermes --version`.

The entrypoint:
- Sets `HERMES_HOME=/sandbox/.hermes`, `HERMES_PROFILE=$SANDBOX_NAME`, `HERMES_DISABLED_BACKENDS=docker,modal,daytona,ssh,singularity` (only `local` is valid inside a kars pod).
- Pins `OPENAI_BASE_URL=http://127.0.0.1:8443/v1` and `ANTHROPIC_BASE_URL=http://127.0.0.1:8443/anthropic/v1` so every model call hits the router (governance applied: InferencePolicy, content safety, token budgets, audit).
- Sets provider API keys to the `router-managed` sentinel so leaked URLs can't bypass the router.
- Translates `/etc/kars/mcp/<server>/meta.json` mounts to Hermes' native `mcp_servers.*` config block (Hermes' MCP client then dispatches through router `/mcp` for kars-governed MCP).
- Translates channel env tokens (TELEGRAM_BOT_TOKEN, SLACK_BOT_TOKEN, DISCORD_BOT_TOKEN, WHATSAPP_ENABLED, TELEGRAM_ALLOW_FROM) and third-party plugin keys (BRAVE_API_KEY, TAVILY_API_KEY, EXA_API_KEY, FIRECRAWL_API_KEY, PERPLEXITY_API_KEY) into `hermes config set` invocations.
- Sets up iptables egress guard in docker dev only (AKS / local-k8s use an init container).
- Asserts the runtime contract version `KARS_RUNTIME_CONTRACT_VERSION=v1` (treats absent as v1 for back-compat until A1.2-pending change makes it controller-injected for every runtime).
- Starts `hermes gateway start --foreground` if any channel tokens present, else `hermes` (TUI mode).

### 2. `runtimes/hermes/` Python package (new)

`kars-runtime-hermes` — the in-pod Hermes plugin. Layout:
```
runtimes/hermes/
├── pyproject.toml          # hatchling, deps: httpx, pyyaml
├── README.md               # architecture overview + Act 1 limitations
├── src/kars_runtime_hermes/
│   ├── __init__.py         # __version__, KARS_RUNTIME_CONTRACT_VERSION
│   └── plugin/             # the actual Hermes plugin tree (also staged into image)
│       ├── __init__.py     # def register(ctx) — Hermes plugin entry point
│       ├── plugin.yaml     # Hermes manifest declaring provides_tools + provides_hooks
│       ├── router_client.py    # shared httpx client to localhost:8443; admin-token discovery
│       ├── governance.py       # pre_tool_call hook — stub (A1.4 ships impl)
│       ├── spawn.py            # kars_spawn family — stub (A1.5 ships impl)
│       ├── discover.py         # kars_discover — stub (A1.6 ships impl)
│       ├── foundry.py          # 9 foundry_* tools — stub (A1.7 ships impl)
│       ├── http_fetch.py       # http_fetch — stub (Phase A1.4 continuation)
│       ├── handoff.py          # kars_handoff_* — stub
│       ├── telemetry.py        # /agt/trust + /agt/signing-counter pushes — stub (A1.10)
│       └── mesh_stubs.py       # kars_mesh_* tools — clear-error stubs (Act 2 swaps for live impl)
└── tests/
    └── test_package_shape.py   # import-shape sanity tests (5 tests, all pass)
```

The plugin entry `register(ctx)` lazy-imports each submodule and calls its `register(ctx)`. Stubs are intentional: this commit ships the package SHAPE so the image build, the test infrastructure, and the CRD enum can land. Subsequent commits (A1.4 → A1.10) fill in the stub bodies one tool family at a time.

### 3. `controller/src/crd.rs` — `RuntimeKind::Hermes` + `HermesConfig`

- New enum variant `RuntimeKind::Hermes`, serialised to wire string `"Hermes"`.
- New struct `HermesConfig` with fields `version`, `agentCode` (OCI or git), `entrypoint`, `extraEnv`. Wire shape mirrors `PydanticAiConfig` for consistency.
- `RuntimeSpec` gains `pub hermes: Option<HermesConfig>` + `hermes: None` in Default.
- New tests:
  - `runtime_kind_serializes_to_pascal_case_literals` extended for `Hermes` + `PydanticAi`.
  - `hermes_runtime_spec_round_trips_through_yaml` — YAML → struct → JSON round-trip with `agentCode.oci` + `extraEnv`.

### 4. `controller/src/reconciler/runtime.rs` — image selection + plan

- New `DEFAULT_HERMES_IMAGE = "karsacr.azurecr.io/kars-runtime-hermes:latest"` + `hermes_default_image()` honouring `KARS_HERMES_IMAGE` env override.
- `kind_str()` adds `RuntimeKind::Hermes => "Hermes"`.
- `validate_runtime_shape()` adds `("Hermes", "hermes", runtime.hermes.is_some())` to the pairs list.
- `build_runtime_plan()` adds `RuntimeKind::Hermes` arm calling `plan_hermes()`.
- New `plan_hermes(cfg)` producer — mirrors `plan_pydantic_ai` shape (image, command, agent_code, extra_env merge with HERMES_VERSION priming).
- `rt_only_kind` test helper updated to include `hermes: None`.
- 7 new tests:
  - `hermes_default_image_falls_back_when_env_unset`
  - `hermes_default_image_honours_operator_override`
  - `plan_hermes_emits_hermes_kind_str_and_default_image` — also asserts no `KARS_*` env leakage from producer
  - `plan_hermes_threads_version_and_user_extra_env`
  - `plan_hermes_user_extra_env_overrides_producer_default`
  - `plan_hermes_carries_agent_code_and_entrypoint_overrides`
  - `plan_hermes_rejects_shape_invalid_input` — shape validation runs before producer

## Risk Assessment

- **Strictly additive**: nothing in OpenClaw or the other 6 runtimes (PydanticAi, LangGraph, Anthropic, OpenAIAgents, MicrosoftAgentFramework, SemanticKernel, BYO) is changed. New enum variant; new struct; new producer function. CRD schema gains an optional field; existing CRs without `runtime.hermes` round-trip unchanged.
- **Mesh stubs by design**: `kars_mesh_*` tools return a clear error directing operators to use Foundry Memory Store / Conversations as a workaround until Act 2 ships the Python AGT MeshClient. This is preferred to silently failing or omitting the tools — the LLM sees a discoverable error message.
- **Image is hardened**: base image pinned by SHA, runs as UID 1000 in K8s, only `local` Hermes terminal backend is allowed (others would break out of K8s isolation), egress guard via iptables in docker dev (init container in AKS).
- **No new secrets in source**: provider keys remain in `<sandbox>-credentials` Secrets / docker env passthrough; entrypoint reads them and pipes to `hermes config set` via subprocess (never written to a file in the container).
- **MCP translation**: entrypoint reads kars-published `/etc/kars/mcp/<server>/meta.json` and emits a YAML fragment to `$HERMES_HOME/config.yaml`. The bearer token references use Hermes' env-var substitution syntax (`${ENV_VAR}`) — actual token bytes never land in the writable config file.
- **Plugin tree is staged read-only**: image stages plugin at `/opt/kars-hermes-stage/plugins/kars/` with `chmod a+rX`. Entrypoint mirrors to writable `$HERMES_HOME` on each boot so the agent cannot tamper with the canonical copy.

## Platform safety

| Change | AKS | local-k8s | docker dev |
|---|---|---|---|
| Sandbox image | builds + ACR-pushed via existing `make image-hermes` (to add in A1.3) | builds + kind-loaded via `cli/src/commands/dev/local-k8s.ts::rebuildDevImages` (to wire in A1.3) | builds + used directly by `docker run` in `cli/src/commands/dev.ts` (to wire in A1.3) |
| CRD enum extension | KarsSandbox CRDs accept `runtime.kind: Hermes` immediately | same | n/a (no CRDs) |
| Controller plan | controller picks Hermes image on `runtime.kind: Hermes` | same | n/a (docker spawn handled by `inference-router/src/spawn/docker.rs` — A1.3 makes it runtime-aware) |

A1.3 (next commit) wires CLI flag + docker spawn runtime-awareness, completing the three-platform integration.

## Testing

- **kars-controller (Rust)**: `cargo test --package kars-controller --bins` → 834 passed (+9 new — 7 in `runtime.rs` + 2 in `crd.rs`).
- **Hermes plugin (Python)**: `pytest` in `runtimes/hermes/` → 5 passed.
- **Lints**: `cargo clippy --all-targets -- -D warnings` → clean; `cargo fmt --all` → clean.
- **Entrypoint syntax**: `bash -n sandbox-images/hermes/entrypoint.sh` → ok.
- **LOC budget**: `ci/check-loc.sh` → pass (no budgeted files grew above caps; `controller/src/crd.rs` and `controller/src/reconciler/runtime.rs` are within budget).

## Foundation rationale (rubber-duck pass)

A1.0 (the runtime contract spec at `docs/runtimes/CONTRACT.md` — shipped in `2aef3c1`) was rubber-ducked before this commit; the critique caught 6 blockers and 5 important issues in the spec — all addressed. The list of "currently OpenClaw-only" env injections + mounts that A1.2 should lift to be generic is tracked in the todo descriptions and will land as a follow-up commit (this commit completes A1.2's CRD enum + plan dispatch; the env-generalization across runtimes is a separate change touching `controller/src/reconciler/mod.rs` and is intentionally not bundled here to keep the diff reviewable).

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: GitHub Copilot CLI <223556219+Copilot@users.noreply.github.com>
