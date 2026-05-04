# Chapter 26: Kernel Security Debugging

## Learning Goals
- Know kernel security auditing and logging tools
- Understand audit subsystem and its security role
- Know techniques for debugging security denials
- Understand security testing methodologies

---

## 26.1 Linux Audit Subsystem

```
Audit: Kernel-level logging of security-relevant events.

Architecture:
  ┌──────────────────────────────────────────────────┐
  │  Kernel                                           │
  │                                                   │
  │  Audit infrastructure (kernel/audit*)             │
  │    Hook points throughout kernel:                 │
  │      syscall entry/exit                           │
  │      file access                                  │
  │      SELinux/AppArmor denials                     │
  │      Network events                               │
  │      User authentication                          │
  │                                                   │
  │  Generates audit records → netlink → auditd       │
  └──────────────────────┬──────────────────────────┘
                          │ netlink
  ┌──────────────────────▼──────────────────────────┐
  │  User Space                                      │
  │                                                   │
  │  auditd — audit daemon                           │
  │    Writes to /var/log/audit/audit.log             │
  │                                                   │
  │  Tools:                                           │
  │    auditctl  — configure audit rules              │
  │    ausearch  — search audit logs                  │
  │    aureport  — generate audit reports             │
  │    autrace   — trace a program like strace        │
  └──────────────────────────────────────────────────┘

Audit record types:
  SYSCALL:    System call event
  AVC:        SELinux access vector cache denial
  APPARMOR:   AppArmor denial event
  PATH:       File path information
  EXECVE:     Program execution
  USER_AUTH:  User authentication
  USER_LOGIN: User login
  ANOM_*:     Anomaly events
```

---

## 26.2 Audit Rules and Configuration

```bash
# Watch file access:
auditctl -w /etc/passwd -p wa -k passwd_changes
# -w: watch file, -p: permissions (r,w,x,a), -k: key for searching

# Watch directory:
auditctl -w /etc/security/ -p wa -k security_config

# Monitor syscalls:
auditctl -a always,exit -F arch=b64 -S execve -k exec_log
# Log all program executions

auditctl -a always,exit -F arch=b64 -S mount -S umount2 -k mount_log
# Log all mount/unmount operations

# Monitor specific user:
auditctl -a always,exit -F arch=b64 -F uid=0 -S open -k root_open
# Log all files opened by root

# Search audit logs:
ausearch -k passwd_changes              # By key
ausearch -m AVC -ts recent              # SELinux denials, recent
ausearch -m EXECVE -ts today            # All executions today
ausearch -ua 1000                       # By user ID
ausearch -i -m USER_AUTH                # Authentication events (-i: interpret)

# Generate reports:
aureport --summary                      # Summary report
aureport -au                            # Authentication report
aureport -f                             # File access report
aureport --anomaly                      # Anomaly report

# Persistent rules (/etc/audit/rules.d/):
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-a always,exit -F arch=b64 -S execve -k exec
```

---

## 26.3 Debugging SELinux/AppArmor Denials

```bash
# SELinux denial debugging workflow:

# 1. Check for denials:
ausearch -m AVC -ts recent

# 2. Read the denial:
# type=AVC msg=audit(1234567890.123:456):
#   avc: denied { read } for pid=1234 comm="httpd"
#   name="config.php" dev="sda1" ino=67890
#   scontext=system_u:system_r:httpd_t:s0
#   tcontext=system_u:object_r:default_t:s0
#   tclass=file permissive=0

# 3. Diagnosis:
sealert -a /var/log/audit/audit.log  # Detailed analysis (RHEL)
audit2why -a                          # Explain why denial happened

# 4. Fix:
# Wrong label → restore:
restorecon -R /var/www/html/
# Missing policy → generate:
audit2allow -a -M my_fix
semodule -i my_fix.pp

# 5. Verify:
sesearch --allow -s httpd_t -t httpd_content_t -c file

# AppArmor denial debugging:
# Check syslog for DENIED:
grep "apparmor=\"DENIED\"" /var/log/syslog

# Or journal:
journalctl -k | grep "apparmor"

# Fix: update profile and reload:
aa-logprof                            # Interactive updates
apparmor_parser -r /etc/apparmor.d/usr.sbin.httpd
```

