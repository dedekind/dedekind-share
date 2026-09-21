# TDX Module Notes

- **Author**: Artem Bityutskiy
- **Version**: 0.1
- **Date**: 2026-07-03
- **Last updated**: 2026-09-06

**Disclaimer**: These notes reflect my current understanding of TDX module concepts and mechanisms
and may contain errors or omissions. They are not an official Intel document. Refer to the
official Intel documentation for authoritative information.

This is not a standalone article. It is a collection of notes and explanations about TDX module
concepts, intended for reference from my other documents.

**Terminology Notes**:

- A **seamcall** is a TDX module ABI leaf function invoked with the `SEAMCALL` instruction.
  For example, `TDH.MIG.SETUP` is a seamcall.
- A **tdcall** is a TDX module ABI leaf function invoked with the `TDCALL` instruction.
  Tdcalls include all `TDG.*` leaf functions.
- `SEAMCALL` and `TDCALL` refer to the CPU instructions, while `seamcall` and `tdcall` refer to
  the leaf functions they invoke.

---

## Table of Contents

- [TDX Module Notes](#tdx-module-notes)
  - [Table of Contents](#table-of-contents)
  - [VM Exit and TD Exit](#vm-exit-and-td-exit)
  - [Host State Across TDH.VP.ENTER](#host-state-across-tdhvpenter)
    - [SEAM Transition Detail](#seam-transition-detail)
  - [VE\_INFO](#ve_info)
    - [TDX-Specific Fields](#tdx-specific-fields)
  - [#VE](#ve)
    - [Architectural #VE](#architectural-ve)
    - [TDX-Extended #VE](#tdx-extended-ve)

---

## VM Exit and TD Exit

- A **VM exit** transfers control from SEAM non-root mode to the TDX module in SEAM root mode.
  The module handles the event and may resume the TD, inject an exception or `#VE`, or initiate a
  TD exit.
- A **TD exit** transfers control from TD execution to the VMM. It completes the
  `TDH.VP.ENTER` seamcall, and the CPU leaves SEAM mode through `SEAMRET`. TD exits are:
  - **Asynchronous**, caused by an event such as an external interrupt, exception, or EPT
    violation.
  - **Synchronous**, initiated by the TD with `TDG.VP.VMCALL`.

Every TD exit is preceded by a VM exit into the TDX module. A VM exit does not necessarily become a
TD exit because the module may handle it internally and resume the TD.

---

## Host State Across TDH.VP.ENTER

There are two state boundaries in TD execution.

- VMM to TDX module: `SEAMCALL` and `SEAMRET` switch between host VMM execution and TDX module
  execution.
- TDX module to TD: TD entry and TD exit switch between TDX module execution and TD execution.

The TDX module is responsible for saving and restoring TD state across TD entries and exits. Host
VMM state across `TDH.VP.ENTER` is a separate contract. Some host state is preserved by hardware or
by the TDX module, and some host state must be saved and restored by the VMM.

This matters when a TD is allowed to use a CPU feature that can modify host-visible state. The VMM
must know whether that state is preserved across `TDH.VP.ENTER`. If it is not preserved, the VMM
must save and restore it itself, or avoid exposing the feature to the TD.

For example, if a TD is allowed to use FRED, the VMM has to know which FRED MSRs are preserved
across `TDH.VP.ENTER`. Some FRED host state is analogous to VMX host-state fields and is expected
to be restored by the TDX module. Other state, such as `IA32_FRED_RSP0` and `IA32_PL0_SSP`, is
expected to be handled by VMM software.

Host [extended state](x86-misc.md#xsave-and-state-components) is another example of VMM-owned state.
Before `TDH.VP.ENTER`, the VMM is responsible for saving any host extended state that the TD is
allowed to use, per `XFAM`, and that the VMM expects to need after `TDH.VP.ENTER` returns.

Intel publishes the `msr_preservation.pdf` file that lists MSRs whose values may not be preserved
across TD entry and exit.

### SEAM Transition Detail

Here is how VMCS guest-state and host-state areas are used during VM entry and VM exit in case of a
traditional VMX guest.

| Direction | Saved State | Saved To         | Restored State | Restored From    |
|-----------|-------------|------------------|----------------|------------------|
| VM entry  | N/A         | N/A              | Guest          | Guest-state area |
| VM exit   | Guest       | Guest-state area | Host           | Host-state area  |

**Note**: `N/A` means that hardware does not save host state as part of VM entry. The VMM has to
prepare the VMCS host-state area, and hardware restores it on VM exit.

`TDH.VP.ENTER` uses the SEAM-transfer VMCS and works differently. From the VMM point of view, the
roles are flipped: the VMM state is saved in and restored from the guest-state area of the
SEAM-transfer VMCS.

| Direction | Saved State | Saved To              | Restored State | Restored From         |
|-----------|-------------|-----------------------|----------------|-----------------------|
| SEAMCALL  | VMM         | SEAM VMCS guest-state | TDX module     | SEAM VMCS host-state  |
| SEAMRET   | N/A         | N/A                   | VMM            | SEAM VMCS guest-state |

**Note**: "State" in the table does not mean full CPU state. Some pieces of state may be saved
only, some restored only, and some are neither saved nor restored by the hardware. The VMM or the
TDX module may handle them explicitly.

---

## VE_INFO

The **VE_INFO** (virtualization exception information area) is a memory area that carries the
details of a [#VE](#ve). It is per vCPU, it is allocated when the vCPU is created, and the VMCS of
the vCPU holds a pointer to it. The first 34 bytes are architectural, and whoever raises the `#VE`
fills them in: the CPU for an EPT violation, or the TDX module when it injects a `#VE` itself.

Intel SDM requires the VE_INFO address to be 4KiB-aligned, but does not require the area to be of
any certain size.

- The TDX module gives it a full 4KiB page inside the [TDVPS](tdx-structs.md#tdvps), and uses the
  space beyond the first 34 bytes for TDX-extended `#VE`s, discussed below.
- Linux KVM also allocates a 4KiB page for VMX guests, but it defines only the architectural fields
  and never uses the rest.

The fields in the architectural part of the VE_INFO area hold the values that would have gone into
the VMCS if a VM exit had happened instead of the `#VE`.

| Offset | Field                | Meaning                                                        |
|--------|----------------------|----------------------------------------------------------------|
| 0      | `EXIT_REASON`        | The VM exit reason, e.g., EPT violation (48) or MSR read (31). |
| 4      | `VALID`              | 0 means VE_INFO is free, 0xFFFFFFFF means it holds a `#VE`.    |
| 8      | `EXIT_QUALIFICATION` | Exit qualification value.                                      |
| 16     | `GLA`                | Guest linear address.                                          |
| 24     | `GPA`                | Guest physical address.                                        |
| 32     | `EPTP_INDEX`         | Current EPTP index VM-execution control.                       |

`EXIT_REASON` tells what caused the `#VE`. In case of an EPT violation the CPU raises the
`#VE` and writes 48, which is the only value the CPU ever writes. But a `#VE` can also be
injected by software, for example the TDX module injects one when it virtualizes an operation on
behalf of the TD, and then the value is the reason of that operation, such as MSR read (31) or
`CPUID` (10).

`VALID` marks the VE_INFO area as "occupied". Whoever raises the `#VE` sets it to 0xFFFFFFFF, and
the guest writes 0 back when it is done. A VMX guest does it directly, a TD calls
`TDG.VP.VEINFO.GET`, which hands over VE_INFO and clears `VALID`.

`EXIT_QUALIFICATION` carries extra details, and its meaning depends on `EXIT_REASON`. For an EPT
violation it includes whether the access was a read, a write, or an instruction fetch, and several
other bits.

`GLA` is the address the guest used, before paging translated it (see
[GLA and GVA](x86-misc.md#gla-and-gva)).

`EPTP_INDEX` identifies which EPTP in the list was active, so the handler knows which EPT the
violation happened in. Not relevant for TDX guests. See [EPTP](tdx-structs.md#eptp) for details.

### TDX-Specific Fields

In addition to the architectural 34 bytes, the TDX module maintains its own fields in the VE_INFO
area. The TD reads both the architectural and the TDX-specific fields with the same
`TDG.VP.VEINFO.GET` tdcall. Here are some of the TDX-specific fields.

- `INSTRUCTION_LENGTH` is the length of the instruction that caused the `#VE`. The handler adds it
  to `RIP` to resume past the instruction it just emulated.
- `INSTRUCTION_INFORMATION` and `EXTENDED_INSTRUCTION_INFORMATION` describe the operands, such as
  the registers and the address size, so that the `#VE` handler could emulate the operation.
- The `CATEGORY` byte, which tells the TD what kind of event it is looking at.

Support for `CATEGORY` is enumerated by `TDX_FEATURES0.VE_REDUCTION` (bit 30), and the TD gets it by
calling `TDG.VP.VEINFO.GET` with version 1 or higher.

| Value | Category              | Meaning                                                    |
|-------|-----------------------|------------------------------------------------------------|
| 0x00  | `ARCH`                | An EPT violation.                                          |
| 0x01  | `PENDING`             | SEPT violation: TD accessed a PENDING page.                |
| 0x02  | `RESERVED_GPA_BITS`   | GPA bits above the `MAXGPA` range were set, a TD bug.      |
| 0x10  | `CONFIG_PARAVIRT`     | Feature the VMM configured to be paravirtualized.          |
| 0x11  | `NON_CONFIG_PARAVIRT` | Feature that must be paravirtualized.                      |
| 0x80  | `UNSUPPORTED_FEATURE` | The TD used an x86 feature TDX does not support, a TD bug. |

A **feature** here is not an instruction, but what the instruction acts on. For example, `RDMSR` on
`MSR_PLATFORM_INFO` and `RDMSR` on another MSR are different features, and the TDX module may let
one through to the hardware, emulate another, and give a `#VE` on a third. Similarly for `CPUID`,
where the decision is per leaf and sub-leaf. But operandless instructions like `HLT` and `WBINVD`
are features in themselves.

The `NON_CONFIG_PARAVIRT` category indicates that the feature does not depend on the TD
configuration. If the TD does not handle it, it is a broken TD.

The `CONFIG_PARAVIRT` category indicates that the feature depends on the TD configuration. If the
TD does not handle it, it means that the TD refuses to support the feature. For example, if the
feature requires communicating with the VMM via `TDG.VP.VMCALL`, the TD may choose to refuse to
handle it because it does not trust the VMM. Instead, it may choose to treat it as `#GP(0)`, which
means `SIGSEGV` for userspace and an OOPS for the kernel.

**Note**: TDX specifications use the notation like `#VE(ARCH)`, which means a `#VE` of the `ARCH`
category.

**Linux Note**: As of version 7.2, the Linux kernel does not use the `CATEGORY` field. It uses
`TDG.VP.VEINFO.GET` with version 0 instead of version 1 or higher.

---

## #VE

**#VE** (Virtualization Exception) is x86 exception vector 20. It is not specific to TDX. Plain VMX
guests support it too, and the idea is that instead of exiting to the VMM so that the VMM emulates
something on behalf of the guest, the CPU delivers an exception to the guest, and the guest handles
the event itself. When a guest knows it is virtualized and handles such events on its own, this is
called **paravirtualization**, and the guest is often referred to as **enlightened**.

A `#VE` can happen only while the guest is running, never while the host is running. In terms of CPU
modes of operation, this means **VMX non-root mode** for a VMX guest and **SEAM non-root mode** for
a TD.

A TD can get two types of `#VE`, and they differ in what triggers them and in whether a
[VM exit](#vm-exit-and-td-exit) happens first.

- **Architectural #VE** is triggered by an EPT violation, the mechanism the x86 architecture
  defines for a memory access. For shared memory, the CPU itself applies this mechanism and
  delivers the `#VE` directly to the TD, with no VM exit. For private memory, the CPU can never
  apply the mechanism directly (see below), so a VM exit into the TDX module always happens first,
  and the module may inject the `#VE` itself, emulating the architectural mechanism.
- **TDX-extended #VE** comes from an operation that TDX has to virtualize, such as `CPUID`, an MSR
  access, or an instruction like `HLT`. The architecture never turns these into a `#VE`, hence the
  "extended" part, and only the TDX module can produce one, always after a VM exit into the module.

### Architectural #VE

Architectural `#VE` is the same mechanism for a VMX guest and for a TD's shared memory accesses.
When the guest attempts to access memory in a way that is not permitted by the EPT, an
**EPT violation** occurs, and the CPU delivers an architectural `#VE` instead of a VM exit only in
the following conditions.

- The **EPT-violation #VE** bit in the VMCS is set. For a VMX guest this is up to the VMM. For a
  TD the TDX module controls it, and it is always set.
- Bit 63 of the EPT entry is 0. This bit means **suppress #VE**, so it is a per page decision, made
  by whoever owns the page tables (VMM for shared memory, TDX module for private memory).
- `VE_INFO.VALID` is 0, which indicates that the VE_INFO area is empty, as opposed to holding
  information from a previous `#VE` that the guest has not cleared, and therefore supposedly has not
  handled. Refer to [VE_INFO](#ve_info) for details.

A TD has two EPT trees: the Secure EPT for private GPAs and the Shared EPT for shared GPAs, and the
`SHARED` bit of the GPA selects between them. Both are ordinary EPTs as far as the CPU is concerned,
so a failed walk in either one is an EPT violation, exit reason 48. There is no separate "SEPT
violation".

The TDX module always sets the EPT-violation `#VE` VMCS bit for a TD, so whether a `#VE` is raised
or not comes down to bit 63 alone, and that bit is owned differently for the two GPA spaces.

- **Shared memory**: The VMM owns the Shared EPT and controls bit 63 per page. This is how MMIO is
  emulated. The TD gets a `#VE(ARCH)`, decodes the faulting instruction, and forwards the access
  to the VMM with `TDG.VP.VMCALL`.
- **Private memory**: The TDX module owns the Secure EPT and always sets bit 63 to 1, so the CPU
  never raises a `#VE`. The EPT violation becomes a [VM exit](#vm-exit-and-td-exit) into the TDX
  module, which reads the SEPT entry and decides how to handle it, depending on the PTE state.
  Unlike the shared memory case, the CPU itself never injects the `#VE` here, only the module does,
  emulating the architectural mechanism.
  - One possibility is to inject a `#VE`. Today, this happens when the page is in the `PENDING` or
    `PENDING_EXPORTED_DIRTY` state. The module injects a `#VE(PENDING)`, so that the TD can accept
    the page with `TDG.MEM.PAGE.ACCEPT`.
  - The other is to turn the VM exit into a [TD exit](#vm-exit-and-td-exit). For example, if the
    page is in the `FREE` state, the VMM can provide a physical page for it.

**Notes**:

- The VMM sets the default `PENDING` page behavior at TD initialization with the `SEPT_VE_DISABLE`
  bit of [ATTRIBUTES](tdx-structs.md#attributes). When it is set, `PENDING` page access causes a TD
  exit rather than `#VE(PENDING)`.
- If the VMM enables the `FLEXIBLE_PENDING_VE` bit of [CONFIG_FLAGS](tdx-structs.md#config_flags),
  the TD can select the behavior at run time by toggling the `PENDING_VE_DISABLE` bit of
  [TD_CTLS](tdx-structs.md#td_ctls) with the `TDG.VM.WR` tdcall. Its initial value is a copy of
  `ATTRIBUTES.SEPT_VE_DISABLE`.

**Linux Notes**:

- Linux does not use `#VE(PENDING)`. It tracks which memory the TD firmware left `PENDING` and
  calls `TDG.MEM.PAGE.ACCEPT` before a page becomes usable, so the access never faults. Its `#VE`
  handler treats an EPT violation on a private GPA as a kernel bug and panics.
- Linux also makes sure it never gets one. If `CONFIG_FLAGS.FLEXIBLE_PENDING_VE` is enabled, it sets
  `TD_CTLS.PENDING_VE_DISABLE` itself. Otherwise it requires `ATTRIBUTES.SEPT_VE_DISABLE` to be 1
  and panics during boot if it is 0.
- The `ATTRIBUTES.SEPT_VE_DISABLE` bit is configurable via KVM ioctls, and QEMU sets it to 1 by
  default.

### TDX-Extended #VE

TDX-extended `#VE` covers operations that are not memory accesses, so the EPT-violation path does
not apply to them. Only the TDX module can inject this kind of `#VE`, always after a VM exit, and
only when the TD is configured to handle the operation itself instead of the module handling it.
There are three main sources.

- **Instructions that must be paravirtualized**. The TD cannot execute them, and there is no
  standard way to enumerate them as unsupported, so they always give `#VE(NON_CONFIG_PARAVIRT)`.
  Examples: `HLT`, `IN`, `OUT`, and `WBINVD`.
- **CPUID virtualization**. The TDX module virtualizes most CPUID leaves, but the ones it does not
  virtualize give `#VE(NON_CONFIG_PARAVIRT)`.
- **MSR accesses**. Some MSR accesses always give a `#VE`. For example, a `WRMSR` to
  `IA32_TIME_STAMP_COUNTER` gives a `#VE(NON_CONFIG_PARAVIRT)`. For other MSRs the TD configuration
  decides between a `#VE(CONFIG_PARAVIRT)` and a `#GP(0)`. The
  [TD_CTLS.REDUCE_VE](tdx-structs.md#td_ctls) control is one such knob, and the TD sets it itself:
  it makes the TDX module give a `#GP(0)` instead of a `#VE(CONFIG_PARAVIRT)` for many MSRs, when
  the TD OS chooses not to paravirtualize them.

Additionally, the TDX module can use `#VE` to notify the TD about anomalous behavior, such as
repeated EPT violations on the same instruction with no progress.
