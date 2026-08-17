# Execute the migration in one PR

Type: task
Status: closed — out of scope
Blocked by: 02, 03, 04, 05, 06

## Closed as out of scope

The map stops at the decisions. The build runs through `/to-spec` → `/to-tickets` →
implement, in its own effort directory at `.scratch/mise-bootstrap-build/`, numbered
from `01`. This ticket writes no diff.

Nothing here is wasted: the body below is the assembled brief for that spec. It names
every file to add, delete, and rewrite, points at the tickets that hold the config
blocks verbatim, and records the four calls under `## Settled before execution`.

Three assumptions still need a real run to confirm — they belong to the build, not to
this map:

1. The dotfiles phase creates missing parent directories (ticket 03).
2. The Claude-settings merge creates `~/.claude/settings.json` when it is absent
   (tickets 05 and 06).
3. `[bootstrap.macos.defaults]` parses on `ubuntu-latest` with no `unknown field`
   warning (ticket 06).

Plus one manual step on the first real run: back up `~/.gitconfig` and pass
`--force-dotfiles`.

## Question

Write the diff. One PR, no hybrid state — the two systems cannot half-coexist.

**Add**: the full `mise.toml` assembled from tickets 02–05, plus any new whole-file
dotfiles ticket 03 produced.

**Delete**: `playbook.yml`, `ansible.cfg`, `requirements.txt`, `requirements.yml`,
`pyproject.toml`, `uv.lock`, `.pre-commit-config.yaml`, and `[tasks.default]` from
`mise.toml`. Check `.python-version` and the `idiomatic_version_file_enable_tools`
setting against whatever `[tools]` ended up holding — they may or may not still earn
their place.

**Rewrite** `.github/workflows/ci.yml` per ticket 06, and the README: install mise
via its official one-liner, then `mise bootstrap`. The old two-step
Homebrew-then-venv instructions go. Keep the xkcd comic.

Pin a mise floor: bootstrap needs ≥ 2026.7.4 to be non-experimental, and ≥ 2026.8.4
for per-package `os` filters. Ticket 01 also corrected `[bootstrap.repos]` — the key
is `ref`, not `version`, and there is no `HEAD`; for the simple-bar clone either omit
`ref` (apply then never pulls) or pin `ref = "main"`.

Acceptance: CI green on `macos-latest`. Running it on the real Mac happens after
merge, and anything it surfaces is a follow-up, not a blocker.

## Added by ticket 06

The full `.github/workflows/ci.yml` is written out in
[06-ci-rewrite.md](06-ci-rewrite.md) — copy it, do not re-derive it. Also remove the
`pip` entry from `.github/dependabot.yml`; the `github-actions` entry stays.

Two assumptions ticket 06 could not check without a real run:

1. **Ticket 05's Claude-settings merge must create `~/.claude/settings.json` when it
   is absent.** Claude Code is not installed on a CI runner, so the file does not
   exist there, and CI asserts `jq -e '.outputStyle == "ASD-STE100"'` against it.
2. **`[bootstrap.macos.defaults]` is assumed to parse on `ubuntu-latest` without an
   `unknown field` warning.** The lint job fails on that warning. If macOS tables warn
   on Linux, move the parse gate into the macOS job instead.

The mise floor (≥ 2026.7.4, ≥ 2026.8.4 for `os` filters) is a README fact only. CI
pins no mise version: `mise-action`'s `version` input stays unset so the weekly cron
finds upstream breakage.

## Added by ticket 05

The `[bootstrap.hooks.pre-packages]` and `[tasks.bootstrap]` blocks are written out in
[05-imperative-block.md](05-imperative-block.md) — copy them, do not re-derive them.
Three things change what earlier tickets said:

1. **The two Homebrew lines live in `[bootstrap.hooks.pre-packages]`, not in
   `[tasks.bootstrap]`.** Ticket 02's text and this ticket's own reference to
   "tickets 02–05" both predate that. Step 17 is too late for ticket 02's reason.
2. **The `[dotfiles]` table is 10 entries, not 9.** Add
   `"~/.gitconfig" = { source = "dotfiles/gitconfig" }`, and add the new file
   `dotfiles/gitconfig` with the contents given in ticket 05.
3. **`jq` does not join `[bootstrap.packages]`.** It comes from `mise x jq@latest`
   inside the task. The package list stays at 20 entries.

One extra step on the first real run, on top of the two ticket 06 named: **back up
`~/.gitconfig` and pass `--force-dotfiles`.** It exists on this Mac as a real file,
and mise refuses to replace an unmanaged real file with a symlink.

## Settled before execution

Four open calls, answered in a session that stopped short of writing the diff. The
ticket stays open; whoever takes it next writes the diff to these answers.

1. **simple-bar `ref = "main"`.** Not omitted. Ansible used `version: "HEAD"`, which
   pulls the default branch on every run, so `"main"` reproduces today's behaviour.
   Omitting `ref` would clone once and freeze the widget.
2. **The mise floor stays a README fact.** No `min_version` key in `mise.toml`. This
   confirms what ticket 06 assumed. Accepted cost: an old mise fails late instead of
   fast.
3. **Delete `dotfiles/caveman.json`. Keep `dotfiles/apm-coding.yml`.** Both are
   referenced by no Ansible task and no bootstrap table — checked, nothing in the repo
   reads either one. The sweep takes the first and leaves the second in place.
4. **Delivery is a branch and a commit, not a push.** The PR is the user's to open.

### Facts gathered for the diff

- `dotfiles/Brewfile` holds 6 formulae and 16 casks. Minus `brew "mise"` and minus the
  AeroSpace cask, `[bootstrap.packages]` is 5 + 15 = the 20 entries ticket 02 settled.
- The local mise is 2026.7.5, which is above the 2026.7.4 bootstrap floor but below the
  2026.8.4 `os`-filter floor. 2026.8.8 is available.
