# Testing

> Integration-test roles in a throwaway container with Molecule.

---

[Molecule](https://molecule.readthedocs.io/) is used to integration-test Ansible roles. It spins up a real container, runs the role against it, then asserts the expected outcome — catching regressions without touching a real VM.

Why test a role this way:

- **Real OS, throwaway cost** — the role runs against an actual Ubuntu container, not a mock, yet it's created and destroyed in seconds with no cloud VM to pay for or clean up.
- **Idempotence is checked for you** — `molecule test` runs the role **twice** and fails if the second run reports any change, automatically catching tasks that aren't idempotent (exactly what `changed_when:` and `creates:` are there to guard against).
- **Repeatable and CI-friendly** — every run starts from an identical clean container, so a pass proves the role works from scratch, not just on a machine you've already configured by hand.

Currently the **`kube_tools` role** has a molecule test suite under [`roles/kube_tools/molecule/default/`](../../ansible/playbooks/roles/kube_tools/molecule/default/).

## Prerequisites

Docker must be running. Install the Python dependencies:

```bash
pip install -r requirements.txt
```

## How the test works

Three files drive the lifecycle:

| File | Purpose |
| ---- | ------- |
| [`molecule.yml`](../../ansible/playbooks/roles/kube_tools/molecule/default/molecule.yml) | Docker driver + Ubuntu 22.04 platform (`geerlingguy/docker-ubuntu2204-ansible`) |
| [`converge.yml`](../../ansible/playbooks/roles/kube_tools/molecule/default/converge.yml) | Runs the `kube_tools` role against the container |
| [`verify.yml`](../../ansible/playbooks/roles/kube_tools/molecule/default/verify.yml) | Asserts the k9s and Helm binaries exist in `/usr/local/bin` and are executable |

## Run

```bash
cd ansible/playbooks/roles/kube_tools

molecule test       # full lifecycle: create → converge → idempotence → verify → destroy
molecule converge   # run the role only (keeps the container for manual inspection)
molecule verify     # run assertions only (container must already exist)
molecule destroy    # tear down the container
```

> `molecule test` is the standard CI command — it always starts from a clean container and destroys it afterwards. Use `converge` + `verify` separately when iterating locally.

---

[← Choosing k3s vs kubeadm](../roles/k3s-vs-kubeadm.md) · [↑ Contents](../../README.md) · [Running the Playbooks →](running-playbooks.md)
