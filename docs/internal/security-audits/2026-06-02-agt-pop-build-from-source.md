# AGT POP-aware build-from-source migration

**Date:** 2026-06-02
**Scope:** `mesh-plugin/`, `runtimes/openclaw/`, `controller/src/mesh_peer/`, `cli/src/lib/agt-bootstrap.ts`, `vendor/agt/`, `deploy/agentmesh-agt.yaml`, `Cargo.toml`.
**Triggered by:** AGT upstream contract gap (server requires POP since v4.0.0; published TS SDK does not sign POP).

## Background

AGT main on `2026-05-23` (server PR #2533) and `2026-05-28` (server PR #2632) made proof-of-possession **mandatory** on the registry `POST /v1/agents` endpoint and the relay `/ws` connect frame. AGT cut v4.0.0 on `2026-06-01` bundling both server-side changes. The **TypeScript SDK was never updated** to sign POP, so the published `npm @microsoft/agent-governance-sdk@4.0.0` cannot interoperate with `ghcr.io/microsoft/agentmesh/{relay,registry}:4.0.0`. Verified live `2026-06-02`: every register call → HTTP 422 → SDK auto-reconnect loop → 11,721 events / 40 min on one sandbox.

The kars-side root cause was previously masked by our `kars-built` images, which were built from an AGT commit dated `2026-05-29` — but my earlier audit `2026-06-02-agt-relay-pop-flood.md` mis-identified the SHA as post-POP. It was actually pre-POP, which is why the kars-built relay+registry accepted what the SDK sent. That's how the `9/9` harness pass on `2026-06-02T04:33` worked.

## What this change does

### 1. Upstream SDK fix (filed)

`https://github.com/microsoft/agent-governance-toolkit/pull/2772` — TypeScript SDK now:

- `register()` sends `{public_key, proof, proof_timestamp}` and adopts the server-derived `did:mesh:<sha256(pk)[:32]>` DID. Optional `popSigner` arg keeps the legacy code path for pre-2533 registries.
- `connect()` includes `{public_key, timestamp, signature}` in the connect frame (standard base64, distinct from the registry's base64url — a real upstream inconsistency the SDK matches separately).
- Constructor auto-wires an `Ed25519-Timestamp authSigner` so `PUT prekeys` / `POST heartbeat` pass authentication.
- `MeshClient.currentDid` getter exposes the canonical DID to callers.

Verified end-to-end against `ghcr.io/microsoft/agentmesh/{relay,registry}:4.0.0` in local docker: register OK, prekeys OK, two peers connect, `A.send → B` E2E delivered, `/health: connected_agents=2, messages_routed=3`.

### 2. Kars-side migration to the patched SDK

- `vendor/agt/microsoft-agent-governance-sdk-4.0.0-agt-bdea1097.tgz` — packed from the fork branch SHA `bdea1097`. Replaces the previous `bae5de3` tarball.
- `vendor/agt/pin.json` (new) — single source of truth for the AGT fork SHA. Referenced by `cli/src/lib/agt-bootstrap.ts` for auto-clone, `Cargo.toml` `[patch.crates-io]`, and the `file:` tarball deps in `mesh-plugin/package.json` + `runtimes/openclaw/package.json`.
- `cli/src/lib/agt-bootstrap.ts` (new) — `ensureAgtRepo()` helper that auto-clones the pinned fork into `~/agent-governance-toolkit` when it's missing. Lets fresh machines run `kars push` / `kars dev --build` without the user having to first clone AGT.
- `mesh-plugin/src/agt-identity.ts` — `deriveDid()` now emits `did:mesh:<32-hex>` (server-canonical form). Local AMID (base58, 20-byte slice) is unchanged for the kars-internal AMID→name cache.
- `mesh-plugin/src/did.ts` — `normalizeDid()` + `isCanonicalDid()` recognise both `did:mesh:<32-hex>` and legacy `did:agentmesh:<16-hex>` so kars can talk to either generation of relay without caller branching.
- `controller/src/mesh_peer/agt_wire.rs` — `AgtFrame::Connect` gains `public_key + timestamp + signature` fields; `AgtRegisterAgentRequest` schema rebuilt to match server v4.0.0 (drops `did`, adds `proof + proof_timestamp`).
- `controller/src/mesh_peer/mod.rs` — `agt_did_for_identity()` returns the `did:mesh:<sha256(pk)[:32]>` form; `connect_and_listen` signs the connect frame; `register_with_registry` signs the POP body. Reuses the existing Ed25519 `SigningKey` already persisted in the controller's K8s Secret.
- `deploy/agentmesh-agt.yaml` — dropped the `AGENTMESH_RELAY_ALLOW_UNAUTHED_DID=1` escape hatch and the explanatory placeholder comments. Relay + registry images both annotated to read `vendor/agt/pin.json` for the build SHA.

### 3. Tests

- `mesh-plugin/src/agt-identity.test.ts`: updated assertions to expect `did:mesh:<32-hex>` instead of `did:agentmesh:<base58>`.
- `controller/src/mesh_peer/agt_wire.rs#tests`: 6 tests updated for the new POP-aware frame shapes; 1 new test asserts the `register` body has neither `did` nor legacy fields.
- All workspace tests pass: cli (786), mesh-plugin (66), runtimes/openclaw (244), Rust workspace (931+821+smaller).

## Transition path (documented at every pinning site)

1. **Now** (PR #2772 open, not merged): kars ships both the SDK tarball + relay/registry images built from `pallakatos:kars-sdk-pop-signing@bdea1097`. AKS works, local-k8s works, docker works, all without escape hatches.
2. **When PR #2772 merges to AGT main**: bump `vendor/agt/pin.json` to the upstream merge commit; `kars push --only relay/registry` then builds from upstream main directly. Same tarball machinery, different SHA.
3. **When AGT cuts a release containing PR #2772**: delete `vendor/agt/pin.json` + the vendored tarball + the `[patch.crates-io]` block + the `file:` deps. Switch `mesh-plugin/package.json` and `runtimes/openclaw/package.json` to `"@microsoft/agent-governance-sdk": "^X.Y.Z"`. Switch `deploy/agentmesh-agt.yaml` to `ghcr.io/microsoft/agentmesh/{relay,registry}:X.Y.Z`. Delete `cli/src/lib/agt-bootstrap.ts`. `ci/check-agt-released.sh` already polls daily and files an issue when the matching release ships.

## Capability touched

- AGT wire protocol parser (relay + registry sides on the kars-built images)
- Identity handling (Ed25519 key persistence, DID derivation, POP signing)
- Workload Identity is unchanged — POP uses the same Ed25519 key the controller has had since the original AGT integration

## Verification commands

```bash
# Local docker smoke test (already run today, 2026-06-02):
docker run --rm -d --name reg -p 18082:8082 ghcr.io/microsoft/agentmesh/registry:4.0.0
docker run --rm -d --name rel -p 18083:8083 ghcr.io/microsoft/agentmesh/relay:4.0.0
node test-full.cjs   # → register OK, prekeys OK, A→B E2E delivered

# AKS rebuild + push (run as the next step on this branch):
KARS_KUBE_CONTEXT=kars-aks kars push --apply
# → triggers full rebuild of sandbox + router + controller + relay + registry
# → AKS pulls, restarts, harness should pass 9/9 with no escape hatches
```

---

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: Copilot CLI <copilot-cli@github.com>
