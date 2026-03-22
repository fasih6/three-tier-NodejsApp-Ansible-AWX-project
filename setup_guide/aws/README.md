# AWS Cloud Infrastructure Setup

This document covers the complete AWS infrastructure setup for the 3-tier Node.js application project. It explains every resource created, why it was created, and how it all connects together.

---

## Overview

The infrastructure uses a custom VPC with public and private subnets to properly separate internet-facing resources from backend services. Four EC2 instances are deployed — one for AWX (the Ansible orchestrator), one for the web server, and two for the backend application and database in the private subnet.

---
![vpc_architecture](../../pictures/vpc_architecture.png)
---
![aws_vpc_setup](../../pictures/aws_vpc_setup.png)
---

## Architecture

```
                        Internet
                            │
                     [Internet Gateway]
                            │
    ┌───────────────────────▼─────────────────────────┐
    │  VPC: vpc-ans  (10.0.0.0/16)                    │
    │                                                  │
    │  ┌────────────────────────────────────────────┐ │
    │  │  Public Subnet: ans-sub-public             │ │
    │  │  10.0.1.0/24                               │ │
    │  │                                            │ │
    │  │  ┌──────────────┐   ┌──────────────────┐  │ │
    │  │  │  awx-server  │   │   web-server     │  │ │
    │  │  │  t3.medium   │   │   t2.micro       │  │ │
    │  │  │  Public IP ✓ │   │   Public IP ✓    │  │ │
    │  │  └──────────────┘   └──────────────────┘  │ │
    │  │                                            │ │
    │  │  [NAT Gateway + Elastic IP]                │ │
    │  └────────────────────────────────────────────┘ │
    │                                                  │
    │  ┌────────────────────────────────────────────┐ │
    │  │  Private Subnet: ans-sub-private-1         │ │
    │  │  10.0.2.0/24                               │ │
    │  │                                            │ │
    │  │  ┌──────────────┐   ┌──────────────────┐  │ │
    │  │  │  app-server  │   │   db-server      │  │ │
    │  │  │  t2.micro    │   │   t2.micro       │  │ │
    │  │  │  No Public IP│   │   No Public IP   │  │ │
    │  │  └──────────────┘   └──────────────────┘  │ │
    │  └────────────────────────────────────────────┘ │
    └─────────────────────────────────────────────────┘
```

---

## Resources Created

### 1. VPC

| Field | Value |
|---|---|
| Name | vpc-ans |
| CIDR Block | 10.0.0.0/16 |
| DNS Hostnames | Enabled |
| DNS Resolution | Enabled |

**Why a custom VPC?** The default VPC puts everything in a flat network with public IPs. A custom VPC lets you control exactly which resources are reachable from the internet and which are not.

---

### 2. Subnets

**Public Subnet**

| Field | Value |
|---|---|
| Name | ans-sub-public |
| CIDR | 10.0.1.0/24 |
| Auto-assign Public IP | Enabled |
| Used for | AWX server, Web server |

**Private Subnet**

| Field | Value |
|---|---|
| Name | ans-sub-private-1 |
| CIDR | 10.0.2.0/24 |
| Auto-assign Public IP | Disabled |
| Used for | App server, DB server |

**Why public/private separation?** The web server needs to be reachable from the internet (port 80). The Node.js app and MySQL database should never be directly reachable from the internet — only from the web server and app server respectively. Private subnet instances have no public IP at all.

---

### 3. Internet Gateway

| Field | Value |
|---|---|
| Name | igw-ans |
| Attached to | vpc-ans |

The Internet Gateway is what gives public subnet instances two-way internet access. Without it, nothing in the VPC can reach the internet regardless of Security Group rules.

---

### 4. NAT Gateway

| Field | Value |
|---|---|
| Name | nat-ans |
| Subnet | ans-sub-public (must be public subnet) |
| Connectivity Type | Public |
| Elastic IP | Auto-allocated |

**Why NAT Gateway?** Private subnet instances have no public IP so they can't initiate outbound internet connections directly. The NAT Gateway allows them to reach the internet outbound (for `apt-get install`, `npm install`, cloning from GitHub) without being reachable inbound from the internet.

**Important:** The NAT Gateway must be placed in the **public subnet**, not the private subnet. If placed in the private subnet it cannot reach the Internet Gateway and all outbound traffic from private instances fails.

---

### 5. Route Tables

**Public Route Table**

| Destination | Target |
|---|---|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | igw-ans |

Associated with: `ans-sub-public`

This route table sends all internet-bound traffic (`0.0.0.0/0`) to the Internet Gateway, giving public subnet instances full internet access.

**Private Route Table**

| Destination | Target |
|---|---|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | nat-ans |

Associated with: `ans-sub-private-1`

This route table sends internet-bound traffic to the NAT Gateway. Private instances can reach the internet outbound but are not reachable inbound.

---

### 6. EC2 Instances

| Name | Type | Subnet | Public IP | Purpose |
|---|---|---|---|---|
| awx-server | t3.medium | Public | Yes | Runs k3s + AWX |
| web-server | t2.micro | Public | Yes | Nginx + React frontend |
| app-server | t2.micro | Private | No | Node.js backend API |
| db-server | t2.micro | Private | No | MySQL database |

**OS:** Ubuntu 22.04 LTS (all instances)
**Key Pair:** Same key pair for all 4 instances

