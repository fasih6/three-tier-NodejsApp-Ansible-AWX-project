# Project — Multi-Tier App Provisioning & Configuration

Automate provisioning of a 3-tier app (Nginx + App Server + MySQL) across multiple environments using AWX job templates and inventories.

`Roles` `Vault` `AWX Inventories` `Job Templates` `Jinja2`

---

## Phase 1 — AWS Setup

**1. Launch 4 EC2 instances** `~20 min`
Ubuntu 22.04 LTS. One for AWX (t3.medium, 4GB RAM). Three for your app tiers — web, app, db (t2.micro each, free tier eligible).

**2. Configure Security Groups** `~10 min`
AWX instance: open port 80, 443, 22. Web tier: 80, 443, 22. App tier: 3000, 22 (from web only). DB tier: 3306, 22 (from app only).

**3. Create one SSH key pair** `free`
Use the same key pair for all 4 instances. Download the .pem file — you'll add it to AWX as a Machine Credential later.

---

## Phase 2 — Install AWX on its EC2 instance

**4. Install k3s (lightweight Kubernetes)** `~5 min`
AWX runs on Kubernetes. k3s is the easiest single-node setup. One command installs it on your AWX EC2 instance.

**5. Install AWX Operator + deploy AWX** `~15 min`
AWX Operator manages AWX on k3s. Apply the operator manifest, then apply an AWX custom resource. AWX UI becomes available on port 30080 of that instance.

**6. Log in to AWX UI** `~2 min`
Default user is `admin`. Password is auto-generated — retrieve it from a Kubernetes secret. Then change it immediately.

---

## Phase 3 — Wire AWX to your project

**7. Add the ansible/ folder to your repo** `~1–2 hrs`
Create the roles, inventories, vault.yml, and site.yml as planned. Push to GitHub main branch.

**8. Create AWX Project, Credentials, Inventory** `~20 min`
Point AWX Project at your GitHub repo. Add SSH key as Machine Credential. Add vault password as Vault Credential. Create Inventory with EC2 IPs.

**9. Create Job Template + run it** `~10 min`
Create a Job Template pointing to site.yml. Launch it. Watch AWX provision your 3 EC2 instances — Nginx, Node.js, MySQL — in one run.
