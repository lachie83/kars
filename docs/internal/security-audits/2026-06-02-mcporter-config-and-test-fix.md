# Security Audit — mcporter config file + governance default test fix

**Scope**: Two commits + roll-up audit for the day's earlier tools.*
churn.

## Changes

1. **`controller/src/crd.rs`** — Update `default_governance_config()`
   test to assert the new defaults from fa97c68 (governance enabled
   by default, empty toolPolicyRef falls back to kars-default). The
   test was still asserting the old `enabled=false` shape and failing
   CI.

2. **`sandbox-images/openclaw/entrypoint.sh`** — Write a sibling
   `/sandbox/.mcporter/mcporter.json` with the Claude-style
   `{"mcpServers": {...}}` shape. mcporter (the runtime the openclaw
   bundle-mcp materializer spawns to actually connect to MCP servers)
   reads its OWN config file, not openclaw.json's nested
   `mcp.servers` block. Without this, mcporter reports
   "No MCP servers configured" and the LLM never sees any callable
   MCP tools regardless of openclaw config.

3. **Roll-up audit** of today's tools.* churn (commits c283109,
   793acfe, bef57fc, 6dd0767 — all reverted-to-or-iterating-on the
   minimal `tools` block). Final state at 6dd0767: deny
   sessions_spawn/sessions_send, exec security=full, NO profile
   override. Default profile "full" gives the agent its built-in
   tools; bundle-mcp materializes automatically per-session when
   mcp.servers is set + mcporter is present + toolsAllow is not
   restricting.

## Capability impact

None for either commit:

- Test fix is just an assertion update to match new defaults
  (which themselves were already audited in
  2026-06-01-governance-default-on-and-reconciler-watches.md).
- The mcporter config file is a different on-disk format of THE
  SAME server list already written to openclaw.json. Same endpoints,
  same headers, same OAuth/bearer config. mcporter is the
  npm-installed runtime that openclaw's bundle-mcp materializer
  invokes; it inherits the same egress allowlist + AGT trust gates
  as every other process in the openclaw container.

## Testing

- `cargo test --release -p kars-controller default_governance_config` → 1 passed.
- cli build clean; 786 tests pass.

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: GitHub Copilot CLI <223556219+Copilot@users.noreply.github.com>
