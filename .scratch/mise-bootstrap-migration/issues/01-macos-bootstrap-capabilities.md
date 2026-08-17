# How does mise bootstrap behave on macOS?

Type: research
Status: resolved
Blocked by: —

## Question

The docs page describes the bootstrap tables but not their macOS specifics. Every
other ticket waits on these facts. Answer each against primary sources — the mise
docs site, the mise source, and its changelog. Say plainly when a fact cannot be
confirmed rather than guessing.

1. **Stability.** Is `mise bootstrap` marked experimental? Does it need
   `MISE_EXPERIMENTAL`? Which mise version introduced it, and how has the config
   schema changed since?
2. **Homebrew packages.** Does `[bootstrap.packages]` with the `brew:` prefix handle
   **casks** and **taps** (this repo needs `nikitabobko/tap/aerospace` with
   `trusted: true`)? Can it consume a `Brewfile` directly, or must every entry be
   written out as a table key?
3. **macOS defaults.** Does `[bootstrap.macos.defaults]` support explicit value
   types (`int`, `bool`) and the `-currentHost` scope? This repo needs
   `com.apple.controlcenter BatteryShowPercentage` written to the current host.
4. **Privilege.** How does bootstrap elevate for the things Ansible does with
   `become: true` — specifically changing the login shell to
   `/opt/homebrew/bin/fish`? Does `[bootstrap.user] login_shell` handle it, and does
   it prompt for sudo, need a flag, or fail unattended in CI?
5. **Dotfiles.** What are the exact semantics of `mode = "symlink"` vs `"copy"`?
   What counts as a conflict, and what does `--force-dotfiles` do to an existing
   file? Are source paths relative to the config file?
6. **Dry run.** Is `mise bootstrap --dry-run` output machine-readable — a JSON flag,
   a stable exit code, or a parseable "nothing to do"? This decides the CI
   idempotence check (settled as Q7 = option b, conditional on this answer).
7. **Shell activation.** Does `[bootstrap.mise_shell_activate]` support **fish**, and
   which keys? How does it interact with a `config.fish` that bootstrap also manages
   as a whole-file dotfile?
8. **Repos.** Does `[bootstrap.repos]` support tracking a moving `HEAD` (the
   simple-bar clone uses `version: HEAD` today), or does it require a pinned ref?
9. **Hooks.** What are the named hook phases, in order? Which one runs before
   packages, so it can install Homebrew itself?
10. **`[tasks.bootstrap]`.** When does it run relative to the declarative phases? Is
    its failure fatal to the whole bootstrap?

Capture findings as a Markdown file in the repo on a throwaway `research/` branch and
link it from the answer.

## Answer

Full findings, with quoted config syntax and source citations:
[`research/mise-bootstrap-macos.md`](../research/mise-bootstrap-macos.md).
Snapshot: `jdx/mise` main @ `6bd4a54` (2026-08-16), latest release 2026.8.6.

