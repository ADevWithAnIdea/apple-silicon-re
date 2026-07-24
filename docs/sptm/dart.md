# DART SPTM Dispatch

This document is written against m1n1 commit
`219b0fc1a84b4b55da120fee4d456b1e8d62d839` on 2026-07-03.

This document covers the T8110 DART SPTM dispatch tables (t8140 registers only
these; the 26.5 GEN3/T6000 DART tables are absent).

- **Table 3** `T8110_DART_XNU` — endpoints 0..16 (17)
- **Table 4** `T8110_DART_SK` — endpoints 0..3 (4) (= XNU table + IOMMU SK offset 1)

The overall behavior of DART (and other IOMMUs) is already understood in m1n1;
this document bridges the existing understanding to the SPTM implementation.

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
- `avoid-tlbi-in-map`: seed the map-side TLBI policy. If present, endpoint 2
    may skip the TLBI for fresh leaf installs, but still flushes when replacing
    an existing valid leaf.
- `relaxed-rw-protections`: seed DART permission-policy state. Current
    emulation records it but does not apply the full real-SPTM
    frame-type-dependent permission policy.
- `piogw-ps-protection`: seed PIOGW power-state protection descriptors for
    DARTs with PIOGW `instance` entries.

SPTM also records DAPF boot-premap ranges from `dapf-instance-N` entries. m1n1
already understands how to program DAPF, but SPTM additionally keeps these
ranges as firmware-owned identity coverage before XNU starts.

SPTM reads the following DART properties, but current emulation does not model
their state:

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
handoff. All endpoints return `x0 = 0` indicating success. All violations
immediately panic; the only possible outcomes are success or fatal error.

This differs from normal m1n1 DART use: m1n1 usually works with one
`dart_dev_t` for a hardware DART/SID pair, while SPTM first selects a logical
DART object by `dart-id` and then selects the SID inside that object. A clean
room implementation should bridge those object/SID endpoints to the existing
m1n1 T8110 table, TCR/TTBR, PTE, stream, and TLB helpers.

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
| 3 | unmap page | mostly | Clear leaf PTEs over a DVA range. Firmware-owned boot-premap leaves are restored rather than cleared. | `x1`=sid, `x2`=dva, `x3`=size |
| 4 | powerdown | mostly | Snapshot and disable translated streams, disable clock/protection side effects, and prepare the DART object for powerdown. | no additional required args |
| 5 | powerup | mostly | Program DAPF/PIOGW state, replay remaps/identity ranges/TCR/TTBR/streams, and enable clock protection. | no additional required args |
| 6 | init | mostly | Import or preserve live hardware state for this DART object's MMIO instances, clear stale errors, and apply bootstrap remaps/identity ranges. | `x1` may carry/recover an MMIO base |
| 7 | disable translation | yes | Disable a translating SID's stream in hardware and mark it disabled in SPTM state. | `x1`=sid |
| 8 | enable translation | yes | Enable a translating SID's stream in hardware, issue a barrier, and flush that SID. | `x1`=sid |
| 9 | clear err | mostly | Clear one bit in the DART error register on a selected hardware instance. | `x1`=instance, `x2`=single-bit error mask |
| 10 | error mask control | mostly | Program the DART error-mask register for a selected hardware instance. | `x1`=instance, `x2`=mask |
| 11 | clear all errors | mostly | Broadly clear DART-local error/status state across this object's hardware instances. | no additional required args |
| 12 | query tlb | mostly | Issue a DART TLB command on one hardware instance and poll completion. Also returns query result words in `x1`/`x2`. | `x1`=instance, `x2..x4`=TLB op fields |
| 13 | set smmu window | no | Real SPTM programs a window in a DART-associated SMMU instance. Ack only in our emulator. | `x1..x2`=control args |
| 14 | read smmu stt index | no | Real SPTM reads SMMU/STT state. Our emulator returns stored SID status in `x1`. | `x1`=sid/index |
| 15 | clamp tlimits | no | Real SPTM clamps translation limits. Ack only in our emulator. | `x1..x2`=control args |
| 16 | clear exception | mostly | Clear DART exception/error state across this object's hardware instances. | no additional required args |

`id` is directly used in the emulator to route commands; `name` is what we
believe the endpoint is supposed to do; `in m1n1` describes if the behavior
already exists in upstream m1n1, mostly means it exists but needs some
refactoring to fit sptm's implementation.

### 2.1 Map and Unmap Table (endpoints 0, 1)

