# OmniSwitch EVPN Fabric Automation using Ansible

## Overview

This repository provides an Ansible-based automation framework for deploying and managing an Alcatel-Lucent Enterprise OmniSwitch EVPN-VXLAN fabric.

The project uses centralized inventory files and reusable playbooks to automate the configuration of spine and leaf switches. It helps reduce manual configuration effort, improve consistency, minimize deployment errors, and simplify future fabric expansion.

The automation supports:

* OSPF underlay deployment
* BGP EVPN overlay deployment
* Loopback and routed interface configuration
* Route reflector configuration
* Multi-device configuration deployment
* Configuration backup
* Fabric verification
* Operational validation

---

## Fabric Topology

The EVPN fabric consists of two spine switches and six leaf switches.

```text
Spine switches:
SPINE-21    Management IP: 10.255.218.21
SPINE-22    Management IP: 10.255.218.22

Leaf switches:
LEAF-31     Management IP: 10.255.218.31
LEAF-32     Management IP: 10.255.218.32
LEAF-33     Management IP: 10.255.218.33
LEAF-34     Management IP: 10.255.218.34
LEAF-35     Management IP: 10.255.218.35
LEAF-36     Management IP: 10.255.218.36
```

The spine switches operate as BGP EVPN route reflectors. Each leaf switch establishes an internal BGP EVPN session with both spine switches using loopback interfaces.

---

## Network Design

### Underlay

The underlay network uses OSPF area `0.0.0.0`.

```text
Routing protocol: OSPF
Link type: Point-to-point
Addressing: /31 subnets
Failure detection: BFD
OSPF area: 0.0.0.0
```

Loopback addressing:

```text
SPINE-21    10.99.99.21
SPINE-22    10.99.99.22
LEAF-31     10.99.99.31
LEAF-32     10.99.99.32
LEAF-33     10.99.99.33
LEAF-34     10.99.99.34
LEAF-35     10.99.99.35
LEAF-36     10.99.99.36
```

### Overlay

The overlay uses internal BGP with the EVPN address family.

```text
BGP autonomous system: 65000
EVPN address family: Enabled
Route reflectors: SPINE-21 and SPINE-22
Route reflector cluster ID: 1.1.1.1
BFD: Enabled
Update source: Loopback0
```

---

## Repository Structure

```text
omniswitch-ansible/
├── inventory.yml
├── apply_full_config.yml
├── underlay.yml
├── overlay_bgp_evpn.yml
├── verify_fabric.yml
├── backup.yml
├── configs/
│   ├── spine-21.cfg
│   ├── spine-22.cfg
│   ├── leaf-31.cfg
│   ├── leaf-32.cfg
│   ├── leaf-33.cfg
│   ├── leaf-34.cfg
│   ├── leaf-35.cfg
│   └── leaf-36.cfg
└── README.md
```

---

## Requirements

The following components are required:

* Ubuntu Linux
* Python 3
* Ansible
* SSH connectivity to all OmniSwitch devices
* Valid OmniSwitch administrative credentials

The tested OmniSwitch software version is:

```text
AOS 8.10.86.R04
```

---

## Installing Ansible

On newer Ubuntu versions, the system Python environment is externally managed. Ansible should therefore be installed using a Python virtual environment or the Ubuntu package manager.

### Create a virtual environment

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip sshpass

python3 -m venv ~/ansible-env
source ~/ansible-env/bin/activate
```

Install Ansible:

```bash
pip install --upgrade pip
pip install ansible
```

Verify the installation:

```bash
ansible --version
```

---

## Inventory Configuration

The `inventory.yml` file contains all spine and leaf switches.

```yaml
all:
  children:

    spines:
      hosts:
        spine-21:
          ansible_host: 10.255.218.21

        spine-22:
          ansible_host: 10.255.218.22

    leafs:
      hosts:
        leaf-31:
          ansible_host: 10.255.218.31

        leaf-32:
          ansible_host: 10.255.218.32

        leaf-33:
          ansible_host: 10.255.218.33

        leaf-34:
          ansible_host: 10.255.218.34

        leaf-35:
          ansible_host: 10.255.218.35

        leaf-36:
          ansible_host: 10.255.218.36

    fabric:
      children:
        spines:
        leafs:

  vars:
    ansible_user: admin
    ansible_password: 'CHANGE_THIS_PASSWORD'
    ansible_connection: ssh
    ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
