# UAT SPTM Dispatch

This document is written against m1n1 commit
`f5f8d2c7027d4ae37e4db27ea33720fe98151445` on 2026-07-27.

This document covers the UAT SPTM dispatch table, which manages the page tables
for the AGX GPU's address-translation unit, as well as SAPT, a novel security
feature present on T8140.

- **Table 7** `UAT` — endpoints 0..12 (13)

The GPU's UAT tables are ARM64-style 16K-granule page tables, already
understood in the GPU RE work; this document bridges that understanding to the
SPTM implementation.

**NOTE:** our emulator has an issue where it hangs very quickly after boot
makes it to the login screen. We believe this documentation is largely correct
despite this shortcoming. If we discover the root cause of this issue, this
document will be updated with that information.

## 1. SAPT

SAPT is a hardware-enforced permission table that governs which physical pages
the GPU may access. We believe it acts as a sort of defense in depth against
a misbehaving GPU even if the page tables are configured incorrectly.

m1n1 has no understanding of SAPT; we describe its behavior in full, though in
practice we expect to configure it at boot to be fully permissive then never
touch it again.

### 1.1 Boot Handoff

Seed one SAPT table from `/arm-io/sapt` and `/chosen`:

- `table-address`: physical base of the SAPT table. SPTM records this range as
  firmware-owned before XNU starts.
- `n-entries`: the number of entries — one 2-bit field per 16K DRAM page (the
  DRAM span ÷ 16K).
- `policy`: selects one of three permission policies (section 1.2).
- `/chosen` `dram-base` and `dram-size`: the physical DRAM window SAPT covers.

The table is a bit-packed array covering all of DRAM at two bits per 16K
physical page, i.e. four pages per byte. The byte for a page is found by its
offset from `dram-base` in 64K units, and the two-bit field within that byte is
selected by which of the four 16K pages it is.

Each field is a two-bit GPU access permission in which each set bit denies an
access: the low bit denies write and the high bit denies read. Clearing both
gives full read/write access, and setting both denies all access.

### 1.2 Policies

The `policy` property selects one of three rule sets. On retype, all of them
set a page's permission from its frame type: the GPU firmware's data, shared,
and handoff regions become read/write, the GPU's own page-table regions become
read only, and all other DRAM becomes no access. They differ in how a later map
or unmap changes permissions:

- **Policy 0** makes no changes on map or unmap; the retype-assigned
  permissions stand.
- **Policy 1** grants each mapped page the access its map options request, and
  requires the page to be no-access first — mapping an already-granted page is
  a firmware-alias violation. Unmap returns pages to no access.
- **Policy 2** grants each mapped page the access its map options request, only
  ever adding access, and does not flag an already-granted page. Unmap leaves
  permissions in place; a page returns to no access only when it is retyped or
  freed.

### 1.3 Publishing Changes

Real SPTM publishes every SAPT change to the fabric using the GXF-guarded
cache-maintenance instructions described in the XNU bootstrap document.

Our emulator performs the SAPT changes then simply skips the guarded flush. So
far this has worked fine (we don't believe the hang is due to this and will
update the doc if this turns out to be false). It is likely that an emulator
could simply give the GPU access to all memory rather than faithfully
implementing SAPT.

## 2. Boot Handoff

Properties that m1n1 already understands and that SPTM uses the same way are
omitted here; only the SPTM-specific properties are listed.

Before XNU boots, seed one UAT state object from `/arm-io/sgx`, `/defaults`, and
`/chosen`. The SAPT table (section 1) is seeded at the same time.

Region properties come as `-base`/`-size` pairs — a physical address and a byte
count.

- `agx-address-space-mgmt-mode`: selects how the GPU's shared upper (TTBR1) half
  is managed. Mode 0 shares one kernel table across all contexts; mode 1 gives
  each context its own table with only a small shared window.
- `gfx-shared-l2-region-base`/`-size`: the shared upper-half (TTBR1) table,
  exactly one 16K page.
- `uat-vaddr-size`: the GPU virtual-address width in bits, which sets the
  top-level index mask. Defaults to a 40-bit width (shift 39).
- `uat-segment-limit`: the largest segment list one map or unmap may carry; it
  also sizes the state object. Defaults to 64.
- `uat-mapping-limit`: the largest number of pages a single begin/continue step
  processes before yielding. Defaults to 256.

