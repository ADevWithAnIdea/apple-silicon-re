# NVMe SPTM Dispatch

This document is written against m1n1 commit
`f5f8d2c7027d4ae37e4db27ea33720fe98151445` on 2026-07-27.

Real SPTM does significant validation on all requests; we do our best to
document it here, even though most if not all is required for a functioning
emulator. However, we know that it is not complete.

- **Table 6** `NVME`, endpoints 0..8 (9)

Related but separate: **Table 5** `SART` (the ANS DMA address-allowlist)

The normal ANS/NVMe controller behavior is already understood in m1n1. This
document describes the SPTM layer around it: boot-time state seeded from ADT,
validated queue-register writes, CoastGuard/NVMMU setup, and per-command TCB
mapping.

`nvme-secure-reg-layout` and the secure BAR path change the endpoint order that
real SPTM allows during queue setup, but the practical hardware operation is
the same logical ANS/NVMe register write through a different MMIO aperture. The
emulator does not need to enforce this as a security boundary, but it does need
to select the same BAR aperture before servicing the queue-register endpoints.

## 1. Boot Handoff

Before XNU boots, seed one NVMe state object from `/arm-io/ans`, `/defaults`,
and `/chosen/carveout-memory-map`.

- `/arm-io/ans/reg[3]`: main ANS BAR. SPTM derives the NVMMU base as
  `reg[3] + 0x28000`.
- `nvme-secure-reg-layout`: if present, SPTM uses `reg[3] + 0x4000` as the
  BAR aperture for its queue-register writes. All operations remain the same,
  just with a shifted base address.
- `/defaults/nvme-iboot-sptm-security`, `nvme-secure-bar`, and `reg[9]`: if
  the defaults property and `nvme-secure-bar` are both present and
  `nvme-secure-reg-layout` is absent, SPTM uses `reg[9]` as the BAR aperture
  for the same logical queue-register writes.
- `nvme-queue-entries`: seeds the queue entry count, so valid CIDs are `0`
  through `queue_entries - 1`. SPTM sizes each TCB span as
  `round_up(queue_entries * 0x80, 0x1000)` and allocates
  `round_up(queue_entries * 0x800, 0x4000)` bytes of PRP-list backing,
  providing one `0x800`-byte slot per CID. m1n1 currently uses a fixed queue
  size.
- `nvme-linear-sq`: seeds the linear submission queue mode. Endpoints 1 and 3
  use this for queue-protocol validation and TCB/queue backing layout.
- `/chosen/carveout-memory-map/region-id-55`: identifies a physical-memory
  region that SPTM's security model additionally permits as backing for certain
  NVMe/ANS objects.
- `nvme-prp-flush-wa`: enables the PRP-list cache flush workaround; see
  endpoints 1 and 2.

The following properties signal extra sideband MMIO operations used by later
endpoints:

- `nvme-tl-wa`, `reg[10]`, `reg[11]`, `reg[12]`, and `nvme-num-sl`: seed the
  TL workaround state: mask, control, status, and slot count.
- `nvme-vdma-wa` and `reg[13]`: seed the VDMA workaround status MMIO state.
- `nvme-ans-sha-present` and `reg[14]`: seed ANS SHA support and the ANS SHA
  register aperture.

SPTM also allocates and records its own TCB backing and per-CID PRP-list
scratch from the seeded queue count. Those buffers are not ADT properties.

### 1.1 Internal State

SPTM keeps one NVMe state object. Boot handoff seeds the selected BAR/NVMMU
apertures, optional TL/VDMA/ANS-SHA sideband apertures, queue count, linear-SQ
mode, secure/packed BAR mode, workaround flags, endpoint allow state, and the
range specified by `region-id-55`.

SPTM also owns memory that XNU does not directly manage: TCB backing for
queues 0 and 1 and one contiguous PRP-list buffer divided into a 0x800-byte
slot for each CID.

Queue-register endpoints cache the values they accept, including admin queue
registers, IO queue registers, and ANS-SHA state. Later calls must match the
cached state rather than silently replacing it.

For each CID, SPTM records the command lifetime state needed by endpoint 2:
queue id, command direction, DMA access mode, PRP count, PRP1, PRP2 or PRP-list
pointer, and whether the CID is free, mapped, busy, or waiting for a retry.
Accepting PRP pages also acquires frame-table references in the command's DMA
access mode; endpoint 2 releases those references before freeing the CID.

## 2. Endpoints

All endpoints except endpoint 2 return `x0 = 0`. Any violation is a fatal error
that immediately panics SPTM. Endpoint 2 is strange, it returns `x0 = 1` on
success and `x0 = 0` which indicates retry.  The endpoint allow-mask is
primarily order/security enforcement; on the normal happy path XNU is expected
to call these in an allowed order.

