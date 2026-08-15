# Exclaves SPTM Dispatch

This document is written against m1n1 commit
`ee4e262ffbdd68d352f7ade798ebc5e4c8c8f6e3` on 2026-08-15.

This document covers the SPTM interfaces required to bootstrap ExclaveCore on
T8142 (running macOS 27.0 Beta 3 2), and dispatch calls between XNU and cL4:

- **Domain 0, table 2** `SK_BOOTSTRAP` — endpoints 0..3 (4)
- **Domain 0, table 4** `T8110_DART_SK`
- **Domain 0, table 12** `GEN3_DART_SK`
- **Domain 3, table 0** cL4 dispatch registration and RingGate

## 1. SK Bootstrap

During bootstrap, cL4 uses domain 0, table 2 to register its entry points and
manage the frame types of memory transferred between XNU and SK. The table has
four endpoints:

| id | name | behavior | inputs | outputs |
|---:|------|----------|--------|---------|
| 0 | REGISTER_DISPATCH_TABLE | Register an entry point through which one caller domain can enter cL4 | `x0`=cL4 table ID, `x1`=entry VA, `x2`=permitted-caller-domain mask | void on success, otherwise panic |
| 1 | RETYPE | Clean the frame for an ownership transition, update its type and type-specific metadata, and apply the SK flags | `x0`=PA, `x1`=expected current type, `x2`=new type, `x3`=flags | `SUCCESS` |
| 2 | PHYS_TO_TYPE | Read a managed frame's current type | `x0`=PA | frame type, or `SPTM_UNTYPED` for an unmanaged PA |
| 3 | IS_DEFAULT_TAGGABLE | Test whether the frame is `SK_DEFAULT` with its tagged-PAPT state set | `x0`=PA | boolean |

### 1.1 Registered dispatch tables

Endpoint 0 is called twice on T8142:

| cL4 table | caller | required caller-mask bit | use |
|---:|--------|-------------------------:|-----|
| 0 | XNU domain 1 | `1 << 1` | XNU-to-SK RingGate calls |
| 1 | SPTM domain 0 | `1 << 0` | SPTM-to-SK calls |

Retain the entry VA and caller mask for each table after bootstrap. RingGate
cannot begin until table 0 has been registered. All this endpoint does,
effectively, is give us two addresses that we can branch to, one used when XNU
calls Exclaves, and another when SPTM calls Exclaves.

### 1.2 Frame state

Endpoints 1 through 3 use the same shared frame table as XNU.  RETYPE must
leave the frame type and type-specific body coherent for both callers.

For transitions involving the SK CPU-frame types (`SK_DEFAULT`,
`SK_SHARED_RO`, `SK_SHARED_RW`, and `SK_XNU_CONTENT`), `x3` has these observed
flags:

- Bit 0 imports tagged XNU content into SK. Acquire the data frame's
  tagged-PAPT state and increment the corresponding `XNU_TAG_STORAGE` frame's
  `tag_storage_count`.
- Bit 1 marks an SK shared frame as realtime. Record that state and make its
  XNU PAPT alias non-cacheable.

When an SK CPU frame returns to `XNU_DEFAULT`, clear its realtime state and,
if it was tagged, release its tag-storage reference. Clean and invalidate the
frame before completing the ownership handoff.

## 2. DART SK

Tables 4 (`T8110_DART_SK`) and 12 (`GEN3_DART_SK`) are nearly identical except
for the behavior of the underlying DART generation. The current emulator just
uses the T8110 endpoints for Gen3 also and it appears to work fine.

| id | name | behavior | inputs | outputs |
|---:|------|----------|--------|---------|
| 0 | FLUSH_SID | Issue a DART TLB invalidation for the requested SID and wait for completion | `x0`=DART ID, `x1`=poller, `x2`=range start, `x3`=range end, `x4`=SID | boolean completion |
| 1 | TLBI_BARRIER | Wait for any outstanding DART invalidation to complete | `x0`=DART ID | boolean completion |
| 2 | ACQUIRE_CLOCK_PROTECTION | Prevent XNU from changing the clock-protection state of an initialized DART while SK is using it | `x0`=DART ID | boolean success |
| 3 | RELEASE_CLOCK_PROTECTION | Release SK's hold on the DART's clock-protection state | `x0`=DART ID | `1` |

