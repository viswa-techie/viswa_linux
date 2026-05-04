# Chapter 8: Linux Security Modules (LSM)

## Learning Goals
- Understand LSM architecture and hook framework
- Know how security modules register and make decisions
- Understand LSM stacking and multiple modules
- Know the LSM API and hook lifecycle

---

## 8.1 LSM Architecture Overview

```
LSM: A framework that inserts security hook calls at every
security-sensitive kernel operation. Security modules register
callbacks to enforce their policies.

  ┌─────────────────── LSM Architecture ───────────────────┐
  │                                                        │
  │  Kernel Operation (e.g., open file)                    │
  │      │                                                 │
  │      ├── DAC check (traditional permissions)           │
  │      │                                                 │
  │      └── LSM Hook: security_inode_permission()         │
  │              │                                         │
  │              ▼                                         │
  │      ┌── LSM Framework (security/security.c) ────┐    │
  │      │                                            │    │
  │      │  Call each registered module's callback:   │    │
  │      │                                            │    │
  │      │  ┌──────────┐  ┌──────────┐  ┌─────────┐  │    │
  │      │  │ SELinux   │  │ AppArmor │  │Landlock │  │    │
  │      │  │callback() │  │callback()│  │callback()│ │    │
  │      │  └──────────┘  └──────────┘  └─────────┘  │    │
  │      │                                            │    │
  │      │  Aggregation: ALL must return 0 (allow)    │    │
  │      │  Any -EACCES → access denied               │    │
  │      └────────────────────────────────────────────┘    │
  │                                                        │
  │      If allowed → operation proceeds                   │
  │      If denied  → return error to caller               │
  │                                                        │
  └────────────────────────────────────────────────────────┘

Key principles:
  1. LSM hooks are RESTRICTIVE only — can deny, never grant
  2. Hooks are called AFTER DAC checks pass
  3. Multiple LSMs can be stacked (all must allow)
  4. No LSM can override another LSM's denial
  5. security_*() functions are the hook wrappers
```

---

## 8.2 LSM Hook Categories

```
~200 hooks covering all security-sensitive operations:

File/Inode hooks:
  security_inode_create()        — Create new file
  security_inode_link()          — Create hard link
  security_inode_unlink()        — Delete file
  security_inode_mkdir()         — Create directory
  security_inode_rename()        — Rename file
  security_inode_permission()    — Check file access (read/write/exec)
  security_inode_setattr()       — Change file attributes (chmod)
  security_inode_getattr()       — Stat file
  security_inode_setxattr()      — Set extended attribute
  security_file_open()           — Open file
  security_file_permission()     — Read/write on open file
  security_mmap_file()           — Memory map file
  security_file_ioctl()          — ioctl on file

Process hooks:
  security_task_alloc()          — fork/clone
  security_task_free()           — Process exit
  security_task_kill()           — Send signal
  security_task_setnice()        — Change priority
  security_bprm_check()          — Check binary before exec
  security_bprm_creds_for_exec() — Compute new creds for exec
  security_ptrace_access_check() — ptrace attach

Network hooks:
  security_socket_create()       — Create socket
  security_socket_bind()         — Bind address
  security_socket_connect()      — Connect to remote
  security_socket_listen()       — Listen for connections
  security_socket_accept()       — Accept connection
  security_socket_sendmsg()      — Send data
  security_socket_recvmsg()      — Receive data
  security_sk_alloc()            — Allocate sock struct

IPC hooks:
  security_msg_queue_alloc()     — Create message queue
  security_shm_alloc()           — Create shared memory
  security_sem_alloc()           — Create semaphore

Module/System hooks:
  security_kernel_module_request() — Module auto-load
  security_kernel_load_data()      — Load firmware/module
  security_locked_down()           — Lockdown check
```

---

## 8.3 LSM Registration

```c
/* How a security module registers with LSM */

/* Define the security hooks */
static struct security_hook_list my_hooks[] = {
    LSM_HOOK_INIT(inode_permission, my_inode_permission),
    LSM_HOOK_INIT(file_open, my_file_open),
    LSM_HOOK_INIT(task_kill, my_task_kill),
    LSM_HOOK_INIT(socket_create, my_socket_create),
};

/* Module initialization */
static int __init my_lsm_init(void)
{
    security_add_hooks(my_hooks, ARRAY_SIZE(my_hooks), "my_lsm");
    return 0;
}

/* Define LSM */
DEFINE_LSM(my_lsm) = {
    .name = "my_lsm",
    .init = my_lsm_init,
};

/* Hook implementation */
static int my_inode_permission(struct inode *inode, int mask)
{
    /* Return 0 to allow, -EACCES to deny */
    if (should_deny(inode, mask))
        return -EACCES;
    return 0;
}
```

