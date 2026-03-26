---
tags:
  - ansible
  - ansible/variables
topic: Variables and Facts
---

# Environment Variables

## Setting Environment Variables

```yaml
# Play level — applies to all tasks
- hosts: all
  environment:
    http_proxy: http://proxy.example.com:8080
    https_proxy: http://proxy.example.com:8080
  tasks: ...

# Task level
- name: Install pip package behind proxy
  ansible.builtin.pip:
    name: flask
  environment:
    http_proxy: http://proxy.example.com:8080

# Block level
- block:
    - name: Step 1
      ansible.builtin.command: make build
    - name: Step 2
      ansible.builtin.command: make install
  environment:
    PATH: "/opt/toolchain/bin:{{ ansible_env.PATH }}"
    LD_LIBRARY_PATH: /opt/toolchain/lib
```

## Using Variables for Reuse

```yaml
vars:
  proxy_env:
    http_proxy: http://proxy.example.com:8080
    https_proxy: http://proxy.example.com:8080
    no_proxy: "localhost,127.0.0.1,.example.com"

tasks:
  - name: Install package
    ansible.builtin.yum:
      name: httpd
    environment: "{{ proxy_env }}"
```
