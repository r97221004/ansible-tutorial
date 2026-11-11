# FAQ

> Short answers to common questions.

---

**Do I need a cloud VM?** No — any reachable Linux host works (a local VM, WSL, a Raspberry Pi). The examples use Azure, but only the inventory IP/user changes.

**Can I run everything locally?** The teaching playbooks (`hello.yml`, `demo_variables.yml`) run against any host. For the Kubernetes roles, use a Linux VM you don't mind wiping.

**Why both k3s and kubeadm?** They teach different things — see [Choosing k3s vs kubeadm](../roles/k3s-vs-kubeadm.md).

**Is it safe to re-run a playbook?** Yes, that's the point — they're idempotent. See [Checking idempotency](running-playbooks.md#checking-idempotency).

**How do I target a different host?** Edit `ansible/inventory/azure.ini` (or pass a different `-i` inventory). No playbook changes needed.

---

[← Troubleshooting](troubleshooting.md) · [↑ Contents](../../README.md) · [Next Steps →](next-steps.md)
