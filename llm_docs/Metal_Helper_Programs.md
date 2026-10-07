# Apple9 / G16 Metal helper programs — behavioural documentation

> **Provenance: this document draws on disassembly of the supplied vendor binaries**, performed
> under explicit authorisation for this workspace. It is therefore **not** clean-room material and
> **must not** be used in Mesa, in `agx-re`, or in any other project whose contribution policy
> requires freedom from vendor disassembly. It is kept in this directory for convenience; the
> clean-room boundary here is the label, not the location.

| | |
|---|---|
| **Bundle** | `metal-helper-review-20260908` |
| **Platform** | M4 Mac mini — `Mac16,10` / `J773g` / `T8132`, GPU G16G / Apple9 |
| **Baseline** | System software 26.6.2, build 25G83 |
| **Scope** | The nine consumed helper inputs plus the render container's retained blocks |

Confidence: **[C]** established, **[P]** probable, **[U]** uncertain. Unmarked reads as **[C]**.

---

## 1. Summary of what these programs are

The helpers fall into three functional groups:

| Group | Members | Function |
|---|---|---|
| **Compute carriers** | ordinary and atomic `constant`/`launch` pairs | Establish the resource/argument environment, then call the generated compute main |
| **Compute support data** | division table | A lookup table, not a program |
| **Render stages** | VS and three FS launchers, adapters, colour/tile programs | Establish per-stage argument environments and perform tile-level colour operations |

Everything else in the render container is template, state, padding or free space (§6).

---

## 2. Common structure

### 2.1 The archive block chain

The archive is a gapless chain of size-prefixed blocks. Each block header is **64 bytes: one
32-bit little-endian total-size field followed by 60 reserved zero bytes.** There is no magic
value, tag or checksum. Walking the chain reproduces every published block offset and terminates
cleanly. **[C]**

Within a block, the main program normally begins at **block + 0x80** — that is, after the 64-byte
block header and a 64-byte constant region. Two blocks in the shipped container instead begin at
**block + 0x40**, having no constant region at all. An implementation must not assume a uniform
entry offset. **[C]**

### 2.2 The call-record encoding

Every patch site named in the bundle inventory is the same construct: a call record whose target is
a **24-bit little-endian field at offset +2 of the record**. The encoding is closed-form:

```
field = 2 × target_offset + 0x2a
```

This was derived independently three times, by two different routes, and reproduces every shipped
template value exactly. It is 18 bits wide in practice, which is why the container's own archive
arena — 64 KiB, not the 96 KiB sometimes assumed — needs the extra bit once extended. **[C]**

### 2.3 Program termination

Programs end on a single stop instruction, followed by verified zero fill. Program extents can
therefore be established exactly rather than inferred, and every program documented here was closed
that way: first byte to terminator, no leftover bytes. **[C]**

---

## 3. Compute carriers

### 3.1 Layout

Neither launcher has a header. **Code begins at byte 0** and runs to a single terminator. **[C]**
Both shipped `constant.bin` files are byte-for-byte **slices of their own launcher**, not
independent programs. **[C]**

### 3.2 The resource contract

Both carriers publish one 128-byte record, zero-filled first, at a fixed package offset with a
128-byte stride. The difference between them is entirely **where the caller's buffers land**:

| | Ordinary | Atomic |
|---|---|---|
| Hidden resource slots | **3** | **0** |
| Slot 0–1 | Dispatch-geometry word and pointer | *(caller buffer 0–1)* |
| Slot 2 | **Division-table address** | *(caller buffer 2)* |
| Slots 3–10 | The eight caller buffers | *(caller buffers 3–7, then unused)* |
| Slot 11 | Zero sentinel | unused |
| Unbound slot fill | **The division-table address**, not null | **Zero** |

So visible resource *j* lands at slot *j + 3* on the ordinary carrier and slot *j* on the atomic
one. The three-slot prefix is read directly off the launcher's own load offsets. **[C]**

The unbound-slot fill difference matters: an ordinary-carrier slot with no bound buffer holds a
**valid pointer to the division table**, not a null. A driver that tests for null to detect an
unbound slot will not work. **[C]**

### 3.3 Ordinary versus atomic — and which difference enables atomics

The complete driver-visible delta is six items: the call-record offset, the hidden-slot count, and
four ABI configuration words. Structurally the two launchers differ only in prefix plumbing —
six versus four record loads, thirty-two versus twenty uniform moves, a different state register
pair, and a longer setup body. **[C]**

Exactly one difference is **flag-shaped**: a single word, same opcode in both, with two bytes
changed from zero to `0x10`, positioned where a coherency or cache descriptor would sit. This is
the most plausible atomic enabler. **[P]**

The competing candidate is one of the ABI configuration words, which also differs. **[P]**
**Neither is confirmed** — the two carriers are independent captures that share only a short
prologue, so most of their byte difference is incidental and cannot be attributed. Settling this
needs one experiment: run device atomics against a carrier built with the ordinary prefix and the
atomic flag word. The bundle contains no such experiment. **[U]**

