---
tags:
  - ansible
  - ansible/organization
topic: Organization
---

# Roles

Roles package reusable automation into a standard directory structure.

## Directory Structure

```
roles/
  my_role/
    tasks/main.yml            # entry point for tasks
    handlers/main.yml         # handlers
    defaults/main.yml         # low-precedence default variables
    vars/main.yml             # high-precedence variables
    files/                    # static files (used by copy, script)
    templates/                # Jinja2 templates (used by template)
    meta/main.yml             # role dependencies, Galaxy metadata
    meta/argument_specs.yml   # argument validation (2.11+)
    tests/                    # tests
```

Only directories with content are required. Omit any you don't need.

## defaults/ vs vars/

- **`defaults/`** — Lowest precedence. Intended to be overridden by the consumer of the role. Use for "sane defaults."
- **`vars/`** — High precedence. Only overridden by extra vars (`-e`), task vars, and a few other sources. Use for internal role constants that shouldn't be casually overridden.

## Using Roles

```yaml
# In the roles: section (static, runs before tasks)
- hosts: webservers
  roles:
    - common
    - role: nginx
      vars:
        nginx_port: 8080

# As a task (dynamic)
- hosts: webservers
  tasks:
    - include_role:
        name: nginx
      when: use_nginx | bool

    - import_role:
        name: common
```

## Role Dependencies (meta/main.yml)

```yaml
---
dependencies:
  - role: common
    vars:
      some_parameter: 3
  - role: apache
    vars:
      apache_port: 80
```

Dependencies run before the dependent role. Duplicate dependencies execute only once unless their parameters differ or `allow_duplicates: true` is set.

## Role Search Path

1. Collections
2. `roles/` directory relative to the playbook
3. Configured `roles_path` (default: `~/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles`)
4. Playbook directory

## Creating a Role Skeleton

```bash
ansible-galaxy role init my_role_name
```
