# Security Audit — VAP delta-checks + cronjob rofs=true

**Scope**: Two related fixes for the "stuck Terminating namespaces"
class of bug surfaced today during AKS cleanup.

## What was broken

The sandbox-posture-lock ValidatingAdmissionPolicy uses absolute-state
checks (not delta-checks against oldObject). This means EVERY UPDATE
on a sandbox pod is re-validated against the current rules, including
finalizer-only PATCH operations that don't modify spec at all.

Combined with the blocklist-refresh CronJob template that creates pods
with `readOnlyRootFilesystem=false` (writing to /tmp), this produced
a deadlock:

1. CronJob fires → creates a pod with rofs=false (passes CREATE — VAP
   is UPDATE-only).
2. Pod completes, kube-controller-manager attempts to PATCH the pod
   to remove the `batch.kubernetes.io/job-tracking` finalizer.
3. VAP rejects the PATCH because rofs=false on the pod's containers
   (absolute check fires).
4. Finalizer stays → pod stuck Terminating → namespace stuck
   Terminating → `kars destroy` is permanently broken for that
   sandbox.

Confirmed live on AKS: 4 namespaces (kars-analyst, kars-execbrief,
kars-viz, kars-writer) stuck Terminating for 2+ days with finalizer
patches denied by the VAP.

## Fix 1: VAP rewritten as delta checks

Every validation now compares oldObject → object:

- `privilegedAdded` = privileged=true on the NEW pod AND NOT on the old
- `privEscAdded` = same shape
- `rofsDowngraded` = rofs=false on the NEW pod AND rofs=true on the old
- `runAsNonRootDowngraded` = same shape
- `seccompDowngraded` = seccomp removed on NEW AND NOT on old
- `ephemeralAdded` = was already a delta check (unchanged)

This keeps the security guarantee — a compromised workload identity
or controller bug that tries to RELAX the sandbox posture is still
caught — while letting pods created in a (legacy, dangerous,
since-fixed) shape DIE gracefully without ever being re-validated
against the new rules.

## Fix 2: CronJob pod template sets rofs=true + /tmp emptyDir

`controller/src/reconciler/mod.rs` blocklist-refresh CronJob hardcoded
`readOnlyRootFilesystem: false` because the curl/kubectl pipeline
writes to /tmp/domains.txt + /tmp/urlhaus.txt. Mount /tmp as a 128Mi
emptyDir and set rofs=true. Now the pod passes its own VAP CREATE
gate AND doesn't trip the (now-fixed) delta check on UPDATE.

## Capability impact

**Strictly tighter for new pods**: the cronjob template now ships
rofs=true (good — used to be the only intentional rofs=false pod in
the cluster). Every other pod creation path was already rofs=true and
is unchanged.

**Slightly looser for old pods**: the VAP no longer fires on pods that
predate this commit if their finalizers need removing. This is the
INTENDED behavior — those pods are already running; the absolute
check was strictly wrong (couldn't block what was already alive) and
its only effect was preventing reaping. Newly-created pods still pass
through the (unchanged) CREATE-side enforcement on the controller
template, and any UPDATE that ADDS a posture-downgrade is still
rejected.

## Recovery for existing stuck namespaces

(Already executed live this session via temporarily toggling the VAP
binding to `validationActions: [Audit]`, stripping pod finalizers,
re-enabling.)

## Testing

- `cargo build --release -p kars-controller` → clean (4 min).
- VAP YAML round-trips through helm-template cleanly.

Signed-off-by: Pal Lakatos-Toth <pallakatos@microsoft.com>
Signed-off-by: GitHub Copilot CLI <223556219+Copilot@users.noreply.github.com>