| Endpoint | Purpose | Args | Behavior |
|---:|---|---|---|
| 0 | Enable CoastGuard | none | Programs the SPTM-owned CoastGuard/TCB queue pointers into the NVMMU, enables CoastGuard, and advances the setup state. |
| 1 | Map pages | `qid`, `cid`, `tcb_template_pa`, `seglist_pa`, `count` | Claims a CID, reads the TCB template, validates PRP pages, builds PRP1/PRP2 or a PRP list, writes the TCB entry, and records per-CID state. |
| 2 | Unmap pages | `qid`, `cid`, `retry` | Validates the CID state, invalidates/clears the TCB, handles workaround waits, releases recorded PRP state, and frees the CID. |
| 3 | Validate queue entries | `queue_entries`, `protocol` | Checks XNU's queue count and queueing protocol against the ADT-seeded `nvme-queue-entries` and `nvme-linear-sq` state. |
| 4 | Program admin queues | `asq_pa`, `asq_size`, `acq_pa`, `acq_size` | Validates and caches admin queue register values, then writes `AQA`, `ASQ`, and `ACQ` through the selected BAR aperture. |
| 5 | Program IO queue sizes | `iosq_size`, `iocq_size` | Validates and caches IO queue sizes, then writes `IOQA` through the selected BAR aperture. |
| 6 | Program IO submission queue | `iosq_pa` | Validates and caches the IO submission queue base, then writes `IOSQ` through the selected BAR aperture. |
| 7 | Program IO completion queue | `iocq_pa` | Validates and caches the IO completion queue base, then writes `IOCQ` through the selected BAR aperture. |
| 8 | Program ANS SHA aperture | `sha_pa`, `sha_size`, packed-write config | If ANS SHA support was seeded, validates the SHA buffer and programs the SHA aperture/register state. |

### 2.1 Enable CoastGuard (endpoint 0)

Endpoint 0 arms the CoastGuard/NVMMU path. Most of the endpoint is SPTM state
management: it checks that CoastGuard enable is allowed in the current setup
phase, uses the CoastGuard queue backing allocated during boot handoff, and
advances the endpoint-order state so XNU can continue queue setup or
per-command mapping.

The hardware-visible operations use the NVMMU register layout already
implemented in `src/nvme.c`. Write the physical address of the first
boot-allocated TCB span to `NVMMU_ASQ_BASE` and the second to
`NVMMU_IOSQ_BASE`, then write `0x3f` to `NVMMU_NUM`. SPTM performs each 64-bit
base write as low/high 32-bit writes and issues `dsb sy` after every MMIO
write.

### 2.2 Map and Unmap Pages (endpoints 1, 2)

Endpoints 1 and 2 manage the per-command DMA authorization for one queue/CID.

The segment list is a per-command array of 64-bit physical addresses supplied
by XNU: the exact pages XNU wants ANS to DMA to or from for this command.

Endpoint 1 takes a mostly-built 0x80-byte TCB template from XNU plus a separate
segment list of requested DMA pages. It then claims the CID and copies the TCB
template and zeros the template PRP fields. Detailed validation is performed on
the segment list.  The segment-list buffer is checked to ensure it's nonzero,
located in main memory, `count * 8` bytes are readable by SPTM, the buffer does
not cross a 16k page boundary, that `count <= 0x101`.  It is also checked
against the TCB template, specifically, if `count=0` then the `length` field in
the TCB must also be zero, otherwise an expected length value is calculated as
follows:

- if PRP1 is 4K-aligned, then expected length = `count - 1`
- if PRP1 has an offset into the first 4K page, then expected length = `count - 2`
- an unaligned address with only one segment-list entry is invalid

Each PA in the segment list is also checked to ensure that the first entry has
a 4K page offset or is 4K aligned, that all subsequent pages are 4K aligned,
and that each target is in an allowed memory range.

After validation, each PRP page has its mapping refcount incremented in the
frame table. If `dma_flags` bit 0 is set (the TCB byte at offset `0x01`), the WX
refcount is incremented, otherwise the RO refcount.

Finally, the PRP fields in the TCB that were previously zeroed are rebuilt
using values from the validated list, and then written to the live TCB slot.
Per CID state is recorded for unmap later.

For one or two requested pages, SPTM writes the validated addresses directly
into TCB PRP1 and PRP2. For more than two pages, it writes the first address to
PRP1, writes the physical address of the CID's `0x800`-byte PRP-list slot to
PRP2, and fills that slot with the remaining addresses. The slot address is
`prp_list_base + cid * 0x800`.

The primary difference compared to m1n1 is that SPTM merely validates (via the
above rebuild process) TCB templates supplied by XNU. It is likely that an
emulator can just pass XNU built TCBs directly to hardware.

The TCB backing selected by endpoint 1 depends on `nvme-linear-sq`. When the
property is present, boot allocates two TCB spans, one for each queue, and
endpoint 1 writes into those spans. When it is absent, boot allocates four
spans and endpoint 1 writes into the later pair. Endpoint 0 always programs the
physical addresses of the first pair into the NVMMU. Each span has the size
specified in Section 1.

