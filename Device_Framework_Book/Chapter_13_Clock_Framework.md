# Chapter 13: Clock Framework

## Learning Goals
- Understand the Common Clock Framework (CCF) architecture
- Know clock types: fixed, gate, mux, divider, PLL
- Write clock provider and consumer drivers
- Debug clock trees with debugfs

---

## 13.1 Clock Framework Architecture

```
Common Clock Framework (CCF):

User Space:
  ┌──────────────────────────┐
  │ cat /sys/kernel/debug/   │
  │     clk/clk_summary     │
  └────────────┬─────────────┘
               │
Kernel:        │
  ┌────────────▼──────────────────────────────────┐
  │  Clock Consumers                               │
  │  ├── UART driver: clk_get(), clk_enable()      │
  │  ├── I2C driver: clk_prepare_enable()          │
  │  ├── Display: clk_set_rate()                    │
  │  └── Any peripheral driver                     │
  ├────────────────────────────────────────────────┤
  │  CCF Core (drivers/clk/clk.c)                  │
  │  ├── Clock tree management                     │
  │  ├── Rate propagation (parent → child)         │
  │  ├── Enable/disable reference counting         │
  │  ├── clk_ops dispatch                          │
  │  └── debugfs: /sys/kernel/debug/clk/           │
  ├────────────────────────────────────────────────┤
  │  Clock Providers (SoC-specific)                │
  │  ├── PLL drivers (lock, rate calculation)      │
  │  ├── Mux (clock source selection)              │
  │  ├── Divider (frequency division)              │
  │  ├── Gate (enable/disable)                     │
  │  └── Fixed-rate / Fixed-factor                 │
  └────────────────────────────────────────────────┘
               │
Hardware:      ▼
  ┌──────────────────────────────────────────────┐
  │  Crystal    PLL0    PLL1                      │
  │  24 MHz ──► 800MHz  1200MHz                   │
  │              │        │                       │
  │              MUX ─────┘                       │
  │              │                                │
  │              DIV (/4)                          │
  │              │                                │
  │              GATE ──► UART_CLK (200 MHz)      │
  └──────────────────────────────────────────────┘
```

---

## 13.2 Clock Types

```
Clock Tree Example (Simplified SoC):

                   ┌─────────┐
                   │ Crystal  │ 24 MHz (fixed-rate)
                   │ Osc      │
                   └─────┬───┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         ┌────────┐ ┌────────┐ ┌────────┐
         │ PLL0   │ │ PLL1   │ │ PLL2   │
         │ 800MHz │ │1200MHz │ │ 48MHz  │
         └────┬───┘ └────┬───┘ └────┬───┘
              │          │          │
              ▼          ▼          │
         ┌────────────────────┐    │
         │    MUX (select)    │    │
         │ PLL0 │ PLL1 │ PLL2│    │
         └────────┬───────────┘    │
                  │                │
                  ▼                │
           ┌──────────┐           │
           │ DIV (/4) │           │
           │ 200 MHz  │           │
           └─────┬────┘           │
                 │                │
           ┌─────▼────┐     ┌────▼─────┐
           │  GATE    │     │  GATE    │
           │(on/off)  │     │(on/off)  │
           └─────┬────┘     └────┬─────┘
                 │               │
                 ▼               ▼
            UART_CLK         USB_CLK
            200 MHz          48 MHz

Clock Types:
  fixed-rate   → Constant frequency (crystal oscillator)
  fixed-factor → Fixed multiplier/divider of parent
  gate         → Enable/disable only (no rate change)
  mux          → Select one parent from multiple sources
  divider      → Divide parent rate by configurable ratio
  pll          → Phase-Locked Loop (frequency synthesis)
  composite    → Combination (mux + div + gate)
```

---

## 13.3 Clock Consumer API

```c
/* Clock consumer — typical peripheral driver */
#include <linux/clk.h>

static int my_uart_probe(struct platform_device *pdev)
{
    struct clk *clk;

    /* Get clock from device tree */
    clk = devm_clk_get(&pdev->dev, "uart_clk");
    if (IS_ERR(clk))
        return PTR_ERR(clk);

    /* Prepare and enable (combined) */
    ret = clk_prepare_enable(clk);
    if (ret)
        return ret;

    /* Get current clock rate */
    unsigned long rate = clk_get_rate(clk);
    dev_info(&pdev->dev, "UART clock: %lu Hz\n", rate);

    /* Set desired clock rate */
    ret = clk_set_rate(clk, 115200 * 16);  /* 1.8432 MHz */

    /* Set parent (mux selection) */
    struct clk *pll = devm_clk_get(&pdev->dev, "pll_clk");
    clk_set_parent(clk, pll);

    /* On remove: */
    clk_disable_unprepare(clk);  /* or devm handles it */

    return 0;
}

/*
 * prepare/unprepare: May sleep. Enables clock hardware path.
 * enable/disable: Atomic. Actually gates the clock signal.
 *
 * Typical flow:
 *   clk_prepare()  → power up PLL, wait for lock (can sleep)
 *   clk_enable()   → open gate (atomic, no sleep)
 *   ...use peripheral...
 *   clk_disable()  → close gate
 *   clk_unprepare()→ power down if no other users
 *
 * clk_prepare_enable() = both in one call (convenience)
 */
```

---

## 13.4 Clock Provider — Registration

