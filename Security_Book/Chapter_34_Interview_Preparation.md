# Chapter 34: Interview Preparation

## Learning Goals
- Master commonly asked Linux security interview questions
- Practice structured answers with real-world examples
- Cover all major security topics in interview-ready format

---

## 34.1 Fundamentals Questions

**Q1: Explain the Linux security model at a high level.**
A: Linux security has multiple layers: (1) DAC — file permissions and POSIX ACLs based on UID/GID, checked by generic_permission(). (2) Capabilities — 41 fine-grained privileges replacing monolithic root. (3) LSM/MAC — mandatory access control via SELinux, AppArmor, etc., with ~200 hooks throughout the kernel. (4) Seccomp — system call filtering via BPF programs. (5) Namespaces — resource isolation (PID, mount, network, user). (6) Cgroups — resource limits preventing DoS. On every syscall: Seccomp runs first, then DAC checks permission bits, then ALL registered LSMs check their policies. If any layer denies, access is blocked. This defense-in-depth approach means no single vulnerability compromises the entire system.

**Q2: What is the difference between DAC and MAC?**
A: DAC (Discretionary Access Control) — the resource owner controls access. File permissions (rwx), POSIX ACLs. Problem: owner can relax security, root bypasses everything, no mandatory policy. MAC (Mandatory Access Control) — system-wide policy enforced by the kernel. Even root cannot override. SELinux uses type enforcement (domain→type rules), AppArmor uses path-based profiles, Smack uses label-based rules. DAC is checked first (generic_permission), then MAC (LSM hooks). Both must allow for access to succeed. MAC addresses DAC's weaknesses: prevents privilege escalation from root, enforces least-privilege, controls information flow.

**Q3: How does struct cred work in the kernel?**
A: struct cred holds all process credentials: real/effective/saved/filesystem UID/GID, supplementary groups, 5 capability sets (effective/permitted/inheritable/bounding/ambient), user namespace reference, LSM security blobs (SELinux context, AppArmor profile), and keyring pointers. It's RCU-protected and immutable once committed — to change credentials: prepare_creds() copies current, modify fields, commit_creds() atomically replaces the task's creds. Readers via current_cred() never see partial updates. The copy-on-write design prevents race conditions during credential changes.

---

## 34.2 Access Control Questions

**Q4: Explain Linux capabilities and why they matter.**
A: Traditional Unix has all-or-nothing root: UID 0 bypasses all checks. Capabilities split root's power into 41 specific privileges. CAP_NET_BIND_SERVICE (bind port <1024), CAP_SYS_ADMIN (mount, namespace, many kernel features), CAP_NET_RAW (raw sockets), etc. A process can have only the capabilities it needs — if a web server only needs CAP_NET_BIND_SERVICE, a compromise gives the attacker only that privilege, not full root. Five capability sets: effective (active now), permitted (can activate), inheritable (can pass to children), bounding (upper limit), ambient (automatically inherited). Check in kernel: capable() → ns_capable() → security_capable(). Docker drops 27 of 41 capabilities to restrict containers.

**Q5: How do file permissions work in the kernel?**
A: On file access, the kernel calls inode_permission() which calls: (1) acl_permission_check() — checks the 12-bit permission model: owner bits if UID matches, group bits if GID matches, other bits otherwise. Also checks POSIX ACLs if present (ACL entries checked in order: user, named users, group, named groups, mask, other). (2) generic_permission() — handles capabilities (CAP_DAC_OVERRIDE, CAP_DAC_READ_SEARCH). Root bypasses DAC unless filesystem is mounted nosuid. (3) security_inode_permission() — LSM hook dispatches to SELinux/AppArmor. Special bits: setuid (bit 11) changes effective UID on exec, setgid (bit 10) changes effective GID, sticky (bit 9) prevents deletion by non-owners in shared directories.

---

## 34.3 LSM and MAC Questions

**Q6: How does the LSM framework work internally?**
A: LSM provides ~200 hooks at security-critical points in the kernel (file operations, process management, networking, IPC). Each security module (SELinux, AppArmor, Smack, TOMOYO, Landlock, Yama) registers callback functions for the hooks it cares about. On a security-sensitive operation (e.g., file open), the kernel calls the hook function (security_file_open()), which iterates through all registered LSMs calling each one's callback. ALL LSMs must allow — any denial stops the operation (AND logic). LSMs store per-object security data in "blobs" attached to kernel objects (inodes, tasks, superblocks). Stacking: one major LSM (SELinux or AppArmor) + multiple minor LSMs (Yama, Lockdown, Landlock). Since Linux 5.x, major LSM stacking is also possible.

