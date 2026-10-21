# TDX Live Migration

- **Author**: Artem Bityutskiy
- **Version**: 0.1
- **Date**: 2026-07-03
- **Last updated**: 2026-10-21

**Disclaimer**: These notes reflect my current understanding of TDX live migration concepts and
data structures and may contain errors or omissions. They are not an official Intel document. Refer
to the official Intel documentation for authoritative information.

This is not a standalone article. It is a collection of notes and explanations about TDX live
migration data structures, intended for reference from my other documents.

**Terminology Notes**:

- A **seamcall** is a TDX module ABI leaf function invoked with the `SEAMCALL` instruction.
  For example, `TDH.MIG.SETUP` is a seamcall.
- A **tdcall** is a TDX module ABI leaf function invoked with the `TDCALL` instruction.
  Tdcalls include all `TDG.*` leaf functions.
- `SEAMCALL` and `TDCALL` refer to the CPU instructions, while `seamcall` and `tdcall` refer to
  the leaf functions they invoke.
- **HPA** means a host physical address of a page owned by the VMM. The TDX specs often call it
  a "shared HPA", but this document uses "HPA".

---

## Table of Contents

- [TDX Live Migration](#tdx-live-migration)
  - [Table of Contents](#table-of-contents)
  - [HPA Linked List](#hpa-linked-list)
  - [HPA List](#hpa-list)
    - [HPA List vs HPA Linked List](#hpa-list-vs-hpa-linked-list)
  - [GPA List](#gpa-list)
    - [GPA List Entry Operation Values](#gpa-list-entry-operation-values)
  - [GPA List-of-Lists](#gpa-list-of-lists)
    - [GPA List vs GPA List-of-Lists](#gpa-list-vs-gpa-list-of-lists)
  - [Range List](#range-list)
  - [Migration Bundle](#migration-bundle)
    - [MBMD](#mbmd)
    - [MBMD-Only Bundles](#mbmd-only-bundles)
    - [State Bundles](#state-bundles)
    - [Memory Bundle](#memory-bundle)
      - [MAC Pages](#mac-pages)

## HPA Linked List

An HPA linked list (TDX spec name: `HPA_LINKED_LIST`) is a TDX module API data structure used by
seamcalls such as `TDH.MIG.SETUP` to pass many physical pages in one argument.

An HPA linked list is a chain of metadata pages. Metadata pages contain 512 64-bit HPA linked
list entries (TDX spec name: `HPA_LIST_ENTRY`). Entries 0-510 point to data pages. Entry 511
points to the next metadata page, or to `NULL_PA` (-1) if this is the last metadata page.

Each entry is the HPA of a page, and it has the following general layout:

- Bits 11:0: reserved, must be 0.
- Bits 51:12: HPA bits 51:12 of the page.
- Bits 63:52: reserved, must be 0.

The only exception is `NULL_PA` (-1), which does not follow this layout.

Example of an HPA linked list with 1000 data pages.

```text
+---------------------+                      +----------------------+
| Entry 0             |---> Data page 0      | Entry 0              |---> Data page 511
| Entry 1             |---> Data page 1      | Entry 1              |---> Data page 512
| Entry 2             |---> Data page 2      | ...                  |
| Entry 3             |---> Data page 3      | Entry 488            |---> Data page 999
| ...                 |                      | Unused Entry         |
| Entry 509           |---> Data page 509    | ...                  |
| Entry 510           |---> Data page 510    | Unused Entry         |
| Entry 511           |--------------------->| Entry 511 = NULL_PA  |
+---------------------+                      +----------------------+
```

In this example, the second metadata page uses entries 0-488 for data pages 511-999. Entries
489-510 on that page are unused.

A seamcall may specify the number of used entries with a separate size parameter. If a seamcall
has no such parameter, unused entries must be set to `NULL_PA` (-1).

## HPA List

An HPA list (TDX spec name: `HPA_LIST`) is a TDX module API data structure used by seamcalls
such as `TDH.EXPORT.STATE.IMMUTABLE` to pass a set of physical pages.

An HPA list is one metadata page with up to 512 64-bit HPA list entries (TDX spec name:
`HPA_LIST_ENTRY`). Each entry is the HPA of one data page. The entry layout is the same as for an
HPA linked list entry, described in the [HPA Linked List](#hpa-linked-list) section.

Seamcalls may use the reserved bits for their own purposes. For example, `TDH.EXPORT.MEM` uses bit
63 (`INVALID`) to report that no data was exported for the page.

**HPA_LIST_INFO**

An HPA list is described by a 64-bit `HPA_LIST_INFO` value with this layout:

- Bits 2:0 (`FORMAT`): reserved, must be 0.
- Bits 11:3 (`FIRST_ENTRY`): index of the first entry to process.
- Bits 51:12: HPA bits 51:12 of the HPA list metadata page.
- Bits 54:52: reserved, must be 0.
- Bits 63:55 (`LAST_ENTRY`): index of the last entry in the HPA list.

Example: HPA list with 32 data pages.

```text
       63:55      54:52  51:12        11:3       2:0
+---------------+------+-------+---------------+-----+
| LAST_ENTRY=31 | 000b | HPA   | FIRST_ENTRY=0 | 000 |
+---------------+------+-------+---------------+-----+

+--------------+
| Entry 0      |--> Data page 0
| Entry 1      |--> Data page 1
| ...          |
| Entry 31     |--> Data page 31
| Unused       |
| ...          |
| Unused       |
+--------------+
```

### HPA List vs HPA Linked List

The two data structures scale differently. An [HPA linked list](#hpa-linked-list) can describe very
large buffers by chaining metadata pages. An [HPA list](#hpa-list) is limited to one metadata page,
so it holds at most 512 data pages.

`TDH.MIG.SETUP` was added to the TDX module ABI later than `TDH.EXPORT.STATE.IMMUTABLE`. The TDX
specs do not state why the two seamcalls use different structures, but the reason is most probably
historical.

## GPA List

A GPA list (TDX spec name: `GPA_LIST`) is a TDX migration data structure used by several seamcalls
to carry per-page guest metadata for a page set. The seamcalls are:

- `TDH.MEM.SCAN*`
- `TDH.EXPORT.BLOCKW`
- `TDH.EXPORT.MEM`
- `TDH.EXPORT.RESTORE`
- `TDH.IMPORT.MEM`

At a high level, it is a single 4KiB page that holds up to 512 64-bit GPA list entries (TDX spec
name: `GPA_LIST_ENTRY`). Each entry describes one guest page. Unlike an HPA list entry, a GPA list
entry does not point at an HPA. Instead, it carries fields such as the guest GPA, operation
requested for that page, and the status returned after processing.

The entry format is shared by all of these seamcalls, so that the output of one seamcall can be the
input of the next. For example, the output of `TDH.EXPORT.BLOCKW` or `TDH.MEM.SCAN*` can be used as
input of `TDH.EXPORT.MEM`, and the output of `TDH.EXPORT.MEM` can be used as input of
`TDH.IMPORT.MEM`. Each seamcall reads some fields, ignores others, and updates others.

Here are some of the most relevant fields in a GPA list entry. Not all seamcalls use all of these
fields.

- Bits 1:0 (`LEVEL`): page level of the GPA: 0=4KiB, 1=2MiB, 2=1GiB. Only `TDH.MEM.SCAN*` can
  return levels 1 and 2. The other seamcalls support only 4KiB pages.
- Bit 2 (`PENDING`): the page is in a `PENDING*` state, which means the TD has not accepted it yet.
  See [Leaf Entry States](tdx-misc.md#leaf-entry-states) for the state definitions and
  [Shared to Private](tdx-misc.md#shared-to-private) for an example of how a page becomes
  `PENDING`.
- Bits 4:3 (`STATE`): hint about the page state. Only `TDH.MEM.SCAN*` sets it, to explain why the
  page is an export candidate. The other seamcalls ignore it or require it to be 0.
- Bits 9:7 (`L2_MAP`): bitmap that indicates whether the page is mapped in L2 VMs of a partitioned
  TD. `TDH.EXPORT.MEM` sets it on output, and `TDH.IMPORT.MEM` reads it on input, so that the VMM
  can prepare L2 SEPT pages before the import. The other seamcalls ignore it.
- Bits 51:12 (`GPA`): guest physical address bits 51:12.
- Bits 53:52 (`OPERATION`): action associated with this entry (see below).
- Bits 60:56 (`STATUS`): result or status after processing.

**GPA_LIST_INFO**

The list itself is described by a 64-bit `GPA_LIST_INFO` value:

- Bits 2:0 (`FORMAT`): list format: 0=`GPA_ONLY`, 1=`GPA_AND_L2_ATTR` (used for partitioned TDs).
- Bits 11:3 (`FIRST_ENTRY`): index of the first entry to process.
- Bits 51:12: HPA bits 51:12 of the GPA list metadata page.
- Bits 54:52: reserved, must be 0.
- Bits 63:55 (`LAST_ENTRY`): index of the last entry in the GPA list.

Example: GPA list with 32 entries.

```text
       63:55      54:52  51:12        11:3       2:0
+---------------+------+-------+---------------+-----+
| LAST_ENTRY=31 | 000b | HPA   | FIRST_ENTRY=0 | 000 |
+---------------+------+-------+---------------+-----+

+------------------+
| GPA Entry 0      |
| GPA Entry 1      |
| ...              |
| GPA Entry 31     |
| Unused           |
| ...              |
| Unused           |
+------------------+
```

### GPA List Entry Operation Values

The `OPERATION` field has four possible values. Their meaning depends on the seamcall and on
whether the entry is a seamcall input or output. On input, the value is a request. On output, the
seamcall writes the operation that it actually performed, which can differ from the request.

For `TDH.EXPORT.MEM` with
[non-blocking export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export), the input
values are:

- `NOP`: skip the entry.
- `EXPORT`: request to export the page. Based on the SEPT entry state, the TDX module decides the
  actual operation: `MIGRATE`, `REMIGRATE`, `CANCEL`, or `NOP`.

With [write-blocking based export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export), the
input values differ:

- `MIGRATE`: request to export the page, instead of `EXPORT`. The TDX module can turn it into
  `MIGRATE`, `REMIGRATE`, or `NOP`.
- `CANCEL`: request to cancel a previous export. The TDX module can turn it into `CANCEL` or `NOP`.

The output values are the same for both modes:

- `NOP`: the page was not exported.
- `MIGRATE`: initial export of the page during this migration session, or following a `CANCEL`.
- `CANCEL`: cancellation of a previous export.
- `REMIGRATE`: re-export of a page with updated content or attributes.

The VMM reads the output `OPERATION` together with `STATUS`:

- An operation other than `NOP` goes with `SUCCESS`.
- `NOP` goes with `SKIPPED` (`NOP` was requested) or a status code that tells why the page was
  skipped. For example, if TLB tracking is not done for the page, the status code is
  `TLB_TRACKING_NOT_DONE`.

The VMM sends the GPA list to the destination together with the exported pages. `TDH.IMPORT.MEM`
takes the values `NOP`, `MIGRATE`, `CANCEL`, and `REMIGRATE` as input, with the matching meaning for
import.

**Note**: Other seamcalls use their own values. For example, `TDH.EXPORT.BLOCKW` uses `NOP` and
`BLOCKW`. This section covers only `TDH.EXPORT.MEM`.

## GPA List-of-Lists

A GPA list-of-lists is the multi-page version of a [GPA list](#gpa-list). It is used by
`TDH.MEM.SCAN.COMP` and `TDH.MEM.SCAN.RANGE`, which may need to report more than 512 entries. A
list-of-lists holds up to 512 * 512 = 262144 GPA list entries.

The top level is a single 4KiB page that holds up to 512 `GPA_LIST_INFO` values. Each value
describes one GPA list page, which holds up to 512 GPA list entries, as described in the
[GPA List](#gpa-list) section. In these `GPA_LIST_INFO` values, only the HPA field is used.
`FORMAT`, `FIRST_ENTRY`, and `LAST_ENTRY` must be 0.

**LIST_OF_LISTS_INFO**

The top-level page is described by a 64-bit `LIST_OF_LISTS_INFO` value. It has the same layout as
`GPA_LIST_INFO`, with these differences:

- Bits 2:0 (`FORMAT`): must be 2 (`LIST_OF_LISTS`).
- Bits 11:3 (`FIRST_ENTRY`): must be 0.
- Bits 51:12: HPA bits 51:12 of the top-level page. The page holds `GPA_LIST_INFO` values, not GPA
  list entries.
- Bits 54:52: reserved, must be 0.
- Bits 63:55 (`LAST_ENTRY`): index of the last valid `GPA_LIST_INFO` value in the top-level page.

The seamcalls do not modify `LIST_OF_LISTS_INFO` or the `GPA_LIST_INFO` values. They write only the
GPA list entries.

Example: list-of-lists with 3 GPA list pages.

```text
       63:55      54:52  51:12        11:3       2:0
+---------------+------+-------+---------------+-----+
| LAST_ENTRY=2  | 000b | HPA   | FIRST_ENTRY=0 | 010 |
+---------------+------+-------+---------------+-----+

+------------------+
| GPA_LIST_INFO 0  |---> GPA list page 0 (entries 0-511)
| GPA_LIST_INFO 1  |---> GPA list page 1 (entries 0-511)
| GPA_LIST_INFO 2  |---> GPA list page 2 (entries 0-511)
| Unused           |
| ...              |
| Unused           |
+------------------+
```

### GPA List vs GPA List-of-Lists

Both structures hold GPA list entries in the same format. They differ in capacity:

- A GPA list is one page with up to 512 GPA list entries.
- A GPA list-of-lists is one top-level page with up to 512 `GPA_LIST_INFO` values. Each value
  describes one GPA list page, so it holds up to 512 * 512 = 262144 GPA list entries.

If every entry describes a 4KiB page:

- A GPA list covers up to 2MiB of guest memory.
- A GPA list-of-lists covers up to 1GiB of guest memory.

## Range List

A range list (TDX spec name: "range list page") is a TDX module API data structure used only by
`TDH.MEM.SCAN.CONFIG` to configure the GPA ranges for the comprehensive scan seamcall
(`TDH.MEM.SCAN.COMP`). It is a single 4KiB page that holds an array of 64-bit range entries (TDX
spec name: "range list entry"). Unlike a GPA list entry, a range entry does not describe a GPA.
It specifies the start of a GPA range and the sub-range size for parallel scanning.

Range entries have the following layout.

- Bits 20:0: reserved, must be 0.
- Bits 50:21 (`RANGE_START`): bits 50:21 of the GPA of the start of the range.
- Bit 51: reserved, must be 0.
- Bits 57:52 (`SUB_RANGE_SIZE_EXP`): sub-range size as a base-2 exponent. The allowed values are
  from 21 (2^21 bytes = 2MiB) to 51 (2^51 bytes = 2PiB).
- Bits 63:58: reserved, must be 0.

Entry 0 must have `RANGE_START`=0, and the entries must be sorted in ascending GPA order. Each
`RANGE_START` must be a valid private GPA below the maximum private GPA of the TD. It must also be
aligned to the sub-range size of its entry.

Each entry defines one range that starts at its `RANGE_START` and ends at the next entry, or at the
end of the private GPA space for the final entry. `SUB_RANGE_SIZE_EXP` divides the range into
sub-ranges of 2^`SUB_RANGE_SIZE_EXP` bytes. If the sub-range size is equal to or larger than the
range size, the scan of that range is single threaded.

**RANGE_LIST_INFO**

The range list is described by a 64-bit `RANGE_LIST_INFO` value with this layout:

- Bits 8:0 (`NUM_RANGES`): number of valid range entries. It must be from 1 through
  `MAX_MEM_SCAN_RANGES`, readable with `TDH.SYS.RD*`.
- Bits 11:9: reserved, must be 0.
- Bits 51:12: HPA bits 51:12 of the range list metadata page.
- Bits 63:52: reserved, must be 0.

Example: range list with three GPA ranges.

```text
 63:52          51:12                11:9        8:0
+--------------+--------------------+--------------+----------------+
| Reserved = 0 | Metadata page HPA  | Reserved = 0 | NUM_RANGES = 3 |
+--------------+--------------------+--------------+----------------+
                         |
                         v
+---------------------------------------------------------+
| Entry 0: RANGE_START = 0,     SUB_RANGE_SIZE_EXP = 21   |
| Entry 1: RANGE_START = GPA_1, SUB_RANGE_SIZE_EXP = 22   |
| Entry 2: RANGE_START = GPA_2, SUB_RANGE_SIZE_EXP = 21   |
+---------------------------------------------------------+

0                GPA_1              GPA_2           GPA space limit
|------------------|------------------|---------------------|
          Range 0          Range 1              Range 2
```

In this example, `GPA_1` must be 4MiB aligned, because its `SUB_RANGE_SIZE_EXP` is 22.

The VMM can use ranges and sub-ranges to run the comprehensive scan in parallel.

The TDX specs describe NUMA optimization as an intended usage scenario: the VMM configures ranges
that correspond to NUMA nodes and runs `TDH.MEM.SCAN.COMP` concurrently on CPUs close to the memory
of each range. For this to work, the VMM should also distribute SEPT pages according to the NUMA
node layout when donating them via the `TDH.MEM.SEPT.ADD` seamcall.

## Migration Bundle

A migration bundle is the unit of data produced by migration export seamcalls and consumed by
matching import seamcalls. For example, `TDH.EXPORT.STATE.IMMUTABLE` produces a bundle, and
`TDH.IMPORT.STATE.IMMUTABLE` consumes it.

Every bundle has one MBMD (Migration Bundle Metadata). It is a small header that describes the
bundle, for example the protocol version and the bundle type. The TDX Migration ABI defines six
bundle types:

| Bundle type        | Produced by                  | Consumed by                  |
| ------------------ | ---------------------------- | ---------------------------- |
| Immutable TD state | `TDH.EXPORT.STATE.IMMUTABLE` | `TDH.IMPORT.STATE.IMMUTABLE` |
| Mutable TD state   | `TDH.EXPORT.STATE.TD`        | `TDH.IMPORT.STATE.TD`        |
| vCPU state         | `TDH.EXPORT.STATE.VP`        | `TDH.IMPORT.STATE.VP`        |
| TD private memory  | `TDH.EXPORT.MEM`             | `TDH.IMPORT.MEM`             |
| Epoch token        | `TDH.EXPORT.TRACK`           | `TDH.IMPORT.TRACK`           |
| Abort token        | `TDH.IMPORT.ABORT`           | `TDH.EXPORT.ABORT`           |

This document groups the bundle types as follows:

- **State bundles**: the immutable TD state, mutable TD state, and vCPU state bundles.
- **Memory bundle**: the TD private memory bundle.
- **MBMD-only bundles**: the epoch token and abort token bundles.

The VMM allocates the buffers of a bundle, and the export seamcall fills them, encrypting the TD
state data. The VMM transports the bundle to the destination, where the import seamcall consume
it.

### MBMD

The **MBMD** (Migration Bundle Metadata) is the header of every migration bundle. It has generic
fields and fields specific to the bundle type. The maximum MBMD size is 128 bytes.

The MBMD is not encrypted, so the VMM can read it. Examples of MBMD fields:

- `SIZE`: total MBMD size in bytes.
- `MIG_VERSION`: migration protocol version.
- `MIGS_INDEX`: migration stream index.
- `MB_TYPE`: migration bundle type.
- `MAC`: MAC over the MBMD fields. In state bundles, it also covers the state pages. In the memory
  bundle, separate MACs cover the pages instead. The destination TDX module verifies this MAC.

### MBMD-Only Bundles

The epoch token and the abort token bundles contain only the MBMD. They carry control information
rather than TD state or memory contents.

The abort token travels from the destination to the source. `TDH.EXPORT.ABORT` consumes it only in
the [out-of-order phase](tdx-migration-misc.md#migration-phases).

```text
MBMD-only bundle

+----------------------+
|        MBMD          |
| Epoch or abort token |
+----------------------+
```

### State Bundles

State bundles is the collective name for the immutable TD state, mutable TD state, and vCPU state
bundles. They contain the MBMD and encrypted state data pages.

```text
State bundle

+----------+  +----------------------+  +----------+
|   MBMD   |  | Encrypted state page |  |   ...    |
+----------+  +----------------------+  +----------+
```

The state payload in these bundles is a serialized and encrypted collection of selected
[metadata fields](tdx-structs.md#metadata-fields).

**Note**: The state pages are in an [HPA list](#hpa-list). The HPA list metadata page is not part of
the state bundle, because its entries are HPAs on the source platform. The VMM builds a new
metadata page on the destination for the import seamcalls.

### Memory Bundle

The memory bundle has the most complex structure. In addition to the MBMD, it contains:

- GPAs of the exported pages, in the form of a [GPA list](#gpa-list) page.
- The encrypted memory pages.
- MACs of the exported pages, in one or two MAC pages.
- Attributes of the exported pages, in an attributes page (partitioned TDs only).

```text
Memory bundle

+------------+  +---------------+
|    MBMD    |  | GPA list page |
|            |  |  GPA entries  |
+------------+  +---------------+

+-------------+  +-----------------------+
| MAC page 0  |  | MAC page 1 (optional) |
| MAC entries |  | MAC entries           |
+-------------+  +-----------------------+

+----------------------+  +----------+
| Encrypted memory     |  |   ...    |
|     data page        |  |          |
+----------------------+  +----------+

+----------------------------+
| Attributes page (optional) |
|   Partitioned TDs only     |
+----------------------------+
```

Only the memory pages and the attributes page are encrypted.

The memory pages are in an [HPA list](#hpa-list). HPA list entry N belongs to GPA list entry N, so
the GPA list defines which entries are processed.

Not every GPA list entry has a memory page. `TDH.EXPORT.MEM` sets the `INVALID` flag (bit 63) in
the HPA list entry of a GPA when it produces no page for it, for example for a
[pending page](tdx-misc.md#leaf-entry-states), a
[cancellation](tdx-misc.md#live-migration-specific-states), or an export error specific to the page.
The pages corresponding to such entries are not part of the memory bundle and do not need to be
transferred to the destination host.

The HPA list metadata page is not part of the memory bundle, because its entries are HPAs on the
source platform. The VMM builds a new metadata page on the destination for the `TDH.IMPORT.MEM`
seamcall.

**Note**: The TDX spec calls the memory bundle HPA list the MMBL (Memory Migration Buffer List).
This document calls it the HPA list for simplicity.

#### MAC Pages

The MAC pages contain one MAC for each exported page. There are one or two 4KiB MAC pages, and each
page contains up to 256 128-bit entries. The MACs follow the order of the [GPA list](#gpa-list):
MAC entry N belongs to GPA list entry N, so the GPA list works as the index of the bundle.

Each MAC covers the page contents together with the GPA list entry of the page, and with the
attributes page entry for partitioned TDs. If an entry has no page contents, the MAC covers only the
list entries. The MACs are generated on export (`TDH.EXPORT.MEM`) and verified on import
(`TDH.IMPORT.MEM`).
