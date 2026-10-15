# TDX Live Migration: Seamcalls

- **Author**: Artem Bityutskiy
- **Version**: 0.1
- **Date**: 2026-07-03
- **Last updated**: 2026-11-04

**Disclaimer**: This document reflects my current understanding of TDX live migration seamcalls and
may contain errors or omissions. It is not an official Intel document. Refer to the official Intel
documentation for authoritative information.

This is not a standalone article. It is a supporting document that explains TDX live migration
seamcalls, intended for reference from my other documents.

This document is not an exhaustive reference for the TDX live migration seamcalls. It covers the
aspects that I consider important and relevant.

The document focuses on
[non-blocking export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export) and does not
cover write-blocking export. It describes pre-copy migration and does not cover
[post-copy migration](tdx-migration-misc.md#pre-copy-vs-post-copy-migration), where the destination
TD runs while the VMM delivers the remaining memory pages on demand.

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

- [TDX Live Migration: Seamcalls](#tdx-live-migration-seamcalls)
  - [Table of Contents](#table-of-contents)
  - [Common Behavior](#common-behavior)
    - [Common Arguments](#common-arguments)
    - [Fatal Status Codes](#fatal-status-codes)
  - [TDH.MIG.SETUP](#tdhmigsetup)
    - [Input Arguments](#input-arguments)
    - [Output Arguments](#output-arguments)
    - [Exit Codes](#exit-codes)
      - [Protocol Status Codes](#protocol-status-codes)
      - [Migration Error Codes](#migration-error-codes)
      - [Common Error Codes](#common-error-codes)
  - [TDH.MIG.SETUP.ABORT](#tdhmigsetupabort)
    - [Exit Codes](#exit-codes-1)
  - [TDH.MIG.STREAM.CREATE](#tdhmigstreamcreate)
    - [Input Arguments](#input-arguments-1)
    - [Output Arguments](#output-arguments-1)
    - [Exit Codes](#exit-codes-2)
  - [TDH.EXPORT.STATE.IMMUTABLE](#tdhexportstateimmutable)
    - [Input Arguments](#input-arguments-2)
    - [Output Arguments](#output-arguments-2)
    - [Exit Codes](#exit-codes-3)
  - [TDH.IMPORT.STATE.IMMUTABLE](#tdhimportstateimmutable)
    - [Input Arguments](#input-arguments-3)
    - [Output Arguments](#output-arguments-3)
    - [Exit Codes](#exit-codes-4)
  - [TDH.MEM.SCAN.RANGE](#tdhmemscanrange)
    - [Input Arguments](#input-arguments-4)
    - [Output Arguments](#output-arguments-4)
    - [GPA List Entries](#gpa-list-entries)
    - [Exit Codes](#exit-codes-5)
    - [EXPORT\_RESTORE](#export_restore)
      - [Arguments](#arguments)
      - [SEPT Entry Handling](#sept-entry-handling)
      - [Exit Codes](#exit-codes-6)
  - [TDH.EXPORT.MEM](#tdhexportmem)
    - [Input Arguments](#input-arguments-5)
    - [Output Arguments](#output-arguments-5)
    - [GPA List Entry Fields](#gpa-list-entry-fields)
      - [Input Fields](#input-fields)
      - [Output Fields](#output-fields)
      - [Cancellation](#cancellation)
      - [Entry Status Values](#entry-status-values)
    - [Exit Codes](#exit-codes-7)
  - [TDH.IMPORT.MEM](#tdhimportmem)
    - [Input Arguments](#input-arguments-6)
    - [Output Arguments](#output-arguments-6)
    - [Page Sources](#page-sources)
    - [Cancellation](#cancellation-1)
    - [GPA List Entry Fields](#gpa-list-entry-fields-1)
      - [Input Fields](#input-fields-1)
      - [Output Fields](#output-fields-1)
      - [Entry Status Values](#entry-status-values-1)
    - [Exit Codes](#exit-codes-8)
    - [Consequences of Import Failures](#consequences-of-import-failures)
  - [TDH.EXPORT.TRACK](#tdhexporttrack)
    - [Input Arguments](#input-arguments-7)
    - [Output Arguments](#output-arguments-7)
    - [Start Token Checks](#start-token-checks)
    - [Exit Codes](#exit-codes-9)
  - [TDH.IMPORT.TRACK](#tdhimporttrack)
    - [Input Arguments](#input-arguments-8)
    - [Output Arguments](#output-arguments-8)
    - [Token Processing](#token-processing)
    - [Exit Codes](#exit-codes-10)
  - [TDH.EXPORT.PAUSE](#tdhexportpause)
    - [Exit Codes](#exit-codes-11)
  - [TDH.EXPORT.STATE.TD](#tdhexportstatetd)
    - [Input Arguments](#input-arguments-9)
    - [Output Arguments](#output-arguments-9)
    - [Exit Codes](#exit-codes-12)
  - [TDH.IMPORT.STATE.TD](#tdhimportstatetd)
    - [Input Arguments](#input-arguments-10)
    - [Output Arguments](#output-arguments-10)
    - [Exit Codes](#exit-codes-13)
  - [TDH.EXPORT.STATE.VP](#tdhexportstatevp)
    - [Input Arguments](#input-arguments-11)
    - [Output Arguments](#output-arguments-11)
    - [Exit Codes](#exit-codes-14)
  - [TDH.IMPORT.STATE.VP](#tdhimportstatevp)
    - [Input Arguments](#input-arguments-12)
    - [Output Arguments](#output-arguments-12)
    - [Exit Codes](#exit-codes-15)
  - [TDH.IMPORT.COMMIT](#tdhimportcommit)
  - [TDH.IMPORT.END](#tdhimportend)
    - [Exit Codes](#exit-codes-16)
  - [TDH.IMPORT.ABORT](#tdhimportabort)
    - [Input Arguments](#input-arguments-13)
    - [Output Arguments](#output-arguments-13)
    - [Exit Codes](#exit-codes-17)
  - [TDH.EXPORT.ABORT](#tdhexportabort)
    - [SEPT Cleanup](#sept-cleanup)
    - [Input Arguments](#input-arguments-14)
    - [Output Arguments](#output-arguments-14)
    - [Exit Codes](#exit-codes-18)
  - [TDH.EXPORT.RESTORE](#tdhexportrestore)
    - [Input Arguments](#input-arguments-15)
    - [Output Arguments](#output-arguments-15)
    - [Restored States](#restored-states)
    - [GPA List Entry Fields](#gpa-list-entry-fields-2)
    - [Exit Codes](#exit-codes-19)
  - [TDH.MEM.SCAN.CONFIG](#tdhmemscanconfig)
    - [Input Arguments](#input-arguments-16)
    - [Output Arguments](#output-arguments-16)
    - [Exit Codes](#exit-codes-20)
  - [TDH.MEM.SCAN.RESET](#tdhmemscanreset)
    - [Exit Codes](#exit-codes-21)
  - [TDH.MEM.SCAN.COMP](#tdhmemscancomp)
    - [Input Arguments](#input-arguments-17)
    - [Output Arguments](#output-arguments-17)
    - [Output GPA List](#output-gpa-list)
    - [Resetting a Scan](#resetting-a-scan)
    - [Exit Codes](#exit-codes-22)
    - [Optional Pre-Scan](#optional-pre-scan)
      - [Accessed Bit Versus Dirty Bit](#accessed-bit-versus-dirty-bit)
      - [Pre-Scan Steps](#pre-scan-steps)
      - [PRECHECK List Mode](#precheck-list-mode)
      - [Input Arguments](#input-arguments-18)
      - [Output Arguments](#output-arguments-18)
      - [Exit Codes](#exit-codes-23)

---

## Common Behavior

This section describes the behavior that many seamcalls in this document share.

### Common Arguments

Most live migration seamcalls in this document take the [TD handle](tdx-structs.md#tdr) of the
source or the destination TD. The register that carries the handle differs between seamcalls, and
the sections below do not list the handle among their input arguments. The handle occupies bits
51:12 of the register. The other bits are reserved and must be 0, except for flags that only some
seamcalls define. The vCPU state seamcalls take a TDVPR page instead.

Most live migration seamcalls accept the `INTERRUPT_MODE` flag in bit 24 of `RAX`. When
`INTERRUPT_MODE` is 0, the TDX module detects a pending interrupt only if `RFLAGS.IF` is 1 in the
VMM. When it is 1, the TDX module detects a pending interrupt regardless of `RFLAGS.IF`. The
`HOST_SIDE_INTERRUPT_MODE` bit of [TDX_FEATURES0](tdx-structs.md#tdx_features0) enumerates whether
`INTERRUPT_MODE` is supported. See [Interruptible Seamcalls](tdx-misc.md#interruptible-seamcalls)
for more information about interruptible seamcalls.

The TDX module ignores the `INTERRUPT_MODE` flag in the following seamcalls, because they are not
interruptible:

- `TDH.EXPORT.PAUSE`
- `TDH.EXPORT.TRACK`
- `TDH.IMPORT.TRACK`
- `TDH.EXPORT.ABORT`
- `TDH.IMPORT.ABORT`
- `TDH.IMPORT.END`

### Fatal Status Codes

The TDX specs append the `_FATAL` suffix to the name of a [status code](tdx-misc.md#status-codes)
that has the `FATAL` bit (61) set. Such a code means that the TD can only be torn down. The import
seamcalls use these codes to report a failure that ends the import session.

---

## TDH.MIG.SETUP

The `TDH.MIG.SETUP` seamcall implements the [setup session](tdx-migration-misc.md#setup-session). It
is served by the [Migration Setup Service](tdx-migration-misc.md#migration-setup-service) (MSS) TDX
module extension.

The VMM invokes `TDH.MIG.SETUP` repeatedly on the source and the destination. The TDX module keeps
the state of the setup session and uses exit codes to tell the VMM what to do next, until the
seamcall returns `TDX_SUCCESS` or an unrecoverable error.

### Input Arguments

The VMM provides an input buffer and an output buffer. Both use the
[HPA linked list](tdx-migration-structs.md#hpa-linked-list) structure. The input buffer is optional
and carries data such as the migration policies, a peer TDX module message, or an attestation quote.
The TDX module writes the output data to the output buffer, for example, a `TDREPORT_STRUCT` or a
message to the peer.

- `CONTROLS` (`RDX`): Control bits, described below.
- `IN_BUFF_HPA_LIST` (`R8`): Input buffer HPA linked list or `NULL_PA` (-1).
- `IN_BUFF_ACTUAL_SIZE` (`R9`): Input buffer size in bytes, or 0.
- `OUT_BUFF_HPA_LIST` (`R10`): Output buffer HPA linked list.
- `OUT_BUFF_SIZE` (`R11`): Output buffer size in bytes.

The control bits (`CONTROLS`) are:

- `DIRECTION` (bit 32): 0 for the source, 1 for the destination. The value must be the same for all
  seamcall invocations in the session.
- `IN_BUFF_TYPE` (bits 39:33): Type of the input buffer contents, listed below.

| Type            | Value | Description                         |
|-----------------|-------|-------------------------------------|
| `EMPTY`         | 0     | No input buffer.                    |
| `MIG_POLICIES`  | 1     | Migration policies.                 |
| `MIG_SETUP_MSG` | 2     | A message from the peer TDX module. |
| `QUOTE`         | 3     | A TDX attestation quote.            |

The `MIG_POLICIES` buffer holds one or two Tag-Length-Value (TLV) structures. Each structure has
these fields:

- `TAG` (4 bytes): 1 for the platform migration policy, 2 for the TD migration policy.
- `LENGTH` (4 bytes): Size of the policy in bytes.
- Policy: A JSON document, see [Migration Policy](tdx-migration-misc.md#migration-policy).

The first invocation of a session depends on the direction:

- Source: Must use `MIG_POLICIES`. It must provide the TD migration policy and can also provide the
  platform migration policy. The TDX module checks that the SHA-384 hash of the TD migration policy
  matches `MIG_SETUP_TD_POLICY_HASH`.
- Destination: Can use `EMPTY` or `MIG_POLICIES`. In the latter case, it provides only the platform
  migration policy.

The `MIG_SETUP_MAX_BUFF_PAGES` global [metadata field](tdx-structs.md#metadata-fields), readable
with `TDH.SYS.RD`, defines the buffer size limit in 4 KiB pages. Two rules follow from it:

- `IN_BUFF_ACTUAL_SIZE` must not exceed 4096 * `MIG_SETUP_MAX_BUFF_PAGES` bytes.
- `OUT_BUFF_SIZE` must be at least 4096 * `MIG_SETUP_MAX_BUFF_PAGES` bytes, so that the output
  buffer can always hold the largest possible output.

To provide no input buffer, the VMM must set all of the following together:

- `IN_BUFF_TYPE` to `EMPTY`.
- `IN_BUFF_HPA_LIST` to `NULL_PA`.
- `IN_BUFF_ACTUAL_SIZE` to 0.

### Output Arguments

The seamcall returns the output data in the output buffer and two values in registers:

- Exit code (`RAX`): Tells the VMM whether the session completed, failed, or needs another
  invocation. See the next section for details.
- `OUT_BUFF_ACTUAL_SIZE` (`RCX`): Actual size of the output data in bytes, 0 if no output data was
  produced.

### Exit Codes

The exit codes fall into the following categories:

- **Success**: The session completed successfully. No further invocation is needed. Indicated by
  `TDX_SUCCESS`.
- **Protocol Status**: The seamcall succeeded, but the VMM needs to take action and invoke it again.
- **Migration Error**: Codes specific to the MSS protocol, all prefixed `TDX_MIG_SETUP_`.
- **Common Error**: Generic TDX module [status codes](tdx-misc.md#status-codes).

**Note**: The subsections below describe the main exit codes of each category. The lists are not
exhaustive.

#### Protocol Status Codes

- `TDX_MIG_SETUP_MSG_EXCHANGE`: The VMM sends the output to the peer, waits for a reply, then
  invokes the seamcall again with `IN_BUFF_TYPE` set to `MIG_SETUP_MSG`.
- `TDX_MIG_SETUP_MSG_SEND`: The VMM sends the output to the peer, then invokes the seamcall again
  with `IN_BUFF_TYPE` set to `EMPTY`.
- `TDX_MIG_SETUP_MSG_RECEIVE`: The VMM waits for a message from the peer, then invokes the seamcall
  again with `IN_BUFF_TYPE` set to `MIG_SETUP_MSG`.
- `TDX_MIG_SETUP_GET_QUOTE`: The VMM gets a quote for the `TDREPORT_STRUCT` in the output buffer
  (see [Quoting Service](tdx-misc.md#quoting-service)), then invokes the seamcall again with
  `IN_BUFF_TYPE` set to `QUOTE`.

#### Migration Error Codes

- `TDX_MIG_SETUP_ABSENT_TD_POLICY`: The
  [TD migration policy](tdx-migration-misc.md#migration-policy) is missing or empty. The session
  ended unsuccessfully.
- `TDX_MIG_SETUP_PLATFORM_POLICY_INVALID`: Platform migration policy is invalid, for example because
  of a bad format. The session ended unsuccessfully.
- `TDX_MIG_SETUP_POLICY_FAILURE`: A migration policy check failed, so migration is not allowed. The
  session ended unsuccessfully.
- `TDX_MIG_SETUP_TD_POLICY_HASH_MISMATCH`: The hash of the provided TD migration policy does not
  match `MIG_SETUP_TD_POLICY_HASH`. The session ended unsuccessfully.
- `TDX_MIG_SETUP_TD_POLICY_HASH_ZERO`: `MIG_SETUP_TD_POLICY_HASH` is zero. Setup has not started.
- `TDX_MIG_SETUP_QUOTE_INVALID`: The quote is invalid. The session ended unsuccessfully.
- `TDX_MIG_SETUP_AKE_FAILURE`: Authenticated key exchange failed. The session ended unsuccessfully.
- `TDX_MIG_SETUP_SESSION_ACTIVE`: The setup session is already active.
- `TDX_MIG_SETUP_SESSION_ABORTING`: The session is aborting, so setup cannot continue. The VMM
  completes the abort with `TDH.MIG.SETUP.ABORT`.

#### Common Error Codes

- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): The VMM resumes by executing the
  `TDH.MIG.SETUP` seamcall again for the same TD handle. The other input operands are ignored. The
  progress stays in the [virtual thread](tdx-misc.md#virtual-threads) of the extension, so the VMM
  must not start a new request instead. It can abort the session with `TDH.MIG.SETUP.ABORT`.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `DIRECTION` differs from the
  previous invocation of the same session.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The source TD must be in `RUNNABLE` or `LIVE_IMPORT`, and the destination TD must
  be in `UNINITIALIZED`.
- `TDX_EXT_FATAL_ERROR`: A fatal error occurred in the extension. The session ended unsuccessfully.
- `TDX_EXT_NOT_INITIALIZED`: The [extension](tdx-misc.md#extension-setup) context is not
  initialized.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A TDX module resource necessary for migration is
  busy. The operand is `MIG_SETUP` when another `TDH.MIG.SETUP` or `TDH.MIG.SETUP.ABORT` is running
  for the same TD, `MIG_SETUP_SESSIONS` when no free
  [sessions](tdx-migration-misc.md#sessions-and-threads) are available, and `VIRTUAL_THREAD_POOL`
  when no free threads are available.

---

## TDH.MIG.SETUP.ABORT

The `TDH.MIG.SETUP.ABORT` seamcall aborts the [setup session](tdx-migration-misc.md#setup-session)
on the source or the destination. The VMM uses it to cancel the migration setup.

The VMM can invoke `TDH.MIG.SETUP.ABORT` at any time when no other setup operation for the same TD
is running.

The only input argument is the [TD handle](tdx-structs.md#tdr) in `RCX`. The only output is the exit
code in `RAX`.

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The session was aborted.
- `TDX_MIG_SETUP_SESSION_INACTIVE`: The TD has no active setup session, so there is nothing to
  abort. This is the only MSS-specific error code of this seamcall.
- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): The VMM resumes by executing the
  `TDH.MIG.SETUP.ABORT` seamcall again for the same TD handle. The other input operands are ignored.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall.
- `TDX_EXT_NOT_INITIALIZED`: The [extension](tdx-misc.md#extension-setup) context is not
  initialized.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): The operand is `MIG_SETUP` when another setup
  operation is running for the same TD.

**Note**: In my opinion, aborting an interrupted `TDH.MIG.SETUP` gives no benefit. The current TDX
module implementation can interrupt `TDH.MIG.SETUP.ABORT` too, and the VMM must resume it. The VMM
handles interruptions either way, so it can simply resume `TDH.MIG.SETUP` instead.

---

## TDH.MIG.STREAM.CREATE

The `TDH.MIG.STREAM.CREATE` seamcall creates a
[migration stream](tdx-migration-structs.md#migration-stream) for the source or the destination TD.
The VMM donates a page, and the TDX module turns it into the stream context (`MIGSC`), the data
structure that holds the state of the stream.

The VMM can create streams after it adds the [TDCS](tdx-structs.md#tdcs) pages with `TDH.MNG.ADDCX`
and before the [migration session](tdx-migration-misc.md#migration-session) starts.

The seamcall does not return the stream ID. The TDX module assigns IDs in creation order, starting
from 0. The number of streams per TD is limited by `MAX_MIGS`.

The VMM can reclaim the stream context page only when it tears down the TD.

### Input Arguments

- `MIGSC` page (`RCX`): HPA of the 4 KiB page for the stream context.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The stream was created.
- `TDX_MISSING_PAMT_PAGE_PAIR`: With dynamic PAMT, a PAMT page pair is missing for the `MIGSC` page.
  The VMM adds it with `TDH.PHYMEM.PAMT.ADD` and retries.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall, because a migration session is in progress.
- `TDX_OPERAND_INVALID`: An input operand is invalid, for example a reserved bit is set.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy.

---

## TDH.EXPORT.STATE.IMMUTABLE

The `TDH.EXPORT.STATE.IMMUTABLE` seamcall exports the
[immutable state](tdx-migration-structs.md#immutable-state) of the source TD. It produces an
[immutable state bundle](tdx-migration-structs.md#state-bundles). The VMM delivers the bundle to the
destination host and imports it there with the `TDH.IMPORT.STATE.IMMUTABLE` seamcall.

When the seamcall completes, the source TD enters the `LIVE_EXPORT`
[operation state](tdx-misc.md#td-operation-states). If the seamcall is interrupted, the TD stays in
the `START_EXPORT` state until the VMM resumes the seamcall or aborts the export session.

### Input Arguments

The VMM provides the memory for the [immutable state bundle](tdx-migration-structs.md#state-bundles)
and the TDX module writes the bundle there. The VMM provides a buffer for the
[MBMD](tdx-migration-structs.md#mbmd) and an [HPA list](tdx-migration-structs.md#hpa-list) that
points to the pages for the encrypted immutable state.

- `EXPORT_TYPE` (`RCX`): Export type, must be 0 for migration.
- MBMD buffer (`R8`): HPA and size of the buffer for the MBMD.
- `HPA_LIST_INFO` (`R9`): [Descriptor](tdx-migration-structs.md#hpa_list_info) of the HPA list for
  the encrypted state pages.
- `MIGS_INDEX` (`R10`): [Migration stream](tdx-migration-structs.md#migration-stream) index, must be
  0.
- `RESUME` (`R10`): 1 when resuming after an interrupted exit code.

Here are the details of the arguments:

- Bit 0 of the `RCX` register is `EXPORT_TYPE`. Value 1 selects S4 hibernation, which is outside the
  scope of this document.
- `R8` has the HPA of the MBMD buffer in bits 51:0 and the buffer size in bits 63:52.
- Bits 2:0 (`FORMAT`) and 11:3 (`FIRST_ENTRY`) of the `R9` register must be 0, also on resumption.
  The rest of `R9` is the HPA list address and the last entry index. The TDX module saves the
  progress in the migration stream context, so the VMM passes the same `R9` value on resumption.
- Bits 15:0 of the `R10` register are `MIGS_INDEX` and bit 63 is `RESUME`. The other bits are
  reserved and must be 0.

The required number of pages in the HPA list is enumerated by the `NUM_IMMUTABLE_STATE_PAGES` global
[metadata field](tdx-structs.md#metadata-fields), readable with `TDH.SYS.RD`.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.
- Page count (`RDX`): Number of 4 KiB state pages in the bundle. It does not count the MBMD buffer
  or the HPA list metadata page.

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The immutable state bundle was exported and the
  [export session](tdx-migration-misc.md#export-session) has started.
- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): The VMM resumes by executing the
  seamcall again with `RESUME` set to 1. The TD stays in `START_EXPORT`.
- `TDX_INVALID_RESUMPTION`: `RESUME` is set while no resumption is expected, or clear while a
  resumption is expected.
- `TDX_INCOMPATIBLE_MBMD_MAC_CONTEXT`: The TDX module was updated while the seamcall was
  interrupted, and the update was configured to detect this incompatibility. The seamcall cannot
  resume, and the TD stays in `START_EXPORT`. The VMM must abort the export session.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The source TD must be in `RUNNABLE` or `LIVE_IMPORT`.
- `TDX_TD_NOT_MIGRATABLE`: The TD does not have [ATTRIBUTES](tdx-structs.md#attributes) bit
  `MIGRATABLE` set.
- `TDX_MIGRATION_DECRYPTION_KEY_NOT_SET`: The migration decryption key is not set, because the setup
  session with `TDH.MIG.SETUP` has not completed successfully. The key must be set again for each
  migration session.
- `TDX_MIN_MIGS_NOT_CREATED`: Migration stream 0 has not been created.
- `TDX_PREVIOUS_EXPORT_CLEANUP_INCOMPLETE`: A previous aborted export session has not been cleaned
  up.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `EXPORT_TYPE` or `MIGS_INDEX` is
  not 0, a reserved bit is set, or the MBMD buffer is too small or misaligned.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example the migration
  stream.

---

## TDH.IMPORT.STATE.IMMUTABLE

The `TDH.IMPORT.STATE.IMMUTABLE` seamcall imports the
[immutable state bundle](tdx-migration-structs.md#state-bundles) produced by
`TDH.EXPORT.STATE.IMMUTABLE`. It starts the [import session](tdx-migration-misc.md#import-session)
on the destination platform.

When the seamcall completes, the destination TD enters the `MEMORY_IMPORT`
[operation state](tdx-misc.md#td-operation-states). If the seamcall is interrupted, the TD stays in
the `START_IMPORT` state until the VMM resumes the seamcall.

To build the destination TD, the VMM first creates it with `TDH.MNG.CREATE`, configures its key with
`TDH.MNG.KEY.CONFIG`, and adds its [TDCS](tdx-structs.md#tdcs) pages with `TDH.MNG.ADDCX`. The
`TDH.IMPORT.STATE.IMMUTABLE` seamcall then imports the TD configuration and the finalized
measurements. The memory is imported later with the `TDH.IMPORT.MEM` seamcall.

In other words, compared to building a TD from scratch, the destination TD skips `TDH.MNG.INIT`,
`TDH.MEM.PAGE.ADD`, `TDH.MR.EXTEND`, and `TDH.MR.FINALIZE`.

The seamcall checks that the imported state is compatible with the destination platform and TDX
module configuration. For example:

- [ATTRIBUTES](tdx-structs.md#attributes) and [XFAM](tdx-structs.md#xfam) bits configured on the
  source are supported by the destination platform and TDX module configuration.
- The source virtual CPUID configuration is supported by the destination CPU.

### Input Arguments

The VMM provides the memory for the [immutable state bundle](tdx-migration-structs.md#state-bundles)
received from the source, and the TDX module reads the bundle from there. The VMM provides a buffer
for the [MBMD](tdx-migration-structs.md#mbmd) and an [HPA list](tdx-migration-structs.md#hpa-list)
that points to the pages with the encrypted immutable state.

- `IMPORT_TYPE` (`RCX`): Import type, must be 0 for migration.
- MBMD buffer (`R8`): HPA and size of the buffer for the MBMD.
- `HPA_LIST_INFO` (`R9`): [Descriptor](tdx-migration-structs.md#hpa_list_info) of the HPA list for
  the encrypted state pages.
- `MIGS_INDEX` (`R10`): [Migration stream](tdx-migration-structs.md#migration-stream) index, must be
  0.
- `RESUME` (`R10`): 1 when resuming after an interrupted exit code.

Here are the details of the arguments:

- Bit 0 of the `RCX` register is `IMPORT_TYPE`. Value 1 selects S4 resumption, which is outside the
  scope of this document.
- `R8` has the HPA of the MBMD buffer in bits 51:0 and the buffer size in bits 63:52.
- Bits 2:0 (`FORMAT`) and 11:3 (`FIRST_ENTRY`) of the `R9` register must be 0, also on resumption.
  Bits 63:55 (`LAST_ENTRY`) must be the number of pages returned in `RDX` by
  `TDH.EXPORT.STATE.IMMUTABLE` minus 1. The rest of `R9` is the HPA list address. The TDX module
  saves the progress in the migration stream context, so the VMM passes the same `R9` value on
  resumption.
- Bits 15:0 of the `R10` register are `MIGS_INDEX` and bit 63 is `RESUME`. The other bits are
  reserved and must be 0.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.
- Error information (`RCX`): The seamcall checks every
  [metadata field](tdx-structs.md#metadata-fields) of the bundle against the destination TD and the
  TDX module configuration. If a check fails, `RCX` identifies the offending field or the field
  sequence header, for example a field whose value the destination does not support. See the exit
  codes below.

### Exit Codes

Codes with the `_FATAL` suffix fail the import and move the destination TD to `FAILED_IMPORT`.

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The immutable state bundle was imported and the import session has started.
- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): The VMM resumes by executing the
  seamcall again with `RESUME` set to 1. If the VMM cannot resume, it can abort the import session
  with [TDH.IMPORT.ABORT](#tdhimportabort).
- `TDX_INVALID_RESUMPTION`: `RESUME` is set while no resumption is expected, or clear while a
  resumption is expected.
- `TDX_INVALID_RESUMPTION_FATAL`: On resumption, `HPA_LIST_INFO` differs from the value used before
  the interruption, or the interrupted seamcall was another one.
- `TDX_INCOMPATIBLE_MBMD_MAC_CONTEXT_FATAL`: The TDX module was updated while the seamcall was
  interrupted, and the update was configured to detect this incompatibility. The VMM must abort the
  import session and can restart it.
- `TDX_INCORRECT_MBMD_MAC_FATAL`: The MAC of the bundle is incorrect. This also happens if the TDX
  module was updated while the seamcall was interrupted and the update was not configured to detect
  the incompatibility.
- `TDX_INVALID_MBMD_FATAL`: The MBMD is invalid.
- `TDX_METADATA_FIELD_NOT_WRITABLE_FATAL`: A metadata field in the bundle is not writable. `RCX`
  contains the ABI field ID.
- `TDX_METADATA_FIELD_VALUE_NOT_VALID_FATAL`: A metadata field in the bundle has an invalid value.
  `RCX` contains the ABI field ID.
- `TDX_REQUIRED_METADATA_FIELD_MISSING_FATAL`: A required metadata field is missing from the bundle.
  `RCX` contains the ABI field ID.
- `TDX_METADATA_FIELD_ID_INCORRECT_FATAL`: A field ID or a field sequence header in the bundle is
  incorrect. `RCX` contains the field ID or the field sequence header.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The destination TD must be in `UNINITIALIZED`.
- `TDX_MIGRATION_DECRYPTION_KEY_NOT_SET`: The migration decryption key is not set, because the setup
  session with `TDH.MIG.SETUP` has not completed successfully. The key must be set again for each
  migration session.
- `TDX_MIN_MIGS_NOT_CREATED`: Migration stream 0 has not been created.
- `TDX_NUM_MIGS_HIGHER_THAN_CREATED_FATAL`: The
  [export session](tdx-migration-misc.md#export-session) on the source uses more migration streams
  than the VMM created on the destination.
- `TDX_RND_NO_ENTROPY`: The TDX module failed to generate a random migration encryption key, which
  is used for the next migration session. This means that the CPU random number generator had no
  entropy. The VMM should retry.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `IMPORT_TYPE` or `MIGS_INDEX` is
  not 0, or a reserved bit is set.
- `TDX_OPERAND_INVALID_FATAL`: The MBMD buffer, the HPA list descriptor, or the HPA of a state page
  is invalid.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example the migration
  stream.

---

## TDH.MEM.SCAN.RANGE

The `TDH.MEM.SCAN.RANGE` seamcall scans the SEPT entries that correspond to a range of private
[GPAs](x86-misc.md#guest-addresses) of the source TD. It performs the `DSCAN` operation, which
reports export candidates for `TDH.EXPORT.MEM`. It also supports the `EXPORT_RESTORE` operation,
which restores SEPT entry states after an aborted export session.

The description below covers `DSCAN` first. `EXPORT_RESTORE` follows at the end.

`DSCAN` is available only if the TDX module is configured for
[non-blocking export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export). The VMM selects
the mode during TDX module initialization, see [Export Mode](tdx-migration-misc.md#export-mode).

The `RANGE_START` and `RANGE_SIZE` input arguments define the GPA range to scan. For `DSCAN`, the
VMM also provides a [GPA list-of-lists](tdx-migration-structs.md#gpa-list-of-lists) as an output
buffer, and `DSCAN` writes one GPA list entry per export candidate there. Each entry holds the GPA
of the page and the reason why it is a candidate, as described below. `DSCAN` writes nothing for the
other GPAs.

A private page is an export candidate if it was never exported, or if it changed after its last
export. There are two kinds of changes:

- **Data changes**: The page was written, so the Dirty bit of its SEPT entry is set.
- **Metadata changes**: The [SEPT entry state](tdx-misc.md#sept-page-states) or the attributes
  changed, and the page content did not necessarily change.

Here are examples of metadata changes that make a page with no data changes an export candidate:

- The TD changes the permissions of the page with `TDG.MEM.PAGE.ATTR.WR`.
- The VMM blocks the page after its export (the [EXPORTED_BLOCKED](tdx-misc.md#sept-export-states)
  SEPT state).
- The VMM removes the page after its export, for example when the TD converts it to shared, see
  [Private to Shared](tdx-misc.md#private-to-shared).

Not every candidate is reported, though. `DSCAN` is best effort, so a single scan can miss some of
them. For example, it skips SEPT entries in the [BLOCKED](tdx-misc.md#leaf-entry-states) state, and
entries used by a concurrent operation.

When `DSCAN` detects a data change in a page, it clears the Dirty bit of the SEPT entry. The next
scan then detects the pages that were written since the previous scan.

The scan also records the current TD epoch in the PAMT entry of the page, see
[Scan Epoch](tdx-misc.md#scan-epoch). The [TLB tracking](tdx-misc.md#tlb-tracking) protocol needs
the epoch, and the protocol must complete before `TDH.EXPORT.MEM` exports the reported pages.

**Note**: The VMM can invoke `DSCAN` concurrently on multiple CPUs, for example on different GPA
ranges. `DSCAN` of one range can also run concurrently with `TDH.EXPORT.MEM` of another range.

### Input Arguments

These are the arguments of `DSCAN`.

- `LIST_OF_LISTS_INFO` (`RCX`): [Descriptor](tdx-migration-structs.md#list_of_lists_info) of the
  output GPA list-of-lists. The VMM sets up the structure, and the TDX module writes only the GPA
  list entries.
- `CONTROLS` (`R8`): Operation and qualifier, described below.
- `RANGE_START` (`R9`): Start of the private GPA range. It must be 4 KiB aligned.
- `RANGE_SIZE` (`R10`): Size of the GPA range in bytes. It must be nonzero and a multiple of 4 KiB.
- `OVERALL_NEXT_ENTRY` (`R11`): Index of the next GPA list entry to write, counted over all GPA list
  pages, see [entry index](tdx-migration-structs.md#overall_next_entry). It is 0 for a new empty
  list-of-lists, and otherwise the value returned by the previous invocation.

Here are the details of the arguments:

- Bits 2:0 (`FORMAT`) of the `RCX` register must be 2 (`LIST_OF_LISTS`), and bits 11:3
  (`FIRST_ENTRY`) must be 0. The rest of `RCX` is the HPA of the top-level page and the last entry
  index.
- The `R8` register is `CONTROLS`:
  - `OPERATION` (bits 7:0): Must be 0, which means `DSCAN`.
  - `QUALIFIER` (bits 15:8): 0 (`EXPORT`) reports both pages that were never exported and pages that
    need re-export. 1 (`REEXPORT`) reports only pages that need re-export.
  - The other bits are reserved and must be 0.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.
- `NEXT_START` (`R9`): GPA where the scan stopped. The next invocation continues from it.
- `REMAINING_SIZE` (`R10`): Remaining size of the GPA range, a multiple of 4 KiB. It is 0 on
  success.
- `OVERALL_NEXT_ENTRY` (`R11`): Index of the next free GPA list entry. The next invocation continues
  filling the list from it.

### GPA List Entries

The GPA list-of-lists contains GPA list entries. Here are the most relevant fields of an entry:

- `GPA` (bits 51:12): [GFN](x86-misc.md#page-numbers-and-page-frames) of the export candidate page.
- `LEVEL` (bits 1:0): Page level of the GPA: 4 KiB, 2 MiB, or 1 GiB. The VMM must split large pages
  before exporting them with `TDH.EXPORT.MEM`.
- `PENDING` (bit 2): The page is in a `PENDING*` SEPT [state](tdx-misc.md#leaf-entry-states), which
  means the TD has not accepted it yet.
- `STATE` (bits 4:3): The reason why the page is an export candidate. The values, except for
  `NOT_EXPORTED`, are named after the [SEPT page state](tdx-misc.md#sept-export-states) that caused
  the report:
  - `NOT_EXPORTED` (0): The page was never exported. `DSCAN` reports such pages only with the
    `EXPORT` qualifier.
  - `EXPORTED_MODIFIED` (1): The page was exported and later modified or its attributes changed.
  - `EXPORTED_BLOCKED` (2): The page was exported and later blocked.
  - `EXPORTED_REMOVED` (3): The page was exported and later removed.
- `OPERATION` (bits 53:52): Always `MIGRATE` (1).
- `STATUS` (bits 60:56): Always `SUCCESS`.

`PENDING` and `STATE` are independent fields. `DSCAN` always writes both, so any combination is
possible. `PENDING` reflects the current acceptance by the TD, while `STATE` reflects the export
history. For example:

- A page that was never exported and is still unaccepted is reported as `PENDING`=1 and
  `STATE`=`NOT_EXPORTED`.
- A page that was exported while pending, is still unaccepted, and now needs re-export because its
  attributes changed is reported as `PENDING`=1 and `STATE`=`EXPORTED_MODIFIED`. The TDX module
  exports the attributes of a pending page but not the content.

`PENDING` and `STATE` are hints for the VMM. `TDH.EXPORT.MEM` ignores both on input and decides what
to export from the SEPT entry state of the page.

The output GPA list-of-lists is a collection of GPA lists. `DSCAN` fills them so that the VMM can
pass each GPA list to `TDH.EXPORT.MEM` as is. This is why `DSCAN` initializes `OPERATION` and
`STATUS` of each entry to fixed values. The VMM can still change `OPERATION` of an entry to `NOP` to
skip the page. See
[GPA List Entry Operation Values](tdx-migration-structs.md#gpa-list-entry-operation-values) for the
`OPERATION` values that `TDH.EXPORT.MEM` accepts.

A GPA list-of-lists holds at most 262,144 GPA list entries, which can be fewer than the number of
export candidates in the scanned range. When the list-of-lists runs out of free entries, the
seamcall returns `TDX_INTERRUPTED_LIST_FULL`. The VMM then processes the reported GPA list entries,
for example with `TDH.EXPORT.MEM`, provides a list-of-lists with free entries, and resumes the
seamcall. It repeats this as many times as needed.

### Exit Codes

Here are the notable exit codes of `DSCAN`.

- `TDX_SUCCESS`: The SEPT entries of the whole GPA range were scanned.
- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): A pending interrupt was
  detected. The VMM resumes by executing the seamcall again with the continuation values returned in
  `R9`, `R10`, and `R11`, keeping `CONTROLS` unchanged. There is no resume flag.
- [TDX_INTERRUPTED_BUSY](tdx-misc.md#tdx_operand_busy-vs-tdx_interrupted_busy): The TDX module
  yielded because a concurrent memory management operation, for example `TDH.MEM.PAGE.PROMOTE`,
  could not lock the SEPT. The VMM resumes the same way.
- `TDX_INTERRUPTED_LIST_FULL`: The GPA list-of-lists ran out of free entries. The VMM provides a
  list-of-lists with free entries, for example a new one with `OVERALL_NEXT_ENTRY` set to 0, and
  resumes with the continuation values returned in `R9` and `R10`.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `RANGE_SIZE` is 0 or a reserved
  bit is set.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy.

### EXPORT_RESTORE

The `EXPORT_RESTORE` operation restores SEPT entry states after the VMM aborted an
[export session](tdx-migration-misc.md#export-session) with [TDH.EXPORT.ABORT](#tdhexportabort). The
abort leaves the exported pages in their [export states](tdx-misc.md#sept-export-states), and the
operation moves them back to the matching non-exported states, for example `EXPORTED_MODIFIED` to
`MAPPED`.

The operation is available only if the `SCAN_EXPORT_RESTORE` bit (43) of
[TDX_FEATURES0](tdx-structs.md#tdx_features0) is set. It works for both
[write-blocking](tdx-migration-misc.md#non-blocking-vs-write-blocking-export) and non-blocking
export.

According to the specification, the operation requires that no export session is in progress. In
practice, this means that `TDH.EXPORT.ABORT` completed successfully and the TD is in the
[RUNNABLE](tdx-misc.md#td-operation-states) state.

On input, the VMM provides a GPA range. The seamcall finds the pages in an export state within the
range and restores them.

**Note**: If the VMM knows which pages need restoring and does not need to scan SEPT to find them,
it can use [TDH.EXPORT.RESTORE](#tdhexportrestore) instead.

#### Arguments

The input and output arguments are the same as for `DSCAN`, except for the following:

- `OPERATION` (`R8`, bits 7:0): Must be `EXPORT_RESTORE` (2).
- `QUALIFIER` (`R8`, bits 15:8): Must be 0.
- `LIST_OF_LISTS_INFO` (`RCX`): Must be 0, because the operation writes no GPA list entries.
- `OVERALL_NEXT_ENTRY` (`R11`): Must be 0 on input and is 0 on output, for the same reason.

#### SEPT Entry Handling

The state changes are the same as for `TDH.EXPORT.RESTORE`, described in
[Restored States](#restored-states).

Unlike `DSCAN`, `EXPORT_RESTORE` does not record the [scan epoch](tdx-misc.md#scan-epoch) in the
PAMT entries of the pages. It reports and exports nothing, so it does not need the epoch.

`DSCAN` is best effort and skips the entries that it cannot process. `EXPORT_RESTORE` cannot skip
them, because it must restore all pages in an export state and bring the
[MIG_COUNT](tdx-migration-structs.md#migration-counters) counter to 0. A new export session requires
it.

This is why the two operations handle busy and blocked entries differently:

- **Busy entry**: A concurrent operation holds the lock of the SEPT entry.
  - `DSCAN`: Skips the entry.
  - `EXPORT_RESTORE`: Yields with
    [TDX_INTERRUPTED_BUSY](tdx-misc.md#tdx_operand_busy-vs-tdx_interrupted_busy) and the VMM invokes
    the seamcall again from `NEXT_START`.
- **Blocked non-leaf entry**: A non-leaf SEPT entry is in the
  [NL_BLOCKED](tdx-misc.md#non-leaf-entry-states) state.
  - `DSCAN`: Skips the blocked range.
  - `EXPORT_RESTORE`: Stops with `TDX_GPA_RANGE_BLOCKED`, because the range can hold pages in an
    export state, such as `EXPORTED_BLOCKED`, and the operation cannot restore them while the range
    is blocked. The VMM must unblock the range and invoke the seamcall again from `NEXT_START`.

#### Exit Codes

The exit codes are the same as for `DSCAN`, except for the following:

- `TDX_GPA_RANGE_BLOCKED`: A non-leaf SEPT entry in the range is blocked, and `NEXT_START` (`R9`)
  identifies the blocked range. The VMM must unblock the range, for example by finishing the
  operation that blocked it, and invoke the seamcall again with the continuation values returned in
  `R9` and `R10`.
- [TDX_INTERRUPTED_BUSY](tdx-misc.md#tdx_operand_busy-vs-tdx_interrupted_busy): Also returned when a
  SEPT entry of a page is busy. The VMM resumes the same way as for `DSCAN`.
- `TDX_INTERRUPTED_LIST_FULL`: Never returned, because the operation writes no GPA list entries.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall, because an export session is in progress. The VMM must abort it first.

---

## TDH.EXPORT.MEM

The `TDH.EXPORT.MEM` seamcall takes a [GPA list](tdx-migration-structs.md#gpa-list) of up to 512
[export candidates](#tdhmemscanrange) and exports their data and metadata. It produces one
[memory bundle](tdx-migration-structs.md#memory-bundle), which the VMM delivers to the destination
platform and imports with the `TDH.IMPORT.MEM` seamcall. Only 4 KiB pages are supported.

The seamcall exports the page data, in an encrypted form, to the pages listed in the
[HPA list](tdx-migration-structs.md#hpa-list) provided by the VMM. It exports the page metadata in
the GPA list entries, one entry for each page. The metadata tells the destination what to do with
the page (import it, import it again, or cancel the earlier import).

Whether the seamcall exports data, and what metadata it produces, depends on the
[SEPT entry state](tdx-misc.md#sept-export-states) of the page. Here are the main cases for
[non-blocking export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export):

- **Data and metadata**: The page is `MAPPED` (which means it was not exported yet) or
  `EXPORTED_MODIFIED`. The destination imports the page, or imports it again.
- **Metadata only**: The page is pending, so the TD has not accepted it and there is no content to
  export. The SEPT state is `PENDING` or `PENDING_EXPORTED_MODIFIED`. The destination imports only
  page metadata.
- **Cancellation**: The VMM removed or blocked the page after its export, so the destination must
  cancel the earlier import of the page. The SEPT state is `EXPORTED_REMOVED`, `EXPORTED_BLOCKED`,
  or `PENDING_EXPORTED_BLOCKED`.

The seamcall also updates the [migration counters](tdx-migration-structs.md#migration-counters).
`MIGRATE` increments `MIG_COUNT`, and `CANCEL` decrements it. `REMIGRATE` and `CANCEL` decrement
`DIRTY_COUNT`, because they process a page that the counter included.

`DSCAN` changes SEPT entries, because it clears the Dirty bit. To make the change visible to the
vCPUs, the VMM must run the [TLB tracking](tdx-misc.md#tlb-tracking) protocol after the scan.

`TDH.EXPORT.MEM` verifies that the VMM ran the protocol after the scan, using the
[scan epoch](tdx-misc.md#scan-epoch) that `DSCAN` or `PRECHECK` recorded. The check applies only
while the TD is running, not when it is paused.

The seamcall can also skip a page, for example because the page was written after `DSCAN` scanned
it, or because a concurrent operation holds its SEPT entry. The `STATUS` field of the GPA list entry
tells the reason, see below.

### Input Arguments

The VMM provides the memory for the memory bundle, and the TDX module writes the bundle there. Here
are the notable input arguments.

- `GPA_LIST_INFO` (`RCX`): [Descriptor](tdx-migration-structs.md#gpa_list_info) of the input GPA
  list.
- MBMD buffer (`R8`): HPA and size of the buffer for the [MBMD](tdx-migration-structs.md#mbmd).
- HPA list (`R9`): HPA of the metadata page of the HPA list that points to the pages for the
  encrypted memory. Entry N of the HPA list belongs to entry N of the GPA list.
- `MIGS_INDEX` (`R10`): [Migration stream](tdx-migration-structs.md#migration-stream) index.
- `RESUME` (`R10`): 1 when resuming after an interrupted exit code.
- MAC page 0 (`R11`): HPA of the [MAC page](tdx-migration-structs.md#mac-pages) for GPA list entries
  0 to 255.
- MAC page 1 (`R12`): HPA of the MAC page for GPA list entries 256 to 511.
- Attributes page (`R14`): HPA of the attributes page, used only for partitioned TDs.

Here are the details of the arguments:

- `GPA_LIST_INFO` (`RCX`): `FIRST_ENTRY` must be 0 on a new invocation. On a resumed invocation, it
  must be the index returned by the previous invocation.
- MBMD buffer (`R8`): The HPA is in bits 51:0 and the buffer size is in bits 63:52.
- `MIGS_INDEX` and `RESUME` (`R10`): `MIGS_INDEX` is in bits 15:0 and `RESUME` is bit 63. The other
  bits are reserved and must be 0.
- MAC page 0 (`R11`): Ignored if `FIRST_ENTRY` is 256 or more.
- MAC page 1 (`R12`): Ignored if `LAST_ENTRY` is less than 256.
- Attributes page (`R14`): Used only if the `FORMAT` field of `GPA_LIST_INFO` is `GPA_AND_L2_ATTR`.
  Otherwise, it is ignored. Partitioned TDs are outside the scope of this document, so the document
  does not cover this argument further.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.
- `GPA_LIST_INFO` (`RCX`): Same as the input, but `FIRST_ENTRY` is the index of the next entry to
  process. After the last entry, it is (`LAST_ENTRY` + 1) modulo 512.
- `NUM_EXPORTED` (`RDX`): On success, the number of 4 KiB pages in the bundle.
- Error count (`R8`): Number of GPA list entries that failed in this invocation. After a resumption,
  it counts only the entries processed since the resumption. The seamcall returns it only when the
  VMM invokes version 1, enumerated by the `LIST_ERROR_COUNT` bit of
  [TDX_FEATURES0](tdx-structs.md#tdx_features0).

`NUM_EXPORTED` counts the 4 KiB pages of the [memory bundle](tdx-migration-structs.md#memory-bundle)
that the VMM must send to the destination, except the MBMD. It counts:

- The GPA list page.
- The MAC pages.
- The attributes page (partitioned TDs only).
- The encrypted memory pages where data was actually exported.

It does not count the HPA list metadata page, because it is not part of the bundle. It also does not
count the pages of HPA list entries with the `INVALID` flag (bit 63) set, because the TDX module
exported no data to them.

**Note**: `INVALID` is a flag of an HPA list entry, not of a GPA list entry. The seamcall sets it
when it exports no page for an entry, for example for a pending page or a cancellation. Even though
no data page is used, the GPA list entry still carries the page metadata (`OPERATION` and `STATUS`).
It is part of the bundle and the destination imports it.

### GPA List Entry Fields

Each GPA list entry has input fields, which the seamcall reads, and output fields, which it writes.
Here is the input and output field description for the
[non-blocking export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export) case
(write-blocking export uses different input `OPERATION` values).

#### Input Fields

The seamcall reads the following fields:

- `GPA` (bits 51:12): [GFN](x86-misc.md#page-numbers-and-page-frames) of the page to export. The
  seamcall does not change it.
- `LEVEL` (bits 1:0): Must be 0 (4 KiB). The seamcall exports only 4 KiB pages, so the VMM must
  split large pages before the export.
- `MIG_TYPE` (bits 11:10): Must be 0 (`PAGE_4K`), the only supported page type.
- `OPERATION` (bits 53:52): `EXPORT` (1) to request the export of the page. `NOP` (0, 2, or 3) to
  skip the entry.
- `PENDING` (bit 2) and `STATE` (bits 4:3): Ignored on input.

The seamcall processes only the entries between `FIRST_ENTRY` and `LAST_ENTRY` of `GPA_LIST_INFO`.

#### Output Fields

The seamcall writes the following fields back:

- `PENDING` (bit 2): 1 if the page is still pending, which means the TD has not accepted it yet. The
  seamcall exports only the metadata of such a page and sets `INVALID` (bit 63) in its HPA list
  entry.
- `OPERATION` (bits 53:52): The operation that the TDX module performed. The module chooses it based
  on the SEPT entry state of the page, so it can differ from the requested `EXPORT`. The values are:
  - `NOP`: The seamcall did not export the page. `STATUS` tells the reason.
  - `MIGRATE`: Initial export of the page, or export after a cancellation.
  - `REMIGRATE`: Re-export of a page with updated content or attributes.
  - `CANCEL`: The destination must cancel the earlier import of the page, see below.
- `STATUS` (bits 60:56): The result of processing the entry, such as success, skipped, or error.
  More details below.
- `STATE` (bits 4:3): Always 0, which has no meaning here. `TDH.IMPORT.MEM` expects 0 on input.

#### Cancellation

The seamcall sets `OPERATION` to `CANCEL` for a page that the VMM removed after its export. It does
the same for a page that the VMM blocked after its export (the `EXPORTED_BLOCKED` and
`PENDING_EXPORTED_BLOCKED` SEPT states). On the destination host, the page is either removed and
returned to the VMM or is turned into a `PENDING` page, depending on the
[NO_REOWN](tdx-migration-misc.md#no_reown) flag. See [TDH.IMPORT.MEM](#tdhimportmem) for details.

**Note 1**: A page that the source VMM blocked is not necessarily removed later, but the TDX module
still cancels it on the destination. The specs do not say why. For example, if the VMM unblocks the
page with `TDH.MEM.RANGE.UNBLOCK` after the cancel was exported, the page stays on the source, but
the destination will remove it, so the page is exported again in a later round.

**Note 2**: The seamcall never sets `OPERATION` to `CANCEL` in the
[out-of-order phase](tdx-migration-misc.md#migration-phases), which corresponds to the `POST_EXPORT`
[TD operation state](tdx-misc.md#td-operation-states). It uses `CANCEL` only in the `LIVE_EXPORT`
and `PAUSED_EXPORT` states.

#### Entry Status Values

The notable `STATUS` values of a GPA list entry are:

- `SUCCESS` (0): The entry was processed successfully.
- `SKIPPED` (1): The input `OPERATION` was `NOP`, so the seamcall skipped the entry.
- `SEPT_WALK_FAILED` (2): The SEPT walk failed for the GPA, for example because a non-leaf SEPT
  entry at an upper level is in the [NL_BLOCKED](tdx-misc.md#non-leaf-entry-states) state.
- `SEPT_ENTRY_BUSY_HOST_PRIORITY` (3): A concurrent operation held the SEPT entry of the page. The
  seamcall skips the entry and the VMM should retry exporting the page. See
  [Host Priority](tdx-misc.md#host-priority).
- `SEPT_ENTRY_STATE_INCORRECT` (4): The SEPT entry state of the page does not allow export. For
  example, the page is `BLOCKED` or `PENDING_BLOCKED`, it is not a TD private page (such as `FREE`),
  or it is an [MMIO page](tdx-misc.md#sept-mmio-states). Blocking is usually temporary, so the VMM
  can try to export a blocked page later.
- `TLB_TRACKING_NOT_DONE` (5): [TLB tracking](tdx-misc.md#tlb-tracking) was not done for the GPA.
- `PAGE_DIRTY` (17): The page was written after `DSCAN` scanned it and before the seamcall processed
  it, so the Dirty bit of its SEPT entry is set. The seamcall skips the entry and sets `OPERATION`
  to `NOP`. The next `DSCAN` reports the page again and clears its Dirty bit, so the VMM can export
  it in a later round.
- `SCAN_EPOCH_NOT_RECORDED` (20): The [scan epoch](tdx-misc.md#scan-epoch) is not recorded for the
  page, so the seamcall cannot verify TLB tracking. Either no scan recorded it, or a memory
  management operation overwrote it after the scan, for example the VMM blocked and then unblocked
  the page. The VMM must scan the page again.

A `STATUS` other than `SUCCESS` does not stop the seamcall. The TDX module skips the page, marks the
entry, continues with the next one, and can still return `TDX_SUCCESS`. The VMM must check the
`STATUS` field of each entry.

**Note**: In case of non-blocking export, [TDH.EXPORT.PAUSE](#tdhexportpause) fails with
`TDX_BLOCKED_MEMORY_EXIST` if the source TD has blocked pages. All of them must leave the blocked
state before the pause. In practice this means that if the VMM is running a sequence like
[Private to Shared](tdx-misc.md#private-to-shared), it must finish before invoking
`TDH.EXPORT.PAUSE`.

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The seamcall processed all entries of the GPA list.
- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): The VMM resumes by executing the
  seamcall again with `RESUME` set to 1 and the `FIRST_ENTRY` value returned in `RCX`, and keeps the
  other inputs unchanged. The TDX module saves the progress in the migration stream context, so the
  VMM must resume on the same stream. The VMM must not send the memory bundle to the destination
  until the seamcall completes with `TDX_SUCCESS`, otherwise the import fails. If the VMM cannot
  resume, it can abort the [export session](tdx-migration-misc.md#export-session) with
  [TDH.EXPORT.ABORT](#tdhexportabort).
- [TDX_INTERRUPTED_BUSY](tdx-misc.md#tdx_operand_busy-vs-tdx_interrupted_busy): The TDX module
  yielded because a concurrent memory management operation, for example `TDH.MEM.PAGE.PROMOTE`,
  could not lock the SEPT. The VMM resumes the same way.
- `TDX_INVALID_RESUMPTION`: `RESUME` is set while no resumption is expected, or clear while a
  resumption is expected.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The source TD must be in `LIVE_EXPORT`, `PAUSED_EXPORT`, or `POST_EXPORT`.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, a reserved bit is set,
  `MIGS_INDEX` is out of range, or the MBMD buffer is too small.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example the migration
  stream.

---

## TDH.IMPORT.MEM

The `TDH.IMPORT.MEM` seamcall is the counterpart of `TDH.EXPORT.MEM` on the destination platform. It
takes a [memory bundle](tdx-migration-structs.md#memory-bundle) produced by `TDH.EXPORT.MEM` and
imports up to 512 pages from it into the destination TD, or cancels their earlier import.

The [GPA list](tdx-migration-structs.md#gpa-list) of the bundle carries the page metadata, and the
[HPA list](tdx-migration-structs.md#hpa-list) points to the encrypted page data. Some pages have no
data, for example pending pages. The MAC pages hold one MAC for each page. A MAC covers the page
data together with its GPA list entry. If the page has no data, it covers only the entry.

The HPA list metadata page is not part of the bundle, because its entries are HPAs on the source
platform. The VMM builds a new one on the destination and writes the HPAs of the received encrypted
data pages to its entries. Entry N must belong to entry N of the GPA list, so the VMM fills the
entries in the order of the GPA list.

Some GPA list entries do not have a data page, and the corresponding HPA list entries should have
the `INVALID` bit (63) set for them, for example when `OPERATION` is `CANCEL` or `STATUS` is not
`SUCCESS`.

### Input Arguments

The VMM provides the memory bundle received from the source, and the TDX module reads it. Here are
the notable input arguments.

- `GPA_LIST_INFO` (`RCX`): [Descriptor](tdx-migration-structs.md#gpa_list_info) of the GPA list.
- `NO_REOWN` (`RDX`, bit 0): [NO_REOWN](tdx-migration-misc.md#no_reown) flag.
- MBMD buffer (`R8`): HPA and size of the buffer with the [MBMD](tdx-migration-structs.md#mbmd).
- HPA list (`R9`): HPA of the metadata page of the HPA list that points to the pages with the
  encrypted memory. Entry N of the HPA list belongs to entry N of the GPA list.
- `MIGS_INDEX` (`R10`): [Migration stream](tdx-migration-structs.md#migration-stream) index.
- `RESUME` (`R10`): 1 when resuming after an interrupted exit code.
- MAC page 0 (`R11`): HPA of the [MAC page](tdx-migration-structs.md#mac-pages) for GPA list entries
  0 to 255.
- MAC page 1 (`R12`): HPA of the MAC page for GPA list entries 256 to 511.
- Page list (`R13`): HPA of an optional list of pages that the TDX module can convert to TD private
  pages, or `NULL_PA` (-1).
- Attributes page (`R14`): HPA of the attributes page, used only for partitioned TDs.

Here are the details of the arguments:

- `GPA_LIST_INFO` (`RCX`): `FIRST_ENTRY` must be 0 on a new invocation. On a resumed invocation, it
  must be the index returned by the previous invocation. `LAST_ENTRY` selects the last entry to
  process. A value below the end of the GPA list requests
  [partial GPA list processing](tdx-migration-misc.md#partial-gpa-list-processing).
- `NO_REOWN` (`RDX`): Forbids changing the ownership of a physical page between the VMM and the TD.
- MBMD buffer (`R8`): The HPA is in bits 51:0 and the buffer size is in bits 63:52.
- `MIGS_INDEX` and `RESUME` (`R10`): `MIGS_INDEX` is in bits 15:0 and `RESUME` is bit 63. The other
  bits are reserved and must be 0.
- MAC page 0 (`R11`): Ignored if `FIRST_ENTRY` is 256 or more.
- MAC page 1 (`R12`): Ignored if `LAST_ENTRY` is less than 256.
- Page list (`R13`): Ignored if `NO_REOWN` is set. See below for how the seamcall uses it.
- Attributes page (`R14`): Used only if the `FORMAT` field of `GPA_LIST_INFO` is `GPA_AND_L2_ATTR`.
  Otherwise, it is ignored. Partitioned TDs are outside the scope of this document, so this argument
  is not covered.

Both `NO_REOWN` and partial GPA list processing are
[enhanced import](tdx-migration-misc.md#enhanced-import) features. The VMM opts in on each
invocation, but only if the TDX module supports them, as enumerated by the `ENHANCED_IMPORT` bit
(60) of [TDX_FEATURES0](tdx-structs.md#tdx_features0).

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.
- `GPA_LIST_INFO` (`RCX`): Same as the input, but `FIRST_ENTRY` is the index of the next entry to
  process. After the last entry, it is (`LAST_ENTRY` + 1) modulo 512.
- Error count (`R8`): Number of GPA list entries that failed in this invocation. After a resumption,
  it counts only the entries processed since the resumption. The seamcall returns it only when the
  VMM invokes version 1, enumerated by the `LIST_ERROR_COUNT` bit of `TDX_FEATURES0`.

### Page Sources

Every imported page needs a TD private page on the destination. The seamcall takes it from one of
the following sources, in this order of priority:

1. **Existing page**: The SEPT entry already has a TD private page, in which case the import uses
   it. For example, this happens when the VMM pre-allocated the page with `TDH.MEM.PAGE.AUG`, or
   when the page is a re-import of a previously imported page.
2. **Page list**: The VMM passes the page list in `R13`. It has the same layout as the HPA list, and
   entry N belongs to entry N of the GPA list. An entry holds the HPA of a VMM owned page, and the
   seamcall converts the page to a TD private page. An entry with the `INVALID` bit (63) set holds
   no page, and its other bits can have any value. This way the VMM chooses which GPA list entries
   get a page.
3. **Migration buffer**: As the last resort, the seamcall reuses the page that the input HPA list
   entry points to. This page holds the encrypted data from the source. The seamcall converts the
   page to a TD private page, then decrypts the data in place.

With [NO_REOWN](tdx-migration-misc.md#no_reown), only source 1 (existing page) works. Sources 2 and
3 do not, because both change the ownership of a page between the VMM and the TD. The seamcall
ignores the page list. An entry that needs an ownership change gets the `REOWN_DISALLOWED` status,
and the seamcall returns `TDX_REOWN_DISALLOWED_FATAL`, which fails the import. The VMM must make
sure that a TD private page exists, either pre-allocated with `TDH.MEM.PAGE.AUG` or imported
earlier.

A pending page has no data page in the bundle, because it has no content to import. But it still
requires a TD private page. If the destination TD does not have one yet, it can come from source 2,
and even from source 3. The VMM recognizes pending pages by the `PENDING` bit in the GPA list entry.
When it builds the HPA list on the destination, it can put the HPA of a spare page in the entry of a
pending page, and the seamcall uses that page as the TD private page if necessary. However, the
seamcall does not report whether it used the spare page from the HPA list.

Newer TDX module versions, which report `TDX_FEATURES0.IMPORT_PAGE_STATUS=1`, provide feedback on
whether the seamcall actually used the pages. If the seamcall did not use a page provided in the
page list, it sets `INVALID` in the page list entry.

### Cancellation

A GPA list entry with `OPERATION` set to `CANCEL` tells the destination to undo the earlier import
of a page, because the source VMM removed or blocked the page after its export. The seamcall handles
it differently depending on the [NO_REOWN](tdx-migration-misc.md#no_reown) flag:

- Without `NO_REOWN`, the seamcall removes the TD private page and returns it to the VMM. The SEPT
  entry moves to the `REMOVED` state.
- With `NO_REOWN`, the seamcall does not remove the page. It converts it to a pre-allocated
  `PENDING` page, which stays TD owned.

If the VMM already removed the imported page, for example with `TDH.MEM.PAGE.REMOVE`, the SEPT entry
is `REMOVED` and there is no page to remove. This case is the same with and without `NO_REOWN`. The
seamcall accepts the cancellation and only updates the
[migration epoch](tdx-migration-misc.md#migration-epochs) and the `IMPORTED` flag of the page
(details below).

When the seamcall removes the TD private page, the page becomes a regular VMM owned page again. If
the TDX module does not report the `TDX_FEATURES0.IMPORT_PAGE_STATUS=1` capability, the VMM receives
no information about which page was removed.

When the capability is present, the TDX module can report the removed page in the page list. This
does not apply with `NO_REOWN`, because the seamcall removes no page and ignores the page list (it
is provided via `R13` on seamcall input). The VMM can set the `INVALID` bit in the page list entries
that correspond to the `CANCEL` GPA list entries. In this case, the TDX module writes the following
information to the page list entry of a removed page:

- Bits 51:0: HPA of the removed page.
- `REMOVED` (bit 61): 1.
- `INVALID` (bit 63): 0.
- `PAMT_REMOVAL_HINT` (bit 62): With dynamic PAMT, hints that the PAMT pages that map the removed
  page are empty and can be removed.

Therefore, the VMM can determine which pages were removed by checking the page list entries with the
`REMOVED` bit set.

### GPA List Entry Fields

Each GPA list entry has input fields, which the seamcall reads, and output fields, which it writes.
The seamcall processes only the entries between `FIRST_ENTRY` and `LAST_ENTRY` of `GPA_LIST_INFO`.

#### Input Fields

The seamcall reads the following fields:

- `GPA` (bits 51:12): [GFN](x86-misc.md#page-numbers-and-page-frames) of the page to import.
- `LEVEL` (bits 1:0): Must be 0 (4 KiB).
- `PENDING` (bit 2): 1 if the page is pending, which means the TD has not accepted it on the source.
- `STATE` (bits 4:3): Must be 0.
- `MIG_TYPE` (bits 11:10): Must be 0 (`PAGE_4K`), the only supported page size.
- `OPERATION` (bits 53:52): The operation requested for the page:
  - `NOP` (0): Skip the entry.
  - `MIGRATE` (1): Import the page for the first time in this
    [import session](tdx-migration-misc.md#import-session), or after a cancel.
  - `CANCEL` (2): Cancel the earlier import of the page.
  - `REMIGRATE` (3): Import again the page with updated content or attributes.

The `PENDING` bit and `OPERATION` together determine the SEPT entry state after the import:

- `MIGRATE` or `REMIGRATE` with `PENDING` set: The seamcall imports only the metadata of the page
  and sets the SEPT entry to `PENDING`.
- `MIGRATE` or `REMIGRATE` with `PENDING` clear: The seamcall imports the content and sets the SEPT
  entry to [MAPPED](tdx-misc.md#leaf-entry-states).
- `CANCEL`: The seamcall ignores the `PENDING` bit. The SEPT entry becomes `REMOVED` or `PENDING`,
  depending on `NO_REOWN`.

The destination TDX module uses the `IMPORTED` flag to make sure that `MIGRATE` imports only a page
that is not imported (never imported, or cancelled), and `REMIGRATE` replaces only an imported page.
The flag is stored next to the migration epoch, in the PAMT entry of the page or in the SEPT entry
of a `REMOVED` page.

- `MIGRATE` and `REMIGRATE` set the flag.
- `CANCEL` clears it.
- `MIGRATE` is rejected if the flag is set.
- `REMIGRATE` is rejected if the flag is clear.

The `IMPORTED` flag does not guarantee that `REMIGRATE` succeeds after `MIGRATE`. The per page
[migration epoch](tdx-migration-misc.md#migration-epochs) forbids importing or cancelling a page
twice in the same epoch. If the migration epoch of the incoming bundle is not greater than the
migration epoch of the page, the entry gets the `MIGRATED_IN_CURRENT_EPOCH` status. This applies to
`REMIGRATE`, `CANCEL`, and `MIGRATE` after a cancellation.

**Note**: In the [out-of-order phase](tdx-migration-misc.md#migration-phases), which starts with the
`POST_IMPORT` state, the seamcall cannot import a page over an older version of the same page. It
rejects `REMIGRATE` and `CANCEL` there.

#### Output Fields

The seamcall writes the following fields back:

- `OPERATION` (bits 53:52): The operation that the TDX module performed. The values are:
  - `NOP`: The seamcall did not import the page. `STATUS` tells the reason.
  - `MIGRATE` or `REMIGRATE`: The page was imported.
  - `CANCEL`: The earlier import of the page was cancelled.
- `STATUS` (bits 60:56): The result of processing the entry, see below.

#### Entry Status Values

The `STATUS` values fall into three groups:

- Values that never fail the import.
- Values that fail the import only in the [in-order phase](tdx-migration-misc.md#migration-phases).
  In the out-of-order phase, the seamcall skips the entry and continues with the next one.
- Values that always fail the import.

A failed import is a fatal failure: the [import session](tdx-migration-misc.md#import-session)
cannot proceed anymore and migration has to be aborted.

These values never fail the import:

- `SUCCESS` (0): The entry was processed successfully.
- `SKIPPED` (1): The input `OPERATION` was `NOP`, so the seamcall skipped the entry.

These values fail the import only in the in-order phase:

- `SEPT_WALK_FAILED` (2): The SEPT has no entry for the GPA. The VMM must add the missing SEPT pages
  before the import, for example with `TDH.MEM.SEPT.ADD`.
- `SEPT_ENTRY_BUSY_HOST_PRIORITY` (3): A concurrent operation held the SEPT entry of the page. See
  [Host Priority](tdx-misc.md#host-priority).
- `SEPT_ENTRY_STATE_INCORRECT` (4): The SEPT entry state of the page does not allow the operation.
- `NEW_PAGE_NOT_AVAILABLE` (9): The operation needs a page from the page list, but the entry has
  `INVALID` set or its HPA is unusable. In the out-of-order phase, the seamcall skips the entry only
  if the page list entry holds an HPA. Otherwise, it always fails the import.

These values always fail the import:

- `OP_STATE_INCORRECT` (6): The TD operation state does not allow the operation, for example
  `CANCEL` in the out-of-order phase.
- `MIGRATED_IN_CURRENT_EPOCH` (7): The page was already imported in the current migration epoch.
- `MIG_BUFFER_NOT_AVAILABLE` (8): The operation needs a migration buffer, but the HPA list entry
  provides none.
- `INVALID_PAGE_MAC` (10): The MAC of the page is incorrect.
- `DISALLOWED_IMPORT_OVER_REMOVED` (11): The page is in the `REMOVED` SEPT state, which does not
  allow the import.
- `GPA_LIST_ENTRY_INVALID` (15): A field of the GPA list entry is invalid.
- `REOWN_DISALLOWED` (18): The operation needs a change of page ownership, but `NO_REOWN` is set.

### Exit Codes

Several errors have both a `_FATAL` variant and a variant without the suffix. The variants differ as
follows:

- With the suffix: Applies to the [in-order phase](tdx-migration-misc.md#migration-phases). The
  seamcall returns the code and fails the import.
- Without the suffix: Applies to errors in the seamcall operands or the TD state, and to the
  out-of-order phase, where the TDX module does not return the code. It skips the entry, reports the
  reason in the `STATUS` field, and continues with the next entry.

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The seamcall processed all entries of the GPA list. Some entries may have errors
  that do not fail the import. The error count in `R8` and the `STATUS` fields tell which.
- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): The VMM resumes by executing the
  seamcall again with `RESUME` set to 1 and the `FIRST_ENTRY` value returned in `RCX`, and keeps the
  other inputs unchanged. The TDX module saves the progress in the migration stream context, so the
  VMM must resume on the same stream. If the VMM cannot resume, it can abort the
  [import session](tdx-migration-misc.md#import-session) with [TDH.IMPORT.ABORT](#tdhimportabort).
- `TDX_INTERRUPTED_LIST_END`: `LAST_ENTRY` is below the end of the GPA list, as requested by the
  VMM. Resumption works the same way, optionally with a higher `LAST_ENTRY`.
- `TDX_INTERRUPTED_PAMT`: With dynamic PAMT, a PAMT page pair is missing. The VMM adds it with
  `TDH.PHYMEM.PAMT.ADD` and resumes with the `FIRST_ENTRY` value returned in `RCX`.
- `TDX_INVALID_RESUMPTION`: `RESUME` is set while no resumption is expected, or clear while a
  resumption is expected.
- `TDX_OPERAND_INVALID` and `TDX_OPERAND_INVALID_FATAL`: An input operand or a list entry is
  invalid. For example, a reserved bit is set, `MIGS_INDEX` is out of range, or an entry has no
  migration buffer or page that it needs.

These report problems with the bundle:

- `TDX_INCORRECT_MBMD_MAC`: The MAC of the MBMD is incorrect.
- `TDX_INVALID_MBMD`: The MBMD is invalid. For example, the bundle carries a migration epoch other
  than the current migration epoch of the destination TD.
- `TDX_INVALID_PAGE_MAC_FATAL`: The MAC of a page is incorrect.
- `TDX_MIGRATED_IN_CURRENT_EPOCH_FATAL`: A page was already imported in the current migration epoch.

These report problems with the destination state:

- `TDX_EPT_WALK_FAILED` and `TDX_EPT_WALK_FAILED_FATAL`: The SEPT walk failed for a GPA.
- `TDX_EPT_ENTRY_STATE_INCORRECT` and `TDX_EPT_ENTRY_STATE_INCORRECT_FATAL`: The SEPT entry state of
  a page does not allow the operation.
- `TDX_MIGRATION_STREAM_STATE_INCORRECT`: The state of the migration stream does not allow this
  seamcall.
- `TDX_OP_STATE_INCORRECT` and `TDX_OP_STATE_INCORRECT_FATAL`: The
  [TD operation state](tdx-misc.md#td-operation-states) does not allow this seamcall or the
  requested operation. The destination TD must be in `MEMORY_IMPORT`, `STATE_IMPORT`, `POST_IMPORT`,
  or `LIVE_IMPORT`. For example, `CANCEL` is not allowed in the out-of-order phase.
- `TDX_REOWN_DISALLOWED_FATAL`: The seamcall refused a change of page ownership or the import over a
  removed page.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy) and `TDX_OPERAND_BUSY_FATAL`: A resource is busy,
  for example the migration stream or a SEPT entry.

### Consequences of Import Failures

Some `STATUS` values and all `*_FATAL` exit codes fail the import. The effect of a failure depends
on the [migration phase](tdx-migration-misc.md#migration-phases) of the destination TD.

In the in-order phase, a failure moves the destination TD to `FAILED_IMPORT`. The TDX module does
not allow such a TD to run, so the VMM can only tear it down. See
[TD operation states](tdx-misc.md#td-operation-states).

In the out-of-order phase, the outcome depends on the destination TD state. In `POST_IMPORT`, the TD
does not run yet and a failure moves it to `FAILED_IMPORT`, the same as in the in-order phase.

In `LIVE_IMPORT`, which [TDH.IMPORT.COMMIT](#tdhimportcommit) enters in
[post-copy migration](tdx-migration-misc.md#pre-copy-vs-post-copy-migration), the TD is already
running. A failure ends the [import session](tdx-migration-misc.md#import-session) and moves the TD
to `RUNNABLE`. The TD keeps running, but the pages that were not imported yet stay missing, and the
VMM can no longer import them.

---

## TDH.EXPORT.TRACK

The `TDH.EXPORT.TRACK` seamcall generates one of the following tokens:

- **Epoch token** (`IN_ORDER_DONE` is 0): Ends the current migration epoch and starts a new one.
  Memory bundles exported after that carry the new epoch. The TD may continue running.
- **Start token** (`IN_ORDER_DONE` is 1): Ends the last migration epoch and the
  [in-order phase](tdx-migration-misc.md#migration-phases) of the export, and starts the
  out-of-order phase.

The VMM delivers the token to the destination platform, which imports it with the
[TDH.IMPORT.TRACK](#tdhimporttrack) seamcall. The VMM uses epoch tokens as barriers between
migration epochs. When the destination imports a token, it verifies that it imported all memory
bundles that the source exported before the token. This prevents rollback attacks. See
[Migration Epochs](tdx-migration-misc.md#migration-epochs) for the details.

The start token is the last token of the in-order phase. The VMM generates it after pausing the
source TD with [TDH.EXPORT.PAUSE](#tdhexportpause) and exporting mutable TD state with
[TDH.EXPORT.STATE.TD](#tdhexportstatetd) and the state of every vCPU with
[TDH.EXPORT.STATE.VP](#tdhexportstatevp). Generating and importing the token have these effects:

- Once it is generated, the source TD cannot run anymore (unless the migration is aborted). Its
  [operation state](tdx-misc.md#td-operation-states) changes from `PAUSED_EXPORT` to `POST_EXPORT`.
- Once the destination imports the start token, the destination enters `POST_IMPORT` and cannot run
  yet. The VMM can call [TDH.IMPORT.COMMIT](#tdhimportcommit) to move it to `LIVE_IMPORT` and let it
  run. In [pre-copy migration](tdx-migration-misc.md#pre-copy-vs-post-copy-migration), which this
  document describes, the VMM calls [TDH.IMPORT.END](#tdhimportend) instead, which moves the
  destination directly from `POST_IMPORT` to `RUNNABLE`.

### Input Arguments

The token is an [MBMD-only bundle](tdx-migration-structs.md#mbmd-only-bundles), so the VMM provides
the MBMD buffer and the seamcall writes the token there.

- MBMD buffer (`R8`): HPA in bits 51:0 and size in bits 63:52. The HPA must be aligned to 128 bytes.
  The size must be at least 128 bytes, the maximum MBMD size.
- `MIGS_INDEX` (`R10`, bits 15:0): Index of the
  [migration stream](tdx-migration-structs.md#migration-stream), must be 0.
- `IN_ORDER_DONE` (`R10`, bit 63): 1 generates the start token, 0 generates an epoch token.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.

On success, the seamcall writes the token to the MBMD buffer. The
[MBMD](tdx-migration-structs.md#mbmd) of the token has these fields:

- `MB_TYPE`: The epoch token type (32), used for the epoch token and the start token.
- `MIG_EPOCH`: The new migration epoch, or `0xFFFFFFFF` for the start token.
- `TOTAL_MB`: Total number of migration bundles that the source exported in this
  [export session](tdx-migration-misc.md#export-session), including the token.

### Start Token Checks

In the current TDX module implementation, with non-blocking export, the VMM must export all TD
private memory before it requests the start token. The seamcall checks the following:

- A comprehensive scan ([TDH.MEM.SCAN.COMP](#tdhmemscancomp) with the `DCHECK` operation and the
  `REEXPORT` qualifier bit clear) completed successfully and scanned the whole GPA space.
- No exported page remains marked as needing re-export.
- All TD private memory was exported.

The TDX module checks the last two conditions using these [TDCS](tdx-structs.md#tdcs) counters,
described in [Migration Counters](tdx-migration-structs.md#migration-counters):

- `DIRTY_COUNT` counts exported pages that need another [TDH.EXPORT.MEM](#tdhexportmem) operation.
- `MEM_COUNT` counts the TD private pages.
- `MIG_COUNT` counts the exported pages.

The checks pass when `DIRTY_COUNT` is 0 and `MIG_COUNT` equals `MEM_COUNT`.

**Note**: The check that all TD private memory was exported before the start token is requested
effectively makes [post-copy migration](tdx-migration-misc.md#pre-copy-vs-post-copy-migration)
impossible with non-blocking export. Based on the TDX specs, this is a limitation of the current
implementation, not of the architecture. See
[Post-Copy Support](tdx-migration-misc.md#post-copy-support).

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The token was generated.
- `TDX_EXPORTED_DIRTY_PAGES_REMAIN`: The start token was requested, but some exported pages changed
  and need re-export. The VMM exports them again and retries.
- `TDX_MEM_SCAN_DCHECK_NOT_DONE`: The start token was requested, but no comprehensive scan completed
  successfully.
- `TDX_UNEXPORTED_MEMORY_REMAINS`: The start token was requested, but some TD private memory was not
  exported yet.
- `TDX_MIGRATION_EPOCH_OVERFLOW`: The migration epoch cannot be incremented because it reached the
  maximum value.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The source TD must be in `LIVE_EXPORT` or `PAUSED_EXPORT`, and in `PAUSED_EXPORT`
  for the start token.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `MIGS_INDEX` is not 0, a reserved
  bit is set, or the MBMD buffer is too small or misaligned.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example the migration
  stream.

---

## TDH.IMPORT.TRACK

The `TDH.IMPORT.TRACK` seamcall imports a token produced by [TDH.EXPORT.TRACK](#tdhexporttrack) on
the source platform. The token is an [MBMD-only bundle](tdx-migration-structs.md#mbmd-only-bundles)
that the VMM delivers to the destination host. There are two kinds of tokens:

- **Epoch token**: Ends the current migration epoch and starts a new one.
- **Start token**: Ends the last migration epoch and the
  [in-order phase](tdx-migration-misc.md#migration-phases) of the import, and starts the
  out-of-order phase.

Before importing a start token, the VMM must import the mutable TD state with
[TDH.IMPORT.STATE.TD](#tdhimportstatetd) and the state of every vCPU with
[TDH.IMPORT.STATE.VP](#tdhimportstatevp).

Both tokens let the destination verify that it imported all memory bundles that the source exported
before the token. This prevents rollback attacks. See
[Migration Epochs](tdx-migration-misc.md#migration-epochs) for the details.

When the destination imports the start token, it enters `POST_IMPORT` and cannot run yet. The VMM
can call [TDH.IMPORT.COMMIT](#tdhimportcommit) to move it to `LIVE_IMPORT` and let it run. In
[pre-copy migration](tdx-migration-misc.md#pre-copy-vs-post-copy-migration), which this document
describes, the VMM calls [TDH.IMPORT.END](#tdhimportend) instead, which moves the destination
directly from `POST_IMPORT` to `RUNNABLE`.

### Input Arguments

The VMM provides the token in a buffer for the [MBMD](tdx-migration-structs.md#mbmd), and the TDX
module reads the token from there.

- MBMD buffer (`R8`): HPA in bits 51:0 and size in bits 63:52. The HPA must be aligned to 128 bytes.
  The size must be at least 128 bytes, the maximum MBMD size.
- `MIGS_INDEX` (`R10`, bits 15:0): Index of the
  [migration stream](tdx-migration-structs.md#migration-stream), must be 0.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.

### Token Processing

The destination TD must be in the [in-order phase](tdx-migration-misc.md#migration-phases), which
means the `MEMORY_IMPORT` or `STATE_IMPORT` [operation state](tdx-misc.md#td-operation-states). A
start token can be imported only once because it ends this phase.

For both token kinds, the seamcall checks the following fields in the token MBMD:

- `MB_TYPE` identifies the token as an epoch token (32).
- `MIG_EPOCH` contains an epoch higher than the current migration epoch on the destination. The
  start token sets `MIG_EPOCH` to `0xFFFFFFFF`, so it satisfies this check for any valid migration
  epoch.
- `TOTAL_MB`, the number of bundles exported in the
  [export session](tdx-migration-misc.md#export-session) including this token, equals the number of
  bundles that the destination imported so far, plus one for the token itself.
- The MAC over the MBMD fields verifies with the migration keys.

For a start token, the seamcall also verifies that the mutable TD state and all vCPU states were
imported.

Importing an epoch token advances the destination migration epoch to `MIG_EPOCH`. The destination
then accepts only memory bundles that carry the new epoch.

Importing a start token moves the destination TD to `POST_IMPORT`, starting the out-of-order phase.
The TD cannot run in this state. This document describes pre-copy migration, where the VMM calls
`TDH.IMPORT.END` to end the [import session](tdx-migration-misc.md#import-session) and move the TD
directly to `RUNNABLE`. It does not cover post-copy migration, where the VMM calls
`TDH.IMPORT.COMMIT` to move the TD to `LIVE_IMPORT` so it can run while remaining pages are
delivered on demand.

### Exit Codes

Codes with the `_FATAL` suffix fail the import and move the destination TD to `FAILED_IMPORT`.

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The token was imported.
- `TDX_INCORRECT_MBMD_MAC_FATAL`: The MAC of the MBMD is incorrect.
- `TDX_INVALID_MBMD_FATAL`: The MBMD is invalid. For example, it is not an epoch token, or
  `MIG_EPOCH` or `TOTAL_MB` has an unexpected value.
- `TDX_SOME_VCPUS_NOT_MIGRATED_FATAL`: The start token arrived, but the state of some vCPUs was not
  imported.
- `TDX_MIGRATION_STREAM_STATE_INCORRECT`: The state of the migration stream does not allow this
  seamcall.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The destination TD must be in `MEMORY_IMPORT` or `STATE_IMPORT`.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `MIGS_INDEX` is not 0, or the
  MBMD buffer is too small or misaligned.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example the migration
  stream.

---

## TDH.EXPORT.PAUSE

The `TDH.EXPORT.PAUSE` seamcall pauses the source TD and starts the downtime period of the
migration. The TD cannot run after the pause, unless the VMM aborts the migration.

When the seamcall completes, the source TD moves from the `LIVE_EXPORT`
[operation state](tdx-misc.md#td-operation-states) to `PAUSED_EXPORT`. The pause prevents the TD
from running, which means that the TD state does not change.

The seamcall needs exclusive access to the TD, so the VMM must stop all vCPUs of the TD and make
sure that no other seamcall for the TD is running.

The seamcall also acts like `TDH.MEM.TRACK`. It increments the TD epoch, and since all vCPUs are
already stopped, no [TLB tracking](tdx-misc.md#tlb-tracking) is needed. If the VMM aborts the
migration, the first `TDH.VP.ENTER` of each vCPU flushes the TLB.

While the TD is paused, the VMM can use `TDH.MEM.PAGE.DEMOTE` and `TDH.MEM.PAGE.RELOCATE` without
page blocking and TLB tracking, which they normally require.

**Note**: The TDX module does not forbid scanning with [DSCAN](#tdhmemscanrange), pausing the TD,
and then exporting the reported pages. In my opinion, this is a bad idea, because it keeps the TD
paused longer than needed, but it is not forbidden. In this case, [TDH.EXPORT.MEM](#tdhexportmem)
does not verify that TLB tracking was done, because no vCPU runs.

The only input argument is the [TD handle](tdx-structs.md#tdr) in `RCX`. The only output is the exit
code in `RAX`.

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The TD was paused.
- `TDX_BLOCKED_MEMORY_EXIST`: The TD has private pages blocked by `TDH.MEM.RANGE.BLOCK`, which
  [non-blocking export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export) does not allow.
  The VMM must make these pages leave the blocked state and retry.
- `TDX_TD_HAS_ATTACHED_DEVICES`: Applies only if TDX Connect is enabled for the TD. Devices are
  still attached to the TD.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The source TD must be in `LIVE_EXPORT`, so the TD must not be paused already.
- `TDX_OPERAND_INVALID`: An input operand is invalid, for example a reserved bit is set.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example another seamcall
  for the TD is running.

---

## TDH.EXPORT.STATE.TD

The `TDH.EXPORT.STATE.TD` seamcall exports the
[TD mutable state](tdx-migration-structs.md#td-mutable-state) of the source TD. It produces a
[TD mutable state bundle](tdx-migration-structs.md#state-bundles). The VMM delivers the bundle to
the destination host and imports it there with the [TDH.IMPORT.STATE.TD](#tdhimportstatetd)
seamcall.

The source TD must be paused with [TDH.EXPORT.PAUSE](#tdhexportpause) before the export. Paused
vCPUs cannot run, so the state does not change while the seamcall runs. The seamcall does not change
the [operation state](tdx-misc.md#td-operation-states) of the TD, which stays in `PAUSED_EXPORT`.

### Input Arguments

The VMM provides the memory for the parts of the
[TD mutable state bundle](tdx-migration-structs.md#state-bundles): a buffer for the
[MBMD](tdx-migration-structs.md#mbmd) and pages for the encrypted state, listed in an
[HPA list](tdx-migration-structs.md#hpa-list). The TDX module writes the bundle there.

- MBMD buffer (`R8`): HPA and size of the buffer for the MBMD.
- `HPA_LIST_INFO` (`R9`): [Descriptor](tdx-migration-structs.md#hpa_list_info) of the HPA list for
  the encrypted state pages.
- `MIGS_INDEX` (`R10`): [Migration stream](tdx-migration-structs.md#migration-stream) index, must be
  0.
- `RESUME` (`R10`): 1 when resuming after an interrupted exit code.

Here are the details of the arguments:

- `R8` has the HPA of the MBMD buffer in bits 51:0 and the buffer size in bits 63:52.
- Bits 2:0 (`FORMAT`) and 11:3 (`FIRST_ENTRY`) of the `R9` register must be 0, also on resumption.
  The rest of `R9` is the HPA list address and the last entry index. The TDX module saves the
  progress in the migration stream context, so the VMM passes the same `R9` value on resumption.
- Bits 15:0 of the `R10` register are `MIGS_INDEX` and bit 63 is `RESUME`. The other bits are
  reserved and must be 0.

The required number of pages in the HPA list is enumerated by the `NUM_TD_STATE_PAGES` global
[metadata field](tdx-structs.md#metadata-fields), readable with `TDH.SYS.RD`.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.
- Page count (`RDX`): Number of 4 KiB state pages in the bundle. It does not count the MBMD buffer
  or the HPA list metadata page. The value is valid only on success.

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The TD mutable state bundle was exported.
- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): The VMM resumes by executing the
  seamcall again with `RESUME` set to 1. If the VMM cannot resume, it can abort the
  [export session](tdx-migration-misc.md#export-session) with [TDH.EXPORT.ABORT](#tdhexportabort).
- `TDX_INVALID_RESUMPTION`: `RESUME` is set while no resumption is expected, or clear while a
  resumption is expected.
- `TDX_INCOMPATIBLE_MBMD_MAC_CONTEXT`: The TDX module was updated while the seamcall was
  interrupted, and the update was configured to detect this incompatibility. The VMM must abort the
  export session and can restart it.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The source TD must be in `PAUSED_EXPORT`.
- `TDX_MIGRATION_STREAM_STATE_INCORRECT`: The state of migration stream 0 does not allow this
  seamcall.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `MIGS_INDEX` is not 0, a reserved
  bit is set, or the MBMD buffer is too small or misaligned.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example the migration
  stream.

---

## TDH.IMPORT.STATE.TD

The `TDH.IMPORT.STATE.TD` seamcall imports the
[TD mutable state bundle](tdx-migration-structs.md#state-bundles) produced by
[TDH.EXPORT.STATE.TD](#tdhexportstatetd).

The seamcall verifies the MAC of the bundle and checks that the imported state is compatible with
the destination platform and TDX module configuration. It also checks that the bundle contains all
the fields that the import requires.

The destination TD must be in the `MEMORY_IMPORT`
[operation state](tdx-misc.md#td-operation-states), which means that
[TDH.IMPORT.STATE.IMMUTABLE](#tdhimportstateimmutable) completed. When the seamcall completes, the
TD enters `STATE_IMPORT`.

### Input Arguments

The VMM provides the parts of the [TD mutable state bundle](tdx-migration-structs.md#state-bundles)
received from the source: a buffer with the [MBMD](tdx-migration-structs.md#mbmd) and pages with the
encrypted state, listed in an [HPA list](tdx-migration-structs.md#hpa-list). Here are the notable
input arguments.

- MBMD buffer (`R8`): HPA and size of the buffer with the MBMD.
- `HPA_LIST_INFO` (`R9`): [Descriptor](tdx-migration-structs.md#hpa_list_info) of the HPA list for
  the encrypted state pages.
- `MIGS_INDEX` (`R10`): [Migration stream](tdx-migration-structs.md#migration-stream) index, must be
  0.
- `RESUME` (`R10`): 1 when resuming after an interrupted exit code.

Here are the details of the arguments:

- `R8` has the HPA of the MBMD buffer in bits 51:0 and the buffer size in bits 63:52.
- Bits 2:0 (`FORMAT`) and 11:3 (`FIRST_ENTRY`) of the `R9` register must be 0, also on resumption.
  Bits 63:55 (`LAST_ENTRY`) must be the number of pages returned in `RDX` by `TDH.EXPORT.STATE.TD`
  minus 1. The rest of `R9` is the HPA list address. The TDX module saves the progress in the
  migration stream context, so the VMM passes the same `R9` value on resumption.
- Bits 15:0 of the `R10` register are `MIGS_INDEX` and bit 63 is `RESUME`. The other bits are
  reserved and must be 0.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.
- Error information (`RCX`): The seamcall checks every
  [metadata field](tdx-structs.md#metadata-fields) of the bundle against the destination TD and the
  TDX module configuration. If a check fails, `RCX` identifies the offending field or the field
  sequence header, for example a field whose value the destination does not support.

### Exit Codes

Codes with the `_FATAL` suffix fail the import and move the destination TD to `FAILED_IMPORT`.

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The TD mutable state bundle was imported.
- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): The VMM resumes by executing the
  seamcall again with `RESUME` set to 1. If the VMM cannot resume, it can abort the
  [import session](tdx-migration-misc.md#import-session) with [TDH.IMPORT.ABORT](#tdhimportabort).
- `TDX_INVALID_RESUMPTION`: `RESUME` is set while no resumption is expected, or clear while a
  resumption is expected.
- `TDX_INVALID_RESUMPTION_FATAL`: On resumption, `HPA_LIST_INFO` differs from the value used before
  the interruption, or the interrupted seamcall was another one.
- `TDX_INCOMPATIBLE_MBMD_MAC_CONTEXT_FATAL`: The TDX module was updated while the seamcall was
  interrupted, and the update was configured to detect this incompatibility. The VMM must abort the
  import session and can restart it.
- `TDX_INCORRECT_MBMD_MAC_FATAL`: The MAC of the bundle is incorrect. This also happens if the TDX
  module was updated while the seamcall was interrupted and the update was not configured to detect
  the incompatibility.
- `TDX_INVALID_MBMD_FATAL`: The MBMD is invalid.
- `TDX_INVALID_METADATA_LIST_HEADER_FATAL`: A metadata list header in the bundle is invalid.
- `TDX_METADATA_FIELD_NOT_WRITABLE_FATAL`: A metadata field in the bundle is not writable. `RCX`
  contains the ABI field ID.
- `TDX_METADATA_FIELD_VALUE_NOT_VALID_FATAL`: A metadata field in the bundle has an invalid value.
  `RCX` contains the ABI field ID.
- `TDX_REQUIRED_METADATA_FIELD_MISSING_FATAL`: A required metadata field is missing from the bundle.
  `RCX` contains the ABI field ID.
- `TDX_METADATA_FIELD_ID_INCORRECT_FATAL`: A field ID or a field sequence header in the bundle is
  incorrect. `RCX` contains the field ID or the field sequence header.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The destination TD must be in `MEMORY_IMPORT`.
- `TDX_MIGRATION_STREAM_STATE_INCORRECT`: The state of migration stream 0 does not allow this
  seamcall.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `MIGS_INDEX` is not 0, a reserved
  bit is set, or the MBMD buffer is too small or misaligned.
- `TDX_OPERAND_INVALID_FATAL`: The HPA list descriptor or the HPA of a state page is invalid.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example the migration
  stream.

---

## TDH.EXPORT.STATE.VP

The `TDH.EXPORT.STATE.VP` seamcall exports the
[vCPU mutable state](tdx-migration-structs.md#vcpu-mutable-state) of one vCPU of the source TD. It
produces a [vCPU state bundle](tdx-migration-structs.md#state-bundles). The VMM delivers the bundle
to the destination host and imports it there with the [TDH.IMPORT.STATE.VP](#tdhimportstatevp)
seamcall.

The VMM exports each vCPU with a separate invocation, after pausing the source TD with
[TDH.EXPORT.PAUSE](#tdhexportpause). The seamcall does not change the
[operation state](tdx-misc.md#td-operation-states) of the TD, which stays in `PAUSED_EXPORT`.

A paused vCPU stays associated with the CPU where it last ran, see
[vCPU Association](tdx-misc.md#vcpu-association). The seamcall requires that the vCPU is not
associated with any CPU or is associated with the current CPU. So the VMM has two options:

- Run the seamcall on that CPU.
- Run `TDH.VP.FLUSH` on that CPU first, then run the seamcall on any CPU.

If the VMM does neither, the seamcall fails.

### Input Arguments

The VMM provides the memory for the parts of the
[vCPU state bundle](tdx-migration-structs.md#state-bundles): a buffer for the
[MBMD](tdx-migration-structs.md#mbmd) and pages for the encrypted state, listed in an
[HPA list](tdx-migration-structs.md#hpa-list). The TDX module writes the bundle there.

- TDVPR page (`RCX`): HPA of the [TDVPR](tdx-structs.md#tdvps) page of the source vCPU.
- MBMD buffer (`R8`): HPA and size of the buffer for the MBMD.
- `HPA_LIST_INFO` (`R9`): [Descriptor](tdx-migration-structs.md#hpa_list_info) of the HPA list for
  the encrypted state pages.
- `MIGS_INDEX` (`R10`): [Migration stream](tdx-migration-structs.md#migration-stream) index.
- `RESUME` (`R10`): 1 when resuming after an interrupted exit code.

Here are the details of the arguments:

- `R8` has the HPA of the MBMD buffer in bits 51:0 and the buffer size in bits 63:52.
- Bits 2:0 (`FORMAT`) and 11:3 (`FIRST_ENTRY`) of the `R9` register must be 0, also on resumption.
  The rest of `R9` is the HPA list address and the last entry index. The TDX module saves the
  progress in the migration stream context, so the VMM passes the same `R9` value on resumption.
- Bits 15:0 of the `R10` register are `MIGS_INDEX` and bit 63 is `RESUME`. The other bits are
  reserved and must be 0.

Unlike [TDH.EXPORT.STATE.IMMUTABLE](#tdhexportstateimmutable) and
[TDH.EXPORT.STATE.TD](#tdhexportstatetd), which require stream 0, this seamcall can use any created
stream. The VMM can export several vCPUs in parallel, each on its own stream.

The required number of pages in the HPA list is enumerated by the `NUM_VP_STATE_PAGES` global
[metadata field](tdx-structs.md#metadata-fields), readable with `TDH.SYS.RD`.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.
- Page count (`RDX`): Number of 4 KiB state pages in the bundle. It does not count the MBMD buffer
  or the HPA list metadata page. The value is valid only on success.

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The vCPU state bundle was exported.
- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): The VMM resumes by executing the
  seamcall again for the same vCPU and stream, with `RESUME` set to 1. If the VMM cannot resume, it
  can abort the [export session](tdx-migration-misc.md#export-session) with
  [TDH.EXPORT.ABORT](#tdhexportabort).
- `TDX_INVALID_RESUMPTION`: `RESUME` is set while no resumption is expected, or clear while a
  resumption is expected.
- `TDX_INCOMPATIBLE_MBMD_MAC_CONTEXT`: The TDX module was updated while the seamcall was
  interrupted, and the update was configured to detect this incompatibility. The VMM must abort the
  export session and can restart it.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The source TD must be in `PAUSED_EXPORT`.
- `TDX_VCPU_STATE_INCORRECT`: The vCPU is not in the `VCPU_READY` state.
- `TDX_VCPU_ALREADY_EXPORTED`: The state of the vCPU was already exported in this export session.
- `TDX_MIGRATION_STREAM_STATE_INCORRECT`: The state of the migration stream does not allow this
  seamcall.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `MIGS_INDEX` is out of range, a
  reserved bit is set, or the MBMD buffer is too small or misaligned.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example the migration
  stream or the vCPU.

---

## TDH.IMPORT.STATE.VP

The `TDH.IMPORT.STATE.VP` seamcall imports the
[vCPU state bundle](tdx-migration-structs.md#state-bundles) produced by
[TDH.EXPORT.STATE.VP](#tdhexportstatevp) into one vCPU of the destination TD.

The seamcall verifies the MAC of the bundle and checks that the imported state is compatible with
the destination platform and TDX module configuration. It also checks that the bundle contains all
the fields that the import requires.

The destination TD must be in the `STATE_IMPORT` [operation state](tdx-misc.md#td-operation-states),
which means that [TDH.IMPORT.STATE.TD](#tdhimportstatetd) completed.

The VMM must create the destination vCPU with `TDH.VP.CREATE` and add its pages with `TDH.VP.ADDCX`
before the import.

The first invocation of the seamcall [associates](tdx-misc.md#vcpu-association) the new vCPU with
the current CPU. If the seamcall is interrupted, the VMM must resume it on the same CPU, otherwise
it fails with `TDX_VCPU_ASSOCIATED_FATAL`.

### Input Arguments

The VMM provides the parts of the [vCPU state bundle](tdx-migration-structs.md#state-bundles)
received from the source: a buffer with the [MBMD](tdx-migration-structs.md#mbmd) and pages with the
encrypted state, listed in an [HPA list](tdx-migration-structs.md#hpa-list). Here are the notable
input arguments.

- TDVPR page (`RCX`): HPA of the [TDVPR](tdx-structs.md#tdvps) page of the destination vCPU.
- MBMD buffer (`R8`): HPA and size of the buffer with the MBMD.
- `HPA_LIST_INFO` (`R9`): [Descriptor](tdx-migration-structs.md#hpa_list_info) of the HPA list for
  the encrypted state pages.
- `MIGS_INDEX` (`R10`): [Migration stream](tdx-migration-structs.md#migration-stream) index.
- `RESUME` (`R10`): 1 when resuming after an interrupted exit code.

Here are the details of the arguments:

- `R8` has the HPA of the MBMD buffer in bits 51:0 and the buffer size in bits 63:52.
- Bits 2:0 (`FORMAT`) and 11:3 (`FIRST_ENTRY`) of the `R9` register must be 0, also on resumption.
  Bits 63:55 (`LAST_ENTRY`) must be the number of pages returned in `RDX` by `TDH.EXPORT.STATE.VP`
  minus 1. The rest of `R9` is the HPA list address. The TDX module saves the progress in the
  migration stream context, so the VMM passes the same `R9` value on resumption.
- Bits 15:0 of the `R10` register are `MIGS_INDEX` and bit 63 is `RESUME`. The other bits are
  reserved and must be 0.

Unlike [TDH.IMPORT.STATE.IMMUTABLE](#tdhimportstateimmutable) and `TDH.IMPORT.STATE.TD`, which
require stream 0, this seamcall can use any created stream. The VMM must import the bundle on the
stream that the source used to export it.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.
- Error information (`RCX`): The seamcall checks every
  [metadata field](tdx-structs.md#metadata-fields) of the bundle against the destination vCPU and
  the TDX module configuration. If a check fails, `RCX` identifies the offending field or the field
  sequence header, for example a field whose value the destination does not support. See the exit
  codes below.

### Exit Codes

Codes with the `_FATAL` suffix fail the import and move the destination TD to `FAILED_IMPORT`.

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The vCPU state bundle was imported.
- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): The VMM resumes by executing the
  seamcall again for the same vCPU and stream, with `RESUME` set to 1. If the VMM cannot resume, it
  can abort the [import session](tdx-migration-misc.md#import-session) with
  [TDH.IMPORT.ABORT](#tdhimportabort).
- `TDX_INVALID_RESUMPTION`: `RESUME` is set while no resumption is expected, or clear while a
  resumption is expected.
- `TDX_INVALID_RESUMPTION_FATAL`: On resumption, `HPA_LIST_INFO` differs from the value used before
  the interruption, or the interrupted seamcall was another one.
- `TDX_INCOMPATIBLE_MBMD_MAC_CONTEXT_FATAL`: The TDX module was updated while the seamcall was
  interrupted, and the update was configured to detect this incompatibility. The VMM must abort the
  import session and can restart it.
- `TDX_INCORRECT_MBMD_MAC_FATAL`: The MAC of the bundle is incorrect. This also happens if the TDX
  module was updated while the seamcall was interrupted and the update was not configured to detect
  the incompatibility.
- `TDX_INVALID_MBMD_FATAL`: The MBMD is invalid.
- `TDX_INVALID_METADATA_LIST_HEADER_FATAL`: A metadata list header in the bundle is invalid.
- `TDX_METADATA_FIELD_NOT_WRITABLE_FATAL`: A metadata field in the bundle is not writable. `RCX`
  contains the ABI field ID.
- `TDX_METADATA_FIELD_VALUE_NOT_VALID_FATAL`: A metadata field in the bundle has an invalid value.
  `RCX` contains the ABI field ID.
- `TDX_REQUIRED_METADATA_FIELD_MISSING_FATAL`: A required metadata field is missing from the bundle.
  `RCX` contains the ABI field ID.
- `TDX_METADATA_FIELD_ID_INCORRECT_FATAL`: A field ID or a field sequence header in the bundle is
  incorrect. `RCX` contains the field ID or the field sequence header.
- `TDX_ALL_VCPUS_IMPORTED_FATAL`: The bundle exceeds the number of vCPUs of the TD, because the
  state of every vCPU was already imported.
- `TDX_X2APIC_ID_NOT_UNIQUE_FATAL`: The x2APIC ID of the vCPU is not unique. The seamcall checks it
  only if topology virtualization is configured for the TD.
- `TDX_VCPU_STATE_INCORRECT_FATAL`: The vCPU is not in the state that the import requires.
- `TDX_VCPU_ASSOCIATED_FATAL`: The vCPU is associated with another CPU.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The destination TD must be in `STATE_IMPORT`.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `MIGS_INDEX` is out of range, a
  reserved bit is set, or the MBMD buffer is too small or misaligned.
- `TDX_OPERAND_INVALID_FATAL`: The MBMD buffer, the HPA list descriptor, or the HPA of a state page
  is invalid.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example the migration
  stream or the vCPU.

---

## TDH.IMPORT.COMMIT

The `TDH.IMPORT.COMMIT` seamcall lets the destination TD run while the import of private memory
pages is still in progress. It belongs to
[post-copy migration](tdx-migration-misc.md#pre-copy-vs-post-copy-migration), where the TD starts
before the VMM imports all pages.

Here is the intended seamcall sequence on the destination in post-copy migration:

1. [TDH.IMPORT.TRACK](#tdhimporttrack) imports the start token. The destination TD enters the
   [POST_IMPORT](tdx-misc.md#td-operation-states) state and cannot run yet.
2. `TDH.IMPORT.COMMIT` moves the destination TD to `LIVE_IMPORT`, where it can run.
3. [TDH.IMPORT.MEM](#tdhimportmem) imports the remaining pages while the destination TD runs, on
   demand or in the background. The TD stays in `LIVE_IMPORT`.
4. [TDH.IMPORT.END](#tdhimportend) ends the [import session](tdx-migration-misc.md#import-session)
   once all pages are imported. The destination TD enters `RUNNABLE`.

`TDH.IMPORT.COMMIT` is the point after which the migration cannot be aborted anymore:

- Before the seamcall, the destination TD never ran, so the VMM can abort the migration with an
  abort token that the destination generates. The source TD then runs again.
- After the seamcall, the destination TD runs and can no longer generate an abort token, so the TDX
  module does not allow the source TD to run again.

After the seamcall, an import failure does not move the destination TD to `FAILED_IMPORT` anymore.
It ends the import session and moves the TD to `RUNNABLE`. The TD keeps running, but the pages that
were not imported yet stay missing.

This document describes pre-copy migration, where the VMM calls `TDH.IMPORT.END` directly from
`POST_IMPORT` and never calls `TDH.IMPORT.COMMIT`.

The document focuses on non-blocking export, and the current TDX module implementation does not
support post-copy migration with it. This is why this section stays short. See
[Post-Copy Support](tdx-migration-misc.md#post-copy-support) for more details.

---

## TDH.IMPORT.END

The `TDH.IMPORT.END` seamcall ends the [import session](tdx-migration-misc.md#import-session) on the
destination. The VMM calls it after the destination imported all private memory pages, the mutable
TD state, and the state of every vCPU. The seamcall then moves the destination TD to the
[RUNNABLE](tdx-misc.md#td-operation-states) state, the normal state of a TD after the import.

The VMM can call the seamcall to end both
[pre-copy](tdx-migration-misc.md#pre-copy-vs-post-copy-migration) and post-copy migration:

- **Pre-copy migration**: The destination TD does not run and is in `POST_IMPORT`
  ([TDH.IMPORT.TRACK](#tdhimporttrack) with the start token moved it there earlier).
  - On success, the seamcall moves the TD to `RUNNABLE`.
  - On failure, the TD stays in `POST_IMPORT`. The VMM can retry the seamcall or abort the migration
    with [TDH.IMPORT.ABORT](#tdhimportabort).
- **Post-copy migration**: The destination TD runs and is in `LIVE_IMPORT`
  ([TDH.IMPORT.COMMIT](#tdhimportcommit) moved it there earlier).
  - On success, the seamcall moves the TD to `RUNNABLE`.
  - On failure, the TD stays in `LIVE_IMPORT` and keeps running. The VMM can retry the seamcall, but
    cannot abort the migration, because `TDH.IMPORT.COMMIT` already committed it.

The only input argument is the [TD handle](tdx-structs.md#tdr) in `RCX`. The only output is the exit
code in `RAX`.

**Note**: In post-copy migration, calling the seamcall is optional. If the VMM does not call it, the
TD keeps running in `LIVE_IMPORT`, but some limitations of the import session stay. In my opinion,
the VMM should call `TDH.IMPORT.END` to fully end the import session and lift those limitations.

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The import session ended and the destination TD entered `RUNNABLE`.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The destination TD must be in `POST_IMPORT` or `LIVE_IMPORT`, so the start token
  must be imported first.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, a reserved bit is set, or the TDX
  module does not support TD migration.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example another seamcall
  for the TD is running.

---

## TDH.IMPORT.ABORT

The `TDH.IMPORT.ABORT` seamcall aborts the [import session](tdx-migration-misc.md#import-session) on
the destination. It moves the destination TD to the `FAILED_IMPORT`
[operation state](tdx-misc.md#td-operation-states), where the TDX module does not allow the TD to
run and the VMM can only tear it down.

The VMM can invoke the seamcall in this window:

- **Earliest**: The start of the import session, when
  [TDH.IMPORT.STATE.IMMUTABLE](#tdhimportstateimmutable) completes and the destination TD enters
  `MEMORY_IMPORT`.
- **Latest**: Before the destination TD starts running. This moment depends on the migration type:
  - **Pre-copy migration**: [TDH.IMPORT.END](#tdhimportend) starts the destination TD, which enters
    `RUNNABLE`.
  - **Post-copy migration**: [TDH.IMPORT.COMMIT](#tdhimportcommit) starts the destination TD, which
    enters `LIVE_IMPORT`.

**Notes**:

- The VMM can also invoke the seamcall while `TDH.IMPORT.STATE.IMMUTABLE` is interrupted and the
  destination TD is in `START_IMPORT`.
- The VMM can also invoke the seamcall in `FAILED_IMPORT`, for example when the TD ended up there
  because of a failed [TDH.IMPORT.MEM](#tdhimportmem).
- A repeated invocation in `FAILED_IMPORT` generates a new abort token.

The seamcall does the following:

- Moves the destination TD to `FAILED_IMPORT`, unless the TD is already in this state.
- Generates the **abort token** for the source platform and writes it to the MBMD buffer.

The abort token is an [MBMD-only bundle](tdx-migration-structs.md#mbmd-only-bundles) that confirms
that the destination TD is in `FAILED_IMPORT` and the TDX module does not allow it to run. The VMM
delivers the token to the source host, where [TDH.EXPORT.ABORT](#tdhexportabort) consumes it.

The two abort seamcalls are asymmetric with respect to the abort token:

- `TDH.IMPORT.ABORT` always generates the abort token regardless of whether the destination imported
  the start token.
- `TDH.EXPORT.ABORT` accepts the abort token only if the source already generated the start token,
  which moves it to the [out-of-order phase](tdx-migration-misc.md#migration-phases). In the
  in-order phase, it rejects the token with `TDX_OPERAND_INVALID`.

### Input Arguments

- MBMD buffer (`R8`): HPA in bits 51:0 and size in bits 63:52 of the buffer for the abort token. The
  HPA must be aligned to 128 bytes. The size must be at least 128 bytes, the maximum MBMD size.
- `MIGS_INDEX` (`R10`, bits 15:0): Index of the
  [migration stream](tdx-migration-structs.md#migration-stream), must be 0. The other bits are
  reserved and must be 0.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS_FATAL` on success.

On success, the seamcall writes the abort token to the MBMD buffer.

### Exit Codes

For this seamcall, the success code is `TDX_SUCCESS_FATAL` (`TDX_SUCCESS` with the `FATAL` bit set).
It tells the VMM that the destination TD can only be torn down.

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS_FATAL`: The import session was aborted and the abort token was written to the MBMD
  buffer. The destination TD is in `FAILED_IMPORT`.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The destination TD must be in `START_IMPORT`, `MEMORY_IMPORT`, `STATE_IMPORT`,
  `POST_IMPORT`, or `FAILED_IMPORT`. In `LIVE_IMPORT` and `RUNNABLE`, the abort is not possible.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `MIGS_INDEX` is not 0, a reserved
  bit is set, or the MBMD buffer is too small or misaligned.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example the migration
  stream.

---

## TDH.EXPORT.ABORT

The `TDH.EXPORT.ABORT` seamcall aborts the [export session](tdx-migration-misc.md#export-session) on
the source and returns the TD to the `RUNNABLE` [operation state](tdx-misc.md#td-operation-states).

The VMM can invoke the seamcall in this window:

- **Earliest**: The start of the export session, when
  [TDH.EXPORT.STATE.IMMUTABLE](#tdhexportstateimmutable) completes and the source TD enters
  `LIVE_EXPORT`. The setup session, which precedes the export session, has its own abort seamcall,
  [TDH.MIG.SETUP.ABORT](#tdhmigsetupabort).
- **Latest**: Before the destination TD starts running. This moment depends on the migration type:
  - **Pre-copy migration**: [TDH.IMPORT.END](#tdhimportend) starts the destination TD, which enters
    `RUNNABLE`.
  - **Post-copy migration**: [TDH.IMPORT.COMMIT](#tdhimportcommit) starts the destination TD, which
    enters `LIVE_IMPORT`.

**Note 1**: The VMM can also invoke the seamcall while `TDH.EXPORT.STATE.IMMUTABLE` is interrupted
and the source TD is in `START_EXPORT`.

**Note 2**: Once the destination TD runs, its state diverges from the source. An abort would have to
move that state back to the source, which TDX does not implement.

The start token gives the destination TD the right to run. Once the source generates it, an abort
needs proof that the destination TD gave away its right to run. The proof is the **abort token**, an
[MBMD-only bundle](tdx-migration-structs.md#mbmd-only-bundles) that the destination generates with
[TDH.IMPORT.ABORT](#tdhimportabort). The token confirms that the destination TD is in
`FAILED_IMPORT`, where the TDX module does not allow it to run.

The abort procedure depends on whether the source generated the start token:

- **Start token not generated** ([in-order phase](tdx-migration-misc.md#migration-phases)): The
  source TD is in `START_EXPORT`, `LIVE_EXPORT`, or `PAUSED_EXPORT`. The destination TD has no right
  to run yet, so the abort needs no abort token.
- **Start token generated** (out-of-order phase): The source TD is in `POST_EXPORT`. The destination
  TD has the right to run, so the abort needs the abort token.

On success, the seamcall marks the migration streams as not initialized and disabled. With
[non-blocking export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export), it also resets
the state of the comprehensive scan, like [TDH.MEM.SCAN.RESET](#tdhmemscanreset) does.

### SEPT Cleanup

After `TDH.EXPORT.ABORT`, the SEPT entries of exported pages remain in their
[export states](tdx-misc.md#sept-export-states), such as `EXPORTED_MODIFIED`, and the VMM has to
clean them up.

The TDX module provides two mechanisms for this:

- [TDH.EXPORT.RESTORE](#tdhexportrestore): Suitable when the VMM knows the GPAs of the pages to
  restore.
- [EXPORT_RESTORE](#export_restore) operation of `TDH.MEM.SCAN.RANGE`: Suitable when the VMM knows a
  GPA range that covers the exported pages.

Each restored page decrements the [MIG_COUNT](tdx-migration-structs.md#migration-counters) counter
of the TD, which counts the exported pages. After an abort, the counter shows how many exported
pages still need cleanup, and the VMM can read it with `TDH.MNG.RD`. A new export session requires
the counter to be 0, otherwise `TDH.EXPORT.STATE.IMMUTABLE` returns
`TDX_PREVIOUS_EXPORT_CLEANUP_INCOMPLETE`.

### Input Arguments

- MBMD buffer (`R8`): Depends on the phase:
  - **In-order phase**: Must be 0, because the abort needs no abort token.
  - **Out-of-order phase**: HPA in bits 51:0 and size in bits 63:52 of the buffer with the abort
    token. The HPA must be aligned to 128 bytes, and the size must be at least 128 bytes, the
    maximum MBMD size.
- `MIGS_INDEX` (`R10`, bits 15:0): Index of the
  [migration stream](tdx-migration-structs.md#migration-stream), must be 0 in both phases. The other
  bits are reserved and must be 0.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The export session was aborted and the source TD entered `RUNNABLE`.
- `TDX_INCORRECT_MBMD_MAC`: The MAC of the abort token is incorrect.
- `TDX_INVALID_MBMD`: The MBMD is invalid. For example, it is not an abort token.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The source TD must be in `START_EXPORT`, `LIVE_EXPORT`, `PAUSED_EXPORT`, or
  `POST_EXPORT`.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `MIGS_INDEX` is not 0, a reserved
  bit is set, `R8` is not 0 in the in-order phase, or the MBMD buffer is too small or misaligned.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example the migration
  stream.

---

## TDH.EXPORT.RESTORE

The `TDH.EXPORT.RESTORE` seamcall moves the SEPT entries of source TD pages out of an
[export state](tdx-misc.md#sept-export-states), for example `EXPORTED_MODIFIED`, back to the
matching non-exported state, for example `MAPPED`.

The VMM uses the seamcall for SEPT cleanup after it aborted the
[export session](tdx-migration-misc.md#export-session) with [TDH.EXPORT.ABORT](#tdhexportabort). The
abort leaves the exported pages in their export states, and the VMM must move them back before it
starts a new export session. The TD must be in the `RUNNABLE`
[operation state](tdx-misc.md#td-operation-states), so the abort must be complete.

**Note**: This seamcall suits the case where the VMM knows which pages need restoring, because it
provides the GPAs. The alternative is the [EXPORT_RESTORE](#export_restore) operation of
`TDH.MEM.SCAN.RANGE`, which finds the pages itself by scanning a GPA range.

### Input Arguments

The VMM provides a [GPA list](tdx-migration-structs.md#gpa-list) of up to 512 pages whose SEPT
entries need restoring.

- `GPA_LIST_INFO` (`RCX`): [Descriptor](tdx-migration-structs.md#gpa_list_info) of the GPA list.
  - `FORMAT` (bits 2:0): Must be 0 (`GPA_ONLY`).
  - `FIRST_ENTRY` (bits 11:3): Index of the first entry to process. On a resumed invocation, it is
    the index returned by the previous invocation.
  - `LAST_ENTRY` (bits 63:55): Index of the last entry to process.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.
- `GPA_LIST_INFO` (`RCX`): Same as the input, but `FIRST_ENTRY` is the index of the next entry to
  process. After the last entry, it is (`LAST_ENTRY` + 1) modulo 512.
- Error count (`R8`): Number of GPA list entries that failed in this invocation. After a resumption,
  it counts only the entries processed since the resumption. The seamcall returns it only when the
  VMM invokes version 1, enumerated by the `LIST_ERROR_COUNT` bit of
  [TDX_FEATURES0](tdx-structs.md#tdx_features0).

### Restored States

With [non-blocking export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export), the
seamcall changes the SEPT entry state of a page as follows:

- `EXPORTED` and `EXPORTED_MODIFIED` -> `MAPPED`
- `EXPORTED_BLOCKED` -> `BLOCKED`
- `EXPORTED_REMOVED` -> `FREE`
- `EXPORTED_REMOVE_IN_PROGRESS` -> `REMOVE_IN_PROGRESS`
- `PENDING_EXPORTED` and `PENDING_EXPORTED_MODIFIED` -> `PENDING`
- `PENDING_EXPORTED_BLOCKED` -> `PENDING_BLOCKED`

Each restored page decrements the [MIG_COUNT](tdx-migration-structs.md#migration-counters) counter.
When it reaches 0, no exported pages are left to restore.

A page in any other state, for example `MAPPED`, needs no restore. The seamcall leaves it unchanged
and sets the `OPERATION` field of its GPA list entry to `NOP` and the `STATUS` field to
`SEPT_ENTRY_STATE_INCORRECT`. Both fields are described below.

**Note**: The seamcall also restores the states of write-blocking export, such as `BLOCKEDW`. This
document does not cover them.

### GPA List Entry Fields

Each GPA list entry has input fields, which the seamcall reads, and output fields, which it writes.
Here are the notable fields.

- `GPA` (bits 51:12): [GFN](x86-misc.md#page-numbers-and-page-frames) of the page to restore. The
  seamcall does not change it.
- `LEVEL` (bits 1:0): Must be 0 (4 KiB).
- `MIG_TYPE` (bits 11:10): Must be 0 (`PAGE_4K`).
- `STATE` (bits 4:3): Must be 0.
- `PENDING` (bit 2): Ignored on input. The seamcall does not change it.
- `OPERATION` (bits 53:52): On input, `RESTORE` (1 or 3) requests the restore and `NOP` (0 or 2)
  skips the entry. On output, `RESTORE` means that the seamcall restored the page and `NOP` means
  that it did not.
- `STATUS` (bits 60:56): Ignored on input. On output, the result of processing the entry, as listed
  below.

The notable `STATUS` values are:

- `SUCCESS` (0): The entry was processed successfully.
- `SKIPPED` (1): The input `OPERATION` was `NOP`, so the seamcall skipped the entry.
- `SEPT_WALK_FAILED` (2): The SEPT walk failed for the GPA, for example because a non-leaf SEPT
  entry at an upper level is in the [NL_BLOCKED](tdx-misc.md#non-leaf-entry-states) state.
- `SEPT_ENTRY_BUSY_HOST_PRIORITY` (3): A concurrent operation held the SEPT entry of the page. The
  VMM should retry restoring the page. See [Host Priority](tdx-misc.md#host-priority).
- `SEPT_ENTRY_STATE_INCORRECT` (4): The SEPT entry state of the page does not allow the restore,
  because the page is not in an export state.
- `GPA_LIST_ENTRY_INVALID` (15): A field of the GPA list entry is invalid. The seamcall stops
  processing the list.

A `STATUS` other than `SUCCESS` or `SKIPPED` does not stop the seamcall, except for
`GPA_LIST_ENTRY_INVALID`. The TDX module skips the page, marks the entry, continues with the next
one, and can still return `TDX_SUCCESS`. The VMM must check the `STATUS` field of each entry.

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The seamcall processed all entries of the GPA list. Some entries may have errors.
  The error count in `R8` and the `STATUS` fields tell which.
- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): The VMM resumes by executing the
  seamcall again with the `FIRST_ENTRY` value returned in `RCX`, and keeps the other inputs
  unchanged. There is no resume flag.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The source TD must be in `RUNNABLE`.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, `FORMAT` is not 0, a reserved bit
  is set, or the TDX module does not support TD migration.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy.

---

## TDH.MEM.SCAN.CONFIG

The `TDH.MEM.SCAN.CONFIG` seamcall splits the private GPA space of the source TD into ranges and
sub-ranges for the [TDH.MEM.SCAN.COMP](#tdhmemscancomp) seamcall. The split makes it possible to
scan different GPA ranges in parallel.

The VMM describes the split in a [range list](tdx-migration-structs.md#range-list) and provides it
to the seamcall as an input argument.

The VMM also donates pages that the TDX module uses to store the scan control structures. The
structures hold the range configuration, the scan state, and the
[scan contexts](tdx-migration-structs.md#scan-contexts) of concurrent `TDH.MEM.SCAN.COMP`
invocations. The VMM can reclaim the pages only when it tears down the TD.

The TDX module allows the VMM to call this seamcall during the downtime period, before the `DCHECK`
operation of `TDH.MEM.SCAN.COMP`. However, it is recommended to call it earlier, while the source TD
is still running, to minimize the downtime. The intended sequence is:

1. `TDH.MEM.SCAN.CONFIG` configures the scan.
2. Optionally, `TDH.MEM.SCAN.COMP` runs the pre-scan operations `PRECLEAR` and `PRECHECK` while the
   TD is still running.
3. [TDH.EXPORT.PAUSE](#tdhexportpause) pauses the TD and starts the downtime period.
4. `TDH.MEM.SCAN.COMP` runs the `DCHECK` operation during the downtime period.

The seamcall is available only if the TDX module is configured for
[non-blocking export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export).

The VMM can call the seamcall again when no `TDH.MEM.SCAN.COMP` invocation is running. A repeated
call is any call after the first one, including the first call of a repeated migration after
[TDH.EXPORT.ABORT](#tdhexportabort). The abort keeps the control structures, including the range
configuration, so a repeated call does two things:

- **Replaces the configuration**: The new range list takes effect.
- **Resets the scan**: The seamcall resets the state of the comprehensive scan and discards the
  contexts of interrupted invocations, so the VMM can start a new scan. This is the same as calling
  [TDH.MEM.SCAN.RESET](#tdhmemscanreset).

### Input Arguments

- `RANGE_LIST_INFO` (`RCX`): [Descriptor](tdx-migration-structs.md#range-list) of the range list. It
  holds the HPA of the range list page and the number of valid entries.
- **Control page registers** (`R8` through `R11`): HPAs of the pages that the TDX module turns into
  scan control structures.

Here are the details of the control pages:

- The required number of control pages, from 1 to 4, is enumerated by the
  `NUM_MEM_SCAN_CONTROL_PAGES` global [metadata field](tdx-structs.md#metadata-fields), readable
  with `TDH.SYS.RD`.
- The VMM passes the pages in `R8` first, then `R9`, and so on. The seamcall ignores the control
  page registers beyond the required number.
- On a repeated call, the control structures already exist, so the seamcall does not use the pages.
  The control page registers within the required number must still hold `NULL_PA` (-1) or a valid
  HPA.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS` on success.
- Control page registers (`R8` through `R11`): On a successful repeated call, the TDX module sets
  bit 63 of each register within the required number to indicate that it did not use the page.

**Note**: The initial configuration never sets bit 63, because it uses all the pages within the
required number. A successful repeated call sets the bit in all of them, because it uses none of
them.

The seamcall does not modify the other registers.

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The scan is configured.
- `TDX_MISSING_PAMT_PAGE_PAIR`: With dynamic PAMT, a PAMT page pair is missing for a control page.
  The VMM adds it with `TDH.PHYMEM.PAMT.ADD` and retries.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. For example, the TD is in `UNINITIALIZED` or `POST_EXPORT`.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, the number of range entries is
  out of bounds, a range entry violates the rules of the range list, or a reserved bit is set.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy.

---

## TDH.MEM.SCAN.RESET

The `TDH.MEM.SCAN.RESET` seamcall resets the state of the comprehensive scan of the source TD. The
VMM uses it to start a new scan, for example after a failed `DCHECK`. See
[Resetting a Scan](#resetting-a-scan) for the details.

The reset discards the contexts of interrupted [TDH.MEM.SCAN.COMP](#tdhmemscancomp) invocations, so
the VMM cannot resume them.

The seamcall is available only if the TDX module is configured for
[non-blocking export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export).

The only input argument is the [TD handle](tdx-structs.md#tdr) in `RDX`. The only output is the exit
code in `RAX`.

### Exit Codes

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The state of the scan was reset.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. For example, the TD is in `UNINITIALIZED` or `POST_EXPORT`.
- `TDX_OPERAND_INVALID`: An input operand is invalid, for example a reserved bit is set.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy.

---

## TDH.MEM.SCAN.COMP

The `TDH.MEM.SCAN.COMP` seamcall performs the comprehensive scan with the `DCHECK` operation. It
also supports an optional pre-scan that shortens the comprehensive scan. The pre-scan consists of
the `PRECLEAR` and `PRECHECK` operations.

The seamcall is available only if the TDX module is configured for
[non-blocking export](tdx-migration-misc.md#non-blocking-vs-write-blocking-export), and only after
[TDH.MEM.SCAN.CONFIG](#tdhmemscanconfig) configured the scan. The VMM can run several concurrent
invocations of the same operation. They scan in parallel as parts of one scan, and the TDX module
distributes the work among them, see [Range List](tdx-migration-structs.md#range-list).

The description below covers `DCHECK` first. The pre-scan follows at the end.

`DCHECK` performs the **comprehensive scan**, the final scan of the entire private GPA space of the
source TD that identifies the pages that have to be migrated. In contrast, [DSCAN](#tdhmemscanrange)
scans only a specific GPA range and gives no guarantee that all migration candidates within that
range are found.

`DCHECK` reports all remaining export candidates, so that the VMM can export the remaining pages
during the downtime period. The VMM must follow this sequence:

1. Pause the TD with [TDH.EXPORT.PAUSE](#tdhexportpause). `DCHECK` fails if the TD is not paused.
2. Run `DCHECK` until the scan completes successfully.
3. Export all reported pages with [TDH.EXPORT.MEM](#tdhexportmem).
4. Export the mutable TD state with [TDH.EXPORT.STATE.TD](#tdhexportstatetd) and the state of every
   vCPU with [TDH.EXPORT.STATE.VP](#tdhexportstatevp).
5. Generate the start token with [TDH.EXPORT.TRACK](#tdhexporttrack).

**Note**: The state export requires only a paused TD, so step 4 can also run before steps 2 and 3.

Here is the logic behind this sequence:

- **Pause first**: The TD does not run, so its memory and state do not change. This is why `DCHECK`
  can find all remaining export candidates, and why the VMM can export the final memory, vCPU, and
  TD state.
- **Scan and export before the start token**: The start token hands execution over, so the source TD
  stops and the destination TD runs with the memory it received. The source TDX module therefore
  generates the token only if all memory was exported. It can tell this only if `DCHECK` found all
  export candidates and the VMM exported them, see [Start Token Checks](#start-token-checks).

Conceptually, `DCHECK` is like `DSCAN` that:

- Scans the entire private GPA space.
- Does not skip pages. Where `DSCAN` skips a page with a busy SEPT entry, `DCHECK` returns
  `TDX_INTERRUPTED_BUSY`, and where `DSCAN` skips a blocked entry, `DCHECK` fails the scan.
- Does not record a [scan epoch](tdx-misc.md#scan-epoch) in PAMT entries and does not need
  [TLB tracking](tdx-misc.md#tlb-tracking), because the TD is paused.

### Input Arguments

These are the arguments of `DCHECK`. The pre-scan operations differ in a few of them, as explained
in the Optional Pre-Scan subsection below.

- `LIST_OF_LISTS_INFO` (`RCX`): [Descriptor](tdx-migration-structs.md#list_of_lists_info) of the
  output GPA list-of-lists. The VMM sets up the structure as for `TDH.MEM.SCAN.RANGE`. It allocates
  the GPA list pages and fills the top-level page with their `GPA_LIST_INFO` values, and the TDX
  module writes only the GPA list entries.
- `CONTROLS` (`R8`): Operation, qualifier, and scan identifiers, described below.
- `OVERALL_NEXT_ENTRY` (`R11`): Index of the next GPA list entry to write, counted over all GPA list
  pages, see [entry index](tdx-migration-structs.md#overall_next_entry). It is 0 for a new empty
  list-of-lists, and otherwise the value returned by the previous invocation.

Here are the details of `CONTROLS`:

- `OPERATION` (bits 7:0): Must be 1, which means `DCHECK`.
- `QUALIFIER` (bits 15:8): `REEXPORT` (bit 8) is the only qualifier bit that `DCHECK` accepts. The
  other bits must be 0.
  - `REEXPORT` is 0: Reports both pages that were never exported and pages that need re-export. The
    start token request requires this value.
  - `REEXPORT` is 1: Reports only pages that need re-export.
- `CONTEXT_ID` (bits 47:32): Identifies the [scan context](tdx-migration-structs.md#scan-contexts)
  of the invocation. Concurrent invocations must use different values.
- `RANGE_ID` (bits 55:48): Index of the range to scan in the
  [range list](tdx-migration-structs.md#range-list) that [TDH.MEM.SCAN.CONFIG](#tdhmemscanconfig)
  configured.
- `RESUME` (bit 63): 1 when resuming after an interrupted exit code.
- The other bits are reserved and must be 0.

**Note**: My interpretation is that `DCHECK` with `REEXPORT` 1 is a provision for
[post-copy migration](tdx-migration-misc.md#pre-copy-vs-post-copy-migration), where the VMM could
leave never exported pages for later on-demand delivery. With non-blocking export, the current TDX
module requires all pages to be exported before the start token, so post-copy is not implemented,
see [Post-Copy Support](tdx-migration-misc.md#post-copy-support). As a result, the final `DCHECK`
always uses `REEXPORT` 0.

### Output Arguments

- Exit code (`RAX`): Seamcall status code, `TDX_SUCCESS`, `TDX_MEM_RANGE_SCAN_SUCCESS`, or
  `TDX_MEM_SCAN_SUCCESS` on success.
- `OVERALL_NEXT_ENTRY` (`R11`): Index of the next free GPA list entry.

The seamcall does not modify the other registers.

### Output GPA List

The entries that `DCHECK` writes have the same format as the entries of
[TDH.MEM.SCAN.RANGE](#tdhmemscanrange). The VMM can pass the GPA lists to `TDH.EXPORT.MEM` without
changes.

The capacity of the list-of-lists and the handling of `TDX_INTERRUPTED_LIST_FULL` are the same as
for `TDH.MEM.SCAN.RANGE`.

### Resetting a Scan

If `DCHECK` finds a blocked SEPT entry, it fails the scan with the
`TDX_MEM_SCAN_FAILED_BLOCKED_RANGE` exit code. After this failure, any `TDH.MEM.SCAN.COMP`
invocation fails with `TDX_MEM_SCAN_FAILED_OTHER_THREAD` until the VMM resets the scan. There are
two ways to do it:

- [TDH.MEM.SCAN.RESET](#tdhmemscanreset) resets the state of the scan and takes only the TD handle.
- [TDH.MEM.SCAN.CONFIG](#tdhmemscanconfig) replaces the configuration and also resets the state of
  the scan.

**Note**: After a completed `DCHECK`, the TDX module also requires a reset before the next scan, but
I see no practical reason for scanning again.

### Exit Codes

The seamcall is resumable. The TDX module saves the progress in the scan context selected by
`CONTEXT_ID`. To resume after an interrupted exit code, the VMM invokes the seamcall again with
`RESUME` set to 1, the same `CONTEXT_ID`, `RANGE_ID`, `OPERATION`, and `QUALIFIER`, and the
`OVERALL_NEXT_ENTRY` value returned in `R11`.

Here are the notable exit codes of this seamcall.

- `TDX_SUCCESS`: The invocation completed its part of the scan. Other invocations are still scanning
  the current range.
- `TDX_MEM_RANGE_SCAN_SUCCESS`: The scan of the current range completed. Other invocations are still
  scanning other ranges. The TDX module returns this code at most once for each range.
- `TDX_MEM_SCAN_SUCCESS`: The scan of the entire private GPA space completed. The TDX module returns
  this code at most once for each scan.
- [TDX_INTERRUPTED_RESUMABLE](tdx-misc.md#interruptible-seamcalls): A pending interrupt was
  detected. The VMM resumes as described above.
- [TDX_INTERRUPTED_BUSY](tdx-misc.md#tdx_operand_busy-vs-tdx_interrupted_busy): A SEPT entry or the
  scan state is locked by another operation. The VMM resumes as described above.
- `TDX_INTERRUPTED_LIST_FULL`: The GPA list-of-lists ran out of free entries. The VMM provides a
  list-of-lists with free entries, for example a new one with `OVERALL_NEXT_ENTRY` set to 0, and
  resumes.
- `TDX_MEM_SCAN_FAILED_BLOCKED_RANGE`: `DCHECK` found a blocked SEPT entry. This fails the whole
  scan, not only this invocation. To start a new scan, the VMM must
  [reset the scan](#resetting-a-scan) first.
- `TDX_MEM_SCAN_FAILED_OTHER_THREAD`: Another invocation failed the scan, for example with
  `TDX_MEM_SCAN_FAILED_BLOCKED_RANGE`. The scan stays failed until the VMM resets it.
- `TDX_MEM_SCAN_CONFIG_REQUIRED`: `TDH.MEM.SCAN.CONFIG` has not configured the scan.
- `TDX_INVALID_RESUMPTION`: `RESUME` is set while no resumption is expected, or clear while a
  resumption is expected.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The source TD must be in `PAUSED_EXPORT`.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, a reserved bit is set,
  `CONTEXT_ID` or `RANGE_ID` is out of range, or the operation does not fit the state of the scan,
  such as `DCHECK` after `PRECLEAR` or `DCHECK`.
- [TDX_OPERAND_BUSY](tdx-misc.md#tdx_operand_busy): A resource is busy, for example another
  invocation uses the same `CONTEXT_ID`.

### Optional Pre-Scan

The downtime period starts when the VMM pauses the source TD with
[TDH.EXPORT.PAUSE](#tdhexportpause). It ends when the destination VMM finishes importing the memory,
the TD state, and the vCPU state, and calls [TDH.IMPORT.END](#tdhimportend). The workload does not
run during this period, so the goal is to minimize it.

`DCHECK` runs during the downtime period and scans the entire private GPA space. The scan time grows
with the size of the private GPA space.

Scanning on several CPUs in parallel and overlapping the scan with memory export and import can
reduce the downtime. However, parallelism needs free CPUs, so it helps less on a heavily loaded
host.

The pre-scan takes a different approach and reduces the amount of work that `DCHECK` has to do. It
runs while the source TD is still running, before the downtime period, and allows `DCHECK` to skip
some SEPT sub-trees.

#### Accessed Bit Versus Dirty Bit

The pre-scan relies on the Accessed bit of SEPT entries, not on the Dirty bit. The CPU sets the
Dirty bit only in the leaf entry of a written page, so it does not show whether a sub-tree contains
changes.

The Accessed bit, in contrast, cascades: during a SEPT walk the CPU sets it in every entry on the
path. The TDX module sets it the same way when it walks the SEPT. So a clear Accessed bit in a
non-leaf entry means that nobody walked the sub-tree below it since the bit was cleared, and
therefore nobody changed the sub-tree.

**Note**: The TDX module also uses the Dirty bit of non-leaf entries, for its own purpose. A clear
bit means that the range has no private pages, and scans skip such a range. The bit does not show
data changes. This applies only where the TD supports the feature, and otherwise scans go down into
every range.

#### Pre-Scan Steps

The idea is to go through the SEPT before the downtime and clear the Accessed bit in the non-leaf
entries that have no export candidates below them. Then `DCHECK` skips the sub-trees with a clear
Accessed bit and scans only the rest.

One operation cannot do this, because the VMM must run the [TLB tracking](tdx-misc.md#tlb-tracking)
protocol in the middle, and the TDX module cannot run it without the VMM. So the pre-scan has two
operations, and the steps are:

1. `PRECLEAR` clears the Accessed bit in all non-leaf entries. When it completes across the entire
   private GPA space, the TDX module records the current TD epoch.
2. The VMM runs TLB tracking by calling `TDH.MEM.TRACK` and sending IPIs. From that moment, any
   access to a page sets the Accessed bit on its SEPT path.
3. `PRECHECK` goes through the SEPT and sets the Accessed bit in the entries above export
   candidates.

After that, `DCHECK` can safely skip the sub-trees with a clear bit. A page that changes after TLB
tracking and before the pause sets the Accessed bits on its path, so `DCHECK` scans the affected
sub-trees and finds the change.

TLB tracking is needed because `PRECLEAR` changes SEPT entries. A CPU with a cached translation does
not walk the SEPT, so it does not set the cleared Accessed bits again. Without the protocol, the TD
could change a page through such a translation without setting these bits, and `DCHECK` would skip
the sub-tree and miss the change.

The VMM runs the operations in the order `PRECLEAR`, TLB tracking, `PRECHECK`, `DCHECK`, and
completes each operation across the whole private GPA space before it starts the next one. It must
run both operations of the pre-scan or none. The TDX module rejects `PRECHECK` without a successful
`PRECLEAR` and rejects `DCHECK` right after `PRECLEAR`.

To repeat the pre-scan, the VMM must reset the scan before the new `PRECLEAR`, see
[Resetting a Scan](#resetting-a-scan).

#### PRECHECK List Mode

The main role of `PRECHECK` is to set the Accessed bits above export candidates. In addition, it can
report the export candidates in a GPA list-of-lists, the same way `DSCAN` does. The `LIST` bit of
the `QUALIFIER` input argument enables the reporting.

When `PRECHECK` reports a page, it clears the Dirty bit of the page and records its
[scan epoch](tdx-misc.md#scan-epoch), as `DSCAN` does. The VMM must therefore run the TLB tracking
protocol before it exports the reported pages with `TDH.EXPORT.MEM`.

**Note 1**: `PRECHECK` clears no Accessed bits, with either value of the `LIST` bit. Only `PRECLEAR`
clears them.

**Note 2**: `PRECHECK` is not a replacement for `DSCAN`. It must follow a successful `PRECLEAR` and
TLB tracking, and every repeated `PRECHECK` needs a new `PRECLEAR`, which in turn requires a scan
reset, see [Resetting a Scan](#resetting-a-scan).

#### Input Arguments

The input arguments of both operations are the same as for `DCHECK`, except for the following.

For `PRECLEAR`:

- `OPERATION` (`R8`, bits 7:0): Must be 3.
- `QUALIFIER` (`R8`, bits 15:8): All bits must be 0.
- `LIST_OF_LISTS_INFO` (`RCX`): Must be 0. `PRECLEAR` writes no GPA list entries, so there is no
  list-of-lists to provide.
- `OVERALL_NEXT_ENTRY` (`R11`): Must be 0, for the same reason.

For `PRECHECK`:

- `OPERATION` (`R8`, bits 7:0): Must be 4.
- `QUALIFIER` (`R8`, bits 15:8): `PRECHECK` accepts `REEXPORT` (bit 8) and `LIST` (bit 9):
  - `LIST` is 0: `PRECHECK` only sets the Accessed bits. It does not fill the GPA list-of-lists and
    works faster.
  - `LIST` is 1: `PRECHECK` also fills the GPA list-of-lists with the export candidates.
  - `REEXPORT`: Ignored if `LIST` is 0. If `LIST` is 1, it has the same meaning as for `DCHECK`.
- `LIST_OF_LISTS_INFO` (`RCX`): Must be 0 if `LIST` is 0. If `LIST` is 1, it has the same meaning as
  for `DCHECK`.
- `OVERALL_NEXT_ENTRY` (`R11`): Must be 0 if `LIST` is 0. If `LIST` is 1, it has the same meaning as
  for `DCHECK`.

#### Output Arguments

The output arguments of both operations are the same as for `DCHECK`.

#### Exit Codes

Both operations resume the same way as `DCHECK`, as described in the Exit Codes subsection above.
The following exit codes also match `DCHECK`:

- `TDX_SUCCESS`
- `TDX_MEM_RANGE_SCAN_SUCCESS`
- `TDX_MEM_SCAN_SUCCESS`
- `TDX_INTERRUPTED_RESUMABLE`
- `TDX_MEM_SCAN_CONFIG_REQUIRED`
- `TDX_INVALID_RESUMPTION`
- `TDX_OPERAND_BUSY`

These exit codes differ from `DCHECK` or exist only for the pre-scan:

- [TDX_INTERRUPTED_BUSY](tdx-misc.md#tdx_operand_busy-vs-tdx_interrupted_busy): A SEPT entry or the
  scan state is locked by another operation, or the operation yielded to a concurrent memory
  management operation, for example `TDH.MEM.PAGE.PROMOTE`. The VMM resumes as described above.
- `TDX_INTERRUPTED_LIST_FULL`: `PRECHECK` with `LIST` set to 1 ran out of free GPA list entries.
  `PRECLEAR` and `PRECHECK` with `LIST` set to 0 write no entries, so they never return this code.
  The VMM provides a list-of-lists with free entries, for example a new one with
  `OVERALL_NEXT_ENTRY` set to 0, and resumes.
- `TDX_INTERRUPTED_BLOCKED_RANGE`: The operation found a blocked SEPT entry. Unlike `DCHECK`, it
  does not fail the scan. The VMM must resolve the blocked range, for example by finishing the
  operation that blocked it, and resume.
- `TDX_TLB_TRACKING_NOT_DONE`: `PRECHECK` compares the epoch recorded by `PRECLEAR` with the current
  TD epoch and found that the VMM did not run TLB tracking after `PRECLEAR`.
- `TDX_MEM_SCAN_ALREADY_SUCCESSFUL`: A scan already completed successfully and the VMM started
  `PRECLEAR` again. To scan again, the VMM must [reset the scan](#resetting-a-scan) first.
- `TDX_MEM_SCAN_FAILED_OTHER_THREAD`: The scan is in the failed state. It stays failed until the VMM
  resets the scan.
- `TDX_OP_STATE_INCORRECT`: The [TD operation state](tdx-misc.md#td-operation-states) does not allow
  this seamcall. The source TD must be in `LIVE_EXPORT`, so it must not be paused.
- `TDX_OPERAND_INVALID`: An input operand is invalid. For example, a reserved bit is set,
  `CONTEXT_ID` or `RANGE_ID` is out of range, or the operation does not fit the state of the scan,
  such as `PRECHECK` without a successful `PRECLEAR`.