**Why t3.medium for AWX?** AWX runs on Kubernetes (k3s) and requires at least 4GB RAM. The t2.micro (1GB) is too small — AWX pods crash with OOMKilled errors.

**Why the same key pair for all instances?** AWX SSHes into all 3 app instances to run Ansible playbooks. Using one key pair means one SSH credential to manage in AWX.

---

### 7. Security Groups

**AWX Server Security Group**

| Direction | Type | Port | Source | Purpose |
|---|---|---|---|---|
| Inbound | SSH | 22 | My IP | SSH access for management |
| Inbound | Custom TCP | 30080 | My IP | AWX web UI access |
| Outbound | All traffic | All | 0.0.0.0/0 | Unrestricted outbound |

**Web Server Security Group**

| Direction | Type | Port | Source | Purpose |
|---|---|---|---|---|
| Inbound | HTTP | 80 | 0.0.0.0/0 | Serve React app to users |
| Inbound | SSH | 22 | AWX Server IP | Ansible SSH from AWX |
| Outbound | All traffic | All | 0.0.0.0/0 | Unrestricted outbound |

**App Server Security Group**

| Direction | Type | Port | Source | Purpose |
|---|---|---|---|---|
| Inbound | Custom TCP | 3000 | Web Server IP | API calls from Nginx |
| Inbound | SSH | 22 | AWX Server IP | Ansible SSH from AWX |
| Outbound | All traffic | All | 0.0.0.0/0 | Unrestricted outbound |

**DB Server Security Group**

| Direction | Type | Port | Source | Purpose |
|---|---|---|---|---|
| Inbound | MySQL/Aurora | 3306 | App Server IP | DB connections from Node.js |
| Inbound | SSH | 22 | AWX Server IP | Ansible SSH from AWX |
| Outbound | All traffic | All | 0.0.0.0/0 | Unrestricted outbound |

**The security model:** Each tier only accepts traffic from the tier directly above it. The database cannot be reached from the internet or the web server — only from the app server. This is the principle of least privilege applied to network access.

---

## Setup Order

The order of creation matters. Some resources depend on others existing first.

```
Step 1 — Create VPC
         └── enables DNS hostnames + resolution

Step 2 — Create Subnets
         ├── Public subnet  (10.0.1.0/24)
         └── Private subnet (10.0.2.0/24)

Step 3 — Create Internet Gateway
         └── Attach to VPC immediately after creation

Step 4 — Create Public Route Table
         ├── Add route: 0.0.0.0/0 → Internet Gateway
         └── Associate with public subnet

Step 5 — Create NAT Gateway
         ├── Place in PUBLIC subnet (critical)
         ├── Allocate new Elastic IP
         └── Wait for status: Available (~2 min)

Step 6 — Create Private Route Table
         ├── Add route: 0.0.0.0/0 → NAT Gateway
         └── Associate with private subnet

Step 7 — Create Security Groups
         └── One per tier (AWX, web, app, db)

Step 8 — Launch EC2 Instances
         ├── awx-server  → public subnet
         ├── web-server  → public subnet
         ├── app-server  → private subnet
         └── db-server   → private subnet
```

---

## Common Issues and Fixes

**Private instances can't install packages (`apt-get update` times out)**

Cause: NAT Gateway is missing, placed in wrong subnet, or private route table is not associated with the private subnet.

Fix: Verify the NAT Gateway is in the public subnet and the private route table has `0.0.0.0/0 → nat-xxxxxxxx`.

**apt fails with IPv6 connection errors**

Cause: Ubuntu 22.04 tries IPv6 addresses first. Private instances don't have IPv6 routing.

Fix: Run on each private instance:
```bash
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4
```

**AWX can't SSH into target instances (`Permission denied (publickey)`)**

Cause: Username not set on AWX Machine Credential, or wrong SSH key stored in AWX.

Fix: Edit the AWX Machine Credential → set Username to `ubuntu` → paste the full `.pem` private key content into the SSH Private Key field.

**MySQL connection refused from app server (`ECONNREFUSED 10.0.2.x:3306`)**

Cause: MySQL binds to `127.0.0.1` by default on Ubuntu 22.04.

Fix: On the DB server:
```bash
sudo sed -i 's/bind-address.*=.*/bind-address = 0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo systemctl restart mysql
```

**VPC deletion fails**

Cause: EC2 instances still running, or NAT Gateway not deleted first.

Fix: Terminate all EC2 instances → delete NAT Gateway → release Elastic IP → then delete VPC.

---

## Teardown Order

Always delete in this order to avoid dependency errors:

```
Step 1 — Terminate all EC2 instances
Step 2 — Delete NAT Gateway (wait until deleted)
Step 3 — Release Elastic IP
Step 4 — Delete VPC (auto-deletes subnets, route tables, IGW, security groups)
```

---

## Cost Estimate

| Resource | Cost |
|---|---|
| t3.medium (AWX) | ~$0.0416/hr (~$30/month) |
| t2.micro × 3 (web/app/db) | Free tier eligible (first 12 months) |
| NAT Gateway | ~$0.045/hr + $0.045/GB (~$32/month) |
| Elastic IP (attached) | Free while attached |
| Internet Gateway | Free |

**Total estimate:** ~$60-65/month if running 24/7. For learning/demo purposes, stop instances when not in use to reduce cost. Delete the NAT Gateway when done — it charges by the hour even with no traffic.
