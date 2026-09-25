# oci_network

## Purpose

Creates the OCI networking for a two-tier application: a public Load Balancer
subnet and an application subnet whose port is reachable only from the Load
Balancer. It manages:

* VCN
* Internet Gateway and NAT Gateway
* Public route table (0.0.0.0/0 → Internet Gateway) and private route table (0.0.0.0/0 → NAT Gateway)
* Security lists for the Load Balancer subnet and the application subnet
* Load Balancer subnet and application subnet

It does nothing else: no instances, no Load Balancer.

## Dependencies

* Collection `oracle.oci` (>= 5.5, < 6) and the `oci` Python SDK on the controller.
* A working `~/.oci/config` (see the root README).
* No role dependencies.

## Variables

| Variable | Default | Description |
|---|---|---|
| `oci_compartment_id` | `""` | **Required.** Compartment OCID (tenancy OCID for the root compartment) |
| `oci_admin_cidr` | `""` | **Required** when `oci_admin_ssh_enabled`. CIDR allowed to SSH to the VM. `0.0.0.0/0` is rejected |
| `oci_vcn_name` | `app-vcn` | VCN display name; also the prefix for every other resource name |
| `oci_vcn_cidr` | `10.20.0.0/16` | VCN CIDR |
| `oci_vcn_dns_label` | `appvcn` | VCN DNS label (alphanumeric, ≤ 15 chars, fixed once created) |
| `oci_public_subnet_cidr` | `10.20.10.0/24` | Load Balancer subnet |
| `oci_public_subnet_name` / `_dns_label` | `<vcn>-lb-subnet` / `lb` | |
| `oci_app_subnet_cidr` | `10.20.20.0/24` | Application subnet |
| `oci_app_subnet_name` / `_dns_label` | `<vcn>-app-subnet` / `app` | |
| `oci_app_subnet_public` | `true` | `true`: public IPs allowed, routed via the Internet Gateway. `false`: private, routed via the NAT Gateway |
| `oci_application_port` | `8000` | Application port opened from the LB subnet to the app subnet |
| `oci_lb_http_port` / `oci_lb_https_port` | `80` / `443` | Ports opened on the LB subnet |
| `oci_lb_ingress_cidr` | `0.0.0.0/0` | Who may reach the Load Balancer |
| `oci_admin_ssh_enabled` | `true` | Create the SSH rule for `oci_admin_cidr` |
| `oci_ssh_port` | `22` | |
| `oci_internet_gateway_name`, `oci_nat_gateway_name`, `oci_public_route_table_name`, `oci_private_route_table_name`, `oci_lb_security_list_name`, `oci_app_security_list_name` | derived from `oci_vcn_name` | Resource display names |
| `oci_freeform_tags` | `{managed_by: ansible}` | Tags applied to every resource |

## Inputs

Only the variables above. No OCIDs are hard-coded.

## Outputs

Set as facts on the host running the role (normally `localhost`):

| Fact | Description |
|---|---|
| `oci_vcn_id` | VCN OCID |
| `oci_public_subnet_id` | Load Balancer subnet OCID |
| `oci_app_subnet_id` | Application subnet OCID |
| `oci_internet_gateway_id` | Internet Gateway OCID |
| `oci_nat_gateway_id` | NAT Gateway OCID |
| `oci_public_route_table_id` | Public route table OCID |
| `oci_private_route_table_id` | Private route table OCID |
| `oci_lb_security_list_id` | Load Balancer security list OCID |
| `oci_app_security_list_id` | Application security list OCID |

## Security rules

| Security list | Direction | Rule |
|---|---|---|
| LB | ingress | TCP 443 and 80 from `oci_lb_ingress_cidr` |
| LB | egress | TCP `oci_application_port` to the app subnet only |
| App | ingress | TCP `oci_application_port` from the **LB subnet only** (never the Internet) |
| App | ingress | TCP 22 from `oci_admin_cidr` (when `oci_admin_ssh_enabled`) |
| App | ingress | ICMP 3/4 from anywhere (path MTU), ICMP 3 from the VCN |
| App | egress | All traffic (OS updates, container images) |

The VCN's default security list, which allows SSH from anywhere, is never
attached to these subnets.

## Idempotency

* VCN and gateways: matched by compartment + VCN + display name (`key_by`). A
  re-run returns them unchanged.
* Route tables, security lists and subnets: looked up by display name and
  updated in place. For example, a new `oci_admin_cidr` changes the existing
  security list instead of creating a new one.
* Fixed after creation: CIDRs, DNS labels, and a subnet's "prohibit public IP"
  flag (`oci_app_subnet_public`). Changing them requires recreating the subnet.

## Example usage

```yaml
- name: Network only
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    oci_compartment_id: "ocid1.compartment.oc1..aaaa..."
    oci_admin_cidr: "198.51.100.7/32"
  roles:
    - oci_network
```

## Example playbook

`playbooks/01_provision_oci.yml` runs this role before `oci_compute` and
`oci_load_balancer`, which consume its outputs.

## Teardown

`tasks/destroy.yml` deletes only the resources this role names, in dependency
order: subnets → route tables → security lists → NAT Gateway → Internet
Gateway → VCN. Delete the Load Balancer and instances first
(`playbooks/99_destroy_oci.yml` does this).

## OCI requirements

* IAM: `manage virtual-network-family` in the compartment.
* Service limits: one VCN, one NAT Gateway (check limits in trial and Always Free tenancies).

## Moving to OCI Bastion (no public IP)

The subnet's public-IP setting cannot be changed in place, so the move needs a
new subnet and instance:

1. `ansible-playbook playbooks/99_destroy_oci.yml -e confirm_destroy=true` (or delete only the instance and app subnet).
2. Set `oci_app_subnet_public: false`, which also sets `oci_compute_assign_public_ip: false` in group_vars.
3. Create an OCI Bastion that targets the app subnet. Set `oci_admin_cidr` to the CIDR the Bastion connects from (the app subnet CIDR works).
4. Set `oci_compute_ssh_common_args` to a ProxyCommand that goes through the Bastion session, then provision again.
