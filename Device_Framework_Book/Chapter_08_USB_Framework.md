# Chapter 8: USB Framework

## Learning Goals
- Understand Linux USB subsystem architecture
- Know USB host controller, hub, device, and gadget models
- Grasp USB transfer types, URBs, and endpoint management
- Write or analyze a USB device/gadget driver

---

## 8.1 USB Subsystem Architecture

```
USB Subsystem Architecture:

User Space:
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ libusb   │  │ lsusb    │  │ usb_mode │
  │          │  │          │  │ switch   │
  └─────┬────┘  └─────┬────┘  └─────┬────┘
        │              │              │
        └──────────────┼──────────────┘
                       │  /dev/bus/usb/*
Kernel:                │
  ┌────────────────────▼─────────────────────────────┐
  │              USB Core (drivers/usb/core/)         │
  │                                                   │
  │  ┌──────────────────────────────────────────┐    │
  │  │  USB Device Drivers                       │    │
  │  │  ├── usb-storage (mass storage)           │    │
  │  │  ├── cdc-acm (serial/modem)               │    │
  │  │  ├── uvc (USB video class)                │    │
  │  │  ├── usbhid (HID devices)                 │    │
  │  │  └── Custom class/vendor drivers          │    │
  │  └──────────────────────────────────────────┘    │
  │                                                   │
  │  ┌──────────────────────────────────────────┐    │
  │  │  USB Core (bus management)                │    │
  │  │  ├── Device enumeration (SET_ADDRESS)     │    │
  │  │  ├── Configuration/Interface selection     │    │
  │  │  ├── URB (USB Request Block) submission    │    │
  │  │  ├── Hub driver (port management)          │    │
  │  │  └── Power management (suspend/resume)     │    │
  │  └──────────────────────────────────────────┘    │
  │                                                   │
  │  ┌──────────────────────────────────────────┐    │
  │  │  Host Controller Drivers (HCD)            │    │
  │  │  ├── xhci-hcd (USB 3.x)                  │    │
  │  │  ├── ehci-hcd (USB 2.0)                  │    │
  │  │  ├── ohci-hcd / uhci-hcd (USB 1.x)       │    │
  │  │  └── dwc3 / dwc2 (DesignWare controller) │    │
  │  └──────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────┐
  │  USB Gadget Framework (device-side)              │
  │  ├── gadget driver (function)                    │
  │  ├── composite framework                         │
  │  └── UDC (USB Device Controller) driver          │
  └──────────────────────────────────────────────────┘
```

---

## 8.2 USB Transfer Types

```
USB Transfer Types:

┌────────────┬───────────────────────────────────────────┐
│ Control    │ Setup/status/data phases                  │
│            │ Used for: device enumeration, config      │
│            │ Endpoint 0 (all devices)                  │
│            │ Guaranteed delivery, low bandwidth        │
├────────────┼───────────────────────────────────────────┤
│ Bulk       │ Large reliable data transfers             │
│            │ Used for: storage, printing, networking   │
│            │ Error detection + retry                   │
│            │ No bandwidth guarantee (best effort)      │
├────────────┼───────────────────────────────────────────┤
│ Interrupt  │ Small periodic data                       │
│            │ Used for: keyboard, mouse, gamepad        │
│            │ Guaranteed latency (polling interval)     │
│            │ Small packets (8-64 bytes USB 1.x/2.0)   │
├────────────┼───────────────────────────────────────────┤
│ Isochronous│ Constant-rate streaming                   │
│            │ Used for: audio, video, webcam            │
│            │ Guaranteed bandwidth, no error retry      │
│            │ Packet loss acceptable (real-time)        │
└────────────┴───────────────────────────────────────────┘
```

---

## 8.3 USB Descriptors and Enumeration

