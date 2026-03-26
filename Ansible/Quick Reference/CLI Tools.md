---
tags:
  - ansible
  - ansible/reference
aliases:
  - ansible-doc
  - ansible-inventory
  - ansible-config
topic: Quick Reference
---

# CLI Tools

Ansible ships several CLI tools beyond `ansible-playbook`. These are invaluable for debugging and exploration.

## ansible-doc

Browse module and plugin documentation without leaving the terminal:

```bash
ansible-doc ansible.builtin.copy              # full docs for a module
ansible-doc -s ansible.builtin.copy            # snippet (just the parameters)
ansible-doc -l                                 # list all modules
ansible-doc -l -t lookup                       # list all lookup plugins
ansible-doc -l -t callback                     # list all callback plugins
ansible-doc -t filter ansible.builtin.combine  # docs for a filter
ansible-doc -t inventory ansible.builtin.yaml  # docs for an inventory plugin
```

## ansible-inventory

Inspect and debug your inventory:

```bash
ansible-inventory -i inventory.yml --list              # dump full inventory as JSON
ansible-inventory -i inventory.yml --graph              # show group/host tree
ansible-inventory -i inventory.yml --graph --vars       # tree with variables
ansible-inventory -i inventory.yml --host web1          # variables for a specific host
```

## ansible-config

Inspect and debug your configuration:

```bash
ansible-config view                  # show the active config file
ansible-config dump                  # show all settings and their sources
ansible-config dump --only-changed   # show only non-default settings
ansible-config list                  # list all available settings with descriptions
```

## ansible-vault

```bash
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault view secrets.yml
ansible-vault encrypt file.yml
ansible-vault decrypt file.yml
ansible-vault encrypt_string 'secret' --name 'var_name'
ansible-vault rekey file.yml
```

## ansible-galaxy

```bash
ansible-galaxy collection init namespace.name
ansible-galaxy collection install namespace.name
ansible-galaxy collection build
ansible-galaxy collection publish tarball.tar.gz
ansible-galaxy collection list
ansible-galaxy role init role_name
ansible-galaxy role install author.role_name
ansible-galaxy role list
ansible-galaxy install -r requirements.yml
```
