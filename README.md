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


# OmniSwitch EVPN Fabric Automation using Nornir

## Overview

This repository provides a Python-based network automation framework for deploying and managing an Alcatel-Lucent Enterprise (ALE) OmniSwitch EVPN-VXLAN fabric using **Nornir**.

The project automates the deployment of the EVPN fabric across multiple spine and leaf switches using SSH, reducing manual configuration effort while ensuring consistent and repeatable deployments.

The automation framework supports:

- Underlay OSPF deployment
- BGP EVPN overlay deployment
- Configuration backup
- Operational verification
- Bulk configuration changes
- Multi-device parallel execution

---

# Fabric Topology

```
                +----------------+
                |   SPINE-21     |
                | 10.99.99.21    |
                +----------------+
                    |    |    |
        -------------------------------
       /       /       |        \      \
      /       /        |         \      \

+-----------+  +-----------+  +-----------+
| LEAF-31   |  | LEAF-32   |  | LEAF-33   |
|10.99.99.31|  |10.99.99.32|  |10.99.99.33|
+-----------+  +-----------+  +-----------+

+-----------+  +-----------+  +-----------+
| LEAF-34   |  | LEAF-35   |  | LEAF-36   |
|10.99.99.34|  |10.99.99.35|  |10.99.99.36|
+-----------+  +-----------+  +-----------+

                +----------------+
                |   SPINE-22     |
                | 10.99.99.22    |
                +----------------+
```

---

# Features

- Agentless automation
- Parallel deployment
- SSH-based configuration
- Configuration verification
- Backup automation
- Multi-device execution
- Reusable playbooks
- Simple inventory management

---

# Repository Structure

```
.
├── config.yaml
├── deploy.py
├── verify.py
├── inventory
│   ├── hosts.yaml
│   ├── groups.yaml
│   └── defaults.yaml
│
├── configs
│   ├── spine-21.cfg
│   ├── spine-22.cfg
│   ├── leaf-31.cfg
│   ├── leaf-32.cfg
│   ├── leaf-33.cfg
│   ├── leaf-34.cfg
│   ├── leaf-35.cfg
│   └── leaf-36.cfg
│
└── README.md
```

---

# Requirements

- Ubuntu 22.04 / 24.04 / 26.xx
- Python 3.10+
- Nornir
- Netmiko
- SSH connectivity to all switches

---

# Installation

Create a virtual environment.

```bash
python3 -m venv ansible-env
source ansible-env/bin/activate
```

Install dependencies.

```bash
pip install nornir
pip install nornir-netmiko
pip install nornir-utils
pip install netmiko
```

---

# Inventory

Device inventory is maintained in

```
inventory/hosts.yaml
```

Example

```yaml
spine-21:
  hostname: 10.255.218.21

leaf-31:
  hostname: 10.255.218.31
```

Authentication credentials are stored in

```
inventory/groups.yaml
```

---

# Configuration Files

Each switch has its own configuration file.

```
configs/
```

Example

```
configs/spine-21.cfg
configs/spine-22.cfg
configs/leaf-31.cfg
```

The filename **must match** the inventory hostname.

---

# Deploy Configuration

Deploy the EVPN fabric.

```bash
python deploy.py
```

Nornir automatically

- Connects to all switches
- Pushes the corresponding configuration
- Executes in parallel
- Displays deployment results

---

# Verify Fabric

Run

```bash
python verify.py
```

Example verification commands

```
show ip ospf neighbor

show ip bgp summary

show ip bgp evpn summary

show configuration snapshot bgp
```

---

# Automation Workflow

```
Inventory

      │

      ▼

Configuration Files

      │

      ▼

Deploy.py

      │

      ▼

Parallel SSH Deployment

      │

      ▼

Fabric Verification
```

---

# Supported EVPN Components

The automation framework supports deployment of:

- OSPF Underlay
- Loopback Interfaces
- Point-to-Point Links
- BGP EVPN Overlay
- Route Reflectors
- VXLAN
- VTEP Interfaces
- VLANs
- VRFs
- Anycast Gateway
- Tenant Services

---

# Advantages of Nornir

- Python native automation
- Highly scalable
- Fast parallel execution
- Easily customizable
- Ideal for large EVPN fabrics
- Integrates with Netmiko, NAPALM, Scrapli, REST APIs and Python libraries

---

# Future Enhancements

- Configuration compliance checks
- Automated rollback
- Zero Touch Provisioning (ZTP)
- Firmware upgrade automation
- Configuration backup scheduling
- Health monitoring
- CI/CD integration
- Git version control
- Dynamic inventory
- Automated EVPN validation

---

# Tested Platform

Platform

```
Alcatel-Lucent Enterprise OmniSwitch
```

Software

```
AOS 8.10.86.R04
```

Automation Framework

```
Nornir 3.x
Netmiko
Python 3.x
```

---

# License

This project is intended as a reference implementation for automating OmniSwitch EVPN fabric deployments. Review and validate all configurations in a lab environment before deploying to production.
