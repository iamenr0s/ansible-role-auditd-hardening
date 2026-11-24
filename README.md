[![Molecule](https://github.com/iamenr0s/ansible-role-auditd-hardening/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-auditd-hardening/actions/workflows/molecule.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/auditd_hardening) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-auditd-hardening/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-auditd-hardening)

# Ansible Role: Auditd Hardening

> Standalone copy of `auditd_hardening` from the [iamenr0s.linux_hardening](https://github.com/iamenr0s/linux-hardening-collection) collection. Variables keep the collection-wide `linux_hardening_*` prefix.

Audit-first auditd for the Red Hat and Debian families (other families are **skipped** with a message and `linux_hardening_auditd_findings: []`). It owns `auditd.conf` and one managed rules file; it never reboots. Log shipping and rsyslog/journald belong to `logging_hardening`.

## What it checks
| Check | Remediation |
|---|---|
| `audit` (EL/Fedora) / `auditd` (Debian) installed | installed |
| `auditd.conf`: `max_log_file 32`, `max_log_file_action keep_logs`, `num_logs 5`, `space_left_action syslog`, `admin_space_left_action single` (override with `linux_hardening_auditd_conf`) | `lineinfile` per key, backed up; running daemon told to re-read (SIGHUP) |
| Managed rules file `/etc/audit/rules.d/99-linux-hardening.rules` (mode 0640) | written from a template |
| `auditd.service` enabled and active | enabled; started where the kernel accepts audit control |
| Rules loaded in the kernel (every `-k` key of the file present, merged `audit.rules` current) | `augenrules --load` |
| `-e 2` active, only with `linux_hardening_auditd_immutable: true` | loaded with the rules |
| `audit=1` on the kernel command line | **reported only** (bootloader) |

## The rules
Groups (drop with `linux_hardening_auditd_exclude_groups`): `identity` (passwd, shadow, group, gshadow, opasswd), `sudoers`, `logins` (lastlog, faillock), `session` (utmp, wtmp, btmp), `system-locale` (issue, hosts, hostname/domain syscalls), `time-change` (adjtimex, settimeofday, clock_settime, localtime), `mac-policy` (SELinux or AppArmor config), `modules` (insmod/rmmod/modprobe, init/finit/delete_module). Add your own with `linux_hardening_auditd_extra_rules`. Watches are only written for paths that exist (auditctl rejects a missing one and would abort the whole load); syscall rules cover `b64` and, on x86_64, `b32`. Not a verified CIS mapping.

## Safety
- **Containers are never touched.** The audit subsystem belongs to the host kernel, and `auditctl` works from a privileged container, so loading rules there would change the host. The role detects a container, edits files only, and reports that the runtime state is not managed.
- `admin_space_left_action: single` drops the machine to single-user mode when the disk is nearly full (auditing must not silently stop). Change it if that is not acceptable.
- **`-e 2` is a one-way door until the next reboot**: even this role cannot change the rules afterwards. A later change is written to the file and reported as `fixable: false` ("locked by -e 2: reboot to change"); it takes effect on the next boot. Rules that are wrong stay active until then, so switch it on last.
- Audit volume and disk use grow with the rules; size `max_log_file`/`num_logs` for your retention.

## Modes
`audit` (default) is read-only and sets `linux_hardening_auditd_findings` (`key, expected, actual, fixable`). `remediate` needs `linux_hardening_confirm_remediation: true`; after it only fixable findings fail the run.

## Role Variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `linux_hardening_mode` | str | `audit` | audit is read-only. Choices: `audit`, `remediate`. |
| `linux_hardening_confirm_remediation` | bool | `false` | Must be true before remediate changes anything. |
| `linux_hardening_backup` | bool | `true` | Back up auditd.conf and the rules file when they change. |
| `linux_hardening_verify` | bool | `true` | Re-audit after remediation. |
| `linux_hardening_fail_on_error` | bool | `true` | Fail the play when audit finds deviations. |
| `linux_hardening_auditd_enabled` | bool | `true` | Run the role. |
| `linux_hardening_auditd_conf_file` | path | `/etc/audit/auditd.conf` | auditd.conf to manage. |
| `linux_hardening_auditd_rules_file` | path | `/etc/audit/rules.d/99-linux-hardening.rules` | Managed rules file. |
| `linux_hardening_auditd_conf` | dict | `{}` | auditd.conf settings {key: value} over the built-in set. |
| `linux_hardening_auditd_exclude_groups` | list of str | `[]` | Rule groups to leave out. |
| `linux_hardening_auditd_extra_rules` | list of str | `[]` | Raw rule lines appended to the managed file. |
| `linux_hardening_auditd_immutable` | bool | `false` | Append -e 2: rules cannot change until reboot. |

**Idempotent:** yes. A second `remediate` run reports no changes (the Molecule `idempotence` step checks this).

**Check mode:** `remediate` with `--check --diff` shows what would change without writing anything; the post-remediation re-audit is skipped in check mode.

## Example
```yaml
- hosts: all
  become: true
  roles:
    - role: iamenr0s.auditd_hardening
      vars:
        linux_hardening_mode: remediate
        linux_hardening_confirm_remediation: true
        linux_hardening_auditd_conf: {num_logs: 10}
        linux_hardening_auditd_extra_rules: ["-w /etc/ssh/sshd_config -p wa -k sshd"]
```

## Verification
Container scenarios `default` (packages, `auditd.conf`, rules file, enable; no kernel part), `audit`, `refuse`. **VM scenario `vagrant`** (libvirt; `sg libvirt -c "molecule test -s vagrant"`; box `MOLECULE_VAGRANT_BOX`, default `almalinux/9`): rules are loaded and the daemon runs; a change to a watched file produces a real event (`ausearch -k system-locale`); the second run changes nothing; the rules are back after a reboot; `-e 2` makes the kernel immutable; a further rule is reported as non-fixable and not forced, then loaded by the next reboot with the kernel still immutable. Tested: AlmaLinux 9 and Debian 13 (VMs), AlmaLinux 9, Debian 13, Ubuntu 24.04, Fedora 44 (containers). CI runs the container scenarios only.

## Rollback
Restore the backups of `auditd.conf` and the rules file, `augenrules --load` (or reboot if `-e 2` is active), or remove the managed rules file.

## Known gaps
- Not tested on an Ubuntu or Fedora VM, EL8/EL10, or with `auditd` forwarding to a remote server (`audisp-remote`).
- No privileged-command (setuid/setgid binary) or file-deletion/permission-change rules: they need a per-host filesystem scan or produce heavy volume. Add them with `linux_hardening_auditd_extra_rules`.
- `audit=1` and `audit_backlog_limit` on the command line are reported, not set. CIS IDs not mapped.

CI & Release (maintainers)
----------------------------
A single workflow (`.github/workflows/molecule.yml`) runs lint and the full Molecule distro matrix on pushes to `main`, PRs, and `v*` tags. On `v*` tags, a `release` job publishes to Ansible Galaxy after all tests pass.

One-time setup:

```bash
# Galaxy publishing key (environment-scoped, get it from galaxy.ansible.com/ui/token)
gh secret set GALAXY_API_KEY --repo iamenr0s/ansible-role-auditd-hardening

# Code scanning notifications (Slack webhook URL; for Discord append /slack to the webhook URL)
gh secret set SECURITY_ALERT_WEBHOOK --repo iamenr0s/ansible-role-auditd-hardening
```

`.github/workflows/code-scanning-notify.yml` polls the code-scanning API every 6 hours and posts new or updated open alerts to that webhook (GitHub Actions cannot trigger on `code_scanning_alert` directly).

To release: tag a commit `vX.Y.Z` and push the tag — CI gates the Galaxy publish.

## Contributing

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the
local pipeline commands and pull request checklist. This project follows the
[Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

## Security

See [SECURITY.md](SECURITY.md) — GitHub private vulnerability reporting, no
public issues for security bugs.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and release notes.

## License

This project is licensed under the [MIT License](LICENSE).

## Author Information

Author: iamenr0s

Galaxy: `iamenr0s.ansible_role_auditd_hardening`
