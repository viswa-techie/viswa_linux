# Chapter 13: Clock Framework

## Learning Goals
- Understand the Common Clock Framework (CCF) architecture
- Learn clock tree topology and clock types
- Know clock gating for power management
- Understand clock provider and consumer APIs
- Learn clock configuration in device tree

---

## 1. Clock Framework Architecture

```
  Common Clock Framework (CCF)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌── Consumer API ────────────────────────────────────┐  │
  │  │  clk_get() / devm_clk_get()                        │  │
  │  │  clk_prepare_enable() / clk_disable_unprepare()    │  │
  │  │  clk_set_rate() / clk_get_rate()                    │  │
  │  │  clk_set_parent()                                   │  │
  │  └────────────────────────┬───────────────────────────┘  │
  │                           │                               │
  │  ┌────────────────────────▼───────────────────────────┐  │
  │  │              CCF Core (drivers/clk/clk.c)           │  │
  │  │                                                      │  │
  │  │  struct clk_hw {                                     │  │
  │  │      struct clk_core *core;                          │  │
  │  │      const struct clk_ops *ops;                      │  │
  │  │  };                                                  │  │
  │  │                                                      │  │
  │  │  Manages: clock tree, reference counting,            │  │
  │  │           rate propagation, parent selection         │  │
  │  └────────────────────────┬───────────────────────────┘  │
  │                           │                               │
  │  ┌────────────────────────▼───────────────────────────┐  │
  │  │              Provider (SoC-specific clock drivers)   │  │
  │  │                                                      │  │
  │  │  drivers/clk/clk-<soc>.c                            │  │
  │  │  Implements clk_ops for each clock type:            │  │
  │  │  - Fixed rate, PLL, divider, mux, gate              │  │
  │  └──────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Clock Types

```
  Clock Tree Building Blocks
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Fixed Rate (crystal oscillator):                        │
  │  ┌─────────┐                                            │
  │  │  24 MHz  │──► Output: always 24 MHz                  │
  │  └─────────┘                                            │
  │                                                           │
  │  PLL (Phase-Locked Loop):                                │
  │  ┌─────────┐    ┌─────┐                                 │
  │  │  24 MHz  │──►│ PLL │──► Output: N × 24 MHz           │
  │  └─────────┘    │M/N/P│    (e.g., 1200 MHz)             │
  │                 └─────┘                                  │
  │                                                           │
  │  Divider:                                                │
  │  ┌──────────┐    ┌─────┐                                │
  │  │ 1200 MHz │──►│ ÷N  │──► Output: 1200/N MHz          │
  │  └──────────┘    └─────┘    (e.g., 600 MHz for N=2)     │
  │                                                           │
  │  Multiplexer (Mux):                                      │
  │  ┌──────────┐                                           │
  │  │ Clock A  │──►┐                                       │
  │  └──────────┘   │  ┌─────┐                              │
  │  ┌──────────┐   ├─►│ MUX │──► Selected clock output    │
  │  │ Clock B  │──►┘  │     │                              │
  │  └──────────┘      └─────┘                              │
  │                                                           │
  │  Gate:                                                   │
  │  ┌──────────┐    ┌──────┐                               │
  │  │ Clock in │──►│ GATE │──► Output: clock or 0          │
  │  └──────────┘    │ EN   │    (gated = saves power)      │
  │                  └──────┘                                │
  │                                                           │
  │  Fixed Factor (hardwired divider):                       │
  │  ┌──────────┐    ┌──────┐                               │
  │  │ 1200 MHz │──►│ ÷3   │──► 400 MHz (fixed ratio)      │
  │  └──────────┘    └──────┘                                │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Example Clock Tree

```
  Typical SoC Clock Tree
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  XTAL (24 MHz)                                           │
  │  │                                                        │
  │  ├──► PLL_CPU ──► MUX_CPU ──► DIV_CPU ──► CPU Cores     │
  │  │    (2400 MHz)   │          (÷1..÷4)                   │
  │  │                 │                                      │
  │  │                 └── bypass (24 MHz, for boot)         │
  │  │                                                        │
  │  ├──► PLL_SYS ──► DIV_AXI ──► GATE_AXI ──► AXI Bus     │
  │  │    (1200 MHz)   (÷2)       (can gate)    (600 MHz)   │
  │  │              │                                         │
  │  │              └► DIV_AHB ──► GATE_AHB ──► AHB Bus     │
  │  │                 (÷4)                     (300 MHz)    │
  │  │                          │                             │
  │  │                          └► DIV_APB ──► APB Bus       │
  │  │                             (÷2)        (150 MHz)     │
  │  │                                                        │
  │  ├──► PLL_GPU ──► DIV_GPU ──► GATE_GPU ──► GPU          │
  │  │    (800 MHz)                                          │
  │  │                                                        │
  │  ├──► PLL_DDR ──► DDR Controller                         │
  │  │    (1600 MHz)                                         │
  │  │                                                        │
  │  └──► PLL_PERI ──► DIV_UART ──► GATE_UART0 ──► UART0   │
  │       (960 MHz)    (÷N)         (can gate)               │
  │                 │                                         │
  │                 ├► DIV_SPI ──► GATE_SPI0 ──► SPI0        │
  │                 │                                         │
  │                 └► DIV_I2C ──► GATE_I2C0 ──► I2C0        │
  │                                                           │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Consumer API

```c
/* Clock consumer API — used by device drivers */
#include <linux/clk.h>

