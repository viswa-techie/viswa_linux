# Chapter 7: Linux Capabilities

## Learning Goals
- Understand the capability model and why root is replaced
- Know capability sets (effective, permitted, inheritable, bounding, ambient)
- Understand capability checks in the kernel
- Know how to use and drop capabilities

---

## 7.1 Limitations of Root and the Capability Solution

```
Problem: Root (UID 0) is ALL-OR-NOTHING.

  ping needs raw sockets      → must be setuid root → gets ALL root powers
  httpd needs port 80         → must be setuid root → gets ALL root powers
  ntpd needs clock adjustment → must be setuid root → gets ALL root powers

  If any of these is compromised → attacker has FULL root access.

Solution: Split root into ~40 fine-grained capabilities.

  ping needs: CAP_NET_RAW only
  httpd needs: CAP_NET_BIND_SERVICE only
  ntpd needs: CAP_SYS_TIME only

  If httpd is compromised → attacker can only bind privileged ports,
  NOT load modules, NOT change time, NOT read any file.
```

---

## 7.2 Capability Sets

```
Every process has 5 capability sets (bitmasks):

  Effective (E):     Capabilities actually checked by kernel
                     "What can I do RIGHT NOW?"

  Permitted (P):     Maximum capabilities available
                     "What COULD I enable?"
                     Superset of Effective

  Inheritable (I):   Capabilities preserved across exec()
                     "What can I pass to child programs?"

  Bounding (B):      Hard limit on capabilities
                     "Maximum capabilities ever obtainable"
                     Can only be reduced, never enlarged

  Ambient (A):       Capabilities auto-added to permitted+effective on exec()
                     "Capabilities for non-setuid programs"
                     Added in Linux 4.3 to simplify capability inheritance

  Relationship:
    P(post-exec) = (P(pre-exec) & B) | (fP & B) | A
    E(post-exec) = fE ? P(post-exec) : A
    I(post-exec) = I(pre-exec) & (fI | B)

  Where fP, fE, fI are FILE capabilities (security.capability xattr)

  Simplified:
    ┌──────────────────────────────────────────────────────┐
    │  Process: Effective ⊆ Permitted ⊆ Bounding          │
    │                                                      │
    │  exec(file):                                         │
    │    File caps (fP, fI, fE) + Process caps             │
    │    → New process caps                                │
    │                                                      │
    │  Ambient:                                            │
    │    Non-root process can pass caps to exec'd programs │
    │    Without file capabilities on the binary           │
    └──────────────────────────────────────────────────────┘
```

---

## 7.3 Important Capabilities

```
┌────────────────────────────────────────────────────────────────────┐
│ Capability              │ Permission Granted                       │
├─────────────────────────┼──────────────────────────────────────────┤
│ CAP_CHOWN               │ Change file ownership                   │
│ CAP_DAC_OVERRIDE        │ Bypass file permission checks           │
│ CAP_DAC_READ_SEARCH     │ Bypass file read permission             │
│ CAP_FOWNER              │ Bypass file owner checks                │
│ CAP_KILL                │ Send signals to any process             │
│ CAP_NET_ADMIN           │ Network admin (interfaces, routing)     │
│ CAP_NET_BIND_SERVICE    │ Bind to ports < 1024                    │
│ CAP_NET_RAW             │ Use raw/packet sockets                  │
│ CAP_SYS_ADMIN           │ Broad admin (mount, setns, bpf, etc.)  │
│ CAP_SYS_BOOT            │ Reboot the system                      │
│ CAP_SYS_MODULE          │ Load/unload kernel modules              │
│ CAP_SYS_PTRACE          │ Trace any process (ptrace)              │
│ CAP_SYS_RAWIO           │ Direct I/O access (iopl, ioperm)       │
│ CAP_SYS_TIME            │ Set system clock                        │
│ CAP_SETUID              │ Change UID (setuid syscall)             │
│ CAP_SETGID              │ Change GID (setgid syscall)             │
│ CAP_LINUX_IMMUTABLE     │ Set/clear immutable/append-only attrs   │
│ CAP_MKNOD               │ Create device special files             │
│ CAP_AUDIT_WRITE         │ Write to audit log                      │
│ CAP_BPF                 │ BPF operations (added in 5.8)           │
│ CAP_PERFMON             │ Performance monitoring (added in 5.8)    │
│ CAP_CHECKPOINT_RESTORE  │ Checkpoint/restore (added in 5.9)       │
└─────────────────────────┴──────────────────────────────────────────┘

CAP_SYS_ADMIN is the "catch-all" — effectively near-root.
Goal: Never grant CAP_SYS_ADMIN. Split into specific capabilities.
```

---

## 7.4 Using Capabilities

```bash
# File capabilities (setcap/getcap):
# Set on binary — takes effect on exec()
setcap cap_net_bind_service+ep /usr/bin/myserver
getcap /usr/bin/myserver
# /usr/bin/myserver cap_net_bind_service=ep

# Flags: e=effective, p=permitted, i=inheritable
# +ep means: add to both effective AND permitted
# File with cap_net_bind_service+ep:
#   When executed, process gets CAP_NET_BIND_SERVICE

# Remove capabilities:
setcap -r /usr/bin/myserver

# Process capabilities (view):
cat /proc/self/status | grep Cap
# CapInh: 0000000000000000   (inheritable)
# CapPrm: 0000000000000000   (permitted)
# CapEff: 0000000000000000   (effective)
# CapBnd: 000001ffffffffff   (bounding)
# CapAmb: 0000000000000000   (ambient)

# Decode:
capsh --decode=000001ffffffffff

# Drop capabilities in code: (see Section 7.5)
```

