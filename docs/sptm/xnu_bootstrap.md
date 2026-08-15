# XNU Bootstrap -- Clean-Room Functional Specification (DRAFT)

This document is written against m1n1 commit
`844ffc7232ec3ee21ece59eab2d70e3cc443fe8e` on 2026-07-31.

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
| code | `0x603` | AttrIndx 0, kernel-RW, executable, Outer Shareable |
| data | `0x60000000000703` | AttrIndx 0, kernel-RW, execute-never (PXN+UXN), Inner Shareable |

Only the kernelcache image is mapped as code, everything else as data. We map
the kernel RWX out of convenience. Real SPTM would never permit this mapping.

**TTBR1 windows** (placement VAs are our choice except where noted):

| window | maps | preset |
|--------|------|--------|
| kernelcache image | kc VA range (real vmin..vmax) --> kc PA | code |
| aux carveout | handoff scratch (structs, page-table pool, CPU/TXM stacks) | data |
| iBoot boot args | the `boot_args` page (1.4 `x1`) | data |
| device tree (ADT) | the ADT blob and TrustCache, just below the kernelcache VA | data |
| physmap | the kernel's linear VA window over SPTM-managed RAM; bounds must match physmap base/end in 1.2 (rel 0x28/0x30) | data |
| UAT L2 | GPU shared-region L2 page table; PA from the ADT `/arm-io/sgx` `gfx-shared-l2-region`; see uat.md | data |

XNU expects empty L3 tables to be present for two additional TTBR1 ranges.  For
every 32 MiB region intersecting either range, allocate a zero-filled L3 table
and install it in the corresponding L2 entry. Create the required L2 table if
it does not already exist.

The first range begins at the first 32 MiB boundary at or above the end of the
physmap. Its requested size is:

```text
memory_segments = ceil(mem_size / 256 MiB)

dynamic_size = round_up(
    2 MiB
    + round_up(Video.height * Video.rowBytes, 16 KiB)
    + 10 MiB * memory_segments,
    8 MiB)
```

XNU uses these tables for allocations made while bootstrapping the VM system,
before normal page-table allocation is operational.

Our emulator uses a fixed 64 MiB framebuffer allowance instead of calculating
`Video.height * Video.rowBytes`. Otherwise, it constructs the same empty table
coverage.

The second range is a single L3 table for the 32 MiB region immediately below
the exclusive upper bound of XNU's kernel VA range. On the current target this
covers `[0xfffffecffe000000, 0xfffffed000000000)`. XNU uses part of this VA
range for its per-CPU copy windows and early-debug data.

The TTBR1 root PA goes in the nested `libsptm` struct at (rel 0x38); we also
program TTBR0/TTBR1/TCR/MAIR/SCTLR into the guest EL1 registers.

### 1.2 Handoff structs

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
| 0x198 | 8 | XNU panic flag | VA of a zero-initialized 8-byte buffer; XNU writes `1` to its first byte on panic |
| 0x1a0 | 312 | nested SPTM client state | see next section |
| 0x2e8 | 8 | AuxKC end | top VA of the Auxiliary Kernel Collection. We ship no AuxKC, but XNU derives the overall top-of-kernelcache (its `end_kern` / `vm_kernelcache_top`) from this field, so set it to the kernelcache image end VA, 0 would break XNU's kernelcache bounds |
| 0x318 | 8 | pmap-io-ranges table pointer | guest VA of the pmap I/O range table described below |
| 0x320 | 4 | pmap-io-ranges count | number of records in the pmap I/O range table |
| 0x328 | 8 | pmap-io-filters table pointer | guest VA of the pmap I/O filter table described below |
| 0x330 | 4 | pmap-io-filters count | number of records in the pmap I/O filter table |
| 0x338 | 8 | feature flags | hardcode `0x10` (the value that boots; bit meanings not reverse-engineered) |

#### 1.2.1 Debug header

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

#### 1.2.2 `libsptm_state` (nested at +0x1a0, 312 B)

