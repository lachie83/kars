# Security Audit — global kubectl context bootstrap

**Scope**: One-commit fix for the "no current kubectl context" papercut
that's bitten the demo flow today across `kars connect`, `kars operator`,
`kars push --apply`, and likely many more (179 kubectl callsites across
the CLI). Touches:

- `cli/src/lib/kube-bootstrap.ts` — NEW, the bootstrap helper
- `cli/src/index.ts` — wires it in at the CLI entrypoint

Both match `cli/src/(commands|migrate|adapters)` regex partially
(`index.ts` is at the top level — does NOT match), but the new
`lib/` file is outside the capability list. The `index.ts` change is
2 lines (import + call). This audit covers both files for
defense-in-depth even though the gate may not require it.

## What it does

At CLI entry (before Commander parses argv) the bootstrap:

1. Skips if `--help` / `-V` / `--version` is in argv (no command will run).
2. Skips if `KUBECONFIG` is already explicitly set by the user.
3. Skips if `kubectl` is not on PATH (dev-only Docker users).
4. Skips if `kubectl config current-context` returns non-empty (the
   common case — fast path, one fork).
5. Else: resolves a reachable context via the existing
   `resolveKubeContext()` helper (probes every kubeconfig context with
   a 3s `kubectl get ns` budget).
6. Writes a temporary kubeconfig at `$(mktemp -d)/config` containing
   the user's existing kubeconfig with `current-context: <resolved>`
   replacing whatever was there (or appending if absent).
7. Sets `process.env.KUBECONFIG=<temp>` — every subsequent kubectl
   invocation in this CLI process inherits via env.
8. Registers a `process.on("exit")` cleanup hook to `rm -r` the temp dir.

## Capability impact

**None — strictly UX.** The bootstrap does not grant any new
permissions, does not modify the user's kubeconfig, does not enable
or disable any kars feature. It just ensures kubectl talks to a
cluster the user has credentials for instead of defaulting to
`http://localhost:8080`.

The temp kubeconfig is 0o600 (user-only readable, like all kubeconfigs)
and lives under `$TMPDIR/kars-kube-XXXXXX/` for the lifetime of the
CLI process only. Cleanup hook removes it on exit.

## Trust boundary

- Reads `~/.kube/config` (user's own file, no escalation).
- Writes a 0o600 file under `$TMPDIR` (user-owned).
- Mutates `process.env.KUBECONFIG` for the current CLI process and its
  spawned children only. Does not affect any other shell or process.

No new outbound network connections beyond what kubectl already does
during the context-probe (`kubectl --context X get ns` with a 3s
timeout per context — the existing `resolveKubeContext` behavior).

## Resilience

- **Race-safe:** if two `kars` invocations run concurrently, each writes
  its own temp dir (`mkdtempSync`); they don't collide.
- **Cleanup-safe:** `process.on("exit")` handler is best-effort. If
  the process is killed -9 the temp dir is leaked but it's ~5 KB and
  in `$TMPDIR` (OS cleans it up periodically).
- **Fork-safe:** child processes (e.g. `kars dev` spawning `docker`)
  inherit `KUBECONFIG` via env — they read the same overlay that the
  parent computed.

## Testing

- `cli npm run build` → clean.
- `cli npm test` → 786 passed | 2 skipped.
- Manual: with `kubectl config current-context` returning
  `"error: current-context is not set"`, ran `kars list` — it
  succeeded by auto-discovering `kars-aks` and listing the demo
  sandbox. Without this fix the same command would have errored
  with "localhost:8080" garbage.

## Conclusion

Safe to merge. Eliminates the most common foot-gun on first-time
fresh-clone AKS demos. Zero new permission surface.

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: GitHub Copilot CLI <223556219+Copilot@users.noreply.github.com>