static int my_driver_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct clk *clk;

    /* Get clock by name (from device tree "clock-names") */
    clk = devm_clk_get(dev, "core");
    if (IS_ERR(clk))
        return PTR_ERR(clk);

    /* Prepare and enable clock */
    /* prepare: may sleep (PLL lock, regulator enable) */
    /* enable: must not sleep (just flips gate bit) */
    ret = clk_prepare_enable(clk);
    if (ret)
        return ret;

    /* Get current rate */
    unsigned long rate = clk_get_rate(clk);
    dev_info(dev, "Clock rate: %lu Hz\n", rate);

    /* Set a new rate */
    ret = clk_set_rate(clk, 100000000); /* 100 MHz */

    /* Set minimum rate */
    ret = clk_set_min_rate(clk, 50000000); /* 50 MHz minimum */

    /* On driver removal or suspend */
    clk_disable_unprepare(clk);

    return 0;
}
```

### 4.1 Prepare/Enable Split

```
  Why prepare() and enable() are separate:
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  clk_prepare():                                          │
  │  - May sleep (uses mutex)                                │
  │  - Enables parent PLL if needed                          │
  │  - Waits for PLL lock                                    │
  │  - Enables regulator for clock source                    │
  │  - Called from process context only                      │
  │                                                           │
  │  clk_enable():                                           │
  │  - Must NOT sleep (uses spinlock)                        │
  │  - Just flips the gate bit in a register                 │
  │  - Can be called from interrupt context                  │
  │  - Very fast operation                                    │
  │                                                           │
  │  Convenience: clk_prepare_enable() does both             │
  │  Reverse:     clk_disable_unprepare()                    │
  │                                                           │
  │  Use case for separate calls:                            │
  │  - Prepare clock once at probe                           │
  │  - Enable/disable rapidly in IRQ handler                 │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Clock Provider Implementation

```c
/* Simple gate clock provider */
#include <linux/clk-provider.h>

static int my_clk_gate_enable(struct clk_hw *hw)
{
    struct my_clk *clk = to_my_clk(hw);
    u32 reg = readl(clk->base + CLK_GATE_REG);
    reg |= BIT(clk->bit);
    writel(reg, clk->base + CLK_GATE_REG);
    return 0;
}

static void my_clk_gate_disable(struct clk_hw *hw)
{
    struct my_clk *clk = to_my_clk(hw);
    u32 reg = readl(clk->base + CLK_GATE_REG);
    reg &= ~BIT(clk->bit);
    writel(reg, clk->base + CLK_GATE_REG);
}

static int my_clk_gate_is_enabled(struct clk_hw *hw)
{
    struct my_clk *clk = to_my_clk(hw);
    return !!(readl(clk->base + CLK_GATE_REG) & BIT(clk->bit));
}

static const struct clk_ops my_gate_ops = {
    .enable     = my_clk_gate_enable,
    .disable    = my_clk_gate_disable,
    .is_enabled = my_clk_gate_is_enabled,
};

/* Registration */
static int my_clk_probe(struct platform_device *pdev)
{
    struct clk_init_data init = {
        .name = "uart0_clk",
        .ops = &my_gate_ops,
        .parent_names = (const char *[]){ "apb_clk" },
        .num_parents = 1,
        .flags = CLK_SET_RATE_PARENT, /* propagate rate to parent */
    };

    my_clk->hw.init = &init;
    ret = devm_clk_hw_register(&pdev->dev, &my_clk->hw);

    /* Register as OF provider */
    of_clk_add_hw_provider(pdev->dev.of_node, of_clk_hw_onecell_get, data);

    return ret;
}
```

---

## 6. Clock Gating for Power Management

```
  Clock Gating Power Savings
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Before clock gating:                                    │
  │  Time ──────────────────────────────────────────►       │
  │  CLK:  ┃╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱┃         │
  │  Power: ████████████████████████████████████████         │
  │  (Clock always running, even when device idle)           │
  │                                                           │
  │  After clock gating:                                     │
  │  Time ──────────────────────────────────────────►       │
  │  CLK:  ┃╲╱╲╱┃────────────────┃╲╱╲╱╲╱┃─────┃╲╱┃        │
  │  Power: █████░░░░░░░░░░░░░░░░█████████░░░░░████         │
  │  (Clock gated when device not in use)                    │
  │                                                           │
  │  Savings: Eliminates dynamic power (αCV²f → 0)          │
  │  Limitation: Leakage power still flows (transistors on)  │
  │                                                           │
  │  Runtime PM + CCF integration:                           │
  │  runtime_suspend: clk_disable_unprepare(clk)            │
  │  runtime_resume:  clk_prepare_enable(clk)               │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. Device Tree Clock Bindings

```dts
/* Clock provider */
clock-controller@10000 {
    compatible = "my-soc,clock-controller";
    reg = <0x10000 0x1000>;
    #clock-cells = <1>;  /* 1 = index-based lookup */

    /* The driver maps indices to clk_hw structures */
};

