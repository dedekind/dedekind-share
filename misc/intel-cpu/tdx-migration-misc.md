# TDX Live Migration

- **Author**: Artem Bityutskiy
- **Version**: 0.1
- **Date**: 2026-07-03
- **Last updated**: 2026-10-20

**Disclaimer**: These notes reflect my current understanding of TDX live migration concepts and
mechanisms and may contain errors or omissions. They are not an official Intel document. Refer to
the official Intel documentation for authoritative information.

This is not a standalone article. It is a collection of notes and explanations about TDX live
migration, intended for reference from my other documents.

**Terminology Notes**:

- A **seamcall** is a TDX module ABI leaf function invoked with the `SEAMCALL` instruction.
  For example, `TDH.MIG.SETUP` is a seamcall.
- A **tdcall** is a TDX module ABI leaf function invoked with the `TDCALL` instruction.
  Tdcalls include all `TDG.*` leaf functions.
- `SEAMCALL` and `TDCALL` refer to the CPU instructions, while `seamcall` and `tdcall` refer to
  the leaf functions they invoke.

---

## Table of Contents

- [TDX Live Migration](#tdx-live-migration)
  - [Table of Contents](#table-of-contents)
    - [Migration Phases](#migration-phases)
  - [Enumeration](#enumeration)
    - [Individual Features](#individual-features)
    - [Enhanced Import](#enhanced-import)
      - [Pre-Allocation](#pre-allocation)
      - [Page Removal](#page-removal)
      - [Page Relocation](#page-relocation)
      - [NO\_REOWN](#no_reown)
      - [Partial GPA List Processing](#partial-gpa-list-processing)

### Migration Phases

The TDX module divides the TD migration session into two phases: **in-order** and **out-of-order**,
corresponding to the pre-copy and post-copy phases of live migration.

**In-order phase (pre-copy)** happens while the source TD is still running and before it is paused.
The TDX module enforces strict page import ordering to ensure consistency of the migrated memory
state.

**Out-of-order phase (post-copy)** happens after the source TD is paused, the TD and vCPU state is
transferred, and the start token is generated and imported. The TDX module does not enforce import
ordering during this phase because the source TD is paused and cannot dirty pages anymore. This
allows the destination to start execution while remaining memory pages are delivered on demand.

**Note**: for a TD configured with TDX Connect, only the in-order phase is supported. Every TD
private page must be exported during the in-order phase, before `TDH.EXPORT.TRACK` generates the
start token. The out-of-order phase relies on a CPU access faulting on a missing page in the
destination TD to fetch it on demand. A TDI cannot fault, so a DMA to a missing page simply fails.
There is no mechanism to deliver pages on demand for TDIs attached to the destination.

## Enumeration

TDX live migration features are enumerated through the
[TDX_FEATURES0](tdx-structs.md#tdx_features0) and [TDX_FEATURES1](tdx-structs.md#tdx_features1)
registers, read with `TDH.SYS.RD` (by the VMM) or `TDG.SYS.RD` (by the TD).

### Individual Features

Here are the individual features enumerated by
[TDX_FEATURES0](tdx-structs.md#tdx_features0).

- `TD_MIGRATION` (bit 0) indicates that the TDX module supports TD migration.
- `FMS_CONFIG` (bit 17) indicates support for configuring the virtual `CPUID(0x1).EAX` value, which
  reports the TD Family, Model and Stepping. This is needed because a TD may migrate between hosts
  with different native Family/Model/Stepping values, and the guest-visible value must stay the
  same across the migration. The TDX module only allows configuring this value for migratable TDs.
- `PARTITIONED_TD_MIGRATION` (bit 21) indicates support for migrating partitioned TDs, i.e., TDs
  configured with one or more L2 VMs.
- `NON_BLOCKING_EXPORT` (bit 41) indicates support for the non-blocking export mode.
- `SCAN_EXPORT_RESTORE` (bit 43) indicates support for the `EXPORT_RESTORE` operation of the
  `TDH.MEM.SCAN.RANGE` seamcall. It can be used in the abort flow as a simpler and more efficient
  alternative to `TDH.EXPORT.RESTORE`.
- `IMPORT_PAGE_STATUS` (bit 44) indicates that `TDH.IMPORT.MEM` reports more detail in the page
  list entries on output. It sets the `INVALID` bit (63) for provided pages it did not use, and for
  a `CANCEL` that removed a page, it returns the HPA of the removed page with the `REMOVED` bit (61)
  set.
- `DEBUG_RO_TD_MIGRATION` (bit 52) indicates support for combining `ATTRIBUTES.DEBUG` and
  `ATTRIBUTES.MIGRATABLE` (see [ATTRIBUTES](tdx-structs.md#attributes)). Without this bit, the two
  are mutually exclusive. With this bit set, a TD can be both debuggable and migratable, but
  debugging becomes read-only: the host VMM can read TD memory but not write it, and only the
  metadata fields already writable for a non-debuggable TD remain writable.
- `LIST_ERROR_COUNT` (bit 54) indicates that `TDH.EXPORT.BLOCKW`, `TDH.EXPORT.MEM`,
  `TDH.EXPORT.RESTORE` and `TDH.IMPORT.MEM` support version 1, which returns a GPA list error count
  on output. Otherwise, only version 0 is supported.
- `MIG_SETUP` (bit 55) indicates support for MSS, the Migration Setup service extension of the TDX
  module.

### Enhanced Import

**Enhanced import** is a set of TDX module capabilities for TD state import seamcalls. These
capabilities are available when `TDX_FEATURES0.ENHANCED_IMPORT` (bit 60) is set.

When a TDX module supports enhanced import, the following features become available during
import.

- **Pre-allocation**: pre-populate the destination TD with physical pages before their content is
  imported.
- **Page removal**: remove a destination TD page at any point during the in-order import phase.
- **Page relocation**: move a destination TD page to a different physical page.
- **Page demotion**: split a large destination TD page with `TDH.MEM.PAGE.DEMOTE` during the
  in-order import phase.
- **NO_REOWN**: forbid a physical page from changing ownership between the VMM and the TD during
  import.
- **Partial GPA list processing**: let `TDH.IMPORT.MEM` process only part of a migration bundle's
  GPA list per invocation.

#### Pre-Allocation

The `TDH.MEM.PAGE.AUG` seamcall can pre-populate the destination TD with physical pages before
their content is imported. The pages start in the
[PENDING](tdx-misc.md#sept-page-states) state. `TDH.IMPORT.MEM` then imports the migrated pages into
these pre-allocated pages and updates the SEPT entries to the states recorded in the
GPA list entries of the migration bundle.

Note, `TDH.MEM.PAGE.ADD` is prohibited, since the destination TD never goes through the normal
build flow. `TDH.IMPORT.STATE.IMMUTABLE` starts the import session directly from the TD's
`UNINITIALIZED` `OP_STATE`, bypassing the state in which `TDH.MEM.PAGE.ADD` is allowed.

#### Page Removal

The `TDH.MEM.PAGE.REMOVE` seamcall can remove a destination TD page at any point during the
in-order import phase. While an import session is in progress, in both the in-order and out-of-order
phases, the TDX module records removed pages with the migration-only `REMOVED` SEPT entry state,
instead of the regular `FREE` state.

**Note**: In the out-of-order phase, the TD is running and import order is not enforced. The
`REMOVED` state prevents a rollback attack in this phase: without it, the VMM could remove a page
that the TD has modified and import it again from a saved copy of an older migration bundle,
rolling back the page content.

#### Page Relocation

The `TDH.MEM.PAGE.RELOCATE` seamcall can move a destination TD page to a different physical page at
any point during the in-order import phase. The GPA mapping does not change: the seamcall copies
the page content to the new HPA (except for `PENDING` pages), updates the SEPT entry to point there,
and returns the old HPA to the VMM.

#### NO_REOWN

`NO_REOWN` is an input flag of `TDH.IMPORT.MEM` that forbids a physical page from changing ownership
between the VMM and the TD during the seamcall.

A physical page is either:

- TD-owned: private, encrypted with the TD's HKID, mapped by a SEPT entry
- VMM-owned: regular host memory.

Each incoming GPA list entry carries an `OPERATION` field. Without `NO_REOWN`, each operation can
change page ownership:

- `MIGRATE` converts a VMM-owned page to a TD-owned page. The page comes from the destination page
  list, or the migration buffer itself is converted (in-place import). If the page was already
  pre-allocated, `MIGRATE` imports into it and ownership does not change.
- `CANCEL` removes the TD-owned page and converts it back to a VMM-owned page. If
  `TDX_FEATURES0.IMPORT_PAGE_STATUS` is set, the page list entry returns its HPA to the VMM.
- `REMIGRATE` can take a new VMM-owned page to replace one that was previously removed with
  `TDH.MEM.PAGE.REMOVE`, converting it to a TD-owned page again.

With `NO_REOWN` set, `TDH.IMPORT.MEM` no longer changes ownership. Instead, it changes what each
operation accepts and requires:

- `MIGRATE` succeeds only if the page was already pre-allocated (TD-owned), and the destination
  page list is not used.
- `CANCEL` does not hand the page back to the VMM. Instead, it is converted to a pre-allocated
  `PENDING` page, so it stays TD-owned.
- `REMIGRATE` succeeds only if the page is already TD-owned, either because it was imported
  earlier and not locally removed, or because it was removed and then re-pre-allocated with
  `TDH.MEM.PAGE.AUG`.

#### Partial GPA List Processing

With partial GPA list processing, the VMM controls how many GPA list entries of a migration bundle
`TDH.IMPORT.MEM` processes in each run, rather than the whole bundle at once.

The VMM sets `GPA_LIST_INFO.LAST_ENTRY` to the last entry to process in the current run. If it is
before the end of the bundle GPA list, `TDH.IMPORT.MEM` returns `TDX_INTERRUPTED_LIST_END` and the
VMM resumes it later with a higher `LAST_ENTRY`.

**Note**: `TDH.EXPORT.MEM` has no equivalent. There, `GPA_LIST_INFO.FIRST_ENTRY` and `LAST_ENTRY`
select the range of GPA list entries to export, and each new invocation produces a new migration
bundle with its own MBMD.