```

Do not store production passwords directly in the inventory. Use Ansible Vault for secure environments.

---

## Validate the Inventory

Run:

```bash
ansible-inventory -i inventory.yml --list
```

Test connectivity to all switches:

```bash
ansible fabric -i inventory.yml -m raw -a "show microcode working"
```

Test only spine switches:

```bash
ansible spines -i inventory.yml -m raw -a "show ip interface"
```

Test only leaf switches:

```bash
ansible leafs -i inventory.yml -m raw -a "show ip interface"
```

---

## Configuration Deployment

Each OmniSwitch has a separate configuration file under the `configs` directory.

```text
configs/spine-21.cfg
configs/spine-22.cfg
configs/leaf-31.cfg
configs/leaf-32.cfg
configs/leaf-33.cfg
configs/leaf-34.cfg
configs/leaf-35.cfg
configs/leaf-36.cfg
```

The configuration filename must match the hostname defined in `inventory.yml`.

Example deployment playbook:

```yaml
---
- name: Apply full OmniSwitch configuration
  hosts: fabric
  gather_facts: no

  tasks:

    - name: Push device configuration
      raw: "{{ lookup('file', 'configs/' + inventory_hostname + '.cfg') }}"

    - name: Save configuration
      raw: write memory
```

Run:

```bash
ansible-playbook -i inventory.yml apply_full_config.yml
```

---

## Underlay Deployment

The underlay playbook configures:

* Loopback interfaces
* Point-to-point routed links
* OSPF area configuration
* OSPF point-to-point network type
* BFD
* OSPF fast convergence timers

Run:

```bash
ansible-playbook -i inventory.yml underlay.yml
```

Verify OSPF:

```bash
ansible fabric -i inventory.yml -m raw -a "show ip ospf neighbor"
```

Verify routing:

```bash
ansible fabric -i inventory.yml -m raw -a "show ip route"
```

Verify interfaces:

```bash
ansible fabric -i inventory.yml -m raw -a "show ip interface"
```

---

## Overlay Deployment

The overlay playbook configures:

* BGP autonomous system 65000
* EVPN address family
* Spine route reflectors
* Route reflector clients
* Loopback-based BGP peering
* BFD for BGP neighbors
* EVPN neighbor activation

Run:

```bash
ansible-playbook -i inventory.yml overlay_bgp_evpn.yml
```

Verify BGP:

```bash
ansible fabric -i inventory.yml -m raw -a "show ip bgp summary"
```

Verify EVPN:

```bash
ansible fabric -i inventory.yml -m raw -a "show ip bgp evpn summary"
```

Verify BGP configuration:

```bash
ansible fabric -i inventory.yml -m raw -a "show configuration snapshot bgp"
```

---

## Fabric Verification

A verification playbook can be used to collect operational status from every switch.

Example:

```yaml
---
- name: Verify OmniSwitch EVPN fabric
  hosts: fabric
  gather_facts: no

  tasks:

    - name: Check OSPF neighbors
      raw: show ip ospf neighbor
      register: ospf_output

    - name: Check BGP neighbors
      raw: show ip bgp summary
      register: bgp_output

    - name: Check EVPN status
      raw: show ip bgp evpn summary
      register: evpn_output

    - name: Display OSPF output
      debug:
        var: ospf_output.stdout_lines

    - name: Display BGP output
      debug:
        var: bgp_output.stdout_lines

    - name: Display EVPN output
      debug:
        var: evpn_output.stdout_lines
```

Run:

```bash
ansible-playbook -i inventory.yml verify_fabric.yml
```

---

## Configuration Backup

Ansible can also collect and store the configuration from all OmniSwitch devices.

Example backup playbook:

```yaml
---
- name: Backup OmniSwitch configurations
  hosts: fabric
  gather_facts: no

  tasks:

    - name: Collect configuration snapshot
      raw: show configuration snapshot
      register: configuration

    - name: Save configuration locally
      delegate_to: localhost
      copy:
        content: "{{ configuration.stdout }}"
        dest: "backups/{{ inventory_hostname }}.cfg"
