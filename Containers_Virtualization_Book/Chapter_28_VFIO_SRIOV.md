# Chapter 28: VFIO Device Passthrough and SR-IOV Practical Guide

## Learning Goals
- Master VFIO device passthrough step-by-step
- Understand IOMMU group management and ACS
- Learn SR-IOV VF creation and assignment
- Know GPU passthrough specifics

---

## 1. VFIO Passthrough Step-by-Step

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Step 1: Enable IOMMU in bootloader                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # GRUB: /etc/default/grub                │            │
  │  │ GRUB_CMDLINE_LINUX="intel_iommu=on"      │            │
  │  │ # or AMD:                                │            │
  │  │ GRUB_CMDLINE_LINUX="amd_iommu=on"        │            │
  │  │                                          │            │
  │  │ # Verify:                                │            │
  │  │ dmesg | grep -i iommu                    │            │
  │  │ → "DMAR: IOMMU enabled"                 │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Step 2: Identify device and IOMMU group                │
  │  ┌──────────────────────────────────────────┐            │
  │  │ lspci -nn | grep -i nvidia               │            │
  │  │ → 01:00.0 VGA: NVIDIA [10de:2204]       │            │
  │  │                                          │            │
  │  │ # Check IOMMU group:                     │            │
  │  │ find /sys/kernel/iommu_groups/ -type l \  │            │
  │  │   | grep 01:00                            │            │
  │  │ → /sys/kernel/iommu_groups/14/devices/   │            │
  │  │   0000:01:00.0  (GPU)                    │            │
  │  │   0000:01:00.1  (Audio)                  │            │
  │  │ → Both in group 14, must pass both!      │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Step 3: Unbind from host driver                        │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Unbind GPU from nvidia/nouveau driver: │            │
  │  │ echo 0000:01:00.0 > \                    │            │
  │  │   /sys/bus/pci/devices/0000:01:00.0/\    │            │
  │  │   driver/unbind                          │            │
  │  │                                          │            │
  │  │ # Unbind audio device too:               │            │
  │  │ echo 0000:01:00.1 > \                    │            │
  │  │   /sys/bus/pci/devices/0000:01:00.1/\    │            │
  │  │   driver/unbind                          │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Step 4: Bind to vfio-pci                               │
  │  ┌──────────────────────────────────────────┐            │
  │  │ modprobe vfio-pci                         │            │
  │  │                                          │            │
  │  │ echo "10de 2204" > \                     │            │
  │  │   /sys/bus/pci/drivers/vfio-pci/new_id   │            │
  │  │ # OR use driver_override:                │            │
  │  │ echo vfio-pci > \                        │            │
  │  │   /sys/bus/pci/devices/0000:01:00.0/\    │            │
  │  │   driver_override                        │            │
  │  │ echo 0000:01:00.0 > \                    │            │
  │  │   /sys/bus/pci/drivers/vfio-pci/bind     │            │
  │  │                                          │            │
  │  │ # Verify:                                │            │
  │  │ ls /dev/vfio/                             │            │
  │  │ → 14  vfio                               │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Step 5: Launch VM with VFIO device                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ qemu-system-x86_64 \                     │            │
  │  │   -device vfio-pci,host=01:00.0 \        │            │
  │  │   -device vfio-pci,host=01:00.1 \        │            │
  │  │   -m 8G -smp 4 \                         │            │
  │  │   -drive file=vm.qcow2,format=qcow2     │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. GPU Passthrough Specifics

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  GPU passthrough challenges:                             │
  │                                                           │
  │  Problem 1: Code 43 (NVIDIA detects VM)                 │
  │  ┌──────────────────────────────────────────┐            │
  │  │ NVIDIA consumer drivers refuse to work   │            │
  │  │ when they detect a hypervisor             │            │
  │  │                                          │            │
  │  │ Fix: hide hypervisor from guest           │            │
  │  │ <features>                                │            │
  │  │   <kvm><hidden state='on'/></kvm>        │            │
  │  │ </features>                               │            │
  │  │ -cpu host,kvm=off,hv_vendor_id=null      │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Problem 2: GPU reset on VM shutdown                    │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Some GPUs can't be reset after VM stops  │            │
  │  │ → re-assignment fails until host reboot  │            │
  │  │                                          │            │
  │  │ Check: lspci -s 01:00.0 -vvv | grep FLR │            │
  │  │ → "FLReset+" means Function Level Reset  │            │
  │  │   is supported                           │            │
  │  │                                          │            │
  │  │ Vendor reset module (workaround):        │            │
  │  │ modprobe vendor-reset                     │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Problem 3: VBIOS for GPU initialization                │
  │  ┌──────────────────────────────────────────┐            │
  │  │ GPU needs VBIOS ROM for initialization   │            │
  │  │ May need to extract and pass explicitly: │            │
  │  │                                          │            │
  │  │ -device vfio-pci,host=01:00.0,\          │            │
  │  │   romfile=/path/to/gpu.rom               │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Mediated devices (mdev) — GPU sharing:                 │
  │  ┌──────────────────────────────────────────┐            │
  │  │ NVIDIA vGPU / Intel GVT-g:              │            │
  │  │ One GPU shared among multiple VMs        │            │
  │  │ Time-sliced or SR-IOV (A100+)            │            │
  │  │                                          │            │
  │  │ # Create mediated device:                │            │
  │  │ echo uuid > /sys/class/mdev_bus/.../\    │            │
  │  │   mdev_supported_types/nvidia-*/create   │            │
  │  │                                          │            │
  │  │ # Assign to VM: -device vfio-pci,\       │            │
  │  │   sysfsdev=/sys/bus/mdev/devices/<uuid> │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. SR-IOV Practical Guide

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  SR-IOV setup (Intel 82599 / ixgbe example):             │
  │                                                           │
  │  Step 1: Check SR-IOV support                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ lspci -s 03:00.0 -vvv | grep -i sriov   │            │
  │  │ → "Total VFs: 64, Initial VFs: 64"      │            │
  │  │                                          │            │
  │  │ # Check driver supports it:              │            │
  │  │ modinfo ixgbe | grep -i sriov            │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Step 2: Enable VFs                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Create 4 VFs:                          │            │
  │  │ echo 4 > /sys/class/net/enp3s0f0/\       │            │
  │  │   device/sriov_numvfs                    │            │
  │  │                                          │            │
  │  │ # Verify VFs were created:               │            │
  │  │ lspci | grep "Virtual Function"          │            │
  │  │ → 03:10.0 Ethernet: Intel VF            │            │
  │  │ → 03:10.2 Ethernet: Intel VF            │            │
  │  │ → 03:10.4 Ethernet: Intel VF            │            │
  │  │ → 03:10.6 Ethernet: Intel VF            │            │
  │  │                                          │            │
  │  │ ip link show enp3s0f0                     │            │
  │  │ → vf 0 MAC ..., vf 1 MAC ..., etc.      │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Step 3: Configure VF                                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Set VF MAC address (from PF):          │            │
  │  │ ip link set enp3s0f0 vf 0 mac \          │            │
  │  │   52:54:00:aa:bb:01                      │            │
  │  │                                          │            │
  │  │ # Set VLAN (optional):                   │            │
  │  │ ip link set enp3s0f0 vf 0 vlan 100       │            │
  │  │                                          │            │
  │  │ # Set rate limit (Mbps):                 │            │
  │  │ ip link set enp3s0f0 vf 0 max_tx_rate 1000│           │
  │  │                                          │            │
  │  │ # Enable spoof checking:                 │            │
  │  │ ip link set enp3s0f0 vf 0 spoofchk on    │            │
  │  │                                          │            │
  │  │ # Trust VF (allows promisc, multicast):  │            │
  │  │ ip link set enp3s0f0 vf 0 trust on        │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Step 4: Bind VF to vfio-pci and assign                 │
  │  ┌──────────────────────────────────────────┐            │
  │  │ echo 0000:03:10.0 > \                    │            │
  │  │   .../driver/unbind                      │            │
  │  │ echo vfio-pci > \                        │            │
  │  │   .../driver_override                    │            │
  │  │ echo 0000:03:10.0 > \                    │            │
  │  │   /sys/bus/pci/drivers/vfio-pci/bind     │            │
  │  │                                          │            │
  │  │ qemu-system-x86_64 \                     │            │
  │  │   -device vfio-pci,host=03:10.0 \        │            │
  │  │   ...                                    │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Live Migration with Passthrough

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Challenge: passed-through device has hardware state     │
  │  that can't be migrated to different hardware            │
  │                                                           │
  │  ┌──────────────┬─────────────────────────────────────┐ │
  │  │ Approach     │ How it works                         │ │
  │  ├──────────────┼─────────────────────────────────────┤ │
  │  │ Hot-unplug   │ 1. Add virtio-net to guest          │ │
  │  │ + replug     │ 2. Switch traffic to virtio         │ │
  │  │              │ 3. Hot-unplug VFIO device           │ │
  │  │              │ 4. Live migrate (virtio works)      │ │
  │  │              │ 5. Hot-plug VFIO on dest host       │ │
  │  │              │ 6. Brief downtime during switch     │ │
  │  ├──────────────┼─────────────────────────────────────┤ │
  │  │ VFIO         │ New kernel feature (v5.18+)         │ │
  │  │ migration    │ VFIO_DEVICE_FEATURE_MIGRATION       │ │
  │  │ (v2 API)     │ Save/restore device state via FD    │ │
  │  │              │ Requires device firmware support    │ │
  │  │              │ mlx5 supports it (ConnectX-6+)      │ │
  │  ├──────────────┼─────────────────────────────────────┤ │
  │  │ vDPA         │ Device implements virtio interface  │ │
  │  │              │ virtio state is well-defined        │ │
  │  │              │ Guest uses standard virtio driver   │ │
  │  │              │ Save/restore virtio device state    │ │
  │  │              │ Transparent migration               │ │
  │  └──────────────┴─────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Walk through the complete process of GPU passthrough with VFIO.**
