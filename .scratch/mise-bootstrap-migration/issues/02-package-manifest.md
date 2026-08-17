# Brewfile or [bootstrap.packages]?

Type: grilling
Status: resolved
Blocked by: 01

## Question

`dotfiles/Brewfile` holds 6 formulae, 15 casks, and 1 trusted tap, installed today by
`brew bundle install --global`. Decide where that list lives after the migration.

The axis is: translate every entry into `[bootstrap.packages]` keys, or keep the
Brewfile as the manifest and invoke `brew bundle` from `[tasks.bootstrap]`.

Ticket 01 narrowed this sharply:

- There is **no Brewfile ingestion**. Translating means hand-writing 22 table keys.
- Casks use a separate prefix, `"brew-cask:name"`, not `"brew:"`.
- **AeroSpace cannot be installed by mise at all.** `nikitabobko/homebrew-tap`
  publishes no Homebrew API metadata, and mise fetches
  `{tap}/api/cask/<token>.json` to verify. There is no config knob around it. So
  the "translate everything" option is not actually available in pure form — at
  minimum AeroSpace needs a `[tasks.bootstrap]` line.

That makes this a real choice rather than a transcription: 22 hand-written keys plus
one shell exception, or keep the Brewfile whole and lose the declarative packages
table. Weigh it against the `ponytail` preference for fewest moving parts.

Also settle: does `mise` itself stay in the Brewfile once mise is the root
dependency, and where do `apm`, `gh`, and `node` live — `[tools]` in the repo config,
or the symlinked `dotfiles/mise-global.toml`?

## Resolution

**Delete `dotfiles/Brewfile`.** Its 20 remaining entries become `[bootstrap.packages]`
keys — 5 formulae under `brew:` (`act`, `fish`, `mole`, `starship`, `tldr`) and 15
casks under `brew-cask:`. All pinned `"latest"`, which matches today (the Brewfile
pins nothing).

`brew "mise"` is **dropped**. mise is the root dependency; it cannot install its own
installer. The README says install mise first.

No Homebrew CLI is needed for those 20 — mise never shells out to `brew`, it pours
bottles and installs casks itself against `/opt/homebrew` (research file, section 2).

**AeroSpace stays on Homebrew.** `[tasks.bootstrap]` gets two guarded one-liners, in
this order:

1. `command -v brew || NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL <install.sh>)"`
2. `test -d /Applications/AeroSpace.app || brew install --cask nikitabobko/tap/aerospace`

Homebrew installs **first**, before mise's packages, so the installer meets a clean
`/opt/homebrew` prefix rather than one mise has already populated. `NONINTERACTIVE=1`
keeps CI from hanging on the password prompt. The fully-qualified cask name taps
implicitly, so no separate `brew tap` line.

Rejected: `ubi:nikitabobko/AeroSpace` extracts binaries only, so it yields the
`aerospace` CLI and not `AeroSpace.app` — the window manager does not run. Also
rejected: dropping AeroSpace from bootstrap, which regresses the destination.

**Known risk.** `trusted: true` has no counterpart in the bootstrap schema and
`brew install --cask` may prompt on the untrusted tap on a fresh machine. Fallback:
a one-entry `Brewfile` plus `brew bundle`, the only form that carries the flag.

**Tools placement.** `apm`, `gh`, `node` stay in `dotfiles/mise-global.toml` — they
are machine-wide, and moving them into the repo would make every shell outside this
directory lose `gh`. The repo's `[tools]` table dies with `uv`, and `.python-version`
plus `[settings] idiomatic_version_file_enable_tools` go with it.

**Consequences for other tickets.** Homebrew is now a hard prerequisite of
`[tasks.bootstrap]`: ticket 06 must install it on the runner or check that
`macos-latest` ships it. Ticket 04 can rely on `/opt/homebrew/bin/fish` existing.
