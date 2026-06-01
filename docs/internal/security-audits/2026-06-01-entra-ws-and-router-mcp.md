# Security Audit — Entra-verified WS + always-on router MCP

**Scope**: Two fixes for the AKS demo flow:

1. `runtimes/openclaw/src/index.ts` — pass a `wsFactory` that attaches
   `Authorization: Bearer ${AGT_OAUTH_TOKEN}` on the WebSocket Upgrade
   to the AGT relay. The AGT SDK's default WebSocket constructor
   sends no headers, so the relay's `entra_verify_enabled` path
   counted every peer as unverified even when registry-tier upgrade
   succeeded. Now `verified_agents` actually increments.

2. `sandbox-images/openclaw/entrypoint.sh` — always register the
   loopback router itself as a `kars-router` MCP source (header-less).
   Was only registering EXTERNAL McpServer CRs. Without this, the
   agent had no built-in tool access (memory, foundry, etc) unless
   the user manually authored an McpServer pointing at their own
   sandbox router — circular.

## Capability impact

**Fix #1**: Adds a Bearer header on the WS Upgrade. The token is the
SAME OAuth token already used for registry tier-upgrade (acquired via
the auth-sidecar's `DownstreamApis__AgentMesh__Scopes__0` =
`<agentmesh-client-id>/.default`). No new permission surface — just
makes the relay's existing verification path actually see the token.

**Fix #2**: Routes localhost:8443/mcp through the existing platform-MCP
sub-router which already enforces every policy gate (tool allow-lists,
rate-limits, content safety, audit). The agent gains visibility into
the router's tool catalog — but every call still passes through the
router's policy stack, identical to today's external-McpServer flow.

## Trust boundary

No change. Both fixes use existing primitives:
- Fix #1: token already minted by the auth-sidecar, already used for
  the registry. Just sent over one more transport.
- Fix #2: localhost loopback through the in-pod router. The
  egress-guard iptables baseline already permits UID 1000 to reach
  127.0.0.1:8443.

## Testing

- `cli npm run build` clean; 786 cli tests pass.
- `openclaw plugin` build clean (tsc strict).
- Verified on live cluster: `kubectl exec -n agentmesh deploy/relay --
  curl localhost:8083/health` shows `verified_agents=0` today — after
  rolling out the new sandbox image, expect that count to match
  `connected_agents` for sandboxes with `MESH_AUTH_BACKEND=EntraAgentIdentity`.

## Conclusion

Both fixes restore advertised behavior (Entra verification + built-in
router MCP) without adding capability. Safe to merge.

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: GitHub Copilot CLI <223556219+Copilot@users.noreply.github.com>
