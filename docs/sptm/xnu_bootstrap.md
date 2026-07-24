# XNU Bootstrap -- Clean-Room Functional Specification (DRAFT)

This document covers the boot handoff and the XNU_BOOTSTRAP table (domain 0,
table 0).

Some values below are from the KDK 26.5 (25F71) published SPTM headers.

---

## 1. Boot handoff

We do the following during the SPTM --> XNU boot transition:

1. Fabricate the initial page tables (1.1).
2. Populate the handoff structs (1.2).
3. Program the guest EL1 system registers (1.3).
4. Jump to the kernelcache's `_start` (1.4).

### 1.1 Initial page tables

The page tables are standard ARMv8 stage-1 tables, 16K granule, TTBR1 walk
starting at Level 1 (TCR.T1SZ=17); descriptors per the ARM ARM.

Geometry is 16K everywhere by default; the per-granule index slicing is the
standard ARM walk. The only 4K roots are sub-page user roots (2.3.7) and the
user roots backing 4K address spaces (x86/Rosetta). A root's geometry is
recorded per-root when it is created (the pt-attr index passed to RETYPE/SURT).

We fabricate a small set of initial mappings for XNU:

- **TTBR0** contains a single zeroed page at VA 0
- **TTBR1** contains the high-half kernel map (below).

Two leaf attribute presets:

| preset | leaf bits | meaning |
|--------|-----------|---------|
| code | `0x603` | AttrIndx 0, kernel-RW, executable |
| data | `0x60000000000703` | same, but execute-never (PXN+UXN) |

Only the kernelcache image is mapped as code, everything else as data.  We map
the kernel RWX out of convenience. Real SPTM would never permit this mapping.

**TTBR1 windows** (placement VAs are our choice except where noted):

| window | maps | preset |
|--------|------|--------|
| kernelcache image | kc VA range (real vmin..vmax) --> kc PA | code |
| aux carveout | handoff scratch (structs, page-table pool, CPU/TXM stacks) | data |
| iBoot boot args | the `boot_args` page (1.4 `x1`) | data |
| device tree (ADT) | the ADT blob, just below the kernelcache VA | data |
| physmap | the kernel's linear VA window over SPTM-managed RAM; bounds must match physmap base/end in 1.2 (rel 0x28/0x30) | data |
| UAT L2 | GPU shared-region L2 page table; PA from the ADT `/arm-io/sgx` `gfx-shared-l2-region`; see uat.md | data |

We also pre-allocate empty L3 tables over a dynamic region above the physmap
and a high top range, so XNU can fill those leaves later without allocating
tables.

The TTBR1 root PA goes in the nested `libsptm` struct at (rel 0x38); we also
program TTBR0/TTBR1/TCR/MAIR/SCTLR into the guest EL1 registers.

### 1.2 Handoff structs

#### `sptm_bootstrap_args_xnu_t` (passed in `x2`)

Size 0x340.  XNU snapshots it into a RO-late kernel global on
entry to `arm_init`, so the buffer only needs to be valid across the call.
Zero the buffer first; set only the fields below.

| off | size | short description | what it does OR the exact value it should be|
|----:|-----:|-------|----------------------------|
| 0x00 | 8 | per CPU scratch page | VA base of a page used by certain SPTM endpoints to return information to XNU. One page per CPU (slot = `base + 16K * cpu_id`). Map kernel RW + cacheable, contiguous, reserved (below the XNU-managed RAM start) |
| 0x08 | 8 | physmap base VA | start VA of the physmap, the kernel's linear VA window onto SPTM-managed RAM (a page at offset N into it appears at physmap_base + N); lets XNU reach any physical page by VA |
| 0x10 | 8 | physmap end VA | end VA of the physmap window |
| 0x18 | 8 | XNU-managed RAM start | first PA XNU can allocate, past the reserved front (m1n1 + kc + ADT + boot args) |
| 0x30 | 8 | TXM thread-stack array base | VA base of the TXM thread-stack array. see `txm.md` |
| 0x38 | 4 | TXM thread-stack count | count of TXM thread stacks, one per CPU |
| 0x40 | 8 | per-CPU kernel-stack window start | start VA of XNU's per-CPU kernel-stack window (physmap VAs). We reserve a single 1 MiB range which XNU subdivides across CPUs. Map kernel RW + cacheable, execute-never, contiguous, reserved (aux carveout, below the XNU-managed RAM start). |
| 0x48 | 8 | per-CPU kernel-stack window end | end VA (exclusive) of that window |
| 0x50 | 8 | executables window start | kernelcache image start VA |
| 0x58 | 8 | executables window end | kernelcache image end VA |
| 0x60 | 8 | debug header | VA of a fabricated `debug_header_t`, see note below |
| 0x68 | 4 | ASID count | ASID count for XNU's pmap; copied straight from ADT `/defaults` `pmap-max-asids` |
| 0x6c | 264 | random seed | the ASCII string `"randseed"` (8 B, no NUL) + 256 random bytes |
| 0x178 | 8 | random seed length | exactly `0x108` (8 + 256) |
| 0x180 | 1 | exclaves enabled flag | 0 - disables exclaves/SK |
| 0x198 | 8 | XNU panic flag | VA of a 1-byte guest scratch (XNU writes `1` here on panic) |
| 0x1a0 | 312 | nested SPTM client state | see next section |
| 0x2e8 | 8 | AuxKC end | top VA of the Auxiliary Kernel Collection. We ship no AuxKC, but XNU derives the overall top-of-kernelcache (its `end_kern` / `vm_kernelcache_top`) from this field, so set it to the kernelcache image end VA, 0 would break XNU's kernelcache bounds |
| 0x318 | 8 | pmap-io-ranges table pointer | VA of the pmap-io-ranges policy table: a sorted array of 24-byte records (addr u64, size u64, flags u32, signature u32), one per protected IO range, copied from ADT `/defaults/pmap-io-ranges`. XNU's pmap consults it directly instead of re-parsing the device tree |
| 0x320 | 4 | pmap-io-ranges count | number of 24-byte records in the table above |
| 0x328 | 8 | pmap-io-filters table pointer | VA of the pmap-io-filters table: a sorted array of 8-byte records (signature u32, offset u16, length u16), copied from ADT `/defaults/pmap-io-filters` |
| 0x330 | 4 | pmap-io-filters count | number of 8-byte records in the filters table |
| 0x338 | 8 | feature flags | hardcode `0x10` (the value that boots; bit meanings not reverse-engineered) |