SPTM reads the following, but current emulation does not model them:

- `issue-gmmu-tlbis-at-retype`: whether to invalidate the GPU MMU TLB when a
  page is retyped. This target enables it; the emulator does not yet issue that
  invalidation, which is a suspected cause of the hang noted above.
- `uat-enforce-gpu-carveout`: whether to restrict GPU virtual addresses to a
  carveout window. Off on T8140.
- `gpu-iouat`: the UAT enable gate; when absent, UAT is enabled.
- `gfx-data-*`, `gfx-data-shared-ro-*`, `gfx-data-shared-rw-*`, and the `gfx1-`
  variants: GPU-firmware data-segment carveouts.

## 3. Endpoints

SPTM splits each map and unmap into a *begin* endpoint and a *continue*
endpoint. The begin endpoint is a minimal shim that sets up the operation and
tail-calls the continue body. That body populates up to a fixed number of pages
(`uat-mapping-limit`, 256 by default); if work remains it returns a continue
result, and XNU re-enters through the matching continue endpoint until the
operation finishes.

Every endpoint identifies its target state object by physical address in `x0`
(section 3.1); `get_info` is the exception, taking a query selector in `x0`
instead. Every `va` below is a GPU virtual address.

| id | name | in m1n1 | behavior | inputs | output |
|---:|------|---------|----------|--------|--------|
| 0  | `init_state`                | no     | Create a UAT state object for a root (TTBR0/TTBR1) pair. | `x1` root0 PA, `x2` root1 PA (`0xffffffff` if single-root) | success |
| 1  | `destroy_state`             | no     | Tear down a root's state object. | — | success |
| 2  | `map_table`                 | mostly | Install a caller-supplied intermediate page-table descriptor. | `x1` va, `x2` level, `x3` table PA | success |
| 3  | `unmap_table`               | no     | Remove an intermediate page-table descriptor. | `x1` va, `x2` level | success |
| 4  | `map_page_begin`            | mostly | Begin populating leaf mappings from a segment list. | `x1` start va, `x2` segment-list PA, `x3` segment count, `x4` options | `0` done, `1` continue |
| 5  | `map_page_continue`         | no     | Resume a map-page operation. | — | `0` done, `1` continue |
| 6  | `prepare_fw_unmap_begin`    | mostly | Begin marking leaves firmware-owned and publish the handoff. | `x1` va, `x2` page count | `0` done, `1` continue, `2` flush requested |
| 7  | `prepare_fw_unmap_continue` | no     | Resume a prepare-firmware-unmap. | — | `0` done, `1` continue, `2` flush requested |
| 8  | `unmap_page_begin`          | no     | Begin clearing leaf mappings over a segment list. | `x1` segment-list PA, `x2` segment count | `0` done, `1` continue |
| 9  | `unmap_page_continue`       | no     | Resume an unmap-page operation. | — | `0` done, `1` continue |
| 10 | `set_ctx_id`                | yes    | Bind a state object's roots into a GPU context slot. | `x1` context id | success |
| 11 | `remove_ctx_id`             | mostly | Unbind a GPU context slot. | — | success |
| 12 | `get_info`                  | mostly | Return a queried UAT parameter. | `x0` selector | queried value |

### 3.1 {Init, Destroy} State (Endpoints 0, 1)

A UAT *state object* is the control block for one GPU address space. It is a
caller-allocated page whose physical address is the `x0` handle every
endpoint uses; it holds the address space's roots and identity plus the working
state of any in-progress map or unmap.

Its layout — total size `segment_limit * 0x10 + 0x250`, which fits one 16K page
(offsets not listed are reserved):

