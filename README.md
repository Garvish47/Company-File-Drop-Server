# Company File Server

A self-hosted Linux file server built as a hands-on systems-administration project. It uses OpesSSH and SFTP for remote file access and standard Linux users, groups, filesystem ownership, and POSIX ACLs to enforce a small company's access policy. Bash utilities automate provisioning, operational checks, policy instection, chroot repair, and verified backups.

## Features demonstrated

- Linux user and group provisioning
- Directory ownership, unix permissions, SGID directory behaviour, and POSIX ACLs
- Restricted SFTP access using OpenSSH and a chroot directory
- Service inspection and operational auditing with `systemctl`
- Backup creation with ACL, extended-attribute, and numeric-owner preservation
- Archive verification, SHA-256 checksums, and retention management

## Architecture at a glance

The server runs on Ubuntu Server. `/srv/company-files` is the storage root and, for restricted SFTP users, the chroot boundary. The root of a chroot must be controlled by root and must not be writable by the restricted user. Access is granted inside the chroot through directory ownership and ACL entries.

- `company-files-bootstrap` installs required packages, provisions configured users and groups, created storage directories, applies the access policy and enables SSH.
- `company-files-admin` exposes operational commands for service status, disk usage, auditing, chroot repair, ACL-policy instection and backups.

The policy file centralises paths, service, group names, user mappings and backup retention.

see [Architecture](docs/architecture.md) for the component and data-flow details.

## Repository layout

```text
.
├── README.md
├── config/
│   └── policy.conf
├── scripts/
│   ├── company-files-bootstrap
│   └── company-files-admin
└── docs/
    ├── architecture.md
    ├── installation.md
    ├── access-control.md
    ├── operations.md
    └── backup-and-recovery.md
```

## Requirements

- Ubuntu Server (or a compatible Debian-based Linux distribution)
- A user with `sudo` access for installation and administration
- Network access to install packages
- OpenSSH server and the `acl` utilities (installed by the bootstrap script)
- A filesystem supporting POSIX ACLs

The commands and service names in this documentation assume the configuration shown in `config/policy.conf`

## Quick start

1. Review and edit `config/policy.conf` for the target environment. It is a shell-sourced configuration file, so only use a trusted copy.
2. Install the configuration and scripts at the paths extedted by the scripts:

    ```bash
    sudo install -D -o root -g root -m 0644 config/policy.conf /etc/company-files/policy.conf
    sudo install -D -o root -g root -m 0755 scripts/company-files-bootstrap /usr/local/sbin/company-files-setup
    sudo install -D -o root -g root -m 0755 scripts/company-files-admin /usr/local/libexec/company-files-admin
    ```

3. Run the bootstrap process:

    ```bash
    sudo company-files-setup
    ```

4. Configure and validate the OpenSSH SFTP restrictions for the intended user groups. The bootstrap process enables the SSH service; the SFTP `Match` rules are a separate SSH server configuration step. See [Installation](docs/installation.md).
5. Run the operational checks:

    ```bash
    sudo company-files-admin status
    sudo company-files-admin policy
    sudo company-files-admin audit
    ```

6. Create a backup after preparing the backup destination:
    ```bach
    sudo install -d -o root -g root -m 07-- /var/backups/company-files
    sudo company-files-admin backup
    ```

For the complete procedure, including user authentication and verification, follow [Installation](docs/installation.md).

## Administrative commands

| Command | Purpose | Privilege |
|---|---|---|
| `company-files-admin status` | Shows SSH service state and storage filesystem capacity | Ordinary user; elevated access may be needed for some details|
| `company-files-admin disk` | Shows filesystem capacity and, when run as root, per-directory usage | Ordinary user / root for extended report |
| `sudo company-files-admin audit` | Checks service state, chroot ownership/mode, and selected SSH settings | Root |
| `company-files-admin policy` | Checks expected group ownership and ACL entries | Run with sufficient access to inspect the directories |
| `sudo company-files-admin repair` | Restores the chroot root ownership and mode | Root |
| `sudo company-files-admin backup` | Creates, verifies, and applies retention to backups | Root |

See [Operations](docs/operations.md) for interpretation and troubleshooting.

## Access policy

The intended access model is:

| Directory | Administrators | Developers | Interns | Other users |
|---|---:|---:|---:|---:|
| `public/` | Read, write, traverse | Read, write, traverse | Read, traverse | No access |
| `projects/` | Read, write, traverse | Read, write, traverse | Read, traverse | No access |
| `private/` | Read, write, traverse | No access | No access | No access |

The exact ACL entries and the distinction between the owning group (`group::`) and named group entries are explained in [Access Control](docs/access-control.md).

## Backup model

Backups are compressed tar archives named `company-files-<timestamp>.tar.gz`. The archive operation preserves ACLs, extended attributes and numeric ownership. The script verifies that the archive can be listed, reports it's size and SHA-256 digest and refrains the configured number of newest matching archives.

A successful archive creation is not a substitute for a restore test. Follow [Backup and Recovery](docs/backup-and-recovery.md) to validate recovery procedures.

## Secutiry considerations

- Keep `/etc/company-files/policy.conf` root-owned and writable only by trusted administrators. It is sourced as Bash code.
- Keep the chroot root (`/srv/company-files` by default) owned by `root:root` and not writable by SFTP users.
- Use SSH public-kep authentication for restricted accounts and validate the effective SSH configuration with `sshd -T`.
- Do not commit private keys, credentials, production user data, backup archives, VM images, or host-specific secrets.
- Restrict network exposure with host and network firewalls; this project does not configure a complete perimeter-security policy.
- Treat the project as a learning implementation, not a hardened multi-tenant storage product.

## Learning outcomes

This project connects Linux fundamentals to day-to-day infrastructure operations: provisioning identities, expressing access policy with ACLs, constraining remote access, inspecting service health, autoating repeatable administration, and designing recoverable backups. It is intended to demonstrate practical systems work rather than merely a standalone Bash script.

## Documentation

- [Architecture](docs/architecture.md)
- [Installation](docs/installation.md)
- [Access control](docs/access-control.md)
- [Operations](docs/operations.md)
- [Backup and recovery](docs/backups-and-recovery.md)