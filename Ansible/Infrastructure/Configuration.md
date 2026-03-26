---
tags:
  - ansible
  - ansible/infrastructure
aliases:
  - ansible.cfg
topic: Infrastructure
---

# Configuration (ansible.cfg)

## Search Order (First Found Wins)

1. `ANSIBLE_CONFIG` environment variable
2. `ansible.cfg` in the current directory
3. `~/.ansible.cfg`
4. `/etc/ansible/ansible.cfg`

## Generate a Default Config

```bash
ansible-config init --disabled > ansible.cfg
ansible-config init --disabled -t all > ansible.cfg   # includes plugin settings
```

## Common Settings

```ini
[defaults]
inventory         = ./inventory
remote_user       = deploy
forks             = 10
host_key_checking = False
roles_path        = ./roles:~/.ansible/roles
collections_path  = ./collections:~/.ansible/collections
timeout           = 30
retry_files_enabled = False
stdout_callback   = yaml
gathering         = smart
fact_caching       = jsonfile
fact_caching_connection = /tmp/ansible_facts_cache

[privilege_escalation]
become       = True
become_method = sudo
become_user   = root
become_ask_pass = False

[ssh_connection]
pipelining = True

[galaxy]
server_list = private_hub, release_galaxy

[galaxy_server.private_hub]
url = https://automation.my_org/
token = my_token

[galaxy_server.release_galaxy]
url = https://galaxy.ansible.com/
token = my_galaxy_token
```

## Settings That Trip You Up

| Setting | What it does | Common mistake |
|---|---|---|
| `collections_path` | Where Ansible searches for installed collections | Forgetting to add `./collections` so local installs aren't found |
| `roles_path` | Where Ansible searches for roles | Not including the project-local `./roles` directory |
| `gathering` | When to gather facts (`implicit`, `explicit`, `smart`) | Leaving at `implicit` and wondering why plays are slow |
| `host_key_checking` | SSH host key verification | Set to `False` for lab/dev, but **keep `True` in production** |
| `vault_password_file` | Default vault password file path | Accidentally committing the password file |
| `stdout_callback` | Output format (`default`, `yaml`, `json`, `debug`) | Using `default` when `yaml` is far more readable |
| `[galaxy] server_list` | Ordered list of Galaxy servers to search | Not listing your private hub first, so it hits public Galaxy |

## Precedence Within Settings

Config file < Environment variables < Command-line options < Playbook keywords < Variables

## Useful Environment Variable Overrides

```bash
ANSIBLE_CONFIG=/path/to/ansible.cfg              # override config search
ANSIBLE_INVENTORY=/path/to/inventory             # override default inventory
ANSIBLE_ROLES_PATH=./roles:~/.ansible/roles      # override roles search
ANSIBLE_COLLECTIONS_PATH=./collections           # override collections search
ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass.txt    # default vault password
ANSIBLE_HOST_KEY_CHECKING=False                  # disable SSH host key checks
ANSIBLE_STDOUT_CALLBACK=yaml                     # output format
ANSIBLE_FORCE_COLOR=1                            # force colored output (CI)
```
