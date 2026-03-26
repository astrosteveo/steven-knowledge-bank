---
tags:
  - ansible
  - ansible/execution
aliases:
  - serial
  - rolling update
topic: Execution
---

# Serial and Rolling Updates

## serial

Control how many hosts are processed at a time (rolling deployments):

```yaml
# Fixed number
- hosts: webservers
  serial: 3
  tasks: ...

# Percentage
- hosts: webservers
  serial: "25%"
  tasks: ...

# Ramping (canary pattern)
- hosts: webservers
  serial:
    - 1        # first, deploy to 1 host
    - 5        # then 5 at a time
    - "25%"    # then 25% at a time
  tasks: ...
```

If any batch fails (respecting `max_fail_percentage`), remaining batches are skipped.

## order

Control the order hosts are processed within each batch:

```yaml
- hosts: webservers
  order: sorted          # alphabetical
  serial: 3
  tasks: ...
```

Options: `inventory` (default), `sorted`, `reverse_sorted`, `reverse_inventory`, `shuffle`.

## strategy

Control how tasks are executed across hosts:

```yaml
# Linear (default) — each task completes on all hosts before the next task starts
- hosts: all
  strategy: linear
  tasks: ...

# Free — each host runs as fast as it can, independent of others
- hosts: all
  strategy: free
  tasks: ...

# Host pinned — like free, but keeps forks per host (useful for interactive debugging)
- hosts: all
  strategy: host_pinned
  tasks: ...
```

Set a default in `ansible.cfg`:

```ini
[defaults]
strategy = linear
```

## throttle

Limit concurrency for a specific task, even when forks or serial allows more:

```yaml
- name: Hit rate-limited API
  ansible.builtin.uri:
    url: https://api.example.com/register/{{ inventory_hostname }}
  throttle: 2
```
