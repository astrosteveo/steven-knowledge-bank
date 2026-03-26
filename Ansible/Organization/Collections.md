---
tags:
  - ansible
  - ansible/organization
topic: Organization
---

# Collections

Collections bundle modules, plugins, roles, and playbooks under a namespace. Always use the **Fully Qualified Collection Name (FQCN)** format: `namespace.collection.resource`.

```yaml
# FQCN examples
ansible.builtin.yum
ansible.builtin.template
community.general.json_query
amazon.aws.ec2_instance
```

## Creating a Collection

```bash
ansible-galaxy collection init my_namespace.my_collection
```

Naming rules: both `namespace` and `name` must be lowercase alphanumeric + underscores, minimum 3 characters, cannot start with `_`. Reserved namespaces: `ansible` (Red Hat official), `community` (community-owned), `local` (for machine-local / git-only collections to avoid Galaxy collisions).

## Collection Directory Structure

```
my_namespace/
  my_collection/
    galaxy.yml              # Required — collection metadata
    README.md
    meta/
      runtime.yml           # requires_ansible, plugin_routing, action_groups
      execution-environment.yml
    plugins/
      modules/              # Custom modules
      inventory/            # Inventory plugins
      lookup/               # Lookup plugins
      filter/               # Filter plugins
      test/                 # Jinja2 test plugins
      callback/             # Callback plugins
      connection/           # Connection plugins
      action/               # Action plugins
      become/               # Become plugins
      cache/                # Cache plugins (fact caching only)
      vars/                 # Vars plugins (never auto-loaded)
      module_utils/         # Shared Python utility code
    roles/
      role_name/
        tasks/main.yml
        defaults/main.yml
        ...
    playbooks/
      deploy.yml
    docs/
    tests/
    changelogs/
```

Only `galaxy.yml` is strictly required. All directories are optional.

## galaxy.yml

```yaml
namespace: my_namespace
name: my_collection
version: 1.0.0
readme: README.md
authors:
  - Jane Doe <jane@example.com>
description: A collection for managing widgets
license:
  - MIT
tags:
  - networking
  - cloud
dependencies:
  ansible.netcommon: ">=2.0.0,<3.0.0"
  ansible.utils: "*"
  community.general: ">=7.0.0"
repository: https://github.com/my_namespace/my_collection
build_ignore:
  - playbooks/sensitive
  - '*.tar.gz'
```

| Field | Required | Description |
|---|---|---|
| `namespace` | Yes | Collection namespace |
| `name` | Yes | Collection name |
| `version` | Yes | SemVer-compatible version |
| `readme` | Yes | Path to README.md |
| `authors` | Yes | List of authors |
| `description` | No | Short summary |
| `license` | No | SPDX license identifiers (mutually exclusive with `license_file`) |
| `dependencies` | No | Dict of `namespace.name: "version_range"` |
| `tags` | No | For Galaxy search/indexing |
| `repository` | No | URL to SCM repo |
| `build_ignore` | No | Glob patterns to exclude from build |

Version range operators: `*`, `==`, `!=`, `>=`, `>`, `<=`, `<`. Combine with commas: `">=2.0.0,<3.0.0"`.

## meta/runtime.yml

```yaml
---
requires_ansible: ">=2.14.0"
plugin_routing:
  modules:
    old_module_name:
      redirect: my_namespace.my_collection.new_module_name
    deprecated_module:
      deprecation:
        removal_version: "3.0.0"
        warning_text: Use new_module instead
action_groups:
  my_group:
    - module1
    - module2
```

## Installing Collections

```bash
ansible-galaxy collection install community.general
ansible-galaxy collection install community.general:">= 7.0.0"
ansible-galaxy collection install my_namespace-my_collection-1.0.0.tar.gz -p ./collections
```

Default install path: `~/.ansible/collections`. The `-p` flag automatically appends `ansible_collections/` to the path unless the parent is already named `ansible_collections`.

## Installing from Git

```bash
# HTTPS
ansible-galaxy collection install git+https://github.com/org/repo.git
ansible-galaxy collection install git+https://github.com/org/repo.git,devel        # branch
ansible-galaxy collection install git+https://github.com/org/repo.git,v1.2.0       # tag
ansible-galaxy collection install git+https://github.com/org/repo.git,7b60ddc      # commit

# Subdirectory (multi-collection repos)
ansible-galaxy collection install git+https://github.com/org/repo.git#/path/to/collection/

# SSH
ansible-galaxy collection install git@github.com:org/repo.git

# Local
ansible-galaxy collection install git+file:///home/user/path/to/repo.git
```

## requirements.yml

```yaml
---
roles:
  - name: geerlingguy.java
    version: "1.9.6"

collections:
  - name: community.general
    version: ">=7.0.0"
    source: https://galaxy.ansible.com
  - name: ansible.posix
  - name: git+https://github.com/org/repo.git
    type: git
    version: devel
```

