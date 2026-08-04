# DART SPTM Dispatch

This document is written against m1n1 commit
`f5f8d2c7027d4ae37e4db27ea33720fe98151445` on 2026-07-27.

This document covers the T8110 DART XNU SPTM dispatch tables (t8140 registers
only these; the 26.5 GEN3/T6000 DART tables are absent, SK is not supported).

- **Table 3** `T8110_DART_XNU` — endpoints 0..16 (17)

The overall behavior of DART (and other IOMMUs) save for some MMIO is already
understood in m1n1; this document bridges the existing understanding to the
SPTM implementation.

While building the emulator, many, many XNU issues were traced to subtle DART
emulation issues, so be very careful when implementing this and it's likely a
good place to start for strange errors with no clear root cause.

## 1. Boot Handoff

Before XNU boots, walk `/arm-io` and seed one SPTM DART instance for each child
node with `compatible = dart,t8110` and a `dart-id`. 

- `dart-id`: logical DART object ID used by DART endpoints to select the
    appropriate DART instance
- `instance`: selects which `reg` entries are DART (`TRAD`), DAPF (`FPAD`), and
    PIOGW registers. Seed the DART base list, optional DAPF base, and optional
    PIOGW base list from those instance tags.
- `sid-count`, `sid`, and mapper child `reg`: define the SID set SPTM knows
    about for this DART. The default `sid` also supplies the fallback SID used
    for DAPF boot-premap ranges when there are no mapper children.
- `dart-options`: bit `0x10` enables dynamic SID assignment. For those DARTs,
    seed valid SID state for unallocated SIDs in `1..sid-count-1`; do not
    combine this with `remap`.
- `vm-size`, `vm-base-N`, and `vm-size-N`: seed per-SID DVA windows. If a SID
    has no per-SID override, use the node `vm-base`/`vm-size` window. Dynamic
    SIDs share the global window.
- `remap`: byte pairs mapping source SID to target SID. Seed the source SID as
    a remap SID instead of a translating SID.
- `bypass-N` and `apf-bypass-N`: seed SID state with DART bypass and/or APF
    bypass enabled.
- `exclave-sid`: seed these preprogrammed secure/exclave-owned streams as
    initially enabled, and reject any XNU requests.
- `pt-region-N`: SPTM treats this differently from plain m1n1. For a
    translating SID, the region start is the persistent root page-table PA, and
    the range is the allowed page-table carveout for that SID.
- `flush-by-dva`: seed the leaf-change flush policy. If present, map/unmap
    changes use DVA-range flushes instead of broader SID flushes.
- `avoid-tlbi-in-map`: records the native map-side TLBI policy. The bring-up
  emulator currently uses the conservative path and invalidates the SID for
  every changed leaf. It cannot safely skip invalid-to-valid updates until
  XNU's complete batched-map/drain contract is modeled; otherwise a cached
  negative translation can survive a fresh mapping.
- `relaxed-rw-protections`: seed DART permission-policy state. Current
  emulation records it but does not apply the full real-SPTM
  frame-type-dependent permission policy.
- `protection-granularity`: identifies newer DARTs whose exact subpage-field
  scaling is not yet recovered. Older nodes default to 4 bytes; T8142 APCIe
  declares 128 bytes and USB declares 16 bytes. For bring-up, the emulator
  deliberately disables subpage enforcement on these coarse-granularity
  instances by emitting the hardware-proven full-page range `0..0xfff`; PA and
  read/write permission bits remain unchanged.
- `piogw-ps-protection`: seed PIOGW power-state protection descriptors for
    DARTs with PIOGW `instance` entries.

SPTM also records DAPF boot-premap ranges from `dapf-instance-N` entries for
use when programming DAPF in endpoint 5.

Additionally, SPTM reads the following DART properties, but current emulation
does not model their state:

- `retention`
- `no-sleep`
- `ignore-secondary`
- `vm-alignment`
- `allow-pte-remap`
- `pte-remap-carveout-only`
- `real-time`
- `tz`
- `inclusive-tz-range`
- `dart-tunables-instance-N`
- `clock-protection-slice-index`
- `dart-ungang-shared-ps`
- `allow-dram-apf-slices-N`
- `n-apfs-instance-N`
- `allow-mixed-bypass-mode`
- `ignore-sid-count-mismatch`

