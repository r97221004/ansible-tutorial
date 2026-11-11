# Architecture Overview

> How the control machine, inventory, playbooks, and target VM fit together — and where everything lives in this repo.

---

```mermaid
---
config:
  look: handDrawn
  theme: dark
  themeVariables:
    fontFamily: '"Comic Sans MS", "Comic Sans", "Segoe Print", "Bradley Hand", cursive'
    darkMode: true
    background: '#11151c'
    primaryColor: '#0B3B2E'
    primaryBorderColor: '#22C55E'
    primaryTextColor: '#F8FAFC'
    lineColor: '#F8FAFC'
    clusterBkg: '#11151c'
    clusterBorder: '#94A3B8'
---
flowchart LR
    subgraph control["💻 Control machine — your laptop"]
        direction TB
        inv["📋 inventory<br/>hosts &amp; connection vars"]
        pb["📜 playbooks + roles<br/>k3s · kubeadm · kube_tools"]
        ans["⚙️ ansible-playbook"]
    end

    subgraph target["🐧 Target VM — Ubuntu 22.04"]
        k8s["☸️ k3s OR kubeadm<br/>single-node cluster"]
    end

    inv --> ans
    pb --> ans
    ans ==>|"🔐 SSH · sudo where needed"| k8s

    classDef ctrl fill:#0B3B2E,stroke:#22C55E,stroke-width:2px,color:#F8FAFC;
    classDef eng fill:#7C4A03,stroke:#F59E0B,stroke-width:3px,color:#F8FAFC;
    classDef tgt fill:#0B3B2E,stroke:#22C55E,stroke-width:2px,color:#F8FAFC;
    class inv,pb ctrl;
    class ans eng;
    class k8s tgt;
    linkStyle 2 stroke:#F59E0B,stroke-width:3px;
```

- You run `ansible-playbook` on the **control machine**; no agent is installed on the target — Ansible just needs SSH access.
- Ansible reads the **inventory** (which hosts, how to connect) and the **playbooks + roles** (what to do) independently, then connects over **SSH** and applies the tasks on the **target VM**.
- The same playbook works for one VM or many — you only change the inventory.

## Repository Map

```
ansible/
├── ansible.cfg                 # default inventory + SSH behavior
├── inventory/
│   ├── localhost.ini           # [control] group pointing at your own machine
│   ├── azure.ini               # [control]/[node] groups for remote VMs  ← edit this
│   ├── group_vars/control/     # vars.yml (plain) + vault.yml (encrypted)
│   └── host_vars/              # per-host variables
└── playbooks/
    ├── hello.yml               # first playbook (debug output)
    ├── demo_variables.yml      # variables + Vault demo
    ├── install_*.yml           # install entrypoints (k3s, kubeadm, kube_tools, argocd)
    ├── uninstall_*.yml         # matching uninstall entrypoints
    └── roles/                  # reusable logic
        ├── k3s/                # lightweight single-node Kubernetes
        ├── kubeadm/            # upstream Kubernetes (containerd + flannel)
        ├── kube_tools/         # k9s terminal UI + Helm
        └── argocd/             # Argo CD (Helm chart) + matching CLI
requirements.yml                # Ansible collections (kubernetes.core) for the argocd role
```

> Start in `inventory/azure.ini` (set your VM), then run a playbook from `playbooks/`.

---

[↑ Contents](../../README.md) · [Install Ansible →](install-ansible.md)
