---
tags:
  - ansible
  - ansible/getting-started
topic: Getting Started
---

# Installation

## pip (Recommended)

```bash
pip install ansible              # full package (ansible-core + community collections)
pip install ansible-core          # minimal (just ansible.builtin, no community content)
```

## Package Managers

```bash
# RHEL/CentOS/Fedora
sudo dnf install ansible-core

# Ubuntu/Debian
sudo apt update && sudo apt install ansible

# macOS
brew install ansible
```

## Verify

```bash
ansible --version
ansible-community --version      # if using the full ansible package
```

## What Gets Installed

| Package | What's included |
|---|---|
| `ansible-core` | The engine, `ansible-playbook`, `ansible-galaxy`, `ansible-vault`, `ansible-doc`, and the `ansible.builtin` collection only |
| `ansible` | `ansible-core` + a curated set of community collections (community.general, ansible.posix, etc.) |

If you install `ansible-core` only, you need to manually install any collections beyond `ansible.builtin`.