**A:** (1) **Enable IOMMU**: add `intel_iommu=on` (or `amd_iommu=on`) to kernel command line. Verify with `dmesg | grep IOMMU`. (2) **Identify the device**: use `lspci -nn` to find the GPU (e.g., `01:00.0`) and its vendor:device ID (e.g., `10de:2204`). (3) **Check IOMMU group**: list devices in `/sys/kernel/iommu_groups/*/devices/`. ALL devices in the same group must be passed through together. GPUs typically have a paired audio device in the same group. (4) **Unbind from host driver**: write BDF to the current driver's unbind file. For NVIDIA, this means unbinding from `nvidia` or `nouveau`. (5) **Bind to vfio-pci**: either write vendor:device ID to `/sys/bus/pci/drivers/vfio-pci/new_id` or use `driver_override`. Verify `/dev/vfio/<group>` appears. (6) **Launch QEMU**: use `-device vfio-pci,host=01:00.0`. Additional considerations: pass VBIOS ROM file if needed (`romfile=`), hide KVM from NVIDIA consumer drivers (`kvm=off`), ensure UEFI firmware (OVMF) for UEFI GPU initialization, and verify the GPU supports Function Level Reset (FLR) for clean VM shutdown/restart. For GPU sharing across VMs: use mediated devices (NVIDIA vGPU or Intel GVT-g) — these are time-sliced or (newer NVIDIA) SR-IOV-based.

