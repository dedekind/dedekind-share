# TDX Module Notes

- **Author**: Artem Bityutskiy
- **Version**: 0.1
- **Date**: 2026-07-03
- **Last updated**: 2026-09-06

**Disclaimer**: These notes reflect the author’s current understanding of TDX module concepts and
mechanisms and may contain errors or omissions. They are not an official Intel document. Refer to
the official Intel documentation for authoritative information.

This is a collection of notes and explanations about the TDX module. It is intended as a reference
for other articles and is not a standalone article.

**Terminology Notes**:

- A **seamcall** is a TDX module ABI leaf function invoked with the `SEAMCALL` instruction.
  For example, `TDH.MIG.SETUP` is a seamcall.
- A **tdcall** is a TDX module ABI leaf function invoked with the `TDCALL` instruction.
  Tdcalls include all `TDG.*` leaf functions.
- `SEAMCALL` and `TDCALL` refer to the CPU instructions, while `seamcall` and `tdcall` refer to
  the leaf functions they invoke.

## Table of Contents

- [TDX Module Notes](#tdx-module-notes)
  - [Table of Contents](#table-of-contents)
  - [VM Exit and TD Exit](#vm-exit-and-td-exit)

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