## 2. Endpoints

SPTM exposes DART operations at a lower abstraction layer than m1n1's normal
`dart_map()` API. m1n1 combines table allocation and leaf population into one
in-house mapping operation; the SPTM ABI separates page-table installation from
leaf mapping, so XNU first gives SPTM table pages and later asks it to populate
or clear mappings inside that topology.

All endpoints take `x0 = dart-id`, selecting the DART object seeded during boot
handoff. In our emulator, all endpoints return `x0 = 0` indicating success. In
real SPTM, all violations immediately panic; the only possible outcomes are
success or fatal error.

SPTM keeps internal state for each DART object and SID: which SIDs are
configured, which ones translate, what root/remap state they use, and which
hardware streams should be enabled. Endpoint handlers update that state and use
it to decide what must be programmed, preserved, disabled, or restored across
init and power transitions.

Current boot traces heavily exercise endpoints 0, 2, 3, 5, 6, and 8. Endpoint
4 has only been observed in crash/recovery powerdown cases so far, not as part
of the normal happy path. We do not currently have normal-boot evidence for
endpoints 1, 7, or 9-16.

| id | name | in m1n1 | behavior | inputs |
|---:|------|----------|----------|--------|
| 0 | map table | mostly | Install a DART page-table page for a SID at the requested DVA/level. Root installs also program SID TCR/TTBR. | `x1`=sid, `x2`=dva, `x3`=level, `x4`=table PA |
| 1 | unmap table | mostly | Remove a root or child table for a SID. Removing the root clears translation state and TTBR for that SID. | `x1`=sid, `x2`=dva, `x3`=level |
| 2 | map page | mostly | Populate leaf PTEs from a PA-list page, one 64-bit PA per mapped DART page. | `x1`=sid, `x2`=dva, `x3`=PA-list PA, `x4`=size, `x5`=options |
| 3 | unmap page | mostly | Write zero to leaf PTEs for every 16 KiB page touched by a DVA range. | `x1`=sid, `x2`=dva, `x3`=size, `x4`=options |
| 4 | powerdown | mostly | Save stream-enable and optional performance-counter state, disable translating streams, and disable GAPF/PIOGW power protection. | no additional required args |
| 5 | powerup | mostly | Restore PIOGW/GAPF protection, tunables, TCR/TTBR, DAPF, errors, optional counters, and stream enables. | no additional required args |
| 6 | init | mostly | Read DART capabilities and update SPTM's internal state; no MMIO or page-table writes. | no additional required args |
| 7 | disable translation | yes | Unless `retention` is present, write the SID bit to `DISABLE_STREAMS` on every TRAD instance for a translating SID. | `x1`=sid |
| 8 | enable translation | yes | Write the SID bit to `ENABLE_STREAMS` on every TRAD instance for a translating SID, then issue a data-synchronization barrier. | `x1`=sid |
| 9 | clear legacy error | mostly | On hardware through version 2.1, clear one selected error bit on one TRAD instance and handle any pending secondary error. | `x1`=instance, `x2`=error bits |
| 10 | clear interrupt status | mostly | Clear selected bits in one TRAD instance's version-dependent interrupt-status register. | `x1`=instance, `x2`=status bits |
| 11 | clear all errors | mostly | Clear all pending error and interrupt status on every TRAD instance, using the layout selected during initialization. | no additional required args |
| 12 | read TLB entry | mostly | Read one indexed TLB entry from a selected TRAD instance and return its tag and PA result words. | `x1`=instance, `x2..x4`=TLB index fields |
| 13 | set legacy SMMU selector | no | On DART version 1.x, write a selector to the auxiliary SMMU associated with one TRAD instance. | `x1`=instance, `x2`=selector |
| 14 | set SMMU STT index | no | On later DART versions, write an STT index to every associated SMMU and wait for completion. | `x1`=STT index |
| 15 | zero transaction limits | no | Write zero to `TEQRESERVE` and `TLIMIT` on every TRAD instance. | no additional required args |
| 16 | clear selected exceptions | mostly | Read a 0x24-byte exception description and clear selected per-SID exceptions on one TRAD instance. | `x1`=exception-description pointer |

