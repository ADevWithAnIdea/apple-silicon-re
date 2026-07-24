# SART SPTM Dispatch

This document is written against commit
`219b0fc1a84b4b55da120fee4d456b1e8d62d839` on 2026-07-03.

This document covers the SART SPTM dispatch tables

- **Table 5** `SART` — endpoints 0..2 (3)

The overall behavior of SART is already understood in m1n1; this document
bridges the existing understanding to the sptm implementation.

## 1. Boot Handoff

Before XNU boots, seed the SART emulator from `/arm-io/sart-ans`: `reg[0]`,
`sart-version`, the current live SART entries, optional `reg[1]`,
`exclusive-bounds`, and optional `power-canary-offset`. The `reg[0]` MMIO
base, versioned register layout, and live-entry protected mask are the same
state that upstream m1n1 already derives in `sart_init()`.

Additional handoff details:

- `exclusive-bounds`: if present, the raw size field is a page count. If
  absent, active entries store `page_count - 1`, so readers add 1 and writers
  subtract 1. This selects the size-field encoding after `sart-version` has
  selected where that field lives.
- `power-canary-offset`: if present, `reg[1]` is interpreted as the
  power-canary MMIO base, and this property is the offset within that region.
  The canary word is `reg[1] + power-canary-offset`; guarded SART mappings
  write the sentinel value `0xabfedeed` there when the canary is first
  acquired. Mappings with the guard parameter set use this state; unguarded
  mappings do not.
- `sart-version`: identical to current m1n1 handling, except defaults to 3 if
  absent

## 2. Endpoints

SART only has three endpoints, 0, 1, and 2.

SART maintains very little internal state: only a copy of all iBoot provided
mappings (described in set state below) and a reference count used for guarded
calls. The emulator can likely ignore both of these.

Return `x0 = 0` on success, `x0 = 1` on failure.

### 2.1 Endpoint 0 Set State

Args: `x0 = state`, either 0 or 1

In our emulator, this is stubbed to return success. In real SPTM, state = 0
marks SART as inactive and 1 active. On the first activation, it iterates over
the iBoot provided mappings and, for each populated entry, validates that it
does not overlap SPTM managed RAM (panics if it does), and then stores that
mapping in SART internal state. On subsequent activations, the stored mappings
are copied to the live state for all non empty mappings, effectively resetting
the SART state to the iBoot baseline for all mappings provided by iBoot.

### 2.2 Endpoint 1 Map Region

Args: `x0 = paddr`, `x1 = size`, `x2 = perm` either 0 or 1, `x3 = guard` bool 

Aside from some extra validation, the behavior is already identical to m1n1
except for the `perm` and `guard` arguments:

- For the v2/v3 layouts, `perm = 1` writes the full allowed flag pattern
  already used by m1n1 (`0xff` for v3); `perm = 0` uses the same valid-bit
  pattern with the access value reduced from 3 to 2 (`0xea` for v3).
- If `guard = 1` and the power canary exists, the first guarded mapping writes
  the sentinel value to the canary word and increments the internal guarded
  reference count. Later guarded mappings only increment the count. SPTM stores
  the guard bit with the mapping so unmap can release it.

### 2.3 Endpoint 2 Unmap Region

Args: `x0 = paddr`, `x1 = size`

Behavior is identical to m1n1 except in handling of guarded entries and
handling of protected entries.  If the stored mapping was guarded and the power
canary exists, SPTM checks that the canary word is still `0xabfedeed` and the
reference count is above 0.  Violation of either of those conditions results in
an immediate panic. The refcount is then decremented.

Note that unlike m1n1, SPTM has no concept of a protected slot, and appears to
be capable of removing iBoot provided mappings. We do not believe this affects
boot because XNU should only unmap mappings it created.

It is very likely the emulator can ignore the guard entirely.
