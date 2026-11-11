# Common Modules

> The handful of built-in modules you'll reach for in almost every playbook.

---

A few modules that show up in almost every playbook.

## command vs shell vs script

| Module    | Description                                                                                        |
| --------- | -------------------------------------------------------------------------------------------------- |
| `command` | runs a command directly, no shell features (no pipes `\|`, `&&`, env vars) — safer, default choice |
| `shell`   | runs through `/bin/sh`, supports pipes/redirects/env vars — use when you need shell features       |
| `script`  | uploads and runs a local script on the remote host                                                 |

```yaml
- name: Safe, no shell features needed
  ansible.builtin.command: k3s kubectl get nodes

- name: Needs a pipe, must use shell
  ansible.builtin.shell: curl -sfL https://get.k3s.io | sh -
```

Unlike `apt` or `file`, `command`/`shell` can't tell whether they actually changed anything, so they **always report `changed`**. Add `changed_when:` to say when a task really counts as a change — use `changed_when: false` for read-only queries so they report `ok` instead of a misleading `changed`:

```yaml
- name: Query node status (read-only, never changes anything)
  ansible.builtin.command: kubectl get nodes
  changed_when: false
```

## apt — package management

```yaml
- name: Install packages
  ansible.builtin.apt:
    name:
      - openssh-server
      - curl
    state: present
    update_cache: true
```

- `state: present` → install if missing (idempotent — does nothing if already installed)
- `update_cache: true` → equivalent to `apt update` before installing

## file — manage files and directories

```yaml
- name: Create a directory
  ansible.builtin.file:
    path: /home/{{ ansible_user }}/.kube
    state: directory
    mode: "0755"

- name: Remove a file
  ansible.builtin.file:
    path: /usr/local/bin/k9s
    state: absent
```

- `state: directory` → create a directory (on the **remote managed host**, not the control node)
- `state: absent` → remove a file or directory
- `mode: "0755"` → set permissions, same as `chmod` (quote it, or YAML may misread the octal)

| mode     | owner | group | other | typical use              |
| -------- | ----- | ----- | ----- | ------------------------ |
| `"0755"` | `rwx` | `r-x` | `r-x` | directories, executables |
| `"0644"` | `rw-` | `r--` | `r--` | regular files            |

## copy vs template

| Module     | Description                                                                                   |
| ---------- | --------------------------------------------------------------------------------------------- |
| `copy`     | copies a file to the target machine — file **content** is sent as-is                          |
| `template` | renders a `.j2` Jinja2 file — file **content** has `{{ variables }}` filled in before copying |

`{{ variables }}` can appear in two different places, and only one of them is affected by which module you use:

1. **Task parameters** (`src`, `dest`, `owner`, `mode`, ...) — always resolved by Ansible, for both `copy` and `template`
2. **File content** (what's inside the file `src` points to) — only resolved for `template`, never for `copy`

**copy — content is fixed, but the destination path can still be dynamic**

```yaml
- name: Copy a static config file into each user's home directory
  ansible.builtin.copy:
    src: files/app.conf
    dest: /home/{{ ansible_user }}/app.conf # task parameter → resolved by Ansible
```

`src` is a file on the control machine, `dest` is the path on the target machine. If `files/app.conf` itself contained the text `{{ ansible_user }}`, it would be copied over literally as `{{ ansible_user }}` — `copy` never touches file content.

**template — both the path and the file content can be dynamic**

**templates/motd.j2**

```
Welcome to {{ ansible_hostname }}
Your IP is {{ ansible_host }}
```

```yaml
- name: Generate motd from template
  ansible.builtin.template:
    src: motd.j2 # on the control machine, under templates/
    dest: /etc/motd # on the target/remote machine
```

Each host ends up with its own `/etc/motd`, e.g. `Welcome to node-a / Your IP is 1.2.3.4` on one host and `Welcome to node-b / Your IP is 5.6.7.8` on another — same template, different output per host.

**remote_src — copy a file that's already on the target machine**

By default `copy`/`unarchive`/etc. expect `src` to be on the control machine. Add `remote_src: true` to instead read `src` from the **target machine itself** — useful for moving/renaming a file that was just downloaded or extracted there.

```yaml
- name: Copy kubeconfig to the user's home directory
  ansible.builtin.copy:
    src: /etc/rancher/k3s/k3s.yaml # already exists on the target machine
    dest: /home/{{ ansible_user }}/.kube/config
    owner: "{{ ansible_user }}"
    mode: "0600"
    remote_src: true # read src from the target machine, not the control machine
```

## service / systemd — manage services

The `systemd` module is Ansible's equivalent of running `systemctl` commands. There's also an older, more generic `service` module (works across systemd/upstart/sysvinit), but on modern Linux `systemd` offers more features.

```yaml
- name: Ensure k3s is running and enabled on boot
  ansible.builtin.systemd:
    name: k3s
    state: started
    enabled: true
```

- `name: k3s` → which service to manage, equivalent to `systemctl status k3s`
- `state: started` → make sure the service is running now
  - already running → does nothing (idempotent)
  - not running → runs `systemctl start k3s`
  - other values: `stopped`, `restarted`, `reloaded`
- `enabled: true` → make sure it starts on boot, equivalent to `systemctl enable k3s` (`false` → `systemctl disable`)

This single task does two things at once — start it now, and make sure it auto-starts on boot — and produces the same result no matter how many times it runs.

---

[← Ad-hoc Commands](ad-hoc-commands.md) · [↑ Contents](../../README.md) · [Idempotency →](../playbooks/idempotency.md)