`id` is directly used in the emulator to route commands; `name` is what we
believe the endpoint is supposed to do; `in m1n1` describes if the behavior
already exists in upstream m1n1, mostly means it exists but needs some
refactoring to fit sptm's implementation.

### 2.1 Map and Unmap Table (endpoints 0, 1)

Endpoint 0 installs a DART page-table page for one SID. It determines the root
level from the SID configuration established during initialization.  For a SID
configured with `FOUR_LEVELS`, level 0 is the root; otherwise, level 1 is the
root.

When the requested level is the SID's root level, SPTM writes the supplied
table PA into the SID's TTBR. It does not modify the TCR, so existing policy
such as `BYPASS_DAPF` remains unchanged. When the requested level is deeper
than the root, SPTM walks the existing table tree to the requested DVA and
writes a table descriptor for the supplied page into the parent table.
Endpoint 0 does not populate leaf mappings; those are installed by endpoint 2.
After changing the table topology, SPTM performs the required DART TLB
invalidation.

Endpoint 1 unlinks a DART page-table page for one SID. Like endpoint 0, it
determines the root level from the SID's initialized TCR configuration.

When the requested level is the root level, SPTM writes zero to the SID's
TTBR. It does not modify the TCR, so the configured translation mode and
`BYPASS_DAPF` policy remain unchanged. The root table page itself is not
zeroed.

For a deeper level, it walks to the descriptor that points to the selected
child table and writes zero to that descriptor. It does not modify the contents
of the unlinked child table. It then invalidates the DVA range previously
covered through that descriptor.

### 2.2 Map and Unmap Pages (endpoints 2, 3)

These endpoints update leaf mappings inside the page-table topology created by
endpoint 0. Endpoint 2 maps one consecutive DVA range for one SID. `x3` is the
physical address of an array containing one 64-bit target PA for each 16 KiB
DART page covered by the range. The DVA and size need not be page-aligned;
SPTM encodes the valid portion of the first and last pages in the leaf
`SP_START` and `SP_END` fields.

It walks the page-table topology previously installed by endpoint 0 and writes
one leaf PTE for each PA-list entry. It does not allocate or install missing
intermediate tables; a missing table is fatal. A zero PA-list entry emits a
valid leaf mapping physical page zero.

The low four bits of endpoint 2's `x5` control the mapping.  Bit 0 sets
`RDPROT`, bit 1 sets `WRPROT`, and bit 2 sets `UNCACHABLE`. Bit 3 permits
changing the PA of an existing valid leaf, but only when that old leaf has
`WRPROT` set. Bit 3 also suppresses the endpoint's inline TLB invalidation.
When `relaxed-rw-protections` is active, SPTM forces `WRPROT` for frame types
14 (`XNU_USER_EXEC`), 15 (`XNU_USER_DEBUG`), 16 (`XNU_USER_JIT`), and 28
(`XNU_COPROCESSOR_RO_IO`).

Before writing each leaf, endpoint 2 also applies the TXM secure-channel rule
from section 3. On the SEP DART, the reserved TXM DVA range may only map the
TXM secure-channel frame, and that frame may only be mapped inside the reserved
range.

Endpoint 3 operates on every 16 KiB page touched by the requested range,
rounding the start down and the end up to page boundaries. It walks the
existing page-table topology and writes zero to each selected valid leaf. A
missing intermediate table is a violation. An already-zero leaf is also a
violation unless option bit 1 is set.

Option bit 0 is accepted only for DARTs with `allow-pte-remap`; it does not
change the hardware-visible unmap operation. Endpoint 3 rejects an attempt to
unmap a TXM secure-channel leaf.

After either endpoint changes a leaf, SPTM issues a barrier and invalidates the
affected page-rounded range on every TRAD instance. It uses a DVA-range or SID
invalidation according to `flush-by-dva`. Endpoint 2 may skip the invalidation
when no PTE changed, `avoid-tlbi-in-map` permits skipping a fresh install, or
option bit 3 requests deferred invalidation.

### 2.3 Power Down/Up (endpoints 4, 5)

Neither endpoint modifies any page-table page or leaf PTE.

Endpoint 4 performs the following operations:

