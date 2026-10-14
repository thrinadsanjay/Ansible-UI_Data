# manage_instance

**Description:** Create or remove groups and users (SSH keys, sudo) and install, update or remove packages

**Author:** Sanjay Ch

**Created:** 2026-10-12

---

## Overview

Runs on the hosts picked in the Ansible UI host picker. Tick one or more sections:

| Section | What it does |
|---------|--------------|
| Manage Groups | Create or delete the listed groups |
| Manage Users | Create/update users: shell, supplementary groups (appended), SSH keys, password, sudo. Or delete them (optionally with home) |
| Manage Packages | Install / update to latest / remove packages; optionally refresh the cache and upgrade everything |

Order: groups → users → packages, so new users can join new groups in the same run.
Works on Debian/Ubuntu (apt) and Oracle Linux/RHEL 8+ (dnf). Re-runs with the same input report no changes.

---

## Files

- **main.yml** - Main playbook
- **vars.yml** - UI inputs and defaults
- **internal_vars.yml** - Comma lists turned into lists
- **ui_schema.yml** - UI form schema
- **checks.yml** / **groups.yml** / **users.yml** / **packages.yml** - Workflows

---

## Variables

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| target_hosts | text | From the host picker | |
| manage_groups | checkbox | Enable the groups section | false |
| group_names | text | Comma separated | "" |
| group_state | select | present / absent | present |
| manage_users | checkbox | Enable the users section | false |
| user_names | text | Comma separated (not root) | "" |
| user_state | select | present / absent | present |
| user_groups | text | Supplementary groups, comma separated (must exist) | "" |
| user_ssh_public_key | textarea | Keys added to authorized_keys, one per line | "" |
| user_password | password | Optional; stored as SHA-512 hash (openssl on the host) | "" |
| user_sudo | checkbox | `/etc/sudoers.d/90-ansible-<user>`; unticked removes it | false |
| user_sudo_nopasswd | checkbox | Sudo without password | false |
| user_shell | text | Login shell | /bin/bash |
| user_remove_home | checkbox | With absent: delete home | false |
| manage_packages | checkbox | Enable the packages section | false |
| package_names | text | Comma or space separated | "" |
| package_state | select | present / latest / absent | present |
| update_package_cache | checkbox | apt update first | true |
| upgrade_all_packages | checkbox | Upgrade every package | false |

---

## Usage

### Via UI

1. Pick the host(s) in the host picker
2. Tick a section and fill in its fields
3. Execute

### Via Command Line

```bash
ansible-playbook playbooks/manage_instance/main.yml -i inventory/inventory.ini \
  -e target_hosts=web01 -e manage_users=true -e user_names=alice \
  -e user_groups=wheel -e user_sudo=true
```

---

## Notes

- Sudo rules are validated with `visudo` before they are written.
- SSH keys are only added; remove old keys by hand.
- Ubuntu hosts report when a reboot is needed after an upgrade.

---

**Last Updated:** 2026-10-12
