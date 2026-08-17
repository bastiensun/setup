# Replace Ansible with `mise bootstrap`

Status: ready-for-agent

Source: the wayfinder map [Replace Ansible with mise bootstrap](../mise-bootstrap-migration/map.md).
Six decision tickets settle everything below. This spec assembles them; the tickets
hold the reasoning and the rejected alternatives.

## Problem Statement

I set up my Mac with an Ansible playbook. To run it I must first install Homebrew,
then create a Python virtualenv, then install Ansible from a pinned requirements file,
then install Ansible collections, then run the playbook with `--ask-become-pass`. That
is five prerequisites before the first thing gets configured.

The cost is not the typing. It is the toolchain:

- A Python toolchain exists in this repo for one reason — to run Ansible. It brings
  `pyproject.toml`, `uv.lock`, `requirements.txt`, `requirements.yml`, `.python-version`,
  and `ansible.cfg` with it. None of it configures my Mac.
- mise is already installed and already manages my tools. It now ships a `bootstrap`
  command that does declaratively what most of the playbook does imperatively:
  packages, dotfiles, macOS defaults, git repos, and the login shell.
- The playbook spends most of its lines on plumbing. Nine of its tasks only create a
  parent directory for the next task. Two more append marker-delimited blocks to files
  that nothing else writes.
- CI must build the same Python toolchain twice to prove the playbook is idempotent.

So the repo carries a whole language runtime and a 240-line playbook to express a
configuration that a single `mise.toml` can hold.

## Solution

A fresh Mac reaches its finished state in two steps: install mise with its official
one-liner, then run `mise bootstrap`.

Ansible leaves the repo completely. There is no hybrid state — the two systems cannot
half-coexist, and no part of the playbook survives. `mise.toml` becomes the single
source of truth: which Homebrew packages to install, which dotfiles to symlink, which
macOS defaults to write, which repo to clone, and which login shell to set. The few
actions with no declarative form become guarded one-liners in a phase hook or in a
`bootstrap` task.

CI proves the same thing it proves today, at the same seam: a real run on a macOS
runner, then an assertion that nothing is left to do.

## User Stories

1. As the owner of a fresh Mac, I want to install mise and run one command, so that I
   do not build a Python toolchain before I configure anything.
2. As the owner of a fresh Mac, I want the README to give me exactly two commands, so
   that I do not read a five-step preamble.
3. As the owner of a fresh Mac, I want the README to name the minimum mise version, so
   that an old mise does not silently skip a table it cannot parse.
4. As the repo maintainer, I want one file to hold the configuration, so that I read
   one file to answer "what does this machine get?".
5. As the repo maintainer, I want no Python in this repo, so that Dependabot stops
   opening pull requests for a toolchain that configures nothing.
6. As the repo maintainer, I want the playbook deleted rather than kept as a fallback,
   so that no one has to decide which of two systems is authoritative.
7. As a Mac user, I want my 5 Homebrew formulae installed, so that `act`, `fish`,
   `mole`, `starship`, and `tldr` are on my PATH.
8. As a Mac user, I want my 15 Homebrew casks installed, so that my applications are
   present without me visiting 15 download pages.
9. As a Mac user, I want AeroSpace installed as a real application bundle, so that the
   window manager runs — not only its command-line binary.
10. As a Mac user, I want Homebrew installed for me when it is absent, so that the
    AeroSpace cask has something to install it.
11. As a Mac user, I want the Homebrew install to skip itself when brew already
    exists, so that a second bootstrap does not reinstall it.
12. As a Mac user, I want fish as my login shell, so that a new terminal opens in fish.
13. As a fish user, I want my `config.fish` to load Homebrew, starship, and mise
    activation, so that my prompt and my tools work in every new shell.
14. As a Mac user, I want my Ghostty theme configured, so that the terminal follows the
    system light and dark modes.
15. As a Mac user, I want my mise global config in place, so that `apm`, `gh`, and
    `node` are available in every directory, not only in this repo.
16. As a Claude Code user, I want my apm global config in place and my skills
    installed, so that my skills are available in every project.
17. As a Claude Code user, I want the ASD-STE100 output style installed and selected,
    so that Claude answers in that style.