Endpoint 2 validates the recorded CID state, invalidates and clears the live
TCB slot, releases the recorded PRP pages by decrementing the refcounts that
were previously incremented, clears scratch state, and frees the CID.

If `nvme-prp-flush-wa` was seeded from ADT and `count > 0x10`, endpoint 1
issues `dsb sy` and clean+invalidates the PRP-list slot. For `count < 0x100`,
the length is `(count * 8 + 0x7f) & 0x1f80`; otherwise it is `0x1000`.

During endpoint 2, if the recorded count is greater than `0xff`, SPTM issues
`dsb sy` and clean+invalidates `0x800` bytes starting at
`prp_list_slot + 0x800`.

Endpoint 2 has the unusual normal return convention: successful teardown
returns `x0 = 1`; if the `nvme-tl-wa` sideband status wait times out, endpoint
2 returns `x0 = 0` so XNU can retry the teardown later. Other invalid arguments
or impossible CID state are fatal SPTM violations.

Endpoint 2 writes the CID to m1n1's `NVMMU_TCB_INVAL` register and issues
`dsb sy`. It then reads the packed 32-bit status table beginning at the NVMMU
base. The first word contains the statuses for CIDs 0-3, and each following
word covers the next four CIDs. Within each word, the four status fields begin
at bits 0, 5, 10, and 15. The low four bits of the selected field must be zero.
This packed status table is not implemented by m1n1. The ADT-seeded workaround
sidebands are separate.

If `nvme-vdma-wa` is present, an additional check is performed.
Endpoint 2 reads `reg[13] + 0x20000 + cid * 0x20` and requires bits `0x300` be
clear, otherwise panic.

If `nvme-tl-wa` is present, SPTM has additional handling before releasing a
command's pages. If a DMA direction is set (`dma_flags & 0x3`, bit 0 meaning
controller writes, bit 1 meaning controller reads) then SPTM waits for ANS's
transaction layer to report that direction complete.  The wait uses
`nvme-num-sl` transaction slots (at most 16) across three sideband apertures:
`reg[10]` (mask), `reg[11]` (control), and `reg[12]` (per-slot status).

On the first (non-retry) entry, SPTM arms the mask and control for the
command's direction. It writes the mask to both `reg[10] + 0x4` and `reg[10] +
0xc`, the control to `reg[11] + 0x4`, and clears its per-CID record of drained
slots. Each direction has its own mask, control, and per-slot status field:

| Direction | mask | control | slot status field |
|---|---|---|---|
| controller-writes (`dma_flags` bit 0) | `0x800000` | `0x400000` | low 8 bits (`0xff`) |
| controller-reads (`dma_flags` bit 1) | `0x80` | `0x40` | bits 16-22 (`0x7f0000`) |

It then polls: for each slot, read the 32-bit status at `reg[12] + slot *
0x10000` (one register per slot, 64 KiB apart). A slot is done for the
direction once every bit of its status field reads 1. SPTM tracks which slots
are done per-CID so a retry does not re-poll them. Once all slots are done, it
writes `0x800080` (`0x800000 | 0x80`) to both mask offsets and `0x400040`
(`0x400000 | 0x40`) to the control, clears the per-CID record, and lets
teardown proceed.

SPTM polls repeatedly for up to approximately .25ms, if TL is still not idle
then it marks the CID for retry and returns 0 so XNU can retry later without
rearming the mask or control registers for the command.

If `retry` is set, SPTM skips the TCB clear, VDMA check, NVMMU invalidation,
and the initial direction-specific writes to the mask and control registers.
It resumes transaction-layer polling using the recorded completed-slot state.
Once all slots are done, it still writes `0x800080` to both mask registers and
`0x400040` to the control register before completing teardown.

### 2.3 Validate Queue Entries (endpoint 3)

Endpoint 3 validates XNU's queue setup against the boot-seeded NVMe mode. The
requested queue entry count must match `nvme-queue-entries`.

The `protocol` argument is checked against `nvme-linear-sq`: protocol `2` is
expected when `nvme-linear-sq` is present, and protocol `1` is expected
otherwise. This endpoint does not program hardware; it only confirms that XNU
and SPTM agree on the queue layout that later endpoints will use.

### 2.4 Program Queues and Apertures (endpoints 4-8)

Endpoints 4 through 7 perform the queue-register writes already implemented
by m1n1, using the BAR aperture selected in Section 1. SPTM issues `dsb sy`
after each 32-bit MMIO write.

Endpoint 8 is available only when `nvme-ans-sha-present` is present. It writes
the SHA buffer page number (`sha_pa >> 14`) to `reg[14] + 0x0` and the 2-bit
packed-write configuration to `reg[14] + 0x4`.

SPTM internal checks consist of validation and caching of the arguments, along
with the allowed-function gating. Generally the checks are: each address is
page-aligned and lies in the trusted NVMe I/O range, queue sizes are within
their limits, and a repeated call must match the value cached the first time.
These can be skipped in the emulator.
