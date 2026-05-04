# Chapter 5: Modules — Build, Load, and Management

## Learning Goals
- Understand kernel module lifecycle: build, sign, load, unload
- Learn modprobe, depmod, and module dependency resolution
- Master module parameters and sysfs interface
- Know DKMS for out-of-tree module management

---

## 1. Module Build and Structure

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Module = dynamically loadable kernel code (.ko file)   │
  │                                                           │
  │  Minimal module:                                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ #include <linux/module.h>                │            │
  │  │ #include <linux/init.h>                  │            │
  │  │                                          │            │
  │  │ static int __init mymod_init(void)       │            │
  │  │ {                                        │            │
  │  │     pr_info("mymod loaded\n");           │            │
  │  │     return 0;                            │            │
  │  │ }                                        │            │
  │  │                                          │            │
  │  │ static void __exit mymod_exit(void)      │            │
  │  │ {                                        │            │
  │  │     pr_info("mymod unloaded\n");         │            │
  │  │ }                                        │            │
  │  │                                          │            │
  │  │ module_init(mymod_init);                 │            │
  │  │ module_exit(mymod_exit);                 │            │
  │  │                                          │            │
  │  │ MODULE_LICENSE("GPL");                   │            │
  │  │ MODULE_AUTHOR("dev");                    │            │
  │  │ MODULE_DESCRIPTION("Example module");    │            │
  │  │ MODULE_VERSION("1.0");                   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  .ko file sections:                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ .text        — code                      │            │
  │  │ .data        — initialized data          │            │
  │  │ .bss         — zero-initialized data     │            │
  │  │ .rodata      — read-only data            │            │
  │  │ .modinfo     — MODULE_* strings:         │            │
  │  │                license=GPL               │            │
  │  │                author=dev                │            │
  │  │                depends=usbcore           │            │
  │  │                alias=usb:v1234p5678*     │            │
  │  │ __versions   — symbol CRCs              │            │
  │  │ .gnu.linkonce.this_module                │            │
  │  │              — struct module instance    │            │
  │  │ .init.text   — init code (freed after)   │            │
  │  │ .exit.text   — exit/cleanup code         │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  __init and __exit:                                      │
  │  __init → placed in .init.text → freed after module_init│
  │  __exit → not compiled if built-in (never called)       │
  │           only compiled for modules                     │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Module Loading Infrastructure

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Loading commands:                                       │
  │  ┌──────────────────────────────────────────┐            │
  │  │ insmod mymod.ko                          │            │
  │  │   - Loads specific .ko file              │            │
  │  │   - No dependency resolution             │            │
  │  │   - Module path must be specified        │            │
  │  │                                          │            │
  │  │ modprobe mymod                           │            │
  │  │   - Searches /lib/modules/$(uname -r)/   │            │
  │  │   - Resolves and loads dependencies      │            │
  │  │   - Uses modules.dep database            │            │
  │  │                                          │            │
  │  │ rmmod mymod                              │            │
  │  │   - Unloads module                       │            │
  │  │   - Fails if module is in use            │            │
  │  │                                          │            │
  │  │ modprobe -r mymod                        │            │
  │  │   - Unloads + removes unused dependencies│            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  depmod and modules.dep:                                 │
  │  ┌──────────────────────────────────────────┐            │
  │  │ depmod -a                                │            │
  │  │   Scans all .ko in /lib/modules/$(uname -r)/│        │
  │  │   Generates:                             │            │
  │  │                                          │            │
  │  │   modules.dep:                           │            │
  │  │     e1000.ko: ptp.ko                     │            │
  │  │     usb-storage.ko: usbcore.ko scsi.ko  │            │
  │  │                                          │            │
  │  │   modules.dep.bin (binary, fast lookup)  │            │
  │  │                                          │            │
  │  │   modules.alias:                         │            │
  │  │     alias usb:v1234p5678* usb-storage    │            │
  │  │     alias pci:v00008086d0000100E* e1000  │            │
  │  │                                          │            │
  │  │   modules.symbols:                       │            │
  │  │     alias symbol:usb_submit_urb usbcore  │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Module loading kernel path:                             │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. insmod() syscall (init_module)        │            │
  │  │ 2. Kernel verifies ELF format            │            │
  │  │ 3. Check module signature (if enforced)  │            │
  │  │ 4. Allocate memory for module sections   │            │
  │  │ 5. Relocate symbols                      │            │
  │  │ 6. Resolve symbols against kernel +      │            │
  │  │    other loaded modules                  │            │
  │  │ 7. Check MODVERSIONS CRCs                │            │
  │  │ 8. Call module_init function             │            │
  │  │ 9. Free .init sections                   │            │
  │  │ 10. Module is live                       │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Module Parameters

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Module parameters: configure at load time               │
  │                                                           │
  │  Code:                                                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ static int debug_level = 0;              │            │
  │  │ module_param(debug_level, int, 0644);    │            │
  │  │ MODULE_PARM_DESC(debug_level,             │            │
  │  │                  "Debug verbosity 0-3"); │            │
  │  │                                          │            │
  │  │ static char *name = "default";           │            │
  │  │ module_param(name, charp, 0444);         │            │
  │  │                                          │            │
  │  │ static bool enable = true;               │            │
  │  │ module_param(enable, bool, 0644);        │            │
  │  │                                          │            │
  │  │ static int arr[4];                       │            │
  │  │ static int arr_count;                    │            │
  │  │ module_param_array(arr, int, &arr_count, │            │
  │  │                    0444);                │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Usage:                                                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Load with parameters:                 │            │
  │  │ modprobe mymod debug_level=2 enable=0    │            │
  │  │ insmod mymod.ko debug_level=2            │            │
  │  │                                          │            │
  │  │ # View parameters:                       │            │
  │  │ cat /sys/module/mymod/parameters/debug_level│         │
  │  │ → 2                                      │            │
  │  │                                          │            │
  │  │ # Modify at runtime (if permissions allow):│          │
  │  │ echo 3 > /sys/module/mymod/parameters/debug_level│   │
  │  │                                          │            │
  │  │ # Built-in modules: set via kernel cmdline│           │
  │  │ mymod.debug_level=2                      │            │
  │  │ # or /etc/modprobe.d/mymod.conf:         │            │
  │  │ options mymod debug_level=2 enable=0     │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Permission bits (0644):                                 │
  │  0444 = read-only in sysfs                              │
  │  0644 = read by all, write by root                      │
  │  0    = no sysfs entry created                          │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Module Signing and DKMS

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Module signing:                                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ CONFIG_MODULE_SIG=y                      │            │
  │  │   Enable signature checking              │            │
  │  │                                          │            │
  │  │ CONFIG_MODULE_SIG_FORCE=y                │            │
  │  │   Refuse to load unsigned modules        │            │
  │  │                                          │            │
  │  │ Signing:                                 │            │
  │  │ scripts/sign-file sha256 \               │            │
  │  │   certs/signing_key.pem \                │            │
  │  │   certs/signing_key.x509 \               │            │
  │  │   mymod.ko                               │            │
  │  │                                          │            │
  │  │ Key generation (auto at build time):     │            │
  │  │ certs/signing_key.pem (private)          │            │
  │  │ certs/signing_key.x509 (cert, embedded   │            │
  │  │   in vmlinux's .system_keyring)          │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  DKMS (Dynamic Kernel Module Support):                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Problem: out-of-tree modules break on    │            │
  │  │ kernel updates (ABI changes)             │            │
  │  │                                          │            │
  │  │ DKMS: auto-rebuild modules after kernel  │            │
  │  │ update                                   │            │
  │  │                                          │            │
  │  │ /usr/src/mymod-1.0/dkms.conf:            │            │
  │  │   PACKAGE_NAME="mymod"                   │            │
  │  │   PACKAGE_VERSION="1.0"                  │            │
  │  │   BUILT_MODULE_NAME[0]="mymod"           │            │
  │  │   DEST_MODULE_LOCATION[0]="/updates"     │            │
  │  │   AUTOINSTALL="yes"                      │            │
  │  │   MAKE[0]="make -C ${kernel_source_dir}  │            │
  │  │     M=${dkms_tree}/${PACKAGE_NAME}/\     │            │
  │  │     ${PACKAGE_VERSION}/build modules"    │            │
  │  │                                          │            │
  │  │ dkms add mymod/1.0                       │            │
  │  │ dkms build mymod/1.0                     │            │
  │  │ dkms install mymod/1.0                   │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain the difference between `insmod` and `modprobe` and how module dependencies work.**
**A:** `insmod` directly loads a specific `.ko` file via the `init_module()` or `finit_module()` syscall. It takes a file path and does NO dependency resolution — if the module depends on symbols from another module that isn't loaded, insmod fails with "Unknown symbol in module." `modprobe` is a userspace tool that adds intelligence: it searches `/lib/modules/$(uname -r)/` for the module by name (not path), reads `modules.dep` (generated by `depmod`), and loads all dependency modules before the requested module. Example: `modprobe usb-storage` first loads `scsi_mod`, then `usbcore`, then `usb-storage`. Dependencies are determined by `depmod`, which scans all `.ko` files, reads their undefined symbols and exports (`EXPORT_SYMBOL`), and computes which modules need which other modules. `modprobe` also reads `/etc/modprobe.d/*.conf` for module options, aliases, and blacklist rules. `modprobe -r` unloads a module and its unused dependencies. Aliases in `modules.alias` (generated by depmod from `MODULE_ALIAS` macros) enable automatic module loading — when the kernel discovers a device with PCI ID `8086:100E`, udev runs `modprobe pci:v00008086d0000100E*`, modprobe matches this alias to `e1000`, and loads it.

**Q2: How does kernel module signing work and why is it important for Secure Boot?**
**A:** Module signing uses asymmetric cryptography to verify module integrity and authenticity. At kernel build time: (1) A key pair is generated (`certs/signing_key.pem`) or a custom key is provided via `CONFIG_MODULE_SIG_KEY`. (2) The public key certificate (`.x509`) is embedded into the vmlinux binary's built-in keyring (`.builtin_trusted_keys`). (3) During `make modules_install`, each `.ko` is signed by `scripts/sign-file` — it computes a hash (SHA-256) of the module's ELF data and signs it with the private key. The signature is appended to the `.ko` file. At load time: (4) The kernel extracts the appended signature, verifies it against the embedded public key. (5) `CONFIG_MODULE_SIG=y`: unsigned modules generate a kernel taint warning but still load. (6) `CONFIG_MODULE_SIG_FORCE=y`: unsigned or incorrectly signed modules are REFUSED — `insmod` returns EKEYREJECTED. This matters for Secure Boot: the UEFI firmware verifies the bootloader, the bootloader verifies the kernel (signed by distribution key), and the kernel verifies modules (signed by build-time key). If any module could be loaded without verification, an attacker could insert a rootkit as a kernel module, breaking the chain. Lockdown mode (`CONFIG_LOCK_DOWN_LSM`) further restricts: even if a module signature verifies, lockdown can prevent modules from accessing raw hardware if the integrity level requires it.

---

## Summary

- Module lifecycle: compile → modpost → link .ko → sign → insmod/modprobe → init → live
- insmod: direct load, no deps; modprobe: dependency resolution via modules.dep
- depmod: scans all .ko, generates modules.dep, modules.alias, modules.symbols
- Module parameters: module_param(), accessible via /sys/module/name/parameters/
- Module signing: private key signs .ko; public key in vmlinux verifies at load
- DKMS: auto-rebuild out-of-tree modules on kernel update
- __init: freed after load; __exit: never compiled for built-in

---

[Previous: Cross-Compilation ←](Chapter_04_Cross_Compilation.md) | [Next: Device Tree and ACPI →](Chapter_06_DT_ACPI.md)
