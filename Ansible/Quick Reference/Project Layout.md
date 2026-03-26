---
tags:
  - ansible
  - ansible/reference
topic: Quick Reference
---

# Project Layout

A typical Ansible project:

```
project/
  ansible.cfg
  requirements.yml
  site.yml                       # master playbook
  webservers.yml                 # play targeting webservers
  dbservers.yml                  # play targeting dbservers
  inventories/
    production/
      hosts.yml
      group_vars/
        all.yml
        webservers.yml
      host_vars/
        web1.example.com.yml
    staging/
      hosts.yml
      group_vars/
      host_vars/
  roles/
    common/
      tasks/main.yml
      handlers/main.yml
      defaults/main.yml
      vars/main.yml
      files/
      templates/
      meta/main.yml
    nginx/
      ...
  collections/
    requirements.yml
  playbooks/
    deploy.yml
    rollback.yml
  group_vars/                    # playbook-level group vars
  host_vars/                     # playbook-level host vars
  files/
  templates/
  filter_plugins/
```
