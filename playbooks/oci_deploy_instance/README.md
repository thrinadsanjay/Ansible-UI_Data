# oci_deploy_instance

**Description:** Create or destroy an OCI Compute instance and its network (generic, any application)

**Author:** Sanjay Ch

**Created:** 2026-10-12

---

## Overview

| Mode | What it does |
|------|--------------|
| `create` | VCN, Internet + NAT gateways, LB subnet, app subnet, security lists, Compute instance |
| `destroy` | Terminates the instance and its boot volume (optionally the network) |

Runs on `localhost` (the OCI API host). Re-runs are idempotent: resources are
found by display name and reused. Several instances can share one VCN.

```text
VCN (oci_vcn_cidr)
├── LB subnet  (oci_public_subnet_cidr, public)  ← used by oci_deploy_load_balancer; 80/443 open
└── App subnet (oci_app_subnet_cidr)             ← instances
      ingress: SSH from oci_admin_cidr
               oci_app_ports from the LB subnet (+ oci_app_ingress_cidr when set)
```

### Playbook chain

1. **oci_deploy_instance** → VM (public/private IP in the output)
2. **oci_deploy_load_balancer** → LB in front of the VM(s)
3. Add the VM to the Ansible UI inventory, then run **deploy_fintracker** / **manage_instance** on it

---

## Files

- **main.yml** - Main playbook; runs create/destroy
- **vars.yml** - UI inputs and defaults (scanned by the Schema Builder)
- **internal_vars.yml** - Derived values (tags, port list, SSH key); not UI inputs
- **controller_vars.yml** - Python interpreter for the OCI API host
- **ui_schema.yml** - UI form schema
- **checks.yml** / **create.yml** / **destroy.yml** - Workflows
- **inventory.ini** - localhost only
- **roles** - Symlink to the project `roles/` (oci_network, oci_compute)

---

## Required Variables

| Variable | Type | Description | Required for |
|----------|------|-------------|--------------|
| target_hosts | text | `localhost` | all |
| operation_mode | select | create / destroy | all |
| oci_compartment_id | text | Compartment OCID | all |
| oci_compute_name | text | Instance name (also hostname and inventory name) | all |
| oci_availability_domain | text | e.g. `AbCd:AP-HYDERABAD-1-AD-1` | create |
| oci_compute_image_id | text | Image OCID matching the shape's CPU architecture | create |
| oci_admin_cidr | text | CIDR allowed to SSH | create |
| confirm_destroy | checkbox | Must be ticked | destroy |

## Optional Variables

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| oci_compute_shape / ocpus / memory_gb / boot_volume_gb | | Instance size | A1.Flex / 1 / 6 / 50 |
| oci_app_ports | text | Comma-separated app ports | 8000 |
| oci_app_ingress_cidr | text | Also open app ports to this CIDR (no LB) | "" |
| oci_ssh_public_key_content | textarea | Public key; empty = `oci_ssh_public_key_file` | "" |
| oci_vcn_name / dns_label / cidrs | text | Network | app-vcn / appvcn / 10.20.0.0/16 |
| oci_app_subnet_public | checkbox | VM gets a public IP | true |
| oci_admin_ssh_allow_any | checkbox | Allow `oci_admin_cidr` 0.0.0.0/0 | false |
| destroy_network | checkbox | With destroy: delete the VCN too | false |

### Network and VNIC names (advanced)

Every name left empty is derived from `oci_vcn_name` / `oci_compute_name`, so
re-runs keep finding the same resources. Set a name to use your own; keep it the
same on later runs (and for destroy), because resources are found by name.

| Variable | Default when empty |
|----------|--------------------|
| oci_public_subnet_name / oci_public_subnet_dns_label | `<vcn>-lb-subnet` / `lb` |
| oci_app_subnet_name / oci_app_subnet_dns_label | `<vcn>-app-subnet` / `app` |
| oci_internet_gateway_name | `<vcn>-igw` |
| oci_nat_gateway_name | `<vcn>-natgw` |
| oci_public_route_table_name / oci_private_route_table_name | `<vcn>-public-rt` / `<vcn>-private-rt` |
| oci_lb_security_list_name / oci_app_security_list_name | `<vcn>-lb-sl` / `<vcn>-app-sl` |
| oci_compute_vnic_name | `<instance>-vnic` |
| oci_compute_hostname_label | instance name, lowercase |
| oci_compute_private_ip_address | assigned by OCI |

| Variable | Description | Default |
|----------|-------------|---------|
| oci_compute_assign_public_ip | Public IP on the VNIC (needs a public app subnet) | true |
| oci_lb_http_port / oci_lb_https_port | Ports opened on the LB subnet (match oci_deploy_load_balancer) | 80 / 443 |
| oci_lb_ingress_cidr | Who may reach the LB ports | 0.0.0.0/0 |
| oci_admin_ssh_enabled / oci_ssh_port | SSH rule on the app subnet | true / 22 |

Subnet CIDRs, DNS labels and "public subnet" cannot change on an existing subnet;
renaming a resource creates a new one under the new name.

---

## Usage

### Via UI

1. Select `localhost` as the target
2. Fill in Compartment, Instance Name, Availability Domain, Image, Admin SSH CIDR
3. Execute; the output shows the instance's IPs
4. Add the VM to the Ansible UI inventory (SSH user `opc` on Oracle Linux, `ubuntu` on Ubuntu)

### Via Command Line

```bash
cd playbooks/oci_deploy_instance
ansible-playbook main.yml -i inventory.ini -e target_hosts=localhost \
  -e oci_compartment_id=ocid1.compartment.oc1..xxxx \
  -e oci_availability_domain="AbCd:AP-HYDERABAD-1-AD-1" \
  -e oci_compute_image_id=ocid1.image.oc1.ap-hyderabad-1.xxxx \
  -e oci_admin_cidr=198.51.100.7/32 -e oci_compute_name=web01
```

---

## Notes

- The UI runs playbooks without the project ansible.cfg, so SSH host key checking is on. Accept the
  new VM's key once (e.g. `ssh-keyscan <ip> >> ~/.ssh/known_hosts` in the UI container), or add
  `ansible_ssh_extra_args=-oStrictHostKeyChecking=accept-new` to its inventory line.
- Oracle Linux 7 cannot be managed by ansible-core 2.17+ (needs Python 3.8+ on the target). Use Oracle Linux 8/9 or Ubuntu.
- Shape, image and size changes are not applied to an existing instance; destroy and create again.

---

**Last Updated:** 2026-10-12
