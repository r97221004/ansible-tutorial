# handlers

> Run a task only when something actually changed — like restarting a service after its config changes.

---

When a config file changes, the service that reads it should restart — but only if the file actually changed, and only once even if multiple tasks modify the config. Handlers solve this.

- A task declares `notify: <handler name>` when it makes a change
- Ansible queues the handler but does not run it immediately
- After all tasks in the play finish, each notified handler runs exactly once

```yaml
tasks:
  - name: Configure containerd to use the systemd cgroup driver
    ansible.builtin.lineinfile:
      path: /etc/containerd/config.toml
      regexp: "SystemdCgroup = false"
      line: "            SystemdCgroup = true"
    notify: Restart containerd # queued only if this task reports changed

handlers:
  - name: Restart containerd
    ansible.builtin.systemd:
      name: containerd
      state: restarted
```

- **Deferred** — the handler runs after all tasks, not at the `notify` line; this prevents unnecessary mid-play restarts
- **Deduplicated** — if ten tasks notify the same handler, it still runs only once
- **Conditional** — if the notifying task reports `ok` (nothing changed), the handler is never queued; no change, no restart
- **Ordered** — multiple handlers run in declaration order, not the order they were notified

**Where handlers live** depends on whether you use a role:

- **Single playbook** — define them inline in the play's `handlers:` block, as shown above.
- **Role** — put them in their own `handlers/main.yml`; Ansible loads it automatically (no import needed). `notify:` matches a handler by its **name**, not by file path, so the task and the handler can sit in different files.

> The kubeadm role uses this pattern in practice: `containerd.yml` notifies `Restart containerd`, which is defined in [`handlers/main.yml`](../../ansible/playbooks/roles/kubeadm/handlers/main.yml) — see [kubeadm Role](../roles/kubeadm.md).

To force handlers to run mid-play instead of waiting until the end:

```yaml
- meta: flush_handlers
```

---

[← loop](loop.md) · [↑ Contents](../../README.md) · [tags →](tags.md)
