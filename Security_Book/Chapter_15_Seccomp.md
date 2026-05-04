# Chapter 15: Seccomp (Secure Computing Mode)

## Learning Goals
- Understand Seccomp's system call filtering mechanism
- Know Seccomp-BPF filter programming
- Understand how containers use Seccomp for sandboxing
- Know the kernel implementation of Seccomp

---

## 15.1 Seccomp Architecture

```
Seccomp: Restricts which system calls a process can make.
Most important sandboxing mechanism in Linux.

Two modes:
  Mode 1 (Strict):  Only read(), write(), exit(), sigreturn()
  Mode 2 (Filter):  BPF program decides per-syscall (Seccomp-BPF)

Architecture:
  ┌──────────────────────────────────────────────────────┐
  │  User Space                                           │
  │                                                       │
  │  Application calls:                                   │
  │    prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog)  │
  │    or                                                 │
  │    seccomp(SECCOMP_SET_MODE_FILTER, flags, &prog)     │
  │                                                       │
  │  BPF filter program:                                  │
  │    Examines: syscall number, architecture, arguments  │
  │    Returns:  ALLOW, KILL, TRAP, ERRNO, TRACE, LOG     │
  └───────────────────────┬──────────────────────────────┘
                           │
  ┌───────────────────────▼──────────────────────────────┐
  │  Kernel: Seccomp Layer                                │
  │                                                       │
  │  syscall entry point (entry.S)                        │
  │    ↓                                                  │
  │  __secure_computing()    ← called before syscall exec │
  │    ↓                                                  │
  │  seccomp_run_filters()   ← run BPF filter chain       │
  │    ↓                                                  │
  │  Decision:                                            │
  │    SECCOMP_RET_ALLOW  → proceed with syscall          │
  │    SECCOMP_RET_KILL   → SIGSYS, kill thread/process   │
  │    SECCOMP_RET_TRAP   → SIGSYS, caught by sighandler  │
  │    SECCOMP_RET_ERRNO  → return -errno to caller       │
  │    SECCOMP_RET_TRACE  → notify ptrace tracer          │
  │    SECCOMP_RET_LOG    → allow but log                 │
  │    SECCOMP_RET_USER_NOTIF → notify userspace manager  │
  └──────────────────────────────────────────────────────┘

Key property: Filters can only be ADDED, never removed.
  Once installed, a filter stays for the process lifetime.
  Child processes inherit parent's filters.
  Filters chain: most restrictive wins (AND logic).
```

---

## 15.2 Seccomp-BPF Filter Programming

```c
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <linux/audit.h>
#include <sys/prctl.h>

/*
 * BPF filter: Allow only read, write, exit, sigreturn.
 * Kill process on any other syscall.
 */
static struct sock_filter filter[] = {
    /* Load architecture */
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
             offsetof(struct seccomp_data, arch)),
    /* Verify architecture is x86_64 */
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, AUDIT_ARCH_X86_64, 1, 0),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),

    /* Load syscall number */
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
             offsetof(struct seccomp_data, nr)),

    /* Allow read (0) */
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    /* Allow write (1) */
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    /* Allow exit (60) */
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    /* Allow exit_group (231) */
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit_group, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),

    /* Kill on anything else */
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),
};

static struct sock_fprog prog = {
    .len = sizeof(filter) / sizeof(filter[0]),
    .filter = filter,
};

int main(void) {
    /* Required: cannot gain new privileges */
    prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);

    /* Install seccomp filter */
    prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);

    /* From here, only read/write/exit allowed */
    write(1, "Hello\n", 6);  /* OK */
    /* open("/etc/passwd", O_RDONLY); would be killed */
    return 0;
}
```

---

## 15.3 seccomp_data Structure

