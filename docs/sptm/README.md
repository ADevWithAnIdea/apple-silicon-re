# SPTM Emulator Clean-Room Documentation

## RE Notes

These docs were written based on extensive disassembly of the SPTM binary as
found on the MacBook Neo (T8140) on firmware version 26.6 beta 4, as well
as the 26.5 KDK.

Parts have been updated for 27.0 Beta 3 2 for the MacBook Air M5.

## Scope

These documents are not a 1:1 description of what SPTM does. We make a best
effort to describe what real SPTM does. However, real SPTM does many detailed
checks, much of which is not relevant to just get an emulator working. In these
cases, we prioritize documenting enough XNU facing behavior to allow the
building of a clean room emulator. This is to say that these documents are not
a full description of SPTM's security posture.

MANDATORY BACKGROUND READING: Steffin & Classen, *Modern iOS Security Features
– A Deep Dive into SPTM, TXM, and Exclaves*, arXiv:2510.09272 — 
<https://arxiv.org/abs/2510.09272>.

Every component of the SPTM emulator has two parts: 

1. the boot handoff between the emulator and XNU

2. endpoint emulation

Together, the completed documents cover the T8140 SPTM tables needed by the
current macOS 26.6 beta 4 boot path. Start with the XNU bootstrap document,
then use the subsystem-specific documents for each runtime dispatch table.

## Emulators

- [xnu_bootstrap.md](xnu_bootstrap.md) — XNU_BOOTSTRAP table (page-table / frame-table) and the boot handoff
- [dart.md](dart.md) — DART IOMMU
- [nvme.md](nvme.md) — NVMe
- [uat.md](uat.md) — GPU UAT and T8140 SAPT
- [sart.md](sart.md) — SART
- [txm.md](txm.md) — minimal TXM/XNU compatibility shim


## SPTM overview

### Dispatch ABI

XNU reaches SPTM via `genter`, we patch `genter` to `HVC #0` so that m1n1
can trap and emulate sptm functionality. One 64-bit dispatch word in `x16`:

```
bits 63:56  reserved (zero)
bits 55:48  domain      (8)
bits 47:40  reserved (zero)
bits 39:32  table       (8)
bits 31:0   endpoint    (32)
x16 = (domain << 48) | (table << 32) | endpoint
```

Arguments in `x0..xN`; primary result in `x0`.  Some endpoints also write a
per-CPU scratch page (specified per-endpoint).

Dispatch flow: on every HVC trap the EL2 handler:

1. Reads `ESR_EL2.ISS` (the `genter` immediate).  We only handle `ISS=0` as
   the rest do not occur during regular operation.
2. Reads `x16`. Rejects if any reserved bit is set, or `domain > 4`, or
   `table > 11`.
3. Retrieve domain, table, and endpoint.  Args remain in `x0..xN`.
4. Resolves `x16` to an endpoint via three nested lookups: `domain` picks
   the component (0=SPTM, 2=TXM, 3=SK, each owns its own table namespace);
   `table` picks one dispatch table within that domain (for SPTM:
   XNU_BOOTSTRAP / DART / NVME / ...); `endpoint` is the function within
   that table.
5. Writes `x0` plus any per-endpoint scratch outputs, `eret`.

### Domains

| val | domain |
|----:|--------|
| 0 | SPTM |
| 1 | XNU |
| 2 | TXM |
| 3 | SK |
| 4 | XNU_HIB |
| 6 | DOMAINS_NONE (sentinel) |
| 255 | NO_PANICKING_DOMAIN |

Only domain 0 (SPTM) and domain 2 (TXM) are brought up in our emulator; domain
3 (SK) is not since we currently do not support exclaves. Domains 1 (XNU), 4
(XNU_HIB), 6 (DOMAINS_NONE), and 255 (NO_PANICKING_DOMAIN) are unused.

### Tables

Within a domain, the table number selects one dispatch table. SPTM (domain 0)
owns these:

| val | table | notes |
|----:|-------|-------|
| 0 | XNU_BOOTSTRAP | the table containing all the page table management operations -- see [xnu_bootstrap.md](xnu_bootstrap.md) |
| 1 | TXM_BOOTSTRAP | unused |
| 2 | SK_BOOTSTRAP | unused |
| 3 | T8110_DART_XNU | see [dart.md](dart.md) |
| 4 | T8110_DART_SK | unused |
| 5 | SART | see [sart.md](sart.md) |
| 6 | NVME | see [nvme.md](nvme.md) |
| 7 | UAT | see [uat.md](uat.md) |
| 8 | SHART | unused |
| 9 | RESERVED | unused |
| 10 | HIB | unused |
| 11 | GEN3_DART_XNU | not present on T8140 |
| 12 | GEN3_DART_SK | not present on T8140 |
| 13 | T6000_DART_XNU | unused |
| 14 | INVALID | unused |
| 0xFD | RETURN_TO_CALLER | unused |
| 0xFE | PANIC | unused |
| 0xFF | EXCEPTION_STATE_SAVED | unused |

TXM (domain 2) only has a single table, see [txm.md](txm.md)

Every file contains details of both the pre xnu handoff and the runtime
endpoint emulation.

## Future Work

Work is ongoing to support Exclaves. When this work is complete, this
documentation will be updated with the new information.

## Legal

Copyright (c) Cody Ho.

Licensed under the CC-BY-SA-NC-4.0
