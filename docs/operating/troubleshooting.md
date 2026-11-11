# Troubleshooting

> Symptom → cause → fix for the problems you're most likely to hit.

---

<details>
<summary><strong>SSH: <code>UNREACHABLE</code> / Permission denied (publickey)</strong></summary>

- Confirm you can SSH manually first: `ssh <user>@<ip>`.
- Make sure your public key is on the target: `ssh-copy-id <user>@<ip>`.
- Check the inventory line matches the real user/IP: `ansible_user=<user>`.
- On a brand-new host, `host_key_checking = False` in `ansible.cfg` avoids the fingerprint prompt.
- On Azure, confirm the **NSG inbound rule** allows TCP 22 from your IP.

</details>

<details>
<summary><strong><code>become</code> / sudo password prompts or failures</strong></summary>

- The login user needs **passwordless sudo**, or set `ansible_become_pass` (ideally via Vault).
- Quick test: `ansible control -m command -a "id" --become`.
- Interactive fallback: add `--ask-become-pass` (`-K`) to the command.

</details>

<details>
<summary><strong>"Attempting to decrypt but no vault secrets found"</strong></summary>

- You have a `group_vars/control/vault.yml` that is Ansible Vault-encrypted. Add `--ask-vault-pass` to your command, or remove the file if you no longer need it.
- `vault.yml` is not included in this repo — if you created one locally and forgot the password, delete it and re-create it with `ansible-vault create`.
- View encrypted contents without decrypting on disk: `ansible-vault view inventory/group_vars/control/vault.yml`.

</details>

<details>
<summary><strong>kubeconfig missing / <code>kubectl</code> can't connect</strong></summary>

- k3s writes `/etc/rancher/k3s/k3s.yaml`; the role copies it to `~/.kube/config`. If `kubectl`/`k9s` can't connect, confirm that copy exists and is owned by your user.
- For k3s you can always use the bundled client: `k3s kubectl get nodes`.
- For kubeadm, kubeconfig comes from `/etc/kubernetes/admin.conf`.

</details>

<details>
<summary><strong>Node stuck in <code>NotReady</code></strong></summary>

- This is almost always the CNI. For kubeadm, check the flannel pods: `kubectl get pods -n kube-flannel` (or `kube-system`).
- Give it a few seconds after install and re-check `kubectl get nodes`.
- If you have multiple NICs, set `flannel_iface` to the correct interface.

</details>

---

[← Running the Playbooks](running-playbooks.md) · [↑ Contents](../../README.md) · [FAQ →](faq.md)
