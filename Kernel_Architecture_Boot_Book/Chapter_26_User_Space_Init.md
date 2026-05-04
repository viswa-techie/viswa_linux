# Chapter 26: User Space Initialization

## Learning Goals
- Understand the transition from kernel to user space
- Know init systems: SysVinit, Upstart, systemd, Android init
- Grasp PID 1 responsibilities
- Understand service management and boot targets

---

## 26.1 Kernel to User Space Transition

```
Last Kernel Steps Before User Space:

kernel_init() (PID 1, still in kernel space):
  │
  ├── kernel_init_freeable()
  │   ├── smp_init()              ← Bring up secondary CPUs
  │   ├── do_basic_setup()
  │   │   ├── driver_init()       ← Initialize driver model
  │   │   ├── do_initcalls()      ← All module_init() functions
  │   │   └── (devices probed, drivers loaded)
  │   │
  │   ├── If initramfs has /init:
  │   │   └── Return (kernel_init will exec /init)
  │   │
  │   └── If no initramfs:
  │       └── prepare_namespace() ← Mount root directly
  │
  ├── free_initmem()              ← Free __init sections
  │
  └── run_init_process()          ← exec() into user space
      ├── Try: /sbin/init
      ├── Try: /etc/init
      ├── Try: /bin/init
      ├── Try: /bin/sh
      └── panic("No init found")

After exec():
  PID 1 is now a user-space process
  It NEVER returns to kernel space (as kernel code)
  All future kernel interaction is via syscalls
```

---

## 26.2 PID 1 — The Init Process

```
PID 1 Responsibilities:

┌────────────────────────────────────────────────┐
│  PID 1 (init) is the ancestor of ALL processes │
│                                                │
│  Core duties:                                  │
│  1. Start system services in correct order     │
│  2. Reap orphaned zombie processes             │
│  3. Handle system signals (SIGCHLD, etc.)      │
│  4. Manage system state transitions            │
│     (boot → running → shutdown)                │
│  5. Restart crashed critical services          │
│                                                │
│  Special properties:                           │
│  - Cannot be killed (SIGKILL ignored)          │
│  - If PID 1 dies → kernel panic               │
│  - Orphaned processes reparented to PID 1      │
│  - First user-space process, last to exit      │
└────────────────────────────────────────────────┘

Process Tree:
  PID 0: idle (swapper) [kernel]
  ├── PID 1: init/systemd [user space]
  │   ├── PID 100: sshd
  │   ├── PID 200: cron
  │   ├── PID 300: login → bash
  │   └── PID 400: nginx
  └── PID 2: kthreadd [kernel]
      ├── PID 3: ksoftirqd/0
      ├── PID 4: kworker/0:0
      └── ...
```

---

## 26.3 SysVinit (Traditional)

```
SysVinit Boot Process:

/sbin/init reads /etc/inittab:

# /etc/inittab
id:3:initdefault:                    ← Default runlevel 3
si::sysinit:/etc/rc.d/rc.sysinit     ← System initialization
l3:3:wait:/etc/rc.d/rc 3             ← Start runlevel 3 scripts

Runlevels:
  0 — Halt
  1 — Single-user (rescue)
  2 — Multi-user (no network)
  3 — Multi-user (with network)
  4 — Unused (custom)
  5 — Graphical (X11)
  6 — Reboot

/etc/rc.d/rc 3 executes:
  /etc/rc.d/rc3.d/
  ├── S10network     → ../init.d/network start
  ├── S12syslog      → ../init.d/syslog start
  ├── S20sshd        → ../init.d/sshd start
  ├── S55cups        → ../init.d/cups start
  └── S99local       → ../init.d/local start

  K scripts run during shutdown (kill order)
  S scripts run during startup (start order)
  Numbers define ordering (10 before 20)

Problem: Sequential startup — each script waits for previous
```

---

## 26.4 systemd (Modern Linux)

```
systemd Boot Process:

systemd (PID 1) reads unit files:
  /etc/systemd/system/     ← Admin overrides (highest priority)
  /run/systemd/system/     ← Runtime units
  /usr/lib/systemd/system/ ← Package-installed units

Boot target (replaces runlevels):
  default.target → multi-user.target or graphical.target

Dependency resolution:
┌─────────────────────────────────────────────────┐
│  default.target (graphical.target)              │
│  ├── Wants: display-manager.service             │
│  ├── Requires: multi-user.target                │
│  │   ├── Wants: sshd.service                    │
│  │   ├── Wants: NetworkManager.service          │
│  │   ├── Wants: crond.service                   │
│  │   ├── Requires: basic.target                 │
│  │   │   ├── Requires: sysinit.target           │
│  │   │   │   ├── Wants: systemd-tmpfiles-setup  │
│  │   │   │   ├── Wants: systemd-sysctl          │
│  │   │   │   └── Wants: dev-hugepages.mount     │
│  │   │   └── Requires: sockets.target           │
│  │   │       └── Wants: dbus.socket             │
│  │   └── After: basic.target                    │
│  └── After: multi-user.target                   │
└─────────────────────────────────────────────────┘

Key advantage: Parallel startup
  Services without dependencies start simultaneously
  Socket activation: service starts only when needed
```