1. **Stability.** Not experimental. The `ensure_experimental` gate was removed in
   [#10869](https://github.com/jdx/mise/pull/10869), released **2026.7.4**; no
   `MISE_EXPERIMENTAL` needed. Introduced **2026.6.6** as `[system.*]`, renamed to
   `[bootstrap.*]` since (no `system` alias survives in current source; the exact
   rename release is unconfirmed). CLI reshaped in 2026.6.14. Still gaining features
   every release — require a recent mise (`os` filters on packages need ≥ 2026.8.4).
2. **Homebrew.** Casks use a *separate* prefix `"brew-cask:name" = "latest"`, not
   `brew:`. Taps work via fully-qualified names plus optional
   `[bootstrap.brew.taps]."owner/tap" = "https://github.com/owner/homebrew-tap.git"`.
   There is **no `trusted` key** — mise verifies by sha256 from tap API metadata.
   ⚠️ **BLOCKER: `nikitabobko/tap/aerospace` cannot work.** mise fetches
   `{tap}/api/cask/<token>.json` and that tap publishes no `api/` directory (404).
   No Brewfile ingestion of any kind; every entry is a table key.
   `packages import --manager brew` can seed formulae from the live machine, but
   "Cask import is not implemented."
3. **macOS defaults.** Types are inferred from TOML (bool→`-bool`, int→`-int`,
   float→`-float`, string→`-string`); there is no explicit `int`/`bool` annotation
   syntax, and comparison is strictly typed. Arrays/dicts/dates/data unsupported.
   ⚠️ **BLOCKER: `-currentHost` is explicitly not supported** ("Host-scoped
   preferences (`defaults -currentHost`) and `sudo defaults` system domains are not
   supported"; confirmed absent from source). `BatteryShowPercentage` must either be
   written to the plain user domain (effect unconfirmed) or moved to a
   `post-defaults` hook / `[tasks.bootstrap]` one-liner. mise never `killall`s apps.
4. **Privilege.** `[bootstrap.user] login_shell = "/opt/homebrew/bin/fish"` exists and
   absolute paths are required. Appending to `/etc/shells` goes through mise's sudo
   path (root → no sudo; interactive → prompt; non-interactive without passwordless
   sudo → **errors with the command to run, never hangs**). But `chsh -s` itself runs
   `raw(true)` with **no sudo wrapper and no prompt automation**. ⚠️ Whether macOS
   `chsh` succeeds unattended in CI is **unconfirmed** — plan for `--skip user` in CI
   and an interactive/follow-up step locally. There is no general `become: true`.
5. **Dotfiles.** `symlink` (default) links target→source, one link for a whole
   directory; `copy` writes a real file, is additive for directories and **never
   pruned**. Also `symlink-each`, `template`, inline `content`, glob sources,
   `exclude`, and `blockinfile`-equivalent edit entries keyed `"<path>/<id>"` with
   `block`/`line`. Relative `source` resolves **against the config file's directory**.
   A conflict = an unmanaged real file/dir where a symlink should go — and a real file
   *always* conflicts on a symlink apply even when content matches.
   `--force-dotfiles` replaces conflicting **whole-file** targets only; edit-entry
   refusals (corrupt markers, symlinked targets) are never forced.
6. **Dry run.** ⚠️ **`mise bootstrap --dry-run` is not machine-readable** — no `--json`
   flag on the top-level command, no sentinel, no exit-code contract (confirmed in
   `src/cli/bootstrap.rs`). It also skips template rendering by design. **Use
   `mise bootstrap status --missing` instead** (exit 1, covers the whole declarative
   surface, `--json` available). `mise bootstrap plan --json --detailed-exitcode`
   (0/2/1) is properly structured but currently covers only accounts, packages, files,
   services, firewall, compose — *not* dotfiles, defaults, repos, or login shell.
   Q7 option (b) stands, retargeted at `status --missing`.
7. **Shell activation.** fish is supported: key `fish`, default mode `activate`,
   writes `~/.config/fish/config.fish` with `mise activate fish | source`. Keys are
   `bash_profile`, `bashrc`, `zprofile`, `zshrc`, `zshenv`, `fish`; values
   `"activate"` / `"shims"` / bool / `{enabled, mode}`. Interaction is clean —
   "**Explicit dotfiles win**": if `[dotfiles]` owns the same rc file as a whole file,
   mise skips the generated entry. So own `config.fish` as a dotfile and put the
   activation line in it.
8. **Repos.** The key is **`ref`**, not `version`; there is no `version = "HEAD"`.
   `ref` takes a branch, tag, or full SHA. Omitting `ref` means "exists with the right
   origin = current" and **apply never pulls it** (only `mise bootstrap repos update`
   does). For simple-bar: omit `ref`, or pin `ref = "main"`. Literal `ref = "HEAD"` is
   unconfirmed. No force-resets; dirty/mismatched targets fail.
9. **Hooks.** Phases: `pre-packages`, `post-packages`, `pre-repos`, `post-repos`,
   `pre-dotfiles`, `post-dotfiles`, `pre-defaults`, `post-defaults`, `pre-user`,
   `post-user`, `pre-tools`, `post-tools`, plus `final`. **`pre-packages` is the
   earliest** and is where a Homebrew installer would go — but note mise's `brew` and
   `brew-cask` managers work **without Homebrew installed** and create `/opt/homebrew`
   themselves, so that hook may be unnecessary. Hooks abort the bootstrap on failure
   and only print under `--dry-run`.
10. **`[tasks.bootstrap]`.** Runs at step 17 of 18 — after every declarative phase and
    after `mise install`, before `[bootstrap.hooks.final]` only. **Failure is fatal**
    (the call site propagates with `?`; the final hook and follow-up summary are
    skipped). It runs every time and is never converged, so each one-liner needs its
    own guard. Not executed under `--dry-run`.
