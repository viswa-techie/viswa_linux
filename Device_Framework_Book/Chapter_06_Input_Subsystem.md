# Chapter 6: Input Subsystem

## Learning Goals
- Understand Linux input subsystem architecture
- Know event types (EV_KEY, EV_ABS, EV_REL) and evdev interface
- Write or analyze an input device driver
- Understand multitouch protocol and input event reporting

---

## 6.1 Input Subsystem Architecture

```
Input Subsystem Architecture:

User Space:
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ libinput │  │ evtest   │  │ Android  │
  │ (Wayland)│  │ (debug)  │  │ InputFlgr│
  └─────┬────┘  └─────┬────┘  └─────┬────┘
        │              │              │
        └──────────────┼──────────────┘
                       │  /dev/input/eventN
Kernel:                │
  ┌────────────────────▼─────────────────────────────┐
  │              Input Core                           │
  │                                                   │
  │  ┌──────────────────────────────────────────┐    │
  │  │  Event Handler Layer                      │    │
  │  │  ├── evdev (generic events → /dev/input/) │    │
  │  │  ├── joydev (joystick → /dev/input/js*)   │    │
  │  │  └── mousedev (legacy mouse emulation)    │    │
  │  └──────────────────┬───────────────────────┘    │
  │                     │                             │
  │  ┌──────────────────▼───────────────────────┐    │
  │  │  Input Core (drivers/input/input.c)       │    │
  │  │  ├── input_register_device()              │    │
  │  │  ├── input_event() / input_report_key()   │    │
  │  │  └── Matching: device ↔ handler           │    │
  │  └──────────────────┬───────────────────────┘    │
  │                     │                             │
  │  ┌──────────────────▼───────────────────────┐    │
  │  │  Input Drivers                            │    │
  │  │  ├── Keyboard (gpio-keys, matrix-keypad)  │    │
  │  │  ├── Touchscreen (goodix, atmel_mxt)      │    │
  │  │  ├── Touchpad (elan, synaptics)           │    │
  │  │  ├── Accelerometer/Gyro (as input)        │    │
  │  │  └── Rotary encoder, buttons, etc.        │    │
  │  └──────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────┘
```

---

## 6.2 Event Types

```
Input Event Structure:

struct input_event {
    struct timeval time;   /* timestamp */
    __u16 type;            /* event type */
    __u16 code;            /* event code */
    __s32 value;           /* event value */
};

Event Types:
┌──────────┬────────────────────────────────────────────┐
│ Type     │ Description                                │
├──────────┼────────────────────────────────────────────┤
│ EV_KEY   │ Key press/release (keyboard, buttons)      │
│          │ code=KEY_A, value=1(press)/0(release)       │
├──────────┼────────────────────────────────────────────┤
│ EV_REL   │ Relative axis (mouse movement)             │
│          │ code=REL_X, value=delta_x                   │
├──────────┼────────────────────────────────────────────┤
│ EV_ABS   │ Absolute axis (touchscreen)                │
│          │ code=ABS_X, value=x_coordinate              │
├──────────┼────────────────────────────────────────────┤
│ EV_SW    │ Switch (lid close, headphone jack)         │
│          │ code=SW_LID, value=1(closed)/0(open)        │
├──────────┼────────────────────────────────────────────┤
│ EV_SYN   │ Synchronization (ends a set of events)    │
│          │ code=SYN_REPORT (marks complete packet)     │
└──────────┴────────────────────────────────────────────┘

Example touch event sequence:
  EV_ABS  ABS_X          512
  EV_ABS  ABS_Y          384
  EV_ABS  ABS_PRESSURE   200
  EV_KEY  BTN_TOUCH      1        (finger down)
  EV_SYN  SYN_REPORT     0        (packet complete)
```

---

## 6.3 Writing an Input Driver

