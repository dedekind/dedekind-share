# TDX CPUID Virtualization

- **Author**: Artem Bityutskiy
- **Version**: 0.1
- **Date**: 2026-09-10
- **Last updated**: 2026-09-21

**Disclaimer**: This document reflects my current understanding of TDX CPUID virtualization, but it
may contain errors or omissions. It is not an official Intel document and should not be treated as
such. For full and authoritative information, refer to the official documentation.

---

## Table of Contents

- [TDX CPUID Virtualization](#tdx-cpuid-virtualization)
  - [Table of Contents](#table-of-contents)
  - [Context](#context)
  - [Introduction](#introduction)
  - [CPUID Leaves Overview](#cpuid-leaves-overview)
  - [The Three Stages](#the-three-stages)
    - [Stage 1: TDX Module Initialization Time](#stage-1-tdx-module-initialization-time)
    - [Stage 2: TD Build Time](#stage-2-td-build-time)
    - [Stage 3: Runtime Emulation](#stage-3-runtime-emulation)
  - [VE Customization](#ve-customization)
    - [Feature Bits Gated by FEATURE\_PARAVIRT\_CTRL](#feature-bits-gated-by-feature_paravirt_ctrl)
    - [Supervisor/User CPL Controls](#supervisoruser-cpl-controls)
  - [Topology and APIC ID Virtualization](#topology-and-apic-id-virtualization)
    - [CPUID Leaves and Fields Overview](#cpuid-leaves-and-fields-overview)
    - [X2APIC\_IDS](#x2apic_ids)
    - [ENUM\_TOPOLOGY](#enum_topology)
    - [APIC ID Virtualization](#apic-id-virtualization)
    - [Domain Enumeration Virtualization](#domain-enumeration-virtualization)
    - [Other CPUID(0x1) Fields](#other-cpuid0x1-fields)
    - [Summary](#summary)
  - [Cache Parameters](#cache-parameters)
    - [CPUID Leaf Overview](#cpuid-leaf-overview)
    - [Virtualization Details](#virtualization-details)
    - [Note on CPUID Leaf 2](#note-on-cpuid-leaf-2)
  - [Appendix: TD CPUID Configuration](#appendix-td-cpuid-configuration)
  - [Appendix: Virtualization Types](#appendix-virtualization-types)
    - [Stage 1 Types](#stage-1-types)
    - [Stage 2 Types](#stage-2-types)
    - [Stage 3 Types](#stage-3-types)

---

## Context

I once tried to review a patch on the KVM mailing list related to TDX CPUID virtualization. I read
parts of the TDX specifications to understand the patch, but quickly realized how complex TDX CPUID
virtualization is. It combines years of x86 legacy with TDX security requirements. I spent a lot of
time studying the specifications and building a mental model of the subject.

This document summarizes what I learned. It is the document I wish I had read before studying the
TDX specifications. In several places, I use my own terminology instead of the terminology from the
specifications because I find it easier to understand.

The goal is to provide a gradual introduction and help the reader build a mental model. Readability
and flow are more important here than completeness and precision. The official Intel TDX
specifications contain the precise details. Topics that I consider too detailed are moved to the
appendices.

This document assumes that the reader already knows the basic TDX and x86 concepts. It does not
explain them. This is not an official specification, so do not use it as an authoritative reference.
It may contain omissions and inaccuracies.

## Introduction

A `CPUID` instruction executed by a TD always causes a
[VM exit](../misc/intel-cpu/tdx-misc.md#vm-exit-and-td-exit)
into the TDX module, which emulates the instruction and constructs the result from several sources:

- Constants compiled into the TDX module.
- Hardware values sampled during TDX module initialization.
- Values the VMM configured at TD build time.
- Values computed at runtime from vCPU state.
- Values produced by the TD in its `#VE` handler.

---

## CPUID Leaves Overview

The `CPUID` instruction takes a leaf number in `EAX` and, for some leaves, a sub-leaf number in
`ECX`. It returns four 32-bit values in `EAX`, `EBX`, `ECX` and `EDX`.

CPUID leaves are split into two ranges:

- The **base range** starting at `0x0`. `CPUID(0x0).EAX` reports the highest supported base leaf.
- The **extended range** starting at `0x80000000`. `CPUID(0x80000000).EAX` reports the highest
  supported extended leaf.

The following are some of the CPUID leaves:

- `0x0`, **Basic CPUID Information**. Highest supported base leaf and the vendor ID string.
- `0x1`, **Version and Features**. Family, model and stepping, initial APIC ID, and feature flags
  in `ECX` and `EDX`.
- `0x2`, **Cache and TLB Information**. Cache and TLB description, superseded by leaf `0x4`.
- `0x4`, **Cache Parameters**. One sub-leaf per cache level, reporting cache type, level, line size,
  ways, sets, and the number of CPUs sharing the cache.
- `0x7`, **Structured Extended Feature Flags**. Feature flags added after leaf `0x1` ran out of
  bits.
- `0xA`, **Architectural Performance Monitoring**. Perfmon version and counter widths.
- `0xB`, **Extended Topology Enumeration**. Topology hierarchy limited to the SMT and core levels.
  Also reports the x2APIC ID of the current CPU.
- `0xD`, **Processor Extended State Enumeration**. Which XSAVE state components exist, and their
  sizes and offsets in the XSAVE area.
- `0x14`, **Processor Trace**. Intel PT capabilities.
- `0x1A`, **Native Model Information**. The core type of the current CPU on a hybrid SoC.
- `0x1C`, **Architectural LBR Capabilities**. Last Branch Record capabilities.
- `0x1D`, **AMX Tile**. AMX tile palette parameters.
- `0x1E`, **TMUL**. AMX matrix multiply unit parameters.
- `0x1F`, **V2 Extended Topology Enumeration**. The successor to leaf `0xB`, supporting more level
  types, such as module, tile and die.
- `0x21`, **TDX Enumeration**. Not architectural. Identifies the TDX module to the guest.
- `0x23`, **Architectural Perfmon Extensions**. The successor to leaf `0xA`, adding counter
  bitmaps, auto counter reload, and PEBS enumeration.
- `0x40000000` to `0x4FFFFFFF`, **Hypervisor Range**. Not architectural. By convention hypervisors
  use this range to identify themselves and to enumerate their paravirtual interfaces.
- `0x80000000`, **Extended CPUID Information**. Highest supported extended leaf.
- `0x80000008`, **Address Sizes**. The physical and linear address widths. Depending on the TD
  configuration, TDX may also use this leaf to report the GPA width to the TD.

---

## The Three Stages

Virtual CPUID data is produced in three stages.

1. **TDX module initialization time**. When the TDX module initializes, it samples the hardware
   CPUID values, applies compiled-in adjustments, and stores the results in the
   **platform CPUID data table**. The VMM does not provide any input.
2. **TD build time**. When the VMM configures the TD features, the TDX module combines the
   platform CPUID data table from the previous step with the VMM-provided TD configuration and
   creates the **TD CPUID configuration**. It is stored in
   [TDCS](../misc/intel-cpu/tdx-structs.md#tdcs) and cannot be changed afterwards. In most cases, the
   virtual CPUID value is served directly from this frozen configuration.
3. **Runtime emulation**. In some cases the TDX module computes the value fresh from vCPU state, or
   injects a `#VE` and lets the TD produce the virtual CPUID value itself.

### Stage 1: TDX Module Initialization Time

The TDX module initialization protocol involves three seamcalls, which the VMM calls in this order.
Each seamcall contributes to building the platform CPUID data table:

- `TDH.SYS.INIT`: called once. Executes `CPUID` for various leaves on the host and stores the
  results in the platform CPUID data table.
- `TDH.SYS.LP.INIT`: called on every CPU. Executes `CPUID` on its own CPU and verifies that the
  per-CPU values are consistent with values sampled in the previous step.
- `TDH.SYS.CONFIG`: called once. Overwrites some of the sampled values in the platform CPUID data
  table with constants compiled into the TDX module.

The table is built once, and it is the input for computing the TD CPUID configuration of every TD
created afterwards. This is the only stage that reads `CPUID` from the hardware.

### Stage 2: TD Build Time

The job of Stage 2 is to "personalize" the platform CPUID data for a specific TD. This happens in
two steps.

1. The main bulk of it happens when the VMM initializes the TD by invoking `TDH.MNG.INIT`. The VMM
   supplies the seamcall with the [TD_PARAMS](../misc/intel-cpu/tdx-structs.md#td_params)
   structure, which contains most of the TD configuration. This includes the
   [CPUID_CONFIG](../misc/intel-cpu/tdx-structs.md#cpuid_config) array, plus
   [ATTRIBUTES](../misc/intel-cpu/tdx-structs.md#attributes),
   [XFAM](../misc/intel-cpu/tdx-structs.md#xfam)
   and [CONFIG_FLAGS](../misc/intel-cpu/tdx-structs.md#config_flags).
   The TDX module applies all of these configuration values on top of the platform CPUID data table,
   producing the TD CPUID configuration, which it stores in
   [TDCS](../misc/intel-cpu/tdx-structs.md#tdcs).
2. The rest happens when the VMM calls `TDH.VP.INIT` for each vCPU. It provides the virtual x2APIC
   ID of the vCPU, which the TDX module stores in TDCS and uses for initial APIC ID
   (`CPUID(0x1).EBX[31:24]`) virtualization later.

Both `TDH.MNG.INIT` and `TDH.VP.INIT` run only once, per TD and per vCPU respectively. Once they
complete, the TD CPUID configuration is final.

TD CPUID configuration is not a term used by the TDX module specifications. This document uses it to
collectively refer to a handful of TDCS fields.  None of these fields are directly visible to the
TD, but some of the fields can be read by the VMM via the `TDH.MNG.RD` seamcall, for example
`TDCS.CPUID_VALUES`. The [Appendix: TD CPUID Configuration](#appendix-td-cpuid-configuration)
section provides more details on the various fields that make up the TD CPUID configuration.

### Stage 3: Runtime Emulation

Most of the CPUID virtualization is done based on the TD CPUID configuration established and
finalized during Stage 2. Stage 3 adds something on top:

- **Dynamic**. A value computed fresh from vCPU state, never stored anywhere in the TD CPUID
  configuration. For example, OSXSAVE (`CPUID(0x1).ECX[27]`) reports whether `CR4.OSXSAVE` is set.
- **#VE**. No value is returned at all. The TDX module injects a `#VE` and the TD produces the
  value itself, optionally invoking a tdcall to ask the VMM for help. For example, depending on
  configuration, the TDX module may inject a `#VE` for the Frequency (`CPUID(0x16)`) and Processor
  Brand String (`CPUID(0x80000002)`) leaves.
- **Custom**. A handful of CPUID leaves use custom virtualization rules that do not fit either
  pattern above cleanly. Some of them are covered separately in the sections that follow.

**Note**: `CR4.OSXSAVE` enables the `XSAVE` family of CPU instructions, and the OS can flip it at
any time, so this requires dynamic emulation. Accessing the `CR4` register requires CPL 0, so this
CPUID bit is essentially a way for unprivileged code to determine whether the `XSAVE` family of
instructions is enabled. Real hardware implements this CPUID bit the same way.

**Linux Note**: Here is how the TDX guest Linux kernel handles CPUID virtualization in its `#VE`
handler as of version 7.3.

- Return zeros for any CPUID leaf outside the hypervisor range (`0x40000000`–`0x4FFFFFFF`),
  matching real hardware's behavior for an unsupported CPUID leaf.
- For hypervisor-range CPUID leaves, invoke `TDG.VP.VMCALL`, which causes a
  [TD exit](../misc/intel-cpu/tdx-misc.md#vm-exit-and-td-exit).
  The VMM responds with the result in `EAX`/`EBX`/`ECX`/`EDX`. The `#VE` handler places them into
  the TD's registers so that the TD code that executed the original `CPUID` instruction sees this
  result.

---

## VE Customization

This section is complementary to the [Stage 3: Runtime Emulation](#stage-3-runtime-emulation)
section, expanding on how the VMM or the TD can control whether access to a CPUID leaf causes a
`#VE` or gets a virtualized result from the TDX module.

A TD can customize how much CPUID virtualization work the TDX module does, versus how much falls
back to a `#VE`, using three related [TD_CTLS](../misc/intel-cpu/tdx-structs.md#td_ctls) controls,
toggled via the `TDG.VM.WR` tdcall.

- `TD_CTLS.REDUCE_VE`. The main, bulk control. When set, TDX module virtualizes many CPUID leaves
  that would otherwise cause a `#VE`. For example, the Frequency CPUID leaf (`CPUID(0x16)`) becomes
  all zeros, and the Processor Brand String CPUID leaf (`CPUID(0x80000002)`) becomes "Intel TDX".
- `TD_CTLS.ENUM_TOPOLOGY`. Controls virtualization of the topology CPUID leaves, `CPUID(0xB)` and
  `CPUID(0x1F)`, as well as the initial APIC ID field (`CPUID(0x1).EBX[31:24]`). This will be
  covered in a section below.
- `TD_CTLS.VIRT_CPUID2`. Controls virtualization of the Cache and TLB Information CPUID leaf
  (`CPUID(0x2)`).

Setting `TD_CTLS.REDUCE_VE` automatically sets the other two controls as well. But
`TD_CTLS.ENUM_TOPOLOGY` and `TD_CTLS.VIRT_CPUID2` can also be set independently, without
`TD_CTLS.REDUCE_VE`.

**Linux Note**: As of version 7.3, the TDX guest Linux kernel sets `TD_CTLS.REDUCE_VE`
unconditionally at boot.

### Feature Bits Gated by FEATURE_PARAVIRT_CTRL

`TD_CTLS.REDUCE_VE` also gates a separate, finer-grained TD-writable control,
[`TDCS.FEATURE_PARAVIRT_CTRL`](../misc/intel-cpu/tdx-structs.md#feature_paravirt_ctrl), which
only takes effect once `TD_CTLS.REDUCE_VE` is set. `TDCS.FEATURE_PARAVIRT_CTRL` in turn controls
12 individual feature bits, spread across `CPUID(0x1)` and `CPUID(0x7, 0)`. By default, all 12 bits
report as unsupported once `TD_CTLS.REDUCE_VE` is set, but setting the matching bit in
`TDCS.FEATURE_PARAVIRT_CTRL` restores that one feature bit to the value it would have without
`TD_CTLS.REDUCE_VE`.

For simplicity, this document assumes `TDCS.FEATURE_PARAVIRT_CTRL` is not used, so those 12 bits
behave like the rest of the CPUID leaf.

**Linux Note**: As of version 7.3, the Linux kernel does not use `TDCS.FEATURE_PARAVIRT_CTRL`.

### Supervisor/User CPL Controls

Here are three more controls that affect CPUID leaves virtualization:

- `TDVPS.CPUID_SUPERVISOR_VE` and `TDVPS.CPUID_USER_VE`. Per-vCPU controls that let the TD force an
  unconditional `#VE` on `CPUID` execution at a given CPL, for all CPUID leaves.
- `TDVPS.CPUID_CONTROL`. A more flexible variation of the above two controls, letting the TD select
  supervisor/user CPL `#VE` for each CPUID leaf individually.

All three controls take priority over `TD_CTLS.REDUCE_VE`: when set, they force a `#VE` even if
`TD_CTLS.REDUCE_VE` would otherwise virtualize the CPUID leaf. For simplicity, the rest of this
document assumes these controls are disabled.

**Linux Note**: As of version 7.3, Linux kernel does not use the supervisor/user CPL `#VE` controls.

---

## Topology and APIC ID Virtualization

This section covers virtualization of three CPUID leaves that provide APIC/x2APIC ID and CPU
topology information. They fall into the "Custom" category mentioned in the
[Stage 3: Runtime Emulation](#stage-3-runtime-emulation) section.

### CPUID Leaves and Fields Overview

**Initial APIC ID** (`CPUID(0x1).EBX[31:24]`) reports the 8-bit ID assigned to the local APIC at
power-up. However, on modern platforms `CPUID(0xB)` or `CPUID(0x1F)` supersede it with the 32-bit
x2APIC ID, where low 8 bits match this original 8-bit APIC ID.

**Extended Topology Enumeration** (`CPUID(0xB)`) reports a hierarchical view of the system topology
using sub-leaves, one per domain:

- Sub-leaf 0: Logical Processor domain (the SMT siblings within a core).
- Sub-leaf 1: Core domain.
- Sub-leaf 2 and beyond: Invalid, terminating the enumeration (Domain Type 0 in `ECX[15:8]`).

Each sub-leaf reports:

- Shift Count (`EAX[4:0]`): how many bits to shift an x2APIC ID right to reach the next
  higher-scoped domain.
- Count of Logical Processors at This Level (`EBX[15:0]`).
- Level Number (`ECX[7:0]`).
- Domain Type (`ECX[15:8]`): 0=Invalid, 1=Logical Processor, 2=Core.
- x2APIC ID (`EDX[31:0]`).

**Extended Topology Enumeration v2** (`CPUID(0x1F)`) is the successor to `CPUID(0xB)`, using the
same fields, but supporting more domain types: Logical Processor, Core, Module, Tile, Die, Die
Group.

**Linux Note**: Linux uses the APIC ID to identify logical processors within the system, although
it uses the ACPI MADT table instead of the CPUID leaves to get logical processor numbers. These
are the same numbers the CPUID leaves would report. Linux uses CPUID leaves `0xB` or `0x1F` to get
additional topology information, for example the number of SMT siblings within a core, or the
number of cores within a module.

### X2APIC_IDS

Virtualization of the topology and APIC ID fields of the three CPUID leaves uses different rules,
so they will be considered separately. However, before delving into that, let's introduce a
couple of concepts: `TDCS.X2APIC_IDS` and `TD_CTLS.ENUM_TOPOLOGY`.

`TDCS.X2APIC_IDS` is an array of 32-bit x2APIC IDs stored in
[TDCS](../misc/intel-cpu/tdx-structs.md#tdcs). The VMM can read it via `TDH.MNG.RD`. The TDX module
builds this array when the VMM initializes the vCPUs with `TDH.VP.INIT`.

- `TDH.VP.INIT` version 1 or higher provides the virtual x2APIC ID for each vCPU, and the TDX
  module stores it in `TDCS.X2APIC_IDS[vcpu]`.
- `TDH.VP.INIT` version 0 does not provide a virtual x2APIC ID, so `TDCS.X2APIC_IDS` stays
  unpopulated.

When the VMM uses `TDH.VP.INIT` version 0, the TDX module also clears
`TDCS.TOPOLOGY_ENUM_CONFIGURED`. Otherwise the flag stays set. The TD can read this flag with the
`TDG.VM.RD` tdcall.

**Note**: In TDX specs terminology, `vcpu` above is
[TDVPS.VCPU_INDEX](../misc/intel-cpu/tdx-structs.md#tdvps),
the unique number of the vCPU within the TD.

**Linux Note**: As of version 7.3, the Linux kernel uses `TDH.VP.INIT` version 1, supplying the
vCPU ID as the virtual x2APIC ID for every vCPU.

### ENUM_TOPOLOGY

`TD_CTLS.ENUM_TOPOLOGY` is a control that the TD can set via the `TDG.VM.WR` tdcall, asking the
TDX module to virtualize the CPUID topology leaves `0xB` and `0x1F`, instead of injecting a `#VE`
for them. The TD can set `TD_CTLS.ENUM_TOPOLOGY` only if `TDCS.TOPOLOGY_ENUM_CONFIGURED` is set. As
covered in the [VE Customization](#ve-customization) section above, setting `TD_CTLS.REDUCE_VE`
also sets `TD_CTLS.ENUM_TOPOLOGY`.

### APIC ID Virtualization

If `TD_CTLS.ENUM_TOPOLOGY` is set, APIC ID is virtualized as follows.

- Initial APIC ID (`CPUID(0x1).EBX[31:24]`) comes from the low 8 bits of `TDCS.X2APIC_IDS[vcpu]`.
- x2APIC ID in both of the Extended Topology Enumeration leaves (`CPUID(0xB)` and `CPUID(0x1F)`)
  comes from `TDCS.X2APIC_IDS[vcpu]`.

If `TD_CTLS.ENUM_TOPOLOGY` is not set:

- Initial APIC ID (`CPUID(0x1).EBX[31:24]`) comes from the low 8 bits of the vCPU index
  (`TDVPS.VCPU_INDEX`).
- Both Extended Topology Enumeration leaves (`CPUID(0xB)` and `CPUID(0x1F)`) cause a `#VE`.

### Domain Enumeration Virtualization

When `TD_CTLS.ENUM_TOPOLOGY` is not set, the two Extended Topology Enumeration CPUID leaves cause a
`#VE`. Here is what happens when `TD_CTLS.ENUM_TOPOLOGY` is set.

- All sub-leaves of the Extended Topology Enumeration v2 (`CPUID(0x1F)`) come from the TD CPUID
  configuration, which is built at [Stage 2](#stage-2-td-build-time). The values come from what the
  VMM supplied in [CPUID_CONFIG](../misc/intel-cpu/tdx-structs.md#cpuid_config) via
  [TD_PARAMS](../misc/intel-cpu/tdx-structs.md#td_params), or, if the VMM leaves `CPUID(0x1F)`
  unconfigured (all-zero), from the platform CPUID data table sampled at [Stage
  1](#stage-1-tdx-module-initialization-time).
- Extended Topology Enumeration (`CPUID(0xB)`).
  - Sub-leaves 0, 1 and 2 come from the TD CPUID configuration, which is built at
    [Stage 2](#stage-2-td-build-time), derived from the `CPUID(0x1F)` sub-leaf values described
    above. This includes sub-leaf 2, the one that terminates the enumeration by reporting the
    "Invalid" domain type.
  - `CPUID(0xB)` has no `CPUID_CONFIG` entry of its own. Instead, when the VMM calls
    `TDH.MNG.INIT`, the TDX module derives sub-leaves 0, 1 and 2 from `CPUID(0x1F)` and stores the
    result in the TD CPUID configuration.
  - Interestingly, sub-leaf 3 and beyond are gated by `TD_CTLS.REDUCE_VE`, not by
    `TD_CTLS.ENUM_TOPOLOGY` like sub-leaves 0, 1 and 2 are. So a TD that sets
    `TD_CTLS.ENUM_TOPOLOGY` without `TD_CTLS.REDUCE_VE` gets a `#VE` on sub-leaf 3 and beyond, even
    though sub-leaves 0, 1 and 2 are virtualized by the TDX module.

### Other CPUID(0x1) Fields

Just for completeness, here is a summary for the fields of the Version and Features CPUID leaf
(`CPUID(0x1)`) other than the Initial APIC ID. Unlike the Initial APIC ID, their virtualization does
not depend on `TD_CTLS.ENUM_TOPOLOGY`.

- The Family/Model/Stepping and most of the feature flag fields come from the TD CPUID
  configuration, the normal pattern covered in the
  [Stage 2: TD Build Time](#stage-2-td-build-time) section.
- The OSXSAVE feature flag (`CPUID(0x1).ECX[27]`) is computed at runtime from `CR4.OSXSAVE`, as
  covered in the [Stage 3: Runtime Emulation](#stage-3-runtime-emulation) section.

### Summary

Version and Features (`CPUID(0x1)`):

| `TD_CTLS.ENUM_TOPOLOGY` | APIC ID                               | Other fields |
|-------------------------|---------------------------------------|--------------|
| Set                     | Low 8 bits of `TDCS.X2APIC_IDS[vcpu]` | Unaffected   |
| Unset                   | Low 8 bits of `TDVPS.VCPU_INDEX`      | Unaffected   |

Extended Topology Enumeration v2 (`CPUID(0x1F)`):

| `TD_CTLS.ENUM_TOPOLOGY` | x2APIC ID               | Domain Enumeration Fields |
|-------------------------|-------------------------|---------------------------|
| Set                     | `TDCS.X2APIC_IDS[vcpu]` | TD CPUID configuration    |
| Unset                   | `#VE`                   | `#VE`                     |

Extended Topology Enumeration (`CPUID(0xB)`):

| `TD_CTLS.ENUM_TOPOLOGY` | x2APIC ID               | Domain Enumeration Fields |
|-------------------------|-------------------------|---------------------------|
| Set                     | `TDCS.X2APIC_IDS[vcpu]` | TD CPUID configuration    |
| Unset                   | `#VE`                   | `#VE`                     |

---

## Cache Parameters

This section covers the **Cache Parameters** leaf (`CPUID(0x4)`), which falls under the "Custom"
category mentioned in the [Stage 3: Runtime Emulation](#stage-3-runtime-emulation) section.

### CPUID Leaf Overview

Here is what the Cache Parameters CPUID leaf (`CPUID(0x4)`) reports:

- One sub-leaf per cache level (e.g., Data L1, Instruction L1, L2, LLC).
- Cache Type (`EAX[4:0]`).
- Cache Level (`EAX[7:5]`).
- Self Initializing Flag (`EAX[8]`).
- Fully Associative Flag (`EAX[9]`).
- Number of Addressable IDs Sharing This Cache (`EAX[25:14]`).
- Number of Addressable IDs for Cores in the Package (`EAX[31:26]`).
- Line Size (`EBX[11:0]`).
- Physical Line Partitions (`EBX[21:12]`).
- Ways of Associativity (`EBX[31:22]`).
- Number of Sets (`ECX[31:0]`).
- Flags (`EDX`).

### Virtualization Details

`CPUID(0x4)` never causes a `#VE`. Sub-leaf 4 and beyond are always reported as an all-zero
terminating entry (Cache Type Invalid).  This means a TD can see at most 4 cache levels, regardless
of how many the host actually has.

If `TD_CTLS.REDUCE_VE` is not set, all fields of sub-leaves 0 to 3 come from the TD CPUID
configuration, the normal pattern.

If `TD_CTLS.REDUCE_VE` is set, sub-leaves 0 to 3 come from `TDCS.CPUID4_NATIVE_VALUES` instead, a
copy of the native platform CPUID data table values ([Stage 1](#stage-1-tdx-module-initialization-time))
copied into TDCS during TD initialization, instead of from the TD CPUID configuration
([Stage 2](#stage-2-td-build-time)). The exception is two `EAX` fields, which keep coming from the
TD CPUID configuration even when `TD_CTLS.REDUCE_VE` is set:

- Number of Addressable IDs Sharing This Cache (`EAX[25:14]`).
- Number of Addressable IDs for Cores in the Package (`EAX[31:26]`).

### Note on CPUID Leaf 2

`CPUID(0x2)` (Cache and TLB Information) also reports cache information and is gated by a dedicated
control, `TD_CTLS.VIRT_CPUID2`. But it fits the same `#VE`-or-fixed-value pattern described in the
[Stage 3: Runtime Emulation](#stage-3-runtime-emulation) section:

- If `TD_CTLS.VIRT_CPUID2` is not set, `CPUID(0x2)` triggers a `#VE`.
- If `TD_CTLS.VIRT_CPUID2` is set, `CPUID(0x2)` returns a fixed value defined by the TDX module.
  `CPUID(0x2)` has no `CPUID_CONFIG` entry of its own.

---

## Appendix: TD CPUID Configuration

This appendix lists some of the [TDCS](../misc/intel-cpu/tdx-structs.md#tdcs) fields that make up the
TD CPUID configuration. None of these fields are directly visible to the TD.  Its only way to
observe them is by executing the `CPUID` instruction. But some of the fields are visible to the VMM
via the `TDH.MNG.RD` seamcall.

The main part of the TD CPUID configuration is `TDCS.CPUID_VALUES`, an array of 128-bit entries, one
for each virtualized CPUID leaf and sub-leaf combination. Each entry packs the virtual `EAX`, `EBX`,
`ECX` and `EDX`, and the VMM can read the whole array via `TDH.MNG.RD`. The rest of the TDCS fields
listed below serve purposes such as:

- Caching information already derivable from `CPUID_VALUES` for faster lookup.
- Flagging which `CPUID_VALUES` entries are valid.
- Storing an alternative value for a leaf or sub-leaf that already has one in `CPUID_VALUES`, for
  cases where the TD itself can influence the virtualized value it receives.

Here are some examples, not an exhaustive list:

- `CPUID_VALID` (validity flag): not readable by the VMM. A boolean array with one flag per
  `CPUID_VALUES` entry, marking whether the entry is valid.
- `CPUID_FIXED0_BITMAP` (caching): readable by the VMM. A 64-bit bitmap, one bit per CPUID leaf,
  marking the leaves that return all zeros.
- `CPUID4_NATIVE_VALUES` (alternative value): not readable by the VMM. An array of 4 128-bit
  entries, a copy of sub-leaves 0 to 3 of the native values of the Cache Parameters leaf
  (`CPUID(0x4)`) from the platform CPUID data table. `CPUID_VALUES` is the source for the whole
  `CPUID(0x4)` leaf, every sub-leaf, like for any other leaf, whenever
  `TD_CTLS.REDUCE_VE` is 0. When the guest TD sets `REDUCE_VE`, that stays true for sub-leaf 4 and
  above, but for sub-leaves 0 to 3 the TDX module instead returns this native copy, discarding the
  values `CPUID_VALUES` holds for that sub-leaf, for most of the sub-leaf's fields.
- `CPUID_LAST_BASE_LEAF` and `CPUID_LAST_EXT_LEAF` (caching): not readable by the VMM. Two 32-bit
  virtual values for the highest supported base leaf (`CPUID(0x0).EAX`) and the highest supported
  extended leaf (`CPUID(0x80000000).EAX`). They help reject out-of-range leaves quickly.
- `CPUID_FLAGS` (caching): not readable by the VMM. Boolean values derived from select virtual
  CPUID bits, kept for fast lookup during CPUID emulation instead of searching `CPUID_VALUES`.

---

## Appendix: Virtualization Types

Every virtual CPUID bit field has a **virtualization type**, which says where its value comes from.
This is a categorization the TDX specifications use to explain the logic behind CPUID field
virtualization. It is not exposed through any ABI, e.g., no seamcall lets the VMM query it.

This document simplifies the specifications' virtualization types and groups them by the stage
that introduces them: Stage 1, Stage 2, and Stage 3. Some types are omitted in this document for
simplicity. For full details, refer to the TDX module specifications.

### Stage 1 Types

Here are the virtualization types assigned by Stage 1.

- **Native**. The sampled hardware value, stored as-is. This is most of the platform CPUID data
  table.
- **Assigned**. A value the TDX module computes during initialization. The specifications
  currently document only one such field: the stepping (`CPUID(0x1).EAX[3:0]`), set to the minimum
  across all packages.
- **Fixed**. A value the TDX module guarantees to the guest, regardless of the TD configuration.
  - **Fixed-0**: Bits forced to 0, used for reserved bits and to hide a feature from every TD. For
    example, the VMX bit (`CPUID(0x1).ECX[5]`) is fixed to 0 because a TD may not execute VMX
    instructions.
  - **Fixed-1**: Bits forced to 1, rarely used, for example the non-architectural bit
    `CPUID(0x1).ECX[31]`. It is reserved and reads as 0 on a physical CPU. Hypervisors set it by
    convention to tell the guest that it runs in a VM.
  - **Fixed, verified**: A value read from the hardware and then checked against a fixed constant.
    For example, the vendor ID string (`CPUID(0x0).EBX`/`ECX`/`EDX`) is checked to be equal to
    "GenuineIntel". If the check fails, TDX module initialization fails.

| This document   | TDX module specifications |
|-----------------|---------------------------|
| Native          | `NATIVE`                  |
| Assigned        | `ASSIGNED`                |
| Fixed           | `FIXED`                   |
| Fixed, verified | `FIXED_AS_VERIFIED`       |

**Notes**

- Fixed-0 and Fixed-1 are not separate types in TDX specifications, both map to the same `FIXED`
  type. Instead, TDX specifications just mention the value separately from the type.
- There are no virtual CPUID fields with the Native type. But it does not mean sampled CPUID field
  values are never seen by the TD. Several virtualization types that will be introduced in the next
  sub-section, such as "Allowed" or "CPUID gated", do serve the actual sampled CPUID field value
  as-is, whenever the gating condition allows it. The Native type is reserved for the cases without
  any gating condition at all, and there are no such fields in practice.

### Stage 2 Types

Stage 2 adds more virtualization types for CPUID fields on top of the ones Stage 1 established.

- **Configured**. The CPUID field value was provided by the VMM at TD build time, either directly
  via the [CPUID_CONFIG](../misc/intel-cpu/tdx-structs.md#cpuid_config), or indirectly via other
  [TD_PARAMS](../misc/intel-cpu/tdx-structs.md#td_params) fields.
  - Example 1: Logical Processors at this Level (`CPUID(0x1F).EBX[15:0]`), the count of logical
    processors contained in a topology level, is configured directly via `CPUID_CONFIG`.
  - Example 2: MAXGPA (the guest physical address width, `CPUID(0x80000008).EAX[31:16]`) is
    configured indirectly, via [CONFIG_FLAGS](../misc/intel-cpu/tdx-structs.md#config_flags).
- **Allowed**. Unlike a Configured field, the VMM does not pick the value itself, it only decides
  whether to expose it. The value comes from the sampled platform CPUID data, but the decision to
  expose it is expressed by enabling or disabling that same bit in `CPUID_CONFIG`.
  - Example: the MONITOR bit (`CPUID(0x1).ECX[3]`) has the "Allowed" virtualization type. If
    enabled, the sampled platform value is exposed, otherwise 0.
- **CPUID gated**. The same idea as Allowed, except the gate bit is not the same as the gated bit:
  the value also comes from the sampled platform CPUID data, or 0 if the gate bit is 0.
  - Example: the Monitor/Mwait CPUID leaf (`CPUID(0x5)`) has the "CPUID gated" virtualization type,
    and its gate is the MONITOR feature bit (`CPUID(0x1).ECX[3]`).
- **ATTRIBUTES gated**. The same idea as CPUID gated, except the gate bit is in
  [ATTRIBUTES](../misc/intel-cpu/tdx-structs.md#attributes) instead of another CPUID field. The value
  is the sampled platform data if the gate bit is 1, otherwise 0.
  - Example: PKS (Protection Keys for Supervisor-mode pages, `CPUID(0x7,0).ECX[31]`) has the
    "ATTRIBUTES gated" virtualization type, gated by `ATTRIBUTES.PKS`.
- **XFAM gated**. The same idea again, but the gate bit is in
  [XFAM](../misc/intel-cpu/tdx-structs.md#xfam).
  - Example: FMA (`CPUID(0x1).ECX[12]`) has the "XFAM gated" virtualization type, gated by `XFAM`'s
    AVX state-component bit (`XFAM[2]`).
- **Special**. A value computed by a rule written specifically for that field.
  - Example: MAXPA (the physical address width, called MAXPHYADDR in the Intel SDM,
    `CPUID(0x80000008).EAX[7:0]`) defaults to the sampled platform value, but the VMM can enable
    `CONFIG_FLAGS.MAXPA_VIRT` to configure a custom value instead.

Virtualization types compose.

- **Allowed, ATTRIBUTES-gated**. An "Allowed" field additionally gated by `ATTRIBUTES`. The value
  comes from the sampled platform CPUID data if both the corresponding `CPUID_CONFIG` bit is set and
  the corresponding `ATTRIBUTES` bit is set.
  - Example: the "Perfmon Extended Leaf Supported" bit (`CPUID(0x7,1).EAX[8]`) is gated by
    `ATTRIBUTES.PERFMON`: the platform value reaches the TD only if the VMM both sets the attribute
    and enables that same CPUID bit in `CPUID_CONFIG`.
- **Allowed, XFAM-gated**. The same idea for `XFAM`.
  - Example: AVX (`CPUID(0x1).ECX[28]`) is gated by both `XFAM`'s AVX state-component bit
    (`XFAM[2]`) and the same CPUID bit in `CPUID_CONFIG`.
- **CPUID gated, ATTRIBUTES gated**. Combines "CPUID gated" and "ATTRIBUTES gated": the value comes
  from the sampled platform CPUID data only if both gate bits are 1, one in another CPUID field and
  one in `ATTRIBUTES`.
  - Example: the Perfmon feature bits (`CPUID(0x23,0).EBX[31:2]`) are gated by both
    `ATTRIBUTES.PERFMON` and the Perfmon Extended Leaf Supported bit (`CPUID(0x7,1).EAX[8]`).
- **CPUID gated, XFAM gated**. Combines "CPUID gated" and "XFAM gated": the value comes from the
  sampled platform CPUID data only if both gate bits are 1, one in another CPUID field and one in
  `XFAM`.
  - Example: the 128-bit vector support bit (`CPUID(0x24,0).EBX[16]`) is gated by both `XFAM`'s
    AVX-512 state bits (`XFAM[7:5]`) and the Converged Vector ISA support bit
    (`CPUID(0x7,1).EDX[19]`).

Composition can go further:

- **Allowed, CPUID gated, XFAM gated**. For example, AVX10_V1_AUX (`CPUID(0x24,1).ECX[2]`).
- **Allowed, CPUID gated, ATTRIBUTES gated**. For example, the Valid sub-leaf bitmap
  (`CPUID(0x23,0).EAX[5:0]`).

| This document                          | TDX module specifications           |
|----------------------------------------|-------------------------------------|
| Configured                             | `CONFIG_DIRECT`, `CONFIG_TD_PARAMS` |
| Allowed                                | `ALLOW_DIRECT`                      |
| CPUID gated                            | `ALLOW_CPUID`                       |
| ATTRIBUTES gated                       | `ALLOW_ATTRIBUTES`                  |
| XFAM gated                             | `ALLOW_XFAM`                        |
| Special                                | `SPECIAL`, `SPECIAL_DIRECT`         |
| CPUID gated, ATTRIBUTES gated          | `ALLOW_ATTRIBUTES_CPUID`            |
| CPUID gated, XFAM gated                | `ALLOW_XFAM_CPUID`                  |
| Allowed, ATTRIBUTES-gated              | `ALLOW_ATTRIBUTES_DIRECT`           |
| Allowed, XFAM-gated                    | `ALLOW_XFAM_DIRECT`                 |
| Allowed, CPUID gated, XFAM gated       | `ALLOW_XFAM_CPUID_DIRECT`           |
| Allowed, CPUID gated, ATTRIBUTES gated | `ALLOW_ATTRIBUTES_CPUID_DIRECT`     |

### Stage 3 Types

Stage 3 adds two virtualization types.

- **Dynamic**. A value computed fresh from vCPU state, no part of it was derived from the TD CPUID
  configuration.
  - Example: OSXSAVE (`CPUID(0x1).ECX[27]`) reports whether `CR4.OSXSAVE` is set.
- **#VE**. No value is returned at all. The TDX module injects a `#VE` and the TD produces the
  value itself.
  - Example: depending on configuration, the TDX module may inject a `#VE` for the Frequency
    (`CPUID(0x16)`) and Processor Brand String (`CPUID(0x80000002)`) leaves.

| This document | TDX module specifications |
|---------------|---------------------------|
| Dynamic       | `DYNAMIC`                 |
| `#VE`         | `#VE`                     |
