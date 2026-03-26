---
tags:
  - ansible
  - ansible/development
aliases:
  - ansible-lint
topic: Development
---

# Ansible Lint

Ansible Lint checks playbooks, roles, and collections for best practices and potential issues.

## Installation

```bash
pip install ansible-lint
```

## Running

```bash
ansible-lint site.yml
ansible-lint roles/my_role/
ansible-lint                          # scans current directory
ansible-lint -p site.yml              # parseable output (for CI)
ansible-lint --fix site.yml           # auto-fix where possible
ansible-lint -L                       # list all rules
```

## Common Rules

| Rule ID                     | Description                                    |
| --------------------------- | ---------------------------------------------- |
| `yaml`                      | YAML syntax and formatting issues              |
| `name[missing]`             | Tasks should be named                          |
| `name[casing]`              | Task names should start with uppercase         |
| `no-changed-when`           | Commands/shell tasks need `changed_when`       |
| `command-instead-of-module` | Use a module instead of command/shell          |
| `risky-file-permissions`    | File tasks should set explicit mode            |
| `no-free-form`              | Avoid free-form module syntax                  |
| `fqcn`                      | Use fully qualified collection names           |
| `schema`                    | YAML content does not match expected schema    |
| `jinja[spacing]`            | Jinja2 template spacing                        |
| `package-latest`            | Package installs should use a specific version |
| `no-tabs`                   | Most YAML should not contain tabs              |
| `role-name`                 | Role names must match `^[a-z][a-z0-9_]+$`      |

## Configuration (.ansible-lint)

Create `.ansible-lint` in your project root:

```yaml
---
skip_list:
  - name[missing]
  - yaml[line-length]

warn_list:
  - package-latest
  - no-changed-when

exclude_paths:
  - .cache/
  - .github/
  - collections/

enable_list:
  - fqcn

use_default_rules: true

offline: false
```

## Inline Skips

```yaml
- name: This is fine
  ansible.builtin.command: echo hello  # noqa: no-changed-when

- name: Skip multiple
  ansible.builtin.shell: make build  # noqa: command-instead-of-module no-changed-when
```