---

## 7.5 Dropping Capabilities in Code

```c
/* Best practice: Drop all capabilities except what's needed */
#include <sys/capability.h>
#include <sys/prctl.h>

void drop_caps(void)
{
    cap_t caps;

    /* Clear all capabilities */
    caps = cap_init();

    /* Keep only what we need */
    cap_value_t keep[] = { CAP_NET_BIND_SERVICE };
    cap_set_flag(caps, CAP_PERMITTED, 1, keep, CAP_SET);
    cap_set_flag(caps, CAP_EFFECTIVE, 1, keep, CAP_SET);

    cap_set_proc(caps);
    cap_free(caps);

    /* Prevent re-escalation */
    prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
}

/* Alternative: use prctl to drop bounding set */
void drop_bounding(void)
{
    /* Drop all capabilities from bounding set except needed ones */
    for (int cap = 0; cap <= CAP_LAST_CAP; cap++) {
        if (cap != CAP_NET_BIND_SERVICE)
            prctl(PR_CAPBSET_DROP, cap, 0, 0, 0);
    }
}
```

```
PR_SET_NO_NEW_PRIVS:
  Once set, cannot be unset (even by root).
  Prevents the process and all descendants from:
    - Gaining capabilities through exec of setuid/setcap binaries
    - Transitioning to different SELinux domains on exec
  Used by: seccomp, container runtimes, systemd services
```

---

## 7.6 Capability Checks in Kernel

```c
/* kernel/capability.c — How capability checks work */

/* Check if current process has capability in its user namespace */
bool capable(int cap)
{
    return ns_capable(&init_user_ns, cap);
}

/* Namespace-aware capability check */
bool ns_capable(struct user_namespace *ns, int cap)
{
    if (unlikely(!cap_valid(cap)))
        return false;

    /* Check if cap is in effective set */
    if (security_capable(current_cred(), ns, cap, CAP_OPT_NONE) == 0)
        return true;

    return false;
}

/* security_capable() calls LSM hook:
   SELinux checks: does the process's domain have the capability?
   AppArmor checks: does the profile allow this capability? */

/* Common check patterns in kernel code: */
if (!capable(CAP_SYS_ADMIN))
    return -EPERM;

if (!capable(CAP_NET_ADMIN) && !ns_capable(net->user_ns, CAP_NET_ADMIN))
    return -EPERM;
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| kernel/capability.c | capable(), ns_capable() |
| security/commoncap.c | Common capability logic |
| include/linux/capability.h | CAP_* definitions |
| include/uapi/linux/capability.h | UAPI capability constants |
| fs/exec.c | Capability transformation on exec |
| kernel/cred.c | Capability storage in cred |

---

## Interview Questions

**Q1: What are Linux capabilities and why were they introduced?**
A: Capabilities split root's monolithic privilege into ~40 fine-grained permissions. Introduced because the root/non-root model is all-or-nothing: a program needing one privilege (like binding port 80) had to run as root with ALL privileges. With capabilities, you grant only CAP_NET_BIND_SERVICE. If compromised, the attacker can only bind privileged ports, not load modules, change time, or bypass file permissions. Principle of least privilege.

**Q2: Explain the five capability sets.**
A: **Effective**: what the kernel actually checks — the "active" capabilities. **Permitted**: ceiling of capabilities the process can enable into effective. **Inheritable**: capabilities preserved across exec() (combined with file inheritable). **Bounding**: hard upper limit — capabilities can never exceed this set. Can only be reduced. **Ambient**: auto-added to permitted+effective on exec of non-setuid binaries (since Linux 4.3). Ambient solves the problem of non-root processes passing capabilities to exec'd programs without needing file capabilities.

**Q3: What is NO_NEW_PRIVS and why is it important?**
A: PR_SET_NO_NEW_PRIVS is a per-process flag that, once set, cannot be unset. It prevents the process (and all descendants) from gaining privileges through exec — setuid bits and file capabilities are ignored. This is critical for: (1) Seccomp — ensures filtered process can't exec a setuid binary to escape the filter. (2) Containers — prevents privilege escalation. (3) systemd services — NoNewPrivileges=true in unit files. Combined with capability dropping, it creates an irreversible privilege reduction.

---

## Summary

- Capabilities split root into ~40 fine-grained privileges
- 5 sets: Effective (active), Permitted (ceiling), Inheritable (across exec), Bounding (hard limit), Ambient (auto-grant)
- File capabilities (setcap) replace setuid for specific privileges
- CAP_SYS_ADMIN is near-root — avoid granting it
- NO_NEW_PRIVS prevents privilege escalation through exec
- Kernel checks: capable() → ns_capable() → security_capable() (LSM)

---

Next: [Chapter 8 — Linux Security Modules (LSM)](Chapter_08_LSM_Framework.md)