```ini
# Example systemd unit file
[Unit]
Description=My Application Service
After=network.target
Requires=postgresql.service

[Service]
Type=simple
ExecStart=/usr/bin/myapp --config /etc/myapp.conf
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# systemd commands
systemctl start myapp.service     # Start service
systemctl enable myapp.service    # Enable at boot
systemctl status myapp.service    # Check status
systemctl list-dependencies       # Show dependency tree
systemd-analyze blame             # Boot time per service
systemd-analyze critical-chain    # Critical path
journalctl -u myapp.service       # View service logs
```

---

## 26.5 Android Init System

```
Android Init Process:

Kernel exec's /init (Android's custom init):

/init reads configuration files:
  /system/etc/init/       ← System init scripts
  /vendor/etc/init/       ← Vendor-specific init scripts
  /odm/etc/init/          ← ODM-specific init scripts
  init.rc                 ← Main init configuration

Android Init Stages:
  ┌─────────────────────────────────────────────┐
  │  1. first_stage_init                        │
  │     ├── Mount /dev, /proc, /sys             │
  │     ├── Initialize SELinux                  │
  │     └── Load kernel modules                 │
  │                                             │
  │  2. second_stage_init                       │
  │     ├── Property service                    │
  │     ├── Parse .rc files                     │
  │     └── Execute actions and services        │
  │                                             │
  │  3. Action execution                        │
  │     ├── early-init actions                  │
  │     ├── init actions                        │
  │     ├── late-init actions                   │
  │     └── boot actions                        │
  └─────────────────────────────────────────────┘
```

```
# Android .rc file format
on init
    mkdir /dev/stune 0755 root root
    write /proc/sys/kernel/sched_child_runs_first 0

on boot
    chown system system /sys/class/leds/lcd-backlight/brightness
    chmod 0660 /sys/class/leds/lcd-backlight/brightness

service surfaceflinger /system/bin/surfaceflinger
    class core animation
    user system
    group graphics drmrpc readproc
    capabilities SYS_NICE
    onrestart restart zygote

service zygote /system/bin/app_process64 -Xzygote \
    /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart restart media
```

---

## 26.6 Boot Flow Comparison

```
Boot Flow Comparison:

SysVinit:
  kernel → /sbin/init → /etc/inittab → rc scripts (sequential)
  Time: ~60+ seconds

systemd:
  kernel → /usr/lib/systemd/systemd → unit files (parallel)
  Time: ~5-15 seconds

Android:
  kernel → /init → init.rc → services (staged, parallel)
  Time: ~10-30 seconds

                    SysVinit    systemd     Android
  ─────────────────────────────────────────────────
  Parallelism       No          Yes         Partial
  Socket activation No          Yes         No
  Cgroups           No          Yes         Yes
  Logging           syslog      journald    logd
  Service restart   Script      Built-in    Built-in
  Dependencies      Ordering    Declarative Trigger
  Config format     Shell       INI-like    .rc format
```

---

## Kernel Source References

| Function/File | Path | Purpose |
|-------|------|---------|
| kernel_init() | init/main.c | PID 1 kernel thread |
| run_init_process() | init/main.c | Exec into user space |
| kernel_init_freeable() | init/main.c | Pre-exec kernel setup |
| free_initmem() | arch/*/mm/init.c | Free __init memory |
| do_execve() | fs/exec.c | Execute user-space binary |
| Android init | system/core/init/ | Android init source |

---

## Interview Questions

**Q1: What happens if PID 1 crashes?**
A: The kernel panics. PID 1 cannot be killed — even SIGKILL is ignored for PID 1. If init exits for any reason, the kernel calls `panic("Attempted to kill init!")` because no other process can reap orphans or manage system state. systemd is designed to be robust against crashes in its own code paths.

**Q2: How does systemd achieve faster boot times than SysVinit?**
A: Three mechanisms: (1) Parallel startup — services without dependencies start simultaneously. (2) Socket activation — sockets are created early, services start on-demand when connections arrive. (3) Lazy loading — services start only when needed via D-Bus activation or socket activation. SysVinit runs scripts sequentially, each waiting for the previous to complete.

**Q3: How does Android's init differ from desktop Linux systemd?**
A: Android init uses `.rc` files with trigger-based actions and service definitions. It implements first-stage/second-stage init for SELinux policy loading. It uses properties (setprop/getprop) for runtime configuration instead of environment variables. Android's init also handles hardware-specific HAL services and uses a different process model (zygote fork for apps). systemd uses declarative unit files with dependency resolution.

---

## Summary

- `kernel_init()` (PID 1) execs into the first user-space process
- PID 1 is the ancestor of all processes — if it dies, the kernel panics
- SysVinit uses sequential shell scripts with runlevels
- systemd achieves parallel boot with unit files, socket activation, and dependency resolution
- Android init uses `.rc` files with trigger-based actions and staged initialization
- The init system manages the entire lifecycle: boot → running → shutdown

---

*Next: [Chapter 27 — Complete Boot Flow](Chapter_27_Complete_Boot_Flow.md)*