| rel | abs | short description | what we put here |
|----:|----:|-------|----------------------------|
| 0x00 | 0x1a0 | version | `10` (the layout version this build implements; XNU selects field offsets by it) |
| 0x08 | 0x1a8 | physical-aperture range count pointer | VA of a `u32` containing the number of records in the physical-aperture range array; `3` for our configuration |
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
| 0x80 | 0x220 | panicking-CPU-id slot pointer | VA of a writable `u16` initialized to `0xffff` |
| 0x88 | 0x228 | trace buffer pointer | VA of a zero-initialized 16 KiB buffer |
| 0x90 | 0x230 | per-CPU dispatch-state array | an array of unique pointers, all pointing to the number 5 |
| 0x98 | 0x238 | max CPU count | boot CPU count |
| 0xa0 | 0x240 | per-CPU saved-state array | an array of pointers pointing to zero-initialized 16 KiB buffers |
| 0xa8 | 0x248 | feature flags | hardcode `0x8` (the value that boots; individual bit meanings not reverse-engineered) |
| 0xb0..0xb8 | | allowed IO frame table + count | left zero |
| 0xc0 | 0x260 | pmap-io-ranges table pointer | identical to the bootstrap args field at 1.2 (0x318) same pointer; record format defined there |
| 0xc8 | 0x268 | pmap-io-ranges count | identical to the bootstrap args count at 1.2 (0x320) |
| 0xd0 | 0x270 | panicking-domain-id slot pointer | VA of a writable `u32` initialized to `0xff` |
| 0xd8 | 0x278 | per-CPU event-counter array | same as per-CPU saved-state array |
| 0xe0..0x137 | | reserved padding | left zero |

Internally, SPTM allocates one `0x1800` byte object per CPU and then the
pointers in the dispatch-state array, saved-state array, and event-counter
array all point to different offsets into these objects, `0x000`, `0x050`, and
`0xc90` respectively. Most of these fields are never read by XNU.

**Note on Duplicate Entries**: some arguments are duplicated across the
structs: the physmap base/end (rel 0x28/0x30) and the pmap-io-ranges pointer
and count (rel 0xc0/0xc8)

The rationale is the bootstrap args are read once by early XNU boot, while this
nested state is copied into the persistent SPTM-client globals and used for the
kernel's lifetime. Both layers independently need the physmap window and the
IO-range policy, so each carries its own copy.

##### 1.2.2.1 Physical-aperture range array (rel 0x10; count at rel 0x08)

The table libsptm uses for PA->VA (`phystokv`). Each record is 24 bytes:

| off | size | field | meaning |
|---:|---:|------|--------|
| 0x00 | 8 | physical base | PA the window starts at |
| 0x08 | 8 | aperture VA base | the VA where this window starts |
| 0x10 | 4 | page count | length in 16K pages |
| 0x14 | 4 | flags | 0 in all our entries |

We emit three, in order: the ADT/devicetree window, the UAT-L2 window, then the
SPTM-managed-RAM physmap (ADT first so its alias wins where its pages overlap
the physmap).

The three pointers at rel 0x40, 0x48, and 0x50 point to tables populated as
follows. All tables are fully zero initialized except for the frame table
where one field is set by default.

##### 1.2.2.2 Frame types

SPTM add the concept of a type for every managed frame. A
frame's type gates how it can be used, and XNU can ask SPTM to change a page's
type with the RETYPE endpoint (2.3.1), such as retyping a generic page to the
page table type.

The table below reproduces the complete current frame-type enum. Its order and
values come from `platform/sptm/sptm_common.h` in the KDK; the type-string table
in the 26.6 beta 4 SPTM binary has the same order. The KDK calls values 36 and
37 `XNU_RESERVED_1` and `XNU_RESERVED_2`; the SPTM binary names them
`XNU_CPUTRACE_PA_BUFFER` and `XNU_CPUTRACE_VA_BUFFER`.

