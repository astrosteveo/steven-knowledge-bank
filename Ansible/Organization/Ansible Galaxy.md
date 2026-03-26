---
tags:
  - ansible
  - ansible/organization
aliases:
  - ansible-galaxy
topic: Organization
---

# Ansible Galaxy

## The Galaxy Ecosystem

"Galaxy" refers to both the protocol and the ecosystem of content hubs — not just the public website.

| Platform | What it is | Who runs it |
|---|---|---|
| **Ansible Galaxy** (`galaxy.ansible.com`) | Free, public community hub | Community / Red Hat |
| **Automation Hub** (`console.redhat.com`) | Red Hat's hosted hub with certified, supported content | Red Hat (requires AAP subscription) |
| **Private Automation Hub** | Self-hosted, on-prem instance for your organization | You (ships with AAP) |
| **Galaxy NG** | Open-source upstream project that powers all of the above | Community (can be deployed standalone without AAP) |

The `ansible-galaxy` CLI speaks the same protocol to all of them. You choose where to pull content from by configuring servers in `ansible.cfg`:

```ini
[galaxy]
server_list = private_hub, certified_hub, community_galaxy

[galaxy_server.private_hub]
url = https://hub.internal.example.com/api/galaxy/
token = my_private_token

[galaxy_server.certified_hub]
url = https://console.redhat.com/api/automation-hub/
auth_url = https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
token = my_rh_token

[galaxy_server.community_galaxy]
url = https://galaxy.ansible.com/
```

Servers are searched in the order listed. Put your private hub first so internal collections are preferred. This is how most organizations work — internal content takes priority, certified Red Hat content is the fallback, and community Galaxy is the last resort.

## Why This Matters

When you run `ansible-galaxy collection install some_namespace.some_collection`, Ansible checks each server in order. If your private hub has a collection with the same namespace and name as one on public Galaxy, the private one wins. This is how teams distribute internal collections without publishing them publicly.

## Roles

```bash
ansible-galaxy role install geerlingguy.apache
ansible-galaxy role install geerlingguy.apache,3.2.0
ansible-galaxy role install git+https://github.com/user/repo.git,commit_or_branch
ansible-galaxy role install -r requirements.yml
ansible-galaxy role list
ansible-galaxy role remove geerlingguy.apache
ansible-galaxy role search elasticsearch --author geerlingguy
ansible-galaxy role info geerlingguy.apache
ansible-galaxy role init my_new_role
```

Default install path: `~/.ansible/roles` (override with `--roles-path` or `ANSIBLE_ROLES_PATH`).

## Collections

```bash
ansible-galaxy collection install community.general
ansible-galaxy collection install -r requirements.yml
ansible-galaxy collection list
```
