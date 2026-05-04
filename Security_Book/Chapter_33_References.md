# Chapter 33: Documentation and References

## Learning Goals
- Know where to find authoritative Linux security documentation
- Have references for further deep-dive study
- Know key papers, books, and online resources

---

## 33.1 Kernel Documentation

```
Official kernel documentation (in kernel source tree):

  Documentation/security/
    LSM.rst                     Linux Security Modules overview
    credentials.rst             Credential management
    keys/                       Keyring subsystem docs
      core.rst                  Key/keyring concepts
      trusted-encrypted.rst     Trusted/encrypted key types
      request-key.rst           Key request upcall
    IMA-templates.rst           IMA measurement templates
    SELinux.rst                 SELinux kernel interface
    apparmor.rst                AppArmor kernel interface
    Smack.rst                   Smack overview
    TOMOYO.rst                  TOMOYO overview
    Yama.rst                    Yama ptrace restrictions
    landlock.rst                Landlock sandboxing
    self-protection.rst         KSPP hardening guide
    siphash.rst                 SipHash for hashtable keys

  Documentation/admin-guide/
    kernel-parameters.txt       Boot parameters (security-related)
    sysctl/kernel.rst           sysctl settings
    LSM/                        LSM admin guides
    module-signing.rst          Module signature verification

  Documentation/process/
    embargoed-hardware-issues.rst  Hardware vulnerability disclosure

Key resources:
  make htmldocs                  Build HTML documentation
  Online: https://www.kernel.org/doc/html/latest/
```

---

## 33.2 Essential Books

```
  ┌──────────────────────────────────────────────────────────┐
  │  Security Books                                          │
  │                                                          │
  │  Linux Kernel:                                           │
  │    "Linux Kernel Development" - Robert Love              │
  │      Chapter on security subsystem                       │
  │                                                          │
  │    "Understanding the Linux Kernel" - Bovet & Cesati     │
  │      Detailed internals including security               │
  │                                                          │
  │    "Linux Device Drivers" - Corbet, Rubini, Kroah-Hartman│
  │      Security considerations for driver development      │
  │                                                          │
  │  Security-Specific:                                      │
  │    "SELinux by Example" - Mayer, MacMillan, Caplan       │
  │      Comprehensive SELinux guide                         │
  │                                                          │
  │    "The SELinux Notebook" - Richard Haines               │
  │      Complete SELinux reference (free online)            │
  │                                                          │
  │    "Container Security" - Liz Rice                       │
  │      Linux security mechanisms for containers            │
  │                                                          │
  │    "Linux Security Cookbook" - Barrett, Silverman, Byrnes │
  │      Practical security recipes                          │
  │                                                          │
  │  General Security:                                       │
  │    "Practical Binary Analysis" - Dennis Andriesse        │
  │      Understanding exploitation and mitigations          │
  │                                                          │
  │    "The Art of Software Security Assessment"             │
  │      Mark Dowd, John McDonald, Justin Schuh              │
  │      Code auditing techniques                            │
  └──────────────────────────────────────────────────────────┘
```

---

## 33.3 Key Research Papers and Standards

```
Foundational Papers:

  "The Inevitability of Failure: The Flawed Assumption of
   Security in Modern Computing Environments" (1998)
    Peter Loscocco, Stephen Smalley (NSA) — motivation for SELinux

  "Implementing SELinux as a Linux Security Module" (2003)
    Smalley, Vance, Salamon — SELinux architecture

  "Linux Security Modules: General Security Support
   for the Linux Kernel" (2002)
    Wright, Cowan, Smalley, James, Morris — LSM framework

  "Seccomp sandboxing in Linux" (2012)
    Will Drewry — Seccomp-BPF design

  "Container Security: Fundamental Technology Concepts
   that Protect Containerized Applications" (2020)
    Liz Rice — comprehensive container security

Standards:
  Common Criteria (ISO 15408): Security evaluation framework
  FIPS 140-2/140-3: Cryptographic module validation
  ISO/SAE 21434: Automotive cybersecurity
  NIST SP 800-53: Security and privacy controls
  CIS Benchmarks: Hardening guidelines for Linux distributions
  STIG (DISA): Security Technical Implementation Guides
```