The debug header (0x60) is unconditionally dereferenced on the cold-boot path,
so we allocate and zero initialize a `debug_header_t` and set:

| off | size | field | what we put here |
|----:|-----:|-------|----------------------------|
| 0x00 | 4 | `magic` | `0x47424544` (`'GBED'`) |
| 0x04 | 4 | `version` | `2` |
| 0x08 | 4 | `count` | `3` |
| 0x10 | 24 | `image[0..2]` (SPTM/XNU/TXM) | inner XNU kernelcache (`com.apple.kernel`) Mach-O header VA in all three slots |

Reusing the XNU Mach-O for all three slots works because the only cold-boot
check is that each image's `__TEXT` segment `vmaddr` is non-zero.  The values
are otherwise only consulted at panic time for associating SPTM and TXM
addresses with symbols, which we don't support.

Full layout information in the APSL2 licensed `sptm_xnu.h` in the KDK

#### `libsptm_state` (nested at +0x1a0, 312 B)

| rel | abs | short description | what we put here |
|----:|----:|-------|----------------------------|
| 0x00 | 0x1a0 | version | `10` (the layout version this build implements; XNU selects field offsets by it) |
| 0x08 | 0x1a8 | physical-aperture range count pointer | VA of a `u32` holding the number of physical-aperture ranges |
| 0x10 | 0x1b0 | physical-aperture range array pointer | VA of the physical-aperture range array (layout below) |
| 0x18 | 0x1b8 | SPTM-managed RAM start | start PA of the RAM SPTM manages (physmap + frame table cover it) |
| 0x20 | 0x1c0 | SPTM-managed RAM end | end PA of the SPTM-managed range |
| 0x28 | 0x1c8 | physmap base VA | identical to the bootstrap args at 1.2 (0x08) |
| 0x30 | 0x1d0 | physmap end VA | identical to the bootstrap args at 1.2 (0x10) |
| 0x38 | 0x1d8 | root page-table physical address | PA of the stage-1 root table we fabricate (1.3) |
| 0x40 | 0x1e0 | frame table pointer | VA of the frame table (one entry per managed 16K frame; layout below) |
| 0x48 | 0x1e8 | frame-type params pointer | VA of the frame-type params table (256 entries; layout below) |
| 0x50 | 0x1f0 | page-table attr table pointer | VA of the pt-attr pointer table (6 kernelcache symbol VAs; layout below) |
| 0x58..0x78 | | cpu features, IO-range count, IO frame table, tag-storage addresses | left zero |
| 0x80 | 0x220 | panicking-CPU-id slot pointer | VA of a u16 panicking-CPU-id slot |
| 0x88 | 0x228 | trace buffer pointer | VA of the SPTM trace buffer |
| 0x90 | 0x230 | per-CPU dispatch-state array | VA of the per-CPU dispatch-state array |
| 0x98 | 0x238 | max CPU count | boot CPU count |
| 0xa0 | 0x240 | per-CPU XNU saved-state array | VA of the per-CPU XNU saved-state array |
| 0xa8 | 0x248 | feature flags | hardcode `0x8` (the value that boots; individual bit meanings not reverse-engineered) |
| 0xb0..0xb8 | | allowed IO frame table + count | left zero |
| 0xc0 | 0x260 | pmap-io-ranges table pointer | identical to the bootstrap args field at 1.2 (0x318) same pointer; record format defined there |
| 0xc8 | 0x268 | pmap-io-ranges count | identical to the bootstrap args count at 1.2 (0x320) |
| 0xd0 | 0x270 | panicking-domain-id slot pointer | VA of the panicking-domain-id slot |
| 0xd8 | 0x278 | per-CPU event-counter array | VA of the per-CPU event-counter array |
| 0xe0..0x137 | | reserved padding | left zero |

