# Chapter 28: Security Flow Diagrams

## Learning Goals
- Visualize complete security check flows in the kernel
- Understand the order of security mechanisms in syscalls
- See how multiple security layers interact

---

## 28.1 System Call Security Check Flow

```
Complete security checks when a process makes a syscall:

  User Process
      │
      │  syscall instruction (int 0x80 / syscall)
      ▼
  ┌─────────────────────────────────────────────────┐
  │  ENTRY: syscall entry point                      │
  │                                                   │
  │  Step 1: SECCOMP CHECK                            │
  │    __secure_computing()                           │
  │    Run BPF filter chain                           │
  │    ALLOW → continue                               │
  │    KILL  → SIGSYS, terminate                      │
  │    ERRNO → return error immediately               │
  │                                                   │
  │  Step 2: AUDIT (if configured)                    │
  │    Record syscall entry for audit log              │
  │                                                   │
  │  Step 3: SYSCALL HANDLER                          │
  │    sys_open(), sys_read(), sys_connect(), etc.     │
  │      │                                            │
  │      ▼                                            │
  │  Step 4: DAC CHECK (Permission bits)              │
  │    generic_permission() / acl_permission_check()  │
  │    Check file mode bits against UID/GID           │
  │    Check POSIX ACLs if present                    │
  │    DENIED → return -EACCES                        │
  │      │                                            │
  │      ▼                                            │
  │  Step 5: LSM HOOKS (MAC check)                    │
  │    security_file_open()                           │
  │    security_inode_permission()                    │
  │    Calls ALL registered LSMs:                     │
  │      SELinux:  selinux_inode_permission()          │
  │      AppArmor: apparmor_file_open()               │
  │      Other:   each LSM checks its policy          │
  │    ALL must ALLOW (AND logic)                     │
  │    ANY deny → return -EACCES                      │
  │      │                                            │
  │      ▼                                            │
  │  Step 6: OPERATION EXECUTES                       │
  │    Actual file open / network connect / etc.       │
  │      │                                            │
  │      ▼                                            │
  │  Step 7: AUDIT (exit)                             │
  │    Record syscall result for audit log             │
  │                                                   │
  │  RETURN to user space                             │
  └─────────────────────────────────────────────────┘
```

---

## 28.2 File Open Security Flow

```
open("/etc/passwd", O_RDONLY):

  User process
      │ open()
      ▼
  Seccomp: Is sys_open allowed? ────────── KILL/ERRNO if no
      │ yes
      ▼
  VFS: do_sys_open()
      │
      ▼
  Path lookup: path_openat() → link_path_walk()
      │
      ▼
  Each path component (/etc, passwd):
      │
      ├── inode_permission() for directory traversal
      │     │
      │     ├── DAC: generic_permission()
      │     │     Check execute bit on directory
      │     │     for current UID/GID
      │     │
      │     └── LSM: security_inode_permission()
      │           SELinux: check search permission on dir
      │           AppArmor: check path against profile
      │
      ▼
  Final file: do_open()
      │
      ├── may_open() → inode_permission()
      │     │
      │     ├── DAC: Check read permission
      │     │     mode & S_IROTH (or owner/group)
      │     │
      │     └── LSM: security_file_open()
      │           SELinux: allow httpd_t httpd_content_t:file { read open }
      │           AppArmor: /etc/passwd r (in profile?)
      │
      ├── IMA: ima_file_check()
      │     Measure file hash / verify signature
      │
      ▼
  Return file descriptor to user
```

---

## 28.3 Process Execution Security Flow

```
execve("/usr/bin/myapp", argv, envp):

  User process (domain A)
      │ execve()
      ▼
  Seccomp: Is sys_execve allowed? ────── KILL if no
      │ yes
      ▼
  do_execveat_common()
      │
      ▼
  open executable file
      │
      ├── DAC: Check execute permission on file
      │
      ├── LSM: security_bprm_creds_for_exec()
      │     SELinux: Compute new domain (type transition?)
      │     AppArmor: Determine new profile
      │
      ├── Check setuid/setgid bits → adjust credentials
      │     But NO_NEW_PRIVS blocks this if set
      │
      ├── LSM: security_bprm_check()
      │     SELinux: Verify domain transition is allowed
      │     AppArmor: Verify profile transition is allowed
      │
      ├── IMA: ima_bprm_check()
      │     Verify file integrity before execution
      │
      ├── Module signature check (if kernel module)
      │
      ▼
  Load ELF binary
      │
      ├── ASLR: Randomize stack, mmap, heap addresses
      │
      ├── NX: Mark stack/heap non-executable
      │
      ├── LSM: security_bprm_committed_creds()
      │     SELinux: Complete domain transition
      │     AppArmor: Complete profile transition
      │
      ▼
  New process running in domain B
  (with new credentials, new security context)
```