18. As a Claude Code user, I want my `settings.json` merged rather than overwritten, so
    that bootstrap does not discard the settings Claude Code writes itself.
19. As a Claude Code user, I want the merge to create `settings.json` when it is
    absent, so that bootstrap works on a machine where Claude Code never ran.
20. As a ponytail user, I want my ponytail config in place, so that the skill reads my
    settings.
21. As an AeroSpace user, I want my `aerospace.toml` in place, so that my window
    layout and key bindings work on the first launch.
22. As a Zed user, I want my Zed settings in place, so that my editor opens configured.
23. As an Übersicht user, I want simple-bar cloned into the widgets directory, so that
    my status bar renders.
24. As an Übersicht user, I want simple-bar tracking its default branch, so that
    bootstrap picks up upstream fixes — which is what the playbook does today.
25. As a simple-bar user, I want my `.simplebarrc` copied and not symlinked, so that
    tweaking a widget from the settings panel does not dirty this repo's worktree.
26. As a repo maintainer, I want every other dotfile symlinked, so that an edit I make
    on the machine shows up as a diff in git.
27. As a Mac user, I want my 7 declarative macOS defaults written, so that tap-to-click,
    the Dock, the menu bar, filename extensions, and the Finder path bar behave as I
    expect.
28. As a Mac user, I want the battery percentage written to the per-host domain, so
    that Control Center actually shows it — the plain domain write is inert.
29. As a Mac user, I want my SSH key generated when it is absent, so that I can push to
    GitHub and sign commits.
30. As a git user, I want my global git config in place, so that my commits carry my
    name, my email, and an SSH signature.
31. As a git user, I want my git config to be a tracked file rather than six imperative
    commands, so that changing it is a diff and CI checks it for free.
32. As the repo maintainer, I want a second bootstrap to report nothing missing, so
    that I can trust the run converged.
33. As the repo maintainer, I want CI to run a real bootstrap on a macOS runner, so
    that a broken config fails in a pull request rather than on my Mac.
34. As the repo maintainer, I want CI to reject an unknown table or key name, so that a
    typo does not silently disable a whole phase.
35. As the repo maintainer, I want CI to assert the effects that no status command can
    see, so that a guard which wrongly skips its own work gets caught.
36. As the repo maintainer, I want CI to skip the login-shell phase, so that a `chsh`
    prompt cannot hang the job until it times out.
37. As the repo maintainer, I want CI to take the latest mise rather than a pinned one,
    so that the weekly scheduled run finds upstream breakage early.
38. As the repo maintainer, I want the weekly cron kept, so that drift in 20 packages
    pinned to `latest` surfaces on a schedule.
39. As the repo maintainer, I want the dead `pip` Dependabot entry removed, so that the
    Dependabot config matches what the repo contains.
40. As the repo maintainer, I want the orphan files swept, so that the repo holds no
    file that nothing reads.
41. As the repo maintainer, I want the xkcd automation comic kept in the README, so
    that the joke survives the migration.

## Implementation Decisions

### Scope and shape

- **Full replacement.** `playbook.yml` and the whole Python toolchain go. No hybrid.
- **One pull request.** Acceptance is CI green on `macos-latest`.
- **mise is the root dependency, and Homebrew is a second one.** mise cannot install
  its own installer, so `brew "mise"` is dropped and the README says to install mise
  first. Homebrew is a hard prerequisite of the AeroSpace cask, so bootstrap installs
  brew itself when it is absent.
- **The repo `mise.toml` holds the bootstrap tables.** `~/.config/mise/config.toml`
  becomes a symlink to the tracked global config.
- **Anything with no declarative table becomes a guarded one-liner** — in a phase hook
  where one fits, otherwise in `[tasks.bootstrap]`. Not a shell script.
- **`[tasks.default]` and `[tasks.lint]` are deleted.** `mise bootstrap` is the only
  verb.
- **pre-commit is deleted entirely.** Accepted cost: no more `detect-private-key`.
  Nothing replaces it. The SSH key lives outside the repo and no step copies it in.

### Packages — ticket [Brewfile or [bootstrap.packages]?](../mise-bootstrap-migration/issues/02-package-manifest.md)

- The Brewfile is deleted. Its entries become `[bootstrap.packages]` keys: 5 formulae
  under the `brew:` prefix, 15 casks under the `brew-cask:` prefix, all pinned
  `"latest"`, which matches a Brewfile that pins nothing.
