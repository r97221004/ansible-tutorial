# tags

> Run (or skip) just part of a playbook.

---

Tags let you run a subset of tasks from a playbook without modifying it or creating a separate playbook. The examples below are illustrative — the playbooks in this repo don't define tags.

```yaml
tasks:
  - name: Install kubelet kubeadm kubectl
    ansible.builtin.apt:
      name: [kubelet, kubeadm, kubectl]
      state: present
    tags: install

  - name: Query node status
    ansible.builtin.command: kubectl get nodes
    changed_when: false
    tags: verify
```

```bash
# Run only tasks tagged "verify"
ansible-playbook <your-playbook>.yml --tags verify

# Run everything except "verify"
ansible-playbook <your-playbook>.yml --skip-tags verify

# List all available tags without running anything
ansible-playbook <your-playbook>.yml --list-tags
```

- A task can carry **multiple tags**: `tags: [install, k8s]`; `--tags "install,verify"` runs tasks matching either
- `always` is a reserved tag — those tasks run on every invocation unless explicitly excluded with `--skip-tags always`
- Tags also apply to roles: `roles: - { role: k3s, tags: k3s }`

---

[← handlers](handlers.md) · [↑ Contents](../../README.md) · [Error Handling →](error-handling.md)
