# Chapter 10: Discretionary Access Control (DAC)

## Learning Goals
- Understand DAC model in detail
- Know how DAC is implemented in the Linux kernel
- Understand DAC limitations and why MAC is needed

---

## 10.1 DAC Model

```
DAC: Object OWNER controls access to their objects.
"Discretionary" because owner has discretion to grant/revoke.

Linux DAC components:
  1. File permission bits (rwx for owner/group/other)
  2. File ownership (UID, GID)
  3. Capabilities (fine-grained root privileges)
  4. POSIX ACLs (extended per-user/group permissions)

Decision flow:
  ┌──────────────────────────────────────────────────────┐
  │ Process wants to read file                            │
  │                                                      │
  │ Check 1: Is process euid == file owner UID?          │
  │   YES → Apply owner bits (rwx position 1)            │
  │   NO  → Continue                                     │
  │                                                      │
  │ Check 2: Is process egid (or supplementary) == file GID?│
  │   YES → Apply group bits (rwx position 2)            │
  │   NO  → Continue                                     │
  │                                                      │
  │ Check 3: Apply "other" bits (rwx position 3)         │
  │                                                      │
  │ If permission bits deny:                              │
  │   Check 4: Does process have CAP_DAC_OVERRIDE?       │
  │     YES → Allow (capability override)                 │
  │     NO  → EACCES                                      │
  │                                                      │
  │ If POSIX ACLs exist:                                  │
  │   ACL entries checked before "other" bits             │
  │   Named user/group entries with mask                  │
  └──────────────────────────────────────────────────────┘
```

---

## 10.2 DAC in Kernel Implementation

```c
/* fs/namei.c — Core DAC check */
static int acl_permission_check(struct inode *inode, int mask)
{
    unsigned int mode = inode->i_mode;

    /* Owner check */
    if (uid_eq(current_fsuid(), inode->i_uid)) {
        mode >>= 6;    /* Use owner bits */
    }
    /* Group check */
    else if (in_group_p(inode->i_gid) ||
             posix_acl_has_group(inode, mask)) {
        mode >>= 3;    /* Use group bits */
    }
    /* Other — no shift */

    /* Check permission bits match request */
    if ((mask & ~mode & (MAY_READ|MAY_WRITE|MAY_EXEC)) == 0)
        return 0;       /* DAC allows */

    return -EACCES;     /* DAC denies */
}

/* After DAC, capability override check: */
static int generic_permission(struct inode *inode, int mask)
{
    int ret = acl_permission_check(inode, mask);
    if (ret != -EACCES)
        return ret;

    /* CAP_DAC_OVERRIDE: bypass read/write/exec for non-dirs */
    /* CAP_DAC_READ_SEARCH: bypass read + directory search */
    if (!(mask & MAY_EXEC) || /* ... */ )
        if (capable_wrt_inode_uidgid(inode, CAP_DAC_OVERRIDE))
            return 0;

    if (mask == MAY_READ ||
        (S_ISDIR(inode->i_mode) && !(mask & MAY_WRITE)))
        if (capable_wrt_inode_uidgid(inode, CAP_DAC_READ_SEARCH))
            return 0;

    return -EACCES;
}
```

---

## 10.3 DAC Limitations

```
Why DAC alone is insufficient:

1. Root bypasses everything:
   Any process with euid=0 can read/write/execute anything.
   One root exploit = total system compromise.

2. Owner can relax security:
   User can chmod 777 sensitive files.
   No system-wide policy prevents this.
   Malicious or careless user can leak data.

3. No information flow control:
   Once data is accessed by a process, it can be copied anywhere.
   Process reads secret file → writes copy to world-readable location.
   DAC cannot prevent this "information laundering."

4. No process isolation:
   Multiple processes under same UID share all permissions.
   Compromised browser → access email, SSH keys, all user files.

5. Ambient authority problem:
   Process inherits all capabilities/permissions of its UID.
   Cannot restrict a specific program to a subset of user's rights
   (without capabilities or MAC).

6. TOCTOU races:
   Check permissions at open time, but file may have changed.
   Symlink races can be exploited.

These limitations are exactly what MAC (SELinux, AppArmor) addresses.
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| fs/namei.c | acl_permission_check, generic_permission |
| fs/posix_acl.c | POSIX ACL evaluation |
| security/commoncap.c | Capability override checks |
| include/linux/fs.h | MAY_READ, MAY_WRITE, MAY_EXEC |

---

## Interview Questions

**Q1: What are the limitations of DAC and how does MAC address them?**
A: DAC limitations: (1) Root bypasses all — MAC constrains even root with labels. (2) Owners can relax security — MAC policy is system-wide, users can't override. (3) No information flow control — MAC labels track data flow. (4) No per-process isolation — MAC gives each process a unique domain/profile. (5) Ambient authority — MAC restricts to only labeled resources. MAC doesn't replace DAC — it adds an additional enforcement layer checked after DAC passes.

**Q2: How does the kernel determine which permission bits to check?**
A: The kernel checks in order: (1) If process fsuid matches file owner UID → use owner bits (bits 8-6). (2) If process fsgid or any supplementary GID matches file GID → use group bits (bits 5-3). (3) Otherwise → use "other" bits (bits 2-0). If the permission check fails, capability override is attempted: CAP_DAC_OVERRIDE for general bypass, CAP_DAC_READ_SEARCH for read/search bypass. Then LSM hooks provide MAC.

---

## Summary

- DAC: owner controls access via permission bits (rwx) and ACLs
- Check order: owner match → group match → other → capability override
- Root (UID 0) bypasses all DAC checks — major limitation
- Users can relax security — no system-wide enforcement
- No process isolation or information flow control
- Capabilities partially address root all-or-nothing problem
- MAC (via LSM) completes the picture with system-wide mandatory policy

---

Next: [Chapter 11 — SELinux](Chapter_11_SELinux.md)