```

Create the backup directory:

```bash
mkdir -p backups
```

Run:

```bash
ansible-playbook -i inventory.yml backup.yml
```

---

## Security Recommendation

Use Ansible Vault to protect credentials.

Create an encrypted variable file:

```bash
ansible-vault create secrets.yml
```

Example encrypted variable:

```yaml
vault_ansible_password: "CHANGE_THIS_PASSWORD"
```

Reference it in the inventory or playbook:

```yaml
ansible_password: "{{ vault_ansible_password }}"
```

Run a playbook with the vault password:

```bash
ansible-playbook -i inventory.yml apply_full_config.yml --ask-vault-pass
```

---

## Recommended Deployment Workflow

The recommended deployment sequence is:

```text
1. Validate SSH connectivity
2. Back up existing configurations
3. Validate the Ansible inventory
4. Deploy loopback and routed interfaces
5. Deploy the OSPF underlay
6. Verify OSPF adjacencies
7. Verify loopback reachability
8. Deploy the BGP EVPN overlay
9. Verify BGP EVPN sessions
10. Deploy EVPN services and VNIs
11. Perform end-to-end service validation
12. Save the configuration
```

---

## Important Notes

* Always validate configuration changes in a lab before production deployment.
* Confirm that management connectivity will not be affected.
* Back up the existing configuration before running deployment playbooks.
* Use small host groups during initial testing.
* Use `--limit` to deploy to one switch at a time.
* Review the output before deploying to the full fabric.
* Configuration files should contain only valid OmniSwitch CLI commands.
* Avoid pushing configuration snapshot comments or unwanted output lines.

Example limited deployment:

```bash
ansible-playbook -i inventory.yml underlay.yml --limit leaf-31
```

Example syntax check:

```bash
ansible-playbook -i inventory.yml underlay.yml --syntax-check
```

Example verbose troubleshooting:

```bash
ansible-playbook -i inventory.yml underlay.yml -vvv
```

---

## Benefits of Ansible Automation

Ansible provides the following benefits for EVPN deployment:

* Centralized configuration management
* Consistent spine and leaf configurations
* Reduced manual configuration errors
* Faster fabric deployment
* Repeatable deployment workflows
* Simplified configuration backup
* Easier fabric expansion
* Automated operational verification
* Improved change control
* Integration with Git and CI/CD platforms

---

## Troubleshooting

### Inventory parsing failure

Validate the YAML format:

```bash
ansible-inventory -i inventory.yml --list
```

### Authentication failure

Test manual SSH:

```bash
ssh admin@10.255.218.21
```

### Ansible connects as root

Ensure that the inventory is parsed correctly and contains:

```yaml
ansible_user: admin
```

### Unsupported network OS error

For the current generic SSH workflow, use:

```yaml
ansible_connection: ssh
```

Run OmniSwitch commands using the Ansible `raw` module.

Example:

```bash
ansible spine-21 -i inventory.yml -m raw -a "show microcode working"
```

### Password contains special characters

Place the password inside quotes:

```yaml
ansible_password: 'PASSWORD_WITH_SPECIAL_CHARACTERS'
```

---

## Conclusion

Ansible provides a simple, agentless, and scalable method for automating OmniSwitch EVPN fabric deployments. By using a centralized inventory, reusable playbooks, and per-device configuration files, network administrators can deploy underlay and overlay configurations consistently across all spine and leaf switches.

The same framework can also be extended for EVPN service provisioning, configuration compliance, backup, validation, software upgrades, and ongoing fabric operations.


OmniSwitch EVPN Fabric Automation using Nornir

Python-based network automation framework for deploying, verifying, and managing an Alcatel-Lucent Enterprise (ALE) OmniSwitch EVPN-VXLAN fabric.

Overview

This repository provides a Nornir-based network automation framework for deploying and validating an Alcatel-Lucent Enterprise OmniSwitch EVPN-VXLAN fabric.

Nornir is a lightweight, Python-native automation framework that executes tasks in parallel, making it ideal for deploying large-scale spine-leaf data center fabrics. Combined with Netmiko, it enables secure SSH-based configuration deployment and operational verification across multiple OmniSwitch devices.

The automation framework was developed to eliminate repetitive manual configuration tasks, reduce deployment time, ensure configuration consistency, and simplify ongoing network operations.

This repository includes automation for:

OSPF Underlay Deployment
BGP EVPN Overlay Deployment
Configuration Deployment
Configuration Verification
Configuration Backup
Multi-device Parallel Execution
EVPN Operational Validation
Features
Python-based automation
Agentless architecture
SSH connectivity using Netmiko
Parallel deployment to multiple switches
OSPF Underlay automation
BGP EVPN Overlay automation
Configuration verification
Configuration backup
Easy inventory management
Scalable spine-leaf deployments
Tested Platform
Component	Version
Operating System	Ubuntu 26.xx
Python	3.x
Nornir	Latest
Netmiko	Latest
OmniSwitch	AOS 8.10.86.R04
EVPN Fabric Topology
Management Network
Device	Management IP	Role
SPINE-21	10.255.218.21	Spine / Route Reflector
SPINE-22	10.255.218.22	Spine / Route Reflector
LEAF-31	10.255.218.31	Leaf
LEAF-32	10.255.218.32	Leaf
LEAF-33	10.255.218.33	Leaf
LEAF-34	10.255.218.34	Leaf
LEAF-35	10.255.218.35	Leaf
LEAF-36	10.255.218.36	Leaf
Underlay Network
Parameter	Value
Routing Protocol	OSPF
Area	0.0.0.0
Link Type	Point-to-Point
Failure Detection	BFD
Addressing	/31

Loopback Addresses

Device	Loopback
SPINE-21	10.99.99.21
SPINE-22	10.99.99.22
LEAF-31	10.99.99.31
LEAF-32	10.99.99.32
LEAF-33	10.99.99.33
LEAF-34	10.99.99.34
LEAF-35	10.99.99.35
LEAF-36	10.99.99.36
Overlay Network
Parameter	Value
Protocol	iBGP
Address Family	EVPN
ASN	65000
Route Reflectors	SPINE-21, SPINE-22
BFD	Enabled
Update Source	Loopback0
Prerequisites

Before starting, ensure the following requirements are met:

Ubuntu 26.xx installed
Python 3.x
SSH connectivity to all OmniSwitch devices
Administrative access to the switches
Management IP connectivity
OmniSwitch AOS 8.10 or later
Installing Python

Update the package repository.

sudo apt update

Install Python and required packages.

sudo apt install python3 python3-pip python3-venv -y

Verify the installation.

python3 --version
Create a Python Virtual Environment

Ubuntu 26 protects the system Python environment. It is recommended to use a virtual environment for Nornir and other Python packages.

Create the virtual environment.

python3 -m venv ~/ansible-env

Activate the environment.

source ~/ansible-env/bin/activate

Verify.

which python

Expected output:

/root/ansible-env/bin/python
Upgrade pip
pip install --upgrade pip
Install Nornir

Install the required Python packages.

pip install nornir
pip install nornir-netmiko
pip install nornir-utils
pip install netmiko

Or install everything together:

pip install nornir nornir-netmiko nornir-utils netmiko

Verify the installation.

pip list

Expected packages:

nornir
nornir-netmiko
nornir-utils
netmiko
Project Directory Structure

Create the project directory.

mkdir ~/nornir-evpn

Change to the project directory.

cd ~/nornir-evpn

Create the required folders.

mkdir inventory
mkdir configs

Your project structure should look like this:

nornir-evpn/
│
├── config.yaml
├── deploy.py
├── verify.py
├── nornir.log
│
├── inventory/
│   ├── hosts.yaml
│   ├── groups.yaml
│   └── defaults.yaml
│
└── configs/
    ├── spine-21.cfg
    ├── spine-22.cfg
    ├── leaf-31.cfg
    ├── leaf-32.cfg
    ├── leaf-33.cfg
    ├── leaf-34.cfg
    ├── leaf-35.cfg
    └── leaf-36.cfg
Configuration Files
config.yaml

Create the main Nornir configuration file.

inventory:
  plugin: SimpleInventory
  options:
    host_file: "inventory/hosts.yaml"
    group_file: "inventory/groups.yaml"
    defaults_file: "inventory/defaults.yaml"

runner:
  plugin: threaded
  options:
    num_workers: 10
hosts.yaml

Create the device inventory.

spine-21:
  hostname: 10.255.218.21
  groups:
    - omniswitch

spine-22:
  hostname: 10.255.218.22
  groups:
    - omniswitch

leaf-31:
  hostname: 10.255.218.31
  groups:
    - omniswitch

leaf-32:
  hostname: 10.255.218.32
  groups:
    - omniswitch

leaf-33:
  hostname: 10.255.218.33
  groups:
    - omniswitch

leaf-34:
  hostname: 10.255.218.34
  groups:
    - omniswitch

leaf-35:
  hostname: 10.255.218.35
  groups:
    - omniswitch

leaf-36:
  hostname: 10.255.218.36
  groups:
    - omniswitch
groups.yaml

Store the common connection parameters.

omniswitch:
  username: admin
  password: "YOUR_PASSWORD"
  platform: alcatel_aos

  connection_options:
    netmiko:
      extras:
        device_type: alcatel_aos
        conn_timeout: 30
        auth_timeout: 30
        banner_timeout: 30
        global_delay_factor: 2
        fast_cli: false

Note: Replace YOUR_PASSWORD with the administrative password for your OmniSwitch devices. For production deployments, avoid storing passwords in plain text. Consider using environment variables or a secure secrets-management solution.

defaults.yaml

Create an empty defaults file.

touch inventory/defaults.yaml

This file can remain empty unless common variables need to be shared across all hosts.
# Deployment

The deployment process uses **Nornir** and **Netmiko** to establish SSH connections to each OmniSwitch defined in the inventory. Each device receives its corresponding configuration file from the `configs/` directory.

Configuration deployment is executed in parallel, significantly reducing the overall deployment time compared to manual CLI configuration.

---

# Configuration Files

Each OmniSwitch has its own configuration file.

```
configs/

