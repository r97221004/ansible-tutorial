# kube_tools Role (k9s, Helm)

> Install the k9s terminal UI and Helm to work with the cluster.

---

**[roles/kube_tools/defaults/main.yml](../../ansible/playbooks/roles/kube_tools/defaults/main.yml)**

```yaml
---
# Default variables for the kube_tools role; override in inventory or the playbook

# present = install, absent = uninstall
kube_tools_state: present

# k9s version: latest grabs the newest release, or pin one like v0.32.5
k9s_version: latest

# binary install location
k9s_bin_dir: /usr/local/bin

# Build the download URL from the version (latest uses /latest/download, pinned uses /download/<tag>)
k9s_url: >-
  {{
    'https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz'
    if k9s_version == 'latest'
    else 'https://github.com/derailed/k9s/releases/download/' ~ k9s_version ~ '/k9s_Linux_amd64.tar.gz'
  }}

# Helm version: latest looks up the newest release via the GitHub API, or pin one like v4.3.1
helm_version: latest

# binary install location
helm_bin_dir: /usr/local/bin
```

- `kube_tools_state` → same present/absent dispatch pattern as the k3s role (see [Ansible Roles](roles.md))
- `k9s_version` → `latest` always grabs the newest release; pinning a version (e.g. `v0.32.5`) keeps installs reproducible
- `k9s_bin_dir` → where the k9s binary gets installed, defaults to `/usr/local/bin` like k3s
- `k9s_url` → a Jinja2 conditional that builds the download URL from `k9s_version` — `latest` and a pinned version use different GitHub Releases paths
- `helm_version` / `helm_bin_dir` → same idea for Helm; there's no `helm_url` because the `latest` tag has to be looked up at runtime (see the install tasks below)

**[roles/kube_tools/tasks/main.yml](../../ansible/playbooks/roles/kube_tools/tasks/main.yml)**

```yaml
---
# Dispatch to install or uninstall based on kube_tools_state
- name: Install kube tools
  ansible.builtin.include_tasks: install.yml
  when: kube_tools_state == "present"

- name: Uninstall kube tools
  ansible.builtin.include_tasks: uninstall.yml
  when: kube_tools_state == "absent"
```

## Install — [tasks/install.yml](../../ansible/playbooks/roles/kube_tools/tasks/install.yml)

```yaml
---
- name: Download k9s
  ansible.builtin.get_url:
    url: "{{ k9s_url }}"
    dest: /tmp/k9s.tar.gz
    mode: "0644"
    timeout: 120 # ~40MB file, give it extra time
  register: k9s_download
  retries: 3 # retry up to 3 times on network blips
  delay: 5
  until: k9s_download is succeeded

- name: Extract k9s
  ansible.builtin.unarchive:
    src: /tmp/k9s.tar.gz
    dest: /tmp/
    remote_src: true

- name: Install the k9s binary
  become: true # only writing to /usr/local/bin needs root
  ansible.builtin.copy:
    src: /tmp/k9s
    dest: "{{ k9s_bin_dir }}/k9s"
    mode: "0755"
    remote_src: true

- name: Check the k9s version
  ansible.builtin.command: k9s version
  register: k9s_check
  changed_when: false

- name: Print the k9s version
  ansible.builtin.debug:
    msg: "{{ k9s_check.stdout_lines }}"

- name: Look up the latest Helm release
  ansible.builtin.uri:
    url: https://api.github.com/repos/helm/helm/releases/latest
    return_content: true
  register: helm_latest
  when: helm_version == 'latest' # get.helm.sh has no /latest URL, so ask GitHub for the newest tag

- name: Download Helm
  ansible.builtin.get_url:
    url: "https://get.helm.sh/helm-{{ helm_latest.json.tag_name if helm_version == 'latest' else helm_version }}-linux-amd64.tar.gz"
    dest: /tmp/helm.tar.gz
    mode: "0644"
    timeout: 120
  register: helm_download
  retries: 3
  delay: 5
  until: helm_download is succeeded

- name: Extract Helm
  ansible.builtin.unarchive:
    src: /tmp/helm.tar.gz
    dest: /tmp/
    remote_src: true

- name: Install the Helm binary
  become: true # only writing to /usr/local/bin needs root
  ansible.builtin.copy:
    src: /tmp/linux-amd64/helm # the tarball unpacks into linux-amd64/
    dest: "{{ helm_bin_dir }}/helm"
    mode: "0755"
    remote_src: true

- name: Check the Helm version
  ansible.builtin.command: helm version
  register: helm_check
  changed_when: false

- name: Print the Helm version
  ansible.builtin.debug:
    msg: "{{ helm_check.stdout_lines }}"
```

