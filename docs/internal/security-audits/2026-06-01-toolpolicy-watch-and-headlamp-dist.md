# Security Audit — ToolPolicy.watches(KarsSandbox) + commit headlamp dist

**Scope**: Two follow-up fixes:

1. `controller/src/tool_policy_reconciler.rs` — add
   `.watches(KarsSandbox)` mapper. Mirrors the same fix that landed
   in fa97c68 for KarsMemory + InferencePolicy. The system-default
   `kars-default` ToolPolicy was stuck `PolicyNotEnforced` for 20+ min
   on the user's local-k8s cluster because the reconciler only re-ran
   on its own 5-min periodic sweep, missing sandbox-create events.

   The mapper also handles the empty-toolPolicyRef → kars-default
   fallback we landed in fa97c68, so a sandbox that omits the ref
   still triggers the kars-default reconciler.

2. `tools/headlamp-plugin/` — commit the pre-built `dist/main.js`
   (47 KB) and untrack `dist/` in `.gitignore`. The headlamp install
   path tries `npm run build` when `dist/main.js` is missing, but
   `@kinvolk/headlamp-plugin@0.13.0` (the pinned version) uses an
   older `yargs` that's incompatible with Node.js 26 ESM detection:

       ReferenceError: require is not defined in ES module scope
       at .../yargs/yargs:3:69

   Bumping headlamp-plugin to 0.14.0 is the long-term fix but
   requires testing API compat. Committing the dist sidesteps the
   build entirely — same pattern used for `vendor/sandbox-wheels/`.

## Capability impact

None for either change:

- ToolPolicy watch is a latency reduction — the reconciler eventually
  promoted to Ready on its 5-min sweep anyway. Just makes it
  near-instant on sandbox events.
- The committed dist is the output of the existing build pipeline. No
  new code shipped; just shipping the artefact so users don't need a
  matching Node version to regen it.

## Testing

- `cargo build --release -p kars-controller` → clean (4 min cold).
- `tools/headlamp-plugin/dist/main.js` is 47 KB, the same binary
  produced by `npm run build` on Node 22.
- After commit, fresh-clone `kars headlamp --install` will see
  dist/main.js exists, skip the build, and apply the ConfigMap
  successfully on Node 26.

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: GitHub Copilot CLI <223556219+Copilot@users.noreply.github.com>