spine-21.cfg
spine-22.cfg
leaf-31.cfg
leaf-32.cfg
leaf-33.cfg
leaf-34.cfg
leaf-35.cfg
leaf-36.cfg
```

The configuration filename **must exactly match** the hostname defined in `hosts.yaml`.

Example

```
hosts.yaml

spine-21:
```

Configuration file

```
configs/spine-21.cfg
```

---

# Deployment Script

Create `deploy.py`

```python
from pathlib import Path

from nornir import InitNornir
from nornir_netmiko.tasks import netmiko_send_config
from nornir_utils.plugins.functions import print_result


def deploy(task):

    config_file = Path("configs") / f"{task.host.name}.cfg"

    print(f"Deploying configuration to {task.host.name}")

    commands = config_file.read_text().splitlines()

    task.run(
        task=netmiko_send_config,
        commands=commands,
    )


nr = InitNornir(config_file="config.yaml")

results = nr.run(task=deploy)

print_result(results)
```

---

# Running Deployment

Activate the Python virtual environment.

```bash
source ~/ansible-env/bin/activate
```

Change to the project directory.

```bash
cd ~/nornir-evpn
```

Run the deployment.

```bash
python deploy.py
```

Nornir will

- Read the inventory
- Connect to every switch
- Load the correct configuration file
- Push the configuration
- Display the deployment results

---

# Configuration Verification

After deployment, verify the fabric to ensure all routing protocols and EVPN services are operational.

---

# Verification Script

Create `verify.py`

```python
from nornir import InitNornir
from nornir_netmiko.tasks import netmiko_send_command
from nornir_utils.plugins.functions import print_result