**Q2: What are the challenges of live migration with device passthrough?**
**A:** Device passthrough assigns a physical device exclusively to a VM. Live migration must transfer the VM to a different host — but the physical device can't move. Challenges: (1) **Device state**: the device has internal state (register values, queue pointers, firmware state) that isn't exposed to the hypervisor. Standard VFIO had no mechanism to save/restore this. (2) **In-flight I/O**: DMA operations may be in progress when migration starts; quiescing I/O without losing data is hard. (3) **Interrupt routing**: MSI-X vectors and IOMMU mappings are host-specific. Solutions: (a) **Hot-unplug approach**: most common today — temporarily add a virtio-net/virtio-blk device, switch the guest's traffic to the virtio device, hot-unplug the passthrough device, live migrate using the virtio backend, then hot-plug a new passthrough device on the destination host. Requires guest OS support for device hot-plug and brief traffic disruption. (b) **VFIO migration v2 API** (Linux 5.18+): adds `VFIO_DEVICE_FEATURE_MIGRATION` — the device exposes a migration state machine (RUNNING → STOP_COPY → RESUMING). QEMU reads device state via a file descriptor and transfers it to the destination. Requires firmware support — currently Mellanox ConnectX-6+ supports this. (c) **vDPA**: uses standard virtio device state (well-defined by the virtio spec), making migration straightforward — save/restore virtio queue state, descriptor ring pointers, and feature bits. This is the cleanest solution but requires vDPA-capable hardware.

---

## Summary

- VFIO passthrough: unbind host driver → bind vfio-pci → QEMU uses device fd → guest accesses directly
- IOMMU groups: all devices in a group must go to the same VM; ACS creates per-device groups
- GPU passthrough: hide KVM from NVIDIA, handle VBIOS ROM, verify FLR support
- Mediated devices (mdev): GPU sharing via NVIDIA vGPU / Intel GVT-g time-slicing
- SR-IOV: echo N > sriov_numvfs → configure VF MAC/VLAN → bind to vfio-pci → assign to VM
- Live migration with passthrough: hot-unplug method, VFIO migration v2, or vDPA

---

[Previous: vhost and vhost-user ←](Chapter_27_Vhost.md) | [Next: Kubernetes and Container Runtime Interface →](Chapter_29_K8s_CRI.md)
