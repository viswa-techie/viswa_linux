# Chapter 12: AppArmor

## Learning Goals
- Understand AppArmor's path-based security model
- Know how AppArmor profiles confine applications
- Understand profile modes and policy language
- Know AppArmor vs SELinux trade-offs

---

## 12.1 AppArmor Architecture

```
AppArmor: Path-based MAC system. Simpler than SELinux.
Default on Ubuntu, SUSE. Uses filesystem paths (not labels).

Architecture:
  ┌──────────────────────────────────────────────────┐
  │  User Space                                       │
  │                                                   │
  │  /etc/apparmor.d/           Profile directory     │
  │    usr.sbin.httpd           Profile files          │
  │    abstractions/            Shared rule fragments  │
  │    tunables/                Variables              │
  │                                                   │
  │  Tools:                                           │
  │    aa-enforce, aa-complain  Mode switching          │
  │    aa-genprof, aa-logprof   Profile generation     │
  │    aa-status                Show status             │
  │    apparmor_parser           Load/compile profiles  │
  └──────────────────────┬──────────────────────────┘
                          │ load profiles
  ┌──────────────────────▼──────────────────────────┐
  │  Kernel: AppArmor LSM Module                      │
  │                                                   │
  │  Profile Engine                                   │
  │    Path-based matching (no labels on files)       │
  │    DFA-compiled rules for fast matching           │
  │    Per-profile: file rules, network rules,         │
  │                 capability rules, mount rules      │
  │                                                   │
  │  LSM Hooks                                        │
  │    file_open → check path against profile rules   │
  │    capable   → check capability rules             │
  │    socket_*  → check network rules                │
  └──────────────────────────────────────────────────┘

Key difference from SELinux:
  SELinux:   Labels on inodes → survives rename/move
  AppArmor:  Paths → simpler but rename can bypass rules
```

---

## 12.2 Profile Structure and Language

```
Profile for Apache (/etc/apparmor.d/usr.sbin.httpd):

  #include <tunables/global>

  /usr/sbin/httpd {
    #include <abstractions/base>
    #include <abstractions/nameservice>

    # Capabilities
    capability net_bind_service,
    capability setuid,
    capability setgid,

    # Network rules
    network inet tcp,
    network inet6 tcp,

    # File rules — path-based
    /usr/sbin/httpd           mr,      # read + mmap
    /etc/httpd/**             r,       # read config
    /var/www/html/**          r,       # read content
    /var/log/httpd/*          w,       # write logs
    /run/httpd.pid            rw,      # pid file
    /tmp/                     rw,      # temp files
    /proc/*/status            r,       # process info

    # Deny explicitly
    deny /etc/shadow          r,
    deny /root/**             rwx,
  }

Permission flags:
  ┌──────┬──────────────────────────────────┐
  │ Flag │ Meaning                           │
  ├──────┼──────────────────────────────────┤
  │ r    │ Read                              │
  │ w    │ Write                             │
  │ a    │ Append                            │
  │ x    │ Execute                           │
  │ m    │ Memory map executable             │
  │ k    │ File locking                      │
  │ l    │ Link                              │
  │ ix   │ Inherit execute (same profile)    │
  │ cx   │ Child profile on execute          │
  │ px   │ Profile transition on execute     │
  │ ux   │ Unconfined execute (dangerous)    │
  │ Px   │ px with environment scrubbing     │
  │ Cx   │ cx with environment scrubbing     │
  │ Ux   │ ux with environment scrubbing     │
  └──────┴──────────────────────────────────┘

Globbing:
  /dir/*        Files in /dir (not subdirs)
  /dir/**       Files in /dir and all subdirs
  /dir/file*    Files starting with "file"
  /dir/{a,b}    /dir/a or /dir/b
  /dir/[abc]    /dir/a or /dir/b or /dir/c
```

---

## 12.3 Profile Modes

```
Enforce mode:
  Violations are blocked AND logged.
  aa-enforce /etc/apparmor.d/usr.sbin.httpd

Complain mode:
  Violations are logged but NOT blocked.
  aa-complain /etc/apparmor.d/usr.sbin.httpd
  Used for profile development/testing.

Unconfined:
  No profile loaded — process runs without AppArmor.

Kill mode:
  SIGKILL process on any violation.

Profile lifecycle:
  ┌─────────┐   aa-genprof   ┌───────────┐   test   ┌──────────┐
  │ No      │ ──────────────► │ Complain  │────────►│ Enforce  │
  │ Profile │   generate      │ Mode      │ refine  │ Mode     │
  └─────────┘                 └───────────┘          └──────────┘
                                   │
                              aa-logprof
                              (update from logs)

Status:
  $ aa-status
  apparmor module is loaded.
  28 profiles are loaded.
  28 profiles are in enforce mode.
  0 profiles are in complain mode.
  5 processes are confined.
```

---

## 12.4 Profile Generation Workflow

