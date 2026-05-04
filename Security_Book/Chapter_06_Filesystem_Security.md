# Chapter 6: File System Security

## Learning Goals
- Understand Unix file permission model in detail
- Know special permission bits (setuid, setgid, sticky)
- Understand POSIX ACLs and extended attributes
- Know how the kernel checks file permissions

---

## 6.1 File Permission Model

```
Permission bits (12 bits total):

  ┌── Special ──┐┌── Owner ──┐┌── Group ──┐┌── Other ──┐
  │ S  S  T     ││ r  w  x   ││ r  w  x   ││ r  w  x   │
  │ 4  2  1     ││ 4  2  1   ││ 4  2  1   ││ 4  2  1   │
  └─────────────┘└───────────┘└───────────┘└───────────┘

  Example: -rwxr-xr-- = 0754
    Owner: rwx (7) = read + write + execute
    Group: r-x (5) = read + execute
    Other: r-- (4) = read only

  File types (first character):
    -  regular file
    d  directory
    l  symbolic link
    c  character device
    b  block device
    p  named pipe (FIFO)
    s  socket

  Directory permissions:
    r: Can list directory contents (ls)
    w: Can create/delete files in directory
    x: Can traverse directory (cd, access files within)

    Important: To delete a file, you need WRITE on the DIRECTORY
    (not on the file itself). This is DAC — owner of directory controls.
```

---

## 6.2 Special Permission Bits

```
Setuid (4000):
  -rwsr-xr-x root root /usr/bin/passwd
  When executed: process euid becomes file owner (root)
  Use: Allow non-root to perform privileged operations
  Risk: Any vulnerability → full root compromise

Setgid (2000):
  -rwxr-sr-x root shadow /usr/bin/chage
  When executed: process egid becomes file group
  On directory: new files inherit directory's group

Sticky bit (1000):
  drwxrwxrwt root root /tmp
  On directory: only file owner (or root) can delete/rename files
  Prevents users from deleting each other's files in shared dirs

Kernel check (fs/namei.c):
  may_delete() checks:
    if (sticky && !uid_eq(dir_uid, current_fsuid())
                && !uid_eq(inode_uid, current_fsuid())
                && !capable(CAP_FOWNER))
        return -EPERM;  // Sticky bit prevents deletion
```

---

## 6.3 Permission Check Flow in Kernel

```c
/* Simplified permission check flow */

/* fs/namei.c — generic_permission() */
int generic_permission(struct inode *inode, int mask)
{
    /* Step 1: Check if owner */
    if (uid_eq(current_fsuid(), inode->i_uid)) {
        /* Use owner permission bits */
        mode >>= 6;  /* Shift to get owner bits */
    }
    /* Step 2: Check if in group */
    else if (in_group_p(inode->i_gid)) {
        /* Use group permission bits */
        mode >>= 3;  /* Shift to get group bits */
    }
    /* Step 3: Use "other" bits (no shift) */

    /* Step 4: Compare requested access vs allowed */
    if ((mask & ~mode & (MAY_READ | MAY_WRITE | MAY_EXEC)) == 0)
        return 0;  /* Access granted */

    /* Step 5: Capability override */
    if (capable(CAP_DAC_OVERRIDE))
        return 0;  /* Root capability: bypass DAC */
    if (capable(CAP_DAC_READ_SEARCH) && (mask == MAY_READ))
        return 0;  /* Can read any file */

    return -EACCES;  /* Access denied */
}

/* After DAC check, LSM hook: */
/* security_inode_permission(inode, mask) */
/* SELinux / AppArmor further restricts */
```

---

## 6.4 POSIX Access Control Lists (ACLs)

```
ACLs extend the basic owner/group/other model.
Allow per-user and per-group permissions on files.

Without ACLs:
  -rw-r----- alice developers file.txt
  Only alice and developers group members can access.
  No way to add specific users without changing groups.

With ACLs:
  setfacl -m u:bob:rw file.txt     # Bob gets read/write
  setfacl -m g:testers:r file.txt  # testers group gets read
  getfacl file.txt

  # Output:
  # file: file.txt
  # owner: alice
  # group: developers
  # user::rw-
  # user:bob:rw-
  # group::r--
  # group:testers:r--
  # mask::rw-
  # other::---

ACL in kernel:
  Stored as extended attributes (system.posix_acl_access)
  posix_acl_permission() in fs/posix_acl.c
  Check order:
    1. Owner UID match → use ACL_USER_OBJ entry
    2. Named user match → use ACL_USER entry (masked)
    3. Owning group or named group → use group entry (masked)
    4. ACL_OTHER entry

  Mask: Limits maximum permissions for named users and groups.
  chmod affects the mask, not individual ACL entries.
```

