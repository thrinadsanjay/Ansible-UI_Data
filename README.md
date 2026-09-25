# Ansible_Data: Finance Tracker on OCI

Ansible project that provisions Finance Tracker's infrastructure on Oracle Cloud
Infrastructure (OCI), prepares the VM, and deploys the application. Each
concern lives in its own role. The playbooks only put the roles in order.

| Layer | Role | Playbook |
|---|---|---|
| Cloud provisioning | `oci_network`, `oci_compute`, `oci_load_balancer` | `01_provision_oci.yml` |
| OS configuration | `docker` | `02_configure_app_server.yml` |
| Application deployment | `fintracker_app` | `03_deploy_fintracker.yml` |
| Teardown | `destroy.yml` in each OCI role | `99_destroy_oci.yml` |

`site.yml` runs 01 → 02 → 03 in one go.

## Architecture

```text
                         Internet
                            │ HTTPS :443  (HTTP :80 → 301 to HTTPS)
                            ▼
      LB subnet 10.20.10.0/24 (public)
                  ┌─────────────────────┐
                  │ OCI Load Balancer   │  flexible 10–10 Mbps, public IP
                  │ TLS termination     │  certificate: oci_lb_certificate_ocid
                  │ Health check /health│
                  └──────────┬──────────┘
                             │ HTTP :8000 to the VM's PRIVATE IP
      App subnet 10.20.20.0/24
                  ┌──────────▼──────────┐
                  │ OCI Compute VM      │  VM.Standard.A1.Flex (ARM64)
                  │ 2 OCPU / 12 GB      │  Ubuntu, Docker, Compose
                  │ FinTracker :8000    │  MongoDB and mongo-express on loopback
                  └─────────────────────┘
      VCN 10.20.0.0/16 · Internet Gateway · NAT Gateway
```

Security rules:

* Port 8000 on the VM is open **only** to the Load Balancer subnet, never to the Internet.
* SSH (22) is open only to `oci_admin_cidr`, which must be a specific CIDR
  (`0.0.0.0/0` is rejected).
* The VCN's default security list (SSH from anywhere) is never attached.
* For the initial setup the VM has a public IP so Ansible can reach it. To
  switch to OCI Bastion and remove it, see "Moving to OCI Bastion" in
  [roles/oci_network/README.md](roles/oci_network/README.md).

## Project layout

```text
Ansible_Data/
├── ansible.cfg
├── requirements.yml              # Ansible collections
├── requirements.txt              # Python packages for the controller
├── inventory/
│   ├── hosts.yml                 # localhost + empty app_server group
│   └── group_vars/
│       ├── all.yml               # environment configuration (non-secret)
│       └── app_server/
│           ├── main.yml          # Finance Tracker settings
│           └── vault.yml.example # template for the encrypted vault.yml
├── playbooks/
│   ├── 01_provision_oci.yml
│   ├── 02_configure_app_server.yml
│   ├── 03_deploy_fintracker.yml
│   ├── 99_destroy_oci.yml
│   ├── discover_app_server.yml   # imported by 02/03
│   └── site.yml
└── roles/
    ├── oci_network/
    ├── oci_compute/
    ├── oci_load_balancer/
    ├── docker/
    └── fintracker_app/
```

`group_vars` lives next to the inventory (`inventory/group_vars/`) because
Ansible only loads `group_vars` from the inventory's or the playbook's
directory. A `group_vars/` folder at the project root would be ignored.

## Prerequisites

* Controller: Linux or macOS with Python ≥ 3.10, `ansible-core` ≥ 2.17, and the
  `oci` Python SDK in the **same** Python that runs Ansible.
* An OCI tenancy, a compartment, and a user with an API signing key.
* A certificate in the OCI Certificates service (same region) for the HTTPS listener.
* An SSH key pair for the VM (default `~/.ssh/id_ed25519`).

## OCI authentication

The `oracle.oci` modules use the standard OCI SDK configuration,
`~/.oci/config`. No credentials are stored in this project.

```ini
[DEFAULT]
user=ocid1.user.oc1..aaaa...
fingerprint=12:34:56:...
tenancy=ocid1.tenancy.oc1..aaaa...
region=ap-hyderabad-1
key_file=~/.oci/oci_api_key.pem
```

