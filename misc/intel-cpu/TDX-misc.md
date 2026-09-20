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

## Table of Contents

- [TDX Module Notes](#tdx-module-notes)
  - [Table of Contents](#table-of-contents)
  - [VM Exit and TD Exit](#vm-exit-and-td-exit)
  - [Host State Across TDH.VP.ENTER](#host-state-across-tdhvpenter)
    - [SEAM Transition Detail](#seam-transition-detail)
  - [TDCS](#tdcs)
  - [TD\_PARAMS](#td_params)
  - [ATTRIBUTES](#attributes)
    - [ATTRIBUTES Groups](#attributes-groups)
    - [ATTRIBUTES Bits](#attributes-bits)
    - [Allowed ATTRIBUTES Values](#allowed-attributes-values)
  - [XFAM](#xfam)
    - [XFAM and the State Component Bitmap](#xfam-and-the-state-component-bitmap)
    - [XFAM Bits](#xfam-bits)
    - [XFAM and Feature Gating](#xfam-and-feature-gating)
    - [XFAM and CPUID Virtualization](#xfam-and-cpuid-virtualization)
    - [Allowed XFAM Values](#allowed-xfam-values)
    - [Extended State Save and Restore](#extended-state-save-and-restore)
  - [CONFIG\_FLAGS](#config_flags)
    - [CONFIG\_FLAGS Bits](#config_flags-bits)
    - [Allowed CONFIG\_FLAGS Values](#allowed-config_flags-values)
  - [CPUID\_CONFIG](#cpuid_config)
    - [Enumeration CPUID\_CONFIG](#enumeration-cpuid_config)
    - [Configuration CPUID\_CONFIG](#configuration-cpuid_config)
  - [EPTP](#eptp)
    - [EPTP Switching](#eptp-switching)
  - [TDVPS](#tdvps)
  - [TD\_CTLS](#td_ctls)
    - [TD\_CTLS Bits](#td_ctls-bits)
  - [FEATURE\_PARAVIRT\_CTRL](#feature_paravirt_ctrl)
  - [TDX\_FEATURES0](#tdx_features0)
  - [VE\_INFO](#ve_info)
    - [TDX-Specific Fields](#tdx-specific-fields)
  - [#VE](#ve)
    - [Architectural #VE](#architectural-ve)
    - [TDX-Extended #VE](#tdx-extended-ve)

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

## TDCS

The **TDCS** (Trust Domain Control Structure) is the per-TD control structure that holds the runtime
state and configuration of the TD. Examples of the kinds of data it contains include:

- TD attributes, such as:
  - `ATTRIBUTES.MIGRATABLE`: whether TD live migration is allowed.
  - `ATTRIBUTES.DEBUG`: VMM can access vCPU state and private memory when debug mode is enabled.
  - `ATTRIBUTES.PERFMON`: whether TD software can use performance monitoring capabilities.
- TD operation state (`OP_STATE`), such as `UNINITIALIZED`, `RUNNABLE` or `LIVE_EXPORT`.
- The virtual CPUID configuration, and full or partial values for some of the MSRs.
- TD virtual TSC parameters (`TSC_MULTIPLIER` and `TSC_OFFSET`).
- The RTMRs (Run-Time extendable Measurement Registers).
- Migration policy `MIG_SETUP_TD_POLICY_HASH`, migration epoch counters, migration keys, and other
  live migration metadata.
- The Secure EPT root page pointer.

The TDCS physical layout is non-architectural and subject to change. It is an internal
multi-page TDX module structure. The VMM does not need to know the HPAs of TDCS pages. The TDX
module finds them from the TDR page.

The physical pages that hold the TDCS are called TDCX (TD Control structure eXtension) pages. The
VMM allocates these pages and adds them to the TDCS with the `TDH.MNG.ADDCX` seamcall.
`TDH.MNG.INIT` initializes the TDCS in these pages.

TDCS pages are encrypted with the TD private HKID and are not directly accessible by the VMM or TD.
The TDX module maps the TDCS and other control pages into its own virtual address space using its
own page tables. The VMM accesses selected TDCS fields with the `TDH.MNG.RD` and `TDH.MNG.WR`
seamcalls. The TD accesses its permitted TDCS fields with the `TDG.VM.RD` and `TDG.VM.WR` tdcalls.

TDCS holds TD-wide runtime state and configuration. Per-vCPU state is held separately in the TDVPS.

## TD_PARAMS

`TD_PARAMS` is the input structure to `TDH.MNG.INIT`, the main seamcall used to initialize a TD
when it is being built. The VMM is allowed to make one successful `TDH.MNG.INIT` call per TD. The
seamcall validates the input `TD_PARAMS`, creates most of the immutable TD configuration, and saves
it in the [TDCS](#tdcs). After that, neither the VMM nor the TD can change those configuration
values.

The VMM fills `TD_PARAMS` with TD configuration and passes its HPA to `TDH.MNG.INIT`. Examples
include whether debug mode or performance monitoring is enabled, and the virtualized values of
directly configurable CPUID leaves.

`TD_PARAMS` is 1024 bytes. Its layout is architectural for a given major version of the TDX module,
and the field offsets, sizes, and reserved ranges are part of the ABI contract.

| Offset | Size | Field                           | Attested |
|--------|------|---------------------------------|----------|
| 0      | 8    | [ATTRIBUTES](#attributes)       | Yes      |
| 8      | 8    | [XFAM](#xfam)                   | Yes      |
| 16     | 2    | `MAX_VCPUS`                     | No       |
| 18     | 1    | `NUM_L2_VMS`                    | No       |
| 19     | 1    | `MSR_CONFIG_CTLS`               | No       |
| 20     | 4    | Reserved                        | No       |
| 24     | 8    | [EPTP_CONTROLS](#eptp)          | No       |
| 32     | 8    | [CONFIG_FLAGS](#config_flags)   | No       |
| 40     | 2    | `TSC_FREQUENCY`                 | No       |
| 42     | 38   | Reserved                        | No       |
| 80     | 48   | `MRCONFIGID`                    | Yes      |
| 128    | 48   | `MROWNER`                       | Yes      |
| 176    | 48   | `MROWNERCONFIG`                 | Yes      |
| 224    | 8    | `IA32_ARCH_CAPABILITIES_CONFIG` | No       |
| 232    | 2    | `MRCONFIGSVN`                   | Yes      |
| 234    | 2    | `MROWNERCONFIGSVN`              | Yes      |
| 236    | 20   | Reserved                        | No       |
| 256    | 16*n | [CPUID_CONFIG](#cpuid_config)   | No       |

The `CPUID_CONFIG` area is a variable-length array of 16-byte entries defined by the ABI for the TDX
module major version in use.

The "Attested" column indicates whether the field is included in `TDREPORT_STRUCT` and therefore in
the TD attestation quote that a remote verifier sees.

**Note**: When a TD is created by live migration, it receives its immutable configuration from the
source platform. Instead of a local `TDH.MNG.INIT`, the destination VMM uses
`TDH.IMPORT.STATE.IMMUTABLE` to import this immutable state. In other words, the VMM does not use
`TD_PARAMS` and `TDH.MNG.INIT` in this case.

## ATTRIBUTES

`ATTRIBUTES` is a 64-bit field in the [TD_PARAMS](#td_params) structure. The VMM uses it to
configure the TD at build time through `TDH.MNG.INIT`. It is stored in the [TDCS](#tdcs) and cannot
be changed afterward, neither by the VMM nor by the TD.

The main dividing line between `ATTRIBUTES` and other `TD_PARAMS` fields is whether a property
affects TD security, either by weakening or by strengthening it. For example, debugging and
profiling make the TD more observable to the untrusted VMM, so they belong in `ATTRIBUTES`.
This line is not strict though.

`ATTRIBUTES` is attested: it ends up in the TD attestation quote, so the remote verifier can decide
whether the TD configuration is acceptable.

The VMM can read `ATTRIBUTES` from the TDCS with `TDH.MNG.RD`. The TD can read its own `ATTRIBUTES`
with `TDG.VP.INFO` and `TDG.VM.RD`.

### ATTRIBUTES Groups

Bits in `ATTRIBUTES` are split into groups based on their impact on TD security. Here are the
groups.

| Bits  | Group    | Meaning                                           |
|-------|----------|---------------------------------------------------|
| 3:0   | `TUD`    | TD is under debug.                                |
| 15:4  | `TUP`    | TD is under profiling.                            |
| 31:16 | `SEC`    | May affect TD security, positively or negatively. |
| 55:32 | Reserved | Must be 0.                                        |
| 63:56 | `OTHER`  | Attested, but no impact on TD security.           |

A verifier does not have to recognize every bit. If an unknown bit is encountered, it can still
decide based on the group the bit belongs to.

Bits in the `SEC` group are further marked positive or negative.

- **Positive** means that setting the bit strengthens TD security. For example, `PKS` lets the TD
  use Supervisor Protection Keys.
- **Negative** means that setting the bit weakens TD security. For example, `MIGRATABLE` allows a
  Migration TD to move the TD to another platform.

The reserved ranges in the `SEC` group follow the same split.

- `RESERVED_P` (bits 22:18) is set aside for future positive bits.
- `RESERVED_N` (bits 26:23) is set aside for future negative bits.

### ATTRIBUTES Bits

| Bit | Group   | Name              | Impact   | Meaning                                      |
|-----|---------|-------------------|----------|----------------------------------------------|
| 0   | `TUD`   | `DEBUG`           | Negative | Off-TD debug.                                |
| 4   | `TUP`   | `HGS_PLUS_PROF`   | Negative | Hardware-Guided Scheduling profiling.        |
| 5   | `TUP`   | `PERF_PROF`       | Negative | Profiling with perfmon counters.             |
| 6   | `TUP`   | `PMT_PROF`        | Negative | Profiling with core out-of-band telemetry.   |
| 16  | `SEC`   | `ICSSD`           | Positive | Instruction-Count based Single-Step Defense. |
| 17  | `SEC`   | `SERVTD_EXT`      | Positive | Extended service TD info in the report.      |
| 27  | `SEC`   | `LASS`            | Positive | Linear Address Space Separation.             |
| 28  | `SEC`   | `SEPT_VE_DISABLE` | Negative | No `#VE(PENDING)` on `PENDING` page access.  |
| 29  | `SEC`   | `MIGRATABLE`      | Negative | The TD can be live migrated.                 |
| 30  | `SEC`   | `PKS`             | Positive | Supervisor Protection Keys.                  |
| 62  | `OTHER` | `TPA`             | None     | The TD is a TDX Connect Provisioning Agent.  |
| 63  | `OTHER` | `PERFMON`         | None     | Perfmon and `PERF_METRICS` capabilities.     |

Additional comments.

- `DEBUG` puts the TD in off-TD debug mode, where the VMM can read and write vCPU state and
  private memory.
- `PERF_PROF` allows system-wide profiling with performance monitoring counters. The counters are
  not saved and restored on TD entry and exit, so they keep counting TD activity as part of the
  whole system. This is why the bit is negative: the untrusted host can observe TD execution.
- `SEPT_VE_DISABLE` is described in the [#VE](#ve) section.
- `MIGRATABLE` allows the TD to be moved to another machine while it keeps running.

### Allowed ATTRIBUTES Values

Not every `ATTRIBUTES` bit is supported by every TDX module version and every platform. The TDX
module reports the allowed values in two bitmaps that the VMM reads with `TDH.SYS.RD` or
`TDH.SYS.RDALL`.

- `ATTRIBUTES_FIXED0` tells which bits may be set. If a bit is 0 there, the corresponding
  `ATTRIBUTES` bit must be 0. If a bit is 1 there, the bit may be 0 or 1, unless
  `ATTRIBUTES_FIXED1` forces it to 1.
- `ATTRIBUTES_FIXED1` tells which bits must be set. If a bit is 1 there, the corresponding
  `ATTRIBUTES` bit must be 1. If a bit is 0 there, the bit may be 0 or 1, as long as
  `ATTRIBUTES_FIXED0` allows it to be 1.

A useful way to read the two bitmaps together is this: `ATTRIBUTES_FIXED0` clears unsupported bits,
and `ATTRIBUTES_FIXED1` forces required bits. The VMM can choose the value only when the bit is 1 in
`ATTRIBUTES_FIXED0` and 0 in `ATTRIBUTES_FIXED1`.

Some bits are mutually exclusive, for example `PERFMON` and `MIGRATABLE`, as well as `PERFMON` and
`ICSSD`.

## XFAM

This subsection assumes familiarity with XSAVE state components, their types, `XCR0`, `IA32_XSS`,
and `CPUID.0xD`, which [XSAVE and State Components](x86-misc.md#xsave-and-state-components)
explains.

`XFAM` (eXtended Features Available Mask) is a 64-bit field in the
[TD_PARAMS](#td_params) structure. The VMM uses it to configure the TD at build time through
`TDH.MNG.INIT`. It defines which CPU extended features the TD is allowed to use. It is stored in the
[TDCS](#tdcs) and cannot be changed afterward, neither by the VMM nor by the TD.

`XFAM` is attested: it ends up in the TD attestation quote, so the remote verifier can decide
whether the TD configuration is acceptable.

### XFAM and the State Component Bitmap

`XFAM` is a state component bitmap covering both user and supervisor components. It defines which
bits the TD may set in `XCR0` and `IA32_XSS`, and what `CPUID(0xD)` reports to the TD. That, in
turn, determines which CPU features the TD may use.

When the TD changes `XCR0` or `IA32_XSS`, the TDX module validates the new value. It applies the
architectural rules the CPU would enforce, and on top of them requires every bit set to also be set
in `XFAM`. A value that fails either check gives `#GP(0)`.

The architectural limits that apply to `XCR0` and `IA32_XSS` also apply to `XFAM`: reserved bits,
unsupported features, and invalid bit combinations are not allowed. On top of that, the TDX module
adds rules about which bits and bit combinations a TD may be configured with.

### XFAM Bits

Here is a summary of the `XFAM` bits and their meanings.

| Bits  | U/S | Feature        | Meaning                                                  |
|-------|-----|----------------|----------------------------------------------------------|
| 0     | U   | FP (x87)       | Always 1.                                                |
| 1     | U   | SSE            | Always 1.                                                |
| 2     | U   | AVX            | Lets the TD enable AVX in `XCR0`.                        |
| 4:3   | U   | MPX            | Always 0, MPX is deprecated.                             |
| 7:5   | U   | AVX-512        | Lets the TD enable AVX-512 in `XCR0`, requires bit 2.    |
| 8     | S   | PT (RTIT)      | Lets the TD access the `IA32_RTIT_*` MSRs.               |
| 9     | U   | PK (PKRU)      | Lets the TD set `CR4.PKE`.                               |
| 10    | S   | ENQCMD (PASID) | Always 0, unless TDX Connect is used.                    |
| 12:11 | S   | CET            | Lets the TD set `CR4.CET`.                               |
| 13    | S   | HDC            | Always 0.                                                |
| 14    | S   | ULI            | Lets the TD set `CR4.UINTR` and use `IA32_UINTR_*`.      |
| 15    | S   | LBR            | Lets the TD use the `IA32_LBR_*` MSRs and LBR logging.   |
| 16    | S   | HWP            | Always 0.                                                |
| 18:17 | U   | AMX            | Lets the TD enable AMX in `XCR0`.                        |
| 19    | U   | APX            | Lets the TD enable APX in `XCR0`.                        |
| Other | N/A | Reserved       | Must be 0.                                               |

The U/S column tells whether the state component is user or supervisor. Bits that are always 0 are
for features the TDX module never allows a TD to use, and bits that are always 1 are for state every
TD needs.

### XFAM and Feature Gating

The [XSAVE and State Components](x86-misc.md#xsave-and-state-components) section explains that CPU
features with an associated extended state component are called **XSAVE-supported** features, and
that the ones requiring their state component bits to be set in `XCR0` before software may use them
are called **XSAVE-enabled** features. Both are Intel SDM terms.

For XSAVE-enabled features, such as AVX, AVX-512, AMX and APX, the TD can use the feature only when
the state component bits are 1 in both `XFAM` and the TD `XCR0`. To disable a feature, it is enough
for the VMM to clear the bits in `XFAM`: the TD then will not be able to set them in `XCR0`, and
the CPU will raise `#UD` on an instruction that tries to use the feature.

For XSAVE-supported features that are not XSAVE-enabled, the state component bits govern only state
save and restore. The CPU lets software use these features regardless of the `XCR0` or `IA32_XSS`
state component bit. In other words, the CPU itself does not enforce any restriction based on
`XFAM`. For the XFAM bits listed above, the TDX module enforces these restrictions.

- PK, CET and ULI: the enable switch is a `CR4` bit, and `XFAM` decides whether the TDX module lets
  the TD set `CR4.PKE`, `CR4.CET` or `CR4.UINTR`. A disallowed write gives `#GP(0)`.
- PT and LBR: there is no enable switch at all. `XFAM` decides whether the TDX module lets the TD
  access the MSRs of the feature. A disallowed access gives `#GP(0)` or a [#VE](#ve).

Conclusion: `XFAM` is more than a bitmap of the `XCR0` and `IA32_XSS` bits the TD may set. It is
more practical to look at it as the mechanism that controls which CPU features the TD may use.

### XFAM and CPUID Virtualization

`XFAM` also shapes the virtual CPUID values the TD sees, so that a TD does not enumerate features it
may not use. For example:

- `CPUID(0xD)` enumerates the state components available to the TD, and these are the `XFAM` bits.
- Feature bits in other leaves are derived from `XFAM` too, for example `CPUID(1).ECX.AVX` comes
  from `XFAM` bit 2.

### Allowed XFAM Values

Which bits is the VMM allowed to set in `XFAM`? The answer comes from `XFAM_FIXED0` and
`XFAM_FIXED1`, which the VMM reads with `TDH.SYS.RD` or `TDH.SYS.RDALL`. They are analogous to
`ATTRIBUTES_FIXED0` and `ATTRIBUTES_FIXED1` of the [ATTRIBUTES](#attributes) field.

- `XFAM_FIXED0` tells which bits may be set. If a bit is 0 there, the corresponding `XFAM` bit must
  be 0. If a bit is 1 there, the bit may be 0 or 1, unless `XFAM_FIXED1` forces it to 1.
- `XFAM_FIXED1` tells which bits must be set. If a bit is 1 there, the corresponding `XFAM` bit must
  be 1. If a bit is 0 there, the bit may be 0 or 1, as long as `XFAM_FIXED0` allows it to be 1.

A useful way to read the two bitmaps together is this: `XFAM_FIXED0` clears unsupported bits, and
`XFAM_FIXED1` forces required bits. The VMM can choose the value only when the bit is 1 in
`XFAM_FIXED0` and 0 in `XFAM_FIXED1`.

**Notes**:

- Just like in `XCR0`, some bit groups in `XFAM` must be set to the same values. For example,
  AVX-512 (bits 7:5) may be set only as all zeros or all ones.
- `TDH.MNG.INIT` rejects an `XFAM` value that breaks any of these rules with `TDX_OPERAND_INVALID`.

### Extended State Save and Restore

The extended state of a vCPU has to survive TD exits and entries, and it must not leak to the
untrusted VMM. The TDX module keeps it in the XSAVE buffer of the [TDVPS](#tdvps), and `XFAM`
defines which components go there.

- On TD entry, `TDH.VP.ENTER` restores from the TDVPS the components allowed by `XFAM`.
- On asynchronous [TD exit](#vm-exit-and-td-exit), the module saves them back to the TDVPS and then
  clears the extended state, so the VMM cannot observe anything the TD left behind.
- As described in [Host State Across TDH.VP.ENTER](#host-state-across-tdhvpenter), the VMM is
  responsible for host extended state that the TD is allowed to use, per `XFAM`, and that the VMM
  needs after `TDH.VP.ENTER` returns.

The XMM registers are a special case, because the `TDG.VP.VMCALL` ABI uses them to pass data between
the TD and the VMM. The TD selects the registers to share in the `RCX` input of the tdcall. The
module saves the other state in the TDVPS, but it leaves the selected XMM registers visible to the
VMM. On the next TD entry after `TDG.VP.VMCALL`, the same `RCX` value selects the XMM registers that
the VMM passes back to the TD.

## CONFIG_FLAGS

`CONFIG_FLAGS` is a 64-bit field in the [TD_PARAMS](#td_params) structure. The VMM uses it to
configure the TD at build time through `TDH.MNG.INIT`. The value is stored in the [TDCS](#tdcs)
and cannot be changed afterward, neither by the VMM nor by the TD.

Unlike [ATTRIBUTES](#attributes) and [XFAM](#xfam), which also configure the TD through the same
seamcall, `CONFIG_FLAGS` is not attested.

The flags fall into three categories.

- **Address space layout**: `GPAW`, `MAXPA_VIRT` and `MAXGPA_VIRT` define where the `SHARED` bit
  sits in a GPA and which address widths the TD sees.
- **Interface hardening**: `NO_RBP_MOD` prevents the VMM from changing the TD's frame pointer
  (`RBP`) across tdcall and TD entry/exit, while `FLEXIBLE_PENDING_VE` lets the TD choose how
  `#VE(PENDING)` is handled on PENDING-page accesses.
- **Feature enabling**: `TDX_CONNECT`, `PAGE_RELEASE` and `SEALING` enable facilities the TD may
  use.

### CONFIG_FLAGS Bits

| Bit  | Category  | Name                  | Meaning                                    |
|------|-----------|-----------------------|--------------------------------------------|
| 0    | Layout    | `GPAW`                | Where the `SHARED` bit sits in a GPA.      |
| 1    | Interface | `FLEXIBLE_PENDING_VE` | The TD may control `#VE(PENDING)`.         |
| 2    | Interface | `NO_RBP_MOD`          | `RBP` is not passed to or from the VMM.    |
| 3    | Layout    | `MAXPA_VIRT`          | The VMM configures the virtual MAXPA.      |
| 4    | Layout    | `MAXGPA_VIRT`         | The TDX module derives the virtual MAXGPA. |
| 5    | Feature   | `TDX_CONNECT`         | Enables TDX Connect for this TD.           |
| 6    | Feature   | `PAGE_RELEASE`        | Enables `TDG.MEM.PAGE.RELEASE`.            |
| 7    | Feature   | `SEALING`             | Enables `TDG.MR.KEY.GET`.                  |
| 63:8 | N/A       | Reserved              | Must be 0.                                 |

Additional comments.

- `GPAW` does not directly configure a guest physical address width, despite the name. It
  selects which GPA bit is the `SHARED` bit, `GPA[47]` when 0 and `GPA[51]` when 1. This determines
  the effective GPA width for the TD: 48 or 52 bits. This also splits the GPA space into a private
  half and a shared half. Setting it to 1 requires 5-level EPT, so the VMM must also set the EPT
  depth field of [TD_PARAMS.EPTP_CONTROLS](#eptp) to 4.
- `NO_RBP_MOD` fixes a legacy TDX module ABI imperfection. The `RBP` register is the frame pointer
  in the x86-64 ABI, classified as callee-saved, so a compiler assumes every function call preserves
  it. The original TDX module ABI let the VMM pass any `RBP` value back to the TD from a tdcall,
  which the untrusted VMM could use to attack the TD. With `NO_RBP_MOD` set: `TDG.VP.VMCALL`
  preserves the `RBP` value of the TD, and `TDH.VP.ENTER` preserves the `RBP` value of the VMM. The
  flag exists only for backward compatibility, new software should always set it.
- `PAGE_RELEASE` is implicitly set to 1 when `TDX_CONNECT` is 1.
- `TDX_CONNECT` may not be set if `ATTRIBUTES.MIGRATABLE` is 1 and the TDX module has been
  configured for write-blocking based export.

### Allowed CONFIG_FLAGS Values

Not every `CONFIG_FLAGS` bit is supported by every TDX module version and platform. The TDX module
reports the allowed values in two bitmaps that the VMM reads with the `TDH.SYS.RD` seamcall. They
are analogous to the `ATTRIBUTES_FIXED0` and `ATTRIBUTES_FIXED1` fields of the
[ATTRIBUTES](#attributes) structure.

- `CONFIG_FLAGS_FIXED0` tells which bits may be set. If a bit is 0 there, the corresponding
  `CONFIG_FLAGS` bit must be 0. If a bit is 1 there, the bit may be 0 or 1, unless
  `CONFIG_FLAGS_FIXED1` forces it to 1.
- `CONFIG_FLAGS_FIXED1` tells which bits must be set. If a bit is 1 there, the corresponding
  `CONFIG_FLAGS` bit must be 1. If a bit is 0 there, the bit may be 0 or 1, as long as
  `CONFIG_FLAGS_FIXED0` allows it to be 1.

A useful way to read the two bitmaps together is this: `CONFIG_FLAGS_FIXED0` clears unsupported
bits, and `CONFIG_FLAGS_FIXED1` forces required bits. The VMM can choose the value only when the bit
is 1 in `CONFIG_FLAGS_FIXED0` and 0 in `CONFIG_FLAGS_FIXED1`.

`TDH.MNG.INIT` rejects a `CONFIG_FLAGS` value that breaks these rules with `TDX_OPERAND_INVALID`.

## CPUID_CONFIG

`CPUID_CONFIG` refers to two related, but different, data structures used in the TDX module seamcall
ABI.

- **Enumeration CPUID_CONFIG**, read by the VMM with `TDH.SYS.INFO`, tells the VMM which
  CPUID leaves, sub-leaves and bits it may configure.
- **Configuration CPUID_CONFIG**, used by the VMM to configure the virtual CPUID leaves and
  sub-leaves when it initializes the TD with `TDH.MNG.INIT`.

### Enumeration CPUID_CONFIG

`TDH.SYS.INFO` returns the `TDSYSINFO_STRUCT` data structure, which includes the `CPUID_CONFIG`
array of 24 byte entries at offset 132. Here is the layout of one entry.

| Offset | Size | Field      | Description                                                     |
|--------|------|------------|-----------------------------------------------------------------|
| 0      | 4    | `LEAF`     | The `EAX` input value to `CPUID`.                               |
| 4      | 4    | `SUB_LEAF` | The `ECX` input value. -1 means the leaf has no sub-leaves.     |
| 8      | 4    | `EAX`      | Mask of configurable bits in the virtual `EAX` return value.    |
| 12     | 4    | `EBX`      | Mask of configurable bits in the virtual `EBX` return value.    |
| 16     | 4    | `ECX`      | Mask of configurable bits in the virtual `ECX` return value.    |
| 20     | 4    | `EDX`      | Mask of configurable bits in the virtual `EDX` return value.    |

A mask bit of 1 means the VMM may configure that bit. The `NUM_CPUID_CONFIG` field at offset 128
specifies how many entries in the array are valid.

**Note**: `TDH.SYS.INFO` and `TDSYSINFO_STRUCT` are legacy ABI. The newer `TDH.SYS.RD` and
`TDH.SYS.RDALL` seamcalls are the recommended interfaces for enumerating TDX module metadata.

### Configuration CPUID_CONFIG

The `TDH.MNG.INIT` seamcall accepts the `TD_PARAMS` structure, which holds the TD initialization
parameters. One of them is the `CPUID_CONFIG` array of 16-byte entries at offset 256, each entry
corresponding to a virtual CPUID leaf and sub-leaf. Here is the layout of one entry.

| Offset | Size | Field | Description                                  |
|--------|------|-------|----------------------------------------------|
| 0      | 4    | `EAX` | Configured value of the virtual `EAX`.       |
| 4      | 4    | `EBX` | Configured value of the virtual `EBX`.       |
| 8      | 4    | `ECX` | Configured value of the virtual `ECX`.       |
| 12     | 4    | `EDX` | Configured value of the virtual `EDX`.       |

There are no leaf and sub-leaf numbers here. The correspondence is positional: the two arrays have
the same number of entries, and entry *i* of the configuration `CPUID_CONFIG` array corresponds to
entry *i* of the enumeration `CPUID_CONFIG` array described above.

## EPTP

The **EPTP** (Extended Page Table Pointer) is the VMCS field that points at the root of the EPT
tree the CPU walks to translate a GPA into an HPA. Besides the root pointer it carries a few
controls for the walk itself.

| Bits  | Field                   | Meaning                                                   |
|-------|-------------------------|-----------------------------------------------------------|
| 2:0   | Memory type             | Memory type for accessing the EPT structures. 6 is WB.    |
| 5:3   | EPT depth               | One less than the EPT levels: 3 is 4-level, 4 is 5-level. |
| 6     | Accessed and dirty      | Enables the accessed and dirty flags in EPT entries.      |
| 7     | Supervisor shadow stack | Enables supervisor shadow stack access rights.            |
| 11:8  | Reserved                | Must be 0.                                                |
| 51:12 | Root HPA                | HPA of the top-level EPT page, 4KiB aligned.              |
| 63:52 | Reserved                | Must be 0.                                                |

Neither the VMM nor the TD can write the EPTP. The VMM supplies the EPTP control bits in the
`EPTP_CONTROLS` field of [TD_PARAMS](#td_params), `TDH.MNG.INIT` combines them with the Secure EPT
root address into the `TDCS.EPTP` field, and `TDH.VP.INIT` copies that into the TD VMCS of each
vCPU.

### EPTP Switching

**EPTP Switching** lets the VMM prepare a list of up to 512 EPTPs, from which the guest picks one
with the `VMFUNC` instruction, without a [VM exit](#vm-exit-and-td-exit).

Each EPTP is a different view of guest memory, with its own mappings and permissions. It is designed
for things like security agents running inside the guest, which switch views depending on the
situation.

In TDX, guests cannot use the `VMFUNC` instruction to switch EPTs. Each TD VM has two EPT trees,
Secure EPT and Shared EPT, and the `SHARED` bit of the GPA selects between them, not an EPTP index.

**Linux Note**: The Linux kernel does not use EPTP switching for its own VMX guests either, so a VMX
guest running directly on KVM cannot switch EPTs. But KVM does support the feature for nested
virtualization, where the guest of KVM is itself a hypervisor and wants to offer EPTP switching to
its own guests. KVM emulates it in software: it leaves the hardware feature disabled, so `VMFUNC` in
the nested guest causes a VM exit to KVM, and KVM reads the requested EPTP from the list and
switches the mappings itself. The nested guest sees the architectural behavior, but pays for a VM
exit anyway.

## TDVPS

The **TDVPS** (Trust Domain Virtual Processor State) is the per-vCPU counterpart of the
[TDCS](#tdcs). It holds the runtime state and configuration of a single vCPU. Examples of the kinds
of data it contains include:

- TD VMCS (Virtual Machine Control Structure): the standard Intel VT-x structure that controls
  virtual CPUs, inaccessible to the VMM in the case of TDs.
- Guest GPR (General Purpose Register), MSR (Model-Specific Register), and other architectural
  CPU state.

The TDVPS physical layout is non-architectural and subject to change. Its pages are encrypted with
the TD private HKID and are not directly accessible by the VMM or the TD. From the VMM perspective,
TDVPS is composed of multiple 4KiB pages.

- **TDVPR** (Trust Domain Virtual Processor Root) is the 4KiB root page of TDVPS.
- **TDCX** (Trust Domain Control structure eXtension) pages extend the TDVPR. In the TDVPS page
  array, the TDVPR is page 0 and the TDCX pages start at page 1.

At vCPU build time, the `TDH.VP.CREATE` seamcall creates the vCPU and its TDVPR root page. Later
the `TDH.VP.ADDCX` seamcall adds the TDCX pages. `TDVPS_BASE_SIZE`, readable with the
`TDH.SYS.RD` seamcall, reports the base TDVPS size in bytes. For a TD with L2 VMs, the VMM adds
`NUM_L2_VMS * TDVPS_SIZE_PER_L2_VM` bytes.

The VMM needs to know the HPA of the TDVPR page, because it is used in various per-vCPU seamcalls,
such as `TDH.VP.RD` and `TDH.VP.WR`, to identify the vCPU. But the VMM does not need to know the
HPAs of TDCX pages.

## TD_CTLS

`TD_CTLS` is a 64-bit field of [TDCS](#tdcs) holding TD configuration bits that the TD itself
controls. The TD can write `TD_CTLS` at run time with the `TDG.VM.WR` tdcall, and read it with
`TDG.VM.RD`.

The purpose of `TD_CTLS` is to let an enlightened TD say how much of the virtualization work it
wants to do. Most of the bits trade paravirtualization for simplicity.

### TD_CTLS Bits

| Bit  | Name                 | Meaning                                                       |
|------|----------------------|---------------------------------------------------------------|
| 0    | `PENDING_VE_DISABLE` | TD exit instead of `#VE(PENDING)` on a `PENDING` page.        |
| 1    | `ENUM_TOPOLOGY`      | Virtual topology vs `#VE` on topology CPUID leaves and MSR.   |
| 2    | `VIRT_CPUID2`        | `CPUID(0x2)` returns fixed values instead of `#VE`.           |
| 3    | `REDUCE_VE`          | Greatly reduces `#VE` on `CPUID`, `RDMSR`, `WRMSR` and more.  |
| 4    | `ENABLE_HW_KEYS`     | A migratable TD may use `TDG.MR.KEY.GET` unconditionally.     |
| 62:5 | Reserved             | Must be 0.                                                    |
| 63   | `LOCK`               | Locks all TD-writable virtualization controls.                |

Additional comments.

- Each bit is available only if the TDX module supports it, which the TD checks in
  [TDX_FEATURES0](#tdx_features0). If a feature is not supported, the corresponding bit must be 0.
- `PENDING_VE_DISABLE` starts as a copy of `ATTRIBUTES.SEPT_VE_DISABLE`, and the TD may change it
  only if the VMM set `CONFIG_FLAGS.FLEXIBLE_PENDING_VE`. See [#VE](#ve) for how this affects access
  to `PENDING` pages.
- `ENUM_TOPOLOGY` enables/disables `#VE` for `CPUID(0xB)`, `CPUID(0x1F)` and the
  `IA32_X2APIC_APICID` MSR.
- `ENABLE_HW_KEYS` lets a migratable TD use `TDG.MR.KEY.GET` regardless of
  `CONFIG_FLAGS.SEALING`.
- `REDUCE_VE`'s main role is reducing `#VE`, but it has one unrelated role too: for sub-leaves 0
  to 3 of the Cache Parameters leaf (`CPUID(0x4)`), it switches the source of the virtual value
  from the TD's configured CPUID values to a native copy sampled at TD build time. `CPUID(0x4)`
  never causes a `#VE`, with `REDUCE_VE` 0 or 1. Here, the bit just picks which value to return,
  and has nothing to do with `#VE`.
- Setting `REDUCE_VE` implicitly sets `ENUM_TOPOLOGY` and `VIRT_CPUID2`.
- `ENUM_TOPOLOGY`, and therefore `REDUCE_VE`, can be set only if the VMM configured a unique
  virtual x2APIC ID for every vCPU.
- `LOCK` is a one-way door. Once set, `TD_CTLS`, [FEATURE_PARAVIRT_CTRL](#feature_paravirt_ctrl),
  and the `CPUID_SUPERVISOR_VE`, `CPUID_USER_VE` and `CPUID_CONTROL` fields of [TDVPS](#tdvps)
  become read-only. The TD can use it to prevent its own software from changing the virtualization
  contract it set up at boot.

**Linux Notes**

As of version 7.3, here is the logic the TDX guest Linux kernel follows for each `TD_CTLS` bit,
early at boot.

- `PENDING_VE_DISABLE` (bit 0): if `CONFIG_FLAGS.FLEXIBLE_PENDING_VE` is set, write
  `PENDING_VE_DISABLE` with `TDG.VM.WR`. Otherwise, require `ATTRIBUTES.SEPT_VE_DISABLE` to already
  be set, and panic at boot if it is not.
- Linux doesn't handle `#VE(PENDING)`. It tracks which memory the TD firmware left `PENDING` and
  calls `TDG.MEM.PAGE.ACCEPT` before a page becomes usable, so the access does not fault. See
  [#VE](#ve) for more details.
- `ENUM_TOPOLOGY` (bit 1): set it only as a fallback, if setting `REDUCE_VE` fails.
- `VIRT_CPUID2` (bit 2): do not set it directly. Get it as a side effect when `REDUCE_VE` succeeds.
- `REDUCE_VE` (bit 3): set it unconditionally. This is the main switch Linux relies on.
- `ENABLE_HW_KEYS` (bit 4): do not use it. Linux does not even define this bit. Its documented
  default is 0.
- `LOCK` (bit 63): do not use it. Linux never locks `TD_CTLS`. Its documented default is 0.

## FEATURE_PARAVIRT_CTRL

`TDCS.FEATURE_PARAVIRT_CTRL` is a [TDCS](#tdcs) field containing a bitmask with one bit per CPU
feature, controlling whether the TD's access to that feature causes a `#VE`.

- **0**: the TD sees the feature as absent. Its CPUID bits are forced to 0, and an MSR access
  gives `#GP(0)`.
- **1**: the TD sees the feature the way the VMM configured it. Its CPUID bits reflect that
  configuration, and an MSR access may give `#VE(CONFIG_PARAVIRT)`.

The TD itself controls these bits, writing `FEATURE_PARAVIRT_CTRL` at run time with `TDG.VM.WR`. It
defaults to all zeros, and becomes usable only once `TD_CTLS.REDUCE_VE` is 1. It is useful to see
this feature as a finer-grained control mechanism for handling `#VE` on top of `TD_CTLS.REDUCE_VE`.

The controlled features include `CORE_CAPABILITIES`, `MCA`, `MTRR`, `PCONFIG`, `RDT-A`, `RDT-M`,
`DCA`, `EST`, `ACPI`, `TM2`, `TME` and `TSC_DEADLINE`.

## TDX_FEATURES0

`TDX_FEATURES0` is a 64-bit global metadata field that enumerates what the running TDX module
supports. The TDX module sets it at initialization time. It depends on platform capabilities
and the capabilities of this specific TDX module version.

When a `TDX_FEATURES0` bit is 1, it permits enabling the corresponding feature. Here are some
examples of the bits, out of the full set.

| Bit | Name                       | Permits                                                  |
|-----|----------------------------|----------------------------------------------------------|
| 0   | `TD_MIGRATION`             | Using TD migration seamcalls and tdcalls.                |
| 12  | `HW_SEALING`               | Setting `TD_CTLS.ENABLE_HW_KEYS`.                        |
| 16  | `PENDING_EPT_VIOLATION_V2` | Setting `TD_CTLS.PENDING_VE_DISABLE`.                    |
| 17  | `FMS_CONFIG`               | Configuring `CPUID(0x1).EAX` for migratable TDs.         |
| 18  | `NO_RBP_MOD`               | Setting `CONFIG_FLAGS.NO_RBP_MOD`.                       |
| 20  | `TOPOLOGY_ENUM`            | Setting `TD_CTLS.ENUM_TOPOLOGY`.                         |
| 27  | `MAXPA_VIRT`               | Setting `CONFIG_FLAGS.MAXPA_VIRT`.                       |
| 29  | `CPUID2_VIRT`              | Setting `TD_CTLS.VIRT_CPUID2`.                           |
| 30  | `VE_REDUCTION`             | Setting `TD_CTLS.REDUCE_VE` and `FEATURE_PARAVIRT_CTRL`. |

`TDX_FEATURES0` can be read by VMM using the `TDH.SYS.RD` seamcall, and by TD using the `TDG.SYS.RD`
seamcall. But it cannot be modified by either the VMM or the TD.

`TDX_FEATURES0` influences [ATTRIBUTES](#attributes), [XFAM](#xfam) and
[CONFIG_FLAGS](#config_flags), but indirectly. It shapes the contents of their `*_FIXED0`/`*_FIXED1`
fields (e.g., `XFAM_FIXED0`), which is what decides the allowed bit combinations.

## VE_INFO

The **VE_INFO** (virtualization exception information area) is a memory area that carries the
details of a [#VE](#ve). It is per vCPU, it is allocated when the vCPU is created, and the VMCS of
the vCPU holds a pointer to it. The first 34 bytes are architectural, and whoever raises the `#VE`
fills them in: the CPU for an EPT violation, or the TDX module when it injects a `#VE` itself.

Intel SDM requires the VE_INFO address to be 4KiB-aligned, but does not require the area to be of
any certain size.

- The TDX module gives it a full 4KiB page inside the [TDVPS](#tdvps), and uses the space beyond the
  first 34 bytes for TDX-extended `#VE`s, discussed below.
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
violation happened in. Not relevant for TDX guests. See [EPTP](#eptp) for details.

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
  bit of [ATTRIBUTES](#attributes). When it is set, `PENDING` page access causes a TD exit rather
  than `#VE(PENDING)`.
- If the VMM enables the `FLEXIBLE_PENDING_VE` bit of [CONFIG_FLAGS](#config_flags), the TD can
  select the behavior at run time by toggling the `PENDING_VE_DISABLE` bit of [TD_CTLS](#td_ctls)
  with the `TDG.VM.WR` tdcall. Its initial value is a copy of `ATTRIBUTES.SEPT_VE_DISABLE`.

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
  decides between a `#VE(CONFIG_PARAVIRT)` and a `#GP(0)`. The [TD_CTLS.REDUCE_VE](#td_ctls) control
  is one such knob, and the TD sets it itself: it makes the TDX module give a `#GP(0)`
  instead of a `#VE(CONFIG_PARAVIRT)` for many MSRs, when the TD OS chooses not to paravirtualize
  them.

Additionally, the TDX module can use `#VE` to notify the TD about anomalous behavior, such as
repeated EPT violations on the same instruction with no progress.
