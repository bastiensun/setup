# Translate the macOS defaults and the login shell

Type: grilling
Status: closed
Assignee: sun
Blocked by: 01

## Question

Produce `[bootstrap.macos.defaults]` for the 8 `osx_defaults` tasks in
`playbook.yml`: trackpad `Clicking`, dock `autohide` and `show-recents`,
`_HIHideMenuBar`, `AppleShowAllExtensions`, finder `ShowPathbar`, and
controlcenter `BatteryShowPercentage` (which uses `host: currentHost`).

Two things need deciding, not just transcribing:

1. **The currentHost one.** Ticket 01 confirmed `defaults -currentHost` is
   **explicitly unsupported**, and mise never `killall`s the affected app. Choose:
   drop `BatteryShowPercentage`, write it to the plain user domain and see whether
   it takes effect, or keep a `defaults -currentHost write` line in
   `[tasks.bootstrap]`.
2. **The login shell.** `[bootstrap.user] login_shell = "/opt/homebrew/bin/fish"`
   exists and takes an absolute path. Appending to `/etc/shells` uses mise's sudo
   path, which errors rather than hangs when non-interactive. But `chsh -s` has no
   sudo wrapper and no prompt automation, and whether it succeeds unattended on a
   `macos-latest` runner is **unconfirmed**. Decide the CI stance (`--skip user` is
   the researched suggestion) and the local stance. If it prompts, that changes the
   README instructions in ticket 07.

The shell-activation interaction is now settled by ticket 01 and needs no decision:
**explicit dotfiles win** — when `[dotfiles]` owns `config.fish` as a whole file,
mise skips the generated activation entry. So `config.fish` carries the
`mise activate fish | source` line itself, alongside `brew shellenv` and
`starship init`. Note the finding for ticket 03 rather than re-deciding it here.

## Resolution

Seven of the eight defaults transcribe declaratively. The eighth becomes a hook. The
login shell uses the native `[bootstrap.user]` table, and CI skips that phase.

```toml
[bootstrap.macos.defaults]
"com.apple.AppleMultitouchTrackpad" = { Clicking = 1 }
"com.apple.dock" = { autohide = true, show-recents = false }
"NSGlobalDomain" = { _HIHideMenuBar = true, AppleShowAllExtensions = true }
"com.apple.finder" = { ShowPathbar = true }

[bootstrap.hooks.post-defaults]
run = "defaults -currentHost write com.apple.controlcenter BatteryShowPercentage -bool true"

[bootstrap.user]
login_shell = "/opt/homebrew/bin/fish"
```

**1. `BatteryShowPercentage` (Q1, Q5).** It cannot be declarative. `-currentHost` is not
a modifier on the write — it selects a different file. Per-host preferences live in
`~/Library/Preferences/ByHost/<domain>.<hardware-UUID>.plist`; the plain domain lives in
`~/Library/Preferences/<domain>.plist`. Control Center reads the ByHost file. Checked on
this machine: `defaults -currentHost read com.apple.controlcenter BatteryShowPercentage`
returns `1`, and the plain-domain read reports the pair does not exist. So writing the
key to the plain user domain creates something nothing reads — a line that looks like it
works and does not. mise states the limit directly: "Host-scoped preferences
(`defaults -currentHost`) and `sudo defaults` system domains are not supported."

It goes in `[bootstrap.hooks.post-defaults]`, **not** `[tasks.bootstrap]`. This
**amends the map's standing preference** that every non-declarative one-liner lands in
`[tasks.bootstrap]`: that preference predates the discovery of this hook. The hook runs
in the defaults phase, next to the settings it completes, and does not re-run when
`[tasks.bootstrap]` runs for other reasons. The preference still holds for brew, the
AeroSpace cask, and the SSH key, which have no phase hook of their own.

**2. No `killall` (Q2).** mise deliberately never restarts the affected apps, and Ansible
did not either. Dock, Finder, and the menu bar settings take effect at next login. This
is not a regression, and bootstrap runs on a fresh Mac that gets rebooted anyway. The
`post-defaults` hook is now the obvious home if that ever changes.

**3. Login shell (Q3, Q4).** Locally, `[bootstrap.user] login_shell` — the table exists
for exactly this, and a sudo prompt on first bootstrap is acceptable. In CI, run
`mise bootstrap --skip user`: whether `chsh` succeeds unattended on `macos-latest` is
unconfirmed, `chsh` runs raw with no sudo wrapper and no prompt automation, and a hang
costs a job timeout rather than a fast failure. Accepted cost: CI does not cover the
step most likely to break. **This binds "Rewrite CI for bootstrap"** — the bootstrap
invocation there carries `--skip user`.

**4. Shell activation** was already settled by "How does mise bootstrap behave on macOS?"
and is not re-decided here: explicit dotfiles win, so `config.fish` carries
`mise activate fish | source` itself. Ticket 03 owns it.
