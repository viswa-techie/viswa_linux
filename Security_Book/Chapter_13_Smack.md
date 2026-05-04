# Chapter 13: Smack (Simplified Mandatory Access Control Kernel)

## Learning Goals
- Understand Smack's label-based simplicity
- Know Smack rule syntax and access control model
- Understand how Smack differs from SELinux
- Know Smack use cases in embedded/IoT systems

---

## 13.1 Smack Architecture

```
Smack: Simple label-based MAC. Designed for minimal complexity.
Each subject/object gets a text label. Rules define label→label access.

Architecture:
  ┌─────────────────────────────────────────────────────┐
  │  User Space                                          │
  │                                                      │
  │  /sys/fs/smackfs/        Smack virtual filesystem    │
  │    load2                 Load access rules           │
  │    access2               Check access                │
  │    cipso2                CIPSO label mapping          │
  │    onlycap               Restrict CAP_MAC_ADMIN      │
  │                                                      │
  │  Tools: smackctl, chsmack                             │
  └──────────────────────┬──────────────────────────────┘
                          │
  ┌──────────────────────▼──────────────────────────────┐
  │  Kernel: Smack LSM Module                            │
  │                                                      │
  │  Label Store                                         │
  │    Simple text labels on subjects and objects         │
  │    Stored in security.SMACK64 xattr                  │
  │                                                      │
  │  Rule Engine                                         │
  │    Linear rule list: subject label → object label    │
  │    Default: deny (unless special labels)              │
  │                                                      │
  │  Built-in special labels:                             │
  │    _        Floor    — readable by all               │
  │    ^        Hat      — readable by all, no read out  │
  │    *        Star     — not accessible by anyone      │
  │    ?        Huh      — accessible by all             │
  │    @        Web      — writable by all               │
  └──────────────────────────────────────────────────────┘

Decision logic:
  1. If subject == object label → ALLOW (same label = same domain)
  2. Check special labels (_ ^ * ? @)
  3. Lookup rule list for subject→object pair
  4. If rule found → apply permissions
  5. If no rule → DENY
```

---

## 13.2 Smack Labels and Rules

```
Label assignment:
  chsmack -a "WebApp" /var/www/html/index.html  # Set access label
  chsmack -e "WebApp" /usr/bin/httpd             # Set exec label

  attr -S -g SMACK64 /var/www/html/index.html    # Read label

Rule syntax:
  subject_label object_label permissions

  Permissions: r (read), w (write), x (execute), a (append),
               t (transmute), l (lock)

Examples:
  # WebApp can read WebContent
  echo "WebApp WebContent r" > /sys/fs/smackfs/load2

  # System can read and write AppData
  echo "System AppData rw" > /sys/fs/smackfs/load2

  # WebApp can execute SystemBins
  echo "WebApp SystemBins rx" > /sys/fs/smackfs/load2

Rule lookup:
  ┌────────────┐              ┌────────────┐
  │ Subject    │   access?    │ Object     │
  │ label:     │ ────────────►│ label:     │
  │ "WebApp"   │              │ "WebContent"│
  └────────────┘              └────────────┘
         │
         ▼
  Rule list lookup:
    WebApp WebContent r    → ALLOW read
    WebApp WebContent w    → NOT in rules → DENY write
```

---

## 13.3 Smack Transmutation

```
Transmute: directory inherits parent's label to new files.

Without transmute:
  Process with label "App" creates file in dir labeled "Data":
    New file gets label "App" (creator's label)

With transmute:
  Directory /data has transmute bit set:
    chsmack -t /data
  Process with label "App" creates file in /data:
    New file gets label "Data" (directory's label)

  Use case: shared directories where multiple processes
  create files that should all have the same label.

  ┌──────────┐  create file  ┌─────────────────────┐
  │ Process  │ ─────────────►│ Dir: label="Shared" │
  │ "App"    │               │ transmute=true       │
  └──────────┘               │                      │
                              │ New file gets label  │
                              │ "Shared" not "App"   │
                              └─────────────────────┘
```

---

## 13.4 Smack Network Labels (CIPSO)

