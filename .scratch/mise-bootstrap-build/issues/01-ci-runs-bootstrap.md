# 01 — CI runs a real `mise bootstrap`

**What to build:** a fresh macOS runner checks out this repo, runs a real
`mise bootstrap`, and proves the run converged. A second job on Linux parses the config
and fails on an unknown table or key name. This is the walking skeleton: it makes the
test seam exist before any phase depends on it.

The smallest real phase rides along as the payload, so the convergence check has
something to prove — the simple-bar clone into the Übersicht widgets directory, pinned
to its default branch. The playbook pulls that repo on every run today, and pinning the
branch reproduces it. Omitting the ref would clone once and freeze the widget.

The two existing Ansible jobs stay untouched. This ticket adds jobs beside them; it
deletes nothing.

**Blocked by:** None — can start immediately.

**Status:** ready-for-agent

Work on the shared integration branch. Do not merge to main — ticket 06 opens the
single pull request.

- [ ] A Linux job runs the mise action with installation disabled, lists the tasks, and
      fails when the output warns about an unknown field. There is no bootstrap
      validate command, and a bad key name only warns and exits 0 — hence the grep.
- [ ] A macOS job runs the mise action with its bootstrap inputs, skipping the user
      phase. Whether `chsh` succeeds unattended on a runner is unconfirmed, and a hang
      costs a job timeout rather than a fast failure.
- [ ] The macOS job then runs the repos status check and requires nothing missing.
- [ ] Both jobs disable the cache. Neither pins a mise version, so the weekly cron
      finds upstream breakage.
- [ ] Verbosity comes from the action's log level input, not a job-level env block —
      GitHub does not resolve the runner context at job level.
- [ ] The weekly cron, the concurrency group, and the pull-request and push triggers
      all survive from the current workflow.
- [ ] The repo's mise config gains the repos table with one entry, tracking the default
      branch.
- [ ] Both Ansible jobs still pass.

The full workflow file is written out verbatim in the decision ticket
[Rewrite CI for bootstrap](../../mise-bootstrap-migration/issues/06-ci-rewrite.md).
Copy the two new jobs from it. Do not re-derive them. That ticket also records why the
mise action is required rather than optional: without it the checked-out config is
untrusted and every mise command fails.
