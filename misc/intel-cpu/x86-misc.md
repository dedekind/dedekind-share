# Intel CPU

- **Author**: Artem Bityutskiy
- **Version**: 0.1
- **Date**: 2026-07-13
- **Last updated**: 2026-10-22

**Disclaimer**: This document reflects my current understanding of various Intel CPU concepts and
mechanisms, but it may contain errors or omissions. It is not an official Intel document and should
not be treated as such. For full and authoritative information, refer to the Intel SDM.

This is not a standalone article. It is a collection of notes and explanations about Intel CPU
architecture, intended for reference from my other documents.

---

## Table of Contents

- [Intel CPU](#intel-cpu)
  - [Table of Contents](#table-of-contents)
  - [Linear and Virtual Addresses](#linear-and-virtual-addresses)
  - [GLA and GVA](#gla-and-gva)
  - [Memory Types](#memory-types)
  - [Memory Cache Control](#memory-cache-control)
    - [MTRRs](#mtrrs)
    - [PAT](#pat)
    - [Non-VM Accesses](#non-vm-accesses)
    - [EPT Memory Type](#ept-memory-type)
    - [VM Accesses](#vm-accesses)
  - [XSAVE and State Components](#xsave-and-state-components)
    - [XCR0 and IA32\_XSS](#xcr0-and-ia32_xss)
    - [Features and XSAVE Components](#features-and-xsave-components)
    - [State Component Enumeration](#state-component-enumeration)
    - [Valid XCR0 and IA32\_XSS Values](#valid-xcr0-and-ia32_xss-values)
  - [EPTP](#eptp)
    - [EPTP Switching](#eptp-switching)
  - [Virtualization Exception Information Area](#virtualization-exception-information-area)

---

## Linear and Virtual Addresses

In Intel terminology the x86 address translation chain is logical address, then linear address
after segmentation, then physical address after paging. Virtual address is a generic term for the
address translated by paging.

```text
logical address           linear address          physical address
(selector:offset) ------> (virtual address) ------> (physical address)
                 segment                 paging
                  base
```

With flat segmentation, which modern operating systems such as Linux use, the segment base is 0,
so linear and virtual addresses refer to the same address.

## GLA and GVA

**GLA** (Guest Linear Address) is the Intel term for a guest
[linear address](#linear-and-virtual-addresses). The SDM uses it, for example, in VMCS field names.

**GVA** (Guest Virtual Address) is the generic term used by other architectures and operating system
documentation. With flat segmentation, GLA and GVA refer to the same address. Paging translates it
to a **GPA** (Guest Physical Address).

---

## Memory Types

The Intel SDM describes each memory type by its caching behavior, speculative reads, and ordering
model.

- **Caching behavior**: whether accesses use the CPU caches and whether writes use write-back
  caching.
- **Speculative loads**: whether the CPU may start a load before knowing if its result will be
  needed.
- **Ordering model**: which pairs of loads and stores are guaranteed to remain in instruction order.
  It has three possible values:
  - **Strong ordering**:
    - Load vs load: loads are observed in instruction order.
    - Store vs store: stores become visible in instruction order.
    - Load vs store: loads are observed and stores become visible in instruction order, regardless
      of whether the address is the same or different.
  - **Speculative processor ordering**:
    - Load vs load: loads are observed in instruction order.
    - Store vs store: stores become visible in instruction order (with exceptions, see Note 1).
    - Load vs store:
      - Load followed by store: the store never becomes visible before the load is observed.
      - Store followed by load, same address: the load always returns the value written by the
        store.
      - Store followed by load, different address: the load may be observed before the store
        becomes visible.
  - **Weak ordering**:
    - Load vs load: no order guarantee.
    - Store vs store: stores may be delayed, combined, and become visible out of order.
    - Load vs store:
      - Load followed by store: no order guarantee.
      - Store followed by load, same address: the load always returns the value written by the
        store.
      - Store followed by load, different address: no order guarantee.

The Intel SDM defines the following memory types:

| Type | Name            | Cacheable  | Write-back | Spec. loads | Ordering model                 |
|------|-----------------|------------|------------|-------------|--------------------------------|
| UC   | Uncacheable     | No         | No         | No          | Strong ordering                |
| WC   | Write combining | No         | No         | Yes         | Weak ordering                  |
| WP   | Write protected | Reads only | No         | Yes         | Speculative processor ordering |
| WT   | Write through   | Yes        | No         | Yes         | Speculative processor ordering |
| WB   | Write back      | Yes        | Yes        | Yes         | Speculative processor ordering |

**Linux examples**:

- The `intel-lpss` driver uses UC memory type (`ioremap_uc()`) for its private configuration
  registers.
- The `i915` graphics driver uses WC memory type (`ioremap_wc()`) for mapping some of the GPU
  memory.
- Regular kernel memory (e.g., allocated with `kmalloc()`) uses WB memory type.

**Note 1**: Ordinary stores, often called temporal stores, do not carry a non-temporal hint. For
memory types that cache writes, their data may be kept in the CPU caches. Non-temporal move
instructions are intended for data unlikely to be reused soon and avoid placing it in the caches.
On P6 and newer Intel processors, writes to write-back cacheable memory are not reordered with
other writes, except for streaming stores using non-temporal move instructions and string
operations.

**Note 2**: The name "Speculative processor ordering" can be misleading. It describes the ordering
model, not whether loads may be speculative. WC permits speculative loads but uses weak ordering.

**Note 3**: WC stores use write-combining buffers, separate from the L1, L2, and L3 caches and the
store buffer. These buffers allow stores to be combined. The SDM does not guarantee ordering between
successive WC buffer evictions or between partial writes.

**Note 4**: The SDM also defines UC- as a memory type, but I think of it as resolving to either UC
or WC, depending on the MTRR/PAT combination. See [Memory Cache Control](#memory-cache-control) for
more information.

## Memory Cache Control

Three mechanisms control which [memory type](#memory-types) applies to a given memory access.

- **MTRRs**: per physical address range control.
- **PAT**: per [linear address](#linear-and-virtual-addresses) control.
- **EPT memory type**: per [GPA](#gla-and-gva) control, used only for VMs.

### MTRRs

The **MTRRs** (Memory Type Range Registers) are MSRs that assign memory types to physical address
ranges, independently of page table mappings. Firmware normally configures them to distinguish
RAM from ROM and memory-mapped I/O.

- **Fixed-range MTRRs**: 11 registers assign memory types to predefined subranges in the first
  1MiB of physical memory. For example, `IA32_MTRR_FIX16K_80000` configures memory types for the
  range 0x80000-0x9FFFF. `IA32_MTRRCAP.FIX` indicates whether the CPU supports them.
- **Variable-range MTRRs**: specify memory types for configurable physical address ranges. Each
  range uses a pair of registers. `IA32_MTRR_PHYSBASEn` specifies its base address and memory type,
  and `IA32_MTRR_PHYSMASKn` contains the mask used to determine the range. The number of supported
  ranges varies by CPU and is reported by the `VCNT` field of `IA32_MTRRCAP`. For example, on my
  Emerald Rapids Xeon processor there are 10 variable-range MTRRs.
- **Default type**: `IA32_MTRR_DEF_TYPE` specifies the memory type for physical address ranges not
  covered by a fixed-range or variable-range MTRR. Its `E` bit enables MTRRs. When `E` is set, its
  `FE` bit enables fixed-range MTRRs.

### PAT

The **PAT** (Page Attribute Table) lets the OS select a memory type for each page mapping through
its page table entries. Unlike MTRRs, which assign memory types to physical address ranges, PAT
assigns them per linear address mapping.

The `IA32_PAT` MSR (0x277) contains eight 8-bit fields (`PA0`-`PA7`). Each field encodes one
memory type.

Page table entries include the `PAT`, `PCD`, and `PWT` bits. `PCD` means page-level cache disable,
and `PWT` means page-level write-through. `PCD` and `PWT` are bits 4 and 3, respectively. `PAT` is
bit 7 in a 4KiB page table entry and bit 12 in an entry for a larger page.

Together, `PAT`, `PCD`, and `PWT` form a 3-bit index that selects one of the eight `IA32_PAT`
fields. `PAT` is the high bit, `PCD` the middle bit, and `PWT` the low bit.

### Non-VM Accesses

This subsection covers how MTRR and PAT combine for non-VM memory accesses. VM accesses use a
different mechanism and are covered later in this section.

For non-VM accesses, the CPU combines the MTRR type for the physical address with the PAT type
selected by the page table entry. The combination depends on two cache-control bits in `CR0`:

- `CD` (Cache Disable, bit 30) controls caching.
- `NW` (Not Write-through, bit 29) controls the cache write policy.

We consider only what the SDM calls normal caching, with both bits clear (`CR0.CD = 0` and
`CR0.NW = 0`). In this configuration, write-back is enabled, subject to the MTRR and PAT types.

The combination rules are not entirely straightforward, but it helps to think about them using the
same two ideas that define each [memory type](#memory-types) above: cacheability and the ordering
model.

- Non-cacheable:
  - **UC**: no cache, no write-combining buffers, strong ordering.
  - **WC**: no cache, loads and stores go through write-combining buffers, weak ordering.
- Cacheable:
  - **WP**: cache reads but never cache writes, speculative processor ordering.
  - **WT**: write through to memory on every store, speculative processor ordering.
  - **WB**: write back normally, speculative processor ordering.

The order within each group also follows an intuition. It tracks how much caching is involved, from
UC, which uses no caching, to WB, which uses full caching. This is only an intuition, not a strict
rule.

In general, PAT takes precedence over MTRR. PAT UC and PAT WC need no caching support from MTRR,
so they always win.

The first exception arises when PAT requests caching that MTRR does not support. PAT WP and PAT WT
use the cache, but MTRR UC and MTRR WC do not support that, so the effective type is UC. The
intuition is that WC (weak ordering) lets stores be delayed and combined, while WP and WT do not.
UC (strong ordering) is therefore the safer choice.

PAT WB is the second exception. It never takes precedence, and for any MTRR type the effective type
is the MTRR type. I cannot offer a simple intuition for this case, so I treat it as an Intel CPU
quirk.

PAT UC- is the third exception. It resolves to either UC or WC depending on the MTRR type. I cannot
offer a simple intuition for it either. The SDM notes that some of these combinations were left
undefined in the past, so I treat it as another Intel CPU quirk.

Here is the complete combination table from the Intel SDM.

| MTRR type / PAT type  | UC | UC- | WC | WP | WT | WB |
|-----------------------|----|-----|----|----|----|----|
| UC                    | UC | UC  | WC | UC | UC | UC |
| WC                    | UC | WC  | WC | UC | UC | WC |
| WP                    | UC | WC  | WC | WP | WT | WP |
| WT                    | UC | UC  | WC | WP | WT | WT |
| WB                    | UC | UC  | WC | WP | WT | WB |

### EPT Memory Type

For VM accesses, EPT translates a GPA to a host physical address. The EPT memory type is encoded in
bits 5:3 of the final EPT entry used for the translation. This is separate from the memory type
field in the [EPTP](#eptp), which controls accesses to the EPT paging structures.

### VM Accesses

For a VM, the effective memory type is determined by only two inputs: the VM PAT type and the EPT
memory type. Two translation steps are involved.

- GVA is translated to GPA via the VM page tables. This step selects the VM PAT type from the VM
  `IA32_PAT`, without combining it with any MTRR type.
- GPA is translated to HPA via EPT. This step determines the effective memory type from the VM PAT
  type and the EPT memory type, as described below. The host MTRRs have no effect here either.

Bit 6 of the final EPT entry is called `Ignore PAT memory type`:

- If the bit is 1, the CPU uses the EPT type alone, ignoring the VM PAT type entirely.
- If the bit is 0, the CPU combines the EPT and VM PAT types using the same matrix as for
  [non-VM accesses](#non-vm-accesses), but the EPT type replaces the MTRR type.

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
| 1        | `EAX`     | XSAVE feature set extensions supported by the CPU.                  |
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

---

## EPTP

The **EPTP** (Extended Page Table Pointer) is the VMCS field that points to the root of the EPT
tree the CPU walks to translate a GPA into an HPA. Besides the root pointer it carries a few
controls for the walk itself.

| Bits  | Field                   | Meaning                                                    |
|-------|-------------------------|------------------------------------------------------------|
| 2:0   | Memory type             | Memory type for EPT structure accesses: 0 is UC, 6 is WB.  |
| 5:3   | EPT depth               | One less than the EPT levels: 3 is 4-level, 4 is 5-level.  |
| 6     | Accessed and dirty      | Enables the accessed and dirty flags in EPT entries.       |
| 7     | Supervisor shadow stack | Enables supervisor shadow stack access rights.             |
| 11:8  | Reserved                | Must be 0.                                                 |
| 51:12 | Root HPA                | HPA of the top-level EPT page, 4KiB aligned.               |
| 63:52 | Reserved                | Must be 0.                                                 |

The root HPA occupies bits MAXPHYADDR-1:12. The table assumes a MAXPHYADDR of 52. With a smaller
MAXPHYADDR, the bits from MAXPHYADDR to 51 are reserved too.

### EPTP Switching

**EPTP Switching** lets the VMM prepare a list of up to 512 EPTPs, from which the guest picks one
with the `VMFUNC` instruction, without a [VM exit](tdx-misc.md#vm-exit-and-td-exit).

Each EPTP is a different view of guest memory, with its own mappings and permissions. The feature
targets use cases such as security agents running inside the guest, which switch views depending on
the situation.

**Linux Note**: KVM does not expose EPTP switching to VMX guests running directly on it, so these
guests cannot switch EPTs. KVM supports the feature for nested virtualization, where the guest of
KVM is itself a hypervisor that offers EPTP switching to its own guests. KVM emulates it in
software: it leaves the hardware feature disabled, so `VMFUNC` in the nested guest causes a VM
exit to KVM. KVM then reads the requested EPTP from the list and switches the mappings. The nested
guest sees the architectural behavior, but each switch still incurs a VM exit.

---

## Virtualization Exception Information Area

The **virtualization exception information area** is a memory area that carries the details of a
[#VE](tdx-misc.md#ve). TDX specifications call it `VE_INFO`, and this section uses the same name.
It is per vCPU, and the VMCS of the vCPU holds its physical address. The first 34 bytes are
architectural, and the component that raises the `#VE` fills them in: the CPU for an EPT violation,
or software that injects a `#VE`, such as the TDX module (see [VE_INFO](tdx-structs.md#ve_info)).

The Intel SDM requires the `VE_INFO` address to be 4KiB aligned, but does not define a size for the
area.

**Linux Note**: KVM does not use `#VE` for VMX guests. It enables the `#VE` for EPT violations only
with the `CONFIG_KVM_INTEL_PROVE_VE` debug option, which intercepts every `#VE` and warns about it,
to check that KVM never lets a guest receive one. With this option, KVM allocates a 4KiB page per
vCPU for `VE_INFO`, defines only the architectural fields, and never uses the rest of the page.

`EXIT_REASON`, `EXIT_QUALIFICATION`, `GLA`, and `GPA` hold the values that would have gone into the
VMCS if a VM exit had happened instead of the `#VE`.

| Offset | Field                | Meaning                                                          |
|--------|----------------------|------------------------------------------------------------------|
| 0      | `EXIT_REASON`        | The VM exit reason, such as EPT violation (48) or MSR read (31). |
| 4      | `VALID`              | 0 means `VE_INFO` is free, 0xFFFFFFFF means it holds a `#VE`.    |
| 8      | `EXIT_QUALIFICATION` | Exit qualification value.                                        |
| 16     | `GLA`                | Guest linear address.                                            |
| 24     | `GPA`                | Guest physical address.                                          |
| 32     | `EPTP_INDEX`         | Index of the active EPTP in the EPTP list.                       |

`EXIT_REASON` tells what caused the `#VE`. For an EPT violation, the CPU raises the `#VE` and writes
48. This is the only value the CPU writes. But a `#VE` can also be injected by software, and then
the value is the exit reason of the operation the software virtualized, such as MSR read (31) or
`CPUID` (10).

`VALID` marks the `VE_INFO` area as "occupied". The component that raises the `#VE` sets it to
0xFFFFFFFF, and the guest writes 0 back when it is done. While `VALID` is not 0, an EPT violation
causes a VM exit instead of a `#VE`.

`EXIT_QUALIFICATION` carries extra details, and its meaning depends on `EXIT_REASON`. For an EPT
violation it includes whether the access was a read, a write, or an instruction fetch, and several
other bits.

`GLA` is the address the guest used, before paging translated it (see [GLA and GVA](#gla-and-gva)).
For an EPT violation, it is valid only if the "guest linear address valid" bit (bit 7) of
`EXIT_QUALIFICATION` is 1.

`EPTP_INDEX` identifies which EPTP in the list was active, so the handler knows which EPT the
violation happened in. See [EPTP](#eptp) for details.
