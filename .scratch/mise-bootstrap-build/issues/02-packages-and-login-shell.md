# 02 — Homebrew packages and casks install through bootstrap

**What to build:** a clean machine gets every application and command-line tool it has
today, from the config rather than from a Brewfile. Bootstrap installs 5 formulae and
15 casks itself. AeroSpace is the one exception: its tap publishes no Homebrew API
metadata, mise verifies casks against that metadata, so mise cannot install it at any
version. A guarded hook installs Homebrew when it is absent, then installs the AeroSpace
cask through it.

The login shell rides along, because fish arrives in this ticket. Bootstrap sets it
natively from an absolute path. CI skips that phase.

**Blocked by:** 01 — the CI seam must exist before this phase can be proved.

**Status:** ready-for-agent

Work on the shared integration branch. Do not merge to main.

- [ ] The repo's mise config gains a packages table: 5 formulae under the `brew:`
      prefix, 15 casks under the `brew-cask:` prefix, every one pinned to latest. That
      matches a Brewfile which pins nothing.
- [ ] The mise binary is **not** among them. mise cannot install its own installer.
- [ ] `jq` is **not** among them either. It is fetched on demand in ticket 05.
- [ ] A pre-packages hook holds two guarded lines, in order: install Homebrew when
      `brew` is absent, then install the AeroSpace cask when the application bundle is
      absent. Use the fully-qualified cask name so the tap is implicit — no separate tap
      line.
- [ ] The hook is `pre-packages` and nothing else. It is the earliest hook and the only
      phase that runs before mise creates and fills the Homebrew prefix. The bootstrap
      task runs far too late.
- [ ] The Homebrew installer runs non-interactively, so CI cannot hang on a
      confirmation prompt.
- [ ] The user table sets the login shell to the Homebrew fish path.
- [ ] CI gains the packages status check and an assertion that the AeroSpace application
      bundle exists. No status command can see a cask that mise did not install.
- [ ] The Brewfile is **not** deleted yet. Ticket 06 owns every deletion.
- [ ] Both Ansible jobs still pass.

The package list and the two hook lines are written out verbatim in the decision tickets
[Brewfile or [bootstrap.packages]?](../../mise-bootstrap-migration/issues/02-package-manifest.md)
and [Write the [tasks.bootstrap] block](../../mise-bootstrap-migration/issues/05-imperative-block.md).
The second supersedes the first on where the Homebrew lines live.

**Known risk**: the AeroSpace tap is untrusted, so the cask install may prompt on a fresh
personal machine. It does not prompt on a runner. The fallback is a one-entry Brewfile
plus `brew bundle`, the only form that carries the trusted flag. Do not build the
fallback now.

**Note**: `macos-latest` ships Homebrew, so the installer guard is always a no-op in CI.
That path never runs there.
