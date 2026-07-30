<!--
-*- coding: utf-8 -*-
vim: ts=4 sw=4 tw=100 et ai si

# Copyright (C) 2024-2026 Intel Corporation
# SPDX-License-Identifier: BSD-3-Clause

Author: Artem Bityutskiy <artem.bityutskiy@linux.intel.com>
-->

# KVM Dirty Page Tracking

- **Author**: Artem Bityutskiy
- **Version**: 1.0
- **Date**: 2026-06-10
- **Last updated**: 2026-07-30

**Disclaimer**: This document describes my understanding of KVM dirty page tracking API and
implementation, not a comprehensive reference. At the time of writing (June 2026), I had no KVM or
virtualization experience. This document contains my study notes. It is not official KVM
documentation and is not endorsed by the KVM community. The document may contain errors or
omissions. It may be expanded over time, and the version number will be updated accordingly.

---

## Table of Contents

- [KVM Dirty Page Tracking](#kvm-dirty-page-tracking)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [How It Works](#how-it-works)
    - [EPT Background](#ept-background)
    - [Tracking Mechanisms](#tracking-mechanisms)
      - [PML Details](#pml-details)
    - [Feature Detection and Control](#feature-detection-and-control)
  - [API](#api)
    - [KVM File Descriptors](#kvm-file-descriptors)
    - [Switching Dirty Tracking on and off](#switching-dirty-tracking-on-and-off)
    - [The Bitmap Interface](#the-bitmap-interface)
    - [Dirty Ring Interface](#dirty-ring-interface)
      - [ACQ\_REL Capability](#acq_rel-capability)
    - [Hybrid Mode](#hybrid-mode)

---

## Introduction

I wrote this document while learning QEMU/KVM live migration, where dirty page tracking is a key
part. This document describes KVM dirty page tracking in general terms, but in practice my
research focused on how QEMU uses KVM on x86.

KVM (Kernel-based Virtual Machine) provides dirty page tracking, which identifies guest memory
pages modified since a point in time. During live migration, userspace (for example, QEMU) enables
tracking, periodically gets dirty-page information from KVM, transfers those pages, and re-arms
tracking for the next round. For more information on QEMU live migration, see
[QEMU live migration](./qemu-live-migration.md).

KVM has two types of dirty page tracking:

- **Synchronous**: KVM traps every guest write to a tracked page and handles it before the guest
  can proceed. This is used internally by KVM to maintain its own data structures, for example to
  keep shadow page tables in sync with guest page tables. It is not exposed to userspace.
- **Asynchronous**: KVM logs dirty pages in the background without necessarily stopping the guest
  on every write. Userspace retrieves the accumulated dirty page information later. This is used
  for live migration, live snapshots, and similar use cases.

This document covers only asynchronous dirty page tracking. The KVM API is easier to understand
with some knowledge of how tracking works under the hood, so this document starts with a
high-level overview of the mechanisms, then covers the API.

## How It Works

The KVM dirty page tracking API works on different CPU architectures, but the underlying mechanisms
are architecture-specific. This section gives a high-level overview of how dirty page tracking
works on Intel processors, which use EPT (Extended Page Tables) for this purpose.

**Note**: EPT can be disabled via the `kvm_intel.ept=0` module parameter. In that case, KVM uses
shadow page tables and dirty tracking works by write-protecting shadow page table entries and
trapping guest writes. Modern Intel processors support EPT, and KVM enables it by default. This
document assumes EPT is enabled.

### EPT Background

EPT (Extended Page Tables) translates guest physical addresses (GPA) to host physical addresses
(HPA). EPT structure is similar to regular page tables and typically has 4 levels:

- PML4 (Page Map Level 4)
- PDPT (Page Directory Pointer Table)
- PD (Page Directory)
- PT (Page Table)

**Note**: Newer Intel processors support a 5th level, but it is only relevant for GPA spaces
larger than 256 TiB and is not common in practice.

EPT entries have page access permissions (`R`, `W`, `X`). Newer Intel processors also support
accessed and dirty bits (`A`, `D`). These flags are independent of the guest page table flags.

Like regular page tables, the processor walks EPT automatically on guest memory accesses. The full
translation path is:

- GVA -> GPA via guest page tables
- GPA -> HPA via EPT

Guest page table faults (#PF) are handled by the guest OS. EPT violations trigger VM exits handled
by KVM. One common cause relevant to dirty tracking is a missing write permission in an EPT entry:
the guest write triggers a VM exit, and KVM handles it.

KVM dirty page tracking uses EPT write permissions and dirty bits, as described next.

### Tracking Mechanisms

KVM dirty page tracking uses different mechanisms depending on the available CPU features.

- **WP-based tracking**: "WP" stands for "write-protect". KVM clears write permission in EPT
  entries for guest pages. On the first write to such a page, an EPT violation triggers a VM exit,
  and KVM records the page as dirty.
- **PML-based tracking**: requires EPT A/D and PML to be enabled. PML stands for "Page Modification
  Logging". Thanks to EPT A/D, the CPU automatically sets the dirty bit in EPT entries. Thanks to
  PML, the CPU logs page modifications in the PML buffer. KVM consumes this buffer on VM exits and
  records the pages as dirty. Page write protection is not used.

In terms of overhead:

- WP-based tracking is generally more expensive because it causes a VM exit every time the guest
  first writes to a write-protected page.
- PML-based tracking is generally less expensive because it avoids first-write VM exits.

**Note**: It is possible that EPT A/D is enabled, but PML is disabled or unavailable. In that case,
KVM still uses WP-based tracking.

#### PML Details

- Each vCPU has its own 4 KiB **PML buffer** (512 entries, 8 bytes each), where it logs dirty page
  GPAs.
- **PML Address**: a VMCS field that contains the HPA of that vCPU's PML buffer.
- **PML Index**: a VMCS field that contains the 16-bit index of the next PML entry.

On each write that changes an EPT dirty bit from 0 to 1, the CPU:

- Writes the page GPA into the PML buffer at the current index.
- Decrements PML index by 1.

PML index fill and reset:

- KVM sets PML index to 511.
- Index decrements toward 0 as the CPU logs entries.
- At index 0: CPU logs one final GPA, index wraps to `FFFFH` (16-bit underflow).
- Next logging attempt: CPU generates a "page-modification log full" VM exit.
- KVM consumes the logged entries, resets PML index to 511, and logging continues.

### Feature Detection and Control

Linux kernel enumerates the following CPU capabilities related to dirty page tracking:

- **EPT A/D support**: MSR 0x48C (`IA32_VMX_EPT_VPID_CAP`), bit 21 (`VMX_EPT_AD_BIT`).
- **PML support**: MSR 0x48B (`IA32_VMX_PROCBASED_CTLS2`), bit 17 (`SECONDARY_EXEC_ENABLE_PML`).

In addition, KVM module parameters control whether KVM enables these features:

- `kvm_intel.eptad`: whether KVM should use EPT A/D bits if the CPU supports them. Default: 1.
- `kvm_intel.pml`: whether KVM should use PML if the CPU supports it. Default: 1. PML also
  requires EPT and EPT A/D to be enabled.

**Note**: EPT A/D without PML does not enable PML-based tracking, but it helps with other KVM
optimizations that are not related to dirty page tracking.

## API

This section covers the KVM userspace API for dirty page tracking.

KVM API uses the following terminology:

- **Page protection**: re-arming dirty tracking for selected pages. In WP-based tracking, this means
  clearing write permission in EPT entries. In PML-based tracking, this means clearing the EPT dirty
  bit so the next write is logged again.
- **Manual dirty-log protect**: userspace explicitly controls when dirty bits are cleared and dirty
  tracking is re-armed for selected pages, instead of this happening automatically in the
  `KVM_GET_DIRTY_LOG` ioctl.

KVM provides two interfaces for tracking guest dirty pages:

- The **bitmap interface** (often called "dirty log" in KVM discussions):
  the original interface, where KVM returns a bitmap of dirty pages via ioctl.
- The **dirty ring interface**, introduced in Linux v5.11 (2020): uses per-vCPU ring buffers
  shared between KVM and userspace.

There is also a hybrid mode that combines both, but it is not available on x86.

### KVM File Descriptors

The KVM API ioctls are called on different file descriptors depending on their scope. KVM uses a
hierarchy of file descriptors, where each level is created from the level above it.

- **System file descriptor**: used for system-wide operations like `KVM_CREATE_VM` and
  `KVM_CHECK_EXTENSION` (KVM capability query). Created by opening the KVM device (`/dev/kvm`).
- **VM file descriptor**: used for VM-wide operations, created by calling `KVM_CREATE_VM` on the
  system descriptor. Examples of operations:
  - `KVM_SET_USER_MEMORY_REGION` or `KVM_SET_USER_MEMORY_REGION2` - create and modify a memslot.
  - `KVM_GET_DIRTY_LOG` - get dirty page bitmap for a memslot.
  - `KVM_ENABLE_CAP` - enable VM-wide capabilities.
  - `KVM_CREATE_VCPU` - create a vCPU within this VM.
- **vCPU file descriptor**: used for vCPU-specific operations, created by calling `KVM_CREATE_VCPU`
  on the VM file descriptor. Example of operations:
  - `KVM_RUN` - execute the vCPU.
  - `KVM_GET_REGS` - get vCPU registers.

### Switching Dirty Tracking on and off

A memslot is a KVM object that maps a contiguous guest physical address range to a host virtual
address range. Each memslot is an independent unit for dirty tracking control.

Userspace enables or disables dirty page tracking by setting or clearing the
`KVM_MEM_LOG_DIRTY_PAGES` flag on a memslot via `KVM_SET_USER_MEMORY_REGION` or
`KVM_SET_USER_MEMORY_REGION2` on the VM file descriptor.

Enabling dirty tracking sets up write-protection or PML as described in
[How It Works](#how-it-works). When dirty tracking is disabled:

- **WP-based tracking**: write-protection is cleared for that memslot.
- **PML-based tracking**: PML is disabled for the entire VM when dirty tracking is disabled for
  all memslots (PML is per-VM, not per-memslot).

In both cases, if huge pages in the memslot were split for tracking, they are merged back.

### The Bitmap Interface

The bitmap interface is based on the `KVM_GET_DIRTY_LOG` ioctl, which returns a bitmap of dirty
pages for a memslot. By default, `KVM_GET_DIRTY_LOG` also clears the dirty bits and re-protects
the pages before returning.

**Note**: QEMU uses the bitmap interface by default, unless the user explicitly requests the
dirty ring interface via command line option.

With manual dirty-log protect enabled, `KVM_GET_DIRTY_LOG` only returns the bitmap without
clearing or re-protecting. Userspace explicitly re-protects pages later via `KVM_CLEAR_DIRTY_LOG`.

To enable manual dirty-log protect, userspace calls `KVM_ENABLE_CAP` with
`KVM_CAP_MANUAL_DIRTY_LOG_PROTECT2` capability. The flags argument is a bitmask:

- `KVM_DIRTY_LOG_MANUAL_PROTECT_ENABLE`: enables manual protect mode.
- `KVM_DIRTY_LOG_INITIALLY_SET`: tells KVM to initialize all dirty bitmap bits to 1 and skip
  initial write protection. Write protection is applied later when userspace calls
  `KVM_CLEAR_DIRTY_LOG`, which can avoid unnecessary initial VM exits. Requires
  `KVM_DIRTY_LOG_MANUAL_PROTECT_ENABLE`.

**Note**: QEMU enables manual dirty-log protect with both flags when available. This reduces VM
exits during live migration by delaying re-protection until pages are about to be sent to the
destination, avoiding unnecessary VM exits for writes to already-known dirty pages.

All ioctls mentioned in this sub-section are called on the VM file descriptor.

### Dirty Ring Interface

To enable this interface, userspace calls `KVM_ENABLE_CAP` on the VM file descriptor, sets the
capability to `KVM_CAP_DIRTY_LOG_RING`, and passes the ring size in bytes.

**Note**: In QEMU, this can be requested with options like
`-accel kvm,dirty-ring-size=4096`. Here `4096` is the number of ring entries per vCPU.

The dirty rings are per-vCPU circular buffers shared between userspace and KVM. To obtain the
dirty ring buffer address for a vCPU, userspace runs `mmap()` on the vCPU file descriptor with an
offset of `KVM_DIRTY_LOG_PAGE_OFFSET * PAGE_SIZE`.

Each ring entry has this format:

```c
struct kvm_dirty_gfn {
  __u32 flags;
  __u32 slot;   /* Lower 16 bits: slot ID, upper 16 bits: address space ID */
  __u64 offset; /* Page index relative to memslot base */
};
```

With WP-based tracking, KVM adds entries one by one. The first write to a write-protected page
causes a VM exit, and KVM adds one `kvm_dirty_gfn` entry for that page.

With PML-based tracking, KVM adds entries in batches. On each vCPU VM exit, KVM flushes that
vCPU's PML buffer to the dirty ring and adds entries for all pages currently logged there.

The `flags` field is used for synchronization between KVM and userspace. Two bits are used:

- Dirty bit, bit 0 (`KVM_DIRTY_GFN_F_DIRTY`): set by KVM when the entry contains dirty page
  information.
- Reset bit, bit 1 (`KVM_DIRTY_GFN_F_RESET`): set by userspace after collecting the entry, so KVM
  can reuse it.

Here is how userspace consumes dirty ring entries:

- Check the dirty bit.
- If the dirty bit is set, consume the entry.
- For a consumed entry, set the reset bit in `flags`.
- Continue until the dirty bit is clear.
- If the dirty bit is clear, stop scanning. Do not set the reset bit on that entry.
- Entries must be consumed in order, and must not be skipped.

Userspace periodically scans all vCPU rings to collect dirty entries.

This is not a traditional producer-consumer ring. After scanning the rings, userspace must call
`KVM_RESET_DIRTY_RINGS` on the VM file descriptor. Only then can KVM reuse collected entries.
In WP-based tracking, this also re-protects the consumed pages.

If userspace does not call `KVM_RESET_DIRTY_RINGS`, KVM can hit the "ring is full" situation.
In this case, `KVM_RUN` returns to userspace with `KVM_EXIT_DIRTY_RING_FULL`. Userspace must reap
and reset the rings before the vCPU can continue running.

#### ACQ_REL Capability

On weakly ordered architectures, such as ARM64 and RISCV, userspace should use
`KVM_CAP_DIRTY_LOG_RING_ACQ_REL` instead of `KVM_CAP_DIRTY_LOG_RING` to enable the dirty ring
interface.

With this capability, userspace must access the entry `flags` field with acquire/release ordering:

- Use load-acquire when checking the dirty bit.
- Use store-release when setting the reset bit.

In practical terms: read the dirty bit with acquire semantics before using entry data, and write
the reset bit with release semantics after processing the entry.

For details, see `Documentation/memory-barriers.txt` in the kernel source tree.

The dirty ring handling algorithm does not change. Only the ordering requirements for `flags`
accesses change.

### Hybrid Mode

The hybrid mode is the `KVM_CAP_DIRTY_LOG_RING_WITH_BITMAP` capability. It combines dirty ring
and bitmap interfaces. KVM does not provide this mode on x86. As of this writing, it is
provided on arm64.

KVM uses dirty rings when it can attribute a write to a running vCPU. If a page is dirtied outside
vCPU dirty-ring context, the page is tracked in the bitmap instead. In this mode, the bitmap is only
for pages dirtied in non-vCPU context.

The general API idea is that some architectures may dirty guest pages without running-vCPU context,
for example via DMA or other non-vCPU paths. In today's Linux tree, the concrete use case is
state save code that can dirty guest memory without a running vCPU.
