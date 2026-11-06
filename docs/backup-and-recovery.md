# Backup and recovery

The administrative backup command creates a compressed archive of the storage tree, verifies that the archive can be read, calculates a SHA-256 checksum, and removes older matching archives according to the configured retention count.

## 1. Backup configuration

The policy file defines:

```bash
backup_destination="/var/backups/company-files"
backup_retention=7
```

The destination should be outside `/srv/company-files` so that backups are not included in subsequent backups of the live storage tree. Create it with restrictive permissions:

```bash
sudo install -d -o root -g root -m 0700 /var/backups/company-files
sudo stat -c '%U:%G %a %n' /var/backups/company-files
```

The backup command requires root privileges and a valid storage directory.

## 2. Create a backup

```bash
sudo company-files-admin backup
```

The archive name follows this pattern:

```text
company-files-YYYYMMDD-HHMMSS.tar.gz
```

The tar operation uses:

- `--acls` to store POSIX ACL information
- `--xattrs` to store extended attributes
- `--numeric-owner` to record numeric user and group IDs
- gzip compression to produce a `.tar.gz` archive

After creation, the script lists the archive contents as a basic integrity check, reports its size, calculates a SHA-256 checksum, and applies retention.

## 3. Verify an archive manually

List the archive contents:

```bash
sudo tar -tzf /var/backups/company-files/company-files-YYYYMMDD-HHMMSS.tar.gz
```

Check the checksum:

```bash
sudo sha256sum /var/backups/company-files/company-files-YYYYMMDD-HHMMSS.tar.gz
```

The checksum printed at backup time can be compared with a later checksum to detect changes to the archive. Store a copy of the checksum somewhere separate from the archive if you need an independent integrity record.

Archive listing confirms that gzip/tar can read the archive structure; it does not prove that every file can be restored correctly. A restore test is still required.

## 4. Retention behaviour

The configured retention value is `7`. The script finds matching archive files in the destination, sorts them by modification time (newest first), and removes entries beyond the configured count.

Retention is based on file modification time, not on parsing the timestamp embedded in the filename. Keep the destination dedicated to this project's matching backup files so unrelated files are not affected.

## 5. Restore to a staging directory

A restore should first be tested in an isolated location. The archive contains the `company-files/` directory, so extracting it under a staging directory creates a nested `company-files` tree.

```bash
ARCHIVE=/var/backups/company-files/company-files-YYYYMMDD-HHMMSS.tar.gz
STAGING=/srv/restore-staging

sudo install -d -o root -g root -m 0700 "$STAGING"
sudo tar --acls --xattrs --numeric-owner \
  -xzf "$ARCHIVE" -C "$STAGING"
```

Inspect the restored tree before using it:

```bash
sudo namei -l "$STAGING/company-files"
sudo getfacl -p "$STAGING/company-files/public" \
  "$STAGING/company-files/projects" \
  "$STAGING/company-files/private"
sudo stat -c '%A %a %U:%G %n' "$STAGING/company-files"
```

Compare the restored ownership, ACLs, and directory structure with the expected policy. Then perform controlled access tests as the intended users.

## 6. Restore into service

For an actual recovery, plan a maintenance window and prevent writes while the live tree is replaced. Keep a separate copy of the current tree until the restored version has been validated.

A cautious high-level sequence is:

1. Notify users and stop or restrict SFTP writes.
2. Preserve the current `/srv/company-files` tree under a clearly named recovery/rollback path on the same filesystem.
3. Move the validated staged `company-files` directory into `/srv/company-files`.
4. Confirm the chroot root is `root:root` with mode `755`.
5. Re-run the policy and security audit commands.
6. Test SFTP access for each role before reopening access.
7. Retain the previous tree until recovery is confirmed.

Do not run a blind extraction over a live tree: it can leave stale files that were not present in the backup and can make rollback difficult. Confirm the target path and the contents of the archive before any replacement operation.

## 7. Recovery validation checklist

- [ ] Archive can be listed with `tar -tzf`.
- [ ] SHA-256 checksum matches the recorded value, if available.
- [ ] Restore completes into an isolated staging directory.
- [ ] Expected files and directories are present.
- [ ] Numeric ownership, ACLs, and extended attributes are preserved as expected.
- [ ] Chroot root is root-owned and not writable by group or others.
- [ ] Policy inspection passes.
- [ ] SFTP access tests pass for permitted users and fail for prohibited operations.
- [ ] Rollback copy is retained until recovery is accepted.

## 8. Backup limitations and next steps

The current workflow stores backups on the same server unless the operator copies them elsewhere. A disk failure, host compromise, or accidental deletion could therefore affect both live data and local backups. For a more resilient deployment, add off-host or off-site replication, encryption at rest where appropriate, monitoring of backup job results, and a recurring restore test. These are operational extensions rather than features of the current scripts.