- **AeroSpace is the one exception and stays on Homebrew.** Its tap publishes no
  Homebrew API metadata, and mise verifies casks against that metadata, so mise cannot
  install it at any version. Rejected: `ubi:`, which extracts binaries and yields the
  CLI without the application bundle.
- mise never shells out to `brew` for the other 20. It pours bottles and installs casks
  itself against the Homebrew prefix.
- `apm`, `gh`, and `node` stay in the tracked global config, not in the repo's
  `[tools]`. Moving them into the repo would make every shell outside this directory
  lose them. The repo `[tools]` table dies with `uv`, and `.python-version` and the
  `idiomatic_version_file_enable_tools` setting go with it.
- **Known risk**: the AeroSpace tap is untrusted, so `brew install --cask` may prompt
  on a fresh machine. The fallback is a one-entry Brewfile plus `brew bundle`, the only
  form that carries `trusted: true`.

### Dotfiles — ticket [What goes in the [dotfiles] table, and symlink or copy?](../mise-bootstrap-migration/issues/03-dotfiles-table.md)

- **Whole files everywhere. No edit entries.** Bootstrap has a marker-based edit form,
  and the spec rejects it: whole files keep one convention, stay forceable with
  `--force-dotfiles`, and avoid the two cases mise always refuses — corrupted markers,
  and an edit target that is itself a symlink.
- The two files Ansible appends to — the fish config and the Ghostty config — become
  whole tracked files. Their Ansible markers become plain content.
- The fish config carries the mise activation line itself. An explicit dotfile beats
  the generated activation entry, so owning the file means owning the line.
- **10 entries. Symlink is the default and applies to 9 of them.**
- **The one `copy` is `.simplebarrc`**, because simple-bar rewrites it from its own
  settings panel and a symlink would turn every widget tweak into a dirty worktree.
  The Zed settings stay a symlink: Zed also writes from its UI, and those edits should
  flow back to git.
- **The Claude `settings.json` is not a dotfile.** Claude Code rewrites it, so it stays
  a merge in the bootstrap task.
- **All 9 directory-creation tasks are deleted.** Every one is only the parent of a
  file the dotfiles phase writes. Assumption to check on the first run: the dotfiles
  phase creates missing parents. If it does not, the fix is a small directories list,
  not a redesign.

### macOS defaults and login shell — ticket [Translate the macOS defaults and the login shell](../mise-bootstrap-migration/issues/04-macos-defaults-and-shell.md)

- 7 of the 8 defaults go in `[bootstrap.macos.defaults]`, keyed by domain.
- **The battery percentage cannot be declarative.** `-currentHost` is not a modifier on
  the write — it selects a different file, and Control Center reads only the per-host
  one. Checked on the machine: the key exists in the per-host plist and not in the
  plain domain. mise states the limit directly. So it becomes a
  `defaults -currentHost write` line in a `post-defaults` hook, which runs in the
  defaults phase next to the settings it completes.
- **No `killall`.** mise never restarts the affected apps, and Ansible did not either.
  These settings take effect at next login, and a fresh Mac gets rebooted anyway.
- **The login shell uses `[bootstrap.user]`**, which takes an absolute path. A sudo
  prompt on the first local bootstrap is acceptable.
- **CI skips the user phase.** Whether `chsh` succeeds unattended on a runner is
  unconfirmed, it runs raw with no sudo wrapper and no prompt automation, and a hang
  costs a job timeout rather than a fast failure. Accepted cost: CI does not cover the
  step most likely to break.

### The imperative surface — ticket [Write the [tasks.bootstrap] block](../mise-bootstrap-migration/issues/05-imperative-block.md)

Two places, not one.

- **A `pre-packages` hook holds the two Homebrew lines**: the guarded installer, then
  the guarded AeroSpace cask. `pre-packages` is the earliest hook and the only phase
  that runs before mise creates and fills the Homebrew prefix. The bootstrap task runs
  far too late for that ordering to hold.
- **`[tasks.bootstrap]` keeps three items**: the guarded SSH keygen, the Claude
  settings merge, and the apm global install.
