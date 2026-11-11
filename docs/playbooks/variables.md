# Variables

> Where variables can be defined, and which one wins when they collide.

---

Variables avoid hardcoding values (package names, paths, versions...) so the same playbook can behave differently per host or environment.

- **Usage**: reference with `{{ variable_name }}` in task parameters

## inventory — tied to a host

Defined directly on the host line as `key=value`, only applies to that host:

```ini
[control]
azure-vm ansible_host=<YOUR_VM_IP> ansible_user=<YOUR_USER> ansible_connection=ssh
```

`ansible_host`, `ansible_user` and `ansible_connection` are variables that tell Ansible which user and connection method to use for this host.

## playbook `vars:` — only valid within this playbook

```yaml
- name: Install packages
  hosts: control
  vars:
    packages:
      - curl
      - git
  tasks:
    - name: Install packages from a variable
      ansible.builtin.apt:
        name: "{{ packages }}"
        state: present
```

Good for variables that belong to this one playbook and don't need to be shared elsewhere.

## group_vars / host_vars — managed separately from inventory

When there are many variables, move them out of the inventory file into their own files:

- `group_vars/<group>.yml` → applies to every host in that group
- `host_vars/<host>.yml` → applies only to that host

Ansible looks for these directories **next to the inventory file**:

```
ansible/
├── ansible.cfg
└── inventory/
    ├── azure.ini
    ├── group_vars/
    │   └── control/
    │       ├── vars.yml
    │       └── vault.yml
    └── host_vars/
        └── azure-vm.yml
```

```yaml
# inventory/group_vars/control/vars.yml
demo_packages:
  - curl
  - git
```

```yaml
# inventory/host_vars/azure-vm.yml
demo_motd_message: "Welcome to the control node"
```

The filename (or directory name) must match the group name from the inventory (e.g. `control`) or the host (IP/hostname).

|                                                   | matches by               | scope                               |
| ------------------------------------------------- | ------------------------ | ----------------------------------- |
| `group_vars/<group>.yml` or `group_vars/<group>/` | inventory **group name** | every host in that group            |
| `host_vars/<host>.yml`                            | inventory **host name**  | only that host, regardless of group |

> `group_vars/<group>/` can be a **directory** instead of a single file — every `.yml` file inside it is loaded and merged. This is the standard way to keep plaintext variables (`vars.yml`) and [Vault-encrypted](vault.md) variables (`vault.yml`) side by side.

> Since `group_vars`/`host_vars` are tied to the **inventory**, their variables are loaded for _every_ playbook run with that inventory — not just one. Prefix variable names (e.g. `demo_packages`, `demo_motd_message`) to avoid collisions with variables used by other playbooks.

## [playbooks/demo_variables.yml](../../ansible/playbooks/demo_variables.yml) — putting it together

```yaml
---
- name: Demonstrate variables
  hosts: control
  become: true
  tasks:
    - name: Install packages from group_vars
      ansible.builtin.apt:
        name: "{{ demo_packages }}"
        state: present
        update_cache: true

    - name: Show host_vars message
      ansible.builtin.debug:
        msg: "{{ demo_motd_message }}"
```

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/demo_variables.yml
```

- `demo_packages` comes from `inventory/group_vars/control/vars.yml` → installs `curl` and `git`
- `demo_motd_message` comes from `inventory/host_vars/azure-vm.yml` → printed via `debug`

## command line `-e` — override at run time

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_k3s.yml -e "k3s_version=v1.29.0"
```

No file changes needed — useful for one-off tests or temporary overrides.

## Precedence

If the same variable is defined in multiple places, Ansible uses the one with the highest precedence (low → high, for the cases above):

`group_vars` < `host_vars` < playbook `vars:` < command line `-e`

For example, if `packages` is `[curl, git]` in `group_vars/control/vars.yml`, but the playbook is run with `-e '{"packages": ["curl"]}'`, only `curl` gets installed — the `-e` value wins.

---

[← become](become.md) · [↑ Contents](../../README.md) · [when →](when.md)
