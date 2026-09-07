# How `mise bootstrap` behaves on macOS

Research for ticket `.scratch/mise-bootstrap-migration/issues/01-macos-bootstrap-capabilities.md`.

**Sources** (all primary): the mise docs site (`https://mise.jdx.dev/bootstrap.html` and
sub-pages, read from their source of truth `jdx/mise:docs/*.md`), the mise Rust source
on `main`, `jdx/mise:CHANGELOG.md`, and GitHub PR descriptions.

**Snapshot**: `jdx/mise` `main` at `6bd4a54` (2026-08-16), latest release in CHANGELOG
`2026.8.6` (2026-08-14).

Anything I could not confirm from a primary source is marked **UNCONFIRMED** and left
unanswered rather than guessed.

---

## 1. Stability, version, schema churn

**Not experimental any more.** `mise bootstrap` and all its subcommands were gated
behind `ensure_experimental` when introduced, and the gate was removed in PR
[#10869](https://github.com/jdx/mise/pull/10869) *"feat(bootstrap): stabilize bootstrap
commands"*, released in **2026.7.4** (2026-07-09). Its description:

> remove experimental guards from mise bootstrap, bootstrap package subcommands, and
> dotfiles commands […] add e2e coverage for bootstrap/dotfiles/packages commands with
> `MISE_EXPERIMENTAL=0`

No `MISE_EXPERIMENTAL` is needed. Grepping `src/cli/bootstrap.rs` on `main` finds no
`experimental` reference at all. The docs page carries no experimental banner.

**Introduced**: `mise bootstrap` landed in **2026.6.6** (2026-06-13) via
[#10365](https://github.com/jdx/mise/pull/10365) *"add `mise bootstrap` and declarative
`[system.files]`"*. The declarative package layer that it builds on
([#10326](https://github.com/jdx/mise/pull/10326), "declarative system packages (apt,
dnf, pacman, and brew without brew)") landed slightly earlier in **2026.6.4**.

**Schema churn since**: substantial. Two shifts matter for us:

- *Config sections renamed `[system.*]` → `[bootstrap.*]`.* The 2026.6.6 changelog
  entries name `[system.defaults]`, `[system.files]`, `[system.edits]`; current docs and
  the current serde model (`src/config/config_file/mise_toml.rs:411`,
  `bootstrap: Option<BootstrapTomlConfig>`) only know `[bootstrap.*]`. I found **no
  `system` serde alias in the current source**, so old `[system.*]` config is no longer
  accepted. The exact release that performed the rename is **UNCONFIRMED** — the
  CHANGELOG has no line naming it.
- *CLI reshaped to mirror config sections* in
  [#10613](https://github.com/jdx/mise/pull/10613) (2026.6.14): `packages apply` replaced
  `install`, and `shell`/`macos-defaults`/`launchd`/`systemd` became
  `mise-shell-activate` / `macos defaults` / `macos launchd-agents` /
  `linux systemd-units`. Old short names survive as aliases for `--skip`/`--only`.

Feature additions have continued every release right up to 2026.8.x (privileged files,
secrets, Linux users/groups, compose, firewall, remote SSH bootstrap, resource plans,
per-package platform filters). The surface is stable but still moving fast. Practical
consequence for this repo: **pin an expectation of a recent mise** — `[bootstrap.packages]`
table-form `os` filters only arrived in 2026.8.4
([#11809](https://github.com/jdx/mise/pull/11809)).

---

## 2. Homebrew packages: casks, taps, Brewfile

### Casks are a *separate manager prefix*, not `brew:`

`docs/bootstrap/packages/brew.md`:

> Casks use the `brew-cask:` manager.

```toml
[bootstrap.packages]
"brew:postgresql@17" = "latest"
"brew-cask:firefox" = "latest"
"brew-cask:homebrew/cask/visual-studio-code" = "latest"
```

Critically, **mise does not shell out to Homebrew at all** — it re-implements pouring
bottles and cask installs itself against `/opt/homebrew`. "mise never shells out to
`brew` for homebrew/core formulae." Intel Macs are unsupported (arm64 macOS only).

### Taps

Fully-qualified tap names work directly, *if the tap publishes Homebrew API metadata*:

> Third-party taps are supported directly when the tap publishes Homebrew API metadata
> (`api/formula/<name>.json` or `api/cask/<token>.json`). Use the same fully-qualified
> name you would pass to Homebrew

```toml
[bootstrap.packages]
"brew:railwaycat/emacsmacport/emacs-mac" = "latest"
"brew-cask:owner/tap/app" = "latest"
```

And for taps whose GitHub URL cannot be inferred:

```toml
[bootstrap.brew.taps]
"acme/tools" = "https://github.com/acme/homebrew-tools.git"

[bootstrap.packages]
"brew:acme/tools/widget" = "latest"
"brew-cask:acme/tools/widget-app" = "latest"
```

> Non-GitHub taps are not currently supported because mise needs direct raw access to
> the generated API metadata.

There is **no `trusted` key** anywhere in the bootstrap schema. Ansible's
`homebrew_tap`/cask `trusted` concept has no counterpart; mise verifies by sha256 from
the API metadata instead.

### ⚠️ `nikitabobko/tap/aerospace` will NOT work

This is a concrete blocker, confirmed two ways:

1. `src/system/packages/brew/cask.rs:833-870` builds the metadata URL for a tapped cask
   as `{tap_raw_base}/api/cask/{token}.json` and errors with
   `"failed to fetch Homebrew cask '{name}' directly. Tapped casks must publish API
   metadata at api/cask/<token>.json"` when that fetch fails.
2. `https://github.com/nikitabobko/homebrew-tap` contains only `Casks/`, `Formula/`,
   `.github/`, `pin.sh`, `run-tests.sh` — **no `api/` directory**. Both
   `raw.githubusercontent.com/.../master/api/cask/aerospace.json` and
   `nikitabobko.github.io/homebrew-tap/api/cask/aerospace.json` return 404.

So `"brew-cask:nikitabobko/tap/aerospace" = "latest"` will fail at metadata fetch. There
is no config knob for this — a tap either publishes API JSON or it doesn't.
(Secondary, moot-if-blocked note: the aerospace cask also uses a Ruby `postflight` block
and a `manpage` artifact. mise supports `postflight` via its own Cask DSL shim, and
`manpage` artifacts are parsed and skipped — `cask.rs:7346`, `cask_shim.rb:288` — so
those two would not have been the problem.)

Workaround options (none confirmed by docs, all mechanical): install AeroSpace from its
GitHub release in `[tasks.bootstrap]`, or keep a real `brew install --cask` one-liner
there, or ask the tap to publish API metadata.

### Brewfile

**No.** There is no way to point `[bootstrap.packages]` at a `Brewfile`; the docs never
mention consuming one. Every package is a table key. The nearest facility is the inverse,
a one-time migration aid:

```sh
mise bootstrap packages import --manager brew   # writes "brew:<formula>" = "latest" entries
```

which reads the live Homebrew `opt` links (`--all` to include dependencies) and infers
`[bootstrap.brew.taps]` entries where it can. Explicit limitation in the same page:
**"Cask import is not implemented."** So formulae can be imported from the current
machine, casks must be hand-written.

Per-package OS filtering uses the table form:

```toml
"brew-cask:1password" = { os = "macos" }
```

---

## 3. macOS defaults: types and `-currentHost`

### Value types: inferred from TOML, no explicit `int`/`bool` annotation

`docs/bootstrap/macos-defaults.md`:

```toml
[bootstrap.macos.defaults]
"com.apple.finder" = { AppleShowAllFiles = true }
```

| TOML value | written as         | example                |
| ---------- | ------------------ | ---------------------- |
| boolean    | `-bool true/false` | `autohide = true`      |
| integer    | `-int <n>`         | `tilesize = 48`        |
| float      | `-float <n>`       | `scale = 1.5`          |
| string     | `-string <s>`      | `orientation = "left"` |

There is **no explicit type-name syntax** — you get `-int` by writing a TOML integer and
`-bool` by writing a TOML boolean. That is sufficient for what the ticket needs, and it
is strictly typed on the read side too:

> **Strictly typed** — an existing value only counts as in sync when both the value and
> the plist type match: an integer `1` does not satisfy a configured `true`.

Arrays, dicts, dates, and data are **not supported** — "entries using them parse fine but
are skipped with a warning."

### ⚠️ `-currentHost` is NOT supported

Verbatim from the same page:

> Host-scoped preferences (`defaults -currentHost`) and `sudo defaults` system domains
> are not supported.

Confirmed in source: grepping `src/` for `currentHost` / `current_host` returns only
unrelated hits in `src/backend/spm.rs`.

So `com.apple.controlcenter BatteryShowPercentage` **written to the current host cannot
be expressed declaratively**. Writing it to the plain user domain is expressible:

```toml
[bootstrap.macos.defaults]
"com.apple.controlcenter" = { BatteryShowPercentage = true }
```

but whether the non-`-currentHost` write actually takes effect for Control Center is
**UNCONFIRMED** — that is an Apple behaviour, not a mise one, and no primary source I
read speaks to it. If it doesn't, this becomes a
`defaults -currentHost write …` one-liner in `[tasks.bootstrap]` or a `post-defaults`
hook.

Other relevant semantics: defaults are **never removed** ("mise never deletes a
default"), are applied only by explicit `apply`/`bootstrap`, are inert on non-macOS, and
mise **deliberately does not `killall`** the affected apps — use `[bootstrap.hooks.post-defaults]`
(`run = "killall Dock || true"`). Missing key/domain is reported as `unset` rather than
erroring since [#11118](https://github.com/jdx/mise/pull/11118) (2026.7.11).

---

## 4. Privilege / login shell

`docs/bootstrap/user.html`:

```toml
[bootstrap.user]
login_shell = "/opt/homebrew/bin/fish"
```

(the absolute-path requirement is documented with this exact path as the example:
"relative shell names are skipped with a warning. Use the full path, such as `/bin/zsh`
or `/opt/homebrew/bin/fish`").

Two separate privileged steps:

1. **Appending to `/etc/shells`** — root-owned. This *does* go through mise's sudo path:
   > `/etc/shells` is usually root-owned. If the file is not writable, mise uses the same
   > non-interactive sudo behavior as system packages: it can prompt in an interactive
   > terminal, uses passwordless sudo in non-interactive contexts, and honors
   > `system_packages.sudo = false`.

   The sudo contract (`docs/bootstrap/packages/index.md`): already root → no sudo;
   interactive terminal → normal sudo prompt; **non-interactive without passwordless
   sudo → mise errors and prints the exact command to run manually — it never hangs.**
   `system_packages.sudo = false` / `MISE_SYSTEM_PACKAGES_SUDO=0` forbids elevation
   entirely.

2. **`chsh -s <shell>`** — this does **not** use sudo. `src/system/login_shell.rs:73-83`:

   ```rust
   crate::cmd::CmdLineRunner::new("chsh").args(args).raw(true).execute()
   ```

   `raw(true)` means inherited stdio, no sudo wrapper, no password automation. When mise
   itself runs under sudo it retargets `SUDO_USER` rather than root.

**⚠️ Unattended risk.** macOS `chsh` authenticates the calling user via
OpenDirectory and prompts for a password when changing your own shell; mise has no
mechanism to answer that prompt. Whether this succeeds or hangs on a GitHub
`macos-latest` runner is **UNCONFIRMED** — no mise primary source addresses macOS chsh
authentication, and I did not run it. Plan defensively: either accept an interactive
first-run prompt, or in CI use `--skip user` (see Q6) and treat login-shell change as a
documented manual/follow-up step. mise itself surfaces it as such:

> Top-level `mise bootstrap` also includes a final follow-up reminder to start a new
> login session when it changes or would change the login shell.

There is **no general `become: true` equivalent**. The only sudo surface in the whole
bootstrap is: Linux system package managers, `/etc/shells`, `[bootstrap.files]` /
`[bootstrap.directories]` (Linux-oriented privileged files), and one-time creation of
`/opt/homebrew`. Explicitly *not* sudo: macOS defaults ("User defaults are per-user, so
unlike system packages no sudo is ever involved") and dotfiles ("Dotfiles write as the
current user — there is no sudo here").

---

## 5. Dotfiles semantics

`docs/dotfiles.md`. Modes, verbatim from the table:

| Mode           | Behavior (abridged from the docs table)                                                                                                      |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `symlink`      | "Symlink the target to the source. Works for files and directories — a directory source gets one link for the whole directory. This is the default." |
| `symlink-each` | Source must be a directory; recreate the tree and symlink each file, so the target dir can also hold unmanaged files. Deleting a source file removes its link next apply. Managed links recorded under `$MISE_STATE_DIR/dotfiles`. |
| `copy`         | "Copy the source file (or directory, recursively). Use when the target must be a real file — e.g. tools that rewrite their config in place. Directory copies are additive […] Copies are never pruned, so removing a source file leaves the copy behind." |
| `template`     | Render through the mise template engine; permissions taken from the source file.                                                             |

Also available: inline `content = "..."` (written `0600`, cannot be combined with
`source`/`mode`/`exclude`), glob sources, `exclude` globs for directory-walking modes,
and **edit entries** keyed `"<path>/<id>"` with `block = '...'` or `line = "..."` —
which is the direct replacement for Ansible `blockinfile`:

```toml
[dotfiles]
"~/.zshrc/activate" = { block = 'eval "$(mise activate zsh)"' }
"/etc/hosts/dev" = { line = "127.0.0.1 dev.local" }
```

**Source paths**: yes, relative to the config file.

> Relative explicit sources resolve against the directory of the config file that
> declares the entry

If `source` is omitted, mise mirrors the home-relative target under `dotfiles.root`
(`[settings] dotfiles.root`, `dotfiles.default_mode`). Targets outside `$HOME` must give
an explicit `source` or `content`.

**Conflicts**:

> mise refuses to _replace_ existing files it doesn't manage: a real file or directory
> where a symlink should go, or a directory where a file should go, is an error listing
> the conflicting paths.

and, importantly for a fresh-Mac run:

> Real files and directories always require `--force` during a standalone symlink apply,
> even when their visible content and permissions match. Portable filesystem APIs cannot
> compare ownership, ACLs, extended attributes, flags, and security labels.

Not conflicts: content updates under `copy`/`template` (overwrite without force — "that
is the declared intent of those modes"), re-pointing symlinks, and edit entries. Two
things are *always* refused rather than guessed: corrupted markers, and edit targets that
are themselves symlinks.

**`--force-dotfiles`** on the top-level command is the pass-through for this:

> By default, bootstrap refuses dotfile conflicts rather than replacing local files. Use
> `mise bootstrap --force-dotfiles` when you explicitly want the dotfiles phase to
> replace conflicting whole-file dotfile targets.

Note "whole-file" — it is scoped ([#10410](https://github.com/jdx/mise/pull/10410),
"scope forced dotfile conflicts"); it does not loosen the edit-entry refusals.

Also: **removing an entry from config does not clean up** — run
`mise bootstrap dotfiles unapply` first.

---

## 6. Dry run / machine-readable output — the CI idempotence check

**`mise bootstrap --dry-run` is *not* machine-readable.** Confirmed against
`src/cli/bootstrap.rs:100-116`: the only top-level flags added are `--dry-run/-n`
("Print what would happen without installing anything") and `--yes/-y`. There is no
`--json` on the top-level command, no documented "nothing to do" sentinel, and no
stable exit code contract for it. Output is human `info!`/`miseprintln!` lines.

Three real machine-readable options exist instead:

1. **`mise bootstrap status --missing`** — best fit for the CI idempotence check.
   > `mise bootstrap status --missing` checks the whole declarative bootstrap surface in
   > one command.

   It "reports every declarative part — packages, repos, dotfiles, shell activation,
   macOS defaults, LaunchAgents, systemd units, and login shell — plus `[tools]`". Exit 1
   if anything is out of sync. `--json` is also available on the aggregate and on every
   per-part status.

2. **`mise bootstrap plan --json` / `--detailed-exitcode`** — properly structured, but
   **narrower coverage**:
   > The provisioning planner reports accounts, system packages, privileged files and
   > directories, system services, firewall policy and rules, and Compose projects in
   > dependency order. Other declarative bootstrap parts will join the same graph as they
   > adopt the resource model.

   i.e. **plan does not yet cover dotfiles, macOS defaults, repos, or the login shell** —
   most of what this repo does. Exit codes:
   > With `--detailed-exitcode`, the command exits 0 when nothing would change, 2 when
   > the plan contains changes, and 1 when planning fails or any resource has an
   > `unknown` state.

3. Per-part `status --json` / `--missing`, e.g.
   `mise bootstrap dotfiles status --missing`,
   `mise bootstrap macos defaults status --missing`. The shell-activate one documents its
   JSON shape: entries include `target`, `shell`, `path`, `mode`, `state`, where `state`
   is `"missing" | "applied" | "differs" | "source_missing"`.

**Recommendation for Q7-option-b**: use `mise bootstrap status --missing` (exit 1) as the
CI idempotence gate, not `--dry-run`. Caveat: `--dry-run` deliberately skips template
rendering — "`--dry-run` is the exception: it promises to execute nothing, so it skips
template rendering and lists those entries as `(if changed)`" — so if we use `template`
mode dotfiles, dry-run under-reports; `status` does render them.

---

## 7. Shell activation and fish

**fish is supported.** `docs/bootstrap/shell.md`:

```toml
[bootstrap.mise_shell_activate]
zprofile = "shims"
zshrc = "activate"
bash_profile = "shims"
bashrc = "activate"
fish = "activate"
```

Keys are: `bash_profile`, `bashrc`, `zprofile`, `zshrc`, `zshenv`, `fish`. The `fish`
target defaults to mode `activate`, writes `~/.config/fish/config.fish`, and emits
`mise activate fish | source`. Shortcut keys exist (`zsh = true` expands to
`zprofile = "shims"` + `zshrc = "activate"`); a table form is accepted for future
options: `zshrc = {enabled = true, mode = "activate"}`. Any target accepts `"activate"`
or `"shims"`; `false` disables.

**Interaction with a whole-file `config.fish` dotfile — clean and documented:**

> **Explicit dotfiles win** — if `[dotfiles]` already manages the same rc file as a whole
> file, or defines an edit for the same target/id such as `"~/.zshrc/activate"`, mise
> skips the generated shell activation entry for that shell.
>
> For fully managed rc files or custom activation blocks, use `[dotfiles]` directly
> instead.

So this repo can own `~/.config/fish/config.fish` as a symlinked dotfile and simply not
declare `fish` in `[bootstrap.mise_shell_activate]` (or declare it harmlessly — it will
be skipped). The activation line must then live inside the repo's own `config.fish`.

---

## 8. Repos and moving `HEAD`

`docs/bootstrap/repos.md`:

```toml
[bootstrap.repos]
"~/src/dotfiles" = { url = "git@github.com:jdx/dotfiles.git", ref = "main" }
"~/src/mise" = { url = "https://github.com/jdx/mise.git" }
```

The key is **`ref`, not `version`** — the ticket's `version: HEAD` has no equivalent key.

> Each key is the target path. The `url` is required. The optional `ref` can be a branch,
> tag, or full commit SHA.

There is **no documented literal `HEAD` value**; whether mise would accept `ref = "HEAD"`
is **UNCONFIRMED**. Two supported shapes cover the simple-bar use case instead:

- **Omit `ref`** — the "track whatever is there" mode, but *apply never updates it*:
  > **Omitted `ref`** — an existing repo with the expected origin is considered current;
  > mise does not fetch or update it.
  > Applying never pulls an existing repo without a configured `ref`; use
  > `mise bootstrap repos update` when you want that imperative behavior.

  `update` "fetches and fast-forward pulls the current branch of repos without a
  configured `ref`" and "warns and skips an unpinned repo with a detached HEAD".

- **Pin a branch** — `ref = "main"`. Then `apply` converges to it and status reports
  `differs` when it drifts.

No force-resets: dirty worktrees, non-empty non-git targets, and mismatched origins fail
rather than clobber. States: `current` / `missing` / `differs` / `dirty` / `conflict`.
Relative paths are allowed but only in a *project* config (resolved against its project
root, cannot escape it) — fine here, since the tables live in the repo `mise.toml`.

Ordering: "Repos run after `[bootstrap.packages]` and before `[dotfiles]`."

---

## 9. Hook phases, in order

Docs list the phases; `src/cli/bootstrap.rs:118-160` gives the same order in the command
doc comment. Full phase list from `docs/bootstrap.md`:

> `pre-packages`, `post-packages`, `pre-repos`, `post-repos`, `pre-dotfiles`,
> `post-dotfiles`, `pre-defaults`, `post-defaults`, `pre-user`, `post-user`,
> `pre-tools`, and `post-tools`.

plus `final` (step 18, after the `bootstrap` task).

Interleaved with the 18 steps, the effective order is:

1. secrets preflight
2. accounts (`[bootstrap.users]`, `[bootstrap.groups]`) — Linux
3. plugins (`[bootstrap.plugins]`)
4. **`pre-packages`**  ← earliest hook
5. packages (`[bootstrap.packages]`, built-in managers)
6. *(`post-packages` runs here only if no plugin packages are pending; otherwise it is
   deferred until after step 16 — `src/cli/bootstrap.rs:1573-1576`)*
7. files / directories
8. services (Linux) → firewall (Linux) → compose
9. **`pre-repos`** → repos → **`post-repos`**
10. **`pre-dotfiles`** → dotfiles → **`post-dotfiles`**
11. shell activation
12. **`pre-defaults`** → macOS defaults → **`post-defaults`**
13. macOS launchd agents → Linux systemd units
14. **`pre-user`** → `[bootstrap.user]` (login shell) → **`post-user`**
15. **`pre-tools`** → `mise install` (`[tools]`) → **`post-tools`**
16. plugin package managers (after their host tools exist), then deferred `post-packages`
17. `mise run bootstrap` (`[tasks.bootstrap]`)
18. **`final`**

**The phase that runs before packages is `pre-packages`.** The docs' own example is
exactly this shape:

```toml
[bootstrap.hooks.pre-packages]
run = "softwareupdate --install-rosetta --agree-to-license"
```

so a Homebrew-installer hook goes there. **Note for the map's "Homebrew is installed by a
bootstrap hook" assumption**: it may not be needed at all — mise's `brew` and `brew-cask`
managers work "**without requiring Homebrew to be installed**", creating `/opt/homebrew`
themselves (the one place the brew manager uses sudo, "mirroring what Homebrew's own
installer does (`mkdir` + `chown` to your user)"). Real Homebrew coexists: mise writes
brew-compatible `INSTALL_RECEIPT.json`, so `brew list`/`upgrade`/`uninstall` work on
mise-poured kegs and vice versa.

Hook mechanics:

> Hooks run only during explicit `mise bootstrap` invocations. A hook can be specified as
> a command string, an array of command strings, or a table with a `run` field. They use
> the same default inline shell setting as tasks, **stop the bootstrap if they fail**, and
> print the command instead of running it during `mise bootstrap --dry-run`. Hooks run in
> the current process environment; use `mise exec -- ...` inside a hook, or use
> `[tasks.bootstrap]`, when the command needs tools from `[tools]` on PATH.

Shorthand form:

```toml
[bootstrap.hooks]
post-defaults = "killall Dock || true"
```

Hooks merge across the config hierarchy (global → local).

---

## 10. `[tasks.bootstrap]`

**When**: second-to-last. Step 17 of 18 — after every declarative phase *and* after
`mise install` has put `[tools]` on PATH, before only `[bootstrap.hooks.final]`. The
docs' "what goes where" table: "`[tasks.bootstrap]` — Anything custom that should run
after tools are installed."

Skippable with `mise bootstrap --skip task`.

**Failure is fatal.** `src/cli/bootstrap.rs:1584-1586`:

```rust
info!("bootstrap: running `bootstrap` task");
self.run_task("bootstrap", skip.contains(&BootstrapPart::Tools))
    .await?;
```

The `?` propagates, so a failing task aborts the whole `mise bootstrap` with a non-zero
exit, and `[bootstrap.hooks.final]` plus the follow-up summary never run.

**Not converged.** "The declarative steps converge […] The `bootstrap` task runs every
time, so keep it idempotent." This is the direct constraint on the map's "guarded
one-liner" preference — every one-liner needs its own guard, e.g. the docs' own
`run = "gh auth status || gh auth login"`.

Under `--dry-run` the task is *not* executed: `run_task` passes `dry_run: self.dry_run`
through to the task runner and also forces `skip_tools`, with the source comment "a dry
run must not auto-install tools before the (not actually run) task".

---

## Summary of blockers and plan changes

| # | Finding | Effect |
| - | ------- | ------ |
| 2 | `nikitabobko/homebrew-tap` publishes no `api/cask/*.json`; mise requires it | **BLOCKER** — AeroSpace cannot come from `[bootstrap.packages]` |
| 3 | `defaults -currentHost` explicitly unsupported | **BLOCKER** for `BatteryShowPercentage` as a declarative entry |
| 4 | `chsh` runs raw with no sudo and no prompt automation | **RISK** — login shell change may need interaction; `--skip user` in CI |
| 6 | `--dry-run` has no `--json` and no exit-code contract | **CHANGES PLAN** — use `mise bootstrap status --missing` for CI idempotence |
| 2 | No Brewfile ingestion; cask import not implemented | Every package hand-written; `packages import --manager brew` helps for formulae only |
| 1 | Bootstrap is **stable** since 2026.7.4, no `MISE_EXPERIMENTAL` | No blocker; but require a recent mise (≥ 2026.8.4 for `os` filters) |
| 9 | mise's `brew`/`brew-cask` need no Homebrew | The planned "install Homebrew in a hook" step may be unnecessary |
| 8 | `[bootstrap.repos]` uses `ref`, has no `version`/`HEAD` | simple-bar: omit `ref` (never auto-updates) or pin a branch |
