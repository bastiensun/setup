# ⚙️ Setup

Ansible playbook to automate the configuration of my Mac.

[![Automation (xkcd)](https://imgs.xkcd.com/comics/automation.png)](https://xkcd.com/1319/)

## Installation

```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
python3 -m venv venv
source venv/bin/activate
pip3 install --requirement requirements.txt
ansible-galaxy install --role-file requirements.yml
```

## Run

```shell
source venv/bin/activate.fish
ansible-playbook --ask-become-pass playbook.yml
```
