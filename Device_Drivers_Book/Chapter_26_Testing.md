# Chapter 26: Driver Testing

## Chapter Overview

Testing kernel drivers is challenging — crashes bring down the entire system, hardware may not be available, and race conditions are hard to reproduce. This chapter covers testing strategies, frameworks, emulation, and stress testing.

---

## 26.1 Testing Strategies Overview

```
┌─────────────────────────────────────────────┐
│               Testing Pyramid               │
│                                             │
│              ┌─────────┐                    │
│              │  System  │  ← Full board/SoC │
│            ┌─┴─────────┴─┐                  │
│            │ Integration  │  ← QEMU + DT    │
│          ┌─┴─────────────┴─┐                │
│          │  KUnit (kernel)  │  ← In-kernel  │
│        ┌─┴─────────────────┴─┐              │
│        │  Static Analysis     │  ← Compile  │
│        └─────────────────────┘              │
└─────────────────────────────────────────────┘
```

---

## 26.2 Static Analysis

### Compile-Time Checks

```bash
# Enable all warnings
make W=1 drivers/my/

# Sparse — check __iomem, __user, endianness annotations
make C=1 drivers/my/driver.o

# Coccinelle — semantic patches
make coccicheck MODE=report M=drivers/my/
```

### Common Sparse Warnings

```
warning: incorrect type in argument 1 (different address spaces)
    expected void *
    got void __iomem *
→ Fix: use readl()/writel() instead of direct pointer dereference

warning: symbol 'my_func' was not declared. Should it be static?
→ Fix: add 'static' keyword
```

### checkpatch.pl — Style Checks

```bash
./scripts/checkpatch.pl --strict -f drivers/my/driver.c

# Common issues caught:
# - Line over 100 characters
# - Missing Signed-off-by
# - Spaces before tabs
# - Open brace on wrong line
```

---

## 26.3 KUnit — In-Kernel Unit Testing

KUnit runs tests inside the kernel, ideal for testing driver logic.

```c
#include <kunit/test.h>

/* Test a helper function */
static void test_parse_config(struct kunit *test)
{
    struct my_config cfg;
    int ret;

    /* Valid input */
    ret = parse_config_data(valid_data, sizeof(valid_data), &cfg);
    KUNIT_EXPECT_EQ(test, ret, 0);
    KUNIT_EXPECT_EQ(test, cfg.width, 1920);
    KUNIT_EXPECT_EQ(test, cfg.height, 1080);

    /* Invalid input */
    ret = parse_config_data(NULL, 0, &cfg);
    KUNIT_EXPECT_EQ(test, ret, -EINVAL);
}

static void test_register_calc(struct kunit *test)
{
    /* Test register value calculation */
    u32 val = calc_divider(48000000, 115200);
    KUNIT_EXPECT_EQ(test, val, 416);
}

static struct kunit_case my_tests[] = {
    KUNIT_CASE(test_parse_config),
    KUNIT_CASE(test_register_calc),
    {},
};

static struct kunit_suite my_test_suite = {
    .name  = "my-driver-tests",
    .test_cases = my_tests,
};
kunit_test_suite(my_test_suite);
```

### Running KUnit

```bash
# Run all KUnit tests
./tools/testing/kunit/kunit.py run

# Run specific suite
./tools/testing/kunit/kunit.py run my-driver-tests

# Run with QEMU (UML by default)
./tools/testing/kunit/kunit.py run --arch=arm64 --cross_compile=aarch64-linux-gnu-
```

---

## 26.4 Testing with QEMU

Emulate hardware for driver testing without physical boards.

```bash
# Boot ARM64 Linux with custom DT
qemu-system-aarch64 \
    -machine virt \
    -cpu cortex-a57 \
    -m 1024 \
    -kernel Image \
    -dtb my-test.dtb \
    -append "console=ttyAMA0 root=/dev/vda" \
    -drive file=rootfs.ext4,format=raw \
    -nographic

# Boot x86_64 with virtio
qemu-system-x86_64 \
    -kernel bzImage \
    -initrd rootfs.cpio.gz \
    -append "console=ttyS0" \
    -nographic \
    -enable-kvm
```

### virtio Devices for Testing

| Virtual Device | Use Case |
|---------------|----------|
| virtio-net | Network driver testing |
| virtio-blk | Block driver testing |
| virtio-gpu | Graphics testing |
| virtio-input | Input device testing |
| vhost-user | Custom device emulation |

