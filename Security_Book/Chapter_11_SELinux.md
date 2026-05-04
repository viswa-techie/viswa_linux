# Chapter 11: SELinux (Security-Enhanced Linux)

## Learning Goals
- Understand SELinux architecture and type enforcement
- Know security contexts, domains, and types
- Understand SELinux policy language basics
- Know how to debug SELinux denials

---

## 11.1 SELinux Architecture

```
SELinux: Developed by NSA, most powerful Linux MAC implementation.
Uses Type Enforcement (TE) as primary access control mechanism.

Architecture:
  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │  User Space:                                           │
  │    Policy tools: semanage, sesearch, audit2allow       │
  │    Policy files: .te, .if, .fc → compiled to policy.X │
  │    Management: setenforce, getenforce, setsebool       │
  │                                                        │
  │  ┌─────────── SELinux Kernel ────────────────────┐     │
  │  │                                                │     │
  │  │  Security Server (ss/)                         │     │
  │  │    Policy database (loaded from user space)    │     │
  │  │    Access decision engine                      │     │
  │  │                                                │     │
  │  │  AVC (Access Vector Cache)                     │     │
  │  │    Caches recent decisions for performance     │     │
  │  │    LRU eviction                                │     │
  │  │                                                │     │
  │  │  LSM Hooks (hooks.c)                           │     │
  │  │    200+ hooks calling avc_has_perm()           │     │
  │  │                                                │     │
  │  │  labeling: Extended attributes (security.selinux) │  │
  │  │                                                │     │
  │  └────────────────────────────────────────────────┘     │
  │                                                        │
  └────────────────────────────────────────────────────────┘

Decision flow:
  LSM hook → avc_has_perm() → AVC cache hit? → return cached
       │
       └→ AVC cache miss → security_compute_av() → policy lookup
          → compute access vector → cache result → return
```

---

## 11.2 Security Contexts

```
Every process, file, socket, port has a security context:

  user:role:type:level
  │     │    │     │
  │     │    │     └── MLS/MCS sensitivity level (s0-s15:c0-c1023)
  │     │    └── TYPE — most important for access control
  │     └── ROLE — constrains which types a user can enter
  └── USER — SELinux user (mapped from Linux UID)

Examples:
  Process:  system_u:system_r:httpd_t:s0
  File:     system_u:object_r:httpd_content_t:s0
  Port:     system_u:object_r:http_port_t:s0 (port 80)
  Device:   system_u:object_r:tty_device_t:s0

  Viewing contexts:
    ps -eZ              # Process contexts
    ls -Z               # File contexts
    id -Z               # Current process context
    semanage port -l    # Port contexts

  Type Enforcement (TE):
    Subjects (processes) have DOMAIN types:  httpd_t, init_t, etc.
    Objects (files, sockets) have types:     httpd_content_t, etc.
    Rules:  allow <domain> <type>:<class> { <permissions> };
```

---

## 11.3 Type Enforcement Policy

```
Policy rule syntax:
  allow source_type target_type : object_class { permissions };

Examples:
  # Apache can read web content
  allow httpd_t httpd_content_t:file { read open getattr };

  # Apache can create TCP sockets
  allow httpd_t self:tcp_socket { create listen accept };

  # Apache can bind to HTTP ports
  allow httpd_t http_port_t:tcp_socket { name_bind };

  # Init can transition to httpd domain on exec
  type_transition init_t httpd_exec_t:process httpd_t;

Object classes and permissions:
  ┌──────────────┬──────────────────────────────────────────┐
  │ Class        │ Permissions                               │
  ├──────────────┼──────────────────────────────────────────┤
  │ file         │ read write open create unlink getattr     │
  │              │ setattr execute execute_no_trans           │
  │ dir          │ read write search add_name remove_name    │
  │ process      │ fork signal transition ptrace             │
  │ tcp_socket   │ create connect listen accept bind         │
  │ udp_socket   │ create connect bind sendto recvfrom       │
  │ unix_stream  │ create connect listen accept              │
  │ capability   │ sys_admin net_admin net_raw net_bind      │
  │ chr_file     │ read write open ioctl                     │
  └──────────────┴──────────────────────────────────────────┘

Domain transitions:
  ┌──────────┐  exec(/usr/sbin/httpd)  ┌──────────┐
  │ init_t   │ ───────────────────────► │ httpd_t  │
  └──────────┘                          └──────────┘

  Requires:
    allow init_t httpd_exec_t:file { read execute open };
    allow init_t httpd_t:process { transition };
    type_transition init_t httpd_exec_t:process httpd_t;
```

---

## 11.4 SELinux Modes and Booleans