These endpoints create and remove DART page-table topology for a SID. Endpoint
0 installs the supplied table page as either the SID root table or a child
table at the requested DVA/level; root installs also program the SID TCR/TTBR.
Endpoint 0 does not populate device mappings. Leaf PTEs are written later by
endpoint 2. Endpoint 1 removes the root or child table; root removal also
clears translation state and TTBR for the SID.

The `level` argument is `0..2`. `level = 0` installs or removes the root for a
FOUR_LEVELS stream. `level = 1` installs or removes the root for a normal
stream, but after a `level = 0` root is installed it selects the first child
table level. `level = 2` selects the next child table level. Leaf PTEs are
handled by endpoints 2/3.

Endpoint 0 installs a root only when the SID has no current root, or when the
requested level is above the current root. Otherwise it treats the request as a
child-table install under the existing root. In particular, after a `level = 0`
root is installed, later `level = 1` requests install child tables rather than
replacing that root.

Table installs/removals perform a SID flush. For SIDs seeded from
`pt-region-N`, table installs extend the imported persistent root tree instead
of replacing the boot-seeded root.

#### 2.1.1 GXF Instruction Handling

Additionally, our emulator has an extra behavior to work around the
unavailability of the GXF cache flush instructions. When DART's map table
endpoint is called to install a child table, we check to see if the page XNU
supplies is in pt-region-N.  If so, we use that page. If it is not in
pt-region-N, we supply our own page from that region. This prevents a cacheable
access to a NC page from ever being formed, sidestepping the issue.

### 2.2 Map and Unmap Pages (endpoints 2, 3)

These endpoints update leaf mappings inside the page-table topology created by
endpoint 0. Endpoint 2 walks from the SID root to the leaf, creates missing
intermediate table mappings from the seeded page-table region when possible,
and writes leaf PTEs for the requested DVA range. Endpoint 3 walks the same
topology and clears or restores leaf PTEs over the requested range.

Endpoint 2's PA-list is not part of the DART page-table tree. It is an
XNU-supplied array of 64-bit physical target pages, one entry for each DART
page covered by the DVA/size range. SPTM consumes that list and writes the
corresponding leaf PTEs into the page tables installed earlier.  A zero PA-list
is treated as a regular target PA and emits a valid leaf for physical page 0.

The DVA and size do not need to be 16K-aligned. If the range starts or ends in
the middle of a DART page, use the T8110 leaf start/end fields to encode the
valid subpage span. Unmap normally clears leaf PTEs, except that DAPF
boot-premap ranges seeded from `dapf-instance-N` are restored as identity
leaves instead, and existing TXM secure-channel leaves are rejected rather than
removed. The SEP TXM secure-channel reservation is handled separately; see
section 3.

Endpoint 2 maps `x5` option bits directly onto existing T8110 leaf protection
bits. Using zero-based bit numbering, bit 0 sets `RDPROT` and bit 1 sets
`WRPROT`; other `x5` bits do not currently change the emitted PTE. The valid
bit comes from the mapping itself, and these options do not set `UNCACHABLE`.

Before writing each leaf, endpoint 2 also applies the TXM secure-channel rule
from section 3. On the SEP DART, the reserved TXM DVA range may only map the
TXM secure-channel frame, and that frame may only be mapped inside the reserved
range.

Leaf map/unmap changes flush every TRAD base attached to the selected DART
object when a flush is required. If `avoid-tlbi-in-map` was seeded from ADT,
endpoint 2 may skip the TLBI for fresh leaf installs, but still flushes when
replacing an existing valid leaf. Endpoint 3 unmaps flush the affected range.
If `flush-by-dva` was seeded from ADT, use a DVA-range flush; otherwise use a
broader SID flush.

### 2.3 Power Down/Up (endpoints 4, 5)

These endpoints bracket DART power transitions. Endpoint 4 prepares a DART
object for powerdown.  Endpoint 5 reprograms hardware-visible state after
powerup from the state seeded at boot and updated by earlier endpoints.

Endpoint 4 snapshots the currently enabled streams into per-SID state, then
disables translated non-exclave streams in each TRAD instance. It also clears
modeled sideband power/protection state such as PIOGW power-state protection
controls and clock-protection state. Page-table roots, leaf mappings, remaps,
boot-premap ranges, and SID policy state remain owned by SPTM and are kept for
later replay.

Endpoint 5 programs ADT-seeded sideband state first: DAPF slices from
`dapf-instance-N` and PIOGW protection descriptors from `piogw-ps-protection`.
It then reapplies bootstrap remaps and firmware-owned identity/premap ranges,
replays TCR/TTBR state for non-exclave SIDs that require hardware programming,
and enables all streams which were internally marked as enabled.

