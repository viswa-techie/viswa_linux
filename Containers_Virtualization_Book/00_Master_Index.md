# Linux Containers & Virtualization — Complete Guide

## Master Index

This book covers Linux containers (namespaces, cgroups, runtimes) and virtualization (KVM, QEMU, virtio) from kernel internals to production infrastructure.

---

## Book Architecture

```
  Part I   — Container Foundations (Ch 1-5)
  Part II  — Namespaces Deep Dive (Ch 6-10)
  Part III — Cgroups and Resource Control (Ch 11-14)
  Part IV  — Container Runtimes and OCI (Ch 15-18)
  Part V   — Virtualization Foundations (Ch 19-22)
  Part VI  — KVM and QEMU Internals (Ch 23-26)
  Part VII — Virtio and Device Models (Ch 27-29)
  Part VIII — Advanced Topics and Interview (Ch 30-32)
```

---

## Part I — Container Foundations

| Ch | Title | File |
|----|-------|------|
| 01 | Foundations of OS-Level Virtualization | [Chapter_01](Chapter_01_Foundations.md) |
| 02 | History: chroot to Containers | [Chapter_02](Chapter_02_History.md) |
| 03 | Container vs VM Architecture | [Chapter_03](Chapter_03_Container_vs_VM.md) |
| 04 | Linux Container Building Blocks | [Chapter_04](Chapter_04_Building_Blocks.md) |
| 05 | Container Security Model | [Chapter_05](Chapter_05_Container_Security.md) |

## Part II — Namespaces Deep Dive

| Ch | Title | File |
|----|-------|------|
| 06 | Namespace Architecture and Kernel Implementation | [Chapter_06](Chapter_06_Namespace_Architecture.md) |
| 07 | PID and Mount Namespaces | [Chapter_07](Chapter_07_PID_Mount_NS.md) |
| 08 | Network and User Namespaces | [Chapter_08](Chapter_08_Net_User_NS.md) |
| 09 | UTS, IPC, Cgroup, and Time Namespaces | [Chapter_09](Chapter_09_UTS_IPC_Cgroup_Time_NS.md) |
| 10 | Namespace Lifecycle and Management | [Chapter_10](Chapter_10_NS_Lifecycle.md) |

## Part III — Cgroups and Resource Control

| Ch | Title | File |
|----|-------|------|
| 11 | Cgroups v1 Architecture | [Chapter_11](Chapter_11_Cgroups_v1.md) |
| 12 | Cgroups v2 Unified Hierarchy | [Chapter_12](Chapter_12_Cgroups_v2.md) |
| 13 | CPU, Memory, and I/O Controllers | [Chapter_13](Chapter_13_Controllers.md) |
| 14 | Cgroup Delegation and Systemd Integration | [Chapter_14](Chapter_14_Delegation_Systemd.md) |

## Part IV — Container Runtimes and OCI

| Ch | Title | File |
|----|-------|------|
| 15 | OCI Specification and Image Format | [Chapter_15](Chapter_15_OCI_Spec.md) |
| 16 | Low-Level Runtimes: runc Internals | [Chapter_16](Chapter_16_Runc.md) |
| 17 | High-Level Runtimes: containerd and CRI-O | [Chapter_17](Chapter_17_Containerd_CRIO.md) |
| 18 | Container Networking and Storage | [Chapter_18](Chapter_18_Container_Networking.md) |

## Part V — Virtualization Foundations

| Ch | Title | File |
|----|-------|------|
| 19 | Foundations of Hardware Virtualization | [Chapter_19](Chapter_19_Virtualization_Foundations.md) |
| 20 | x86 Virtualization Extensions (VT-x, EPT) | [Chapter_20](Chapter_20_VTx_EPT.md) |
| 21 | ARM Virtualization (EL2, Stage-2) | [Chapter_21](Chapter_21_ARM_Virtualization.md) |
| 22 | Hypervisor Types and Architecture | [Chapter_22](Chapter_22_Hypervisor_Types.md) |

## Part VI — KVM and QEMU Internals

| Ch | Title | File |
|----|-------|------|
| 23 | KVM Architecture and Kernel Module | [Chapter_23](Chapter_23_KVM_Architecture.md) |
| 24 | QEMU Userspace and Device Emulation | [Chapter_24](Chapter_24_QEMU.md) |
| 25 | Memory Virtualization (EPT, Shadow PT, Ballooning) | [Chapter_25](Chapter_25_Memory_Virtualization.md) |
| 26 | I/O Virtualization and Interrupt Delivery | [Chapter_26](Chapter_26_IO_Virtualization.md) |

## Part VII — Virtio and Device Models

| Ch | Title | File |
|----|-------|------|
| 27 | Virtio Specification and Architecture | [Chapter_27](Chapter_27_Virtio.md) |
| 28 | Vhost and Vhost-User Acceleration | [Chapter_28](Chapter_28_Vhost.md) |
| 29 | Device Passthrough: VFIO and SR-IOV | [Chapter_29](Chapter_29_VFIO_SRIOV.md) |

## Part VIII — Advanced Topics and Interview

| Ch | Title | File |
|----|-------|------|
| 30 | Kubernetes, Pods, and CRI | [Chapter_30](Chapter_30_Kubernetes_CRI.md) |
| 31 | Performance, Debugging, and Monitoring | [Chapter_31](Chapter_31_Performance_Debug.md) |
| 32 | Interview Preparation | [Chapter_32](Chapter_32_Interview.md) |

---

## Architecture Overview

```
  User Space
  ┌─────────────────────────────────────────────────────────┐
  │  Container Runtime (containerd / CRI-O)                 │
  │       │                                                  │
  │  Low-Level Runtime (runc)                                │
  │       │                                                  │
  │  ┌────▼─────────────────────────────┐                   │
  │  │  clone3() + unshare()            │                   │
  │  │  pivot_root() + mount()          │                   │
  │  │  seccomp() + capabilities        │                   │
  │  └────┬─────────────────────────────┘                   │
  │       │                                                  │
  ├───────▼──────────────────────────────────────────────────┤
  │  Kernel                                                  │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
  │  │  Namespaces   │  │  Cgroups v2  │  │  LSM/Seccomp │  │
  │  │  PID,Net,Mnt, │  │  cpu,memory, │  │  SELinux,    │  │
  │  │  User,UTS,IPC │  │  io,pids     │  │  AppArmor    │  │
  │  └──────────────┘  └──────────────┘  └──────────────┘  │
  │                                                          │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
  │  │  KVM Module   │  │  Virtio      │  │  VFIO        │  │
  │  │  /dev/kvm     │  │  Virtqueue   │  │  Passthrough │  │
  │  │  VM_ENTRY/EXIT│  │  vhost       │  │  SR-IOV      │  │
  │  └──────────────┘  └──────────────┘  └──────────────┘  │
  │                                                          │
  ├──────────────────────────────────────────────────────────┤
  │  Hardware                                                │
  │  VT-x/AMD-V | EPT/NPT | VT-d/IOMMU | SR-IOV VFs       │
  └──────────────────────────────────────────────────────────┘
```

---

[Start Reading → Chapter 1: Foundations](Chapter_01_Foundations.md)