```c
/* Clock provider — registering clocks */
#include <linux/clk-provider.h>

/* Fixed-rate clock */
struct clk_hw *hw;
hw = clk_hw_register_fixed_rate(&pdev->dev, "osc24m", NULL,
                                 0, 24000000);

/* Gate clock */
hw = clk_hw_register_gate(&pdev->dev, "uart_gate",
                           "uart_div",  /* parent name */
                           CLK_SET_RATE_PARENT,
                           reg_base + CLK_GATE_REG,
                           bit_offset,
                           0,           /* flags */
                           &lock);

/* Divider clock */
hw = clk_hw_register_divider(&pdev->dev, "uart_div",
                              "pll0",    /* parent */
                              CLK_SET_RATE_PARENT,
                              reg_base + CLK_DIV_REG,
                              shift, width,
                              CLK_DIVIDER_ONE_BASED,
                              &lock);

/* Mux clock */
static const char *const mux_parents[] = { "pll0", "pll1", "osc24m" };
hw = clk_hw_register_mux(&pdev->dev, "uart_mux",
                          mux_parents, ARRAY_SIZE(mux_parents),
                          CLK_SET_RATE_PARENT,
                          reg_base + CLK_MUX_REG,
                          shift, width,
                          0, &lock);
```

---

## 13.5 Device Tree Clock Bindings

```dts
/* Clock provider */
clk: clock-controller@10000 {
    compatible = "vendor,soc-clk";
    reg = <0x10000 0x1000>;
    #clock-cells = <1>;  /* one cell = clock ID */
};

/* Clock consumer */
uart0: serial@40010000 {
    compatible = "vendor,uart";
    reg = <0x40010000 0x100>;
    clocks = <&clk CLK_UART0>, <&clk CLK_UART0_PARENT>;
    clock-names = "uart_clk", "pll_clk";
};

/* Multiple clocks */
display: display@50000000 {
    compatible = "vendor,display";
    clocks = <&clk CLK_PIXEL>, <&clk CLK_AHB>;
    clock-names = "pixel", "ahb";
};
```

---

## 13.6 Debug

```bash
# Clock tree dump
$ cat /sys/kernel/debug/clk/clk_summary
                                 enable  prepare  protect
   clock                          count    count    count    rate
--------------------------------------------------------------------
 osc24m                               1        1        0    24000000
    pll0                              1        1        0   800000000
       uart_mux                       1        1        0   800000000
          uart_div                    1        1        0   200000000
             uart_gate                1        1        0   200000000
    pll1                              0        0        0  1200000000

# Check specific clock
$ cat /sys/kernel/debug/clk/uart_gate/clk_rate
$ cat /sys/kernel/debug/clk/uart_gate/clk_enable_count
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| CCF core | drivers/clk/clk.c | Clock tree management |
| clk.h | include/linux/clk.h | Consumer API |
| clk-provider.h | include/linux/clk-provider.h | Provider API |
| Gate | drivers/clk/clk-gate.c | Gate clock type |
| Divider | drivers/clk/clk-divider.c | Divider clock type |
| Mux | drivers/clk/clk-mux.c | Mux clock type |
| SoC clocks | drivers/clk/`<vendor>`/ | SoC clock drivers |

---

## Interview Questions

**Q1: Explain clk_prepare vs clk_enable.**
A: `clk_prepare()` performs potentially-sleeping operations to make a clock ready — e.g., enabling a PLL and waiting for it to lock, or powering a clock domain. It can be called from process context only. `clk_enable()` performs the final atomic step to start the clock signal — typically opening a gate register. It can be called from atomic context (IRQ handlers). This two-phase design allows drivers in IRQ context to call `clk_enable()` if `clk_prepare()` was done earlier. Both maintain reference counts — a clock is only actually disabled when all users have called disable/unprepare.

**Q2: How does rate propagation work in the clock tree?**
A: When a consumer calls `clk_set_rate(clk, rate)`, the CCF walks up the tree to find a clock that can change rate (has `set_rate` op and `CLK_SET_RATE_PARENT` flag on children). It then calculates what parent rate produces the closest possible output rate considering dividers and muxes. The `determine_rate()` or `round_rate()` callback of each clock is used to find the best achievable rate. After determining the rate, changes propagate down: parent rate is set first, then children adjust their dividers. Notifications are sent so dependents can reconfigure.

**Q3: What are the main clock types in the CCF?**
A: (1) **fixed-rate**: Constant frequency (crystal oscillator, 24MHz). (2) **fixed-factor**: Fixed multiplier/divider of parent (×2, /3). (3) **gate**: Only enable/disable, no rate change. (4) **mux**: Selects one parent from multiple sources. (5) **divider**: Programmable divider with configurable ratio. (6) **PLL**: Frequency synthesizer generating high frequencies from a reference. (7) **composite**: Combines mux + divider + gate in one structure (many SoC clocks are composite).

---

## Summary

- CCF manages hierarchical clock trees with reference-counted enable/disable
- Clock types: fixed-rate, gate, mux, divider, PLL, composite
- Two-phase enable: `clk_prepare()` (sleeping) + `clk_enable()` (atomic)
- Consumers: `devm_clk_get()` → `clk_prepare_enable()` → `clk_get_rate()`
- Rate changes propagate through the tree using parent/child relationships
- debugfs `clk_summary` shows the entire clock tree with rates and enable counts

---

*Next: [Chapter 14 — Regulator Framework](Chapter_14_Regulator_Framework.md)*