| value | name |
|------:|------|
| 0 | `SPTM_UNTYPED` |
| 1 | `SPTM_UNUSED` |
| 2 | `SPTM_DEFAULT` |
| 3 | `SPTM_RO` |
| 4 | `SPTM_CODE` |
| 5 | `SPTM_TXM_CODE` |
| 6 | `SPTM_XNU_CODE` |
| 7 | `SPTM_XNU_CODE_DBG_RW` |
| 8 | `SPTM_KERNEL_ROOT_TABLE` |
| 9 | `SPTM_PAGE_TABLE` |
| 10 | `SPTM_IOMMU_BOOTSTRAP` |
| 11 | `XNU_DEFAULT` |
| 12 | `XNU_RO` |
| 13 | `XNU_RO_DBG_RW` |
| 14 | `XNU_USER_EXEC` |
| 15 | `XNU_USER_DEBUG` |
| 16 | `XNU_USER_JIT` |
| 17 | `XNU_USER_TPRO` |
| 18 | `XNU_USER_ROOT_TABLE` |
| 19 | `XNU_SHARED_ROOT_TABLE` |
| 20 | `XNU_PAGE_TABLE` |
| 21 | `XNU_PAGE_TABLE_SHARED` |
| 22 | `XNU_PAGE_TABLE_ROZONE` |
| 23 | `XNU_PAGE_TABLE_COMMPAGE` |
| 24 | `XNU_IOMMU` |
| 25 | `XNU_ROZONE` |
| 26 | `XNU_IO` |
| 27 | `XNU_PROTECTED_IO` |
| 28 | `XNU_COPROCESSOR_RO_IO` |
| 29 | `XNU_COMMPAGE_RW` |
| 30 | `XNU_COMMPAGE_RO` |
| 31 | `XNU_COMMPAGE_RX` |
| 32 | `XNU_TAG_STORAGE` |
| 33 | `XNU_STAGE2_ROOT_TABLE` |
| 34 | `XNU_STAGE2_PAGE_TABLE` |
| 35 | `XNU_KERNEL_RESTRICTED` |
| 36 | `XNU_CPUTRACE_PA_BUFFER` |
| 37 | `XNU_CPUTRACE_VA_BUFFER` |
| 38 | `XNU_RESTRICTED_IO` |
| 39 | `XNU_RESTRICTED_IO_TELEMETRY` |
| 40 | `XNU_SUBPAGE_USER_ROOT_TABLES` |
| 41 | `TXM_DEFAULT` |
| 42 | `TXM_RO` |
| 43 | `TXM_RW` |
| 44 | `TXM_CPU_STACK` |
| 45 | `TXM_THREAD_STACK` |
| 46 | `TXM_ADDRESS_SPACE_TABLE` |
| 47 | `TXM_MALLOC_PAGE` |
| 48 | `TXM_FREE_LIST` |
| 49 | `TXM_SLAB_TRUST_CACHE` |
| 50 | `TXM_SLAB_PROFILE` |
| 51 | `TXM_SLAB_CODE_SIGNATURE` |
| 52 | `TXM_SLAB_CODE_REGION` |
| 53 | `TXM_SLAB_ADDRESS_SPACE` |
| 54 | `TXM_BUCKET_1024` |
| 55 | `TXM_BUCKET_2048` |
| 56 | `TXM_BUCKET_4096` |
| 57 | `TXM_BUCKET_8192` |
| 58 | `TXM_BULK_DATA` |
| 59 | `TXM_BULK_DATA_READ_ONLY` |
| 60 | `TXM_LOG` |
| 61 | `TXM_SEP_SECURE_CHANNEL` |
| 62 | `SK_DEFAULT` |
| 63 | `SK_SHARED_RO` |
| 64 | `SK_SHARED_RW` |
| 65 | `SK_IO` |

The enum then defines `N_FRAME_TYPES=66`, `FRAME_TYPE_INVALID=67`, and
`FRAME_TYPE_ANY=68`; these are metadata or API sentinels, not frame types.

Our emulator initially fills every frame-table entry with `XNU_DEFAULT`, then
overwrites the TTBR1 root with `SPTM_KERNEL_ROOT_TABLE`, the TTBR0 root with
`XNU_USER_ROOT_TABLE`, bootstrap page tables with `SPTM_PAGE_TABLE`,
executable kernelcache pages with `SPTM_XNU_CODE`, read-only kernelcache pages
with `XNU_RO`, and the SEP secure-channel page with
`TXM_SEP_SECURE_CHANNEL`.

`kind` is a libsptm classification byte (its use is in the frame-type params,
below). We only ever emit **2** (page-table/root) and **5** (data). Kinds 0, 1,
3, 4 also exist but we don't use them; libsptm treats 1 as another table class,
and 0/3/4 we haven't characterized.

##### 1.2.2.3 Frame table (rel 0x40)

The type of each frame is recorded in the frame
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
        struct {             // XNU_TAG_STORAGE
            u32 tag_storage_count;
            u8  _rsvd8[8];
        } tag_storage;
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

