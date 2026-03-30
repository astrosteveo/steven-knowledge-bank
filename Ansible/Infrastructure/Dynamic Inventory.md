---
tags:
  - ansible
  - ansible/infrastructure
topic: Infrastructure
---

# Dynamic Inventory

## Why Static Inventory Doesn't Scale

Static inventory files work well when your infrastructure is stable — a handful of servers that rarely change. In cloud environments, this breaks down quickly:

- **Instances are ephemeral.** Auto-scaling groups spin up and terminate instances constantly. A static inventory is stale the moment you save it.
- **IP addresses change.** Cloud instances get new IPs on restart. Hardcoding IPs means constant manual updates.
- **Multiple environments multiply the problem.** Dev, staging, production, each with their own dynamic infrastructure, means maintaining multiple inventory files.
- **Infrastructure as Code creates drift.** Terraform or CloudFormation provisions new servers, but nobody updates the Ansible inventory.

Dynamic inventory solves this by querying your infrastructure provider's API at runtime and building the inventory on the fly.

## Inventory Plugins vs Inventory Scripts

Ansible supports two mechanisms for dynamic inventory. **Plugins are the modern approach** and should be used for all new work.

| Aspect | Inventory Plugins | Inventory Scripts |
|---|---|---|
| **Format** | YAML configuration file | Executable script (Python, Bash, etc.) |
| **Integration** | Native Ansible integration | External process called by Ansible |
| **Caching** | Built-in caching support | You implement caching yourself |
| **Composability** | Can be layered with constructed plugin | Standalone |
| **Performance** | Better — uses Ansible's internal data structures | Slower — JSON parsing overhead |
| **Status** | Actively maintained | Legacy, not recommended for new use |

## Enabling Inventory Plugins

Inventory plugins must be enabled in `ansible.cfg` before they can be used.

```ini
# ansible.cfg
[inventory]
# Comma-separated list of enabled inventory plugins
# Order matters — Ansible tries each plugin in order
enable_plugins = amazon.aws.aws_ec2, azure.azcollection.azure_rm, constructed, yaml, ini
```

If you use a plugin from a collection, install the collection first:

```bash
# Install the AWS collection
ansible-galaxy collection install amazon.aws

# Install the Azure collection
ansible-galaxy collection install azure.azcollection

# Install the GCP collection
ansible-galaxy collection install google.cloud

# Install the VMware collection
ansible-galaxy collection install community.vmware
```

## Common Inventory Plugins

### amazon.aws.aws_ec2

The most widely used dynamic inventory plugin. The inventory file must end with `aws_ec2.yml` or `aws_ec2.yaml`.

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2

# Regions to query (omit to use all regions — slower)
regions:
  - us-east-1
  - us-west-2

# Filter which instances to include (uses AWS API filters)
filters:
  instance-state-name: running
  "tag:Environment":
    - production
    - staging

# Control how hostnames are generated (first match wins)
hostnames:
  - tag:Name                          # use the Name tag
  - private-ip-address                # fallback to private IP
  - dns-name                          # fallback to public DNS

# Automatically create groups from instance attributes
keyed_groups:
  # Group by the "Environment" tag → groups like "env_production", "env_staging"
  - key: tags.Environment
    prefix: env
    separator: "_"

  # Group by instance type → groups like "instance_type_t3_micro"
  - key: instance_type
    prefix: instance_type

  # Group by region → groups like "region_us_east_1"
  - key: placement.region
    prefix: region

  # Group by a tag, but only if the tag exists
  - key: tags.Role
    prefix: role
    default_value: untagged

# Set host variables from instance attributes
compose:
  ansible_host: private_ip_address
  ansible_user: "'ec2-user'"
  # Use single quotes inside double quotes to set a literal string
  ansible_ssh_private_key_file: "'~/.ssh/aws-key.pem'"
  # Dynamic variable based on instance attribute
  instance_name: tags.Name | default("unnamed")

# Use IAM role or set credentials
# (prefer environment variables or IAM roles over hardcoding)
# aws_access_key: "{{ lookup('env', 'AWS_ACCESS_KEY_ID') }}"
# aws_secret_key: "{{ lookup('env', 'AWS_SECRET_ACCESS_KEY') }}"

