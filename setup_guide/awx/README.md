# AWX Setup on AWS EC2

This document covers the complete AWX installation and configuration for the 3-tier Node.js application project. It explains why AWX was chosen, how it was installed on Kubernetes, and how every AWX object was configured.

---
![awx_setup](../../pics/pictures/awx_control_flow.png)
---
## AWX in simple terms

AWX = Web-based UI + API for Ansible
It does not replace Ansible — it just makes Ansible easier to manage and run.

---
## Why AWX?

When you first learn Ansible, you run playbooks from your local terminal like this:

```bash
ansible-playbook site.yml -i inventories/dev/hosts.ini --ask-vault-pass
```

This works fine for learning, but it has real problems in a team or production environment:

**No access control** — anyone with the playbook files and SSH key can run anything against any server. There is no way to say "this person can deploy to dev but not prod".

**No audit trail** — when something breaks at 2am, you have no record of who ran what playbook, against which inventory, at what time, and what the output was.

**Secrets are exposed** — the vault password, SSH keys, and other credentials live on developer laptops. If someone leaves the team or a laptop is lost, you have a security problem.

**No centralized execution** — playbooks run from whoever's machine happens to have Ansible installed. Different people might have different Ansible versions, different collection versions, different variable files.

**No UI for non-engineers** — a junior team member or developer who needs to trigger a deployment has to learn Ansible CLI, understand inventories, and handle vault passwords themselves.

AWX solves all of these problems:

- **Credentials are stored encrypted in AWX** — the SSH key and vault password never leave AWX. Team members can trigger jobs without ever seeing the secrets.
- **Role-based access control (RBAC)** — you can give a developer permission to run the deploy job against dev but not prod. You can give a manager read-only access to see job history.
- **Full audit trail** — every job run is logged with who launched it, when, which Git commit it used, and the complete Ansible output.
- **Centralized, consistent execution** — every job runs in the same container environment on the same machine. No more "it works on my machine" problems.
- **Self-service deployments** — with Surveys, a developer can trigger a deployment by filling out a form in the AWX UI — no Ansible knowledge required.
- **Scheduling** — jobs can run on a schedule (nightly patches, weekly compliance checks) without anyone manually triggering them.

In short, AWX is what turns Ansible from a personal automation tool into a team infrastructure platform. This is how real companies run Ansible at scale.

---

## AWX Architecture

AWX itself runs as a set of containers on Kubernetes. In this project, a lightweight Kubernetes distribution called **k3s** is used to run AWX on a single EC2 instance.

```
AWX Server (EC2 t3.medium)
│
└── k3s (lightweight Kubernetes)
    │
    └── awx namespace
        ├── awx-operator     (manages AWX lifecycle)
        ├── awx-postgres     (AWX internal database)
        ├── awx-web          (Django web UI + REST API)
        └── awx-task         (Ansible job execution engine)
```

When you launch a Job Template in AWX:
1. `awx-web` receives the request and queues the job
2. `awx-task` picks up the job and spawns an execution container
3. The execution container SSHes into your target EC2 instances
4. Ansible runs the playbook inside the container
5. Output streams back to `awx-web` in real time
6. Job result is stored in `awx-postgres`

---

## Installation

### Prerequisites

- EC2 instance: Ubuntu 22.04, t3.medium (4GB RAM minimum)
- Public IP assigned
- Port 30080 open in Security Group for your IP
- SSH access

### Step 1 — Install k3s

```bash
curl -sfL https://get.k3s.io | sh -

# Verify k3s is running
sudo kubectl get nodes
```

Expected output:
```
NAME            STATUS   ROLES                  AGE   VERSION
ip-10-0-1-185   Ready    control-plane,master   30s   v1.34.5+k3s1
```

### Step 2 — Create the AWX install directory

```bash
mkdir ~/awx-install && cd ~/awx-install
```

### Step 3 — Create the Kustomization file

The AWX Operator uses Kustomize for installation. The `kube-rbac-proxy` image from `gcr.io` is no longer available — it must be overridden with the working mirror on `quay.io`.

```bash
cat <<EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - github.com/ansible/awx-operator/config/default?ref=2.19.1
  - awx-instance.yaml
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.19.1
  - name: gcr.io/kubebuilder/kube-rbac-proxy
    newName: quay.io/brancz/kube-rbac-proxy
    newTag: v0.18.1
namespace: awx
EOF
```

### Step 4 — Create the AWX instance manifest

```bash
cat <<EOF > awx-instance.yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: nodeport
  nodeport_port: 30080
EOF
```

### Step 5 — Deploy AWX

```bash
sudo kubectl apply -k .
```

### Step 6 — Watch the pods come up

```bash
sudo kubectl get pods -n awx --watch
```

Wait until all pods show `Running`. This takes 10-15 minutes on first install as it pulls several large images from quay.io.

