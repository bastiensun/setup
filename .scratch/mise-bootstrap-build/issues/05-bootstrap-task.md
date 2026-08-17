# 05 — The bootstrap task runs the SSH key, the Claude merge, and apm

**What to build:** the three actions that no declarative table covers. A bootstrap run
generates an SSH key when none exists, selects the ASD-STE100 output style inside the
Claude settings file without discarding what Claude Code wrote there, and installs the
global apm skills.

Each is a guarded one-liner in the bootstrap task, not a shell script, and each is
idempotent — the task runs on every bootstrap.

**Blocked by:** 03 — apm reads a global config that the dotfiles phase symlinks, and the
output style itself is a dotfile.

**Status:** ready-for-agent

Work on the shared integration branch. Do not merge to main.

- [ ] The SSH key generation is guarded on the key file and takes an **empty
      passphrase**, matching the playbook. The line runs unattended, so CI's assertion
      passes. Minor known drift: `ssh-keygen` writes a user-and-host comment where
      Ansible wrote none. Nothing reads it.
- [ ] The Claude merge is **4 steps, not 1**: create the directory, seed an empty JSON
      object when the file is absent, merge into a temporary file, then move it into
      place.
- [ ] The seed is a hard requirement, not a nicety. Claude Code does not exist on a CI
      runner, so the file is absent there, and CI asserts the merged value reads back.
- [ ] The merge and the move are **two separate array entries**, not one joined command.
      A failing entry then stops the task on its own.
- [ ] `jq` is fetched on demand rather than installed as a package. The package list from
      ticket 02 stays at 20 entries. Checked while deciding: the mise registry resolves
      it.
- [ ] apm keeps the command the playbook uses today.
- [ ] CI gains three assertions: the AeroSpace bundle exists, the SSH key exists, and the
      merged Claude setting reads back. No status command can see these effects, and the
      risk is a guard that silently skips its own work.
- [ ] Both Ansible jobs still pass.

The full task block is written out verbatim in the decision ticket
[Write the [tasks.bootstrap] block](../../mise-bootstrap-migration/issues/05-imperative-block.md).

**An assumption that reports on itself**: the apm command rests on the tool-install phase
picking up the global config that the dotfiles phase symlinked earlier in the same run. If
that is false, apm is not on PATH, the command exits non-zero, and a failing bootstrap task
aborts the whole run. CI symlinks the same global config, so every run exercises it. No
extra assertion is needed.

**Accepted cost**: the CI assertion uses the runner's own `jq`, not the mise one. The macOS
image ships it. If a future image drops it, that assertion breaks while the bootstrap stays
green.
