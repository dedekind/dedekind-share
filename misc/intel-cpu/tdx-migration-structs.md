# TDX Live Migration: Data Structures

- **Author**: Artem Bityutskiy
- **Version**: 0.1
- **Date**: 2026-07-03
- **Last updated**: 2026-11-03

**Disclaimer**: This document reflects my current understanding of TDX live migration concepts and
data structures and may contain errors or omissions. It is not an official Intel document. Refer to
the official Intel documentation for authoritative information.

This is not a standalone article. It is a supporting document that explains TDX live migration data
structures, intended for reference from my other documents.

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

- [TDX Live Migration: Data Structures](#tdx-live-migration-data-structures)
  - [Table of Contents](#table-of-contents)
  - [HPA Linked List](#hpa-linked-list)
  - [HPA List](#hpa-list)
    - [HPA\_LIST\_INFO](#hpa_list_info)
    - [HPA List vs HPA Linked List](#hpa-list-vs-hpa-linked-list)
  - [GPA List](#gpa-list)
    - [GPA\_LIST\_INFO](#gpa_list_info)
    - [GPA List Entry Operation Values](#gpa-list-entry-operation-values)
  - [GPA List-of-Lists](#gpa-list-of-lists)
    - [LIST\_OF\_LISTS\_INFO](#list_of_lists_info)
    - [OVERALL\_NEXT\_ENTRY](#overall_next_entry)
    - [GPA List vs GPA List-of-Lists](#gpa-list-vs-gpa-list-of-lists)
  - [Range List](#range-list)
    - [Range List Entry](#range-list-entry)
    - [RANGE\_LIST\_INFO](#range_list_info)
    - [Parallel Scanning](#parallel-scanning)
  - [Scan Contexts](#scan-contexts)
  - [Migration Bundle](#migration-bundle)
    - [MBMD](#mbmd)
    - [MBMD-Only Bundles](#mbmd-only-bundles)
    - [State Bundles](#state-bundles)
      - [Immutable State](#immutable-state)
      - [TD Mutable State](#td-mutable-state)
      - [vCPU Mutable State](#vcpu-mutable-state)
    - [Memory Bundle](#memory-bundle)
      - [MAC Pages](#mac-pages)
  - [Migration Stream](#migration-stream)
  - [Migration Counters](#migration-counters)

---

## HPA Linked List

An HPA linked list (TDX spec name: `HPA_LINKED_LIST`) is a TDX module API data structure used by
seamcalls such as [TDH.MIG.SETUP](tdx-migration-seamcalls.md#tdhmigsetup) to pass many physical
pages in one argument.

An HPA linked list is a chain of metadata pages. Metadata pages contain 512 64-bit HPA linked list
entries (TDX spec name: `HPA_LIST_ENTRY`). Entries 0-510 point to data pages. Entry 511 points to
the next metadata page, or to `NULL_PA` (-1) if this is the last metadata page.

Each entry is the HPA of a page, and it has the following layout:

- Bits 11:0: Reserved, must be 0.
- Bits 51:12: [PFN](x86-misc.md#page-numbers-and-page-frames) of the page.
- Bits 63:52: Reserved, must be 0.

The only exception is `NULL_PA` (-1), which does not follow this layout.

Example: HPA linked list with 1000 data pages.

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

In this example, the second metadata page uses entries 0-488 for data pages 511-999. Entries 489-510
on that page are unused.

A seamcall may specify the number of used entries with a separate size parameter. If a seamcall has
no such parameter, unused entries must be set to `NULL_PA` (-1).

---

## HPA List

An HPA list (TDX spec name: `HPA_LIST`) is a TDX module API data structure used by seamcalls such as
[TDH.EXPORT.STATE.IMMUTABLE](tdx-migration-seamcalls.md#tdhexportstateimmutable) to pass a set of
physical pages.

An HPA list is one 4KiB aligned metadata page with up to 512 64-bit HPA list entries (TDX spec name:
`HPA_LIST_ENTRY`). Each entry is the HPA of one data page. The entry layout is the same as for an
HPA linked list entry, described in the [HPA Linked List](#hpa-linked-list) section.

Seamcalls may use the reserved bits for their own purposes. For example,
[TDH.EXPORT.MEM](tdx-migration-seamcalls.md#tdhexportmem) uses bit 63 (`INVALID`) to report that no
data was exported for the page.

### HPA_LIST_INFO

An HPA list is described by a 64-bit `HPA_LIST_INFO` value with this layout:

- Bits 2:0 (`FORMAT`): Reserved, must be 0.
- Bits 11:3 (`FIRST_ENTRY`): Index of the first entry to process.
- Bits 51:12: [PFN](x86-misc.md#page-numbers-and-page-frames) of the HPA list metadata page.
- Bits 54:52: Reserved, must be 0.
- Bits 63:55 (`LAST_ENTRY`): Index of the last entry in the HPA list.

Example: HPA list with 32 data pages.

```text
       63:55      54:52  51:12        11:3       2:0
+---------------+------+-------+---------------+-----+
| LAST_ENTRY=31 | 000b | PFN   | FIRST_ENTRY=0 | 000 |
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

[TDH.MIG.SETUP](tdx-migration-seamcalls.md#tdhmigsetup) was added to the TDX module ABI later than
`TDH.EXPORT.STATE.IMMUTABLE`. The TDX specs do not state why the two seamcalls use different
structures.

---

## GPA List

A GPA list (TDX spec name: `GPA_LIST`) is a TDX module API data structure used by several seamcalls
to carry per-page guest metadata for a page set. The seamcalls are:

- `TDH.MEM.SCAN*`
- `TDH.EXPORT.BLOCKW`
- [TDH.EXPORT.MEM](tdx-migration-seamcalls.md#tdhexportmem)
- [TDH.EXPORT.RESTORE](tdx-migration-seamcalls.md#tdhexportrestore)
- [TDH.IMPORT.MEM](tdx-migration-seamcalls.md#tdhimportmem)

It is a single 4KiB page that holds up to 512 64-bit GPA list entries (TDX spec name:
`GPA_LIST_ENTRY`). Each entry describes one guest page. Unlike an HPA list entry, a GPA list entry
does not point at an HPA. Instead, it carries fields such as the [GPA](x86-misc.md#guest-addresses),
the operation requested for that page, and the status returned after processing.

The entry format is shared by all of these seamcalls, so that the output of one seamcall can be the
input of the next. For example, the output of `TDH.EXPORT.BLOCKW` or `TDH.MEM.SCAN*` can be used as
input of `TDH.EXPORT.MEM`, and the output of `TDH.EXPORT.MEM` can be used as input of
`TDH.IMPORT.MEM`. Seamcalls read some fields, ignore some, and update the rest.

Here are some of the most relevant fields in a GPA list entry. Not all seamcalls use all of these
fields.

- Bits 1:0 (`LEVEL`): Page level of the GPA: 0=4KiB, 1=2MiB, 2=1GiB. Only `TDH.MEM.SCAN*` can return
  levels 1 and 2. The other seamcalls support only 4KiB pages.
- Bit 2 (`PENDING`): The page is in a `PENDING*` state, which means the TD has not accepted it yet.
  See [Leaf Entry States](tdx-misc.md#leaf-entry-states) for the state definitions and
  [Shared to Private](tdx-misc.md#shared-to-private) for an example of how a page becomes `PENDING`.
- Bits 4:3 (`STATE`): Hint about the page state. Only `TDH.MEM.SCAN*` sets it, to explain why the
  page is an export candidate. The other seamcalls ignore it or require it to be 0.
- Bits 9:7 (`L2_MAP`): Bitmap that indicates whether the page is mapped in L2 VMs of a partitioned
  TD. `TDH.EXPORT.MEM` sets it on output, and `TDH.IMPORT.MEM` reads it on input, so that the VMM
  can prepare L2 SEPT pages before the import. The other seamcalls ignore it.
- Bits 51:12 (`GPA`): [GFN](x86-misc.md#page-numbers-and-page-frames) of the page.
- Bits 53:52 (`OPERATION`): Action associated with this entry (see below).
- Bits 60:56 (`STATUS`): Result or status after processing.

### GPA_LIST_INFO

The list itself is described by a 64-bit `GPA_LIST_INFO` value:

- Bits 2:0 (`FORMAT`): List format: 0=`GPA_ONLY`, 1=`GPA_AND_L2_ATTR` (used for partitioned TDs).
- Bits 11:3 (`FIRST_ENTRY`): Index of the first entry to process.
- Bits 51:12: [PFN](x86-misc.md#page-numbers-and-page-frames) of the GPA list metadata page.
- Bits 54:52: Reserved, must be 0.
- Bits 63:55 (`LAST_ENTRY`): Index of the last entry in the GPA list.

Example: GPA list with 32 entries.

```text
       63:55      54:52  51:12        11:3       2:0
+---------------+------+-------+---------------+-----+
| LAST_ENTRY=31 | 000b | PFN   | FIRST_ENTRY=0 | 000 |
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

The 2-bit `OPERATION` field has four encodings. Their meaning depends on the seamcall and on whether
the entry is a seamcall input or output. On input, the value is a request. On output, the seamcall
writes the operation that it actually performed, which can differ from the request.

For `TDH.EXPORT.MEM` with
[non-blocking export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export), the input values
are:

- `NOP` (encodings 0, 2, and 3): Skip the entry.
- `EXPORT` (encoding 1): Request to export the page. Based on the SEPT entry state, the TDX module
  decides the actual operation: `MIGRATE`, `REMIGRATE`, `CANCEL`, or `NOP`.

With write-blocking export, the input values differ:

- `MIGRATE` (encodings 1 and 3): Request to export the page, instead of `EXPORT`. The TDX module can
  turn it into `MIGRATE`, `REMIGRATE`, or `NOP`. The two encodings exist for compatibility with the
  output of `TDH.EXPORT.BLOCKW`.
- `CANCEL`: Request to cancel a previous export. The TDX module can turn it into `CANCEL` or `NOP`.

The output values are the same for both modes:

- `NOP`: The page was not exported.
- `MIGRATE`: Initial export of the page during this
  [export session](tdx-migration-misc.md#export-session), or following a `CANCEL`.
- `CANCEL`: Cancellation of a previous export.
- `REMIGRATE`: Re-export of a page with updated content or attributes.

The VMM reads the output `OPERATION` together with `STATUS`:

- An operation other than `NOP` comes with the `SUCCESS` status.
- `NOP` goes with `SKIPPED` (`NOP` was requested) or a status code that tells why the page was
  skipped. For example, if TLB tracking is not done for the page, the status code is
  `TLB_TRACKING_NOT_DONE`.

The VMM sends the GPA list to the destination together with the exported pages. `TDH.IMPORT.MEM`
takes the values `NOP`, `MIGRATE`, `CANCEL`, and `REMIGRATE` as input. They instruct the destination
to skip, import, cancel, or re-import the page.

**Note**: Other seamcalls use their own values. For example, `TDH.EXPORT.BLOCKW` uses `NOP` and
`BLOCKW`. This section covers only `TDH.EXPORT.MEM` and `TDH.IMPORT.MEM`.

---

## GPA List-of-Lists

A GPA list-of-lists is the multi-page version of a [GPA list](#gpa-list). It is used by
[TDH.MEM.SCAN.COMP](tdx-migration-seamcalls.md#tdhmemscancomp) and
[TDH.MEM.SCAN.RANGE](tdx-migration-seamcalls.md#tdhmemscanrange), which may need to report more than
512 entries. A list-of-lists holds up to 512 * 512 = 262144 GPA list entries.

The top level is a single 4KiB page that holds up to 512 `GPA_LIST_INFO` values. Each value
describes one GPA list page, which holds up to 512 GPA list entries. In these `GPA_LIST_INFO`
values, only the HPA field is used. `FORMAT`, `FIRST_ENTRY`, and `LAST_ENTRY` must be 0.

### LIST_OF_LISTS_INFO

The top-level page is described by a 64-bit `LIST_OF_LISTS_INFO` value. It has the same layout as
`GPA_LIST_INFO`, with these differences:

- Bits 2:0 (`FORMAT`): Must be 2 (`LIST_OF_LISTS`).
- Bits 11:3 (`FIRST_ENTRY`): Must be 0.
- Bits 51:12: [PFN](x86-misc.md#page-numbers-and-page-frames) of the top-level page. The page holds
  `GPA_LIST_INFO` values, not GPA list entries.
- Bits 54:52: Reserved, must be 0.
- Bits 63:55 (`LAST_ENTRY`): Index of the last valid `GPA_LIST_INFO` value in the top-level page.

The seamcalls do not modify `LIST_OF_LISTS_INFO` or the `GPA_LIST_INFO` values. They write only the
GPA list entries.

Example: list-of-lists with 3 GPA list pages.

```text
       63:55      54:52  51:12        11:3       2:0
+---------------+------+-------+---------------+-----+
| LAST_ENTRY=2  | 000b | PFN   | FIRST_ENTRY=0 | 010 |
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

### OVERALL_NEXT_ENTRY

`OVERALL_NEXT_ENTRY` is the index of the next GPA list entry to write. The `TDH.MEM.SCAN.COMP` and
`TDH.MEM.SCAN.RANGE` seamcalls take it as input and return the updated value.

The index counts entries across all GPA list pages, which is why it is called "overall". The page
and the entry within the page follow from the index:

- GPA list page: `OVERALL_NEXT_ENTRY` / 512. This is also the index of the `GPA_LIST_INFO` value of
  the page in the top-level page.
- Entry in that page: `OVERALL_NEXT_ENTRY` % 512.

Example: `OVERALL_NEXT_ENTRY` is 1030, so the next entry to write is entry 6 of GPA list page 2.

```text
 Top-level page          GPA list pages

+-----------------+     +-----------------------------------+
| GPA_LIST_INFO 0 |---> | Page 0: indexes 0-511 (full)      |
| GPA_LIST_INFO 1 |---> | Page 1: indexes 512-1023 (full)   |
| GPA_LIST_INFO 2 |-+   +-----------------------------------+
| Unused          | |
| ...             | |   +-----------------------------------+
+-----------------+ +-> | Page 2: indexes 1024-1535         |
                        |   Entry 0 = index 1024            |
                        |   ...                             |
                        |   Entry 6 = index 1030  <-- next  |
                        |   ...                             |
                        +-----------------------------------+
```

The valid input range is 0 to 512 * (`LAST_ENTRY` + 1) - 1, where `LAST_ENTRY` is the field of
`LIST_OF_LISTS_INFO`. When the list-of-lists is full, the seamcall returns a value above this range.

### GPA List vs GPA List-of-Lists

Both structures hold GPA list entries in the same format. They differ in capacity:

- A GPA list is one page with up to 512 GPA list entries.
- A GPA list-of-lists is one top-level page with up to 512 `GPA_LIST_INFO` values. Each value
  describes one GPA list page, so it holds up to 512 * 512 = 262144 GPA list entries.

If every entry describes a 4KiB page:

- A GPA list covers up to 2MiB of guest memory.
- A GPA list-of-lists covers up to 1GiB of guest memory.

---

## Range List

A range list (TDX spec name: "range list page") is a TDX module API data structure used only by
[TDH.MEM.SCAN.CONFIG](tdx-migration-seamcalls.md#tdhmemscanconfig) to configure the GPA ranges for
the comprehensive scan seamcall ([TDH.MEM.SCAN.COMP](tdx-migration-seamcalls.md#tdhmemscancomp)). It
is a single 4KiB page that holds an array of 64-bit range entries (TDX spec name: "range list
entry"). Unlike a GPA list entry, a range entry does not describe a GPA. It specifies the start of a
GPA range and the sub-range size for parallel scanning.

### Range List Entry

Range entries have the following layout:

- Bits 20:0: Reserved, must be 0.
- Bits 50:21 (`RANGE_START`): Bits 50:21 of the GPA of the start of the range.
- Bit 51: Reserved, must be 0.
- Bits 57:52 (`SUB_RANGE_SIZE_EXP`): Sub-range size as a base-2 exponent. The allowed values are
  from 21 (2^21 bytes = 2MiB) to 51 (2^51 bytes = 2PiB).
- Bits 63:58: Reserved, must be 0.

Entry 0 must have `RANGE_START`=0, and the entries must be sorted in ascending GPA order. Each
`RANGE_START` must be a valid private GPA below the maximum private GPA of the TD. It must also be
aligned to the sub-range size of its entry.

Each entry defines one range that starts at its `RANGE_START` and ends at the next entry, or at the
end of the private GPA space for the final entry. `SUB_RANGE_SIZE_EXP` divides the range into
sub-ranges of 2^`SUB_RANGE_SIZE_EXP` bytes. If the sub-range size is equal to or larger than the
range size, the VMM cannot scan that range in parallel.

### RANGE_LIST_INFO

The range list is described by a 64-bit `RANGE_LIST_INFO` value with this layout:

- Bits 8:0 (`NUM_RANGES`): Number of valid range entries. It must be from 1 to
  `MAX_MEM_SCAN_RANGES`, readable with `TDH.SYS.RD*`.
- Bits 11:9: Reserved, must be 0.
- Bits 51:12: [PFN](x86-misc.md#page-numbers-and-page-frames) of the range list metadata page.
- Bits 63:52: Reserved, must be 0.

Example: range list with three GPA ranges.

```text
 63:52          51:12                11:9        8:0
+--------------+--------------------+--------------+----------------+
| Reserved = 0 | Metadata page PFN  | Reserved = 0 | NUM_RANGES = 3 |
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

### Parallel Scanning

The VMM can use ranges and sub-ranges to run the comprehensive scan in parallel. The TDX specs
describe NUMA optimization as an intended usage scenario: the VMM configures ranges that correspond
to NUMA nodes and runs `TDH.MEM.SCAN.COMP` concurrently on CPUs close to the memory of each range.
For this to work, the VMM should also distribute SEPT pages according to the NUMA node layout when
donating them via the `TDH.MEM.SEPT.ADD` seamcall.

The sub-ranges are not indexed or assigned in advance. An invocation of `TDH.MEM.SCAN.COMP` asks the
TDX module for the next unscanned sub-range of its range, scans it, and asks for another, until the
whole range is scanned. The VMM selects the range with `RANGE_ID`, which is the index of a range
entry. The first entry is `RANGE_ID` 0, the second is `RANGE_ID` 1, and so on. To scan one range in
parallel, the VMM runs concurrent invocations with the same `RANGE_ID`, each with its own
[scan context](#scan-contexts).

---

## Scan Contexts

A **scan context** is an internal TDX module data structure that holds the progress of one
invocation of the comprehensive scan seamcall
([TDH.MEM.SCAN.COMP](tdx-migration-seamcalls.md#tdhmemscancomp)). Like a
[stream context](#migration-stream), it is non-architectural and the VMM never accesses it. The VMM
refers to a scan context only by its ID, which it passes in the `CONTEXT_ID` input argument of
`TDH.MEM.SCAN.COMP`.

The TDX module keeps all scan contexts, the [range configuration](#range-list), and the state of the
scan in the control pages that the VMM donates with
[TDH.MEM.SCAN.CONFIG](tdx-migration-seamcalls.md#tdhmemscanconfig). The number of scan contexts is
fixed by the TDX module and enumerated by the `NUM_MEM_SCAN_CONTEXTS` global
[metadata field](tdx-structs.md#metadata-fields), readable with `TDH.SYS.RD*`. The VMM cannot change
it with `TDH.MEM.SCAN.CONFIG`. The VMM selects a scan context for each invocation by its ID, from 0
through `NUM_MEM_SCAN_CONTEXTS` - 1.

Each concurrent invocation needs its own scan context, so the number of scan contexts limits the
parallelism of the scan. Two invocations that scan different sub-ranges of the same range need two
scan contexts, and so do two invocations that scan different ranges.

`TDH.MEM.SCAN.COMP` is a [resumable](tdx-misc.md#resumable-vs-restartable) seamcall. On
interruption, it saves its progress in the scan context, including the sub-range that the invocation
was scanning. To resume, the VMM invokes the seamcall again with the same `CONTEXT_ID`. An
interrupted invocation still counts as an active invocation of its range, so the scan cannot
complete until the VMM resumes the invocation.

---

## Migration Bundle

A migration bundle is the unit of data produced by migration export seamcalls and consumed by
matching import seamcalls. For example,
[TDH.EXPORT.STATE.IMMUTABLE](tdx-migration-seamcalls.md#tdhexportstateimmutable) produces a bundle,
and [TDH.IMPORT.STATE.IMMUTABLE](tdx-migration-seamcalls.md#tdhimportstateimmutable) consumes it.

Every bundle has one MBMD (Migration Bundle Metadata). It is a small header that describes the
bundle, for example the protocol version and the bundle type. The TDX Migration ABI defines six
bundle types:

| Bundle type        | Produced by                  | Consumed by                  |
| ------------------ | ---------------------------- | ---------------------------- |
| Immutable state    | `TDH.EXPORT.STATE.IMMUTABLE` | `TDH.IMPORT.STATE.IMMUTABLE` |
| TD mutable state   | `TDH.EXPORT.STATE.TD`        | `TDH.IMPORT.STATE.TD`        |
| vCPU mutable state | `TDH.EXPORT.STATE.VP`        | `TDH.IMPORT.STATE.VP`        |
| TD private memory  | `TDH.EXPORT.MEM`             | `TDH.IMPORT.MEM`             |
| Epoch token        | `TDH.EXPORT.TRACK`           | `TDH.IMPORT.TRACK`           |
| Abort token        | `TDH.IMPORT.ABORT`           | `TDH.EXPORT.ABORT`           |

This document groups the bundle types as follows:

- **MBMD-only bundles**: The epoch token and abort token bundles.
- **State bundles**: The immutable state, TD mutable state, and vCPU mutable state bundles.
- **Memory bundle**: The TD private memory bundle.

The VMM allocates the buffers of a bundle, and the export seamcall fills them, encrypting the TD
state data. The VMM transports the bundle to the destination, where the import seamcall consumes it.

### MBMD

The MBMD has generic fields and fields specific to the bundle type. The maximum MBMD size is 128
bytes. Seamcalls that produce or consume an MBMD take the HPA and the size of the buffer that holds
it. The buffer size must be at least the maximum MBMD size, and the HPA must be aligned to 128
bytes. In practice, a buffer of 128 bytes aligned to 128 bytes works for all seamcalls.

The MBMD is not encrypted, so the VMM can read it. Examples of MBMD fields:

- `SIZE`: Total MBMD size in bytes.
- `MIG_VERSION`: Migration protocol version.
- `MIGS_INDEX`: Migration stream index.
- `MB_COUNTER`: Migration bundle counter of the stream, see [Migration Stream](#migration-stream).
- `MIG_EPOCH`: Migration epoch of the bundle, see
  [Migration Epochs](tdx-migration-misc.md#migration-epochs).
- `MB_TYPE`: Migration bundle type.
- `TOTAL_MB`: Total number of migration bundles that the source exported in the
  [export session](tdx-migration-misc.md#export-session), including the token. Present only in epoch
  tokens.
- `MAC`: MAC over the MBMD fields. In state bundles, it also covers the state pages. In the memory
  bundle, separate MACs cover the pages instead. The destination TDX module verifies this MAC.

### MBMD-Only Bundles

The epoch token and the abort token bundles contain only the MBMD. They carry control information
rather than TD state or memory contents.

The epoch token travels from the source to the destination, and the abort token travels from the
destination to the source. [TDH.EXPORT.ABORT](tdx-migration-seamcalls.md#tdhexportabort) consumes
the abort token only in the [out-of-order phase](tdx-migration-misc.md#migration-phases).

```text
MBMD-only bundle

+----------------------+
|        MBMD          |
| Epoch or abort token |
+----------------------+
```

### State Bundles

The term state bundles covers the immutable state, TD mutable state, and vCPU mutable state bundles.
They contain the MBMD and encrypted state data pages.

```text
State bundle

+----------+  +----------------------+  +----------+
|   MBMD   |  | Encrypted state page |  |   ...    |
+----------+  +----------------------+  +----------+
```

The state payload in these bundles is a serialized and encrypted collection of selected
[metadata fields](tdx-structs.md#metadata-fields).

**Note**: The state pages are in an [HPA list](#hpa-list). The HPA list metadata page is not part of
the state bundle, because its entries are HPAs on the source platform. The VMM builds a new metadata
page on the destination for the import seamcalls.

The following subsections describe the state that each of the three bundles carries.

#### Immutable State

**TD immutable state** is the TD state that is never modified after the TD measurement is finalized
by the `TDH.MR.FINALIZE` seamcall. It is composed of fields stored in [TDR](tdx-structs.md#tdr) and
[TDCS](tdx-structs.md#tdcs), such as [ATTRIBUTES](tdx-structs.md#attributes) or
[CPUID_CONFIG](tdx-structs.md#cpuid_config).

The immutable state bundle also carries the TDX module version, which is technically a piece of
platform immutable state. This document calls the combined state in the bundle **immutable state**.

The TDX module version is represented as a string with the following format:
`MAJOR_VERSION.MINOR_VERSION.UPDATE_VERSION.INTERNAL_VERSION.BUILD_NUM`, for example
"1.5.08.04.0234".

The number of child pages of the TD is not part of immutable state. Child pages are the 4KiB pages
associated with the TD, including private memory pages, SEPT pages, and
[TDVPS](tdx-structs.md#tdvps) control structure pages. The `TDR.CHLDCNT` field holds their count,
and it changes after finalization, for example when the VMM adds a pending page with
`TDH.MEM.PAGE.AUG` or removes a page to convert it from private to shared. See
[Private to Shared](tdx-misc.md#private-to-shared) for details.

**Note**: TDX specs also use the terms "TD immutable state" and "migrated immutable state"
interchangeably. This document uses **immutable state**.

#### TD Mutable State

**TD mutable state** is TD state that can change after the TD is finalized by `TDH.MR.FINALIZE`. For
example, the TD can modify the [TD_CTLS](tdx-structs.md#td_ctls) field of
[TDCS](tdx-structs.md#tdcs) at run time with the `TDG.VM.WR` tdcall.

TD mutable state is migrated only after the source TD is paused with
[TDH.EXPORT.PAUSE](tdx-migration-seamcalls.md#tdhexportpause), which prevents vCPUs from running, so
the state does not change during export.

**Note**: TDX specs also use the terms "TD-scope mutable state" and "TD-scope mutable non-memory
state". This document uses **TD mutable state**.

#### vCPU Mutable State

**vCPU mutable state** is the state of a vCPU that can change after the TD is finalized by
`TDH.MR.FINALIZE`. It is stored in [TDVPS](tdx-structs.md#tdvps). For example, guest GPRs change as
the vCPU runs.

vCPU mutable state is migrated after the TD mutable state, with one bundle per vCPU. The MBMD of the
bundle holds the vCPU index in the `VP_INDEX` field.

Before the import, the VMM must create the vCPU on the destination with `TDH.VP.CREATE` and add its
pages with `TDH.VP.ADDCX`.

**Note**: TDX specs also use the terms "VCPU-scope mutable state" and "VCPU-scope mutable non-memory
state". This document uses **vCPU mutable state**.

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

Not every GPA list entry has a memory page.
[TDH.EXPORT.MEM](tdx-migration-seamcalls.md#tdhexportmem) sets the `INVALID` flag (bit 63) in the
HPA list entry of a GPA when it produces no page for it, for example for a
[pending page](tdx-misc.md#leaf-entry-states), a [cancellation](tdx-misc.md#sept-export-states), or
an export error specific to the page. Such entries have no page, so there is nothing to transfer to
the destination host.

The HPA list metadata page is not part of the memory bundle, because its entries are HPAs on the
source platform. The VMM builds a new metadata page on the destination for the
[TDH.IMPORT.MEM](tdx-migration-seamcalls.md#tdhimportmem) seamcall.

**Note**: The TDX spec calls the memory bundle HPA list the MMBL (Memory Migration Buffer List).
This document calls it the HPA list for simplicity.

#### MAC Pages

The MAC pages contain one MAC for each exported page. There are one or two 4KiB MAC pages, and each
page contains up to 256 128-bit entries. The MACs follow the order of the [GPA list](#gpa-list): MAC
entry N belongs to GPA list entry N.

Each MAC covers the page contents together with the GPA list entry of the page, and with the
attributes page entry for partitioned TDs. If an entry has no page contents, the MAC covers only the
list entries. The MACs are generated on export (`TDH.EXPORT.MEM`) and verified on import
(`TDH.IMPORT.MEM`).

---

## Migration Stream

A **migration stream** is a TDX module abstraction, identified by a stream ID. Streams let the VMM
export and import bundles concurrently. A stream is not a network connection between the source and
destination hosts. The VMM chooses how to transport the bundles.

Seamcalls that use different streams can run concurrently. Two seamcalls cannot run on the same
stream at the same time. For example, the VMM cannot call `TDH.EXPORT.MEM` and `TDH.EXPORT.STATE.VP`
concurrently with the same stream ID. The VMM must serialize such calls, and a concurrent call fails
with a busy status.

To transfer in parallel, the VMM must use multiple streams on both the source and the destination.
For example, several threads on the source can run `TDH.EXPORT.MEM` on different pages, each thread
on its own stream, and the destination can run `TDH.IMPORT.MEM` in parallel on the same streams. The
VMM can also transfer memory and vCPU state at the same time. The intended usage is one or more
streams for each independent transfer flow of the VMM, such as a migration thread.

During the [in-order phase](tdx-migration-misc.md#migration-phases) of a migration session, the TDX
module enforces the order of bundles within each stream. Bundles exported on a stream must be
imported on the same stream in the same order. The [MBMD](#mbmd) of every bundle carries the stream
ID in the `MIGS_INDEX` field and a bundle counter of that stream in the `MB_COUNTER` field. The
destination TDX module uses the counter to enforce the order.

A **stream context** is an internal TDX module data structure (`MIGSC`) that holds the state of a
migration stream. The [TDH.MIG.STREAM.CREATE](tdx-migration-seamcalls.md#tdhmigstreamcreate)
seamcall creates it and takes the HPA of a page for the `MIGSC` as an input argument.
[TDCS](tdx-structs.md#tdcs) references this 4KiB page through an array of links.

**Note**: Each stream has exactly one stream context. This document mostly says stream, and uses
stream context or `MIGSC` only when the data structure itself matters.

A **stream ID** is an integer that identifies a migration stream within a TD.
`TDH.MIG.STREAM.CREATE` does not return it. The TDX module assigns stream IDs in creation order,
starting from 0, so the VMM must track creation order and treat the first created stream as ID 0,
the second as ID 1, and so on. The valid IDs are 0 up to the number of created streams minus 1.

Seamcalls that operate on migration streams take a stream ID as input. These seamcalls let the VMM
choose the stream ID:

- [TDH.EXPORT.MEM](tdx-migration-seamcalls.md#tdhexportmem)
- [TDH.IMPORT.MEM](tdx-migration-seamcalls.md#tdhimportmem)
- [TDH.EXPORT.STATE.VP](tdx-migration-seamcalls.md#tdhexportstatevp)
- [TDH.IMPORT.STATE.VP](tdx-migration-seamcalls.md#tdhimportstatevp)

For these seamcalls, the stream ID must be 0:

- [TDH.EXPORT.STATE.IMMUTABLE](tdx-migration-seamcalls.md#tdhexportstateimmutable)
- [TDH.IMPORT.STATE.IMMUTABLE](tdx-migration-seamcalls.md#tdhimportstateimmutable)
- [TDH.EXPORT.STATE.TD](tdx-migration-seamcalls.md#tdhexportstatetd)
- [TDH.IMPORT.STATE.TD](tdx-migration-seamcalls.md#tdhimportstatetd)
- [TDH.EXPORT.TRACK](tdx-migration-seamcalls.md#tdhexporttrack)
- [TDH.IMPORT.TRACK](tdx-migration-seamcalls.md#tdhimporttrack)
- [TDH.EXPORT.ABORT](tdx-migration-seamcalls.md#tdhexportabort)
- [TDH.IMPORT.ABORT](tdx-migration-seamcalls.md#tdhimportabort)

The number of streams per TD is limited by `MAX_MIGS`, which counts all streams of the TD.
`MAX_MIGS` is a global [metadata field](tdx-structs.md#metadata-fields) that the VMM can read with
`TDH.SYS.RD*`.

`TDH.MIG.STREAM.CREATE` fails if a migration session is in progress, so the VMM must create the
stream contexts before a session starts. The VMM can create more streams between sessions, up to the
`MAX_MIGS` limit. `TDH.EXPORT.STATE.IMMUTABLE` requires that stream 0 has been created.

The VMM must create at least as many streams on the destination as the
[export session](tdx-migration-misc.md#export-session) on the source uses. Otherwise,
`TDH.IMPORT.STATE.IMMUTABLE` fails with `TDX_NUM_MIGS_HIGHER_THAN_CREATED_FATAL`.

Stream contexts belong to the TD, not to a migration session. Each session reinitializes them in
`TDH.EXPORT.STATE.IMMUTABLE` or `TDH.IMPORT.STATE.IMMUTABLE`, and they stay unused when the session
ends. There is no explicit stream destruction seamcall. The donated stream context pages do not go
back to the VMM. They can be reclaimed only as part of the TD teardown.

---

## Migration Counters

The [TDCS](tdx-structs.md#tdcs) holds three page counters that the TDX module maintains for live
migration. Each counter counts 4KiB pages. The VMM can read the counters with `TDH.MNG.RD`, but it
cannot change them.

- `MEM_COUNT`: Number of TD private pages. Adding a page to the TD increments it, and removing a
  page from the TD decrements it.
- `MIG_COUNT`: Number of exported pages. Each page counts once, no matter how many times the VMM
  exports it. The counter changes as follows:
  - Incremented when [TDH.EXPORT.MEM](tdx-migration-seamcalls.md#tdhexportmem) exports a page with
    the `MIGRATE` operation. It is the first export of the page, or the export after a cancellation.
    A re-export (`REMIGRATE`) does not change it.
  - Decremented when `TDH.EXPORT.MEM` exports a cancellation of the page (`CANCEL`), and when
    [TDH.EXPORT.RESTORE](tdx-migration-seamcalls.md#tdhexportrestore) restores the SEPT entry of the
    page.
- `DIRTY_COUNT`: Number of exported pages that changed after their export, so they need another
  `TDH.EXPORT.MEM` operation. The counter changes as follows:
  - Incremented when a page exported in the current
    [export session](tdx-migration-misc.md#export-session) changes in a way that requires another
    `TDH.EXPORT.MEM` invocation. In
    [non-blocking mode](tdx-migration-misc.md#non-blocking-vs-write-blocking-export), there are two
    cases:
    - A memory scan detects a write from the Dirty bit, changes the SEPT entry to
      [EXPORTED_MODIFIED](tdx-misc.md#sept-export-states), and increments the counter. The scan is
      the `DSCAN` operation of [TDH.MEM.SCAN.RANGE](tdx-migration-seamcalls.md#tdhmemscanrange) or
      the `DCHECK` operation of [TDH.MEM.SCAN.COMP](tdx-migration-seamcalls.md#tdhmemscancomp).
    - A memory management operation changes the SEPT entry of the page directly. For example,
      `TDH.MEM.RANGE.BLOCK` sets it to `EXPORTED_BLOCKED`, which increments the counter unless the
      page is already counted as `EXPORTED_MODIFIED`. Blocked and removed (`EXPORTED_REMOVED`) pages
      need a `CANCEL` operation.
  - Decremented when `TDH.EXPORT.MEM` successfully processes the page, either by re-exporting it or
    canceling its earlier export.

The counters have two uses:

- **Start token**: With non-blocking export,
  [TDH.EXPORT.TRACK](tdx-migration-seamcalls.md#tdhexporttrack) generates the start token only if
  `DIRTY_COUNT` is 0 and `MIG_COUNT` equals `MEM_COUNT`. See
  [Start Token Checks](tdx-migration-seamcalls.md#start-token-checks).
- **Cleanup**: After an aborted export session, `MIG_COUNT` shows how many exported pages still need
  cleanup. A new export session requires it to be 0. See
  [SEPT Cleanup](tdx-migration-seamcalls.md#sept-cleanup).
