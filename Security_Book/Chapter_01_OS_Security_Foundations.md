# Chapter 1: Foundations of Operating System Security

## Learning Goals
- Understand the CIA triad and core security goals
- Know threat models and attack surfaces in operating systems
- Understand security terminology and definitions
- Know defense-in-depth principles

---

## 1.1 What Is System Security

```
System security: Protecting computing resources against
unauthorized access, modification, denial of service, and destruction.

Security protects:
  - Data (at rest, in transit, in use)
  - Computation (correct execution)
  - Resources (CPU, memory, I/O, network)
  - Identity (authentication, authorization)

Why OS security matters:
  The OS is the TRUST BOUNDARY between hardware and software.
  If the kernel is compromised → everything above it is compromised.

  ┌─────────────────────────────────────────────────────┐
  │  Application (untrusted user code)                   │
  ├─────────────────────────────────────────────────────┤
  │  Libraries (libc, crypto, etc.)                      │
  ├═════════════════════════════════════════════════════┤
  │  KERNEL (trust boundary)                             │ ← If this falls,
  │  - Access control decisions                         │    everything falls
  │  - Resource management                              │
  │  - Hardware abstraction                             │
  ├─────────────────────────────────────────────────────┤
  │  Hardware (CPU, memory, I/O)                         │
  └─────────────────────────────────────────────────────┘
```

---

## 1.2 Security Goals — The CIA Triad

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│              Confidentiality                                 │
│             /              \                                 │
│            /    Security    \                                │
│           /      Goals       \                               │
│          /                    \                               │
│     Integrity ──────────── Availability                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘

Confidentiality:
  Only authorized entities can access information.
  Mechanisms: encryption, access control, permissions
  OS example: File permissions (rwx), SELinux labels, KPTI
  Attack: Information disclosure, side-channel (Spectre/Meltdown)

Integrity:
  Information cannot be modified by unauthorized entities.
  Mechanisms: checksums, signatures, MAC policies
  OS example: dm-verity (filesystem integrity), secure boot
  Attack: Privilege escalation, rootkit, code injection

Availability:
  Resources available to authorized users when needed.
  Mechanisms: resource limits, redundancy, DoS protection
  OS example: cgroups, rate limiting, OOM killer
  Attack: Denial of service, resource exhaustion, fork bomb

Additional goals:
  Authenticity:     Verifiable identity of entities
  Non-repudiation:  Cannot deny having performed an action
  Accountability:   Actions traceable to responsible entity
```

---

## 1.3 Threat Models

```
Threat model: Systematic analysis of who might attack,
what they target, and how they attack.

Attacker types:
  ┌──────────────────────────────────────────────────────────┐
  │ Attacker          │ Capability    │ Target               │
  ├────────────────────┼───────────────┼──────────────────────┤
  │ Remote network     │ Network access│ Services, daemons    │
  │ Local unprivileged │ User shell    │ Kernel, root privs   │
  │ Local privileged   │ Root shell    │ Kernel, other root   │
  │ Physical access    │ Hardware      │ Boot, firmware, DMA  │
  │ Supply chain       │ Build system  │ Source, binaries      │
  │ Insider           │ Dev access    │ Backdoors, data       │
  └────────────────────┴───────────────┴──────────────────────┘

Privilege escalation:
  Vertical:   Unprivileged user → root (most common kernel attack)
  Horizontal: User A accesses User B's data

Common kernel attack vectors:
  1. System call interface (primary attack surface)
  2. /proc, /sys, /dev interfaces
  3. Network packet processing
  4. Device drivers (largest codebase, most bugs)
  5. Race conditions (TOCTOU)
  6. Memory corruption (buffer overflow, use-after-free)
  7. Side channels (Spectre, Meltdown, cache timing)
```

---

## 1.4 Attack Surfaces in Operating Systems

```
Attack surface: All points where an attacker can interact with the system.

Linux kernel attack surface:
  ┌───────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌─────────────┐   ┌──────────┐   ┌──────────────────┐  │
  │  │ Syscall     │   │ /proc    │   │ Network stack    │  │
  │  │ Interface   │   │ /sys     │   │ (packet parsing) │  │
  │  │ (400+ calls)│   │ /dev     │   │                  │  │
  │  └──────┬──────┘   └────┬─────┘   └────────┬─────────┘  │
  │         │               │                   │            │
  │         ▼               ▼                   ▼            │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │              KERNEL                              │    │
  │  │                                                  │    │
  │  │  Drivers (60%+ of kernel code, most vulns)       │    │
  │  │  File systems                                    │    │
  │  │  Memory management                               │    │
  │  │  Process scheduler                                │    │
  │  └──────────────────────────────────────────────────┘    │
  │         │               │                   │            │
  │         ▼               ▼                   ▼            │
  │  ┌──────────┐   ┌──────────┐   ┌────────────────────┐   │
  │  │ Hardware │   │ Firmware │   │ Physical interface │   │
  │  │ (DMA)    │   │ (UEFI)   │   │ (USB, PCIe, JTAG) │   │
  │  └──────────┘   └──────────┘   └────────────────────┘   │
  │                                                           │
  └───────────────────────────────────────────────────────────┘

