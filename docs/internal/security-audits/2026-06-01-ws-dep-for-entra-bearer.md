# Security Audit — `ws` dep for Entra Bearer WS

**Scope**: `runtimes/openclaw/package.json` — add `ws@^8.18.0` as a
direct dependency.

## Why

Today's `5c65476` added a `wsFactory` to `runtimes/openclaw/src/index.ts`
that uses the `ws` package's `headers` option to attach the AGT
OAuth Bearer token on the relay WebSocket handshake. But `ws` was
NOT declared as a dep of `@kars/runtime-openclaw`, only transitively
present in the workspace root. Result at runtime:

    [plugins] mesh transport init failed: Cannot find module 'ws'
    Require stack: /sandbox/.openclaw/extensions/kars/dist/index.js
    [plugins] @kars/mesh not installed: Cannot find module 'ws' ...

The whole mesh transport was failing → no relay connection → no
verified-tier registration → `auth:entra 0/1 verified` red on the
operator dashboard despite the entrypoint successfully acquiring an
Entra token via `/v1/mesh-token`.

## Fix

Add `ws@^8.18.0` to `runtimes/openclaw/package.json` dependencies.
The sandbox Dockerfile's `npm ci --omit=dev` will install it into
`/opt/kars-plugin/node_modules/ws` so the dynamic `import("ws")` in
`index.ts` succeeds.

## Capability impact

None. The `ws` package is a node-native WebSocket client already
transitively present everywhere — adding it as a direct dep just makes
the wsFactory hook actually work. No new permission surface; no new
network access (mesh transport was attempting WebSocket → relay
regardless, just failing to construct the connection).

## Testing

- `cd runtimes/openclaw && npm install` → 63 packages added, clean.
- `npm run build` → clean (tsc strict).

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: GitHub Copilot CLI <223556219+Copilot@users.noreply.github.com>
