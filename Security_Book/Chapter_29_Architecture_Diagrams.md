# Chapter 29: Important Security Architecture Diagrams

## Learning Goals
- Visualize the complete Linux security architecture
- Understand relationship between security components
- Have reference diagrams for interview discussions

---

## 29.1 Complete Linux Security Stack

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                        USER SPACE                                │
  │                                                                  │
  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐│
  │  │ Application│  │ Container  │  │ System     │  │ Security   ││
  │  │ Processes  │  │ Runtime    │  │ Services   │  │ Daemons    ││
  │  │            │  │ (Docker/   │  │ (systemd/  │  │ (auditd/   ││
  │  │            │  │  Podman)   │  │  sshd)     │  │  IMA/EVM)  ││
  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘│
  │        │               │               │               │        │
  ├────────┴───────────────┴───────────────┴───────────────┴────────┤
  │                     SYSTEM CALL INTERFACE                        │
  ├─────────────────────────────────────────────────────────────────┤
  │                                                                  │
  │  ┌─── SECCOMP ──────────────────────────────────────────────┐   │
  │  │  BPF filter: allow/deny syscalls before execution        │   │
  │  └─────────────────────────────┬────────────────────────────┘   │
  │                                 │                                │
  │  ┌─── DAC ─────────────────────┼───────────────────────────┐    │
  │  │  UID/GID permission bits    │  POSIX ACLs              │    │
  │  │  generic_permission()       │  acl_permission_check()  │    │
  │  └─────────────────────────────┼──────────────────────────┘    │
  │                                 │                                │
  │  ┌─── LSM FRAMEWORK ──────────┼──────────────────────────┐     │
  │  │                             │                           │     │
  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │     │
  │  │  │ SELinux  │ │ AppArmor │ │ Smack    │ │ TOMOYO   │  │     │
  │  │  │ (Type    │ │ (Path    │ │ (Label   │ │ (Domain  │  │     │
  │  │  │  Enforce)│ │  Based)  │ │  Based)  │ │  Based)  │  │     │
  │  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │     │
  │  │                                                         │     │
  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐               │     │
  │  │  │ Yama     │ │ Lockdown │ │ Landlock │               │     │
  │  │  │ (Ptrace) │ │ (Boot    │ │ (User    │               │     │
  │  │  │          │ │  Integ.) │ │  sandbox)│               │     │
  │  │  └──────────┘ └──────────┘ └──────────┘               │     │
  │  └─────────────────────────────────────────────────────────┘     │
  │                                                                  │
  │  ┌─── CAPABILITIES ────────────────────────────────────────┐    │
  │  │  41 fine-grained privileges replacing root              │    │
  │  │  capable() / ns_capable() / security_capable()          │    │
  │  └─────────────────────────────────────────────────────────┘    │
  │                                                                  │
  │  ┌─── NAMESPACES ─────────────────────────────────────────┐     │
  │  │  Mount │ PID │ Net │ UTS │ IPC │ User │ Cgroup │ Time │     │
  │  └─────────────────────────────────────────────────────────┘     │
  │                                                                  │
  │  ┌─── CGROUPS ─────────────────────────────────────────────┐    │
  │  │  CPU │ Memory │ PIDs │ I/O │ Devices │ Resource limits  │    │
  │  └─────────────────────────────────────────────────────────┘    │
  │                                                                  │
  │  ┌─── CRYPTO ──────────────────────────────────────────────┐    │
  │  │  Crypto API │ Keyring │ dm-crypt │ fscrypt │ Random     │    │
  │  └─────────────────────────────────────────────────────────┘    │
  │                                                                  │
  │  ┌─── INTEGRITY ──────────────────────────────────────────┐     │
  │  │  IMA (measurement) │ EVM (xattr protection) │ dm-verity│     │
  │  └─────────────────────────────────────────────────────────┘     │
  │                                                                  │
  │  ┌─── AUDIT ───────────────────────────────────────────────┐    │
  │  │  Syscall logging │ AVC denials │ Auth events │ File ops │    │
  │  └─────────────────────────────────────────────────────────┘    │
  │                                                                  │
  ├──────────────────────────────────────────────────────────────────┤
  │                        HARDWARE                                  │
  │  NX bit │ SMEP │ SMAP │ AES-NI │ TPM │ IOMMU │ TEE │ RDRAND   │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 29.2 Credential Structure (struct cred)

