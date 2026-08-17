# Rewrite CI for bootstrap

Type: grilling
Status: closed
Assignee: sun
Blocked by: 01

## Question

`.github/workflows/ci.yml` has a `lint` job (ansible-lint + uv-run pre-commit) and an
`integration` job that runs the playbook twice on `macos-latest`, grepping
`changed=0.*failed=0` for idempotence. Both die with Ansible.

Settled: pre-commit and ansible-lint are **deleted outright**, so the `lint` job goes
away entirely. Accepted cost — no more `detect-private-key`, in a repo that generates
SSH keys. Decide whether anything cheap replaces that one guard.

Settled: the idempotence check runs after a real bootstrap and requires it to report
nothing left to do. Ticket 01 **retargeted** which command does it — `--dry-run` has
no `--json`, no sentinel, and no exit-code contract. Use
`mise bootstrap status --missing` (exit 1 when something is missing, `--json`,
covers the whole declarative surface). `mise bootstrap plan --json
--detailed-exitcode` is better structured but does not yet cover dotfiles, defaults,
repos, or the login shell — check whether that changed before choosing it.

Note also: `[tasks.bootstrap]` never converges and is skipped under `--dry-run`, so
no status command will ever check the imperative block from ticket 05. Decide whether
CI needs a separate assertion for those four effects.

Decide: how the runner gets mise, whether the `schedule:` weekly cron survives,
whether `RUNNER_DEBUG` still maps to a verbosity flag, and what a genuine failure
looks like now that no task-level `changed`/`failed` counters exist.

Update `.github/dependabot.yml` — its ansible-lint action entry becomes dead.

## Added by ticket 02

Homebrew is now a hard prerequisite of `[tasks.bootstrap]`, not an optional one.
Check whether `macos-latest` still ships brew preinstalled; if it does, the guarded
installer line is a no-op on CI and the job needs nothing extra. If it does not, the
`curl | bash` install runs on every CI run — decide whether to cache it or to install
brew as an explicit workflow step instead.

Also check that the second bootstrap run reports nothing missing even though the
AeroSpace cask is installed by a shell line, not by mise — `mise bootstrap status
--missing` cannot see it, so idempotence rests on the `test -d /Applications/...`
guard.

## Resolution

Two jobs. `jdx/mise-action@v4` does the install, the trust, and the bootstrap in one
step. The idempotence check runs four per-part status commands, not the aggregate one.

```yaml
name: CI

on:
  pull_request:
  push:
    branches:
      - main
  schedule:
    - cron: 0 0 * * 0

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: jdx/mise-action@v4
        with:
          install: false
          cache: false
          log_level: ${{ runner.debug && 'debug' || 'info' }}
      - name: Check mise.toml
        run: |
          mise tasks ls 2>&1 | tee parse.log
          ! grep --quiet 'unknown field' parse.log

  bootstrap:
    name: Bootstrap
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v7
      - uses: jdx/mise-action@v4
        with:
          bootstrap: true
          bootstrap_skip: user
          cache: false
          log_level: ${{ runner.debug && 'debug' || 'info' }}
      - name: Idempotence check
        run: |
          mise bootstrap packages status --missing
          mise bootstrap repos status --missing
          mise bootstrap dotfiles status --missing
          mise bootstrap macos defaults status --missing
      - name: Check the imperative block ran
        run: |
          test -d /Applications/AeroSpace.app
          test -f ~/.ssh/id_ed25519
          jq -e '.outputStyle == "ASD-STE100"' ~/.claude/settings.json
```

### Facts checked while deciding

Against mise 2026.7.5 on this Mac, the `jdx/mise-action` source, and the GitHub
runner images.

- **`macos-latest` ships Homebrew.** The `command -v brew` guard from ticket 02 is a
  no-op on the runner, so CI needs no brew step.
- **`mise-action` trusts the config itself.** `src/index.ts:360` sets
  `MISE_TRUSTED_CONFIG_PATHS` to the working directory, and `MISE_YES=1` next to it.
  Without an action, `actions/checkout` leaves an untrusted config and every mise
  command fails with "Config files ... are not trusted". A workflow that drops the
  action must set that variable itself.
