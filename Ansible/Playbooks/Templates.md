---
tags:
  - ansible
  - ansible/playbooks
aliases:
  - Jinja2
  - jinja2
topic: Playbooks
---

# Templates (Jinja2)

All templating runs on the **control node** before anything is sent to targets.

```yaml
- name: Deploy config
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/app/app.conf
    owner: root
    mode: "0644"
```

## Common Filters

```yaml
# Defaults
"{{ my_var | default('fallback') }}"
"{{ my_var | default(omit) }}"
"{{ my_var | mandatory }}"

# Type conversion
"{{ value | int }}"
"{{ value | bool }}"

# Data formats
"{{ data | to_json }}"
"{{ data | to_nice_yaml(indent=2) }}"
"{{ json_string | from_json }}"

# String
"{{ name | upper }}"
"{{ name | lower }}"
"{{ name | replace('old', 'new') }}"
"{{ name | regex_replace('^foo(.+)$', 'bar\\1') }}"
"{{ cmd | quote }}"

# Path
"{{ path | basename }}"
"{{ path | dirname }}"
"{{ paths | path_join }}"

# Lists and dicts
"{{ list | unique }}"
"{{ list1 | union(list2) }}"
"{{ list1 | difference(list2) }}"
"{{ list | flatten }}"
"{{ list | sort }}"
"{{ dict1 | combine(dict2) }}"
"{{ dict | dict2items }}"
"{{ items | items2dict }}"

# Hashing and encoding
"{{ secret | hash('sha256') }}"
"{{ secret | password_hash('sha512') }}"
"{{ data | b64encode }}"
"{{ encoded | b64decode }}"

# Ternary
"{{ condition | ternary('yes', 'no') }}"
```

## Common Tests

```yaml
when: my_var is defined
when: my_var is undefined
when: result is failed
when: result is succeeded
when: result is changed
when: result is skipped
when: my_version is version('2.0', '>=')
when: my_list is subset(other_list)
when: path is file
when: path is directory
when: path is exists
when: value is string
when: value is number
when: value is mapping
```

## Lookups

Lookups pull data from external sources on the control node:

```yaml
vars:
  motd_content: "{{ lookup('file', '/etc/motd') }}"
  home_dir: "{{ lookup('env', 'HOME') }}"
  my_password: "{{ lookup('password', '/tmp/pwfile length=16') }}"
  secret: "{{ lookup('pipe', 'openssl rand -hex 16') }}"

# query() returns a list (preferred in loops)
loop: "{{ query('fileglob', 'files/*.conf') }}"
```

List available lookups: `ansible-doc -l -t lookup`