---

## 6.5 File Security Extended Attributes

```
Extended attributes store additional metadata on files.

Security-relevant namespaces:
  security.*     — LSM labels (SELinux contexts, AppArmor)
  system.*       — ACLs, capabilities
  user.*         — User-defined (restricted by permissions)
  trusted.*      — Only root can read/write

Examples:
  # SELinux context
  getfattr -n security.selinux /usr/bin/httpd
  # security.selinux="system_u:object_r:httpd_exec_t:s0"

  # File capabilities
  getcap /usr/bin/ping
  # /usr/bin/ping = cap_net_raw+ep

  # Set file capability (stored in security.capability)
  setcap cap_net_bind_service+ep /usr/bin/myserver

  Kernel implementation:
    __vfs_getxattr(), __vfs_setxattr() in fs/xattr.c
    LSM hook: security_inode_setxattr()
    SELinux restricts which processes can set security.* attrs
```

---

## 6.6 Immutable and Append-Only Files

```
File attributes beyond permissions (ext4, xfs):

  chattr +i file.txt    # Immutable: cannot be modified, deleted, renamed
  chattr +a file.txt    # Append-only: can only append data
  chattr +s file.txt    # Secure deletion: zero on delete
  lsattr file.txt       # List attributes

  Immutable files:
    - Even root cannot modify/delete (unless chattr -i first)
    - Requires CAP_LINUX_IMMUTABLE capability
    - Useful for protecting critical config files
    - Kernel checks: IS_IMMUTABLE(inode) in VFS operations

  Append-only:
    - Only append operations allowed (no overwrite, no truncate)
    - Useful for log files — prevents log tampering
    - Requires CAP_LINUX_IMMUTABLE to set/unset
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| fs/namei.c | Path resolution, permission checks |
| fs/posix_acl.c | POSIX ACL implementation |
| fs/xattr.c | Extended attributes |
| fs/attr.c | Inode attribute changes (chmod, chown) |
| include/linux/fs.h | MAY_READ, MAY_WRITE, MAY_EXEC |
| security/commoncap.c | Capability override checks |

---

## Interview Questions

**Q1: How does the kernel decide file access permissions?**
A: The kernel checks in order: (1) If euid matches file owner → owner bits apply. (2) If egid or any supplementary GID matches file group → group bits apply. (3) Otherwise → "other" bits apply. (4) If basic check fails, capabilities are checked — CAP_DAC_OVERRIDE bypasses read/write/exec for files, CAP_DAC_READ_SEARCH bypasses read/search. (5) After DAC passes, LSM hooks (security_inode_permission) apply MAC policy. Both DAC and MAC must allow.

**Q2: What are the security risks of setuid binaries?**
A: Setuid root binaries run with euid=0 regardless of who executes them. Risks: (1) Any vulnerability (buffer overflow, injection) gives attacker full root. (2) Setuid grants ALL root privileges, not just what the program needs. (3) Environment variables can be manipulated (LD_PRELOAD, PATH). Mitigations: (1) Replace with capabilities — setcap for specific privileges. (2) Minimize setuid binaries. (3) Use SELinux to restrict even setuid processes. Linux ignores setuid on scripts.

**Q3: What is the sticky bit and why is it important?**
A: The sticky bit (chmod +t) on a directory prevents users from deleting or renaming files they don't own, even if they have write permission on the directory. Critical for shared directories like /tmp — without it, any user could delete any other user's files. In the kernel, may_delete() checks: if sticky bit is set AND deleter is not file owner AND deleter is not directory owner AND doesn't have CAP_FOWNER → deny.

---

## Summary

- 12 permission bits: 3 special (setuid/setgid/sticky) + 3 x 3 (owner/group/other rwx)
- Permission check: owner match → group match → other → capability override → LSM
- Setuid: process runs as file owner (dangerous with root)
- Sticky bit: prevents deletion by non-owners in shared directories
- POSIX ACLs: per-user/group permissions beyond basic model
- Extended attributes: store LSM labels, file capabilities, ACLs
- Immutable/append-only: chattr protections even root must explicitly override

---

Next: [Chapter 7 — Linux Capabilities](Chapter_07_Capabilities.md)
