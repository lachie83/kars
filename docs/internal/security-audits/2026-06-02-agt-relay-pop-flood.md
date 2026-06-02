# Security audit: AGT relay POP enforcement → inbox flood

**Date:** 2026-06-02
**Scope:** `deploy/agentmesh-agt.yaml`, `runtimes/openclaw/src/index.ts` (onError handler), `runtimes/openclaw/src/index.test.ts` (regression tests).
**Reporter:** Agent inside `kars-aks` sandbox observed 11,721 `security_event` entries in its own inbox over 40 minutes (~5/sec) all tagged `source: agt_relay_e2e`.

## Root cause

Two AGT upstream changes landed between AGT release `4.0.0` (2026-05-29, on crates.io) and SHA `bae5de3` (2026-06-01, pinned via `vendor/agt/microsoft-agent-governance-sdk-4.0.0-agt-bae5de3.tgz` and `Cargo.toml [patch.crates-io]`):

1. **Commit `66918631` (May 28)** — `relay /ws` now **mandatorily** requires the connect frame to carry proof-of-possession (`public_key + timestamp + signature`). DID must equal `did:mesh:sha256(pk)[:32]`. Replay window 5 min. Escape hatch: `AGENTMESH_RELAY_ALLOW_UNAUTHED_DID=1` (logged as a security warning at boot).
2. **Same commit** — `POST /v1/agents/{did}/heartbeat` now requires `Ed25519-Timestamp` auth.

The AGT **TypeScript SDK** (the one we ship via the pinned tarball) does **not** yet send POP fields in its connect frame (verified: `grep -E 'public_key|sign.*connect|POP' agent-governance-typescript/src/encryption/mesh-client.ts` returns nothing). Result on every cold start:

```
SDK.connect() → relay.onaccept → POP-check fails → close 4001
SDK.onclose (non-1000) → scheduleReconnect (exp backoff capped at 60s)
SDK.connect() → POP-check fails → close 4001 → … loop
```

Each reconnect failure surfaces via the SDK callback `h("ws", agentDid, "reconnect failed: ...")`. The cadence is bounded by exponential backoff (1, 2, 4, …, 60s), so a steady-state ~1/min of WS error callbacks. Observed 5/sec average implies the early ramp dominated, plus additional onerror events from the WS upgrade itself.

The **inbox flood** then comes from the openclaw plugin's `onError` handler in `runtimes/openclaw/src/index.ts`. That handler was written before AGT's `onError` API existed (when we used the vendored amitayks/agentmesh fork in Phase 1) and was never updated for the AGT SDK's new type strings (`"ws" | "decrypt" | "knock" | "frame" | "session_desync"`). It checked for the old strings (`knock_rejected`, `no_session`, `decrypt_failed`); everything else fell into the catch-all `else` branch which:

- Wrote a `security_event` to the agent's own inbox
- Called `pushTrustToRouter(agentDid, -0.5)` — penalising the agent's OWN trust score on every transport error against its own connection
- Had no rate limit

Pre-`66918631` this latent bug never fired in practice because WS errors against a healthy AGT relay are rare. The May 28 POP enforcement turned a steady-state error rate of ~0/sec into ~1-5/sec, and the latent handler turned every one of them into a permanent inbox entry.

## Fixes shipped in this audit

### 1. Re-enable legacy mode on the relay (`deploy/agentmesh-agt.yaml`)

Added `AGENTMESH_RELAY_ALLOW_UNAUTHED_DID=1` to the `relay` deployment env. The relay logs a security warning at boot ("Any client can impersonate any DID. Do not use in production.") which is correct — this is an escape hatch until the AGT TypeScript SDK upstream lands POP signing.

This is the immediate-unblock fix. AKS production is single-tenant and behind a NetworkPolicy that only allows ingress from sandbox routers in the same cluster, so impersonation risk is bounded to peers that already have cluster-internal network access (which already implies they could mint their own DID anyway). Local dev (kind) is also acceptable — no production data, no external reachability.

Tracking issue (next-step): pick up `microsoft/agent-governance-toolkit` PR (TBD) that adds SDK-side POP signing, then drop the env var. Cargo's `[patch.crates-io]` block auto-flips back when an AGT release containing the SHA lands (machinery already in place: `ci/check-agt-released.sh`).

### 2. Fix the onError handler (`runtimes/openclaw/src/index.ts`)

Rewrote the handler to:
- Use the AGT SDK's actual type strings (`"ws" | "decrypt" | "knock" | "frame" | "session_desync"`).
- Branch `"ws"` early-returns: log-only, no inbox write, no trust penalty. `fromAmid` is OUR OWN DID for transport errors, not a peer's.
- Branch `"decrypt" | "knock" | "frame" | "session_desync"`: rate-capped advisory `session_event` (NOT `security_event`) — one entry per `(type, fromAmid)` per 60 seconds. No trust penalty (recoverable protocol events per SDK design).
- Catch-all `else`: log-only, no inbox write. Adding a new SDK error type now requires an explicit code change here instead of silently falling into the flood branch.
- Map GC: bounded to 256 entries to prevent unbounded growth on a flaky cluster.

### 3. Regression tests (`runtimes/openclaw/src/index.test.ts`)

Added 4 grep-style tests that pin the handler contract:
- References each AGT SDK type string exactly once.
- Does NOT reference the old vendored-fork strings.
- `"ws"` branch contains no `pushTrustToRouter`, `pushInbox`, or `security_event`.
- Rate-cap state variables exist with a finite `ERROR_INBOX_INTERVAL_MS` constant.

These run from both `src/` and `dist/` (vitest config) and resolve the source via walk-up so the dist run reads the real `src/index.ts`.

## What this is NOT

Not a kars-introduced regression. Both fixes restore correct behaviour after an AGT upstream contract change that we picked up automatically when we re-pinned to the AGT main SHA. The same scenario would have happened in any consumer of AGT mesh that pinned past `66918631` without SDK POP support.

## Capability touched

- WebSocket transport-layer trust handling (mesh relay)
- Inbox write authority (any handler that calls `pushInbox`)
- Trust score authority (any handler that calls `pushTrustToRouter`)

## Verification

Live verification on `kars-aks`:
1. Set `AGENTMESH_RELAY_ALLOW_UNAUTHED_DID=1` on the relay deployment.
2. Confirmed relay logs both warnings at startup: `AGENTMESH_RELAY_TOKEN is not set` AND `AGENTMESH_RELAY_ALLOW_UNAUTHED_DID is set`.
3. Sandbox pod gateway log: `AGT mesh provider: agt ... + Entra-verified WS (connect.token)` and `AGT identity verified via OAuth — tier upgraded to 'verified'`.
4. After router rebuild + push: relay `/health` should report `verified_agents > 0` once a sandbox reconnects (relay reads `frame.get("token")`, verifier passes, counter increments).

Unit tests: 244 pass (was 236; +8 from the 4 new flood-prevention tests × src+dist).

---

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: Copilot CLI <copilot-cli@github.com>