Our emulator does not split roots from page tables; we use the `page_table`
layout for every table frame, including roots. This violation is non fatal
because it does not violate the SPTM contract. However, it is possible (and
quite likely) that our understanding of this area is subtly incorrect both in
how XNU works and how real SPTM works.

##### 1.2.2.4 Frame-type params (rel 0x48)

256 entries of length 0x90 bytes, indexed
by frame-type value, zeroed first. Set offset 0x01 to 2 for page-table and
root types (8, 9, 18-23 inclusive, 33, 34, 40) and 5 for all others. XNU's
libsptm reads this table (its pointer is handed over in `libsptm_state`) to
interpret a frame's refcount body when XNU queries a frame directly — e.g.
`sptm_frame_is_last_mapping`, which runs inside XNU — so it must be seeded,
not left zero.

##### 1.2.2.5 pt-attr table (rel 0x50)

6 pointer slots (0..5), each the kernelcache VA
of a `_pmap_pt_attr_*` global: 0=`16k`, 1=`4k`, 2=`16k_kern`, 3=`16k_stage2`,
4=`16k_36b_stage2`, 5=`4k_stage2`. Resolve each by its symbol name from the
kernelcache symbol table.

Full layout information in the APSL2 licensed `sptm_common.h` in the KDK

#### 1.2.3 pmap I/O policy tables

These tables are constructed from `/defaults/pmap-io-ranges` and
`/defaults/pmap-io-filters` in the ADT. They describe XNU's protected-MMIO
policy and are separate from the physical-aperture ranges in `libsptm_state`,
which describe PA-to-VA aliases.

Each pmap I/O range is a 24-byte record:

| off | size | meaning |
|----:|-----:|---------|
| 0x00 | 8 | physical address |
| 0x08 | 8 | size |
| 0x10 | 4 | flags |
| 0x14 | 4 | signature |

Require the physical address and size to be multiples of 16 KiB. The signature
comes from the ADT range entry's four-character `name` field; interpret its
four bytes as a big-endian `u32`. Sort the records by
`(physical address, size, signature, flags)`, then serialize every field
little-endian.

Each pmap I/O filter is an 8-byte record:

| off | size | meaning |
|----:|-----:|---------|
| 0x00 | 4 | signature |
| 0x04 | 2 | offset within a 16 KiB page |
| 0x06 | 2 | length |

The signature comes from the ADT filter entry's four-character `signature`
field and is converted in the same way as the range signature. Require
`offset + length` to be at most 16 KiB, so a filter cannot cross a page
boundary. Sort the records by `(signature, offset, length)`, then serialize
every field little-endian.

Place each resulting array in a separate zero-initialized allocation aligned
to 16 KiB, with its allocation size rounded up to 16 KiB. Store the arrays'
guest VAs and record counts at bootstrap-argument offsets 0x318 through 0x330.
The pmap I/O range pointer and count are also copied into `libsptm_state` at
relative offsets 0xc0 and 0xc8. The filter table is referenced only by the
outer bootstrap arguments.

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
| `x1` | guest VA of `boot_args` (iBoot-style; m1n1 already builds one for HV mode) |
| `x2` | guest VA of the `sptm_bootstrap_args_xnu_t` populated in 1.2 |
| `x3` | `0` |

Entry routine: `BOOT_COLD=0` (the normal cold-boot path), `BOOT_SECONDARY=1`,
`BOOT_WARM=2`, `BOOT_HIB=3`, `PANIC=4`.