```
BPF filter operates on this structure:

  struct seccomp_data {
      int   nr;           /* System call number */
      __u32 arch;         /* AUDIT_ARCH_* value */
      __u64 instruction_pointer;  /* CPU IP */
      __u64 args[6];      /* Syscall arguments */
  };

  Filter can examine:
    - Which syscall is being made (nr)
    - Architecture (important for compat syscalls)
    - All 6 arguments to the syscall
    - Instruction pointer (for JIT analysis)

  Limitations:
    - Cannot dereference pointers in args
    - args[0] might be a pointer to a filename, but
      filter cannot read the filename string
    - Can only check numeric argument values

  seccomp_data layout at BPF offsets:
    ┌─────────────┬────────┬──────────────────┐
    │ Offset      │ Size   │ Field            │
    ├─────────────┼────────┼──────────────────┤
    │ 0           │ 4      │ nr (syscall num) │
    │ 4           │ 4      │ arch             │
    │ 8           │ 8      │ instruction_ptr  │
    │ 16          │ 8      │ args[0]          │
    │ 24          │ 8      │ args[1]          │
    │ 32          │ 8      │ args[2]          │
    │ 40          │ 8      │ args[3]          │
    │ 48          │ 8      │ args[4]          │
    │ 56          │ 8      │ args[5]          │
    └─────────────┴────────┴──────────────────┘
```

---

## 15.4 Seccomp in Containers

```
Docker/Kubernetes default seccomp profiles block dangerous syscalls.

Docker default profile blocks (partial list):
  - mount, umount2           No filesystem mounting
  - reboot                   No system reboot
  - kexec_load               No kernel replacement
  - init_module, delete_module  No kernel modules
  - pivot_root               No root filesystem change
  - swapon, swapoff          No swap management
  - keyctl                   No kernel keyring manipulation
  - ptrace                   No process tracing
  - userfaultfd              No userfaultfd (exploit primitive)

  Total: blocks ~44 of ~300+ syscalls

Docker usage:
  # Run with default seccomp profile
  docker run --security-opt seccomp=default.json myimage

  # Run with custom profile (JSON)
  docker run --security-opt seccomp=my-profile.json myimage

  # Disable seccomp (dangerous)
  docker run --security-opt seccomp=unconfined myimage

Kubernetes:
  apiVersion: v1
  kind: Pod
  spec:
    securityContext:
      seccompProfile:
        type: RuntimeDefault    # Use container runtime's default

  # Or custom profile:
      seccompProfile:
        type: Localhost
        localhostProfile: profiles/my-profile.json
```

---

## 15.5 User Notification (SECCOMP_RET_USER_NOTIF)

```
Seccomp user notification: delegate syscall decisions to user space.

  Process installs filter with SECCOMP_RET_USER_NOTIF for some syscalls.
  When those syscalls occur, kernel notifies a supervisor process.
  Supervisor can: inspect, emulate, allow, or deny the syscall.

  Use case: Container manager handles mount() on behalf of container.

  ┌──────────┐  mount()  ┌──────────┐  notify   ┌───────────┐
  │ Container│ ─────────►│ Kernel   │ ─────────►│ Supervisor│
  │ Process  │           │ Seccomp  │           │ (runtime) │
  └──────────┘           └──────────┘           └─────┬─────┘
                              ▲                        │
                              │  respond (allow/deny)  │
                              └────────────────────────┘

  API:
    seccomp(SECCOMP_SET_MODE_FILTER, SECCOMP_FILTER_FLAG_NEW_LISTENER, &prog)
    → returns notification fd

    Supervisor:
      ioctl(notif_fd, SECCOMP_IOCTL_NOTIF_RECV, &req)    // receive
      ioctl(notif_fd, SECCOMP_IOCTL_NOTIF_SEND, &resp)   // respond
```

---

## 15.6 Kernel Implementation