def verify(task):

    task.run(
        name="OSPF Neighbors",
        task=netmiko_send_command,
        command_string="show ip ospf neighbor"
    )

    task.run(
        name="BGP Neighbors",
        task=netmiko_send_command,
        command_string="show ip bgp neighbor"
    )

    task.run(
        name="EVPN Ethernet Segments",
        task=netmiko_send_command,
        command_string="show service evpn ethernet-segment"
    )

    task.run(
        name="Running Software",
        task=netmiko_send_command,
        command_string="show microcode working"
    )


nr = InitNornir(config_file="config.yaml")

results = nr.run(task=verify)

print_result(results)
```

---

# Running Verification

```bash
python verify.py
```

The script automatically verifies every switch defined in the inventory.

---

# Example Verification Output

```
**************************************************
* spine-21
**************************************************

show ip ospf neighbor

Neighbor ID      State
10.99.99.31      Full
10.99.99.32      Full
10.99.99.33      Full
10.99.99.34      Full
10.99.99.35      Full
10.99.99.36      Full

--------------------------------------------------

show ip bgp neighbor

Neighbor          State
10.99.99.31       Established
10.99.99.32       Established
10.99.99.33       Established
10.99.99.34       Established
10.99.99.35       Established
10.99.99.36       Established

--------------------------------------------------