```c
/* Simple GPIO button input driver */
#include <linux/input.h>
#include <linux/gpio/consumer.h>
#include <linux/interrupt.h>

struct my_button {
    struct input_dev *input;
    struct gpio_desc *gpio;
    int irq;
};

static irqreturn_t button_isr(int irq, void *data)
{
    struct my_button *btn = data;
    int pressed = gpiod_get_value_cansleep(btn->gpio);

    input_report_key(btn->input, KEY_POWER, pressed);
    input_sync(btn->input);  /* sends EV_SYN SYN_REPORT */

    return IRQ_HANDLED;
}

static int my_button_probe(struct platform_device *pdev)
{
    struct my_button *btn;
    int ret;

    btn = devm_kzalloc(&pdev->dev, sizeof(*btn), GFP_KERNEL);

    /* Get GPIO from device tree */
    btn->gpio = devm_gpiod_get(&pdev->dev, "button", GPIOD_IN);

    /* Allocate input device */
    btn->input = devm_input_allocate_device(&pdev->dev);
    btn->input->name = "My Power Button";
    btn->input->phys = "my-button/input0";

    /* Declare capabilities */
    input_set_capability(btn->input, EV_KEY, KEY_POWER);

    /* Register input device — creates /dev/input/eventN */
    ret = input_register_device(btn->input);

    /* Setup interrupt */
    btn->irq = gpiod_to_irq(btn->gpio);
    ret = devm_request_irq(&pdev->dev, btn->irq, button_isr,
                           IRQF_TRIGGER_BOTH, "my-button", btn);
    return ret;
}
```

---

## 6.4 Touchscreen Driver (Multitouch)

```c
/* Multitouch touchscreen driver (Type B protocol) */

static int touchscreen_probe(struct i2c_client *client)
{
    struct input_dev *input;

    input = devm_input_allocate_device(&client->dev);
    input->name = "My Touchscreen";

    /* Declare multitouch capabilities */
    input_set_abs_params(input, ABS_MT_POSITION_X, 0, 1920, 0, 0);
    input_set_abs_params(input, ABS_MT_POSITION_Y, 0, 1080, 0, 0);
    input_set_abs_params(input, ABS_MT_PRESSURE, 0, 255, 0, 0);

    /* Type B: slot-based tracking */
    ret = input_mt_init_slots(input, MAX_CONTACTS,
                              INPUT_MT_DIRECT);

    input_register_device(input);
    /* ... setup I2C interrupt handler ... */
}

/* ISR — report multitouch events (Type B protocol) */
static irqreturn_t touch_isr(int irq, void *data)
{
    struct touch_data contacts[MAX_CONTACTS];
    int count;

    /* Read touch data from controller via I2C */
    count = read_touch_data(client, contacts);

    for (int i = 0; i < count; i++) {
        input_mt_slot(input, contacts[i].id);
        input_mt_report_slot_state(input, MT_TOOL_FINGER, true);
        input_report_abs(input, ABS_MT_POSITION_X, contacts[i].x);
        input_report_abs(input, ABS_MT_POSITION_Y, contacts[i].y);
        input_report_abs(input, ABS_MT_PRESSURE, contacts[i].pressure);
    }

    /* Report lifted fingers */
    input_mt_sync_frame(input);
    input_sync(input);

    return IRQ_HANDLED;
}
```

```
Multitouch Protocol Type A vs Type B:

Type A (anonymous contacts):
  For each contact: ABS_MT_POSITION_X, ABS_MT_POSITION_Y
  Then: SYN_MT_REPORT (end of contact)
  After all contacts: SYN_REPORT
  Problem: No tracking between frames

Type B (slot-based tracking):
  input_mt_slot(input, slot_id)      ← select slot
  input_mt_report_slot_state(...)    ← finger up/down
  ABS_MT_POSITION_X, Y, PRESSURE    ← per-slot data
  input_sync()                       ← end of frame
  Advantage: Kernel tracks each finger across frames
  Used by: Modern touchscreens, Android requires Type B
```

---

## 6.5 Device Tree Bindings