# Include only specific accounts (for cross-account inventory)
# iam_role_arn: arn:aws:iam::123456789012:role/ansible-inventory
```

### azure.azcollection.azure_rm

The inventory file must end with `azure_rm.yml` or `azure_rm.yaml`.

```yaml
# inventory/azure_rm.yml
plugin: azure.azcollection.azure_rm

# Include VMs from specific resource groups
include_vm_resource_groups:
  - production-rg
  - staging-rg

# Authentication (prefer environment variables or managed identity)
# auth_source: auto  # auto-detect: env vars → MSI → CLI → credential file

# Create groups based on conditions
conditional_groups:
  # All Linux VMs
  linux: "'linux' in os_profile.system"
  # All VMs in East US
  eastus: "location == 'eastus'"
  # All running VMs
  running: "powerstate == 'running'"

# Keyed groups
keyed_groups:
  - key: tags.Environment
    prefix: env
    separator: "_"
  - key: location
    prefix: location

# Set host variables
compose:
  ansible_host: public_ip_addresses[0] | default(private_ip_addresses[0], true)
  ansible_user: "'azureuser'"
```

### google.cloud.gcp_compute

The inventory file must end with `gcp.yml` or `gcp.yaml`.

```yaml
# inventory/gcp.yml
plugin: google.cloud.gcp_compute

# Projects to query
projects:
  - my-gcp-project

# Zones (omit to query all zones)
zones:
  - us-central1-a
  - us-east1-b

# Filter instances
filters:
  - status = RUNNING
  - "labels.environment = production"

# Authentication
auth_kind: serviceaccount
service_account_file: /path/to/service-account.json
# Or use application default credentials:
# auth_kind: application

keyed_groups:
  - key: labels.environment
    prefix: env
  - key: zone
    prefix: zone

compose:
  ansible_host: networkInterfaces[0].accessConfigs[0].natIP | default(networkInterfaces[0].networkIP)
```

### community.vmware.vmware_vm_inventory

```yaml
# inventory/vmware.yml
plugin: community.vmware.vmware_vm_inventory

hostname: vcenter.example.com
username: administrator@vsphere.local
password: "{{ lookup('env', 'VMWARE_PASSWORD') }}"
validate_certs: false

# Filter properties
properties:
  - name
  - guest.ipAddress
  - config.cpuHotAddEnabled
  - config.hardware.numCPU
  - runtime.powerState
  - config.guestId

# Only include powered on VMs
filters:
  - runtime.powerState == "poweredOn"

keyed_groups:
  - key: config.guestId
    prefix: os
```

## Constructed Inventory Plugin

The `constructed` plugin builds new groups and variables from existing inventory data. It runs after other inventory sources and can combine information from multiple plugins.

```yaml
# inventory/constructed.yml
plugin: constructed
strict: false    # don't fail on errors in expressions

# Create groups using Jinja2 expressions over existing host variables
groups:
  # Group all web servers regardless of cloud provider
  webservers: "'web' in group_names"

  # Group by operating system
  ubuntu_hosts: "ansible_distribution == 'Ubuntu'"

  # Combine region and environment
  us_production: "'region_us' in group_names and 'env_production' in group_names"

# Create keyed groups
keyed_groups:
  - key: ansible_distribution | default("unknown")
    prefix: os

# Set new variables based on existing ones
compose:
  # Standardize the connection user across providers
  ansible_user: >-
    (ansible_user is defined) | ternary(ansible_user,
      ('ubuntu' in group_names) | ternary('ubuntu', 'ec2-user'))
```

## Verifying Dynamic Inventory

Always verify your dynamic inventory before running playbooks.

```bash
# List all hosts and variables as JSON
ansible-inventory -i inventory/aws_ec2.yml --list

# Show inventory as a tree graph
ansible-inventory -i inventory/aws_ec2.yml --graph
# @all:
#   |--@env_production:
#   |  |--web-server-1
#   |  |--web-server-2
#   |--@env_staging:
#   |  |--staging-web-1
#   |--@ungrouped:

# Show variables for a specific host
ansible-inventory -i inventory/aws_ec2.yml --host web-server-1

