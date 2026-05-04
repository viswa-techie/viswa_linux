# Chapter 27: Kernel Source Code Map for Security

## Learning Goals
- Know the directory structure of security-related kernel code
- Understand which files implement key security mechanisms
- Have a reference map for navigating security source code

---

## 27.1 Security Subsystem Directory Map

```
linux/
├── security/                          ← Main security directory
│   ├── security.c                     LSM infrastructure, hook dispatch
│   ├── commoncap.c                    Common capability handling
│   ├── min_addr.c                     Minimum mmap address
│   │
│   ├── selinux/                       ← SELinux
│   │   ├── hooks.c                    LSM hook implementations (~200)
│   │   ├── avc.c                      Access Vector Cache
│   │   ├── ss/                        Security Server (policy engine)
│   │   │   ├── avtab.c                Access vector table
│   │   │   ├── policydb.c             Policy database
│   │   │   ├── services.c             Core policy services
│   │   │   └── ebitmap.c              Extensible bitmap
│   │   ├── selinuxfs.c                /sys/fs/selinux interface
│   │   └── netlabel.c                 Network label integration
│   │
│   ├── apparmor/                      ← AppArmor
│   │   ├── lsm.c                      LSM hook implementations
│   │   ├── policy.c                   Profile management
│   │   ├── domain.c                   Domain transitions
│   │   ├── file.c                     File access mediation
│   │   ├── match.c                    DFA path matching
│   │   └── apparmorfs.c              securityfs interface
│   │
│   ├── smack/                         ← Smack
│   │   ├── smack_lsm.c               LSM hook implementations
│   │   ├── smack_access.c            Access decisions
│   │   ├── smackfs.c                  Virtual filesystem
│   │   └── smack_netfilter.c         Network labels
│   │
│   ├── tomoyo/                        ← TOMOYO
│   │   ├── tomoyo.c                   LSM hook implementations
│   │   ├── domain.c                   Domain management
│   │   ├── file.c                     File access mediation
│   │   └── common.c                   Shared utilities
│   │
│   ├── landlock/                      ← Landlock (user-space sandboxing)
│   │   ├── syscalls.c                 Landlock syscall handlers
│   │   ├── fs.c                       Filesystem restrictions
│   │   └── ruleset.c                  Ruleset management
│   │
│   ├── yama/                          ← Yama (ptrace restrictions)
│   │   └── yama_lsm.c                Ptrace scope enforcement
│   │
│   ├── lockdown/                      ← Kernel lockdown
│   │   └── lockdown.c                 Lockdown enforcement
│   │
│   ├── integrity/                     ← IMA/EVM
│   │   ├── ima/                       Integrity Measurement
│   │   │   ├── ima_main.c             IMA hook implementations
│   │   │   ├── ima_policy.c           IMA policy engine
│   │   │   ├── ima_crypto.c           Hash computation
│   │   │   └── ima_appraise.c         File appraisal
│   │   └── evm/                       Extended Verification
│   │       ├── evm_main.c             EVM core
│   │       └── evm_crypto.c           EVM HMAC/signature
│   │
│   ├── keys/                          ← Keyring subsystem
│   │   ├── key.c                      Key lifecycle
│   │   ├── keyring.c                  Keyring management
│   │   ├── keyctl.c                   keyctl syscall
│   │   ├── process_keys.c            Process/session keyrings
│   │   ├── trusted-keys/             TPM-sealed keys
│   │   └── encrypted-keys/           Encrypted key type
│   │
│   └── safesetid/                     ← UID/GID transition restrictions
│       └── lsm.c                      Safe setid enforcement
```

---

## 27.2 Kernel Core Security Files

