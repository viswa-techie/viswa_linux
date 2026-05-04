# Linux Security Subsystem — Complete Book

## Master Index

A comprehensive guide covering Linux kernel security architecture, access control models, security frameworks, isolation mechanisms, hardening, and embedded system security — with interview preparation throughout.

---

## Part I: Foundations

| # | Chapter | Key Topics |
|---|---------|------------|
| 01 | [Foundations of OS Security](Chapter_01_OS_Security_Foundations.md) | CIA triad, threat models, attack surfaces, terminology |
| 02 | [History and Evolution of Linux Security](Chapter_02_History_Evolution.md) | Unix permissions, Linux security timeline, modern frameworks |
| 03 | [Security Architecture in Linux](Chapter_03_Security_Architecture.md) | Kernel vs userspace, enforcement points, policy vs mechanism |

## Part II: Access Control Basics

| # | Chapter | Key Topics |
|---|---------|------------|
| 04 | [Authentication Mechanisms](Chapter_04_Authentication.md) | PAM, passwords, multi-factor, login flow |
| 05 | [User and Group Management](Chapter_05_Users_Groups.md) | UID/GID, privileged users, credential structures |
| 06 | [File System Security](Chapter_06_Filesystem_Security.md) | Permissions, ownership, ACLs, special bits |
| 07 | [Linux Capabilities](Chapter_07_Capabilities.md) | Capability sets, checks, dropping root |

## Part III: Security Frameworks

| # | Chapter | Key Topics |
|---|---------|------------|
| 08 | [Linux Security Modules (LSM)](Chapter_08_LSM_Framework.md) | Architecture, hooks, stacking, registration |
| 09 | [Mandatory Access Control (MAC)](Chapter_09_MAC.md) | MAC model, Bell-LaPadula, Biba, policy enforcement |
| 10 | [Discretionary Access Control (DAC)](Chapter_10_DAC.md) | Traditional Unix, owner-based, limitations |
| 11 | [SELinux](Chapter_11_SELinux.md) | Type enforcement, contexts, RBAC, policy language |
| 12 | [AppArmor](Chapter_12_AppArmor.md) | Profile-based, path-based, application confinement |
| 13 | [Smack Security Module](Chapter_13_Smack.md) | Label-based, simple MAC, IoT/embedded |
| 14 | [TOMOYO Security Module](Chapter_14_TOMOYO.md) | Domain-based, learning mode, pathname-based |

## Part IV: Isolation and Sandboxing

| # | Chapter | Key Topics |
|---|---------|------------|
| 15 | [Seccomp](Chapter_15_Seccomp.md) | Syscall filtering, BPF policies, container use |
| 16 | [Namespaces and Security Isolation](Chapter_16_Namespaces.md) | PID/NET/MNT/USER/UTS/IPC namespaces, container isolation |
| 17 | [Control Groups (cgroups)](Chapter_17_Cgroups.md) | Resource limits, DoS prevention, v1 vs v2 |

## Part V: Kernel Hardening

| # | Chapter | Key Topics |
|---|---------|------------|
| 18 | [Kernel Address Space Protection](Chapter_18_Address_Space_Protection.md) | ASLR, KASLR, stack canaries |
| 19 | [Memory Protection Mechanisms](Chapter_19_Memory_Protection.md) | NX bit, SMEP/SMAP, KPTI, stack protector |
| 20 | [Secure Boot](Chapter_20_Secure_Boot.md) | UEFI, chain of trust, kernel signing, dm-verity |

## Part VI: Cryptography and Network Security

| # | Chapter | Key Topics |
|---|---------|------------|
| 21 | [Cryptographic Services in Linux](Chapter_21_Crypto_Services.md) | Kernel crypto API, algorithms, hw acceleration |
| 22 | [Key Management](Chapter_22_Key_Management.md) | Keyring service, key types, kernel key API |
| 23 | [Network Security in Kernel](Chapter_23_Network_Security.md) | Netfilter, IPsec, TLS, packet filtering |

## Part VII: Advanced Topics