The three pointers at rel 0x40, 0x48, and 0x50 point to tables populated as
follows. All tables are fully zero initialized except for the frame table
where one field is set by default.

**Frame types**: SPTM add the concept of a type for every managed frame. A
frame's type gates how it can be used, and XNU can ask SPTM to change a page's
type with the RETYPE endpoint (2.3.1), such as retyping a generic page to the
page table type.

| value | name | class | seeded at boot? |
|------:|------|-------|-----------------|
| 6 | kernel code | data (kind 5) | yes, executable kernelcache pages |
| 11 | default rw | data (kind 5) | yes, the default fill |
| 12 | read only | data (kind 5) | yes, RO kc pages |
| 8 | kernel root table | table/root (kind 2) | yes, TTBR1 |
| 18 | user root table | table/root (kind 2) | yes, TTBR0 |
| 9 | page table | table/root (kind 2) | yes, bootstrap PTs |
| 19 | shared root table | table/root (kind 2) | no |
| 20 | xnu page table | table/root (kind 2) | no |
| 21 | shared page table | table/root (kind 2) | no |
| 22 | rozone page table | table/root (kind 2) | no |
| 23 | commpage page table | table/root (kind 2) | no |
| 27 | io | io (kind 5) | no |
| 28 | protected io | io (kind 5) | no |
| 29 | coprocessor ro io | io (kind 5) | no |
| 33 | stage2 root table | table/root (kind 2) | no |
| 34 | stage2 page table | table/root (kind 2) | no |
| 40 | subpage user roots | table/root (kind 2) | no |
| 61 | sep secure channel | data (kind 5) | yes, the SEP secure-channel page (txm.md) |
| 0xff | invalid | -- | no |

`kind` is a libsptm classification byte (its use is in the frame-type params,
below). We only ever emit **2** (page-table/root) and **5** (data). Kinds 0, 1,
3, 4 also exist but we don't use them; libsptm treats 1 as another table class,
and 0/3/4 we haven't characterized.

**Frame table** (rel 0x40) The type of each frame is recorded in the frame
table, seeded at boot, and is used to communicate type information to XNU.

Every entry in the frame table is composed of a 4 byte header, which contains
the most important value, the frame type, and then a 12 byte body whose layout
changes depending on frame type. The body is overloaded: some parts are read
by XNU and thus form part of the SPTM ABI, others are used purely for SPTM
internal bookkeeping but are stored in the shared frame table struct. The
fields read by XNU are marked in the struct below; these must be maintained
in the emulator as XNU expects.

One entry per managed 16K frame, indexed by `(pa - SPTM-managed RAM start) >>
14`; 16-byte entry, field names ours, rsvd means the particular field meaning
has not yet been reverse engineered:

```c
struct frame_entry {
    u16 in_use;
    u8  type;                // read by XNU
    u8  flags;
    union {
        struct {             // cpu_page — data page (RAM, kernel data, DMA targets)
            u8  _rsvd0[4];
            u32 ro_refcount; // read by XNU: ro+wx summed is the total mapping
            u32 wx_refcount; // count, freed when count = 0
        } cpu_page;
        struct {             // page table (non-root)
            u8  level;
            u8  _rsvd5;
            u16 parent_links;
            u16 valid_ptes;  // read by XNU
            u8  _rsvd10[6];
        } page_table;
        struct {             // root table
            u8  _rsvd4[2];
            u16 valid_ptes;  // read by XNU
            u16 _rsvd8;
            u16 attrs;
            u8  geometry;
            u8  _rsvd13[3];
        } root;
        struct {             // XNU_IOMMU — coprocessor / IOMMU page table
            u16 iommu_refcount; // read by XNU (total for IOMMU frames)
            u8  iommu_id;
            u8  _rsvd7[9];
        } iommu;
        u8 raw[12];          // possibly other body formats
    } body;
};
```

To initialize this table, allocate one entry per managed frame (SPTM-managed
RAM end - SPTM-managed RAM start) / 16K total entries. Zero the entire list and
set the type to default for every entry. Populate only the entries that already
hold something, including kernelcache pages (RO, RW, and executable), bootstrap
page tables, and TTBR{0,1} roots. Set the mapping refcount to 1 for all pages
with an initial mapping — `ro_refcount` for a read-only mapping, `wx_refcount`
for a writable or executable one. Everything else (free RAM) keeps the default.
XNU will retype frames as it brings them into use. For each seeded page-table
page, set `valid_ptes` to the number of valid descriptors it holds (the
per-page valid-entry count from building the initial tables in 1.1).

