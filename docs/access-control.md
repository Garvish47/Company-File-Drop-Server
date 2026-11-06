# Access control

This project uses Linux users and supplementary groups together with directory ownership, standard mode bits, and POSIX ACLs. The objective is to grant access by role while keeping the storage root suitable for OpenSSH chrooting.

## 1. Users and groups

| Account | Role | Supplementary group |
|---|---|---|
| `alice` | Administrator | `admins` |
| `bob` | Developer | `developers` |
| `charlie` | Developer | `developers` |
| `david` | Intern | `interns` |

The actual names are configurable in `/etc/company-files/policy.conf`. Group membership can be inspected with:

```bash
id bob
getent group developers
```

A user may need to start a new login session for updated supplementary group membership to be reflected in that session.

## 2. Intended directory policy

| Directory | Owning group | `admins` | `developers` | `interns` | Others |
|---|---|---:|---:|---:|---:|
| `public/` | `admins` | `rwx` | `rwx` | `r-x` | `---` |
| `projects/` | `developers` | `rwx` | `rwx` | `r-x` | `---` |
| `private/` | `admins` | `rwx` | `---` | `---` | `---` |

Here, `rwx` on a directory means the role can list entries, create/remove entries where other filesystem conditions permit, and traverse the directory. `r-x` permits listing and traversal but not creating or removing directory entries. `---` denies access through that directory.

The chroot root itself is different from these child directories: `/srv/company-files` is expected to be owned by `root:root` and have mode `755` so it satisfies the OpenSSH chroot ownership requirement. Access is granted within the child directories.

## 3. How Linux directory permissions work

For directories:

- **Read (`r`)** allows listing directory entries (filenames).
- **Write (`w`)** allows modifying directory entries, such as creating, renaming, and deleting entries, subject to the required search permission and other controls.
- **Execute/search (`x`)** allows traversing the directory and accessing entries by name.

A user generally needs `x` on every directory component in a path to reach a file. A directory's `r` bit alone does not grant traversal. File permissions and directory permissions are distinct: permission to list a directory does not automatically grant permission to read the contents of every file in it.

## 4. POSIX ACL structure

Inspect ACLs with:

```bash
getfacl -p /srv/company-files/public
getfacl -p /srv/company-files/projects
getfacl -p /srv/company-files/private
```

A directory ACL can include:

- `user::` — permissions for the owning user
- `group::` — permissions for the directory's owning group
- `group:<name>:` — permissions for an additional named group
- `mask::` — maximum effective permissions for the owning group and named user/group entries
- `other::` — permissions for users not matched by the owner or ACL user/group entries

For example, on `public/`, the owning group is `admins`. The `group::` entry therefore represents `admins`; a separate named entry such as `group:developers:rwx` grants the developer group access too. On `projects/`, the owning group is `developers`, so `group::` represents `developers` and `admins` is a named group entry.

The ACL mask is important. A named group entry may request `rwx` while its effective permissions are reduced by a mask of `r-x`. `getfacl` displays effective permissions when they differ from the ACL entry. Do not interpret the named entry without checking the mask.

## 5. SGID and group inheritance

The set-group-ID bit on a directory causes newly created children to inherit the directory's group rather than the creating user's primary group. It is commonly used on shared team directories so group ownership remains consistent.

Inspect mode bits with:

```bash
stat -c '%A %a %U:%G %n' /srv/company-files/projects
```

If the setgid bit is part of the deployed policy, the directory mode will display an `s` in the group-execute position (for example, `drwxrws---`). SGID controls group inheritance; it does not by itself guarantee that new files are group-writable. The creating process's umask and any default ACLs also influence newly created file permissions.

## 6. Default ACLs

An access ACL applies to the current object. A default ACL on a directory provides an inheritance template for newly created children. Inspect both access and default entries with `getfacl`.

Default ACLs are useful when a shared directory needs consistent permissions for future files and subdirectories. They should be designed and tested with the application's file-creation behaviour and umask in mind; they do not retroactively repair existing files.

## 7. Chroot boundary

OpenSSH's `ChrootDirectory` is a security boundary, not an ordinary shared folder. The chroot path and its parent components must be owned by root and must not be writable by group or others. Users should not be able to replace or modify the chroot root itself.

The storage tree is arranged so that the root remains controlled by root while the child directories carry the role-specific ACL policy. Inside the SFTP session, the chroot root is presented as `/`, so the host path `/srv/company-files/public` appears as `/public` to the restricted user.

## 8. Validate effective access

ACL text is configuration evidence; testing as the real account verifies the combined effect of group membership, path traversal, mode bits, ACL entries, and masks.

Examples:

```bash
# Expected to succeed for a developer in the configured policy:
sudo -u bob -- test -r /srv/company-files/projects
sudo -u bob -- test -w /srv/company-files/projects
sudo -u bob -- test -x /srv/company-files/projects

# Expected to fail for an intern where write access is not granted:
sudo -u david -- test -w /srv/company-files/projects

# Inspect the path components and ACLs when diagnosing a result:
sudo namei -l /srv/company-files/projects
sudo getfacl -p /srv/company-files/projects
```

For a complete functional test, create and remove temporary files as the target user in a controlled test directory. A `test -w` check is useful, but a real create/write/delete test validates the intended workflow more directly.

## 9. Policy inspection utility

Run:

```bash
company-files-admin policy
```

The command checks the expected owning group and ACL entries for the managed directories. It is a policy inspection aid; it does not replace end-to-end access tests performed as each user.