`arm_init` immediately struct-copies `*x2` into a RO-late kernel global, so
neither the args struct nor `libsptm_state` need to persist past the call --
but the memory referenced by its persistent pointer fields must remain live for
the life of the boot. These pointers, including `papt_ranges` and
`xnu_triggered_panic`, are guest VAs. `root_table_paddr` and the managed-RAM
bounds are physical addresses.

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
| 2 | MAP_PAGE | walk to L3, install leaf PTE, refcount | `x0`=root, `x1`=va, `x2`=PTE, `x3`=previous-PTE output mode | SUCCESS / MAP_VALID / TABLE_NOT_PRESENT / MAP_PADDR_CONFLICT; scratch=one 16-byte pair `{displaced PTE, PTE-slot aperture VA}` |
| 3 | MAP_TABLE | install table descriptor(s) at the parent level | `x0`=root, `x1`=va, `x2`=target level, `x3`=TTE | SUCCESS / TABLE_NOT_PRESENT / TABLE_ALREADY_PRESENT |
| 4 | UNMAP_TABLE | clear parent TTE(s), TLBI | `x0`=root, `x1`=va, `x2`=target level | SUCCESS |
| 5 | UPDATE_REGION | masked-merge each leaf with template[i] over N VAs | `x0`=root, `x1`=start va, `x2`=count, `x3`=templates-array ptr, `x4`=flags (0x100=defer TLBI) | SUCCESS / UPDATE_DELAYED_TLBI; scratch=8-byte displaced PTE per page, VA order (<=2048) |
| 6 | UPDATE_DISJOINT | same merge, per op | `x1`=ops-array ptr, `x2`=count, `x3`=flags; op=`{root,va,template}` | SUCCESS / UPDATE_DELAYED_TLBI; scratch=8-byte displaced PTE per op, op order (<=2048) |
| 7 | UNMAP_REGION | clear each leaf over N VAs, refcount-- | `x0`=root, `x1`=start va, `x2`=count, `x3`=options | SUCCESS / UPDATE_DELAYED_TLBI; scratch=8-byte displaced PTE per page, VA order |
| 8 | UNMAP_DISJOINT | clear each op's leaf | `x1`=ops-array ptr, `x2`=count; op=`{root,va,_}` | SUCCESS; scratch=8-byte displaced PTE per op, op order |
| 9 | CONFIGURE_SHAREDREGION | set frame type --> shared-root-table | `x0`=pa | SUCCESS |
| 10 | NEST_REGION | copy shared root's L2 TTEs into the user root over the range | `x0`=user root, `x1`=shared root, `x2`=start va, `x3`=page count | SUCCESS |
| 11 | UNNEST_REGION | zero the user root's L2 TTEs over the range; TLBI | `x0`=user root, `x2`=start va, `x3`=page count | SUCCESS |
| 12 | CONFIGURE_ROOT | no-op | -- | SUCCESS |
| 13 | SWITCH_ROOT | select kernel-only or user address space | `x0`=root, `x1`=flags, `x2`=mask | SUCCESS |
| 14 | REGISTER_CPU | assign a logical CPU id to a physical CPU id| `x0`=phys CPU id | SUCCESS |
| 15 | FIXUPS_COMPLETE | no-op | -- | SUCCESS |
| 16 | SIGN_USER_POINTER | PAC sign (key A); echo if guest key disabled | `x0`=pointer, `x1`=key, `x2`=discriminator | signed pointer (or unchanged) |
| 17 | AUTH_USER_POINTER | PAC auth (key A) | `x0`=pointer, `x1`=key, `x2`=discriminator | authed pointer / `UINT64_MAX` on failure / unchanged |
| 18 | REGISTER_EXC_RETURN | no-op without Exclaves; with Exclaves, saves the return trampoline | `x0`=va of XNU's exception return trampoline | SUCCESS |
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
| 41 | SURT_ALLOC | zero a 128-byte sub-page user root slot, record attr-index/ASID | `x0`=frame, `x1`=slot index, `x2`=attr-index, `x3`=flags, `x4`=ASID | SUCCESS |
| 42 | SURT_FREE | zero a 128-byte sub-page user root slot | `x0`=frame, `x1`=slot index | SUCCESS |
| 43 | CONDEMN_LEAF_TABLE | walk to L2, set bit 2 of the (twig) descriptor | `x0`=root, `x1`=va | SUCCESS / TABLE_NOT_PRESENT |
| 44 | UNCONDEMN_LEAF_TABLE | walk to L2, clear bit 2 | `x0`=root, `x1`=va | SUCCESS |
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
not known. Experiments have shown that a simple `tlbi` is insufficient to
replicate their behavior, even when combined with various `dsb` variants.

#### 2.3.2 MAP_PAGE (endpoint 2)

MAP_PAGE installs one leaf PTE. `x0`=root, `x1`=va, `x2`=PTE, and `x3`
selects whether the previous mapping is returned. It walks to the L3 table for
the VA and writes the leaf.