```dts
/* Input device in device tree */

gpio-keys {
    compatible = "gpio-keys";

    power-button {
        label = "Power Button";
        linux,code = <KEY_POWER>;
        gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
        wakeup-source;
    };

    volume-up {
        label = "Volume Up";
        linux,code = <KEY_VOLUMEUP>;
        gpios = <&gpio1 6 GPIO_ACTIVE_LOW>;
    };
};

/* I2C touchscreen */
&i2c3 {
    touchscreen@5d {
        compatible = "goodix,gt911";
        reg = <0x5d>;
        interrupt-parent = <&gpio1>;
        interrupts = <9 IRQ_TYPE_LEVEL_LOW>;
        reset-gpios = <&gpio1 10 GPIO_ACTIVE_LOW>;
        irq-gpios = <&gpio1 9 GPIO_ACTIVE_HIGH>;
        touchscreen-size-x = <1920>;
        touchscreen-size-y = <1080>;
    };
};
```

---

## 6.6 User-Space Debug

```bash
# List input devices
$ cat /proc/bus/input/devices
$ ls -la /dev/input/

# Monitor events live
$ evtest /dev/input/event0

# Get device info
$ evtest --query /dev/input/event0 EV_KEY KEY_POWER

# Android
$ getevent -l              # list devices
$ getevent -lt /dev/input/event2   # monitor with timestamps

# Check input device capabilities
$ cat /sys/class/input/input0/capabilities/ev
$ cat /sys/class/input/input0/capabilities/key
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| Input core | drivers/input/input.c | Event registration/dispatch |
| evdev | drivers/input/evdev.c | /dev/input/eventN handler |
| gpio-keys | drivers/input/keyboard/gpio_keys.c | GPIO button driver |
| Multitouch | drivers/input/input-mt.c | MT protocol helpers |
| Touchscreen | drivers/input/touchscreen/ | Touch controller drivers |
| input.h | include/uapi/linux/input-event-codes.h | Event type/code defines |

---

## Interview Questions

**Q1: Explain the Linux input event model.**
A: Input devices report events as (type, code, value) tuples via the evdev interface. The driver calls `input_report_key()`, `input_report_abs()`, etc. to queue events, then `input_sync()` to send `EV_SYN SYN_REPORT` marking a complete packet. User space reads these from `/dev/input/eventN` as `struct input_event`. Event types include EV_KEY (buttons/keys), EV_REL (relative axes like mouse), EV_ABS (absolute axes like touchscreen), and EV_SW (switches like lid). The SYN_REPORT boundary tells user space that all events in one hardware sample are complete.

**Q2: What is the difference between multitouch Type A and Type B protocols?**
A: Type A reports anonymous contacts — each contact sends ABS_MT_POSITION_X/Y followed by SYN_MT_REPORT, with no tracking ID. User space must track fingers across frames. Type B uses slots: `input_mt_slot(input, id)` selects a numbered slot, then per-slot data is reported. The kernel tracks each finger by slot ID across frames, simplifying user-space processing. Type B also supports partial updates — only changed slots need to be reported. Android requires Type B.

**Q3: How does a GPIO button driver work?**
A: (1) In probe, allocate `input_dev` with `devm_input_allocate_device()`. (2) Set capabilities with `input_set_capability(input, EV_KEY, KEY_POWER)`. (3) Register with `input_register_device()` — creates `/dev/input/eventN`. (4) Request GPIO interrupt on both edges. (5) In ISR, read GPIO value with `gpiod_get_value()`, call `input_report_key(input, KEY_POWER, value)`, then `input_sync()`. The input core dispatches the event to all connected handlers (evdev), making it available to user space.

---

## Summary

- The input subsystem handles keyboards, touchscreens, buttons, mice, and switches
- Events are (type, code, value) tuples: EV_KEY, EV_ABS, EV_REL, EV_SYN
- `input_sync()` marks the end of a complete event packet (SYN_REPORT)
- Multitouch Type B (slot-based) is standard for modern touchscreens
- evdev handler creates `/dev/input/eventN` for user-space access
- Drivers declare capabilities at registration so handlers know what events to expect

---

*Next: [Chapter 7 — Network Device Framework](Chapter_07_Network_Device.md)*
