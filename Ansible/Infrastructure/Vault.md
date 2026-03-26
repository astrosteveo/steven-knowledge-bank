---
tags:
  - ansible
  - ansible/infrastructure
aliases:
  - ansible-vault
topic: Infrastructure
---

# Vault

Vault encrypts sensitive data (passwords, keys, credentials) so it can live safely in version control.

## Encrypting Files

```bash
ansible-vault create secrets.yml
ansible-vault encrypt vars/prod.yml
ansible-vault view secrets.yml
ansible-vault edit secrets.yml
ansible-vault decrypt secrets.yml
ansible-vault rekey secrets.yml --new-vault-id newlabel@prompt
```

## Encrypting Individual Strings

```bash
ansible-vault encrypt_string 'supersecret' --name 'db_password'
ansible-vault encrypt_string --vault-id prod@prompt 'supersecret' --name 'db_password'
echo -n 'supersecret' | ansible-vault encrypt_string --stdin-name 'db_password'
```

Use the output directly in a variable file:

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  6563...
```

## Using Vault in Playbooks

```bash
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass.txt
ansible-playbook site.yml --vault-id dev@dev-password --vault-id prod@prompt
```

## Vault IDs

Labels that differentiate between multiple vault passwords:

```bash
ansible-vault encrypt --vault-id dev@prompt dev-secrets.yml
ansible-vault encrypt --vault-id prod@prompt prod-secrets.yml
ansible-playbook site.yml --vault-id dev@dev-pass --vault-id prod@prod-pass
```

## Persistent Defaults

Set `ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass.txt` in your environment or `vault_password_file` in `ansible.cfg` to avoid passing the flag every time.
