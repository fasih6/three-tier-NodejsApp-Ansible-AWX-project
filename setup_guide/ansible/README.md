# Ansible Folder Overview

This folder contains all Ansible code used to provision and configure the 3-tier application across environments.

---

## site.yml

The master playbook. Runs the 3 roles in the correct order — database first, then app, then web.

```yaml
---
- name: Provision database tier
  hosts: db
  become: yes
  roles:
    - mysql

- name: Provision app tier
  hosts: app
  become: yes
  roles:
    - nodejs

- name: Provision web tier
  hosts: web
  become: yes
  roles:
    - nginx
```

Order matters — the Node.js app needs MySQL running before it starts, and Nginx needs the app server IP to configure the reverse proxy.

---

## Inventories

Each environment has its own folder with two files:

**hosts.ini** — defines which servers belong to which group

```ini
[web]
10.0.1.124

[app]
10.0.2.234

[db]
10.0.2.132

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

**group_vars/all.yml** — defines environment-specific variables

```yaml
env: dev
node_port: 3000
node_env: development
app_repo: "https://github.com/fasih6/three-tier-NodejsApp-Ansible-AWX-project.git"
app_dir: /opt/app
db_name: crud_app
db_user: root
db_host: "{{ groups['db'][0] }}"
db_password: "{{ vault_db_password }}"
jwt_secret: "{{ vault_jwt_secret }}"
```

The same roles run against all 3 environments. Switching the inventory changes which servers get provisioned and which variable values are used — no changes to the playbook code.

---

## Roles

### nginx

Provisions the web tier. Installs Nginx, builds the React frontend, and serves it as static files with a reverse proxy to the Node.js backend.

| File | Purpose |
|---|---|
| `tasks/main.yml` | Installs Nginx, clones repo, runs `npm run build`, deploys config |
| `templates/nginx.conf.j2` | Nginx config — serves `client/build/`, proxies `/api/*` to app server |
| `handlers/main.yml` | Reloads Nginx when config changes |

Key template snippet — the app server IP is injected automatically from the inventory:
```nginx
location /api/ {
    proxy_pass http://{{ groups['app'][0] }}:{{ node_port }};
}
```

### nodejs

Provisions the app tier. Installs Node.js, clones the repository, installs dependencies, and runs the backend as a systemd service.

| File | Purpose |
|---|---|
| `tasks/main.yml` | Installs Node.js 18, clones repo, runs `npm install`, deploys service |
| `templates/app.service.j2` | systemd service file with environment variables injected from vault |
| `handlers/main.yml` | Restarts the service when the service file changes |

Key template snippet — secrets injected as environment variables, never stored in plaintext:
```ini
Environment=DB_HOST={{ db_host }}
Environment=DB_PASSWORD={{ vault_db_password }}
Environment=JWT_SECRET={{ vault_jwt_secret }}
```

### mysql

Provisions the DB tier. Installs MySQL, imports the database schema, and configures remote access.

| File | Purpose |
|---|---|
| `tasks/main.yml` | Installs MySQL, sets root password, imports schema, opens remote access |
| `templates/init.sql.j2` | Creates database and users table on first run |
| `handlers/main.yml` | Restarts MySQL when config changes |

---

## Vault

Secrets are stored encrypted in `group_vars/all/vault.yml` using Ansible Vault (AES-256). The file is safe to commit to Git.

```bash
# Create and encrypt
ansible-vault create ansible/group_vars/all/vault.yml

# Edit existing
ansible-vault edit ansible/group_vars/all/vault.yml

# View contents
ansible-vault view ansible/group_vars/all/vault.yml
```

Contents:
```yaml
vault_db_password: "..."
vault_mysql_root_password: "..."
vault_jwt_secret: "..."
```

These variables are referenced in role templates as `{{ vault_db_password }}` etc. AWX decrypts them automatically at runtime using the stored Vault Credential — the vault password never appears in any log or output.
