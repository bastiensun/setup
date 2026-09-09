# Adopt this repo as the global mise configuration

Status: ready-for-agent

Folded into the open pull request that replaces Ansible with `mise bootstrap`. That
pull request's own spec describes the intermediate flow — clone the repo, run
`mise bootstrap` inside it — which this spec replaces before either reaches the
default branch.

## Problem Statement

Installing a fresh Mac still takes three steps, not two: install mise, clone this repo
to some directory I choose, then run `mise bootstrap` inside it. The middle step is the
problem, and it is not only typing.

- The repo has no home. I pick a directory on each machine, and nothing records which
  one I picked. The dotfile symlinks on the machine point into that directory, so the
  choice is load-bearing and invisible.
- The repo has to symlink its own global config into place. `[dotfiles]` carries an
  entry that points `~/.config/mise/config.toml` at a tracked file, and two ordering
  assumptions rest on that symlink landing before the tool-install phase runs.
- The configuration lives in two files that must agree. Machine-wide tools sit in
  `dotfiles/mise-global.toml`; everything else sits in `mise.toml`. Answering "what
  does this machine get?" means reading both and knowing why they are apart.
- The first command a fresh machine runs is not the command the repo is about.

mise ships `mise bootstrap --adopt <url>`, which clones a repository into the global
mise configuration directory and bootstraps from it. That collapses the clone, the
placement, and the run into one command, and it removes the reason the config was split
across two files.

## Solution

A fresh machine reaches its finished state in two commands: install mise, then
`mise bootstrap --adopt bastiensun/setup`.

The repo stops being a project that is cloned somewhere and becomes the global mise
configuration directory itself. mise clones it into `$MISE_CONFIG_DIR` — normally
`~/.config/mise` — and loads `config.toml` from the repo root. That checkout is the
only copy on the machine and the working copy: edits, commits, and pushes happen there.

Because the repository is now the global configuration, the entry that symlinked the
global config into place is deleted along with the file it pointed at, and the
machine-wide `[tools]` block moves into the one `config.toml` at the root. One file
answers "what does this machine get?".

Everything the previous migration decided about the content of that configuration
stands unchanged: the same packages, the same dotfiles, the same macOS defaults, the
same repo clone, the same login shell, the same two hooks, and the same bootstrap task.
This spec changes where the configuration lives and how it arrives, not what it does.

CI keeps its single seam and applies it twice, to two different inputs: a change gate
that bootstraps the checked-out configuration from the real path it will occupy, and a
scheduled workflow that runs the documented command against the published default
branch.

## User Stories

1. As the owner of a fresh machine, I want to install mise and run one command, so that
   I do not choose a directory to clone into before I configure anything.
2. As the owner of a fresh machine, I want that one command to be the command the
   README shows, so that there is nothing to adapt between reading and typing.
3. As the owner of a fresh machine, I want the repository to land in the global mise
   configuration directory, so that every later mise command reads it without any
   further setup.
4. As the owner of a machine that already has a `~/.config/mise`, I want the README to
   tell me that directory must be cleared first, so that my first adopt does not fail
   on a destination mise refuses to overwrite.
5. As the owner of a machine that already has a real `~/.gitconfig`, I want the README
   to tell me the first run needs `--force-dotfiles`, so that mise does not refuse to
   replace an unmanaged file with a symlink.
6. As the repo maintainer, I want one `config.toml` at the repository root, so that I
   read one file to answer what this machine gets.
7. As the repo maintainer, I want the machine-wide tools in that same file, so that
   `apm`, `gh`, and `node` are not described in a second file that has to agree with
   the first.
8. As the repo maintainer, I want the global-config dotfile entry deleted, so that the
   configuration no longer symlinks itself into the position it already occupies.
9. As the repo maintainer, I want the ordering assumptions that rested on that symlink
   removed, so that no phase depends on an earlier phase having placed the config.
10. As the repo maintainer, I want the dotfile sources to keep their relative paths, so
    that the move costs no path rewriting and no new failure mode.
11. As the repo maintainer, I want `~/.config/mise` to be my working copy, so that
    there is exactly one checkout on the machine and no second copy to drift.
12. As the repo maintainer, I want `mise use -g` and `mise settings set` to show up as
    diffs in this repository, so that a machine-wide change I make by hand becomes a
    commit rather than an untracked edit.
13. As the repo maintainer, I want the planning directory swept, so that my global
    configuration directory does not carry the paper trail of a merged migration.
14. As the repo maintainer, I want the orphan dotfile deleted, so that the repository
    holds no file that nothing reads.
15. As the repo maintainer, I want the repository's own agent manifest kept, so that
    working on this repository from its new location still has its skills.
16. As the repo maintainer, I want the README to name the minimum mise version, so that
    an old mise does not silently skip a table it cannot parse.
