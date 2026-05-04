# Chapter 31: Security Models in Other Operating Systems

## Learning Goals
- Compare Linux security with Windows, macOS, and other OS security models
- Understand unique security features of each OS
- Know similarities and differences for interview discussions

---

## 31.1 Windows Security Model

```
Windows security is built on different fundamentals than Linux.

  ┌──────────────────────────────────────────────────────┐
  │  Windows Security Architecture                       │
  │                                                       │
  │  Security Reference Monitor (SRM)                     │
  │    Kernel component enforcing access control          │
  │    Equivalent role to Linux LSM + DAC combined        │
  │                                                       │
  │  Access Control:                                       │
  │    Security Descriptors on every object               │
  │    DACL: Discretionary ACL (who can access)           │
  │    SACL: System ACL (auditing rules)                  │
  │    Owner + primary group                              │
  │                                                       │
  │  Tokens:                                               │
  │    Every process has an Access Token                   │
  │    Contains: SID (user), group SIDs, privileges        │
  │    Similar to Linux struct cred                        │
  │                                                       │
  │  Mandatory Integrity Control (MIC):                    │
  │    4 trust levels: Untrusted, Low, Medium, High, System│
  │    Cannot write to higher integrity objects             │
  │    Browser runs at Low → can't modify system files     │
  └──────────────────────────────────────────────────────┘

  Linux vs Windows comparison:
  ┌──────────────────┬──────────────────┬──────────────────┐
  │ Concept          │ Linux            │ Windows           │
  ├──────────────────┼──────────────────┼──────────────────┤
  │ Identity         │ UID/GID          │ SID               │
  │ Permissions      │ rwx + ACLs       │ DACL/SACL         │
  │ Privileges       │ Capabilities     │ Privileges (SE_*) │
  │ MAC              │ SELinux/AppArmor │ MIC + Defender    │
  │ Credentials      │ struct cred      │ Access Token      │
  │ Admin            │ root (UID 0)     │ Administrator SID │
  │ Least privilege  │ Capabilities     │ UAC + MIC         │
  │ Sandboxing       │ Namespaces+Seccomp│ AppContainers    │
  │ Integrity        │ IMA/dm-verity    │ Code Integrity    │
  │ Secure Boot      │ UEFI + shim      │ UEFI + WHQL      │
  └──────────────────┴──────────────────┴──────────────────┘
```

---

## 31.2 macOS/iOS Security Model

```
Apple's security model — hardware + software integration.

  ┌──────────────────────────────────────────────────────┐
  │  macOS/iOS Security Architecture                      │
  │                                                       │
  │  Hardware:                                            │
  │    Secure Enclave (separate processor for keys)       │
  │    T2/M1+ chip: hardware root of trust                │
  │    Touch ID / Face ID biometrics                      │
  │                                                       │
  │  Kernel (XNU):                                         │
  │    Mandatory Access Control (TrustedBSD MAC framework)│
  │    Sandboxing: Seatbelt profiles (like AppArmor)      │
  │    Entitlements: capability-like permissions           │
  │    Code signing: ALL code must be signed               │
  │                                                       │
  │  Application level:                                    │
  │    App Sandbox: mandatory for App Store apps           │
  │    Entitlements: declared permissions                  │
  │    Hardened Runtime: W^X, library validation           │
  │    Gatekeeper: verify downloaded app signatures        │
  │    Notarization: Apple-verified apps                   │
  │    SIP: System Integrity Protection (protect OS files) │
  └──────────────────────────────────────────────────────┘

  Linux vs macOS/iOS comparison:
  ┌──────────────────┬──────────────────┬──────────────────┐
  │ Concept          │ Linux            │ macOS/iOS         │
  ├──────────────────┼──────────────────┼──────────────────┤
  │ MAC framework    │ LSM              │ TrustedBSD MAC    │
  │ Sandboxing       │ Seccomp+NS       │ Seatbelt sandbox  │
  │ Capabilities     │ POSIX caps       │ Entitlements      │
  │ Code signing     │ Optional (IMA)   │ Mandatory         │
  │ Hardware keys    │ TPM (optional)   │ Secure Enclave    │
  │ OS protection    │ Lockdown/IMA     │ SIP               │
  │ App verification │ Package manager  │ Gatekeeper/Notarize│
  │ Process isolation│ Namespaces       │ App Sandbox        │
  │ Disk encryption  │ dm-crypt/LUKS    │ FileVault (T2/SE) │
  └──────────────────┴──────────────────┴──────────────────┘
```

---

## 31.3 BSD Security Features