```
Smack uses CIPSO (Common IP Security Option) for network labeling.

CIPSO encodes Smack labels in IP option headers:
  - Labels mapped to CIPSO DOI (Domain of Interpretation)
  - Packets carry label in IP options field
  - Receiving host checks label against Smack rules

Configuration:
  # Map Smack label to CIPSO level/category
  echo "WebApp 4/2" > /sys/fs/smackfs/cipso2

  # Set ambient label for unlabeled packets
  echo "Internet" > /sys/fs/smackfs/ambient

  # Add host-based label
  echo "192.168.1.0/24 Internal" > /sys/fs/smackfs/netlabel
```

---

## 13.5 Smack Use Cases

```
Primary use case: Embedded and IoT systems.

Why Smack for embedded:
  1. Simple policy — a few labels and rules
  2. Small code footprint (~6000 lines vs SELinux ~30000)
  3. No complex policy compiler needed
  4. Easy to understand and audit
  5. Good for resource-constrained devices

Tizen OS (Samsung):
  Uses Smack as primary MAC mechanism
  Apps get unique labels
  Platform services have defined labels
  Rules define inter-app and service access

Automotive (AGL - Automotive Grade Linux):
  Uses Smack for process isolation
  Different ECU functions get different labels
  Simple policy for safety-critical separation

  ┌──────────────────────────────────────────────┐
  │  Embedded System with Smack                   │
  │                                               │
  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐     │
  │  │App1  │  │App2  │  │Sys   │  │Net   │     │
  │  │label │  │label │  │label │  │label │     │
  │  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘     │
  │     │         │         │         │          │
  │  Rules: App1→Sys:rx, App2→Sys:rx             │
  │         Sys→Net:rw, App1→App2:DENY           │
  └──────────────────────────────────────────────┘
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| security/smack/smack_lsm.c | Smack LSM hook implementations |
| security/smack/smack_access.c | Access decision logic |
| security/smack/smackfs.c | Smack virtual filesystem |
| security/smack/smack_netfilter.c | Network label enforcement |
| security/smack/smack.h | Smack data structures |

---

## Interview Questions

**Q1: What makes Smack suitable for embedded systems?**
A: Smack is a simplified MAC implementation (~6000 lines of code vs ~30000 for SELinux). It uses simple text labels on subjects/objects and straightforward rules: `subject_label object_label permissions`. No complex policy compiler is needed — rules are written as plain text lines. The simple model is easy to audit and verify, which matters for safety-critical embedded systems. Tizen OS and Automotive Grade Linux (AGL) use Smack because embedded systems need security isolation without the overhead and complexity of SELinux. The small footprint and minimal configuration make it practical for resource-constrained devices.

**Q2: How does Smack's access control model work?**
A: Smack assigns text labels to every process (subject) and every file/object. Access rules are defined as triples: `subject_label object_label permissions`. When a process accesses an object, Smack checks: (1) if same label → allow (same domain). (2) Check special labels (_ floor = readable by all, * star = denied to all, ? huh = accessible by all). (3) Look up rule list for the subject→object label pair. (4) If a matching rule is found, apply its permissions. (5) If no rule matches, deny. The model is simple: no types, roles, or transitions — just labels and rules.

**Q3: What is Smack transmutation?**
A: Normally, when a process creates a file, the file inherits the process's Smack label. With transmutation enabled on a directory (`chsmack -t`), new files inherit the directory's label instead. This enables shared directories where files from multiple processes all get the same label, making cross-process access work without complex rules. For example, if processes labeled "App1" and "App2" create files in a transmuted directory labeled "SharedData", all files get the "SharedData" label, and any process with access to "SharedData" can access them.

---

## Summary

- Smack uses simple text labels on subjects and objects
- Rules: `subject_label object_label permissions` — plain text
- Same-label access is always allowed; default is deny
- Special labels: _ (floor), ^ (hat), * (star), ? (huh), @ (web)
- Transmutation: files inherit directory label instead of creator label
- CIPSO for network packet labeling
- ~6000 LOC — designed for embedded/IoT (Tizen, AGL)
- Trade-off: simplicity over granularity

---

Next: [Chapter 14 — TOMOYO](Chapter_14_TOMOYO.md)