17. As a Mac user, I want every package, dotfile, macOS default, repo clone, and login
    shell from the previous migration to behave identically, so that changing how the
    configuration arrives does not change what it produces.
18. As the repo maintainer, I want CI to bootstrap the checked-out configuration from
    the path it will really occupy, so that the gate tests the arrangement the machine
    will have rather than an arrangement that exists only in CI.
19. As the repo maintainer, I want that gate to run on every pull request, so that a
    broken configuration fails in review rather than on my machine.
20. As the repo maintainer, I want a second workflow that runs the documented adopt
    command against the published repository, so that the one command in the README is
    proven and not merely written down.
21. As the repo maintainer, I want that workflow to run when the default branch
    changes, so that a merge tells me within minutes whether the documented command
    still works.
22. As the repo maintainer, I want that workflow to run weekly, so that drift in
    packages pinned to `latest` and upstream mise changes surface on a schedule.
23. As the repo maintainer, I want to be able to start that workflow by hand, so that I
    can answer "does the one-liner still work?" without waiting for the schedule.
24. As the repo maintainer, I want the two workflows separated, so that a red result
    tells me which kind of thing broke: my change, or the world.
25. As the repo maintainer, I want the workflow named for the scenario rather than for
    the flag, so that renaming a mise flag does not strand the file name.
26. As the repo maintainer, I want the workflow named without an operating system, so
    that adding Linux later grows a matrix rather than a second file.
27. As the repo maintainer, I want the scheduled workflow to clear the runner's global
    configuration directory first, so that adopt does not fail on a destination the
    mise installer created.
28. As the repo maintainer, I want both workflows to skip the login-shell phase, so
    that a `chsh` prompt cannot hang a job until it times out.
29. As the repo maintainer, I want both workflows to assert the effects no status
    command can see, so that a guard which wrongly skips its own work gets caught.
30. As the repo maintainer, I want a second run to report nothing missing, so that I can
    trust the run converged.
31. As the repo maintainer, I want the weekly cron to live in the scheduled workflow
    only, so that the change gate is not re-run against a branch nothing changed.
32. As the repo maintainer, I want this change folded into the open pull request, so
    that the default branch never carries a documented flow that is replaced a day
    later.

## Implementation Decisions

### Shape

- **The repository becomes the global mise configuration directory.** `--adopt` clones
  it into `$MISE_CONFIG_DIR` and bootstraps from it. The repo is no longer cloned to a
  directory of my choosing.
- **`--adopt` is the global-configuration form, not the shared-dotfile-history form.**
  The same flag dispatches on repository contents: a repository carrying a
  `.mise-history/format.toml` marker is treated as a setup repository whose contents a
  background watcher owns and synchronises. That shape is rejected — it replaces a
  hand-authored, reviewable repository with a machine-written history store, which ends
  pull requests and CI, and it only pays off across several machines.
- **The repository is addressed by its `owner/repo` shorthand**, which `--adopt`
  accepts alongside a full Git URL. The repository is public, so no authentication is
  involved.
- **One `config.toml` at the repository root.** No `conf.d/` fragments. The
  configuration is about sixty lines; splitting it would add an alphabetical load-order
  rule to hold in mind and would defeat the one-file goal. Splitting stays available if
  it grows.
- **The machine-wide `[tools]` block moves into that `config.toml`**, and the file that
  held it is deleted along with the `[dotfiles]` entry that symlinked it to
  `~/.config/mise/config.toml`. The dotfiles table goes from ten entries to nine.
- **Two ordering assumptions disappear.** The previous design required the dotfiles
  phase to symlink the global config before the tools phase installed from it, and
  before the bootstrap task ran `apm install --global`. Under adopt the configuration is
  present from the moment mise clones it, so neither assumption exists to be violated.
  The apm global config remains a dotfile and keeps its ordering requirement.
- **Dotfile sources keep their relative paths.** Relative sources resolve against the
  directory containing the configuration file, so `dotfiles/…` resolves under
  `~/.config/mise/dotfiles/…` with no edit. Note for future work: `{{config_root}}` for
  a global config is `$HOME`, not the repository — nothing templates on it today.
- **The checkout at `~/.config/mise` is the working copy.** Edits, commits and pushes
  happen there. A second clone elsewhere is rejected: the dotfile symlinks point into
  `~/.config/mise` regardless, so edits made in a second checkout would not take effect
  until someone remembered to pull.
- **No configuration content changes.** Packages, dotfile targets and modes, macOS
  defaults, the repo clone, the login shell, both hooks and the bootstrap task carry
  over from the previous migration unchanged.

### Consequences accepted

- **`mise use -g` and `mise settings set` write to a tracked file.** The global config
  is now in the repository, so machine-wide mise changes dirty the worktree. This is the
  same trade already accepted for the git config and the Zed settings, and it is
  arguably the point: such a change becomes a diff to commit.
