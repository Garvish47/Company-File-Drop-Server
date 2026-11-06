# Architecture

## 1. Purpose and scope

The Company File Server provides a shared directory tree for a small team. Users authenticate through OpenSSH and transfer files using SFTP. Linux account membership, directory ownership, standard mode bits, and POSIX ACLs determine what each user can access.

The implementation deliberately uses native Linux facilities rather than introducing a separate application server or database. This keeps the system small enough to inspect and makes the underlying operating-system behaviour explicit.

## 2. High-level components

```text
                 Administrators / Developers / Interns
                                |
                         SSH authentication
                                |
                         OpenSSH / SFTP
                                |
                    Chroot: /srv/company-files
                                |
                +---------------+---------------+
                |               |               |
             public/         projects/        private/
                |               |               |
               POSIX ACLs + ownership + mode bits

       company-files-bootstrap       company-files-admin
                 |                            |
        Provisioning and policy       Status / disk / audit /
        application                   policy / repair / backup
                 |                            |
                 +------ policy.conf ----------+
```

The diagram is conceptual: SSH/SFTP handles remote transport, while the filesystem enforces file and directory access. The Bash scripts automate system configuration and administration; they are not in the data-transfer path for each SFTP operation.

## 3. Configuration

`/etc/company-files/policy.conf` is the shared configuration source. It defines:

| Setting | Default value | Meaning |
|---|---|---|
| `storage` | `/srv/company-files` | Storage and chroot root |
| `storage_dirs` | `public projects private` | Managed subdirectories |
| `service` | `ssh` | systemd service name |
| `group_admins` | `admins` | Administrator group |
| `group_developers` | `developers` | Developer group |
| `group_interns` | `interns` | Intern group |
| `users_admins` | `alice` | Configured administrator account(s) |
| `users_developers` | `bob charlie` | Configured developer account(s) |
| `users_interns` | `david` | Configured intern account(s) |
| `backup_destination` | `/var/backups/company-files` | Backup archive directory |
| `backup_retention` | `7` | Number of newest matching archives to retain |

The configuration is sourced by Bash. It must therefore be treated as executable code and protected from modification by untrusted users.

## 4. Script responsibilities

### Bootstrap script

`company-files-bootstrap` is the initial provisioning entry point. It requires root privileges, reads the policy file, installs OpenSSH server and ACL utilities, creates configured groups and users, assigns supplementary group membership, creates the storage directories, applies ownership and ACL policy, and enables/restarts the SSH service.

The bootstrap script is intended to establish the baseline. It does not replace the separate SSH configuration needed to restrict selected accounts to SFTP or the operational verification performed after setup.

### Administrative script

`company-files-admin` is a command-dispatching Bash utility. Its subcommands separate read-only inspection from operations that modify system state:

- `status` and `disk` provide operational visibility.
- `audit` checks selected service, chroot, and SSH security conditions.
- `policy` inspects expected group ownership and ACL entries.
- `repair` restores the chroot root's required ownership and mode.
- `backup` creates and verifies a backup archive, calculates a checksum, and applies retention.

## 5. Storage layout

```text
/srv/company-files/       # Chroot root; root:root, not user-writable
├── public/               # Shared by admins and developers; readable by interns
├── projects/             # Writable by admins and developers; readable by interns
└── private/              # Restricted to admins
```

The chroot root has stricter ownership requirements than the directories inside it. OpenSSH requires each component of a `ChrootDirectory` path to be root-owned and not writable by group or others. Users receive their intended access below that boundary through ACLs.

## 6. Identity and access model

The configured users are associated with three supplementary groups:

| Role | Example account | Group |
|---|---|---|
| Administrator | `alice` | `admins` |
| Developer | `bob`, `charlie` | `developers` |
| Intern | `david` | `interns` |

Group membership provides the identity inputs used by filesystem ACL evaluation. The directory's owning group is represented by the ACL entry `group::`; separately named groups appear as entries such as `group:interns:r-x`.

See [Access Control](access-control.md) for the full permission matrix and ACL interpretation.

## 7. Operational dependencies

- `openssh-server` provides the SSH daemon and internal SFTP subsystem.
- `acl` provides `setfacl` and `getfacl` for managing and inspecting POSIX ACLs.
- `systemd` manages the SSH service.
- GNU core utilities and common system tools provide filesystem inspection, archive, checksum, and retention operations.
- `tar` is invoked with ACL, extended-attribute, and numeric-owner support for backups.

## 8. Design boundaries

This project focuses on a single Linux server and local filesystem permissions. It does not implement a web interface, distributed storage, high availability, centralized identity management, off-site backup replication, monitoring/alerting infrastructure, or a complete firewall policy. Those are possible future extensions, not implied features of the current implementation.
