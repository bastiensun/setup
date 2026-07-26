# ⚙️ Setup

Ansible playbook to automate the configuration of my Mac.

[![Automation (xkcd)](https://imgs.xkcd.com/comics/automation.png)](https://xkcd.com/1319/)

## Initial installation (without `homebrew` and `uv`)

```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
python3 -m venv .venv
source .venv/bin/activate
pip3 install --requirement requirements.txt
ansible-galaxy install --role-file requirements.yml
ansible-playbook --ask-become-pass playbook.yml
```

## Installation

```
mise run
```
