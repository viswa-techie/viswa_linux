# Chapter 30: Definitions of Important Terms (Glossary)

## Learning Goals
- Quick reference for all security terminology
- Precise definitions for interview and daily use

---

## 30.1 Access Control Terms

| Term | Definition |
|------|-----------|
| **DAC** | Discretionary Access Control — resource owner decides access. Linux file permissions (rwx). Owner can relax security. |
| **MAC** | Mandatory Access Control — system-wide policy decides access. SELinux, AppArmor. Owner cannot override policy. |
| **RBAC** | Role-Based Access Control — access determined by assigned roles. SELinux roles constrain which types users can enter. |
| **ACL** | Access Control List — fine-grained permissions beyond owner/group/other. POSIX ACLs use setfacl/getfacl. |
| **ABAC** | Attribute-Based Access Control — policies based on subject/object/environment attributes. |
| **Type Enforcement** | SELinux primary mechanism. Subjects have domain types, objects have types. Rules: `allow domain type:class {perms}`. |
| **Security Context** | SELinux label: `user:role:type:level`. Every process and file has one. |
| **AVC** | Access Vector Cache — SELinux cache for recent access decisions. Avoids re-computing policy for repeated checks. |
| **Profile** | AppArmor policy for a specific program. Defines allowed file paths, capabilities, network access. |
| **Label** | Smack text label assigned to subjects/objects. Access rules define label→label permissions. |
| **Domain** | TOMOYO execution history path. Same binary has different domains based on exec chain from boot. |

---

## 30.2 Isolation and Sandboxing Terms

| Term | Definition |
|------|-----------|
| **Namespace** | Kernel resource partition — processes see isolated view. 8 types: mount, PID, net, UTS, IPC, user, cgroup, time. |
| **Cgroup** | Control Group — organizes processes for resource limits. Controllers: CPU, memory, PIDs, I/O. Prevents DoS. |
| **Seccomp** | Secure Computing Mode — BPF filter restricts which syscalls a process can make. Returns ALLOW/KILL/ERRNO. |
| **Capability** | Fine-grained kernel privilege. 41 capabilities replace monolithic root. E.g., CAP_NET_BIND_SERVICE for port <1024. |
| **LSM** | Linux Security Modules — framework for pluggable MAC. ~200 hooks. SELinux, AppArmor, Smack, TOMOYO, Landlock, Yama. |
| **Landlock** | Unprivileged sandboxing LSM. Applications restrict their own filesystem access without root. |
| **Container** | Process isolation using namespaces + cgroups + seccomp + MAC + capabilities. Shares host kernel. |
| **User Namespace** | UID/GID remapping. Root inside = unprivileged outside. Enables rootless containers. |
| **veth** | Virtual Ethernet pair — two connected virtual NICs for container networking across network namespaces. |
| **OverlayFS** | Union filesystem for containers. Read-only base layer + writable upper layer. |

---

## 30.3 Memory and Hardware Protection Terms

| Term | Definition |
|------|-----------|
| **ASLR** | Address Space Layout Randomization — random placement of stack, heap, libraries, code in memory. |
| **KASLR** | Kernel ASLR — randomize kernel base address at boot. ~9 bits entropy on x86_64. |
| **PIE** | Position Independent Executable — compiled for ASLR of main executable code. `gcc -pie -fPIE`. |
| **NX** | No-eXecute — page table bit preventing code execution from data pages. W^X: write XOR execute. |
| **SMEP** | Supervisor Mode Execution Prevention — CPU blocks kernel from executing user-space code. |
| **SMAP** | Supervisor Mode Access Prevention — CPU blocks kernel from reading/writing user-space memory. copy_*_user() exempted. |
| **KPTI** | Kernel Page Table Isolation — separate user/kernel page tables. Mitigates Meltdown. ~5% overhead. |
| **Stack Canary** | Random value between local variables and return address. Buffer overflow detected before return. |
| **Shadow Call Stack** | Separate stack for return addresses only. Main stack overflow cannot corrupt returns. ARM64. |
| **CFI** | Control Flow Integrity — compiler verifies indirect call targets match expected types. Blocks ROP/JOP. |
| **FORTIFY_SOURCE** | Compile/runtime detection of buffer overflows in string/memory functions (memcpy, strcpy). |
| **STACKLEAK** | GCC plugin that erases kernel stack at end of every syscall. Prevents info leaks. |

---

## 30.4 Cryptography and Key Terms

| Term | Definition |
|------|-----------|
| **Crypto API** | Kernel cryptographic framework — pluggable algorithms with priority-based selection. Hardware auto-offload. |
| **AES-NI** | Intel CPU instruction set for hardware AES acceleration. Used by dm-crypt, IPsec, fscrypt. |
| **dm-crypt** | Device-mapper target for block device encryption. LUKS standard. AES-XTS typical cipher. |
| **fscrypt** | Per-file filesystem encryption. Used by Android (ext4/f2fs). Per-file keys derived from master key. |
| **LUKS** | Linux Unified Key Setup — standard for disk encryption key management. Multiple key slots. PBKDF2/Argon2. |
| **Keyring** | Kernel key management facility. Types: user, logon, trusted, encrypted, asymmetric. |
| **Trusted Key** | Key sealed by TPM — raw material never leaves TPM + kernel memory. |
| **TPM** | Trusted Platform Module — hardware security chip. Key storage, sealing, attestation, RNG. |
| **PCR** | Platform Configuration Register in TPM. Extend-only hash chain for boot measurement. |
| **CSPRNG** | Cryptographically Secure Pseudo-Random Number Generator. Kernel uses ChaCha20-based CSPRNG. |
| **AEAD** | Authenticated Encryption with Associated Data — provides confidentiality + integrity. AES-GCM, ChaCha20-Poly1305. |

