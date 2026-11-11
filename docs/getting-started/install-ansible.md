# Install Ansible

> Install Ansible on the control machine and set up the `ansible.cfg` every later command relies on.

---

```bash
sudo apt update && sudo apt install -y ansible
```

Check the version:

```bash
ansible --version
# ansible [core 2.14+]
```

---

## [ansible.cfg](../../ansible/ansible.cfg)

Placed in the `ansible/` directory and automatically applied when running `ansible-playbook` from there:

```ini
[defaults]
inventory = inventory/azure.ini   # default inventory, so the -i flag can be omitted
host_key_checking = False         # avoids getting stuck on the host key prompt when SSH-ing into a new machine for the first time
```

> Ansible only auto-loads `ansible.cfg` if you run `ansible-playbook` from the **same directory it's in** (`ansible/`). From the project root, it's ignored, so add `-i ansible/inventory/azure.ini` explicitly.

---

[← Architecture Overview](architecture.md) · [↑ Contents](../../README.md) · [Hello World (Local Connection) →](../fundamentals/hello-world.md)
