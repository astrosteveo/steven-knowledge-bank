---
tags:
  - ansible
  - ansible/development
topic: Development
---

# Debugging

## Verbosity Levels

```bash
ansible-playbook site.yml -v        # basic output
ansible-playbook site.yml -vv       # task input shown
ansible-playbook site.yml -vvv      # connection details, task input/output
ansible-playbook site.yml -vvvv     # adds connection plugin details, script contents
ansible-playbook site.yml -vvvvv    # adds WinRM details (Windows)
```

## Debug Module

```yaml
- name: Print a variable
  ansible.builtin.debug:
    var: my_variable

- name: Print a message
  ansible.builtin.debug:
    msg: "Host {{ inventory_hostname }} has IP {{ ansible_host }}"

- name: Only show at high verbosity
  ansible.builtin.debug:
    msg: "Detailed info: {{ complex_var }}"
    verbosity: 2
```

## Task Debugger

Enable an interactive debugger when a task fails:

```yaml
# On a play
- hosts: all
  debugger: on_failed
  tasks: ...

# On a task
- name: Tricky task
  ansible.builtin.command: /opt/finicky_script.sh
  debugger: on_failed

# On a block
- block:
    - name: Something fragile
      ...
  debugger: on_failed
```

Debugger triggers: `always`, `never`, `on_failed`, `on_unreachable`, `on_skipped`.

Debugger commands at the interactive prompt:

| Command | Action |
|---|---|
| `p task` | Print task name |
| `p task.args` | Print task arguments |
| `p result` | Print task result |
| `p vars` | Print all variables |
| `p host` | Print current host |
| `r` | Rerun the task |
| `c` | Continue to next task |
| `q` | Quit the debugger and playbook |
| `u key=value` | Update a task argument and rerun |

## Enable Debugger via Environment

```bash
ANSIBLE_ENABLE_TASK_DEBUGGER=True ansible-playbook site.yml
```

Or in `ansible.cfg`:

```ini
[defaults]
enable_task_debugger = True
```
