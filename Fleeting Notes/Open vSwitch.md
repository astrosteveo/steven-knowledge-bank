# Open vSwitch

## Installation

1. Pull down the RPM's from some repo (likely just upload the RPM's to Nexus and pull them from there).
2. Install the RPMs.
```bash
sudo dnf localinstall python3-openvswitch3.1-3.1.0-116.el8s.x86_64.rpm openvswitch3.1-3.1.0-116.el8s.x86_64.rpm openvswitch-selinux-extra-policy-1.0-29.el8s.noarch.rpm
```
3. Enable the `openvswitch` systemd daemon.
```bash
sudo systemctl enable --now penvswitch
```
4. Label the configuration and runtime directories for SELinux
```bash
sudo restorecon -Rv /etc/openvswitch
sudo restorecon -Rv /var/run/openvswitch
```
5. Add the OVS bridge
```bash
sudo ovs-vsctl add-br br-isolated
```
6. Add the main interface to the bridge
```bash
sudo ovs-vsctl add-port br-isolated ens19
```
7. Remove the host IP from the main interface
```bash
sudo ip addr del 10.22.1.166/24 dev ens19
```
8. Add the host IP to the OVS bridge
```bash
sudo ip addr add 10.22.1.166/24 dev br-isolated
```
9. Bring the OVS bridge up
```bash
sudo ip link set br-isolated up
```