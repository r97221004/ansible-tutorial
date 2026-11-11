# Idempotency

> Why re-running a playbook should be safe — and how to keep it that way.

---

Idempotency means: running the same playbook multiple times always produces the same result. The first run makes the necessary changes; every run after that does nothing, because the system is already in the desired state.

This is one of the biggest differences between Ansible and a plain shell script — a shell script that does `apt install` and `mkdir` will error or duplicate work if run twice, but an idempotent playbook can be run as many times as needed without side effects.

## changed vs ok

When you run a playbook, each task reports one of:

- `changed` → the task made a change (first run)
- `ok` → the task checked the current state and found nothing to do (later runs)

```bash
PLAY RECAP *********************************************************
control : ok=5  changed=2  unreachable=0  failed=0
```

For example, an `apt` task installing `curl`:

- 1st run → package not present → installs it → `changed`
- 2nd run → package already present → does nothing → `ok`

The same applies to `file` (directory already exists), `systemd` (service already started/enabled), `template`/`copy` (destination file already matches), etc.

## command / shell are not idempotent by default

`command` and `shell` just run a command — Ansible has no way to know whether it "already happened", so they report `changed` and re-run **every time**.

```yaml
- name: Always runs, always shows changed
  ansible.builtin.shell: curl -sfL https://get.k3s.io | sh -
```

To make a `shell`/`command` task idempotent, add a `creates:` (or `removes:`) argument — Ansible skips the task if that path already exists:

```yaml
- name: Only runs if k3s isn't installed yet
  ansible.builtin.shell: curl -sfL https://get.k3s.io | sh -
  args:
    creates: /usr/local/bin/k3s
```

---

[← Common Modules](../fundamentals/common-modules.md) · [↑ Contents](../../README.md) · [become →](become.md)
