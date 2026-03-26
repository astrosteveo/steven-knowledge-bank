---
tags:
  - ansible
  - ansible/execution
aliases:
  - async
  - poll
topic: Execution
---

# Async Tasks

For long-running operations that might exceed the SSH timeout, or when you want to run tasks in parallel on a single host.

## Fire and Check Later

```yaml
- name: Run long database export
  ansible.builtin.command: /opt/export_db.sh
  async: 3600        # max runtime in seconds
  poll: 0            # don't wait, fire and forget
  register: export_job

- name: Do other work while export runs
  ansible.builtin.command: /opt/prep_storage.sh

- name: Wait for export to finish
  async_status:
    jid: "{{ export_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 30
```

## Fire and Poll

```yaml
- name: Run long task, checking every 10 seconds
  ansible.builtin.command: /opt/slow_task.sh
  async: 1800
  poll: 10
```

`poll: 0` means fire-and-forget. Any positive value polls at that interval. `async` without `poll` defaults to `poll: 15`.
