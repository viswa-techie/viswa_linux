# Chapter 2: History and Evolution of Linux Security

## Learning Goals
- Understand early Unix security model and its limitations
- Know the evolution from simple permissions to modern frameworks
- Understand key milestones in Linux kernel security

---

## 2.1 Security in Early Unix Systems

```
1969-1970: Original Unix (Ken Thompson, Dennis Ritchie)
  - Multi-user system from the start
  - Simple but effective: UID/GID + rwx permissions
  - Root (UID 0) = all-powerful superuser
  - No mandatory access control
  - Trust model: users are cooperative (academic environment)

Unix Permission Model:
  ┌──────────────────────────────────────────────┐
  │  -rwxr-xr-- 1 user group 4096 file.txt      │
  │   │││││││││                                  │
  │   │││││││││   Owner: rwx (read, write, exec) │
  │   │││││││└─   Group: r-x (read, exec)        │
  │   │││││└───   Other: r-- (read only)          │
  │   ││││└────   type: - (file), d (dir), l (link)│
  │   │                                          │
  │   Special bits:                               │
  │     setuid (4000): Execute as file owner       │
  │     setgid (2000): Execute as file group       │
  │     sticky (1000): Only owner can delete       │
  └──────────────────────────────────────────────┘

  Key limitation: root bypasses ALL permission checks.
  One root compromise = total system compromise.
```

---

## 2.2 Traditional Unix Security Limitations

```
Problems with the Unix model:
  1. Root is all-or-nothing
     - ping needs ICMP raw socket → must be setuid root
     - But setuid root gives ALL privileges, not just ICMP
     - Any vulnerability in setuid binary → full root compromise

  2. DAC only (Discretionary)
     - File owner controls permissions
     - No system-wide mandatory policy
     - Compromised user can give away access to their files

  3. Coarse granularity
     - Only 3 permission sets: owner, group, other
     - No per-user ACLs (added later with POSIX ACLs)
     - No object labeling

  4. No process isolation
     - All processes in same UID share permissions
     - No way to restrict a process to a subset of user's rights
     - Compromised browser → access to all user files

  5. No audit trail designed in
     - Basic syslog, no structured security auditing
     - Hard to determine what happened post-breach
```

---

## 2.3 Evolution Timeline

```
1971    Unix permission model (rwx, UID/GID)
1983    Orange Book (TCSEC) — DoD security classifications
1988    Morris Worm — first major Unix security incident
1992    Linux 0.96 — basic Unix permissions only
1995    POSIX ACLs specification
1996    Linux 2.0 — netfilter precursors (ipfwadm)
1998    NSA starts SELinux research
1999    Linux 2.2 — capabilities infrastructure (partial)
2000    Netfilter/iptables in Linux 2.4
2001    SELinux released by NSA as open source
2002    Linux 2.5 — LSM framework merged
2003    Linux 2.6 — SELinux merged into mainline
2003    AppArmor open-sourced by Immunix/SUSE
2005    Audit subsystem merged
2005    Seccomp (strict mode) — kernel 2.6.12
2006    Smack merged (simplified MAC)
2008    TOMOYO merged
2009    AppArmor merged into mainline
2012    Seccomp-BPF — kernel 3.5 (programmable filtering)
2013    User namespaces — kernel 3.8 (unprivileged)
2015    Kernel lockdown patches proposed
2016    LoadPin LSM merged
2017    Kernel 4.14 — KPTI (Meltdown mitigation)
2018    Spectre/Meltdown hardware vulnerabilities disclosed
2019    Kernel 5.4 — Lockdown LSM merged
2020    Kernel 5.6 — WireGuard merged
2020    BPF LSM merged (programmable security)
2021    Kernel 5.13 — Landlock LSM merged (unprivileged sandboxing)
2022+   LSM stacking improvements, io_uring security restrictions
```

---

## 2.4 Key Security Milestones in Detail

