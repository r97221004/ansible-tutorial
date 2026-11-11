# Error Handling

> Decide what counts as failure, and what happens when a task fails.

---

By default, if a task fails, Ansible stops the play on that host. These directives let you control that behavior.

## ignore_errors

- **`ignore_errors: true`** → lets the task fail without stopping the play; Ansible still reports it as `failed`, but moves on to the next task

```yaml
- name: Update apt cache (repo may be temporarily broken)
  ansible.builtin.apt:
    update_cache: true
  ignore_errors: true
```

## failed_when

- **`failed_when:`** → overrides what counts as "failed". By default `command`/`shell` only fail on a non-zero exit code; `failed_when` lets you fail based on the output instead

```yaml
- name: Check root disk usage
  ansible.builtin.command: df -h /
  register: disk_usage
  failed_when: "'100%' in disk_usage.stdout"
```

- the command itself exits `0` (success), but the task is still marked `failed` if `100%` appears in the output

## block / rescue / always

- **`block:`** groups tasks together; **`rescue:`** runs only if a task in the block fails; **`always:`** always runs — similar to try/catch/finally

```yaml
tasks:
  - block:
      - name: Run risky script
        ansible.builtin.command: /usr/local/bin/risky-script.sh
    rescue:
      - name: Notify on failure
        ansible.builtin.debug:
          msg: "risky-script.sh failed, continuing anyway"
    always:
      - name: Remove lock file
        ansible.builtin.file:
          path: /tmp/risky.lock
          state: absent
```

---

[← tags](tags.md) · [↑ Contents](../../README.md) · [Ansible Vault →](vault.md)
