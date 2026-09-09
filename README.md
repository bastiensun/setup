# ⚙️ Setup

A mise bootstrap config that sets up my Mac.

[![Automation (xkcd)](https://imgs.xkcd.com/comics/automation.png)](https://xkcd.com/1319/)

## Installation

Install mise, then adopt this repository as your global mise configuration. `mise
bootstrap` needs mise 2026.7.4 or later — earlier versions gate it behind an
experimental flag.

```shell
curl -fsSL https://mise.run | sh
mise bootstrap --adopt bastiensun/setup
```

On a machine that already has a `~/.config/mise`, clear it first — adopt only accepts
an existing destination that is already a checkout of this repository. On a machine
with a real `~/.gitconfig`, add `--force-dotfiles`, since mise refuses to replace an
unmanaged file with a symlink.

The clone lands at `~/.config/mise` and is the working copy: edit, commit and push from
there.
