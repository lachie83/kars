# Security Audit — kube-context helper + AGT preflight

**Scope**: Two follow-up commits to today's governance defaults change
(`fa97c68`), both touching `cli/src/commands/` (capability-list
matches in `ci/security-audit-required.sh`):

- `8a23f4e` — refactor: extracted `cli/src/lib/kube-context.ts`
  out of `cli/src/commands/operator.ts` to keep operator.ts under its
  LOC cap. **No behavior change.** Identical auto-discover semantics,
  just moved out of operator.ts and called via dynamic import.

- `3b61e14` — feat: `cli/src/commands/dev.ts` `preflightTools()` now
  also checks for the agent-governance-toolkit clone and surfaces a
  copy-pasteable `git clone` + `$KARS_AGT_REPO` command when missing.
  **UX only** — the same hard-fail used to happen 5 minutes later
  during the mesh-image docker build. Earlier, clearer error.

## Capability impact

None for either commit:

- The kube-context helper is a pure-read auto-discovery (`kubectl
  config get-contexts` + `kubectl --context <c> get ns`). No mutations,
  no new privileges. Reduces UX confusion when no current-context is
  set.
- The AGT preflight just runs `existsSync()` on a path and prints a
  message. No mutation, no network.

## Trust boundary

Both changes execute on the developer workstation under the user's UID.
No new IPC, no new outbound network beyond what kubectl/`existsSync`
already do.

## Testing

- `cli npm run build` → clean.
- `cli npm test` → 786 passed | 2 skipped.

## Conclusion

Safe to merge. Both commits reduce time-to-error in fresh-clone
flows without altering any capability surface.

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: GitHub Copilot CLI <223556219+Copilot@users.noreply.github.com>