1. Read every `ENABLE_STREAMS` word from each TRAD instance. Endpoint 5 writes
   these values back after the power transition.
2. If performance counters are enabled, read the counters at `0x760`, `0x764`,
   `0x768`, `0x770`, `0x774`, `0x778`, and, on supported hardware, `0x780`,
   `0x784`, and `0x788`. Endpoint 5 writes these values back.
3. For every configured translating SID, write its bit to `DISABLE_STREAMS` on
   each TRAD instance.
4. For every GAPF clock-protection slice associated with the DART, read its
   control word at `base + 0x100 + slice * 0x40` and write it back with bit 4
   set to zero.
5. For every descriptor in `piogw-ps-protection`, write zero to that PIOGW
   slice's control word.

Endpoint 5 performs the inverse hardware setup:

1. Program the PIOGW protection slices described below.
2. For every GAPF clock-protection slice associated with the DART, read its
   control word and write it back with bit 4 set.
3. Apply every entry from `dart-tunables-instance-N` as a masked update to the
   corresponding TRAD instance.
4. For every configured SID, program its TCR and TTBR on every TRAD instance.
   The TCR comes from the SID's boot configuration, including translation,
   bypass, remap, and page-table-level settings. The TTBR is the root installed
   or removed by endpoints 0 and 1.
5. Issue a full DART translation-cache invalidation after programming the SID
   registers.
6. On the first endpoint-5 call, read and retain `TLIMIT` and `TEQRESERVE`.
   After an endpoint-4 powerdown, write those same values back.
7. Program each DAPF instance from its `dapf-instance-N` slice descriptions.
   The existing m1n1 DAPF implementation provides the corresponding slice
   register layout.
8. Clear the DART error state.
9. If endpoint 4 saved performance counters, write those values back.
10. Write the boot-seeded stream-enable words on the initial power-up, or the
    `ENABLE_STREAMS` words read by endpoint 4 after a power transition.

If TCR, TTBR, tunable, or DAPF registers are hardware-locked, SPTM verifies
that they already contain the required values instead of writing them.

PIOGW bases come from `reg` entries whose matching `instance` tag is ` WGP`.
`piogw-ps-protection` is a list of descriptors containing a protection slice
number and a physical address. Protection slice registers start at
`base + 0x1120`, with stride `0x20`: control at `+0x00`, start low/high at
`+0x04/+0x08`, and end low/high at `+0x0c/+0x10`.

For each PIOGW base, only program the protection table if the 32-bit words at
`base + 0x0c` and `base + 0x10` are both zero. On power-up, program slice 0 as
a full-range entry with control value `3`. For each ADT descriptor, program the
selected slice to cover `[addr, addr + 3]` and write control value `6`, then
set bit 0 at `base + 0x4`. On power-down, clear the control word for each ADT
descriptor slice.

Exclave SIDs are preserved but not reprogrammed on XNU's behalf. In particular,
endpoint 5 may keep their initially enabled stream bits live, but it should not
install or rewrite their TCR/TTBR state from normal XNU requests.

### 2.4 Init (endpoint 6)

Endpoint 6 only updates SPTM's internal state. It performs no MMIO or
page-table writes.

SPTM reads the following information from the DART hardware:

- the hardware version and physical-address width from `PARAMS_8`;
- the number of supported SIDs from `PARAMS_C`;
- CTC-prefetch support from `PARAMS_4`;
- the auxiliary SMMU limit from the auxiliary SMMU register block on older
  hardware, or the TRAD register at offset `0x18` on newer hardware; and
- any hardware-locked TCR, TTBR, and APF registers needed to verify that their
  live values match the configuration established during boot.

From this information, SPTM records the common hardware version, the exclusive
physical-address limit, the SID capacity, the number of 32-bit words required
to represent all SIDs, CTC-prefetch support, the effective `flush-by-dva`
policy, the auxiliary SMMU limit, and the version-dependent register layout
used by later endpoints. It then initializes its internal asynchronous-operation
bookkeeping and marks the DART initialized.

Endpoint 6 does not import live TCR, TTBR, page-table root, or stream-enable
state.

### 2.5 Disable/Enable Translation (endpoints 7, 8)