**Q7: Compare SELinux, AppArmor, and Smack.**
A: SELinux — label-based type enforcement. Every object has a security context (user:role:type:level). Complex policy language with ~30K LOC, finest granularity, MLS/MCS support, used in RHEL/Fedora/Android. AppArmor — path-based profiles. Rules reference filesystem paths (/var/www/**). Simpler to learn, but rename can bypass rules. Default on Ubuntu/SUSE, Docker uses it by default. Smack — simple label-based. Text labels on subjects/objects, rules as plain text triples. ~6K LOC, designed for embedded/IoT. Used in Tizen (Samsung) and AGL (automotive). TOMOYO — domain-based with execution history. Auto-learning mode generates policy from observed behavior. ~10K LOC, easiest to get started. Trade-offs: SELinux for maximum security, AppArmor for ease of use, Smack for embedded simplicity, TOMOYO for policy learning.

---

## 34.4 Isolation and Container Questions

**Q8: How do Linux namespaces provide security isolation?**
A: 8 namespace types partition kernel resources: Mount (filesystem visibility), PID (process visibility), Network (separate network stack, IPs, ports), UTS (hostname), IPC (shared memory, semaphores), User (UID/GID remapping), Cgroup (cgroup hierarchy view), Time (clock offsets). APIs: clone(CLONE_NEW*) creates child in new namespace, unshare() moves current process, setns() joins existing namespace. User namespace is most security-critical: maps container root (UID 0) to unprivileged host UID (100000+), so container escape lands as unprivileged user. Capabilities inside user NS only apply to that NS's resources. Containers combine ALL namespace types + seccomp + MAC + cgroups for defense in depth.

**Q9: Explain Seccomp-BPF and its role in container security.**
A: Seccomp-BPF installs a BPF (Berkeley Packet Filter) program that runs on every syscall entry. The filter examines seccomp_data: syscall number, architecture, and arguments (but cannot dereference pointers — no TOCTOU). Returns: ALLOW, KILL (SIGSYS), ERRNO, TRAP, TRACE, LOG, or USER_NOTIF (delegate to supervisor). Filters are additive-only (never removed), inherited by children, chain with AND logic (most restrictive wins). PR_SET_NO_NEW_PRIVS required before installation. Docker's default profile blocks ~44 dangerous syscalls: mount, kexec_load, init_module, ptrace, userfaultfd, etc. Seccomp runs BEFORE LSM hooks — it's the first defense layer. For containers, it reduces the kernel attack surface by blocking syscalls the workload doesn't need.

**Q10: Why should you never use --privileged containers in production?**
A: --privileged removes ALL container security: all 41 capabilities added, seccomp disabled, AppArmor/SELinux profiles removed, all host devices accessible (/dev/sda, /dev/mem), can mount host filesystem. It's equivalent to root on the host. Any escape gives full host access. Instead: add only specific capabilities needed (--cap-add NET_ADMIN), keep seccomp/AppArmor active, use --device for specific devices only. If an application "requires" --privileged, audit exactly which capabilities/syscalls it actually needs and grant only those.

---

## 34.5 Memory Protection Questions

**Q11: Explain ASLR, KASLR, and their limitations.**
A: ASLR randomizes user-space memory layout every run: stack (22 bits entropy), mmap/libraries (28 bits), heap (13 bits), executable (28 bits if PIE). Makes exploit address prediction unreliable. KASLR randomizes kernel base address at boot (only ~9 bits entropy — 512 positions on x86_64). KASLR limitations: (1) low entropy enables brute-force, (2) per-boot only — fixed for entire uptime, (3) side-channel attacks (Spectre, Meltdown) bypass it, (4) any kernel address leak defeats it. Defenses: kptr_restrict=2 (hide pointers), dmesg_restrict=1 (protect logs), KPTI (remove kernel from user page tables), %pK format specifier. ASLR alone is insufficient — needs NX, SMEP, SMAP, canaries for real protection.

**Q12: How do SMEP, SMAP, and KPTI protect the kernel?**
A: SMEP (CPU CR4 bit 20): prevents kernel from executing code in user-space pages. Blocks ret2usr attacks where attackers redirect kernel execution to user-controlled code. SMAP (CR4 bit 21): prevents kernel from reading/writing user-space pages except through copy_to_user()/copy_from_user() which use stac/clac to temporarily disable SMAP. Blocks data-only attacks from user memory. KPTI: maintains separate page tables for user and kernel mode — user page table doesn't map kernel memory (only tiny trampoline). Mitigates Meltdown (CVE-2017-5754) where speculative execution reads kernel memory via cache side-channels. ~5% overhead, reduced by PCID. Together: kernel can't execute user code (SMEP), can't freely access user data (SMAP), and user can't read kernel memory even speculatively (KPTI).

---

## 34.6 Cryptography and Boot Questions

**Q13: How does the kernel Crypto API work?**
A: Pluggable framework with algorithm types (cipher, hash, AEAD, RNG) and a template system for composing algorithms: cbc(aes), hmac(sha256), gcm(aes). Each implementation has a priority — hardware accelerators (AES-NI, ARM CE) register with higher priority than software fallbacks. When code requests crypto_alloc_skcipher("cbc(aes)", 0, 0), the API selects the highest-priority available implementation. Uses scatterlists for zero-copy I/O. Main consumers: dm-crypt (disk encryption), IPsec (VPN), fscrypt (file encryption), WireGuard, module signing, IMA. AF_ALG socket interface exposes kernel crypto to user space.

**Q14: Explain the Secure Boot chain of trust.**
A: Hardware root of trust (ROM/eFuses) contains the Platform Key. UEFI firmware verifies the bootloader against the signature database (db). Shim bootloader (Microsoft-signed) verifies GRUB against the distro key or MOK (Machine Owner Key). GRUB verifies the kernel image. The kernel verifies loadable modules (CONFIG_MODULE_SIG_FORCE). Kernel lockdown blocks root from bypassing this chain at runtime (no /dev/mem, no unsigned kexec, no unsigned modules). Optional: IMA/EVM extends verification to user-space binaries. Each stage cryptographically verifies the next — any break in the chain is detected. For embedded: ROM → SPL → U-Boot → kernel → dm-verity rootfs. For Android: AVB with vbmeta + dm-verity + rollback protection.

---

## 34.7 Debugging and Practical Questions

**Q15: How do you debug a SELinux denial?**
A: (1) Find denial: `ausearch -m AVC -ts recent`. (2) Read AVC: identify scontext (process type), tcontext (file type), tclass, denied permission. (3) Diagnose: `audit2why -a` explains root cause. (4) Fix based on cause: Wrong label → `restorecon -R /path`. Missing policy → `audit2allow -a -M fix && semodule -i fix.pp`. Boolean needed → `setsebool -P boolean_name 1`. (5) Verify: `sesearch --allow -s source_t -t target_t`. (6) For development: `semanage permissive -a domain_t` makes one type permissive. Never run setenforce 0 in production — it disables ALL SELinux enforcement.

**Q16: Design a secure embedded Linux system.**
A: Boot: Secure boot with hardware root of trust (eFuses for key hash), U-Boot verified boot (FIT images with RSA signatures), dm-verity on rootfs. Kernel: KASLR, STRICT_KERNEL_RWX, STACKPROTECTOR_STRONG, FORTIFY_SOURCE, MODULE_SIG_FORCE. MAC: Smack or minimal SELinux for process isolation. Filesystem: Read-only rootfs, separate writable data partition. Network: TLS everywhere, nftables with default-deny, certificate pinning. Keys: TrustZone/TEE for key storage, unique per-device keys, RPMB for rollback counters. Updates: Signed OTA with A/B partitioning, rollback protection. Debug: Disable JTAG/UART in production, audit logging. User-space: BusyBox minimal, drop privileges for services, no shell in production.

---

## 34.8 Scenario-Based Questions

**Q17: A container keeps getting SELinux denials. Walk through your troubleshooting.**
A: (1) Run `ausearch -m AVC -ts recent | grep container_name` to find relevant denials. (2) Check if the container is trying to access host resources it shouldn't — could indicate misconfiguration (wrong volume mount, host path). (3) If legitimate access: check if a boolean resolves it (`getsebool -a | grep container`), try `setsebool -P container_manage_cgroup 1` etc. (4) If custom policy needed: `audit2allow -a -M container_fix`, review generated rules for sanity (never blindly apply audit2allow output), install with `semodule -i container_fix.pp`. (5) If file labeling: `restorecon -R /path` or `chcon -R -t container_file_t /path`. (6) Verify with `sesearch` that the rule exists and makes sense.

**Q18: An IoT device was compromised. How would you harden it?**
A: Immediate: (1) Analyze the compromise vector — firmware dump, network capture, audit logs. (2) Patch the vulnerability, rotate all keys/credentials. Long-term hardening: (3) Enable secure boot — eFuses + signed bootloader + dm-verity. (4) Implement Smack/SELinux for process isolation. (5) Disable debug interfaces (JTAG, UART, ADB). (6) Read-only rootfs, separate writable data partition. (7) Network: TLS with certificate pinning, firewall default-deny, disable unused services. (8) Signed OTA with A/B partitioning and rollback protection. (9) Unique per-device keys stored in TEE/secure element. (10) SBOM tracking for vulnerability monitoring. (11) Minimize installed packages — remove shells, compilers, package managers from production image.

**Q19: How would you secure a Kubernetes deployment?**
A: (1) Pod Security Standards: enforce "restricted" profile — runAsNonRoot, readOnlyRootFilesystem, drop ALL capabilities, seccompProfile: RuntimeDefault. (2) Network Policies: default deny all, allow only needed pod-to-pod traffic. (3) RBAC: least privilege for service accounts, no cluster-admin for workloads. (4) Image security: scan images for CVEs, use minimal base images (distroless), enforce image signing. (5) Runtime: never use --privileged, use rootless container runtime. (6) Secrets: use external secrets manager (Vault), not Kubernetes Secrets (base64 encoded, not encrypted by default). (7) Node security: hardened host OS, kernel updates, CIS benchmark compliance. (8) Monitoring: Falco for runtime threat detection, audit logging, network traffic monitoring.

---

## 34.9 Quick Reference: One-Liner Answers

```
┌──────────────────────────────────────────────────────────────────┐
│ Question                           │ Key Answer                  │
├────────────────────────────────────┼────────────────────────────┤
│ DAC vs MAC?                        │ Owner controls vs system    │
│                                    │ policy controls             │
│ What checks first: Seccomp or LSM? │ Seccomp (before syscall)    │
│ How many capabilities?             │ 41 (as of Linux 6.x)       │
│ SELinux label format?              │ user:role:type:level        │
│ AppArmor approach?                 │ Path-based profiles         │
│ Smack designed for?                │ Embedded/IoT simplicity     │
│ TOMOYO unique feature?             │ Auto-learning mode          │
│ KASLR entropy on x86_64?           │ ~9 bits (512 positions)     │
│ SMEP prevents?                     │ Kernel exec of user code    │
│ SMAP prevents?                     │ Kernel access to user data  │
│ KPTI mitigates?                    │ Meltdown (CVE-2017-5754)    │
│ Seccomp can deref pointers?        │ No (TOCTOU risk)            │
│ Docker default capabilities?       │ 14 of 41 kept              │
│ --privileged danger?               │ Removes ALL security        │
│ User namespace benefit?            │ Root inside = unpriv outside│
│ dm-verity provides?                │ Block-level integrity check │
│ Trusted keys sealed by?            │ TPM hardware                │
│ WireGuard lines of code?           │ ~4000 LOC                   │
│ KASAN detects?                     │ OOB, UAF, double-free       │
│ syzkaller is?                      │ Coverage-guided kernel fuzzer│
│ CIS Benchmark is?                  │ Hardening checklist/standard│
└──────────────────────────────────────────────────────────────────┘
```

---

## Summary

This chapter covers 19 interview questions across all security domains:
- Fundamentals: security model, DAC vs MAC, struct cred
- Access control: capabilities, file permissions
- LSM/MAC: framework internals, SELinux vs AppArmor vs Smack
- Containers: namespaces, seccomp, --privileged dangers
- Memory: ASLR/KASLR, SMEP/SMAP/KPTI
- Crypto/Boot: Crypto API, Secure Boot chain
- Debugging: SELinux denial troubleshooting
- Scenarios: embedded hardening, container troubleshooting, Kubernetes security
- Quick reference table for one-liner answers

---

**End of Security Book**

[Return to Master Index](00_Master_Index.md)
