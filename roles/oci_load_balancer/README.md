# oci_load_balancer

## Purpose

Creates a public, flexible-shape OCI Load Balancer that terminates TLS and
forwards plain HTTP to the application VM's **private IP**:

```text
Client ──HTTPS :443──▶ OCI Load Balancer ──HTTP :8000──▶ VM private IP
Client ──HTTP  :80───▶ 301 redirect to HTTPS
```

It manages the Load Balancer, one backend set with an HTTP health check, one
backend, an HTTPS listener, an HTTP listener, and the HTTP→HTTPS redirect rule
set. It does **not** create certificates: it uses the certificate OCID you supply.

## Dependencies

* Collection `oracle.oci` (>= 5.5, < 6) and the `oci` Python SDK on the controller.
* `oci_public_subnet_id` from `oci_network` and `oci_compute_private_ip` from
  `oci_compute`, both run earlier in the same play.
* A certificate in the OCI Certificates service, in the same region.

## Variables

| Variable | Default | Description |
|---|---|---|
| `oci_compartment_id` | `""` | **Required.** Compartment OCID |
| `oci_lb_certificate_ocid` | `""` | **Required.** `ocid1.certificate...` for the HTTPS listener |
| `oci_lb_subnet_id` | `oci_public_subnet_id` | Subnet for the Load Balancer |
| `oci_lb_backend_ip_addresses` | `[oci_compute_private_ip]` | Backend addresses. Must be private (RFC 1918) IPs |
| `oci_lb_https_enabled` | `true` | `false` = HTTP listener only, no certificate |
| `oci_lb_health_check_protocol` | `HTTP` | `HTTP` or `TCP` |
| `oci_lb_name` | `app-lb` | Display name (how the role finds it again) |
| `oci_lb_min_bandwidth_mbps` / `oci_lb_max_bandwidth_mbps` | `10` / `10` | Flexible shape bandwidth |
| `oci_lb_backend_set_name` | `app_backend_set` | |
| `oci_lb_backend_policy` | `ROUND_ROBIN` | |
| `oci_lb_backend_port` | `8000` | Application port on the VM |
| `oci_lb_health_check_path` | `/health` | HTTP health check path |
| `oci_lb_health_check_return_code` | `200` | |
| `oci_lb_health_check_interval_ms` / `_timeout_ms` / `_retries` | `10000` / `3000` / `3` | |
| `oci_lb_https_listener_name` / `oci_lb_https_port` | `https_listener` / `443` | |
| `oci_lb_http_listener_name` / `oci_lb_http_port` | `http_listener` / `80` | |
| `oci_lb_http_redirect_to_https` | `true` | Answer HTTP with a 301 to HTTPS |
| `oci_lb_redirect_rule_set_name` | `http_to_https` | Letters, digits and `_` only |
| `oci_lb_ssl_protocols` | `["TLSv1.2"]` | |
| `oci_lb_ssl_cipher_suite_name` | `oci-modern-ssl-cipher-suite-v1` | Predefined OCI cipher suite |
| `oci_freeform_tags` | `{managed_by: ansible}` | |

## Inputs

| Variable | From |
|---|---|
| `oci_public_subnet_id` | `oci_network` |
| `oci_compute_private_ip` | `oci_compute` |

## Outputs

| Fact | Description |
|---|---|
| `oci_load_balancer_id` | Load Balancer OCID |
| `oci_load_balancer_public_ip` | Public IP. Point your DNS record here |
| `oci_load_balancer_backend_set_name` | Backend set name |

## Idempotency

* The Load Balancer is matched by compartment + display name. Its shape
  bandwidth is set at creation only.
* The backend set, listeners and rule set are identified by name and updated in
  place (for example, a new certificate OCID or health check path).
* The backend is identified by `ip:port`. If the VM is replaced and gets a new
  private IP, the stale backend is removed on the next run.

## Example usage

```yaml
- name: Provision the full stack
  hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - oci_network
    - oci_compute
    - oci_load_balancer
```

## Example playbook

See `playbooks/01_provision_oci.yml`.

## OCI requirements

* IAM: `manage load-balancers` and `use virtual-network-family` in the
  compartment, plus permission to use the certificate (e.g.
  `use leaf-certificate-family`). Check the OCI Certificates documentation for
  the exact policy your tenancy needs to attach certificates to load balancers.
* The certificate's domain must match the DNS name clients use; create a DNS A
  record for it pointing to `oci_load_balancer_public_ip`.
* The application must answer `GET /health` with HTTP 200 on the backend port,
  or the backend is marked unhealthy and the Load Balancer returns 502.
