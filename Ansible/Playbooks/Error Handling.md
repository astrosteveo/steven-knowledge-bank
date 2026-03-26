---
tags:
  - ansible
  - ansible/playbooks
aliases:
  - rescue
  - block
  - ignore_errors
topic: Playbooks
---

# Error Handling

## ignore_errors

Continue execution even if a task fails:

```yaml
- name: Try something that might fail
  ansible.builtin.command: /opt/maybe-missing-binary
  ignore_errors: true
```

## failed_when

Override what Ansible considers a failure:

```yaml
- name: Run a script that uses exit codes creatively
  ansible.builtin.command: /opt/check_status.sh
  register: result
  failed_when: result.rc not in [0, 2]

# Multiple conditions
- name: Check output
  ansible.builtin.command: grep -c ERROR /var/log/app.log
  register: result
  failed_when:
    - result.rc != 0
    - "'No such file' not in result.stderr"
```

## changed_when

Control when a task reports "changed" (which also controls handler notifications):

```yaml
- name: Check if reboot is needed
  ansible.builtin.command: needs-restarting -r
  register: reboot_check
  changed_when: reboot_check.rc == 1
  failed_when: reboot_check.rc > 1

# Never report changed
- name: Read-only check
  ansible.builtin.command: cat /etc/hostname
  changed_when: false
```

## any_errors_fatal

Stop the entire play immediately if any host fails:

```yaml
- hosts: webservers
  any_errors_fatal: true
  tasks:
    - name: Critical migration
      ansible.builtin.command: /opt/migrate.sh
```

## max_fail_percentage

Allow a play to continue as long as a minimum percentage of hosts succeed:

```yaml
- hosts: webservers
  max_fail_percentage: 30
  serial: 10
  tasks:
    - name: Deploy app
      ...
```

If more than 30% of the current batch fails, the play aborts.

## ignore_unreachable

Continue running tasks on a host that becomes unreachable:

```yaml
- name: Try to connect
  ansible.builtin.ping:
  ignore_unreachable: true
  register: ping_result

- name: Only if reachable
  ansible.builtin.debug:
    msg: "Host is up"
  when: ping_result is not unreachable
```