---

## 26.4 Security Testing Tools

```
Kernel security testing:

  syzkaller (Google):
    Kernel fuzzer — generates random syscall sequences.
    Finds: memory corruption, lock issues, info leaks.
    Has found 1000+ kernel bugs.
    Uses coverage-guided fuzzing (kcov).

  KASAN (Kernel Address Sanitizer):
    CONFIG_KASAN=y
    Detects: out-of-bounds access, use-after-free, double-free.
    ~2-3x memory overhead, significant performance impact.
    For development/testing only.

  KMSAN (Kernel Memory Sanitizer):
    CONFIG_KMSAN=y
    Detects: uninitialized memory reads.
    Catches info leak bugs (kernel stack → user space).

  KCSAN (Kernel Concurrency Sanitizer):
    CONFIG_KCSAN=y
    Detects: data races (concurrent unsynchronized access).

  KFENCE (Kernel Electric-Fence):
    CONFIG_KFENCE=y
    Low-overhead use-after-free/out-of-bounds detector.
    Suitable for production kernels (~1% overhead).
    Samples allocations and places guard pages.

  Static analysis:
    sparse (kernel-specific): make C=1     # run sparse
    smatch: Additional static checks
    Coccinelle: Automated pattern-based source transformation
    Clang static analyzer

  ┌──────────────────────────────────────────────────┐
  │  Testing Pyramid for Kernel Security              │
  │                                                   │
  │         syzkaller (fuzzing)                       │
  │        ┌───────────────┐                          │
  │       / Dynamic analysis \                        │
  │      ┌─────────────────────┐                      │
  │     / KASAN, KMSAN, KCSAN   \                     │
  │    ┌───────────────────────────┐                   │
  │   / Static analysis (sparse,   \                  │
  │  /  smatch, Clang analyzer)     \                 │
  │ ┌─────────────────────────────────┐               │
  │ │ Compiler hardening (FORTIFY,    │               │
  │ │ STACKPROTECTOR, CFI)            │               │
  │ └─────────────────────────────────┘               │
  └──────────────────────────────────────────────────┘
```

---

## 26.5 Security Tracing with ftrace/perf

```bash
# Trace security-related kernel functions:

# ftrace: trace LSM hooks
echo 'security_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe

# Trace capability checks:
echo 'cap_capable' > /sys/kernel/debug/tracing/set_ftrace_filter
# Shows every capability check in the kernel

# perf: security event profiling
perf stat -e 'syscalls:sys_enter_*' -a sleep 5
# Count all syscalls system-wide

# Trace specific security syscalls:
perf trace -e 'open,execve,connect,bind' -p $PID
# Monitor file/network access of a specific process

# BPF tracing for security:
bpftrace -e 'tracepoint:syscalls:sys_enter_execve {
    printf("%s executed %s\n", comm, str(args->filename));
}'
# Log all program executions with process name

bpftrace -e 'kprobe:cap_capable {
    printf("%s checking CAP_%d\n", comm, arg2);
}'
# Log all capability checks
```

---

## 26.6 Crash Analysis for Security Bugs