---

## 33.4 Online Resources

```
Official:
  kernel.org/doc       Kernel documentation
  selinuxproject.org   SELinux project
  apparmor.net         AppArmor project
  kernsec.org          Kernel Security mailing list
  cve.mitre.org        CVE database

Mailing lists:
  linux-security-module@vger.kernel.org   LSM development
  selinux@vger.kernel.org                 SELinux development
  linux-kernel@vger.kernel.org            General kernel (security patches)

Tools:
  github.com/google/syzkaller             Kernel fuzzer
  github.com/google/kasan                 Address sanitizer docs
  github.com/a13xp0p0v/kernel-hardening-checker
    Checks kernel config against KSPP recommendations

Conferences (security research):
  Linux Security Summit (LSS)
  Linux Plumbers Conference (security microconference)
  Kernel Recipes
  USENIX Security Symposium
  IEEE S&P / CCS / NDSS

CVE tracking:
  nvd.nist.gov                            NIST vulnerability database
  ubuntu.com/security/cves               Ubuntu CVE tracker
  access.redhat.com/security/cve         Red Hat CVE tracker
```

---

## 33.5 Source Code Navigation Tools

```
Tools for navigating kernel security source code:

  Elixir (bootlin.com/linux/source):
    Online kernel source cross-reference
    Search symbols, navigate code
    Browse all kernel versions

  cscope:
    cscope -R -b    # Build cross-reference database
    cscope -d       # Query mode
    Find: definition, callers, callees, text strings

  ctags / etags:
    ctags -R .      # Build tags file
    Navigate to definition / declaration

  grep + git:
    git log --oneline security/  # Security subsystem history
    git log --all --grep="CVE-"  # Find CVE fix commits
    git blame security/selinux/hooks.c  # Line-by-line history

  kernel config search:
    make menuconfig → / (search for CONFIG option)
    scripts/config --state CONFIG_SECURITY_SELINUX

  /proc/crypto:      List available crypto algorithms
  /proc/keys:        List keys in session keyring
  /proc/self/ns/:    Show namespace membership
  /proc/self/status: Show seccomp mode and capability sets
```

---

## 33.6 Security Hardening References

```
Distribution hardening guides:

  CIS Benchmarks:
    CIS Ubuntu Linux Benchmark
    CIS Red Hat Enterprise Linux Benchmark
    CIS Debian Linux Benchmark
    Comprehensive sysctl, file permissions, audit rules

  DISA STIGs:
    Security Technical Implementation Guides
    DoD-required hardening (very strict)
    Available for RHEL, Ubuntu, SUSE

  KSPP (Kernel Self Protection Project):
    kernsec.org/wiki/index.php/Kernel_Self_Protection_Project
    Recommended kernel CONFIG options
    Tracking of mainlined and pending protections

  kernel-hardening-checker:
    Compares running kernel config against KSPP recommendations
    ./kernel-hardening-checker -c /boot/config-$(uname -r)
    Reports: OK, FAIL, or missing for each security option

  NSA/CISA hardening guides:
    "Hardening Linux" guides for various distributions
    Container security guides
    Kubernetes hardening guide
```

---

## Summary

- Kernel docs: Documentation/security/ in kernel source tree
- Key books: SELinux Notebook, Container Security (Rice), Linux Kernel Development (Love)
- Papers: LSM (2002), SELinux (2003), Seccomp-BPF (2012)
- Standards: Common Criteria, FIPS 140, ISO 21434, CIS Benchmarks, STIGs
- Online: kernel.org/doc, selinuxproject.org, kernsec.org
- Tools: Elixir (bootlin), cscope, kernel-hardening-checker, syzkaller
- Mailing lists: linux-security-module@vger, selinux@vger
- CVE tracking: NVD, Ubuntu/Red Hat CVE trackers

---

Next: [Chapter 34 — Interview Preparation](Chapter_34_Interview_Preparation.md)