```
Modes:
  Enforcing:   Policy is enforced. Denials block access.
               setenforce 1 / getenforce

  Permissive:  Policy is checked but NOT enforced. Denials logged.
               setenforce 0
               Useful for policy development/debugging.

  Disabled:    SELinux completely off. Requires reboot to re-enable.
               SELINUX=disabled in /etc/selinux/config

Per-domain permissive:
  semanage permissive -a httpd_t
  # Only httpd_t is permissive, rest remain enforcing

Booleans — Runtime policy switches:
  getsebool -a                          # List all booleans
  setsebool httpd_can_network_connect 1 # Allow httpd network
  setsebool -P httpd_can_network_connect 1 # Persistent

  Common booleans:
    httpd_can_network_connect: Allow HTTP to connect outbound
    httpd_can_sendmail: Allow HTTP to send email
    allow_ptrace: Allow ptrace
    container_manage_cgroup: Allow containers to manage cgroups
```

---

## 11.5 Debugging SELinux Denials

```bash
# Check for denials in audit log:
ausearch -m AVC -ts recent

# Typical denial message:
# type=AVC msg=audit(1234567890.123:456):
#   avc: denied { read } for pid=1234 comm="httpd"
#   name="index.html" dev="sda1" ino=67890
#   scontext=system_u:system_r:httpd_t:s0
#   tcontext=system_u:object_r:default_t:s0
#   tclass=file permissive=0

# Interpretation:
#   httpd_t (Apache) tried to READ a file labeled default_t
#   No allow rule exists for httpd_t → default_t:file { read }

# Fix: Properly label the file
restorecon -R /var/www/html/     # Restore default labels
chcon -t httpd_content_t /var/www/html/index.html  # Manual label

# Or generate policy from denials:
audit2allow -a                   # From all denials
audit2allow -i /var/log/audit/audit.log  # From log file
audit2allow -M my_module -i /var/log/audit/audit.log
semodule -i my_module.pp         # Install generated module

# Search existing policy:
sesearch --allow -s httpd_t -t httpd_content_t
sesearch --allow -s httpd_t -c file
```

---

## 11.6 SELinux in Android

```
Android uses SELinux in enforcing mode since Android 5.0.

Key differences from desktop SELinux:
  - Minimal policy — only allow what's explicitly needed
  - No targeted policy — everything is confined
  - No unconfined domain (unlike Fedora/RHEL)
  - Policy compiled into boot image

Android domain examples:
  init_t          → Init process
  system_server   → System Server
  platform_app    → Platform-signed apps
  untrusted_app   → Third-party apps
  hal_*           → Hardware abstraction layers
  vendor_*        → Vendor-specific processes

Android sepolicy files:
  system/sepolicy/         → AOSP base policy
  device/<vendor>/sepolicy/ → Device-specific policy
  vendor/<vendor>/sepolicy/ → Vendor policy
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| security/selinux/hooks.c | SELinux LSM hook implementations |
| security/selinux/avc.c | Access Vector Cache |
| security/selinux/ss/ | Security Server (policy engine) |
| security/selinux/ss/avtab.c | Access vector table |
| security/selinux/ss/policydb.c | Policy database |
| security/selinux/include/security.h | SELinux internals |

---

## Interview Questions

**Q1: How does SELinux Type Enforcement work?**
A: Every process has a domain type (e.g., httpd_t) and every object has a type (e.g., httpd_content_t). Policy rules define allowed access: `allow httpd_t httpd_content_t:file { read open }`. When httpd tries to read a file, the kernel checks the AVC (Access Vector Cache). On cache miss, the Security Server computes the access decision from the policy database. If no allow rule exists, access is denied — even for root. This confines each process to only the resources its type permits.

**Q2: How do you debug an SELinux denial?**
A: (1) Check audit log: `ausearch -m AVC -ts recent`. (2) Read the denial: identify scontext (process type), tcontext (target type), tclass (object class), permission denied. (3) Determine cause: wrong file label → `restorecon`. Missing policy → `audit2allow` to generate rule. (4) If file labeling issue: `chcon` or `restorecon`. (5) If legitimate access: write custom policy module, compile, install with `semodule -i`. (6) Use `sesearch` to verify existing rules. (7) For testing: `semanage permissive -a <domain>` to make one domain permissive.

**Q3: How is SELinux used in Android?**
A: Android enforces SELinux since 5.0 with a strict policy — everything is confined, no unconfined domains. Third-party apps run as untrusted_app, system services as system_server, HALs as hal_*. The policy is compiled into the boot image and loaded at boot. AOSP provides base policy, vendors add device-specific policy. This prevents privileged apps from accessing other apps' data, restricts what root processes can do, and is a critical layer in Android's security model alongside sandboxing and permissions.

---

## Summary

- SELinux uses Type Enforcement: domain types (processes) + types (objects) + allow rules
- Security context: user:role:type:level — TYPE is most important
- AVC caches decisions for performance; Security Server handles misses
- Modes: enforcing (blocks), permissive (logs), disabled
- Booleans: runtime policy switches (setsebool)
- Debug: ausearch → read denial → restorecon or audit2allow
- Android: full SELinux enforcement since 5.0, no unconfined domains

---

Next: [Chapter 12 — AppArmor](Chapter_12_AppArmor.md)
