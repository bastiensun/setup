# Replace Ansible with mise bootstrap

## Destination

**Reached.** Every decision needed to replace Ansible with `mise bootstrap` is made and
recorded: the full `mise.toml`, the dotfiles it symlinks, the CI workflow, and the file
sweep. The build itself is a separate effort — `/to-spec` → `/to-tickets` → implement,
in `.scratch/mise-bootstrap-build/`.

The change those decisions describe: Ansible is gone from this repo, a single
`mise.toml` bootstraps a fresh Mac to the state `playbook.yml` produces today, CI is
green on `macos-latest`, and the README tells you to install mise and run
`mise bootstrap`.

## Notes

**Domain**: personal macOS setup repo (`/Users/sun/Documents/setup`). Today
`playbook.yml` does 9 blocks: fish shell + config, Brewfile, mise global config, apm
skills, ponytail config, Claude output style + `settings.json` merge, macOS defaults,
SSH key + git config, and 5 app config files (Ghostty, AeroSpace, simple-bar, Zed).

**No execution override.** The map charted one, then dropped it: the build goes through
`/to-spec` → `/to-tickets` → implement instead. Those tickets get their own effort
directory, `.scratch/mise-bootstrap-build/`, numbered from `01`, so build tickets never
mix with the decision tickets here.

**Skills**: `grilling` + `domain-modeling` for decision tickets; `ponytail` is
active repo-wide — prefer the fewest files and the plainest config.

**Standing preferences** (settled while charting, binding on every ticket):

- Full replacement. No hybrid, no surviving Python toolchain.
- `blockinfile` appends become whole-file symlinked dotfiles in `dotfiles/`.
- Repo `mise.toml` holds the bootstrap tables; `~/.config/mise/config.toml` becomes
  a symlink to `dotfiles/mise-global.toml`.
- Anything without a native bootstrap table becomes a guarded one-liner in
  `[tasks.bootstrap]` — not a shell script. **Amended by ticket 04**: when a phase hook
  such as `[bootstrap.hooks.post-defaults]` fits, use the hook instead.
- pre-commit is deleted entirely. Accepted cost: no more `detect-private-key`.
- `[tasks.default]` is deleted. `mise bootstrap` is the only verb.
- mise is the root dependency. Homebrew is a second one: ticket 02 settled that the
  bootstrap installs brew itself, first, for the AeroSpace cask alone. **Amended by
  ticket 05**: that happens in `[bootstrap.hooks.pre-packages]`, not
  `[tasks.bootstrap]`.
- One PR. CI green on `macos-latest` is the acceptance bar.

## Decisions so far

- [How does mise bootstrap behave on macOS?](issues/01-macos-bootstrap-capabilities.md)
  — bootstrap is stable (not experimental) since 2026.7.4, so the migration is on.
  Three findings bend the plan: the AeroSpace cask **cannot** be installed by mise
  (its tap publishes no API metadata), `defaults -currentHost` is explicitly
  unsupported, and `--dry-run` is not machine-readable (use
  `mise bootstrap status --missing`). Full findings on branch
  `research/mise-bootstrap-macos` at
  [research/mise-bootstrap-macos.md](research/mise-bootstrap-macos.md).
- [Brewfile or [bootstrap.packages]?](issues/02-package-manifest.md) — delete the
  Brewfile; its 20 entries become `[bootstrap.packages]` keys, all `"latest"`.
  `brew "mise"` is dropped. **AeroSpace keeps Homebrew**: two guarded one-liners in
  `[tasks.bootstrap]` install brew (`NONINTERACTIVE=1`) *first*, then the cask. So
  Homebrew is a hard prerequisite after all. `apm`/`gh`/`node` stay in
  `mise-global.toml`; the repo `[tools]` table, `.python-version`, and
  `idiomatic_version_file_enable_tools` are all deleted.

- [What goes in the [dotfiles] table, and symlink or copy?](issues/03-dotfiles-table.md)
  — 9 whole-file entries, no edit entries; `symlink` everywhere except `.simplebarrc`,
  which is `copy` because simple-bar's settings panel rewrites it. `config.fish` and
  the Ghostty config become whole files in `dotfiles/`, and `config.fish` carries
  `mise activate fish | source` itself. All 9 directory-creation tasks are deleted;
  ticket 07 checks the assumption that the dotfiles phase creates parent directories.

