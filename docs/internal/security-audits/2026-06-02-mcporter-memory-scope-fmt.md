# Security Audit — mcporter install + memory scope + cargo fmt

**Scope**: Three commits since the last audit:

- `c7c3fa4` — `sandbox-images/openclaw/Dockerfile.base`: drop the
  `2>/dev/null || true` mask on `npm install -g clawhub mcporter
  @steipete/oracle`. Install failures now fail the build instead of
  silently shipping an image without mcporter.
- `e3e3c25` — `runtimes/openclaw/src/core/memory-binding.ts`: change
  default memory scope from `agent:<sandbox>` to `agent_<sandbox>`
  to satisfy Foundry Memory Store's scope character allowlist
  (`letters/digits/_/-/./%/+/@//` — no colon).
- (this commit) `controller/src/tool_policy_reconciler.rs`: cargo
  fmt of the ToolPolicy.watches(KarsSandbox) mapper from `14ad44e`.

## Capability impact

**None for any of the three:**

- mcporter install fix: makes a NETWORK-DEPENDENT step (npm fetch)
  observable instead of silent. Doesn't add any new package — same
  three packages were always intended. Build-time failure is strictly
  better than runtime missing-module errors.
- Memory scope change: pure string-format change. The scope is a
  client-side label for the Foundry memory store; the change just
  uses a separator Foundry allows. New scope `agent_demo` has IDENTICAL
  isolation semantics to the old `agent:demo` (per-agent namespace).
- cargo fmt: whitespace only, no logic change.

## Trust boundary

No change. mcporter (when actually installed at build time) is a
runtime CLI invoked by the openclaw gateway during MCP session setup.
It runs under the agent UID (1000) and is subject to all existing
egress + tool policy gates. Adding it doesn't grant any new
permissions — without it, MCP doesn't work at all; with it, MCP works
within the same security envelope as every other plugin.

## Testing

- `cargo fmt --all --check` → clean.
- `cargo build --release -p kars-controller` → clean.
- `cli npm run build` → clean; 786 tests pass.
- Verified live on AKS: `require.resolve('mcporter')` fails inside
  the current pod (confirming the bug); `npm view mcporter` confirms
  the package is live on npm and the build-time install will succeed.

## Conclusion

Safe to merge. All three are pure correctness fixes with zero new
capability surface.

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: GitHub Copilot CLI <223556219+Copilot@users.noreply.github.com>
