# deploy_fintracker

**Description:** Install Docker and deploy or update the Finance Tracker application with Docker Compose

**Author:** Sanjay Ch

**Created:** 2026-10-12

---

## Overview

Runs on the hosts picked in the Ansible UI host picker (servers you added to the
inventory, for example a VM created by **oci_deploy_instance**):

1. Validates the settings and the OS (Ubuntu/Debian, Oracle Linux/RHEL 8+, Python 3.8+)
2. Installs Docker Engine + Compose (role `docker`), unless Install Docker is unticked
3. Checks out the deployment bundle to `/opt/fintracker`, writes `.env`, starts the stack (role `fintracker_app`)
4. Opens the app port in firewalld when firewalld is running (Oracle Linux)

Re-run to update: Image Version `1.0.6` pins/rolls back; Image Pull Policy `always` fetches a new `latest`.

---

## Files

- **main.yml** - Main playbook
- **vars.yml** - UI inputs and defaults
- **internal_vars.yml** - Derived values (docker group users)
- **ui_schema.yml** - UI form schema
- **checks.yml** / **deploy.yml** - Workflows
- **roles** - Symlink to the project `roles/` (docker, fintracker_app)

No `inventory.ini`: the UI uses the master inventory, so the host picker lists every host.

---

## Required Variables

| Variable | Type | Description |
|----------|------|-------------|
| target_hosts | text | From the host picker |
| fintracker_base_url | text | `https://fin.example.com` (behind HTTPS LB) or `http://<ip>:8000` |
| fintracker_default_admin_email | text | Admin login email |
| fintracker_session_secret | password | 32+ characters (`openssl rand -hex 32`) |
| fintracker_default_admin_password | password | 12+ characters |
| fintracker_mongo_express_password | password | 12+ characters |

## Optional Variables

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| fintracker_app_port | number | Host port (= LB backend port) | 8000 |
| fintracker_image_version | text | Image tag | latest |
| fintracker_pull_policy | select | missing / always | missing |
| fintracker_default_admin_username | text | Admin user | admin |
| fintracker_install_dir | text | Install path | /opt/fintracker |
| install_docker | checkbox | Install Docker first | true |
| open_firewall_port | checkbox | Open the port in firewalld | true |

---

## Usage

### Via UI

1. Pick the server in the host picker
2. Fill in URL, admin email and the three secrets
3. Execute; the run ends with a health check on `http://127.0.0.1:<port>/health`

### Via Command Line

```bash
ansible-playbook playbooks/deploy_fintracker/main.yml -i inventory/inventory.ini \
  -e target_hosts=web01 -e @secrets.yml --ask-vault-pass
```

---

**Last Updated:** 2026-10-12
