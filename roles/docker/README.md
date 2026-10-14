# docker

## Purpose

Installs Docker Engine and the Docker Compose plugin from Docker's official
package repository, starts Docker, enables it at boot, and optionally adds
users to the `docker` group.

The role has no cloud-specific logic. It works on any supported Linux host,
whether it runs on OCI, Proxmox, VMware or bare metal. It never pipes `curl`
into a shell.

## Supported platforms

| OS family | Status |
|---|---|
| Debian / Ubuntu (22.04, 24.04; amd64 and arm64) | Supported |
| RedHat / Oracle Linux / RHEL / Rocky / Alma 8 and 9 | Supported (Docker RHEL repository). Not yet tested on a live host. Oracle Linux 7 is not supported: ansible-core 2.17+ needs Python 3.8+ on the target |

## Dependencies

None. Needs `become: true` and internet access to `download.docker.com`.

## Variables

| Variable | Default | Description |
|---|---|---|
| `docker_install_compose_plugin` | `true` | Install `docker-compose-plugin` (`docker compose ...`) |
| `docker_manage_service` | `true` | Start the `docker` service (and restart it when `daemon.json` changes) |
| `docker_service_enabled` | `true` | Start Docker at boot |
| `docker_users` | `[]` | Users added to the `docker` group. Membership is root-equivalent |
| `docker_repo_channel` | `stable` | Docker repository channel |
| `docker_packages` | engine, CLI, containerd, buildx | Packages installed |
| `docker_compose_package` | `docker-compose-plugin` | |
| `docker_conflicting_packages` | `docker.io`, `podman-docker`, ... | Removed before installation |
| `docker_daemon_options` | json-file logs, 10 MB × 3 | Content of `/etc/docker/daemon.json`; `{}` leaves the file unmanaged |

## Inputs

Only the variables above. Uses gathered facts (`os_family`, `distribution`,
`distribution_release`, `architecture`).

## Outputs

None. Docker and `docker compose` are available on the host afterwards.

## Handlers

* `Restart docker`: runs when `daemon.json` changes, flushed at the end of the role.

## Idempotency

A second run reports no changes. This was verified on Ubuntu 24.04.

## Example usage

```yaml
- name: Configure Docker hosts
  hosts: app_server
  become: true
  roles:
    - role: docker
      vars:
        docker_users:
          - ubuntu
```

## Example playbook

See `playbooks/02_configure_app_server.yml`.
