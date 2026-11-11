# become

> Privilege escalation: run tasks as root (or another user) only where needed.

---

Many tasks (installing packages, managing services, writing to system paths) require root privileges. `become: true` tells Ansible to run with sudo.

## Play-level vs task-level

```yaml
- name: Install k3s
  hosts: control
  become: true # applies to every task in this play
  tasks:
    - name: Download and run install script
      ansible.builtin.shell: curl -sfL https://get.k3s.io | sh -
```

- `become: true` at the **play level** → every task runs with sudo
- `become: true` at the **task level** → only that one task runs with sudo

```yaml
tasks:
  - name: Read a normal file
    ansible.builtin.command: cat /etc/hostname

  - name: Read a root-only file
    ansible.builtin.command: cat /etc/shadow
    become: true # only this task uses sudo
```

## become_user and the ad-hoc equivalent

- `become_user: someuser` → become a specific user instead of root (default is root)
- ad-hoc equivalent: add `--become` (or `-b`) to the command

## Avoiding password prompts

`become: true` only works without prompting if the SSH login user has passwordless sudo, or `ansible_become_pass` is configured.

- **Passwordless sudo (recommended)** — set up on the **target machine** itself, independent of Ansible:

  ```bash
  echo "<YOUR_USER> ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/<YOUR_USER>
  ```

  Cloud VM default users (e.g. on Azure) usually already have this configured — that's why `become: true` works without any password prompt in this project.

- **`ansible_become_pass`** — only needed if sudo still requires a password:
  - in inventory (avoid plain text passwords): `ansible_become_pass=xxxx` per host
  - at runtime: `ansible-playbook ... --ask-become-pass` (or `-K`), prompts interactively
  - encrypted with Ansible Vault in group_vars/host_vars (advanced topic)

---

[← Idempotency](idempotency.md) · [↑ Contents](../../README.md) · [Variables →](variables.md)
