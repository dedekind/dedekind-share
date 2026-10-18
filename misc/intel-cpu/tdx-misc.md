# TDX Module Notes

- **Author**: Artem Bityutskiy
- **Version**: 0.1
- **Date**: 2026-07-03
- **Last updated**: 2026-10-17

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
  - [Status Codes](#status-codes)
    - [TDX\_OPERAND\_BUSY](#tdx_operand_busy)
    - [Recoverability Hints](#recoverability-hints)
  - [Interruptible Seamcalls](#interruptible-seamcalls)
    - [Resumable vs Restartable](#resumable-vs-restartable)
    - [Interruptible Exit Codes](#interruptible-exit-codes)
    - [Non-Extension Seamcalls](#non-extension-seamcalls)
    - [Extension Seamcalls](#extension-seamcalls)
    - [Concurrency as a Reason to Interrupt](#concurrency-as-a-reason-to-interrupt)
    - [TDX\_OPERAND\_BUSY vs TDX\_INTERRUPTED\_BUSY](#tdx_operand_busy-vs-tdx_interrupted_busy)
    - [Must the VMM Resume?](#must-the-vmm-resume)
  - [Host Priority](#host-priority)
    - [Problem Statement](#problem-statement)
    - [Solution](#solution)
    - [Timeout Mechanism](#timeout-mechanism)
    - [Status Codes Summary](#status-codes-summary)
    - [Is Retry Mandatory?](#is-retry-mandatory)
  - [#VE](#ve)
    - [#VE Categories](#ve-categories)
    - [Architectural #VE](#architectural-ve)
    - [TDX-Extended #VE](#tdx-extended-ve)
  - [TD Operation States](#td-operation-states)
  - [SEPT Page States](#sept-page-states)
    - [Leaf Entry States](#leaf-entry-states)
    - [Live Migration-Specific States](#live-migration-specific-states)
    - [TDX Connect-Specific States](#tdx-connect-specific-states)
    - [Non-Leaf Entry States](#non-leaf-entry-states)
    - [SEPT State Encoding](#sept-state-encoding)
  - [TLB Tracking](#tlb-tracking)
    - [The TLB Tracking Protocol](#the-tlb-tracking-protocol)
    - [Why the VMM Must Drive the Flush](#why-the-vmm-must-drive-the-flush)
    - [Which CPUs Need the IPI](#which-cpus-need-the-ipi)
    - [Enforcing the Protocol](#enforcing-the-protocol)
    - [Example: Reclaiming a Private Page](#example-reclaiming-a-private-page)
  - [Page State Transitions](#page-state-transitions)
    - [Private to Shared](#private-to-shared)
    - [Shared to Private](#shared-to-private)
    - [Pending to Private](#pending-to-private)
    - [Private to Pending](#private-to-pending)
    - [Private or Pending to Unmapped](#private-or-pending-to-unmapped)
    - [Shared to Unmapped](#shared-to-unmapped)

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

## Status Codes

TDX seamcalls and tdcalls return a 64-bit completion status in `RAX`. This includes
[TD exits](#vm-exit-and-td-exit), which complete the `TDH.VP.ENTER` seamcall, so its status reports
how the TD exited. The status has the following structure.

```text
 63      60 59        48 47       40 39          32 31             0
+----------+------------+-----------+--------------+----------------+
|  FLAGS   |  RESERVED  |   CLASS   |  DETAILS_L1  |   DETAILS_L2   |
+----------+------------+-----------+--------------+----------------+
```

Here is a more detailed breakdown.

| Bits  | Name                       | Meaning                                          |
|-------|----------------------------|--------------------------------------------------|
| 63    | `ERROR`                    | The function was aborted because of an error.    |
| 62    | `NON_RECOVERABLE`          | Recoverability hint, details below.              |
| 61    | `FATAL`                    | Seamcalls only, the TD can only be torn down.    |
| 60    | `HOST_RECOVERABILITY_HINT` | Recoverability hint from the VMM, details below. |
| 59:48 | Reserved                   | Set to 0.                                        |
| 47:40 | `CLASS`                    | Status class. For example, 2 is "Resource Busy". |
| 39:32 | `DETAILS_L1`               | Distinguishes base values within the class.      |
| 31:0  | `DETAILS_L2`               | Additional details.                              |

TDX status codes consist of two halves that carry independent information.

- **Upper half** (bits 63:32): what happened. TDX specifications refer to it with two terms:
  - **Base value**: the upper half written as a 64-bit value with `DETAILS_L2` set to 0, for
    example `0x8000020000000000`.
  - **Status name**: the symbolic name of a base value, for example `TDX_OPERAND_BUSY`. There is
    one name per base value.
- **Lower half** (bits 31:0): additional details specific to the base value. Each base value may
  define its own interpretation of these bits.

Most often the lower half holds the **operand ID**, naming the operand that the base value refers
to. The operand ID of a SEPT entry is `SEPT_ENTRY`, 147 (0x93), so a busy SEPT entry gives:

```text
TDX_OPERAND_BUSY(SEPT_ENTRY) = 0x8000020000000000 | 0x93 = 0x8000020000000093
```

TDX specifications write such combined values as `TDX_OPERAND_BUSY(SEPT_ENTRY)`.

Operands are of two kinds:

- **Explicit**: the GPRs the caller passes in, and components of explicit inputs. GPR operand IDs
  encode the register number, for example `RCX` is 1 and `RDX` is 2. Components of explicit
  inputs have IDs from 64, for example `TD_PARAMS.ATTRIBUTES` is 64.
- **Implicit**: resources the seamcall or tdcall touches, such as the TDR page (128), the TDCS
  (144), the TDVPS (145), or a SEPT entry (147).

Other base values use the lower half differently:

| Status name                            | `DETAILS_L2`                                 |
|----------------------------------------|----------------------------------------------|
| `TDX_SUCCESS` from `TDH.VP.ENTER`      | TD exit reason.                              |
| `TDX_SUCCESS` from a list operation    | Number of list entries that failed.          |
| `TDX_INCORRECT_MSR_VALUE`              | MSR index.                                   |
| `TDX_CPUID_MAX_SUBLEAVES_UNRECOGNIZED` | CPUID leaf.                                  |
| `TDX_TD_VMCS_FIELD_NOT_INITIALIZED`    | VMCS field code.                             |
| `TDX_EVENT_FILTER_INVALID`             | Position of the bad event in the input list. |
| `TDX_INVALID_PKG_ID`                   | Package ID.                                  |

### TDX_OPERAND_BUSY

In most cases `TDX_OPERAND_BUSY` reports that a resource the seamcall or tdcall needs is locked by a
concurrent operation. For example:

- `TDX_OPERAND_BUSY(SEPT_ENTRY)` indicates that the SEPT entry the seamcall or tdcall needs is
  locked.
- `TDX_OPERAND_BUSY(TDR)` indicates that the TDR the seamcall or tdcall needs is locked.

`TDX_OPERAND_BUSY` does not always mean that a lock is held. For example:

- [TDX extension seamcalls](#extension-seamcalls), such as `TDH.MIG.SETUP`, return
  `TDX_OPERAND_BUSY(VIRTUAL_THREAD_POOL)` when there is no free thread available.
- A seamcall may try to update a value using the `LOCK CMPXCHG` instruction but lose the race (read
  value, compute new, attempt to write but notice it changed in the meantime). For example,
  `TDH.MNG.WR` updates some [metadata fields](tdx-structs.md#metadata-fields) using this technique
  and may return `TDX_OPERAND_BUSY(METADATA_FIELD)` when the race is lost.

This return status is recoverable and indicates that the operation can be retried.

### Recoverability Hints

`NON_RECOVERABLE` is a hint. It means that an error is probably not recoverable, or, in the
`TDH.VP.ENTER` status, that the TD failed in a non-recoverable way.

`HOST_RECOVERABILITY_HINT` lets the VMM tell the TD whether a failed tdcall is worth retrying. When
a tdcall causes a trap-like TD exit, `TDH.VP.ENTER` returns a status with this flag set, for
example:

- `TDX_HOST_PRIORITY_BUSY_TIMEOUT`, a tdcall kept failing for too long on a lock that the TDX
  module reserved for the VMM after a failed seamcall, but the VMM did not retry the seamcall,
  which would take the lock and clear the reservation. See [Host Priority](#host-priority) for
  details.
- `TDX_CROSS_TD_TRAP`, a service TD tdcall such as `TDG.SERVTD.RD` failed because of an error
  related to the target TD.

The VMM supplies the hint in bit 52 of the `RCX` input on the next `TDH.VP.ENTER`, and the TDX
module copies it into this flag in the tdcall status of the TD. If set, the error is probably not
recoverable.

---

## Interruptible Seamcalls

Many TDX seamcalls are interruptible, which means they may return before finishing their work and
expect the VMM to invoke them again so that they continue from where they left off.

### Resumable vs Restartable

The TDX module defines two kinds of interruptible seamcalls:

- A **resumable** seamcall saves its intermediate state before returning, so the VMM continues it
  from the interruption point when invoked again. Resumable seamcalls are designed so that every
  resumption makes some progress, and most interruptible seamcalls are resumable.
- A **restartable** seamcall saves no intermediate state and starts from scratch when invoked
  again. It does not guarantee progress. There are only a few restartable seamcalls, for example
  `TDH.MEM.PAGE.DEMOTE` and `TDH.MEM.PAGE.PROMOTE`.

A seamcall is either resumable or restartable, never both.

### Interruptible Exit Codes

All interruptible exit codes start with `TDX_INTERRUPTED_`. They have the `ERROR` bit set (see
[Status Codes](#status-codes)), but they do not report a failure.

Here are some of the resumable exit codes:

- `TDX_INTERRUPTED_RESUMABLE`: a pending event was detected.
- `TDX_INTERRUPTED_BUSY`: a required TDX module resource was busy.
- `TDX_INTERRUPTED_LIST_FULL`: the output GPA list-of-lists ran out of free entries.
- `TDX_INTERRUPTED_LIST_END`: the VMM asked to process only part of an input GPA list. This code is
  interesting because it reports a deliberate VMM request, not an interrupt or contention.
- `TDX_INTERRUPTED_BLOCKED_RANGE`: a blocked SEPT entry was found during a scan.
- `TDX_INTERRUPTED_PAMT`: with dynamic PAMT, the operation needs more PAMT pages. The VMM has to
  add them with `TDH.PHYMEM.PAMT.ADD` and resume.

There is only one restartable exit code, `TDX_INTERRUPTED_RESTARTABLE`. All other interruptible
exit codes indicate a resumable seamcall, each for a different reason.

Notice that these exit codes use different [base values](#status-codes), instead of sharing one
base value with different operand IDs, the way `TDX_OPERAND_BUSY` does.

### Non-Extension Seamcalls

While the VMM runs a seamcall other than `TDH.VP.ENTER`, the TDX module runs in **SEAM
root mode** and does not recognize external events such as interrupts, NMI, SMI, and INIT. A long
seamcall may therefore delay the host's response to an event. Interruptible seamcalls limit this
delay by periodically checking for pending events. If one is detected, the seamcall returns early
with an interruptible exit code, allowing the host to handle the event and then resume the seamcall
from where it left off.

For example, `TDH.EXPORT.MEM` and `TDH.IMPORT.MEM` export or import up to 512 pages and check for
pending events after each 4KiB page. This works well because each page is encrypted independently:
the seamcall can stop and later resume on a page boundary.

Interrupted seamcalls save their intermediate progress so they can resume later. Here are some
examples of where that progress is saved:

- For some seamcalls, the progress is saved in the seamcall's output registers, and the VMM just
  has to pass them back as input when it resumes the seamcall. For example, `TDH.EXT.MEM.ADD`
  takes a list of up to 512 pages as input, and on interruption returns the index of the next list
  entry to process. To resume, the VMM calls it again with that index.
- `TDH.EXPORT.MEM` and `TDH.IMPORT.MEM` operate on a migration stream, identified by a stream ID
  input argument. Each stream has a stream context, a page the VMM donates to the TDX module with
  `TDH.MIG.STREAM.CREATE` before migration starts. On interruption, the seamcall saves its progress
  in that stream context and returns the updated GPA list position in `RCX`. To resume, the VMM
  calls the same seamcall again with that `RCX` value, the other inputs unchanged, and the `RESUME`
  command in the stream index operand.

### Extension Seamcalls

Some cryptographic operations do not have natural checkpoints where their progress can be saved
and restored easily. A signature calculation or key exchange may run as one long computation, and
saving its internal state for later resumption can be difficult.

TD attestation, live migration, and TDX Connect involve cryptographic operations such as:

- Signing an attestation quote requires a digital signature computation.
- The migration setup session runs an AKE (Authenticated Key Exchange) based on Diffie-Hellman to
  derive the migration session keys.
- TDX Connect establishes SPDM sessions with devices.

Operations like these run in TDX module extensions. In the current design, these extensions are
hidden TDs managed by the TDX module. They run in SEAM non-root mode and are invisible to the VMM. A
pending event can cause a [VM exit](#vm-exit-and-td-exit) from a hidden TD to the TDX module. The
hidden TD resumes at the interrupted instruction after the event is handled. The cryptographic
code continues without saving and restoring its own internal state.

**Note**: TDX module specs do not guarantee that future designs will continue using hidden TDs for
extensions. This is an internal TDX module implementation detail.

Extensions process requests using a pool of virtual threads. A virtual thread here is a TDX
module concept, not an OS thread: it is one of the hidden TD's own vCPUs, assigned to serve a
request. A request stays assigned to the virtual thread that started it until it completes, even
across an interruption: the VMM resumes an interrupted request by calling the same seamcall
again, and the extension continues on the same virtual thread.

### Concurrency as a Reason to Interrupt

The primary reason a seamcall returns an interruptible exit code is a pending interrupt. But there
may also be other reasons. Here are a couple of examples.

- A seamcall may encounter a busy resource it needs, such as a lock on a SEPT entry. Instead of
  failing, it returns `TDX_INTERRUPTED_BUSY`.
- A seamcall may own a resource that a concurrent seamcall needs. To avoid starving that seamcall,
  it yields the resource and returns `TDX_INTERRUPTED_BUSY`.

### TDX_OPERAND_BUSY vs TDX_INTERRUPTED_BUSY

Both codes report contention, but they ask the VMM to do different things:

- [`TDX_OPERAND_BUSY`](#tdx_operand_busy) is an error. The seamcall saved no continuation state, so
  the VMM repeats the invocation.
- `TDX_INTERRUPTED_BUSY` is not an error. The seamcall saved its state, so it is resumable, and the
  VMM resumes it instead.

### Must the VMM Resume?

When a seamcall exits with an interrupted exit code, the VMM is free to do other things instead of
immediately resuming it. But can the VMM ignore the interrupted operation and call the same
seamcall from scratch instead?

For restartable seamcalls, this question does not apply: they keep no state, so calling them again
from scratch is correct behavior by design, not just an option.

For resumable seamcalls, it depends on the kind of seamcall.

- For extension seamcalls, the VMM must not start a new request in place of an interrupted one. The
  backing hidden TD processes each request on a virtual thread, which stays assigned to the
  interrupted request until the VMM resumes it. Otherwise the virtual thread cannot serve any other
  request, effectively leaking a resource.
  - However, some extension seamcalls have an abort seamcall, such as `TDH.MIG.SETUP.ABORT` for
    `TDH.MIG.SETUP`. The VMM can use it to abort the Migration Setup session instead of resuming
    `TDH.MIG.SETUP`.
- For non-extension seamcalls, it depends on where the progress is kept.
  - `TDH.EXT.MEM.ADD` returns the next list-entry index in `RCX`. It stores no continuation state
    for the list in memory owned by the TDX module, so the module does not tie a later call to the
    interrupted list. A later call can use a different list. Pages added before the interruption
    remain added.
  - In other cases, the TDX module itself retains progress across calls, and the VMM must resume
    rather than start fresh to make use of it. The memory export/import seamcalls are one example:
    `TDH.EXPORT.MEM` and `TDH.IMPORT.MEM` take a stream ID as input and operate on that stream, so
    this is a per-stream context, and starting fresh on an interrupted stream fails with
    `TDX_INVALID_RESUMPTION`.

---

## Host Priority

Host priority is a TDX module mechanism that gives the host VMM precedence over the TD when
both compete for the same resource. It is not specific to live migration.

### Problem Statement

Seamcalls and tdcalls lock the shared resources they operate on, such as
the [TDR](https://github.com/dedekind/dedekind-share/blob/master/misc/intel-cpu/tdx-structs.md#tdr),
[TDCS](https://github.com/dedekind/dedekind-share/blob/master/misc/intel-cpu/tdx-structs.md#tdcs),
[TDVPS](https://github.com/dedekind/dedekind-share/blob/master/misc/intel-cpu/tdx-structs.md#tdvps),
and SEPT entries. For example:

- TD scope seamcalls, such as `TDH.MNG.RD`, lock the TDR and TDCS of the TD they operate on.
- vCPU scope seamcalls, such as `TDH.VP.ENTER`, lock TDR and TDCS too, but also can lock TDVPS.
- Tdcalls, such as `TDG.VM.RD`, lock the same shared resources as the seamcalls.

If the lock is busy, the TDX module does not spin waiting for it. It gives up, returns
`TDX_OPERAND_BUSY` (see [Status Codes](#status-codes)), and the caller is expected to retry.

The same locks can be taken from both sides:

- by the host through a seamcall
- by the TD through a tdcall

When the TD and VMM compete for the same lock simultaneously, it is conceivable that the TD wins
most of the time, and VMM keeps retrying and does not make progress. This potentially leads to TD
attacking VMM with a denial of service.

Therefore, the contention has to be resolved in favor of the VMM.

### Solution

The TDX module adds a "host priority" (`HOST_PRIORITY` or HP) bit to most locks that the VMM and
TDs may compete for. These rules govern it:

- When the VMM fails to take the lock (via a seamcall), the TDX module sets `HOST_PRIORITY` and
  returns `TDX_OPERAND_BUSY`. The [operand ID](#status-codes) in the lower half may include the ID
  of the lock that could not be acquired.
- When the VMM takes the lock, the TDX module clears `HOST_PRIORITY`.
- The TD (via a tdcall) fails to take the lock while `HOST_PRIORITY` is 1, gets `TDX_OPERAND_BUSY`
  as well, and is expected to retry.

So one failed VMM attempt closes the lock to the TD, and the VMM can take it as soon as the current
holder releases it. The TDX module makes sure this does not take long: a TD flow that holds such a
lock has a bounded duration, and normal completion of the flow releases the lock.

### Timeout Mechanism

Because only a successful seamcall clears `HOST_PRIORITY`, the VMM should retry on busy status
codes. Otherwise the TD cannot make progress because the lock remains unavailable to it.

The TDX module has a per vCPU timeout mechanism for detecting the case when the VMM does not clear
`HOST_PRIORITY` within `TDCS.HP_LOCK_TIMEOUT`, which defaults to 1 second and is configurable from
10 ms to 100 seconds with `TDH.MNG.WR`. The timeout measures how long the vCPU has been invoking
tdcalls that keep failing because `HOST_PRIORITY` is 1. Any tdcall that completes with a different
status restarts the measurement.

The TDX module checks the timeout only when such a tdcall fails. The first failure after the timeout
expires causes a trap-like TD exit. The VMM sees `TDH.VP.ENTER` return
`TDX_HOST_PRIORITY_BUSY_TIMEOUT`, which tells the VMM that it did not clear `HOST_PRIORITY` in
time. The VMM should then retry the seamcall that previously failed until it succeeds, and re-enter
the TD with `TDH.VP.ENTER`.

The TD exit is trap-like, which means the tdcall counts as executed. The TD resumes at the next
instruction and reads the tdcall completion status, rather than re-executing the tdcall. This gives
the VMM a way to tell the TD whether retrying is worth it. The VMM supplies the hint in bit 52
(`HOST_RECOVERABILITY_HINT`) of the `RCX` input on the next entry, and the TDX module reflects it
into the `HOST_RECOVERABILITY_HINT` flag (bit 60) of the TD `RAX`:

- 0: the VMM corrected the condition, so the TD may retry the tdcall.
- 1: the error is probably not recoverable.

Setting that flag turns the `TDX_OPERAND_BUSY` base value (`0x8000020000000000`) into
`TDX_OPERAND_BUSY_HOST_RECOVERABILITY_HINT` (`0x9000020000000000`).

### Status Codes Summary

The TDX ABI defines `TDX_OPERAND_BUSY_HOST_PRIORITY` for the host priority busy case, but in the
current TDX module implementation neither side observes it. The module uses it internally to drive
the timeout, and converts it to `TDX_OPERAND_BUSY` before returning to the TD, for compatibility
with older TDs. The remaining codes are:

On the TD side:

- `TDX_OPERAND_BUSY` (`0x8000020000000000`): the lock was busy, either because another flow holds
  it or because `HOST_PRIORITY` is set. The status does not distinguish the two.
- `TDX_OPERAND_BUSY_HOST_RECOVERABILITY_HINT` (`0x9000020000000000`): the `TDX_OPERAND_BUSY` base
  value with the `HOST_RECOVERABILITY_HINT` flag set.

On the VMM side:

- `TDX_OPERAND_BUSY` (`0x8000020000000000`): the lock was busy, and the TDX module has set
  `HOST_PRIORITY` if this was a host priority lock.
- `TDX_HOST_PRIORITY_BUSY_TIMEOUT` (`0x9000020500000000`): a `TDH.VP.ENTER` status telling the VMM
  that the TD cannot make progress because the host did not clear `HOST_PRIORITY` in time.
- `SEPT_ENTRY_BUSY_HOST_PRIORITY` (3) and `TD_PAGE_BUSY_HOST_PRIORITY` (12): not base values, but
  per-page `STATUS` values in the GPA list of the migration seamcalls. `TDH.EXPORT.MEM` reports the
  first one, and `TDH.IMPORT.MEM` reports either.

### Is Retry Mandatory?

On the TD side, retrying is up to the TD. Nothing in the TDX module depends on it.

On the VMM side, one could conclude from the above that retrying is important only when the busy
status is due to host priority, and optional otherwise. That is not the case. The TDX module
expects the VMM to always retry whenever a busy status is returned. There is no way for VMM to tell
if the busy status was due to host priority or not. So effectively, it is mandatory for the VMM to
retry whenever it encounters a busy status.

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

### #VE Categories

The `CATEGORY` byte of [VE_INFO](tdx-structs.md#ve_info) tells the TD what kind of event it is
looking at.

| Value | Category              | Meaning                                                    |
|-------|-----------------------|------------------------------------------------------------|
| 0x00  | `ARCH`                | An EPT violation.                                          |
| 0x01  | `PENDING`             | EPT violation on a `PENDING` page.                         |
| 0x02  | `RESERVED_GPA_BITS`   | GPA bit above `MAXGPA` (not `SHARED`) set, a TD bug.       |
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

The `CONFIG_PARAVIRT` category indicates that the feature depends on the TD configuration. The TD
either handles it, or treats it as an error caused by an incorrect VMM configuration. In the latter
case, the TD refuses to support the feature. For example, if the feature requires communicating
with the VMM via `TDG.VP.VMCALL`, the TD may choose to refuse to handle it because it does not trust
the VMM. Instead, it may choose to treat it as `#GP(0)`, which means `SIGSEGV` for userspace and an
OOPS for the kernel.

**Note**: TDX specifications use the notation like `#VE(ARCH)`, which means a `#VE` of the `ARCH`
category.

**Linux Note**: As of version 7.2, the Linux kernel does not use the `CATEGORY` field. It uses
`TDG.VP.VEINFO.GET` with version 0 instead of version 1 or higher.

### Architectural #VE

Architectural `#VE` is the same mechanism for a VMX guest and for a TD's shared memory accesses.
When the guest attempts to access memory in a way that is not permitted by the EPT, an
**EPT violation** occurs, and the CPU delivers an architectural `#VE` instead of a VM exit only in
the following conditions.

- The **EPT-violation #VE** bit in the VMCS is set. For a VMX guest this is up to the VMM. For a
  TD the TDX module controls it, and it is always set.
- Bit 63 of the EPT entry is 0. This bit means **suppress #VE**, so it is a per page decision, made
  by whoever owns the page tables (VMM for shared memory, TDX module for private memory).
- `VE_INFO.VALID` is 0, which indicates that the `VE_INFO` area is empty, as opposed to holding
  information from a previous `#VE` that the guest has not cleared, and therefore supposedly has not
  handled. Refer to
  [virtualization exception information area](x86-misc.md#virtualization-exception-information-area)
  for details.

A TD has two EPT trees: the SEPT for private GPAs and the Shared EPT for shared GPAs, and the
`SHARED` bit of the GPA selects between them. Both are ordinary EPTs as far as the CPU is concerned,
so a failed walk in either one is an EPT violation, exit reason 48. There is no separate "SEPT
violation".

The TDX module always sets the EPT-violation `#VE` VMCS bit for a TD, so whether a `#VE` is raised
or not comes down to bit 63 alone, and that bit is owned differently for the two GPA spaces.

- **Shared memory**: The VMM owns the Shared EPT and controls bit 63 per page. This is how MMIO is
  emulated. The TD gets a `#VE(ARCH)`, decodes the faulting instruction, and forwards the access
  to the VMM with `TDG.VP.VMCALL`.
- **Private memory**: The TDX module owns the SEPT and always sets bit 63 to 1, so the CPU never
  raises a `#VE`. The EPT violation becomes a [VM exit](#vm-exit-and-td-exit) into the TDX module,
  which reads the SEPT entry and decides how to handle it, depending on its state. Unlike the shared
  memory case, the CPU itself never injects the `#VE` here, only the module does, emulating the
  architectural mechanism.
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

---

## TD Operation States

**OP_STATE** (operation state) is the current state of a TD in the TD operation state machine,
which controls what the TD can do. It is stored in [TDCS](tdx-structs.md#tdcs). The baseline
states are the following.

- `UNALLOCATED`: TDCS consists of multiple pages, added one at a time with `TDH.MNG.ADDCX`. The TD
  stays in this state until all required TDCS pages have been added. This state has no `OP_STATE`
  value in TDCS, because TDCS is not fully allocated yet.
- `UNINITIALIZED`: entered once all required TDCS pages have been added. TDCS is allocated but not
  yet initialized.
- `INITIALIZED`: entered after `TDH.MNG.INIT` initializes TDCS. The TD is being built: memory is
  added and measured, and vCPUs are created.
- `RUNNABLE`: entered after `TDH.MR.FINALIZE` completes the TD build. The TD is allowed to run.
  This is the same state whether the TD has been entered or never ran.

Live migration adds more `OP_STATE` values. On the source TD:

- `START_EXPORT`: `TDH.EXPORT.STATE.IMMUTABLE` was interrupted and has not been resumed yet. The TD
  can still run.
- `LIVE_EXPORT`: entered when `TDH.EXPORT.STATE.IMMUTABLE` completes. The TD can still run while
  memory is exported.
- `PAUSED_EXPORT`: entered by `TDH.EXPORT.PAUSE`. No vCPU may run. Mutable TD and vCPU state is
  exported with `TDH.EXPORT.STATE.TD` and `TDH.EXPORT.STATE.VP`, and memory export continues.
- `POST_EXPORT`: entered once the start token has been exported with `TDH.EXPORT.TRACK`. The TD
  cannot run anymore, but the destination may already be running and requesting remaining pages on
  demand (post-copy), so memory can still be exported.

**Note**: if TDX Connect is enabled for the source TD, on-demand page delivery is not allowed in
`POST_EXPORT`. Instead, all pages must have been exported in `LIVE_EXPORT` or `PAUSED_EXPORT`,
before `TDH.EXPORT.TRACK`.

The only way for the source TD `OP_STATE` to return to `RUNNABLE` is the abort path,
`TDH.EXPORT.ABORT`. In `POST_EXPORT`, the abort requires an abort token from the destination,
confirming that the destination TD will not run. There is no successful completion path: the source
TD cannot run after a successful migration, a security rule enforced by the TDX module. The host VMM
should tear down the source TD instead.

On the destination TD, the import starts from `UNINITIALIZED`, and `TDH.MNG.INIT` is never called:

- `START_IMPORT`: `TDH.IMPORT.STATE.IMMUTABLE` was interrupted and has not been resumed yet.
- `MEMORY_IMPORT`: entered when `TDH.IMPORT.STATE.IMMUTABLE` completes. TD memory can be imported.
- `STATE_IMPORT`: entered by `TDH.IMPORT.STATE.TD`. vCPU state can be imported, and memory import
  continues.
- `POST_IMPORT`: entered once the start token has been imported with `TDH.IMPORT.TRACK`. The
  destination TD cannot run yet, but memory import can continue into GPAs that have no page yet.
- `LIVE_IMPORT`: entered by `TDH.IMPORT.COMMIT`. The destination TD can run. Memory import can
  still continue in the background, possibly on demand (post-copy), triggered by an EPT violation
  when the TD accesses a page that has not been imported yet.
- `FAILED_IMPORT`: entered on an import error before `TDH.IMPORT.COMMIT`, or when the VMM aborts the
  import with `TDH.IMPORT.ABORT`. The TD will never run and can only be torn down. An import error
  in `LIVE_IMPORT` does not lead here. It ends the import session and moves the TD to `RUNNABLE`.

`TDH.IMPORT.END` ends the import session and moves the destination TD to `RUNNABLE`, either from
`LIVE_IMPORT` or directly from `POST_IMPORT`.

---

## SEPT Page States

This section describes L1 SEPT leaf and non-leaf entry states, which track private GPA pages. L2
SEPTs use different names and encodings.

### Leaf Entry States

- `FREE`: no page is associated with the GPA yet. This is the initial state of a leaf entry.
- `MAPPED`: the page is accessible to the TD, for example because the TD accepted it, or because
  the VMM added it at TD build time with `TDH.MEM.PAGE.ADD`.
- `PENDING`: the VMM added a private page to the GPA with `TDH.MEM.PAGE.AUG`, but the TD has not
  yet accepted it with `TDG.MEM.PAGE.ACCEPT`. The TD cannot access the page in this state.
  Depending on TD configuration, an access either raises a `#VE(PENDING)` to the TD or causes a TD
  exit to the VMM (see [Architectural #VE](#architectural-ve)). A `MAPPED` page also becomes
  `PENDING` if the TD releases it with `TDG.MEM.PAGE.RELEASE`.
- `BLOCKED`: the VMM blocked a `MAPPED` range with `TDH.MEM.RANGE.BLOCK`, so the CPU can no longer
  create new cached translations to it. An access causes a [TD exit](#vm-exit-and-td-exit).
- `PENDING_BLOCKED`: the VMM blocked a `PENDING` range with `TDH.MEM.RANGE.BLOCK`, so the CPU can
  no longer create new cached translations to it. Unlike plain `PENDING`, an access always causes
  a TD exit rather than a `#VE`.
- `REMOVE_IN_PROGRESS`: the `TDH.MEM.PAGE.REMOVE` seamcall was interrupted while reclaiming the
  page. This can only happen on
  [ACT platforms](../../2026/tdx-hw-layers.md#act-vs-non-act-mechanisms).

### Live Migration-Specific States

The following states track page export in the source TD. They belong to non-blocking export, which
uses the EPT Dirty bit to detect modified pages.

- `EXPORTED`: the page was exported and not modified since.
- `EXPORTED_MODIFIED`: the page was exported, but its content or attributes changed since. The
  `TDH.EXPORT.MEM` seamcall re-exports it and moves it back to the `EXPORTED` state.
- `PENDING_EXPORTED` and `PENDING_EXPORTED_MODIFIED`: same as above, but the page is `PENDING`.
  For example, if the TD releases an exported page with `TDG.MEM.PAGE.RELEASE`, the page becomes
  `PENDING_EXPORTED_MODIFIED`.
- `EXPORTED_BLOCKED`, `PENDING_EXPORTED_BLOCKED`, and `EXPORTED_REMOVED`: the VMM blocked or
  removed an exported page. `TDH.EXPORT.MEM` exports a CANCEL operation for the page, which moves
  it to the `BLOCKED`, `PENDING_BLOCKED`, or `FREE` state.

If the export is aborted, the VMM moves the remaining exported pages back to non-exported states
with `TDH.EXPORT.RESTORE`. If migration succeeds, the source TD is torn down.

The `REMOVED` state is used only on the destination TD.

- `REMOVED`: the page was removed while an import session is in progress, either by the VMM with
  `TDH.MEM.PAGE.REMOVE` or by an imported CANCEL operation. The entry is not present, like
  `FREE`, but the TDX module uses it to reject importing an older copy of the page to the GPA.

Write-blocking export is the other export mode. Instead of using the Dirty bit, the VMM blocks pages
for writing with `TDH.EXPORT.BLOCKW` before exporting them. A TD write to such a page causes an EPT
violation [TD exit](#vm-exit-and-td-exit), and the VMM is expected to unblock the page with
`TDH.EXPORT.UNBLOCKW`, and later block and re-export it. It uses the following states.

- `BLOCKEDW`: the page is blocked for writing, so the CPU can no longer create new cached
  translations for writes to it.
- `EXPORTED_BLOCKEDW`: the page was blocked for writing and then exported.
- `EXPORTED_DIRTY`: the page was exported, but it is not blocked for writing, so its content or
  attributes may have changed since.
- `EXPORTED_DIRTY_BLOCKEDW`: same as `EXPORTED_DIRTY`, but the page was blocked for writing again.
- `PENDING_BLOCKEDW`: the page is both `PENDING` and blocked for writing.
- `PENDING_EXPORTED_BLOCKEDW`: same as `EXPORTED_BLOCKEDW`, but the page is `PENDING`.
- `PENDING_EXPORTED_DIRTY`: same as `EXPORTED_DIRTY`, but the page is `PENDING`.
- `PENDING_EXPORTED_DIRTY_BLOCKEDW`: same as `EXPORTED_DIRTY_BLOCKEDW`, but the page is `PENDING`.

### TDX Connect-Specific States

MMIO pages are pages of the MMIO space of a TDI, such as its device registers, that the VMM maps
into the private GPA space of the TD with `TDH.MMIO.MAP`. The TD uses them to access the device via
CPU loads and stores, for example to read device status or to write commands to the device. The TDX
module tracks them in a separate MMIO metadata table (`MMIOMT`), not in the PAMT. The MMIOMT entry
records the TDI that owns the page.

**Note**: TDI DMA accesses ordinary `MAPPED` private pages of the TD, not MMIO pages.

The following TDX Connect states are leaf SEPT states and apply only to MMIO pages.

- `MMIO_MAPPED`: the MMIO page is accessible to the TD.
- `MMIO_BLOCKED`: the VMM blocked the MMIO page with `TDH.MMIO.BLOCK`, so the CPU can no longer
  create new cached translations to it.
- `MMIO_PENDING`: the MMIO page is pending acceptance by the TD using `TDG.TDI.MMIO.ACCEPT`.

### Non-Leaf Entry States

Non-leaf SEPT entries also have states.

- `NL_MAPPED`: the CPU can walk through the entry to the child SEPT page.
- `NL_BLOCKED`: the VMM blocked the range with `TDH.MEM.RANGE.BLOCK`, so the CPU can no longer
  create new cached translations through this entry.

### SEPT State Encoding

SEPT states are encoded in combinations of bits in an EPT entry. Some encoding bits also control
processor behavior. Others are ignored by the processor.

**Example 1: `MAPPED` and `BLOCKED`**

Here are selected SEPT fields and their corresponding EPT bits for the `MAPPED` and `BLOCKED`
states. Other EPT fields are omitted.

| SEPT field | EPT bit(s) | `MAPPED` | `BLOCKED` |
|------------|------------|----------|-----------|
| R/W/X      | 2:0        | `111`    | `000`     |
| Bits 56:53 | 56:53      | `0000`   | `0100`    |

The fields mean:

- R/W/X (bits 2:0): EPT read, write, and execute permissions.
- Bits 56:53: Ignored by the processor in EPT entries. The TDX module uses them for SEPT state
  encoding.

Because `BLOCKED` clears all R/W/X permissions, an access causes an EPT violation and a
[VM exit](#vm-exit-and-td-exit). This is exactly the behavior intended by the `BLOCKED` state.

**Example 2: `MAPPED` and `MMIO_MAPPED`**

Here are selected SEPT fields and their corresponding EPT bits for the `MAPPED` and `MMIO_MAPPED`
states. Other EPT fields are omitted.

| SEPT field        | EPT bit(s) | `MAPPED` | `MMIO_MAPPED` |
|-------------------|------------|----------|---------------|
| `IPAT`            | 6          | `1`      | `0`           |
| EPT memory type   | 5:3        | `110` WB | `000` UC      |
| Bits 56:53        | 56:53      | `0000`   | `0000`        |

Bit 6 in the EPT entry is called "Ignore PAT memory type" (`IPAT`).

- If `IPAT` is set, the processor ignores the TD PAT memory type and uses only the EPT memory type.
- If `IPAT` is clear, the processor combines the EPT memory type with the TD PAT memory type.

For definitions of WB, WC, and UC, see [Memory Types](x86-misc.md#memory-types). For background on
PAT and EPT memory types and how they interact, see
[Memory Cache Control](x86-misc.md#memory-cache-control).

Here are the final memory types for the `MAPPED` and `MMIO_MAPPED` states.

| State         | Effective Memory Type |
|---------------|-----------------------|
| `MAPPED`      | WB                    |
| `MMIO_MAPPED` | WC/UC                 |

Comments on the table:

- `MAPPED`: `IPAT=1` makes the effective type WB regardless of the TD PAT type.
- `MMIO_MAPPED`: `IPAT=0` combines EPT UC with the TD PAT type:
  - WC when the TD PAT type is WC.
  - UC for other TD PAT types.

**Note**: Bits 56:53 are `0000` in both states. The difference is in `IPAT` and the EPT memory type,
which determine the effective memory type.

---

## TLB Tracking

TLB tracking is a TDX module protocol that ensures there are no cached TLB entries for a given
TD private GPA range.

The problem TLB tracking solves is that when the TDX module changes a SEPT entry, a CPU that holds a
cached translation of that entry does not see the change. The CPU keeps using the cached translation
instead of walking the SEPT and reading the updated entry. So whenever a SEPT entry change must
become visible to every vCPU, the VMM performs TLB tracking.

The protocol works by flushing cached translations. Ideally it would flush only the affected GPAs,
but on Intel CPUs the `INVEPT` instruction flushes all TLB entries of a TD on one CPU at a time. So
the result of the TLB tracking protocol is that all TLB entries of the TD are flushed on every CPU
that could hold a cached translation for it.

TLB tracking is required before some seamcalls, for example:

- `TDH.MEM.PAGE.REMOVE` (reclaim a TD private page).
- `TDH.MEM.SEPT.REMOVE` (reclaim a SEPT page).
- The blocking variants of `TDH.MEM.PAGE.PROMOTE` (merge small pages into a larger page) and
  `TDH.MEM.PAGE.DEMOTE` (split a large page into smaller pages).

Without TLB tracking, a CPU could keep using a stale translation after the mapping has been changed
or removed, which would let the TD read or write memory it should no longer have access to.

### The TLB Tracking Protocol

The TLB tracking protocol has two steps:

1. The VMM calls `TDH.MEM.TRACK` once, on any CPU.
2. The VMM sends an IPI to each CPU that currently runs a vCPU of the TD, forcing a TD exit.

Before a vCPU of the TD runs on a CPU again, the TDX module flushes the TLB of the TD on that CPU,
preventing the vCPU from using a stale translation.

### Why the VMM Must Drive the Flush

Could the TDX module just flush the stale TLB entries itself, transparently, as part of the
operation that needs the flush, instead of making the VMM run a separate protocol?

No, because a seamcall runs only on the CPU the VMM invoked it from, not on every CPU where a stale
translation might exist. Only the VMM can reach those other CPUs, by sending them an IPI, which is
why the TLB tracking protocol exists as a distinct, VMM-visible step.

### Which CPUs Need the IPI

The TLB is per CPU, so a stale translation can sit on any CPU where a vCPU of the TD has run and
whose TLB has not been flushed since. But step 2 of the protocol only needs to reach CPUs that
currently run a vCPU of the TD, because the TDX module takes care of the other CPUs.

The TDX module has an `ASSOC_LPID` field in [TDVPS](tdx-structs.md#tdvps), which records the CPU a
vCPU is currently associated with, or a "no CPU" value if it is not associated with any CPU.
Seamcalls that operate on a vCPU (such as `TDH.VP.ENTER`) associate a non-associated vCPU with the
current CPU, and fail if the vCPU is associated with a different CPU.

To move a vCPU to a new CPU, the VMM must first invoke `TDH.VP.FLUSH` on the old CPU. This flushes
the TLB of the TD on the old CPU and resets `ASSOC_LPID` to the "no CPU" value.

So a CPU that does not currently run a vCPU of the TD is in one of two cases.

- **No vCPU of the TD is associated with it**. `TDH.VP.FLUSH` already flushed its TLB, so it holds
  no translation for the TD.
- **A vCPU of the TD is associated with it, but is not running**. The next `TDH.VP.ENTER` on this
  CPU flushes the TLB of the TD, as described in [Enforcing the Protocol](#enforcing-the-protocol).

### Enforcing the Protocol

If the VMM skips or mis-sequences the protocol, for example by letting the dependent operation run
before every vCPU has exited, a stale translation could let a CPU keep using an old mapping.
So the TDX module does not just trust the VMM to have run the protocol correctly. It verifies this
with the following fields.

- **TD epoch** (`TD_EPOCH`): a counter in [TDCS](tdx-structs.md#tdcs) for the entire TD.
  `TDH.MEM.TRACK` increments it.
- **vCPU epoch** (`VCPU_EPOCH`): enforces the flush when a vCPU enters after `TDH.MEM.TRACK`. It
  is a value in [TDVPS](tdx-structs.md#tdvps) that stores the `TD_EPOCH` value sampled at the most
  recent `TDH.VP.ENTER` of the vCPU. During `TDH.VP.ENTER`, the TDX module compares the current
  `TD_EPOCH` against `VCPU_EPOCH`.
  - If they are equal, `TDH.MEM.TRACK` was not called since the vCPU last entered, and no flush is
    needed.
  - If `TD_EPOCH` is greater, `TDH.MEM.TRACK` was called since the vCPU last entered. The TDX module
    flushes the TLB of the TD on the current CPU with the `INVEPT` instruction and updates
    `VCPU_EPOCH`.
  - Optimization: the flush is skipped if the vCPU was not associated with the current CPU before
    this `TDH.VP.ENTER`, because the CPU then holds no translation for the TD.
- **Running vCPU counters** (`REFCOUNT`): prove that every vCPU that was running at
  `TDH.MEM.TRACK` time has exited, which is what the IPIs of step 2 achieve. `REFCOUNT` is two
  counters in [TDCS](tdx-structs.md#tdcs), indexed by `TD_EPOCH % 2`.
  - `TDH.VP.ENTER` increments `REFCOUNT[TD_EPOCH % 2]`.
  - The [TD exit](#vm-exit-and-td-exit) decrements `REFCOUNT[VCPU_EPOCH % 2]`.
  - `TDH.MEM.TRACK` increments `TD_EPOCH`, so new entries start using the other counter. The old
    counter now belongs to the previous epoch, and it goes down to 0 as the vCPUs exit.
  - `TDH.MEM.TRACK` fails with `TDX_PREVIOUS_TLB_EPOCH_BUSY` if the previous epoch counter is not
    0, because the next epoch reuses that counter. This may be because the IPI is still in flight,
    or because the VMM did not send it. Either way, from the TDX module point of view, the previous
    round of the protocol has not completed.
- **Block epoch** (`BEPOCH`): proves that `TDH.MEM.TRACK` ran after the page was blocked. It is a
  field in the PAMT entry of the page.
  - Blocking the page (for example with `TDH.MEM.RANGE.BLOCK`) copies the current `TD_EPOCH` value
    into `BEPOCH`.
  - A dependent seamcall, such as `TDH.MEM.PAGE.REMOVE`, fails with `TDX_EPT_ENTRY_STATE_INCORRECT`
    if the page is not blocked.
  - It also checks that TLB tracking for the page is complete. Tracking is complete in two cases,
    otherwise the seamcall fails with `TDX_TLB_TRACKING_NOT_DONE`.
    - `TD_EPOCH` equals `BEPOCH + 1`, and the `REFCOUNT` of epoch `BEPOCH` is 0. This means
      `TDH.MEM.TRACK` ran after the page was blocked, and every vCPU that entered before that has
      since had a TD exit.
    - `TD_EPOCH` is greater than `BEPOCH + 1`. This means `TDH.MEM.TRACK` ran at least twice, and
      the second call could only succeed once the `REFCOUNT` of epoch `BEPOCH` dropped to 0.

### Example: Reclaiming a Private Page

The TD can ask the VMM to reclaim a private page, for example as part of the
[Private to Shared](#private-to-shared) page transition. Once reclaimed, the VMM can reuse the
physical page for other purposes.

The TDX module provides the `TDH.MEM.PAGE.REMOVE` seamcall, which moves the SEPT entry to `FREE`
and reclaims the physical page. If it did so directly on a `MAPPED` entry, a CPU might still hold a
cached translation to the old `MAPPED` entry, and could keep reading or writing the physical page
through that stale translation after the VMM has reused it elsewhere. This would let the TD access
memory it no longer owns.

So while the TD may be running, `TDH.MEM.PAGE.REMOVE` accepts only a blocked page with completed TLB
tracking, and the VMM goes through the following steps:

1. The VMM calls `TDH.MEM.RANGE.BLOCK` on the page's GPA, which marks the SEPT entry as `BLOCKED`.
   The entry still exists, but a CPU that walks the SEPT for this GPA now gets an EPT violation
   instead of a valid translation, so it cannot create a new cached translation to the page.
2. The VMM runs the TLB tracking protocol, which flushes the cached translations that already exist,
   so the `BLOCKED` state becomes visible to every CPU.
3. The VMM calls `TDH.MEM.PAGE.REMOVE`, which verifies that TLB tracking is complete, moves the SEPT
   entry to `FREE`, and reclaims the physical page.

In other words, the VMM first closes off new cached translations by marking the SEPT entry
`BLOCKED`, then flushes the existing cached translations by running the TLB tracking protocol, and
only then makes the final SEPT entry change to `FREE`.

---

## Page State Transitions

This section describes transitions between private and shared mappings for a TD GPA. The `SHARED`
bit selects the mapping: when clear, the GPA maps through the SEPT; when set, it maps through the
Shared EPT. These are mapping types, not SEPT entry states such as `PENDING`, `MAPPED`, and `FREE`.
Refer to [SEPT page states](#sept-page-states) for details.

### Private to Shared

1. The TD sets the `SHARED` bit of the GPA in its own page table entries that map the page. It then
   has two options to make the VMM aware of the conversion (not required by TDX specifications):
   - Call `TDG.VP.VMCALL<MapGPA>`, a GHCI-defined VMCALL sub-function that tells the VMM about the
     conversion and lets it prepare the Shared EPT mapping in advance, avoiding an EPT violation on
     the next access. This is the option the Linux kernel uses.
   - Just access the GPA. Because the GPA now has the `SHARED` bit set, the CPU walks the Shared
     EPT instead of the SEPT, and the Shared EPT has no mapping for this GPA yet, so this
     causes an EPT violation TD exit.
2. The VMM calls `TDH.MEM.RANGE.BLOCK` on the private GPA, so the CPU can no longer create new
   cached translations to it.
3. The VMM runs the [TLB tracking protocol](#the-tlb-tracking-protocol), which flushes the TLB
   entries of the TD, so no CPU can still hold a cached translation for the private GPA.
4. The VMM calls `TDH.MEM.PAGE.REMOVE` to clear the SEPT entry and reclaim the physical page.
5. The VMM maps the GPA in the Shared EPT to a physical page (not required by TDX specifications).
6. The TD can now access the GPA as a shared page, backed by VMM-managed shared memory.

### Shared to Private

1. The TD decides that it needs the shared GPA as private and notifies the VMM (not required by TDX
   specifications, but since it is the TD that controls its GPAs, it must initiate the transition
   and be ready to accept a private page).
   - Linux kernel uses the GHCI-defined VMCALL sub-function `TDG.VP.VMCALL<MapGPA>` to notify the
     VMM about the transition from shared to private for the GPA, the same sub-function used for
     the "Private to Shared" direction, but with the `SHARED` bit of the GPA cleared instead of set.
2. The VMM unmaps the GPA from the Shared EPT and can reclaim the physical page (not required by
   TDX specifications).
3. If required, the VMM adds SEPT pages with `TDH.MEM.SEPT.ADD` to build the SEPT mapping path to
   the GPA, for example when the GPA falls into a region with no existing SEPT mapping.
   - In Linux kernel, KVM maintains a copy of SEPT called the **mirror page table**. So Linux knows
     in advance whether new SEPT pages need to be added.
4. The VMM calls `TDH.MEM.PAGE.AUG` to map the GPA to a private page, and the corresponding SEPT
   entry is set to the `PENDING` state.
   - In Linux kernel terms, `PENDING` pages are called "unaccepted" pages.
   - The HPA of the private page is an input argument of `TDH.MEM.PAGE.AUG`, so the VMM chooses
     which physical page to donate.
   - In Linux, KVM keeps track of private TD memory in a separate pool (`guest_memfd`), so the
     physical page will not be the same one that was previously used for the shared GPA.
5. The VMM re-enters the TD.
6. The TD calls `TDG.MEM.PAGE.ACCEPT` for the GPA. The TDX module zeroes the page and marks the SEPT
   entry as `MAPPED`, completing the transition from shared to private.

### Pending to Private

The TD can just call `TDG.MEM.PAGE.ACCEPT` to transition a page from `PENDING` to private
(`MAPPED`). This is covered in step 6 of the [Shared to Private](#shared-to-private) transition.

### Private to Pending

The TD can just call `TDG.MEM.PAGE.RELEASE` to transition a private (`MAPPED`) page back to
`PENDING`.

**Note**: support for the `TDG.MEM.PAGE.RELEASE` tdcall is optional, enumerated by `PAGE_RELEASE` in
[TDX_FEATURES0](tdx-structs.md#tdx_features0),
and the VMM must enable it per TD at TD initialization time (`TDH.MNG.INIT`) via `PAGE_RELEASE` in
[CONFIG_FLAGS](tdx-structs.md#config_flags).

### Private or Pending to Unmapped

The VMM removes a SEPT entry with `TDH.MEM.PAGE.REMOVE` for a leaf entry (moving it to `FREE`) or
`TDH.MEM.SEPT.REMOVE` for a non-leaf one whose child entries are all `FREE`. Before that, it must
block the range and run the [TLB tracking protocol](#the-tlb-tracking-protocol). Whether the VMM
needs the TD to cooperate depends on whether a TDI is attached to the TD.

- **No TDI attached**: the VMM can remove a `MAPPED` or `PENDING` entry at any time, without asking
  the TD. This happens as part of [Private to Shared](#private-to-shared) (steps 2 to 4), and also
  on its own, for example when the VMM reclaims memory the TD gave up with `TDG.MEM.PAGE.RELEASE`.
- **TDX Connect with a TDI attached**: `TDH.MEM.RANGE.BLOCK` accepts only `PENDING` pages, so the
  TD has to release a `MAPPED` page with `TDG.MEM.PAGE.RELEASE` before the VMM can remove it. This
  prevents a page from going away while the TDI may still DMA to it, which would silently drop
  data.

### Shared to Unmapped

The VMM can just unmap any shared GPA at any time. The VMM exclusively owns the Shared EPT.