```
FreeBSD / OpenBSD — security-focused Unix variants.

  FreeBSD:
    TrustedBSD MAC framework (inspired macOS security)
    Jails — precursor to Linux containers/namespaces
    Capsicum — capability-based sandboxing
    GELI — disk encryption
    securelevel — restrict kernel modifications

  OpenBSD — security as primary design goal:
    W^X enforced by default system-wide
    pledge() — declare syscalls program will use (like seccomp)
    unveil() — declare filesystem paths program will access
    arc4random() — secure random everywhere
    Stack protection enabled by default
    ASLR on all architectures
    Minimal default installation
    Heavy code auditing culture

  Linux vs BSD:
  ┌──────────────────┬──────────────────┬──────────────────┐
  │ Feature          │ Linux            │ OpenBSD           │
  ├──────────────────┼──────────────────┼──────────────────┤
  │ Syscall filter   │ Seccomp-BPF      │ pledge()          │
  │ Path restriction │ Landlock         │ unveil()          │
  │ W^X              │ CONFIG option    │ Default, strict   │
  │ Random           │ ChaCha20-CSPRNG  │ arc4random()      │
  │ Containers       │ Namespaces       │ N/A               │
  │ Jails            │ N/A              │ chroot (limited)  │
  │ Audit            │ audit subsystem  │ syslog-based       │
  │ Mitigations      │ Many CONFIG opts │ Default-on         │
  └──────────────────┴──────────────────┴──────────────────┘
```

---

## 31.4 RTOS and Embedded OS Security

```
Real-Time OS security — different constraints than general-purpose OS.

  QNX (BlackBerry):
    Microkernel — minimal attack surface in kernel
    Processes communicate via IPC messages
    POSIX DAC + custom policies
    Used in automotive (ISO 26262 certified)
    Each service isolated in separate process
    Service crashes don't affect kernel

  Zephyr RTOS:
    Memory protection via MPU (Memory Protection Unit)
    Thread isolation (kernel threads vs user threads)
    Hardware stack overflow detection
    Crypto library: mbedTLS integration
    No file system security (often no filesystem)

  FreeRTOS:
    MPU support for task isolation
    Limited — no MAC, no DAC
    Trust boundaries defined by hardware (TrustZone)
    AWS FreeRTOS adds TLS + secure OTA

  ┌──────────────────┬──────────────────┬──────────────────┐
  │ Feature          │ Linux            │ RTOS (embedded)   │
  ├──────────────────┼──────────────────┼──────────────────┤
  │ Isolation model  │ Process + NS     │ MPU / TrustZone   │
  │ MAC              │ SELinux/AppArmor │ None or custom     │
  │ File security    │ DAC + ACLs       │ Often no FS        │
  │ Networking       │ Full stack       │ Minimal (lwIP)     │
  │ Attack surface   │ Large            │ Small (microkernel)│
  │ Secure boot      │ UEFI + dm-verity │ Hardware bootrom   │
  │ Crypto           │ Kernel Crypto API│ mbedTLS/wolfSSL    │
  │ Updates          │ APT/dnf/OTA      │ Secure OTA (A/B)   │
  └──────────────────┴──────────────────┴──────────────────┘
```

---

## 31.5 seL4 (Formally Verified Microkernel)

```
seL4: Only OS kernel with complete formal verification.
Mathematical proof that kernel implementation matches specification.

  Verified properties:
    - Functional correctness: behaves exactly as specified
    - Integrity: memory isolation between components
    - Confidentiality: no information leakage between components
    - Availability: guaranteed response times

  ┌──────────────────────────────────────────────────────┐
  │  seL4 Architecture                                    │
  │                                                       │
  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐               │
  │  │ App1 │ │ App2 │ │ FS   │ │ Net  │  User space   │
  │  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘               │
  │     │        │        │        │                     │
  │  ───┼────────┼────────┼────────┼──── IPC messages    │
  │     │        │        │        │                     │
  │  ┌──▼────────▼────────▼────────▼───┐                 │
  │  │          seL4 Microkernel        │                 │
  │  │  ~10,000 lines of C              │                 │
  │  │  IPC + scheduling + memory mgmt  │                 │
  │  │  Capability-based access control │                 │
  │  └─────────────────────────────────┘                 │
  └──────────────────────────────────────────────────────┘

  Linux comparison:
    Linux kernel: ~30M LOC, never fully verified
    seL4: ~10K LOC, mathematically proven correct
    Trade-off: Linux has rich features, seL4 has provable security
    seL4 used in: military, aerospace, automotive (safety-critical)
```

---

## Summary

- Windows: SRM, DACL/SACL, SIDs, MIC integrity levels, UAC, AppContainers
- macOS/iOS: TrustedBSD MAC, Seatbelt sandbox, entitlements, Secure Enclave, SIP, mandatory code signing
- OpenBSD: pledge(), unveil(), W^X by default, security-first culture
- FreeBSD: TrustedBSD MAC, Jails, Capsicum capability sandboxing
- RTOS: MPU/TrustZone isolation, small attack surface, hardware-based security
- seL4: formally verified microkernel — mathematical proof of correctness
- Linux has the most comprehensive security framework, but also the largest attack surface

---

Next: [Chapter 32 — Embedded Linux Security](Chapter_32_Embedded_Security.md)