# Show graph with variables
ansible-inventory -i inventory/aws_ec2.yml --graph --vars

# Use multiple inventory sources
ansible-inventory -i inventory/aws_ec2.yml -i inventory/static.yml --graph
```

## Runtime Inventory Manipulation

Sometimes you need to modify inventory during playbook execution — for instance, when you provision a new server and immediately configure it.

### add_host

```yaml
# Provision a server, then add it to the inventory for subsequent plays
- name: Provision and configure
  hosts: localhost
  tasks:
    - name: Create EC2 instance
      amazon.aws.ec2_instance:
        name: new-web-server
        instance_type: t3.micro
        image_id: ami-0abcdef1234567890
        wait: true
      register: ec2

    - name: Add new instance to inventory
      ansible.builtin.add_host:
        name: "{{ ec2.instances[0].public_ip_address }}"
        groups:
          - webservers
          - just_created
        ansible_user: ec2-user
        ansible_ssh_private_key_file: ~/.ssh/aws-key.pem

- name: Configure the new server
  hosts: just_created
  become: true
  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present
```

### group_by

```yaml
# Dynamically create groups based on facts gathered at runtime
- name: Group hosts by OS family
  hosts: all
  tasks:
    - name: Create OS-based groups
      ansible.builtin.group_by:
        key: "os_{{ ansible_facts['os_family'] | lower }}"

- name: Configure Debian-based hosts
  hosts: os_debian
  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true

- name: Configure RedHat-based hosts
  hosts: os_redhat
  tasks:
    - name: Update yum cache
      ansible.builtin.yum:
        update_cache: true
```

## Multiple Inventory Sources

You can point Ansible at a directory of inventory files. Ansible loads and merges all of them.

```
inventory/
  01-static.yml          # static hosts
  02-aws_ec2.yml         # AWS dynamic inventory
  03-azure_rm.yml        # Azure dynamic inventory
  04-constructed.yml     # constructed groups from all above
  group_vars/
    all.yml
    webservers.yml
  host_vars/
    specific-host.yml
```

```bash
# Use the entire directory as inventory
ansible-playbook -i inventory/ site.yml

# Or set it in ansible.cfg
# [defaults]
# inventory = ./inventory/
```

Files are processed in **alphabetical order**. The numerical prefixes (`01-`, `02-`, etc.) ensure the constructed plugin runs last, after cloud inventories have loaded.

## Caching

Querying cloud APIs on every Ansible run is slow. Enable caching to store inventory results locally.

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1

# Enable caching
cache: true
cache_plugin: ansible.builtin.jsonfile    # store cache as JSON files
cache_timeout: 3600                       # cache TTL in seconds (1 hour)
cache_connection: /tmp/ansible_inventory_cache  # directory for cache files
cache_prefix: aws_ec2                     # prefix for cache filenames
```

Other cache plugins include `ansible.builtin.redis`, `ansible.builtin.memcached`, and `community.general.yaml`.

```bash
# Force a cache refresh (ignore existing cache)
ansible-inventory -i inventory/aws_ec2.yml --list --flush-cache

# The next ansible-playbook run will also use the refreshed cache
```

## Migration Path: Static to Dynamic Inventory

Moving from static to dynamic inventory is best done incrementally.

```mermaid
flowchart LR
    A["Static inventory<br>(hosts.yml)"] --> B["Static + dynamic<br>side by side"]
    B --> C["Dynamic with<br>constructed groups"]
    C --> D["Fully dynamic<br>with caching"]
```

**Step 1 — Run both side by side.** Keep your static inventory. Add a dynamic inventory file next to it. Use `ansible-inventory --graph` to compare.

**Step 2 — Map static groups to keyed_groups.** If your static inventory has a `webservers` group, configure `keyed_groups` to create the same group from tags (e.g., `tag:Role = web` creates the group `role_web`).

**Step 3 — Update playbook `hosts:` lines.** Change playbooks to target the new group names, or use the constructed plugin to create groups with the original names.

**Step 4 — Enable caching.** Once you trust the dynamic inventory, enable caching to keep performance on par with static files.

**Step 5 — Remove the static file.** Once all playbooks use dynamic groups and you have verified correctness, remove the static inventory.
