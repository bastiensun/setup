# 06 — Ansible leaves the repo

**What to build:** the destination. A fresh Mac reaches its finished state in two steps —
install mise with its official one-liner, then run `mise bootstrap`. Nothing in the repo
mentions Ansible, Python, or a virtualenv.

This is the contract half of the migration. Tickets 01–05 added the whole bootstrap
surface beside the playbook and proved each phase in CI. This ticket removes the old
system, and it is the integrate-and-verify: it opens the single pull request.

**Blocked by:** 02, 03, 04, 05 — every phase must be proved in CI before the playbook can
go.

**Status:** ready-for-agent

- [ ] Delete the playbook and the Ansible config.
- [ ] Delete the whole Python toolchain: both requirements files, the Python project
      file, the uv lockfile, and the Python version file.
- [ ] Delete the pre-commit config. **Nothing replaces `detect-private-key`** — the SSH
      key lives outside the repo and no step copies it in. This is an accepted cost, not
      an oversight.
- [ ] Delete the Brewfile. Ticket 02 moved all 20 entries into the packages table, and
      dropped the mise entry.
- [ ] Delete the one orphan dotfile that nothing reads — the caveman config. Checked:
      nothing in the repo references it. **Keep the apm coding config**, by explicit
      decision, even though nothing references it either.
- [ ] Remove the repo's `[tools]` table, which holds only `uv`, and the idiomatic version
      file setting that exists only for Python.
- [ ] Remove the default task and the lint task. `mise bootstrap` is the only verb.
- [ ] Delete both Ansible CI jobs. The two jobs from tickets 01–05 are the whole workflow.
- [ ] Remove the `pip` entry from the Dependabot config. The `github-actions` entry stays
      and keeps the mise action current. There is **no ansible-lint entry** to remove —
      that action is covered by the `github-actions` entry.
- [ ] Rewrite the README: install mise with its official one-liner, then run
      `mise bootstrap`. Name the minimum mise version as prose — bootstrap is stable
      rather than experimental only from a certain version. Do **not** add a
      minimum-version key to the config and do **not** pin mise in CI.
- [ ] Keep the xkcd automation comic.
- [ ] The whole workflow is green on the macOS runner.
- [ ] Open the pull request from the integration branch.

**Record in the pull request description**, so the follow-up work is visible:

1. What the first real Mac run still needs — back up the existing global git config and
   pass the force-dotfiles flag, because mise refuses to replace an unmanaged real file
   with a symlink.
2. Whether the dotfiles phase created missing parent directories, from ticket 03.
3. Whether the macOS tables parsed on Linux without a warning, from ticket 04.

**Two sudo surfaces remain** after Ansible goes, and both are known: the Homebrew
installer in the pre-packages hook, and the login shell change. CI reaches neither — the
runner ships Homebrew, and the user phase is skipped.

**Out of scope for this ticket**: the apm lockfile and the installed modules directory in
the repo root. The playbook does not touch them, so reproducing what Ansible produces
today does not decide their fate. That is a separate cleanup.
