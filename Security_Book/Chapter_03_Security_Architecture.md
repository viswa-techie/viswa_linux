# Chapter 3: Security Architecture in Linux

## Learning Goals
- Understand the complete Linux security model
- Know security enforcement points in the kernel
- Understand separation of policy and mechanism
- Know how security decisions flow through the kernel

---

## 3.1 Linux Security Model Overview

```
┌─────────────────────── Linux Security Architecture ───────────────────────┐
│                                                                           │
│  USER SPACE                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  Application                                                    │      │
│  │    │                                                            │      │
│  │    ├── Authentication: PAM verifies identity                     │      │
│  │    ├── Credentials: UID, GID, groups, capabilities              │      │
│  │    └── Seccomp: self-imposed syscall restrictions               │      │
│  └──────────────────────────────┬──────────────────────────────────┘      │
│                                 │ system call                             │
│  ────────────────────────────── │ ─────────────────────────────────       │
│                                 ▼                                         │
│  KERNEL SPACE                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  1. Seccomp Filter Check                                       │      │
│  │     Is this syscall allowed by the BPF filter?                  │      │
│  │     → DENY: return -EPERM (or SIGKILL)                         │      │
│  │     → ALLOW: continue                                           │      │
│  │                                                                 │      │
│  │  2. Syscall Handler                                             │      │
│  │     Validate arguments, copy from user                          │      │
│  │                                                                 │      │
│  │  3. DAC Check (Discretionary Access Control)                    │      │
│  │     Check UID/GID vs file owner/group + rwx bits                │      │
│  │     Check capabilities (CAP_*)                                  │      │
│  │     → DENY: return -EACCES / -EPERM                            │      │
│  │     → ALLOW: continue to LSM                                    │      │
│  │                                                                 │      │
│  │  4. LSM Hook (security_*() call)                                │      │
│  │     ┌──────────────────────────────────────────────┐            │      │
│  │     │  SELinux:  Check type enforcement policy      │            │      │
│  │     │  AppArmor: Check profile rules                │            │      │
│  │     │  Smack:    Check label comparison             │            │      │
│  │     │  TOMOYO:   Check domain policy                │            │      │
│  │     │  Landlock: Check self-imposed restrictions    │            │      │
│  │     └──────────────────────────────────────────────┘            │      │
│  │     → DENY: return -EACCES                                      │      │
│  │     → ALLOW: proceed with operation                             │      │
│  │                                                                 │      │
│  │  5. Operation executes (file open, socket connect, etc.)        │      │
│  │                                                                 │      │
│  │  6. Audit log (if configured)                                   │      │
│  └─────────────────────────────────────────────────────────────────┘      │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘

KEY PRINCIPLE:
  DAC allows → then LSM checks
  Both DAC and MAC must allow for access to succeed.
  MAC can ONLY deny what DAC would allow (never grant more).
```

---

## 3.2 Security Enforcement Points

```
Every security-sensitive kernel operation has enforcement points:

File Operations:
  open()     → security_file_open()
  read()     → security_file_permission(MAY_READ)
  write()    → security_file_permission(MAY_WRITE)
  mmap()     → security_mmap_file()
  ioctl()    → security_file_ioctl()

Process Operations:
  fork()     → security_task_alloc()
  exec()     → security_bprm_check()
  kill()     → security_task_kill()
  ptrace()   → security_ptrace_access_check()

IPC Operations:
  msgget()   → security_msg_queue_alloc()
  shmget()   → security_shm_alloc()
  semget()   → security_sem_alloc()

Network Operations:
  socket()   → security_socket_create()
  bind()     → security_socket_bind()
  connect()  → security_socket_connect()
  accept()   → security_socket_accept()
  sendmsg()  → security_socket_sendmsg()
  recvmsg()  → security_socket_recvmsg()

Module Operations:
  init_module() → security_kernel_module_request()
  finit_module()→ security_kernel_load_data()

Mount Operations:
  mount()    → security_sb_mount()
  umount()   → security_sb_umount()
```

---

## 3.3 Kernel vs User-Space Security

```
┌────────────────────────────────────────────────────────────────┐
│ Responsibility       │ Kernel                │ User Space      │
├──────────────────────┼───────────────────────┼─────────────────┤
│ Authentication       │ Validates creds       │ PAM, login      │
│ Authorization        │ DAC + MAC + caps      │ Application     │
│ Credential storage   │ struct cred           │ /etc/passwd     │
│ Policy enforcement   │ LSM hooks             │ Policy tools    │
│ Crypto primitives    │ crypto API            │ OpenSSL, etc.   │
│ Key storage          │ Keyring service       │ Key agents      │
│ Isolation            │ Namespaces, cgroups   │ Container runtime│
│ Syscall filtering    │ Seccomp BPF           │ Configures rules│
│ Audit               │ Audit subsystem       │ auditd daemon   │
│ Memory protection    │ ASLR, NX, KPTI       │ PIE, stack guard│
│ Network filtering    │ Netfilter             │ iptables/nft    │
│ Secure boot         │ Module sig verify     │ UEFI firmware   │
└──────────────────────┴───────────────────────┴─────────────────┘

Key principle: Policy in user space, Enforcement in kernel.
  - User space defines WHAT should be allowed (policy files)
  - Kernel enforces HOW the policy is applied (hooks)
  - Neither alone is sufficient
```

