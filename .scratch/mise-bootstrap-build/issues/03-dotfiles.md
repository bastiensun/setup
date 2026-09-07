# 03 — Dotfiles are symlinked from the repo

**What to build:** every config file this machine needs lands in place from the repo, and
an edit made on the machine shows up as a diff in git. Bootstrap owns 10 files. Nine are
symlinks. One is a copy.

Two files that the playbook appends marker-delimited blocks to become whole tracked files
instead. A third file, the global git config, joins the table and replaces six imperative
commands. All three are new tracked files created by this ticket.

**Blocked by:** 01 — the CI seam must exist before this phase can be proved.

**Status:** ready-for-agent

Work on the shared integration branch. Do not merge to main.

- [ ] The repo's mise config gains a dotfiles table with 10 entries.
- [ ] Symlink is the default mode, so only the one exception names a mode. That
      exception is the simple-bar settings file, which is a **copy**: simple-bar rewrites
      it from its own settings panel, and a symlink would turn every widget tweak into a
      dirty worktree.
- [ ] The Zed settings stay a **symlink**. Zed also writes from its UI, and those edits
      should flow back to git. This is a deliberate difference from the simple-bar call.
- [ ] The fish shell config becomes a whole tracked file carrying three lines: the
      Homebrew shell environment, the starship prompt, and the mise activation. An
      explicit dotfile beats the generated activation entry, so owning the file means
      owning that line.
- [ ] The Ghostty config becomes a whole tracked file. Its Ansible markers become plain
      content.
- [ ] The global git config becomes a whole tracked file holding exactly the 6 keys the
      playbook sets — name, email, editor, signing key, signature format, and sign-by-
      default. Checked while deciding: the live file holds those 6 keys and nothing
      else, so the symlink drops nothing.
- [ ] No edit entries anywhere. Whole files keep one convention, stay forceable, and
      avoid the two cases mise always refuses: corrupted markers, and an edit target that
      is itself a symlink.
- [ ] No directories table. All 9 directory-creation tasks in the playbook exist only as
      the parent of a file the dotfiles phase writes.
- [ ] The Claude settings file is **not** in the table. Claude Code rewrites it, so it
      stays a merge in ticket 05.
- [ ] CI gains the dotfiles status check.
- [ ] Both Ansible jobs still pass.

The full table and the contents of the three new files are written out verbatim in the
decision tickets
[What goes in the [dotfiles] table, and symlink or copy?](../../mise-bootstrap-migration/issues/03-dotfiles-table.md)
and [Write the [tasks.bootstrap] block](../../mise-bootstrap-migration/issues/05-imperative-block.md).
The second adds the git config entry, taking the table from 9 entries to 10.

**Open assumption this ticket answers**: the dotfiles phase creates missing parent
directories. If the CI run shows it does not, the fix is a small directories list — not a
redesign. Record what you find.

**Accepted risk**: any tool that writes the global git config — a credential helper, a
`gh` auth setup, a plain global `git config` — now dirties this repo's worktree.

**Manual step, for the first real Mac run only**: back up the existing global git config
and pass the force-dotfiles flag. It exists as a real file, and mise refuses to replace an
unmanaged real file with a symlink. This does not affect CI, where the file is absent.