| offset | size | field | meaning |
|--------|-----:|-------|---------|
| `0x00` | 1 | type | 1 = TTBR0 only, 4 = dual TTBR0+TTBR1 (2 and 8 are SPTM's shared-TTBR1 globals) |
| `0x08` | 8 | root0 | TTBR0 root-table PA (`0xffffffff` when unbound) |
| `0x10` | 8 | root1 | TTBR1 root-table PA (dual only) |
| `0x18` | 2 | context id | bound GPU context 0–63; `0xffff` when unbound |
| `0x1a` | 1 | guard | operation / publish state (below) |
| `0x20`–`0x38` | 4×8 | work cursor | resume state for an in-progress operation |
| `0x40` | 4 | options | map permission and attribute flags |
| `0x44` | 1 | flush level | accumulated SAPT flush level for the operation |
| `0x48` … | 16·N | segment list | up to `segment_limit` `{first, count}` entries |
| `0x248` | 8 | unmap flush count | SAPT pending count during unmap |
| `0x250` … | 16·N | segment list | up to `segment_limit` `{first, count}` entries |

Types 2 and 8 are boot-created global state objects. In mode 0, the type-2
object supplies the shared TTBR1 used when a type-1 object is bound to a
context. Endpoint 12 selector 1 returns this global object's physical address.
In mode 1, the global object has type 8 and endpoint 12 selector 2 returns its
physical address; ordinary type-4 objects supply both of their own roots.

The guard byte tracks the object's state: `0` uninitialized, `1` busy,`2`
idle/ready, `3` map in progress, `4` prepare-firmware-unmap in progress, `5`
unmap in progress.

There are two segment-list fields: `map_page`'s at `0x48` ({physical base,
count} runs) and `unmap_page`'s at `0x250` ({virtual base, count} ranges), each
up to `segment_limit` entries. They sit at different offsets but overlap within
the page (a long map list runs past `0x250`), which is harmless because an
object only ever runs a map or an unmap at once. Thus, the page never needs
room for both at full size.

`init_state` installs the supplied roots in the state object. The roots must
already be retyped to the `XNU_IOMMU` (type 24) page-table type. It increments
the `iommu_refcount` on each root and on the object page itself, so they cannot
be freed or repurposed while the address space is live; our emulator does not
model this frame-table bookkeeping.

`destroy_state` reverses `init_state`. It requires the object to be unbound
first (so `remove_ctx_id` must run before it), then releases the pins
`init_state` took by decrementing the `iommu_refcount` on each root (and on the
object page itself), so those pages can be freed or retyped again. Finally it
resets the roots to the `0xffffffff` sentinel and guard to 0.

Binding to a GPU context is a separate step (`set_ctx_id`), described in
section 3.4.

### 3.2 {Un}Map {Table, Page} (Endpoints 2, 3, 4, 5, 8, 9)

These are the core endpoints that manage the GPU page-table tree.  `map_table`
walks from the state object's root to the slot for the given `va` at the
requested `level` (1 or 2) and writes a table descriptor pointing at the
caller-supplied table page, which — like the roots — must already be an
`XNU_IOMMU` frame. `unmap_table` walks to the same slot, clears its descriptor,
and invalidates its range. Both endpoints increment/decrement `iommu_refcount`
of both the table and its parent respectively.

`map_page` writes the leaf entries that back a run of GPU pages, and
`unmap_page` clears them. Each takes a *segment list* rather than a single
range: for `map_page`, a start `va` and a list of {physical base, page count}
runs laid onto a contiguous virtual range; for `unmap_page`, a list of {virtual
base, page count} ranges. The begin endpoint first copies that list into the
state object (bounded by `uat-segment-limit`), then processes it in
`uat-mapping-limit`-page chunks, resuming from the copy (location determined by
the cursor) on each continue; both drive the SAPT updates described in section
1.

`map_page_begin` accepts option bits 0–3, 8–9, and 16–19; all other option bits
must be zero. Bits 16–19 have no effect on the leaf descriptor. Combinations
marked `invalid` below are rejected. These checks can be skipped in an
emulator.

`map_page` constructs each leaf using the following values:

| `Page_PTE` field | value |
|---|---|
| `OFFSET` | physical address from the segment |
| `VALID`, `TYPE`, `OS`, `AF` | 1 |
| `SH` | 0 |
| `nG` | 1 for state-object types 1 and 4; 0 for global types 2 and 8 |
| `AttrIndex` | selected below |
| `AP`, `PXN`, `UXN` | selected below |

`AttrIndex` is selected as follows:

| options | `AttrIndex` |
|---|---:|
| bit 2 set | 1 |
| bit 2 clear, bit 3 set | 2 |
| bits 2 and 3 clear | 0 |

The remaining permission fields are selected by options bits 1:0 and 9:8. Each
cell is `AP, PXN, UXN`:

