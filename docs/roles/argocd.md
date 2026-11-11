# argocd Role

> Install Argo CD from its Helm chart, plus a CLI that matches the server version.

---

[Argo CD](https://argo-cd.readthedocs.io/) is a GitOps controller: it runs **inside the cluster** and keeps it in sync with manifests in a Git repo. This role installs it with the official Helm chart, plus the `argocd` CLI used to log in and manage it.

> **Why a separate role instead of adding it to `kube_tools`?** `kube_tools` holds general-purpose tools every cluster benefits from. Argo CD is a cluster workload that not every lab needs, and its CLI is useless without the server — so both live here, installed and removed together.

**Prerequisites** — run these first:

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_k3s.yml        # or install_kubeadm.yml
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_kube_tools.yml # provides Helm
ansible-galaxy collection install -r requirements.yml                                    # provides kubernetes.core
```

**[requirements.yml](../../requirements.yml)**

```yaml
---
# Ansible collections used by the playbooks
# Install: ansible-galaxy collection install -r requirements.yml
collections:
  - name: kubernetes.core # helm / helm_repository modules (argocd role); 6.4.0+ supports Helm v4
    version: ">=6.4.0,<7.0.0"
```

- This is the first role that needs a module outside `ansible.builtin`. `kubernetes.core` isn't bundled with `ansible-core`, so it's declared in `requirements.yml` and installed with `ansible-galaxy`, the collection equivalent of `pip install -r requirements.txt`
- The version range keeps installs reproducible: `>=6.4.0` is the first release with Helm v4 support (`kube_tools` installs the latest Helm), `<7.0.0` avoids surprise breaking changes. kubernetes.core 6.x needs **Ansible 2.16+**, newer than the rest of this repo

**[roles/argocd/defaults/main.yml](../../ansible/playbooks/roles/argocd/defaults/main.yml)**

```yaml
---
# Default variables for the argocd role; override in inventory or the playbook

# present = install, absent = uninstall
argocd_state: present

# namespace Argo CD is installed into
argocd_namespace: argocd

# argo-cd Helm chart version, pinned so cluster installs are reproducible
# The CLI version follows the chart's appVersion, so CLI and server always match
argocd_chart_version: 10.17.2

# binary install location for the argocd CLI
argocd_bin_dir: /usr/local/bin
```

- `argocd_chart_version` → **pinned**, unlike `k9s_version` / `helm_version`. Those are client tools you can freely replace; Argo CD is a service running in the cluster, so a re-run should never silently upgrade it. Bump it on purpose when you want to upgrade
- There's no CLI version variable — the CLI version is taken from the installed chart, so client and server can't drift apart

## Install — [tasks/install.yml](../../ansible/playbooks/roles/argocd/tasks/install.yml)

```yaml
---
- name: Add the Argo CD Helm repository
  kubernetes.core.helm_repository:
    name: argo
    repo_url: https://argoproj.github.io/argo-helm

- name: Install the Argo CD chart
  kubernetes.core.helm:
    name: argocd
    chart_ref: argo/argo-cd
    chart_version: "{{ argocd_chart_version }}"
    release_namespace: "{{ argocd_namespace }}"
    create_namespace: true
    update_repo_cache: true # so a newly pinned chart version can be found
    wait: true # don't finish until every Argo CD pod is ready
  register: argocd_release

- name: Install the Argo CD CLI matching the server version
  become: true # only writing to /usr/local/bin needs root
  ansible.builtin.get_url:
    url: "https://github.com/argoproj/argo-cd/releases/download/{{ argocd_release.status.app_version }}/argocd-linux-amd64"
    checksum: "sha256:https://github.com/argoproj/argo-cd/releases/download/{{ argocd_release.status.app_version }}/cli_checksums.txt"
    dest: "{{ argocd_bin_dir }}/argocd"
    mode: "0755"
    timeout: 120
  register: argocd_cli_download
  retries: 3
  delay: 5
  until: argocd_cli_download is succeeded

- name: Check the Argo CD CLI version
  ansible.builtin.command: argocd version --client # without --client it tries to reach the server
  register: argocd_check
  changed_when: false

- name: Print the Argo CD CLI version
  ansible.builtin.debug:
    msg: "{{ argocd_check.stdout_lines }}"

- name: Print how to log in
  ansible.builtin.debug:
    msg:
      - "Admin password: kubectl -n {{ argocd_namespace }} get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d"
      - "UI / API:       kubectl -n {{ argocd_namespace }} port-forward svc/argocd-server 8080:443"
```

- `kubernetes.core.helm` instead of `command: helm install` → the module checks the current release first and only installs/upgrades when needed, so it's idempotent without a hand-written `changed_when:`
- `helm_repository` + `chart_ref: argo/argo-cd` → registering the repo first is the module's recommended way; passing `chart_repo_url` directly can miss changes (see the module notes)
- `create_namespace` / `wait` → creates the `argocd` namespace if needed, and the task only finishes once every Argo CD pod is Ready
- `argocd_release.status.app_version` → the `helm` module returns the deployed app version (e.g. `v3.6.1`), which is reused to download the **matching** CLI
- `checksum: sha256:<url>` → `get_url` fetches the release's `cli_checksums.txt`, finds the line for `argocd-linux-amd64` and verifies the download. It also makes re-runs skip the download when the installed file already matches
- The CLI is a single binary, not a tarball, so `get_url` writes it straight into `/usr/local/bin` — no extract/copy steps like k9s and Helm
- Only that task has `become: true`; the Helm tasks run as the normal user so they use **their** `~/.kube/config`
- The last task prints the commands to fetch the admin password instead of reading it — keeping secrets out of Ansible output

## Uninstall — [tasks/uninstall.yml](../../ansible/playbooks/roles/argocd/tasks/uninstall.yml)

```yaml
---
- name: Uninstall the Argo CD chart
  kubernetes.core.helm:
    name: argocd
    release_namespace: "{{ argocd_namespace }}"
    release_state: absent
    wait: true

- name: Delete the Argo CD namespace
  # k3s's bundled kubectl ignores ~/.kube/config and reads root-only /etc/rancher/k3s/k3s.yaml, so point at it explicitly
  ansible.builtin.command: kubectl --kubeconfig {{ ansible_env.HOME }}/.kube/config delete namespace {{ argocd_namespace }} --ignore-not-found
  register: argocd_namespace_delete
  changed_when: "'deleted' in argocd_namespace_delete.stdout"

- name: Remove the Argo CD CLI
  become: true
  ansible.builtin.file:
    path: "{{ argocd_bin_dir }}/argocd"
    state: absent

- name: Confirm the Argo CD CLI is removed
  ansible.builtin.stat:
    path: "{{ argocd_bin_dir }}/argocd"
  register: argocd_binary

- name: Print the uninstall result
  ansible.builtin.debug:
    msg: "{{ 'Argo CD uninstalled successfully' if not argocd_binary.stat.exists else 'argocd CLI is still present, uninstall failed' }}"
```

- `release_state: absent` → `helm uninstall`, a no-op if the release is already gone
- The namespace is deleted with `kubectl` because `kubernetes.core.k8s` would need the Python `kubernetes` package on the target. `--ignore-not-found` plus `changed_when` on `deleted` keeps it idempotent
- `--kubeconfig` is explicit because k3s's bundled `kubectl` reads root-only `/etc/rancher/k3s/k3s.yaml` instead of `~/.kube/config` when `KUBECONFIG` isn't set
- The chart **keeps its CRDs** (`applications.argoproj.io`, etc.) on uninstall so your Application definitions aren't wiped by accident. To remove them too: `kubectl get crd -o name | grep argoproj.io | xargs kubectl delete`

## Run

```yaml
---
# Install Argo CD (Helm chart) and its matching CLI on the control group
# Needs a running cluster (install_k3s.yml or install_kubeadm.yml) and Helm (install_kube_tools.yml),
# plus the kubernetes.core collection: ansible-galaxy collection install -r requirements.yml
# Usage: ansible-playbook playbooks/install_argocd.yml
- name: Install Argo CD
  hosts: control
  roles:
    - argocd
```

```yaml
---
# Uninstall Argo CD and its CLI (the chart keeps its CRDs; see README to remove them)
# Usage: ansible-playbook playbooks/uninstall_argocd.yml
- name: Uninstall Argo CD
  hosts: control
  roles:
    - role: argocd
      argocd_state: absent
```

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_argocd.yml
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/uninstall_argocd.yml
```

- No play-level `become: true`, same as `kube_tools` — only the CLI install/removal escalates

## Verify

```bash
export KUBECONFIG=~/.kube/config                 # on k3s: otherwise kubectl reads root-only /etc/rancher/k3s/k3s.yaml
kubectl -n argocd get pods                       # all pods Running
argocd version --client                          # CLI matches the server version

# Log in (in one terminal, keep the port-forward running)
kubectl -n argocd port-forward svc/argocd-server 8080:443
# in another terminal
argocd login localhost:8080 --username admin --insecure \
  --password "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)"
```

The UI is at `https://localhost:8080` (self-signed certificate) with the same `admin` credentials. Change the password after the first login and delete `argocd-initial-admin-secret`, as Argo CD recommends.

---

[← kubeadm Role](kubeadm.md) · [↑ Contents](../../README.md) · [Choosing k3s vs kubeadm →](k3s-vs-kubeadm.md)
