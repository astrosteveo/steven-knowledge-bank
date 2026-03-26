---
tags:
  - ansible
  - ansible/execution
aliases:
  - delegate_to
topic: Execution
---

# Delegation

## delegate_to

Run a task on a different host than the one being targeted:

```yaml
- name: Remove host from load balancer
  ansible.builtin.uri:
    url: "https://lb.example.com/api/remove/{{ inventory_hostname }}"
    method: POST
  delegate_to: localhost

- name: Deploy application
  ansible.builtin.yum:
    name: myapp
    state: latest

- name: Add host back to load balancer
  ansible.builtin.uri:
    url: "https://lb.example.com/api/add/{{ inventory_hostname }}"
    method: POST
  delegate_to: localhost
```

## local_action

Shorthand for `delegate_to: localhost`:

```yaml
- name: Send notification
  local_action:
    module: ansible.builtin.uri
    url: https://hooks.slack.com/services/XXX
    method: POST
    body: '{"text": "Deploy starting"}'
    body_format: json
```

## run_once

Execute a task only once for the entire play, regardless of how many hosts are targeted:

```yaml
- name: Run database migration
  ansible.builtin.command: /opt/app/migrate.sh
  run_once: true
  delegate_to: "{{ groups['dbservers'][0] }}"
```

`run_once` runs on the first host in the batch. Combine with `delegate_to` when the task should run on a specific host.

## delegate_facts

When delegating, store gathered facts on the delegated host instead of the current host:

```yaml
- name: Gather facts from db server
  ansible.builtin.setup:
  delegate_to: db1.example.com
  delegate_facts: true
```