```
  ┌────────────────────────────────────────────────────────────┐
  │  struct cred (include/linux/cred.h)                        │
  │                                                            │
  │  Identity:                                                 │
  │  ┌────────────────┬────────────────────────────────────┐   │
  │  │ uid / gid      │ Real UID/GID (who started process) │   │
  │  │ euid / egid    │ Effective (used for permission)    │   │
  │  │ suid / sgid    │ Saved (for setuid restore)         │   │
  │  │ fsuid / fsgid  │ Filesystem (for file access)       │   │
  │  └────────────────┴────────────────────────────────────┘   │
  │                                                            │
  │  Groups:                                                   │
  │  ┌────────────────────────────────────────────────────┐    │
  │  │ group_info      → supplementary group list          │    │
  │  └────────────────────────────────────────────────────┘    │
  │                                                            │
  │  Capabilities:                                             │
  │  ┌────────────────────────────────────────────────────┐    │
  │  │ cap_effective   │ Currently active capabilities     │    │
  │  │ cap_inheritable │ Can be inherited across exec      │    │
  │  │ cap_permitted   │ Maximum allowed set               │    │
  │  │ cap_bset        │ Bounding set (upper limit)        │    │
  │  │ cap_ambient     │ Unprivileged inheritance          │    │
  │  └────────────────────────────────────────────────────┘    │
  │                                                            │
  │  Namespaces:                                               │
  │  ┌────────────────────────────────────────────────────┐    │
  │  │ user_ns         → user namespace membership         │    │
  │  └────────────────────────────────────────────────────┘    │
  │                                                            │
  │  LSM Security Blobs:                                       │
  │  ┌────────────────────────────────────────────────────┐    │
  │  │ security        → LSM-specific data                 │    │
  │  │                   SELinux: security context (SID)    │    │
  │  │                   AppArmor: profile pointer          │    │
  │  │                   Smack: label pointer               │    │
  │  └────────────────────────────────────────────────────┘    │
  │                                                            │
  │  Keyrings:                                                 │
  │  ┌────────────────────────────────────────────────────┐    │
  │  │ session_keyring → session keyring pointer           │    │
  │  │ process_keyring → process keyring pointer           │    │
  │  │ thread_keyring  → thread keyring pointer            │    │
  │  └────────────────────────────────────────────────────┘    │
  └────────────────────────────────────────────────────────────┘

  Lifecycle:
    prepare_creds() → copy current creds
    modify fields
    commit_creds()  → atomically replace task's creds
    (RCU-protected — readers never see partial updates)
```

---

## 29.3 LSM Hook Architecture

