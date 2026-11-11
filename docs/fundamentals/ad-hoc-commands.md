# Ad-hoc Commands

> One-off commands across your hosts, without writing a playbook.

---

Quick one-off commands without writing a playbook — useful for checking connectivity or running simple commands across hosts.

## Test connectivity

The first thing to check is whether Ansible can actually reach the host. This runs the `ping` module against every host in the `control` group:

```bash
ansible control -i ansible/inventory/azure.ini -m ping
```

- `ansible` → the ad-hoc command itself — runs a single module without writing a playbook (different from `ansible-playbook`)
- `control` → **who to run against**, not a command — it's a group (or host) name from your inventory. Here it matches the `[control]` group in `azure.ini`; you could just as well use `azure-vm`, `all`, or `control:node`.
- `-i ansible/inventory/azure.ini` → which inventory file to read. Required from the repo root (no `ansible.cfg` there); optional if you run from `ansible/`, where `ansible.cfg` sets a default inventory.
- `-m` → which **module** to run (a module is Ansible's unit of work — `ping`, `copy`, `service`, …; it's the same thing as the module name under `tasks:` in a playbook). `-m ping` confirms SSH login **and** a usable Python on the target — this is **not** an ICMP ping. Omit `-m` and Ansible defaults to the `command` module (used in the next section).

A successful run looks like:

```text
azure-vm | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## Run arbitrary commands

```bash
ansible control -i ansible/inventory/azure.ini -a "uptime"
ansible control -i ansible/inventory/azure.ini -a "df -h" --become
```

- No `-m` here → Ansible falls back to its default, the `command` module, so you don't have to type `-m command`.
- `-a "<command>"` → the **arguments** passed to that module. For the `command` module, the argument is the command line to run on the target (e.g. `uptime`, `df -h`).
- `--become` → run with sudo (same as `become: true` in a playbook)

> Ad-hoc commands are great for quick checks; playbooks are for anything repeatable.

---

[← Switch to SSH Connection](ssh-connection.md) · [↑ Contents](../../README.md) · [Common Modules →](common-modules.md)