```c
/* kernel/seccomp.c — core seccomp implementation */

/* Called on every syscall entry */
int __secure_computing(const struct seccomp_data *sd)
{
    int this_syscall;
    struct seccomp_filter *f;

    /* Check if seccomp is enabled for this task */
    if (!current->seccomp.mode)
        return 0;

    this_syscall = sd->nr;

    switch (current->seccomp.mode) {
    case SECCOMP_MODE_STRICT:
        /* Only allow read, write, exit, sigreturn */
        /* ... */
    case SECCOMP_MODE_FILTER:
        return seccomp_run_filters(sd);
    }
}

/* Run the BPF filter chain */
static u32 seccomp_run_filters(const struct seccomp_data *sd)
{
    struct seccomp_filter *f;
    u32 ret = SECCOMP_RET_ALLOW;

    /* Walk filter chain, most restrictive wins */
    for (f = current->seccomp.filter; f; f = f->prev) {
        u32 cur_ret = bpf_prog_run(f->prog, sd);
        if ((cur_ret & SECCOMP_RET_ACTION_FULL) <
            (ret & SECCOMP_RET_ACTION_FULL))
            ret = cur_ret;
    }
    return ret;
}
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| kernel/seccomp.c | Core seccomp implementation |
| include/linux/seccomp.h | Seccomp definitions |
| include/uapi/linux/seccomp.h | User-space API (modes, flags, return values) |
| arch/x86/entry/common.c | Syscall entry calling __secure_computing() |
| tools/testing/selftests/seccomp/ | Kernel selftests |

---

## Interview Questions

**Q1: How does Seccomp-BPF work?**
A: A process installs a BPF (Berkeley Packet Filter) program that runs on every syscall entry. The filter examines the syscall number, architecture, and arguments (via `struct seccomp_data`). It returns a decision: ALLOW, KILL (SIGSYS), ERRNO (return error), TRAP (signal handler), TRACE (ptrace), or LOG (allow but log). Filters can only be added, never removed — they persist for the process lifetime and are inherited by children. Multiple filters chain together with AND logic — the most restrictive result wins. Before installing a filter, `PR_SET_NO_NEW_PRIVS` must be set to prevent privilege escalation.

**Q2: Why can't Seccomp filters dereference pointer arguments?**
A: Seccomp filters run in the kernel at syscall entry time and operate on a snapshot of `seccomp_data` containing only the numeric values of the 6 syscall arguments. If an argument is a pointer (e.g., the filename in `open()`), the filter only sees the pointer address, not the string it points to. Dereferencing user-space pointers from BPF would create TOCTOU (time-of-check-time-of-use) vulnerabilities — the user could change the pointed-to data between the filter check and the actual syscall execution. This is a fundamental design decision for safety. For pointer argument inspection, use `SECCOMP_RET_USER_NOTIF` to let a supervisor process safely inspect via `/proc/pid/mem`.

**Q3: How do containers use Seccomp for security?**
A: Docker applies a default Seccomp profile that blocks ~44 dangerous syscalls: `mount`, `reboot`, `kexec_load`, `init_module`, `ptrace`, `userfaultfd`, etc. These are syscalls that could allow container escape or host compromise. The profile is a JSON file listing allowed/blocked syscalls. Kubernetes supports Seccomp via `securityContext.seccompProfile` with RuntimeDefault (container runtime's profile) or Localhost (custom profile). Custom profiles can be more restrictive for high-security workloads. Seccomp is applied before LSM checks, making it the first line of defense — it blocks the syscall before any SELinux/AppArmor check even runs.

---

## Summary

- Seccomp restricts which syscalls a process can make
- Mode 1 (strict): only read/write/exit/sigreturn
- Mode 2 (filter): BPF program decides per-syscall
- seccomp_data: syscall number, arch, arguments (no pointer deref)
- Return values: ALLOW, KILL, TRAP, ERRNO, TRACE, LOG, USER_NOTIF
- Filters are additive-only — can never be removed
- Docker default blocks ~44 dangerous syscalls
- USER_NOTIF: delegate syscall decisions to supervisor process
- Runs before LSM hooks — first layer of defense

---

Next: [Chapter 16 — Namespaces and Security Isolation](Chapter_16_Namespaces.md)
