---
tags:
  - ansible
  - ansible/variables
aliases:
  - gather_facts
  - ansible_facts
topic: Variables and Facts
---

# Facts

## Gathering Facts

Facts are automatically gathered at the start of each play (via the `setup` module). Disable for speed when you don't need them:

```yaml
- hosts: all
  gather_facts: false
  tasks: ...
```

## Accessing Facts

```yaml
"{{ ansible_facts['os_family'] }}"
"{{ ansible_facts['distribution'] }}"
"{{ ansible_facts['default_ipv4']['address'] }}"
"{{ ansible_facts['hostname'] }}"
"{{ ansible_facts['memtotal_mb'] }}"
"{{ ansible_facts['processor_vcpus'] }}"
"{{ ansible_facts['mounts'] }}"
```

## Custom Local Facts

Place `.fact` files (INI or JSON) in `/etc/ansible/facts.d/` on managed hosts. They appear under `ansible_local`:

```ini
# /etc/ansible/facts.d/app.fact
[general]
app_version=1.2.3
environment=production
```

```yaml
"{{ ansible_local['app']['general']['app_version'] }}"
```

## Fact Caching

Cache facts across playbook runs to avoid re-gathering. Configure in `ansible.cfg`:

```ini
[defaults]
gathering = smart             # only gather if not cached
fact_caching = jsonfile        # or redis, memcached, yaml
fact_caching_connection = /tmp/ansible_facts_cache
fact_caching_timeout = 86400   # seconds (24 hours)
```

Gathering options:
- `implicit` (default) — gather at play start unless `gather_facts: false`
- `explicit` — only gather when `gather_facts: true`
- `smart` — only gather if facts are not already cached

## Filtering Facts

Gather only what you need for speed:

```yaml
- name: Only gather network facts
  ansible.builtin.setup:
    gather_subset:
      - network
      - "!hardware"

- name: Gather a specific fact
  ansible.builtin.setup:
    filter: ansible_default_ipv4
```
