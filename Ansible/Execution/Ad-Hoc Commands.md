---
tags:
  - ansible
  - ansible/execution
topic: Execution
---

# Ad-Hoc Commands

**Syntax:** `ansible <pattern> -m <module> -a "<args>"`

```bash
ansible all -m ping
ansible webservers -a "/sbin/reboot" -f 10
ansible all -m shell -a "echo $HOME"
ansible all -m copy -a "src=/etc/hosts dest=/tmp/hosts"
ansible all -m file -a "dest=/tmp/test mode=0755 state=directory"
ansible all -m yum -a "name=httpd state=present"
ansible all -m service -a "name=httpd state=started"
ansible all -m setup                                    # gather facts
ansible all -m copy -a "src=file dest=/tmp/file" -C     # check mode (dry run)
```

**Useful flags:** `-f 10` (forks), `--become -K` (privilege escalation), `-u username` (remote user).
