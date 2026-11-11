# Choosing k3s vs kubeadm

> Both give you a single-node cluster — pick based on what you want to practice.

---

Both roles give you a single-node Kubernetes control plane on the same VM. Pick based on what you're practicing:

|                     | **k3s**                                       | **kubeadm**                                   |
| ------------------- | --------------------------------------------- | --------------------------------------------- |
| **Best for**        | Fast labs, edge/IoT, "just give me a cluster" | Learning how upstream Kubernetes is assembled |
| **Install effort**  | One script, one task file                     | Multi-step pipeline (runtime, CNI, init)      |
| **Components**      | Bundled (containerd, CNI, etc. built in)      | You wire up containerd + flannel yourself     |
| **Startup time**    | Seconds                                       | Minutes (pulls control-plane images)          |
| **Footprint**       | Lightweight (~512MB RAM)                      | Heavier                                       |
| **Closest to prod** | Conformant but opinionated                    | Vanilla upstream Kubernetes                   |

- **Just want a working cluster to run k9s against?** Use **k3s**.
- **Want to understand kubelet, containerd, CNI, and `kubeadm init`?** Use **kubeadm**.
- **`kube_tools` (k9s, Helm) works with either** — both tools just read `~/.kube/config`, which both roles set up.

> Don't install both on the same node — they fight over ports and the container runtime. Uninstall one before installing the other.

---

[← argocd Role](argocd.md) · [↑ Contents](../../README.md) · [Testing →](../operating/testing.md)
