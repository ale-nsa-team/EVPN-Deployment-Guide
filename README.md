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
