# loop

> Repeat one task over a list instead of copy-pasting it.

---

- **Basic syntax**: `loop:` takes a list, and `{{ item }}` refers to the current value inside the task

```yaml
- name: Create multiple directories
  ansible.builtin.file:
    path: "/home/{{ ansible_user }}/{{ item }}"
    state: directory
  loop:
    - .kube
    - .ssh
```

- **vs. `apt: name: [...]`**:
  - Package modules like `apt`/`yum` already accept a list directly — no `loop` needed
  - `loop` is for modules whose parameters **don't** accept a list, e.g. `file`, `copy`, `user` — each item runs as its own task

- **`with_items` (old syntax)**: works similarly to `loop`, but is the older form — new playbooks should use `loop`

---

[← when](when.md) · [↑ Contents](../../README.md) · [handlers →](handlers.md)