These endpoints gate an already-configured translating SID at the
stream-enable registers. They do not change the SID's TCR, TTBR, root table,
leaf mappings, or any saved stream-enabled state.

For endpoint 7, if the DART does not have the `retention` property and the
SID's configured TCR has translation enabled, write the SID bit to
`DISABLE_STREAMS` on every TRAD instance. If `retention` is present, endpoint 7
performs no register writes.

For endpoint 8, if the SID's configured TCR has translation enabled, write the
SID bit to `ENABLE_STREAMS` on every TRAD instance and issue a
data-synchronization barrier. Endpoint 8 does not invalidate the DART
translation cache.

A known SID without translation enabled succeeds without modifying any
registers.

### 2.6 Error Handling (endpoints 9, 10, 11)

Endpoint 9 is supported only on DART hardware through version 2.1. `x1`
selects a TRAD instance. For versions 1.0 and 1.1, the permitted error bits are
`0x7fd`; for versions 2.0 and 2.1, they are `0x7fff`. Exactly one bit in
`x2 & permitted_bits` must be set. Write that masked value to the selected
instance's `ERROR` register.

Before clearing the selected error, read `ERROR`. If its secondary-error bit is
set, the behavior depends on `ignore-secondary`: without that property, the
error is fatal; with it, read the secondary-error word at offset `0x1c0` and
write the same value back to acknowledge the reported secondary errors.

On versions 2.0 and 2.1, if bit 1 of `DIAG_LOCK` is set after clearing the
error, issue a flush-unlock operation on every TRAD instance.

Endpoint 10 clears interrupt-status bits on one TRAD instance. `x1` selects
the instance. Mask `x2` with `0x777`; if the result is nonzero, write it to the
version-dependent interrupt-status register selected during endpoint 6. A zero
result performs no register write.

Endpoint 11 clears all pending status on every TRAD instance. On hardware
through version 2.1, perform the same secondary-error handling described above,
then read `ERROR` and write the same value back. On version 2.2 and later, read
and write back each 32-bit per-SID exception-status word beginning at offset
`0x4000`, using one word for every 32 supported SIDs. Finally, write
`0xffffffff` to the version-dependent interrupt-status register.

### 2.7 SMMU (endpoints 13, 14)

These endpoints operate on optional auxiliary SMMU register blocks associated
with the DART's TRAD instances. The functional meaning of the programmed
selector is not understood; only the observable register operations are
documented.

Endpoint 13 is supported only on DART versions 1.0 and 1.1. `x1` selects a
TRAD instance that has an associated SMMU register block. Read the 17-bit value
at SMMU offset `0x00`; `x2` must be less than that value rounded up in units of
`0x800`. Write `x2` to SMMU offset `0x80`.

Endpoint 14 is supported only on later DART versions. `x1` supplies an STT
index and must be below the SMMU limit recorded by endpoint 6. Write the low
16 bits of `x1` to offset `0x80` of every associated SMMU register block, then
poll each register until bit 31 is zero.

### 2.8 Misc (endpoints 12, 15, 16)

Endpoint 12 reads one indexed TLB entry from the TRAD instance selected by
`x1`. Construct the index as `((x2 & 0x3fff) << 8) |
((x3 & 0xf) << 4) | (x4 & 0x7)` and write it to `TLB_OP_IDX` at offset
`0x84`. Write `0x200`, selecting TLB operation 2, to `TLB_OP` at offset
`0x80`, then poll until its busy bit becomes zero.

The first returned result word is read at offset `0x88`. It is a zero-extended
32-bit value on version 1.x and a 64-bit value on later versions. The second
result word is the 64-bit value at offset `0x90`. The dispatch returns these as
the endpoint's two output words.

Endpoint 15 is available for DARTs with the `clamp-tlimits` property. It takes
no additional required arguments. For every TRAD instance, write zero to
`TEQRESERVE` at offset `0x22c`, followed by zero to `TLIMIT` at offset
`0x228`.

Endpoint 16 is supported on DART version 2.2 and later. `x1` points to a
0x24-byte structure in XNU-owned memory. The first four bytes contain the
32-bit TRAD instance index. Everything after that is an array of 32-bit SID
bitmaps: bitmap `i` covers SIDs `32 * i` through `32 * i + 31`.

