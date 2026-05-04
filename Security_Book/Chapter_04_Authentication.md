# Chapter 4: Authentication Mechanisms

## Learning Goals
- Understand user authentication concepts and flow
- Know PAM (Pluggable Authentication Modules) architecture
- Understand password storage and verification
- Know multi-factor authentication in Linux

---

## 4.1 Authentication Concepts

```
Authentication: Verifying "you are who you claim to be."

Authentication factors:
  1. Something you know:  Password, PIN, passphrase
  2. Something you have:  Hardware token, smart card, phone
  3. Something you are:   Biometrics (fingerprint, face)

Linux login flow:
  ┌──────────┐     ┌────────┐     ┌──────────┐     ┌────────┐
  │ User     │────►│ login/ │────►│ PAM      │────►│ Kernel │
  │ (tty/ssh)│     │ sshd   │     │ Stack    │     │ setuid │
  └──────────┘     └────────┘     └──────────┘     └────────┘
   username+        Service        Authenticate      Create
   password         program        + Authorize        session

  Detailed flow:
    1. User provides username + credentials
    2. Service (login/sshd/su) calls PAM
    3. PAM modules authenticate (password check, LDAP, etc.)
    4. If success: PAM authorizes (check access rules)
    5. Service calls setuid/initgroups to set credentials
    6. Kernel creates session with proper UID/GID/groups
    7. Shell or service started with user's credentials
```

---

## 4.2 Password Storage

```
/etc/passwd (world-readable):
  user:x:1000:1000:User Name:/home/user:/bin/bash
  │     │  │    │      │         │          │
  │     │  │    │      │         │          └── Login shell
  │     │  │    │      │         └── Home directory
  │     │  │    │      └── GECOS (full name)
  │     │  │    └── Primary GID
  │     │  └── UID
  │     └── 'x' = password in shadow file
  └── Username

/etc/shadow (root-readable only):
  user:$6$salt$hash:18000:0:99999:7:::
  │     │              │    │   │   │
  │     │              │    │   │   └── Expiry info
  │     │              │    │   └── Warn days
  │     │              │    └── Max days
  │     │              └── Last changed (days since epoch)
  │     └── Password hash ($algorithm$salt$hash)
  └── Username

  Hash algorithms:
    $1$  = MD5 (deprecated, weak)
    $5$  = SHA-256
    $6$  = SHA-512 (recommended)
    $y$  = yescrypt (modern, memory-hard)

  Salt: Random value to prevent rainbow table attacks
  Each user gets unique salt → same password → different hash
```

---

## 4.3 PAM (Pluggable Authentication Modules)

```
PAM: Modular framework separating authentication logic
from applications. Applications don't know HOW authentication
works — they just call PAM.

Architecture:
  ┌──────────────────────────────────────────────────────┐
  │  Application (login, sshd, su, sudo)                  │
  │    │                                                  │
  │    ├── pam_authenticate()    ← "Check credentials"   │
  │    ├── pam_acct_mgmt()       ← "Is account valid?"   │
  │    ├── pam_open_session()    ← "Start session"       │
  │    └── pam_setcred()         ← "Set credentials"     │
  │                                                       │
  │  ┌──────────────── PAM Library ──────────────────┐   │
  │  │                                                │   │
  │  │  Reads /etc/pam.d/<service> config file        │   │
  │  │                                                │   │
  │  │  Module Stacks:                                │   │
  │  │  ┌─────────────────────────────────────────┐   │   │
  │  │  │ auth:     Identity verification          │   │   │
  │  │  │ account:  Account validity/restrictions  │   │   │
  │  │  │ password: Password change operations     │   │   │
  │  │  │ session:  Session setup/teardown         │   │   │
  │  │  └─────────────────────────────────────────┘   │   │
  │  │                                                │   │
  │  │  Module chain evaluation:                      │   │
  │  │    required:    Must pass, continue checking   │   │
  │  │    requisite:   Must pass, fail immediately    │   │
  │  │    sufficient:  Pass → done, fail → continue   │   │
  │  │    optional:    Result ignored unless only one  │   │
  │  └────────────────────────────────────────────────┘   │
  │                                                       │
  │  ┌──────────────── PAM Modules ──────────────────┐   │
  │  │  pam_unix.so:     Traditional /etc/shadow auth │   │
  │  │  pam_ldap.so:     LDAP authentication          │   │
  │  │  pam_google_authenticator.so: TOTP 2FA         │   │
  │  │  pam_limits.so:   Resource limits (/etc/limits)│   │
  │  │  pam_securetty.so: Root login restrictions     │   │
  │  │  pam_deny.so:     Always deny                  │   │
  │  │  pam_permit.so:   Always allow                 │   │
  │  └────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────┘

Example PAM config (/etc/pam.d/sshd):
  auth     required   pam_sepermit.so
  auth     required   pam_env.so
  auth     sufficient pam_unix.so    try_first_pass
  auth     required   pam_deny.so
  account  required   pam_unix.so
  session  required   pam_limits.so
  session  required   pam_unix.so
```