- **Nothing else of mise's lands in the repository.** Trust records, installed tools and
  the cache live in the state, data and cache directories respectively, so the adopted
  checkout needs no new ignore entries for mise's own files.
- **The working copy accumulates agent artifacts.** Because the checkout is where I work
  on the repository, running the agent tooling there creates its module and settings
  directories inside `~/.config/mise`. All are already ignored, and mise reads only the
  configuration file.
- **Two first-run prerequisites exist on a machine that is not fresh**: an existing
  non-empty `~/.config/mise` must be cleared, because adopt accepts an existing
  destination only when it is already a checkout of the same repository; and an existing
  real `~/.gitconfig` needs `--force-dotfiles`, because mise refuses to replace an
  unmanaged real file with a symlink. Both are README facts, not code. Neither can be
  automated from inside the repository, because neither has anywhere to run before the
  repository exists.

### Repository sweep

- **Delete the planning directory.** It holds the map, the decision tickets and the spec
  for the migration being merged, none of which were ever committed — the directory has
  always been untracked working-tree state, on every branch including the one that
  carries the open pull request. Nothing is lost from version control by removing it,
  and a global configuration directory is the wrong place to carry the paper trail of
  finished work regardless. Called out explicitly in the pull request rather than
  folded in silently, since a reviewer with the same untracked files locally would
  otherwise see them vanish with no explanation in the diff.
- **Delete the orphan dotfile** that no `[dotfiles]` entry points at. The previous spec
  kept it by explicit decision; moving the repository into the configuration directory
  is the moment to stop carrying a file nothing reads.
- **Keep the repository's own agent manifest and its lockfile.** They are what give the
  agent its skills when working in the checkout, and the checkout is now where that work
  happens. They are distinct from the machine-wide agent config, which is a dotfile.
- **Keep the README, the ignore file and the workflows.** All are ordinary repository
  contents and none is read by mise.

### README

- Two commands: the mise installer, then the adopt command in its shorthand form.
- The minimum mise version stays a README fact. No minimum-version key in the
  configuration and no pin in CI.
- A short first-run note carries both prerequisites above. Three or four lines under the
  code block, not a section.
- The xkcd automation comic stays.

### CI

Two workflows, split by purpose rather than by subject.

- **The change gate** keeps its existing file and runs on pull requests and on pushes to
  the default branch. Before mise runs, the checkout is placed at the real global
  configuration path, so the job exercises the same path, the same trust status and the
  same relative source resolution the machine will have. Rejected alternatives: pointing
  the global-config-file variable at the checkout, which tests the same file at a path
  that never exists in reality and leaves trust behaviour at a non-standard path
  untested; and adopting a local path, which is undocumented and would still clone the
  default branch rather than the pull request.
- **The scheduled workflow is a new file**, named for the scenario — a new machine —
  rather than for the `--adopt` flag or for macOS. Naming it for the flag pins the file
  to mise's current vocabulary; naming it for the operating system blocks the growth
  path, which is a matrix over runners in one workflow when Linux is added. It runs on
  pushes to the default branch, on a weekly schedule, and on manual dispatch. It clears
  the runner's global configuration directory, then runs the literal documented command
  against the published repository, then repeats the assertions.
- **The weekly cron moves out of the change gate** into the scheduled workflow. Drift
  detection is that workflow's job, and re-running a change gate against a branch nothing
  changed produces an ambiguous signal.
- **The split is the point.** A red change gate means the pull request is wrong. A red
  scheduled workflow means the world moved — a cask vanished, or mise shipped a
  regression. Mixing them into one workflow makes both signals ambiguous, and a job
  gated to skip on pull requests would render as skipped on every pull request.
- **Sharing the assertions across the two workflows is not attempted.** Two jobs in one
  file duplicate their assertion steps exactly as much as two files do; real sharing
  would need a composite action or a reusable workflow, which is more machinery than a
  handful of assertions deserves.
- **Both skip the login-shell phase**, for the reason already established: `chsh` has no
  sudo path and no way to answer a prompt on a runner with no TTY, and a hang costs a
  job timeout rather than a fast failure.

### Delivery

- **Folded into the open pull request** that replaces Ansible, pushed directly to its
  branch. The default branch never carries the intermediate clone-then-bootstrap flow.
- **The pull request description is updated** to describe the adopt flow. The spec of
  the intermediate design was never committed, so nothing stands in git history as its
  record; this spec's own text is the record of what superseded it.

## Testing Decisions

**The seam is unchanged, and there is still only one.** This repository holds no
application code, so there is no unit to test and no module boundary to test at. The one
honest seam is a real `mise bootstrap` on a clean runner, asserted from outside by
inspecting the state of the machine. This spec adds no seam. It applies the existing
seam to a second input.