For each nonzero bitmap word, write that word to the selected TRAD instance at
offset `0x4000 + 4 * i`. Each set bit acknowledges the corresponding pending
SID exception. Read back the final exception-status word after performing the
writes.

If bit 1 of `DIAG_LOCK` is set afterward, issue a flush-unlock operation on
every TRAD instance.

### 2.9 Deviations in Our Emulator

Unlike other tables, our DART emulator is full of hacks from trying to debug
various strange hardware problems. Thus, we document our behavior here. It
should likely not be replicated, but is here for future reference.

- **Endpoint 0:** The first level-0 or level-1 request becomes the SID root,
  and a later request at a shallower level may replace it. A root installation
  derives and writes a new TCR, enables translation, and writes TTBR. Child
  installations use a full SID invalidation, while root installation performs
  no separate invalidation. A missing parent descriptor is accepted without a
  write. To avoid the unavailable GXF cache operations, a child table outside
  `pt-region-N` is replaced with a table page inside that region.
- **Endpoint 2:** Missing intermediate tables may be installed using pages from
  `pt-region-N`; if that cannot be done, processing stops and the endpoint
  still returns success. A zero PA-list entry writes zero to the leaf instead
  of mapping physical page zero. Only option bits 0 and 1 affect the PTE:
  `UNCACHABLE`, the bit-3 replacement rule, bit-3 deferred invalidation, and
  the `relaxed-rw-protections` frame-type policy are not implemented.
- **Endpoint 3:** The options argument is ignored. Missing intermediate tables
  and already-zero leaves are accepted. Leaves recorded from
  `dapf-instance-N` are restored as identity mappings instead of being written
  to zero. Our emulator always invalidates a nonempty valid range.
- **Endpoint 5:** Our emulator programs the PIOGW and DAPF slices, reapplies
  ADT-seeded SID remaps and boot-premap identity leaves, writes the known TCR
  and TTBR values on every TRAD instance, writes the required stream-enable
  bits, sets bit 4 in each GAPF clock-protection slice, and flushes every SID
  whose registers or stream state it touched. It does not apply DART tunables,
  restore `TLIMIT`, `TEQRESERVE`, TrustZone state, or performance counters, or
  verify locked registers.
- **Endpoint 6:** Our emulator makes extensive state changes during
  initialization: it clears the DART error registers, reapplies the ADT-seeded
  SID remaps and boot-premap identity mappings, and reads live TCR, TTBR, and
  stream-enable state for use by endpoint 5. These are emulator shortcuts;
  SPTM's endpoint 6 only updates internal state and performs no MMIO or
  page-table writes.
- **Endpoint 8:** Our emulator records the stream as enabled internally and
  performs an additional full-SID invalidation after writing `ENABLE_STREAMS`;
  SPTM does neither.

## 3. TXM Secure Channel

Note: in the latest version of the SPTM emulator, this secure channel is never
allocated; rather, in the TXM emulator, we return not supported to XNU. This
information is left for informational purposes only, it likely isn't needed in
the emulator.

The SEP DART has a TXM secure-channel reservation. During boot handoff, record
`txm-secure-channel-base` and `txm-secure-channel-size` from `/arm-io/dart-sep`
if they are present. This defines a reserved SEP-DART DVA range; it is scoped
to the DART object for `/arm-io/dart-sep`, not to other DARTs that may use the
same numeric DVA range.

TXM also owns one physical secure-channel page. That page is frame type 61
(`TXM_SEP_SECURE_CHANNEL`) in the SPTM frame table, and TXM selector 7 returns
its PA and size to XNU. DART does not create that TXM state, but endpoint 2
uses the frame type to validate SEP-DART mappings.

For endpoint 2, if the secure-channel DVA range is present, enforce this rule
for every mapped leaf on the SEP DART: the reserved DVA range may only map
frame type 61, and frame type 61 may only be mapped inside the reserved DVA
range. A mismatch is an SPTM violation. Normal DARTs with the same numeric DVA
range are not affected.

For endpoint 3, do not remove an existing leaf whose target PA has frame type
61. Treat an attempt to unmap that leaf as an SPTM violation. Other unmaps
write zero to the selected DART leaf as described in section 2.2; DAPF state
does not change the DART unmap operation.
