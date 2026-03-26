---
tags:
  - ansible
  - ansible/playbooks
aliases:
  - when
  - loop
  - with_items
topic: Playbooks
---

# Conditionals and Loops

## When (Conditionals)

Bare expressions — no `{{ }}` needed:

```yaml
- name: Only on Debian
  ansible.builtin.apt:
    name: nginx
  when: ansible_facts['os_family'] == "Debian"

# AND (list form)
when:
  - ansible_facts['distribution'] == "CentOS"
  - ansible_facts['distribution_major_version'] == "8"

# OR
when: (env == "staging") or (force_deploy | bool)

# Check defined
when: my_var is defined

# Check result
when: result is failed
when: result is changed
when: result is skipped
```

## Loops

```yaml
# Simple list
- name: Create users
  ansible.builtin.user:
    name: "{{ item }}"
    state: present
  loop:
    - alice
    - bob

# List of dicts
- name: Create users with groups
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
  loop:
    - { name: alice, groups: wheel }
    - { name: bob, groups: root }

# Iterate over a dict
- name: Show config
  ansible.builtin.debug:
    msg: "{{ item.key }}={{ item.value }}"
  loop: "{{ server_configs | dict2items }}"

# Nested loops (cartesian product)
loop: "{{ ['alice','bob'] | product(['clientdb','employeedb']) | list }}"
```

## loop_control

```yaml
loop_control:
  label: "{{ item.name }}"           # cleaner output
  pause: 3                           # seconds between iterations
  index_var: idx                     # 0-indexed position
  loop_var: outer_item               # rename item (required for nested includes)
  extended: true                     # enables ansible_loop.first, .last, .index, .length
```

## Retry Until

```yaml
- name: Wait for service
  ansible.builtin.uri:
    url: http://localhost:8080/health
  register: result
  until: result.status == 200
  retries: 10
  delay: 5
```

Defaults: 3 retries, 5-second delay.

## Blocks (Try/Rescue/Always)

```yaml
- block:
    - name: Install package
      ansible.builtin.yum:
        name: httpd
        state: present

    - name: Start service
      ansible.builtin.service:
        name: httpd
        state: started

  rescue:
    - name: Handle failure
      ansible.builtin.debug:
        msg: "Installation failed, running recovery"

  always:
    - name: Always run cleanup
      ansible.builtin.debug:
        msg: "Cleanup complete"

  when: ansible_facts['os_family'] == "RedHat"
  become: true
```

Block-level `when` and `become` are inherited by all tasks inside. Loops cannot be applied at the block level.

In `rescue`, you can access `ansible_failed_task` and `ansible_failed_result`.
