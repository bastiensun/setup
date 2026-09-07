# 04 — macOS defaults are written

**What to build:** tap-to-click, the Dock, the menu bar, filename extensions, the Finder
path bar, and the battery percentage all behave as they do today, written from the config
instead of from 8 playbook tasks.

Seven of the eight transcribe declaratively. The eighth cannot: the battery percentage
lives in a per-host preference file, and `-currentHost` is not a modifier on the write —
it selects a different file. Control Center reads only the per-host one. Checked on the
machine: the key exists in the per-host file and does not exist in the plain domain. So a
plain-domain write would create something nothing reads. It becomes a one-line hook in the
defaults phase instead.

**Blocked by:** 01 — the CI seam must exist before this phase can be proved.

**Status:** ready-for-agent

Work on the shared integration branch. Do not merge to main.

- [ ] The repo's mise config gains a macOS defaults table covering 7 keys across 4
      domains: the trackpad, the Dock (two keys), the global domain (two keys), and the
      Finder.
- [ ] A post-defaults hook writes the battery percentage to the per-host domain. It goes
      in that hook and **not** in the bootstrap task: the hook runs inside the defaults
      phase, next to the settings it completes, and does not re-run when the bootstrap
      task runs for other reasons.
- [ ] Nothing restarts the Dock, the Finder, or the menu bar. mise never does, and the
      playbook does not either. These settings take effect at next login, and a fresh Mac
      gets rebooted anyway. This is not a regression.
- [ ] CI gains the macOS defaults status check.
- [ ] Both Ansible jobs still pass.

The full table and the hook line are written out verbatim in the decision ticket
[Translate the macOS defaults and the login shell](../../mise-bootstrap-migration/issues/04-macos-defaults-and-shell.md).

**Open assumption this ticket answers**: the Linux parse gate from ticket 01 must not warn
about an unknown field when it reads macOS tables. If it does warn, move the parse gate
into the macOS job. Do not redesign the tables.