---

## 8.4 LSM Stacking

```
Modern kernels support multiple LSMs simultaneously.

Major LSMs (exclusive, pick one):
  SELinux, AppArmor, Smack, TOMOYO
  Historically only ONE major LSM could be active.
  Since kernel 6.x: stacking improvements (multiple major possible)

Minor LSMs (can coexist with major):
  Yama:     ptrace restrictions
  LoadPin:  Restrict kernel module loading to one filesystem
  Lockdown: Restrict kernel features (integrity/confidentiality)
  Landlock: Unprivileged sandboxing
  BPF LSM:  Programmable security hooks

Boot parameter to select:
  lsm=landlock,lockdown,yama,selinux    # Order matters
  lsm=landlock,lockdown,yama,apparmor

  CONFIG_LSM="landlock,lockdown,yama,apparmor" # Default
  CONFIG_DEFAULT_SECURITY="apparmor"           # Default major

Stack evaluation:
  security_inode_permission() calls each registered hook:
    1. Landlock: check → allow (0)
    2. Yama: not relevant → allow (0)
    3. SELinux: check type enforcement → allow (0) or deny (-EACCES)
  All must return 0 for access to be granted.
```

---

## 8.5 LSM Blob Management

```
LSMs need to store per-object security state:

  Object          │ LSM Data (security blob)
  ────────────────┼─────────────────────────────
  struct cred     │ Process security context
  struct inode    │ File security label
  struct file     │ Open file security state
  struct super_block │ Filesystem security
  struct msg_msg  │ IPC message label
  struct sk_buff  │ Packet security label
  struct sock     │ Socket security

  Blobs: Each LSM allocates a portion of the security blob.
  Managed by LSM framework (lsm_*_alloc functions).

  struct cred {
      ...
      void *security;  // Points to blob containing ALL LSM data
  };

  Blob layout (stacked LSMs):
  ┌────────────────┬────────────────┬─────────────────┐
  │ SELinux data   │ AppArmor data  │ Landlock data   │
  │ (task context) │ (profile ref)  │ (domain ref)    │
  └────────────────┴────────────────┴─────────────────┘
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| security/security.c | LSM framework core, hook dispatching |
| include/linux/security.h | security_*() declarations |
| include/linux/lsm_hooks.h | LSM hook definitions |
| include/linux/lsm_hook_defs.h | Auto-generated hook list |
| security/Kconfig | LSM config options |
| security/Makefile | LSM build configuration |

---

## Interview Questions

**Q1: What is the LSM framework and how does it work?**
A: LSM provides ~200 security hook points throughout the kernel — at every security-sensitive operation (file access, process creation, network operations, IPC). Security modules (SELinux, AppArmor, etc.) register callbacks for hooks they care about. When a security-sensitive operation occurs, after DAC checks pass, the LSM framework calls each registered module's callback. If ANY module returns -EACCES, the operation is denied. LSM hooks are restrictive only — they can deny but never grant access beyond what DAC allows.

**Q2: How does LSM stacking work?**
A: Multiple LSMs can be active simultaneously. Minor LSMs (Yama, Lockdown, Landlock) always stack. Major LSMs (SELinux, AppArmor, Smack) historically were exclusive, but recent kernels support stacking multiple major LSMs. The boot parameter `lsm=` controls which LSMs are active and their order. For a security check, ALL registered hooks must return 0 (allow). If any hook returns -EACCES, access is denied. Each LSM stores its per-object data in "blobs" — shared security data attached to kernel objects.

**Q3: Can an LSM grant permissions that DAC denied?**
A: No. LSM hooks are called AFTER DAC checks pass. If DAC denies access (wrong UID/GID, no capability), the LSM is never consulted. LSM can only further restrict access that DAC would allow. This is by design — MAC provides additional security on top of DAC, not a replacement.

---

## Summary

- LSM provides ~200 hooks at security-sensitive kernel operations
- Hooks are RESTRICTIVE: can deny, never grant beyond DAC
- Called after DAC passes — both must allow for access
- Multiple LSMs can stack — all must allow (AND logic)
- Major LSMs: SELinux, AppArmor, Smack, TOMOYO (historically exclusive)
- Minor LSMs: Yama, LoadPin, Lockdown, Landlock, BPF LSM (always stack)
- Per-object security data stored in blobs (cred, inode, socket, etc.)

---

Next: [Chapter 9 — Mandatory Access Control (MAC)](Chapter_09_MAC.md)
