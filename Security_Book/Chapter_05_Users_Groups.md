# Chapter 5: User and Group Management

## Learning Goals
- Understand UID/GID model and kernel credential structures
- Know privileged vs unprivileged user distinctions
- Understand real, effective, saved, and filesystem IDs
- Know supplementary groups and their security implications

---

## 5.1 User Identifiers (UID)

```
Every process has multiple UIDs:

  Real UID (ruid):      Who started the process (login user)
  Effective UID (euid): Used for permission checks
  Saved UID (suid):     Saved euid before setuid change
  Filesystem UID (fsuid): Used for filesystem checks (Linux-specific)

  Normal process:  ruid == euid == suid == fsuid

  Setuid binary:
    Before exec:   ruid=1000, euid=1000
    After exec:    ruid=1000, euid=0, suid=0  (binary owned by root)
    Dropped privs: ruid=1000, euid=1000, suid=0 (can re-escalate)

  Special UIDs:
    0:     root (superuser, bypasses most checks)
    1-999: System/service accounts (convention)
    1000+: Regular users
    65534: nobody (least privilege account)

  UID checks in kernel:
    uid_eq(cred->euid, GLOBAL_ROOT_UID)  // Is effective UID root?
    capable(CAP_DAC_OVERRIDE)            // Has capability? (replaces uid==0)
```

---

## 5.2 Group Identifiers (GID)

```
Group model:
  Primary GID:        From /etc/passwd (user's default group)
  Supplementary GIDs: From /etc/group (additional memberships)
  Max supplementary:  NGROUPS_MAX (65536 on Linux)

  File access check:
    1. Is euid == file owner UID? → use owner permissions
    2. Is egid == file group GID? → use group permissions
       OR is file GID in supplementary groups?
    3. Use "other" permissions

  /etc/group format:
    developers:x:1001:alice,bob,charlie
    │          │  │    │
    │          │  │    └── Members
    │          │  └── GID
    │          └── Group password (x = in gshadow)
    └── Group name

  Groups in kernel (struct group_info):
    groups_search(group_info, gid)  // Is GID in supplementary list?
    Sorted array for O(log n) search
```

---

## 5.3 Privileged vs Unprivileged Users

```
Traditional model:
  UID 0 (root):  ALL permissions, bypasses ALL checks
  UID != 0:      Subject to DAC + MAC checks

  Root bypasses:
    - File permission checks (read/write/exec any file)
    - File ownership changes (chown any file)
    - Process signals (kill any process)
    - Network binding (bind to ports < 1024)
    - Module loading
    - System clock changes
    - Reboot

  Problem: "root" is too powerful → capabilities solve this.

  Modern model (capabilities):
    UID 0 still exists but capabilities provide fine-grained control.
    A process with euid=0 gets full capability set.
    A process with euid!=0 can still have specific capabilities.

    Example: Network daemon only needs CAP_NET_BIND_SERVICE
    setcap cap_net_bind_service+ep /usr/bin/myserver
    Now myserver can bind port 80 without root.

  Service accounts (UID 1-999):
    Run system services with minimal privileges.
    No login shell (/usr/sbin/nologin).
    Own specific files/directories for their service.
    Combined with capabilities for least privilege.
```

---

## 5.4 Credential Kernel API

```c
/* Credential operations in kernel */

/* Get current credentials (read-only) */
const struct cred *cred = current_cred();
uid_t uid = __kuid_val(cred->uid);

/* Change credentials */
struct cred *new = prepare_creds();
if (!new) return -ENOMEM;

new->uid = KUIDT_INIT(1000);
new->euid = KUIDT_INIT(1000);

commit_creds(new);  /* Atomically replace task credentials */

/* Check capabilities */
if (capable(CAP_SYS_ADMIN))
    /* Has sys_admin capability */

/* Check credentials of specific task */
if (has_capability(task, CAP_NET_RAW))
    /* Task can create raw sockets */

/* Namespace-aware UID check */
kuid_t kuid = make_kuid(user_ns, uid);
if (uid_eq(kuid, GLOBAL_ROOT_UID))
    /* UID is root in init namespace */
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| kernel/cred.c | Credential management (prepare, commit) |
| kernel/sys.c | setuid, setgid, setgroups syscalls |
| kernel/groups.c | Supplementary group management |
| kernel/uid16.c | 16-bit UID compat (old syscalls) |
| include/linux/cred.h | struct cred definition |
| include/linux/uidgid.h | kuid_t, kgid_t types |

---

## Interview Questions

**Q1: What is the difference between real, effective, saved, and filesystem UIDs?**
A: **Real UID**: identity of the user who started the process (set at login). **Effective UID**: used for all permission checks (file access, IPC, etc.). **Saved UID**: saved value of euid before a setuid change, allows re-escalation via seteuid(). **Filesystem UID**: Linux-specific, used for filesystem checks (usually equals euid, but NFS server uses it to act as different users). When a setuid binary runs: ruid stays as user, euid/suid become the binary owner (e.g., root).

**Q2: Why does Linux have a separate filesystem UID (fsuid)?**
A: The fsuid was created for the NFS server which runs as root but needs to perform filesystem operations as if it were different users. Setting fsuid changes only filesystem permission checks without changing effective UID, so the NFS daemon keeps its root privileges for other operations while accessing files as the requesting user. It's a Linux-specific extension — POSIX only defines ruid, euid, and suid.

**Q3: How does the kernel handle supplementary groups?**
A: Supplementary groups are stored in struct group_info (sorted array of GIDs). During file access, after checking owner UID, the kernel calls in_group_p(gid) which searches the sorted supplementary group array (O(log n) binary search). If the file's GID matches any supplementary group, group permissions apply. Set via setgroups() syscall, limited to NGROUPS_MAX (65536).

---

## Summary

- Process has 4 UID types: real, effective, saved, filesystem
- Effective UID is used for all permission checks
- Root (UID 0) bypasses DAC — capabilities provide finer granularity
- Supplementary groups extend the basic owner/group/other model
- Credentials are immutable — prepare_creds() → modify → commit_creds()
- Service accounts (UID 1-999) + capabilities = least privilege for daemons

---

Next: [Chapter 6 — File System Security](Chapter_06_Filesystem_Security.md)