---

## 3.4 The Credential Model

```c
/* include/linux/cred.h — struct cred */
struct cred {
    atomic_t    usage;          /* Reference count */

    /* POSIX IDs */
    kuid_t      uid;            /* Real UID */
    kgid_t      gid;            /* Real GID */
    kuid_t      suid;           /* Saved UID */
    kgid_t      sgid;           /* Saved GID */
    kuid_t      euid;           /* Effective UID (used for access checks) */
    kgid_t      egid;           /* Effective GID */
    kuid_t      fsuid;          /* UID for filesystem checks */
    kgid_t      fsgid;         /* GID for filesystem checks */

    /* Capabilities */
    kernel_cap_t cap_inheritable; /* Kept across exec */
    kernel_cap_t cap_permitted;   /* Max allowed */
    kernel_cap_t cap_effective;   /* Currently active */
    kernel_cap_t cap_bset;        /* Bounding set */
    kernel_cap_t cap_ambient;     /* Ambient capabilities */

    /* Groups */
    struct group_info *group_info; /* Supplementary groups */

    /* LSM security context */
    void        *security;      /* LSM-specific data (SELinux label, etc.) */

    /* Keyrings */
    struct key  *session_keyring;
    struct key  *process_keyring;
    struct key  *thread_keyring;
};
```

```
Credential lifecycle:
  fork()   → child inherits parent's cred (copy-on-write)
  exec()   → cred may change (setuid, capabilities, LSM transition)
  setuid() → changes real/effective UID

  Credentials are IMMUTABLE once attached to a task.
  To change: prepare_creds() → modify → commit_creds()
  This prevents race conditions in credential checks.

  ┌────────┐  fork()   ┌────────┐  exec(/bin/su)  ┌────────┐
  │ uid=1000│ ────────► │uid=1000│ ───────────────► │uid=0   │
  │ euid=  │          │euid=  │  setuid binary   │euid=0  │
  │  1000  │          │ 1000  │                   │        │
  └────────┘          └────────┘                   └────────┘
```

---

## 3.5 Security Policy vs Mechanism

```
Separation of concerns:

  Policy:    WHAT is allowed/denied (human-readable rules)
  Mechanism: HOW rules are enforced (kernel code)

  Example — SELinux:
    Policy:    "httpd process can read /var/www files"
               Written in SELinux policy language (.te files)
               Compiled to binary policy, loaded into kernel
    Mechanism: LSM hook in VFS checks process label vs file label
               security_inode_permission() → selinux_inode_permission()

  Example — Seccomp:
    Policy:    "This process can only call read, write, exit, sigreturn"
               Written as BPF program, attached via prctl()
    Mechanism: Seccomp filter runs before syscall dispatch
               If syscall not in allowlist → process killed/returns error

  Benefits:
    - Policy can be updated without kernel changes
    - Same mechanism supports different policies
    - Policy can be audited independently
    - Different environments can have different policies
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| security/security.c | LSM framework, security_*() wrappers |
| kernel/cred.c | Credential management |
| include/linux/security.h | All LSM hook declarations |
| include/linux/cred.h | struct cred definition |
| kernel/seccomp.c | Seccomp filter implementation |
| fs/namei.c | Path resolution with DAC checks |
| fs/open.c | File open with security checks |

---

## Interview Questions

**Q1: Describe the order of security checks when a process opens a file.**
A: (1) Seccomp: if filter attached, check if open() syscall is allowed. (2) Path resolution: resolve pathname. (3) DAC: check file owner/group/other bits against process's effective UID/GID. Check capabilities (CAP_DAC_OVERRIDE can bypass). (4) LSM: call security_inode_permission() — SELinux checks type enforcement (process domain vs file type), AppArmor checks profile rules. Both DAC and LSM must allow. If any denies, -EACCES is returned. (5) If permitted, the file descriptor is created and returned.

**Q2: Why are credentials immutable in the Linux kernel?**
A: Credentials (struct cred) are read by many code paths concurrently (access checks, signal delivery, etc.). If credentials were mutable, TOCTOU races would occur — checking UID in one place, then it changes before the operation completes. By making creds immutable, once attached to a task, all readers see a consistent snapshot. To change credentials: prepare_creds() creates a new copy, modify it, then atomically commit_creds() to the task.

**Q3: What is the relationship between DAC and MAC in Linux?**
A: DAC (Discretionary Access Control) is checked first — traditional Unix permissions based on UID/GID. MAC (Mandatory Access Control via LSM) is checked after DAC passes. Both must allow for access to succeed. MAC can only further restrict access — it cannot grant access that DAC denied. This means MAC is an additional security layer on top of DAC, following the defense-in-depth principle.

---

## Summary

- Security checks flow: Seccomp → Syscall → DAC → LSM (MAC) → Operation → Audit
- Both DAC and MAC must allow — MAC further restricts, never grants more than DAC
- Enforcement points: ~200 LSM hooks covering files, processes, IPC, network, modules
- Credentials are immutable (copy-on-write) to prevent TOCTOU races
- Policy (what) is separated from mechanism (how) — enables flexibility
- Kernel enforces policy; user space defines policy

---

Next: [Chapter 4 — Authentication Mechanisms](Chapter_04_Authentication.md)
