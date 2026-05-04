# Chapter 12: Dynamic Debug

## Learning Goals
- Understand pr_debug/dev_dbg and CONFIG_DYNAMIC_DEBUG
- Learn dyndbg control file syntax and usage
- Master per-file, per-function, per-line debug enable
- Know format flags and module-level control

---

## 1. Dynamic Debug Overview

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  pr_debug() / dev_dbg() are NO-OPs by default           │
  │  Dynamic debug: enable them at runtime, selectively     │
  │                                                           │
  │  Without dynamic debug:                                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ CONFIG_DYNAMIC_DEBUG not set:            │            │
  │  │ pr_debug() → compiled out entirely       │            │
  │  │ (only visible if #define DEBUG before     │            │
  │  │  including <linux/printk.h>)             │            │
  │  │ → No runtime control                    │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  With dynamic debug:                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ CONFIG_DYNAMIC_DEBUG=y                   │            │
  │  │                                          │            │
  │  │ All pr_debug()/dev_dbg() compiled in but │            │
  │  │ disabled (a NOP check + branch)          │            │
  │  │                                          │            │
  │  │ Can be enabled/disabled at runtime:      │            │
  │  │ - Per file                               │            │
  │  │ - Per function                           │            │
  │  │ - Per line number                        │            │
  │  │ - Per module                             │            │
  │  │ - By format string match                │            │
  │  │                                          │            │
  │  │ Cost when disabled: single branch check  │            │
  │  │ (negligible overhead)                    │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. dyndbg Control Interface

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Control file: /sys/kernel/debug/dynamic_debug/control   │
  │  (or: /proc/dynamic_debug/control)                      │
  │                                                           │
  │  View available debug statements:                        │
  │  ┌──────────────────────────────────────────┐            │
  │  │ cat /sys/kernel/debug/dynamic_debug/control│          │
  │  │                                          │            │
  │  │ drivers/usb/core/hub.c:1234 [usbcore]    │            │
  │  │   hub_port_connect =_ "port %d\n"        │            │
  │  │                                          │            │
  │  │ Fields: file:line [module] func =flags "fmt"│        │
  │  │ Flags: p=print, f=function, l=line,      │            │
  │  │        m=module, t=thread, _=disabled    │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Enable by file:                                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ echo 'file hub.c +p' > \                 │            │
  │  │   /sys/kernel/debug/dynamic_debug/control │            │
  │  │ # Enable all pr_debug in hub.c           │            │
  │  │                                          │            │
  │  │ echo 'file hub.c -p' > \                 │            │
  │  │   /sys/kernel/debug/dynamic_debug/control │            │
  │  │ # Disable all pr_debug in hub.c          │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Enable by function:                                     │
  │  echo 'func hub_port_connect +p' > ...control            │
  │                                                           │
  │  Enable by module:                                       │
  │  echo 'module usbcore +p' > ...control                   │
  │                                                           │
  │  Enable by line:                                         │
  │  echo 'file hub.c line 1234 +p' > ...control             │
  │                                                           │
  │  Enable by format string:                                │
  │  echo 'format "port %d" +p' > ...control                 │
  │                                                           │
  │  Combined:                                               │
  │  echo 'file hub.c func hub_probe +p' > ...control        │
  │                                                           │
  │  Format flags:                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ +p  — enable printing (the main one)    │            │
  │  │ +f  — include function name in output   │            │
  │  │ +l  — include line number               │            │
  │  │ +m  — include module name               │            │
  │  │ +t  — include thread ID                 │            │
  │  │                                          │            │
  │  │ echo 'file hub.c +pflmt' > ...control    │            │
  │  │ → prints with func + line + module + tid│            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Boot-time activation:                                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Kernel command line:                     │            │
  │  │ dyndbg="file hub.c +p; module e1000 +p"  │            │
  │  │                                          │            │
  │  │ Module load time:                        │            │
  │  │ modprobe drm dyndbg=+p                    │            │
  │  │ → enable all debug in drm module at load │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does dynamic debug work internally and what's its runtime cost?**
**A:** Each `pr_debug()`/`dev_dbg()` call site is registered in a `struct _ddebug` descriptor table (in the `__dyndbg` ELF section), containing the file name, function name, line number, format string, and a flags field. At compile time with `CONFIG_DYNAMIC_DEBUG=y`, the debug print is compiled as: `if (unlikely(__dynamic_dbg_enabled(descriptor))) printk(KERN_DEBUG ...)`. The `__dynamic_dbg_enabled()` check reads a single byte (the flags field) from the descriptor — if the `p` flag is set, the printk executes; otherwise it's skipped. Runtime cost when disabled: one unlikely branch (an `if` that's predicted not-taken by the CPU branch predictor). This is essentially zero cost — no function call, no string formatting. When enabled: normal printk cost (format string processing, ring buffer write). The control file (`/proc/dynamic_debug/control`) is a kernel interface that modifies the flags byte in `struct _ddebug` entries matching the user's query pattern (file, function, module, format). This is a simple byte write to each matching descriptor — takes effect immediately, no module reload needed.

**Q2: When should you use pr_debug/dynamic debug vs printk with explicit log levels?**
**A:** Use `pr_debug()`/`dev_dbg()` for: (1) Verbose debugging output that's too noisy for production — function entry/exit, register dumps, protocol state machines, data flow tracing. These should be compiled in but disabled by default. (2) Per-subsystem/per-driver debug that operators can enable at runtime to diagnose specific issues without kernel rebuilds. (3) Development-time debugging — commit the debug prints, they have zero cost when disabled. Use explicit log levels (`pr_err`, `pr_warn`, `pr_info`) for: (1) Error conditions that indicate bugs or hardware failures (pr_err). (2) Unusual but recoverable situations operators should know about (pr_warn). (3) Important lifecycle events: device probe success, driver loaded, firmware version (pr_info — these should be in production output). Rule of thumb: if you'd want to see it in a production dmesg, use pr_info or higher. If it's only useful when actively debugging a problem, use pr_debug. The worst anti-pattern is using `pr_info` for verbose debug output — it fills the dmesg log and can't be silenced without rebuilding the kernel. Dynamic debug + pr_debug gives you the best of both worlds: verbose debugging available on demand with zero production cost.

---

## Summary

- CONFIG_DYNAMIC_DEBUG: compiles pr_debug/dev_dbg as conditional branches (zero cost when off)
- Control: echo 'file X +p' > /sys/kernel/debug/dynamic_debug/control
- Selectors: file, func, module, line, format — can combine
- Flags: +p (print), +f (function), +l (line), +m (module), +t (thread ID)
- Boot-time: dyndbg= on kernel cmdline; load-time: modprobe mod dyndbg=+p
- Use pr_debug for verbose debugging; pr_info/pr_err for production-visible messages

---

[Previous: printk Architecture ←](Chapter_11_printk.md) | [Next: dmesg and Kernel Logging →](Chapter_13_dmesg_Logging.md)