```
kernel/
├── seccomp.c                          Seccomp core (BPF filter)
├── audit.c                            Audit framework
├── auditsc.c                          Audit syscall logging
├── auditfilter.c                      Audit rule filtering
├── capability.c                       Capability system
├── cred.c                             Credential management
├── user_namespace.c                   User namespace (UID mapping)
├── pid_namespace.c                    PID namespace
├── nsproxy.c                          Namespace proxy
├── cgroup/                            Cgroup subsystem
│   ├── cgroup.c                       Cgroup v2 core
│   └── pids.c                         PID controller
├── stackleak.c                        STACKLEAK runtime
├── cfi.c                              Control Flow Integrity
└── module/
    └── signing.c                       Module signature verification

fs/
├── namespace.c                        Mount namespace
├── exec.c                             execve security checks
├── namei.c                            Path lookup + permission checks
│                                       (generic_permission, acl_permission_check)
├── crypto/                            fscrypt (file encryption)
│   ├── keysetup.c                     Per-file key derivation
│   └── policy.c                       Encryption policy
└── proc/
    └── base.c                          /proc/<pid> security checks

mm/
├── mmap.c                             mmap security (ASLR)
├── kasan/                             Kernel Address Sanitizer
├── kmsan/                             Kernel Memory Sanitizer
└── kfence/                            Kernel Electric Fence

net/
├── core/
│   ├── filter.c                       BPF/eBPF socket filtering
│   └── net_namespace.c                Network namespace
├── netfilter/                         Netfilter/nftables
├── xfrm/                             IPsec (XFRM framework)
├── tls/                               Kernel TLS
└── ipv4/
    └── syncookies.c                   SYN cookie DDoS mitigation

crypto/
├── api.c                              Crypto API core
├── skcipher.c                         Symmetric cipher framework
├── hash.c                             Hash framework
├── aead.c                             AEAD framework
├── af_alg.c                           AF_ALG user-space interface
└── drbg.c                             Deterministic RNG

drivers/
├── char/
│   ├── random.c                       /dev/random CSPRNG
│   └── tpm/                           TPM driver subsystem
└── net/
    └── wireguard/                     WireGuard VPN

arch/x86/
├── boot/compressed/kaslr.c            KASLR offset selection
├── mm/
│   ├── kaslr.c                        Kernel address randomization
│   └── pti.c                          KPTI (page table isolation)
├── kernel/cpu/bugs.c                  Spectre/Meltdown mitigations
└── include/asm/
    ├── pgtable.h                      Page table flags (NX)
    └── smap.h                         SMAP stac/clac
```

---

## 27.3 Key Data Structures

```
  ┌──────────────────────┬─────────────────────────────────────┐
  │ Structure            │ File / Purpose                       │
  ├──────────────────────┼─────────────────────────────────────┤
  │ struct cred          │ include/linux/cred.h                 │
  │                      │ Process credentials (UID, caps, etc.)│
  │                      │                                      │
  │ struct security_hook_list │ include/linux/lsm_hooks.h      │
  │                      │ LSM hook registration                │
  │                      │                                      │
  │ struct kern_ipc_perm │ include/linux/ipc.h                  │
  │                      │ IPC security context                 │
  │                      │                                      │
  │ struct seccomp_filter│ include/linux/seccomp.h              │
  │                      │ Seccomp BPF filter chain             │
  │                      │                                      │
  │ struct key           │ include/linux/key.h                  │
  │                      │ Keyring key structure                │
  │                      │                                      │
  │ struct nsproxy       │ include/linux/nsproxy.h              │
  │                      │ Process namespace membership         │
  │                      │                                      │
  │ struct user_namespace│ include/linux/user_namespace.h       │
  │                      │ User namespace UID/GID mapping       │
  │                      │                                      │
  │ struct selinux_state │ security/selinux/include/security.h  │
  │                      │ SELinux runtime state                │
  │                      │                                      │
  │ struct aa_profile    │ security/apparmor/include/policy.h   │
  │                      │ AppArmor profile                     │
  │                      │                                      │
  │ struct ima_iint_cache│ security/integrity/integrity.h       │
  │                      │ IMA per-inode integrity info         │
  └──────────────────────┴─────────────────────────────────────┘
```

---

## Summary

- security/ directory: LSM framework + SELinux, AppArmor, Smack, TOMOYO, Landlock, Yama
- security/keys/: kernel keyring subsystem
- security/integrity/: IMA/EVM file integrity
- kernel/: seccomp, audit, capabilities, credentials, namespaces
- fs/: permission checks (namei.c), fscrypt, mount namespace
- mm/: KASAN, KMSAN, KFENCE sanitizers; mmap ASLR
- crypto/: Crypto API framework + algorithm implementations
- arch/x86/: KASLR, KPTI, SMAP, Spectre/Meltdown mitigations
- drivers/char/random.c: kernel CSPRNG
- Key structs: cred, security_hook_list, seccomp_filter, key, nsproxy

---

Next: [Chapter 28 — Security Flow Diagrams](Chapter_28_Flow_Diagrams.md)
