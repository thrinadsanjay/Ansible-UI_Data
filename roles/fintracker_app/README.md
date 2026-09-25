# fintracker_app

## Purpose

Deploys Finance Tracker on a Docker host. It checks out the FinTrack `main`
branch (the deployment bundle: `docker-compose.yml`, `env.example`, README),
writes `.env` from Ansible variables and Vault secrets, and starts the stack
(FinTracker, MongoDB and mongo-express) with `community.docker.docker_compose_v2`.

No image is built on the host. The FinTracker image is pulled from Docker Hub
(`automationbuilder/fintracker`).

## Dependencies

* Docker Engine and the Compose plugin on the host (use the `docker` role first).
* Collection `community.docker` (>= 5.0).
* Outbound HTTPS from the host to GitHub and Docker Hub.

## Variables

| Variable | Default | Description |
|---|---|---|
| `fintracker_repo_url` | FinTrack GitHub URL | Repository holding the deployment bundle |
| `fintracker_repo_version` | `main` | Branch or tag to check out |
| `fintracker_install_dir` | `/opt/fintracker` | Checkout and Compose project directory |
| `fintracker_image_version` | `latest` | FinTracker image tag (`FINTRACKER_VERSION`) |
| `fintracker_mongo_image_tag` | `8.0` | MongoDB image tag (pin the major version) |
| `fintracker_pull_policy` | `missing` | `docker_compose_v2` `pull`: `missing`, `always`, `never`, `policy` |
| `fintracker_app_port` | `8000` | Host port for the app (`FT_PUBLIC_PORT`); must match `oci_lb_backend_port` |
| `fintracker_base_url` | `""` | **Required.** Public URL, e.g. `https://fin.example.com` |
| `fintracker_default_admin_username` | `admin` | First admin account |
| `fintracker_default_admin_email` | `""` | **Required.** First admin e-mail |
| `fintracker_extra_env` | `{}` | Extra `.env` entries (any key from `env.example`) |
| `fintracker_session_secret` | `""` | **Required, secret.** 32+ characters |
| `fintracker_default_admin_password` | `""` | **Required, secret.** 12+ characters |
| `fintracker_mongo_express_password` | `""` | **Required, secret.** 12+ characters |
| `fintracker_health_check_retries` / `_delay` | `30` / `10` | Wait for `GET /health` after start |

## Inputs

Secrets come from Ansible Vault. In this project,
`inventory/group_vars/app_server/main.yml` maps `vault_fintracker_*` from the
encrypted `vault.yml` to the variables above.

## Outputs

* `fintracker_health`: the `/health` response (includes the running version).

## Example playbook

```yaml
- name: Deploy Finance Tracker
  hosts: app_server
  become: true
  roles:
    - fintracker_app
```

## Operations

* Update to the newest image: run the deploy playbook with `-e fintracker_pull_policy=always`.
* Pin or roll back a release: `-e fintracker_image_version=1.0.6`.
* Idempotency: a re-run with unchanged settings reports no changes. A changed
  `.env` recreates only the affected containers.
* Data lives in named Docker volumes (`fin_tracker_mongodb_data`, ...). This
  role never removes volumes.
