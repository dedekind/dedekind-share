# TDX Hardware Layers

- **Author**: Artem Bityutskiy
- **Version**: 0.5
- **Date**: 2026-06-03
- **Last updated**: 2026-11-09

**Disclaimer**: This document describes my current understanding of TDX hardware layers, but may
contain errors or omissions. This is not an official Intel document and should not be treated as
such. For authoritative information, refer to the Intel TDX white paper and official documentation.

**Topics Not Covered**: many topics related to TDX and hardware are not covered, for example:

- **TDX Connect**: how memory protection works with TDX Connect (direct device assignment to a TD).
- **MMU and SEPT walk mechanics**: for example, how the MMU handles private vs. shared GPA and
  the two-EPT hierarchy.
- **ACT mechanism**: client platforms use the Access Control Table (ACT) mechanism, which is not
  covered by this document. This document focuses on non-ACT mechanisms used on Intel server
  platforms.

---

## Table Of Contents

- [TDX Hardware Layers](#tdx-hardware-layers)
  - [Table Of Contents](#table-of-contents)
  - [Introduction](#introduction)
    - [Key Concepts](#key-concepts)
    - [ACT vs Non-ACT Mechanisms](#act-vs-non-act-mechanisms)
    - [Li vs Ci Modes](#li-vs-ci-modes)
  - [Hardware Overview](#hardware-overview)
  - [DRAM](#dram)
    - [What DRAM Stores](#what-dram-stores)
    - [What a Physical Attacker Sees in DRAM](#what-a-physical-attacker-sees-in-dram)
  - [Memory Controller](#memory-controller)
    - [HKID Classes](#hkid-classes)
    - [On Write](#on-write)
    - [On Read](#on-read)
    - [Software Path](#software-path)
    - [DMA Path](#dma-path)
  - [Cache](#cache)
    - [HKID in Cache Tags](#hkid-in-cache-tags)
    - [From HKID to TD Owner Bit](#from-hkid-to-td-owner-bit)
  - [Core](#core)
    - [Modes of Operation](#modes-of-operation)
    - [Physical Address Layout](#physical-address-layout)
    - [VMX Root Mode](#vmx-root-mode)
    - [SEAM Root Mode](#seam-root-mode)
    - [SEAM Non-Root Mode](#seam-non-root-mode)
  - [Summary](#summary)

---

## Introduction

This document walks through TDX hardware layers bottom up, one component at a time: DRAM, memory
controller, cache, core. Each section describes the role of that component in TDX: what it stores,
what it checks, and what it enforces. It covers only the hardware side - the TDX module is not
covered.

The reader is expected to already know about TDX. This document offers a bottom-up overview that
complements the Intel TDX white paper ("Intel Trust Domain Extensions", June 2025). The white paper
covers both hardware and software, taking the opposite angle: top down, from goals to mechanisms.

The hardware descriptions here are simplified. Some details are omitted when they do not change the
fundamental picture.

But before diving into the individual hardware layers, it is helpful to briefly introduce a few key
concepts. They will be discussed in more detail later.

### Key Concepts

- **HKID** (Host Key ID): an identifier for the hardware key used to encrypt the memory content.
- **TD Owner bit**: a per-cache-line 1-bit integrity tag that tells whether the cache line belongs
  to a trust domain (TD) or a shared memory region.
- **MAC** (Message Authentication Code): an additional per-cache-line integrity tag, computed on
  writes, verified on reads to ensure data integrity.
- **Poison bit**: a per-cache-line bit set if data is corrupted (ECC failure) or integrity is
  compromised (MAC failure).

### ACT vs Non-ACT Mechanisms

Server platforms use ECC (Error-Correcting Code) to detect and correct memory errors, and DRAM
is provisioned with extra per-cache-line bits for ECC. TDX on server platforms leverages this
extra room for storing the TD Owner bit and, optionally, the MAC.

Client platforms typically use non-ECC memory, so there are no extra per-cache-line bits available.
TDX has to use a separate, per-memory-controller table called the Access Control Table (ACT) to
store integrity tags.

This document focuses on Non-ACT platforms.

### Li vs Ci Modes

Non-ACT platforms support two integrity modes:

- **Li mode** (Logical Integrity Mode): the integrity tag is the TD Owner bit. No MAC is used.
- **Ci mode** (Cryptographic Integrity Mode): adds a MAC on top of the TD Owner bit.

Li mode needs only the TD Owner bit from the ECC area. In Ci mode, the MAC needs more bits from
that same area, leaving fewer bits available for error correction. This is a tradeoff between
recoverability (RAS) and integrity protection.

---

## Hardware Overview

```text
   Cores                      Devices (DMA)
     |                            |
     |                            |
  +-----------------+             |
  | Cache Hierarchy |<-- snoop ---+
  +-----------------+             |
     |                            |
     |                            |
  +-----------------------------------+
  | Memory Controller (MK-TME engine) |
  +-----------------------------------+
                                  |
                                  |
                          +--------------+
                          |     DRAM     |
                          +--------------+
```

There are two data paths:

- **Software path:** core (MMU) -> cache hierarchy -> memory controller. On a cache hit the request
  is served from the cache and never reaches the memory controller.  On a cache miss the request
  goes to the memory controller.
- **DMA path:** device -> memory controller directly, bypassing the core and its MMU. The memory
  controller snoops the caches to maintain coherency (if the line is cached, the snoop resolves it
  without a DRAM access).

---

## DRAM

All accesses to DRAM go through the memory controller. All software (including the TDX module) and
all devices doing DMA reach DRAM only through the memory controller.

The minimum unit of encryption and integrity is the cache line (64 bytes). Each cache line is
encrypted independently and protected by the TD Owner bit. In Ci mode, each cache line is
additionally protected by a MAC.

The memory controller can operate with memory blocks larger than a cache line (e.g., an entire page
consisting of 64 cache lines) or smaller than a cache line. Transfer sizes on the DRAM bus vary, but
encryption and integrity are still applied independently per cache line.

### What DRAM Stores

DRAM stores the encrypted data (ciphertext) and the metadata, which includes:

- **TD Owner bit**: one bit indicating whether the cache line was written with a private HKID. It
  is part of the integrity tag in both Li and Ci modes.
- **Poison bit**: one bit set when a read detects corrupted or tampered data, either from an
  uncorrectable ECC error or, in Ci mode, a MAC verification failure. The poison indication is
  sticky: once set, it stays in DRAM and is returned on subsequent reads until the whole cache line
  is overwritten.
- **MAC**: a keyed SHA-3-based integrity tag, present only in Ci mode.

**Note**: Neither the encryption key nor the HKID is stored in DRAM. There is no record of which
key produced the ciphertext.

### What a Physical Attacker Sees in DRAM

A physical attacker who reads DRAM (cold-boot, bus probing) sees:

- Ciphertext, not plaintext.
- The TD Owner bit and the poison bit.
- In Ci mode, also the MAC.

In Ci mode, the attacker cannot forge a valid MAC without the MAC key, and modifying, relocating,
or splicing data in DRAM invalidates the MAC, which the memory controller detects on the next
read.

---

## Memory Controller

The memory controller receives DRAM access requests from the cache hierarchy (software path) and
from devices (DMA path). Its security component, the MK-TME engine, performs encryption, decryption,
integrity checking, and TD Owner bit enforcement on every DRAM access.

All requests to the memory controller are processed at cache-line (64-byte) granularity. Each
cache line has its own host physical address (HPA) carrying an HKID in its upper bits. The HKID
selects which encryption key the MK-TME engine uses.

### HKID Classes

The HKID is not just an opaque key index for the memory controller. Every HKID also belongs to one
of two classes, and security checks depend on which class the HKID of the request belongs to.

- **Private HKIDs**: reserved for TDs. Each TD gets one private HKID at creation.
- **Shared HKIDs**: available to all software and devices.

MK-TME stores per-HKID encryption keys in an on-die key table. Keys are generated by the engine and
are not accessible to any software or device.

**Note**: Linux uses only one shared HKID 0.

### On Write

The MK-TME engine performs three operations on every DRAM write:

1. **TD Owner bit assignment.** Set the TD Owner bit based on the HKID class: private HKID
   sets it, shared HKID clears it.
2. **Encryption.** Encrypt the cache line using the memory encryption key selected by the HKID.
3. **MAC generation.** Compute a MAC over the ciphertext, the encryption tweak, and the TD Owner
   bit, then store both ciphertext and MAC in DRAM.

**Note**: The TD Owner bit is not checked on writes. The goal of the TD Owner bit is to
prevent ciphertext disclosure, which is a read-side concern. A write replaces the entire cache line
without reading the old content, so there is nothing to disclose.

**More About the MAC**

The memory controller computes a MAC over the following inputs: ciphertext, the 128-bit encryption
tweak, and the TD Owner bit. The encryption tweak is derived from the physical address together with
a per-HKID seed. The MAC key is generated by hardware at initialization and kept inside the memory
controller.

```text
tweak = T(paddr, seed)
message = ciphertext ∥ tweak ∥ TD Owner
MAC = F(message, MAC_key)
```

Because the tweak depends on the address and the HKID, the MAC cryptographically binds the
ciphertext to both. This prevents undetected relocation or tampering of ciphertext or metadata.

### On Read

The MK-TME engine performs the following operations on every DRAM read, regardless of whether the
request came from the cache hierarchy (software) or from a device (DMA). The actual order and
parallelism are implementation-defined.

1. **TD Owner bit check.** Compare the TD Owner bit against the HKID class of the request. If
   they disagree, the engine returns a fixed pattern instead of the data.
2. **MAC verification.** Recompute the MAC over the ciphertext, the encryption tweak, and the TD
   Owner bit, using the MAC key selected by the HKID. If the recomputed MAC does not match the
   stored MAC, mark the cache line as poisoned.
3. **Decryption.** Decrypt the cache line using the memory encryption key selected by the HKID.

### Software Path

Software requests reach the memory controller as cache read misses, dirty cache line evictions
(write-backs), or immediate writes (write-through).

Writes work just like [On Write](#on-write) describes. The rest focuses on read outcomes.

The TD Owner bit check on reads has four possible cases. The poison behavior differs between Ci mode
and Li mode:

Li mode:

| HKID class | TD Owner bit   | Result                         |
|------------|----------------|--------------------------------|
| Private    | TD-private     | Read is allowed                |
| Private    | Not TD-private | Read fixed pattern + poison    |
| Shared     | TD-private     | Read fixed pattern (no poison) |
| Shared     | Not TD-private | Read is allowed                |

Ci mode:

| HKID class | TD Owner bit   | Result                      |
|------------|----------------|-----------------------------|
| Private    | TD-private     | Read is allowed             |
| Private    | Not TD-private | Read fixed pattern + poison |
| Shared     | TD-private     | Read fixed pattern + poison |
| Shared     | Not TD-private | Read is allowed             |

**Fixed Pattern and Poison**

The MK-TME engine uses two response mechanisms when a TD Owner bit check fails on reads:

- **Fixed pattern**: replace the actual data with a fixed pattern to prevent ciphertext disclosure.
- **Poison**: mark the cache line as corrupted or tampered. Consuming poisoned data typically
  results in a machine check exception (`#MC`).

The MK-TME engine sets the poison bit to signal one of two things:

- **Corruption**: an uncorrectable ECC error in DRAM.
- **Integrity failure**: MAC verification failed (Ci mode only).

Every failed TD Owner bit check returns a fixed pattern. Whether it also returns poison depends on
the mode and the mismatch direction:

- **Private HKID/not-TD-private data read** (both modes): return poison. The TD Owner bit is
  clear, meaning the address was last written with a shared HKID, which indicates a potential
  attack.
- **Shared HKID/TD-private data read in Ci mode**: return poison. The MAC verification would fail
  and trigger poison. Rather than re-computing the MAC, the engine short-circuits based on the TD
  Owner bit mismatch alone.
- **Shared HKID/TD-private data read in Li mode**: no poison. Without a MAC there is no
  tampering evidence, so the fixed pattern alone is sufficient to prevent ciphertext disclosure.

### DMA Path

DMA requests from devices go directly (or via IOMMU) to the memory controller, bypassing the core.
The memory controller snoops the cache hierarchy to maintain coherency: if the line is cached, the
snoop resolves it without a DRAM access. If the line is not cached, the request goes to DRAM.

Two mechanisms prevent DMA from accessing TD-private memory:

- **Private HKID abort.** DMA with a private HKID is aborted. Devices are restricted to shared
  HKIDs. **Note**: This is different for devices assigned to the TD with TDX Connect, but this is
  out of scope of this document.
- **TD Owner bit check.** If a device uses a shared HKID to read a TD-private cache line, the
  TD Owner bit mismatch triggers a fixed pattern response (with poison in Ci mode,
  without in Li mode).

---

## Cache

The cache stores plaintext. The memory controller decrypts and verifies the MAC before filling the
cache, and encrypts and computes a new MAC on eviction. At the cache level there is no encryption,
no MAC, and no TD Owner bit. The cache is also unaware of CPU mode, for example it does not
distinguish the SEAM mode.

A physical address is split into three fields for cache lookup:

```text
  [    Tag     |    Index    |   Offset   ]
    upper bits   middle bits   low 6 bits
```

The offset selects a byte within the 64-byte cache line. The index selects which cache set to
search. The tag is stored alongside the data, and a cache hit occurs only when the incoming tag
matches a stored tag in the selected set.

### HKID in Cache Tags

The HKID is part of the cache tag, so a cache lookup with one HKID cannot hit a line tagged with a
different HKID. This prevents one security domain from reading data cached by another domain. For
example, a snoop from a DMA request with a shared HKID cannot hit a cache line tagged with a private
HKID, because the tags do not match.

**Note**: The cache tag is the physical address bits above the index and offset, and the HKID is
embedded in the uppermost bits of that address.

### From HKID to TD Owner Bit

After a cache line is written to DRAM, the HKID is lost. The HKID is not stored in DRAM. The
TD Owner bit takes over: it records whether the data is TD-private, preserving the
private/shared distinction at the DRAM level.

---

## Core

The core, which includes the MMU, performs address translation and access control before a memory
request reaches the cache. The translation and checks depend on the mode of operation.

### Modes of Operation

The Intel CPU has several modes of operation. Below are the three relevant to this document,
introduced in a simplified manner:

- **VMX root mode**: the core is executing the hypervisor/host. Untrusted in the TDX threat model.
  Full Intel SDM term: VMX root, non-SEAM.
- **SEAM root mode**: the core is executing the TDX module. Full Intel SDM term: SEAM VMX root.
- **SEAM non-root mode**: the core is executing TD guest code. Full Intel SDM term: SEAM VMX
  non-root.

Each mode will be discussed in more detail in the following sections.

Mode transitions:

| From          | To            | How                                                 |
|---------------|---------------|-----------------------------------------------------|
| VMX root      | SEAM root     | `SEAMCALL`                                          |
| SEAM root     | VMX root      | `SEAMRET`                                           |
| SEAM root     | SEAM non-root | VM entry (`VMLAUNCH`, `VMRESUME`)                   |
| SEAM non-root | SEAM root     | VM exit (`TDCALL`, interrupt, SEPT violation, etc.) |
| VMX root      | SEAM non-root | Not possible (must go through SEAM root)            |
| SEAM non-root | VMX root      | Not possible (must go through SEAM root)            |

**Note**: A VM exit takes the TD to the TDX module, not to the VMM. The TDX module then either
resumes the TD or exits to the VMM.

### Physical Address Layout

The HKID occupies the upper bits of the physical address. The HKID field width is configurable.
It is configured at boot and enumerated by `IA32_TME_ACTIVATE.MK_TME_KEYID_BITS`, denoted here as
***k***. The physical address width is enumerated by `CPUID.80000008H:EAX[7:0]`, denoted here as
***m***.

Bits ***m-k-1***:***0*** are the physical address bits, bits ***m-1:m-k*** are the HKID bits.

The following diagrams use values from an Emerald Rapids Xeon system, where ***m = 52*** and
***k = 6***

```text
Bit: 51        46 45                              0
     +-----------+---------------------------------+
     |   HKID    |       Physical Address          |
     |  (6 bits) |          (46 bits)              |
     +-----------+---------------------------------+
```

There is no dedicated private/shared flag inside the HKID field. The split into the two classes is a
numeric range split enumerated by `IA32_MKTME_KEYID_PARTITIONING`:

- HKIDs 1 to `NUM_MKTME_KEYIDS` are shared.
- HKIDs `NUM_MKTME_KEYIDS + 1` to `NUM_MKTME_KEYIDS + NUM_TDX_PRIV_KEYIDS` are private.

A separate field, `IA32_TME_ACTIVATE.TDX_RESERVED_KEYID_BITS`, denoted here as ***p***, gives the
number of uppermost HKID bits that only SEAM mode may set (***p <= k***, and ***p*** is not
necessarily 1). Because those bits must be zero outside SEAM, every shared HKID fits below
***2^(k-p)***.

On the Emerald Rapids system, ***k = 6*** and ***p = 1***, so the ranges work out as:

- HKID 0: the default key (no encryption)
- HKID 1-31: shared keys (available but not used by Linux)
- HKID 32: TDX module global key on this system. The specific HKID is not architecturally fixed,
  it is just that Linux assigns the first HKID in the private range to be the TDX module global key,
  which happens to be 32 in this case.
- HKID 33-63: TD keys (one per TD, max 31 concurrent TDs on this system)

**MAXPHYADDR**

`MAXPHYADDR` is an architectural parameter defined in Intel SDM that specifies the maximum physical
address width supported by the processor.

- Outside SEAM mode, `MAXPHYADDR` is reduced by `IA32_TME_ACTIVATE.TDX_RESERVED_KEYID_BITS`
  (***p***), so the core cannot issue a memory access with a private HKID.
- In SEAM mode, `MAXPHYADDR` is not reduced, so the TDX module can use private HKIDs.

Bits above `MAXPHYADDR` are reserved in page table entries and must be zero, otherwise the CPU
generates a `#PF`.

### VMX Root Mode

In VMX root mode, the core uses standard host paging, which translates host virtual addresses (HVA)
to host physical addresses (HPA) via CR3. No EPT is involved. All memory accesses carry a shared
HKID. Private HKIDs are inaccessible because `MAXPHYADDR` is reduced in VMX root mode.

On the Emerald Rapids system, ***p = 1***, so `MAXPHYADDR` = 51 outside SEAM and bit 51 is reserved:

```text
VMX root mode (MAXPHYADDR = 51):

Bit: 51 50     46 45                              0
     +--+--------+---------------------------------+
     |R | Shared |     Addressable from PTEs       |
     |  | HKID   |       MAXPHYADDR = 51           |
     +--+--------+---------------------------------+
      ^
      Reserved (#PF if set)
```

So the core in VMX root mode can only issue memory accesses with shared HKIDs. The hypervisor can
only manage TD memory through a `SEAMCALL` (e.g., `TDH.MEM.PAGE.ADD`).

### SEAM Root Mode

In SEAM root mode, the core executes SEAM code, which in this context means the TDX module. Like in
VMX root mode, it uses standard host paging (HVA to HPA via CR3) with no EPT, but with its own page
tables stored in the `SEAMRR`-protected range. `SEAMRR` (Secure Arbitration Mode Range Register) is
a hardware range register that blocks all accesses from outside SEAM mode. The hardware encrypts all
TDX module physical memory with the TDX module global key (HKID 32 on the Emerald Rapids system).

Unlike VMX root mode, `MAXPHYADDR` is not reduced in SEAM root mode. The core can address the full
physical address range (52 bits on the Emerald Rapids system), including the private HKID bits,
which allows the TDX module to perform operations such as:

- Build the Secure EPT (SEPT) that maps guest-physical addresses (GPA) to host-physical addresses
  (HPA) for each TD.
- Write the HKID of the TD into the Virtual Machine Control Structure (VMCS) so the hardware knows
  which key to use when entering SEAM non-root mode.

SEAM mode also grants access to instructions unavailable to other modes. `PCONFIG` is one example:
it programs encryption keys into the MK-TME engine and cannot be executed outside SEAM mode. The
TDX module uses it to provision the private key of each TD.

The TDX module physical pages fall into two categories:

- **Pages inside SEAMRR** (e.g., code and stack): set up by the SEAM loader at boot. Neither the
  VMM, TDs, nor devices can access these pages. They have two layers of protection: `SEAMRR` blocks
  outside-SEAM accesses, and the TDX module global key encrypts them.
- **VMM-donated pages** (e.g., PAMT and SEPT): donated by the VMM during TD lifecycle operations.
  Not inside `SEAMRR`, so the TDX module global key is the only protection. The VMM cannot use it
  because `MAXPHYADDR` reduction makes private HKIDs unreachable. TDs cannot use it because the
  hardware restricts each TD to its own assigned HKID.

### SEAM Non-Root Mode

In SEAM non-root mode, the core executes TD guest code. Address translation uses two stages: guest
paging (GVA to GPA via CR3) followed by EPT (GPA to HPA). The GPA contains a shared bit that
selects between two types of memory:

- **Private memory**: uses the SEPT, managed exclusively by the TDX module and not accessible to the
  VMM or TD.
- **Shared memory**: uses the shared EPT, managed by the VMM. Both the TD and the VMM can access
  this memory.

**Note**: The shared bit and the HKID live in different address spaces (GPA and HPA respectively),
so they do not interfere with each other.

For private memory, the MMU adds the HKID of the TD into the upper bits of the HPA produced by the
SEPT walk. The HKID comes from a VMCS field that was configured when the TD was created.

```text
Private memory access:

GVA ------> GPA -------> HPA -----------------> HKID + HPA
     CR3         SEPT         HKID from VMCS
```

For shared memory, the MMU walks the shared EPT instead. The VMM manages the shared EPT and maps
GPAs to HPAs with shared HKIDs.

```text
Shared memory access:

GVA ------> GPA -------> HPA (shared HKID)
     CR3    Shared EPT
```

---

## Summary

- **DRAM**: stores only ciphertext, the TD Owner bit, and the MAC. No keys, no HKIDs.
- **Memory controller**: the MK-TME engine encrypts and decrypts data (confidentiality), computes
  and verifies the MAC (integrity), and checks the TD Owner bit against the HKID class (access
  control) on every DRAM access.
- **Cache**: the HKID in the cache tag prevents cross-domain cache hits. No explicit access control
  logic is needed.
- **Core**: controls which HKIDs are reachable. VMX root mode can only use shared HKIDs
  (`MAXPHYADDR` reduction). SEAM non-root mode uses the HKID of the TD (from the VMCS). SEAM root
  mode has unrestricted access.

Each layer addresses the threats visible at its own level and together they form a layered defense:

- **DRAM**: no plaintext is exposed to a physical attacker.
- **Memory controller**: prevents ciphertext disclosure and detects tampering.
- **Cache**: prevents cross-domain cache snooping.
- **Core**: enforces mode-based access control - which HKIDs, instructions, and address translation
  structures are available depends on the current mode.

The TDX module adds software-level protections on top of these hardware mechanisms (e.g., PAMT, page
acceptance), but those are outside the scope of this document.
