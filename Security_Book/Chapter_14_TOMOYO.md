# Chapter 14: TOMOYO Linux

## Learning Goals
- Understand TOMOYO's pathname-based and domain-based model
- Know TOMOYO's learning mode for automatic policy generation
- Understand domain transitions and policy structure
- Know TOMOYO's unique approach compared to other LSMs

---

## 14.1 TOMOYO Architecture

```
TOMOYO: Pathname-based + domain-based MAC.
Unique feature: automatic policy learning from system behavior.

Architecture:
  ┌──────────────────────────────────────────────────┐
  │  User Space                                       │
  │                                                   │
  │  /sys/kernel/security/tomoyo/   securityfs        │
  │    domain_policy                Domain definitions │
  │    exception_policy             Exceptions         │
  │    profile                      Mode per domain    │
  │    manager                      Policy managers    │
  │                                                   │
  │  Tools: tomoyo-editpolicy (ncurses UI)            │
  │         tomoyo-loadpolicy, tomoyo-savepolicy       │
  │         tomoyo-queryd (interactive daemon)         │
  └──────────────────────┬──────────────────────────┘
                          │
  ┌──────────────────────▼──────────────────────────┐
  │  Kernel: TOMOYO LSM Module                       │
  │                                                   │
  │  Domain Tree                                      │
  │    <kernel>                      Root domain      │
  │    ├── <kernel> /sbin/init       Init domain      │
  │    │   ├── <kernel> /sbin/init /usr/sbin/sshd    │
  │    │   │   └── <kernel> /sbin/init /usr/sbin/sshd /bin/bash │
  │    │   ├── <kernel> /sbin/init /usr/sbin/httpd   │
  │    │   └── ...                                    │
  │    └── ...                                        │
  │                                                   │
  │  Each domain has:                                 │
  │    - File access rules (read/write/execute)       │
  │    - Network rules (connect/listen/bind)           │
  │    - Capability rules                              │
  │    - Mount rules                                   │
  │    - Environment variable rules                    │
  └──────────────────────────────────────────────────┘

Domain = full execution history path from kernel boot.
  NOT just the current process — the entire chain of execs.
```

---

## 14.2 Domain Transitions

```
Domain is defined by the complete execution path from boot:

  <kernel>
  <kernel> /sbin/init
  <kernel> /sbin/init /usr/sbin/sshd
  <kernel> /sbin/init /usr/sbin/sshd /bin/bash
  <kernel> /sbin/init /usr/sbin/sshd /bin/bash /usr/bin/vim

  Same binary, different domains:
    /bin/bash started from sshd:
      <kernel> /sbin/init /usr/sbin/sshd /bin/bash
    /bin/bash started from httpd:
      <kernel> /sbin/init /usr/sbin/httpd /bin/bash

  These are DIFFERENT domains with different permissions.
  The access path matters — context of HOW a program was reached.

Transition:
  ┌──────────┐   exec(sshd)   ┌──────────┐  exec(bash)  ┌──────────┐
  │ <kernel> │ ──────────────► │ <kernel> │ ────────────►│ <kernel> │
  │ /sbin/   │                 │ /sbin/   │              │ /sbin/   │
  │ init     │                 │ init     │              │ init     │
  │          │                 │ /usr/    │              │ /usr/    │
  │          │                 │ sbin/    │              │ sbin/    │
  │          │                 │ sshd     │              │ sshd     │
  │          │                 │          │              │ /bin/    │
  │          │                 │          │              │ bash     │
  └──────────┘                 └──────────┘              └──────────┘
```

---

## 14.3 Four Operating Modes

```
TOMOYO has four modes per domain (most unique feature):

  ┌────────────────┬────────────────────────────────────────┐
  │ Mode           │ Behavior                                │
  ├────────────────┼────────────────────────────────────────┤
  │ Disabled       │ No checking, no logging                 │
  │ Learning       │ Allow all, auto-generate policy rules   │
  │ Permissive     │ Log violations, don't block             │
  │ Enforcing      │ Log violations AND block                │
  └────────────────┴────────────────────────────────────────┘

Learning mode workflow:
  ┌──────────┐   deploy    ┌──────────┐  refine   ┌──────────┐
  │ Learning │ ──────────► │ Permissive│ ────────► │ Enforcing│
  │ Mode     │  auto-gen   │ Mode      │  fix gaps │ Mode     │
  └──────────┘  rules      └──────────┘            └──────────┘

  1. Set domain to "learning" mode
  2. Exercise the application through ALL normal use cases
  3. TOMOYO automatically records every access and creates rules
  4. Review auto-generated policy
  5. Switch to permissive — check for missed accesses
  6. Switch to enforcing — production

  This is TOMOYO's killer feature:
    No manual policy writing needed.
    The system learns what the application does.
```

---

## 14.4 Policy Language