- `retries` / `delay` / `until` → retries the download up to 3 times, 5 seconds apart, to handle network blips
- `unarchive` → k9s releases ship as a `.tar.gz`; extract to `/tmp/` and pull the binary out
- `become: true` only on the task that installs the k9s binary → task-level become (see [become](../playbooks/become.md)): downloading and extracting happen in `/tmp`, only writing to `/usr/local/bin` needs root, so the whole play doesn't need `become: true`
- `changed_when: false` → just a version check, never reports `changed`
- Helm follows the same download → extract → install → check flow, with two differences:
  - **Version lookup**: Helm binaries live on `get.helm.sh`, which has no `/latest` URL like GitHub Releases does. So when `helm_version` is `latest`, the `uri` task asks the GitHub API for the newest release and the download URL uses its `tag_name`; a pinned version skips that task (`when`) and goes straight into the URL
  - **Tarball layout**: the archive unpacks into a `linux-amd64/` directory, so the binary is copied from `/tmp/linux-amd64/helm` rather than `/tmp/helm`

## Uninstall — [tasks/uninstall.yml](../../ansible/playbooks/roles/kube_tools/tasks/uninstall.yml)

```yaml
---
- name: Remove the k9s binary
  become: true
  ansible.builtin.file:
    path: "{{ k9s_bin_dir }}/k9s"
    state: absent

- name: Confirm the k9s binary is removed
  ansible.builtin.stat:
    path: "{{ k9s_bin_dir }}/k9s"
  register: k9s_binary

- name: Print the uninstall result
  ansible.builtin.debug:
    msg: "{{ 'k9s uninstalled successfully' if not k9s_binary.stat.exists else 'k9s is still present, uninstall failed' }}"

- name: Remove the Helm binary
  become: true
  ansible.builtin.file:
    path: "{{ helm_bin_dir }}/helm"
    state: absent

- name: Confirm the Helm binary is removed
  ansible.builtin.stat:
    path: "{{ helm_bin_dir }}/helm"
  register: helm_binary

- name: Print the Helm uninstall result
  ansible.builtin.debug:
    msg: "{{ 'Helm uninstalled successfully' if not helm_binary.stat.exists else 'Helm is still present, uninstall failed' }}"
```

- k9s and Helm are both standalone binaries — no systemd service, config file, or data directory — so removing each is a single `file: state: absent`; the closing `stat` + `debug` confirms it's gone, mirroring the k3s uninstall

## Run

```yaml
# playbooks/install_kube_tools.yml — kube_tools_state defaults to "present"
---
- name: Install kube tools
  hosts: control
  roles:
    - kube_tools
```

```yaml
# playbooks/uninstall_kube_tools.yml — overrides kube_tools_state to "absent"
---
- name: Uninstall kube tools
  hosts: control
  roles:
    - role: kube_tools
      kube_tools_state: absent
```

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_kube_tools.yml
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/uninstall_kube_tools.yml
```

- Unlike the k3s play, there's **no play-level `become: true`** here — only the tasks that write `/usr/local/bin/k9s` and `/usr/local/bin/helm` declare their own `become: true`; everything else runs as the normal user
- `hosts: control` → installs on the control node by default, so you can SSH in and use k9s to inspect the cluster; to run it locally against a remote cluster instead, pass `-i ansible/inventory/localhost.ini` — its group is also `control`, so no `hosts:` change is needed

## Verify

```bash
k9s version           # check version
k9s                   # launch the TUI, reads ~/.kube/config (the copy made by the k3s role)
helm version          # check Helm version
helm list -A          # list releases in all namespaces, also reads ~/.kube/config
```

Once inside, you'll see nodes, pods, and other resources; press `:q` or `Ctrl-C` to exit.

---

[← k3s Role](k3s.md) · [↑ Contents](../../README.md) · [kubeadm Role →](kubeadm.md)