The OCI playbooks pass `oci_config_profile` (default `DEFAULT`) and
`oci_region` to the modules through the `OCI_CONFIG_PROFILE` and `OCI_REGION`
environment variables. `oci_region` takes precedence over the `region` in the
profile.

### API key setup

```sh
mkdir -p ~/.oci && chmod 700 ~/.oci
openssl genrsa -out ~/.oci/oci_api_key.pem 2048
chmod 600 ~/.oci/oci_api_key.pem
openssl rsa -pubout -in ~/.oci/oci_api_key.pem -out ~/.oci/oci_api_key_public.pem
```

In the OCI Console, open **Profile → My profile → API keys → Add API key**,
paste `oci_api_key_public.pem`, and copy the configuration snippet it shows into
`~/.oci/config`. (`oci setup config` from the OCI CLI does the same
interactively.) Check the setup with:

```sh
python3 -c "import oci; c=oci.config.from_file(); oci.config.validate_config(c); print(oci.identity.IdentityClient(c).get_user(c['user']).data.name)"
```

## Required IAM permissions

Put the automation user in a group, e.g. `FinTrackAutomation`, and grant:

```text
Allow group FinTrackAutomation to manage virtual-network-family in compartment <compartment>
Allow group FinTrackAutomation to manage instance-family        in compartment <compartment>
Allow group FinTrackAutomation to manage volume-family          in compartment <compartment>
Allow group FinTrackAutomation to manage load-balancers         in compartment <compartment>
Allow group FinTrackAutomation to use leaf-certificate-family   in compartment <compartment>
Allow group FinTrackAutomation to read app-catalog-listing      in tenancy
```

* `volume-family` covers the boot volume, which is deleted on destroy.
* `leaf-certificate-family` lets the listener use the certificate. Confirm the
  exact certificate policy in the OCI Certificates documentation for your tenancy.
* `app-catalog-listing` is only needed for Marketplace images. Platform Ubuntu
  images do not need it.

## Installation

```sh
cd /data/Ansible_Data
python3 -m venv ~/.venvs/ansible && . ~/.venvs/ansible/bin/activate
python3 -m pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

## Configuration

1. Edit `inventory/group_vars/all.yml` and replace every `REPLACE_ME`:

   | Variable | How to find it |
   |---|---|
   | `oci_compartment_id` | Console → Identity → Compartments |
   | `oci_availability_domain` | `oci iam availability-domain list --query 'data[].name'` |
   | `oci_compute_image_id` | Ubuntu **aarch64** image for your region (command in [roles/oci_compute/README.md](roles/oci_compute/README.md)) |
   | `oci_admin_cidr` | Your public IP as `/32` (`curl -s https://ifconfig.me`) |
   | `oci_lb_certificate_ocid` | Console → Identity & Security → Certificates |
   | `oci_compute_server_name` | Inventory name for the VM, e.g. `fintrack-prod-01` |

   Also check `oci_region`, `oci_ssh_public_key_file` and
   `oci_compute_ssh_private_key_file`.

2. Edit `inventory/group_vars/app_server/main.yml`: set `fintracker_base_url`
   (the HTTPS URL that matches the certificate) and `fintracker_default_admin_email`.

3. Create the vault with the application secrets:

   ```sh
   cp inventory/group_vars/app_server/vault.yml.example inventory/group_vars/app_server/vault.yml
   # edit: session secret (openssl rand -hex 32), admin and mongo-express passwords
   ansible-vault encrypt inventory/group_vars/app_server/vault.yml
   ```

   `vault.yml` is listed in `.gitignore`. Keep it out of Git even when encrypted.

The playbooks refuse to call the OCI API while any required value still holds
`REPLACE_ME`.

## Validation

```sh
yamllint .
ansible-lint
for p in playbooks/*.yml; do ansible-playbook --syntax-check "$p"; done
ansible-inventory --graph
```

## Provisioning

```sh
ansible-playbook playbooks/01_provision_oci.yml
```

The playbook prints the Load Balancer's public IP and the VM's addresses.
Create a DNS A record for the certificate's hostname that points to the Load
Balancer's IP.