**A good test here asserts the state of the machine, never the mechanism that produced
it.** "The window manager's application bundle is present" is a good assertion. "mise
ran the pre-packages hook" is not: it tests the implementation and breaks the moment a
line moves between a hook and a task, which already happened twice while this
configuration was being charted. The same rule is why the scheduled workflow is named
for the scenario and not for the flag.

**The two inputs to the seam:**

1. **The checkout**, on every pull request and on pushes to the default branch. Proves
   the configuration in front of the reviewer is correct. This is the gate.
2. **The published repository, through the documented command**, on pushes to the
   default branch, weekly, and on demand. Proves the entrypoint the README promises
   still works. It always tests the default branch, which is why it cannot serve as the
   pull request gate.

**The assertions, in three groups, applied to both inputs:**

1. **Parse.** Listing the tasks on Linux must not warn about an unknown field. A typo'd
   table or key name only warns and exits zero, so a phase would otherwise fail silently
   by doing nothing. This job is unchanged from the previous migration and keeps its
   place in the change gate.
2. **Convergence.** After a real run, the status checks must report nothing missing
   across packages, repos, dotfiles and macOS defaults. This is the idempotence proof.
   Whether the single aggregate status command can now replace the four narrower ones is
   an open implementation question: one of the two reasons it was split — that the
   aggregate tools check also read the separately symlinked global config — is removed by
   this change, but the other reason, that skipping the login-shell phase makes the
   aggregate report the login shell as differing forever, may still hold. Try the
   aggregate; keep the narrow checks if it still fails.
3. **Effects invisible to status.** Direct checks for what the imperative surface
   produces — the window manager's application bundle and the generated SSH key —
   because no status command models them.

**Prior art**: the change gate is the job the previous migration introduced, and before
that the Ansible integration job that ran the playbook twice and grepped for zero changed
and zero failed. The shape is unchanged across all three: a real run, then a proof that
nothing is left to do. Only the input and the question's phrasing change.

**Not tested, knowingly:**

- The login-shell phase. Both workflows skip it, so the step most likely to break has no
  coverage.
- The Homebrew installer line. The macOS runner ships Homebrew, so the guard is a no-op
  there and the installer path never executes.
- The window manager's cask on an untrusted tap. CI installs it, but a trust prompt on a
  fresh personal machine would not reproduce on a runner.
- The two first-run prerequisites. Both exist only on a machine that already has a global
  mise configuration directory or a real global git config; a runner has neither, and the
  scheduled workflow deliberately clears the one it might have.

**Assumptions the first real run must check**, each with a known small fix:

1. Adopt refuses the existing `~/.config/mise` as documented, and clearing it is
   sufficient. Fix: a more detailed README note.
2. The dotfiles phase creates missing parent directories. Fix: a directories list.
3. Relative dotfile sources resolve under the adopted checkout as documented. Fix:
   absolute or templated sources.

## Out of Scope

- **The shared-dotfile-history form of `--adopt`.** Rejected in favour of the
  global-configuration form; revisiting it is a separate decision that would end the
  repository's reviewable, hand-authored shape.
- **Linux or any non-macOS support.** The destination is one Mac. The naming decisions
  deliberately leave room for it; nothing else does.
- **Multi-machine provisioning and remote SSH targets**, though bootstrap supports both.
- **Changing any configuration content.** No package, dotfile, default, hook or task
  changes meaning in this spec.
- **The repository's own agent manifest and lockfile.** Kept as they are; their long-term
  fate is a separate cleanup.
- **Sharing CI assertions through a composite action or reusable workflow.** Rejected as
  more machinery than the assertions justify.
- **Running the bootstrap on the real machine.** That happens after merge. Anything it
  surfaces is a follow-up, not a blocker.
- **Making the pull request gate exercise `--adopt` itself.** Adopt clones the default
  branch, so it can never test the change under review.

## Further Notes

**Why the flag, and not a documented clone step.** Adopt is not only shorter. It fixes
where the repository lives, which fixes where the dotfile symlinks point, which is the
one piece of state the previous design left to the operator's memory on each machine.

**Interaction and privileges are unchanged.** Exactly two prompts can appear, both
outside CI: the Homebrew installer in the pre-packages hook, and the login-shell change.
The generated SSH key takes an empty passphrase and needs no prompt.

**The naming decision has a growth path attached.** When Linux support arrives, the
scheduled workflow becomes a matrix over runners inside the same file, with each leg
named by its runner. A per-OS file name would instead push toward a second workflow file
duplicating the same assertions — the drift the two-workflow split was chosen to avoid.

**One decision was superseded while charting.** The scheduled verification was first
placed as a second job inside the change gate's workflow, on the argument that adjacency
keeps the assertions from drifting. That argument is weak: two jobs in one file duplicate
their steps as much as two files do, and the gated job would render as skipped on every
pull request. The separate workflow is the corrected position.