---

## 28.4 Network Connection Security Flow

```
connect(sockfd, &addr, sizeof(addr)):

  User process
      │ connect()
      ▼
  Seccomp: Is sys_connect allowed? ────── KILL if no
      │ yes
      ▼
  sys_connect()
      │
      ▼
  LSM: security_socket_connect()
      │
      ├── SELinux: Check domain can connect to target IP/port
      │     allow httpd_t http_port_t:tcp_socket { name_connect }
      │
      ├── AppArmor: Check network rules in profile
      │     network inet tcp,
      │
      ├── Smack: Check subject→object label access
      │
      ▼
  Netfilter: OUTPUT chain
      │
      ├── nftables/iptables rules
      │     Match: source, dest, port, protocol
      │     Action: ACCEPT / DROP / REJECT
      │
      ├── Connection tracking (conntrack)
      │     Create NEW connection entry
      │
      ▼
  TCP: tcp_v4_connect()
      │
      ├── Send SYN packet
      │
      ▼
  Netfilter: POSTROUTING chain
      │
      ├── NAT (if configured)
      │
      ▼
  NIC: transmit packet
      │
      ▼
  Response arrives → Netfilter PREROUTING → INPUT → socket
  (conntrack marks as ESTABLISHED)
```

---

## 28.5 Container Creation Security Flow

```
Container runtime creates a new container:

  Container Runtime (Docker/Podman)
      │
      ▼
  1. Create namespaces:
     clone(CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWNET |
           CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWUSER)
      │
      ▼
  2. Setup user namespace mapping:
     Write /proc/<pid>/uid_map: "0 100000 65536"
     Write /proc/<pid>/gid_map: "0 100000 65536"
      │
      ▼
  3. Setup mount namespace:
     pivot_root() to container rootfs
     Mount /proc, /sys, /dev (minimal)
     Apply read-only bind mounts
      │
      ▼
  4. Setup network namespace:
     Create veth pair
     Move one end to container NS
     Assign IP address
     Setup routing
      │
      ▼
  5. Setup cgroup limits:
     echo $PID > /sys/fs/cgroup/container_1/cgroup.procs
     echo 512M > memory.max
     echo 100  > pids.max
     echo "200000 100000" > cpu.max
      │
      ▼
  6. Drop capabilities:
     Drop all except minimal set (14 of 41)
      │
      ▼
  7. Install seccomp filter:
     prctl(PR_SET_NO_NEW_PRIVS, 1)
     seccomp(SECCOMP_SET_MODE_FILTER, 0, &filter)
     Block ~44 dangerous syscalls
      │
      ▼
  8. Apply MAC profile:
     SELinux: set context container_t
     AppArmor: set docker-default profile
      │
      ▼
  9. exec() container entrypoint
     All security layers now active
     Process runs in fully sandboxed environment
```

---

## 28.6 Secure Boot Flow

```
Power On
    │
    ▼
Hardware Root of Trust (ROM)
    │ Verify firmware signature
    ▼
UEFI Firmware
    │ Check db/dbx signature databases
    │ Verify bootloader signature
    ▼
Shim (signed by Microsoft)
    │ Check MOK (Machine Owner Key) database
    │ Verify GRUB signature
    ▼
GRUB bootloader
    │ Verify kernel signature
    ▼
Linux Kernel
    │ CONFIG_MODULE_SIG_FORCE
    │ Verify module signatures
    │ CONFIG_LOCKDOWN
    │ Block /dev/mem, unsigned kexec, etc.
    ▼
Kernel Modules (.ko)
    │ Each verified against builtin/platform/machine keys
    ▼
IMA/EVM (optional)
    │ Verify user-space binary integrity
    │ Measure into TPM PCRs
    ▼
User Space
    │ Fully verified chain from hardware to applications
    ▼
Running System (integrity maintained)
```

---

## Summary

- Syscall flow: Seccomp → Audit → DAC → LSM (MAC) → Operation → Audit
- File open: Seccomp → path walk (DAC+LSM per component) → final check → IMA
- Execve: Seccomp → DAC → LSM domain transition → IMA → ASLR/NX → new domain
- Network: Seccomp → LSM → Netfilter → conntrack → transmit
- Container: clone NS → uid_map → pivot_root → cgroups → capabilities → seccomp → MAC
- Secure Boot: ROM → UEFI → shim → GRUB → kernel → modules → IMA

---

Next: [Chapter 29 — Important Security Diagrams](Chapter_29_Architecture_Diagrams.md)
