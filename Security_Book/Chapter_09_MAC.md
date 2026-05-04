# Chapter 9: Mandatory Access Control (MAC)

## Learning Goals
- Understand MAC vs DAC security models
- Know formal security models (Bell-LaPadula, Biba, Clark-Wilson)
- Understand how MAC is implemented in Linux via LSM

---

## 9.1 MAC Security Model

```
MAC (Mandatory Access Control):
  Access decisions enforced by SYSTEM POLICY, not by object owners.
  Users CANNOT override or change MAC policy.

DAC (Discretionary Access Control):
  File OWNER decides who can access (chmod, chown).
  Owner CAN relax security.

  ┌──────────────────────── DAC vs MAC ────────────────────────┐
  │                                                            │
  │  DAC:                                                      │
  │  ┌──────────┐     "I'll make my file                      │
  │  │ User     │      world-readable"                        │
  │  │ (owner)  │────► chmod 644 secret.txt                   │
  │  └──────────┘     User has discretion                      │
  │                                                            │
  │  MAC:                                                      │
  │  ┌──────────┐     "I want to make it                      │
  │  │ User     │      world-readable"                        │
  │  │ (owner)  │────► chmod 644 secret.txt                   │
  │  └──────────┘            │                                │
  │                          ▼                                │
  │                 ┌──────────────────┐                       │
  │                 │ MAC Policy says: │                       │
  │                 │ "secret.txt has  │                       │
  │                 │ label TOP_SECRET"│                       │
  │                 │ Only cleared     │                       │
  │                 │ processes can    │                       │
  │                 │ access it."      │                       │
  │                 │ → DENIED         │                       │
  │                 └──────────────────┘                       │
  │                                                            │
  │  Even root is subject to MAC policy (in enforcing mode).   │
  └────────────────────────────────────────────────────────────┘
```

---

## 9.2 Formal Security Models

```
Bell-LaPadula Model (Confidentiality):
  "No read up, no write down"
  ┌──────────────┐
  │ TOP SECRET   │ ← Can write here (no write down)
  ├──────────────┤
  │ SECRET       │ ← Subject at this level
  ├──────────────┤
  │ CONFIDENTIAL │ ← Can read here (no read up)
  ├──────────────┤
  │ UNCLASSIFIED │ ← Can read here
  └──────────────┘
  Prevents information from flowing to lower levels.
  Used in military/government systems.

Biba Model (Integrity):
  "No read down, no write up"
  Opposite of Bell-LaPadula.
  Prevents low-integrity data from corrupting high-integrity data.
  Subject can only write to same or lower integrity level.

Clark-Wilson Model (Integrity):
  Well-formed transactions + separation of duty.
  Constrained Data Items (CDI) accessed only through
  Transformation Procedures (TP).
  Commercial systems — ensures data integrity.

Linux MAC implementations:
  SELinux:  Type Enforcement (most powerful, most complex)
  AppArmor: Profile-based (simpler, path-based)
  Smack:    Simple label-based (embedded/IoT)
  TOMOYO:   Domain-based (learning mode, path-based)
```

---

## 9.3 MAC in Linux — How Policy Is Enforced

```
MAC policy enforcement in Linux kernel:

  1. Object labeling:
     Every file, process, socket, IPC object gets a security label.
     SELinux: "system_u:object_r:httpd_content_t:s0"
     Smack:    "WebContent"

  2. Subject labeling:
     Every process has a security context/label.
     SELinux: "system_u:system_r:httpd_t:s0"
     Smack:    "Apache"

  3. Policy rules:
     Rules define which subject labels can access which object labels.
     SELinux: allow httpd_t httpd_content_t:file { read open };
     Smack:   Apache WebContent rx

  4. Enforcement:
     ┌───────────────────────────────────────────────────────┐
     │ Process (httpd_t) wants to read file (httpd_content_t)│
     │                                                       │
     │  DAC: Is file readable by process UID? → YES          │
     │  MAC: Does policy allow httpd_t to read               │
     │       httpd_content_t? → Check AVC (Access Vector     │
     │       Cache) → Match found → ALLOW                    │
     │                                                       │
     │  Process (httpd_t) wants to read /etc/shadow           │
     │  (shadow_t)                                           │
     │                                                       │
     │  DAC: Is file readable? → YES (process is root)       │
     │  MAC: Does policy allow httpd_t to read shadow_t?     │
     │       → NO RULE → DENY (even though root!)            │
     └───────────────────────────────────────────────────────┘

  5. Modes:
     Enforcing:  Policy is enforced (denials block access)
     Permissive: Policy is checked but NOT enforced (log only)
     Disabled:   MAC is completely off
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| security/security.c | LSM hook dispatch (DAC then MAC) |
| security/selinux/avc.c | SELinux Access Vector Cache |
| security/selinux/hooks.c | SELinux MAC hook implementations |
| security/apparmor/lsm.c | AppArmor MAC hooks |
| security/smack/smack_lsm.c | Smack MAC hooks |

---

## Interview Questions

**Q1: What is the difference between DAC and MAC?**
A: **DAC** (Discretionary): object owner controls access (chmod, chown). Users can relax security on their files. Problem: compromised root can access anything, users can leak data. **MAC** (Mandatory): system-wide policy controls access regardless of owner. Even root is constrained. Policy is set by administrator, not individual users. In Linux, DAC checks first, then MAC (LSM). Both must allow. A web server running as root under MAC can only access files its security label allows — it cannot read arbitrary system files.

**Q2: How does MAC prevent privilege escalation?**
A: In DAC, root bypasses all checks — a compromised setuid binary gives full access. In MAC: (1) even root processes have a security label (e.g., httpd_t in SELinux). (2) The label determines what the process can access, regardless of UID. (3) An attacker who compromises httpd can only access httpd_content_t files — not /etc/shadow, not kernel modules, not other services. (4) Domain transitions on exec are controlled by policy — can't just exec a shell to escape. This is called "defense in depth" or "sandboxing."

---

## Summary

- MAC: System-wide policy enforcement, users cannot override
- DAC: Owner-controlled permissions, root bypasses all
- Bell-LaPadula: no read up, no write down (confidentiality)
- Biba: no read down, no write up (integrity)
- Linux MAC via LSM: SELinux, AppArmor, Smack, TOMOYO
- Labels on subjects (processes) and objects (files, sockets)
- Policy rules define allowed access between labels
- Even root is constrained under MAC in enforcing mode

---

Next: [Chapter 10 — Discretionary Access Control (DAC)](Chapter_10_DAC.md)