| bits 1:0 \ bits 9:8 | `00` | `01` | `10` | `11` |
|---|---|---|---|---|
| `00` | `0,0,0` | `2,0,0` | `2,1,0` | `2,0,1` |
| `01` | `1,0,0` | `0,1,0` | invalid | invalid |
| `10` | `1,1,0` | invalid | `0,0,1` | invalid |
| `11` | `1,0,1` | `1,1,1` | invalid | `0,1,1` |

`unmap_page` writes zero to each leaf descriptor's valid and type bits, leaving
its physical address and other fields unchanged. After ordering those writes,
it invalidates the UAT translations covering each segment. If endpoints 6 or 7
created a firmware handoff for the operation, it then writes `0` to the low 32
bits of the bound context's handoff-slot state field and issues an
outer-shareable memory barrier. Finally, it decrements the `iommu_refcount`
associated with each unmapped frame.

The guarded per-page or global operations performed here publish the
corresponding SAPT permission changes and are described in section 1.3.

Our emulator instead writes `0` to the entire leaf descriptor, cleans that
write to the point of coherency, and performs a broad UAT invalidation. It
applies the SAPT permission change but substitutes a point-of-coherency clean
for the guarded publication operation. It does not write `0` to the
handoff-slot state field or model the frame-reference accounting.

Similar to DART, m1n1 fully understands the page-table format, but provides a
higher layer of abstraction (combining table + leaf installation) than SPTM
provides XNU.

### 3.3 Prepare Firmware Unmap (Endpoints 6, 7)

For firmware owned leaves, these endpoints are used to request the firmware
release the pages so they can be unmapped later. There are two parts to these
endpoints: walking the page tables to mark pages as firmware owned, and then
submitting a handoff to notify the GPU to stop using the pages.

Over the range (`x1` va, `x2` page count, in `uat-mapping-limit` chunks) walk
to each leaf.  They check for leaves with the AttrIdx=0 and its permission bits
are either AP 0 with UXN set or AP 1 with PXN or UXN set. For these leaves,
they set bit 3 to mark them as firmware owned.

When the walk finishes it posts the range into the firmware handoff page: the
slot for the bound context receives the range's address and size, and a state
marking it either a flush request (when the firmware is live) or an unmap
notice (when it is not). It requests a flush only when the firmware is
accepting and the pages could still be cached — a shared context, or the bound
context currently running on the GPU — otherwise it goes straight to the unmap
notice.  The firmware then flushes its view and acknowledges, after which XNU
issues the `unmap_page`.

The page table walk is not in m1n1; the handoff is already fully understood
(`GFXHandoff`).

### 3.4 {Set, Remove} Context ID (Endpoints 10, 11)

`set_ctx_id` makes the address space live by writing its two roots into the
selected hardware context slot. For a type-1 state object, it installs the
object's TTBR0 and the shared TTBR1 from the boot-created global mode-0 state
object. For a type-4 state object, it installs both TTBR0 and TTBR1 from the
object.

It also records the context ID in the state object, issues the required
barrier, returns the guard to idle, and publishes the updated cache lines. The
context ID must be below 64, the object must be unbound and idle, and both
entries in the selected hardware context slot must be invalid.

`remove_ctx_id` writes the valid bit of both hardware context entries to 0.
When `uat-enforce-gpu-carveout` is disabled, it issues an ASID-wide GMMU TLB
invalidation; when enforcement is enabled, it instead invalidates the permitted
GPU virtual-address window. It then writes `0xffff` to the state object's
context ID, returns the guard to idle, and publishes the updated cache lines.

These operations are already basically understood by m1n1.

### 3.5 Get Info (Endpoint 12)

`get_info` is a stateless query. Unlike other calls, `x0` is a query selector
rather than a state object. It returns a single UAT parameter, letting XNU read
the boot-seeded geometry instead of hardcoding it.

| selector | returns |
|---:|---|
| 0 | the address-space management mode |
| 1 | physical address of the global state object in shared mode / mode 0 (invalid otherwise) |
| 2 | physical address of the global state object in per-context mode / mode 1 |
| 3 | the virtual-address shift (VA width − 1) |
| 4 | the top-level index mask |
| 5 | the segment limit |
| 6 | lower bound of the permitted GPU virtual-address window |
| 7 | upper bound of the permitted GPU virtual-address window |
| 8 | the state-object size |
| 9 | XNU/PAPT virtual address mapping the shared TTBR1 table whose physical address is `gfx-shared-l2-region-base` |