```
Analyzing security-related kernel crashes:

  KASAN report example:
    BUG: KASAN: slab-out-of-bounds in vulnerable_function+0x42/0x60
    Read of size 4 at addr ffff888012345678 by task exploit/1234
    Allocated by task victim/5678:
      kmalloc+0x...
      ...
    Freed by task cleanup/9012:  (if use-after-free)
      kfree+0x...

  Reading KASAN reports:
    1. Bug type: slab-out-of-bounds / use-after-free / null-ptr-deref
    2. Location: function+offset where access happened
    3. Allocation stack: where the object was created
    4. Free stack: where the object was freed (UAF)
    5. Task: which process triggered it

  kdump/crash for post-mortem analysis:
    # After kernel oops/panic:
    crash /var/crash/vmcore /usr/lib/debug/vmlinux

    crash> bt           # Backtrace
    crash> log           # Kernel log
    crash> ps            # Process list at crash time
    crash> struct cred <addr>  # Examine credentials
    crash> rd -d <addr> 64     # Read memory

  CVE analysis workflow:
    1. Read CVE description and affected versions
    2. Find the fix commit in kernel git
    3. Understand the vulnerability (what goes wrong)
    4. Check if your kernel is affected
    5. Apply patch or update kernel
    6. Verify fix with reproducer (if available)
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| kernel/audit.c | Audit framework core |
| kernel/auditsc.c | Audit system call logging |
| kernel/auditfilter.c | Audit rule filtering |
| mm/kasan/ | KASAN runtime |
| mm/kmsan/ | KMSAN runtime |
| kernel/kcsan/ | KCSAN runtime |
| mm/kfence/ | KFENCE runtime |

---

## Interview Questions

**Q1: How does the Linux audit subsystem work?**
A: The audit subsystem provides kernel-level logging of security events. Audit hooks throughout the kernel capture syscall entry/exit, file access, authentication events, SELinux/AppArmor denials, and more. Events are sent via netlink to the auditd daemon, which writes to /var/log/audit/audit.log. Administrators configure rules with `auditctl`: watch specific files (`-w /etc/passwd`), log syscalls (`-S execve`), filter by user/capability. `ausearch` queries logs by type, key, user, or time. `aureport` generates summary reports. For compliance (PCI-DSS, HIPAA), audit can log all privileged actions, authentication events, and configuration changes. Rules persist in /etc/audit/rules.d/.

**Q2: What kernel sanitizers are used for security testing?**
A: KASAN (Address Sanitizer) detects out-of-bounds reads/writes, use-after-free, and double-free — the memory corruption bugs that are the #1 source of kernel security vulnerabilities. ~2-3x memory overhead. KMSAN (Memory Sanitizer) detects reads of uninitialized memory — catches information leak bugs where kernel stack data is passed to user space. KCSAN (Concurrency Sanitizer) detects data races. KFENCE (Electric-Fence) is a low-overhead (~1%) production-suitable detector that samples allocations and places guard pages. These are complementary: KASAN for development testing, KFENCE for production, KMSAN for info leak prevention. Google's syzkaller fuzzer uses KASAN/KMSAN to find bugs in the kernel syscall surface.

**Q3: Describe your approach to debugging a SELinux denial.**
A: (1) Find the denial: `ausearch -m AVC -ts recent`. (2) Read the AVC message — identify scontext (process type, e.g., httpd_t), tcontext (target type, e.g., default_t), tclass (file/socket/etc.), and denied permission (read/write/etc.). (3) Use `audit2why -a` to explain the root cause — usually wrong file label, missing policy rule, or boolean needs toggling. (4) Fix based on cause: wrong label → `restorecon -R /path`. Missing policy → `audit2allow -a -M module && semodule -i module.pp`. Boolean needed → `setsebool -P httpd_can_network_connect 1`. (5) Verify: `sesearch --allow -s httpd_t -t target_t -c file`. (6) Test in permissive mode first if possible: `semanage permissive -a httpd_t`.

---

## Summary

- Audit subsystem: kernel-level logging of syscalls, file access, auth events
- auditctl/ausearch/aureport for rule management and log analysis
- SELinux debugging: ausearch -m AVC → audit2why → restorecon or audit2allow
- AppArmor debugging: grep DENIED in syslog → aa-logprof → reload profile
- KASAN/KMSAN/KCSAN/KFENCE: kernel sanitizers for development and production
- syzkaller: coverage-guided fuzzer, found 1000+ kernel bugs
- ftrace/bpftrace: trace LSM hooks, capability checks, security functions
- CVE analysis: read → find fix → check versions → patch → verify

---

Next: [Chapter 27 — Kernel Source Code for Security](Chapter_27_Source_Code_Map.md)
