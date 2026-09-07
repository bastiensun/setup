# Write the [tasks.bootstrap] block

Type: grilling
Status: closed
Assignee: sun
Blocked by: 01

## Question

Four Ansible tasks have no native bootstrap table. Settled: all four become guarded
one-liners in `[tasks.bootstrap]`, not a shell script. Write them, and make each
idempotent — `[tasks.bootstrap]` runs on every bootstrap.

1. **SSH key.** `ssh-keygen -t ed25519` at `~/.ssh/id_ed25519`, only when absent.
   Decide the passphrase behaviour (empty, or prompt) and what happens in CI.
2. **Git config.** 6 global keys: `user.name`, `user.email`, `core.editor`,
   `gpg.format`, `user.signingkey`, `commit.gpgsign`. Decide whether these stay as
   `git config --global` calls or become a symlinked `~/.gitconfig` dotfile — the
   round-1 discussion flagged the dotfile option as genuinely tempting here.
3. **Claude settings merge.** Ansible reads `~/.claude/settings.json`, merges
   `{"outputStyle": "ASD-STE100"}`, and writes it back. Claude Code owns this file,
   so the merge must survive. Decide the tool — `jq`, and where it comes from.
4. **apm.** `mise exec -- apm install --global`, after `~/.apm/apm.yml` is in place.
   Confirm the ordering holds given when `[tasks.bootstrap]` runs.

Also decide the Homebrew-install hook: which phase, and the guard that stops it
re-running on a machine that already has brew.

## Added by ticket 02

`[tasks.bootstrap]` now opens with two more guarded one-liners, **before** everything
else, so the Homebrew installer meets a clean `/opt/homebrew` prefix:

1. `command -v brew || NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL <install.sh>)"`
2. `test -d /Applications/AeroSpace.app || brew install --cask nikitabobko/tap/aerospace`

Known risk to check on a fresh machine: the tap is untrusted and ships no API
metadata, so the install may prompt. Fallback is a one-entry `Brewfile` plus
`brew bundle`, the only form that carries `trusted: true`.

## Resolution

Two places, not one. Homebrew moves to a `pre-packages` hook, because that is the
only phase that runs before mise touches `/opt/homebrew`. Git config leaves the
imperative block entirely and becomes a 10th `[dotfiles]` entry. Three items remain
in `[tasks.bootstrap]`.

### The config

```toml
[bootstrap.hooks.pre-packages]
run = [
  'command -v brew || NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"',
  'test -d /Applications/AeroSpace.app || brew install --cask nikitabobko/tap/aerospace',
]

[tasks.bootstrap]
run = [
  'test -f ~/.ssh/id_ed25519 || ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519',
  'mkdir -p ~/.claude',
  'test -f ~/.claude/settings.json || echo "{}" > ~/.claude/settings.json',
  '''mise x jq@latest -- jq '.outputStyle = "ASD-STE100"' ~/.claude/settings.json > ~/.claude/settings.json.new''',
  'mv ~/.claude/settings.json.new ~/.claude/settings.json',
  'mise exec -- apm install --global',
]
```

Plus one new `[dotfiles]` entry, which amends the 9-entry table in ticket 03:

```toml
"~/.gitconfig" = { source = "dotfiles/gitconfig" }
```

and one new file, `dotfiles/gitconfig`, copied verbatim from the live `~/.gitconfig`:

```ini
[user]
	name = Bastien Sun
	email = 43788700+bastiensun@users.noreply.github.com
	signingkey = ~/.ssh/id_ed25519.pub
[core]
	editor = zed --wait
[gpg]
	format = ssh
[commit]
	gpgsign = true
```

### The decisions

1. **SSH key: empty passphrase** (Q1a). `-N ""` matches today —
   `community.crypto.openssh_keypair` writes no passphrase either. The line runs
   unattended, so ticket 06's `test -f ~/.ssh/id_ed25519` assert passes on the runner.
   One small drift: `ssh-keygen` writes a `user@host` comment and Ansible wrote an
   empty one. Nothing reads that comment.
2. **Git config becomes a symlinked dotfile** (Q2a). This deletes 6 imperative lines
   and buys free CI coverage: `mise bootstrap dotfiles status --missing` already runs
   in ticket 06's idempotence step. **Checked**: the live `~/.gitconfig` holds exactly
   these 6 keys and nothing else, so the symlink drops nothing. Accepted risk: any
   tool that writes `~/.gitconfig` — `gh auth setup-git`, a credential helper, a plain
   `git config --global` — now makes the repo worktree dirty. `zed.json` carries the
   same trade and the same answer.
3. **`jq` comes from `mise x jq@latest`** (Q3b), not from a `[bootstrap.packages]`
   entry. **Checked**: `mise registry jq` resolves to `aqua:jqlang/jq`, so the tool
   exists. The package list stays at the 20 entries ticket 02 settled.
4. **The merge is 4 lines, not 1.** `mkdir -p` and the `{}` seed together satisfy
   ticket 06's hard constraint: the merge must *create* `~/.claude/settings.json`,
   because Claude Code does not exist on a CI runner. `jq` then writes through
   `.json.new` and `mv` puts it in place. Splitting `jq` and `mv` into two array
   entries beats an `&&`: a failing entry stops the task on its own.
5. **apm keeps today's command** (Q4b): `mise exec -- apm install --global`. It rests
   on an assumption — that step 15 (`mise install`) picks up the global config the
   dotfiles phase symlinked at step 10 of the same process. **The assumption is
   self-reporting**: if it is false, `apm` is not on PATH, `mise exec` exits non-zero,
   and a failing `[tasks.bootstrap]` aborts the whole bootstrap. CI symlinks that same
   global config, so the runner exercises it every run. No extra CI assert is needed.
6. **Homebrew goes in `[bootstrap.hooks.pre-packages]`** (Q5a), both lines together.
   `pre-packages` is the earliest hook in the 18-step order, and it is the only one
   that runs before mise creates and fills `/opt/homebrew`. The AeroSpace cask sits in
   the same hook, right after the installer that it needs.
7. **Two places total** (Q6a): the hook, and the task. Ticket 02's text put the
   Homebrew lines in `[tasks.bootstrap]`; that is superseded — step 17 is far too late
   for ticket 02's own reason to hold.

### Accepted costs

- **The Homebrew installer asks for a sudo password once**, on a fresh Mac.
  `NONINTERACTIVE=1` removes the confirmation prompt but not the sudo prompt. CI never
  reaches the line, because `macos-latest` already ships Homebrew.
- **CI's `jq -e` assert uses the runner's own `jq`**, not the mise one. The
  `macos-latest` image ships `jq`. If a future image drops it, that assert breaks
  while the bootstrap stays green.

### For ticket 07

1. **The first real run needs `--force-dotfiles`.** `~/.gitconfig` exists on this Mac
   as a real file, and mise refuses to replace an unmanaged real file with a symlink.
   Back it up before the first run.
2. **Add the `[dotfiles]` entry and `dotfiles/gitconfig`** from this resolution to
   ticket 03's table. The table is 10 entries, not 9.