```
USB Descriptor Hierarchy:

Device Descriptor (1 per device)
│ ├── idVendor, idProduct (matching)
│ ├── bDeviceClass
│ └── bNumConfigurations
│
├── Configuration Descriptor (usually 1)
│   ├── bNumInterfaces
│   ├── bMaxPower
│   │
│   ├── Interface Descriptor (e.g., interface 0)
│   │   ├── bInterfaceClass (e.g., USB_CLASS_HID)
│   │   ├── bNumEndpoints
│   │   │
│   │   ├── Endpoint Descriptor (IN, interrupt)
│   │   │   ├── bEndpointAddress (0x81 = EP1 IN)
│   │   │   ├── bmAttributes (interrupt/bulk/iso)
│   │   │   ├── wMaxPacketSize
│   │   │   └── bInterval (polling rate)
│   │   │
│   │   └── Endpoint Descriptor (OUT, interrupt)
│   │
│   └── Interface Descriptor (interface 1, ...)

Enumeration sequence:
  1. Device connects (VBUS detected)
  2. Hub detects and resets port
  3. Host sends GET_DESCRIPTOR (device)
  4. Host assigns address (SET_ADDRESS)
  5. Host reads full descriptors
  6. Host selects configuration (SET_CONFIGURATION)
  7. USB core matches driver (vid/pid or class)
  8. Driver probe() called
```

---

## 8.4 Writing a USB Device Driver

```c
/* USB device driver example */

static const struct usb_device_id my_usb_ids[] = {
    { USB_DEVICE(0x1234, 0x5678) },       /* match by VID/PID */
    { USB_DEVICE_INFO(USB_CLASS_HID,       /* match by class */
                      USB_SUBCLASS_BOOT,
                      USB_PROTOCOL_MOUSE) },
    { }  /* terminator */
};
MODULE_DEVICE_TABLE(usb, my_usb_ids);

static int my_usb_probe(struct usb_interface *intf,
                        const struct usb_device_id *id)
{
    struct usb_device *udev = interface_to_usbdev(intf);
    struct usb_endpoint_descriptor *ep;
    struct urb *urb;

    /* Find the interrupt IN endpoint */
    ep = &intf->cur_altsetting->endpoint[0].desc;
    if (!usb_endpoint_is_int_in(ep))
        return -ENODEV;

    /* Allocate URB */
    urb = usb_alloc_urb(0, GFP_KERNEL);

    /* Fill interrupt URB */
    usb_fill_int_urb(urb, udev,
                     usb_rcvintpipe(udev, ep->bEndpointAddress),
                     buf, buf_size,
                     my_urb_complete,    /* completion callback */
                     priv,               /* context */
                     ep->bInterval);     /* polling interval */

    /* Submit URB — starts receiving data */
    ret = usb_submit_urb(urb, GFP_KERNEL);

    return 0;
}

/* URB completion callback (interrupt context) */
static void my_urb_complete(struct urb *urb)
{
    if (urb->status == 0) {
        /* Process received data */
        process_data(urb->transfer_buffer, urb->actual_length);
    }

    /* Re-submit URB for next transfer */
    usb_submit_urb(urb, GFP_ATOMIC);
}

static void my_usb_disconnect(struct usb_interface *intf)
{
    usb_kill_urb(urb);
    usb_free_urb(urb);
}

static struct usb_driver my_usb_driver = {
    .name       = "my-usb-driver",
    .id_table   = my_usb_ids,
    .probe      = my_usb_probe,
    .disconnect = my_usb_disconnect,
};
module_usb_driver(my_usb_driver);
```

---

## 8.5 USB Gadget Framework

```
USB Gadget (Device Mode):

When Linux acts as a USB device (e.g., Android phone):

  ┌───────────────────────────────────────┐
  │  USB Host (PC)                        │
  └───────────────┬───────────────────────┘
                  │ USB cable
  ┌───────────────▼───────────────────────┐
  │  Linux USB Gadget Stack               │
  │                                       │
  │  ┌─────────────────────────────────┐  │
  │  │ Gadget Functions                 │  │
  │  │ ├── f_mass_storage (USB drive)   │  │
  │  │ ├── f_ecm / f_rndis (USB net)   │  │
  │  │ ├── f_acm (USB serial)          │  │
  │  │ ├── f_uvc (USB webcam)          │  │
  │  │ └── f_adb (Android Debug)       │  │
  │  └──────────────┬──────────────────┘  │
  │                 │                      │
  │  ┌──────────────▼──────────────────┐  │
  │  │ Composite Framework             │  │
  │  │ (combine multiple functions)    │  │
  │  └──────────────┬──────────────────┘  │
  │                 │                      │
  │  ┌──────────────▼──────────────────┐  │
  │  │ UDC Driver (USB Device Ctrl)    │  │
  │  │ ├── dwc3                        │  │
  │  │ ├── dwc2                        │  │
  │  │ └── chipidea                    │  │
  │  └─────────────────────────────────┘  │
  └───────────────────────────────────────┘

ConfigFS-based configuration:
  $ mkdir /config/usb_gadget/g1
  $ echo 0x1d6b > idVendor
  $ echo 0x0104 > idProduct
  $ mkdir functions/acm.usb0
  $ ln -s functions/acm.usb0 configs/c.1/
  $ echo dwc3.0.auto > UDC
```