```
Domain policy example:

  <kernel> /sbin/init /usr/sbin/httpd
  use_profile 3                            # Enforcing mode
  
  # File access rules
  allow_read /etc/httpd/conf/httpd.conf
  allow_read /var/www/html/\*
  allow_read /var/www/html/\{\*\}/\*       # Recursive
  allow_write /var/log/httpd/\*
  allow_create /run/httpd.pid
  allow_execute /usr/sbin/httpd
  
  # Network rules
  allow_network TCP bind 0.0.0.0 80
  allow_network TCP listen 0.0.0.0 80
  allow_network TCP accept 0.0.0.0 0-65535
  
  # Capability-like rules
  allow_capability SYS_CHROOT
  allow_capability NET_BIND_SERVICE

Exception policy (global):
  # Initialize domain transition
  initialize_domain /usr/sbin/sshd from <kernel> /sbin/init

  # Keep domain (prevent transition)
  keep_domain <kernel> /sbin/init /bin/sh

  # Path group (alias for multiple paths)
  path_group SYSTEM_LIBS /lib/\*.so\*
  path_group SYSTEM_LIBS /lib64/\*.so\*
  path_group SYSTEM_LIBS /usr/lib/\*.so\*
```

---

## 14.5 TOMOYO Interactive Mode

```
tomoyo-queryd: Interactive policy daemon.

When TOMOYO encounters an access not covered by policy,
it can ask the administrator interactively:

  Process httpd (domain: <kernel> /sbin/init /usr/sbin/httpd)
  wants to read /etc/ssl/certs/ca-bundle.crt

  Allow? (y/n/r/q)
    y = Allow and add to policy
    n = Deny
    r = Retry (don't decide yet)
    q = Quit queryd

This enables real-time policy building.
Useful for development but NOT for production.
```

---

## 14.6 TOMOYO Compared to Other LSMs

```
  ┌──────────────────┬──────────┬──────────┬──────────┬──────────┐
  │ Feature          │ SELinux  │ AppArmor │ Smack    │ TOMOYO   │
  ├──────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Model            │ Label    │ Path     │ Label    │ Path+    │
  │                  │          │          │          │ Domain   │
  │ Auto-learn       │ No       │ Partial  │ No       │ Yes      │
  │ Policy complexity│ High     │ Medium   │ Low      │ Low      │
  │ Execution context│ No       │ No       │ No       │ Yes      │
  │ Network control  │ Strong   │ Basic    │ CIPSO    │ Socket   │
  │ Admin tool       │ semanage │ aa-*     │ chsmack  │ ncurses  │
  │ Code size (LOC)  │ ~30K     │ ~15K     │ ~6K      │ ~10K     │
  │ Primary users    │ RHEL,    │ Ubuntu,  │ Tizen,   │ Research,│
  │                  │ Android  │ SUSE     │ AGL      │ Embedded │
  └──────────────────┴──────────┴──────────┴──────────┴──────────┘

TOMOYO unique strengths:
  1. Execution history awareness — same binary, different policy
  2. Learning mode — no manual policy writing
  3. Interactive query daemon — real-time decisions
  4. Low barrier to entry — easiest to start using
  5. Good for system behavior analysis/auditing
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| security/tomoyo/tomoyo.c | TOMOYO LSM hook implementations |
| security/tomoyo/domain.c | Domain transition logic |
| security/tomoyo/file.c | File access mediation |
| security/tomoyo/network.c | Network access mediation |
| security/tomoyo/common.c | Shared utilities and policy parsing |
| security/tomoyo/securityfs_if.c | securityfs interface |

---

## Interview Questions

**Q1: What makes TOMOYO unique among Linux MAC implementations?**
A: TOMOYO has two unique features. First, domain-based access control using the full execution history — a domain is the complete chain of executables from boot (e.g., `<kernel> /sbin/init /usr/sbin/sshd /bin/bash`). The same binary has different permissions depending on how it was reached, providing context-aware security. Second, learning mode — TOMOYO can automatically generate policy by observing normal application behavior. You exercise the app in learning mode, TOMOYO records every access, then you switch to enforcing. No manual policy writing is needed, making it the easiest LSM to deploy.

**Q2: Explain TOMOYO's four operating modes.**
A: (1) Disabled: no checking, no logging. (2) Learning: allows all access but automatically records rules for every access made — this generates policy from actual behavior. (3) Permissive: checks rules and logs violations but doesn't block — used to verify learned policy is complete. (4) Enforcing: blocks and logs violations. The workflow is: start in learning mode, exercise the application, review auto-generated rules, test in permissive mode, then deploy in enforcing mode. Modes can be set per-domain, so different parts of the system can be in different modes.

**Q3: How does TOMOYO's domain model differ from SELinux domains?**
A: SELinux domains are type labels assigned to processes (e.g., `httpd_t`). Any httpd process has the same domain regardless of how it was started. TOMOYO domains encode the full execution path from boot: `<kernel> /sbin/init /usr/sbin/httpd`. If bash is started from sshd, its domain is `<kernel> /sbin/init /usr/sbin/sshd /bin/bash`. If bash is started from httpd, its domain is `<kernel> /sbin/init /usr/sbin/httpd /bin/bash`. These are different domains with different permissions. This captures execution context — the same program can have different privileges depending on who started it and how, without needing explicit domain transition rules like SELinux.

---

## Summary

- TOMOYO uses pathname-based + execution-history-based domains
- Domain = full exec chain from boot: <kernel> /sbin/init /usr/sbin/sshd /bin/bash
- Same binary in different contexts gets different permissions
- Four modes: disabled, learning, permissive, enforcing
- Learning mode auto-generates policy from observed behavior
- Interactive query daemon for real-time policy decisions
- ~10K LOC — simpler than SELinux, good for embedded and research
- Primary users: research, system analysis, embedded Linux

---

Next: [Chapter 15 — Seccomp](Chapter_15_Seccomp.md)
