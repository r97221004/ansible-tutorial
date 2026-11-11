# Contributing

> How to propose fixes and improvements.

---

Contributions and corrections are welcome.

1. Fork the repo and create a branch: `git checkout -b improve-xyz`.
2. Keep changes focused; match the existing style (English, bold-keyword bullet lists).
3. Run `ansible-lint` from the repo root before opening a PR — it must pass with no failures. Lint rules are configured in [`.ansible-lint`](../../.ansible-lint) and [`.yamllint`](../../.yamllint).
4. If you edited the `kube_tools` role, run `molecule test` from `ansible/playbooks/roles/kube_tools/` as well — CI runs both checks on every PR via [`.github/workflows/lint.yml`](../../.github/workflows/lint.yml).
5. Open a pull request describing what changed and why.

Found a typo or unclear explanation? Open an issue — small fixes help every future reader.

---

[← Next Steps](next-steps.md) · [↑ Contents](../../README.md)
