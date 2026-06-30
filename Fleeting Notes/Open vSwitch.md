# Open vSwitch

## Installation

1. Pull down the RPM's from some repo (likely just upload the RPM's to Nexus and pull them from there).
2. Install the RPMs.
```bash
dnf localinstall python3-openvswitch3.1-3.1.0-116.el8s.x86_64.rpm openvswitch3.1-3.1.0-116.el8s.x86_64.rpm openvswitch-selinux-extra-policy-1.0-29.el8s.noarch.rpm
```
3. Enable the openvswitch systemd daemon.
   ```bash
   systemctl enable --now penvswitch
   ```
4. Label the configuration and runtime directories for SELinux
```bash
sudo restorecon -Rv /etc/openvswitch
sudo restorecon -Rv /var/run/openvswitch
```
1. Add the OVS bridge
2. 