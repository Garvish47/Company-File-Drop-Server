# Installation and configuration

This guide describes a clean installation on an Ubuntu Server VM. Commands that change users, groups, ownership, ACLs, or service configuration require root privileges; use `sudo` where shown.

## 1. Prepare the server

Update package metadata and install basic tools if they are not already present:

```bash
sudo apt-get update
sudo apt-get install -y openssh-server acl
```

The bootstrap script also installs `openssh-server` and `acl`, so this step is useful as an explicit prerequisite check rather than a separate requirement.

Confirm that the server has a working network interface and that you can access its console before changing SSH configuration. Keep an existing administrative session open while validating SSH changes so a configuration mistake does not lock you out.

## 2. Install the project files

From the repository root, install the policy file and scripts at the paths expected by the scripts:

```bash
sudo install -d -o root -g root -m 0755 /etc/company-files
sudo install -D -o root -g root -m 0644 config/policy.conf /etc/company-files/policy.conf
sudo install -D -o root -g root -m 0755 scripts/company-files-bootstrap /usr/local/sbin/company-files-setup
sudo install -D -o root -g root -m 0755 scripts/company-files-admin /usr/local/libexec/company-files-admin
```

Verify the installed files:

```bash
sudo ls -l /etc/company-files/policy.conf \
  /usr/local/sbin/company-files-setup \
  /usr/local/libexec/company-files-admin
```

The configuration is sourced by Bash. Keep it owned by root and prevent ordinary users from editing it. If you change installation paths, ensure the command paths and any systemd unit referring to the administrative script are updated consistently.

## 3. Review the policy configuration

Edit the configuration before bootstrapping:

```bash
sudoedit /etc/company-files/policy.conf
```

Review at least:

- Storage root and managed directory names
- SSH service name for the target distribution
- Group and user names
- Backup destination and retention count

The default mapping is `alice` in `admins`, `bob` and `charlie` in `developers`, and `david` in `interns`. The script creates missing users and groups and adds users to their configured supplementary groups. Existing accounts are not automatically removed if you later remove them from the configuration.

## 4. Run bootstrap

```bash
sudo company-files-setup
```

The bootstrap operation installs dependencies, creates groups and users, assigns supplementary group membership, creates the storage tree, applies directory ownership and ACLs, and enables/restarts the SSH service.

Review the output for errors. Then inspect the resulting identity and directory state:

```bash
getent group admins developers interns
id alice
id bob
id charlie
id david

sudo namei -l /srv/company-files
sudo getfacl -p /srv/company-files/public \
  /srv/company-files/projects \
  /srv/company-files/private
```

## 5. Configure restricted SFTP access

The bootstrap process enables the SSH service. SFTP-only restrictions are configured separately in the OpenSSH server configuration, typically in `/etc/ssh/sshd_config` or a file included from it.

A representative configuration pattern is shown below. Apply it only to the intended account or group set; the `Match` selector must match the users you actually intend to restrict:

```sshconfig
Match Group developers,interns
    ChrootDirectory /srv/company-files
    ForceCommand internal-sftp
    X11Forwarding no
    AllowTcpForwarding no
    PasswordAuthentication no
```

This is an example, not a substitute for checking the exact policy you have chosen. In particular, confirm whether all developers and interns should be SFTP-only, and whether password authentication is disabled globally or only for matched users. Configure SSH public keys for the restricted accounts if password authentication is disabled.

Validate syntax before applying changes:

```bash
sudo sshd -t
```

Inspect the effective configuration for a representative account (replace the user and connection values as appropriate):

```bash
sudo sshd -T -C user=bob,host=localhost,addr=127.0.0.1 \
  | grep -E '^(forcecommand|chrootdirectory|passwordauthentication) '
```

Reload or restart SSH only after validation:

```bash
sudo systemctl reload ssh
sudo systemctl status ssh --no-pager
```

Keep a working console or existing administrator session available while testing. Test both the allowed SFTP workflow and the expected denial of an interactive shell. A successful SSH service status alone does not prove that the account-level restrictions are correct.

## 6. Prepare the backup destination

Create a root-controlled directory for backup archives:

```bash
sudo install -d -o root -g root -m 0700 /var/backups/company-files
```

The backup command expects this destination to exist and be writable by root. Confirm it:

```bash
sudo stat -c '%U:%G %a %n' /var/backups/company-files
```

## 7. Verify the installation

Run the administrative checks:

```bash
sudo company-files-admin status
company-files-admin disk
sudo company-files-admin policy
sudo company-files-admin audit
```

Test access as the actual users rather than relying only on ACL text. For example, use `sudo -u bob -- ...` and `sudo -u david -- ...` to test operations against the intended directories. Validate read, write, and traversal separately; directory permissions have different meanings from file permissions.

Finally, create a test backup and perform a restore test in a safe staging location. See [Backup and Recovery](backup-and-recovery.md).

## 8. Operational handover checklist

- [ ] Policy file is root-owned and not writable by ordinary users.
- [ ] Expected users and supplementary groups exist.
- [ ] Storage root is root-owned and not user-writable.
- [ ] Directory ownership and ACL entries match the access matrix.
- [ ] SSH configuration passes `sshd -t`.
- [ ] Effective SSH configuration matches the intended SFTP restrictions.
- [ ] SFTP upload/download tests pass for permitted users.
- [ ] Interactive shell access is denied where intended.
- [ ] Audit and policy commands report the expected state.
- [ ] Backup creation, verification, and restoration have been tested.