- **There is no `mise bootstrap validate`.** `mise tasks ls` is the parse gate. It
  exits 1 on malformed TOML, but an unknown key only prints
  `mise WARN unknown field` and exits 0 — hence the `grep`.
- **`mise bootstrap status` has no `--skip`.** Its flags are `-J`, `--missing`, `-C`,
  `-E`, `-j`, `-q`. With `[bootstrap.user]` in the config and `chsh` skipped on the
  runner, `login_shell.state` reads `"differs"` and `--missing` exits 1 every run.
- **The aggregate `tools` array reads the global config.** The dotfiles phase
  symlinks `~/.config/mise/config.toml` on the runner, so the aggregate check would
  start asserting `apm`, `gh`, and `node` as well.
- **`mise-action` latest tag is `v4.2.5`**, not `v3`.

### The decisions

1. **Runner gets mise from `jdx/mise-action@v4`** (Q1a), and the action runs the
   bootstrap through its own `bootstrap: true` / `bootstrap_skip: user` inputs (Q10a)
   — one step rather than two. `bootstrap: true` also sets `MISE_EXPERIMENTAL=1`,
   which is harmless now that the gate is gone.
2. **The `lint` job survives, on `ubuntu-latest`, as a config-parse gate** (Q2c, Q8b).
   It fails on a warning as well as on an error: a typo'd table name is the likely bug
   in a hand-written config, and it is the one that fails silently — the phase just
   does nothing. pre-commit and ansible-lint are deleted. **Nothing replaces
   `detect-private-key`**: the key lives at `~/.ssh/id_ed25519`, outside the repo,
   and no task copies it into the tree.
3. **Idempotence runs four per-part status commands** (Q3a as retargeted, Q11a). The
   aggregate command cannot work — see the facts above.
4. **CI asserts the imperative block** with three `test`/`jq` lines (Q4b). Those five
   effects are invisible to every status command, and the risk is a guard that
   silently skips its own work, which a plain non-zero exit cannot catch.
5. **The weekly cron survives** (Q5a). Drift risk went up, not down: the AeroSpace tap
   is untrusted and ships no API metadata, and all 20 packages are pinned `"latest"`.
6. **Verbosity rides on the action's `log_level` input**, not a job-level env var
   (Q6c, adjusted). GitHub does not expose the `runner` context in
   `jobs.<job_id>.env`, only at step level, so `MISE_VERBOSE: ${{ runner.debug }}` as
   a job env var does not resolve. The action's own input takes the same expression.
7. **`cache: false` on both jobs.** The user asked for it against the scheduled run;
   it is unconditional here because the repo `[tools]` table dies with ticket 02, so
   the cache holds little more than the mise binary. One line beats an expression for
   what it saves.
8. **`.github/dependabot.yml`: remove the `pip` entry** (Q7a). **The ticket text was
   wrong** — there is no ansible-lint entry in that file. `ansible/ansible-lint@v26`
   is a workflow action, covered by the `github-actions` entry, which survives and
   keeps `mise-action` current. The `pip` entry is the dead one; it exists only for
   `requirements.txt`.
9. **The action's `version` input stays unset**, so CI takes the latest mise. The
   version floor from ticket 07 is a README fact for a fresh Mac, not a CI pin, and
   the weekly cron exists precisely to find upstream breakage early.

### Two things ticket 07 must check on the first run

- **The `jq -e` line depends on ticket 05.** Claude Code does not exist on the
  runner, so `~/.claude/settings.json` is absent. Ticket 05's merge one-liner has to
  create the file when it is missing, or this assertion fails on a green bootstrap.
- **The lint job parses macOS tables on Linux.** `[bootstrap.macos.defaults]` is
  assumed to parse without an `unknown field` warning on `ubuntu-latest`. If it
  warns, the fix is to move the parse gate into the macOS job (Q8c), not to redesign.