---

## 8.6 Debug and User-Space Tools

```bash
# USB debugging

# List USB devices
$ lsusb
$ lsusb -v -d 1234:5678

# USB device tree
$ lsusb -t

# Kernel USB messages
$ dmesg | grep -i usb

# sysfs
$ cat /sys/bus/usb/devices/1-1/idVendor
$ cat /sys/bus/usb/devices/1-1/idProduct
$ cat /sys/bus/usb/devices/1-1/product

# usbmon (packet tracing)
$ mount -t debugfs none /sys/kernel/debug
$ cat /sys/kernel/debug/usb/usbmon/0u

# USB power
$ cat /sys/bus/usb/devices/1-1/power/control  # auto/on
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| USB core | drivers/usb/core/ | Device/hub/URB management |
| USB HCD | drivers/usb/host/ | xHCI/EHCI/OHCI drivers |
| USB gadget | drivers/usb/gadget/ | Device-mode framework |
| USB functions | drivers/usb/gadget/function/ | Gadget functions |
| DWC3 | drivers/usb/dwc3/ | DesignWare USB3 controller |
| USB class | drivers/usb/class/ | CDC-ACM, etc. |

---

## Interview Questions

**Q1: Explain the URB lifecycle in a USB driver.**
A: URB (USB Request Block) is the fundamental I/O unit. (1) Driver allocates URB with `usb_alloc_urb()`. (2) Driver fills it with `usb_fill_int_urb()` / `usb_fill_bulk_urb()` specifying endpoint, buffer, size, and completion callback. (3) Driver submits with `usb_submit_urb()` — USB core queues it to the HCD. (4) HCD schedules the transfer on the bus. (5) When transfer completes (or fails), the completion callback fires with status in `urb->status`. (6) For continuous polling (interrupt endpoints), the callback re-submits the URB. (7) On disconnect, `usb_kill_urb()` cancels pending URBs.

**Q2: What is the difference between USB host mode and gadget mode?**
A: Host mode: Linux controls the bus, enumerates connected devices, and runs device drivers (e.g., usb-storage for USB drives). Uses HCD (xHCI/EHCI) and USB core. Gadget mode: Linux acts as a USB device connected to another host (e.g., Android phone connected to PC). Uses UDC (USB Device Controller) driver and gadget functions (f_mass_storage, f_adb). The composite framework allows combining multiple functions. OTG (On-The-Go) hardware can switch between host and gadget modes.

**Q3: How does USB device enumeration work?**
A: (1) Device plugs in — hub detects VBUS change. (2) Hub resets the port. (3) Host sends GET_DESCRIPTOR on default address 0 to read device descriptor. (4) Host issues SET_ADDRESS to assign unique bus address. (5) Host reads full device/config/interface/endpoint descriptors. (6) Host selects configuration with SET_CONFIGURATION. (7) USB core matches device against registered drivers using VID/PID or class/subclass/protocol. (8) Matching driver's probe() is called.

---

## Summary

- USB subsystem has three layers: device drivers, USB core, and HCD
- Four transfer types: control, bulk, interrupt, isochronous
- URBs are the I/O primitive — allocate, fill, submit, complete, resubmit
- Device matching uses VID/PID or class/subclass/protocol tables
- Gadget framework makes Linux act as a USB device (ADB, mass storage, network)
- DWC3/DWC2 are common USB controllers on embedded SoCs

---

*Next: [Chapter 9 — I2C Framework](Chapter_09_I2C_Framework.md)*