- [Translate the macOS defaults and the login shell](issues/04-macos-defaults-and-shell.md)
  — 7 of 8 defaults go declarative in `[bootstrap.macos.defaults]`. The 8th,
  `BatteryShowPercentage`, becomes a `defaults -currentHost write` line in
  `[bootstrap.hooks.post-defaults]`: checked on this machine, the key exists only in the
  ByHost plist, so a plain-domain write is inert. No `killall` — matches today. Login
  shell uses `[bootstrap.user]`; CI runs `mise bootstrap --skip user`, which **binds
  ticket 06**.

- [Rewrite CI for bootstrap](issues/06-ci-rewrite.md) — two jobs; the full workflow
  file is in the ticket. `jdx/mise-action@v4` installs mise, trusts the config, and
  runs the bootstrap in one step via its `bootstrap: true` / `bootstrap_skip: user`
  inputs. Four corrections came out of it: `mise bootstrap status` has **no `--skip`**,
  so idempotence uses four per-part status commands instead of the aggregate one; there
  is no `mise bootstrap validate`, so the lint job greps `mise tasks ls` for
  `unknown field`; `runner.debug` does not resolve in a job-level `env`, so verbosity
  rides the action's `log_level` input; and `dependabot.yml` has no ansible-lint entry
  — the **`pip`** entry is the dead one. `macos-latest` ships Homebrew, so ticket 02's
  guard is a no-op there. The cron survives, `cache: false` on both jobs, and CI
  asserts the imperative block with three `test`/`jq` lines.

- [Write the [tasks.bootstrap] block](issues/05-imperative-block.md) — two places, not
  one. The two Homebrew lines move to `[bootstrap.hooks.pre-packages]`, the only phase
  that runs before mise touches `/opt/homebrew`; this **supersedes ticket 02**, which
  put them in the task. Git config leaves the imperative block and becomes a 10th
  `[dotfiles]` entry, `~/.gitconfig` symlinked from a new `dotfiles/gitconfig` —
  checked: the live file holds exactly those 6 keys. `[tasks.bootstrap]` keeps three
  items: an `ssh-keygen` with an empty passphrase, a 4-line Claude-settings merge that
  creates the file when absent, and today's `mise exec -- apm install --global`. `jq`
  comes from `mise x jq@latest`, not from `[bootstrap.packages]`.

## Not yet specified

Empty. The fog is cleared and the map is done.

The last patch was the **repo hygiene sweep**, and it is now fully specified in
[Execute the migration in one PR](issues/07-execute-migration.md) rather than here:
`pyproject.toml`, `uv.lock`, `requirements.txt`, `requirements.yml`, `ansible.cfg`,
`.python-version`, `.pre-commit-config.yaml`, `dotfiles/Brewfile`,
`dotfiles/caveman.json`, and the `pip` entry in `dependabot.yml` all go;
`dotfiles/apm-coding.yml` stays; `dotfiles/gitconfig` is created.

## Out of scope

- **Writing the diff.** [Execute the migration in one PR](issues/07-execute-migration.md)
  is closed unresolved: the map charted an execution override, then dropped it. The
  build runs as its own effort through `/to-spec` → `/to-tickets` → implement. The
  closed ticket is not dead — it is the assembled brief that feeds the spec, and it
  carries the four calls settled just before closure (simple-bar pins `ref = "main"`,
  the mise floor is a README fact only, `dotfiles/caveman.json` goes and
  `dotfiles/apm-coding.yml` stays, delivery is a branch and a commit).
- Linux or non-macOS support. The destination is this Mac.
- Multi-machine / remote SSH provisioning, though bootstrap supports it.
- **`apm.lock.yaml` / `apm_modules` in the repo root.** Ticket 05 settled the
  `[tasks.bootstrap]` shape, which was the blocker on this fog patch. The answer is
  that it never belonged: `playbook.yml` does not touch these files, so reproducing
  what Ansible produces today does not decide their fate. Same call as
  `dotfiles/caveman.json` and `dotfiles/apm-coding.yml`. A separate cleanup, not this
  migration.
- **`gh auth`.** `playbook.yml` never runs it, so bootstrap does not either. `gh`
  stays a tool in `mise-global.toml` and you authenticate it by hand, exactly as
  today.

*Secrets and first-run interaction* left the fog fully answered rather than as a
ticket. With Ansible gone there is no `--ask-become-pass` and exactly two sudo
surfaces remain, both already named: the Homebrew installer in the `pre-packages`
hook (ticket 05) and `/etc/shells` for the login shell (ticket 04, skipped in CI).
The SSH key takes an empty passphrase (ticket 05).