For a fresh mapping, increment the target frame's mapping refcount and the L3
table's `valid_ptes`. Updating an existing mapping of the same physical page
does not change either refcount.

If `x3` is zero, write a 16-byte `{previous PTE, PTE-slot aperture VA}` pair to
the invoking CPU's scratch page. If `x3` is one, do not write this result. Our
emulator ignores `x3` and always writes the pair.

No L3 table for the VA returns TABLE_NOT_PRESENT. Re-mapping the same physical
page updates the PTE in place and returns MAP_VALID. Attempting to replace it
with a different physical page returns MAP_PADDR_CONFLICT without changing the
PTE, so XNU must unmap the old page and retry. A fresh mapping returns SUCCESS.
Invalidate the old translation after updating an existing valid mapping.

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

These endpoints set each selected leaf PTE to zero. For every valid mapping
removed, decrement the mapped frame's mapping refcount and the leaf table's
`valid_ptes`. Write each previous PTE to the invoking CPU's scratch page, eight
bytes per requested leaf and in request order.

UNMAP_REGION (7) operates on a consecutive run. `x0`=root, `x1`=start VA,
`x2`=count, and `x3`=options. If option bit 8 (`0x100`) is zero, invalidate the
affected translations and return SUCCESS. If it is set, leave the invalidation
to XNU and return UPDATE_DELAYED_TLBI when at least one valid mapping was
removed.

UNMAP_DISJOINT (8) takes an array of `{root, VA, unused}` operations in `x1`
and the operation count in `x2`. It invalidates the affected translations and
returns SUCCESS.

#### 2.3.5 {MAP, UNMAP}_TABLE (endpoints 3, 4)

These endpoints link and unlink page-table pages in the tree, called after
RETYPE to add or remove page tables from the tree. Linking increments the child
frame's `parent_links`; unlinking decrements it. Neither operation changes the
child table's `valid_ptes`.

