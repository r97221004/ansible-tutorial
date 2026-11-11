# Running the Playbooks

> A quick reference for the full install → verify → tear-down lifecycle.

---

A quick reference for the full lifecycle. All commands assume `ansible/inventory/azure.ini` points at your VM.

## Typical workflow

```bash
# 1. Confirm connectivity
ansible control -i ansible/inventory/azure.ini -m ping

# 2. Install a cluster (pick one)
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_k3s.yml
# or
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_kubeadm.yml

# 3. Install tooling
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_kube_tools.yml

# 4. (Optional) Install Argo CD — needs Helm from step 3
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_argocd.yml

# 5. Tear down when done (reverse order: Argo CD, tooling, then the cluster)
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/uninstall_argocd.yml
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/uninstall_kube_tools.yml
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/uninstall_k3s.yml
# or, if you installed kubeadm
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/uninstall_kubeadm.yml
```

> With `ansible.cfg`'s default inventory, you can drop `-i ansible/inventory/azure.ini` **when running from inside the `ansible/` directory**.

## Checking idempotency

The hallmark of a good playbook: run it twice, the second run changes nothing.

```bash
ansible-playbook ansible/playbooks/install_k3s.yml      # first run:  changed=5
ansible-playbook ansible/playbooks/install_k3s.yml      # second run: changed=0
```

Look at the `PLAY RECAP` — a second run should report `changed=0`. Any task still reporting `changed` on every run is a candidate for a `creates:`/`changed_when:` fix (see [Idempotency](../playbooks/idempotency.md)).

---

[← Testing](testing.md) · [↑ Contents](../../README.md) · [Troubleshooting →](troubleshooting.md)
