---
tags:
  - ansible
  - ansible/variables
topic: Variables and Facts
---

# Variables

## Precedence (Lowest to Highest)

| # | Source |
|---|---|
| 1 | Command-line values (`-u my_user` — not extra vars) |
| 2 | Role defaults (`defaults/main.yml`) |
| 3 | Inventory file or script group vars |
| 4 | Inventory `group_vars/all` |
| 5 | Playbook `group_vars/all` |
| 6 | Inventory `group_vars/*` |
| 7 | Playbook `group_vars/*` |
| 8 | Inventory file or script host vars |
| 9 | Inventory `host_vars/*` |
| 10 | Playbook `host_vars/*` |
| 11 | Host facts / cached `set_facts` |
| 12 | Play `vars:` |
| 13 | Play `vars_prompt:` |
| 14 | Play `vars_files:` |
| 15 | Role vars (`vars/main.yml`) |
| 16 | Block vars |
| 17 | Task vars (inline) |
| 18 | `include_vars` |
| 19 | `set_facts` / registered vars |
| 20 | Role and `include_role` params |
| 21 | `include` params |
| 22 | Extra vars (`-e`) — **always wins** |

## Registered Variables

```yaml
- name: Check disk space
  ansible.builtin.command: df -h /
  register: disk_result

- name: Show output
  ansible.builtin.debug:
    msg: "{{ disk_result.stdout }}"
```

Common attributes: `.stdout`, `.stderr`, `.rc`, `.stdout_lines`, `.changed`, `.failed`, `.skipped`. In loops, results land in `.results` as a list.

## Magic Variables

| Variable | Description |
|---|---|
| `inventory_hostname` | Name of the current host as defined in inventory |
| `ansible_host` | Actual IP/hostname to connect to |
| `groups` | Dict of all groups → list of hosts |
| `group_names` | Groups the current host belongs to |
| `hostvars` | All host variables (access other hosts: `hostvars['db1'].ip`) |
| `playbook_dir` | Directory of the current playbook |
| `role_path` | Directory of the current role |
| `ansible_facts` | Gathered facts for the current host |
| `ansible_check_mode` | `true` if running in check mode |
| `ansible_version` | Dict with version info |
| `omit` | Special value to skip a module parameter |

## Variable Naming Rules

- Letters, numbers, underscores only
- Cannot start with a number
- Cannot use Python or Ansible keywords

## YAML Quoting Rule

If a value starts with a Jinja2 expression, quote the whole thing:

```yaml
# Wrong
var: {{ some_value }}

# Correct
var: "{{ some_value }}"
```