/* Consumer device */
uart0: serial@20000 {
    compatible = "my-soc,uart";
    reg = <0x20000 0x100>;

    /* Reference to clock provider + index */
    clocks = <&clock_controller 42>;  /* clock index 42 */
    clock-names = "baudclk";          /* driver uses this name */
};

/* Multiple clocks */
gpu: gpu@30000 {
    compatible = "my-soc,gpu";
    reg = <0x30000 0x10000>;

    clocks = <&clock_controller 10>,   /* core clock */
             <&clock_controller 11>,   /* bus clock */
             <&clock_controller 12>;   /* shader clock */
    clock-names = "core", "bus", "shader";
};
```

---

## 8. Clock Debugging

```bash
# View clock tree (debugfs)
cat /sys/kernel/debug/clk/clk_summary

# Example output:
#                               enable  prepare  protect
# clock                   count   count    count    rate
# ─────────────────────────────────────────────────────
# xtal_24m                    1       1        0  24000000
#    pll_cpu                  1       1        0  2400000000
#       cpu_clk               1       1        0  2400000000
#    pll_sys                  1       1        0  1200000000
#       axi_clk               1       1        0  600000000
#       ahb_clk               1       1        0  300000000
#          apb_clk            1       1        0  150000000
#             uart0_clk       0       0        0  150000000  <-- gated!
#             uart1_clk       1       1        0  150000000
#             i2c0_clk        0       0        0  150000000  <-- gated!

# Orphan clocks (no parent found)
cat /sys/kernel/debug/clk/clk_orphan_summary

# Per-clock details
cat /sys/kernel/debug/clk/uart0_clk/clk_rate
cat /sys/kernel/debug/clk/uart0_clk/clk_enable_count
cat /sys/kernel/debug/clk/uart0_clk/clk_prepare_count
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `drivers/clk/clk.c` | CCF core implementation |
| `drivers/clk/clk-gate.c` | Generic gate clock |
| `drivers/clk/clk-divider.c` | Generic divider clock |
| `drivers/clk/clk-mux.c` | Generic mux clock |
| `drivers/clk/clk-fixed-rate.c` | Fixed rate clock |
| `drivers/clk/clk-fixed-factor.c` | Fixed factor clock |
| `include/linux/clk.h` | Consumer API |
| `include/linux/clk-provider.h` | Provider API |
| `drivers/clk/<vendor>/` | SoC-specific clock drivers |

---

## Interview Questions

**Q1: Why are clk_prepare() and clk_enable() separate?**
**A:** clk_prepare() may sleep (e.g., to lock a PLL or enable a regulator) and uses a mutex. clk_enable() must not sleep (just toggles a gate bit) and uses a spinlock. This split allows: (1) preparing clocks from process context at probe time, then rapidly enabling/disabling from interrupt context, and (2) the framework to properly handle clock hierarchy where enabling a child may need to prepare a parent PLL.

**Q2: How does clock gating save power and what are its limitations?**
**A:** Clock gating stops the clock signal to a hardware block, eliminating dynamic power (αCV²f → 0 because f=0). The gate is a simple AND gate controlled by an enable bit. Limitation: it only saves dynamic power — leakage (static) power still flows because transistors remain powered. For complete power elimination, power gating (cutting VDD) is needed, which is handled by the power domain framework, not CCF.

**Q3: Explain CLK_SET_RATE_PARENT flag.**
**A:** When a consumer calls clk_set_rate() on a clock with CLK_SET_RATE_PARENT, the framework propagates the rate change request up to the parent. This is used when the clock itself can't change rate (e.g., a gate or fixed-factor divider) but the parent PLL/divider can. Without this flag, the framework only tries to change the rate at the requested clock level.

---

## Summary

- CCF provides a unified clock tree management framework for all SoCs
- Clock types: fixed rate, PLL, divider, mux, gate, fixed factor
- Consumer API: clk_get, clk_prepare_enable, clk_set_rate, clk_disable_unprepare
- prepare/enable split allows sleeping operations (PLL lock) separate from fast gate toggling
- Clock gating eliminates dynamic power; used with runtime PM for per-device clock management
- Device tree binds clocks to devices via `clocks` and `clock-names` properties
- Debug via `/sys/kernel/debug/clk/clk_summary` for full clock tree visualization

---

[Previous Chapter: Power Domains ←](Chapter_12_Power_Domains.md) | [Next Chapter: Regulator Framework →](Chapter_14_Regulator_Framework.md)
