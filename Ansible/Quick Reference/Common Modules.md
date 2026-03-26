---
tags:
  - ansible
  - ansible/reference
topic: Quick Reference
---

# Common Modules

| Module (FQCN) | Purpose |
|---|---|
| `ansible.builtin.command` | Run a command (no shell features) |
| `ansible.builtin.shell` | Run a command through the shell |
| `ansible.builtin.raw` | Run a raw SSH command (no Python needed) |
| `ansible.builtin.script` | Run a local script on remote hosts |
| `ansible.builtin.copy` | Copy files to remote hosts |
| `ansible.builtin.template` | Deploy Jinja2 templates |
| `ansible.builtin.file` | Manage file properties |
| `ansible.builtin.lineinfile` | Manage lines in text files |
| `ansible.builtin.blockinfile` | Manage blocks of text in files |
| `ansible.builtin.yum` | Manage packages (RHEL/CentOS) |
| `ansible.builtin.apt` | Manage packages (Debian/Ubuntu) |
| `ansible.builtin.pip` | Manage Python packages |
| `ansible.builtin.service` | Manage services |
| `ansible.builtin.systemd` | Manage systemd services |
| `ansible.builtin.user` | Manage user accounts |
| `ansible.builtin.group` | Manage groups |
| `ansible.builtin.cron` | Manage cron jobs |
| `ansible.builtin.git` | Manage git checkouts |
| `ansible.builtin.uri` | Interact with HTTP/HTTPS APIs |
| `ansible.builtin.get_url` | Download files |
| `ansible.builtin.unarchive` | Unpack archives |
| `ansible.builtin.debug` | Print debug messages |
| `ansible.builtin.assert` | Assert conditions |
| `ansible.builtin.fail` | Fail with a message |
| `ansible.builtin.set_fact` | Set host-level variables |
| `ansible.builtin.include_vars` | Load variables from files |
| `ansible.builtin.wait_for` | Wait for a condition |
| `ansible.builtin.setup` | Gather facts |
| `ansible.builtin.meta` | Execute internal actions (flush_handlers, end_play) |
| `ansible.posix.synchronize` | Rsync wrapper |
| `kubernetes.core.k8s` | Manage Kubernetes resources |
