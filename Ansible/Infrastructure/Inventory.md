---
tags:
  - ansible
  - ansible/infrastructure
topic: Infrastructure
---

# Inventory

An inventory defines the hosts and groups Ansible manages. Every inventory has two implicit groups: `all` (every host) and `ungrouped` (hosts not in any named group).

## YAML Format (Recommended)

```yaml
ungrouped:
  hosts:
    mail.example.com:
webservers:
  hosts:
    foo.example.com:
    bar.example.com:
dbservers:
  hosts:
    one.example.com:
    two.example.com:
prod:
  children:
    webservers:
    dbservers:
atlanta:
  hosts:
    host1:
      http_port: 80
    host2:
      http_port: 303
  vars:
    ntp_server: ntp.atlanta.example.com
```

## INI Format

```ini
mail.example.com

[webservers]
foo.example.com
bar.example.com

[dbservers]
one.example.com
two.example.com

[prod:children]
webservers
dbservers

[atlanta:vars]
ntp_server=ntp.atlanta.example.com
proxy=proxy.atlanta.example.com
```

## Group Nesting (Children)

Groups can contain other groups using `children`. This lets you build hierarchies — target a parent group and Ansible automatically includes all hosts in its child groups.

```yaml
# YAML
production:
  children:
    webservers:          # all webservers hosts are also in production
    dbservers:           # all dbservers hosts are also in production
    loadbalancers:

# Target "production" and you hit every host across all three child groups
```

```ini
# INI
[production:children]
webservers
dbservers
loadbalancers
```

You can nest multiple levels deep:

```yaml
all_environments:
  children:
    production:
      children:
        prod_web:
          hosts:
            web1.prod.example.com:
        prod_db:
          hosts:
            db1.prod.example.com:
    staging:
      children:
        staging_web:
          hosts:
            web1.staging.example.com:
```

Variables set on a parent group are inherited by all children. Child group variables override parent group variables. See [Inventory Merge Order](#inventory-merge-order) below.

## Ranges

```
www[01:50].example.com
db-[a:f].example.com
www[01:50:2].example.com   # stride of 2
```

## Host and Group Variables (File-Based)

Store variables in separate files alongside your inventory for clarity and version control:

```
inventory/
  hosts.yml
  group_vars/
    all.yml
    webservers.yml
  host_vars/
    specific_host.yml
```

## Common Connection Variables

| Variable | Purpose |
|---|---|
| `ansible_host` | IP or hostname to connect to |
| `ansible_port` | SSH port |
| `ansible_user` | Login user |
| `ansible_connection` | Connection plugin (`ssh`, `local`, `docker`) |
| `ansible_ssh_private_key_file` | Private key path |
| `ansible_python_interpreter` | Python path on target |
| `ansible_become` | Enable privilege escalation |
| `ansible_become_user` | Escalation target user |

## Inventory Merge Order

`all` group → parent group → child group → host. Child group vars override parent. Use `ansible_group_priority` (higher number = higher priority) to break ties between groups at the same level.
