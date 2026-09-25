# oci_compute

## Purpose

Provisions one OCI Compute instance in the application subnet created by
`oci_network`, publishes its addresses, and adds it to the in-memory inventory
group `app_server` with `ansible.builtin.add_host`.

It does not install Docker or deploy anything on the VM. Use the `docker` and
`fintracker_app` roles for that.

## Dependencies

* Collection `oracle.oci` (>= 5.5, < 6) and the `oci` Python SDK on the controller.
* `oci_app_subnet_id` from `oci_network` (run it earlier in the same play), or
  an explicit `oci_compute_subnet_id`.

## Variables

| Variable | Default | Description |
|---|---|---|
| `oci_compartment_id` | `""` | **Required.** Compartment OCID |
| `oci_availability_domain` | `""` | **Required.** e.g. `AbCd:AP-HYDERABAD-1-AD-1` |
| `oci_compute_image_id` | `""` | **Required.** Image OCID; an aarch64 image for A1 shapes |
| `oci_ssh_public_key` | `""` | **Required.** OpenSSH public key installed for the default user |
| `oci_compute_subnet_id` | `oci_app_subnet_id` | Subnet of the primary VNIC |
| `oci_compute_name` | `app-server` | Instance display name (how the role finds it again) |
| `oci_compute_hostname_label` | `oci_compute_name` | Hostname in the VCN DNS |
| `oci_compute_shape` | `VM.Standard.A1.Flex` | |
| `oci_compute_ocpus` | `2` | |
| `oci_compute_memory_gb` | `12` | |
| `oci_compute_boot_volume_gb` | `50` | Minimum 50 |
| `oci_compute_assign_public_ip` | `true` | Requires `oci_app_subnet_public: true` |
| `oci_freeform_tags` | `{managed_by: ansible}` | |

Inventory registration:

| Variable | Default | Description |
|---|---|---|
| `oci_compute_add_host` | `true` | Register the VM with `add_host` |
| `oci_compute_inventory_group` | `app_server` | Group the VM joins |
| `oci_compute_server_name` | `oci_compute_name` | Inventory hostname of the VM |
| `oci_compute_ssh_user` | `ubuntu` | `ansible_user` (`opc` on Oracle Linux) |
| `oci_compute_ssh_private_key_file` | `~/.ssh/id_ed25519` | `ansible_ssh_private_key_file` |
| `oci_compute_ssh_common_args` | `-o StrictHostKeyChecking=accept-new` | `ansible_ssh_common_args`; add a Bastion ProxyCommand here |
| `oci_compute_wait_for_ssh` | `true` | Wait until SSH answers before returning |
| `oci_compute_wait_for_ssh_timeout` | `600` | Seconds |

## Inputs

`oci_app_subnet_id` from `oci_network`.

## Outputs

| Fact | Description |
|---|---|
| `oci_compute_instance_id` | Instance OCID |
| `oci_compute_private_ip` | Private IP of the primary VNIC (the Load Balancer backend) |
| `oci_compute_public_ip` | Public IP, or `""` when none |

The VM also becomes an inventory host (`oci_compute_server_name`) in group
`app_server`. `ansible_host` is the public IP when there is one, otherwise the
private IP.

## Task files

| File | Use |
|---|---|
| `main.yml` | Validate, create the instance, register the host |
| `compute.yml` | The instance itself |
| `register_host.yml` | Read VNIC addresses, publish outputs, `add_host`, wait for SSH |
| `discover.yml` | Read-only: find the running instance by name and register it (used by `playbooks/discover_app_server.yml`) |
| `destroy.yml` | Terminate the instance and its boot volume |

## Idempotency

The instance is matched by compartment + availability domain + display name,
so a re-run returns it unchanged. Shape, OCPU, memory, image and boot volume
changes are **not** applied to an existing instance. To change them, replace
the instance (destroy it, then provision again).

## Example usage

```yaml
- name: Provision network and VM
  hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - oci_network
    - oci_compute

- name: Use the new VM
  hosts: app_server
  tasks:
    - name: Ping
      ansible.builtin.ping:
```

## Example playbook

See `playbooks/01_provision_oci.yml`.

## OCI requirements

* IAM: `manage instance-family` and `manage volume-family` in the compartment
  (plus `use virtual-network-family` to attach the VNIC).
* A1 capacity: `VM.Standard.A1.Flex` is often out of capacity in popular
  regions. On "Out of host capacity", retry later or try another AD.
* Find an Ubuntu aarch64 image OCID:

  ```sh
  oci compute image list --compartment-id <compartment-ocid> \
    --operating-system "Canonical Ubuntu" --operating-system-version "24.04" \
    --shape VM.Standard.A1.Flex --sort-by TIMECREATED --sort-order DESC \
    --query 'data[0].{id:id,name:"display-name"}'
  ```