| # | Chapter | Key Topics |
|---|---------|------------|
| 24 | [Container Security](Chapter_24_Container_Security.md) | Docker, Kubernetes, runtime security, breakouts |
| 25 | [Kernel Security Hardening](Chapter_25_Kernel_Hardening.md) | Exploit mitigation, sysctl, lockdown, CFI |
| 26 | [Kernel Security Debugging](Chapter_26_Security_Debugging.md) | Audit subsystem, logging, vulnerability analysis |

## Part VIII: Reference and Interview

| # | Chapter | Key Topics |
|---|---------|------------|
| 27 | [Kernel Source Code for Security](Chapter_27_Source_Code_Map.md) | security/, LSM files, directory structure |
| 28 | [Security Flow Diagrams](Chapter_28_Flow_Diagrams.md) | Access control flow, LSM hooks, SELinux decisions |
| 29 | [Important Security Diagrams](Chapter_29_Architecture_Diagrams.md) | Architecture diagrams, container isolation, hardening |
| 30 | [Definitions of Important Terms](Chapter_30_Glossary.md) | 60+ security terms defined |
| 31 | [Security in Other Operating Systems](Chapter_31_Other_OS_Security.md) | Windows, macOS, QNX, RTOS security models |
| 32 | [Embedded Linux Security](Chapter_32_Embedded_Security.md) | Secure boot, device auth, firmware integrity |
| 33 | [Documentation and References](Chapter_33_References.md) | Papers, books, RFCs, kernel docs |
| 34 | [Interview Preparation](Chapter_34_Interview_Preparation.md) | Q&A, scenarios, coding, checklist |

---

## Reading Paths

```
Embedded/Automotive Engineer:
  Ch 1-3 → Ch 6-7 → Ch 8,11 → Ch 15-17 → Ch 18-20 → Ch 25 → Ch 32 → Ch 34

Kernel Security Developer:
  Ch 1-3 → Ch 7-14 → Ch 15 → Ch 18-19 → Ch 21-22 → Ch 25-26 → Ch 27-28 → Ch 34

Container/Cloud Engineer:
  Ch 1,3 → Ch 7-8 → Ch 11-12 → Ch 15-17 → Ch 24 → Ch 25 → Ch 34

Interview Preparation (Fast Track):
  Ch 3 → Ch 7 → Ch 8 → Ch 11 → Ch 15 → Ch 18-19 → Ch 24-25 → Ch 34
```

---

## Security Stack Overview

```
┌──────────────────────────── User Space ────────────────────────────┐
│                                                                    │
│  Applications    PAM       login/sshd     Container Runtime        │
│      │           │            │                │                   │
│      └─── Authentication ────┘                 │                   │
│           (Ch 4-5)                             │                   │
│                                                │                   │
│  File Access    Capabilities    Seccomp        │                   │
│  (Ch 6)         (Ch 7)         (Ch 15)         │                   │
│                                                │                   │
└────────────────────── syscall boundary ────────────────────────────┘
│                                                                    │
│  ┌──────────────── Linux Security Modules ────────────────┐        │
│  │                                                        │        │
│  │   DAC Check    LSM Hook    MAC Decision                │        │
│  │   (Ch 10)      (Ch 8)      (Ch 9)                     │        │
│  │                   │                                    │        │
│  │        ┌──────────┼──────────┬──────────┐              │        │
│  │     SELinux   AppArmor    Smack    TOMOYO              │        │
│  │     (Ch 11)   (Ch 12)    (Ch 13)  (Ch 14)             │        │
│  └────────────────────────────────────────────────────────┘        │
│                                                                    │
│  Namespaces (Ch 16)    Cgroups (Ch 17)    Crypto API (Ch 21)      │
│                                                                    │
│  Memory Protection (Ch 18-19)    Secure Boot (Ch 20)               │
│                                                                    │
│  Network Security (Ch 23)    Key Management (Ch 22)                │
│                                                                    │
│  Audit Subsystem (Ch 26)    Kernel Hardening (Ch 25)               │
│                                                                    │
└────────────────────────── Hardware ────────────────────────────────┘
│  TPM    UEFI Secure Boot    NX Bit    IOMMU    TrustZone           │
└────────────────────────────────────────────────────────────────────┘
```
