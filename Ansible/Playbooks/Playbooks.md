---
tags:
  - ansible
  - ansible/playbooks
topic: Playbooks
---

# Playbooks

A playbook contains one or more plays. Each play targets a set of hosts and defines tasks to run on them.

```yaml
---
- name: Configure web servers
  hosts: webservers
  remote_user: deploy
  become: true

  tasks:
    - name: Install nginx
      ansible.builtin.yum:
        name: nginx
        state: present

    - name: Deploy config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx

  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

## Play Execution Order

1. `pre_tasks` (and any triggered handlers)
2. Role dependencies
3. `roles:`
4. `tasks:` (and any triggered handlers)
5. `post_tasks` (and any triggered handlers)

## Running Playbooks

```bash
ansible-playbook site.yml
ansible-playbook site.yml -f 10                      # 10 forks
ansible-playbook site.yml --check                    # dry run
ansible-playbook site.yml --diff                     # show file diffs
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --list-tasks
ansible-playbook site.yml --tags "config,deploy"
ansible-playbook site.yml --skip-tags "debug"
ansible-playbook site.yml -e "env=prod version=2.1"  # extra vars
```

## Imports vs Includes

| | `import_*` (Static) | `include_*` (Dynamic) |
|---|---|---|
| Processed | At parse time | At runtime |
| Loops | Not supported | Supported |
| `--list-tasks` | Shows child tasks | Shows only the include |
| Tags/when | Applied to all child tasks | Applied to the include itself |
| Performance | Faster | More overhead |

```yaml
tasks:
  - import_tasks: common.yml            # static
  - include_tasks: dynamic.yml          # dynamic, supports loops/conditionals
    when: some_condition

  - import_role:
      name: my_role                      # static role import
  - include_role:
      name: my_role                      # dynamic role include
```

Use `import_playbook` to reuse entire playbooks (there is no `include_playbook`).
