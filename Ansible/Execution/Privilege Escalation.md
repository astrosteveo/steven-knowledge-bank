---
tags:
  - ansible
  - ansible/execution
aliases:
  - become
  - sudo
topic: Execution
---

# Privilege Escalation

## Directives

| Directive | Default | Description |
|---|---|---|
| `become` | `false` | Activate privilege escalation |
| `become_user` | `root` | Target user |
| `become_method` | `sudo` | Escalation method (`sudo`, `su`, `doas`, `pbrun`, `runas`) |
| `become_flags` | — | Extra flags (e.g., `-s /bin/sh`) |

## Usage Levels

```yaml
# Play level
- hosts: webservers
  become: true
  become_user: root
  tasks:
    - name: Install package
      ansible.builtin.yum:
        name: httpd

# Task level
- name: Run as app user
  ansible.builtin.command: whoami
  become: true
  become_user: appuser

# Block level
- block:
    - name: Task 1
      ...
    - name: Task 2
      ...
  become: true
  become_user: deploy
```

## CLI Flags

```bash
ansible-playbook site.yml --become -K          # -K prompts for password
ansible-playbook site.yml --become-user deploy
ansible-playbook site.yml --become-method su
```

## Key Limitation

Privilege escalation methods cannot be chained (e.g., `sudo` to user A then `su` to user B). The become method must have general, unrestricted access because Ansible runs modules from randomly named temp files.
