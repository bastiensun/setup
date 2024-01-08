# Setup

## Installation

```shell
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