show service evpn ethernet-segment

Operational State : Up
```

---

# Underlay Deployment

The underlay configuration includes

- Loopback interfaces
- Routed point-to-point interfaces
- OSPF Area 0
- OSPF Point-to-Point
- BFD
- Fast SPF timers

Deployment sequence

```
Loopback

↓

Point-to-Point Interfaces

↓

OSPF

↓

BFD

↓

Verify Neighbors
```

Verification commands

```
show ip ospf neighbor

show ip route

show ip interface
```

Expected result

- All OSPF neighbors should be in the **Full** state.
- All loopback addresses should be reachable.

---

# Overlay Deployment

The overlay configuration includes

- Internal BGP
- EVPN Address Family
- Route Reflectors
- Route Reflector Clients
- Loopback Peering
- BFD
- EVPN Activation

Deployment sequence

```
BGP

↓

EVPN Address Family

↓

Neighbors

↓

Route Reflectors

↓

Activate EVPN

↓

Verify Sessions
```

Verification commands

```
show ip bgp summary

show ip bgp neighbor

show ip bgp evpn summary

show configuration snapshot bgp
```

Expected result

- All BGP neighbors should be **Established**.
- EVPN sessions should be active on all spine and leaf switches.

---

# Configuration Backup

Nornir can also be used to collect configuration backups from every switch.

Example

```python
from nornir import InitNornir
from nornir_netmiko.tasks import netmiko_send_command
from nornir_utils.plugins.functions import print_result


def backup(task):

    result = task.run(
        task=netmiko_send_command,
        command_string="show configuration snapshot"
    )

    with open(f"backup/{task.host.name}.cfg","w") as f:
        f.write(result.result)


nr = InitNornir(config_file="config.yaml")

results = nr.run(task=backup)

print_result(results)
```

Run

```bash
python backup.py
```

Configuration files will be saved in

```
backup/

spine-21.cfg
spine-22.cfg
leaf-31.cfg
...
```

---

# Operational Validation

The following commands are recommended after deployment.

```
show ip ospf neighbor

show ip bgp neighbor

show ip bgp evpn summary

show service evpn ethernet-segment

show service evpn mac

show service evpn tunnel

show configuration snapshot bgp

show configuration snapshot ospf
```

Successful deployment should confirm

- OSPF neighbors are **Full**
- BGP neighbors are **Established**
- EVPN Ethernet Segments are **Up**
- VXLAN tunnels are operational
- Route Reflectors are advertising EVPN routes
- All leaf switches are reachable via the overlay

  # Troubleshooting

This section describes common issues encountered during deployment and their possible resolutions.

---

## Python Virtual Environment Not Activated

### Symptom

```
error: externally-managed-environment
```

### Cause

Ubuntu protects the system Python installation.

### Solution

Activate the virtual environment before installing or running Nornir.

```bash
source ~/ansible-env/bin/activate
```

Verify

```bash
which python
```

Expected

```
/root/ansible-env/bin/python
```

---

## SSH Authentication Failed

### Symptom

```
Authentication failed
```

### Possible Causes

- Incorrect username
- Incorrect password
- SSH disabled on the switch
- Network connectivity issue

### Verify

```bash
ssh admin@10.255.218.21
```

If SSH login fails, Nornir will also fail.

---

## Connection Timeout

### Symptom

```
Connection timed out
```

### Verify

```bash
ping 10.255.218.21
```

Verify SSH

```bash
telnet 10.255.218.21 22
```

or

```bash
nc -zv 10.255.218.21 22
```

---

## Netmiko Cannot Detect Device

### Symptom

```
ValueError
Unsupported device_type
```

### Solution

Verify the platform in

```
inventory/groups.yaml
```

Example

```yaml
omniswitch:
  username: admin
  password: "PASSWORD"

  connection_options:
    netmiko:
      extras:
        device_type: alcatel_aos
```

---

## Configuration File Missing

### Symptom

```
FileNotFoundError
```

### Verify

```
configs/
```

contains

```
spine-21.cfg
spine-22.cfg
leaf-31.cfg
leaf-32.cfg
leaf-33.cfg
leaf-34.cfg
leaf-35.cfg
leaf-36.cfg
```

The filename must match the hostname defined in `hosts.yaml`.

---

## Inventory Errors

Verify inventory.

```python
from nornir import InitNornir