- **The SSH key takes an empty passphrase**, which matches the Ansible module. The line
  runs unattended, so CI's assertion passes. Minor drift: `ssh-keygen` writes a
  `user@host` comment where Ansible wrote none. Nothing reads it.
- **Git config becomes a symlinked dotfile**, not six imperative commands. This deletes
  six lines and buys free CI coverage, because the dotfiles status check already runs.
  Checked: the live global git config holds exactly those 6 keys and nothing else, so
  the symlink drops nothing. Accepted risk: any tool that writes the global git config
  now dirties this repo's worktree. The Zed settings carry the same trade.
- **The Claude merge is 4 steps, not 1**: create the directory, seed an empty JSON
  object when the file is absent, run the merge into a temporary file, then move it
  into place. The seed is a hard requirement — Claude Code does not exist on a CI
  runner, and CI asserts the merged value. Splitting the merge and the move into two
  array entries beats joining them, because a failing entry then stops the task on its
  own.
- **`jq` is fetched on demand, not installed as a package.** The package list stays at
  20 entries. Checked: the mise registry resolves `jq`.
- **apm keeps today's command.** It rests on an assumption — that the tool-install
  phase picks up the global config the dotfiles phase symlinked earlier in the same
  run. The assumption is self-reporting: if it is false, `apm` is not on PATH, the
  command exits non-zero, and a failing bootstrap task aborts the whole run. CI
  symlinks the same global config, so every CI run exercises it.

### CI — ticket [Rewrite CI for bootstrap](../mise-bootstrap-migration/issues/06-ci-rewrite.md)

The full workflow file is written out in that ticket. Copy it. Do not re-derive it.

- **Two jobs.** A parse gate on `ubuntu-latest`, and a real bootstrap on
  `macos-latest`.
- **The mise action does the install, the trust, and the bootstrap in one step**, using
  its own bootstrap inputs. Without an action the checked-out config is untrusted and
  every mise command fails, so a workflow that drops the action must set the trusted
  paths variable itself.
- **The lint job is a parse gate**, not a linter. There is no bootstrap validate
  command. Listing the tasks exits non-zero on malformed TOML, but an unknown key only
  prints a warning and exits 0 — so the job greps for that warning and fails on it. A
  typo'd table name is the likely bug in a hand-written config and the one that fails
  silently: the phase simply does nothing.
- **Idempotence runs four per-part status commands, not the aggregate one.** The
  aggregate command has no skip flag, so with the user table present and `chsh` skipped
  the login shell reads as differing and the check fails every run. The aggregate tools
  check also reads the global config, which the dotfiles phase symlinked onto the
  runner, so it would start asserting the machine-wide tools too.
- **CI asserts the imperative block** with three checks: the AeroSpace bundle exists,
  the SSH key exists, and the merged Claude setting reads back. No status command can
  see those effects, and the risk is a guard that silently skips its own work.
- **Verbosity rides on the action's log level input.** GitHub does not resolve the
  runner context in a job-level env block, only at step level.
- **No cache on either job**, and **no pinned mise version**. The weekly cron exists to
  find upstream breakage.
- **Homebrew is preinstalled on `macos-latest`**, so the guarded installer is a no-op
  there and CI needs no brew step.
- **Dependabot loses its `pip` entry.** The `github-actions` entry stays and keeps the
  mise action current. There is no ansible-lint entry to remove — that action is
  covered by the `github-actions` entry.

### Repo sweep — ticket [Execute the migration in one PR](../mise-bootstrap-migration/issues/07-execute-migration.md)

- **Delete**: the playbook, the Ansible config, both requirements files, the Python
  project file, the uv lockfile, the Python version file, the pre-commit config, the
  Brewfile, and one orphan dotfile that nothing reads.
- **Keep**: a second orphan dotfile, by explicit decision.
- **Create**: the tracked git config, the fish config, and the Ghostty config.
- **Rewrite**: the CI workflow, the Dependabot config, and the README.
- **The simple-bar clone pins its default branch**, not a bare clone with no ref. The
  playbook pulls on every run today, and pinning the branch reproduces that. Omitting
  the ref would clone once and freeze the widget.
- **The mise version floor is a README fact only.** No minimum-version key in the
  config, and no pin in CI. Accepted cost: an old mise fails late rather than fast.
