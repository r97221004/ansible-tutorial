# Ansible Vault

> Keep secrets encrypted in the repo and decrypt them only at run time.

---

Encrypts sensitive data (passwords, tokens, private keys) so it can be safely committed to version control.

## Basic commands

| Command                        | Description                              |
| ------------------------------ | ---------------------------------------- |
| `ansible-vault create <file>`  | create a new encrypted file              |
| `ansible-vault edit <file>`    | edit an encrypted file in place          |
| `ansible-vault encrypt <file>` | encrypt an existing plaintext file       |
| `ansible-vault decrypt <file>` | decrypt back to plaintext                |
| `ansible-vault view <file>`    | view contents without decrypting on disk |

## Example

```bash
ansible-vault create inventory/group_vars/control/vault.yml
```

```yaml
# inventory/group_vars/control/vault.yml (encrypted on disk)
ansible_become_pass: supersecret
```

## Running playbooks with Vault

You only need a vault password flag when the run **loads at least one Vault-encrypted file** (e.g. an encrypted `group_vars`/`host_vars` file or one pulled in via `vars_files`). If nothing encrypted is loaded, omit it — passing `--ask-vault-pass` with no encrypted files errors out with `Attempting to decrypt but no vault secrets found`.

- `--ask-vault-pass` → prompt for the vault password interactively
- `--vault-password-file <path>` → read the password from a file (the file itself should not be committed)

```bash
ansible-playbook -i ansible/inventory/azure.ini <your-playbook>.yml --ask-vault-pass
```

---

[← Error Handling](error-handling.md) · [↑ Contents](../../README.md) · [Ansible Roles →](../roles/roles.md)