Expected final state:
```
NAME                                               READY   STATUS      AGE
awx-migration-24.6.1-xxxxx                         0/1     Completed   5m
awx-operator-controller-manager-xxxxxxxxx          2/2     Running     8m
awx-postgres-15-0                                  1/1     Running     7m
awx-task-xxxxxxxxx                                 4/4     Running     6m
awx-web-xxxxxxxxx                                  3/3     Running     6m
```

### Step 7 — Get the admin password

```bash
sudo kubectl get secret awx-admin-password -n awx \
  -o jsonpath="{.data.password}" | base64 --decode && echo
```

Copy this password — you will use it to log into AWX.

### Step 8 — Open AWX in your browser

```
http://<awx-server-public-ip>:30080
```

Login with:
- Username: `admin`
- Password: the output from Step 7

---

## Troubleshooting the Installation

### ImagePullBackOff on the operator pod

**Symptom:**
```
awx-operator-controller-manager   1/2   ImagePullBackOff
```

**Cause:** The `gcr.io/kubebuilder/kube-rbac-proxy` image no longer exists on Google Container Registry.

**Fix:** Use the Kustomize image override in the `kustomization.yaml` as shown above to redirect to `quay.io/brancz/kube-rbac-proxy:v0.18.1`.

### web and task pods never appear

**Symptom:** Only `awx-operator` and `awx-postgres` pods appear. `awx-task` and `awx-web` never start.

**Cause:** The AWX operator playbook is failing internally. Check the operator logs:

```bash
sudo kubectl logs -n awx deployment/awx-operator-controller-manager \
  -c awx-manager --tail=30
```

Look for `fatal:` lines in the Ansible output inside the logs.

### Pods keep restarting (OOMKilled)

**Symptom:** Pods show `RESTARTS` count increasing.

**Cause:** Not enough RAM. t3.medium (4GB) is the minimum. t2.micro (1GB) will not work.

**Fix:** Stop the instance, change instance type to t3.medium or larger, restart.

---

## AWX Configuration

After logging in, configure AWX in this exact order. Each object depends on the previous ones existing.

### 1. Organization

An Organization is the top-level container for all AWX resources. Everything — projects, inventories, credentials, job templates — belongs to an organization.

**Create:**
- Left sidebar → **Organizations** → **Add**

```
Name:         3tier-project
Description:  3-tier Node.js app provisioning project
```

---

### 2. Credentials

Credentials are stored encrypted in AWX. Team members can use credentials to run jobs without ever seeing the underlying secret values.

#### Machine Credential (SSH Key)

This credential is used by AWX to SSH into your EC2 instances.

- Left sidebar → **Credentials** → **Add**

```
Name:             EC2 SSH Key
Organization:     3tier-project
Credential Type:  Machine
Username:         ubuntu
SSH Private Key:  <paste full contents of your .pem file>
```

**Important:** The `Username` field must be set to `ubuntu` for AWS EC2 Ubuntu instances. Without this, AWX tries to SSH as the wrong user and gets `Permission denied`.

#### Vault Credential

This credential stores the Ansible Vault password used to decrypt `vault.yml`.

- **Credentials** → **Add**

```
Name:             3-tier-vault
Organization:     3tier-project
Credential Type:  Vault
Vault Password:   <your vault password>
Vault Identifier: (leave empty)
```

**Important:** The Vault Identifier must be empty (not `default`, not anything). If it has a value that doesn't match the vault file, AWX will prompt for the password interactively instead of passing it automatically.

---

### 3. Project

A Project links AWX to a Git repository. AWX syncs the repo before each job run so it always uses the latest playbooks.

- Left sidebar → **Projects** → **Add**

```
Name:                    3tier-nodejs-project
Organization:            3tier-project
Source Control Type:     Git
Source Control URL:      https://github.com/fasih6/three-tier-NodejsApp-Ansible-AWX-project.git
Source Control Branch:   main
Update Revision on Launch: ✓ (checked)
```

After saving, AWX immediately syncs the repo. Wait for the status indicator next to the project to turn green. If it turns red, click the project name and check the **Jobs** tab for the sync error.

---

### 4. Inventory

An Inventory defines the target hosts that Ansible will connect to. For this project, one inventory per environment (dev/staging/prod).

- Left sidebar → **Inventories** → **Add** → **Add Inventory**

```
Name:         3tier-dev-inventory
Organization: 3tier-project
Variables:
---
env: dev
node_port: 3000
node_env: development
app_repo: "https://github.com/fasih6/three-tier-NodejsApp-Ansible-AWX-project.git"
app_dir: /opt/app
db_name: crud_app
db_user: root
db_host: "10.0.2.132"
db_password: "root"
jwt_secret: "devopsShackSuperSecretKey"
```

