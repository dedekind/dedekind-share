# TDX Live Migration

- **Author**: Artem Bityutskiy
- **Version**: 1.0
- **Date**: 2026-10-03
- **Last updated**: 2026-11-04

**Disclaimer**: This document describes my current understanding of TDX guest live migration, but
may contain errors or omissions. This is not an official Intel document and should not be treated as
such. For full and authoritative information, refer to the official documentation.

---

## Table of Contents

- [TDX Live Migration](#tdx-live-migration)
  - [Table of Contents](#table-of-contents)
  - [Context](#context)
  - [The Spiral Structure](#the-spiral-structure)
  - [Migration Overview](#migration-overview)
    - [Security Properties](#security-properties)
    - [The Two Sessions](#the-two-sessions)
    - [Simple Overview](#simple-overview)
    - [More Detailed Overview](#more-detailed-overview)
    - [Device State Migration](#device-state-migration)
  - [Setup Session](#setup-session-1)
    - [How the Setup Session Works](#how-the-setup-session-works)
    - [Migration Policy](#migration-policy)
    - [Setup Session Abort](#setup-session-abort)
    - [Who Runs the Setup Session](#who-runs-the-setup-session)
  - [Migration Session](#migration-session-1)
    - [Export and Import Error Handling](#export-and-import-error-handling)
    - [Step 1: Immutable State Transfer](#step-1-immutable-state-transfer)
    - [Step 2: Scanning for Export Candidates](#step-2-scanning-for-export-candidates)
    - [Step 3: TLB Tracking Between Scan and Export](#step-3-tlb-tracking-between-scan-and-export)
    - [Step 4: Memory Pages Export](#step-4-memory-pages-export)
    - [Step 5: Memory Pages Import](#step-5-memory-pages-import)
    - [Step 6: Migration Epoch Tracking](#step-6-migration-epoch-tracking)
    - [Step 7: Configure the Comprehensive Scan](#step-7-configure-the-comprehensive-scan)
    - [Step 8: Optional Pre-Scan](#step-8-optional-pre-scan)
    - [Step 9: Pause the Source TD](#step-9-pause-the-source-td)
    - [Step 10: Transfer TD Mutable State](#step-10-transfer-td-mutable-state)
    - [Step 11: Transfer vCPU State](#step-11-transfer-vcpu-state)
    - [Step 12: Transfer Final Memory State](#step-12-transfer-final-memory-state)
    - [Step 13: The Start Token](#step-13-the-start-token)
    - [Step 14: Complete the Migration](#step-14-complete-the-migration)
  - [Migration Abort](#migration-abort)
    - [Before the Immutable State Export](#before-the-immutable-state-export)
    - [Before the Start Token](#before-the-start-token)
    - [After the Start Token](#after-the-start-token)
  - [Configuration](#configuration)

---

## Context

It is important to understand two things about this document:

- This is not a specification. It is a high level overview of the TDX live migration model. Its
  goal is to help software engineers build a mental model of the migration process and understand
  the design choices behind it. The document starts with the big picture and adds detail later,
  rather than reading like a reference manual.
- This is not about Linux TDX guest live migration support. It is about the live migration model
  offered by the Intel TDX module and is written in an OS-agnostic way. But because my practical
  context is Linux, it includes some Linux-specific notes and focuses on the aspects relevant to our
  initial TDX live migration Linux implementation.

The document assumes that the reader is familiar with live migration in general, for example, the
difference between pre-copy and post-copy migration. If it helps, here is my study of
[QEMU/KVM live migration](./qemu-live-migration.md) for traditional VMs.

This document is not a complete TDX live migration reference. For example, the following aspects are
not covered yet:

- TDX post-copy live migration.
- Migration of partitioned TDs.
- Write-protect dirty tracking (TDX term: Write-Blocking Based Memory Export).

This document is evolving over time, and I increment the version number after significant updates.

**Terminology Notes**:

- I use the term "seamcall" in lowercase to refer to a TDX module ABI leaf invoked via the
  `SEAMCALL` CPU instruction. So `SEAMCALL` is the instruction, and seamcall is what TDX specs
  refer to as "leaves". For example, `TDH.MIG.SETUP` is a seamcall.
- Similarly, I use the term "tdcall" in lowercase to refer to a TDX module ABI leaf invoked via the
  `TDCALL` CPU instruction, which includes all `TDG.*` leaves. So `TDCALL` is the instruction,
  and tdcall is what TDX specs refer to as "leaves".

---

## The Spiral Structure

This document uses a spiral structure: it revisits the same concepts several times, starting high
level and going deeper on each pass. This approach is meant to help readers who are not TDX experts.

The migration flow is covered in five passes, each going deeper than the last:

1. [The Two Sessions](#the-two-sessions): The big picture, illustrated with one diagram.
2. [Simple Overview](#simple-overview): The same flow again, with a short step list.
3. [More Detailed Overview](#more-detailed-overview): The same flow again, mapped to the actual
   TDX seamcalls.
4. [Setup Session](#setup-session-1): The full story for the setup session.
5. [Migration Session](#migration-session-1): The full story for the migration session, broken
   into 14 steps.

The most detailed parts live in separate supporting documents. They are not standalone articles, and
they are simpler and less detailed than the official TDX specifications. The main ones are:

- [TDX Live Migration: Seamcalls](../misc/intel-cpu/tdx-migration-seamcalls.md): Reference for the
  live migration seamcalls.
- [TDX Live Migration: Data Structures](../misc/intel-cpu/tdx-migration-structs.md): Migration
  bundles, streams, and other live migration data structures.
- [TDX Live Migration: Miscellaneous](../misc/intel-cpu/tdx-migration-misc.md): Migration policy,
  migration epochs, configuration, and other live migration topics.
- [TDX Module: Miscellaneous](../misc/intel-cpu/tdx-misc.md): General TDX module concepts that
  migration builds on, such as TLB tracking, SEPT page states, and TD operation states.
- [TDX Module: Data Structures](../misc/intel-cpu/tdx-structs.md): General TDX module data
  structures, such as TDCS and TDVPS.

The links in this article lead to these documents. They are written as references and are more
detailed than this article. Following a link is optional.

---

## Migration Overview

With a traditional VM, the VMM has full access to the VM memory and vCPU state. So live migration is
implemented entirely by the VMM, which reads VM state, copies it to the destination host, and
restores it in the destination VM.

With a confidential VM (CoCo VM), the VMM does not have direct access to the VM private memory and
vCPU state. Therefore, the live migration process requires an intermediary that can fetch the guest
state in encrypted form on the source host and restore it on the destination host.

In Intel TDX, the TDX module is that intermediary: it provides seamcalls to export guest state in
encrypted form on the source host and import it on the destination host. The role of the VMM is to
transport the encrypted blobs between the source and destination hosts.

### Security Properties

The TDX module must enforce the following security properties during migration. They explain many of
the special mechanisms that TDX live migration has compared to a traditional VM.

- **Confidentiality of TD state**: The VMM does not have access to the private memory or vCPU state
  of the TD. The general migration principle is that the TDX module exports memory and other TD
  state data in encrypted form on the source, the VMM delivers the data to the destination, and the
  TDX module imports and decrypts it into the destination TD.
- **Non-clonability of TDs**: Only one instance of the TD may run at a time. Either the source or
  the destination may run, but never both. Cloning a TD is not allowed.
- **Consistency of TD state**: When migration completes, the destination must have the same memory
  and vCPU state as the source. It must not end up with a partial or mixed state. Otherwise, the
  migration mechanism itself could be used to tamper with TD state, for example by replaying an
  older version of a page, or by adding or removing pages that do not match the source.

#### Why Non-Clonability?

Cloning a TD would give the VMM a way to attack confidential VMs. Here are two example attack
approaches:

- With a clone, the attacker (e.g., a rogue cloud service provider) could attest one copy running
  up-to-date, patched software, while continuing to run the other copy with older, vulnerable
  software that the update would have fixed.
- The TDX attestation process assures the TD owner that it is communicating with one specific TD
  instance. If cloning were allowed, multiple instances could exist at once, and that assurance
  would no longer hold.

### The Two Sessions

The TDX migration model splits the migration process into two stages: setup session and migration
session. Here is a brief overview of each stage, with more details in later sections.

```text
+------------------------+                +------------------------+
|    Source TDX module   |                | Destination TDX module |
| +--------------------+ |    Messages    | +--------------------+ |
| |   Setup Session    | |<-------------->| |   Setup Session    | |
| +--------------------+ |                | +--------------------+ |
|           |            |                |           |            |
|           v            |                |           v            |
| +--------------------+ | TD State Data  | +--------------------+ |
| |  Migration Session | |--------------->| |  Migration Session | |
| +--------------------+ |                | +--------------------+ |
+------------------------+                +------------------------+
```

#### Setup Session

During the setup session, the source and destination TDX modules exchange encrypted messages through
the VMM. The VMM also delivers attestation quotes and migration policy inputs when requested. Here
is how the process looks in a simplified form.

- The source and destination TDX modules run the `TDH.MIG.SETUP` seamcall repeatedly, exchanging
  encrypted messages through the VMM.
- The process repeats until the TDX module reports success or failure.

The setup session has the following goals:

- Establish a secure communication channel between the source and destination TDX modules.
- Run mutual attestation between the source and destination.
- Generate and exchange migration keys.
- Load the migration policy.

#### Migration Session

The migration session can start only after the setup session completes on both hosts. Here is how
the process looks in a simplified form.

- The source exports TD state as encrypted blobs. Different seamcalls export different parts of the
  TD state, such as memory pages or vCPU state.
- The VMM transports the exported state to the destination.
- The destination imports the state into the destination TD.
- Eventually, the source TD pauses, and the destination TD takes over, ending the migration session.

**Terminology Note**: In this document, TD state means all the state that the TDX module migrates:
private memory, vCPU state, and the TD level state, such as the immutable state and the TD mutable
state. Device state migration is outside the TDX module migration model and is handled by the VMM,
as described later in [Device State Migration](#device-state-migration).

### Simple Overview

The following diagram shows the overall migration flow:

```text
Run the migration setup session
  |
  v
Transfer immutable state
  |
  v
Copy dirty memory pages while the TD runs <--------+
  |                                                |
  +-- iterate until convergence criteria are met --+
  |
  v
Pause the source TD and copy the remaining state
  |
  v
Start the destination TD
```

The setup session runs first. The migration session then transfers the TD state, in the steps shown
above:

1. **Immutable state**: The source exports the immutable state and initializes the destination TD.
   Immutable state is data fixed when the TD was finalized, such as the vCPU count and topology.
2. **Iterative memory copy**: While the source TD keeps running, the source and destination copy
   memory in rounds. The first round exports all mapped private pages. Later rounds export pages
   that changed since an earlier export and import them on the destination. The rounds repeat until
   few enough pages change per round to meet the convergence criteria.
3. **Stop and copy**: The source TD is paused, the VMM transfers the remaining TD state (such as
   vCPU state) and the final dirty pages, and the destination TD starts.

**Terminology Note**: Software engineers call any page that must be copied to the destination a
"dirty page", since in traditional VMs only content changes trigger a copy. TDX also copies pages
whose SEPT attributes changed but content did not, and this document calls them "dirty pages" above,
for familiarity. From here on, the document uses the term "export candidate" for this broader
concept and reserves "dirty" and "dirty bit" for the hardware Dirty bit (part of the Accessed/Dirty
bits Intel CPU feature).

### More Detailed Overview

The following diagram maps the flow to the TDX seamcalls used by the source and destination:

```text
    Source host                                                Destination host
    ===========                                                ================
    TDH.MIG.SETUP (repeated) <-- crypto keys, attestation --> TDH.MIG.SETUP (repeated)
          |                                                          |
     TDH.EXPORT.STATE.IMMUTABLE - immutable state --> TDH.IMPORT.STATE.IMMUTABLE
          |                                                          |
+--> TDH.MEM.SCAN.RANGE(DSCAN)                                       |
|    TDH.MEM.TRACK + IPIs                                            |
|    TDH.EXPORT.MEM    ------------ memory data -------------> TDH.IMPORT.MEM
+<-- TDH.EXPORT.TRACK  ------------ epoch token -------------> TDH.IMPORT.TRACK
          |                                                          |
    TDH.MEM.SCAN.CONFIG                                              |
    TDH.MEM.SCAN.COMP(PRECLEAR)  (optional)                          |
    TDH.MEM.TRACK + IPIs         (optional)                          |
    TDH.MEM.SCAN.COMP(PRECHECK)  (optional)                          |
          |                                                          |
    TDH.EXPORT.PAUSE                                                 |
          |                                                          |
    TDH.EXPORT.STATE.TD ----------- TD mutable state -----> TDH.IMPORT.STATE.TD
          |                                                          |
    TDH.EXPORT.STATE.VP ---------- vCPU state ------------> TDH.IMPORT.STATE.VP
          |                                                          |
    TDH.MEM.SCAN.COMP(DCHECK)                                        |
    TDH.EXPORT.MEM    ------------ final memory data -------> TDH.IMPORT.MEM
    TDH.EXPORT.TRACK  ------------ start token -------------> TDH.IMPORT.TRACK
                                                                     |
                                                               TDH.IMPORT.END
```

The source and destination go through the following stages.

#### Setup

The source and destination TDX modules invoke the `TDH.MIG.SETUP` seamcall iteratively. On the first
invocation, the VMM also provides the migration policy, described later in
[Migration Policy](#migration-policy). Each invocation returns an exit code that tells the VMM what
to do next, for example:

- Send an encrypted message to the peer.
- Deliver an encrypted message from the peer.
- Deliver an attestation quote from a quoting service.

Together, the invocations establish a secure channel, perform mutual attestation, and generate the
migration encryption keys. The setup session completes when the TDX module returns success.

#### Immutable State Transfer

The source invokes the `TDH.EXPORT.STATE.IMMUTABLE` seamcall to export the immutable state, and the
source VMM delivers it to the destination VMM.

Before importing the immutable state, the destination VMM runs a few preparation steps:

- Create the destination TD skeleton.
- Configure the
  [migration streams](../misc/intel-cpu/tdx-migration-structs.md#migration-stream),
  the channels for parallel state transfer. The immutable state transfer always uses a single
  stream, but additional streams may be configured for the parallel memory and vCPU state transfer
  in later steps.

The destination VMM then invokes the `TDH.IMPORT.STATE.IMMUTABLE` seamcall to import the immutable
state into the destination TD.

#### Iterative Memory Copy

While the source TD continues running, the source and destination run iterative memory copy rounds:

- The source invokes the `TDH.MEM.SCAN.RANGE` seamcall to find export candidate pages (see
  [Step 2: Scanning for Export Candidates](#step-2-scanning-for-export-candidates)).
- The source invokes the `TDH.MEM.TRACK` seamcall and sends IPIs to the vCPUs of the TD, running
  the TLB tracking protocol to ensure scanning correctness (see
  [Step 3: TLB Tracking Between Scan and Export](#step-3-tlb-tracking-between-scan-and-export)).
- The source invokes the `TDH.EXPORT.MEM` seamcall to export private memory pages, which the VMM
  sends to the destination.
- The destination invokes the `TDH.IMPORT.MEM` seamcall to import the received memory pages.
- The source invokes the `TDH.EXPORT.TRACK` seamcall to generate an epoch token, a marker that
  separates memory copy rounds (see
  [Step 6: Migration Epoch Tracking](#step-6-migration-epoch-tracking)). The VMM sends it to the
  destination, which invokes the `TDH.IMPORT.TRACK` seamcall to consume the token. This verifies
  that all data exported from the source was imported on the destination.

The rounds continue until the number of pages that change between rounds is small enough to meet the
convergence criteria.

#### Final Scan Preparation

This stage gets everything ready for the final round of scanning for export candidates, which TDX
specifications call the comprehensive scan. The comprehensive scan itself runs after the source TD
is paused, which is described in the next section. This section covers the preparations that happen
before that, while the source TD is still running.

- The VMM configures the comprehensive scan with the `TDH.MEM.SCAN.CONFIG` seamcall (see
  [Step 7: Configure the Comprehensive Scan](#step-7-configure-the-comprehensive-scan)).
- The VMM may optionally run the pre-scan to shorten the comprehensive scan, and therefore the
  downtime (see [Step 8: Optional Pre-Scan](#step-8-optional-pre-scan)).

#### Stop and Copy

Once the memory copy rounds converge, the VMM runs the following steps.

- The VMM pauses the source TD by invoking the `TDH.EXPORT.PAUSE` seamcall, which starts the
  downtime period, the time when the TD runs on neither host.
- The source invokes the `TDH.EXPORT.STATE.TD` seamcall to export TD mutable state and the
  `TDH.EXPORT.STATE.VP` seamcall to export each vCPU state.
- The source performs the final comprehensive scan (`DCHECK` operation of the `TDH.MEM.SCAN.COMP`
  seamcall) and invokes the `TDH.EXPORT.MEM` seamcall to export the remaining candidate pages.
- The VMM delivers the exported TD mutable state, vCPU state, and remaining candidate pages to the
  destination.
- The destination invokes the `TDH.IMPORT.STATE.TD` seamcall to import TD mutable state and the
  `TDH.IMPORT.STATE.VP` seamcall to import each vCPU state.
- The destination invokes the `TDH.IMPORT.MEM` seamcall to import the remaining candidate pages.

**Note**: TDX specifications use the term "blackout period" for what this document calls the
downtime period.

At this point the destination TD state is identical to the source TD state at the moment the source
TD was paused, and the destination TD is ready to run, which is implemented through the start token
mechanism (see [Step 13: The Start Token](#step-13-the-start-token)).

- The source invokes the `TDH.EXPORT.TRACK` seamcall again, with an input argument that selects the
  start token instead of an epoch token. The VMM sends the start token to the destination.
- The start token acts as the right to run the TD. Once the source exports it, the source TD can no
  longer run (`TDH.VP.ENTER` fails). This enforces the non-clonability security property described
  in [Security Properties](#security-properties).
- The destination imports the start token by invoking the `TDH.IMPORT.TRACK` seamcall, which
  verifies that all state exported in this round was imported. This is the last check before the
  destination TD can start.
- Finally, the destination invokes the `TDH.IMPORT.END` seamcall, which starts the destination TD
  and ends the migration session.

### Device State Migration

The TDX module migration model covers the TD state: its private memory, vCPU state, and TD level
state. It does not cover shared pages or device state, which the VMM handles separately.

#### Virtualized Devices

Virtualized devices use shared pages, so their migration is outside the TDX module migration model
and is handled by the VMM. For example, in a traditional QEMU/KVM virtual machine, QEMU migrates the
state of virtualized virtio devices, such as virtio-net and virtio-blk. The same model would be used
for a TDX guest.

#### TDX Connect Devices

TDX Connect lets a PCIe or CXL function of a physical device be assigned directly to a TD and access
private memory of the TD over a cryptographically protected link. TDX specifications call such an
assigned function a TDI (Trusted Device Interface). A TDI is a physical function (PF) or a virtual
function (VF), so one physical device can host several TDIs.

Unlike an emulated device, a TDI is inside the trust boundary of the TD: its MMIO space is mapped
into TD private memory (see [SEPT MMIO States](../misc/intel-cpu/tdx-misc.md#sept-mmio-states)), and
it can DMA directly into private memory.

In the current TDX module model, every TDI assigned to the TD must be unassigned before stop and
copy begins, no new TDI can be assigned while migration is in progress, and a TDI can be reassigned
only after the destination TD is running.

Unassigning a TDI tears down its entire footprint in the TD private space: its MMIO regions are
unmapped from the SEPT, and its trusted DMA mappings are removed. So there is no state related to
the TDI left to migrate. Ordinary memory that the TDI driver used, as opposed to its MMIO space, is
not affected: the TDX module treats it like any other private memory and migrates it normally.

Unassigning and reassigning a TDI cannot be done without TD involvement, so the VMM cannot do it
transparently for the TD. From the TD point of view, this looks like a hot-unplug on the source and
a hot-plug on the destination, so unlike private memory migration, this part of migration is visible
to the guest.

As I understand it, one reason a TDI cannot be reassigned transparently is that the TD establishes
trust directly with the specific physical device function behind the TDI, not through the VMM. The
destination host has a different physical device, with different cryptographic keys, so the TD must
explicitly re-establish that trust after migration. Preserving this trust across migration appears
to be a hard problem, and unassigning then reassigning the TDI is the current compromise solution.

---

## Setup Session

This section is the detailed discussion of the setup session, which is the first stage of the TDX
guest migration process, discussed in [The Two Sessions](#the-two-sessions).

### How the Setup Session Works

To start the setup session, the VMM executes the `TDH.MIG.SETUP` seamcall on both source and
destination hosts.

`TDH.MIG.SETUP` is invoked multiple times on both hosts. The TDX module keeps the setup session
state and uses exit codes to tell the VMM what to do next. For example:

- `TDX_MIG_SETUP_MSG_EXCHANGE`: The output buffer contains an encrypted message. The VMM sends it
  to the peer, then invokes `TDH.MIG.SETUP` again with the peer response in the input buffer.
- `TDX_MIG_SETUP_GET_QUOTE`: The output buffer contains the platform report. The VMM obtains an
  attestation quote for it, then invokes `TDH.MIG.SETUP` again with the quote in the input buffer.
  For example, the VMM can acquire the quote using the
  [Quoting Service](../misc/intel-cpu/tdx-misc.md#quoting-service).
- `TDX_SUCCESS`: The setup session is complete.

In other words, `TDH.MIG.SETUP` behaves like a state machine driven by TDX module exit codes, and
the VMM follows the requested action on each iteration.

Each `TDH.MIG.SETUP` seamcall invocation follows the same pattern:

1. The VMM executes the seamcall for the source TD or the destination TD.
2. The TDX module reads the optional input buffer, updates its internal state, and may produce an
   output buffer.
3. The exit code tells the VMM what to do before the next invocation.

Together, those invocations accomplish the following:

1. Receive and verify the TD or platform migration policy from the VMM.
2. Establish a secure channel between the source and destination TDX modules.
3. TDX modules attest each other to verify the peer is a genuine TDX platform.
4. Generate and exchange migration session keys: forward channel keys (for encrypted memory, TD
   state, and vCPU state) and backward channel keys (for control messages).

The messages exchanged between peer TDX modules are opaque blobs to the VMM. The VMM knows the
message type and size but does not parse or modify the contents. The VMM has three roles:

- **Transport**: Delivers messages between peer TDX modules.
- **Quote delivery**: Provides an attestation quote when the TDX module requests it with
  `TDX_MIG_SETUP_GET_QUOTE`.
- **Policy delivery**: Optionally provides a TD migration policy, a platform migration policy, or
  both. Unlike the other two roles, the policy is not an opaque blob: it is a JSON file that the
  VMM received from the platform owner or tenant. The VMM can inspect it, but is not required to.
  The policy is described below.

The setup session ends when `TDH.MIG.SETUP` returns `TDX_SUCCESS` or an unrecoverable error. For
more details, see [TDH.MIG.SETUP](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhmigsetup).

### Migration Policy

The VMM provides migration policies to `TDH.MIG.SETUP`. A migration policy is a JSON document that
defines rules, such as where a TD is allowed to migrate. There are two migration policy kinds:

- **TD migration policy**: Controlled by the TD owner, for example a tenant of a cloud service
  provider (CSP). It reflects what the TD owner wants, for example blocking migration to an older
  TDX module or CPU microcode version.
- **Platform migration policy**: Controlled by the platform owner, for example the CSP. It is
  optional and adds restrictions on top of the TD migration policy. Unlike the TD migration policy,
  it is not included in TD attestation quotes.

The VMM supplies the policies in the following way:

- On the source, the first `TDH.MIG.SETUP` invocation must include the TD migration policy. It may
  also include the platform migration policy.
- On the destination, only the platform migration policy may be provided.

See the [Migration Policy](../misc/intel-cpu/tdx-migration-misc.md#migration-policy) section for the
policy hash check and more details.

### Setup Session Abort

The VMM can abort a setup session in progress with the `TDH.MIG.SETUP.ABORT` seamcall, for example
if the destination becomes unreachable or the VMM decides to cancel the migration. This releases the
TDX module resources held for the ongoing setup session, so a new setup session can start later. For
details, see [TDH.MIG.SETUP.ABORT](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhmigsetupabort).

### Who Runs the Setup Session

The setup session is run by the **Migration Setup Service (MSS)**, which in current TDX module
design is implemented as a special hidden TD that implements the `TDH.MIG.SETUP` seamcall. See the
[Migration Setup Service](../misc/intel-cpu/tdx-migration-misc.md#migration-setup-service) section
for details.

**Note**: MSS is the newer design, and this document focuses on it. Its older, still supported
counterpart is the Migration TD (MigTD), a TD the VMM manages directly.

---

## Migration Session

This section is the detailed discussion of the migration session, which is the second stage of the
TDX guest migration process, discussed in [The Two Sessions](#the-two-sessions). It runs after the
setup session completes and transfers the TD state from the source to the destination. This section
splits the migration session into numbered steps, which follow the stages in
[More Detailed Overview](#more-detailed-overview).

- **Step 1** is the immutable state transfer.
- **Steps 2-6** are the iterative memory copy rounds. Each round scans for export candidates,
  exports them on the source, and imports them on the destination, then ends with an epoch token
  (Step 6). The loop runs until few enough pages change per round.
- **Steps 7-8** are the final scan preparation, done while the source TD still runs.
- **Steps 9-14** are stop and copy:
  - **Step 9** pauses the source TD and starts the downtime period.
  - **Steps 10-12** transfer the TD mutable state, the vCPU state, and the final memory.
  - **Step 13** exports and imports the start token, which transfers the right to run the TD from
    the source to the destination.
  - **Step 14** completes the migration.

Before going through the steps, two concepts need to be introduced:

- A **migration bundle** is the unit of data produced by migration export seamcalls and consumed by
  matching import seamcalls. For example, `TDH.EXPORT.STATE.IMMUTABLE` produces a bundle, and
  `TDH.IMPORT.STATE.IMMUTABLE` consumes it. Different seamcalls produce and consume different kinds
  of bundles, for example covering immutable state or memory pages. See
  [Migration Bundle](../misc/intel-cpu/tdx-migration-structs.md#migration-bundle)
  for more details.
- A **migration stream** is a TDX module construct, identified by a stream ID, that the TDX module
  uses to enforce ordering of migration bundles: on a given stream, bundles are exported and
  imported one at a time, in order. Using multiple streams lets the VMM transfer TD state in
  parallel. The VMM creates streams with the
  [TDH.MIG.STREAM.CREATE](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhmigstreamcreate) seamcall
  before the migration session starts. See the
  [Migration Stream](../misc/intel-cpu/tdx-migration-structs.md#migration-stream)
  section for more details.

### Export and Import Error Handling

Most of the steps below involve an export seamcall on the source and a matching import seamcall on
the destination. Across all of them, export and import errors follow the same general pattern.

- Export errors are not fatal.
- Most import errors are fatal and move the destination TD to the
  [FAILED_IMPORT](../misc/intel-cpu/tdx-misc.md#td-operation-states) state, which means that the
  destination TD can never run and must be torn down. Some errors are recoverable, so the VMM can
  fix the cause and retry. The VMM can use the `FATAL` bit of the
  [completion status](../misc/intel-cpu/tdx-misc.md#status-codes) to distinguish fatal and
  recoverable errors.

---

### Step 1: Immutable State Transfer

The migration session begins when the VMM executes the
[TDH.EXPORT.STATE.IMMUTABLE](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhexportstateimmutable)
seamcall on the source TD to export immutable state. The resulting bundle is transferred to the
destination host and imported using the
[TDH.IMPORT.STATE.IMMUTABLE](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhimportstateimmutable)
seamcall.

Immutable state is the TD configuration fixed when the TD measurement is finalized with the
`TDH.MR.FINALIZE` seamcall. It includes information such as vCPU count and topology,
[ATTRIBUTES](../misc/intel-cpu/tdx-structs.md#attributes),
[XFAM](../misc/intel-cpu/tdx-structs.md#xfam), and
[CPUID_CONFIG](../misc/intel-cpu/tdx-structs.md#cpuid_config).

For more details, see the
[Immutable State](../misc/intel-cpu/tdx-migration-structs.md#immutable-state) section.

Immutable state transfer has two goals:

1. Transfer the TD state that was fixed when the source TD was finalized.
2. Initialize the destination TD and verify the source configuration is compatible with the
   destination platform.

The immutable bundle is transferred on
[migration stream 0](../misc/intel-cpu/tdx-migration-structs.md#migration-stream).

**Note 1**: The destination TD must not have been initialized by `TDH.MNG.INIT` before the immutable
state import. `TDH.IMPORT.STATE.IMMUTABLE` replaces that step: it initializes the destination TD
from the immutable state bundle of the source and checks imported fields against destination
platform capabilities and TDX module configuration.

**Note 2**: Exporting the immutable state also enables SEPT A/D bits, which the scan for export
candidates seamcalls require. See
[Step 2: Scanning for Export Candidates](#step-2-scanning-for-export-candidates) for more details.

---

### Step 2: Scanning for Export Candidates

The VMM uses the [TDH.MEM.SCAN.RANGE](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhmemscanrange)
seamcall to get the list of export candidates. The seamcall supports two operations: `DSCAN`, used
to scan for export candidates and described here, and `EXPORT_RESTORE`, used to restore the state of
SEPT entries after an aborted
[export session](../misc/intel-cpu/tdx-migration-misc.md#export-session), see
[Migration Abort](#migration-abort).

The rest of this document covers non-blocking export. For the mode comparison and TDX Connect
constraint, see
[Non-Blocking vs Write-Blocking Export](../misc/intel-cpu/tdx-migration-misc.md#non-blocking-vs-write-blocking-export).
For selecting the mode at TDX module initialization, see
[Export Mode](../misc/intel-cpu/tdx-migration-misc.md#export-mode).

#### Export Candidates

Export candidates are private pages with the dirty bit set or with changed SEPT attributes. In other
words, the seamcall reports more than literally dirty pages: it also reports other events that make
a page need export, for example:

- The TD uses `TDG.MEM.PAGE.ATTR.WR` to change the R/W/X permissions of a page.
- The TD converts an already-exported private page to shared, which `DSCAN` reports as an export
  candidate with the `EXPORTED_REMOVED`
  [SEPT page state](../misc/intel-cpu/tdx-misc.md#sept-page-states).
- The TD converts a shared page to private.
- The TD accepts a pending page.

Later sections cover some of these cases in more detail.

**Terminology Note**: Software engineers familiar with traditional VM live migration sometimes call
this step "dirty page scanning". This document calls it "scanning for export candidates" instead,
because `DSCAN` finds export candidates, not just pages with the dirty bit set.

#### Best Effort Scanning

`DSCAN` is a best effort seamcall. It does not scan every page in all cases. Two examples:

- **Blocked pages**: `DSCAN` skips a private page that was never exported and is currently blocked,
  and returns no output for it. A page blocked after it was already exported is different: `DSCAN`
  still reports it.
- **Pages locked by a concurrent seamcall**: If another seamcall holds a lock on a SEPT entry or
  SEPT sub-tree, `DSCAN` skips the page or page range and returns no output for those skipped
  entries.

The comprehensive scan (`TDH.MEM.SCAN.COMP`) seamcall provides a complete list of export candidates,
but only after the source TD is paused. It is described later.

#### Parallelism

The VMM can invoke `DSCAN` concurrently on multiple CPUs, each covering a different
[GPA](../misc/intel-cpu/x86-misc.md#guest-addresses) range, to scan the memory of the TD faster.
`DSCAN` can also run concurrently with `TDH.EXPORT.MEM`, as long as the two calls do not target the
same pages, for example scanning one GPA range while exporting pages from a different range already
scanned in an earlier round.

#### Only Works During Migration

`DSCAN` only works during an active migration session. In theory, the underlying SEPT Accessed/Dirty
(A/D) bit scanning mechanism could work as a general purpose page modification tracking mechanism,
but the current TDX module implementation restricts it to migration.

The A/D feature is controlled by the TDX module on a per-TD basis (via a bit in
[EPTP](../misc/intel-cpu/x86-misc.md#eptp)). The feature is disabled by default due to performance
cost, and gets enabled by a successful `TDH.EXPORT.STATE.IMMUTABLE` completion. If `DSCAN` runs
before that point, it does not return an error. It finds no modified pages, because the A/D bits are
not enabled yet.

#### First vs Subsequent Scans

The VMM uses the same `DSCAN` invocation for the first scan and later scans. On the first scan,
every currently mapped private TD page is still unexported, so `DSCAN` reports all of them as export
candidates, regardless of whether they are dirty or clean. On later scans, `DSCAN` reports only
pages that changed since their earlier export.

---

### Step 3: TLB Tracking Between Scan and Export

The TDX module requires the VMM to run the
[TLB tracking protocol](../misc/intel-cpu/tdx-misc.md#tlb-tracking) after scanning and before
exporting the export candidate pages identified by `DSCAN`.

#### Stale TLB Entries: The Problem

Hardware sets the dirty bit in a SEPT entry only during a SEPT walk, which it performs when
translating a [GPA](../misc/intel-cpu/x86-misc.md#guest-addresses) for a write. A write that uses a
cached TLB translation skips the walk and leaves the dirty bit unset. `DSCAN` clears the dirty bit
when it detects a modified page, relying on the next write to set it again so that a future scan can
detect the change.

The following hypothetical sequence shows the failure that TLB tracking prevents:

- Page A is dirty, and a CPU holds a cached TLB translation for it.
- `DSCAN` detects page A, reports it, and clears its dirty bit.
- The VMM does not flush the cached TLB translations of the TD.
- `TDH.EXPORT.MEM` exports page A, which is still clean from the `DSCAN` step.
- The TD writes to page A through the cached TLB translation. The write skips the SEPT walk, so it
  does not set the dirty bit, and page A stays clean.
- A later scan may never detect this write, because the dirty bit stays clear while writes keep
  using the cached translation, so the new contents of page A may never be exported.

The destination TD would then hold stale contents for page A, different from the source TD. This
violates the TDX security model, which requires migration to preserve the exact memory contents of
the TD.

#### Stale TLB Entries: The Fix

The fix is to flush the cached TLB translation of page A before exporting it. Once the cached
translation is gone, the next write to page A re-walks the SEPT and sets the dirty bit, so a later
scan detects the write and the page is re-exported.

This flush must happen before the export, not after. With the flush done first, a write that races
with the export is handled correctly either way:

- Write before the export: The dirty bit is set, so the export skips page A, and the next `DSCAN`
  round reports it as an export candidate.
- Write after the export: The old contents are exported, but the dirty bit is set, so the next
  `DSCAN` round reports page A and it is re-exported.

If the flush happened after the export instead, a write in the gap between export and flush would
use the stale translation, leave the dirty bit clear, and be lost.

This is exactly the generic flush-before-operation ordering the
[TLB tracking protocol](../misc/intel-cpu/tdx-misc.md#tlb-tracking) enforces, which also explains
why the flush must be a separate, VMM-driven step, which CPUs need the IPI, and how the TDX module
verifies the VMM actually completed the protocol.

#### Applying the Protocol Between Scan and Export

Migration runs the common [TLB tracking protocol](../misc/intel-cpu/tdx-misc.md#tlb-tracking) on the
GPA range covered by a scan round, between `DSCAN` and `TDH.EXPORT.MEM`:

- `DSCAN` finds modified pages, reports them as export candidates, and clears their dirty bits.
- The VMM runs the TLB tracking protocol: `TDH.MEM.TRACK`, then IPIs to the CPUs running the vCPUs
  of the TD, then re-entering each vCPU with `TDH.VP.ENTER`, which flushes its TLB.
- The VMM executes the `TDH.EXPORT.MEM` seamcall to export the pages.

By the time `TDH.EXPORT.MEM` runs, every TLB that could have held a stale translation has been
flushed.

---

### Step 4: Memory Pages Export

The VMM invokes the [TDH.EXPORT.MEM](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhexportmem)
seamcall on the source host to export candidate pages. It sends the resulting migration bundle to
the destination host, where the VMM imports it using the `TDH.IMPORT.MEM` seamcall.

The next section covers the memory import process and some aspects common to both export and import.

Each completed `TDH.EXPORT.MEM` operation produces one memory migration bundle. The bundle contains
up to 512 encrypted 4 KiB pages (2 MiB of encrypted data) and the metadata that the destination
needs to import them. Refer to the
[Migration Bundle](../misc/intel-cpu/tdx-migration-structs.md#migration-bundle) section for more
information about the bundle structure.

`TDH.EXPORT.MEM` is interruptible. If interrupted, the operation returns before processing is
complete, and the VMM can resume it. Refer to
[Interruptible Seamcalls](../misc/intel-cpu/tdx-misc.md#interruptible-seamcalls) for more
information.

#### Memory Export Errors

`TDH.EXPORT.MEM` processes up to 512 pages per operation. A failure on one page does not stop
processing the remaining pages. Instead, the seamcall records the failure status in the output
[GPA list](../misc/intel-cpu/tdx-migration-structs.md#gpa-list) entry of that page and continues to
the next page.

#### Page Changes Between Scan and Export

When `DSCAN` reports a page as an export candidate, the dirty bit of the page is cleared. If the
page remains clean until the VMM executes the `TDH.EXPORT.MEM` seamcall, the page is exported.
However, if the page is written to again after the scan and before the export, `TDH.EXPORT.MEM`
skips the page. It does not return an error and omits the page from the output migration bundle. The
page is dirty again, so the next `DSCAN` round reports it.

The reader may be wondering: why not export whatever contents the page has at that moment? There
would be two ways to do that, and neither would work well:

1. Export page data without marking it as clean.
2. Mark the page as clean first, then export page data.

The first option would leave the page as an export candidate, so it would have to be exported again
later, which is wasteful. The second option would run into the stale TLB entry problem described in
[Stale TLB Entries: The Problem](#stale-tlb-entries-the-problem) and would require another
`TDH.MEM.TRACK` and IPI round.

#### Large Pages

The source TD may be using large pages (TDX specs term, huge pages in Linux), and `DSCAN` reports
them too. However, `TDH.EXPORT.MEM` accepts only 4 KiB pages, so the VMM has to split large pages
into 4 KiB pages before exporting them, and there is the `TDH.MEM.PAGE.DEMOTE` seamcall for this.

The TDX specs do not explain why large pages cannot be migrated directly, but my guess is
interruptibility. The `TDH.EXPORT.MEM` seamcall is interruptible, but only on page boundaries,
because the encryption of a page cannot be interrupted in the middle. A 2 MiB or 1 GiB page would be
one long operation, and the host could not handle interrupts and other events until it completes.
See [Non-Extension Seamcalls](../misc/intel-cpu/tdx-misc.md#non-extension-seamcalls) for details.

#### Pending Pages

The TD may not have accepted a private page yet, so the page can still be in the
[PENDING](../misc/intel-cpu/tdx-misc.md#sept-page-states) state. When `DSCAN` finds such a page, it
reports it as an export candidate, and the VMM can export it. The resulting bundle contains only
page metadata, not page contents. On the destination, the import seamcall creates the page in the
`PENDING` state.

If the TD accepts a pending page after it was exported, the page becomes dirty again, because the
TDX module zeroes the page contents on accept. `DSCAN` then reports the page again as an export
candidate for a later round.

**Note**: Linux refers to pages in the `PENDING` state as "unaccepted" pages.

#### Private to Shared Conversion

If the TD converts a private page to shared before it was ever exported, the source SEPT entry
becomes `FREE` when the VMM removes the private mapping. `DSCAN` does not report it as an export
candidate.

If the TD converts a private page to shared after it was exported, the source SEPT entry becomes
`EXPORTED_REMOVED`. `DSCAN` reports it as an export candidate, and `TDH.EXPORT.MEM` produces a
`CANCEL` operation with no page contents. On the destination, `TDH.IMPORT.MEM` applies the operation
and removes the previously imported private page.

#### Shared to Private Conversion

If the TD converts a shared page to private during migration, the page becomes a new private page
from the TDX migration point of view. A newly added private page starts in the `PENDING` state and
becomes `MAPPED` when the TD accepts it. `DSCAN` reports the page as an export candidate in either
state.

**Note**: See
[Page State Transitions](../misc/intel-cpu/tdx-misc.md#page-state-transitions) for the general
mechanics of these page state changes, outside the migration context.

---

### Step 5: Memory Pages Import

The destination VMM receives migration bundles from the source host and imports them with the
[TDH.IMPORT.MEM](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhimportmem) seamcall.

Like `TDH.EXPORT.MEM`, `TDH.IMPORT.MEM` is interruptible, so the VMM can resume it after an
interruption.

#### Where Imported Pages Land

The `TDH.IMPORT.MEM` seamcall decrypts page data from the input migration bundle into pages in
destination memory, and those pages become TD private memory. The TDX module supports three options
for where the decrypted data lands:

- **In-place import**: The encrypted page data is decrypted into an internal bounce buffer, and then
  copied into the same page where the encrypted data was stored, overwriting it.
- **Destination page list**: The VMM passes a list of
  [HPAs](../misc/intel-cpu/x86-misc.md#host-addresses) as a seamcall input operand, and the TDX
  module decrypts page data directly into those pages and converts them into TD private pages.
- **Pre-allocated page**: The import writes into a placeholder page the VMM pre-allocated ahead of
  time. This option is part of the
  [enhanced import](../misc/intel-cpu/tdx-migration-misc.md#enhanced-import)
  capability and is available only when it is enabled.

When a page is imported, there are 2 possible situations, depending on the
[SEPT page state](../misc/intel-cpu/tdx-misc.md#sept-page-states) of the destination
[GPA](../misc/intel-cpu/x86-misc.md#guest-addresses):

- **The GPA has no page yet**. This happens on the first import of the page (SEPT state `FREE`). It
  also happens when the page was imported earlier, but was then cancelled (SEPT state `REMOVED`). In
  this situation, the TDX module needs a page, so it takes the one the VMM offers.
- **The GPA already has a page**. This happens on a re-import (SEPT state `MAPPED` or `PENDING`),
  and on the first import over a page the VMM pre-allocated with `TDH.MEM.PAGE.AUG`. In this
  situation the TDX module decrypts into the existing page instead of taking the one the VMM
  offered.

**Note**: Adding pages to the destination TD with `TDH.MEM.PAGE.ADD` is prohibited. Private pages
appear on the destination only through import, or through pre-allocation with `TDH.MEM.PAGE.AUG`.

#### Provisioning the Destination SEPT

Before a page can be imported, the destination TD must have a SEPT mapping for the GPA of the page.
So the VMM must provision the destination SEPT for the incoming GPA range, donating pages for
missing SEPT levels. This can be done using the `TDH.MEM.SEPT.ADD` seamcall, either in advance or
on-demand as migration bundles arrive.

The on-demand SEPT provisioning works as follows. If a SEPT mapping is missing, `TDH.IMPORT.MEM`
does not fail. It reports a SEPT walk failure in the output GPA list entry of that page and
continues with the next page, following the general pattern described in
[Export and Import Error Handling](#export-and-import-error-handling). The VMM can then call
`TDH.MEM.SEPT.ADD` for the missing GPA range and retry importing that page.

#### Cancelling a Previously Imported Page

When the source exports a cancel operation, `TDH.IMPORT.MEM` removes the previously imported private
page. The SEPT entry goes to the `REMOVED` state rather than `FREE`, so the TDX module keeps
tracking the GPA and prevents another import for it in the same migration epoch.

#### Parallelism and Stream Matching

The VMM can run multiple export and import seamcalls in parallel on multiple CPUs and
[migration streams](../misc/intel-cpu/tdx-migration-structs.md#migration-stream). For example, with
4 streams, the VMM can run 4 `TDH.EXPORT.MEM` seamcalls in parallel on the source host and 4
`TDH.IMPORT.MEM` seamcalls in parallel on the destination host.

A bundle must be imported on the same stream on which it was exported. Step 6 describes how the TDX
module enforces bundle ordering.

#### Re-Importing Changed Pages

The destination allows at most one successful `TDH.IMPORT.MEM` operation for a GPA in each migration
epoch. Importing the same GPA again in that epoch fails with a fatal error. The destination can
import an updated version only after it imports an epoch token that advances the migration epoch.
See [Step 6: Migration Epoch Tracking](#step-6-migration-epoch-tracking).

**Note**: The source does not enforce this rule. `TDH.EXPORT.MEM` can export a page again within
the same migration epoch if the page changed after its last export. The destination then rejects the
second import, so the VMM must export the epoch token before re-exporting a changed page. See
[TDH.IMPORT.MEM](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhimportmem) for details.

---

### Step 6: Migration Epoch Tracking

The [in-order phase](../misc/intel-cpu/tdx-migration-misc.md#migration-phases) of the migration
session is divided into **migration epochs**. Each memory copy round forms one epoch:

1. The source exports the memory bundles of the round.
2. The source exports an **epoch token** with
   [TDH.EXPORT.TRACK](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhexporttrack) and sends it to
   the destination.
3. The destination imports the token with
   [TDH.IMPORT.TRACK](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhimporttrack) before it imports
   memory bundles from the next round.

The source does not have to wait for the destination to import the token and may start the next
round of export candidates scanning and memory exporting.

The epoch token is an
[MBMD-only bundle](../misc/intel-cpu/tdx-migration-structs.md#mbmd-only-bundles) that carries no TD
state data, only metadata.

`TDH.IMPORT.TRACK` failures are fatal, following the general pattern described in
[Export and Import Error Handling](#export-and-import-error-handling).

**Note**: `TDH.MEM.TRACK` and `TDH.EXPORT.TRACK` serve different purposes. The former is about TLB
tracking, which ensures scanning for export candidates is correct, see
[Step 3: TLB Tracking Between Scan and Export](#step-3-tlb-tracking-between-scan-and-export). The
latter is about enforcing the migration bundle ordering rules.

#### The Role of Migration Epochs

The TDX module must guarantee that the destination TD ends up with the exact same memory contents as
the source TD. Encryption alone cannot provide that guarantee: the untrusted VMM can still drop
memory bundles, or replay old ones. A replayed bundle that holds an older version of a page still
has a valid MAC.

Within a migration stream, the TDX module orders bundles with a bundle counter. Streams are
independent, so there is no order across them. Migration epochs provide that order.

Here are the three main roles of migration epochs.

- **Barrier across streams**: The destination accepts only bundles of the current epoch, and
  importing the epoch token advances it. So the destination imports all bundles of one epoch before
  any bundle of the next epoch, on any stream.
- **Completeness check**: The epoch token carries the number of bundles that the source exported. If
  the destination has not imported all of them, importing the token fails, which detects dropped
  bundles.
- **Replay protection**: The destination accepts at most one import of a page per epoch. Together
  with the barrier and the completeness check, this prevents a replay attack, where an older version
  of a page replaces a newer one.

See the [Migration Epochs](../misc/intel-cpu/tdx-migration-misc.md#migration-epochs) section for the
details of these checks.

#### Stream 0 and Bundle Completeness

Migration epoch tokens are always exported and imported on
[migration stream 0](../misc/intel-cpu/tdx-migration-structs.md#migration-stream), regardless of how
many streams are used for memory bundles. Before executing the `TDH.EXPORT.TRACK` seamcall, the
source VMM must let all export seamcalls on all streams complete, so the bundle count of the token
covers the whole epoch.

---

### Step 7: Configure the Comprehensive Scan

The goal of this step is to configure the final scan for export candidates, also referred to as the
comprehensive scan. The comprehensive scan itself runs after the TD is paused, and uses a different
seamcall than the regular export candidate scanning (`TDH.MEM.SCAN.RANGE`): `TDH.MEM.SCAN.COMP`.

**Note**: In my opinion, having two different seamcalls for scanning for export candidates is not
the best ABI design.

This step conceptually belongs to
[Step 12: Transfer Final Memory State](#step-12-transfer-final-memory-state). The VMM is not
required to run it here. It can call
[TDH.MEM.SCAN.CONFIG](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhmemscanconfig) at any time,
but running it before the source TD is paused keeps this work out of the downtime.

The comprehensive scan always scans the entire TD
[GPA](../misc/intel-cpu/x86-misc.md#guest-addresses) space. To shorten the downtime, it is better to
scan the space in parallel on multiple CPUs. The `TDH.MEM.SCAN.CONFIG` seamcall configures the
ranges used for parallel scanning. For example, if the VMM uses 4 CPUs, it can split the TD GPA
space into 4 ranges, one per CPU.

Refer to the [Range List](../misc/intel-cpu/tdx-migration-structs.md#range-list) for details on how
ranges are specified and how to choose range boundaries for NUMA-aware parallel scanning.

---

### Step 8: Optional Pre-Scan

This step is optional but recommended. Its goal is to shorten the comprehensive scan, and therefore
the downtime, by letting it skip SEPT sub-trees that contain no export candidates.

The [TDH.MEM.SCAN.COMP](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhmemscancomp) seamcall
supports three operations, listed in the order the VMM runs them:

- `PRECLEAR`: Scans the SEPT and clears the accessed bit in every non-leaf SEPT entry.
- `PRECHECK`: Scans the SEPT and sets the accessed bit in every non-leaf SEPT entry whose sub-tree
  contains export candidates.
- `DCHECK`: Runs the comprehensive scan after the source TD is paused.

`PRECLEAR` and `PRECHECK` form the pre-scan and run while the source TD is still running. `DCHECK`
runs later, in [Step 12: Transfer Final Memory State](#step-12-transfer-final-memory-state).

#### Why the Accessed Bit

The pre-scan relies on the accessed bit, not the dirty bit. During a SEPT walk, the CPU sets the
accessed bit in every entry on the walk path, from the root down to the leaf. A cleared accessed bit
in a non-leaf entry therefore shows that no SEPT walk went through that sub-tree, without scanning
the sub-tree itself. The CPU sets the dirty bit only in the leaf entry, so it carries no information
about sub-trees.

#### How the Pre-Scan Works

1. `PRECLEAR` clears the accessed bits in all non-leaf SEPT entries, marking every sub-tree as
   unchanged.
2. The VMM runs the
   [TLB tracking protocol](../misc/intel-cpu/tdx-misc.md#tlb-tracking).
   After the flush, the next TD access to a page walks the SEPT and sets the accessed bits along the
   path. Without the flush, an access through a cached TLB translation would skip the walk and leave
   the bits cleared. This is the same hazard as the one described in
   [Stale TLB Entries: The Problem](#stale-tlb-entries-the-problem), but for the accessed bit
   instead of the dirty bit.
3. `PRECHECK` scans the SEPT and sets the accessed bit in the non-leaf ancestors of every export
   candidate. This covers pages that became export candidates before the TLB tracking protocol
   completed, whose changes may not have set the accessed bits of their ancestors.

As a result, `DCHECK` can skip any sub-tree whose non-leaf accessed bit is still cleared. Such a
sub-tree had no export candidates when `PRECHECK` ran, and the TD has not accessed it since the TLB
tracking protocol.

**Note**: The pre-scan needs two separate operations instead of one because the TLB tracking
protocol must run between them, and only the VMM can drive it.

---

### Step 9: Pause the Source TD

Once the memory copy rounds meet the convergence criteria, the VMM pauses the source TD with the
[TDH.EXPORT.PAUSE](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhexportpause) seamcall. The pause
starts the downtime period, the time when the TD runs on neither host.

Before invoking `TDH.EXPORT.PAUSE`, the VMM must:

- Stop all vCPUs of the source TD. Otherwise the seamcall fails.
- For a TD configured for TDX Connect, detach all TDIs from the TD. Otherwise, the seamcall fails.
  See [Device State Migration](#device-state-migration).

After `TDH.EXPORT.PAUSE` succeeds, the TDX module rejects `TDH.VP.ENTER`, so the vCPUs cannot resume
and the source TD state does not change.

---

### Step 10: Transfer TD Mutable State

After the source TD is paused, the VMM exports the TD mutable state using the
[TDH.EXPORT.STATE.TD](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhexportstatetd) seamcall. The
destination VMM imports the resulting migration bundle using the
[TDH.IMPORT.STATE.TD](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhimportstatetd) seamcall.

TD mutable state is per-TD data that can change while the TD runs, for example the virtual TSC
state. See [TD Mutable State](../misc/intel-cpu/tdx-migration-structs.md#td-mutable-state) for more
details.

The `TDH.IMPORT.STATE.TD` seamcall checks that the imported TD mutable state is compatible with the
destination platform capabilities and TDX module configuration.

The TD mutable state is imported once, on
[migration stream 0](../misc/intel-cpu/tdx-migration-structs.md#migration-stream).

---

### Step 11: Transfer vCPU State

After the TD mutable state is transferred, the VMM exports the mutable state of each vCPU using the
[TDH.EXPORT.STATE.VP](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhexportstatevp) seamcall. The
destination VMM imports each resulting migration bundle using the
[TDH.IMPORT.STATE.VP](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhimportstatevp) seamcall.

The vCPU state includes the guest GPRs and other [TDVPS](../misc/intel-cpu/tdx-structs.md#tdvps)
data.

Each vCPU is exported and imported once. vCPU state can be transferred in parallel using multiple
[migration streams](../misc/intel-cpu/tdx-migration-structs.md#migration-stream).

---

### Step 12: Transfer Final Memory State

This is the final memory transfer, run after the source TD is paused. The VMM runs the comprehensive
scan (the `DCHECK` operation of the `TDH.MEM.SCAN.COMP` seamcall), configured in
[Step 7: Configure the Comprehensive Scan](#step-7-configure-the-comprehensive-scan), to find all
remaining export candidates in one pass.

Unlike the iterative memory copy rounds, TLB tracking is not needed, because the TD is paused and
does not write to memory.

To shorten the downtime, the VMM can run `DCHECK` concurrently on multiple CPUs, one per GPA range
configured in Step 7.

The VMM exports the final pages with the `TDH.EXPORT.MEM` seamcall and sends the resulting migration
bundles to the destination, which imports them with the `TDH.IMPORT.MEM` seamcall. The export and
import rules are the same as in [Step 4: Memory Pages Export](#step-4-memory-pages-export) and
[Step 5: Memory Pages Import](#step-5-memory-pages-import).

**Note**: `DCHECK` can run concurrently with `TDH.EXPORT.MEM`, so the VMM can export already scanned
ranges while `DCHECK` still scans the rest.

---

### Step 13: The Start Token

Like the memory copy rounds described in
[Step 6: Migration Epoch Tracking](#step-6-migration-epoch-tracking), the final round ends with the
source exporting a special version of an epoch token called **the start token**.

The start token is generated by the
[TDH.EXPORT.TRACK](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhexporttrack) seamcall with
`IN_ORDER_DONE`=1 on the source, and imported by the destination using the
[TDH.IMPORT.TRACK](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhimporttrack) seamcall.

The start token performs all the checks of a regular epoch token, described in
[The Role of Migration Epochs](#the-role-of-migration-epochs). In addition, it ensures that the TD
never runs on both hosts:

- Exporting the start token forbids the source TD from running.
- Importing the start token gives the destination TD the right to run, but the TD starts only in
  the [next step](#step-14-complete-the-migration).

After the destination imports the start token, there are only two paths forward:

- Successfully complete the migration, run the TD on the destination and tear down the source TD.
- Abort the migration, which involves generating an abort token on the destination and consuming it
  on the source. See [Migration Abort](#migration-abort).

**Note**: The start token ends the in-order phase of the migration session. The source and
destination then move to the out-of-order phase, described in
[Migration Phases](../misc/intel-cpu/tdx-migration-misc.md#migration-phases).

---

### Step 14: Complete the Migration

Once the destination imports the start token, it is ready to run the TD. In pre-copy migration, the
destination VMM executes the
[TDH.IMPORT.END](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhimportend) seamcall, which starts
the destination TD and ends the migration session.

**Note**: With post-copy migration, the destination VMM may execute the
[TDH.IMPORT.COMMIT](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhimportcommit) seamcall after
importing the start token, so the TD runs before all private memory pages are imported. Post-copy
migration is currently not supported with non-blocking export, the focus of this document. See
[Post-Copy Support](../misc/intel-cpu/tdx-migration-misc.md#post-copy-support) for details.

---

## Migration Abort

Live migration can be aborted, but only if the destination TD has not started running. Otherwise the
state of the destination TD diverges from the source TD, and TDX has no mechanism to move that state
back.

A successful abort returns the source TD to normal operation on the source host. If the source TD
was paused, it runs again. The destination TD is not needed anymore, so the VMM tears it down. The
TDX module will not allow it to run anyway.

The abort procedure depends on how far the migration progressed. The key events are:

- The setup session completes.
- The VMM exports the immutable state (see [Step 1](#step-1-immutable-state-transfer)).
- The source generates the start token (see [Step 13](#step-13-the-start-token)).
- The destination TD starts running (see [Step 14](#step-14-complete-the-migration)).

During the setup session, the VMM aborts with a separate mechanism, described in
[Setup Session Abort](#setup-session-abort). After the [setup session](#setup-session-1) completes,
there are three cases:

1. **Before the immutable state export**: The [migration session](#migration-session-1) has not
   started yet. The source TD keeps running as usual, and the VMM only tears down the destination TD
   skeleton, if it created one.
2. **Before the start token**: The source VMM can abort the migration on its own.
3. **After the start token**: The source TD gave up its right to run to the destination TD. To abort
   the migration, the destination VMM must generate the abort token, proof that the destination TD
   will not be allowed to run. The VMM delivers the token to the source, and the source can then
   abort the migration.

Once the destination TD starts running, the migration cannot be aborted anymore and must proceed to
completion.

The following subsections describe cases 1 to 3 in detail.

### Before the Immutable State Export

The migration session starts when the VMM executes `TDH.EXPORT.STATE.IMMUTABLE` on the source TD,
see [Step 1: Immutable State Transfer](#step-1-immutable-state-transfer). Before that, the TDX
module does not track anything for the migration on the source TD. For example, it has not:

- Enabled the SEPT A/D bits for dirty tracking.
- Started recording exported pages in SEPT entries, which have dedicated
  [export states](../misc/intel-cpu/tdx-misc.md#sept-export-states).

So there is nothing to undo, and the source TD keeps running.

If the VMM has already created the destination TD skeleton in preparation for the immutable state
import, it tears the skeleton down.

### Before the Start Token

To abort the migration before the source [generates the start token](#step-13-the-start-token), the
VMM runs these steps:

- **Source**:
  - Call [TDH.EXPORT.ABORT](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhexportabort) without the
    abort token.
  - Restore the SEPT entries using one of the two mechanisms described below.
- **Destination**: Tear down the TD.

The destination step is trivial, so the rest of this section explains the two source steps.

`TDH.EXPORT.ABORT` ends the [export session](../misc/intel-cpu/tdx-migration-misc.md#export-session)
and returns the source TD to normal operation. If the source TD was already paused (see
[Step 9: Pause the Source TD](#step-9-pause-the-source-td)), it runs again, which ends the downtime
period.

After the abort seamcall, the source TD runs normally, and the next step is to restore the SEPT
entries. The TDX module provides two mechanisms for this:

- [TDH.EXPORT.RESTORE](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhexportrestore): The VMM
  provides the list of GPAs to restore. It suits a VMM that knows the GPAs of all exported pages.
- [EXPORT_RESTORE](../misc/intel-cpu/tdx-migration-seamcalls.md#export_restore): An operation of the
  `TDH.MEM.SCAN.RANGE` seamcall. The VMM provides a GPA range, and the seamcall finds the exported
  pages within the range and restores them. It suits a VMM that does not track exported pages.

The restore cleans up the SEPT entry states. During migration, the TDX module uses special
[export states](../misc/intel-cpu/tdx-misc.md#sept-export-states), such as `EXPORTED`, to track
which pages the VMM exported. The module needs this tracking to enforce the consistency of the TD
state (see [Security Properties](#security-properties)).

`TDH.EXPORT.ABORT` does not restore the SEPT entries, so the source TD resumes without delay. The
VMM restores them later, while the TD runs. The restore moves each entry from its export state back
to its normal state, for example from `EXPORTED` to `MAPPED`.

If the VMM skips the restore, the source TD keeps running, but the next migration of the TD cannot
start, because `TDH.EXPORT.STATE.IMMUTABLE` fails until the restore completes.

### After the Start Token

To abort the migration after the source generates the start token, the VMM runs these steps:

- **Destination**:
  - Call [TDH.IMPORT.ABORT](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhimportabort) to generate
    the abort token.
  - Deliver the token to the source.
  - Tear down the TD.
- **Source**:
  - Call [TDH.EXPORT.ABORT](../misc/intel-cpu/tdx-migration-seamcalls.md#tdhexportabort) with the
    abort token.
  - Restore the SEPT entries in the same way as in the "Before the Start Token" case.

The abort needs the destination because of the right to run. The
[start token](#step-13-the-start-token) transfers this right from the source TD to the destination
TD. The source TD loses it when the source generates the token, and the destination TD gets it when
it imports the token. This enforces the non-clonability property (see
[Security Properties](#security-properties)).

The abort token moves the right to run back. It proves that the destination TD will not be allowed
to run, so the source TD can resume.

`TDH.IMPORT.ABORT` moves the destination TD to the `FAILED_IMPORT` state, where the TDX module does
not allow it to run, and generates the abort token. The source consumes the token with
`TDH.EXPORT.ABORT`, which returns the source TD to normal operation and ends the downtime period.

The abort token is a small
[MBMD-only bundle](../misc/intel-cpu/tdx-migration-structs.md#mbmd-only-bundles), like the epoch and
start tokens, but it travels from the destination to the source.

---

## Configuration

Live migration requires configuration at three levels:

- **TDX module**: The VMM enables the migration features when it initializes the TDX module.
- **TD**: The VMM creates the TD as migratable.
- **Migration session**: The VMM creates the migration streams for parallel state transfer before
  the [migration session](#migration-session-1) starts.

See the [Configuration](../misc/intel-cpu/tdx-migration-misc.md#configuration) section for details.
