# Chapter 25: Debugging Techniques

## Chapter Overview

Debugging kernel drivers requires specialized tools since conventional debuggers are impractical in kernel space. This chapter covers the full debugging toolkit: kernel logging, dynamic debug, tracing, ftrace, devcoredump, and common debug workflows.

---

## 25.1 Kernel Logging — printk and dev_*

### printk Log Levels

```c
#define KERN_EMERG   "0"   /* System is unusable */
#define KERN_ALERT   "1"   /* Must take action immediately */
#define KERN_CRIT    "2"   /* Critical conditions */
#define KERN_ERR     "3"   /* Error conditions */
#define KERN_WARNING "4"   /* Warning conditions */
#define KERN_NOTICE  "5"   /* Normal but significant */
#define KERN_INFO    "6"   /* Informational */
#define KERN_DEBUG   "7"   /* Debug-level messages */
```

### Device-Aware Logging (Always Prefer)

```c
dev_err(&pdev->dev, "register read timeout\n");
dev_warn(&pdev->dev, "fallback to polling mode\n");
dev_info(&pdev->dev, "initialized, IRQ %d\n", irq);
dev_dbg(&pdev->dev, "reg[0x%x] = 0x%08x\n", offset, val);
```

Output: `my-device 1e000000.uart: register read timeout`

### Rate-Limited Logging

```c
/* Don't flood logs in hot paths */
dev_err_ratelimited(dev, "FIFO overflow\n");
dev_warn_once(dev, "deprecated feature used\n");
printk_ratelimited(KERN_ERR "repeated error\n");
```

### dev_err_probe() — Deferred Probe Handling

```c
clk = devm_clk_get(dev, "core");
if (IS_ERR(clk))
    return dev_err_probe(dev, PTR_ERR(clk), "failed to get core clock\n");
/* If -EPROBE_DEFER: logs at debug level (not error), returns the error */
/* If real error: logs at error level, returns the error */
```

---

## 25.2 Dynamic Debug

Enable/disable `dev_dbg()` and `pr_debug()` at runtime without recompilation.

```bash
# Enable all debug messages in a file
echo 'file drivers/my/driver.c +p' > /sys/kernel/debug/dynamic_debug/control

# Enable for a specific function
echo 'func my_probe +p' > /sys/kernel/debug/dynamic_debug/control

# Enable for a module
echo 'module my_driver +p' > /sys/kernel/debug/dynamic_debug/control

# Add function name and line number to output
echo 'file drivers/my/driver.c +pfl' > /sys/kernel/debug/dynamic_debug/control

# Disable
echo 'file drivers/my/driver.c -p' > /sys/kernel/debug/dynamic_debug/control
```

Flags: `p` (print), `f` (function name), `l` (line number), `m` (module name), `t` (thread ID)

Requires: `CONFIG_DYNAMIC_DEBUG=y`

---

## 25.3 Ftrace — Function Tracer

The most powerful kernel tracing tool for drivers.

```bash
# Available tracers
cat /sys/kernel/debug/tracing/available_tracers
# nop function function_graph

# Trace a specific function
echo 'my_probe' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# Read trace output
cat /sys/kernel/debug/tracing/trace

# Function graph tracer (shows call hierarchy)
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'my_*' > /sys/kernel/debug/tracing/set_graph_function
```

### Function Graph Output Example

```
 0)               |  my_probe() {
 0)   0.850 us    |    devm_platform_ioremap_resource();
 0)   0.420 us    |    devm_clk_get();
 0)   0.380 us    |    clk_prepare_enable();
 0)   1.200 us    |    devm_request_threaded_irq();
 0)   0.150 us    |    pm_runtime_enable();
 0) + 12.500 us   |  }
```

### Trace Events

```bash
# List available events for a subsystem
ls /sys/kernel/debug/tracing/events/irq/

# Enable IRQ enter/exit tracing
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable

# Enable DMA mapping events
echo 1 > /sys/kernel/debug/tracing/events/dma/enable

# Output
cat /sys/kernel/debug/tracing/trace_pipe
```

---

## 25.4 trace_printk() — Printf Inside Trace Buffer

```c
/* Much faster than printk — writes to trace ring buffer */
trace_printk("DMA transfer complete, bytes=%zu\n", len);

/* View output in ftrace */
cat /sys/kernel/debug/tracing/trace
```

**Warning**: `trace_printk()` is for debugging only. The kernel build system warns if left in production code.

---

## 25.5 Debugfs

For driver-internal debug information not part of the stable ABI.