- The README keeps the xkcd comic and loses the two-step Homebrew-then-virtualenv
  instructions.

## Testing Decisions

**The seam is CI, and it is the only one.** This repo holds no application code, so
there is no unit to test and no module boundary to test at. The one honest seam is a
real `mise bootstrap` on a clean macOS runner, asserted from outside. That is the same
seam the current `integration` job uses, so this replaces prior art rather than
inventing a new pattern.

**What makes a good test here**: it asserts the state of the machine, never the
mechanism that produced it. "The AeroSpace bundle is present" is a good assertion.
"mise ran the pre-packages hook" is not — it tests the implementation, and it would
break the moment a decision moves a line between a hook and a task, which already
happened twice while charting.

The assertions, in three groups:

1. **Parse.** Listing the tasks on Linux must not warn about an unknown field. This
   catches a typo'd table or key name, which otherwise disables a phase in silence.
2. **Convergence.** After a real bootstrap, four per-part status checks must each
   report nothing missing: packages, repos, dotfiles, and macOS defaults. This is the
   idempotence proof, and it replaces today's grep for a changed-and-failed count.
3. **Effects invisible to status.** Three direct checks for the things the imperative
   surface produces, because no status command models them.

**Prior art**: today's `integration` job runs the playbook twice on `macos-latest` and
greps the second run's summary for zero changed and zero failed. The new job keeps the
shape — real run, then prove nothing is left — and changes only how "nothing is left"
is asked.

**Not tested, knowingly**:

- The login shell phase. CI skips it, so the step most likely to break has no coverage.
- The Homebrew installer line. `macos-latest` ships brew, so the guard is always a
  no-op on the runner and the installer path never executes in CI.
- The AeroSpace cask on an untrusted tap. CI installs it, but a prompt on a fresh
  personal machine would not reproduce on a runner.

**Assumptions the first real run must check.** CI cannot reach these, and each has a
known small fix:

1. The dotfiles phase creates missing parent directories. Fix: a directories list.
2. The Claude merge creates the settings file when absent. CI does assert the merged
   value, so this one fails loudly if wrong.
3. The macOS defaults tables parse on Linux without a warning. Fix: move the parse gate
   into the macOS job.

One manual step on the first real run: back up the existing global git config and pass
`--force-dotfiles`. It exists as a real file, and mise refuses to replace an unmanaged
real file with a symlink.

## Out of Scope

- **Linux or non-macOS support.** The destination is one Mac.
- **Multi-machine or remote SSH provisioning**, though bootstrap supports it.
- **The apm lockfile and the installed modules directory in the repo root.** The
  playbook does not touch them, so reproducing what Ansible produces today does not
  decide their fate. A separate cleanup.
- **`gh auth`.** The playbook never runs it, so bootstrap does not either. `gh` stays a
  tool in the global config and you authenticate it by hand, exactly as today.
- **Running the bootstrap on the real Mac.** That happens after merge. Anything it
  surfaces is a follow-up, not a blocker.
- **Replacing `detect-private-key`.** pre-commit goes and nothing takes its place.
- **A local smoke-check task** wrapping the CI assertions. Rejected as a second seam to
  keep in sync.

## Further Notes

**Secrets and interaction.** With Ansible gone there is no become-password flag, and
exactly two sudo surfaces remain: the Homebrew installer in the pre-packages hook, and
the login-shell change. CI reaches neither — brew is preinstalled, and the user phase
is skipped. The SSH key takes an empty passphrase, so it needs no prompt.

**Version floors**, for the README: bootstrap is stable rather than experimental from
one version, and per-package OS filters need a later one. The machine that charted this
map runs a version above the first floor and below the second. No table in this spec
uses an OS filter, so only the lower floor binds.

**Two decisions were superseded while charting.** Both are already corrected above, and
the ticket text that states the old position is left in place as a record:

1. The Homebrew lines moved from the bootstrap task into the pre-packages hook.
2. The dotfiles table grew from 9 entries to 10 when the git config joined it.

**The ordering that matters.** The dotfiles phase runs before the tool-install phase,
which runs before the bootstrap task. Two decisions rest on that order: the global mise
config must be symlinked before tools install, and the apm config must be symlinked
before apm installs skills. Both fail loudly rather than silently if the order is
wrong.
