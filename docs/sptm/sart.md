# SART SPTM Dispatch

This document is written against commit
`f5f8d2c7027d4ae37e4db27ea33720fe98151445` on 2026-07-27.

This document covers the SART SPTM dispatch tables

- **Table 5** `SART` — endpoints 0..2 (3)

The overall behavior of SART is already understood in m1n1; this document
bridges the existing understanding to the sptm implementation.

## 1. Boot Handoff

Before XNU boots, seed the SART emulator from `/arm-io/sart-ans`: `reg[0]`,
`sart-version`, the current live SART entries, optional `reg[1]`,
`exclusive-bounds`, and optional `power-canary-offset`. Upstream m1n1 already
derives the MMIO base, versioned register layout, and current entries in
`sart_init()`, however, SPTM has no protected-slot policy. There is nothing in
SPTM that prevents it from removing a boot seeded entry.

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
- `sart-throttle-version`: selects one of the built-in throttle-register
  layouts below. If absent, use `sart-version`. Valid values are 1 through 4.
- `sart-throttle-offset`: an additional byte offset applied to every throttle
  register. If absent, use zero. Each register is a 32-bit register at
  `reg[0] + sart-throttle-offset + table_offset`.

| Version | Status-register offsets | Control-register offsets | Activity mask |
|---|---|---|---|
| 1 | `0x4010`, `0x4018`, `0x4020` | `0x4014`, `0x401c`, `0x4024` | `0x0000ffff` |
| 2 | `0x8010`, `0x8018`, `0x8020` | `0x8014`, `0x801c`, `0x8024` | `0x0000ffff` |
| 3 | `0x8010`, `0x8018`, `0x8020` | `0x8014`, `0x801c`, `0x8024` | `0x0000ffff` |
| 4 | `0x8000`, `0x8004`, `0x8010`, `0x8018` | `0x8008`, `0x800c`, `0x8014`, `0x801c` | `0x0000ffff` |

## 2. Endpoints

SART only has three endpoints, 0, 1, and 2.

SPTM retains the complete current mapping set, initially imported from live
hardware state on first activation and updated by endpoints 1 and 2, so it can
replay those mappings on later activations. It also tracks which mappings are
guarded, and the power-canary reference count.

All endpoints return `x0`=0 on success; endpoint 2 additionally returns `x0`=1
on retry.

### 2.1 Endpoint 0 Set State

Args: `x0 = state`, either 0 or 1

In our emulator, this is stubbed to return success. When `state = 0`, SPTM
marks SART inactive without modifying the live mapping slots. On the first
`state = 1` call, it reads every populated hardware slot, rejects mappings that
overlap SPTM-managed RAM, and records them as the initial current mapping set.
On later `state = 1` calls, it reprograms every currently recorded mapping into
its hardware slot.

### 2.2 Endpoint 1 Map Region

Args: `x0 = paddr`, `x1 = size`, `x2 = perm` either 0 or 1, `x3 = guard` bool 

Hardware-visible behavior is the same as m1n1's existing SART region
programming, with the following differences:

- `perm`: for the v2/v3 layouts, `perm = 1` uses the full allowed flag pattern
  already used by m1n1 (`0xff` for v3), while `perm = 0` uses the
  reduced-access pattern (`0xea` for v3).
- `guard`: if `guard = 1` and the power canary exists, the first guarded
  mapping writes `0xabfedeed` to the canary word. Every guarded mapping
  increments the power-canary reference count.
- `exclusive-bounds`: encode the size as a page count when the property is
  present and as `page_count - 1` otherwise.

SPTM records the selected slot, physical range, `perm`, and `guard` value.
Endpoint 0 uses this record when replaying mappings, and endpoint 2 requires
the supplied physical range to match it exactly.

For every 16 KiB frame in the range, increment `ro_refcount` when `perm = 0`
or `wx_refcount` when `perm = 1`. Endpoint 2 decrements the corresponding
counter when the mapping is removed. Our emulator does not enforce these
frame-table counters.

### 2.3 Endpoint 2 Unmap Region

Args: `x0 = paddr`, `x1 = size`

Endpoint behavior depends on whether `guard` is set in the stored mapping
record.

If `guard = 1`, SPTM first writes zero to every register forming the selected
SART mapping slot. It then writes zero to each throttle control register and
issues `DSB SY` after every write. It reads the corresponding status registers
and considers activity present if any value has a nonzero bit under the
version's activity mask.

If either no activity was detected or any status register has a bit set under
`0x01010000`, SPTM restores the recorded control-register values and issues
`DSB SY` after every write. If activity was detected, endpoint 2 returns
`x0 = 1`, including when the control registers were restored. The SART mapping
slot remains zero, while the internal mapping record, power-canary reference
count, and frame-table reference counts retain their previous values for the
retry.

Once no activity is detected, if the power canary exists, SPTM requires its
reference count to be nonzero and its word to contain `0xabfedeed`, then
decrements the reference count.

If `guard = 0`, behavior is close to m1n1's existing SART removal. SPTM does
not itself write zero to the hardware slot; the slot's physical-address
register must already contain zero and its decoded size must be smaller than
two pages.

After either path succeeds, SPTM writes zero to the mapping's internal record
and decrements `ro_refcount` or `wx_refcount` for every 16 KiB frame according
to the mapping's stored permission.

Our emulator writes zero to the selected SART mapping slot, performs the
power-canary check when applicable, and returns success. It does not access the
throttle registers or return the retry result.