If a DART node has PIOGW `instance` entries and `piogw-ps-protection`, endpoint
5 programs the PIOGW protection table and endpoint 4 clears the programmed
descriptor slices.

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

m1n1 understands all required DART concepts, all SPTM does differently is
include sufficient bookkeeping to tear down and restore DART state across power
transitions and the aforementioned PIOGW mmio accesses.

### 2.4 Init (endpoint 6)

Endpoint 6 initializes a DART object against its current hardware state.  It
makes SPTM's per-DART/per-SID state match the TRAD instances, clears stale DART
error state, and reapplies boot-seeded remaps and identity/premap ranges.

For each known TRAD base, read the hardware stream count, enabled-stream
bitmap, TCR/TTBR registers, and TTBR/TCR protection state. If the instance is
protected or already contains live translation/remap state, preserve that
hardware state instead of treating the DART as reset. Imported SID state should
include TCR, TTBR, root PA, root level, translation-enabled state, and whether
the stream was enabled. Reset-looking unused SID slots should remain
unconfigured.

After live-state import, clear DART-local error/status state on each TRAD base.
Then replay the ADT-seeded SID remaps and boot-premap identity ranges for this
DART object. This is the same boot-owned state replayed by endpoint 5, but
without the full power-up sideband sequence.

`dart_init()` in m1n1 has much of the required logic, but it should be adapted
to preserve firmware provided roots for later use, reapply SID remap
configuration, and restore boot-seeded identity leaves for firmware owned
premap ranges.

### 2.5 Disable/Enable Translation (endpoints 7, 8)

These endpoints gate an already-configured translating SID at the stream-enable
level. They do not change the SID's TCR, TTBR, root table, or leaf mappings.

Endpoint 7 disables a translating SID by internally recording that the stream
should be disabled and writing the SID bit to each TRAD instance's
disable-stream register. Endpoint 8 enables a translating SID by internally
recording that the stream should be enabled, writing the SID bit to each TRAD
instance's enable-stream register, issuing a barrier, and flushing that SID.

Requests for unknown SIDs or exclave SIDs are violations. Requests for known
SIDs that do not have translation enabled succeed without programming stream
state.

This logic is effectively the same as existing m1n1 code, except keyed by
`dart-id` and SID. SPTM also updates its internal bookkeeping to save which
streams should be enabled across power transitions.

### 2.6 Error Handling (endpoints 9, 10, 11)

Some internal SPTM names in this area refer to interrupt clearing, but most
names and all observed side effects are DART error handling.

These endpoints operate on DART-local error registers. They do not change
page-table state, SID translation state, stream-enable state, or boot handoff
state.

Endpoint 9 clears one error bit on one TRAD instance. `x1` selects the TRAD
base index inside the selected DART object. `x2 & 0x7fff` must contain exactly
one bit; the handler writes that bit to the instance error register.

Endpoint 10 programs the error-mask register for one TRAD instance. `x1`
selects the TRAD base index and `x2 & 0x777` supplies the mask. Endpoint 11
clears DART-local error/status state across every TRAD base in the selected
DART object.

The error bits are the same as in `dart8110.py`.

### 2.7 SMMU (endpoints 13, 14)

SMMU instance state is not currently modeled. That may need real emulation
later, but acking the SMMU endpoints has been sufficient through late userspace
boot with WindowServer alive and GPU initialization reached.

### 2.8 Misc (endpoints 12, 15, 16)

Endpoint 12 is a direct TLB-command helper for one TRAD instance. `x1` selects
the TRAD base index. It packs `x2..x4` into the command word as `((x2 & 0x3fff)
<< 8) | ((x3 & 0xf) << 4) | (x4 & 0x7)`, writes that to `TLB_OP`, and polls
until the busy bit clears. It then returns the raw TLB result registers:
`x1 = TLB_TAG_LO` and `x2 = TLB_PA_LO`. `dart8110.py` already defines the
underlying TLB registers.

Endpoint 15 is not currently modeled. Real SPTM appears to clamp translation
limits; our emulator records the call arguments and returns success.

Endpoint 16 clears DART exception/error state across every TRAD base in the
selected DART object. In current emulation this is equivalent to the broad
error clear path used by endpoint 11.

## 3. TXM Secure Channel

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
61. Treat an attempt to unmap that leaf as an SPTM violation. Other unmaps keep
the normal behavior described above: restore DAPF boot-premap leaves where
applicable, otherwise clear the leaf.
