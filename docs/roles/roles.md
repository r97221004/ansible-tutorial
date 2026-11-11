# Ansible Roles

> Package tasks, defaults, and metadata into a reusable, conventionally-named directory.

---

So far, every playbook has put its tasks directly under `tasks:`. That works for a handful of tasks, but doesn't scale — there's no clean place to keep default settings, and no easy way to reuse the same logic across multiple playbooks.

A **Role** solves this: it packages tasks, default variables, and metadata into a self-contained, conventionally-named directory. Ansible automatically finds `tasks/main.yml`, `defaults/main.yml`, etc. inside it — no explicit imports needed. The playbook itself shrinks down to "use this role":

```yaml
- hosts: control
  become: true
  roles:
    - k3s
```

```
ansible/playbooks/
├── install_k3s.yml      ← playbook, just points at a role
├── uninstall_k3s.yml
└── roles/
    └── k3s/
        ├── tasks/
        │   ├── main.yml      ← entry point, dispatches to install.yml / uninstall.yml
        │   ├── install.yml
        │   └── uninstall.yml
        ├── defaults/
        │   └── main.yml      ← default variables (lowest precedence)
        └── meta/
            └── main.yml      ← role metadata (description, dependencies...)
```

> Ansible looks for `roles/` next to the playbook by default, so roles must live under `playbooks/roles/`.

---

[← Ansible Vault](../playbooks/vault.md) · [↑ Contents](../../README.md) · [k3s Role →](k3s.md)