The only additional internal state is, for each DART, whether SK is currently
holding it.

## 3. Exclave Boot and RingGate

RingGate is essentially a function call ABI, except the function is called via
a context switch to a secure world (the GL levels). We model Exclaves as a
separate virtual machine and patch all the `*_GL1` registers to their EL1
counterparts. In general, cL4 appears to be mostly self contained, not
requiring much in the way of context save/restore.

### 3.1 Address discovery

Runtime entry points are obtained through runtime registration as follows: 

| purpose | source |
|---------|--------|
| XNU-to-cL4 entry | cL4 dispatch-table 0 registration |
| SPTM-to-cL4 entry | cL4 dispatch-table 1 registration |
| XNU exception-return trampoline | XNU endpoint 18, `REGISTER_EXC_RETURN` |
| XNU and cL4 exception vectors | the `VBAR_EL1` in each saved per-CPU context |
| ordinary RingGate return address | the continuation saved when XNU enters RingGate |

### 3.2 Primary cL4 Boot

cL4 receives the physical address of this 0x134-byte handoff in `x0` at cold
entry:

| offset | size | field |
|-------:|-----:|-------|
| 0x000 | 8 | SPTM managed RAM base PA (same as `libsptm_state.first_phys`) |
| 0x008 | 8 | SPTM managed DRAM size (`libsptm_state.last_phys` - `libsptm_state.first_phys`) |
| 0x010 | 8 | first available PA after the cL4 allocations |
| 0x018 | 8 | `CL4-ro` PA |
| 0x020 | 8 | `CL4-rx` PA |
| 0x028 | 8 | `CL4-rw` PA |
| 0x030 | 8 | `CL4-le` PA |
| 0x038 | 8 | ADT PA |
| 0x040 | 8 | ADT size |
| 0x048 | 8 | CPU-topology pointer; unused by cL4 on this target |
| 0x050 | 0xc0 | `/chosen/cl4-entropy` |
| 0x110 | 8 | first scratch-page PA |
| 0x118 | 8 | second scratch-page PA |
| 0x120 | 8 | shared frame-table PA (same as XNU_BOOTSTRAP) |
| 0x128 | 8 | frame-table byte size (same as XNU_BOOTSTRAP) |
| 0x130 | 4 | managed DRAM page count |

The ExclaveCore bundle contains a series of MachO executables; we use the
metadata in these executables to calculate the `CL4-*` values, specifically:

The Type-4 BUND header points to a DER configuration in its `urst` range. That
configuration contains one set of `__MACHO__TEXTOFF`, `__MACHO__TEXTSZ`,
`__MACHO__DATAOFF`, `__MACHO__DATASZ`, `__MACHO__LINKEDITOFF`, and
`__MACHO__LINKEDITSZ` values for each embedded Mach-O.
`__MACHO__PDATAOFF` and `__MACHO__PDATASZ` pair.

The bundle packs all executable content first, followed by all writable data,
then all LinkEdit and protected-data content, and finally the read-only Mach-O
headers and BUND configuration. Derive the four protection groups as follows:

```text
CL4-rx = [0, minimum __MACHO__DATAOFF)
CL4-rw = [end of CL4-rx, minimum __MACHO__LINKEDITOFF)
CL4-le = [end of CL4-rw, maximum end of any LINKEDIT or PDATA range)
CL4-ro = [end of CL4-le, page-aligned end of the BUND)
```

`CL4-le` is the read-only, non-executable LinkEdit protection group. The
offsets above are relative to the beginning of the BUND. After choosing the
BUND's physical load address, add it to each offset to obtain the four PAs
placed in the handoff.

For the T8142 ExclaveCore bundle used here, the calculation produces:

| region | BUND offset | size |
|--------|------------:|-----:|
| `CL4-rx` | 0x0000000 | 0x18c4000 |
| `CL4-rw` | 0x18c4000 | 0x01d8000 |
| `CL4-le` | 0x1a9c000 | 0x005c000 |
| `CL4-ro` | 0x1af8000 | 0x00c4000 |

Map them at their image-defined virtual offsets with their corresponding
execute, write, and read permissions. Also provide the dummy page, the two
scratch pages, cL4's initial page tables, and an SK-owned bootstrap-allocation
area beginning at first available PA.