A second run changes nothing. Existing resources are found by display name and
reused, not recreated.

## Application deployment

```sh
ansible-playbook playbooks/02_configure_app_server.yml
ansible-playbook playbooks/03_deploy_fintracker.yml --ask-vault-pass
```

Or everything at once, from an empty compartment to a running application:

```sh
ansible-playbook playbooks/site.yml --ask-vault-pass
```

02 and 03 find the VM through the OCI API (`discover_app_server.yml`), so you
can run them on their own at any time. To manage a VM without OCI discovery,
list it under `app_server` in `inventory/hosts.yml`.

Updates:

```sh
ansible-playbook playbooks/03_deploy_fintracker.yml --ask-vault-pass -e fintracker_pull_policy=always  # newest image
ansible-playbook playbooks/03_deploy_fintracker.yml --ask-vault-pass -e fintracker_image_version=1.0.6 # pin / roll back
```

Check the application at `https://<your-domain>/health`.

## Destroy procedure

```sh
ansible-playbook playbooks/99_destroy_oci.yml -e confirm_destroy=true
```

Without `confirm_destroy=true` the playbook stops before touching anything.
Deletion order: Load Balancer → Compute instance and boot volume → subnets →
route tables → security lists → NAT Gateway → Internet Gateway → VCN. Only
resources with this project's display names are deleted. Running it again
when everything is gone is safe.

**All data on the VM, including the MongoDB volume, is lost.** Take a backup
(Admin → Backups in Finance Tracker, or `mongodump`) first.

## Host key checking

`host_key_checking` stays enabled. The VM is added with
`-o StrictHostKeyChecking=accept-new`: the first connection records its key
and later connections fail if the key changes. OCI can reuse a public IP after
you destroy and recreate the VM. In that case remove the old key with
`ssh-keygen -R <ip>`.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Replace REPLACE_ME in inventory/group_vars/all.yml for: ...` | Set the listed variables. |
| `ConfigFileNotFound` / `InvalidConfig` / 401 `NotAuthenticated` | Check `~/.oci/config`, the key file path and permissions (600), and that the fingerprint matches the uploaded key. |
| 404 `NotAuthorizedOrNotFound` | IAM policy missing, or a wrong compartment or other OCID. |
| `No module named 'oci'` | The OCI SDK is not in the Python that runs Ansible. `localhost` uses `ansible_playbook_python`; install `oci` there. |
| `Out of host capacity` when creating the VM | A1 capacity is exhausted in that AD. Retry later or use another AD. |
| `LimitExceeded` | Service limit (VCNs, NAT Gateways, LBs, A1 OCPUs) reached. Request an increase or free resources. |
| SSH times out after provisioning | `oci_admin_cidr` does not match your current public IP. Update it and re-run 01 (the security list is updated in place). |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | The VM was recreated on a reused IP: `ssh-keygen -R <ip>`. |
| Load Balancer returns 502 / backend `CRITICAL` | On the VM: `curl -s http://127.0.0.1:8000/health` and `docker compose -f /opt/fintracker/docker-compose.yml ps`. OCI Ubuntu images ship restrictive iptables rules. Docker inserts its own FORWARD rules ahead of them, but check `sudo iptables -L FORWARD -n --line-numbers` if the port is unreachable from the LB subnet. |
| HTTPS certificate warning | The certificate does not cover the hostname, or DNS points elsewhere. |
| `Expected exactly one RUNNING instance` (02/03) | Provision with 01 first, or check `oci_compute_name`. |
| `Attempting to decrypt but no vault secrets found` | Add `--ask-vault-pass` (needed for `app_server` hosts). |
| Destroy fails deleting a subnet | Something outside this project (another VNIC, a Bastion) still uses the subnet. Remove it and re-run. |

## Assumptions

* The VM runs Ubuntu (22.04 or 24.04, aarch64). The default user is `ubuntu`.
* Finance Tracker is deployed from the FinTrack `main` branch deployment bundle
  (`docker-compose.yml`), and the `automationbuilder/fintracker` image supports
  arm64.
* One environment per compartment and display names. Resources are identified
  by display name, so keep names unique in the compartment.
* The certificate is managed in the OCI Certificates service, not uploaded to
  the Load Balancer.