Reducing attack surface:
  1. Minimize enabled kernel modules (CONFIG_*)
  2. Disable unnecessary syscalls (seccomp)
  3. Restrict /proc, /sys access
  4. Use network namespaces to isolate
  5. Principle of least privilege (capabilities, MAC)
```

---

## 1.5 Defense in Depth

```
Defense in depth: Multiple layers of security.
If one layer fails, others still protect.

  ┌──────────── Defense Layers ─────────────────┐
  │                                              │
  │  Layer 1: Authentication (PAM, passwords)    │
  │  Layer 2: DAC (file permissions, UID/GID)    │
  │  Layer 3: Capabilities (fine-grained privs)  │
  │  Layer 4: MAC (SELinux, AppArmor)            │
  │  Layer 5: Seccomp (syscall filtering)        │
  │  Layer 6: Namespaces (isolation)             │
  │  Layer 7: Memory protection (ASLR, NX, KPTI)│
  │  Layer 8: Kernel hardening (CFI, stack guard)│
  │  Layer 9: Secure boot (firmware → kernel)    │
  │  Layer 10: Hardware (TPM, TrustZone, IOMMU)  │
  │                                              │
  └──────────────────────────────────────────────┘

  Principle: No single mechanism is sufficient.
  Example: Even if attacker bypasses seccomp,
           SELinux + capabilities + ASLR still protect.
```

---

## 1.6 Security Terminology

```
Authentication:      Verifying identity ("who are you?")
Authorization:       Checking permissions ("are you allowed?")
Access Control:      Mechanism that enforces authorization
Audit:               Recording security-relevant events
Credential:          Proof of identity (password, token, key)
Exploit:             Code that uses a vulnerability
Privilege:           Permission to perform a sensitive operation
Vulnerability:       Weakness that can be exploited
Zero-day:            Vulnerability unknown to vendor
CVE:                 Common Vulnerabilities and Exposures ID
CVSS:                Common Vulnerability Scoring System (0-10)
Sandbox:             Restricted execution environment
TCB:                 Trusted Computing Base — components that must be correct
                     for security to hold (smaller = better)
TOCTOU:              Time-of-Check to Time-of-Use race condition
ROP:                 Return-Oriented Programming — bypass NX with gadgets
Buffer overflow:     Writing beyond buffer bounds (stack, heap)
Use-after-free:      Using memory after it's freed
Information leak:    Unintended data disclosure (kernel → user)
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| security/security.c | Core security framework |
| security/commoncap.c | Common capability checks |
| kernel/cred.c | Credential management |
| include/linux/security.h | Security hook declarations |
| include/linux/cred.h | Credential structures |

---

## Interview Questions

**Q1: What is the CIA triad?**
A: Confidentiality (only authorized access), Integrity (no unauthorized modification), Availability (resources accessible when needed). These are the three core security goals. In the Linux kernel: file permissions and encryption provide confidentiality, dm-verity and secure boot provide integrity, cgroups and resource limits provide availability.

**Q2: What is the primary attack surface of the Linux kernel?**
A: The syscall interface (~400+ system calls) is the primary attack surface since every unprivileged user can invoke it. Beyond that: /proc, /sys, /dev pseudo-filesystems, the network stack (packet parsing), and device drivers (largest codebase, most vulnerabilities). Reducing attack surface involves seccomp (filter syscalls), hiding /proc entries, disabling unused modules, and using namespaces for isolation.

**Q3: What is defense in depth?**
A: Multiple overlapping security layers so that compromise of one layer doesn't compromise the system. In Linux: authentication (PAM) → DAC (permissions) → capabilities → MAC (SELinux) → seccomp → namespaces → memory protections (ASLR, NX) → kernel hardening → secure boot → hardware (TPM). Each layer independently limits attacker capability.

---

## Summary

- OS security protects data, computation, resources, and identity
- CIA triad: Confidentiality, Integrity, Availability
- Threat models define attackers, capabilities, and targets
- Primary kernel attack surface: syscalls, /proc, network, drivers
- Defense in depth: multiple independent security layers
- TCB should be minimized — fewer trusted components means fewer vulnerabilities

---

Next: [Chapter 2 — History and Evolution of Linux Security](Chapter_02_History_Evolution.md)