The bundle, boot handoff, dummy page, page tables, and bootstrap-allocation
area begin as `SK_DEFAULT`. The first scratch page has the macOS 27 scratch
type 0x40, and the second has type 0x02. The shared frame table must cover the
managed DRAM described by the handoff.

Modify the ADT to describe the four `CL4-*` regions, `CL4-entry`, `CL4-virt`,
and `CL4-dummypage`. It must also supply `/chosen/cl4-entropy` and the
ExclaveOS integrity-catalog and trust-cache payloads under
`/chosen/exclave-memory-map`.

cL4 starts in a separate execution context and address space at `CL4-entry`,
with `x0` equal to the handoff PA and `x1` through `x3` zero. Its initial
translation and control registers must describe the mappings above.

cL4 boots on a single core; later cores are bootstrapped when they drop out of
reset and registered via the `CPU_ONLINE` endpoint (described below). During
bootstrap cL4 uses the SK-bootstrap and DART-SK tables described above, and
registers its two runtime dispatch tables. cL4 completes bootstrap with control
command 1 and returns its final physical allocation cursor in `x0`.

At completion:

1. Save the completed cL4 context to be restored later upon Exclave call
2. Set XNU's `first_avail_phys` to the returned cursor.
3. Set `sk_bootstrapped` and record the cL4 carveout size as the difference
   between the initial and final cursors.
4. Return the unused tail of the reserved bootstrap-allocation area to
   `XNU_DEFAULT`.

### 3.3 Secondary CPU Bootstrap

Construct a fresh cL4 cold-entry context for each secondary CPU from the
following values:

| state | cold-entry value |
|-------|------------------|
| PC | `CL4-entry` |
| PSTATE | EL1h with exceptions masked (`0x3c5` in the EL1-based model) |
| `x0` | cL4 handoff PA |
| `x1`–`x28` | unspecified; zero is safe |
| `x29`–`x30` | zero |
| Stage-1 translation | initial cL4 `TTBR0_EL12`, `TTBR1_EL12`, `TCR_EL12`, `MAIR_EL12`, `AMAIR_EL12`, and `SCTLR_EL12` |

The following initial control values are known to work for this target:

```text
TCR_EL12   = 0x310800336511a511
MAIR_EL12  = 0x0c0d00f001a040ff
SCTLR_EL12 = 0x12001d50fc14793d
VBAR_EL12  = 0
```

Initialize CPU identity and all per-CPU architectural state independently. cL4
establishes its own stack and thread state during cold entry.

A secondary begins at SPTM's locked reset vector. Initialize its SPTM/m1n1
per-CPU state, then enter `CL4-entry` with the common handoff PA in `x0`.
When cL4 issues `RETURN_TO_CALLER`, retain the resulting cL4 execution context
for later RingGate calls and transfer control to XNU's secondary entrypoint.

### 3.4 Dispatch tables and selectors

| id | name | purpose |
|---:|------|---------|
| 0 | ENTER | Execute an ordinary XNU-to-Exclave request |
| 1 | INFO | Obtain cL4 boot information and the early-enter indication |
| 2 | CPU_ONLINE | Notify cL4 that the current CPU has come online |
| 3 | CPU_OFFLINE | Notify cL4 that the current CPU is going offline |

All four endpoints are identical as far as SPTM is concerned. Validate the
caller and selector, copy XNU's `x0`–`x7` and `x16` to cL4, and invoke cL4's
registered table-0 entrypoint. When cL4 returns, pass its result in `x0` back
to XNU. The endpoint-specific meaning is opaque to SPTM.

In addition to domain/table/endpoint calls, SPTM recognizes nonreturning
control commands in `x16[63:56]`; the rest of `x16` is zero. These commands
may be issued by either cL4 or XNU:

| command | name | issuer | behavior | inputs |
|--------:|------|--------|----------|--------|
| 1 | RETURN_TO_CALLER | cL4 | Complete the current call or cL4 bootstrap | `x0`=call result or final bootstrap cursor |
| 2 | PANIC | cL4 | Report a terminal cL4 failure | panic state |
| 3 | EXCEPTION_STATE_SAVED | cL4 | Report a saved interrupted RingGate execution | `x0`=vector type, `x1`=`scheduler_interrupted` |
| 6 | RESUME_SK | XNU | Resume cL4 after XNU handles the exception | `x0`=original caller return address |

