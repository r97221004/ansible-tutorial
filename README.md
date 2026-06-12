# Ansible in Practice

> **Problem**: Manually SSH-ing into multiple machines to install packages, copy configs, and start services is slow, repetitive, and easy to get inconsistent.
>
> **Solution**: Ansible lets you describe the desired state of your machines in YAML playbooks and apply them to any number of hosts over SSH — repeatably and idempotently.

## Table of Contents

- [**Install Ansible**](#install-ansible)
- [**ansible.cfg**](#ansiblecfg)
- [**Hello World (Local Connection)**](#hello-world-local-connection)
- [**Switch to SSH Connection**](#switch-to-ssh-connection)
- [**Ad-hoc Commands**](#ad-hoc-commands)
- [**Common Modules**](#common-modules)
- [**Idempotency**](#idempotency)
- [**become — Privilege Escalation**](#become-privilege-escalation)
- [**Variables**](#variables)
- [**when**](#when)
- [**loop**](#loop)
- [**Error Handling**](#error-handling)
- [**Install k3s**](#install-k3s)

---

## Install Ansible

```bash
sudo apt update && sudo apt install -y ansible
```

Check the version:

```bash
ansible --version
# ansible 2.10.8
```

---

## ansible.cfg

Placed in the `ansible/` directory and automatically applied when running `ansible-playbook` from there:

```ini
[defaults]
inventory = inventory/azure.ini   # default inventory, so the -i flag can be omitted
host_key_checking = False         # avoids getting stuck on the host key prompt when SSH-ing into a new machine for the first time
```

---

## Hello World (Local Connection)

### Concept

```
Ansible reads inventory  → knows which machines to run on
Ansible reads playbook   → knows what tasks to run
```

### File Structure

```
ansible/
├── inventory/
│   └── localhost.ini    ← machine list
└── playbooks/
    └── hello.yml        ← task to run
```

### inventory/localhost.ini

```ini
[local]
localhost ansible_connection=local
```

- `[local]` → group name
- `localhost` → machine name (itself)
- `ansible_connection=local` → don't use SSH, run directly on the local machine

### playbooks/hello.yml

```yaml
---
- name: Hello World
  hosts: local # corresponds to the [local] group in inventory
  tasks:
    - name: Print message
      debug:
        msg: "Ansible is working! This machine is {{ ansible_hostname }}"

    - name: Check OS
      debug:
        msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
```

- `hosts` → which group to run against
- `tasks` → list of tasks to execute
- `debug` → module that prints to the screen
- `{{ }}` → variable syntax, automatically gathered by Ansible during the Gathering Facts phase

### Run

```bash
ansible-playbook -i ansible/inventory/localhost.ini ansible/playbooks/hello.yml
```

### Reading the Output

```
TASK [Gathering Facts]   ← automatically gathers machine info (IP, OS, hostname...)
TASK [Print message]     ← ok = success
TASK [Check OS]          ← ok = success

PLAY RECAP
  ok=3      ← all 3 tasks succeeded
  failed=0  ← no failures
```

---

## Switch to SSH Connection

### Concept

In real-world scenarios, Ansible runs commands on remote machines over SSH.

```
Ansible (deployer)  ──SSH──>  target machine
```

### SSH Key Setup

SSH connections support two authentication methods:

- **Password login**: must enter a password every time
- **SSH key (mainstream)**: copy the public key to the target machine in advance, then no password is needed afterwards

**Generate a key pair**

```bash
ssh-keygen -t ed25519
# press Enter through all prompts
```

This generates two files:

- `~/.ssh/id_ed25519` → private key (stays on your own machine, never share it)
- `~/.ssh/id_ed25519.pub` → public key (copy to the target machine)

> `ed25519` is an encryption algorithm that's shorter, faster, and more secure than the older `rsa`, and is the mainstream choice today.

### Connecting to a Remote Azure VM

**Get the VM's public IP**

On the VM page in the Azure Portal, find the "Public IP address", e.g. `40.81.188.168`.

**Open port 22 (NSG rule)**

Azure blocks external connections by default. You need to add an inbound rule to the VM's "Network Security Group (NSG)"
allowing TCP 22 from your source, otherwise SSH won't be able to connect.

> Security tip: restrict the source to "My IP" — don't open port 22 to `0.0.0.0/0` (the whole world).

**Copy the public key to the Azure VM**

```bash
ssh-copy-id matt2_chang@40.81.188.168
# password required the first time (or use the key downloaded when creating the Azure VM), then no password afterwards
```

`ssh-copy-id` automatically writes the public key into the target machine's `~/.ssh/authorized_keys`.

Confirm you can connect:

```bash
ssh matt2_chang@40.81.188.168
# connects without a password = success
```

### inventory/azure.ini

The inventory used for the remote VM:

```ini
[control]
40.81.188.168 ansible_user=matt2_chang ansible_connection=ssh

[node]
```

- `[control]` → the machine that will later become the k3s control node
- `ansible_user=matt2_chang` → the login account on the Azure VM
- `40.81.188.168` → the VM's public IP
- `[node]` → reserved for worker nodes to be added later (currently empty)

### Run (connecting to the remote VM)

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/hello.yml
```

> From Ansible's perspective, whether connecting locally or to a remote Azure VM, the playbook doesn't need to change at all —
> **just swap the IP and account in the inventory**. This is exactly the value Ansible provides.

---

## Ad-hoc Commands

Quick one-off commands without writing a playbook — useful for checking connectivity or running simple commands across hosts.

### Test connectivity

```bash
ansible control -i ansible/inventory/azure.ini -m ping
```

- `control` → the group/host from inventory to target
- `-m ping` → runs the `ping` module (checks SSH + Python are working, not ICMP)

### Run arbitrary commands

```bash
ansible control -i ansible/inventory/azure.ini -a "uptime"
ansible control -i ansible/inventory/azure.ini -a "df -h" --become
```

- `-a "<command>"` → runs a shell command via the `command` module
- `--become` → run with sudo (same as `become: true` in a playbook)

> Ad-hoc commands are great for quick checks; playbooks are for anything repeatable.

---

## Common Modules

A few modules that show up in almost every playbook.

### command vs shell vs script

| Module    | Description                                                                                        |
| --------- | -------------------------------------------------------------------------------------------------- |
| `command` | runs a command directly, no shell features (no pipes `\|`, `&&`, env vars) — safer, default choice |
| `shell`   | runs through `/bin/sh`, supports pipes/redirects/env vars — use when you need shell features       |
| `script`  | uploads and runs a local script on the remote host                                                 |

```yaml
- name: Safe, no shell features needed
  command: k3s kubectl get nodes

- name: Needs a pipe, must use shell
  shell: curl -sfL https://get.k3s.io | sh -
```

### apt — package management

```yaml
- name: Install packages
  apt:
    name:
      - openssh-server
      - curl
    state: present
    update_cache: true
```

- `state: present` → install if missing (idempotent — does nothing if already installed)
- `update_cache: true` → equivalent to `apt update` before installing

### file — manage files and directories

```yaml
- name: Create a directory
  file:
    path: /home/{{ ansible_user }}/.kube
    state: directory
    mode: "0755"

- name: Remove a file
  file:
    path: /usr/local/bin/k9s
    state: absent
```

- `state: directory` → create a directory
- `state: absent` → remove a file or directory

### copy vs template

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
  copy:
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
  template:
    src: motd.j2 # on the control machine (deployer), under templates/
    dest: /etc/motd # on the target/remote machine
```

Each host ends up with its own `/etc/motd`, e.g. `Welcome to node-a / Your IP is 1.2.3.4` on one host and `Welcome to node-b / Your IP is 5.6.7.8` on another — same template, different output per host.

**remote_src — copy a file that's already on the target machine**

By default `copy`/`unarchive`/etc. expect `src` to be on the control machine. Add `remote_src: true` to instead read `src` from the **target machine itself** — useful for moving/renaming a file that was just downloaded or extracted there.

```yaml
- name: Copy kubeconfig to the user's home directory
  copy:
    src: /etc/rancher/k3s/k3s.yaml # already exists on the target machine
    dest: /home/{{ ansible_user }}/.kube/config
    owner: "{{ ansible_user }}"
    mode: "0600"
    remote_src: true # read src from the target machine, not the control machine
```

### service / systemd — manage services

The `systemd` module is Ansible's equivalent of running `systemctl` commands. There's also an older, more generic `service` module (works across systemd/upstart/sysvinit), but on modern Linux `systemd` offers more features.

```yaml
- name: Ensure k3s is running and enabled on boot
  systemd:
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

## Idempotency

Idempotency means: running the same playbook multiple times always produces the same result. The first run makes the necessary changes; every run after that does nothing, because the system is already in the desired state.

This is one of the biggest differences between Ansible and a plain shell script — a shell script that does `apt install` and `mkdir` will error or duplicate work if run twice, but an idempotent playbook can be run as many times as needed without side effects.

### changed vs ok

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

### command / shell are not idempotent by default

`command` and `shell` just run a command — Ansible has no way to know whether it "already happened", so they report `changed` and re-run **every time**.

```yaml
- name: Always runs, always shows changed
  shell: curl -sfL https://get.k3s.io | sh -
```

To make a `shell`/`command` task idempotent, add a `creates:` (or `removes:`) argument — Ansible skips the task if that path already exists:

```yaml
- name: Only runs if k3s isn't installed yet
  shell: curl -sfL https://get.k3s.io | sh -
  args:
    creates: /usr/local/bin/k3s
```

---

## become — Privilege Escalation

Many tasks (installing packages, managing services, writing to system paths) require root privileges. `become: true` tells Ansible to run with sudo.

### Play-level vs task-level

```yaml
- name: Install k3s
  hosts: control
  become: true # applies to every task in this play
  tasks:
    - name: Download and run install script
      shell: curl -sfL https://get.k3s.io | sh -
```

- `become: true` at the **play level** → every task runs with sudo
- `become: true` at the **task level** → only that one task runs with sudo

```yaml
tasks:
  - name: Read a normal file
    command: cat /etc/hostname

  - name: Read a root-only file
    command: cat /etc/shadow
    become: true # only this task uses sudo
```

### become_user and the ad-hoc equivalent

- `become_user: someuser` → become a specific user instead of root (default is root)
- ad-hoc equivalent: add `--become` (or `-b`) to the command

### Avoiding password prompts

`become: true` only works without prompting if the SSH login user has passwordless sudo, or `ansible_become_pass` is configured.

- **Passwordless sudo (recommended)** — set up on the **target machine** itself, independent of Ansible:

  ```bash
  echo "matt2_chang ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/matt2_chang
  ```

  Cloud VM default users (e.g. on Azure) usually already have this configured — that's why `become: true` works without any password prompt in this project.

- **`ansible_become_pass`** — only needed if sudo still requires a password:
  - in inventory (avoid plain text passwords): `ansible_become_pass=xxxx` per host
  - at runtime: `ansible-playbook ... --ask-become-pass` (or `-K`), prompts interactively
  - encrypted with Ansible Vault in group_vars/host_vars (advanced topic)

---

## Variables

Variables avoid hardcoding values (package names, paths, versions...) so the same playbook can behave differently per host or environment.

- **Usage**: reference with `{{ variable_name }}` in task parameters

### inventory — tied to a host

Defined directly on the host line as `key=value`, only applies to that host:

```ini
[control]
40.81.188.168 ansible_user=matt2_chang ansible_connection=ssh
```

`ansible_user` and `ansible_connection` are variables that tell Ansible which user and connection method to use for this host.

### playbook `vars:` — only valid within this playbook

```yaml
- name: Install packages
  hosts: control
  vars:
    packages:
      - curl
      - git
  tasks:
    - name: Install packages from a variable
      apt:
        name: "{{ packages }}"
        state: present
```

Good for variables that belong to this one playbook and don't need to be shared elsewhere.

### group_vars / host_vars — managed separately from inventory

When there are many variables, move them out of the inventory file into their own files:

- `group_vars/<group>.yml` → applies to every host in that group
- `host_vars/<host>.yml` → applies only to that host

Ansible looks for these directories **next to the inventory file**:

```
ansible/
├── ansible.cfg
└── inventory/
    ├── azure.ini
    ├── group_vars/
    │   └── control.yml
    └── host_vars/
        └── 40.81.188.168.yml
```

```yaml
# inventory/group_vars/control.yml
demo_packages:
  - curl
  - git
```

```yaml
# inventory/host_vars/40.81.188.168.yml
demo_motd_message: "Welcome to the control node"
```

The filename must match the group name from the inventory (e.g. `control`) or the host (IP/hostname).

|                          | matches by               | scope                               |
| ------------------------ | ------------------------ | ----------------------------------- |
| `group_vars/<group>.yml` | inventory **group name** | every host in that group            |
| `host_vars/<host>.yml`   | inventory **host name**  | only that host, regardless of group |

> Since `group_vars`/`host_vars` are tied to the **inventory**, their variables are loaded for _every_ playbook run with that inventory — not just one. Prefix variable names (e.g. `demo_packages`, `demo_motd_message`) to avoid collisions with variables used by other playbooks.

### playbooks/demo_variables.yml — putting it together

```yaml
---
- name: Demonstrate variables
  hosts: control
  become: true
  tasks:
    - name: Install packages from group_vars
      apt:
        name: "{{ demo_packages }}"
        state: present
        update_cache: true

    - name: Show host_vars message
      debug:
        msg: "{{ demo_motd_message }}"
```

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/demo_variables.yml
```

- `demo_packages` comes from `inventory/group_vars/control.yml` → installs `curl` and `git`
- `demo_motd_message` comes from `inventory/host_vars/40.81.188.168.yml` → printed via `debug`

### command line `-e` — override at run time

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_k3s.yml -e "k3s_version=v1.29.0"
```

No file changes needed — useful for one-off tests or temporary overrides.

### Precedence

If the same variable is defined in multiple places, Ansible uses the one with the highest precedence (low → high, for the cases above):

`group_vars` < `host_vars` < playbook `vars:` < command line `-e`

For example, if `packages` is `[curl, git]` in `group_vars/control.yml`, but the playbook is run with `-e '{"packages": ["curl"]}'`, only `curl` gets installed — the `-e` value wins.

---

## when

- **Basic syntax**: `when:` is followed by a condition expression — **no** `{{ }}` needed (unlike other parameters)

```yaml
- name: Only on Debian-based systems
  apt:
    name: curl
    state: present
  when: ansible_os_family == "Debian"
```

### Common patterns

- **Based on a variable**: `when: demo_packages | length > 0`
- **Based on facts**: `when: ansible_os_family == "Debian"` (`ansible_os_family` comes from `ansible_facts`)
- **Based on a previous task's result**: use `register` to capture the result, then `when` to check it

```yaml
- name: Check if k3s is already installed
  stat:
    path: /usr/local/bin/k3s
  register: k3s_binary

- name: Install k3s
  shell: curl -sfL https://get.k3s.io | sh -
  when: not k3s_binary.stat.exists
```

> Similar in effect to the `creates:` argument mentioned in [Idempotency](#idempotency), but `register` + `when` is more flexible — it can check any condition, not just whether a file exists.

---

## loop

- **Basic syntax**: `loop:` takes a list, and `{{ item }}` refers to the current value inside the task

```yaml
- name: Create multiple directories
  file:
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

## Error Handling

By default, if a task fails, Ansible stops the play on that host. These directives let you control that behavior.

### ignore_errors

- **`ignore_errors: true`** → lets the task fail without stopping the play; Ansible still reports it as `failed`, but moves on to the next task

```yaml
- name: Update apt cache (repo may be temporarily broken)
  apt:
    update_cache: true
  ignore_errors: true
```

### failed_when

- **`failed_when:`** → overrides what counts as "failed". By default `command`/`shell` only fail on a non-zero exit code; `failed_when` lets you fail based on the output instead

```yaml
- name: Check root disk usage
  command: df -h /
  register: disk_usage
  failed_when: "'100%' in disk_usage.stdout"
```

- the command itself exits `0` (success), but the task is still marked `failed` if `100%` appears in the output

### block / rescue / always

- **`block:`** groups tasks together; **`rescue:`** runs only if a task in the block fails; **`always:`** always runs — similar to try/catch/finally

```yaml
tasks:
  - block:
      - name: Run risky script
        command: /usr/local/bin/risky-script.sh
    rescue:
      - name: Notify on failure
        debug:
          msg: "risky-script.sh failed, continuing anyway"
    always:
      - name: Remove lock file
        file:
          path: /tmp/risky.lock
          state: absent
```

---

## Install k3s

### playbooks/install_k3s.yml

```yaml
---
- name: Install k3s
  hosts: local
  become: true # run with sudo
  tasks:
    - name: Download and run the k3s install script
      shell: curl -sfL https://get.k3s.io | sh -
      args:
        creates: /usr/local/bin/k3s # skip if this file already exists

    - name: Confirm the k3s service is running
      systemd:
        name: k3s
        state: started
        enabled: true

    - name: Check node status
      command: k3s kubectl get nodes
      register: nodes_output

    - name: Print node status
      debug:
        msg: "{{ nodes_output.stdout }}"
```

Key points:

- `become: true` → run with sudo, since installing k3s requires root privileges
- `creates:` → skip this task if the file already exists, to avoid reinstalling
- `register` → store the command output in a variable for use by the next task

### Run

```bash
ansible-playbook -i ansible/inventory/ssh_test.ini ansible/playbooks/install_k3s.yml
```

### Confirm k3s installed successfully

```bash
k3s --version                      # check version
sudo k3s kubectl get nodes         # confirm node status is Ready
sudo k3s kubectl get pods -A       # view pods across all namespaces (-A = all namespaces)
sudo systemctl status k3s          # confirm the service is running
```

> No output from `get pods` (without -A) is normal, since no application has been deployed yet.
> Add `-A` to see system pods.

---

## Uninstall k3s

### Manual uninstall

```bash
sudo /usr/local/bin/k3s-uninstall.sh
```

### playbooks/uninstall_k3s.yml

```yaml
---
- name: Uninstall k3s
  hosts: local
  become: true
  tasks:
    - name: Run the k3s uninstall script
      shell: /usr/local/bin/k3s-uninstall.sh
      args:
        removes: /usr/local/bin/k3s # skip if this file doesn't exist

    - name: Remove k9s binary
      file:
        path: /usr/local/bin/k9s
        state: absent

    - name: Remove kubeconfig
      file:
        path: /home/{{ ansible_user }}/.kube/config
        state: absent

    - name: Confirm k3s binary removed
      stat:
        path: /usr/local/bin/k3s
      register: k3s_binary

    - name: Print result
      debug:
        msg: "{{ 'k3s uninstalled successfully' if not k3s_binary.stat.exists else 'k3s is still present, uninstall failed' }}"
```

Key points:

- `removes:` → skip if the file doesn't exist (the opposite of `creates:`), to avoid running the uninstall script when k3s isn't installed
- `stat` module → checks whether a file exists
- `k3s_binary.stat.exists` → `true` means it's still there, `false` means it's been removed

### Run

```bash
ansible-playbook -i ansible/inventory/ssh_test.ini ansible/playbooks/uninstall_k3s.yml
```

---

## Install k9s and Configure kubeconfig

### Tasks added to install_k3s.yml

**Download and install k9s:**

```yaml
- name: Download k9s
  get_url:
    url: https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz
    dest: /tmp/k9s.tar.gz
    mode: "0644"

- name: Extract k9s
  unarchive:
    src: /tmp/k9s.tar.gz
    dest: /tmp/
    remote_src: true

- name: Install k9s binary
  copy:
    src: /tmp/k9s
    dest: /usr/local/bin/k9s
    mode: "0755"
    remote_src: true
```

**Configure kubeconfig (so k9s works without passing arguments):**

```yaml
- name: Create .kube directory
  file:
    path: /home/{{ ansible_user }}/.kube
    state: directory
    owner: "{{ ansible_user }}"
    mode: "0755"

- name: Copy kubeconfig
  copy:
    src: /etc/rancher/k3s/k3s.yaml
    dest: /home/{{ ansible_user }}/.kube/config
    owner: "{{ ansible_user }}"
    mode: "0600"
    remote_src: true
```

Key points:

- **`get_url`** → downloads a file from the network
- **`unarchive`** → extracts an archive; `remote_src: true` means the file is on the target machine
- **`file` module** → creates directories or removes files; `state: directory` creates a directory, `state: absent` removes it
- **`{{ ansible_user }}`** → automatically resolves to the user configured in inventory, so it doesn't need to be hardcoded
- **`mode: '0600'`** → kubeconfig contains credentials, so only the owner can read/write it

### Manually configure kubeconfig (without Ansible)

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER ~/.kube/config
```

---

## kubectl Basics

### Common commands

```bash
kubectl get nodes                        # view node status
kubectl get pods                         # view pods in the current namespace
kubectl get pods -A                      # view pods across all namespaces
kubectl describe pod <pod-name>          # view pod details (the Events section is most useful)
kubectl logs <pod-name>                  # view pod logs
kubectl exec -it <pod-name> -- bash      # enter a pod's shell
kubectl delete deployment <name>         # delete a deployment (its pods are deleted too)
kubectl apply -f <file>.yaml             # deploy resources from a yaml file
```

### What is kubectl apply -f

A more common approach than `kubectl create` — write the configuration as a yaml file and apply it:

- configuration can be version controlled
- can be applied repeatedly
- can be shared across the team

### k8s/nginx.yaml structure

```yaml
apiVersion: apps/v1 # K8s API version
kind: Deployment # resource type
metadata:
  name: nginx # name
spec:
  replicas: 1 # number of pods to run
  selector:
    matchLabels:
      app: nginx # manages pods with this label
  template: # pod template
    metadata:
      labels:
        app: nginx # pods get this label
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

**Relationship between selector and labels:**
A Deployment finds the pods it manages via `matchLabels`, and pods mark which Deployment they belong to via `labels`.

### How kubectl works (real-world scenario)

kubectl uses `~/.kube/config` to know which cluster to connect to, without needing to SSH into the target machine.
If the deployer and k3s are on different machines, you need to fetch the target machine's kubeconfig back:

```yaml
# Ansible fetch module: pulls a file from the target machine back to the deployer
- name: Fetch kubeconfig back to the deployer
  fetch:
    src: /etc/rancher/k3s/k3s.yaml
    dest: ~/.kube/config
    flat: true
```

### k9s common operations

k9s is a TUI (text user interface) tool for kubectl, reading the same `~/.kube/config` as kubectl.
The difference is it lets you browse the cluster in real time with keyboard shortcuts instead of typing commands repeatedly.

Start it:

```bash
k9s                  # start directly, defaults to the pod view of the current namespace
k9s -n kube-system   # start in a specific namespace
```

**Switch resources (press `:` then type the resource name, equivalent to kubectl get):**

| Action    | Equivalent kubectl                       |
| --------- | ---------------------------------------- |
| `:pods`   | `kubectl get pods`                       |
| `:deploy` | `kubectl get deployments`                |
| `:svc`    | `kubectl get services`                   |
| `:nodes`  | `kubectl get nodes`                      |
| `:ns`     | switch namespace                         |
| `0`       | show all namespaces (equivalent to `-A`) |

**Common keys after selecting a resource:**

| Key      | Action                                     | Equivalent kubectl         |
| -------- | ------------------------------------------ | -------------------------- |
| `d`      | view details                               | `kubectl describe`         |
| `l`      | view logs                                  | `kubectl logs`             |
| `s`      | enter the pod's shell                      | `kubectl exec -it -- bash` |
| `y`      | view full YAML                             | `kubectl get -o yaml`      |
| `Ctrl+d` | delete resource                            | `kubectl delete`           |
| `Enter`  | drill down (e.g. deploy → pod → container) | —                          |

**General operations:**

| Key              | Action                           |
| ---------------- | -------------------------------- |
| `/`              | filter / search the current list |
| `Esc`            | go back / cancel                 |
| `?`              | show keyboard shortcuts          |
| `:q` or `Ctrl+c` | quit k9s                         |

> Cheat sheet: **`:` to switch resources, `/` to find things, then `d`/`l`/`s` after selecting** — covers about 90% of daily checks.

---

## Service

### Concept

```
Browser → Service → pod
```

A Service is how external traffic reaches pods, finding the matching pods via `selector`.

**Service types:**

| Type           | Description                                            |
| -------------- | ------------------------------------------------------ |
| `ClusterIP`    | accessible only within the cluster (default)           |
| `NodePort`     | opens a port on the node, accessible externally        |
| `LoadBalancer` | for cloud use, automatically provisions an external IP |

### k8s/nginx-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx # finds pods with this label (matches the Deployment template's labels)
  ports:
    - port: 80 # the Service's own port
      targetPort: 80 # forwarded to the pod's port
      nodePort: 30080 # the port used to connect from outside
```

**Traffic path:**

```
Browser (Windows) → 172.21.243.214:30080 → Service:80 → nginx pod:80
```

**The selector must match the Deployment template's labels:**

```yaml
# nginx.yaml (Deployment)
template:
  metadata:
    labels:
      app: nginx    ← the Service's selector looks at this (not the Deployment's metadata)

# nginx-service.yaml (Service)
selector:
  app: nginx        ← these two must match
```

> WSL's 127.0.0.1 isn't reachable from a Windows browser — use WSL's actual IP (`ip addr show eth0`).

### Run

```bash
kubectl apply -f k8s/nginx-service.yaml
kubectl get services
curl http://127.0.0.1:30080    # test inside WSL
# use http://172.21.243.214:30080 in the browser
```

---

## Ansible Role

### Concept

Split a playbook into a modular Role structure, with each piece of functionality managed independently:

```
Before: install_k3s.yml  → everything crammed into one file
Now:    site.yml         → only 3 lines, all the details live in the Role
```

### Directory Structure

```
ansible/
├── inventory/
│   ├── localhost.ini
│   └── ssh_test.ini
└── playbooks/
    ├── site.yml              ← playbook that calls the Role
    ├── install_k3s.yml       ← old version (kept for reference)
    ├── uninstall_k3s.yml
    └── roles/
        └── k3s/
            ├── tasks/
            │   ├── main.yml          ← imports the other task files
            │   ├── k3s.yml           ← k3s installation
            │   ├── k9s.yml           ← k9s installation
            │   └── kubeconfig.yml    ← kubeconfig setup
            ├── defaults/
            │   └── main.yml          ← default variables
            └── handlers/
```

### File descriptions

**playbooks/site.yml**

```yaml
---
- name: Install k3s
  hosts: local
  become: true
  roles:
    - k3s
```

**roles/k3s/tasks/main.yml**

```yaml
---
- import_tasks: k3s.yml
- import_tasks: k9s.yml
- import_tasks: kubeconfig.yml
```

**roles/k3s/defaults/main.yml**

```yaml
---
k9s_url: "https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz"
```

### Variable precedence (highest to lowest)

```
1. passed at runtime       -e "k9s_url=xxx"
2. inventory's group_vars / host_vars
3. vars in the playbook
4. roles/k3s/defaults/main.yml   ← default values
```

### Run

```bash
ansible-playbook -i ansible/inventory/ssh_test.ini ansible/playbooks/site.yml
```

> Ansible looks for the `roles/` directory next to the playbook by default, so roles must be placed under `playbooks/roles/`.

---

## Handlers + Templates

### Handlers

Only triggered when a task's status is `changed`; `skipped` or `ok` won't trigger it. All handlers run once, together, at the end of the play.

```yaml
# handlers/main.yml
- name: log k3s installed
  lineinfile:
    path: /var/log/ansible-k3s.log
    line: "{{ ansible_date_time.iso8601 }} k3s installation complete"
    create: true
```

```yaml
# tasks/k3s.yml
- name: Download and run the k3s install script
  shell: curl -sfL https://get.k3s.io | sh -
  args:
    creates: /usr/local/bin/k3s
  notify: log k3s installed # only triggers on changed, not on skipped
```

- `lineinfile` → adds a line to a file; `create: true` creates the file automatically if it doesn't exist
- `ansible_date_time.iso8601` → a timestamp variable automatically gathered by Ansible

### Templates

Fill in variables in a `.j2` template to generate a config file — more flexible than `copy`.

```
templates/config.yaml.j2  → (fill in variables) →  /etc/rancher/k3s/config.yaml
```

**roles/k3s/templates/config.yaml.j2**

```yaml
node-ip: {{ ansible_host }}
flannel-iface: {{ flannel_iface }}
tls-san:
  - {{ ansible_host }}
```

**roles/k3s/defaults/main.yml**

```yaml
k9s_url: "https://..."
flannel_iface: "eth0"
```

**Using the template in tasks/k3s.yml:**

```yaml
- name: Create k3s config directory
  file:
    path: /etc/rancher/k3s
    state: directory
    mode: '0755'

- name: Generate k3s config.yaml
  template:
    src: config.yaml.j2       # automatically looked up under roles/k3s/templates/
    dest: /etc/rancher/k3s/config.yaml
    mode: '0644'

- name: Download and run the k3s install script  # reads the config.yaml generated above once installed
  ...
```

### Meaning of the three parameters

| Parameter       | Description                                        | Default value issue                         |
| --------------- | -------------------------------------------------- | ------------------------------------------- |
| `node-ip`       | which IP k3s itself uses                           | might pick an external IP                   |
| `flannel-iface` | which network interface pod traffic uses           | might pick the wrong one with multiple NICs |
| `tls-san`       | which IPs are allowed to connect to the API server | only localhost by default                   |

### Verify k3s picked up config.yaml

In k9s, view node describe and look for this line:

```
k3s.io/node-args: ["server","--node-ip","172.21.243.214","--flannel-iface","eth0","--tls-san","172.21.243.214"]
```

### Note

`node-ip` cannot be set to `127.0.0.1` (loopback) — k3s will reject it.
The inventory's `ansible_host` should be set to the machine's real IP (`172.21.243.214`).

---

## Current Progress

- [x] Install Ansible
- [x] Hello World (local connection)
- [x] Set up SSH server
- [x] Generate SSH key pair
- [x] Copy public key to target machine (ssh-copy-id)
- [x] Run hello.yml over SSH to confirm connectivity
- [x] Write the install_k3s.yml playbook
- [x] Install k3s with Ansible
- [x] Confirm k3s is working (get nodes Ready, get pods -A)
- [x] Add k9s installation and kubeconfig setup
- [x] Write the uninstall_k3s.yml playbook (cleans up k9s, kubeconfig, rancher directory)
- [x] kubectl basics (get / describe / logs / exec / delete / apply)
- [x] Deploy an nginx deployment
- [x] Create a Service and access nginx from the browser
- [x] Learn the Ansible Role structure
- [x] Learn Handlers (conditional triggers)
- [x] Learn Templates (dynamically generate k3s config.yaml)

## kubeadm Installation (Ansible Role)

### Role Structure

```
roles/kubeadm/
├── tasks/
│   ├── main.yml          ← import order: prerequisites → containerd → init → flannel → k9s
│   ├── prerequisites.yml ← installs prerequisite packages + kubelet/kubeadm/kubectl
│   ├── containerd.yml    ← installs the container runtime
│   ├── init.yml          ← kubeadm init + wait for readiness + kubeconfig
│   ├── flannel.yml       ← installs the CNI, specifying the network interface
│   └── k9s.yml           ← installs k9s
├── defaults/
│   └── main.yml          ← flannel_iface, k9s_url
└── handlers/
    └── main.yml          ← restart containerd, restart kubelet
```

### Multi-NIC parameters

| kubeadm parameter               | Equivalent k3s parameter | Description                              |
| ------------------------------- | ------------------------ | ---------------------------------------- |
| `--apiserver-advertise-address` | `node-ip`                | which IP the API server binds to         |
| `--apiserver-cert-extra-sans`   | `tls-san`                | which IPs the SSL certificate trusts     |
| `--control-plane-endpoint`      | no equivalent            | unified cluster entry point (for HA)     |
| flannel `--iface`               | `flannel-iface`          | which network interface pod traffic uses |

### Run

```bash
ansible-playbook -i ansible/inventory/ssh_test.ini ansible/playbooks/install_kubeadm.yml
```

### WSL2 Limitations

kubeadm cannot run reliably on WSL2, because:

- incomplete cgroup support
- etcd needs reliable fsync, which the WSL2 filesystem doesn't guarantee
- container restart mechanisms conflict with systemd

**Verified:**

- kubeadm init can succeed
- multi-NIC parameters are configured correctly
- flannel can be installed

**A bare-metal machine or cloud VM is required to run this end-to-end.**

### Notes

- WSL2 needs swap disabled first: `sudo swapoff -a`
- when apt has a broken repo, separate `update_cache` into its own task and add `ignore_errors: true`
- after kubeadm init, wait for etcd (port 2379) and the API server (port 6443) to become ready before continuing

---

## Current Progress

- [x] Install Ansible
- [x] Hello World (local connection)
- [x] Set up SSH server
- [x] Generate SSH key pair
- [x] Copy public key to target machine (ssh-copy-id)
- [x] Run hello.yml over SSH to confirm connectivity
- [x] Write the install_k3s.yml playbook
- [x] Install k3s with Ansible
- [x] Confirm k3s is working (get nodes Ready, get pods -A)
- [x] Add k9s installation and kubeconfig setup
- [x] Write the uninstall_k3s.yml playbook (cleans up k9s, kubeconfig, rancher directory)
- [x] kubectl basics (get / describe / logs / exec / delete / apply)
- [x] Deploy an nginx deployment
- [x] Create a Service and access nginx from the browser
- [x] Learn the Ansible Role structure
- [x] Learn Handlers (conditional triggers)
- [x] Learn Templates (dynamically generate k3s config.yaml)
- [x] Write the kubeadm Ansible Role (multi-NIC parameters: advertise-address, cert-extra-sans, flannel iface)
- [x] Partially verified kubeadm on WSL2 (init succeeds, but WSL2 limitations prevent stable operation)

## Next Steps

- [ ] Fully verify kubeadm on a cloud VM or QTC bare-metal machine
- [ ] Implement multi-NIC configuration (corresponds to k3s-multi-nic-verification.md)