Note that our emulator deivates from real SPTM behavior in a number of ways:

- we do not split roots from page tables; we use the `page_table` layout for
  every table frame, roots included.

- we do not maintain separate ro and wx refcounts, since XNU only cares about
  the sum of these values to decide when to free a table

These violations are non fatal because they do not violate the SPTM contract.
However, it is possible (and quite likely) that our understanding of this area
is subtly incorrect both in how XNU works and how real SPTM works.

**Frame-type params** (rel 0x48): 256 entries of length 0x90 bytes, indexed
by frame-type value, zeroed first. Set offset 0x01 to 2 for page-table and
root types (8, 9, 18-23 inclusive, 33, 34, 40) and 5 for all others. XNU's
libsptm reads this table (its pointer is handed over in `libsptm_state`) to
interpret a frame's refcount body when XNU queries a frame directly — e.g.
`sptm_frame_is_last_mapping`, which runs inside XNU — so it must be seeded,
not left zero.

**pt-attr table** (rel 0x50): 6 pointer slots (0..5), each the kernelcache VA
of a `_pmap_pt_attr_*` global: 0=`16k`, 1=`4k`, 2=`16k_kern`, 3=`16k_stage2`,
4=`16k_36b_stage2`, 5=`4k_stage2`. Resolve each by its symbol name from the
kernelcache symbol table.

**Physical-aperture range array** (rel 0x10; count at rel 0x08): the table
libsptm uses for PA->VA (`phystokv`). Each record is 24 bytes:

| off | size | field | meaning |
|---:|---:|------|--------|
| 0x00 | 8 | physical base | PA the window starts at |
| 0x08 | 8 | aperture VA base | the VA where this window starts |
| 0x10 | 4 | page count | length in 16K pages |
| 0x14 | 4 | flags | 0 in all our entries |

We emit three, in order: the ADT/devicetree window, the UAT-L2 window, then the
SPTM-managed-RAM physmap (ADT first so its alias wins where its pages overlap
the physmap).

**Note on Duplicate Entries**: some arguments are duplicated across the
structs: the physmap base/end (rel 0x28/0x30) and the pmap-io-ranges pointer
and count (rel 0xc0/0xc8)

The rationale is the bootstrap args are read once by early XNU boot, while this
nested state is copied into the persistent SPTM-client globals and used for the
kernel's lifetime. Both layers independently need the physmap window and the
IO-range policy, so each carries its own copy.

Full layout information in the APSL2 licensed `sptm_common.h` in the KDK

### 1.3 EL1 system registers

Before entry we set up the guest's EL1 translation regime via the EL12 alias:

| register | value |
|----------|-------|
| `TTBR0_EL1` | the TTBR0 root PA from 1.1 |
| `TTBR1_EL1` | the TTBR1 root PA from 1.1 |
| `TCR_EL1` | `0x10800336511A511` (16K granule, T1SZ=17) |
| `MAIR_EL1` | `0x0C0D00FF01A040FF` (AttrIndx 0 = normal cacheable) |
| `VBAR_EL1` | `0` |
| `SCTLR_EL1` | `0x100010103414593D` (XNU's boot SCTLR with EPAN (57) and the PAC-enable bits (31/30/27/13) cleared) |

These values are taken from what XNU writes during its boot process.

### 1.4 Jump to `_start`

`_start` is a small asm trampoline that calls `arm_init(boot_args*, sptm_bootstrap_args_xnu_t*)`.
Jump to it with the following registers:

| reg | value |
|-----|-------|
| `x0` | entry-routine enum (below) |
| `x1` | PA/pointer to `boot_args` (iBoot-style; m1n1 already builds one for HV mode) |
| `x2` | pointer to the `sptm_bootstrap_args_xnu_t` populated in 1.2 |
| `x3` | `0` |

Entry routine: `BOOT_COLD=0` (the normal cold-boot path), `BOOT_SECONDARY=1`,
`BOOT_WARM=2`, `BOOT_HIB=3`, `PANIC=4`.

`arm_init` immediately struct-copies `*x2` into a RO-late kernel global, so
neither the args struct nor `libsptm_state` need to persist past the call --
but the memory pointed at by `libsptm_state`'s PA-valued fields
(`papt_ranges`, `root_table_paddr`'s table, `xnu_triggered_panic`, etc.)
must remain live for the life of the boot.

---

## 2. XNU_BOOTSTRAP table emulation

### 2.1 sptm_return_t values

`sptm_return_t` is an uint32_t enum:

| val | name | meaning |
|----:|------|---------|
| 0 | SUCCESS | ok |
| 1 | MAP_VALID | map ok, replaced an existing valid mapping |
| 2 | MAP_FLUSH_PENDING | blocked: prior-mapping TLBI still in flight |
| 3 | MAP_CODESIGN_ERROR | TXM rejected mapping (signature) |
| 4 | UNMAP_FLUSH_PENDING | (unused) |
| 5 | UPDATE_DELAYED_TLBI | update ok, TLBI deferred as requested |
| 6 | MAP_PADDR_CONFLICT | mapping present with different PA |
| 7 | TABLE_NOT_PRESENT | no page table at some level for the VA |
| 8 | TABLE_ALREADY_PRESENT | table already present at the install level |

libsptm utility errors are a separate enum: `LIBSPTM_SUCCESS=0`,
`NOT_INITTED=1`, `INVALID_ARG=2`, `TYPE_MISMATCH=3`, `FAILURE=4`.

This emulator only ever returns 0, 1, 5, 6, 7, 8. Values 2/3/4 and the libsptm
utility-error enum are part of the real-SPTM definition but are never emitted
here.

### 2.2 Internal state

Beyond the frame table and page tables, SPTM keeps two small pieces of state
that matter for the endpoints below.

**Per-CPU active-root cache.** SPTM records, per CPU, the user root currently
installed in that core's TTBR0 (its ASID and identity, and the VA range it
covers) and refreshes it on every user-root switch. The point is to keep
SWITCH_ROOT's TLB work minimal: on the next switch it compares the incoming
root against this record and
- switching to the kernel root, or to the root already live, flushes nothing
  (it just reprograms the control registers);
- switching to a different root issues a range TLBI scoped to the *outgoing*
  root's VA span, recoverable only from this record, since overwriting TTBR0
  loses it, and falls back to a full local flush only when that span is too
  large to encode as one range op.
It is per-CPU because TTBR0 is per-CPU.

**CPU registry.** SPTM assigns each CPU a logical id at REGISTER_CPU (keyed off
the physical MPIDR) and keeps the physical->logical map; CPU_ID returns it, and
XNU uses the logical id to index the per-CPU scratch page (1.2, args 0x00).

### 2.3 XNU_BOOTSTRAP endpoints

The endpoints SPTM exposes for XNU under (domain=0, table=0). Most endpoints
are trivial. The nontrivial endpoints are written up in more depth in the 2.3.x
sections after the table.

| id | name | behavior | inputs | outputs |
|---:|------|----------|--------|---------|
| 0 | LOCKDOWN | no-op (real SPTM locks CTRR / retypes text) | -- | SUCCESS |
| 1 | RETYPE | set frame-type byte; zero page entering a PT type; record root geom/ASID for root types; TLBI on PT/IO/root | `x0`=pa, `x1`=cur type, `x2`=new type, `x3`=params (attr-idx/ASID/flags) | SUCCESS |
| 2 | MAP_PAGE | walk to L3, install leaf PTE, refcount | `x0`=root, `x1`=va, `x2`=PTE | SUCCESS / MAP_VALID / TABLE_NOT_PRESENT / MAP_PADDR_CONFLICT; scratch=one 16-byte pair `{displaced PTE, PTE-slot aperture VA}` |
| 3 | MAP_TABLE | install table descriptor(s) at the parent level | `x0`=root, `x1`=va, `x2`=target level, `x3`=TTE | SUCCESS / TABLE_NOT_PRESENT / TABLE_ALREADY_PRESENT |
| 4 | UNMAP_TABLE | clear parent TTE(s), TLBI | `x0`=root, `x1`=va, `x2`=target level | SUCCESS |
| 5 | UPDATE_REGION | masked-merge each leaf with template[i] over N VAs | `x0`=root, `x1`=start va, `x2`=count, `x3`=templates-array ptr, `x4`=flags (0x100=defer TLBI) | SUCCESS / UPDATE_DELAYED_TLBI; scratch=8-byte displaced PTE per page, VA order (<=2048) |
| 6 | UPDATE_DISJOINT | same merge, per op | `x1`=ops-array ptr, `x2`=count, `x3`=flags; op=`{root,va,template}` | SUCCESS / UPDATE_DELAYED_TLBI; scratch=8-byte displaced PTE per op, op order (<=2048) |
| 7 | UNMAP_REGION | clear each leaf over N VAs, refcount-- | `x0`=root, `x1`=start va, `x2`=count | SUCCESS; scratch=8-byte displaced PTE per page, VA order |
| 8 | UNMAP_DISJOINT | clear each op's leaf | `x1`=ops-array ptr, `x2`=count; op=`{root,va,_}` | SUCCESS; scratch=8-byte displaced PTE per op, op order |
| 9 | CONFIGURE_SHAREDREGION | set frame type --> shared-root-table | `x0`=pa | SUCCESS |
| 10 | NEST_REGION | copy shared root's L2 TTEs into the user root over the range | `x0`=user root, `x1`=shared root, `x2`=start va, `x3`=page count | SUCCESS |
| 11 | UNNEST_REGION | zero the user root's L2 TTEs over the range; TLBI | `x0`=user root, `x2`=start va, `x3`=page count | SUCCESS |
| 12 | CONFIGURE_ROOT | no-op | -- | SUCCESS |
| 13 | SWITCH_ROOT | kernel root --> set TTBR1; else TTBR0+ASID; TLBI | `x0`=root; `x1`/`x2`=flag set/clear masks (we ignore flags) | SUCCESS |
| 14 | REGISTER_CPU | add phys id to the CPU registry (fuzzy match) | `x0`=phys CPU id | SUCCESS |
| 15 | FIXUPS_COMPLETE | no-op | -- | SUCCESS |
| 16 | SIGN_USER_POINTER | PAC sign (key A); echo if guest key disabled | `x0`=pointer, `x1`=key, `x2`=discriminator | signed pointer (or unchanged) |
| 17 | AUTH_USER_POINTER | PAC auth (key A) | `x0`=pointer, `x1`=key, `x2`=discriminator | authed pointer / `UINT64_MAX` on failure / unchanged |
| 18 | REGISTER_EXC_RETURN | no-op | -- | SUCCESS |
| 19 | CPU_ID | look up the logical CPU id for a phys id | `x0`=phys CPU id | logical id |
| 20 | SLIDE_REGION | no-op | -- | SUCCESS |
| 21 | UPDATE_DISJOINT_MULTIPAGE | per-page header + inner ops, masked-merge | `x0`=ops-array ptr, `x1`=count; entry=`{paddr, papt template, inner N, opts}` | SUCCESS / UPDATE_DELAYED_TLBI; scratch=8-byte displaced PTE per inner op |
| 22 | REG_READ | no-op read | -- | 0 |
| 23 | REG_WRITE | no-op | -- | SUCCESS |
| 24 | GUEST_VA_TO_IPA | no guest stage-2 support | `x0`=va (ignored) | `UINT64_MAX` |
| 25 | GUEST_STAGE1_TLBOP | broadcast stage-1 TLBI (`tlbi vmalle1is`) | -- | SUCCESS |
| 26 | GUEST_STAGE2_TLBOP | broadcast stage-1+2 TLBI (`tlbi vmalls12e1is`) | -- | SUCCESS |
| 27 | GUEST_DISPATCH | unhandled -- should never fire on the boot path | -- | -- |
| 28 | GUEST_EXIT | no-op | -- | SUCCESS |
| 29 | MAP_SK_DOMAIN | no-op | -- | SUCCESS |
| 30 | HIB_BEGIN | no-op | -- | SUCCESS |
| 31 | HIB_VERIFY_HASH_NON_WIRED | no-op | -- | SUCCESS |
| 32 | HIB_FINALIZE_NON_WIRED | no-op | -- | SUCCESS |
| 33 | IOFILTER_PROTECTED_WRITE | unhandled -- should never fire on the boot path | -- | -- |
| 34-36 | *(gap)* | | | |
| 37 | SPTM_SYSCTL | no-op | -- | 0 |
| 38 | DISABLE_KERNEL_MODE_CPA2 | no-op | -- | SUCCESS |
| 39 | SET_SHARED_REGION | no-op | -- | SUCCESS |
| 40 | BATCH_SIGN_USER_POINTER | PAC sign each pointer in the ops array | `x0`=ops-array ptr, `x1`=count | SUCCESS; scratch=8-byte signed pointer per op, input order |
| 41 | SURT_ALLOC | zero a 128-byte sub-page user root slot, record attr-index/ASID | `x0`=frame, `x1`=slot index, `x2`=attr-index, `x4`=ASID | SUCCESS |
| 42 | SURT_FREE | zero a 128-byte sub-page user root slot | `x0`=frame, `x1`=slot index | SUCCESS |
| 43 | CONDEMN_LEAF_TABLE | walk to L2, set bit 55 of the (twig) descriptor | `x0`=root, `x1`=va | SUCCESS / TABLE_NOT_PRESENT |
| 44 | UNCONDEMN_LEAF_TABLE | walk to L2, clear bit 55 | `x0`=root, `x1`=va | SUCCESS |
| 45 | SPTM_SERIAL_PUTC | no-op (char dropped) | -- | SUCCESS |
| 46 | SPTM_SERIAL_DISABLE | no-op | -- | SUCCESS |
| 47 | *(gap)* | | | |
| 48 | PROGRAM_IRGKEY | no-op | -- | SUCCESS |
| 49 | REG_SNAPSHOT | no-op | -- | SUCCESS |

Unhandled / gap / `>=50` IDs should not occur in the boot path.

#### 2.3.1 RETYPE (endpoint 1)

RETYPE (1): Update the frame table's type for a given frame. PA is stored in
`x0`, current type in `x1`, and the requested new type is in `x2` with `x3`
contains various flags:

- bits [7:0]: entry in the pt-attr table
- bits [31:16]: flags, some are involved in PAC configuration but the exact meanings have not been reverse engineered
- bits [47:32]: ASID

Real SPTM validates type transitions; we unconditionally set update the type of
the page to the new type. Certain type transitions carry side effects:

- if the previous type was a page table type, set `valid_ptes` to 0
- if the requested type is a page table type, zero the frame and set `valid_ptes` to 0
- if the requested type is a root type, record 4k vs 16k and ASID

Note: when to flush TLB is a complicated operation that was optimized in real
SPTM.  Rather than understand the exact nature of this mechanism, we clear TLB
any time any type involved is a page-table/root type, or the new type is an IO
type.

##### 2.3.1.1 GXF Instructions

SPTM contains GXF gated cache flush instructions. They are called here in the
RETYPE endpoint and in UAT. SPTM keeps a dirty page list.  Whenever a page is
retyped in a way that affects coprocessor visibility (to/from type 24), its
page number is added to this list. If there are more than 63 dirty pages, then
instruction `00201401` is used to flush all pages, otherwise `00201328 x8` is
used to individually flush every page.

Exactly what these instructions do is unknown and cannot be easily reverse
engineered because we do not have the ability to actually run the instructions.
We know they involve coprocessor cache visibility, but the exact behavior is
not known.  Experiments have shown that a simple `tlbi` is insufficient to
replicate their behavior, even when combined with various `dsb` variants.
Instead, we implement a mitigation: special handling in DART (that likely needs
to be extended to UAT) that ensures that no cachable accesses to NC pages.

#### 2.3.2 MAP_PAGE (endpoint 2)

MAP_PAGE installs one leaf (L3) PTE. `x0`=root, `x1`=va, `x2`=PTE. It walks to
the L3 table for va and writes the leaf. It increments two refcounts (the
target frame's mapping refcount and the L3 table's `valid_ptes`), then writes a
16-byte {displaced PTE, PTE-slot aperture VA} pair to the per-CPU scratch (args
0x00) so XNU sees what it overwrote.

No L3 table for va --> TABLE_NOT_PRESENT. If a valid mapping is already there,
re-mapping the same PA upgrades it in place (MAP_VALID); a different PA is
refused with MAP_PADDR_CONFLICT, so XNU must unmap the old page and retry. A
fresh map --> SUCCESS. It flushes (TLBI) only when it replaced a valid mapping.

#### 2.3.3 Leaf attribute updates (endpoints 5, 6, 21)

UPDATE_REGION / DISJOINT / MULTIPAGE (5, 6, 21) change only the attributes of
existing leaves, never the mapping. Each op carries a template (an ARM-layout
leaf PTE) and a flags word. The flags word is overloaded: its low 6 bits are a
field mask where each bit corresponds to a different field on an ARM PTE, and
bits 8-9 are separate control flags.

| bit | role | meaning |
|----:|------|---------|
| 0 | mask | wired (software bit) |
| 1 | mask | permissions (PXN, UXN, AP, writeable) |
| 2 | mask | nG |
| 3 | mask | AF |
| 4 | mask | SH |
| 5 | mask | AttrIndx |
| 8 | ctrl | defer TLBI: return UPDATE_DELAYED_TLBI; XNU flushes later via GUEST_STAGE1_TLBOP (25) |
| 9 | ctrl | skip the physmap update (MULTIPAGE only) |

For every call, the process is: walk to the correct leaf table, stash the old
PTE in the scratch page (8 bytes per op, in the order given), then use the mask
to select what bits to copy from the template to the live PTE, keeping the rest
of the PTE unmodified. A bit should be copied from the PTE if the corresponding
mask bit is set. Because only attributes change, an UPDATE never moves a
refcount. An empty mask is illegal and SPTM panics if it is passed.

REGION (5): one root, a contiguous VA run. `x0`=root, `x1`=start va,
`x2`=count, `x3`=template array, `x4`=flags. The i-th mapping is va = start_va
+ i*pagesize with templates[i]; nothing is read per-op.

DISJOINT (6): instead of contiguous VAs, they are scattered, with a pointer to
structs containing the scattered VAs and the corresponding roots and templates
in `x1`. `x2` and `x3` still store count and flags, respectively. Each op is
one 24-byte struct (also reused as the MULTIPAGE inner op):

```c
struct update_op {
    u64 root;
    u64 va;
    u64 template;
};
```

so the pages need not be adjacent and may sit in different roots in one call.

MULTIPAGE (21): essentially a more complicated version of DISJOINT that
includes a different type of control struct and can update the AttrIndx and SH
bits in a per-page physmap update. `x0` still points to an ops array, but it
interleaves header structs that contain the flag information for that group
with the actual op structs. The header struct is defined as:

```c
struct update_mp_header {
    u64 paddr;          // physical page whose physmap alias to fix
    u64 papt_template;  // its desired memory-type template
    u32 inner_n;        // number of update_op slots that follow this header
    u32 opts;           // mask + control flags for this group
};
```

The ops array can have multiple groups of header structs and ops structs all
interleaved in the same call (count passed in `x1`). The ops structs carry no
type tag, so the array is parsed statefully: read a header, fix the physmap
alias for its paddr by setting its AttrIndx and SH bits from papt_template,
then treat the next inner_n slots as that header's inner ops and run each like
DISJOINT; the slot after them is the next header. Repeat until `x1` slots are
consumed. inner_n may be 0 (a header that only fixes the physmap). The physmap
fix keeps a page's alias consistent when its memory type changes.

#### 2.3.4 UNMAP_{REGION, DISJOINT} (endpoints 7, 8)

These endpoints clear each leaf PTE to 0, decrementing both refcounts (1.2),
and write each old PTE to the scratch (8 bytes per leaf, in order). REGION = a
consecutive run (`x0`=root, `x1`=start va, `x2`=count); DISJOINT = a {root, va, _} op
array (`x1`=ptr, `x2`=count, template slot unused). Both endpoints flush TLB.

#### 2.3.5 {MAP, UNMAP}_TABLE (endpoints 3, 4)

These endpoints link and unlink page-table pages in the tree, called after
RETYPE to add or remove page tables from the tree. Neither modifies a refcount
or `valid_ptes`.

MAP_TABLE (3): install a table descriptor at the requested level. `x0`=root,
`x1`=va, `x2`=target level, `x3`=TTE (carries the child PA). It walks to that level:
no path to it --> TABLE_NOT_PRESENT; a table already linked at
that slot --> TABLE_ALREADY_PRESENT (it won't overwrite a live link); otherwise
it writes the descriptor and returns SUCCESS. For a 4K root, four 4K page
tables share one 16K frame, so MAP_TABLE links the whole frame at once: it
rounds the parent index down to a multiple of 4 and writes four consecutive
descriptors, the n-th pointing to the n-th 4K sub-table (child + n*4K).

UNMAP_TABLE (4): clear the table descriptor at the target level, TLBI, return
SUCCESS.  `x0`=root, `x1`=va, `x2`=target level. A missing table is an immediate
panic() because it indicates SPTM and XNU's views have diverged.

#### 2.3.6 {NEST, UNNEST}_REGION, SWITCH_ROOT (endpoints 10, 11, 13)

NEST_REGION (10): map a shared region into a user address space cheaply.
`x0`=user root, `x1`=shared root, `x2`=start va, `x3`=page count. For each L2 block the
range covers it copies the shared root's L2 descriptor into the user root's
matching L2 slot. The user root then reaches the shared region's L3 tables at
the same VAs without duplicating them; it just points at the shared root's L2
tables.

UNNEST_REGION (11): undo a nest. `x0`=user root, `x2`=start va, `x3`=page count
(`x1` unused). For each L2 block it zeroes the user root's L2 slot, dropping the
link to the shared L2 tables (which are left intact). TLBI, SUCCESS.

SWITCH_ROOT (13): install a page-table root. `x0`=root. If it is the kernel root
(kernel_root_table type) set the guest's TTBR1 (via its EL12 alias) to the root
PA, with no ASID. Otherwise it is a user root: set TTBR0 to the root PA tagged
with the root's ASID, looked up from the size/ASID recorded for it at
RETYPE/SURT time (2.3.1). SPTM normally reads `x1`/`x2` and determines if it can
skip the TLB flush, or do a narrowly scoped flush, but our emulator does a
full flush every call.

#### 2.3.7 SURT_{ALLOC, FREE} (endpoints 41, 42)

A sub-page user root packs several roots into one frame as 128-byte slots.
`x0`=frame, `x1`=slot index, `x2`=attr-index (geometry) from pt-attr table,
`x4`=ASID (`x3` unused by the emulator, meaning not yet reverse engineered). The
slot is at frame + index*128. Both endpoints zero the 128-byte slot. SURT_ALLOC
(41) additionally writes the slot's geometry + ASID into the same per-root
record RETYPE keeps for roots (2.3.1), so SWITCH_ROOT can later find this root's
ASID. SURT_FREE (42) just zeroes. Both return SUCCESS.

#### 2.3.8 REGISTER_CPU and CPU_ID (endpoints 14, 19)

REGISTER_CPU (14):  Every CPU, when it spins up, calls REGISTER_CPU to be
assigned an arbitrary logical ID used to identify which per cpu scratch page it
will use (slot = base + 16K * id; args 0x00 in 1.2). The boot CPU always gets
zero; the secondaries 1...5 are assigned arbitrarily in order of spinup.
`x0`=lower 32 bits of MPIDR.  We check to see if we've seen a CPU before using a
fuzzy MPIDR compare (exact, then low 32 / 24 / 16 bits, to tolerate
affinity-field differences). Update SPTM internal state with the new
logical ID.

CPU_ID (19): `x0`=physical CPU id. Returns the logical id for that phys id, via
the same fuzzy match against internal state, or 0 if it is not registered.
