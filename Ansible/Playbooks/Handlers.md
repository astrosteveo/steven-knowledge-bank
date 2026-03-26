---
tags:
  - ansible
  - ansible/playbooks
topic: Playbooks
---

# Handlers

Handlers are tasks that run only when notified by a task reporting `changed`. They run once per section (pre_tasks, roles/tasks, post_tasks), regardless of how many tasks notify them.

```yaml
tasks:
  - name: Deploy app config
    ansible.builtin.template:
      src: app.conf.j2
      dest: /etc/app/app.conf
    notify:
      - Restart app
      - Reload nginx

handlers:
  - name: Restart app
    ansible.builtin.service:
      name: app
      state: restarted

  - name: Reload nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
```

## Listen Groups

Group multiple handlers under one notification topic:

```yaml
handlers:
  - name: Restart memcached
    ansible.builtin.service:
      name: memcached
      state: restarted
    listen: "restart web services"

  - name: Restart apache
    ansible.builtin.service:
      name: apache
      state: restarted
    listen: "restart web services"
```

Notify with `notify: "restart web services"` to trigger both.

## Flush Handlers Early

```yaml
tasks:
  - name: Some task
    ...
    notify: My handler

  - meta: flush_handlers

  - name: Task that depends on handler having run
    ...
```
