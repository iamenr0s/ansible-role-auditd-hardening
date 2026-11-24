# Changelog

## Unreleased
- Internal: the audit is one task per check, `auditd.conf` is parsed with one `regex_findall`, and the fail decision is two named tasks.
- Change: dropped the `linux_hardening_mode` assert that repeated `meta/argument_specs.yml` (which already rejects other values).
- Fix: `--check` no longer fails. The read-only `auditctl`/`augenrules --check` probes run in check mode, the unit is not touched after a simulated install, and the post-remediation re-audit is skipped.
- Fix: with `gather_facts: false` or a minimal subset a privileged container was treated as a host; the role now gathers the `virtual` facts it needs.
- Change: task names in sub-task files are prefixed (`audit |`, `facts |`, `configure |`).
- Docs: README adds a Role Variables table, the example uses the standalone role name with `become: true`, idempotency and check-mode notes.
- Test: the default Molecule scenario runs a `check` (dry-run remediate) step first.
- CI: every platform in `meta/main.yml` gates the release (no `continue-on-error` legs); lint runs the pinned pre-commit hooks; pre-commit adds gitleaks; actions are pinned by commit SHA; the release uses `ansible-galaxy role import` like the sibling roles.

## 1.0.0
- Split out of the iamenr0s.linux_hardening collection; no functional change.