---

## 4.4 Multi-Factor Authentication

```
MFA adds additional authentication factors beyond password.

TOTP (Time-based One-Time Password):
  - Shared secret + current time → 6-digit code
  - Google Authenticator, FreeOTP
  - PAM module: pam_google_authenticator.so

  Flow:
    1. User enrolls: scan QR code → shared secret stored
    2. Login: enter password + 6-digit code from app
    3. Server computes expected code from shared secret + time
    4. Compare → match = authenticated

Hardware tokens:
  - YubiKey: FIDO2/U2F, OTP, smart card
  - PAM module: pam_u2f.so
  - Challenge-response: kernel sends challenge, token signs with private key

SSH key authentication:
  - Public key in ~/.ssh/authorized_keys
  - Private key on client (optionally hardware-backed)
  - No password needed (or combined with password for MFA)
  - PAM module: pam_ssh.so

Certificate-based:
  - X.509 certificates for user authentication
  - Smart card + PIN
  - Common in enterprise/government
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| kernel/cred.c | Credential commit after auth |
| kernel/sys.c | setuid, setgid syscalls |
| kernel/groups.c | Group management |
| fs/exec.c | Exec with credential change |
| security/commoncap.c | Capability checks during exec |

---

## Interview Questions

**Q1: How does PAM work and why is it useful?**
A: PAM separates authentication logic from applications. Applications call PAM API functions (pam_authenticate, pam_acct_mgmt). PAM reads a per-service config (/etc/pam.d/<service>) that defines a stack of modules. Each module performs one check (password, LDAP, 2FA, etc.). Modules are evaluated with control flags (required, sufficient, etc.). This means: (1) Applications don't need auth code. (2) Auth methods can be changed without recompiling applications. (3) MFA can be added by inserting a module in the stack.

**Q2: Why is the password hash stored in /etc/shadow instead of /etc/passwd?**
A: /etc/passwd must be world-readable because many programs need to map UID→username. If password hashes were there, any user could try offline brute-force attacks. /etc/shadow is readable only by root (mode 0600), protecting hashes. The 'x' in /etc/passwd's password field indicates the hash is in /etc/shadow.

**Q3: What happens in the kernel when a user logs in?**
A: After PAM authenticates the user, the login program calls: (1) setuid/setgid to change its real/effective UID/GID. (2) initgroups() to set supplementary groups. (3) These trigger kernel credential changes via commit_creds(). (4) The kernel creates a new struct cred with the user's UID, GID, groups, and capabilities. (5) exec() is called to start the user's shell. (6) LSM hooks (security_task_fix_setuid, security_bprm_check) validate transitions.

---

## Summary

- Authentication verifies identity using factors: knowledge, possession, biometrics
- /etc/shadow stores password hashes (SHA-512 or yescrypt with salt)
- PAM provides modular, configurable authentication with stacked modules
- Control flags: required (must pass), sufficient (pass → done), requisite (fail → stop)
- MFA: TOTP (Google Authenticator), hardware tokens (YubiKey), SSH keys
- Kernel sets credentials after auth: UID, GID, groups, capabilities via commit_creds()

---

Next: [Chapter 5 — User and Group Management](Chapter_05_Users_Groups.md)
