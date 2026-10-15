# TDX Live Migration: Miscellaneous

- **Author**: Artem Bityutskiy
- **Version**: 0.1
- **Date**: 2026-07-03
- **Last updated**: 2026-11-04

**Disclaimer**: This document reflects my current understanding of TDX live migration concepts and
mechanisms and may contain errors or omissions. It is not an official Intel document. Refer to the
official Intel documentation for authoritative information.

This is not a standalone article. It is a supporting document that explains TDX live migration
concepts, intended for reference from my other documents.

**Terminology Notes**:

- A **seamcall** is a TDX module ABI leaf function invoked with the `SEAMCALL` instruction. For
  example, `TDH.MIG.SETUP` is a seamcall.
- A **tdcall** is a TDX module ABI leaf function invoked with the `TDCALL` instruction. Tdcalls
  include all `TDG.*` leaf functions.
- `SEAMCALL` and `TDCALL` refer to the CPU instructions, while seamcall and tdcall refer to the leaf
  functions they invoke.
- **HPA** is a [host physical address](x86-misc.md#host-addresses). The TDX specs often call an HPA
  of a page owned by the VMM a "shared HPA", but this document uses "HPA".

---

## Table of Contents

- [TDX Live Migration: Miscellaneous](#tdx-live-migration-miscellaneous)
  - [Table of Contents](#table-of-contents)
  - [Migration Sessions](#migration-sessions)
    - [Setup Session](#setup-session)
    - [Migration Session](#migration-session)
    - [Export Session](#export-session)
    - [Import Session](#import-session)
  - [Non-Blocking vs Write-Blocking Export](#non-blocking-vs-write-blocking-export)
  - [Migration Phases](#migration-phases)
  - [Pre-Copy vs Post-Copy Migration](#pre-copy-vs-post-copy-migration)
  - [Post-Copy Support](#post-copy-support)
  - [Migration Epochs](#migration-epochs)
    - [Per Page Epoch](#per-page-epoch)
  - [Migration Setup Service](#migration-setup-service)
    - [Sessions and Threads](#sessions-and-threads)
  - [Migration Policy](#migration-policy)
  - [Enumeration](#enumeration)
    - [Individual Features](#individual-features)
    - [Enhanced Import](#enhanced-import)
      - [Pre-Allocation](#pre-allocation)
      - [Page Removal](#page-removal)
      - [Page Relocation](#page-relocation)
      - [Page Demotion](#page-demotion)
      - [NO\_REOWN](#no_reown)
      - [Partial GPA List Processing](#partial-gpa-list-processing)
  - [Configuration](#configuration)
    - [TDX Module Configuration](#tdx-module-configuration)
      - [Configuration Flow](#configuration-flow)
      - [MSS Configuration](#mss-configuration)
      - [Export Mode](#export-mode)
    - [TD Creation](#td-creation)
    - [Migration Streams](#migration-streams)

---

## Migration Sessions

The TDX module divides the live migration of a TD into two sessions: the setup session and the
migration session. Both sessions run on the source and on the destination, and the migration session
can start only after the setup session completes on both hosts.

### Setup Session

During the setup session, the source and destination TDX modules exchange encrypted messages through
the VMM. The VMM also delivers attestation quotes and migration policy inputs when requested. Here
is how the process looks in a simplified form.

- The VMM runs the [TDH.MIG.SETUP](tdx-migration-seamcalls.md#tdhmigsetup) seamcall repeatedly on
  the source and the destination. The TDX modules exchange encrypted messages through the VMM.
- The process repeats until the seamcall returns success or an unrecoverable error.
- The VMM can cancel an active setup session with
  [TDH.MIG.SETUP.ABORT](tdx-migration-seamcalls.md#tdhmigsetupabort).

The setup session has the following goals:

- Load the migration policy, see [Migration Policy](#migration-policy).
- Run mutual attestation between the source and the destination.
- Establish a secure channel and generate the migration keys.

The [Migration Setup Service](#migration-setup-service) section describes how the TDX module runs
the setup session.

### Migration Session

During the migration session, the source TDX module exports the TD state as encrypted bundles, and
the destination TDX module imports them. The migration keys that the setup session generated protect
the bundles. Here is how the process looks in a simplified form.

- The session starts with the immutable TD state. The source exports it with
  [TDH.EXPORT.STATE.IMMUTABLE](tdx-migration-seamcalls.md#tdhexportstateimmutable). The destination
  imports it with [TDH.IMPORT.STATE.IMMUTABLE](tdx-migration-seamcalls.md#tdhimportstateimmutable).
- The source exports the rest of the TD state as encrypted bundles. Different seamcalls export
  different parts of the state:
  - [TDH.EXPORT.MEM](tdx-migration-seamcalls.md#tdhexportmem): Memory pages.
  - [TDH.EXPORT.STATE.TD](tdx-migration-seamcalls.md#tdhexportstatetd): Mutable TD state.
  - [TDH.EXPORT.STATE.VP](tdx-migration-seamcalls.md#tdhexportstatevp): vCPU state.
- The VMM transports the bundles to the destination.
- The destination imports the bundles with the matching import seamcalls, such as
  [TDH.IMPORT.MEM](tdx-migration-seamcalls.md#tdhimportmem).
- Eventually, the source TD pauses, and the destination TD takes over, ending the migration session.

The session has two phases, in-order and out-of-order, described in
[Migration Phases](#migration-phases).

The source and the destination run different parts of the migration session. The source runs the
export session and the destination runs the import session. The subsections below describe them.

### Export Session

The **export session** is the part of the [migration session](#migration-session) that runs on the
source. It starts with
[TDH.EXPORT.STATE.IMMUTABLE](tdx-migration-seamcalls.md#tdhexportstateimmutable), which exports the
immutable TD state and moves the source TD to the `LIVE_EXPORT`
[operation state](tdx-misc.md#td-operation-states), where the TD keeps running while the VMM exports
the memory.

The export session ends in one of these ways:

- [TDH.EXPORT.ABORT](tdx-migration-seamcalls.md#tdhexportabort) aborts it and returns the source TD
  to `RUNNABLE`.
- There is no seamcall that explicitly ends the export session on success. The export session
  implicitly ends when the destination TD takes over and the source TD is torn down.

### Import Session

The **import session** is the part of the [migration session](#migration-session) that runs on the
destination. It starts with
[TDH.IMPORT.STATE.IMMUTABLE](tdx-migration-seamcalls.md#tdhimportstateimmutable), which imports the
immutable TD state and moves the destination TD from `UNINITIALIZED` to `MEMORY_IMPORT`.

The import session ends in one of these ways:

- [TDH.IMPORT.END](tdx-migration-seamcalls.md#tdhimportend) ends it successfully and moves the
  destination TD to `RUNNABLE`.
- [TDH.IMPORT.ABORT](tdx-migration-seamcalls.md#tdhimportabort) aborts it and moves the destination
  TD to `FAILED_IMPORT`.
- An import failure ends it. Before [TDH.IMPORT.COMMIT](tdx-migration-seamcalls.md#tdhimportcommit),
  the failure moves the destination TD to `FAILED_IMPORT`. In `LIVE_IMPORT`, it moves the TD to
  `RUNNABLE`.

---

## Non-Blocking vs Write-Blocking Export

The TDX module specification defines two page modification tracking mechanisms:

- **Non-blocking export** (TDX spec name: Non-Blocking Memory Export): Detects page modifications by
  scanning the SEPT for the Accessed and Dirty bits and for the page state (see
  [SEPT Page States](tdx-misc.md#sept-page-states)).
- **Write-blocking export** (TDX spec name: Write-Blocking Based Memory Export): Blocks a page for
  writing before exporting it. A TD write to the page causes an EPT violation TD exit, and the VMM
  is expected to unblock the page and later block and re-export it. This mode is outside the scope
  of this document.

The choice between the two modes is a TDX module configuration parameter that the VMM selects during
TDX module initialization, not a per-TD or per-migration choice. The default is write-blocking. A
TD-preserving TDX module update may change the mode to non-blocking if write-blocking export has not
been used before.

In the current TDX module implementation, [post-copy migration](#pre-copy-vs-post-copy-migration)
works only with write-blocking export. See [Post-Copy Support](#post-copy-support) for details.

---

## Migration Phases

The TDX module divides the TD migration session into two phases: **in-order** and **out-of-order**.
They differ in whether the TDX module enforces the order in which the destination imports pages.

**In-order phase** starts while the source TD is still running and continues after the VMM pauses
it, until the VMM requests the start token. The VMM exports the remaining memory and the TD and vCPU
state during the pause. The TDX module enforces strict page import ordering to ensure consistency of
the migrated memory state.

**Out-of-order phase** begins when the source generates the start token with
[TDH.EXPORT.TRACK](tdx-migration-seamcalls.md#tdhexporttrack) and the destination imports it with
[TDH.IMPORT.TRACK](tdx-migration-seamcalls.md#tdhimporttrack). The source enters `POST_EXPORT`, and
the destination enters `POST_IMPORT`. The TDX module does not enforce import ordering during this
phase because the source TD is paused and cannot dirty pages anymore.

The destination cannot run in `POST_IMPORT`. The VMM can call
[TDH.IMPORT.COMMIT](tdx-migration-seamcalls.md#tdhimportcommit) to move it to `LIVE_IMPORT`, where
it can run. Alternatively, the VMM can call
[TDH.IMPORT.END](tdx-migration-seamcalls.md#tdhimportend) directly from `POST_IMPORT` to end the
[import session](#import-session) and move the TD to `RUNNABLE`.

The phases correspond to these [TD operation states](tdx-misc.md#td-operation-states):

- In-order phase:
  - Source: `LIVE_EXPORT` and `PAUSED_EXPORT`.
  - Destination: `MEMORY_IMPORT` and `STATE_IMPORT`.
- Out-of-order phase:
  - Source: `POST_EXPORT`.
  - Destination: `POST_IMPORT` and `LIVE_IMPORT`.

---

## Pre-Copy vs Post-Copy Migration

This document distinguishes the TDX migration phases from the migration types.

- [Phases](#migration-phases), **in-order** and **out-of-order**, are stages of the migration
  session defined by the TDX module.
- Migration types, **pre-copy** and **post-copy**, define whether the destination TD starts running
  after the full state (CPU and memory) is transferred or while some state remains on the source.

Here is how the two migration types relate to [operation states](tdx-misc.md#td-operation-states) of
the destination TD:

- In **pre-copy migration**, the destination TD starts only after the full state is transferred from
  the source. The destination TD goes through the in-order phase and enters `POST_IMPORT`, but skips
  `LIVE_IMPORT`, because the VMM calls [TDH.IMPORT.END](tdx-migration-seamcalls.md#tdhimportend)
  directly from `POST_IMPORT`.
- In **post-copy migration**, the destination TD starts before the full state is transferred from
  the source. The destination TD goes through the in-order phase and enters `POST_IMPORT`. Then the
  VMM calls [TDH.IMPORT.COMMIT](tdx-migration-seamcalls.md#tdhimportcommit), which moves the
  destination to `LIVE_IMPORT` before all memory pages are imported. The missing pages are delivered
  on demand or in the background while the TD runs in `LIVE_IMPORT`. The VMM ends the
  [import session](#import-session) with `TDH.IMPORT.END`.

In other words, both migration types go through the out-of-order phase, but pre-copy migration skips
`LIVE_IMPORT`.

**Note**: Post-copy migration is not available with TDX Connect. In the current TDX module
implementation, it is also not available with non-blocking export. See
[Post-Copy Support](#post-copy-support).

---

## Post-Copy Support

In [post-copy migration](#pre-copy-vs-post-copy-migration), the destination TD runs in the
`LIVE_IMPORT` [operation state](tdx-misc.md#td-operation-states) while some private pages are still
missing. When the TD accesses a missing page, the access faults and the VMM delivers the page on
demand by exporting it from the paused source TD.

Here is the post-copy compatibility table depending on TDX Connect and the
[export mode](#non-blocking-vs-write-blocking-export).

| Export Mode    | TDX Connect TD | Post-Copy                              | Architectural Limit |
|----------------|----------------|----------------------------------------|---------------------|
| Write-blocking | No             | Supported                              | N/A                 |
| Write-blocking | Yes            | Not possible                           | Yes                 |
| Non-blocking   | No             | Not implemented in current TDX modules | No                  |
| Non-blocking   | Yes            | Not possible                           | Yes                 |

- **Post-copy with write-blocking and no TDX Connect**: Supported. A CPU access to a missing page of
  the destination TD faults, and the VMM delivers the page on demand.
- **Post-copy with write-blocking and TDX Connect**: Not possible, architectural limit. The TDX
  specs allow only non-blocking export for TDX Connect TDs, because DMA from a bound TDI to the TD
  memory must not fail. Write-blocking export removes the Write permission, so a DMA write by the
  TDI to a write-blocked page would fail.
- **Post-copy with non-blocking and no TDX Connect**: Not implemented. The current TDX module
  generates the start token only after the VMM paused the source TD and exported all remaining TD
  private memory, so no pages remain for on demand delivery. The TDX specs define post-copy
  migration with non-blocking export as an optional TDX module capability.
- **Post-copy with non-blocking and TDX Connect**: Not possible, architectural limit. A TDI cannot
  fault on a missing page, so a DMA to it fails. There is no mechanism to deliver pages on demand to
  TDIs. So a TDX Connect TD must export every private page during the
  [in-order phase](#migration-phases). The VMM must detach the TDIs before it pauses the TD.

---

## Migration Epochs

The VMM controls the order in which the destination imports
[memory bundles](tdx-migration-structs.md#memory-bundle), so it can attempt a rollback attack. For
example, it can save a bundle with an earlier version of a page and import it later, after the
destination has a newer version. Or it can remove the page directly with `TDH.MEM.PAGE.REMOVE` and
import the saved bundle. The saved bundle still has a valid MAC, so the MAC check alone cannot
detect this.

During the [in-order phase](#migration-phases) of the migration, the TDX module orders bundles
within a [migration stream](tdx-migration-structs.md#migration-stream). Every
[MBMD](tdx-migration-structs.md#mbmd) carries a bundle counter, and the destination imports the
bundles of a stream only in the order of the counter. There is no such ordering across streams, so
the in-order phase is divided into migration epochs.

The source TDX module writes the current migration epoch into the MBMD of every memory bundle.
[TDH.EXPORT.TRACK](tdx-migration-seamcalls.md#tdhexporttrack) ends the epoch and produces an
[epoch token](tdx-migration-structs.md#mbmd-only-bundles) with the total number of bundles exported
so far. The destination imports the token with
[TDH.IMPORT.TRACK](tdx-migration-seamcalls.md#tdhimporttrack), which fails the
[import session](#import-session) if the destination has not imported all of these bundles. After
that, the destination accepts only memory bundles that carry the new epoch. So it imports the
bundles epoch by epoch, regardless of the stream.

However, memory bundles of different streams within one migration epoch are still not ordered. The
source can export two versions of the same page in one migration epoch, on different streams. Then
the VMM can import the newer version first and the older version after it. Or it can import the
newer version, remove the page with `TDH.MEM.PAGE.REMOVE`, and import the older version.

**Note**: The last invocation of `TDH.EXPORT.TRACK` ends the in-order phase and produces the start
token, which is an epoch token of the same type.

### Per Page Epoch

To prevent the attacks above, the destination TDX module records the migration epoch of the last
import or cancellation of each page. A page accepts an import or cancellation only if its recorded
epoch is lower than the current migration epoch. Otherwise the TDX module rejects the operation for
that page. This way a page accepts at most one import or cancellation per epoch.

Normally the per page migration epoch value is stored in PAMT. However, a `REMOVED` page has no
associated physical page and no PAMT entry, so the migration epoch is kept in the SEPT entry itself.
This way a local removal of an imported page with `TDH.MEM.PAGE.REMOVE` does not erase the epoch.

**Note**: In the PAMT entry, the migration epoch lives in the `BEPOCH` field, which also holds a
sampled TD epoch for TLB tracking and scanning, see [BEPOCH](tdx-misc.md#bepoch). The `MIG_FLAG` bit
of the field tells which of the two uses applies. It is set only for pages that an import or a
cancellation touched on the destination TD.

---

## Migration Setup Service

The migration setup protocol is implemented either by MSS (Migration Setup Service), which is a TDX
module extension, or by a MigTD, which is a service TD. This document covers only MSS.

As an [extension](tdx-misc.md#tdx-module-extensions), MSS runs as a hidden TD. The VMM drives it
with the [TDH.MIG.SETUP](tdx-migration-seamcalls.md#tdhmigsetup) seamcall. The VMM must either
resume an interrupted `TDH.MIG.SETUP` or abort the setup with
[TDH.MIG.SETUP.ABORT](tdx-migration-seamcalls.md#tdhmigsetupabort), as described in
[Must the VMM Resume?](tdx-misc.md#must-the-vmm-resume).

**Note**: MSS also parses the migration policy during the [setup session](#setup-session). See
[Migration Policy](#migration-policy) for details.

### Sessions and Threads

MSS capacity is defined in terms of two resources:

- A **setup session** holds the state of the `TDH.MIG.SETUP` protocol for one TD. It starts on the
  first `TDH.MIG.SETUP` seamcall invocation for that TD and lasts until the protocol completes,
  fails, or is aborted. A TD has at most one setup session. When no free sessions are available,
  `TDH.MIG.SETUP` returns `TDX_OPERAND_BUSY(MIG_SETUP_SESSIONS)`.
- A **thread** is a virtual thread of the hidden TD that implements MSS (see
  [Virtual Threads](tdx-misc.md#virtual-threads)). A session takes a thread for the duration of one
  `TDH.MIG.SETUP` seamcall invocation and releases it on return. An interrupted invocation
  (`TDX_INTERRUPTED_RESUMABLE`) keeps the thread reserved until it completes. When no free threads
  are available, `TDH.MIG.SETUP` returns `TDX_OPERAND_BUSY(VIRTUAL_THREAD_POOL)`.

**Note**: See [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy) for more information about this
status code.

Sessions limit how many different TDs can run the setup protocol at the same time. Threads limit how
many `TDH.MIG.SETUP` seamcalls can run at the same time. Both counts are platform-wide resources of
MSS.

---

## Migration Policy

A **migration policy** is a JSON document that defines where a TD is allowed to migrate. The TDX
specifications define its schema. Example rules:

- Allow migration only to GNR platforms.
- Allow migration only to TDX module version 2.0 or newer.

There are two policy types:

- **TD migration policy**: Controlled by the TD owner, for example a tenant of a cloud service
  provider (CSP).
- **Platform migration policy**: Controlled by the platform owner, for example the CSP.

The TD migration policy reflects what the TD owner wants. For example, the TD owner may want to
block migration to an older TDX module or CPU microcode version. In practice, CSPs are unlikely to
allow arbitrary tenant policies. They are more likely to offer a small set of allowed options.

The VMM supplies the policies to [TDH.MIG.SETUP](tdx-migration-seamcalls.md#tdhmigsetup) during the
[setup session](#setup-session):

- On the source, the first `TDH.MIG.SETUP` invocation must include the TD migration policy JSON. It
  may also include the platform migration policy JSON.
- On the destination, only the platform migration policy JSON may be provided.

The platform migration policy is optional. It adds restrictions on top of the TD migration policy
and, unlike the TD migration policy, is not included in TD attestation quotes.

On the source, `TDH.MIG.SETUP` verifies that the TD migration policy JSON matches the policy hash
loaded into the TD during [TD creation](#td-creation). If the hash does not match, the seamcall
fails with `TDX_MIG_SETUP_TD_POLICY_HASH_MISMATCH`. If no hash was loaded, it fails with
`TDX_MIG_SETUP_TD_POLICY_HASH_ZERO`.

The policy hash is also reflected in TD attestation data, so the TD owner can verify from the
attestation quote that the expected policy was loaded.

**Note**: [MSS](#migration-setup-service) supports only the JSON policy schema defined by the TDX
specifications. Users who need more complex rules can use a MigTD instead, which allows arbitrary
custom rules.

---

## Enumeration

TDX live migration features are enumerated through the [TDX_FEATURES0](tdx-structs.md#tdx_features0)
register, read with `TDH.SYS.RD` (by the VMM) or `TDG.SYS.RD` (by the TD).

### Individual Features

[TDX_FEATURES0](tdx-structs.md#tdx_features0) enumerates the following individual features:

- `TD_MIGRATION` (bit 0) indicates that the TDX module supports TD migration.
- `FMS_CONFIG` (bit 17) indicates support for configuring the virtual `CPUID(0x1).EAX` value, which
  reports the TD Family, Model and Stepping. This is needed because a TD may migrate between hosts
  with different native Family/Model/Stepping values, and the guest-visible value must stay the same
  across the migration. The TDX module only allows configuring this value for migratable TDs.
- `PARTITIONED_TD_MIGRATION` (bit 21) indicates support for migrating partitioned TDs, i.e., TDs
  configured with one or more L2 VMs.
- `NON_BLOCKING_EXPORT` (bit 41) indicates support for the
  [non-blocking export](#non-blocking-vs-write-blocking-export) mode.
- `SCAN_EXPORT_RESTORE` (bit 43) indicates support for the `EXPORT_RESTORE` operation of the
  [TDH.MEM.SCAN.RANGE](tdx-migration-seamcalls.md#tdhmemscanrange) seamcall. It can be used in the
  abort flow as a simpler and more efficient alternative to
  [TDH.EXPORT.RESTORE](tdx-migration-seamcalls.md#tdhexportrestore).
- `IMPORT_PAGE_STATUS` (bit 44) indicates that
  [TDH.IMPORT.MEM](tdx-migration-seamcalls.md#tdhimportmem) reports more detail in the page list
  entries on output. It sets the `INVALID` bit (63) for provided pages it did not use, and for a
  [CANCEL operation](tdx-migration-structs.md#gpa-list-entry-operation-values) that removed a page,
  it returns the HPA of the removed page with the `REMOVED` bit (61) set.
- `DEBUG_RO_TD_MIGRATION` (bit 52) indicates support for combining `ATTRIBUTES.DEBUG` and
  `ATTRIBUTES.MIGRATABLE` (see [ATTRIBUTES](tdx-structs.md#attributes)). Without this bit, the two
  are mutually exclusive. With this bit set, a TD can be both debuggable and migratable, but
  debugging becomes read-only: the host VMM can read TD memory but not write it, and only the
  metadata fields already writable for a non-debuggable TD remain writable.
- `LIST_ERROR_COUNT` (bit 54) indicates that `TDH.EXPORT.BLOCKW`,
  [TDH.EXPORT.MEM](tdx-migration-seamcalls.md#tdhexportmem), `TDH.EXPORT.RESTORE` and
  `TDH.IMPORT.MEM` support version 1, which returns a [GPA list](tdx-migration-structs.md#gpa-list)
  error count on output. Otherwise, only version 0 is supported.
- `MIG_SETUP` (bit 55) indicates support for [MSS](#migration-setup-service), the Migration Setup
  service extension of the TDX module.

### Enhanced Import

**Enhanced import** is a set of TDX module capabilities for TD state import seamcalls. These
capabilities are available when `TDX_FEATURES0.ENHANCED_IMPORT` (bit 60) is set.

When a TDX module supports enhanced import, the following features become available during import:

- **Pre-allocation**: Pre-populate the destination TD with physical pages before their content is
  imported.
- **Page removal**: Remove a destination TD page during the [in-order phase](#migration-phases).
- **Page relocation**: Move a destination TD page to a different physical page.
- **Page demotion**: Split a large destination TD page with `TDH.MEM.PAGE.DEMOTE` during the
  in-order import phase.
- **NO_REOWN**: Forbid a physical page from changing ownership between the VMM and the TD during
  import.
- **Partial GPA list processing**: Let `TDH.IMPORT.MEM` process only part of the GPA list of a
  migration bundle per invocation.

#### Pre-Allocation

The `TDH.MEM.PAGE.AUG` seamcall can pre-populate the destination TD with physical pages before their
content is imported. The pages start in the [PENDING](tdx-misc.md#sept-page-states) state.
[TDH.IMPORT.MEM](tdx-migration-seamcalls.md#tdhimportmem) then imports the migrated pages into these
pre-allocated pages and updates the SEPT entries to the states recorded in the GPA list entries of
the migration bundle.

**Note**: `TDH.MEM.PAGE.ADD` is prohibited because the destination TD never goes through the normal
build flow. [TDH.IMPORT.STATE.IMMUTABLE](tdx-migration-seamcalls.md#tdhimportstateimmutable) starts
the [import session](#import-session) directly from the `UNINITIALIZED` `OP_STATE` of the TD,
bypassing the state in which `TDH.MEM.PAGE.ADD` is allowed.

#### Page Removal

Without enhanced import, the VMM cannot remove a destination TD page with `TDH.MEM.PAGE.REMOVE`
during the [in-order phase](#migration-phases). The seamcall fails with `TDX_OP_STATE_INCORRECT`.

With enhanced import, page removal is allowed in this phase. It does not need to follow the normal
[block and TLB tracking procedure](tdx-misc.md#example-reclaiming-a-private-page). The VMM can
remove the page directly, because the destination TD does not run.

**Note 1**: Removal in the out-of-order phase does not depend on enhanced import. In `POST_IMPORT`,
the TD does not run, so the VMM also removes the page directly. In `LIVE_IMPORT`, the TD may be
running, so the VMM must block the page and complete TLB tracking before calling
`TDH.MEM.PAGE.REMOVE`, as for any running TD.

**Note 2**: During an [import session](#import-session), in both phases, `TDH.MEM.PAGE.REMOVE`
changes the SEPT entry state to [REMOVED](tdx-misc.md#sept-page-states) instead of `FREE`. This is
part of the migration epoch tracking mechanism, as described in
[Migration Epochs](#migration-epochs).

#### Page Relocation

The `TDH.MEM.PAGE.RELOCATE` seamcall can move a destination TD page to a different physical page
during the [in-order phase](#migration-phases). The [GPA](x86-misc.md#guest-addresses) mapping does
not change: the seamcall copies the page content to the new HPA (except for `PENDING` pages),
updates the SEPT entry to point there, and returns the old HPA to the VMM.

#### Page Demotion

The `TDH.MEM.PAGE.DEMOTE` seamcall can split a large destination TD page into smaller pages during
the [in-order phase](#migration-phases). It is needed after pre-allocating a 2MB page with
`TDH.MEM.PAGE.AUG`, because the VMM must demote the page before any page is imported over it.

**Note**: `TDH.MEM.PAGE.PROMOTE` remains prohibited during the in-order phase.

#### NO_REOWN

`NO_REOWN` is an input flag of [TDH.IMPORT.MEM](tdx-migration-seamcalls.md#tdhimportmem) that
forbids a physical page from changing ownership between the VMM and the TD during the seamcall.

A physical page is either:

- TD-owned: Private, encrypted with the HKID of the TD, mapped by a SEPT entry.
- VMM-owned: Regular host memory.

Incoming [GPA list](tdx-migration-structs.md#gpa-list) entries carry an
[OPERATION](tdx-migration-structs.md#gpa-list-entry-operation-values) field. Without `NO_REOWN`,
operations can change page ownership:

- `MIGRATE` converts a VMM-owned page to a TD-owned page. The page comes from the destination page
  list, or the migration buffer itself is converted (in-place import). If the page was already
  pre-allocated, `MIGRATE` imports into it and ownership does not change.
- `CANCEL` removes the TD-owned page and converts it back to a VMM-owned page. If
  `TDX_FEATURES0.IMPORT_PAGE_STATUS` is set, the page list entry returns its HPA to the VMM.
- `REMIGRATE` can take a new VMM-owned page to replace one that was previously removed with
  `TDH.MEM.PAGE.REMOVE`, converting it to a TD-owned page again.

With `NO_REOWN` set, `TDH.IMPORT.MEM` no longer changes ownership. Instead, it changes what each
operation accepts and requires:

- `MIGRATE` succeeds only if the page was already pre-allocated (TD-owned), and the destination page
  list is not used.
- `CANCEL` does not hand the page back to the VMM. Instead, it is converted to a pre-allocated
  `PENDING` page, so it stays TD-owned.
- `REMIGRATE` succeeds only if the page is already TD-owned, either because it was imported earlier
  and not locally removed, or because it was removed and then re-pre-allocated with
  `TDH.MEM.PAGE.AUG`.

#### Partial GPA List Processing

With partial GPA list processing, the VMM controls how many GPA list entries of a migration bundle
[TDH.IMPORT.MEM](tdx-migration-seamcalls.md#tdhimportmem) processes in each run, rather than the
whole bundle at once.

The VMM sets `GPA_LIST_INFO.LAST_ENTRY` to the last entry to process in the current run. If
`LAST_ENTRY` is before the end of the GPA list of the bundle, `TDH.IMPORT.MEM` returns
`TDX_INTERRUPTED_LIST_END` and the VMM resumes it later with a higher `LAST_ENTRY`.

**Note**: [TDH.EXPORT.MEM](tdx-migration-seamcalls.md#tdhexportmem) handles the range differently.
`GPA_LIST_INFO.FIRST_ENTRY` and `LAST_ENTRY` select the range of GPA list entries to export, and
invocations produce new migration bundles, each with its own MBMD.

---

## Configuration

TDX live migration requires configuration at three levels: the TDX module, the TD, and the migration
session. The subsections below describe them in the order the VMM performs them.

### TDX Module Configuration

#### Configuration Flow

The VMM configures the TDX module for migration in these steps:

1. Read [TDX_FEATURES0](tdx-structs.md#tdx_features0) with `TDH.SYS.RD` and verify that the required
   features are supported (see [Enumeration](#enumeration)).
2. Optionally, read the maximum supported values of the configurable parameters with `TDH.SYS.RD`.
3. Optionally, write the desired values with `TDH.SYS.WR`. The MSS parameters described below are
   examples.
4. Execute the `TDH.SYS.CONFIG` seamcall, passing the configuration and the
   [FEATURES_ENABLE0](tdx-structs.md#features_enable0) bitmap with the required features enabled.

The migration flow requires these features:

- `TD_MIGRATION`.
- `SCAN_EXPORT_RESTORE`, for the abort flow.

These features are optional:

- `NON_BLOCKING_EXPORT` selects the export mode, as explained below.
- `MIG_SETUP` enables MSS. A MigTD can perform the migration setup instead. This document focuses on
  MSS, so the steps below assume that it is enabled. See
  [Migration Setup Service](#migration-setup-service) for details.

#### MSS Configuration

MSS configuration consists of the following steps:

1. Optionally, read the maximum number of [sessions and virtual threads](#sessions-and-threads) that
   MSS supports with `TDH.SYS.RD`: `MIG_SETUP_MAX_SESSIONS` and `MIG_SETUP_MAX_THREADS`.
2. Optionally, write the number of sessions and virtual threads to use with `TDH.SYS.WR`:
   `MIG_SETUP_NUM_SESSIONS` and `MIG_SETUP_NUM_THREADS`. If the VMM skips this step, the TDX module
   uses the default values.
3. Set `MIG_SETUP` (bit 55) to 1 in the `FEATURES_ENABLE0` bitmap passed to `TDH.SYS.CONFIG`. This
   makes [TDH.MIG.SETUP](tdx-migration-seamcalls.md#tdhmigsetup) available and adds MSS to the
   extensions that the TDX module counts when it computes the size of the memory pool (see
   [Extension Setup](tdx-misc.md#extension-setup)).
4. After `TDH.SYS.CONFIG`, set up the extension memory pool and initialize the extensions.

**Note 1**: The configured numbers of sessions and virtual threads apply to the whole TDX module.

**Note 2**: The setup session needs a platform attestation quote, so the VMM must be ready to
deliver one, using either the [Quoting Service](tdx-misc.md#quoting-service) or the Quoting Enclave.
This assumes that the attestation flow is already configured.

#### Export Mode

The VMM selects the [non-blocking export](#non-blocking-vs-write-blocking-export) mode by setting
`NON_BLOCKING_EXPORT` (bit 41) to 1 in the `FEATURES_ENABLE0` bitmap passed to `TDH.SYS.CONFIG`.
Otherwise, the TDX module uses write-blocking export. The mode is selected during TDX module
initialization and cannot change until the TDX module is re-initialized or updated.

### TD Creation

The VMM creates the TD as migratable by setting [ATTRIBUTES.MIGRATABLE](tdx-structs.md#attributes)
to 1 when executing the `TDH.MNG.INIT` seamcall. Migration operations fail with
`TDX_TD_NOT_MIGRATABLE` if this bit is 0.

The TD also needs a migration policy hash. Before finalizing the TD with `TDH.MR.FINALIZE`, the VMM
loads a nonzero hash of the migration policy:

1. Compute SHA-384 of the [TD migration policy](#migration-policy) JSON.
2. Write the hash to `MIG_SETUP_TD_POLICY_HASH` using `TDH.MNG.WR`.

### Migration Streams

The VMM creates the [migration stream contexts](tdx-migration-structs.md#migration-stream) on the
source and the destination with
[TDH.MIG.STREAM.CREATE](tdx-migration-seamcalls.md#tdhmigstreamcreate). The destination must have at
least as many streams as the source.

Streams can be created within this time range:

- After the [TDCS](tdx-structs.md#tdcs) pages have been added with `TDH.MNG.ADDCX`.
- Before the [export session](#export-session) starts on the source and before the
  [import session](#import-session) starts on the destination. The sessions start with
  [TDH.EXPORT.STATE.IMMUTABLE](tdx-migration-seamcalls.md#tdhexportstateimmutable) and
  [TDH.IMPORT.STATE.IMMUTABLE](tdx-migration-seamcalls.md#tdhimportstateimmutable), respectively.

The number of created streams is limited by `MAX_MIGS`, the maximum number of streams per TD. It is
a global [metadata field](tdx-structs.md#metadata-fields) that the VMM can read with `TDH.SYS.RD`.
