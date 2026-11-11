# Next Steps

> Where to take the lab after this guide.

---

- **Add a worker node**: populate the `[node]` group in the inventory and extend the roles to join workers.
- **Pin versions**: set `k9s_version` / `helm_version` / kubeadm Kubernetes version for reproducible builds.
- **Try GitOps**: point an Argo CD `Application` at a Git repo of manifests and watch it sync.
- **Deploy a workload**: `kubectl create deployment web --image=nginx` and explore it in `k9s`.
- **Go multi-host**: point the inventory at several VMs and run the same playbook unchanged.
- **Add molecule tests**: only `kube_tools` has a test suite today — `k3s` and `kubeadm` are good candidates to cover next.

---

[← FAQ](faq.md) · [↑ Contents](../../README.md) · [Contributing →](contributing.md)