For `EXCEPTION_STATE_SAVED`, vector type 0 is IRQ, 1 is FIQ, 2 is SError, and
3 is synchronous. `scheduler_interrupted` is a boolean indicating whether the
cL4 scheduler was interrupted.

### 3.5 Ordinary RingGate call and return

We describe two context types: an execution context and an ABI context. Each
CPU has a persistent execution context for XNU and cL4, which includes the
stack-pointer state, stage-1 translation state, exception and execution-control
registers, and thread/per-CPU registers. The ABI context consists of the
registers both sides expects. For cL4, this is `x0`-`x7` and `x16`. For XNU,
this is `x19`-`x30`, `d8`-`d15`, and `SP_EL0`.

On a normal XNU RingGate call:

1. Save XNU's execution and ABI context and PC and PSTATE.
2. Restore cL4's execution context.
3. Set cL4's PC to its registered table-0 entry and `x0` through `x7` and `x16`
   with the XNU supplied values; set PSTATE.
4. Run cL4 until it issues a nonreturning control command.

`RETURN_TO_CALLER` restores the saved XNU state and returns cL4's `x0`-`x7`
(though only `x0` is meaningful). Only one synchronous
RingGate call may be active on a CPU at a time.

### 3.6 Interrupt and resume

An interrupt during a RingGate call requires three distinct continuations to
coexist:

- the original XNU RingGate caller;
- the interrupted cL4 execution; and
- XNU's temporary exception-handler execution.

The original XNU call remains active while the other two contexts switch. When
an interrupt is received during cL4 execution, cL4's interrupt handler will
run, report `EXCEPTION_STATE_SAVED`, and `hvc` to SPTM. Then, to divert an IRQ,
FIQ, or SError to XNU:

1. Save cL4's execution context without completing the RingGate call.
2. Restore XNU's execution and ABI contexts.
3. Construct an architectural exception entry using XNU's saved `VBAR_EL1`
   and the vector offset selected by the saved PSTATE and exception class.
4. Place XNU's registered exception-return trampoline in `ELR_EL1` (previously
   registered by `REGISTER_EXC_RETURN`), and the saved XNU PSTATE in
   `SPSR_EL1`.
5. Enter the selected XNU exception vector in EL1h with exceptions masked.

After XNU handles the exception, its return trampoline observes there is an
interrupted RingGate call and invokes the `RESUME_SK` control command. SPTM
will then restore cL4 execution context and invoke the standard entrypoint.
cL4's eventual `RETURN_TO_CALLER` still completes the original XNU RingGate
call.

Our RingGate implementation goes beyond what SPTM does. We save and restore cL4
state as if it was any other VM, including full registers. We do not believe
this is necessary.

### 3.7 Internal state

#### 3.7.1 Bootstrap state

During primary cL4 bootstrap, retain the initial allocation cursor and reserved
allocation bounds until cL4 returns its final cursor. Use those values to update
the XNU bootstrap handoff, calculate the cL4 carveout, and release the unused
reservation. They are not required by later RingGate calls.

Retain the cL4 handoff address, `CL4-entry`, and cold-entry configuration for as
long as another CPU may require cL4 cold entry.

#### 3.7.2 Runtime-global state

The following state is shared by all CPUs:

- cL4 dispatch-table 0 and table-1 entry VAs and caller masks; and
- XNU's registered exception-return trampoline.

The caller mask must be checked whenever a registered dispatch table is used.

#### 3.7.3 Per-CPU state

Each CPU requires:

- the current execution domain and RingGate dispatch state;
- persistent XNU and cL4 execution environments;
- at most one active XNU-to-cL4 caller continuation; and

The active RingGate caller continuation must remain separate from XNU's live
execution environment. This allows XNU to handle an exception without
overwriting the state to which cL4's eventual `RETURN_TO_CALLER` must return.

Native SPTM enters cL4 with stage-2 translation disabled. An implementation
that uses separate virtual machines must additionally select its own stage-2
root and VMID during each context switch.