```bash
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml    # collections only
```

The `type` field accepts: `galaxy` (default), `git`, `url`, `file`, `dir`, `subdirs`.

## collections_path in ansible.cfg

```ini
[defaults]
collections_path = ./collections:~/.ansible/collections:/usr/share/ansible/collections
```

For local-to-project installation, structure it like this:

```
project/
  ansible.cfg
  requirements.yml
  collections/
    ansible_collections/
      my_namespace/
        my_collection/
  playbooks/
  roles/
  inventories/
```

## Multiple Galaxy Servers

```ini
[galaxy]
server_list = private_hub, release_galaxy

[galaxy_server.private_hub]
url=https://automation.my_org/
username=my_user
password=my_pass

[galaxy_server.release_galaxy]
url=https://galaxy.ansible.com/
token=my_token
```

Servers are searched in listed order when installing. Override tokens via environment: `ANSIBLE_GALAXY_SERVER_PRIVATE_HUB_TOKEN=secret`.

## Using Collections in Playbooks

```yaml
# Always prefer FQCN
- hosts: all
  tasks:
    - name: Install package
      ansible.builtin.yum:
        name: httpd
        state: present

    - name: Use a lookup (FQCN required)
      ansible.builtin.debug:
        msg: "{{ lookup('my_namespace.my_collection.my_lookup', 'param1') }}"

    - name: Use a filter (FQCN required)
      ansible.builtin.debug:
        msg: "{{ data | my_namespace.my_collection.my_filter }}"
```

## Calling Playbooks from a Collection

Requires ansible-core 2.11+. Playbook filenames must be lowercase alphanumeric + underscores only (no dashes).

```bash
# From the CLI
ansible-playbook my_namespace.my_collection.deploy -i inventory.yml
```

```yaml
# From another playbook via import
- ansible.builtin.import_playbook: my_namespace.my_collection.deploy
```

## Using Roles from a Collection

```yaml
# FQCN in roles section
- hosts: webservers
  roles:
    - my_namespace.my_collection.web_role

# FQCN with import/include
- hosts: webservers
  tasks:
    - ansible.builtin.import_role:
        name: my_namespace.my_collection.web_role
    - ansible.builtin.include_role:
        name: my_namespace.my_collection.web_role
```

## Building and Publishing

```bash
# Build the tarball (max 20 MB)
ansible-galaxy collection build

# Publish to Galaxy (configure token in ansible.cfg first)
ansible-galaxy collection publish my_namespace-my_collection-1.0.0.tar.gz

# Publish without waiting for import to complete
ansible-galaxy collection publish my_namespace-my_collection-1.0.0.tar.gz --no-wait
```

Each publish requires a **new version number** — you cannot modify an already-published version.

## Roles in Collections vs Standalone Roles

| | Standalone Role | Collection Role |
|---|---|---|
| Plugins | Can contain plugins in `library/`, `filter_plugins/`, etc. | **Cannot** contain plugins — all plugins must be in collection's top-level `plugins/` |
| Naming | Flexible | Lowercase alphanumeric + `_` only, must start with alpha |
| `role_name` in meta | Used as the role name | **Ignored** — directory name is the role name |
| Plugin resolution | Searches role dirs, then playbook dirs | Implicitly searches own collection first |
| `collections:` keyword inheritance | Inherits from playbook | **Does not** inherit — must declare in own `meta/main.yml` |

## Common Gotchas

**FQCN is not optional for non-action plugins.** The `collections:` keyword only creates a search path for modules and action plugins. Lookups, filters, and tests always need FQCN:

```yaml
# This won't work for lookups/filters even with the collections keyword set
msg: "{{ lookup('my_lookup', 'param') | my_filter }}"

# This works
msg: "{{ lookup('my_namespace.my_collection.my_lookup', 'param') | my_namespace.my_collection.my_filter }}"
```

**Roles don't inherit the playbook's `collections:` list.** Each role must declare its own:

```yaml
# In roles/my_role/meta/main.yml
collections:
  - my_namespace.first_collection
  - my_namespace.second_collection
```

**Vars plugins are never auto-loaded.** They must be explicitly enabled via FQCN.

**Collection playbook names cannot contain dashes.** `deploy-app.yml` won't be addressable by FQCN. Use `deploy_app.yml` instead.

**Pre-release versions are ignored by default.** To install them: `ansible-galaxy collection install ns.coll:==1.0.0-beta.1`.

**Python imports must use the full path.** Inside a collection's Python code:

```python
# Correct
from ansible_collections.my_namespace.my_collection.plugins.module_utils.helpers import my_func

# Wrong — relative imports don't work
from .module_utils.helpers import my_func
```
