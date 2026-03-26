---
tags:
  - ansible
  - ansible/playbooks
topic: Playbooks
---

# Tags

Tags let you selectively run or skip parts of a playbook.

## Applying Tags

```yaml
# On a task
- name: Install nginx
  ansible.builtin.yum:
    name: nginx
  tags:
    - packages
    - nginx

# On a block
- block:
    - name: Install deps
      ansible.builtin.yum:
        name: "{{ item }}"
      loop: "{{ packages }}"
  tags: setup

# On a role
roles:
  - role: common
    tags: common
  - role: nginx
    tags:
      - nginx
      - web

# On an include/import
- import_tasks: db.yml
  tags: database

- include_tasks: monitoring.yml
  tags: monitoring
```

## Tag Inheritance

- `import_*` — tags are applied to **every task** inside the import
- `include_*` — tags apply only to the **include statement itself**, not the tasks within

## Running with Tags

```bash
ansible-playbook site.yml --tags "config,deploy"
ansible-playbook site.yml --skip-tags "debug,test"
ansible-playbook site.yml --list-tags             # show available tags
```

## Special Tags

| Tag | Behavior |
|---|---|
| `always` | Runs unless explicitly skipped with `--skip-tags always` |
| `never` | Skipped unless explicitly included with `--tags never` |

```yaml
- name: Debug output
  ansible.builtin.debug:
    msg: "{{ some_var }}"
  tags:
    - never
    - debug
```

This task only runs when you pass `--tags debug`.
