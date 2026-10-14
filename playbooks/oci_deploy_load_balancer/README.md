# oci_deploy_load_balancer

**Description:** Create or destroy an OCI Load Balancer in front of one or more instances (generic, any application)

**Author:** Sanjay Ch

**Created:** 2026-10-12

---

## Overview

| Mode | What it does |
|------|--------------|
| `create` | Flexible LB in the VCN's LB subnet, backend set with the named instances, HTTP (and optional HTTPS) listeners |
| `destroy` | Deletes the LB with its listeners and backend sets |

Runs on `localhost`. Needs the VCN and LB subnet created by **oci_deploy_instance**.
Backends are found by instance display name; their private IPs are registered,
and IPs no longer listed are removed (re-run after replacing an instance).

```text
Internet ─ 80 (HTTP) / 443 (HTTPS, optional) ─▶ LB ─ oci_lb_backend_port ─▶ instance(s)
```

---

## Files

- **main.yml** - Main playbook; runs create/destroy
- **vars.yml** - UI inputs and defaults
- **internal_vars.yml** - Derived values (instance list, subnet name, tags)
- **controller_vars.yml** - Python interpreter for the OCI API host
- **ui_schema.yml** - UI form schema
- **checks.yml** / **lookup.yml** / **create.yml** / **destroy.yml** - Workflows
- **inventory.ini** - localhost only
- **roles** - Symlink to the project `roles/` (oci_load_balancer)

---

## Required Variables

| Variable | Type | Description | Required for |
|----------|------|-------------|--------------|
| target_hosts | text | `localhost` | all |
| oci_compartment_id | text | Compartment OCID | all |
| oci_lb_name | text | LB name | all |
| oci_vcn_name | text | VCN from oci_deploy_instance | create |
| oci_lb_backend_instances | text | Instance names, comma separated | create |
| oci_lb_backend_port | number | App port on the instances (must be in their Application Ports) | create |
| oci_lb_certificate_ocid | text | OCI Certificates OCID | create with HTTPS |
| confirm_destroy | checkbox | Must be ticked | destroy |

## Optional Variables

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| oci_lb_https_enabled | checkbox | HTTPS on 443 with the certificate | false |
| oci_lb_http_redirect_to_https | checkbox | With HTTPS: HTTP 80 → 301 to HTTPS | true |
| oci_lb_health_check_protocol | select | HTTP / TCP | HTTP |
| oci_lb_health_check_path | text | HTTP health check path | / |
| oci_lb_health_check_return_code | number | Expected HTTP status | 200 |
| oci_lb_backend_policy | select | ROUND_ROBIN / LEAST_CONNECTIONS / IP_HASH | ROUND_ROBIN |
| oci_lb_min/max_bandwidth_mbps | number | Flexible shape bandwidth | 10 / 10 |
| oci_lb_subnet_name | text | LB subnet; empty = `<vcn>-lb-subnet` | "" |
| oci_lb_http_port / oci_lb_https_port | number | Listener ports (match oci_deploy_instance) | 80 / 443 |

---

## Usage

### Via UI

1. Select `localhost` as the target
2. Fill in Compartment, LB Name, VCN Name, Backend Instances, Backend Port
3. Optional: tick Enable HTTPS and set the Certificate OCID
4. Execute; the output shows the LB public IP

### Via Command Line

```bash
cd playbooks/oci_deploy_load_balancer
ansible-playbook main.yml -i inventory.ini -e target_hosts=localhost \
  -e oci_compartment_id=ocid1.compartment.oc1..xxxx \
  -e oci_vcn_name=app-vcn -e oci_lb_backend_instances=web01,web02 -e oci_lb_backend_port=8000
```

---

## Notes

- The LB subnet's security list (from oci_deploy_instance) opens only its LB HTTP/HTTPS Ports.
- Creating a Load Balancer takes several minutes.

---

**Last Updated:** 2026-10-12