### 3.4 The division table

**Fully solved.** 1024 entries of 8 bytes: a 32-bit magic multiplier and a 32-bit shift, indexed by
**divisor − 1** for divisors 1…1024 — the threadgroup-size limit.

```
s = floor(log2(d − 1))
m = floor((2^(32+s) − 1) / d)
```

It implements `ceil(n / d) = (mulhi_u32(n, m) >> s) + 1`, converting per-axis thread counts into
threadgroup counts. Verified across all 1024 entries with zero mismatches, and across tens of
thousands of dividend/divisor pairs with zero failures, over exactly the dispatch range the driver
enforces. **[C]**

It is installed unconditionally because it serves the carrier's dispatch-geometry machinery, not
the application's arithmetic — which is why it is present even when the compute main performs no
division. **[C]**

### 3.5 The constant-region size question

The supplier describes these files as the constant-program **portion** of the archive block, and
that wording is accurate and worth noting.

The driver mandates a **64-byte** constant region and places the main immediately after it. The
shipped ordinary launcher's **unpatched** call field, however, encodes a main 128 bytes further
along, implying the original capture had a **192-byte** constant region. The driver re-patches the
call, so its own layout is self-consistent and this is **not** a defect in that path. **[C]**

What is **not established** is whether the shipped 64 bytes are a self-contained program or the
first third of a longer one that happens to work because control falls through into the main. Anyone
building a carrier outside the driver's layout should determine this rather than assume. **[U]**

---

## 4. Render stages

### 4.1 The three fragment variants

Buffer-only, textured and coverage/discard setups differ in **exactly four** respects; everything
else is byte-identical. **[C]**

| | Buffer-only | Textured | Coverage / discard |
|---|---|---|---|
| Root pointer loads | 2 | 3 | 4 |
| Publication records | 8 | 12 | 16 |
| Fixed table pointer pair | — | — | **present** |
| Pixel-order byte | `f3` | `f3` | **`f2`** |

They are therefore one parameterised model with four parameters, answering the bundle's question 4
in the affirmative. Coverage's fourth root pointer has **no driver-populated source** — an open
gap. **[U]**

### 4.2 The vertex path

The VS launcher's call record targets **stage adapter B**, which is therefore the **vertex prolog**.
This is established from the call site, not from the block's integration label. **[C]**

The vertex adapter block is likewise a **live called program**: a container-internal call field
targets its entry, and the driver re-derives the same target and never overwrites the block. **[C]**

### 4.3 Colour and tile programs

The background/reload and clear-path load programs are **byte-identical except for one byte**. That
byte is a scaled compact pointer selecting which target-graph record to operate on. They are one
program and an operand, not two programs. **[C]** The end-of-tile store program is separate.

### 4.4 The opaque suffix blocks

| Block | Finding |
|---|---|
| **Suffix A** (34 KB) | A **single program**, not a library and not data. Branch-dense: hundreds of matched push/reconverge pairs and several hundred conditional jumps, roughly five loads and **zero stores**. A pure decision body. **Called by exactly one thing: the end-of-tile store program.** **[C]** |
| **Suffix B** (6.8 KB) | Two programs — one short and straight-line, one larger and memory-active. **[C]** |
| **Suffix C** (384 B) | One program, fully decoded. **[C]** |

**No data was found in any suffix block.** They are entirely code. **[C]**

Nine archive blocks have **no static caller anywhere** in the container. That does not prove they
are unreachable — an indirect or table-driven entry would not show — but it bounds the search for
minimal closure. **[P]**

---

## 5. On the stall-when-removed result

Existing notes report render stalls after removing nonzero suffix entries. That establishes an
**envelope or layout dependency only.** It cannot distinguish "this block executes" from "removing
it broke an intra-archive reference or shifted a required offset", because the archive is a gapless
size-prefixed chain in which any removal moves everything after it.

It should not be cited as evidence that every retained block executes on every draw. **[C]**

---

## 6. What the container actually is

Roughly **92% of the 4 MiB container is fill**, including about 1 MiB of a repeated filler byte
covering regions that are **destinations rather than data** — space published into a fixed view and
written at runtime. Approximately **1.5%** is established as load-bearing. **[C]**

The container's own archive arena terminates at 64 KiB. **[C]**

---

## 7. Open questions

1. **Which difference enables device atomics** (§3.3). One experiment settles it.
2. **Whether the shipped 64-byte ordinary constant region is self-contained** (§3.5).
3. **The source of the coverage variant's fourth root pointer** (§4.1).
4. **What the stage adapters, runtime libraries and colour epilog do.** Suffix A now has a
   behavioural answer; these do not. **[U]**
5. **Whether the nine uncalled archive blocks are reachable indirectly** (§4.4).
6. **The low selector field of the tile-program operand** (§4.3). **[U]**