MAP_TABLE (3): install a table descriptor at the requested level. `x0`=root,
`x1`=va, `x2`=target level, `x3`=TTE (carries the child PA). It walks to that level:
no path to it --> TABLE_NOT_PRESENT; a table already linked at
that slot --> TABLE_ALREADY_PRESENT (it won't overwrite a live link); otherwise
it writes the descriptor and returns SUCCESS.

For a 4K root, four 4K page tables share one 16K frame, so MAP_TABLE links the
whole frame at once: it rounds the parent index down to a multiple of 4 and
writes four consecutive descriptors, the n-th pointing to the n-th 4K sub-table
(child + n*4K). `parent_links` is only updated once.

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

SWITCH_ROOT (13): select the page-table root for the invoking CPU. `x0`=root.
There are two cases depending on what's inside `x0`:

If it is a kernel root (kernel_root_table type) set an empty (valid root that
contains no mappings) TTBR0 (via its EL12 alias), with no ASID.  The kernel
root would have been installed earlier in bootstrap.  

Otherwise, it is a user root. set TTBR0 to the root PA tagged with the root's
ASID, looked up from the size/ASID recorded for it at RETYPE/SURT time (2.3.1).
SPTM normally reads `x1=flags`/`x2=mask` to configure other things: JIT, JOP,
x86_64 compatability, TPRO, and how to flush.

The bring-up emulator installs a dedicated pinned empty root in TTBR0 for the
kernel-root case and leaves the already-live TTBR1 alone. It still
unconditionally emits a `tlbi`. An earlier implementation incorrectly rewrote
TTBR1 and left the prior user TTBR0 live; after XNU freed and reused that root,
a CPU could walk ordinary data as a page table and generate a late LLC/AMCC
fabric error.

#### 2.3.7 SURT_{ALLOC, FREE} (endpoints 41, 42)

These endpoint packs several user roots into one frame as 128-byte slots. The
first 64 bytes of this slot are an ordinary ARM root page table containing eight
descriptors.  The second 64 bytes of this slot are internal SPTM bookkeeping
that is not reverse engineered, but it contains the ASID and geometry.

`x0`=frame, `x1`=slot index, for both. SURT_ALLOC also has `x2`=attr-index
(geometry) from pt-attr table, `x3`=flags currently unused in emulator,
`x4`=ASID. The slot is at frame + index * 128. In our emulator (real SPTM
diverges) we just zero the full 128 byte slot first.

SURT_ALLOC (41): allocate a sub page user root. Other endpoints actually
populate it. Record the geometry and the ASID (later used by SWITCH_ROOT).

SURT_FREE (42) just frees internal resources and zeros the slot.

#### 2.3.8 REGISTER_CPU and CPU_ID (endpoints 14, 19)

REGISTER_CPU (14): Every CPU, when it spins up, calls REGISTER_CPU to be
assigned an arbitrary logical ID used to identify which per cpu scratch page it
will use (slot = base + 16K * id; args 0x00 in 1.2). The boot CPU always gets
zero; the secondaries are assigned sequentially in order of spinup. `x0`=lower
32 bits of MPIDR. Update SPTM internal state with the new logical ID. Real
SPTM additionally checks that `x0` is actually one of the CPUs present in the
ADT.

CPU_ID (19): `x0`=physical CPU id. Returns the logical id for that phys id,
panic if unregistered.

---

## 3. Updates in macOS 27

For macOS 27, use the handoff structure below and add the endpoint behavior in
3.3. The page-table operations described above are otherwise unchanged.

### 3.1 XNU handoff structure

The T8142 structure is 0x368 bytes. Zero it before filling the fields below.
All pointers are XNU kernel virtual addresses unless the field explicitly says
PA or PAPT.

| off | size | field | required value |
|----:|-----:|-------|----------------|
| 0x000 | 8 | version | `0xd00f000000000000` |
| 0x008 | 8 | physmap base | first VA in XNU's physical aperture |
| 0x010 | 8 | physmap end | exclusive end VA of the physical aperture |
| 0x018 | 8 | first available PA | first page available to XNU; after SK bootstrap this is the final cursor returned by cL4 |
| 0x020 | 8 | physical-slide PAPT | start of the range XNU may release, or zero when unused |
| 0x028 | 8 | physical-slide size | size of that range, or zero |
| 0x030 | 8 | TXM thread-stack array | VA of the array of TXM thread-stack PAPT addresses |
| 0x038 | 4 | TXM thread-stack count | number of entries in the array |
| 0x040 | 8 | per-CPU stack window start | inclusive PAPT bound |
| 0x048 | 8 | per-CPU stack window end | exclusive PAPT bound |
| 0x050 | 8 | executables window start | inclusive kernel executable VA bound |
| 0x058 | 8 | executables window end | exclusive kernel executable VA bound |
| 0x060 | 8 | debug header | VA of the debug header described in 1.2.1 |
| 0x068 | 4 | ASID count | maximum virtual ASID count |
| 0x06c | 264 | random seed | `"randseed"` followed by 256 random bytes |
| 0x178 | 8 | random seed length | `0x108` |
| 0x180 | 1 | SK bootstrapped | `1` after successful cL4 bootstrap |
| 0x188 | 8 | SK carveout size | final cL4 cursor minus its initial allocation cursor |
| 0x190 | 4 | SPTM variant | `0` for release, `1` for development |
| 0x198 | 8 | XNU panic flag | VA of the persistent zero-initialized flag |
| 0x1a0 | 312 | `libsptm_state` | version-12 state described in 3.2 |
| 0x2d8 | 4 | tag-storage frame count | number of MTE tag-storage frames, dram size >> 19 |
| 0x2e0 | 8 | first tag-storage PA | `S3_0_C11_C9_0 & 0x3fffff00000` |
| 0x2e8 | 8 | AuxKC base | AuxKC PAPT base, or zero |
| 0x2f0 | 8 | AuxKC Mach-O | AuxKC Mach-O PAPT, or zero |
| 0x2f8 | 8 | AuxKC end | exclusive AuxKC PAPT bound; use the kernelcache end when no AuxKC is present |
| 0x300 | 8 | SK bootstrap timestamp | may be zero |
| 0x308 | 8 | XNU bootstrap timestamp | may be zero |
| 0x310 | 8 | TXM bootstrap timestamp | may be zero |
| 0x318 | 8 | SPTM initialization timestamp | may be zero |
| 0x320 | 8 | SK completion timestamp | may be zero |
| 0x328 | 8 | TXM completion timestamp | may be zero |
| 0x330 | 8 | reserved/hibernation metadata | zero for cold boot |
| 0x338 | 8 | hibernation scratch-page PA | zero for cold boot |
| 0x340 | 8 | pmap I/O ranges | PAPT pointer to the parsed range table |
| 0x348 | 4 | pmap I/O range count | number of range records |
| 0x350 | 8 | pmap I/O filters | PAPT pointer to the parsed filter table |
| 0x358 | 4 | pmap I/O filter count | number of filter records |
| 0x360 | 8 | SPTM feature flags | feature mask supplied to XNU |

Reserve the MTE range and initialize its frame-table entries as
`XNU_TAG_STORAGE` (type 32), so XNU cannot allocate it as ordinary memory.

The old `sptm_prev_ptes` pointer was removed. XNU obtains each CPU's output
area through endpoint 50 instead.

### 3.2 `libsptm_state` version 12

`libsptm_state` remains 0x138 bytes. Its version is `12`. The fields through
the per-CPU event-counter pointer at relative offset 0xd8 retain the layout in
1.2.2, however some previously reserved offsets are now used and some fields
have had their values changed:

| rel | size | field | required value |
|----:|-----:|-------|----------------|
| 0x000 | 4 | version | `12` (previously 10) |
| 0x070 | 8 | first tag-storage PA | same address as the outer handoff field at 0x2e0 |
| 0x078 | 8 | tag-storage end PA | first tag-storage PA plus the outer tag-storage frame count times 16 KiB |
| 0x0a8 | 8 | feature flags | `0x89`: MTE (`1 << 0`), the existing baseline feature (`1 << 3`), and SAPT (`1 << 7`) (previously 0x10) |

It also had more items appended to the end of the struct:

| rel | size | field | required value |
|----:|-----:|-------|----------------|
| 0x0e0 | 8 | DRAM base PA | first PA covered by SAPT |
| 0x0e8 | 8 | DRAM end PA | exclusive end PA covered by SAPT |
| 0x0f0 | 8 | SAPT table PAPT | PAPT VA of the SAPT range |
| 0x0f8 | 8 | version string | persistent VA of any NUL-terminated string |
| 0x100 | 8 | reserved | zero on physical systems |
| 0x108 | 48 | reserved | zero |

### 3.3 New XNU_BOOTSTRAP endpoints

macOS 27 adds the following domain-0, table-0 endpoints:
jj
| id | name | behavior | inputs | outputs |
|---:|------|----------|--------|---------|
| 35 | TAG_PAPT_MULTIPAGE | Set each data frame's tagged-PAPT state, change its PAPT leaf to MTE AttrIndx 4, increment `tag_storage_count` of the backing tag-storage frame, and invalidate the affected translations | `x0`=PA of an array of page PAs, `x1`=count (`1..64`), `x2`=options (`0x100` defers the TLBI) | `SUCCESS`, or `UPDATE_DELAYED_TLBI` when the TLBI is deferred |
| 36 | UNTAG_PAPT_MULTIPAGE | Clear each data frame's tagged-PAPT state, restore PAPT AttrIndx 0, decrement `tag_storage_count`, and invalidate the affected translations | same | `SUCCESS` |
| 50 | OUTPUT_AREA | Return the PAPT VA of the CPU's 16 KiB output area; XNU caches it during CPU initialization | `x0`=SPTM logical CPU ID | output-area PAPT VA |

Endpoint 18, `REGISTER_EXC_RETURN`, is also required when SK is enabled. It
takes XNU's exception-return trampoline VA in `x0` and retains it for exception
delivery; treating this endpoint as a no-op prevents interrupted RingGate calls
from resuming.

### 3.4 Other required ABI updates

- `UPDATE_DISJOINT` accepts option `0x400` (`ASYNC_TLBI`): update the PTE and
  issue the TLBI without the trailing synchronization. It applies only to
  generic XNU pages and only to endpoint 6.
- `SIGN_USER_POINTER` and `AUTH_USER_POINTER` take an additional saved JOP key
  in `x4`. `BATCH_SIGN_USER_POINTER` takes it in `x3`.
- A sub-page user root is 1024 bytes containing 128 TTEs. A 16 KiB frame has
  15 usable roots; the final 1024-byte slot is reserved for metadata.
- Frame type 39 is `XNU_RESTRICTED_IO_RO`; the later frame-type values move up
  by one. Frame type 67 is the new `SK_XNU_CONTENT` type.