```
  ┌───────────────────────────────────────────────────────────┐
  │  VFS / Networking / IPC / Process Management               │
  │                                                            │
  │  Example: vfs_open() calls security_file_open()            │
  │           tcp_connect() calls security_socket_connect()    │
  └───────────────────┬───────────────────────────────────────┘
                       │
                       ▼
  ┌───────────────────────────────────────────────────────────┐
  │  security/security.c: LSM Multiplexer                      │
  │                                                            │
  │  security_file_open(file):                                 │
  │    for each registered LSM:                                │
  │      ret = lsm->file_open(file)                            │
  │      if (ret != 0) return ret    ← first deny wins         │
  │    return 0                      ← all allowed             │
  │                                                            │
  │  Hook categories (~200 hooks total):                       │
  │                                                            │
  │  File hooks:     file_open, file_permission, file_mmap     │
  │  Inode hooks:    inode_permission, inode_create, inode_link│
  │  Process hooks:  task_kill, task_setnice, bprm_check       │
  │  Network hooks:  socket_connect, socket_bind, socket_listen│
  │  IPC hooks:      msg_queue_msgsnd, shm_shmat              │
  │  Key hooks:      key_alloc, key_permission                 │
  │  Module hooks:   kernel_module_request, kernel_load_data   │
  │  Mount hooks:    sb_mount, sb_umount, move_mount           │
  └───────────────────────────────────────────────────────────┘
                       │
          ┌────────────┼────────────┬────────────┐
          ▼            ▼            ▼            ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ SELinux  │ │ AppArmor │ │ Smack    │ │ Yama     │
  │ hooks.c  │ │ lsm.c    │ │ smack_   │ │ yama_    │
  │          │ │          │ │ lsm.c    │ │ lsm.c    │
  │ AVC →    │ │ Profile  │ │ Label    │ │ Ptrace   │
  │ Policy   │ │ → DFA    │ │ → Rules  │ │ scope    │
  │ engine   │ │ match    │ │          │ │ check    │
  └──────────┘ └──────────┘ └──────────┘ └──────────┘
```

---

## 29.4 Container Security Architecture

```
  ┌──────────────────────────────────────────────────────────────┐
  │  HOST                                                        │
  │                                                              │
  │  ┌─────────── Container 1 ──────────────────────────────┐   │
  │  │                                                       │   │
  │  │  Mount NS: OverlayFS root → /var/lib/containers/...  │   │
  │  │  PID NS: PID 1 = entrypoint                          │   │
  │  │  Net NS: eth0 (172.17.0.2) via veth pair             │   │
  │  │  User NS: root(0) → host uid 100000                  │   │
  │  │  UTS NS: hostname = container_1                      │   │
  │  │  IPC NS: isolated shared memory                      │   │
  │  │                                                       │   │
  │  │  Seccomp: ~44 syscalls blocked                        │   │
  │  │  Capabilities: 14/41 (SYS_ADMIN dropped)             │   │
  │  │  AppArmor: docker-default profile                     │   │
  │  │  Cgroup: mem=512M, cpu=2, pids=100                   │   │
  │  │  NO_NEW_PRIVS: set                                    │   │
  │  │  Read-only rootfs: OverlayFS lower layer              │   │
  │  │                                                       │   │
  │  │  ┌──────────────────────────────────────────┐        │   │
  │  │  │  Application                              │        │   │
  │  │  │  Sees: own PID 1, own root, own network   │        │   │
  │  │  │  Cannot: see host PIDs, access host files  │        │   │
  │  │  │  Cannot: load modules, mount filesystems   │        │   │
  │  │  │  Cannot: use kexec, reboot, ptrace         │        │   │
  │  │  └──────────────────────────────────────────┘        │   │
  │  └───────────────────────────────────────────────────────┘   │
  │                                                              │
  │  ┌─────────── Container 2 ─── (similar isolation) ──────┐   │
  │  │  ...                                                  │   │
  │  └───────────────────────────────────────────────────────┘   │
  │                                                              │
  │  Shared kernel ← Single point of failure for ALL containers  │
  └──────────────────────────────────────────────────────────────┘
```

---

## 29.5 Secure Boot Chain Diagram

