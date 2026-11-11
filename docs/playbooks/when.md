# when

> Run a task only when a condition holds.

---

- **Basic syntax**: `when:` is followed by a condition expression — **no** `{{ }}` needed (unlike other parameters)

```yaml
- name: Only on Debian-based systems
  ansible.builtin.apt:
    name: curl
    state: present
  when: ansible_os_family == "Debian"
```

## Common patterns

- **Based on a variable**: `when: demo_packages | length > 0`
- **Based on facts**: `when: ansible_os_family == "Debian"` (`ansible_os_family` comes from `ansible_facts`)
- **Based on a previous task's result**: use `register` to capture the result, then `when` to check it

```yaml
- name: Check if k3s is already installed
  ansible.builtin.stat:
    path: /usr/local/bin/k3s
  register: k3s_binary

- name: Install k3s
  ansible.builtin.shell: curl -sfL https://get.k3s.io | sh -
  when: not k3s_binary.stat.exists
```

> Similar in effect to the `creates:` argument mentioned in [Idempotency](idempotency.md), but `register` + `when` is more flexible — it can check any condition, not just whether a file exists.

---

[← Variables](variables.md) · [↑ Contents](../../README.md) · [loop →](loop.md)
