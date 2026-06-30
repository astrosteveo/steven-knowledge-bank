# Open vSwitch

## Archived CentOS NFV OpenvSwitch Repo

```text
# CentOS-NFV-OpenvSwitch.repo
#
# Please see http://wiki.centos.org/SpecialInterestGroup/NFV for more
# information

[centos-nfv-openvswitch]
name=CentOS-$releasever - NFV OpenvSwitch
baseurl=http://vault.centos.org/centos/$nfvsigdist/nfv/$basearch/openvswitch-2/
#mirrorlist=http://#mirrorlist.centos.org/?release=$nfvsigdist&arch=$basearch&repo=nfv-openvswitch-2
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-NFV
module_hotfixes=1

[centos-nfv-openvswitch-testing]
name=CentOS-$releasever - NFV OpenvSwitch Testing
baseurl=http://buildlogs.centos.org/centos/$nfvsigdist/nfv/$basearch/openvswitch-2/
gpgcheck=0
enabled=0
module_hotfixes=1

[centos-nfv-openvswitch-debuginfo]
name=CentOS-$releasever - NFV OpenvSwitch - Debug
baseurl=http://debuginfo.centos.org/centos/$nfvsigdist/nfv/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-NFV
module_hotfixes=1

[centos-nfv-openvswitch-source]
name=CentOS-$releasever - NFV OpenvSwitch - Source
baseurl=http://vault.centos.org/centos/$releasever/nfv/Source/openvswitch-2/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-NFV
module_hotfixes=1
```

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