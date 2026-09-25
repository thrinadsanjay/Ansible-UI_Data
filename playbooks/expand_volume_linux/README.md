# expand_volume_linux

**Description:** Add space to a Linux Mount Point

**Author:** Sanjay Ch

**Created:** 2026-05-28 12:00:37

**Template Updated:** 2026-05-25

---

## Overview

This playbook was generated from a template and is ready for customization. It includes all the necessary files and structure to get started quickly.

### Latest Platform Updates

- UI now supports componentized page templates and centralized theme variables.
- Upload handling supports destination folder controls and include-tag-based visibility.
- Schema Builder and Playbook execution UI are aligned with the new `pages/` and `components/` template structure.

---

## Files

- **main.yml** - Main playbook file with tasks and role definitions
- **ui_schema.yml** - UI form schema defining input fields for web execution
- **vars.yml** - Default variable definitions
- **README.md** - This file

---

## Required Variables

The following variables are required and can be set through the UI or in `vars.yml`:

| Variable | Type | Description | Required |
|----------|------|-------------|----------|
| service_name | text | Name of the service to configure | Yes |
| environment | select | Target environment (dev/staging/production) | Yes |
| api_secret | vault | API secret key from Ansible Vault | Yes |

*Refer to `ui_schema.yml` for the complete list of variables.*

---

## Optional Variables

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| port_number | number | Service port number | 8080 |
| enable_ssl | checkbox | Enable SSL/TLS | false |
| timeout_seconds | number | Service timeout | 30 |

---

## Usage

### Via UI

1. Navigate to the playbook in the Ansible UI
2. Fill in the required fields in the form
3. Configure optional/advanced options as needed
4. Select target inventory
5. Click "Execute Playbook"

### Via Command Line

```bash
ansible-playbook playbooks/expand_volume_linux/main.yml \
  -e service_name="my-service" \
  -e environment="dev" \
  -i inventory/hosts
```

### With Variables File

```bash
ansible-playbook playbooks/expand_volume_linux/main.yml \
  -e @playbooks/expand_volume_linux/vars.yml \
  -i inventory/hosts
```

---

## Customization Guide

### Adding Tasks

Edit `main.yml` and add tasks under the `tasks:` section:

```yaml
tasks:
  - name: Your custom task
    module_name:
      parameter: value
```

### Adding Variables

1. Add field definition to `ui_schema.yml`:
   ```yaml
   fields:
     - name: new_variable
       type: text
       label: "New Variable"
       required: true
   ```

2. Add default value to `vars.yml`:
   ```yaml
   new_variable: "default_value"
   ```

3. Use in `main.yml`:
   ```yaml
   - name: Use new variable
     debug:
       msg: "Value is {{ new_variable }}"
   ```

### Using Roles

Uncomment and modify the `roles:` section in `main.yml`:

```yaml
roles:
  - role: my_role
    vars:
      role_variable: "{{ playbook_variable }}"
```

---

## Advanced Configuration

### Environment-Specific Settings

Use conditional variables based on environment:

```yaml
vars:
  config: "{{ config_dev if environment == 'dev' else config_prod }}"
  config_dev:
    debug: true
    timeout: 60
  config_prod:
    debug: false
    timeout: 30
```

### Tags for Selective Execution

Add tags to tasks for selective execution:

```yaml
tasks:
  - name: Install packages
    apt:
      name: nginx
      state: present
    tags:
      - install
      - packages
```

Run only tagged tasks:
```bash
ansible-playbook main.yml --tags "install"
```

---

## Testing

### Dry Run

Enable dry run mode through the UI or:

```bash
ansible-playbook main.yml --check
```

### Verbose Output

```bash
ansible-playbook main.yml -v    # verbose
ansible-playbook main.yml -vv   # more verbose
ansible-playbook main.yml -vvv  # very verbose
```

---

## Troubleshooting

### Common Issues

1. **Variable not defined**
   - Ensure variable is defined in `vars.yml` or passed via `-e`
   - Check ui_schema.yml for required fields

2. **Permission denied**
   - Use `become: yes` for tasks requiring privileges
   - Verify SSH access and sudo permissions

3. **Module not found**
   - Install required Ansible collections
   - Check module name and parameters

### Debug Mode

Add debug tasks to inspect variables:

```yaml
- name: Debug all variables
  debug:
    var: hostvars[inventory_hostname]
```

---

## Notes

- Review and customize all template placeholder content before production use
- Store sensitive data in Ansible Vault (use `type: vault` in ui_schema.yml)
- Test playbooks in development environment first
- Use version control (git) to track changes
- Document any custom modifications in this README

---

## References

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [YAML Syntax](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)

---

**Last Updated:** 2026-05-25