```
LSM Framework (2002):
  Problem: Multiple MAC projects (SELinux, AppArmor, etc.)
           each patching the kernel differently
  Solution: Standard hook interface in the kernel
  Result: Any security module can plug into the same hooks
  Impact: Enabled coexistence of multiple security models

SELinux Mainlining (2003):
  Problem: No mandatory access control in mainline Linux
  Solution: NSA contributed SELinux to mainline kernel
  Result: Enterprise-grade MAC for Linux
  Impact: Foundation for Android security (since Android 4.3)

Seccomp-BPF (2012):
  Problem: Seccomp strict mode too restrictive (only read/write/exit)
  Solution: BPF programs filter syscalls with argument inspection
  Result: Per-process syscall whitelisting
  Impact: Foundation for Chrome sandbox, Docker, systemd

User Namespaces (2013):
  Problem: Unprivileged users can't create containers
  Solution: Map UID ranges — appear as root inside namespace
  Result: Rootless containers possible
  Impact: Enabled rootless Docker, Podman

KPTI (2017-2018):
  Problem: Meltdown vulnerability leaks kernel memory
  Solution: Separate kernel/user page tables
  Result: Kernel addresses not mapped in user space
  Impact: Performance cost ~5%, but essential security fix

Landlock (2021):
  Problem: No way for unprivileged apps to sandbox themselves
           (SELinux/AppArmor require root to configure)
  Solution: Unprivileged process can restrict its own access
  Result: Application-level sandboxing without root
  Impact: User-space sandboxing without admin privileges
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| security/security.c | LSM framework core (2002+) |
| security/selinux/ | SELinux implementation (2003+) |
| security/apparmor/ | AppArmor implementation (2009+) |
| security/landlock/ | Landlock LSM (2021+) |
| kernel/seccomp.c | Seccomp implementation |
| kernel/user_namespace.c | User namespaces |
| arch/x86/mm/pti.c | KPTI (Meltdown mitigation) |

---

## Interview Questions

**Q1: What are the limitations of the traditional Unix permission model?**
A: (1) Root is all-or-nothing — any setuid binary vulnerability gives full root. (2) DAC only — file owner controls access, no mandatory system-wide policy. (3) Coarse granularity — only owner/group/other, no per-user ACLs originally. (4) No process isolation — same UID processes share all privileges. (5) Setuid is dangerous — grants ALL root powers instead of specific capabilities.

**Q2: What is the LSM framework and why was it created?**
A: The Linux Security Modules framework (merged 2002) provides standard security hook points throughout the kernel. It was created because multiple MAC projects (SELinux, AppArmor, Smack) each maintained their own kernel patches. LSM provides a unified interface: ~200 hooks at security-sensitive operations (file access, socket operations, process control). Each security module registers callbacks for hooks it cares about. This allows different security models to coexist without conflicting kernel patches.

**Q3: Why was Seccomp-BPF significant?**
A: Original Seccomp (2005) was too restrictive — only allowed read(), write(), exit(), and sigreturn(). Seccomp-BPF (2012) attached BPF programs that can inspect syscall numbers and arguments, making fine-grained filtering practical. This enabled: Chrome sandbox (restrict renderer processes), Docker (container syscall filtering), systemd (service hardening). It's a critical defense-in-depth layer that reduces the kernel attack surface per-process.

---

## Summary

- Early Unix: UID/GID + rwx permissions, root bypasses all
- Key limitation: root is all-or-nothing, DAC only, no process isolation
- LSM (2002) standardized security hook interface for MAC modules
- SELinux (2003) brought mandatory access control to mainline
- Seccomp-BPF (2012) enabled per-process syscall filtering
- User namespaces (2013) enabled unprivileged container isolation
- KPTI (2018) mitigated Meltdown hardware vulnerability
- Landlock (2021) enabled unprivileged application sandboxing
- Modern Linux security is defense-in-depth: 10+ independent layers

---

Next: [Chapter 3 — Security Architecture in Linux](Chapter_03_Security_Architecture.md)
