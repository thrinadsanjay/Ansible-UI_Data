# python_bootstrap

Makes sure a managed host has a Python that the controller's ansible-core can run
modules with (3.8+ for ansible-core 2.17 and later), before facts are gathered.

* Looks for python3.13 … python3.8 / python3 / platform-python and uses the first new enough one.
* On Oracle Linux / RHEL 7 (Python 3.6 only) installs `rh-python38-python` from
  `ol7_software_collections` and uses `/opt/rh/rh-python38/root/usr/bin/python3.8`.
* Sets `ansible_python_interpreter` for the rest of the run. Local connections are skipped.

Use it in a first play with `gather_facts: false`:

```yaml
- name: Prepare Python for Ansible
  hosts: "{{ target_hosts }}"
  gather_facts: false
  roles: [python_bootstrap]
```
