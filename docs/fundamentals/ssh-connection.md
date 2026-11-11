# Switch to SSH Connection

> Set up key-based SSH and point the same playbook at a remote VM.

---

## Concept

In real-world scenarios, Ansible runs commands on remote machines over SSH.

```
laptop (Ansible)  ──SSH──▶  target machine
```

## SSH Key Setup

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

## Connecting to a Remote Azure VM

**Get the VM's public IP**

On the VM page in the Azure Portal, find the "Public IP address", e.g. `203.0.113.10`.

**Open port 22 (NSG rule)**

Azure blocks external connections by default. You need to add an inbound rule to the VM's "Network Security Group (NSG)"
allowing TCP 22 from your source, otherwise SSH won't be able to connect.

> Security tip: restrict the source to "My IP" — don't open port 22 to `0.0.0.0/0` (the whole world).

**Copy the public key to the Azure VM**

```bash
ssh-copy-id <YOUR_USER>@<YOUR_VM_IP>
# password required the first time (or use the key downloaded when creating the Azure VM), then no password afterwards
```

`ssh-copy-id` automatically writes the public key into the target machine's `~/.ssh/authorized_keys`.

Confirm you can connect:

```bash
ssh <YOUR_USER>@<YOUR_VM_IP>
# connects without a password = success
```

## [inventory/azure.ini](../../ansible/inventory/azure.ini)

The inventory used for the remote VM:

```ini
[control]
azure-vm ansible_host=<YOUR_VM_IP> ansible_user=<YOUR_USER> ansible_connection=ssh

[node]
```

- `[control]` → the machine that will later become the k3s control node
- `azure-vm` → a logical name for the host (used by `host_vars/azure-vm.yml`)
- `ansible_host=<YOUR_VM_IP>` → the actual IP Ansible connects to
- `ansible_user=<YOUR_USER>` → the login account on the Azure VM
- `[node]` → reserved for worker nodes to be added later (currently empty)

## Run (connecting to the remote VM)

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/hello.yml
```

> From Ansible's perspective, whether connecting locally or to a remote Azure VM, the playbook doesn't need to change at all —
> **just swap the IP and account in the inventory**. This is exactly the value Ansible provides.

> **Targeting specific groups:** this demo uses `hosts: all` so the one file runs against
> whatever inventory you pass. Real playbooks here (k3s, kubeadm) instead target a group —
> e.g. `hosts: control` — so they only touch the intended machines. Once you add workers to
> `[node]`, `all` would hit those too; `control` keeps it to the control node.
> Need more than one group? Combine them with `hosts: control:node`.

---

[← Hello World (Local Connection)](hello-world.md) · [↑ Contents](../../README.md) · [Ad-hoc Commands →](ad-hoc-commands.md)
