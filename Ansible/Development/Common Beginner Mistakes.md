---
tags:
  - ansible
  - ansible/development
topic: Development
---

# Common Beginner Mistakes

Pitfalls that catch almost everyone when starting with Ansible.

## Forgetting to quote Jinja2 expressions

```yaml
# BROKEN — YAML parser sees { as the start of a dict
app_dir: {{ base_dir }}/app

# CORRECT
app_dir: "{{ base_dir }}/app"
```

**Rule:** If a value starts with `{{`, the entire value must be quoted.

## Using command/shell instead of modules

```yaml
# Bad — not idempotent, always reports changed
- name: Install nginx
  ansible.builtin.command: yum install -y nginx

# Good — idempotent, only changes when needed
- name: Install nginx
  ansible.builtin.yum:
    name: nginx
    state: present
```

Modules check current state before acting. `command` and `shell` don't — they always run and always report "changed" unless you add `changed_when`/`creates`/`removes`.

## Confusing role defaults/ vs vars/

```yaml
# defaults/main.yml — meant to be overridden by the consumer
nginx_port: 80

# vars/main.yml — internal constants, hard to override
nginx_user: www-data
```

New users put everything in `vars/` and then wonder why they can't override values from their playbook. Use `defaults/` for anything the consumer should be able to customize.

## Not understanding variable precedence

A variable set in `roles/x/vars/main.yml` (precedence 15) beats one set in `group_vars/` (precedence 6-7). Extra vars (`-e`) always win (precedence 22). When a variable isn't what you expect, check the [precedence table](#precedence-lowest-to-highest).

## Mutating variables and expecting them to persist

```yaml
# This won't work the way you think
- set_fact:
    my_list: "{{ my_list + ['new_item'] }}"
```

Variables set with `set_fact` are **host-scoped**. If you set a fact on `host1`, `host2` doesn't see it. Access other hosts' facts via `hostvars['host1'].my_list`.

## Running playbooks from the wrong directory

Ansible resolves relative paths (inventory, roles, `ansible.cfg`) from the **current working directory**, not the playbook's location. If your `ansible.cfg` says `inventory = ./inventory`, you must run `ansible-playbook` from the project root.

```bash
# From inside playbooks/ — won't find inventory or ansible.cfg
cd ~/project/playbooks && ansible-playbook deploy.yml

# From project root — correct
cd ~/project && ansible-playbook playbooks/deploy.yml
```

## Using `ansible.cfg` in your home directory without realizing it

Ansible picks up `~/.ansible.cfg` as a fallback. If you have one left over from testing, it can silently override settings in your project's `ansible.cfg`. Run `ansible-config dump --only-changed` to see where your active settings are coming from.

## Ignoring check mode

Always test with `--check --diff` before running for real against production:

```bash
ansible-playbook site.yml --check --diff -i inventories/production/hosts.yml
```

This shows you what _would_ change without making any changes. Not all modules support check mode, but most do.

## Hardcoding hosts in playbooks

```yaml
# Bad — tightly coupled to specific hosts
- hosts: web1.example.com, web2.example.com

# Good — targets a group, let inventory decide membership
- hosts: webservers
```

Playbooks should target **groups**. Which hosts are in a group is an inventory concern, not a playbook concern. This keeps playbooks reusable across environments.

## Not using `--diff` when debugging templates

```bash
ansible-playbook site.yml --diff
```

`--diff` shows line-by-line changes for files managed by `template`, `copy`, `lineinfile`, etc. Without it, you just see "changed" with no idea what actually changed.

## Treating Ansible like a scripting language

Ansible is **declarative** — describe the end state, not the steps. If you find yourself writing long chains of `command`/`shell` tasks with `register` and `when`, step back and look for a module that does what you want. There's almost always one.
