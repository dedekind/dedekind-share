# Intel CPU

- **Author**: Artem Bityutskiy
- **Version**: 0.1
- **Date**: 2026-07-13
- **Last updated**: 2026-09-06

**Disclaimer**: This document reflects my current understanding of various Intel CPU concepts and
mechanisms, but it may contain errors or omissions. It is not an official Intel document and should
not be treated as such. For full and authoritative information, refer to the Intel SDM.

This is not a standalone article. It is a collection of notes and explanations about Intel CPU
architecture, intended for reference from my other documents.

---

## Table of Contents

- [Intel CPU](#intel-cpu)
  - [Table of Contents](#table-of-contents)
  - [GLA and GVA](#gla-and-gva)
  - [XSAVE and State Components](#xsave-and-state-components)
    - [XCR0 and IA32\_XSS](#xcr0-and-ia32_xss)
    - [Features and XSAVE Components](#features-and-xsave-components)
    - [State Component Enumeration](#state-component-enumeration)
    - [Valid XCR0 and IA32\_XSS Values](#valid-xcr0-and-ia32_xss-values)

---

## GLA and GVA

**GLA** (Guest Linear Address) is the Intel term. The SDM uses it consistently, for example in
VMCS field names.

**GVA** (Guest Virtual Address) is the generic term used by other architectures and operating system
documentation.

In Intel terminology the x86 address translation chain is logical address, then linear address
after segmentation, then physical address after paging.

```text
logical address           linear address          physical address
(selector:offset) ------> (GLA or GVA) ---------> (GPA in a guest)
                 segment                 paging
                  base
```

With flat segmentation, which modern operating systems such as Linux use, the segment base is 0,
so GLA and GVA are the same address.

---

## XSAVE and State Components

`XSAVE` is the x86 instruction that saves CPU **extended state** to a memory buffer, and `XRSTOR`
loads it back. Extended state is the architectural state of the CPU feature extensions. Most of it
is data registers, such as the XMM registers of SSE or the tile registers of AMX, but it also
includes control and status registers such as `MXCSR` and `PKRU`, and MSRs such as `IA32_RTIT_*`.

`XSAVE` and `XRSTOR` belong to a larger family of instructions that Intel calls the XSAVE feature
set, which also includes optimized and privileged variants.

The OS chooses how much of the extended state to save. Extended state is split into numbered
**state components**, and the `XSAVE` and `XRSTOR` instructions take a 64-bit
**state component bitmap** that tells which components to save or restore.

The bit numbers are architectural: one bit per state component. Bits 62:19 are reserved for future
expansion. Bit 63 is not a state component. It is used only for special bitmap semantics in some
contexts.

A bit in the bitmap corresponds to a piece of CPU extended state, not to a CPU feature. Here are
possible relationships between them.

- **One component per feature**. For example, bit 1 is the SSE component, which includes the
  `XMM0`-`XMM15` registers and `MXCSR` state.
- **Several components per feature**. For example, AMX has two state components: the `TILECFG`
  register (bit 17) and the `TMM0`-`TMM7` tile data registers (bit 18).
- **One component shared by multiple features**. AVX widens the existing SSE `XMM0`-`XMM15`
  registers to `YMM0`-`YMM15`: the same underlying register storage is viewed as either the lower
  128 bits (`XMMn`) or the full 256 bits (`YMMn`). The lower 128 bits stay in the SSE state
  component (bit 1), and the upper 128 bits are the AVX extension state (bit 2). The full AVX
  register state therefore spans both bits 1 and 2.

State components are split into two kinds based on which privilege levels may save and restore them.

- **User components** can be saved and restored at any privilege level. All XSAVE family save and
  restore instructions can handle them.
- **Supervisor components** can be saved and restored only at CPL 0, and only by the privileged
  XSAVE family instructions: `XSAVES` and `XRSTORS`.

### XCR0 and IA32_XSS

`XCR0` is a control register that holds a state component bitmap for user components. A bit set to
1 allows the corresponding component to be handled by XSAVE family save and restore instructions.
For example, `XSAVE` saves the SSE state only when bit 1 is set in both the instruction mask and
`XCR0`. If bit 1 is clear in `XCR0`, `XSAVE` ignores the SSE state. Bits for supervisor components
must be 0 in `XCR0`.

`IA32_XSS` is an MSR that holds the state component bitmap for supervisor components, and bits for
user components must be 0 in `IA32_XSS`. Only `XSAVES` and `XRSTORS` save and restore supervisor
components, and both require CPL 0.

**Note**: `XSAVES` and `XRSTORS` save and restore both user and supervisor components, so they are
governed by both `XCR0` and `IA32_XSS`.

| Criterion             | `XCR0`                                 | `IA32_XSS`                  |
| --------------------- | -------------------------------------- | --------------------------- |
| State Components      | User                                   | Supervisor                  |
| Instructions          | All save and restore instructions      | Only `XSAVES` and `XRSTORS` |
| Read                  | `XGETBV`, any CPL                      | `RDMSR`, CPL 0              |
| Write                 | `XSETBV`, CPL 0                        | `WRMSR`, CPL 0              |

### Features and XSAVE Components

State components are associated with CPU features. The SDM calls features with associated state
components **XSAVE-supported features**.

Some XSAVE-supported features are also **XSAVE-enabled features**. For these features, software may
use the feature only when its state components are enabled in `XCR0`. Otherwise, instructions using
the feature cause the invalid-opcode exception (`#UD`). AVX, AVX-512, and AMX are examples of
XSAVE-enabled features.

Other XSAVE-supported features are not XSAVE-enabled. For them, `XCR0` and `IA32_XSS` control only
state save and restore. Software may use the feature even when its state component is disabled in
`XCR0` or `IA32_XSS`, but the XSAVE family instructions will not save or restore that component.
Examples include x87, SSE, PKRU, PT, CET, UINTR, LBR, and HWP.

### State Component Enumeration

`CPUID` leaf 0xD enumerates the state components the CPU supports and their layout in the XSAVE
area. Software needs it to decide what to enable in `XCR0` and `IA32_XSS`, and how large an XSAVE
buffer to allocate. Sub-leaves 0 and 1 describe the CPU as a whole, and sub-leaf *i* for *i* > 1
describes state component *i*.

| Sub-leaf | Register  | Meaning                                                             |
|----------|-----------|---------------------------------------------------------------------|
| 0        | `EDX:EAX` | Bitmap of the supported user components. Bits settable in `XCR0`.   |
| 0        | `EBX`     | XSAVE area size for the components currently enabled in `XCR0`.     |
| 0        | `ECX`     | XSAVE area size for all supported user components.                  |
| 1        | `EAX`     | Which XSAVE feature set extensions the CPU supports.                |
| 1        | `EBX`     | `XSAVES` area size for the components enabled in `XCR0`+`IA32_XSS`. |
| 1        | `EDX:ECX` | Bitmap of supported supervisor components. Settable in `IA32_XSS`.  |
| *i*      | `EAX`     | Size in bytes of state component *i*.                               |
| *i*      | `EBX`     | Offset of component *i* in the XSAVE area, 0 for supervisor ones.   |
| *i*      | `ECX`     | Flags for state component *i*.                                      |

### Valid XCR0 and IA32_XSS Values

Not every combination of state component bits is a legal `XCR0` or `IA32_XSS` value. Writing an
illegal value with `XSETBV` or `WRMSR` causes `#GP(0)`. The rules are the following.

- Every bit set must correspond to a feature the CPU supports, as enumerated by `CPUID` leaf 0xD.
  Reserved bits must be 0.
- Supervisor state component bits must be 0 in `XCR0`, and user state component bits must be 0 in
  `IA32_XSS`.
- `XCR0` bit 0, the x87 component, must be 1.
- Some components must be enabled together, because their state overlaps or because one feature
  builds on another.
  - AVX (bit 2) requires SSE (bit 1).
  - The three AVX-512 bits (7:5) must all have the same value, and setting them requires the AVX
    bit.
  - The two AMX bits (18:17) must have the same value.

This dependency rule applies only to the `XCR0` and `IA32_XSS` enablement masks. It does not apply
to the instruction mask operand used by the XSAVE family instructions. Software may still choose any
subset of the enabled components when calling `XSAVE`, `XRSTOR`, `XSAVES`, or `XRSTORS`.
