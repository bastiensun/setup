# What goes in the [dotfiles] table, and symlink or copy?

Type: grilling
Status: resolved
Blocked by: 01

## Question

Ansible copies 7 config files and appends blocks to 2 more. Produce the full
`[dotfiles]` table.

**Straight copies today**: `dotfiles/mise-global.toml`, `apm-global.yml`,
`ponytail.json`, `claude-output-styles/asd-ste100.md`, `aerospace.toml`,
`simple-bar.json`, `zed.json`.

**Appends today** (settled: these become whole files in `dotfiles/`) —
`~/.config/fish/config.fish` and `~/.config/ghostty/config`. Write their full
contents as part of this ticket. `config.fish` must carry `mise activate fish |
source` itself: ticket 01 found that an explicit dotfile wins over the generated
`[bootstrap.mise_shell_activate]` entry, so owning the file means owning that line.

**Reopened by ticket 01**: the premise that bootstrap manages whole files only was
wrong. It has a `blockinfile` equivalent — edit entries keyed `"<path>/<id>"` with
`block`/`line`, marker-delimited like Ansible's. That makes the round-1 answer worth
one look before committing to whole files, at least for `config.fish`, which several
tools write to. Note also that `--force-dotfiles` never forces an edit entry, and a
real file always conflicts with a symlink even when the content matches.

Decide per file: `symlink` or `copy`. The standing preference is symlink, so edits
flow back to git — but name the exceptions. `~/.claude/settings.json` is explicitly
**not** a dotfile: Claude Code rewrites it, so it stays a merge in
`[tasks.bootstrap]` (ticket 05).

Also settle whether the directory-creation tasks (`~/.config/fish`, `~/.homebrew`,
`~/.config/mise`, `~/.apm`, and the rest) disappear because the dotfiles table
creates parents, or whether some still need `[bootstrap.directories]`.

## Resolution

Whole files everywhere, symlink everywhere except `.simplebarrc`, no
`[bootstrap.directories]` table. No edit entries at all.

### The table

```toml
[dotfiles]
"~/.config/fish/config.fish" = { source = "dotfiles/fish-config.fish" }
"~/.config/ghostty/config" = { source = "dotfiles/ghostty.config" }
"~/.config/mise/config.toml" = { source = "dotfiles/mise-global.toml" }
"~/.apm/apm.yml" = { source = "dotfiles/apm-global.yml" }
"~/.config/ponytail/config.json" = { source = "dotfiles/ponytail.json" }
"~/.claude/output-styles/asd-ste100.md" = { source = "dotfiles/claude-output-styles/asd-ste100.md" }
"~/.config/aerospace/aerospace.toml" = { source = "dotfiles/aerospace.toml" }
"~/.config/zed/settings.json" = { source = "dotfiles/zed.json" }
"~/.simplebarrc" = { source = "dotfiles/simple-bar.json", mode = "copy" }
```

`symlink` is the default mode, so only the one exception names a mode.

### Why

- **`config.fish` and the Ghostty config become whole files** (Q1a, Q2a), not edit
  entries. The user is the only writer of both. Whole files keep one convention
  across the table, stay forceable with `--force-dotfiles` (edit entries are never
  forceable), and avoid the two things mise always refuses: corrupted markers and an
  edit target that is itself a symlink. The Ansible markers in both live files
  become plain content.
- **`.simplebarrc` is the one `copy`.** simple-bar writes it from its own settings
  panel, so a symlink turns every widget tweak into a dirty worktree. `zed.json`
  stays a symlink: Zed also writes from its UI, and the user wants those edits to
  flow back to git.
- **`~/.claude/settings.json` stays out of the table** — Claude Code rewrites it. It
  remains a merge one-liner in `[tasks.bootstrap]` (ticket 05).
- **All 9 `file: state=directory` tasks are deleted** (Q4a). Every one exists only as
  the parent of a file the next task writes, and the dotfiles phase writes those
  files. `~/.homebrew` has no file left at all — ticket 02 deleted the Brewfile.

### File contents

`dotfiles/fish-config.fish` — the mise line is here because ticket 01 found that an
explicit dotfile beats the generated `[bootstrap.mise_shell_activate]` entry, so
owning the file means owning the line:

```fish
eval "$(/opt/homebrew/bin/brew shellenv)"
starship init fish | source
mise activate fish | source
```

`dotfiles/ghostty.config`:

```
theme = dark:Catppuccin Mocha,light:Catppuccin Latte
```

### One unverified fact for ticket 07

The ticket-01 findings do not state that the dotfiles phase creates missing parent
directories. This resolution assumes it does. Ticket 07 checks it on the first real
run. If it is false, the fix is a small `[bootstrap.directories]` list — not a
redesign.
