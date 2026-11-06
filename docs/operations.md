# Operations and maintenance

The `company-files-admin` utility provides a small command-line interface for routine inspection and maintenance. Run it from an administrative shell or use its installed absolute path if it is not in `PATH`.

## 1. Command reference

| Command | Function | Notes |
|---|---|---|
| `company-files-admin status` | Reports SSH service state and storage filesystem capacity | Useful for a quick health check |
| `company-files-admin disk` | Reports filesystem capacity; root receives a per-directory usage breakdown | `du` output is limited to the storage filesystem |
| `sudo company-files-admin audit` | Checks service state, storage presence, chroot owner/mode, and selected effective SSH settings | Requires root |
| `company-files-admin policy` | Checks expected directory group ownership and ACL entries | Needs permission to inspect the storage tree |
| `sudo company-files-admin repair` | Restores chroot root ownership and mode | Changes the storage root only; it is not a full ACL repair |
| `sudo company-files-admin backup` | Creates and verifies an archive, reports checksum, and applies retention | Requires root and a prepared destination |

## 2. Service status

```bash
systemctl status ssh --no-pager
systemctl is-active ssh
systemctl is-enabled ssh
```

The `status` subcommand combines service state with a filesystem capacity report:

```bash
company-files-admin status
```

A service being active means the daemon is running; it does not prove that a particular user can authenticate or that SFTP restrictions are correct.

## 3. Storage and disk usage

```bash
company-files-admin disk
```

The basic report uses `df` to show filesystem capacity and available space. When run as root, the utility also uses `du` to show usage by immediate subdirectory. Use `df` to understand filesystem-level capacity and `du` to identify which directories account for usage; the two commands answer different questions.

## 4. Security audit

Run the audit with root privileges:

```bash
sudo company-files-admin audit
```

The audit checks selected operational and security conditions, including:

- SSH service is active and enabled
- Configured storage path exists
- Chroot root ownership and mode match the expected values
- Effective SSH configuration for the audited account includes the expected SFTP command and password-authentication setting
- Configured role groups can be resolved

The audit is intentionally a set of explicit checks, not a comprehensive security scanner. A healthy result should be combined with user-level SFTP tests and review of the SSH server configuration.

## 5. ACL policy inspection

```bash
company-files-admin policy
```

This checks directory group ownership and expected ACL entries. If it reports a mismatch, inspect the full state before changing anything:

```bash
stat -c '%A %a %U:%G %n' /srv/company-files/{public,projects,private}
getfacl -p /srv/company-files/{public,projects,private}
namei -l /srv/company-files/projects
```

Remember that the owning group appears as `group::`, while additional groups appear as named entries. Also inspect the ACL mask and any default ACL entries.

## 6. Chroot repair

```bash
sudo company-files-admin repair
```

The repair operation restores the storage root's ownership and mode to the values required by the project's chroot design (`root:root`, mode `755`). It does not rebuild child directory ownership, access ACLs, default ACLs, user memberships, or SSH configuration. Re-run the audit after repair and inspect child directory policy separately.

## 7. Safe operational habits

- Keep an active administrative console/session while changing SSH configuration.
- Validate SSH syntax with `sshd -t` before reloading the daemon.
- Test access as the actual account after changing memberships or ACLs.
- Inspect the current ACL and ownership state before applying corrective commands.
- Avoid broad recursive `chmod` or `chown` commands on the storage tree; they can destroy carefully designed ACLs and ownership relationships.
- Keep backup archives outside the live storage tree.
- Perform periodic restore tests; archive creation alone does not prove recoverability.

## 8. Troubleshooting quick reference

| Symptom | First checks |
|---|---|
| SSH service is inactive | `systemctl status ssh`; inspect `journalctl -u ssh` |
| SSH configuration reload fails | `sudo sshd -t`; inspect the relevant `Match` block and included files |
| SFTP login fails | Check authentication method, account state, effective `sshd -T -C` output, and SSH logs |
| User can list but cannot upload | Check directory write and execute permissions, group membership, ACL mask, and default ACL behaviour |
| User cannot traverse a path | Inspect execute/search permission on every parent directory with `namei -l` |
| Policy command reports a mismatch | Compare `stat` and `getfacl` output with the documented matrix before modifying permissions |
| Backup command cannot create an archive | Check that the destination exists, is root-writable, and has sufficient free space |

Useful logs:

```bash
sudo journalctl -u ssh --since '30 minutes ago'
sudo journalctl -xe
```