```bash
# Step 1: Generate initial profile interactively
aa-genprof /usr/sbin/httpd
# Starts httpd, monitors access, prompts for allow/deny

# Step 2: Switch to complain mode and exercise application
aa-complain /etc/apparmor.d/usr.sbin.httpd
# Run all use cases, generate all needed access patterns

# Step 3: Update profile from logged accesses
aa-logprof
# Reviews /var/log/syslog for AppArmor messages
# Prompts to add rules for each denied access

# Step 4: Enforce
aa-enforce /etc/apparmor.d/usr.sbin.httpd

# Load/reload profile manually:
apparmor_parser -r /etc/apparmor.d/usr.sbin.httpd

# Remove profile:
apparmor_parser -R /etc/apparmor.d/usr.sbin.httpd
```

---

## 12.5 Abstractions and Tunables

```
Abstractions — reusable rule sets:
  /etc/apparmor.d/abstractions/base
    Common rules: /dev/null, /dev/zero, /proc/self
  /etc/apparmor.d/abstractions/nameservice
    DNS resolution: /etc/resolv.conf, NSS libraries
  /etc/apparmor.d/abstractions/audio
    Audio device access rules
  /etc/apparmor.d/abstractions/X
    X11 display access

Tunables — variables:
  /etc/apparmor.d/tunables/global
    @{HOME} = /home/*/ /root/
    @{PROC} = /proc/
    @{sys} = /sys/

  Usage in profile:
    @{HOME}/.config/app/** r,
```

---

## 12.6 AppArmor vs SELinux

```
  ┌──────────────────┬──────────────────┬──────────────────┐
  │ Feature          │ AppArmor         │ SELinux           │
  ├──────────────────┼──────────────────┼──────────────────┤
  │ Approach         │ Path-based       │ Label-based       │
  │ Complexity       │ Simpler          │ More complex      │
  │ Granularity      │ Good             │ Very fine-grained │
  │ File rename      │ Policy bypassed  │ Label follows     │
  │ Default distro   │ Ubuntu, SUSE     │ RHEL, Fedora      │
  │ Learning curve   │ Lower            │ Higher            │
  │ Policy language  │ Profile files    │ .te/.if/.fc files │
  │ Network control  │ Basic            │ Port labeling     │
  │ MLS/MCS          │ No               │ Yes               │
  │ Stacking         │ Yes (minor LSM)  │ Yes (major LSM)   │
  │ Android          │ No               │ Yes               │
  │ Container support│ Docker default   │ Kubernetes/OpenShift│
  └──────────────────┴──────────────────┴──────────────────┘
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| security/apparmor/lsm.c | AppArmor LSM hook implementations |
| security/apparmor/apparmorfs.c | AppArmor securityfs interface |
| security/apparmor/policy.c | Profile management |
| security/apparmor/domain.c | Domain transitions |
| security/apparmor/file.c | File access mediation |
| security/apparmor/match.c | DFA-based path matching |

---

## Interview Questions

**Q1: How does AppArmor differ from SELinux?**
A: AppArmor uses path-based access control — rules reference filesystem paths like `/var/www/**`. SELinux uses label-based access control — rules reference types assigned to inodes. This makes AppArmor simpler to understand and configure, but AppArmor rules can be bypassed by renaming/moving files since the path changes while SELinux labels stick to inodes. AppArmor is default on Ubuntu/SUSE, SELinux on RHEL/Fedora. AppArmor lacks MLS/MCS (multi-level security) support. SELinux has finer granularity, especially for network security (port labels).

**Q2: How do you create and deploy an AppArmor profile?**
A: (1) Use `aa-genprof /path/to/program` to generate an initial profile interactively. (2) Switch to complain mode: `aa-complain /etc/apparmor.d/profile`. (3) Exercise the application through all use cases — access attempts are logged but not blocked. (4) Run `aa-logprof` to review logs and add rules for each denied access. (5) Repeat steps 3-4 until all needed accesses are covered. (6) Switch to enforce mode: `aa-enforce /etc/apparmor.d/profile`. (7) Monitor for issues and refine as needed. Use abstractions for common rule sets (base, nameservice, etc.).

**Q3: What are AppArmor execution modes for child processes?**
A: When a confined program executes another program: `ix` (inherit) — child runs under same profile. `cx` — child transitions to a named child profile defined within the parent profile. `px` — child transitions to a separate named profile (profile must exist). `ux` — child runs unconfined (no AppArmor restrictions, dangerous). Capital versions (`Px`, `Cx`, `Ux`) add environment variable scrubbing for safety. The choice depends on the security model: `px` for well-defined services, `ix` when child needs same access, `ux` only when no other option works.

---

## Summary

- AppArmor uses path-based MAC — rules reference filesystem paths, not labels
- Profiles define per-program: file access, capabilities, network, mount rules
- Complain mode logs violations without blocking; enforce mode blocks
- Profile generation: aa-genprof → aa-complain → aa-logprof → aa-enforce
- Abstractions provide reusable rule fragments (base, nameservice, etc.)
- Simpler than SELinux but less granular; default on Ubuntu/SUSE
- Docker uses AppArmor by default for container confinement

---

Next: [Chapter 13 — Smack](Chapter_13_Smack.md)