```c
#include <linux/debugfs.h>

static struct dentry *my_debugfs_dir;

static int my_regs_show(struct seq_file *s, void *unused)
{
    struct my_priv *priv = s->private;
    seq_printf(s, "CTRL:   0x%08x\n", readl(priv->base + REG_CTRL));
    seq_printf(s, "STATUS: 0x%08x\n", readl(priv->base + REG_STATUS));
    seq_printf(s, "IRQ:    0x%08x\n", readl(priv->base + REG_IRQ));
    return 0;
}
DEFINE_SHOW_ATTRIBUTE(my_regs);

static int my_probe(struct platform_device *pdev)
{
    /* ... */

    my_debugfs_dir = debugfs_create_dir("my-driver", NULL);
    debugfs_create_file("registers", 0444, my_debugfs_dir,
                        priv, &my_regs_fops);
    debugfs_create_u32("irq_count", 0444, my_debugfs_dir,
                       &priv->irq_count);
    return 0;
}

static void my_remove(struct platform_device *pdev)
{
    debugfs_remove_recursive(my_debugfs_dir);
}
```

```bash
cat /sys/kernel/debug/my-driver/registers
cat /sys/kernel/debug/my-driver/irq_count
```

---

## 25.6 Devcoredump

Capture driver state on error (like a mini core dump).

```c
#include <linux/devcoredump.h>

/* On error: */
void *dump = vmalloc(dump_size);
/* Fill dump with register state, DMA descriptors, firmware state... */
dev_coredumpv(dev, dump, dump_size, GFP_KERNEL);

/* Userspace reads from: /sys/class/devcoredump/devcd*/data */
```

---

## 25.7 Common Debug Workflow

```
Problem: Driver probe fails
    │
    ├── 1. Check dmesg: dmesg | grep my-driver
    ├── 2. Enable dynamic debug: echo 'module my_driver +p' > dyndbg
    ├── 3. Check deferred probes:
    │       cat /sys/kernel/debug/devices_deferred
    ├── 4. Verify DT binding:
    │       ls /sys/firmware/devicetree/base/soc/my-device@1e000000/
    ├── 5. Check driver binding:
    │       ls /sys/bus/platform/drivers/my-driver/
    │       cat /sys/bus/platform/devices/1e000000.my-device/driver
    └── 6. Ftrace the probe:
            echo 'my_probe' > set_ftrace_filter
```

```
Problem: Interrupt not firing
    │
    ├── 1. cat /proc/interrupts | grep my-driver
    ├── 2. Verify IRQ in DT: check interrupts property
    ├── 3. Enable IRQ tracing events
    ├── 4. Check GIC/interrupt controller config
    └── 5. Read hardware interrupt status register via debugfs
```

---

## 25.8 KASAN, UBSAN, KCSAN — Sanitizers

```bash
# Kernel Address Sanitizer (use-after-free, out-of-bounds)
CONFIG_KASAN=y

# Undefined Behavior Sanitizer
CONFIG_UBSAN=y

# Concurrency Sanitizer (data races)
CONFIG_KCSAN=y
```

KASAN output example:
```
BUG: KASAN: use-after-free in my_read+0x1c/0x40
Read of size 4 at addr ffff8880123456c0 by task cat/1234
```

---

## OS Comparison — Debugging Tools

| Feature | Linux | Windows | QNX | macOS |
|---------|-------|---------|-----|-------|
| Kernel printf | printk / dev_* | DbgPrint / KdPrint | slogf | IOLog |
| Dynamic tracing | ftrace / BPF | ETW | tracelogger | DTrace |
| Memory sanitizer | KASAN | Driver Verifier | none | ASAN (user) |
| Crash dump | kdump / devcoredump | minidump | dumper | panic log |
| Debug FS | debugfs | none | none | none |

---

## Interview Questions

**Q1: How do you debug a driver that works on one board but not another?**
A: Check: 1) Device Tree differences (compatible, reg, interrupts, clocks). 2) Clock/reset provider availability (deferred probe?). 3) MMIO base address mapping. 4) Pin muxing differences. 5) Power domain configuration. Use dynamic debug + ftrace on the failing board.

**Q2: What is the difference between `printk()`, `dev_dbg()`, and `trace_printk()`?**
A: `printk()`: writes to kernel log buffer (dmesg), always compiled in at given level. `dev_dbg()`: same but with device prefix, compiled out unless `DEBUG` or `CONFIG_DYNAMIC_DEBUG`. `trace_printk()`: writes to ftrace ring buffer — much faster, for hot paths. Never leave `trace_printk()` in production.

**Q3: How do you use ftrace to find why probe is slow?**
A: Use function_graph tracer: `echo function_graph > current_tracer && echo my_probe > set_graph_function`. This shows each function call within probe and its duration, revealing bottlenecks (slow firmware load, regulator ramp, etc.).

---

*Next: [Chapter 26 — Driver Testing](Chapter_26_Testing.md)*