**Why set variables on the inventory?** AWX does not automatically read `group_vars` files from the project repo the same way the Ansible CLI does. Setting variables directly on the AWX Inventory object ensures they are always available to every playbook run against that inventory.

#### Add Hosts

After saving the inventory, click the **Hosts** tab and add the 3 target servers:

| Hostname | Variables |
|---|---|
| `10.0.1.124` | `ansible_user: ubuntu` |
| `10.0.2.234` | `ansible_user: ubuntu` |
| `10.0.2.132` | `ansible_user: ubuntu` |

#### Add Groups

Click the **Groups** tab and create 3 groups:

| Group | Host to add |
|---|---|
| `web` | `10.0.1.124` |
| `app` | `10.0.2.234` |
| `db` | `10.0.2.132` |

These group names must match the `hosts:` values in `ansible/site.yml` exactly.

---

### 5. Job Template

A Job Template ties together the Project, Inventory, and Credentials into a runnable unit. This is what you click to deploy the application.

- Left sidebar → **Templates** → **Add** → **Add Job Template**

```
Name:             3tier-provision-dev
Job Type:         Run
Inventory:        3tier-dev-inventory
Project:          3tier-nodejs-project
Playbook:         ansible/site.yml
Credentials:      EC2 SSH Key  (Machine)
                  3-tier-vault  (Vault)
Privilege Escalation: ✓ Enabled (equivalent to become: yes)
```

**Important:** Both credentials must be attached — Machine for SSH access and Vault for secret decryption. If either is missing the job will fail.

---

## Running the First Job

- **Templates** → `3tier-provision-dev` → click the rocket icon 🚀

AWX will:
1. Sync the project from GitHub
2. Show a real-time output stream of every Ansible task
3. Store the complete job log when finished

A successful run ends with:
```
PLAY RECAP
10.0.1.124 : ok=12  changed=9   unreachable=0  failed=0
10.0.2.132 : ok=7   changed=3   unreachable=0  failed=0
10.0.2.234 : ok=8   changed=5   unreachable=0  failed=0
```

---

## AWX Objects Summary

| Object | Name | Purpose |
|---|---|---|
| Organization | 3tier-project | Container for all resources |
| Credential (Machine) | EC2 SSH Key | SSH into EC2 instances |
| Credential (Vault) | 3-tier-vault | Decrypt ansible-vault secrets |
| Project | 3tier-nodejs-project | Linked to GitHub repo |
| Inventory | 3tier-dev-inventory | Target hosts + environment variables |
| Job Template | 3tier-provision-dev | Runs `ansible/site.yml` |

---

## Common Issues After Setup

**Job fails with `Permission denied (publickey)`**

The SSH key or username is wrong in the Machine Credential. Verify the Username is `ubuntu` and the full `.pem` content is pasted in the SSH Private Key field.

**Job fails with `Vault password:`prompt in output**

The Vault Credential password doesn't match the password used to encrypt `vault.yml`. Re-encrypt the vault file with the correct password and update the AWX Vault Credential to match.

**Job fails with `couldn't resolve module/action 'mysql_user'`**

The `community.mysql` collection is not available in the AWX execution environment. Replace `mysql_user` tasks with `ansible.builtin.shell` commands that call MySQL directly — these work in any execution environment without additional collections.

**Job fails with `app_repo is undefined`**

AWX did not pick up the `group_vars` from the project repo. Set all required variables directly on the AWX Inventory object under the **Variables** field.

**AWX UI is not accessible on port 30080**

Check the EC2 Security Group for the AWX server — port 30080 must be open for your IP. Also verify the AWX pods are all running with `sudo kubectl get pods -n awx`.

---

## Useful kubectl Commands

```bash
# Check all AWX pods
sudo kubectl get pods -n awx

# View AWX operator logs (for installation issues)
sudo kubectl logs -n awx deployment/awx-operator-controller-manager \
  -c awx-manager --tail=50

# View AWX web logs
sudo kubectl logs -n awx deployment/awx-web -c awx-web --tail=50

# View AWX task logs
sudo kubectl logs -n awx deployment/awx-task -c awx-task --tail=50

# Get AWX admin password
sudo kubectl get secret awx-admin-password -n awx \
  -o jsonpath="{.data.password}" | base64 --decode && echo

# Restart AWX if something hangs
sudo kubectl rollout restart deployment/awx-web -n awx
sudo kubectl rollout restart deployment/awx-task -n awx

# Check resource usage
sudo kubectl top pods -n awx
```

---

## Teardown

To remove AWX completely:

```bash
cd ~/awx-install

# Delete AWX instance first
sudo kubectl delete awx awx -n awx

# Delete the operator and all resources
sudo kubectl delete -k .

# Delete the namespace
sudo kubectl delete namespace awx

# Optionally uninstall k3s entirely
/usr/local/bin/k3s-uninstall.sh
```