---

## 26.5 Hardware Testing Tools

### Device Tree Overlay for Test Scenarios

```dts
/* Test overlay: inject different configuration */
/dts-v1/;
/plugin/;

&my_device {
    status = "okay";
    test-mode = <1>;
    clock-frequency = <100000>;  /* Slow mode for testing */
};
```

### Sysfs-Based Testing

```bash
# Test driver bind/unbind
echo "1e000000.my-device" > /sys/bus/platform/drivers/my-driver/unbind
echo "1e000000.my-device" > /sys/bus/platform/drivers/my-driver/bind

# Test sysfs attributes
echo 1 > /sys/devices/platform/1e000000.my-device/enable
cat /sys/devices/platform/1e000000.my-device/status

# Test power management
echo on > /sys/devices/platform/1e000000.my-device/power/control
echo auto > /sys/devices/platform/1e000000.my-device/power/control
cat /sys/devices/platform/1e000000.my-device/power/runtime_status
```

---

## 26.6 Stress Testing

### Concurrent Access

```bash
# Multiple processes reading device simultaneously
for i in $(seq 1 10); do
    cat /dev/mydev0 > /dev/null &
done
wait

# Rapid bind/unbind cycles
for i in $(seq 1 100); do
    echo "1e000000.my-device" > /sys/bus/platform/drivers/my-driver/unbind
    echo "1e000000.my-device" > /sys/bus/platform/drivers/my-driver/bind
done
```

### Fault Injection

```bash
# Enable memory allocation failures
echo 1 > /sys/kernel/debug/failslab/probability
echo 10 > /sys/kernel/debug/failslab/interval

# Enable I/O errors on block device
echo 1 > /sys/block/sda/make-it-fail
```

### Lock Testing

```bash
# Enable lockdep (compile-time)
CONFIG_PROVE_LOCKING=y
CONFIG_LOCKDEP=y

# Runtime: lockdep reports deadlocks
# BUG: possible circular locking dependency detected
```

---

## 26.7 kselftest — Kernel Self-Tests

```bash
# Run specific subsystem tests
make -C tools/testing/selftests TARGETS=drivers/dma-buf run_tests

# Common driver-related kselftest targets
tools/testing/selftests/
├── drivers/
│   ├── dma-buf/
│   └── s390x/
├── gpio/
├── ipc/
├── net/
└── timers/
```

---

## 26.8 Debugging After Crash

### oops / panic Analysis

```
Unable to handle kernel NULL pointer dereference at virtual address 0000000000000048
pc : my_read+0x1c/0x40 [my_driver]          ← Faulting instruction
lr : vfs_read+0xbc/0x1c0                     ← Caller
...
Call trace:
 my_read+0x1c/0x40 [my_driver]              ← Faulting function
 vfs_read+0xbc/0x1c0
 ksys_read+0x74/0x100
 __arm64_sys_read+0x20/0x30
```

```bash
# Decode: find faulting line
# addr2line or objdump
aarch64-linux-gnu-objdump -dS drivers/my/driver.ko | less
# Search for offset 0x1c in my_read
```

---

## Interview Questions

**Q1: How do you test a driver without physical hardware?**
A: 1) QEMU with virtio or custom device model. 2) KUnit for testing pure logic. 3) Mock hardware registers with software arrays. 4) Device Tree overlays to configure virtual devices. 5) UML (User Mode Linux) for module testing.

**Q2: What is KUnit and when should you use it?**
A: KUnit is the in-kernel unit test framework. Use it for: parsing functions, calculation helpers, state machine logic — any code that can be tested without real hardware. It runs as part of the kernel build, providing fast feedback.

**Q3: How do you test driver power management?**
A: 1) `echo mem > /sys/power/state` for system suspend. 2) `echo auto > /sys/devices/.../power/control` for runtime PM. 3) Verify with `cat .../power/runtime_status`. 4) Stress test: rapid suspend/resume cycles. 5) Check for resource leaks after many cycles.

**Q4: What is fault injection and why is it important?**
A: Kernel fault injection (`failslab`, `fail_make_request`) intentionally fails allocations/IO to test error paths. Most driver bugs hide in error handling that's rarely exercised in normal operation.

---

*Next: [Chapter 27 — Security](Chapter_27_Security.md)*