```
  ┌─────────────────────┐
  │ Hardware Root of     │
  │ Trust (ROM/fuses)    │
  │ PK (Platform Key)   │
  └─────────┬───────────┘
            │ verify
            ▼
  ┌─────────────────────┐     ┌──────────────────┐
  │ UEFI Firmware       │────►│ db: Allowed keys  │
  │ (KEK enrolled)      │     │ dbx: Revoked keys │
  │                     │     └──────────────────┘
  └─────────┬───────────┘
            │ verify against db
            ▼
  ┌─────────────────────┐     ┌──────────────────┐
  │ Shim bootloader     │────►│ MOK: User keys    │
  │ (Microsoft signed)  │     │ Shim built-in key │
  └─────────┬───────────┘     └──────────────────┘
            │ verify against MOK/built-in
            ▼
  ┌─────────────────────┐
  │ GRUB2 bootloader    │
  │ (distro signed)     │
  └─────────┬───────────┘
            │ verify
            ▼
  ┌─────────────────────┐     ┌──────────────────────┐
  │ Linux Kernel        │────►│ Lockdown:             │
  │ (distro signed)     │     │   Block /dev/mem      │
  │                     │     │   Block kexec unsigned │
  │                     │     │   Block unsigned mods  │
  └─────────┬───────────┘     └──────────────────────┘
            │ verify MODULE_SIG
            ▼
  ┌─────────────────────┐     ┌──────────────────────┐
  │ Kernel Modules      │     │ Keyrings:             │
  │ (.ko files)         │────►│   .builtin_trusted    │
  │ (build-key signed)  │     │   .platform (UEFI db) │
  └─────────┬───────────┘     │   .machine (MOK)      │
            │                  └──────────────────────┘
            │ IMA appraise (optional)
            ▼
  ┌─────────────────────┐
  │ User Space Binaries │
  │ (IMA verified)      │
  └─────────────────────┘
```

---

## 29.6 Attack Surface Diagram

```
  ┌──────────────────────────────────────────────────────────────┐
  │  KERNEL ATTACK SURFACE                                       │
  │                                                              │
  │  From User Space (unprivileged attacker):                    │
  │  ┌─────────────────────────────────────────────────────┐     │
  │  │ Syscall interface (~300 syscalls)            HIGH   │     │
  │  │ /proc filesystem                            MEDIUM │     │
  │  │ /sys filesystem                             MEDIUM │     │
  │  │ /dev device files                           MEDIUM │     │
  │  │ Netlink sockets                             MEDIUM │     │
  │  │ io_uring                                    HIGH   │     │
  │  │ eBPF (if unprivileged allowed)              HIGH   │     │
  │  │ Futex                                       MEDIUM │     │
  │  └─────────────────────────────────────────────────────┘     │
  │                                                              │
  │  From Network (remote attacker):                             │
  │  ┌─────────────────────────────────────────────────────┐     │
  │  │ TCP/IP stack                                HIGH   │     │
  │  │ Netfilter/nftables                          MEDIUM │     │
  │  │ Bluetooth stack                             HIGH   │     │
  │  │ WiFi drivers                                HIGH   │     │
  │  │ USB stack (physical access)                 HIGH   │     │
  │  └─────────────────────────────────────────────────────┘     │
  │                                                              │
  │  From Hardware:                                              │
  │  ┌─────────────────────────────────────────────────────┐     │
  │  │ DMA attacks (without IOMMU)                 HIGH   │     │
  │  │ Side channels (Spectre/Meltdown)            HIGH   │     │
  │  │ Rowhammer                                   MEDIUM │     │
  │  └─────────────────────────────────────────────────────┘     │
  │                                                              │
  │  Mitigations per surface:                                    │
  │    Syscalls → Seccomp filtering                              │
  │    /proc,/sys → Mount NS + hidepid                           │
  │    Network → Netfilter + Net NS                              │
  │    Hardware → IOMMU + KPTI + Spectre mitigations             │
  │    All → ASLR + NX + SMEP/SMAP + CFI + sanitizers           │
  └──────────────────────────────────────────────────────────────┘
```

---

## Summary

- Complete Linux security stack: Seccomp → DAC → LSM → Capabilities → Namespaces → Cgroups → Crypto → Integrity → Audit → Hardware
- struct cred: identity (UIDs) + capabilities + namespace + LSM blobs + keyrings
- LSM multiplexer: iterates all registered LSMs, all must allow (AND logic)
- Container: combines ALL mechanisms for defense in depth
- Secure boot: ROM → UEFI → shim → GRUB → kernel → modules → IMA
- Attack surface: syscalls, /proc, network stack, hardware — each has mitigations

---

Next: [Chapter 30 — Definitions of Important Terms](Chapter_30_Glossary.md)