---

## 30.5 Boot and Integrity Terms

| Term | Definition |
|------|-----------|
| **Secure Boot** | UEFI feature — verify each boot stage's signature. PK → KEK → db/dbx chain. |
| **Lockdown** | Kernel LSM that restricts root when Secure Boot is active. Blocks /dev/mem, unsigned modules, kexec. |
| **IMA** | Integrity Measurement Architecture — hash files on access, extend TPM PCRs, optionally enforce signatures. |
| **EVM** | Extended Verification Module — HMAC protects extended attributes (IMA, SELinux labels, capabilities). |
| **dm-verity** | Device-mapper target for block-level integrity verification. Hash tree verified on every read. Used by Android AVB. |
| **AVB** | Android Verified Boot — vbmeta + dm-verity for complete boot and runtime verification. |
| **MOK** | Machine Owner Key — user-managed key for Secure Boot. Enrolled via MokManager. Signs custom kernels/modules. |
| **Shim** | Microsoft-signed bootloader that chains to distro bootloaders. Bridges UEFI Secure Boot and Linux. |
| **Attestation** | Proving system configuration to remote party using TPM measurements. |
| **Measured Boot** | Record (measure) each boot component into TPM PCRs. Does NOT enforce — only records for later verification. |

---

## 30.6 Network Security Terms

| Term | Definition |
|------|-----------|
| **Netfilter** | Kernel packet filtering framework. 5 hooks: PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING. |
| **nftables** | Modern replacement for iptables. Unified framework for packet filtering, NAT, mangling. |
| **conntrack** | Connection tracking — stateful firewall. Tracks NEW/ESTABLISHED/RELATED connection states. |
| **IPsec** | Kernel-level IP encryption. ESP (encrypt+auth), AH (auth only). XFRM framework. SA/SP databases. |
| **kTLS** | Kernel TLS — offload TLS record layer to kernel. Enables sendfile() with TLS. Hardware offload possible. |
| **WireGuard** | Modern in-kernel VPN. ~4000 LOC. ChaCha20-Poly1305, Curve25519. Noise protocol. Mainline since 5.6. |
| **XDP** | eXpress Data Path — eBPF programs at NIC driver level. Fastest packet filtering. |
| **SYN cookies** | DDoS mitigation — kernel doesn't allocate TCB until handshake completes. |
| **CIPSO** | Common IP Security Option — carries Smack labels in IP option headers. |

---

## 30.7 Attack and Vulnerability Terms

| Term | Definition |
|------|-----------|
| **CVE** | Common Vulnerabilities and Exposures — unique identifier for security vulnerabilities. |
| **ROP** | Return-Oriented Programming — chain existing code gadgets to build exploits. Bypasses NX. |
| **ret2usr** | Return to userspace — redirect kernel execution to attacker code in user memory. Blocked by SMEP. |
| **Meltdown** | CPU vulnerability — speculative execution reads kernel memory from user space. Mitigated by KPTI. |
| **Spectre** | CPU vulnerability — speculative execution leaks data via branch prediction manipulation. |
| **TOCTOU** | Time-of-Check-Time-of-Use — race condition between security check and resource use. |
| **Use-After-Free** | Access memory after it's been freed. #1 kernel vulnerability class. Detected by KASAN. |
| **Fork bomb** | :(){ :\|:& };: — infinite process creation. Mitigated by cgroup PID controller. |
| **Container escape** | Break out of container to access host. Vectors: kernel exploits, --privileged, Docker socket. |
| **Privilege escalation** | Gain higher privileges. Local (user→root) or vertical (unprivileged→admin). |

---

## 30.8 Auditing and Testing Terms

| Term | Definition |
|------|-----------|
| **Audit** | Kernel subsystem for security event logging. auditd daemon, auditctl rules, ausearch queries. |
| **KASAN** | Kernel Address Sanitizer — detects out-of-bounds, use-after-free, double-free. 2-3x memory overhead. |
| **KMSAN** | Kernel Memory Sanitizer — detects uninitialized memory reads (info leak bugs). |
| **KCSAN** | Kernel Concurrency Sanitizer — detects data races. |
| **KFENCE** | Kernel Electric Fence — low-overhead (~1%) production UAF/OOB detector. Samples allocations. |
| **syzkaller** | Coverage-guided kernel fuzzer. Generates random syscall sequences. Found 1000+ kernel bugs. |
| **sparse** | Static analysis tool for kernel code. Checks locking, address spaces, endianness. |

---

## Summary

This glossary covers 100+ security terms across 8 categories:
- Access control (DAC, MAC, RBAC, ACL, Type Enforcement)
- Isolation (namespaces, cgroups, seccomp, capabilities, containers)
- Memory protection (ASLR, KASLR, NX, SMEP, SMAP, KPTI, CFI)
- Cryptography (Crypto API, dm-crypt, fscrypt, keyring, TPM)
- Boot integrity (Secure Boot, IMA, EVM, dm-verity, AVB)
- Network security (Netfilter, IPsec, kTLS, WireGuard, XDP)
- Attacks (ROP, ret2usr, Meltdown, Spectre, TOCTOU, UAF)
- Testing (KASAN, KMSAN, KCSAN, KFENCE, syzkaller, audit)

---

Next: [Chapter 31 — Security Models in Other Operating Systems](Chapter_31_Other_OS_Security.md)