nr = InitNornir(config_file="config.yaml")

print(nr.inventory.hosts)
```

---

## Verify Installed Packages

```bash
pip list
```

Expected

```
nornir
nornir-netmiko
nornir-utils
netmiko
```

---

# Logging

Deployment logs can be redirected to a file for troubleshooting.

Example

```bash
python deploy.py > deployment.log
```

Verification

```bash
python verify.py > verification.log
```

---

# Best Practices

The following best practices are recommended when automating OmniSwitch EVPN fabrics.

## Use Version Control

Maintain all configuration files in a Git repository.

Benefits

- Version history
- Rollback capability
- Collaboration
- Change tracking

---

## Test Before Production

Always validate

- Inventory
- SSH connectivity
- Configuration syntax

Deploy to a single switch before deploying to the entire fabric.

---

## Backup Before Deployment

Collect the running configuration before making changes.

Example command

```
show configuration snapshot
```

---

## Use Consistent Naming

Hostnames should match

```
hosts.yaml

↓

Configuration filename

↓

Device hostname
```

Example

```
leaf-31

↓

configs/leaf-31.cfg

↓

Hostname: LEAF-31
```

---

## Validate After Every Deployment

Recommended commands

```
show ip ospf neighbor

show ip bgp neighbor

show ip bgp evpn summary

show service evpn ethernet-segment

show service evpn tunnel

show service evpn mac
```

---

## Deploy in Stages

Recommended order

```
Loopback Interfaces

↓

Underlay

↓

Verify OSPF

↓

Overlay

↓

Verify BGP

↓

Deploy EVPN Services

↓

Verify End-to-End Connectivity
```

---

# Deployment Workflow

```
                    Inventory
                        │
                        ▼
                 Nornir Inventory
                        │
                        ▼
              Configuration Files
                        │
                        ▼
               Parallel SSH Sessions
                        │
                        ▼
            Configuration Deployment
                        │
                        ▼
               Save Configuration
                        │
                        ▼
             Operational Verification
                        │
                        ▼
                EVPN Fabric Ready
```

---

# Project Workflow

```
Install Ubuntu

↓

Install Python

↓

Create Virtual Environment

↓

Install Nornir

↓

Configure Inventory

↓

Copy Device Configurations

↓

Deploy Configuration

↓

Verify Fabric

↓

Production Deployment
```

---

# Future Enhancements

Future versions of this project may include

- Configuration compliance validation
- Automatic rollback
- Configuration drift detection
- Dynamic inventory support
- YAML-based configuration generation
- Jinja2 templates
- Configuration diff reporting
- Firmware upgrade automation
- Zero Touch Provisioning (ZTP)
- REST API integration
- GitHub Actions / CI/CD pipeline integration
- Automated EVPN service provisioning
- VXLAN VNI creation
- Tenant onboarding automation
- Health monitoring dashboards
- Scheduled configuration backup

---

# Contributing

Contributions are welcome.

If you find an issue or have suggestions for improvement:

- Open an Issue
- Submit a Pull Request
- Improve documentation
- Add automation examples
- Report bugs

Please test all changes in a lab environment before proposing them.

---

# References

- Nornir Documentation  
  https://nornir.readthedocs.io/

- Netmiko Documentation  
  https://ktbyers.github.io/netmiko/

- Alcatel-Lucent Enterprise OmniSwitch Documentation  
  https://www.al-enterprise.com/

---

# Disclaimer

This project is intended as a reference implementation for automating Alcatel-Lucent Enterprise OmniSwitch EVPN-VXLAN fabrics.

Always validate configurations in a non-production environment before deployment. The author assumes no responsibility for outages or configuration issues resulting from the use of this repository.

---

# License

This project is released under the MIT License.

You are free to use, modify, and distribute this project in accordance with the terms of the MIT License.

---

# Author

**Jadiyappa K**

Senior Solution Architect – Data Center & Enterprise Networking

Specialization:

- EVPN-VXLAN
- BGP
- OSPF
- Data Center Networking
- OmniSwitch Automation
- Network Automation (Python, Ansible, Nornir)
