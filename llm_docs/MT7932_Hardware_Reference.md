# MT7932 Hardware Reference

**Scope.** This document specifies the MediaTek **MT7932** combined Wi-Fi / Bluetooth
controller as it appears to a host driver over PCI Express. It is written to be sufficient,
together with hardware tracing, to implement an independent kernel driver for the Wi-Fi
function.

It documents *silicon behaviour*: register addresses, bit encodings, DMA structures,
firmware container formats and the firmware control protocol. It does not describe any
existing driver's software design. Where a value is a host implementation choice rather
than a hardware constraint, it is labelled as such.

---

## What MT7932 is

| Property | Value |
|---|---|
| Class | 2×2 **802.11ax (Wi-Fi 6E)**, tri-band 2.4 / 5 / 6 GHz |
| Maximum bandwidth | 40 MHz (2.4 GHz), 160 MHz (5 GHz), 160 MHz (6 GHz) |
| 802.11be (EHT) / multi-link | **Not supported** |
| Host interface generation | **CONNAC2** (`mt792x` class) |
| TX / RX descriptors | TXD v2 (32 B + 32 B append), RXD v2 (24 B) |
| MCU control scheme | Legacy CONNAC2 command/event — **no** unified/TLV command set |
| Register aperture | BAR 0, 1 MiB static window plus programmable remap slots |
| DMA addressing | 32-bit |
| Spatial streams | 2 |

**The closest publicly documented parts are MT7921 and MT7922**, covered by the in-tree
Linux `mt76` driver (`mt792x` common code plus `mt7921/`) and by MediaTek's published
`gen4m` driver. On every axis that matters to a driver's DMA, descriptor, interrupt and
MCU plumbing, MT7932 behaves as an MT7922.

**MT7932 is not related to MT7925 or MT7927.** Those are CONNAC3 parts with a different
descriptor format, a TLV command scheme, a 2 MiB register window and 34-bit DMA. Starting
an MT7932 driver from the `mt7925` code would be starting from the wrong place, despite the
part number ordering.

The MT7932 firmware is built from the MT7923 project tree; the part is best understood as
MT7923-class MAC/PHY with 6 GHz added.

## How to read this document

Sections 1–10 specify the hardware by subsystem. Section 11 is the delta analysis — read it
first if you already have a working MT7921/MT7922 driver. Section 12 is the initialisation
ordering. Section 13 is the consolidated register index, address map, command/event quick
reference and a day-one bring-up sequence.

Assume MT7921/MT7922 behaviour wherever this document does not say otherwise; every point
at which MT7932 diverges is called out explicitly.

**Confidence markers.** Every non-obvious claim carries one of:

| Marker | Meaning |
|---|---|
| `[C]` | Confirmed — established directly and unambiguously from the analysed artefacts |
| `[L]` | Likely — strongly implied, including behaviour inherited from public parts of the same family and assumed to hold here |
| `[U]` | Unverified — requires hardware tracing to confirm |

## Platform context

The part is used in an Apple Silicon Mac whose board identifier is `j700ap` and whose SoC
identifier is `t8140`. On that platform, Wi-Fi and Bluetooth are **separate PCI functions**
with separate host drivers, and the Wi-Fi function is declared tunnel-compatible, so a
driver must tolerate the device appearing behind a PCIe switch rather than on a root
port. `[C]`

## PCI identity

| Function | Vendor ID | Device ID | Class |
|---|---|---|---|
| Wi-Fi | `0x14C3` | `0x7932` | `02 / 80 / 00` |
| Bluetooth | `0x14C3` | `0x793B` | `02 / 80 / 00` |

The chip-ID register value is `0x7932`. `[C]` The same host interface is presented by the
sibling device IDs `0x7922` (MT7922, also `0x0616`) and `0x7923` (MT7923). `[C]`

The Bluetooth function is **MT7923-class** and is served by MT7923 Bluetooth firmware; no
MT7932-specific Bluetooth image exists. `[L]` A Bluetooth driver for this package should be
an MT7923 Bluetooth driver with the device ID added. This document does not otherwise cover
the Bluetooth function.

## Firmware and data artefacts

The part requires host-supplied firmware. A platform may prefix the shipped copies with a
vendor-specific string; the names below are the un-prefixed forms.

| Artefact | Size | Role |
|---|---:|---|
| `WIFI_MT7932_patch_mcu_1_2_hdr.bin` | 33 568 B | ROM patch, applied before the RAM firmware is started `[C]` |
| `W7932_2.bin` | 1 190 788 B | Main Wi-Fi MCU firmware, 5 regions `[C]` |
| `WIFI_MFG_MT7932_2.bin` | 787 328 B | Manufacturing / test-mode firmware `[C]` |
| `WIFI_MFG_MT7932_patch_mcu_1_2_hdr.bin` | 46 816 B | Manufacturing ROM patch `[C]` |
| `EEPROM_MT7932_1.bin` | 2 560 B | Radio provisioning, PCIe identity for both functions, MAC address `[C]` |
| `PPR_MT7932.bin` | 412 B | Per-rate transmit-power overlay, `BLOB` container `[C]` |
| `TxPwrLimit*.dat` | — | Regulatory limits, 6 GHz limits, SAR, antenna gain, common path `[C]` |
| `WIFI_RAM_CODE_MT7932_1_2_idxlog.bin` | 192 778 B | Decoding table for compressed firmware log records `[C]` |

Development-labelled variants of the RAM firmware and ROM patch also ship; they are separate
builds, not duplicates. `[C]` Section 5 specifies the container formats; section 9 specifies
the calibration data.

## Summary of MT7932-specific behaviour

A driver derived from an MT7921/MT7922 code base must change these. Each is specified in
full in the section named.

| # | Difference from MT7922 | Section |
|---|---|---|
| 1 | Wi-Fi function readiness is read from **PCI configuration space**, not from a chip register | 1, 2, 5 |
| 2 | A **fabric/clock validity check** in configuration space must pass at probe | 1, 2 |
| 3 | An **MTCMOS power-domain sequence** is required that MT7922 does not perform | 2 |
| 4 | The ownership-interrupt status clear after a subsystem reset must be **skipped** | 2, 4 |
| 5 | **ROM-patch semaphore request and reply values are shifted by +1** | 5, 11 |
| 6 | **Boot status codes are shifted by +1 and extended** — success is `1`, not `0` | 5, 6, 11 |
| 7 | A **secure-boot gate** must be held around patch-finish and firmware-start | 5 |
| 8 | The band-selection command uses a different body version and a per-BSS band bitmap | 10, 11 |
| 9 | Larger 2.4 GHz calibration block counts | 9, 11 |
| 10 | A **conn-infra remap array** must be programmed to reach regions the static map misses | 1 |
| 11 | The revision table's second stepping differs from the public MT7921 table | 11 |
| 12 | The 6 GHz capability record is honoured (it is discarded on MT7923) | 11 |

Item 6 is the one most likely to be missed: a driver that tests for the public success code
will read every successful firmware boot as a failure.


---

# MT7932 — PCIe Function, MMIO Address Space and Register Access

**Scope.** This document specifies the host-visible PCIe interface of the MediaTek MT7932
combo Wi-Fi/Bluetooth part: PCI configuration-space presentation and required host setup,
DMA addressing capability, MMIO register-access rules, the complete fixed BAR→chip address
decode map, the programmable remap windows used to reach chip addresses outside that map,
the chip-identity/version registers, and the PCIe-MAC block registers the host must
program. It does **not** cover WFDMA ring programming, TX/RX descriptor formats, MCU
command/event formats or firmware download, which are specified elsewhere.

The MT7932 presents a **CONNAC2 (mt792x-class) PCIe host interface**. Its
hardware-configuration record is byte-for-byte identical to the MT7922 record except for
the chip-ID constant, so every register address below is shared with MT7922 unless
explicitly marked as a delta. Where MT7932/MT7922 differ from the *public* MT7921
(gen4m `mt7961`, upstream `mt76/mt7921`) that is called out explicitly.

Confidence markers: `[C]` confirmed, `[L]` likely, `[U]` unverified.

---

## 1. PCI function presentation

### 1.1 Identification

| Field | Value | Notes |
|---|---|---|
| Vendor ID | `0x14C3` | MediaTek Inc. `[C]` |
| Device ID | `0x7932` | MT7932 `[C]` |
| Subsystem vendor/device | wildcard | not used for matching `[C]` |
| Class / class mask | wildcard | not used for matching `[C]` |

The same host interface is presented by the sibling device IDs `0x7922`, `0x7923` and
`0x0616` (the last reports itself as an MT7922-class part). `[C]`
Chip-ID register value and EEPROM chip-ID field are both `0x7932`. `[C]`

### 1.2 BARs

| BAR | Type | Size | Contents |
|---|---|---|---|
| 0 | 32-bit memory `[L]`, non-prefetchable `[L]` | ≥ 1 MiB; **only the first `0x0010_0000` is used** `[C]`; the size the BAR actually decodes is `[U]` | Entire register aperture (see §4) |
| others | not used | — | The host maps BAR 0 only; no second BAR, no I/O BAR, no expansion ROM is touched `[C]` |

* The register aperture used is exactly **1 MiB** (`0x100000`). Every host register access is a
  BAR-0 offset in `0x00000 … 0xFFFFF`. `[C]` (that the BAR decodes no more than this is `[U]`, OQ 1)
* BAR-0 offsets **`0x00000 … 0x02000` are not used** by the host: the address-translation
  path only accepts direct BAR offsets strictly greater than `0x2000` and strictly less
  than `0x100000`, and no chip-address map entry resolves below `0x02000`. `[C]`
  On the closest public CONNAC3 part this region holds the `WF_MCU_BUS_CR` (ap2wf remap)
  block; for MT7932 the contents are unverified. `[U]`
* Delta vs. public MT7921: upstream `mt76` treats *any* address below `0x100000` as a
  direct BAR offset. On MT7932 the first 8 KiB is additionally excluded from
  direct BAR-offset addressing. `[C]` for the exclusion; that the *silicon* requires it (rather
  than it being a host convention) is `[L]`.

### 1.3 Required configuration-space setup

Standard header:

| Offset | Access | Action |
|---|---|---|
| `0x04` COMMAND | RMW 16-bit | Set Bus Master Enable (and Memory Space Enable) before any DMA `[C]` |
| `0x10`, `0x14` | read | BAR 0 base (low/high dwords); read back for diagnostics `[C]` |
| `0xFC` | read | read for diagnostics; contents unverified `[U]` |
| PM capability | — | Device is placed in D0 via the standard PCI PM capability before use `[C]` |
| PCIe capability | — | Capability offset is discovered at runtime, not hard-coded `[C]` |
| PCIe cap + `0x08` (Device Control) | write 16-bit | FLR is issued by setting **Initiate Function Level Reset (bit 15)** `[C]` |
| PCIe cap + `0x0A` (Device Status) | poll 16-bit | Wait for **Transactions Pending (bit 5)** to clear before FLR; poll budget ≈ 1 s `[C]` |

Advanced Error Reporting extended capability is at **`0x200`** `[C]`:

| Offset | Register | Host use |
|---|---|---|
| `0x204` | AER Uncorrectable Error Status | Written `0xFFFFFFFF` to clear all before arming AER; read/cleared in the error handler `[C]` |
| `0x208` | AER Uncorrectable Error Mask | **Arm**: write `0x0040_0000` (unmask only bit 22). **Disarm**: write `0x0057_F010` `[C]` |
| `0x210` | AER Correctable Error Status | Read/cleared in the error handler `[C]` |

**Vendor-specific configuration registers.** MT7932/MT7922 expose a block of
vendor-defined dwords used for combo (Wi-Fi + Bluetooth) coordination and for reporting
internal fabric/power state. These are *not* present in any public MediaTek driver.

| Offset | Direction | Meaning |
|---|---|---|
| `0x484` | W | **Notify Bluetooth of an imminent Wi-Fi FLR.** Write `0x0000_0001`, then wait ≥ 1 ms before issuing the FLR `[C]` |
| `0x488` | R | **Internal fabric / clock status.** Bit map validated at probe (see below) `[C]` |
| `0x48C` | R | **Wi-Fi function state**, in bits **[19:16]** `[C]` |

`0x48C[19:16]` encoding:

| Value | Meaning |
|---|---|
| `0` | Wi-Fi function is off / held in reset (also the "power-off completed" condition) `[C]` value, `[L]` meaning |
| `2` | Wi-Fi function ready / running `[C]` value, `[L]` meaning |

`0x48C[19:16] == 0` is the completion condition polled after FLR (up to 3 FLR attempts) and
after a requested power-down; `== 2` is the readiness condition polled (5 s budget, 5 ms
interval) after firmware start. `[C]`

**This is a genuine MT7932/MT7923 delta.** On MT7922 (device ID `0x7922`) the equivalent
readiness/power-off condition is read from the chip register `CONN_ON_MISC`
(`0x7C06_00F0`), bits `[1:0]` (`FW_PWR_ON | FW_N9_ON`, the public
`MT_TOP_MISC2_FW_N9_RDY` field). On MT7932 the host instead uses configuration
register `0x48C[19:16]`. `[C]` that the source is selected by device ID; `[L]` that the
`CONN_ON_MISC` bits are unusable on MT7932 silicon — that has not been tested.

**Fabric validity check (`0x488`).** Performed on MT7932/MT7923 only (skipped for device ID
`0x7922`) `[C]`. It is a general bus/fabric-liveness guard rather than a one-off probe step:
it is invoked from the DMA-hang detector, from error recovery and from the debug dumps `[C]`.
Two depths exist — the seven checks below always, plus four more in the deeper mode. All of
the checked bits must hold or the fabric or one of its power domains is treated as down:

| Bit | Required value | Check group |
|---|---|---|
| 0 | 1 | fabric 1.1/1.2 |
| 1 | 1 | fabric 1.1/1.2 |
| 4 | 0 | fabric 1.1/1.2 |
| 6 | 1 | fabric 1.1/1.2 |
| 9 | 1 | fabric 1.1/1.2 |
| 25 | 1 | fabric 1.1/1.2 |
| 26 | 1 | fabric 1.1/1.2 |
| 10 | 0 | fabric 2 (checked only in the deeper validation mode) |
| 13 | 1 | fabric 2 |
| 14 | 1 | fabric 2 |
| 15 | 1 | fabric 2 |

A read of `0x488` returning `0xFFFFFFFF` means configuration access itself has failed. `[C]`

### 1.4 Interrupts / MSI

* The hardware-configuration record declares a maximum of **8 MSI vectors** and carries two
  vector-layout tables: an 8-entry multi-vector layout and a 1-entry single-vector layout `[C]`.
  That the endpoint's MSI capability really advertises *Multiple Message Capable = 8* is `[U]`
  (OQ 14). Single-vector
  operation is what the observed host configuration actually uses; the interrupt section
  (§4 of this specification) is the authority on the vector map and on the service
  discipline.
* Each vector-layout entry carries **three routing fields**: a **TX-ring-index bitmap**, an
  **RX-ring-index bitmap** and a literal host-interrupt-status mask. The two bitmaps are
  *ring indices*, not host-interrupt-status bit numbers: bit *n* of the RX bitmap selects RX
  ring *n*, and the concrete acknowledge mask is obtained by expanding each selected ring
  through the per-ring interrupt-bit map of §7.2. `[C]`

In the table below the bitmap and literal-mask **values** are `[C]`, and the expanded
acknowledge masks are `[C]` arithmetic over the confirmed ring→bit map of §7.2; the **Source**
column is an interpretation of each entry's ring bitmap and is `[L]`.

| Vector | Source | TX-ring bitmap | RX-ring bitmap | Literal status bits | Expanded acknowledge mask |
|---|---|---|---|---|---|
| 0,1,2 | unused | — | — | — | `0x0000_0000` `[C]` |
| 3 | RX data | — | `0x0000_0004` → RX ring 2 | — | `0x0000_0004` `[C]` |
| 4 | TX-free-done report | — | `0x0000_0008` → RX ring 3 | — | `0x0000_0008` `[C]` |
| 5,6 | unused | — | — | — | `0x0000_0000` `[C]` |
| 7 | "lump" (everything else) | `0x0007_0000` → TX rings 16, 17, 18 | `0x0000_01F1` → RX rings 0, 4, 5, 6, 7, 8 | `0x2000_0000` | `0xEEC8_0001` `[C]` |
| single-vector layout | all | `0x0007_0000` → TX rings 16, 17, 18 | `0x0000_01FD` → RX rings 0, 2, 3, 4, 5, 6, 7, 8 | `0x2000_0000` | `0xEEC8_000D` `[C]` |

  **Do not read `0x0000_01F1` / `0x0000_01FD` as host-interrupt-status masks.** They are RX
  ring-index bitmaps; read as status masks they would wrongly imply data-TX-done bits are in
  use, and **no data-TX-done bit is used on this part at all** (§7.2, and §4 of this
  specification). `[C]`
* Vector 0..7 ordering is *not* the gen4m `pcie_msi_wfdma_ring` ordering; treat the table
  above as authoritative. `[C]`
* MSI/MSI-X capability enable itself is performed by the host OS; **no** vendor-specific
  programming beyond the PCIe-MAC interrupt-enable register (§7.1) is performed to enable
  MSI. `[C]` (absence). That the device *requires* none — in particular for multi-vector
  operation — is `[L]`; see OQ 13.

### 1.5 Bridge / tunnel considerations

The Wi-Fi function is one function of a **combo die that also carries a Bluetooth
function** `[C]`; that the two share the conn-infra fabric, the CB-TOP reset unit and the
vendor-specific configuration dwords at `0x484`–`0x48C` is `[L]` — inferred from the shared
register blocks and the BT-notification step, not directly established. Consequences for the
host:

* A Wi-Fi FLR must be preceded by the BT notification write to `0x484` `[C]`.
* Resetting the Wi-Fi subsystem via CB-TOP (§7.4) affects only the WF path
  (`WF_SUBSYS_RST`), not the BT path `[C]`.
* No PCIe-level bridge/tunnel is interposed by the device itself; the function is a normal
  PCIe endpoint. `[L]`

---

## 2. DMA addressing

| Property | Value |
|---|---|
| DMA address width | **32 bits.** The hardware-configuration record declares a DMA mask width of 32 and the host sets the PCI DMA mask to `0xFFFF_FFFF` `[C]` |
| High-address extension | **Not used.** `[C]` |
| Descriptor address fields | 32-bit physical addresses only `[C]` |

Notes:

* The WFDMA global-configuration register has a `PDMA_ADDR_EXT_EN` bit (`WPDMA_GLO_CFG`
  bit 26) which, when set, would change the TX/RX descriptor format to carry extended
  addresses. In the CONNAC2 register documentation for this generation this bit is
  annotated "no function"; MT7932 is operated with a flat 32-bit DMA mask. `[L]`
* The per-ring `..._EXT_CTRL` registers (`WFDMA host DMA0 + 0x600` for TX,
  `+ 0x680` for RX) are **prefetch descriptors, not address extension**: the written value
  is `(prefetch_base << 16) | prefetch_depth`, with `prefetch_base` in bits **[31:16]** and
  `depth` in bits **[7:0]**. Prefetch base advances by `depth * 0x10` per ring. `[C]`
  Do not mistake these for high-address fields. (The WFDMA section §4.1/§5 is the authority.)
* Ring memory and buffers must be allocated as **DMA-coherent** (uncached / hardware
  coherent) memory. Ring base pointers are written as raw 32-bit physical addresses to the
  ring `CTRL0` registers. `[C]`
* Descriptor entries are 16 bytes; ring bases should be at least 16-byte aligned, and in
  practice page-aligned coherent allocations are used. `[L]`
* Matches public MT7921 exactly (upstream `mt76` also uses `DMA_BIT_MASK(32)` for
  mt792x/mt7921). Note that MT7925/MT7927 use a 34-bit mask; MT7932 does **not**. `[C]`

---

## 3. Register access rules

### 3.1 Width, endianness, alignment

* **All MMIO register accesses are 32-bit**, naturally aligned, little-endian. `[C]`
  Byte and half-word accesses to the register aperture are never performed and must be
  assumed illegal. `[C]`
* Configuration-space accesses are 32-bit dword reads/writes, except COMMAND/Device
  Control/Device Status which are 16-bit. `[C]`

### 3.2 Address translation (host address → BAR offset)

Given a 32-bit *chip* address `A`, the BAR-0 offset is computed as:

1. If `0x0000_2000 < A < 0x0010_0000` → the BAR offset is `A` itself (direct BAR-offset
   addressing). `[C]`
2. Otherwise, search the fixed map of §4 in table order for the first entry with
   `chip_base <= A <= chip_base + size` and return `bar_offset + (A - chip_base)`. `[C]`
   (Note the *inclusive* upper bound — the same off-by-one as upstream `mt76`'s
   `if (ofs > size) continue`. An address exactly one byte past a window's end resolves
   into that window rather than the next entry.)
3. If no entry matches, the address is **not reachable** and the access must be rejected;
   the caller has to install a remap window first (§5). `[C]`

The map is searched linearly and **the first match wins**, so duplicate chip bases later in
the table are unreachable (see the `0x820C_C000` duplicate in §4). `[C]`

### 3.3 Failure sentinels

| Read value | Meaning |
|---|---|
| `0xFFFF_FFFF` | Bus access failure (link down, function unreachable) — **unless** the address is on the read-`0xFFFFFFFF`-is-legal whitelist (§3.4) `[C]` |
| `0xDEAD_FEED` | Bus-error sentinel; the host treats it as "target did not respond" `[C]`. That it specifically means the block is powered down, clock-gated or behind an unprogrammed remap is `[L]` |
| `0xDEAD_0001` | Host-bus timeout indication (CONNAC2 family convention) `[L]` |

Liveness probe: read chip address **`0x8002_1010`** (BAR offset `0xB1010`). If that read
*also* returns `0xDEAD_FEED`, the chip is dead. `[C]` The register's identity is **not**
established here: the name `CONN_CFG_CHIP_ID` comes from a public gen4m macro for
`top_cfg_base + 0x1010`, and gen4m's own CONNAC1 map names the same offset `STRAP_STA`.
`[U]` — all that is confirmed is that it is read as a liveness canary.

### 3.4 Whitelist of registers that may legitimately read as all-ones

A read returning `0xFFFF_FFFF` is treated as a bus failure **except** at these chip
addresses, where all-ones is a valid data value: `[C]`

| Address(es) | Block |
|---|---|
| `0x820C_0600`, `0x820C_0604` | WF_UMAC_TOP (PLE) — queue/group empty status |
| `0x820C_0680`, `0x820C_0684` | WF_UMAC_TOP (PLE) |
| `0x820C_0700`, `0x820C_0704` | WF_UMAC_TOP (PLE) |
| `0x820C_0780`, `0x820C_0784` | WF_UMAC_TOP (PLE) |
| any address with bits [31:16] == `0x820D` | `WF_LMAC_TOP (WF_WTBLON)` — station table window, `0x820D_0000 … 0x820D_FFFF` |

This is effectively a statement about which regions can legitimately contain all-ones data;
it is *not* a general "safe to touch" list.

### 3.5 Ordering and read-back requirements

* After programming a **remap window base** (§5), the host must **read the same register
  back** and wait ≥ 2 µs before using the newly-mapped BAR window. The read-back also
  serves as the write-push barrier. `[C]`
* After clearing the WFDMA host interrupt-enable register, the host reads it back to force
  the write out before proceeding. `[C]`
* Descriptor memory must be flushed/ordered before the ring CPU-index register write; the
  CPU-index write is the doorbell. `[L]`

### 3.6 Accesses that are illegal or unreliable in particular states

MMIO register access **must not** be performed when any of the following is true `[C]`:

1. **Firmware owns the chip** (FW-own asserted, see §7.3). The host must first assert
   driver-own via `CONN_ON_LPCTL` and observe the own-sync bit clear.
2. **An FLR or L0/L0.5 reset is in progress.**
3. **After the suspend sequence has completed** (device parked in low power).
4. **After a bus-access failure has been latched** — further MMIO is meaningless until
   recovery.
5. Debug-only MMIO reads should be avoided while the datapath is running: stray reads in
   that state can perturb the part's power management. Suppressing them outside
   diagnostics is a host policy, not a hardware rule. `[L]`

Configuration-space reads remain valid in more states than MMIO (they are used precisely to
determine whether MMIO is usable), except after the suspend sequence completes. `[C]`

---

## 4. Fixed BAR-window → chip-address map

The 1 MiB BAR-0 aperture decodes to chip addresses according to the following table. It has
**exactly 50 entries**; there is **no all-zero terminator entry** — the map is
length-counted, and the word immediately following the last entry is unrelated data. The
last entry is `0x7C00_0000 → BAR 0x0F0000`. `[C]`

Entries are listed in the hardware-configuration record's own order (which is also the
linear-search order; the first three entries are the hottest lookups).

| # | Chip base | BAR offset | Size | Functional block |
|---:|---|---|---|---|
| 0 | `0x7C02_0000` | `0x0D0000` | `0x10000` | CONN_INFRA — host-side WFDMA CSR aperture (host WPDMA0 @ `0x7C02_4000`, host WPDMA1 @ `0x7C02_5000`, DMASHDL @ `0x7C02_6000`, WFDMA ext-wrap CSR @ `0x7C02_7000`) |
| 1 | `0x7403_0000` | `0x010000` | `0x10000` | PCIe MAC internal registers (`PCIE_MAC_IREG`) |
| 2 | `0x7C06_0000` | `0x0E0000` | `0x10000` | CONN_INFRA `conn_host_csr_top` (ownership, host CSR) |
| 3 | `0x5400_0000` | `0x002000` | `0x1000` | WFDMA PCIE0 MCU DMA0 |
| 4 | `0x5500_0000` | `0x003000` | `0x1000` | WFDMA PCIE0 MCU DMA1 |
| 5 | `0x5600_0000` | `0x004000` | `0x1000` | WFDMA reserved |
| 6 | `0x5700_0000` | `0x005000` | `0x1000` | WFDMA MCU wrap CR |
| 7 | `0x5800_0000` | `0x006000` | `0x1000` | WFDMA PCIE1 MCU DMA0 (MEM_DMA) |
| 8 | `0x5900_0000` | `0x007000` | `0x1000` | WFDMA PCIE1 MCU DMA1 |
| 9 | `0x820C_0000` | `0x008000` | `0x4000` | WF_UMAC_TOP (PLE) |
| 10 | `0x820C_8000` | `0x00C000` | `0x2000` | WF_UMAC_TOP (PSE) |
| 11 | `0x820C_C000` | `0x00E000` | `0x2000` | WF_UMAC_TOP (PP) **and** WF_MDP_TOP (`0x820C_D000`) |
| 12 | `0x820E_0000` | `0x020000` | `0x0400` | WF_LMAC_TOP BN0 — WF_CFG |
| 13 | `0x820E_1000` | `0x020400` | `0x0200` | WF_LMAC_TOP BN0 — WF_TRB |
| 14 | `0x820E_2000` | `0x020800` | `0x0400` | WF_LMAC_TOP BN0 — WF_AGG |
| 15 | `0x820E_3000` | `0x020C00` | `0x0400` | WF_LMAC_TOP BN0 — WF_ARB |
| 16 | `0x820E_4000` | `0x021000` | `0x0400` | WF_LMAC_TOP BN0 — WF_TMAC |
| 17 | `0x820E_5000` | `0x021400` | `0x0800` | WF_LMAC_TOP BN0 — WF_RMAC |
| 18 | `0x820C_E000` | `0x021C00` | `0x0200` | WF_LMAC_TOP — WF_SEC |
| 19 | `0x820E_7000` | `0x021E00` | `0x0200` | WF_LMAC_TOP BN0 — WF_DMA |
| 20 | `0x820C_F000` | `0x022000` | `0x1000` | WF_LMAC_TOP — WF_PF |
| 21 | `0x820E_9000` | `0x023400` | `0x0200` | WF_LMAC_TOP BN0 — WF_WTBLOFF |
| 22 | `0x820E_A000` | `0x024000` | `0x0200` | WF_LMAC_TOP BN0 — WF_ETBF |
| 23 | `0x820E_B000` | `0x024200` | `0x0400` | WF_LMAC_TOP BN0 — WF_LPON |
| 24 | `0x820E_C000` | `0x024600` | `0x0200` | WF_LMAC_TOP BN0 — WF_INT |
| 25 | `0x820E_D000` | `0x024800` | `0x0800` | WF_LMAC_TOP BN0 — WF_MIB |
| 26 | `0x820C_A000` | `0x026000` | `0x2000` | WF_LMAC_TOP BN0 — WF_MUCOP |
| 27 | `0x820D_0000` | `0x030000` | `0x10000` | WF_LMAC_TOP — WF_WTBLON (station table) |
| 28 | `0x8800_0000` | `0x040000` | `0x10000` | **WF_MCU_CFG_LS** (holds `TOP_FVR` at `0x8800_0004`, WFSYS bus-status CRs at `0x8800_0430/0444/044C/0450`). Remappable slot — see §5 |
| 29 | `0x7000_0000` | `0x070000` | `0x10000` | **CB-TOP / CONN2AP** (CB-TOP RGU at `0x7000_2xxx`, MTCMOS control at `0x7000_3020`). Covers only `0x7000_0000`–`0x7000_FFFF`; the CB-TOP chip-ID/HW-version block at `0x7001_02xx` is **outside** it and needs a remap. Remappable slot — see §5 |
| 30 | `0x0040_0000` | `0x080000` | `0x10000` | WF_MCU_SYSRAM |
| 31 | `0x7C05_0000` | `0x090000` | `0x10000` | **CONN_INFRA SYSRAM** |
| 32 | `0x820F_0000` | `0x0A0000` | `0x0400` | WF_LMAC_TOP BN1 — WF_CFG |
| 33 | `0x820F_1000` | `0x0A0600` | `0x0200` | WF_LMAC_TOP BN1 — WF_TRB |
| 34 | `0x820F_2000` | `0x0A0800` | `0x0400` | WF_LMAC_TOP BN1 — WF_AGG |
| 35 | `0x820F_3000` | `0x0A0C00` | `0x0400` | WF_LMAC_TOP BN1 — WF_ARB |
| 36 | `0x820F_4000` | `0x0A1000` | `0x0400` | WF_LMAC_TOP BN1 — WF_TMAC |
| 37 | `0x820F_5000` | `0x0A1400` | `0x0800` | WF_LMAC_TOP BN1 — WF_RMAC |
| 38 | `0x820F_7000` | `0x0A1E00` | `0x0200` | WF_LMAC_TOP BN1 — WF_DMA |
| 39 | `0x820F_9000` | `0x0A3400` | `0x0200` | WF_LMAC_TOP BN1 — WF_WTBLOFF |
| 40 | `0x820F_A000` | `0x0A4000` | `0x0200` | WF_LMAC_TOP BN1 — WF_ETBF |
| 41 | `0x820F_B000` | `0x0A4200` | `0x0400` | WF_LMAC_TOP BN1 — WF_LPON |
| 42 | `0x820F_C000` | `0x0A4600` | `0x0200` | WF_LMAC_TOP BN1 — WF_INT |
| 43 | `0x820F_D000` | `0x0A4800` | `0x0800` | WF_LMAC_TOP BN1 — WF_MIB |
| 44 | `0x820C_C000` | `0x0A5000` | `0x2000` | **Second aperture of WF_UMAC_TOP (PP)/WF_MDP_TOP.** Unreachable by chip-address lookup because entry 11 matches first; reachable by direct BAR offset `0xA5000` |
| 45 | `0x820C_4000` | `0x0A8000` | `0x4000` | WF_LMAC_TOP BN1 — WF_MUCOP / UWTBL (unified WTBL base `0x820C_4094`) |
| 46 | `0x820B_0000` | `0x0AE000` | `0x1000` | [APB2] WFSYS_ON |
| 47 | `0x8002_0000` | `0x0B0000` | `0x10000` | WF_TOP_MISC_OFF (`TOP_CFG` base; chip-ID/HW-version/chip-ID-mirror registers) |
| 48 | `0x8102_0000` | `0x0C0000` | `0x10000` | WF_TOP_MISC_ON |
| 49 | `0x7C00_0000` | `0x0F0000` | `0x10000` | CONN_INFRA off-domain (WFSYS reset/init-done, own-IRQ status, PCIe2AP remap array) |

Unmapped BAR ranges: `0x00000–0x01FFF`, `0x00F000–0x00FFFF` (folded into entry 11),
`0x010000` is entry 1, `0x050000–0x06FFFF`, `0x0A6000–0x0A7FFF`, `0x0AF000–0x0AFFFF`,
`0x0D0000` region is entry 0. `[C]`

### 4.1 Deltas versus public parts

| Item | Public MT7921 (`mt76` fixed_map / gen4m `mt7961`) | MT7932 / MT7922 |
|---|---|---|
| `0x40000` slot | not mapped | `0x8800_0000` (WF_MCU_CFG_LS) — **new** `[C]` |
| `0x70000` slot | `0x4000_0000` (WF_UMAC_SYSRAM) in upstream `mt76`; absent in gen4m `mt7961` | `0x7000_0000` (CB-TOP/CONN2AP) — **changed** `[C]` |
| `0x90000` slot | `0x0041_0000` (WF_MCU_SYSRAM configure region) | `0x7C05_0000` (CONN_INFRA SYSRAM) — **changed** `[C]` |
| `0xA5000` slot | not mapped | second `0x820C_C000` aperture — **new** `[C]` |
| PP window size | `0x1000` (upstream `mt76`), plus a separate `0x820C_D000 → 0x0F000` WF_MDP_TOP entry | single `0x2000` window covering both `[C]` |
| Entry count | 44 (upstream `mt76`), 47 + terminator (gen4m `mt7961`) | 50, length-counted `[C]` |
| Everything else | — | **identical** to gen4m `mt7961` / upstream MT7921, including all LMAC BN0/BN1, PLE/PSE, WTBLON, UWTBL, WFSYS_ON, TOP_MISC_ON/OFF, conn-infra and PCIe-MAC entries `[C]` |

MT7932 and MT7922 share this table bit-for-bit. `[C]`

---

## 5. Register remapping windows

### 5.1 Mechanism

Each 64 KiB BAR slot's chip base is programmed by a **16-bit "PCIe2AP public remapping"
field**, two fields packed per 32-bit register, in the conn-infra bus-control block. The field
value is the target base address **shifted right by 16** (i.e. the top 16 bits of the target
address). `[C]` — established for the three registers the host writes (§5.3). That the array
covers all **sixteen** 64 KiB slots of the aperture in the same packing is `[L]`, inferred from
the register spacing and the CONNAC3 naming; only slots 4–9 are directly established (OQ 3).

The remap targets are expressed on the **conn-infra/AP internal bus**, which for the
conn-infra region is the *same* address space as the host-view `0x7Cxx_xxxx` aliases with a
fixed `-0x6400_0000` offset:

```
host-view 0x7C0n_xxxx  ==  AP-bus 0x180n_xxxx
```

so `0x7C00_0140` and `0x1800_0140` are the same register (this is the same duality upstream
`mt76` encodes as `is_connac2() ? 0x18000140 : 0x7c000140`). Targets outside conn-infra
(CB-TOP `0x7000_xxxx`, PCIe MAC `0x7403_xxxx`) are used unmodified. `[C]` for the selector
*values* written; the `-0x6400_0000` host-view/AP-bus aliasing itself is `[L]`, inherited from
public `mt76` and consistent with every value observed here.

### 5.2 Remap control registers

Base of the field array: **host-view `0x7C00_E244`** (= AP-bus `0x1800_E244`), reachable
statically at BAR offset `0xFE244`. `[L]` — anchored by three independently confirmed
fields; the two lowest registers of the array are not written during bring-up.

| Register (chip / host view) | Field | BAR slot controlled | Value | Target |
|---|---|---|---|---|
| `0x7C00_E244` | [15:0] | `0x00000`–`0x0FFFF` | — | composite WFDMA/UMAC decode `[U]` |
| `0x7C00_E244` | [31:16] | `0x10000`–`0x1FFFF` | — | `0x7403` → PCIe MAC IREG `[L]` |
| `0x7C00_E248` | [15:0] | `0x20000`–`0x2FFFF` | — | composite LMAC BN0 decode `[U]` |
| `0x7C00_E248` | [31:16] | `0x30000`–`0x3FFFF` | — | `0x820D` → WF_WTBLON `[L]` |
| **`0x7C00_E24C`** | **[15:0]** | **`0x40000`–`0x4FFFF`** | default **`0x184F`**; host temporarily sets `0x7000`, `0x7001`, `0x1807` | `[C]` |
| `0x7C00_E24C` | [31:16] | `0x50000`–`0x5FFFF` | `0x1845` (always written back unchanged) | `[C]` |
| **`0x7C00_E250`** | [15:0] | `0x60000`–`0x6FFFF` | `0x1846` (written back unchanged) | `[C]` |
| **`0x7C00_E250`** | **[31:16]** | **`0x70000`–`0x7FFFF`** | host **must program `0x7000`** | CB-TOP / CONN2AP `[C]` |
| **`0x7C00_E254`** | [15:0] | `0x80000`–`0x8FFFF` | `0x1848` | WF_MCU_SYSRAM (`0x0040_0000`) `[C]` |
| **`0x7C00_E254`** | **[31:16]** | **`0x90000`–`0x9FFFF`** | host **must program `0x1805`** | CONN_INFRA SYSRAM (host view `0x7C05_0000`) `[C]` |
| `0x7C00_E258` | [15:0]/[31:16] | `0xA0000` / `0xB0000` | — | composite LMAC BN1 / `0x8002` `[L]` |
| `0x7C00_E25C` | [15:0]/[31:16] | `0xC0000` / `0xD0000` | — | `0x8102` / `0x1802` `[L]` |
| `0x7C00_E260` | [15:0]/[31:16] | `0xE0000` / `0xF0000` | — | `0x1806` / `0x1800` `[L]` |

The three registers that the host actually writes are `0x7C00_E24C`, `0x7C00_E250` and
`0x7C00_E254`; all three are written as full 32-bit words, so the "other half" must be
written back with its correct default value.

Named equivalence: this is the CONNAC2 instance of the block that MediaTek's public
CONNAC3 headers call
`CONN_BUS_CR_VON_CONN_INFRA_PCIE2AP_REMAP_WF_0_xy_cr_pcie2ap_public_remapping_wf_nn`
(`WF_0_10`, `WF_0_32`, `WF_0_54`, `WF_0_76`, `WF_0_98`, …). `0x7C00_E24C` is the `WF_0_54`
register — i.e. its low half is public remapping field **4**, matching the MT7927 community
patch's `PCIE2AP_REMAP_WF_0_54` (BAR offset `0x21008` there) whose documented value
`0x1807` selects the conn-infra semaphore block for the BAR window at `0x40000` — exactly
the value and window MT7932 uses. `[C]`

### 5.3 Confirmed window values and their uses

| Value in slot-4 field (`0x7C00_E24C[15:0]`) | Window `0x40000` then reaches | Used for |
|---|---|---|
| `0x184F` (**default — must be restored**) | WF_MCU_CFG_LS `0x8800_0000` | `TOP_FVR` at BAR `0x40004`; WFSYS bus-status CRs at BAR `0x40430/0444/044C/0450` `[C]` |
| `0x7000` | CB-TOP `0x7000_0000` | CB-TOP RGU WF subsystem reset at BAR `0x42600`; MTCMOS power control at BAR `0x43020`; CB-TOP debug CRs at BAR `0x40100/0108/0148/2010/2014/2030/300C/3014/3100/320C` `[C]` |
| `0x7001` | CB-TOP `0x7001_0000` | `TOP_HCR` at BAR `0x40200`; further CB-TOP CRs at BAR `0x43008/3018/3090/3100/3120/3124/360C` `[C]` |
| `0x1807` | conn-infra `0x7C07_0000` (semaphore block) | Firmware-download security-protect handshake: write BAR `0x40260` = 1, poll BAR `0x40060` bit 0 `[C]` |

### 5.4 Safe usage sequence

To reach a chip address `X` that has no static map entry:

1. Choose a remappable slot (in practice slot 4, BAR `0x40000`).
2. Read-modify-write the controlling register so that the slot's 16-bit field becomes
   `X >> 16` and the sibling field keeps its current value. (The host writes the whole
   32-bit word with hard-coded sibling defaults; a read-modify-write is safer.) `[C]`
3. Wait **≥ 2 µs**. `[C]`
4. Read the control register back and verify the field took the intended value; a mismatch
   means the conn-infra bus is not responding and the access must be abandoned. `[C]`
5. Access BAR offset `0x40000 + (X & 0xFFFF)` with 32-bit accesses.
6. **Restore the default value** (`0x1845_184F` for `0x7C00_E24C`) when done. Leaving slot 4
   remapped breaks all subsequent chip-address accesses in the `0x8800_0000` range. `[C]`

### 5.5 Required one-time programming at init

Two remap writes are part of bring-up and must be done before the corresponding static-map
entries (§4 rows 29 and 31) are usable:

| Step | Write | Verify |
|---|---|---|
| "MMIO mapping set" | `0x7C00_E250 = 0x7000_1846` | after ≥ 2 µs, read back and require bits [31:16] == `0x7000`. Failure = fatal `[C]` |
| "conn-infra / MCU mapping set" | `0x7C00_E254 = 0x1805_1848` | after ≥ 2 µs, read back and require bits [31:16] == `0x1805`. Failure = fatal `[C]` |

Both are absent from public MT7921 support (gen4m `mt7961` declares no remap descriptor at
all and upstream `mt76/mt7921` uses a different `MT_HIF_REMAP_L1`-style mechanism inherited
from MT7915). **This is a real MT7922/MT7932 delta.** `[C]`

### 5.6 "ap2wf" vs "pcie2ap"

* The mechanism above is the **pcie2ap** remap (PCIe BAR slot → AP/conn-infra bus).
  Base `0x7C00_E244` (host view) / `0x1800_E244` (AP bus), sixteen 16-bit fields,
  **64 KiB granularity**, appearing across the whole 1 MiB BAR aperture. `[C]`
* No separate **ap2wf** remap register is programmed by the host on MT7932. WF-subsystem
  addresses (`0x0040_0000`, `0x8002_0000`, `0x8102_0000`, `0x820x_xxxx`, `0x8800_0000`) are
  reached through fixed pcie2ap slot values that are assumed to point at the conn-infra "WF"
  apertures (`0x1848`, `0x184F`, …) `[L]`. On CONNAC3 parts the corresponding block is
  `CONN_MCU_BUS_CR_AP2WF_REMAP_1` at `0x830C_0120`; there is no evidence of the host
  touching an equivalent on MT7932. `[C]` (absence), `[U]` (whether one exists)

### 5.7 Region validity

There is no general per-address whitelist gate on register writes for this part. The only
address-based filtering is:

* the address-translation rule of §3.2 (unmapped ⇒ rejected), and
* the all-ones whitelist of §3.4.

The regions upstream `mt76` considers legal for its L1 remap on MT7921 —
`0x1800_0000–0x18BF_FFFF`, `0x7000_0000–0x77FF_FFFF`, `0x7C00_0000–0x7C3F_FFFF` — are a good
guide to what is addressable on MT7932's AP bus, and all MT7932 remap target values in use
(`0x1805`, `0x1807`, `0x1845`, `0x1846`, `0x1848`, `0x184F`, `0x7000`, `0x7001`) fall inside
them. `[L]`

---

## 6. Chip identity, hardware/ROM/factory version

### 6.1 Registers

| Chip address | BAR offset (default mapping) | Name | Contents |
|---|---|---|---|
| `0x8002_1000` | `0xB1000` | `TOP_HW_VERSION` | Hardware version; low 4 bits are the revision ID `[C]` |
| `0x8002_1008` | `0xB1008` | `TOP_HW_CONTROL` (`WCIR`) | bits **[15:0] = chip ID** (expect `0x7932`). The revision nibble is **not** read from this register — see `0x8002_1000` above `[C]` |
| `0x8002_1010` | `0xB1010` | *(identity not established)* | Used as the bus-liveness probe (§3.3) `[C]`; that it is a chip-ID mirror is `[U]` — the name is public-macro provenance only |
| `0x7001_0200` | via remap slot 7 or slot 4 = `0x7001` | `TOP_HCR` (public `MT_HW_CHIPID`) | Chip ID `[C]` |
| `0x7001_0204` | as above | `TOP_HVR` (public `MT_HW_REV`) | **bits [7:0] = HW version, bits [11:8] = factory (ECO) version** `[C]` |
| `0x8800_0004` | `0x40004` (slot 4 default `0x184F`) | `TOP_FVR` | **bits [7:0] = ROM/SW version** `[C]` |
| `0x7001_0020` | via remap | `MT_HW_BOUND` — A-die selector on public MT7921 (`BIT(7)`: 0 = MT7921 A-die, 1 = MT7920 A-die) | **not used on MT7932**, see §6.4 `[C]` |

`TOP_CFG_BASE` for this part is **`0x8002_0000`** (WF_TOP_MISC_OFF), identical to public
MT7921/MT7922. `[C]`

### 6.2 Field extraction

| Quantity | Source register | Field | Note |
|---|---|---|---|
| `chip_id` | `0x8002_1008` | bits [15:0] | expect `0x7932` |
| `revision_id` | `0x8002_1000` | bits [3:0] | read immediately after a successful chip-ID compare; **not** `0x8002_1008[19:16]`. Because chip-ID verification is disabled on this part (§6.4), this read **never happens at run time** on MT7932 — the addresses and field widths are `[C]`, but the path is dead |
| `hw_ver` | `0x7001_0204` | bits [7:0] | |
| `factory_ver` | `0x7001_0204` | bits [11:8] | |
| `rom_ver` | `0x8800_0004` | bits [7:0] | |

`[C]`

Note: `TOP_HVR` (`0x7001_0204`) is **outside** the statically mapped `0x7000_0000` window
(which covers only `0x7000_0000 … 0x7001_0000`). It is read either through remap slot 4
programmed to `0x7001`, or — as the host does — through the boot-ROM "access register"
command, *before* any firmware image is downloaded, rather than by MMIO. `[C]`

### 6.3 Version (ECO) table

The ECO lookup matches the triple `{hw_ver, rom_ver, factory_ver}` against:

| hw_ver | rom_ver | factory_ver | ECO version |
|---|---|---|---|
| `0x00` | `0x00` | `0x0A` | **E1** |
| `0x01` | `0x01` | `0x0A` | **E2** |
| — | — | — | end of table (all-zero) |

If no entry matches, the *previous* entry (the highest known revision below the match
point) is used. `[C]`

**Delta:** the public MT7921 (gen4m `mt7961`) table is `{0x00,0x00,0x0A}→E1`,
`{0x10,0x01,0x0A}→E2`. MT7932/MT7922 use `hw_ver = 0x01` for E2, not `0x10`. `[C]`
MT7932 and MT7922 share the same ECO table — no MT7932-specific revision entries exist. `[C]`

### 6.4 Chip-ID verification and A-die

* The chip-ID verification step against `TOP_HW_CONTROL` is **disabled** for MT7932/MT7922
  (the hardware-configuration record clears the "verify chip id" flag). A driver may still perform
  it; the expected value is `0x7932` in bits [15:0]. `[C]`
* **A-die (companion RF die) version is not read from a register on MT7932.** It is
  reported by firmware as a 16-bit value in the chip-capability event and cached by the
  host. `[C]` The public MT7921 register-based method (`0x7001_0020` bit 7 selecting
  MT7921 vs. MT7920 A-die, with flavor codes `0x01`/`0x1A`) is not used. `[C]`
* Bluetooth firmware version register on public MT7921 is `0x7C81_2004[7:0]`; not used on
  MT7932. `[U]`

---

## 7. PCIe-MAC and host-CSR registers the host must program

### 7.1 PCIe MAC block (`0x7403_0000`, BAR `0x010000`)

| Chip address | BAR offset | Name / role | Programming |
|---|---|---|---|
| `0x7403_0188` | `0x10188` | **PCIe MAC interrupt enable** (public `MT_PCIE_MAC_INT_ENABLE`) | Write **`0x0000_01FF`** to enable; write `0` to disable. `[C]` **Delta:** upstream MT7921 writes `0xFF` (8 sources); MT7932/MT7922 enable 9 bits. `[C]` |
| `0x7403_0184` | `0x10184` | **PCIe MAC interrupt status** | Read once, as BAR offset `0x10184`, in the bus-failure diagnostic dump below `[C]`; **never read as part of interrupt service** `[C]`. The "interrupt status" role is public gen4m sibling-chip naming only — `[U]` for MT7932 |
| `0x7403_018C` | `0x1018C` | **PCIe MAC interrupt clear** | Write the bitmap of serviced sources to acknowledge. Write-1-to-clear semantics. `[C]` address, `[L]` semantics |
| `0x7403_0194` | `0x10194` | PCIe MAC power management (public `MT_PCIE_MAC_PM`) | `BIT(8)` = L0s disable `[L]` (public MT7921/MT7925 set this; it is not written on MT7932) `[U]` |
| `0x7403_1484` | `0x11484` | **Host→device doorbell.** `[L]` that this is the MMIO alias of PCI configuration dword `0x484` (alias base `0x7403_1000 + cfg_offset`, with the Bluetooth function's configuration space at `0x7403_9000 + cfg_offset`) — the address arithmetic and the shared bit meanings imply it, but the aliasing was not tested | One bit is written at a time: `0x8000_0000` = host-forced firmware assert/coredump; `0x0000_2000` = host entering hibernate; `0x0000_0800` = host is collecting an assert dump; `0x0000_0001` (written through a configuration cycle at `0x484`) = set-firmware-own / notify Bluetooth before a Wi-Fi FLR. `[C]` Remaining bits `[U]` |
| `0x7403_0168` | `0x10168` | PCIe MAC debug **group/mode select** | **Written** (`0xCCCC_0100`, `0x9999_0100`) `[C]` |
| `0x7403_0164` | `0x10164` | PCIe MAC debug **signal select** | **Written** with a 4×8-bit signal selector (ten values are swept) `[C]` |
| `0x7403_002C` | `0x1002C` | PCIe MAC debug **status read-back** | **Read** after each selector write `[C]` |

> **Correction.** An earlier version of this table had these three rows' directions wrong —
> `0x0164` as the only selector, `0x0168` as read-only data, and `0x002C` as a register
> written with `0`. The sequence is: write the group/mode selector to `0x7403_0168`, write the
> signal selector to `0x7403_0164`, then **read** `0x7403_002C`. The power/reset section §6.1
> gives the same sequence, and the DMA-hang predicate of §6.6 there depends on `0x7403_002C`
> being the read port. `[C]`

Diagnostic register set read wholesale on error (all in the PCIe MAC block, BAR-offset
form): `0x10150, 0x10154, 0x101D8, 0x101E0, 0x11204, 0x19204, 0x11210, 0x1121C, 0x11220,
0x11224, 0x11228, 0x19210, 0x1921C, 0x19220, 0x19224, 0x19228, 0x11480, 0x103C8, 0x103CC,
0x1818C, 0x10E00, 0x10E04, 0x10E08, 0x10184, 0x101D4`. `[C]`

> **Correction.** An earlier note claimed BAR `0x1818C` lay in the LMAC BN0 window. It does
> not: entry 1 of the §4 map covers BAR `0x010000`–`0x01FFFF`, so `0x1818C` is chip
> `0x7403_818C`, **inside the PCIe MAC block** like every other offset in this list. The
> power/reset section §6.1 lists it in chip-address form as `0x7403_818C`. `[C]`

Probe selector values used for the PCIe MAC debug read-out: `0x4F4E4D4C`, `0x57565553`,
`0x26222120`, `0x5B5A5958`, `0x5F5E5D5C`, `0x27252423`, `0xB7B6B5B4`, `0xB7B6B098`,
`0xBBBAB9B8`, `0xBFBEBDBC`. `[C]`

### 7.2 WFDMA host interrupt registers (host WPDMA0 at `0x7C02_4000`)

Included here because they are the MSI back-end.

| Chip address | Name | Notes |
|---|---|---|
| `0x7C02_4200` | Host interrupt status (`HOST_INT_STA`) | **Write-1-to-clear** `[C]` |
| `0x7C02_4204` | Host interrupt enable (`HOST_INT_ENA`) | `[C]` |
| `0x7C02_4118` | Host interrupt status EXT | `[L]` |
| `0x7C02_41F0` / `0x7C02_41F4` | MCU→host software interrupt status / mask (WPDMA0) | `[C]` |
| `0x7C02_51F0` / `0x7C02_51F4` | same, WPDMA1 instance (not populated on this part) | `[C]` |

Bit assignment. Bits **0–3, 4–18, 19, 22, 23, 25, 26, 27, 29, 30, 31** are this part's own
ring→bit assignment and are `[C]`; bits **11, 20, 21, 28** carry only the public CONNAC2
definition, which is assumed to hold here — `[L]`:

| Bit | Meaning |
|---|---|
| 0–3 | RX ring 0–3 done |
| 4–10 | TX ring 0–6 done |
| 11 | TX ring 7 done in the public map; **no interrupt bit is assigned to TX ring 7 on this part** |
| 12–18 | TX ring 8–14 done |
| 19 | **RX ring 6 done** (MT7932/MT7922 placement, not in any public header) |
| 20 | RX coherent error |
| 21 | TX coherent error |
| 22, 23 | RX ring 4, 5 done |
| 25 | **RX ring 7 done** (MT7932/MT7922 placement) |
| 26, 27 | TX ring 16, 17 done |
| 28 | sub-system error / reset request; never enabled here |
| 29 | MCU→host software interrupt |
| 30 | TX ring 18 done |
| 31 | **RX ring 8 done** (MT7932/MT7922 placement) |

The interrupt section (§4 of this specification) carries the full bit map, the per-ring
tables and the aggregate masks and is the authority for them.

Values used by MT7932/MT7922:

| Purpose | Value |
|---|---|
| TX-done interrupt set | `0x4C00_0000` (TX rings 16, 17, 18) `[C]` — **delta**: gen4m `mt7961` uses `0x0C00_07F0` (TX rings 0–6, 16, 17) |
| RX-done interrupt set | `0x00C0_000D` (RX rings 0, 2, 3, 4, 5) `[C]` — identical to gen4m `mt7961` |
| Always-enabled base | `0x6C00_0000` (TX 16/17/18 + MCU sw int), or `0x6000_0000` while the driver is still in its **pre-firmware-ready (initialisation)** state `[C]` — see the interrupt section §5, which is the authority for the trigger |
| Per-ring bits OR'd in | masked with `0x93CF_FFFF` `[C]` |
| Enable order | write `0x7403_0188` first, then `0x7C02_4204`, then (if low-power own supported) `0x7C02_41F4 = 0xFFFF` `[C]` |
| Disable order | write `0x7C02_4204 = 0`, then `0x7403_0188 = 0` `[C]` |

Other host-side WFDMA/PCIe-glue registers the host writes at bring-up:

| Chip address | Value | Meaning |
|---|---|---|
| `0x7C02_4298` | `0x0000_000C` | `WPDMA_INT_RX_PRI_SEL` — priority-select for RX rings 2 and 3 `[C]`. Written **only when MSI is enabled**; the coalescing path later read-modify-writes it `[C]` |
| `0x7C02_7038` | `0x0000_0013` | WFDMA ext-wrap CSR (adjacent to the public `MT_WFDMA_HOST_CONFIG` at `0x7C02_7030`). Written **only when MSI is enabled** `[C]`; field meaning unverified `[U]` |
| `0x7C02_42F0` | `0x8032_800A` | `PRI_DLY_INT_CFG0` — delayed/coalesced interrupt config, written **only when MSI is enabled** `[C]`. Runtime form: `0x8032_0000 \| (band << 15) \| ((pkt_cnt & 0x7F) << 8) \| timeout` `[C]` |
| `0x7C02_42E8` | `0x01FD_0032` (enable) / `0x0` (disable) | RX per-ring delayed-interrupt config `[C]` |
| `0x7C02_4208` | RMW, clear `BIT(15)` | `WPDMA_GLO_CFG`: clear `CSR_DISP_BASE_PTR_CHAIN_EN` before programming manual prefetch; TX/RX DMA enable are bits 0 and 2 `[C]` |
| `0x7C02_420C` | `0xFFFF_FFFF` | `WPDMA_RST_DTX_PTR` — reset all TX descriptor pointers `[C]` |
| `0x5400_0120` | RMW, set `BIT(1)` | WFDMA dummy CR "needs re-init" flag (public `MT_WFDMA_DUMMY_CR` / `MT_WFDMA_NEED_REINIT`) `[C]` |
| `<dma_base> + 0x100` | `0` then `0x30` | WFDMA HIF reset (`WPDMA_HIF_RST`) `[C]` |

### 7.3 Ownership / low-power control (`conn_host_csr_top`, `0x7C06_0000`)

| Chip address | Name | Semantics |
|---|---|---|
| `0x7C06_0010` | `CONN_ON_LPCTL` (BN0 LPCR) | **Write `1`** = set firmware own; **write `2`** = clear own (take driver own); **read bit 2** = own-sync status — `0` means the host owns the chip `[C]`. Bit names: `PCIE_LPCR_HOST_SET_OWN` = `BIT(0)`, `PCIE_LPCR_HOST_CLR_OWN` = `BIT(1)`, `PCIE_LPCR_HOST_OWN_SYNC` = `BIT(2)` |
| `0x7C06_0014` | BN0 IRQ status | Address and clear bit `BIT(0)` are carried per-chip `[C]`; the register is **not accessed on MT7932** (the "check driver-own interrupt" flag is false), so its behaviour is `[L]` from public CONNAC2 |
| `0x7C06_0018` | BN0 IRQ enable | Written `1` in the diagnostic/ICAP mode `[C]` |
| `0x7C06_00F0` | `CONN_ON_MISC` | Firmware state; `BIT(0)` = FW power on, `BIT(1)` = FW N9 on. Used as the readiness gate on MT7922 but **not** on MT7932 (see §1.3) `[C]` |
| `0x7C06_0204` / `0x7C06_0208` | WFSYS CPU PC / low-power state | diagnostic `[C]` |

Own/driver-own must be asserted before any other MMIO access after a low-power episode.
`[C]`

### 7.4 Reset and power-sequencing registers

| Chip address (or BAR offset + required slot-4 remap) | Value | Meaning |
|---|---|---|
| `0x7C00_0140` (= AP-bus `0x1800_0140`) | `BIT(0)` = `WFSYS_SW_RST_B`, `BIT(4)` = `WFSYS_SW_INIT_DONE` | WF subsystem software reset / init-done. Init-done is polled with a 100 ms interval, 3 attempts `[C]` |
| `0x7C00_1620` | `0x0000_0003` | Own-IRQ status clear, followed by a 2 ms delay. **Only performed for device ID `0x7922`; skipped on MT7932/MT7923** `[C]` |
| BAR `0x42600` with slot 4 = `0x7000` (chip `0x7000_2600`) | `1` = assert, `0` = deassert | CB-TOP RGU **WF subsystem reset** (public `MT_CBTOP_RGU_WF_SUBSYS_RST`, `BIT(0)` = whole WF path). Sequence: set slot 4 to `0x7000`, wait 2 µs, write `1`, wait 50 ms, write `0`, restore slot 4 to `0x184F` `[C]` |
| BAR `0x43020` with slot 4 = `0x7000`, **or** chip `0x7000_3020` once slot 7 has been programmed | `0` | CB-TOP **MTCMOS hardware-mode** power control; followed by a 1 ms delay. **Only executed when the device ID is not `0x7922`** — i.e. this step is specific to MT7932/MT7923 `[C]` |
| BAR `0x40260` / `0x40060` with slot 4 = `0x1807` | write `1` / poll bit 0 | Firmware-download security-protect handshake in the conn-infra semaphore block `[C]` |

### 7.5 Conn-infra diagnostic registers

Reachable through the static map at BAR `0xE0000` (chip `0x7C06_0000`, AP-bus
`0x1806_0000`) and BAR `0xF0000` (chip `0x7C00_0000`, AP-bus `0x1800_0000`). The **addresses
and the read/write sequences** are `[C]`; the **Role** column below is a naming inference from
the access pattern and public CONNAC naming, `[L]`, except the `0x7C06_02D4` bit-0 test, which
is `[C]`:

| Host view | AP-bus alias | Role |
|---|---|---|
| `0x7C06_015C` / `0x7C06_02C8` | `0x1806_015C` / `0x1806_02C8` | conn-infra debug **select** / **status** pair (6 selector values swept) |
| `0x7C06_0138` / `0x7C06_0150` | `0x1806_0138` / `0x1806_0150` | conn-infra signal-status select / status pair (11 selector values swept) |
| `0x7C06_0294` | `0x1806_0294` | conn-infra strap pins |
| `0x7C06_0000` | `0x1806_0000` | conn-infra off-domain control (written `1` then read back) |
| `0x7C06_02D4` | `0x1806_02D4` | conn-infra off-domain bus alive: `BIT(0)` clear ⇒ **off-domain bus not responding** |
| `0x7C00_1000` | `0x1800_1000` | conn-infra off-domain register |
| `0x8800_0430`, `0x8800_0444`, `0x8800_044C`, `0x8800_0450` | — | WFSYS bus-status CRs (slot 4 at default `0x184F`) |

---

## Open questions / needs hardware tracing

1. **BAR-0 attributes.** 64-bit vs 32-bit BAR, prefetchable bit, and the exact decoded size
   (≥ 1 MiB) are not determinable from the programming interface; confirm with
   `lspci -vv` on real MT7932 hardware.
2. **BAR `0x00000–0x01FFF`.** Never accessed. Confirm whether it holds the `WF_MCU_BUS_CR` /
   ap2wf remap block as on CONNAC3 parts, and whether an ap2wf remap register exists on
   MT7932.
3. **Remap array extent.** Only the fields at `0x7C00_E24C`, `0x7C00_E250`, `0x7C00_E254`
   are established. The array base `0x7C00_E244` and the slots for BAR windows `0x00000`,
   `0x10000`, `0x20000`, `0x30000`, `0xA0000`–`0xF0000` are inferred. Read the whole range
   `0x7C00_E240 … 0x7C00_E264` on live hardware and compare against §4.
4. **Composite windows.** BAR slots `0x00000`, `0x20000` and `0xA0000` each decode to
   several non-contiguous chip bases (WFDMA/UMAC, LMAC BN0, LMAC BN1). Determine whether the
   compaction is fixed hardware behaviour in the conn-infra WF aperture or is selected by
   the corresponding remap field value.
5. **Slot values `0x1845` and `0x1846`** (BAR windows `0x50000` and `0x60000`) are written
   back but never used. Identify the blocks at AP-bus `0x1845_0000` and `0x1846_0000`.
6. **`0x7403_018C`** — confirm write-1-to-clear semantics and the bit↔MSI-vector mapping.
7. **`0x7403_1484` doorbell** — determine the field layout and the meaning of `0x8000_0000`.
8. **`0x7C02_7038 = 0x13`** — identify the fields of this WFDMA ext-wrap CSR.
9. **Config `0x488`** — obtain the full field definition; only 11 bits are validated.
10. **Config `0x48C[19:16]`** — enumerate the states other than `0` and `2`.
11. **Config `0xFC`** — identify (read for diagnostics only).
12. **PCIe MAC PM (`0x7403_0194`) / L0s / ASPM / LTR** — no ASPM, LTR, Max Payload Size or
    Max Read Request programming is performed on this part; confirm the device works correctly with
    OS-default settings, and whether `MT_PCIE_MAC_PM_L0S_DIS` needs setting as it does on
    MT7925.
13. **Multi-vector MSI has never been exercised.** The 8-vector layout of §1.4 is carried
    by the part but the observed configuration always allocates one vector. Confirm that
    requesting 8 vectors yields the per-source delivery the layout implies, and whether
    anything must be programmed on the device to achieve that routing.
14. **MSI vector count in practice** — the hardware descriptor allows 8; verify that the
    device actually advertises `Multiple Message Capable = 8` in its MSI capability.


---

# MT7932 — Power-on, Reset and Firmware/Driver Ownership

## Scope

This document specifies the host-visible power-up state, the driver/firmware ownership
("LP-own") handshake, every reset and error-recovery path, the PCIe link-power
interactions, and the bus-hang / DMA-hang / MCU-off detection facilities of the MediaTek
**MT7932** combo Wi-Fi/BT part as seen from its PCIe Wi-Fi function (vendor `0x14C3`,
device `0x7932`). The MT7932 presents a CONNAC2 (mt792x-class) PCIe host interface that is
register-compatible with MT7922; only the deltas are called out in detail. All addresses
below are **chip addresses** unless explicitly labelled "BAR offset" or "PCI config
offset". Public MediaTek register naming (`mt76` / `gen4m`) is used throughout.

Confidence markers: `[C]` confirmed, `[L]` likely, `[U]` unverified.

---

## 1. Address spaces used by the power/reset paths

The reset and ownership procedures touch three different address spaces. A driver must be
able to reach all three before it can bring the part up.

### 1.1 PCI configuration space

Vendor-specific configuration registers are used for *power-domain-independent* status.
They remain readable when the Wi-Fi subsystem core is powered down or the PCIe link has
just come out of a low-power state, which is why the MT7932 bring-up path prefers them over
MMIO. [C]

Addresses and the accesses made to them are `[C]`; the functional *names* are `[C]` where the
section referenced in the Notes column establishes them and `[L]` otherwise.

| Cfg offset | Name (functional) | Notes |
|---|---|---|
| `0x004` | PCI_COMMAND | dumped on bus failure |
| `0x010` / `0x014` | BAR0 low / high | dumped on bus failure |
| `0x0FC` | vendor status `[U]` | dumped on bus failure; contents never interpreted |
| `0x204` | AER Uncorrectable Error **Status** | read/cleared around FLR and AER events |
| `0x208` | AER Uncorrectable Error **Mask** | masked around FLR (see §5.4) |
| `0x210` | AER Correctable Error Status | read/cleared around FLR and AER events |
| `0x484` | **PCIe doorbell push** | host→chip doorbell, see §2.5 |
| `0x488` | **Fabric / power-domain status** | bus-hang detector, see §6.2 |
| `0x48C` | **Firmware boot-stage** (bits `[19:16]`) | readiness + MCU-off detector, see §2.4 / §6.3 |

The AER capability structure therefore sits at configuration offset `0x200`. [C]

Configuration offsets `0x484`, `0x488`, `0x48C` are believed to be aliased into MMIO at chip
address `0x7403_1000 + cfg_offset` (i.e. `0x7403_1484`, `0x7403_1488`, `0x7403_148C`), with
the second PCIe function (Bluetooth) aliased at `0x7403_9000 + cfg_offset`. **`[L]`** — the
aliasing is inferred from the address arithmetic and from the two forms being used for
related purposes; nothing reads or writes both forms of the same register, so it was not
demonstrated. What *is* `[C]`: the doorbell values `0x800`, `0x2000` and `0x8000_0000` are
written to chip `0x7403_1484` in the runtime paths, and value `1` is written to configuration
dword `0x484` in the reset path — a bit that the MMIO form is never given.

### 1.2 Static BAR window ("static map")

A chip address in the range **`0x0000_2000` < A < `0x0010_0000`** is used directly as a
BAR0 byte offset with no translation; the first 8 KiB is excluded (see the PCIe/register-map
section §3.2). The upper limit is `0x0010_0000` (1 MiB) on MT7932 **and MT7922**, versus
`0x000F_0000` in the public `gen4m` `mt7961` record — upstream `mt76`'s MT7921 path uses
1 MiB. [C]

Chip addresses at or above that limit are translated through a fixed chip→BAR table. The
entries relevant to power/reset are:

| Chip base | BAR offset | Size | Block |
|---|---|---|---|
| `0x7403_0000` | `0x01_0000` | `0x10000` | PCIe MAC (`PCIE_MAC_IREG`) |
| `0x8800_0000` | `0x04_0000` | `0x10000` | *L1 remap window 0* — see §1.3 `[U]` |
| `0x7000_0000` | `0x07_0000` | `0x10000` | CB-TOP (RGU, MTCMOS, HW ID) — usable only after §3.2 |
| `0x0040_0000` | `0x08_0000` | `0x10000` | WF MCU SYSRAM |
| `0x7C05_0000` | `0x09_0000` | `0x10000` | CONN_INFRA SW-defined CR / share-info mailbox |
| `0x8002_0000` | `0x0B_0000` | `0x10000` | WF_TOP_MISC_OFF / WF_TOP_CFG |
| `0x8102_0000` | `0x0C_0000` | `0x10000` | WF_TOP_MISC_ON |
| `0x7C02_0000` | `0x0D_0000` | `0x10000` | CONN_INFRA, WFDMA |
| `0x7C06_0000` | `0x0E_0000` | `0x10000` | CONN_INFRA, `conn_host_csr_top` (LPCTL) |
| `0x7C00_0000` | `0x0F_0000` | `0x10000` | CONN_INFRA (incl. remap CSRs, WFSYS reset status) |

**Correction — do not use BAR `0x4000` as a software-interrupt alias.** An earlier reading of
this section claimed a legacy `PCIE_HIF` aliasing of the host WFDMA0 CSR page at BAR `0x4000`,
with `BAR 0x4108` = `HOST2MCU_SW_INT_SET` and `BAR 0x41F0` = `MCU2HOST_SW_INT_STA`. That is
wrong for MT7932:

* BAR `0x4000` decodes to chip `0x5600_0000`, the **reserved** WFDMA instance (§1 §4 of the
  PCIe/register-map section). Writing there pokes a block this part does not use. [C]
* The host→MCU software interrupt on MT7932 is chip **`0x5400_0108`** (BAR `0x2108`), reached
  through the per-chip software-interrupt hook, which is populated for this part. [C]
  A bare write to offset `0x4108` exists in the generic code but is the **fallback for parts
  that supply no such hook**, and is dead code on MT7932. [C]
* `MCU2HOST_SW_INT_STA` is chip **`0x7C02_41F0`** (BAR `0x0D41F0`); no `0x41F0` bare-offset
  form is used. [C]

See §8.1 and the interrupt section §3.2, which carry the correct addresses.

### 1.3 Programmable L1 remap windows

Four 64 KiB BAR windows are re-targeted by the power/reset paths, through two CONN_INFRA
registers. Each register holds **two 16-bit page selectors**; a selector value is chip-address
bits `[31:16]` of the 64 KiB page to expose. [C] The remap array as a whole is larger and
extends beyond these two registers — see the PCIe/register-map section §5, which is the
authority for its extent.

| Register | Field | BAR window | Host default written |
|---|---|---|---|
| `0x7C00_E24C` | `[15:0]` | BAR `0x04_0000` | `0x184F` |
| `0x7C00_E24C` | `[31:16]` | BAR `0x05_0000` | `0x1845` |
| `0x7C00_E250` | `[15:0]` | BAR `0x06_0000` | `0x1846` |
| `0x7C00_E250` | `[31:16]` | BAR `0x07_0000` | `0x7000` (programmed at init) |

* The composite value the host writes to restore the idle state of `0x7C00_E24C` is
  **`0x1845_184F`**. [C]
* A **2 µs** settling delay is required between writing a remap selector and using the
  window. [C]
* Temporary retargets of window 0 that bring-up and reset use: `0x7000` (CB-TOP), `0x7001` (CB-TOP HW-ID /
  GALS page), `0x1807` (CONN semaphore block). [C]
* The idle selector value `0x184F` **is** the setting that makes the static-map entry
  BAR `0x04_0000` ↔ chip `0x8800_0000` work: `0x184F` selects the conn-infra/AP-bus
  aperture through which the WF-subsystem `0x8800_xxxx` block is reached, so with the
  default restored, `0x8800_0000`-range chip addresses resolve normally through the static
  map. Any temporary retarget of window 0 therefore breaks `0x8800_xxxx` addressing until
  `0x1845_184F` is written back — which is exactly why every borrow of the window restores
  it. [C]

---

## 2. Initial state at PCI function appearance

### 2.1 What is running

The states below are the ones the bring-up path assumes and tests for; they are `[L]` unless a
row says otherwise, because the reset-time contents of these registers were not read out.

| Item | State at enumeration |
|---|---|
| Wi-Fi subsystem (WFSYS) core | held in low power; owned by firmware/boot ROM `[L]` |
| MCU | boot ROM only; no RAM code |
| `MT_CONN_ON_LPCTL[2]` (`HOST_OWN_SYNC`) | `1` — **firmware owns** the chip `[L]` (the host unconditionally performs a driver-own acquisition first, `[C]`) |
| `MT_CONN_ON_MISC[1:0]` (`FW_PWR_ON`,`FW_N9_ON`) | `0` — Wi-Fi function not ready `[L]` |
| PCI cfg `0x48C[19:16]` (boot stage) | `0` — initial state `[C]` (this is the value bring-up polls for, §6.3) |
| Host WFDMA | disabled `[L]` |
| BAR-0 L1 remap window 0 selector | `0x184F` (i.e. **not** pointing at CB-TOP) `[L]` |
| BAR-0 L1 remap window 3 selector | undefined until programmed (§3.2) `[L]` |

The part may also come up with firmware *already running* — e.g. after a warm restart or
after the Bluetooth function has powered the shared CONNINFRA domain. The bring-up path
therefore explicitly tests for "Wi-Fi is already ON" and, if so, performs a full power-off
(§4.1) before firmware download. [C]

### 2.2 What is readable before any handshake

* All PCI configuration space, including `0x484` / `0x488` / `0x48C` and the AER block. [C]
* `conn_host_csr_top` (`0x7C06_0000`–`0x7C06_FFFF`), which contains the ownership register
  itself — this must be readable with firmware own, by construction. [C]
* CONN_INFRA (`0x7C00_0000` page), used to poll WFSYS "sw init done" immediately after a
  subsystem reset, i.e. before any ownership is taken. [C]
* The remap CSRs `0x7C00_E24C` / `0x7C00_E250`. [C]
* MMIO reads that return `0xFFFF_FFFF` are treated as a PCIe access failure **except** for
  a small whitelist of CRs that legitimately read all-ones: PLE `0x820C_0600`,
  `0x820C_0604`, `0x820C_0680`, `0x820C_0684`, `0x820C_0700`, `0x820C_0704`,
  `0x820C_0780`, `0x820C_0784`, and the entire `0x820D_xxxx` (WTBLON) page. [C]

### 2.3 Reading the ASIC identity / revision

WF_TOP_CFG lives at `0x8002_0000` and is inside the Wi-Fi power domain, so it is only
reliable **after driver ownership has been acquired** (§3):

| Chip address | Contents |
|---|---|
| `0x8002_1000` | version register; bits `[3:0]` = ECO / revision |
| `0x8002_1008` | chip-ID register; bits `[15:0]` = `0x7932` |
| `0x8002_1010` | liveness canary — reads `0xDEAD_FEED` when the WF bus is down (§6.4) |

**MT7932 delta:** the hardware-configuration record for MT7932 sets *should_verify_chip_id
= 0*, i.e. the MMIO chip-ID compare is **skipped** on this part; the chip ID
is taken from the PCI device ID instead. [C] Why it is skipped — for instance because
`0x8002_1008` is not dependably readable at the point the check would run — is not
established. `[U]`

The ECO/hardware/firmware version words actually used for firmware-image selection are
fetched **before the firmware download**, through the boot-ROM (initialisation)
register-access command while only the boot ROM is running — see §4.1 step 13 and the
firmware-boot section — from:

| Name | Chip address |
|---|---|
| `TOP_HCR` | `0x7001_0200` |
| `TOP_HVR` | `0x7001_0204` |
| `TOP_FVR` | `0x8800_0004` |

`TOP_HVR[7:0]` → HW version, `TOP_HVR[11:8]` → factory version, `TOP_FVR[7:0]` → SW
version; these are combined into the ECO index. [C]

### 2.4 Firmware readiness indication — MT7932 delta

Two different mechanisms exist and the choice is made **at run time on the PCI device ID**:

| Device | Ready indication |
|---|---|
| `0x7922` | poll MMIO `MT_CONN_ON_MISC` = `0x7C06_00F0`, ready when bits `[1:0]` (`FW_PWR_ON` \| `FW_N9_ON`) are both set (shift 0, mask `0x3`) |
| `0x7932` (and `0x7923`) | poll **PCI config `0x48C`**, ready when bits `[19:16]` == `2` |

Both use a 5 ms poll interval and a **5000 ms** total timeout; on timeout the graded
recovery is entered with reason *CR access fail*. Power-*off* completion is the same poll
with the inverted predicate (ready bits clear / boot stage != 2). [C]

Boot-stage field encoding of `0x48C[19:16]`:

| Value | Meaning |
|---|---|
| `0` | MCU in initial / reset state (Wi-Fi function off) |
| `2` | Wi-Fi function ready / firmware running |

This config-space boot-stage field is the single most useful MT7932 bring-up signal: it is
readable with no ownership, no MMIO mapping and no clocks. [C]

### 2.5 PCIe doorbell push (PCI config `0x484`)

A write-only doorbell that the host uses to signal the chip's always-on logic. Bits in
use:

| Bit | Value written | Meaning |
|---|---|---|
| `0` | `0x0000_0001` | set firmware own (config-space ownership path); also the "notify Bluetooth before a Wi-Fi FLR" write (§5.5) `[C]` value, `[L]` name |
| `1` | `0x0000_0002` | clear own / request driver own (config-space ownership path) `[L]` |
| `11` | `0x0000_0800` | host is collecting a firmware assert dump / coredump `[C]` |
| `13` | `0x0000_2000` | host entering hibernate `[C]` |
| `31` | `0x8000_0000` | **host-forced firmware assert** — makes the MCU take an exception and produce a coredump `[C]` |

Bits 0/1 carry the public `gen4m` names `CR_PCIE_CFG_SET_OWN` / `CR_PCIE_CFG_CLEAR_OWN`.
Only one bit is ever written at a time; the register is written through the MMIO alias
`0x7403_1484` in the runtime paths (bits 11, 13, 31) and through a configuration cycle at
`0x484` in the reset paths (bit 0). [C]

---

## 3. Driver / firmware ownership handshake

### 3.1 Registers

| Chip address | Name | Field | Semantics |
|---|---|---|---|
| `0x7C06_0010` | `MT_CONN_ON_LPCTL` (`CONNAC2X_BN0_LPCTL_ADDR`) | `BIT(0)` `PCIE_LPCR_HOST_SET_OWN` | W1: hand ownership back to firmware |
| | | `BIT(1)` `PCIE_LPCR_HOST_CLR_OWN` | W1: request driver ownership |
| | | `BIT(2)` `PCIE_LPCR_HOST_OWN_SYNC` | RO level: `0` = **driver owns**, `1` = firmware owns |
| `0x7C06_0014` | `MT_CONN_ON_IRQ_STAT` (`CONNAC2X_BN0_IRQ_STAT_ADDR`) | `BIT(0)` `PCIE_LPCR_FW_CLR_OWN` | W1C: firmware-cleared-own interrupt |
| `0x7C06_0018` | `MT_CONN_ON_IRQ_ENA` (`CONNAC2X_BN0_IRQ_ENA_ADDR`) | — | interrupt enable for the above |
| `0x7C06_00F0` | `MT_CONN_ON_MISC` | `[1:0]` | firmware power-on / N9-ready |

**Confirmation is a level, not an edge**: `LPCTL[2]` is polled. The interrupt-status bit
`0x7C06_0014 BIT(0)` *can* participate — the hardware-configuration record carries a
`fw_own_clear_addr` = `0x7C06_0014` and `fw_own_clear_bit` = `BIT(0)`, and a boolean
"check driver-own interrupt". **On MT7932 that boolean is `FALSE`**, so the interrupt path
is not used; the host neither waits on nor acknowledges `0x7C06_0014` during a normal
driver-own. (When the boolean is `TRUE`, the sequence is: poll the interrupt-status bit
instead of `LPCTL[2]`, then write `BIT(0)` back to `0x7C06_0014` to acknowledge.) [C]

### 3.2 Acquire driver own — register path

```
1.  write  0x7C06_0010 = BIT(1)            # HOST_CLR_OWN
2.  loop:
3.      delay 1000 us
4.      read 0x7C06_0010 ; owned = (val & BIT(2)) == 0
5.      if owned -> done
6.      if PCIe function removed or bus-access-failure latched -> abort
7.      if (now - last_clr_own_write) > 200 ms -> repeat step 1
8.      if (now - start) > 2048 ms -> timeout
```

* Poll interval: **1000 µs** (busy delay, not a sleep). [C]
* Re-issue interval for the `HOST_CLR_OWN` write: **200 ms**; the write is unconditionally
  re-issued on the first poll iteration. [C]
* Total timeout: **2048 ms** (0x800 ms). [C]
* Early-abort conditions: PCI function removed, sticky "bus access failed" flag, or the
  "chip no-ack" condition (see §6.5). [C]
* Required follow-up actions, in order: (a) if the driver-own-interrupt mode is enabled,
  write `fw_own_clear_bit` to `0x7C06_0014` to acknowledge; (b) if the device was resumed
  from suspend and the shared block reports an in-suspend SER (§4.3), complete the
  in-suspend SER sequence. The chip-specific "check dummy CR" step defined for other
  CONNAC2 parts does not apply to MT7932. [C]

There is **no ASPM settling delay** in this path — contrast with the public `mt792x`
driver, which inserts a 2–3 ms `usleep` before each poll when ASPM is enabled. See §7. [C]

### 3.3 Release ownership back to firmware

```
1.  (chip-specific "set dummy CR" step - not applicable to MT7932)
2.  write 0x7C06_0010 = BIT(0)                   # HOST_SET_FW_OWN
```
The host must then treat the chip as firmware-owned and refuse further MMIO.

The register-based release performs **no read-back confirmation** — it is fire-and-forget
and always reports success. Confirmation, where needed, is obtained indirectly:
* the *IPC* release variant clears the grant flag in the shared block (§3.4);
* the suspend path verifies the transition by re-acquiring and re-releasing in a loop
  (§4.3).

Release must **not** be issued while any of the following holds, because the chip would be
put to sleep with work outstanding: firmware already owns; a host-triggered SER is in
progress; a coredump is in progress (shared-block flag, §6.6); a WFSYS reset is pending;
some other part of the host still needs register access; or a host interrupt is still
pending. In the last case the interrupt sources must be drained first and the release
deferred. [C]

### 3.3a Firmware-initiated ownership requests, and the preconditions for a release

The ownership handshake is not purely host-driven. Two things the host must know:

**(a) The firmware asks for ownership back.** When the firmware has drained its work it
emits the unsolicited MCU event `EVENT_ID_SLEEPY_INFO` (`ucEID = 0x07`) whose body byte 0 is
a "sleepy" flag. A non-zero flag is a request to the host to run the release-ownership
sequence (§3.3). The host must latch the flag and perform the release. `[C]`

* The request is **not repeated**. If the host drops the event, the firmware stays awake
  indefinitely; there is no timeout, no error and no second notification — the only symptom
  is that the part never reaches its low-power state. `[C]`
* The flag is level-like: a zero value withdraws the request. `[C]`
* The host may still refuse the release for any of the reasons in §3.3; the request is
  advisory as to *timing*, mandatory as to *eventually honouring it*.

**(b) The firmware appears not to complete a sleep entry while host interrupts are
outstanding.** Before parking, the firmware is understood to sample the host-side WFDMA
interrupt status and to treat a non-zero value as a reason not to proceed. `[L]` — this is a
firmware behaviour inferred from the host-side ordering rule and the firmware's own log sites,
not something observed on silicon. It is the reason for the
host-side rule already stated in §3.3 (a release is deferred while the interrupt service
reports work outstanding), and it makes the ordering **mandatory rather than merely
prudent**:

```
service the interrupt  ->  write the status back (acknowledge)  ->  drain the rings
                       ->  only then write HOST_SET_OWN
```

A host that releases ownership with an unacknowledged interrupt latched will find that the
part does not enter low power, and — because the release is fire-and-forget and reports
success (§3.3) — the host will believe it did. `[L]` (consequence of the inferred behaviour
above; the fire-and-forget release itself is `[C]`)

**(c) There is no host keep-alive.** Nothing in the interface requires the host to poll,
ping or otherwise prove liveness to the firmware while it holds ownership: there is no
watchdog the host must pet, no periodic command and no heartbeat event. The `0x00`
"dummy/reserved" command exists and is header-only, but is not used as a keep-alive on this
part. The firmware's own watchdog supervises the firmware, not the host. `[C]` The
host→MCU doorbell value `0x0000_2000` (§8.5) is the nearest thing to a liveness signal and
is a one-shot "host is going away" notification, not a periodic one. `[C]`

### 3.4 IPC / shared-memory ownership path (low-power path)

MT7932 supports a second ownership status channel that does **not** require an MMIO read of
`LPCTL`. Firmware maintains a host-DMA "share info" block; the host reads the ownership
grant from host memory instead of from the chip. The register writes are identical.

| Operation | Register write | Status source |
|---|---|---|
| read own | — | shared block `+0x08`, `BIT(0)`: `1` = driver owns |
| request driver own | `0x7C06_0010 = BIT(1)` | shared block `+0x08`, `BIT(0)` |
| release to firmware | `0x7C06_0010 = BIT(0)` | host clears shared block `+0x08 BIT(0)` locally |

Selection rule: the IPC variants are used **once firmware has signalled "Wi-Fi function
ready"** (§2.4) and the share-info block has been published; the plain register variants
are used before that and after any reset. The selector is cleared on: WFSYS (L0.5) reset
entry, L0 reset entry, hibernate, and whenever the "already ON, power off first" path runs. [C]

Rationale (consistent with the public `gen4m` comment on this register): the `LPCTL` status
bit is not reliable while the chip is asleep. [L]

The handshake is only meaningful on parts that implement ASIC low-power support
(`is_support_asic_lp`); MT7932 does, so it is live. [C]

### 3.5 Share-info block — publication and layout

The share-info block is a host-allocated DMA buffer of **`0x300` bytes** (a second,
`0x320`-byte, buffer is published alongside it). It is handed to firmware through a
CONN_INFRA SW-defined-CR mailbox before firmware download completes:

| # | Chip address | Written with |
|---|---|---|
| 1 | `0x7C05_3C28` | `1` (enable / doorbell; written **first**) |
| 2 | `0x7C05_3A38` | control flags: `BIT(0)` = 64-bit host addressing in use, `BIT(1)` = share info valid |
| 3 | `0x7C05_3A3C` | **share-info block bus address, low 32 bits** |
| 4 | `0x7C05_3A30` | second host buffer bus address, low 32 bits |
| 5 | `0x7C05_3A54` | third host buffer bus address, low 32 bits |
| — | `0x7C05_3A34` / `0x7C05_3A58` | high 32 bits of the pairs at `0x3A30` / `0x3A54`; written **only** in 64-bit addressing mode |
| — | `0x7C05_3A50`, `0x7C05_3A5C` | present in the per-chip register list, never written `[C]`; purpose `[U]` |

The two buffer sizes (`0x300` and `0x320`) are carried as a per-chip constant beside these
register addresses and are **not** written to any register. `[C]` value, `[L]` identification.

MT7932 declares a **32-bit DMA mask**, so only the low-address form is used and
`BIT(0)` of `0x7C05_3A38` stays clear. On a failed suspend the host clears `BIT(1)` of
`0x7C05_3A38` to invalidate the block. [C]

Fields of the block that participate in power/reset (byte offsets into the block):

| Offset | Field |
|---|---|
| `+0x00` | status word: bits `[5:2]` = "SER L1 happened"; `BIT(30)`/`BIT(31)` = SER triggered / SER done **while suspended** |
| `+0x08` | `BIT(0)` = driver-own grant (IPC ownership path) |
| `+0x10`..`+0x2C` | LMAC error 6 (×2), LMAC error 7 (×2), PSE error (×2), PLE error (×2) latches |
| `+0x30` | `BIT(0)` = "WFDMA idle" acknowledgement during suspend |
| `+0x158` | `BIT(0)` = coredump in progress (W1C by host) |
| `+0x15C` | `BIT(0)` = firmware memory leak / command not processed (W1C by host) |
| `+0x164` | health-monitor error code (§6.7) |
| `+0x168`..`+0x2E8` | per-monitor detail payloads |

---

## 4. Power-on and power-off sequences

### 4.1 Cold bring-up order

```
 1. pci_enable_device; enable MSI (1 vector); set 32-bit DMA mask; map BAR0; set bus master
 2. "MCU back to initial state" check  -> see §6.3
 3. acquire driver own                 -> §3.2
 4. establish the CB-TOP MMIO mapping:
       write 0x7C00_E250 = 0x7000_1846
       delay 2 us
       read  0x7C00_E250 ; require bits[31:16] == 0x7000
    (if the read-back does not show 0x7000, the mapping is unusable and every CB-TOP
     access must instead borrow L1 window 0 as in §5.6)
 5. stop host WFDMA                    -> §5.3
 6. verify chip ID                     -> skipped on MT7932 (§2.3)
 7. "wake up Wi-Fi":
       read readiness (§2.4)
       if already ready -> full power-off (§4.2) then re-acquire driver own
       read/clear ownership again
 8. allocate and initialise WFDMA rings; init MSDU token table
 9. publish the share-info block       -> §3.5
10. enable firmware download mode
11. start host-side packet servicing
12. enable interrupts                  -> §4.4
13. read ECO/version via MCU register access (§2.3)
14. acquire the secure-boot semaphore  -> §4.5
15. download ROM patch + RAM code
16. release the secure-boot semaphore
17. poll for "Wi-Fi function ready"    -> §2.4  (5 ms period, 5000 ms timeout)
18. switch the ownership handshake to the IPC path (§3.4)
19. query NIC capability, apply configuration
```
[C]

### 4.2 Graceful power-off ("power off Wi-Fi")

```
1. acquire driver own
2. disable host interrupts
3. if no FLR is in flight:
       send the NIC power-control command (power off) to firmware
   else:
       run the WFSYS reset-to-initial procedure (§5.1), up to 2 attempts
4. mark "power off in progress"
5. poll for power-off completion:
       device 0x7922 : MT_CONN_ON_MISC[1:0] == 0
       device 0x7932 : PCI cfg 0x48C[19:16] != 2
   (5 ms period, 5000 ms timeout)
6. release ownership to firmware
```
[C]

### 4.3 Suspend / resume (relevant ownership behaviour)

```
suspend:
  - set "suspending"; block the driver-own-timeout recovery trigger
  - wait until no reset is in progress          (400 x 5 ms = 2000 ms)
  - acquire driver own; stop TX queues
  - send the PCIe pre-suspend MCU command; wait for "pre-suspend done"
        (501 x 2 ms ~= 1002 ms; on timeout dump the HIF state)
  - poll the shared block "WFDMA idle" flag     (10 ms period, 500 ms timeout)
  - stop host WFDMA
  - poll "no pending L1 reset" (shared block +0x00 bits[5:2] == 0)
        (40 ms period, 200 ms timeout; expiry means an L1 reset is still outstanding)
  - disable interrupts
  - release ownership to firmware
  - if ownership was not actually released, retry up to 500 times:
        acquire driver own; sleep 1000-3000 us; release ownership
        -> expiry of all 500 iterations is a firmware-own timeout
  - wait again until no reset is in progress    (400 x 5 ms)
  - set "suspend done, must not touch CRs" (all MMIO is refused past this point)
```
On resume, the first successful driver-own additionally checks shared-block `+0x00`
`BIT(31)`/`BIT(30)`; if a SER occurred while suspended, it waits (10 ms period, **200 ms**
timeout) for `BIT(31)`, clears bits `[31:30]`, then re-runs DMASHDL re-init, ring
re-allocation, MSDU-token reset, ring re-init and beacon re-init. [C]

### 4.4 Interrupt enable/disable (power-relevant)

| Chip address | Enable value | Disable value |
|---|---|---|
| `0x7403_0188` (`MT_PCIE_MAC_INT_ENABLE`) | `0x0000_01FF` | `0x0000_0000` |
| `0x7C02_4204` (`MT_WFDMA0_HOST_INT_ENA`) | `0x6C00_0000` \| (ring bits & `0x93CF_FFFF`) | `0x0000_0000` |
| `0x7C02_41F4` (`MCU2HOST_SW_INT_ENA`) | `0x0000_FFFF` (only when asic-LP is supported) | — |
| `0x7403_018C` | PCIe MAC interrupt clear (write status back) | — |

`0x6C00_0000` = `TX_DONE_INT16`(FWDL, BIT26) \| `TX_DONE_INT17`(MCU cmd, BIT27) \|
`MCU2HOST_SW_INT_ENA`(BIT29) \| `TX_DONE_INT18`(BIT30). While the driver is still in its
pre-firmware-ready (initialisation) state the base mask is reduced to `0x6000_0000`
(BIT29 \| BIT30 only). [C] — see the interrupt section §5.

A **1 ms delay precedes the interrupt enable** `[C]`. It is described as a work-around; its
scope, and whether the silicon requires it at all, are not established — treat it as
mandatory. `[U]`

### 4.5 Secure-boot semaphore (firmware download gate)

Wrapping the firmware download, the host takes a hardware semaphore in the CONN semaphore
block, reached by temporarily retargeting L1 window 0:

```
write 0x7C00_E24C = 0x1845_1807          # window 0 -> chip 0x1807_0000
acquire: poll BAR 0x4_0060 (= chip 0x1807_0060) BIT(0)
         1000 us period, 5000 iterations  => 5 s timeout
release: write BAR 0x4_0260 (= chip 0x1807_0260) = 1
write 0x7C00_E24C = 0x1845_184F          # restore
```
[C]

---

## 5. Reset sequences

### 5.1 WFSYS (Wi-Fi subsystem) reset — "reset to initial state" / L0.5

This is the CB-TOP RGU "WF whole path" reset. On MT7932 the CB-TOP RGU is reached through
L1 remap window 0.

```
1. mask AER (§5.4, disable)
2. assert:
     a. run the MTCMOS hardware-mode sequence (§5.6)
     b. write 0x7C00_E24C = 0x1845_7000        # window 0 -> chip 0x7000_0000
     c. delay 2 us
     d. write BAR 0x4_2600 (= chip 0x7000_2600) = 0x0000_0001
3. delay 50 000 us  (50 ms)
4. de-assert:
     a. write BAR 0x4_2600 (= chip 0x7000_2600) = 0x0000_0000
     b. write 0x7C00_E24C = 0x1845_184F        # restore window 0
5. unmask AER (§5.4, enable)
6. poll "sw init done":
     read 0x7C00_0140
       - 0xFFFF_FFFF -> MMIO read failure, retry
       - BIT(4) set  -> success
     3 attempts total, 100 ms sleep between attempts   (=> ~200 ms budget)
7. failure -> return error; caller escalates
```

Register detail:

| Chip address | Name | Bits |
|---|---|---|
| `0x7000_2600` | `CBTOP_RGU_WF_SUBSYS_RST` (`MT_CBTOP_RGU_WF_SUBSYS_RST`) | `BIT(0)` `WF_WHOLE_PATH_RST` (1 = assert), `BIT(6)` `BYPASS_WFDMA_SLP_PROT` |
| `0x7C00_0140` | WFSYS software reset / status (`MT_WFSYS_SW_RST_B` equivalent) | `BIT(0)` `WFSYS_SW_RST_B`, `BIT(4)` `WFSYS_SW_INIT_DONE` |

The value written to assert is `0x0000_0001` (i.e. `WF_WHOLE_PATH_RST` only; the
`BYPASS_WFDMA_SLP_PROT` bit is **not** set on this path). [C]

The public `mt76` MT7921 flow uses the same `BIT(4)` "sw init done" bit but reaches the
register at `0x1800_0140` through its own L1 remap and polls for up to 500 ms; the MT7932
path reaches it at CONN_INFRA `0x7C00_0140` through the *static* map and polls only
~200 ms. `[C]`

### 5.2 "Clear own IRQ status" after a WFSYS reset — MT7922-only

After a WFSYS reset and before handing ownership back to firmware, the following step is
executed **only when the PCI device ID is `0x7922`**; it is skipped on MT7932:

```
write 0x7C00_1620 = 0x0000_0003
sleep 2 ms
```

`0x7C00_1620` is a CONN_INFRA ownership/interrupt status register; bits `[1:0]` are
write-1-to-clear. On MT7932 the ownership interrupt state apparently does not need to be
cleared after a subsystem reset. This is a genuine MT7932 delta. [C]

### 5.3 WFDMA / HIF reset

Two distinct operations exist.

**(a) HIF reset (`WPDMA_HIF_RST`)** — resets DMASHDL and WPDMA together:

```
for each WFDMA instance base B in {host_dma0_base, host_dma1_base}:
    write (B + 0x100) = 0x0000_0000
    write (B + 0x100) = 0x0000_0030
```
`B + 0x100` is `CONNAC2X_WPDMA_HIF_RST` / `MT_WFDMA0_RST`; `0x30` =
`MT_WFDMA0_RST_LOGIC_RST` (BIT 4) \| `MT_WFDMA0_RST_DMASHDL_ALL_RST` (BIT 5).
On MT7932 `host_dma0_base` = `0x7C02_4000` so the first target is **`0x7C02_4100`**;
`host_dma1_base` is **0** on MT7932 (WFDMA1 is not present), so only instance 0 must be
reset; a driver should skip the absent instance rather than write BAR offset `0x100`. [C]

**(b) WFDMA stop / idle wait** — used before suspend, reset and power-off:

```
1. read  GLO_CFG (0x7C02_4208)
2. write GLO_CFG = value & 0xE7DF_7FFA          # clears TX_DMA_EN, RX_DMA_EN,
                                                 # CSR_DISP_BASE_PTR_CHAIN_EN,
                                                 # OMIT_RX_INFO_PFET2, OMIT_RX_INFO,
                                                 # OMIT_TX_INFO
3. poll GLO_CFG until (value & 0x0000_000A) == 0 # TX_DMA_BUSY | RX_DMA_BUSY clear
   101 iterations, 1000 us apart  => ~101 ms
```

The enable direction ORs in `0x5020_9040` for WFDMA0 (`TX_WB_DDONE` BIT6,
`FIFO_LITTLE_ENDIAN` BIT12, `CSR_DISP_BASE_PTR_CHAIN_EN` BIT15, `OMIT_RX_INFO_PFET2` BIT21,
`OMIT_TX_INFO` BIT28, `CLK_GAT_DIS` BIT30) — `0x5820_9040` for WFDMA1 — and then separately
ORs `0x5` (`TX_DMA_EN` \| `RX_DMA_EN`). A separate "hard stop" variant writes the cached
GLO_CFG value with bits 1 and 3 masked out and either `0x0` or `0x5` reinstated. [C]

**(c) WFDMA "needs re-init" dummy CR** — `0x5400_0120` (`MT_WFDMA_DUMMY_CR`), `BIT(1)`
(`MT_WFDMA_NEED_REINIT`). Firmware sets this bit when it has torn WFDMA down during deep
sleep. The host reads it after taking ownership; if clear, it re-initialises the rings,
re-enables interrupts and then writes the bit back (set). [C]

**(d) In-suspend WFDMA idle** does not use `GLO_CFG` at all: it polls the shared-block
"WFDMA idle" flag (`+0x30 BIT(0)`) with a 10 ms period and a **500 ms** timeout, then
clears it. A read of `0` is explicitly treated as "bypass this timeout". [C]

### 5.4 PCIe function-level reset (FLR)

```
retry = 1
loop:
  1. AER mask disable:  write cfg 0x208 = 0x0057_F010
  2. run the MTCMOS hardware-mode sequence (§5.6)
  3. mark the function as resetting (all MMIO must be refused from here)
  4. issue a PCIe function-level reset on this function
  5. read cfg 0x48C
  6. if the FLR call succeeded AND (cfg 0x48C & 0x000F_0000) == 0 -> success, break
  7. dump the PCIe debug register set (§6.1); retry++
  8. if retry == 4 -> give up
on success:
  9. AER re-enable:  write cfg 0x204 = 0xFFFF_FFFF ; write cfg 0x208 = 0x0040_0000
```

* **Success criterion**: the firmware boot-stage field `cfg 0x48C[19:16]` must read `0`,
  i.e. the MCU is back in its initial state. [C]
* **Retries**: 3 attempts. [C]
* **AER mask value during FLR**: `0x0057_F010` masks Data-Link-Protocol (b4), Poisoned TLP
  (b12), Flow-Control-Protocol (b13), Completion-Timeout (b14), Completer-Abort (b15),
  Unexpected-Completion (b16), Receiver-Overflow (b17), Malformed-TLP (b18),
  Unsupported-Request (b20) and b22. The restored mask is `0x0040_0000` (b22 only). [C]
* **What FLR clears**: the MCU state (boot stage → 0), the Wi-Fi subsystem, host WFDMA. It
  does **not** clear the PCIe MAC's L1 remap windows, and it does not restore the AER mask
  — both must be re-programmed by the host afterwards. [L]
* An FLR initiated by the platform must be handled the same way: on the pre-reset
  notification the host marks the function as resetting and enters graded recovery with
  reason *FLR*; the post-reset notification releases it. A **2000 ms** budget is allowed
  for the pair. [C]

### 5.5 Bluetooth coordination across a Wi-Fi reset

Wi-Fi and Bluetooth are separate PCIe functions of the same die and share the CONNINFRA
power domain. The chip-support layer provides an explicit **"notify BT before Wi-Fi FLR"**
operation:

```
write PCI config 0x484 = 0x0000_0001      # PCIe doorbell push, CR_PCIE_CFG_SET_OWN
sleep 1 ms
```

That is the entire mechanism: a single doorbell write plus a **1 ms** settling delay. [C]
The operation is defined for MT7932, but the FLR sequence itself (§5.4) consists only of AER
masking and the MTCMOS sequence, so whether the notification is mandatory is unconfirmed. A
driver should assume it is required and issue it before any FLR or WFSYS reset. `[U]`

Related evidence of shared-domain coupling:
* The second PCIe function's configuration space is mirrored at chip `0x7403_9000 +
  cfg_offset`, and the Wi-Fi bus-failure dump reads the BT function's AER status
  (`0x7403_9204`) and link registers (`0x7403_9210`, `0x7403_921C`, `0x7403_9220`,
  `0x7403_9224`, `0x7403_9228`) alongside its own. [C]
* The MTCMOS sequence and the CB-TOP RGU register both live in the shared CB-TOP block. [C]

### 5.6 MTCMOS / power-domain sequencing

Two variants exist; the choice depends on whether the permanent CB-TOP MMIO mapping
(§4.1 step 4) was established.

**Hardware mode, mapping *not* established** (borrow L1 window 0):
```
1. acquire driver own (register or IPC variant)
2. write 0x7C00_E24C = 0x1845_7000        # window 0 -> chip 0x7000_0000
3. delay 2 us
4. write BAR 0x4_3020 (= chip 0x7000_3020) = 0x0000_0000     # VLP_UDS_CTRL
5. delay 1000 us
6. write 0x7C00_E24C = 0x1845_184F        # restore window 0
```

**Hardware mode, mapping established** (direct):
```
1. acquire driver own
2. write 0x7000_3020 = 0x0000_0000        # via the permanent BAR 0x7_0000 window
3. delay 1000 us
```

* `0x7000_3020` is the CB-TOP `VLP_UDS_CTRL` register (same address as in the public
  MT7925 / MT6639 debug flows). [C]
* Both variants require a valid PCIe device handle; the sequence is skipped entirely when
  the PCI device ID is `0x7922`. **On MT7932 it runs** — another MT7922/MT7932 delta. [C]
* Only the hardware-mode sequence applies to MT7932; the software-mode (host-sequenced
  power-switch chain) variant defined for other parts is not used. [C]
* The sequence is invoked from the FLR path and from the WFSYS-reset assert path. [C]

### 5.7 Connectivity-bus / RGU reset path

Present, and it is exactly the WFSYS reset of §5.1: the CB-TOP **RGU** register
`0x7000_2600` with `WF_WHOLE_PATH_RST`. There is no separate "conn-infra reset" beyond
this; a full conn-infra/whole-chip reset is achieved through PCIe FLR (§5.4) or through the
L0 path (§6.8, level L0), which tears the driver down and re-probes the function. [C]

---

## 6. Fault detection

### 6.1 PCIe debug register dump

On any detected bus failure the host reads this fixed list of PCIe-MAC registers. They are
the most useful first-line probes for a new driver:

```
0x7403_0150  0x7403_0154  0x7403_01D8  0x7403_01E0
0x7403_1204  0x7403_9204                          # AER unc. status, WiFi / BT function
0x7403_1210  0x7403_121C  0x7403_1220  0x7403_1224  0x7403_1228   # WiFi function link regs
0x7403_9210  0x7403_921C  0x7403_9220  0x7403_9224  0x7403_9228   # BT function link regs
0x7403_1480  0x7403_03C8  0x7403_03CC  0x7403_818C
0x7403_0E00  0x7403_0E04  0x7403_0E08
0x7403_0184  0x7403_01D4
```
plus PCI configuration `0x10`, `0x14`, `0x04`, `0xFC`, `0x204`, `0x210`. [C]

It then runs a scripted PCIe-MAC debug-mux capture:

| Step | Write | Then read |
|---|---|---|
| 1 | `0x7403_0168 = 0xCCCC_0100`, `0x7403_0164 = 0x4F4E_4D4C` | `0x7403_002C` |
| 2 | `0x7403_0168 = 0x9999_0100`, `0x7403_0164 = 0x5756_5553` | `0x7403_002C` |
| 3.. | `0x7403_0164 = 0x2622_2120`, `0x5B5A_5958`, `0x5F5E_5D5C`, `0x2725_2423`, `0xB7B6_B5B4`, `0xB7B6_B098`, `0xBBBA_B9B8`, `0xBFBE_BDBC` | `0x7403_002C` after each |

So: `0x7403_0168` = debug group/mode select, `0x7403_0164` = 4×8-bit debug-signal select,
`0x7403_002C` = debug status read-back. [C]

### 6.2 Fabric / power-domain check (PCI config `0x488`)

Reads PCI configuration `0x488`. A value of `0xFFFF_FFFF` means the configuration read
itself failed → latch "bus access failed" and stop.

Level 1.1 / 1.2 (always checked):

| Bit | Required value |
|---|---|
| 0 | 1 |
| 1 | 1 |
| 4 | 0 |
| 6 | 1 |
| 9 | 1 |
| 25 | 1 |
| 26 | 1 |

Level 2 (checked only when the caller asks for level 2), additionally:

| Bit | Required value |
|---|---|
| 10 | 0 |
| 13 | 1 |
| 14 | 1 |
| 15 | 1 |

Any mismatch means the connectivity fabric or one of its power domains is not up.
**The whole check is skipped when the PCI device ID is `0x7922`; it runs on MT7932** —
MT7932 delta. [C]

### 6.3 "MCU back to initial state" check

```
read PCI cfg 0x48C
if (value & 0x000F_0000) == 0 -> MCU is in its initial state, done
loop 50 times:
    read PCI cfg 0x48C
    if (value & 0x000F_0000) == 0 -> done
    release ownership to firmware (LPCTL BIT(0))
    delay 50 us
if still not 0:
    the MCU is stuck; run the WFSYS reset-to-initial procedure (§5.1)
```
Used at the very start of bring-up, before ownership is taken. Total budget
50 × 50 µs = 2.5 ms plus 50 configuration reads. [C]

### 6.4 Chip-dead / bus-access failure

* Any MMIO read that returns the sentinel **`0xDEAD_FEED`** triggers a confirmation read of
  chip address **`0x8002_1010`**; if that also returns `0xDEAD_FEED` the chip is declared
  dead, the sticky "bus access failed" flag is latched and the graded recovery is entered
  with reason *CR access fail*. [C]
* Any MMIO read that returns `0xFFFF_FFFF` for a register **not** on the whitelist of §2.2
  is a PCIe access failure: it latches an access-failed flag, triggers the PCIe debug dump
  (§6.1) and enters the graded recovery with reason *PCIe access fail*. [C]
* A read/write on an address with no static-map entry and no active remap is rejected
  before it reaches the bus. [C]
* All MMIO must be refused while an FLR or an L0 reset is in progress, and after the
  suspend sequence has completed. [C]

### 6.5 "Chip no-ack"

A composite predicate used to abort ownership polling early: true when a reset is in
progress, when the sticky bus-access-failed flag is set, or when the reset-retry counter is
non-zero. [C]

### 6.6 DMA hang caused by a bus glitch

The dedicated detector:

```
1. run the fabric check (§6.2, level 1); if it fails, give up (cannot decide)
2. write 0x7403_0168 = 0xCCCC_0100          # PCIe MAC debug group select
3. write 0x7403_0164 = 0x4F4E_4D4C          # PCIe MAC debug signal select
4. read  0x7403_002C            -> pcie_sts
5. write 0x7C00_E24C = 0x1845_7001          # L1 window 0 -> chip 0x7001_0000
6. read  BAR 0x4_3100 (= chip 0x7001_3100)  -> gals_sts
7. write 0x7C00_E24C = 0x1845_184F          # restore
8. HANG  iff  (pcie_sts & 0x0340_0000) == 0x0240_0000
          AND (gals_sts & 0x0080_00C0) == 0x0080_0080
```
Both operands are worth capturing when the predicate fires. When the condition is met
after a WFSYS reset it is treated as an unrecoverable DMA hang caused by a supply/clock
glitch. `0x7001_3100` is the CB-TOP GALS (bus bridge) status register. [C]

### 6.7 Firmware health monitor

Shared-block field `+0x164` carries a bitmask error code; a non-zero value means the
firmware health monitor tripped:

| Value | Monitor |
|---|---|
| `0x000001` | clock |
| `0x000002` | thermal |
| `0x000004` | power |
| `0x000008` | radio |
| `0x000010` | channel switch |
| `0x000020` | internal state machine |
| `0x000040` | radio probe |
| `0x100000` | security |

The block must be at least `0x300` bytes for this to be valid. [C]

### 6.8 Sub-system error latches ("SER occurred")

Shared-block offsets `+0x10` … `+0x2C` hold eight 32-bit error latches — LMAC error 6
(2 words), LMAC error 7 (2 words), PSE error, PSE error 1, PLE error, PLE error 1. Any
non-zero word means a sub-system error was recorded and forces the watchdog recovery to
escalate to FLR. [C]

Shared-block `+0x00` bits `[5:2]` non-zero means "an L1 (SER) reset happened"; this is
polled before suspend. [C]

---

## 7. PCIe link power management

**MT7932 is operated with no host ASPM, LTR or L1-substate management.**
Bring-up, runtime and reset perform no write to the Link Control register, to the L1 PM
Substates capability or to an LTR value; the only PCI configuration offsets written are
`0x204` and `0x208` (AER, §5.4) and `0x484` (doorbell, §2.5). [C] (absence) That the silicon
does not *need* such management is `[L]` — see the MT7927 comparison below and OQ 8.
Consequently:

* There is no "disable ASPM around register access", no "hold off L1 during DMA" and no
  "keep PCIe link awake" register sequence. Link power state is left entirely to the
  platform's PCIe stack.
* `MT_PCIE_MAC_PM` (`0x7403_0194`) is **never written**. The public `mt7921` driver sets
  `MT_PCIE_MAC_PM_L0S_DIS` (BIT 8) there immediately after its first driver-own; the
  MT7932 path does not, implying L0s does not have to be disabled for correct
  operation. `[L]`
* The driver-own poll loop uses a fixed 1 ms tick with **no** ASPM settling delay, in
  contrast to the public `mt792x` path which inserts a 2–3 ms wait per retry when ASPM
  is enabled. If L1 exit latency on a given platform exceeds 1 ms, the first one or two
  poll iterations after `HOST_CLR_OWN` may read a stale `LPCTL`; the 2048 ms budget and
  the 200 ms re-request absorb this. `[L]`

**Comparison with the MT7927 patch series.** The community MT7927 series disables PCIe
ASPM unconditionally at probe (`mt76_pci_disable_aspm()` on device and parent bridge)
because "CONNINFRA power domain and WFDMA register access are unreliable with PCIe L1
active", and additionally disables runtime PM/deep sleep because `SET_OWN`/`CLR_OWN`
transitions on `LPCTL` crash the Bluetooth firmware on the shared CONNINFRA domain.
**Neither workaround is present here.** MT7932 keeps runtime `SET_OWN`/`CLR_OWN` fully
enabled and leaves link power management entirely to the platform. MT7927 is a CONNAC3
(mt7925-class) part, so the erratum is not necessarily shared; but a new MT7932 driver
should treat "WFDMA register access under L1" as an open risk and be prepared to disable
ASPM if register reads return `0xFFFF_FFFF` or `0xDEAD_FEED` under load. `[U]`

**What does exist instead of ASPM control:**

* **A "keep PCIe awake" doorbell.** PCI config `0x484` (or its MMIO alias `0x7403_1484`)
  is rung to force the chip's always-on logic to service the host: value `0x800` before
  collecting a firmware assert dump, value `0x2000` before hibernating. The same doorbell
  can be used to hold PCIe power up across time-sync operations; that use is optional. [C]
* **A firmware-side deep-sleep control**, driven by a firmware configuration command
  ("keep full power" 0/1) rather than by any register. [C]
* **The WFDMA "needs re-init" dummy CR** (§5.3c) is the mechanism by which the host learns
  that the link/subsystem went to deep sleep and tore WFDMA down.

**AER handling that implies an erratum.** Two things stand out: [C]

1. All uncorrectable AER error classes are **masked** (`cfg 0x208 = 0x0057_F010`) for the
   duration of every function-level reset and every WFSYS reset, and restored to
   `0x0040_0000` afterwards, with the uncorrectable **status** register cleared
   (`cfg 0x204 = 0xFFFF_FFFF`) on the way out. A reset that did not generate spurious
   uncorrectable errors would not need this.
2. AER errors reported while a reset is already in progress must be swallowed: clear
   `cfg 0x204` and `cfg 0x210` and request a slot reset, because such errors are raised by
   the reset activity itself. An uncorrectable status that is non-zero while no reset is in
   progress is a genuine PCIe link error and must escalate to the whole-chip (L0) level.

---

## 8. Graded error recovery

The silicon/firmware pair exposes three recovery levels; all three are usable.

### 8.1 L1 — sub-system error recovery (SER), firmware-driven

Signalling registers:

| Chip address | Name | Direction |
|---|---|---|
| `0x7C02_41F0` | `MCU2HOST_SW_INT_STA` (`CONNAC2X_WPDMA_MCU2HOST_SW_INT_STA` on host WFDMA0) | firmware → host, W1C |
| `0x7C02_41F4` | `MCU2HOST_SW_INT_ENA` | enable, set to `0xFFFF` |
| `0x5400_0108` | `HOST2MCU_SW_INT_SET` (MCU WFDMA0 + `0x108`) | host → firmware |
| ~~BAR `0x4108`~~ | **not used on MT7932** — generic fallback for parts without a software-interrupt hook; see §1.2 | — |

Event bits in `MCU2HOST_SW_INT_STA`. Bits **2, 3, 4, 5** are `[C]` (they are the acted-on
mask); the remaining names are the public `connac` allocation assumed to hold here, `[L]`.
The interrupt section §3.1 carries the same table with per-bit markers.

| Bit | Name |
|---|---|
| 1 | `ERROR_DETECT_STOP_PDMA_WITH_FW_RELOAD` |
| 2 | `ERROR_DETECT_STOP_PDMA` |
| 3 | `ERROR_DETECT_RESET_DONE` |
| 4 | `ERROR_DETECT_RECOVERY_DONE` |
| 5 | `ERROR_DETECT_MCU_NORMAL_STATE` |
| 6 | `ERROR_DETECT_SER_TRIGGER_IN_SUSPEND` |
| 7 | `ERROR_DETECT_SER_DONE_IN_SUSPEND` |
| 10 | `ERROR_DETECT_SER_BUS_HANG` |
| 24–28 | LMAC / PSE / PLE / PDMA / PCIe error |

`ERROR_DETECT_MASK` = bits 2, 3, 4, 5.

Host→MCU response bits written to `HOST2MCU_SW_INT_SET`:

| Value | Meaning |
|---|---|
| `0x01` | host has stopped its PDMA rings |
| `0x02` | host has re-initialised PDMA |
| `0x08` | host has finished SER handling |
| `0x10` | **driver-initiated SER** (host asks firmware to run an L1 recovery) |

Host state machine (states 0→1→2→3→0):

| State | Trigger | Host actions | Response |
|---|---|---|---|
| 0 (idle) | `BIT(2)` stop-PDMA | arm a 10 s SER timer; stop TX/RX rings | write `0x01` |
| 1 | `BIT(3)` reset-done | DMASHDL re-init; re-allocate rings; reset MSDU tokens; re-init rings | write `0x02` |
| 2 | `BIT(4)` recovery-done | — | write `0x08` |
| 3 | `BIT(5)` MCU-normal | cancel the SER timer; re-init beacons; kick command and data rings; drain RX; restart TX/RX | — |

* **What is reset:** MAC/UMAC sub-blocks inside the Wi-Fi subsystem plus host WFDMA. `[L]`
  Firmware drives it; the host only re-initialises DMA rings, MSDU tokens and beacons `[C]`.
  Station/BSS context survives `[L]` (inferred from what the host does *not* re-create).
* **The host may also initiate L1** by writing `0x10` to the host→MCU doorbell. [C]
* If L1 is disabled by configuration, the "bypass L1 reset" path escalates to L0.5 (or L0
  if L0.5 is unavailable) with reason *SER L1 fail*. [C]

### 8.2 L0.5 — Wi-Fi subsystem reset, host-driven

The sequence is driven entirely by the host and advances through five states:
`0` idle, `1` requested, `2` torn down, `3` brought back up, `4` postponed.

```
 1. set state = 1; block firmware-own
 2. set state = 2; stop host TX/RX
 3. tear the host side down: stop packet servicing, release the interrupt, free the DMA
    rings and TX descriptor resources, cancel the SER timeout
 4. reset the hardware:
      - if a kernel-initiated FLR is already in flight: wait on its completion, 2000 ms
      - else: issue a PCIe function-level reset (§5.4)
      - (init-time variant: run the WFSYS reset-to-initial procedure, §5.1, up to 2 tries)
 5. clear the "use IPC ownership" selector
 6. run the "clear own IRQ status" step (§5.2 - MT7922 only, skipped on MT7932)
 7. release ownership to firmware
 8. if a postpone was requested -> state = 4, stop here
 9. set state = 3; restart host TX/RX
10. bring the host side back up: re-acquire the interrupt, re-arm the interrupt paths, run
    the full bring-up sequence (§4.1 from step 2), re-expose the network interfaces
11. on failure: run the DMA-hang-by-glitch detector (§6.6) and raise a fault; then
    set state = 0 and escalate with reason "SER L0.5 fail"
12. on success: state = 0
```

* **What is reset:** the entire Wi-Fi subsystem, the MCU (firmware must be re-downloaded)
  and host WFDMA. The PCIe link and the Bluetooth function survive. [C]
* **What the host must re-initialise:** everything above the bus — rings, MSDU tokens,
  share-info block, firmware images, NIC capability, network address, all 802.11 state.
  The remap windows and the AER mask must be re-programmed. [C]
* **Signalling:** entirely host-side; there is no in-band device notification. [C]

### 8.3 L0 — whole-chip reset / re-probe, host-driven

```
before the reset:  state = 2; mark "L0 resetting" (all MMIO must now be refused);
                   stop TX/RX; release all host-side resources; clear the IPC-ownership
                   selector
  <platform performs the whole-chip reset / function re-enumeration>
after the reset:   clear "L0 resetting"; state = 3; restart TX/RX; re-run the full
                   bring-up; state = 0
```

* **What is reset:** the whole die including the PCIe MAC. Everything must be
  re-initialised from `pci_enable_device` onwards. [C]
* Entered directly on: an AER uncorrectable error while no other reset is running; or as
  the fallback when L1 and L0.5 are both bypassed/unavailable. [C]

### 8.4 Recovery-reason → action selection

Reason codes (ordinal values, in order). The enumeration, the action flags and the selection
rules below are host-side recovery policy and are `[C]` as such; they are not silicon
behaviour.

| # | Reason |
|---|---|
| 0 | unknown |
| 1 | abnormal interrupt processed |
| 2 | **driver-own failure** |
| 3 | firmware assert (dump complete) |
| 4 | firmware assert (dump timeout) |
| 5 | Bluetooth-triggered |
| 6 | OID timeout |
| 7 | command-triggered |
| 8 | **CR access failure** |
| 9 | command/event failure |
| 10 | group-4 null |
| 11 | TX error |
| 12 | RX error |
| 13 | **watchdog timeout** |
| 14 | SER L1 failed |
| 15 | SER L0.5 failed |
| 16 | FLR |
| 17 | **PCIe access failure** |
| 18 | FLR requested by SER |
| 19 | FLR requested by health monitor |
| 20 | JTAG error |
| 21 | driver init failure |
| 22 | suspend-to-RAM failure timeout |

Action flags: `BIT(0)` do core dump, `BIT(1)` prevent power-off, `BIT(2)` do whole-chip
(L0) reset, `BIT(3)` do L0.5 reset, `BIT(4)` do L1 reset.

Selection rules:

| Reason | Preferred action | Fallback chain |
|---|---|---|
| 1 (abnormal interrupt) | L1 | L0.5 → L0 |
| 2 (driver-own fail) | L0.5 | L0 |
| 3 (assert done) | L0.5 if the part supports it | L0 |
| 12 (RX error) | none | — |
| 4, 6, 8, 9, 13, 14, 16, 17, 18, 19, 20, 21, 22 | L0.5 | L0 |
| everything else | L0 | — |

The part's "supports L0.5 reset" capability flag is **set** for MT7932. [C]

Watchdog handling: on a watchdog timeout the host first checks the sub-system error
latches (§6.8); if set the reason becomes *FLR requested by SER*, otherwise it checks the
firmware health monitor (§6.7) and, if set, uses *FLR requested by health monitor*;
otherwise plain *watchdog timeout*. [C]

Driver-own timeout handling: a persistent counter is incremented and recovery is entered
with reason *driver-own failure*; **6 consecutive driver-own timeouts are treated as
fatal**. While the host is suspending, the recovery trigger is suppressed. [C]

---

### 8.5 Out-of-band host→MCU notifications (doorbell `0x7403_1484`)

Three host→firmware notifications do **not** travel over the MCU command ring and are still
delivered when the command path is dead. They are single 32-bit writes to the PCIe-MAC
doorbell register at chip address `0x7403_1484` (static BAR offset `0x0001_1484`). `[C]`

| Value | Meaning | When the host **must** write it |
|---|---|---|
| `0x0000_0800` | "host is collecting the assert dump" | once, on the first `EVENT_ID_ASSERT_DUMP` event of a dump |
| `0x0000_2000` | "host is going away" | immediately before a hibernate or whole-chip (L0) reset, **after** the host has entered the resetting state and stopped its TX/RX rings, and **before** the platform reset is performed |
| `0x8000_0000` | force a firmware exception (host-initiated assert) | only deliberately, and only with the dump-collection path already armed |

The register must not be written once the host has entered the post-suspend state in which
MMIO is refused. `[C]` See §9.5/§9.6 of the firmware-boot section for the coredump flow that
the first two values belong to.

### 8.6 What the firmware is waiting for, and for how long

The L1 (SER) sequence in §8.1 is a *lock-step* protocol: at each of its four checkpoints the
firmware stops and waits for the host's write to the host→MCU software-interrupt register.
The firmware log sites make the sequence explicit — the firmware announces
"MCU interrupt host to stop PDMA TX/RX ring operation", then "host all modules were reset
done", then "host system error recovery done", then "host system can normal TX/RX". `[C]`

Consequences a driver must design for `[C]`:

* **No firmware-side timeout that advances the sequence was found.** `[L]` If the host never
  answers a checkpoint, the firmware stays in that step; the MAC stays reset and no traffic
  flows. The host's own 10 s SER timer is the only escape, and its expiry must escalate to a
  Wi-Fi-subsystem (L0.5) reset — the recovery cannot simply be abandoned.
* **The host must not issue MCU commands between the "stop PDMA" checkpoint and the
  "MCU normal state" checkpoint.** The firmware is not servicing the command ring; the
  commands are not queued, and the host will collect one 10 s response timeout per command.
* **The firmware only runs L1 while the host is active.** `[L]` It logs "L1 reset when host is
  active" and, in other states, "skip L1 reset" `[C]`; the behavioural reading of those log
  sites is the inference. A fault that arises while the host has
  released ownership is therefore reported after the fact through the shared block's
  SER-in-suspend bits (§6.8, §4.3) rather than through the live handshake, and the host must
  check those bits on every resume.
* **Each checkpoint's host work is prescribed, not free-form.** In particular the "reset
  done" checkpoint requires the DMA-scheduler re-initialisation, ring re-allocation, MSDU
  token-pool reset and ring re-initialisation to have *completed* before the acknowledgement
  is written; acknowledging early lets the firmware restart the MAC against rings the host
  has not finished programming.

### 8.7 Level-by-level: what the host must re-establish

| Level | Survives | Host must re-create |
|---|---|---|
| **L1** (SER, firmware-driven) | firmware image, MCU state, BSS contexts, station records, keys, calibration | host WFDMA rings, DMA-scheduler configuration, MSDU token pool, beacon templates; then restart TX/RX. **Do not** re-create BSS/station/key state — it is still there. `[C]` that the host re-creates only the listed objects; `[L]` that firmware-side BSS/station/key state genuinely survives |
| **L0.5** (Wi-Fi subsystem reset) | PCIe link, PCI function, Bluetooth function, host DMA mask | everything above the bus: remap windows, AER mask, rings, token pool, share-info block, ROM patch + RAM firmware download, capability query, calibration/RF provisioning, all 802.11 state `[C]` |
| **L0** (whole chip / re-probe) | nothing | everything from PCI enable onwards `[C]` |

The distinction between L1 and L0.5 is the single most dangerous place to get this wrong:
re-creating BSS and station contexts after an L1 recovery allocates *second* copies in a
firmware that still holds the first ones, exhausting the 5-context and 15-record pools.
`[L]` (the pool sizes themselves are `[C]`)

## 9. Required delays and timeouts (consolidated)

All values are the ones the host actually uses and are `[C]` unless the row says otherwise;
whether each is a silicon requirement or a host margin is not established.

| Delay / timeout | Value | Waiting for |
|---|---|---|
| Remap-selector settle | 2 µs | new L1 window base to take effect |
| Driver-own poll tick | 1000 µs | `LPCTL[2]` to clear |
| Driver-own re-request | 200 ms | re-issue `HOST_CLR_OWN` |
| Driver-own total timeout | 2048 ms | ownership grant |
| Driver-own timeout log throttle | 2000 ms | — |
| WFSYS reset assertion hold | 50 000 µs (50 ms) | `WF_WHOLE_PATH_RST` to propagate |
| WFSYS "sw init done" poll | 100 ms × 2 (3 reads) | `0x7C00_0140 BIT(4)` |
| MTCMOS `VLP_UDS_CTRL` settle | 1000 µs | power domain to switch |
| Own-IRQ-status clear settle (MT7922 only) | 2 ms | — |
| BT notification settle | 1 ms | BT firmware to observe the doorbell |
| MCU-back-to-initial poll | 50 µs × 50 | `cfg 0x48C[19:16]` → 0 |
| FLR completion wait (kernel-initiated) | 2000 ms | `reset_done` notification |
| Wi-Fi function ready/off poll | 5 ms tick, 5000 ms timeout | readiness indication (§2.4) |
| WFDMA idle after disable | 1000 µs × 101 (~101 ms) | `GLO_CFG` busy bits |
| WFDMA idle in suspend | 10 ms tick, 500 ms timeout | shared-block `+0x30 BIT(0)` |
| SER-done-in-suspend | 10 ms tick, 200 ms timeout | shared-block `+0x00 BIT(31)` |
| Pending-L1-reset check before suspend | 40 ms tick, 200 ms timeout | shared-block `+0x00 [5:2]` == 0 |
| PCIe pre-suspend done | 2 ms tick, ~1002 ms (501 ticks) timeout | firmware acknowledgement |
| Reset-state idle wait (suspend entry/exit) | 5 ms tick, 2000 ms timeout | recovery state machine idle |
| Firmware-own retry loop in suspend | 1000–3000 µs per iteration, 500 iterations | firmware to take ownership |
| Secure-boot semaphore acquire | 1000 µs tick, 5 s timeout | semaphore bit |
| Interrupt-enable workaround | 1 ms | undocumented; see §4.4 `[U]` |
| SER watchdog timer | 10 000 ms | firmware to complete an L1 sequence |
| Command-halt detection window | 5001 ms | firmware to consume a command |

---

## 10. MT7932-specific deltas (summary)

The MT7932 hardware-configuration record is byte-identical to the MT7922 record except for
the chip-ID value; every real behavioural difference in the power/reset domain is therefore
selected at run time from the PCI device ID:

**Selected at run time on the PCI device ID** — these four are the whole power/reset-domain
delta between MT7922 and MT7932:

| Behaviour | `0x7922` | `0x7932` (and `0x7923`) |
|---|---|---|
| Wi-Fi ready / power-off indication | MMIO `MT_CONN_ON_MISC[1:0]` | **PCI cfg `0x48C[19:16]` == 2** |
| "Clear own IRQ status" after WFSYS reset (`0x7C00_1620` = 3, +2 ms) | performed | **skipped** |
| MTCMOS hardware-mode sequence | skipped | **performed** |
| Fabric / power-domain check on cfg `0x488` | skipped | **performed** |

**Shared by MT7932 *and* MT7922, and different from the public MT7921/mt7961 records** — do
not mistake these for MT7932-specific behaviour:

| Behaviour | Public MT7921 / mt7961 | MT7922 **and** MT7932 |
|---|---|---|
| Static-map limit | `0x000F_0000` in gen4m `mt7961` (upstream `mt76` uses 1 MiB) | **`0x0010_0000`** |
| MMIO chip-ID verification | performed | **disabled** |

Common to both, and different from the public mt7921/mt792x flow: no `MT_PCIE_MAC_PM`
L0s-disable write; no ASPM handling; a config-space firmware boot-stage field; a
host-memory share-info block used as the ownership status channel; AER masking around
every reset.

---

## Open questions / needs hardware tracing

1. **L1 remap window 0 default.** The static translation table claims BAR `0x04_0000` maps
   chip `0x8800_0000`, but the host's idle selector value is `0x184F`. Determine the
   hardware reset default of `0x7C00_E24C[15:0]` and whether `0x8800_0000` is ever
   reachable without an explicit remap.
2. **Bit meanings of PCI config `0x488`.** Only the required values are known
   (bits 0,1,6,9,25,26 = 1; bit 4 = 0; and for level 2 bits 13,14,15 = 1, bit 10 = 0).
   The functional meaning of each bit (which power domain / which fabric leg) is unknown.
3. **Boot-stage encoding of `0x48C[19:16]`** beyond values 0 and 2 — the intermediate
   values presumably encode ROM/patch/RAM-code stages.
4. **`0x7C00_1620`** — the CONN_INFRA ownership/interrupt-status register cleared with
   value `3` on MT7922 only. Confirm bit meanings and whether MT7932 silently requires the
   same clear under some condition.
5. **BT notification.** Confirm on hardware whether `cfg 0x484 = 1` must precede a Wi-Fi
   FLR/WFSYS reset to keep the Bluetooth function alive, and what the doorbell's remaining
   bits do.
6. **`0x7001_3100` (CB-TOP GALS status)** — full bit map; only the mask `0x0080_00C0`
   and the hang value `0x0080_0080` are known.
7. **PCIe MAC debug mux** — the meaning of the individual signal-select values written to
   `0x7403_0164` and the layout of `0x7403_002C`; only the hang predicate
   `(sts & 0x0340_0000) == 0x0240_0000` is known.
8. **ASPM/L1 behaviour.** MT7932 does no ASPM management at all. Whether the MT7927-class
   erratum ("WFDMA register access unreliable with L1 active", "SET_OWN/CLR_OWN crashes BT
   firmware") applies to this CONNAC2 part must be measured, not assumed.
9. **`MT_PCIE_MAC_PM` (`0x7403_0194`)** — MT7932 never writes it. Verify whether L0s must
   be disabled for reliable operation as it is on the public MT7921.
10. **`0x7C05_3A50` and `0x7C05_3A5C`** share-info mailbox slots — purpose unknown.
11. **`0x8002_1010`** — confirm it is a general liveness register rather than something
    with side effects.
12. **The 1 ms delay before enabling interrupts** — scope and root cause unknown.


---

# MT7932 — WFDMA Engine and DMA Ring Layout

**Scope.** This document specifies the host-facing Wi‑Fi packet DMA engine (WFDMA / "WPDMA")
of the MediaTek MT7932 as seen across the PCIe host interface: the DMA instances and their
chip base addresses, the global configuration register, the complete hardware TX/RX ring
inventory with sizes and purposes, the per-ring control register block, the prefetch-SRAM
programming, the WFDMA-level descriptor formats, the producer/consumer protocol and its
ordering rules, the DMA scheduler (DMASHDL) register block and its initialisation values,
and the DMA enable/disable/idle procedures. MT7932 is a CONNAC2 (mt792x-class) part and is
register-compatible with MT7922; everything below therefore reads against the public
MT7921/MT7922 baseline (`mt76` `mt792x_*`, gen4m `mt7961`), and every place where MT7932's
ring assignment, ring sizes, interrupt bit allocation or DMA-scheduler programming differs
from that baseline is called out explicitly. MAC-level TX/RX descriptor (TXD/TXP/RXD)
contents, interrupt delivery/MSI routing, and firmware download are covered by other
sections; only the DMA-visible parts are described here.

Confidence markers: `[C]` confirmed, `[L]` likely, `[U]` unverified.

---

## 1. WFDMA instances

### 1.1 Host-facing instances

| Instance | Chip base | PCIe BAR offset (static window) | Used on MT7932? |
|---|---|---|---|
| Host WFDMA0 (WPDMA0) | `0x7C02_4000` | `0x000D_4000` | **Yes — the only host packet-DMA instance** `[C]` |
| Host WFDMA1 (WPDMA1) | `0x7C02_5000` | `0x000D_5000` | **Not used.** The part is configured as "single host WFDMA"; the WFDMA1 base is not populated and all per-instance loops terminate after instance 0 `[C]`. Whether the *silicon* implements a second host instance is `[U]` |
| Host DMASHDL | `0x7C02_6000` | `0x000D_6000` | Yes (see §8) `[C]` |
| WFDMA ext-wrap CSR (`WFDMA_EXT_CSR`) | `0x7C02_7000` | `0x000D_7000` | Yes (host config / prefetch-control CSRs) `[C]` |

The static PCIe BAR→chip window maps chip `0x7C02_0000`+`0x10000` to BAR offset `0x000D_0000`,
so all four blocks above are reachable without touching the programmable remap window `[C]`.
This matches the public `mt792x` mapping (`MT_WFDMA0_BASE = 0xd4000`,
`MT_WFDMA_EXT_CSR_BASE = 0xd7000`) exactly.

Instance selection in the register API is by index: instance 0 → base `0x7C02_4000`,
instance 1 → base `0x7C02_5000`. All per-instance registers are at the same offsets from the
respective base. On MT7932 only instance 0 is ever addressed `[C]`.

### 1.2 MCU-facing instances

The MCU (WM) side of the same DMA complex is visible to the host through the static window
and must be known for two reasons: the WFDMA re-initialisation handshake flag lives there,
and the host debug/recovery path reads MCU-side ring state.

| Instance | Chip base | BAR offset | Notes |
|---|---|---|---|
| MCU WFDMA0 | `0x5400_0000` | `0x0000_2000` | Global config at `+0x208`; **WFDMA re-init handshake dummy CR at `0x5400_0120`** `[C]` |
| MCU DMA CSR 1..5 | `0x5500_0000`, `0x5600_0000`, `0x5700_0000`, `0x5800_0000`, `0x5900_0000` | `0x3000`, `0x4000`, `0x5000`, `0x6000`, `0x7000` | 4 KiB each, statically mapped `[C]`; contents not host-programmed |

MCU-side rings the host may observe (base of ring control block) `[C]`:

| MCU ring | Chip address | Function |
|---|---|---|
| WM TX ring 2 | `0x5400_0320` | MCU→ air data / management |
| WM RX ring 1 | `0x5400_0510` | host→MCU command |
| WM RX ring 2 | `0x5400_0520` | host→MCU management |
| WM RX ring 3 | `0x5400_0530` | host→MCU data |
| WM RX ring 4 | `0x5400_0540` | TX-free-done |
| WM RX ring 5 | `0x5400_0550` | RX report |

Register layout inside an MCU ring block is identical to the host block (§4) `[L]`.

**WFDMA re-init handshake (`0x5400_0120`)** `[C]`
* bit 1 = "WFDMA has been initialised". The host **sets** this bit after it has programmed the
  ring registers, and **tests** it after a SER/L1-class reset: if it reads back 0, the whole
  ring register set (bases, counts, indices, prefetch) must be reprogrammed and the host
  interrupt enable re-armed. Same CR and bit as public `MT_WFDMA_DUMMY_CR` /
  `MT_WFDMA_NEED_REINIT`.

---

## 2. WPDMA global configuration register

**Address:** WFDMA*n* base + `0x208` → `0x7C02_4208` for host WFDMA0 `[C]`
(MCU instance: `0x5400_0208`).

Full bitfield map. This is the public CONNAC2 definition, assumed to hold unchanged on MT7932:
`[L]` for the map as a whole. The bits this part actually reads or writes — 0, 1, 2, 3, 6, 8,
12, 15, 21, 26, 27, 28, 30 — and the values written to them are `[C]` (§2.1).

| Bits | Name | Access | Meaning |
|---|---|---|---|
| 0 | `TX_DMA_EN` | RW | Enable TX DMA. Must not be set until every ring's prefetch `MAX_CNT` (§5) has been programmed, **including neighbouring DMA instances**. |
| 1 | `TX_DMA_BUSY` | RO | TX engine busy |
| 2 | `RX_DMA_EN` | RW | Enable RX DMA (same prerequisite as bit 0) |
| 3 | `RX_DMA_BUSY` | RO | RX engine busy |
| 5:4 | `PDMA_BT_SIZE` | RW | AXI burst size: 0=4 DW (16 B), 1=8 DW (32 B), 2=16 DW (64 B), 3=32 DW (128 B) |
| 6 | `TX_WB_DDONE` | RW | 1 = TX engine defers the completion IRQ until the TX descriptor write-back has actually landed on the AXI/host-memory bus; 0 = IRQ on write-back ACK |
| 7 | `BIG_ENDIAN` | RW | Payload / TX-RX-info endianness. 0 = little endian |
| 8 | `DMAD_32B_EN` | RW | Descriptor size: 0 = 16 byte, 1 = 32 byte |
| 9 | `FW_DWLD_BYPASS_DMASHDL` | RW | Bypass DMASHDL resource control for the firmware-download TX ring; must be cleared once download completes |
| 10 | `CSR_WFDMA_DUMMY_REG` | RW | Spare |
| 11 | `CSR_AXI_BUFRDY_BYP` (`FIFO_DIS_CHECK`) | RW | Do not check read-data-FIFO availability before issuing the next AXI read |
| 12 | `FIFO_LITTLE_ENDIAN` | RW | 1 = FIFO-side little endian |
| 13 | `CSR_RX_WB_DDONE` | RW | RX analogue of bit 6 |
| 14 | `CSR_PP_HIF_TXP_ACTIVE_EN` | RW | Legacy packet-processor TX lock |
| 15 | `CSR_DISP_BASE_PTR_CHAIN_EN` | RW | 1 = hardware auto-arranges prefetch-SRAM base pointers as a chain; 0 = host must program `DISP_BASE_PTR` in each ring's `EXT_CTRL` (§5) |
| 17:16 | `CSR_LBK_RX_Q_SEL` | RW | Loopback destination RX ring |
| 19:18 | — | RO | Reserved |
| 20 | `CSR_LBK_RX_Q_SEL_EN` | RW | Force loopback RX ring select |
| 21 | `OMIT_RX_INFO_PFET2` | RW | Second-stage RX-info omission (prefetch path) |
| 23:22 | — | RO | Reserved |
| 24 | `CSR_SW_RST` | RO/W | Software reset of the whole PDMA |
| 25 | `FORCE_TX_EOF` | RW | Force EOF after a PDMA reset |
| 26 | `PDMA_ADDR_EXT_EN` | RW | 0 = 32-bit buffer addresses; 1 = TX descriptor DW3 and RX descriptor DW2 carry the address extension |
| 27 | `OMIT_RX_INFO` | RW | 1 = RX packets end with an EOF instead of a trailing rx_info word. **Must be 1 for the MCU-side (`0x51xx_0xxx`/CPU-DMA) instances.** For normal host Wi‑Fi data it must be 0 |
| 28 | `OMIT_TX_INFO` | RW | 1 = tx_info word is not prepended. **Must be 1 for normal Wi‑Fi operation** (the UMAC does not accept TXINFO) |
| 29 | `BYTE_SWAP` | RW | Byte-swap TX/RX descriptors |
| 30 | `CLK_GATE_DIS` | RW | 1 = disable PDMA clock gating |
| 31 | `RX_2B_OFFSET` | RW | 1 = skip the first two bytes of the RX PBF |

### 2.1 Values programmed on MT7932

The bring-up sequence writes the configuration bits and the two enable bits separately.

**Step 1 — configuration (DMA still stopped).** Read-modify-write of `0x7C02_4208`:

* set: `0x5020_9040` for instance 0 `[C]`
  = `CLK_GATE_DIS` (30) | `OMIT_TX_INFO` (28) | `OMIT_RX_INFO_PFET2` (21) |
    `CSR_DISP_BASE_PTR_CHAIN_EN` (15) | `FIFO_LITTLE_ENDIAN` (12) | `TX_WB_DDONE` (6)
* for instance 1 the set-mask is `0x5820_9040` (adds `OMIT_RX_INFO`, bit 27) `[C]` — present in
  the code but never executed on MT7932, because the record declares no second host instance

**Step 2 — enable.** OR in `0x0000_0005` (`TX_DMA_EN | RX_DMA_EN`) `[C]`.

**Disable path.** AND with `0xE7DF_7FFA`, i.e. clear bits 0, 2, 15, 21, 27, 28 `[C]`, then poll
for idle (§9).

**Stop/resume path** (used by SER, suspend and error recovery) touches only the enable bits:
write back the cached global-config value with bits 0 and 2 cleared (stop) or set (resume) `[C]`.

> **Delta vs public MT7921/MT7922.** Upstream `mt76` and downstream gen4m both additionally set
> `PDMA_BT_SIZE = 3` (128-byte bursts), `FIFO_DIS_CHECK` (bit 11) and `CSR_RX_WB_DDONE` (bit 13)
> — public enable mask `0x5020_B870`. MT7932 leaves all three at
> their reset values (mask `0x5020_9040`) `[C]`. `PDMA_ADDR_EXT_EN` (bit 26) is **not** set on
> MT7932, consistent with the 32-bit DMA mask this part advertises `[C]`; MT7925 does set it.

---

## 3. Complete ring inventory

MT7932 exposes a far larger host ring set than MT7921/MT7922. Two distinct facts must be
kept apart:

* **Register-file extent.** The host's ring table is dimensioned **20 TX (0..19)** and **10 RX
  (0..9)**, with TX control blocks at `0x7C02_4300 + N*0x10`, TX prefetch at
  `0x7C02_4600 + N*4`, RX control blocks at `0x7C02_4500 + N*0x10` and RX prefetch at
  `0x7C02_4680 + N*4`. `[C]` for the table dimension and for the address arithmetic; `[L]`
  that the silicon implements exactly 20/10 — the unused slots (TX 15, TX 19, RX 1, RX 9)
  carry **no register address at all** in that table, so it does not assert those registers
  exist. The read-only `WPDMA_INFO` register at WFDMA base + `0x284`, whose
  `TX_RING_NUMBER[7:0]` / `RX_RING_NUMBER[15:8]` fields would settle it, is **never read** on
  this part `[C]` — see open question 11.
* **Host policy.** Of those, this configuration populates **18 TX rings and 8 RX rings**,
  with the descriptor counts, prefetch depths and buffer sizes tabulated below. The counts,
  sizes and which rings are used are all host choices carried in a per-chip ring table;
  public MT7921/MT7922 hosts populate only 4 TX and 3 RX rings of the same hardware. `[C]`

The only firmware-controlled variation is the capability TLV `0x17` (open question 9 at the
end of this section): it overwrites a command-class **ring-index** field whose static value
is `17` with `18`, moving a command class from hardware TX ring 17 to ring 18. It changes a
ring *index*, not the ring count. `[C]` for the substitution, `[L]` for which command class.

Common properties `[C]`:
* TX and RX descriptors are **16 bytes** (`DMAD_32B_EN = 0`).
* All RX rings use a uniform **2352-byte (`0x930`)** receive buffer. This is a **host sizing
  choice**, not a hardware or firmware requirement: 2352 = `28 + 2312 + 12` is the public gen4m
  host constant (maximum 802.11 MPDU + descriptor headroom + HIF header), and the value is
  carried per ring in the host's ring table. Upstream `mt76` sizes the same silicon's buffers
  at 2048. The engine writes whatever the descriptor's `SDL0` capacity permits.
* Index registers are entry counts, 12 bits wide, and wrap modulo the ring's descriptor count.

**Alignment** (applies to every ring):

| Object | Requirement |
|---|---|
| Descriptor ring | Physically contiguous; base written raw into `CTRL0`. Public MediaTek code for this family requires **8-byte** alignment; **16-byte** (one descriptor) alignment is the safe choice and is what a coherent DMA allocation gives in practice `[L]` |
| TX data buffer (cut-through TXD+TXP record) | 4-byte `[L]` |
| TX command / firmware-download / management buffer | 4-byte `[L]`; command payloads are additionally required to be a multiple of 4 bytes by the MCU interface `[C]` |
| RX buffer | **4-byte minimum** — a misaligned receive buffer is rejected `[C]`. Buffers are 2352 bytes each |

### 3.1 Hardware TX rings

"Producer" is always the host: the host writes descriptors and advances CPU index; the engine
consumes and advances DMA index.

The descriptor counts, buffer sizes, prefetch depths and done-IRQ bits are `[C]`. The
**Function** column's mapping of ring *n* to LMAC access-class queue *n* is `[L]` — it follows
from the DMASHDL identity queue→group map (§8.3) and the descriptor `Q_IDX` rule, not from a
direct statement.

| HW ring | Function | Descriptors | Desc size | Buffer | Prefetch depth | Done-IRQ bit | Notes |
|---|---|---|---|---|---|---|---|
| 0 | Wi‑Fi data, access class 0 (LMAC queue AC00) | 512 | 16 B | 64 B TXD+TXP record | 4 | 4 | `[C]` |
| 1 | Wi‑Fi data, AC01 | 512 | 16 B | 64 B | 4 | 5 | `[C]` |
| 2 | Wi‑Fi data, AC02 | 512 | 16 B | 64 B | 4 | 6 | `[C]` |
| 3 | Wi‑Fi data, AC03 | 512 | 16 B | 64 B | 4 | 7 | `[C]` |
| 4 | Wi‑Fi data, AC10 | 512 | 16 B | 64 B | **12** | 8 | `[C]` |
| 5 | Wi‑Fi data, AC11 | 512 | 16 B | 64 B | 4 | 9 | `[C]` |
| 6 | Wi‑Fi data, AC12 | 512 | 16 B | 64 B | 4 | 10 | `[C]` |
| 7 | Wi‑Fi data, AC13 | 512 | 16 B | 64 B | 4 | 11 | `[C]`/`[U]` — see note |
| 8 | Wi‑Fi data, AC20 | 512 | 16 B | 64 B | **12** | 12 | `[C]` |
| 9 | Wi‑Fi data, AC21 | 512 | 16 B | 64 B | 4 | 13 | `[C]` |
| 10 | Wi‑Fi data, AC22 | 512 | 16 B | 64 B | 4 | 14 | `[C]` |
| 11 | Wi‑Fi data, AC23 | 512 | 16 B | 64 B | 4 | 15 | `[C]` |
| 12 | Wi‑Fi data, AC30 | 512 | 16 B | 64 B | **12** | 16 | `[C]` |
| 13 | Wi‑Fi data, AC31 | 512 | 16 B | 64 B | 4 | 17 | `[C]` |
| 14 | Wi‑Fi data, AC32 | 512 | 16 B | 64 B | 4 | 18 | `[C]` |
| 15 | Wi‑Fi data, AC33 | — | — | — | — | 19 | **present in the register file but not populated** `[C]` |
| 16 | Firmware download | 256 | 16 B | variable, ≤16 KB | 4 | 26 | `[C]` |
| 17 | MCU command (WM) | **24** | 16 B | variable | 4 | 27 | `[C]` |
| 18 | Second command-class ring; selected in place of ring 17 by capability TLV `0x17` (open question 9) | **16** | 16 B | variable | 4 | 30 | `[C]` |
| 19 | — | — | — | — | — | — | not populated `[C]` |

Notes:
* The TX ring number equals the LMAC access-class queue number for rings 0..15 and equals the
  DMASHDL group number (§8). Rings are grouped in fours: {0,1,2,3} = AC0*, {4,5,6,7} = AC1*,
  {8,9,10,11} = AC2*, {12,13,14,15} = AC3* `[L]`.
* **TX-done interrupt bit for ring 7:** the public CONNAC2 map assigns bit 11 to TX ring 7. The
  MT7932 ring-to-interrupt map leaves this ring's done-interrupt mask at zero, so ring 7
  TX-done is never signalled `[C]`; whether the hardware bit exists is `[U]`
  (public `mt792x` defines `HOST_TX_DONE_INT_ENA7 = BIT(11)`, so it almost certainly does).
* Data-ring TX-done interrupts (bits 4..18) are **not enabled** on MT7932. Data completion is
  learned from the TX-free-done event ring (RX ring 3), which returns MSDU tokens `[C]`.
* Rings 16, 17, 18 are the only TX rings whose done interrupt is armed (bits 26, 27, 30) `[C]`.
* The 64-byte TX data "buffer" is the cut-through TXD+TXP record; the frame payload itself is
  not DMA'd by WFDMA on the TX path (the MAC fetches it via the MSDU token). Firmware-download,
  command and management rings carry the real payload buffer.

> **Delta vs MT7921/MT7922.** Public parts use **two** data TX rings (0, 1) plus TX ring 16
> (FWDL) and TX ring 17 (MCU command), 4 rings total; `mt76` sizes them 2048 / 128 / 256.
> MT7932 populates **15 data rings (0..14)**, adds a third command-class ring (**18**),
> and uses much smaller command rings (24 and 16 descriptors) and a 256-entry
> firmware-download ring `[C]`. TX ring 18 and its done bit 30 exist in public `mt792x` headers
> (`HOST_TX_DONE_INT_ENA18`) but are unused by MT7921/MT7922.

### 3.2 Hardware RX rings

"Producer" is the hardware: the host posts empty buffers and advances the CPU index; the engine
fills them and advances the DMA index. All rings are host-refilled and host-consumed.

| HW ring | Function | Descriptors | Desc size | Buffer size | Prefetch depth | Done-IRQ bit | Parsed as |
|---|---|---|---|---|---|---|---|
| 0 | Initial / pre-firmware MCU event ring | 32 | 16 B | 2352 B | 4 | 0 | event `[C]` |
| 1 | — | — | — | — | — | (1) | not populated `[C]` |
| 2 | Wi‑Fi data RX (band 0 + band 1 merged) | **512** | 16 B | 2352 B | **8** | 2 | data `[C]` |
| 3 | TX-free-done / MSDU-token return (band 0 + band 1) | 32 | 16 B | 2352 B | 4 | 3 | data-path `[C]` |
| 4 | Firmware event ring (post-download MCU events, delivered via PSE) | 32 | 16 B | 2352 B | 4 | **22** | event `[C]` |
| 5 | Coredump event ring | 224 | 16 B | 2352 B | 4 | **23** | event `[C]` counts/bit, `[L]` role |
| 6 | Firmware-log event ring | 128 | 16 B | 2352 B | 4 | **19** | event `[C]` counts/bit, `[L]` role |
| 7 | Low-latency Wi‑Fi (LLW) ring | **512** | 16 B | 2352 B | 4 | **25** | data-path `[C]` counts/bit, `[L]` role |
| 8 | Management-frame RX ring | 64 | 16 B | 2352 B | 4 | **31** | event `[C]` counts/bit, `[L]` role |
| 9 | — | — | — | — | — | — | not populated `[C]` |

* **RX event-ring switch.** Before firmware is running, MCU events arrive on **RX ring 0**.
  Once firmware download completes, the host must switch its event port to **RX ring 4**; RX
  ring 0 is then idle `[C]`. This is the same "RX event from PSE" behaviour as gen4m's
  `CFG_SUPPORT_HOST_RX_WM_EVENT_FROM_PSE`, but on MT7932 it is unconditional.
* RX rings 2, 3 and 7 are processed through the RX-data path (RXD parsing); rings 0, 4, 5, 6, 8
  through the event path `[C]`.
* RX ring 8 uses a distinct buffer class from the other event rings `[C]`; functionally it is
  the management-frame receive ring `[L]`.
* The **Function** column above is `[C]` for rings 0, 2, 3 and 4 (each is used for that purpose
  by an identified code path) and `[L]` for rings 5–8, whose names come from the buffer class
  and the parse path they feed rather than from an observed traffic assignment — see open
  question 8 and §12 open question 7 of the interrupt section.

> **Delta vs MT7921/MT7922.** Public parts use RX rings 0 (MCU event), 2 (data) and 4 (MCU
> event after firmware download) only — `mt76`'s `MT_INT_RX_DONE_WM/DATA/WM2` = bits 0, 2, 22.
> MT7932 adds **RX ring 3 (TX-free-done), 5 (coredump), 6 (firmware log), 7 (low-latency data)
> and 8 (management)**, with done-interrupt bits **3, 23, 19, 25 and 31** `[C]`. Bits 19, 25 and
> 31 are **not documented in any public MediaTek header** and are new information. Note also that
> the community MT7927 patch series assigns RX rings 4/6/7 to bits 12/14/15 — MT7932 does **not**
> follow that assignment.
> Ring sizes also differ: `mt76` uses 1536 data / 8 MCU / 512 MCU-WA descriptors on MT7921.

### 3.3 Ring-count summary

| | MT7932 `[C]` | MT7921/MT7922 (public) |
|---|---|---|
| Host TX rings populated | 18 (0–14, 16, 17, 18) | 4 (0, 1, 16, 17) |
| Host RX rings populated | 8 (0, 2–8) | 3 (0, 2, 4) |
| Host WFDMA instances | 1 | 1 |
| Descriptor size | 16 B | 16 B |
| RX buffer size (host choice) | 2352 B (`0x930`) | 2048 B in `mt76` (`MT_RX_BUF_SIZE`); 2352 B in gen4m (`CFG_RX_MAX_PKT_SIZE`) |

---

## 4. Per-ring register block

### 4.1 TX rings

* Ring 0 control block base: **`0x7C02_4300`**
* Stride: **`0x10`** bytes per ring
* Ring *N* control block: `0x7C02_4300 + N*0x10` (ring 18 → `0x7C02_4420`) `[C]`
* Ring 0 extended-control register: **`0x7C02_4600`**
* Extended-control stride: **`0x04`**; ring *N* → `0x7C02_4600 + N*4` (ring 18 → `0x7C02_4648`) `[C]`

| Offset | Register | Contents |
|---|---|---|
| `+0x00` | `CTRL0` — descriptor base | Bits 31:0 = physical address of descriptor 0 `[C]` |
| `+0x04` | `CTRL1` — count / base extension | Bits 11:0 = number of descriptors (`MAX_CNT`); **bits 19:16 = descriptor-ring base address bits 35:32** `[C]` |
| `+0x08` | `CTRL2` — CPU index (`CIDX`) | Bits 11:0, entry index, wraps modulo `MAX_CNT` `[C]` |
| `+0x0C` | `CTRL3` — DMA index (`DIDX`) | Bits 11:0, read-only to the host, entry index `[C]` |
| ext `+0x00` | `EXT_CTRL` | Bits **7:0** = prefetch depth (`DISP_MAX_CNT`); bits 31:16 = prefetch-SRAM base pointer (`DISP_BASE_PTR`) `[C]` |

Concrete first-ring addresses `[C]`:
`0x7C02_4300` (base), `0x7C02_4304` (count + base-ext), `0x7C02_4308` (CIDX),
`0x7C02_430C` (DIDX), `0x7C02_4600` (ext ctrl).

### 4.2 RX rings

* Ring 0 control block base: **`0x7C02_4500`**, stride `0x10`; ring *N* → `0x7C02_4500 + N*0x10`
  (ring 8 → `0x7C02_4580`) `[C]`
* Ring 0 extended control: **`0x7C02_4680`**, stride `0x04`; ring *N* → `0x7C02_4680 + N*4`
  (ring 8 → `0x7C02_46A0`) `[C]`

Field layout is identical to the TX block (base / count+base-ext / CIDX / DIDX / ext ctrl) `[C]`.
Concrete first-ring addresses: `0x7C02_4500`, `0x7C02_4504`, `0x7C02_4508`, `0x7C02_450C`,
`0x7C02_4680`.

### 4.3 Index semantics

* `CIDX` and `DIDX` are **entry indices, not byte offsets**, 12 bits wide, and wrap modulo the
  ring's descriptor count as programmed in `CTRL1[11:0]` `[C]`.
* TX: the ring is empty when `CIDX == DIDX`; the host advances `CIDX` after writing a descriptor.
* RX: the host initialises `CIDX = MAX_CNT − 1` at ring init, i.e. it hands the engine all but
  one descriptor `[C]`. The engine advances `DIDX` as it fills buffers.
* There is **no software→hardware ring remapping** on MT7932: the software ring index equals the
  hardware ring number for both directions; the ring-index remapping that other CONNAC2
  parts apply is an identity mapping here `[C]`.

### 4.4 Address extension

The 4-bit field `CTRL1[19:16]` extends the **descriptor-ring base** to 36 bits `[C]`. MT7932
advertises a 32-bit DMA mask, so this field is programmed to 0 in practice `[C]`. Buffer
pointers inside descriptors are 32-bit unless global-config bit 26 (`PDMA_ADDR_EXT_EN`) is set;
it is not set on MT7932 (§2.1).

> **Delta vs `mt76`.** Upstream writes the ring size to `CTRL1` without the base-extension
> nibble. MT7932 (like newer gen4m chips) requires the combined write
> `CTRL1 = ((addr_hi & 0xF) << 16) | (count & 0xFFF)` `[C]`.

---

## 5. Prefetch configuration

Each ring owns a window in an on-chip prefetch SRAM, described by its `EXT_CTRL` register:

```
EXT_CTRL = (DISP_BASE_PTR << 16) | (PREFETCH_DEPTH & 0xFF)
```

The depth field is **8 bits** (`DISP_MAX_CNT[7:0]`), not 12: the host loads it as a byte and
the public CONNAC2 definition sizes it the same way. `[C]`

`DISP_BASE_PTR` is expressed in the same units as `depth * 0x10`, i.e. a ring of depth *d*
occupies `d * 0x10` units and the next ring's base is the running sum `[C]`. This is the same
`PREFETCH(base, depth)` encoding as public `mt792x`.

**Programming order and rules** `[C]`:

1. Clear global-config bit 15 (`CSR_DISP_BASE_PTR_CHAIN_EN`) so the hardware does not
   auto-arrange the windows.
2. Program every **RX** ring's `EXT_CTRL` in ascending ring order, then every **TX** ring's, with
   a single monotonically increasing base accumulator. Rings that are not populated are skipped
   and consume no prefetch space.
3. Write `0xFFFF_FFFF` to `WPDMA_RST_DTX_PTR` (`0x7C02_420C`) to reset all TX DMA pointers.
4. Only then enable TX/RX DMA.

Because the accumulator is shared across both directions and never rewinds, the windows are
disjoint by construction.

**Exact MT7932 values** `[C]`:

| Ring | `EXT_CTRL` address | Value | Base | Depth |
|---|---|---|---|---|
| RX 0 | `0x7C02_4680` | `0x0000_0004` | `0x000` | 4 |
| RX 2 | `0x7C02_4688` | `0x0040_0008` | `0x040` | 8 |
| RX 3 | `0x7C02_468C` | `0x00C0_0004` | `0x0C0` | 4 |
| RX 4 | `0x7C02_4690` | `0x0100_0004` | `0x100` | 4 |
| RX 5 | `0x7C02_4694` | `0x0140_0004` | `0x140` | 4 |
| RX 6 | `0x7C02_4698` | `0x0180_0004` | `0x180` | 4 |
| RX 7 | `0x7C02_469C` | `0x01C0_0004` | `0x1C0` | 4 |
| RX 8 | `0x7C02_46A0` | `0x0200_0004` | `0x200` | 4 |
| TX 0 | `0x7C02_4600` | `0x0240_0004` | `0x240` | 4 |
| TX 1 | `0x7C02_4604` | `0x0280_0004` | `0x280` | 4 |
| TX 2 | `0x7C02_4608` | `0x02C0_0004` | `0x2C0` | 4 |
| TX 3 | `0x7C02_460C` | `0x0300_0004` | `0x300` | 4 |
| TX 4 | `0x7C02_4610` | `0x0340_000C` | `0x340` | 12 |
| TX 5 | `0x7C02_4614` | `0x0400_0004` | `0x400` | 4 |
| TX 6 | `0x7C02_4618` | `0x0440_0004` | `0x440` | 4 |
| TX 7 | `0x7C02_461C` | `0x0480_0004` | `0x480` | 4 |
| TX 8 | `0x7C02_4620` | `0x04C0_000C` | `0x4C0` | 12 |
| TX 9 | `0x7C02_4624` | `0x0580_0004` | `0x580` | 4 |
| TX 10 | `0x7C02_4628` | `0x05C0_0004` | `0x5C0` | 4 |
| TX 11 | `0x7C02_462C` | `0x0600_0004` | `0x600` | 4 |
| TX 12 | `0x7C02_4630` | `0x0640_000C` | `0x640` | 12 |
| TX 13 | `0x7C02_4634` | `0x0700_0004` | `0x700` | 4 |
| TX 14 | `0x7C02_4638` | `0x0740_0004` | `0x740` | 4 |
| TX 16 | `0x7C02_4640` | `0x0780_0004` | `0x780` | 4 |
| TX 17 | `0x7C02_4644` | `0x07C0_0004` | `0x7C0` | 4 |
| TX 18 | `0x7C02_4648` | `0x0800_0004` | `0x800` | 4 |

Total prefetch SRAM consumed: `0x840` units (end of TX ring 18's window) `[C]`. TX rings 15 and
19 and RX rings 1 and 9 are skipped entirely.

> **Delta vs MT7921/MT7922.** Public parts program 5 RX + 9 TX windows all at depth 4, ending at
> `0x3C0`. MT7932 programs 8 RX + 18 TX windows, gives RX ring 2 depth 8 and TX rings 4/8/12
> depth 12, and consumes `0x840` `[C]`. The `WFDMA_EXT_CSR` prefetch-control registers
> (`0x7C02_7030`, `0x7C02_70F0`..`0x7C02_70FC`) used by MT7925/MT7927 are **not** written on
> MT7932 `[C]`.

**Caution / open point.** Bit 15 (`CSR_DISP_BASE_PTR_CHAIN_EN`) is cleared before the per-ring
windows are programmed but is set again by the subsequent global-config write that enables DMA
(§2.1). The resulting steady state therefore has manually-programmed base pointers *and* the
hardware chain-arrangement bit asserted. Public gen4m behaves identically, so this is either
harmless (the chain bit only affects auto-calculation at ring-enable time) or the manual bases
are simply ignored and only the depth fields matter. `[U]` — needs hardware tracing.

---

## 6. WFDMA-level descriptor formats

Descriptors are 16 bytes, little-endian, 4 double-words `[C]` (16 is the descriptor size the
ring allocators are given). `DMAD_32B_EN` (global-config bit 8) is therefore 0 — but the host
**never writes bit 8**: the enable OR-mask leaves it clear and the disable AND-mask preserves
it, so this rests on the register's reset value. `[L]`

### 6.1 TX descriptor

The field map is the public CONNAC2 WFDMA descriptor definition, assumed to hold here: `[L]`.
The fields this part actually writes and reads — `SDP0`, `SDL0`, `LAST_SEC0`, `SDP1`,
`DMA_DONE` — and the values it writes are `[C]` (bullets below).

| DW | Bits | Field | Meaning |
|---|---|---|---|
| 0 | 31:0 | `SDP0` | Segment 0 buffer physical address, low 32 bits |
| 1 | 13:0 | `SDL1` | Segment 1 length (bytes) |
| 1 | 14 | `LAST_SEC1` | Segment 1 is the last segment of the frame |
| 1 | 15 | `BURST` (`M_DONE` in `mt76`) | Burst / multi-descriptor marker |
| 1 | 29:16 | `SDL0` | Segment 0 length (bytes) |
| 1 | 30 | `LAST_SEC0` | Segment 0 is the last segment |
| 1 | 31 | `DMA_DONE` | Written by hardware when the descriptor has been consumed; cleared by the host when posting |
| 2 | 31:0 | `SDP1` | Segment 1 buffer physical address, low 32 bits |
| 3 | 15:0 | `SDP0_EXT` | Segment 0 address bits 47:32 (only meaningful when `PDMA_ADDR_EXT_EN` = 1) |
| 3 | 31:16 | `SDP1_EXT` | Segment 1 address extension |

* **Scatter-gather:** a maximum of **2 segments per descriptor** (`SDP0`/`SDP1`); longer frames
  are chained across descriptors using `BURST` and terminated with `LAST_SEC*` `[C]`.
* **How MT7932 uses it** `[C]`:
  * Wi‑Fi data rings (0..14): a single segment, `SDL0 = 0x40` (64 bytes = the cut-through
    TXD+TXP record), `LAST_SEC0 = 1`, `SDP1 = 0`, `SDP0_EXT = 0`, `DMA_DONE` cleared. The frame
    body is not DMA'd here.
  * Command / firmware-download / management rings (16, 17, 18): single segment, `SDL0` = the
    real payload length, `LAST_SEC0 = 1`, `SDP1 = 0`.
* Maximum representable segment length is 16383 bytes (14-bit field).

### 6.2 RX descriptor

As for the TX form: the field map is the public CONNAC2 definition `[L]`; the fields written
and tested by the host (`SDP0`, `SDL0`, `LAST_SEC0`, `DMA_DONE`, `SDP1`) are `[C]`.

| DW | Bits | Field | Meaning |
|---|---|---|---|
| 0 | 31:0 | `SDP0` | Receive buffer physical address, low 32 bits |
| 1 | 13:0 | `SDL1` | Segment 1 received length |
| 1 | 14 | `LAST_SEC1` | — |
| 1 | 15 | `BURST` | — |
| 1 | 29:16 | `SDL0` | On post: buffer capacity. On completion: bytes written |
| 1 | 30 | `LAST_SEC0` | **Set by hardware when this descriptor holds the last fragment of a packet.** Cleared ⇒ the packet continues in the next descriptor (scatter RX) |
| 1 | 31 | `DMA_DONE` | Set by hardware when the buffer has been filled; cleared by the host on refill |
| 2 | 31:0 | `SDP1` | Second buffer pointer / address-extension word when `PDMA_ADDR_EXT_EN` = 1. Written as 0 by the host `[C]` |
| 3 | 27:0 | `RX_INFO` | Hardware-written receive info |
| 3 | 31:28 | `MAGIC_CNT` | Rolling generation counter `[L]` |

Host-side descriptor initialisation for RX `[C]`:
`DW0 = buffer_pa_lo`, `DW1 = (DW1 & 0x4000FFFF) | ((buf_size & 0x3FFF) << 16)` — i.e. programme
`SDL0`, preserve `LAST_SEC0`, and clear `DMA_DONE`; `DW2 = 0`.
`buf_size` is `0x930` on all rings.

---

## 7. Ring operation

### 7.1 TX submit ("kick")

Per descriptor, serialised per ring `[C]`:

1. Check `CIDX < MAX_CNT` and that the ring has free space.
2. Fill the descriptor: `DW3[15:0] = 0`, `DW1 = (len << 16) | LAST_SEC0`, `DW2 = 0`,
   `DW0 = buffer_pa_lo`. `DMA_DONE` (bit 31) is left cleared.
3. `CIDX = (CIDX + 1) mod MAX_CNT`.
4. **Data memory barrier (inner-shareable, full)** — this is mandatory: the descriptor stores
   must be visible to the device before the doorbell.
5. Write `CIDX` to the ring's `CTRL2` register. This single MMIO write is the doorbell; there is
   no separate kick register.

### 7.2 TX completion

* The **descriptor `DMA_DONE` bit in host memory** is the completion indicator, not the DMA index
  register. Software walks forward from its own "oldest outstanding" index while
  `DW1 & BIT(31)` is set, clears the bit, releases the buffer, and issues a barrier between
  descriptors `[C]`.
* For rings 16, 17 and 18 completion is additionally signalled by the per-ring TX-done interrupt
  (status bits 26, 27, 30 in `0x7C02_4200`) `[C]`.
* For data rings 0..14 there is **no TX-done interrupt**; the host learns of transmit completion
  from the TX-free-done / MSDU-token-return events delivered on **RX ring 3** `[C]`.

### 7.3 RX receive and refill

1. Read the descriptor at `(CIDX + 1) mod MAX_CNT` and poll its `DMA_DONE` bit (barrier before
   each read) `[C]`.
2. Received length = `DW1[29:16]`, clamped to the 2352-byte buffer size. `DW1[30]`
   (`LAST_SEC0`) clear means the packet is segmented and continues in the following descriptor;
   the host reassembles by concatenating fragments until a descriptor with `LAST_SEC0` set `[C]`.
   Reassembly is gated on a host-side option; with the option off, a descriptor with
   `LAST_SEC0` clear is dropped instead. A reassembled record longer than `0xF80` = 3968 bytes
   is abandoned `[C]`. (The RX-descriptor section §8 states the same rule.)
3. Swap in a fresh buffer, then rewrite the descriptor:
   `DW0 = new_pa_lo`, `DW2 = 0`,
   `DW1 = (DW1 & (BIT(30) | 0xFFFF)) | (buf_size << 16)` — which clears `DMA_DONE` `[C]`.
4. Advance the software CPU index.
5. After the batch, write the accumulated `CIDX` **once** to the ring's `CTRL2` register `[C]`.
   Refill is therefore batched: one MMIO write per interrupt/poll pass, not per packet.

At ring start-up the host may wait up to 5 × 20 µs for `DMA_DONE` on the head descriptor,
and up to 3 × 20 µs in the general receive path `[C]` (host-chosen bounds).

### 7.4 Index write-back to host memory

MT7932 **does not use** any DMA-index write-back mode: the DMA index is always
read from the ring's `CTRL3` MMIO register (and in the fast path is not read at all, because
`DMA_DONE` in host memory is authoritative) `[C]`. The CONNAC2/CONNAC3 "WFDMA write-back / EMI
index" feature present in newer gen4m trees is absent from this part's programming `[C]`; whether
the silicon implements it is `[U]`.

### 7.5 Ordering / barrier requirements (summary)

| Point | Requirement |
|---|---|
| Before writing `CIDX` on TX | Full data memory barrier after the descriptor stores `[C]` |
| Between successive descriptor reads when scanning for `DMA_DONE` | Data memory barrier `[C]` |
| Before writing `CIDX` on RX refill | Descriptor rewrite (including `DMA_DONE` clear) must be visible first `[C]` |
| Ring base/count writes | Must complete before `TX_DMA_EN`/`RX_DMA_EN` are set `[C]` |
| Prefetch `EXT_CTRL` writes | Must complete for **all** rings of **all** DMA instances before either enable bit is set (explicit hardware requirement in the global-config definition) `[C]` |

### 7.6 Related WFDMA registers used during ring operation

| Address | Function | Value written on MT7932 |
|---|---|---|
| `0x7C02_4200` | Host interrupt status (write-1-to-clear) | — `[C]` |
| `0x7C02_4204` | Host interrupt enable | `0xEEC8_000D` in the normal case (RX rings 0,2,3,6,4,5,7,8 = bits 0,2,3,19,22,23,25,31; TX rings 16,17,18 = bits 26,27,30; MCU→host software interrupt = bit 29). While the driver is still in its pre-firmware-ready state the constant term drops bits 26/27, becoming `0x6000_0000` `[C]` |
| `0x7C02_41F4` | MCU→host software-interrupt enable (instance 0; instance 1 at `0x7C02_51F4`) | `0x0000_FFFF` `[C]` |
| `0x7C02_420C` | `WPDMA_RST_DTX_PTR` — reset TX DMA pointers | `0xFFFF_FFFF` at end of prefetch programming `[C]` |
| `0x7C02_4280` | `WPDMA_RST_DRX_PTR` — reset RX DMA pointers | not written on MT7932 `[C]` |
| `0x7C02_4100` | `WPDMA_HIF_RST` | write `0`, then `0x30` (bit 4 = logic reset, bit 5 = DMASHDL reset) `[C]` |
| `0x7C02_4298` | `WPDMA_INT_RX_PRI_SEL` | `0x0000_000C` written in the WPDMA-config path when MSI is enabled — RX rings 2 and 3 marked high-priority `[C]`. The interrupt-coalescing path instead read-modify-writes it: OR `0xC` to enable, AND `~0x4` to disable, so **bit 3 is never cleared once set** `[C]` |
| `0x7C02_42E8` | Per-RX-ring delayed-interrupt configuration | `0x01FD_0032` to enable (upper half `0x01FD` = the eight populated RX rings 0,2..8; lower half `0x0032` = 50, the delay parameter), `0` to disable `[C]`. Not present in public headers |
| `0x7C02_42F0` | `WPDMA_PRI_DLY_INT_CFG0` | `0x8032_800A` written in the WPDMA-config path when MSI is enabled `[C]`; the delayed-interrupt set/stop paths rewrite it from the programmable form `0x8032_0000 \| (en << 15) \| ((cnt & 0x7F) << 8) \| time` and restore the same constant when the boost mode stops `[C]`. `mt76` writes 0 here |
| `0x7C02_7030` | `WFDMA_HOST_CONFIG` | bit 9 (`pcie_dly_rx_int_en`) set/cleared together with interrupt coalescing `[C]` |
| `0x7C02_7038` | `WFDMA_EXT_WRAP` host configuration | `0x0000_0013` at DMA enable, when MSI is enabled `[C]`; field meaning `[U]` |

---

### 7.7 Ring-level back-pressure — what the host learns, and when

No congestion report from the WFDMA block is used. Every back-pressure signal the host acts on
is either an index gap it computes itself or an MCU event; no "ring full", "buffer starved" or
"credit exhausted" interrupt is enabled or serviced. `[C]` (absence). That the block provides
no such indication *at all* is `[L]` — bits 20, 21, 24 and 28 of the host interrupt status are
permitted by the enable mask and unaccounted for (interrupt section, open question 2). A driver
must build its own detection for each of the following. `[C]` unless marked.

| Resource | How the host learns it is exhausted | What is *not* reported |
|---|---|---|
| A TX ring (data or command) | free count = `(MAX_CNT − 1) − ((CIDX − DIDX) mod MAX_CNT)` computed by the host before each submission | no interrupt, no status bit. Over-running `CIDX` past `DIDX` silently overwrites descriptors the engine has not consumed |
| MSDU token pool | the host's own accounting reports it empty | nothing; the firmware never sees the shortage |
| Firmware packet buffer (PSE/PLE pages) | indirectly: the DMA scheduler stops consuming descriptors from the affected group, so the TX ring backs up `[L]` | no event, no readable free-page count on the host path. The per-group quotas of §8 are the only control |
| Per-station / per-BSS transmit credit | `STA_UPDATE_FREE_QUOTA` and `BSS_ABSENCE_PRESENCE` events (see the MCU-protocol and MAC-control sections) | these are the *only* firmware-originated transmit credits; they are unsolicited and are not retransmitted if lost |
| An RX ring running out of refilled descriptors | nothing on the host side — the hardware drops the frame `[L]` | the drop is invisible except in the MIB counters ("RX dropped for FIFO full") read through the MCU `[L]` |

Two derived rules `[C]`:

* **The receive path must be serviced even when the host has nothing to receive**, because
  transmit completion (the TX-free / MSDU-token report) travels on it. A driver that
  throttles or defers receive processing under load throttles its own transmit path with it.
* **A single stalled group stalls everything eventually.** The DMA scheduler's per-group
  page quotas (§8.3, §8.4) bound how much one traffic class can occupy, but the free-page
  pool is shared; a class that never drains — frames queued for an absent BSS, or for a
  power-saving peer whose credit the host ignored — consumes pages until the frames hit
  their transmit lifetime, and every other class is squeezed in the meantime.

## 8. DMA scheduler (DMASHDL)

### 8.1 What it is

DMASHDL is the page-credit arbiter that sits between the host TX rings and the PLE/PSE packet
buffers. Each host TX ring's traffic is accounted against a **group**; a group has a minimum
(reserved) and maximum page quota, an optional automatic refill, and a priority. When a group is
out of credit the DMA engine stops fetching from the corresponding ring, which is how per-access-
class back-pressure is implemented without host involvement.

### 8.2 Register block

**Base: `0x7C02_6000`** (BAR offset `0x000D_6000`) `[C]` — same base as public
`MT_DMA_SHDL(ofs)` and as gen4m's `WF_HIF_DMASHDL_TOP_BASE`. **Every** DMASHDL access on this
part — the initialisation writes below and the read-only status/counter reads of the
diagnostic path alike — uses this host-view base `[C]`.

> **Note on `0x5200_0000`.** The bus descriptor also carries the CONNAC2 constant
> `HIF_DMASHDL = 0x5200_0000` (gen4m `CONNAC2X_HIF_DMASHDL_BASE`). It is not in the static
> PCIe-BAR map and **no host access on MT7932 is made through it** `[C]`. That it is the
> **MCU/AXI-internal-bus view of the same block** is `[L]` — it is the standard CONNAC2
> reading of that constant, but the two views were not shown to alias. Where the chip-delta
> section quotes `0x5200_0000` as "the DMASHDL base" it means that constant, not the address
> the host writes.

| Address | Register | Fields |
|---|---|---|
| `0x7C02_600C` | Scheduler control | The host **clears bits 17:16 and sets bit 16** `[C]`. In the public CONNAC2 definition bit 16 is `PAGE_SETTING.GROUP_SEQUENCE_ORDER_TYPE` and bit **17** is `SLOT_TYPE_ARBITER_CONTROL` `[L]` — i.e. this write selects the group-sequence order type and leaves the slot arbiter **off**. "Slot arbiter" below is the host routine's own name for the step, not the register field |
| `0x7C02_6010` | Refill control | bit `16+n` = **refill *disable*** for group *n* (n = 0..15). Set the bit to disable refill, clear it to enable `[C]` |
| `0x7C02_601C` | Packet max page | bits 11:0 = PLE packet max page; bits 27:16 = PSE packet max page (write mask `0x0FFF_0FFF`) `[C]` |
| `0x7C02_6020 + 4*n` (n = 0..15) | Group *n* quota (public `MT_HIF_DMASHDL_GROUP0_CTRL` = base + `0x20`) | bits 11:0 = **min (reserved) quota**; bits 27:16 = **max quota**; bits 31:28 preserved `[C]`. The array occupies `0x7C02_6020`–`0x7C02_605C`; the read-only status registers are **not** inside it — an earlier version of this table placed them at `0x7C02_6024`, which overlapped group 1's quota `[C]` |
| `0x7C02_6060 + 4*k` (k = 0..3) | Queue→group map | 32 LMAC queues, 4 bits each, 8 queues per register: queue *q* occupies bits `4*(q mod 8)` of register `k = q >> 3` `[C]` |
| `0x7C02_6070 + 4*k` (k = 0..1) | Priority→group map | 16 priority slots, 4 bits each, 8 per register `[C]` |
| `0x7C02_6100`, `0x7C02_6140 + 4*n` (n = 0..15), `0x7C02_6180 + 4*k` | Read-only scheduler status and per-group counters | Read back by the diagnostic path: overall status, per-group page/reserved/source counts and per-group packet counts `[C]` that the diagnostic path reads them. The **offsets** are the public gen4m `MT_HIF_DMASHDL_STATUS_RD` (base + `0x100`), `STATUS_RD_GP0..15` (base + `0x140`..`0x17C`) and `RD_GP_PKT_CNT_*` (base + `0x180`…) — `[L]`, not confirmed for this part |

### 8.3 MT7932 group→queue mapping

The host programmes an **identity map for LMAC queues 0..15** — queue *q* → group *q* `[C]`.
Queues 16..31 keep the CONNAC2 default map `[C]`:

| Queue | 0–2 | 3 | 4–8 | 9–10 | 11–15 |
|---|---|---|---|---|---|
| Group | 0 | 1 | 1 | 0 | 1 |

(queues 16 = ALTX, 17 = BMC, 18 = BCN → group 0; 19–23 reserved → group 1; 24 = NAF,
25 = NBCN, 26 = FIXFID → group 0; 27–31 reserved → group 1) `[C]`.

Because host TX ring *n* carries LMAC queue *n* for *n* = 0..15, group *n* is effectively the
credit pool of host TX ring *n* `[L]`.

### 8.4 MT7932 initialisation values

| Register / field | Value |
|---|---|
| `0x7C02_600C` bits 17:16 | **`01`** — bit 16 set, bit 17 clear `[C]` (see §8.2 on the field naming) |
| PLE packet max page (`0x7C02_601C[11:0]`) | `0x001` `[C]` |
| PSE packet max page (`0x7C02_601C[27:16]`) | `0x000` `[C]` |
| Refill enable, groups 0–14 | **enabled** (bits 16..30 of `0x7C02_6010` cleared) `[C]` |
| Refill enable, group 15 | disabled (bit 31 set) `[C]` |
| Priority→group map | identity: priority *i* → group *i*, *i* = 0..15 `[C]` |

Per-group quotas (`0x7C02_6020 + 4*n`) `[C]`:

| Group | Max quota (pages) | Min quota (pages) | Refill |
|---|---|---|---|
| 0 | 512 (`0x200`) | 40 (`0x28`) | on |
| 1 | 512 | 40 | on |
| 2 | 512 | 40 | on |
| 3 | 512 | 40 | on |
| 4 | 256 (`0x100`) | **20 (`0x14`)** | on |
| 5 | 256 | 40 | on |
| 6 | 256 | 40 | on |
| 7 | 256 | 40 | on |
| 8 | 256 | **20** | on |
| 9 | 256 | 40 | on |
| 10 | 256 | 40 | on |
| 11 | 256 | 40 | on |
| 12 | 256 | **20** | on |
| 13 | 256 | 40 | on |
| 14 | 256 | 40 | on |
| 15 | 0 | 0 | off |

Total reserved (min) pages = 540 `[C]`. Group 15 is left at zero because the command class is
flow-controlled by WFDMA itself, not by DMASHDL (see global-config bit 9).

**Runtime quota update.** The maximum quota of a WMM/access-class set can be changed at runtime:
for WMM index *w*, the four groups `queue_map[4*w + j]`, *j* = 0..3, have their max-quota field
(`bits 27:16`) rewritten; a requested value greater than `0xFFF` means "restore this group's
configured default" `[C]`.

> **Delta vs MT7921/MT7922.** Public gen4m for this family programmes **only groups 0 and 1**
> (max `0xFFF`, min `0x3`, refill on) and leaves groups 2–15 at zero, because the driver has only
> two data TX rings. MT7932 populates **all 15 data groups with finite quotas** and sets bit 16
> of the scheduler-control register `[C]` (see §8.2 — that bit is the group-sequence-order-type
> selector, not the slot arbiter, which stays clear). The queue→group map for queues
> 16–31 is unchanged from the public map; queues 0–15 change from the public
> `{0,0,0,0,1,1,1,1,0,0,0,0,0,0,0,0}` to the identity map `[C]`.

### 8.5 Re-initialisation after reset

Two distinct initialisation paths exist, because firmware backs up and restores part of the
block across a UMAC (L1) reset `[C]`:

| Register class | Full init (cold boot, L0.5 reset, probe) | Re-init (L1 / UMAC reset) |
|---|---|---|
| Packet max page (`0x7C02_601C`) | yes | **yes** |
| Queue→group map (`0x7C02_6060..606C`) | yes | **yes** |
| Priority→group map (`0x7C02_6070..6074`) | yes | **yes** |
| Slot arbiter (`0x7C02_600C`) | yes | **yes** |
| Refill control (`0x7C02_6010`) | yes | **no** — restored by firmware |
| Group quotas (`0x7C02_6020 + 4n`) | yes | **no** — restored by firmware |

Rationale (from the hardware/firmware contract): the quota and refill CRs must be live before
firmware releases the UMAC reset, so firmware saves and restores them; the remaining CRs are
re-programmed by the host to save firmware DLM space.

---

## 9. DMA start, stop and idle

### 9.1 Full ring bring-up sequence

1. Write the global configuration with the enable bits *clear*
   (§2.1 step 1 + disable mask), then poll for idle (§9.3).
2. For every populated **TX** ring *n*: write `CTRL0 = desc_pa_lo`, `CTRL2 (CIDX) = 0`,
   `CTRL1 = ((desc_pa_hi & 0xF) << 16) | (MAX_CNT & 0xFFF)` `[C]`.
3. For every populated **RX** ring *n*: write `CTRL0 = desc_pa_lo`,
   `CTRL2 (CIDX) = MAX_CNT − 1`, `CTRL1 = ((desc_pa_hi & 0xF) << 16) | (MAX_CNT & 0xFFF)`;
   clear `DMA_DONE` in every descriptor `[C]`.
4. Manual prefetch programming (§5), ending with `WPDMA_RST_DTX_PTR = 0xFFFF_FFFF`.
5. Write the global configuration set-mask, then OR in
   `TX_DMA_EN | RX_DMA_EN`; then the chip-specific extras
   (`0x7C02_4298 = 0xC`, `0x7C02_7038 = 0x13`, `0x7C02_42F0 = 0x8032_800A`) `[C]`.
   The three extras are conditional on **MSI being enabled** — they execute on the normal
   path (one MSI vector is requested) and are skipped on the legacy-line-interrupt fallback
   `[C]`.
6. Set the WFDMA re-init flag: `0x5400_0120` bit 1 = 1 `[C]`.
7. Enable host interrupts (`0x7C02_4204`, §7.6) and the MCU software-interrupt enable
   (`0x7C02_41F4 = 0xFFFF`) `[C]`.
8. Enable per-RX-ring delayed interrupts (`0x7C02_42E8 = 0x01FD_0032`) `[C]`.

### 9.2 Stop

* **Soft stop / resume** (SER, suspend): read-modify-write the global config clearing (stop) or
  setting (resume) only bits 0 and 2 `[C]`. Iterated over all populated DMA instances.
* **Full disable**: AND the global config with `0xE7DF_7FFA` (clears `TX_DMA_EN`, `RX_DMA_EN`,
  `CSR_DISP_BASE_PTR_CHAIN_EN`, `OMIT_RX_INFO_PFET2`, `OMIT_RX_INFO`, `OMIT_TX_INFO`), then poll
  for idle `[C]`.
* **HIF reset** (part of L1/SER recovery): write `0` then `0x30` to `WFDMA*n* + 0x100` —
  bit 4 = logic reset, bit 5 = DMASHDL reset `[C]`. Equivalent to public
  `MT_WFDMA0_RST_LOGIC_RST | MT_WFDMA0_RST_DMASHDL_ALL_RST`.

### 9.3 Idle poll

| Step | Action |
|---|---|
| 1 | Read the global configuration register (WFDMA base + `0x208`) |
| 2 | If `value & 0x0000_000A` is zero (`TX_DMA_BUSY` bit 1 and `RX_DMA_BUSY` bit 3 both clear), the instance is idle — stop |
| 3 | Wait 1000 µs and repeat from step 1 |

Bound used on MT7932: **101 iterations × 1000 µs ⇒ ~100 ms timeout** `[C]`. The check is
performed per DMA instance; on MT7932 only instance 0 exists.

Additional busy indication is available in `WFDMA0 + 0x13C` (`BUSY_ENA`: bit 0 = TX FIFO 0,
bit 1 = TX FIFO 1, bit 2 = RX FIFO) — defined in public `mt792x` headers; not required for
the idle check `[C]`.

### 9.4 Post-reset re-initialisation

After an L1/UMAC-class reset the host must:
1. Read `0x5400_0120`; if bit 1 is clear, the DMA context was lost.
2. Re-run the full ring bring-up of §9.1 steps 1–5.
3. Re-arm host interrupts.
4. Re-assert `0x5400_0120` bit 1.
5. Re-run the DMASHDL **re-init** subset (§8.5).

After a cold boot or L0.5 reset the **full** DMASHDL init (§8.4) is required.

---

## 10. Summary of deltas vs public MT7921 / MT7922

| Area | MT7921/MT7922 (public) | MT7932 |
|---|---|---|
| Host data TX rings | 2 (rings 0, 1) | **15** (rings 0–14), one per LMAC access-class queue `[C]` |
| Command-class TX rings | 16 (FWDL), 17 (CMD) | 16 (FWDL, 256), 17 (CMD, **24**), **18 (alternate CMD, 16)** `[C]` |
| Host RX rings | 0, 2, 4 | **0, 2, 3, 4, 5, 6, 7, 8** `[C]` |
| New RX ring functions | — | 3 = TX-free-done, 5 = coredump, 6 = firmware log, 7 = low-latency, 8 = management `[C]` |
| New RX done-interrupt bits | — | ring 6 → **bit 19**, ring 7 → **bit 25**, ring 8 → **bit 31** (undocumented publicly) `[C]` |
| RX buffer size | 2048 B | **2352 B (`0x930`)** on every ring `[C]` |
| Data RX ring depth | 1536 | 512, prefetch depth 8 `[C]` |
| Global-config enable mask | `0x5020_B870` | `0x5020_9040` — burst size, FIFO-check-disable and RX write-back-done left at reset `[C]` |
| Ring count register | size only | size **plus 4-bit descriptor-base address extension in bits 19:16** `[C]` |
| Prefetch windows | 14 windows, all depth 4, `0x000`–`0x3C0` | 26 windows, depths 4/8/12, `0x000`–`0x840` `[C]` |
| DMASHDL groups in use | 2 (max `0xFFF`, min `0x3`) | 15 (max 512/256, min 40/20), scheduler-control bit 16 **set** (not the slot arbiter — §8.2) `[C]` |
| DMASHDL queue→group | `{0,0,0,0,1,1,1,1,0×8}` for queues 0–15 | identity for queues 0–15 `[C]` |
| Static BAR window | 0x000F_0000 (gen4m mt7961) | **0x0010_0000 (1 MiB)** `[C]` |
| Delayed-interrupt CRs | `PRI_DLY_INT_CFG0 = 0` | `PRI_DLY_INT_CFG0` programmed from the runtime form by the coalescing path, plus per-RX-ring delayed interrupt at `0x7C02_42E8 = 0x01FD_0032` at ring init `[C]` |
| Index write-back to host memory | not used | not used `[C]` |

---

## 11. Open questions / needs hardware tracing

1. **`CSR_DISP_BASE_PTR_CHAIN_EN` interaction.** The bit is cleared to program manual prefetch
   bases and then set again by the DMA-enable write. Determine empirically whether the manual
   `DISP_BASE_PTR` values are honoured in that state, or whether only the depth fields matter.
2. **TX ring 7 done-interrupt bit.** Public CONNAC2 assigns bit 11; the MT7932 configuration
   leaves it unmapped. Confirm that bit 11 is functional so a driver can use ring 7 completion.
3. **RX ring 1 and TX rings 15/19.** Present in the register file but never programmed. Confirm
   they are functional and determine what (if anything) firmware expects on them.
4. **`WFDMA_EXT_WRAP` CSR `0x7C02_7038 = 0x13`.** Field meaning unknown; determine whether it is
   required for correct DMA operation or is a tuning knob.
5. **Delayed-interrupt encodings.** Confirm the field split of `0x7C02_42E8`
   (ring mask in `[31:16]`, delay/count in `[15:0]`) and of `0x7C02_42F0`
   (`[15]` enable, `[14:8]` packet count, `[7:0]` time) on real hardware.
6. **DMASHDL status/counter offsets.** The read-only per-group page, reserved, source and
   packet counters are given above at the public gen4m offsets (base + `0x100`, `0x140`,
   `0x180`); confirm them on MT7932 by enumerating `0x7C02_6100`–`0x7C02_61FF`.
7. **Index write-back.** Determine whether the silicon supports writing CPU/DMA indices to host
   memory (the CONNAC2/3 "WFDMA WB / EMI index" feature); it is unused on this part.
8. **Low-latency RX ring 7.** Confirm what traffic firmware steers there and whether it needs a
   separate MAC-level configuration command.
9. **TX ring 18.** The ring is provisioned (16 descriptors) and its done interrupt is armed
   (bit 30), but the command-submission path routes commands to ring 17 by default. Ring 18 is
   brought into use by a **firmware capability TLV** (tag `0x17`, the WFDMA-reallocation
   descriptor): when its selector byte is non-zero the host rewrites a command-class TX ring
   *index* field in the bus descriptor from `17` to `18` `[C]`. Two adjacent fields both hold
   `17` (the MCU-command ring index and the "WA command" ring index, which this part sets
   equal because it has no WA CPU), so which command class moves is `[L]`. Confirm on hardware
   whether MT7932 firmware ever sets the selector, and what the substitution changes.
10. **Whether the smaller command rings (24 / 16 descriptors) are a hardware constraint** or
    simply a host-side sizing choice.
11. **Real TX/RX ring count.** Read `WPDMA_INFO` (WFDMA base + `0x284`, `TX_RING_NUMBER[7:0]`
    / `RX_RING_NUMBER[15:8]`) on live silicon. The 20/10 figure in §3 is the dimension of the
    host's ring table, not a register-file read-out, and the four unused slots carry no
    register address at all.
12. **The MSI-conditional extras.** `0x7C02_4298 = 0xC`, `0x7C02_7038 = 0x13` and
    `0x7C02_42F0 = 0x8032_800A` are written only when MSI is enabled (§9.1 step 5).
    Determine whether a legacy-line-interrupt driver needs any of them.


---

# MT7932 — Interrupt Architecture

## Scope

This document specifies the host-facing interrupt architecture of the MediaTek MT7932
combo Wi-Fi/BT part as seen across the PCIe host interface: the WFDMA0 host interrupt
status/enable registers and their complete bit assignment, the per-ring bit map, the
bidirectional MCU↔host software-interrupt channel, MSI configuration and the
vector→event map (both the 8-vector and the single-vector layouts the part's descriptor
carries), the service discipline the hardware requires, the interrupt-moderation
(delayed / coalescing) registers, the INTx fallback, and the low-power / wake-up
interrupt path. MT7932 presents a CONNAC2 (mt792x-class) PCIe host interface; the
interrupt block is register-compatible with MT7922 and the bit *positions* are shared
with MT7921, but the **bit-to-ring assignment and the aggregate masks differ
substantially from the publicly documented MT7921/MT7922 mt76 driver** — see
§11 (Deltas). All addresses below are chip addresses in the device's flat address space,
reached through the 1 MiB static PCIe-BAR mapping window (or, above it, a programmable
remap window). Confidence markers: `[C]` confirmed, `[L]` likely, `[U]` unverified.

---

## 1. Interrupt register inventory

WFDMA0 host block base: `0x7C02_4000` (host view). The same block is visible from the
on-chip MCU at `0x5400_0000`. WFDMA1 is **not present** on this part (`is_support_wfdma1`
is false), so no `0x7C02_5xxx` / `0x5500_0xxx` alias is ever used. `[C]`

| Chip address | Public (gen4m / mt76) name | Width | Access | Function |
|---|---|---|---|---|
| `0x7C02_4200` | `WF_WFDMA_HOST_DMA0_HOST_INT_STA` / `MT_WFDMA0_HOST_INT_STA` | 32 | R / W1C | Primary host interrupt status. `[C]` |
| `0x7C02_4204` | `WF_WFDMA_HOST_DMA0_HOST_INT_ENA` / `MT_WFDMA0_HOST_INT_ENA` | 32 | R/W | Primary host interrupt enable (mask). `[C]` |
| `0x5400_0200` | same register, MCU view | 32 | R | Alias used only for diagnostics. `[C]` |
| `0x5400_0204` | same register, MCU view | 32 | R | Alias used only for diagnostics. `[C]` |
| `0x7C02_41F0` | `CONNAC2X_WPDMA_MCU2HOST_SW_INT_STA` / `MT_MCU_CMD` | 32 | R / W1C | MCU→host software-interrupt status / reason. `[C]` |
| `0x7C02_41F4` | `CONNAC2X_WPDMA_MCU2HOST_SW_INT_MASK` / `MT_MCU2HOST_SW_INT_ENA` | 32 | R/W | MCU→host software-interrupt enable. `[C]` |
| `0x5400_0108` | `CONNAC2X_WPDMA_HOST2MCU_SW_INT_SET` | 32 | W1S | Host→MCU software-interrupt set (MCU-view address, written by the host). `[C]` |
| `0x7C02_4118` | `WF_WFDMA_HOST_DMA0_HOST_INT_STA_EXT` | 32 | R / W1C | Extended host interrupt status. Present in the CONNAC2 register map; **never accessed on this part**. `[C]` (unused) |
| `0x7C02_7010` / `0x7C02_7014` | `CONNAC2X_WPDMA_EXT_INT_STA` / `_MASK` (ext conn-hif wrap) | 32 | R/W1C, R/W | Second-level WFDMA1 aggregate status/mask. **Never accessed on this part** (WFDMA1 absent). `[C]` (unused) |
| `0x7403_0184` | PCIe MAC interrupt status | 32 | R | **Never read in the interrupt-service path** `[C]`; it appears only in the bus-failure diagnostic dump (PCIe section §7.1). The "interrupt status" role is public gen4m sibling-chip naming (mt7925/mt6639 dump it as "PCIE INT status") — `[U]` for MT7932 |
| `0x7403_0188` | `MT_PCIE_MAC_INT_ENABLE` | 32 | R/W | PCIe-MAC-level master interrupt enable. Programmed to **`0x0000_01FF`** on this part. `[C]` |
| `0x7403_018C` | PCIe MAC interrupt clear | 32 | W1C | PCIe-MAC-level interrupt acknowledge; the host writes the bit mask of the vectors it has serviced. `[C]` address, `[L]` W1C semantics |
| `0x7C06_0010` | `CONNAC2X_BN0_LPCTL` / `MT_CONN_ON_LPCTL` | 32 | R/W | Ownership control: BIT0 = host-set-own (give ownership to FW), BIT1 = host-clear-own, BIT2 = owner-state-sync (1 = firmware owns). `[C]` |
| `0x7C06_0014` | `CONNAC2X_BN0_IRQ_STAT` | 32 | R / W1C | Host-CSR interrupt status. BIT0 = firmware-cleared-own (ownership handed back to host). Address and bit are carried per-chip and are `[C]`; the register's name, its W1C behaviour and the bit's meaning are the public CONNAC2 definition, `[L]`. **Not accessed on MT7932** — see §8.1. |
| `0x7C06_0018` | `CONNAC2X_BN0_IRQ_ENA` | 32 | R/W | Host-CSR interrupt enable; programmed to `BIT(0)` during bring-up. `[C]` |
| `0x7C00_1620` | (conn-infra ownership IRQ status) | 32 | W | Written with `0x3` followed by a 2 ms settle, **only when the PCI device ID is `0x7922`**. Not executed for MT7932. `[C]` |

### Acknowledge semantics

* `0x7C02_4200` (`HOST_INT_STA`) is treated as **write-1-to-clear**: the host writes exactly
  the set of bits it has decided to service, never a read-modify-write. `[C]` for the write
  discipline; `[L]` for the W1C property itself, which is the public CONNAC2 definition and
  was not observed (the register is never read in the service path — see §5 step 2).
* `0x7C02_41F0` (`MCU2HOST_SW_INT_STA`) is likewise **write-1-to-clear**: the host reads
  it and writes the read-back value to acknowledge. `[C]` for the read-then-write-back
  discipline; `[L]` for W1C itself.
* `0x5400_0108` (`HOST2MCU_SW_INT_SET`) is **write-1-to-set**: writing a bit raises the
  corresponding software interrupt inside the MCU; there is no host-side status read for
  this direction. `[C]`
* `0x7C06_0014` (`BN0_IRQ_STAT`) is **write-1-to-clear**: writing `BIT(0)` clears the
  firmware-cleared-own latch. `[L]` — public CONNAC2 convention; the write is not performed on
  this part (§8.1), so the semantics were not observed.
* Status bits are **assumed** to latch independently of the enable register — i.e.
  `HOST_INT_ENA` gates only the assertion of the outbound interrupt, a masked source still
  sets its status bit, and it must still be cleared by a W1C write. `[U]` — nothing in the
  analysed host establishes this. The host never reads `HOST_INT_STA` in the service path at
  all (§5 step 2), so it cannot have observed the behaviour; the acknowledge masks it writes
  are the same either way. Open question 6.
* There is **no atomic set-alias / clear-alias for `HOST_INT_ENA`**. Any partial
  re-enable is a software read-modify-write and must be serialised by the host. `[C]`

---

## 2. Complete bit map — `HOST_INT_STA` (`0x7C024200`) / `HOST_INT_ENA` (`0x7C024204`)

Both registers use identical bit positions. Names follow the public CONNAC2 / mt76
naming. The "ring" column is this part's actual assignment as carried in the device's own
ring-to-interrupt map (this is where MT7932 deviates from public MT7921 code).

| Bit | Mask | Name | Ring / source | Notes |
|---|---|---|---|---|
| 0 | `0x0000_0001` | `rx_done_int_sts_0` | RX ring 0 | MCU event ring. In RX-done aggregate. `[C]` |
| 1 | `0x0000_0002` | `rx_done_int_sts_1` | RX ring 1 | Ring present in hardware but **not enabled** on this part; no source mapped. `[C]` |
| 2 | `0x0000_0004` | `rx_done_int_sts_2` | RX ring 2 | Band-0 RX data. In RX-done aggregate. `[C]` |
| 3 | `0x0000_0008` | `rx_done_int_sts_3` | RX ring 3 | TX-free-done / MSDU-report ring (see §11). In RX-done aggregate. `[C]` |
| 4 | `0x0000_0010` | `tx_done_int_sts_0` | TX ring 0 | Data ring. **Not** in this part's TX-done aggregate. `[C]` |
| 5 | `0x0000_0020` | `tx_done_int_sts_1` | TX ring 1 | Data ring. `[C]` |
| 6 | `0x0000_0040` | `tx_done_int_sts_2` | TX ring 2 | Data ring. `[C]` |
| 7 | `0x0000_0080` | `tx_done_int_sts_3` | TX ring 3 | Data ring. `[C]` |
| 8 | `0x0000_0100` | `tx_done_int_sts_4` | TX ring 4 | Data ring. `[C]` |
| 9 | `0x0000_0200` | `tx_done_int_sts_5` | TX ring 5 | Data ring. `[C]` |
| 10 | `0x0000_0400` | `tx_done_int_sts_6` | TX ring 6 | Data ring. `[C]` |
| 11 | `0x0000_0800` | `tx_done_int_sts_7` | (TX ring 7) | TX ring 7 exists and is enabled, but this part's ring-to-interrupt map assigns it **no interrupt bit**. See Open Questions. `[C]` |
| 12 | `0x0000_1000` | `tx_done_int_sts_8` | TX ring 8 | Data ring. `[C]` |
| 13 | `0x0000_2000` | `tx_done_int_sts_9` | TX ring 9 | Data ring. `[C]` |
| 14 | `0x0000_4000` | `tx_done_int_sts_10` | TX ring 10 | Data ring. `[C]` |
| 15 | `0x0000_8000` | `tx_done_int_sts_11` | TX ring 11 | Data ring. `[C]` |
| 16 | `0x0001_0000` | `tx_done_int_sts_12` | TX ring 12 | Data ring. `[C]` |
| 17 | `0x0002_0000` | `tx_done_int_sts_13` | TX ring 13 | Data ring. `[C]` |
| 18 | `0x0004_0000` | `tx_done_int_sts_14` | TX ring 14 | Data ring. `[C]` |
| 19 | `0x0008_0000` | `rx_done_int_sts_6` | RX ring 6 | **MT7932/MT7922-specific placement.** Not in the RX-done aggregate. `[C]` |
| 20 | `0x0010_0000` | `rx_coherent_int` | RX DMA coherency error | Public CONNAC2 bit; excluded by the per-ring enable mask `0x93CF_FFFF` used on this part. `[C]` position, `[L]` meaning |
| 21 | `0x0020_0000` | `tx_coherent_int` | TX DMA coherency error | As above. `[C]` position, `[L]` meaning |
| 22 | `0x0040_0000` | `rx_done_int_sts_4` | RX ring 4 | Second MCU/event ring. In RX-done aggregate. `[C]` |
| 23 | `0x0080_0000` | `rx_done_int_sts_5` | RX ring 5 | In RX-done aggregate. `[C]` |
| 24 | `0x0100_0000` | (unassigned) | — | Permitted by the per-ring enable mask but no ring maps to it on this part. `[U]` |
| 25 | `0x0200_0000` | `rx_done_int_sts_7` | RX ring 7 | Not in the RX-done aggregate. `[C]` |
| 26 | `0x0400_0000` | `tx_done_int_sts_16` | TX ring 16 | Firmware-download ring. **In TX-done aggregate.** `[C]` |
| 27 | `0x0800_0000` | `tx_done_int_sts_17` | TX ring 17 | MCU command ring. **In TX-done aggregate.** `[C]` |
| 28 | `0x1000_0000` | `subsys_int` (`CONNAC_SUBSYS_INT`) | Sub-system error / reset request | Permitted by the per-ring enable mask but never enabled on this part. `[C]` position, `[L]` meaning |
| 29 | `0x2000_0000` | `mcu2host_sw_int` (`CONNAC_MCU_SW_INT`, `MT_INT_MCU_CMD`) | MCU→host software interrupt | Always enabled. Second-level status in `0x7C0241F0`. `[C]` |
| 30 | `0x4000_0000` | `tx_done_int_sts_18` | TX ring 18 | 16-entry alternate command-class ring. **In TX-done aggregate.** `[C]` |
| 31 | `0x8000_0000` | `rx_done_int_sts_8` | RX ring 8 | Not in the RX-done aggregate. `[C]` |

No watchdog bit in this register is used on this part; the watchdog / abnormal
condition is delivered as an MCU→host software interrupt (§3) `[C]`. That no such bit
*exists* is `[L]` — bits 24 and 28 are unaccounted for (open question 2). The dedicated MSI
source on CONNAC3 siblings is public information, `[L]` as it applies here.

### 2.1 Ring → interrupt-bit map

Each hardware ring index is paired with an `HOST_INT_STA` bit mask and a ring
`EXT_CTRL` register address, as follows: `[C]`

**TX rings**

| HW TX ring | INT_STA bit mask | Ring `EXT_CTRL` address | Role |
|---|---|---|---|
| 0 | `0x0000_0010` | `0x7C02_4600` | data |
| 1 | `0x0000_0020` | `0x7C02_4604` | data |
| 2 | `0x0000_0040` | `0x7C02_4608` | data |
| 3 | `0x0000_0080` | `0x7C02_460C` | data |
| 4 | `0x0000_0100` | `0x7C02_4610` | data |
| 5 | `0x0000_0200` | `0x7C02_4614` | data |
| 6 | `0x0000_0400` | `0x7C02_4618` | data |
| 7 | `0x0000_0000` | `0x7C02_461C` | data, **no interrupt bit assigned** |
| 8 | `0x0000_1000` | `0x7C02_4620` | data |
| 9 | `0x0000_2000` | `0x7C02_4624` | data |
| 10 | `0x0000_4000` | `0x7C02_4628` | data |
| 11 | `0x0000_8000` | `0x7C02_462C` | data |
| 12 | `0x0001_0000` | `0x7C02_4630` | data |
| 13 | `0x0002_0000` | `0x7C02_4634` | data |
| 14 | `0x0004_0000` | `0x7C02_4638` | data |
| 15 | `0x0000_0000` | — | not present / disabled |
| 16 | `0x0400_0000` | `0x7C02_4640` | firmware download (256 entries) |
| 17 | `0x0800_0000` | `0x7C02_4644` | MCU command (24 entries) |
| 18 | `0x4000_0000` | `0x7C02_4648` | alternate command-class ring (16 entries), selected by capability TLV `0x17` |
| 19 | `0x0000_0000` | — | not present / disabled |

**RX rings**

| HW RX ring | INT_STA bit mask | Ring `EXT_CTRL` address | Role |
|---|---|---|---|
| 0 | `0x0000_0001` | `0x7C02_4680` | MCU event ring (32 entries) |
| 1 | `0x0000_0000` | — | not present / disabled |
| 2 | `0x0000_0004` | `0x7C02_4688` | band-0 RX data (512 entries) |
| 3 | `0x0000_0008` | `0x7C02_468C` | TX-free-done / MSDU report (32 entries) |
| 4 | `0x0040_0000` | `0x7C02_4690` | secondary MCU ring (32 entries) |
| 5 | `0x0080_0000` | `0x7C02_4694` | coredump events (224 entries) |
| 6 | `0x0008_0000` | `0x7C02_4698` | firmware-log events (128 entries) |
| 7 | `0x0200_0000` | `0x7C02_469C` | low-latency data (512 entries) |
| 8 | `0x8000_0000` | `0x7C02_46A0` | management-frame receive (64 entries) |
| 9 | `0x0000_0000` | — | not present / disabled |

### 2.2 Aggregate masks

| Aggregate | Value | Bits |
|---|---|---|
| **TX done** | `0x4C00_0000` | 26, 27, 30 |
| **RX done** | `0x00C0_0000` + `0x0000_000D` = `0x00C0_000D` | 0, 2, 3, 22, 23 |
| MCU→host software interrupt | `0x2000_0000` | 29 |
| Enable-register constant term | `0x6C00_0000` | 26, 27, 29, 30 (= TX-done aggregate + software interrupt) |
| Enable-register constant term, **pre-ready (initialisation) state** | `0x6000_0000` | 29, 30 only |
| Enable-mask sanitiser applied to per-ring RX bits | `0x93CF_FFFF` | permits 0–19, 22–25, 28, 31 |
| Typical programmed `HOST_INT_ENA` with all present RX rings enabled | `0xEEC8_000D` | 0, 2, 3, 19, 22, 23, 25, 26, 27, 29, 30, 31 |

### 2.3 Bit-by-bit decode of the two aggregate constants

**TX-done aggregate `0x4C000000`** `[C]`

| Bit | Mask | Ring | Meaning |
|---|---|---|---|
| 26 | `0x0400_0000` | TX ring 16 | Firmware-download ring TX-done. Required so the host can reclaim FW-download descriptors. |
| 27 | `0x0800_0000` | TX ring 17 | MCU command ring TX-done. Required for command completion accounting. |
| 30 | `0x4000_0000` | TX ring 18 | Third MCU-class TX ring done. |

No data TX ring (bits 4–18) appears in the aggregate. On this part **data TX completion is
not reported through `tx_done_int_sts_*` at all**; it is reported through the TX-free-done
RX ring, i.e. `rx_done_int_sts_3` (bit 3), which is why the MSI vector map for this part
designates the vector carrying bit 3 as the TX-free-done source. `[C]`

**RX-done aggregate `0x00C0000D`** `[C]`

| Bit | Mask | Ring | Meaning |
|---|---|---|---|
| 0 | `0x0000_0001` | RX ring 0 | MCU event ring done. |
| 2 | `0x0000_0004` | RX ring 2 | Band-0 RX data done. |
| 3 | `0x0000_0008` | RX ring 3 | TX-free-done / MSDU-report ring done. |
| 22 | `0x0040_0000` | RX ring 4 | Secondary MCU ring done (post-firmware-download MCU RX path). |
| 23 | `0x0080_0000` | RX ring 5 | Additional RX ring done. |

Bits 19, 25 and 31 (RX rings 6, 7, 8) are **not** in the aggregate even though those rings
have interrupt bits and are marked present; they are, however, included in the value the
host actually programs into `HOST_INT_ENA` (because the enable value is built by OR-ing
the bit of every present RX ring, not from the aggregate). `[C]`

---

## 3. Software-interrupt channel

### 3.1 MCU → host

* Level-1 indication: `HOST_INT_STA` bit 29 (`0x2000_0000`). `[C]`
* Level-2 status / reason register: `0x7C02_41F0`, read-write-1-to-clear. `[C]`
* Enable: `0x7C02_41F4`, programmed with **`0x0000_FFFF`** (all 16 defined reason bits
  unmasked) during interrupt enable, gated on the part's "supports ASIC low power" flag,
  which is set for MT7932. `[C]`
* Host acknowledge procedure: read `0x7C0241F0`; if the value is `0xFFFF_FFFF` the bus is
  considered dead (function-level reset in progress) and **nothing is written**;
  otherwise the read value is written straight back to `0x7C0241F0`. `[C]`
* After acknowledging, if `status & 0x0000_003C` is non-zero the host must enter the
  system-error-recovery (SER) state machine. `[C]`
* A single-bit acknowledge is also used: when the wake-by-RX-packet reason is consumed the
  host writes exactly `0x0000_0001` to `0x7C0241F0`. `[C]`

Reason-code map (CONNAC2 `MCU2HOST_SW_INT_STA`; MT7932 uses the standard CONNAC2
allocation):

| Bit | Mask | Name | Meaning |
|---|---|---|---|
| 0 | `0x0000_0001` | `MT_MCU_CMD_WAKE_RX_PCIE` | Firmware woke the host because an RX packet arrived while the host was in low power. `[C]` |
| 1 | `0x0000_0002` | `ERROR_DETECT_STOP_PDMA_WITH_FW_RELOAD` | Stop PDMA, firmware reload required. `[L]` |
| 2 | `0x0000_0004` | `ERROR_DETECT_STOP_PDMA` / `MT_MCU_CMD_STOP_DMA` | SER: host must stop the DMA engine. In the acted-on mask. `[C]` |
| 3 | `0x0000_0008` | `ERROR_DETECT_RESET_DONE` | SER: sub-system reset complete. In the acted-on mask. `[C]` |
| 4 | `0x0000_0010` | `ERROR_DETECT_RECOVERY_DONE` | SER: recovery complete. In the acted-on mask. `[C]` |
| 5 | `0x0000_0020` | `ERROR_DETECT_MCU_NORMAL_STATE` | SER: MCU back to normal. In the acted-on mask. `[C]` |
| 6 | `0x0000_0040` | `ERROR_DETECT_SER_TRIGGER_IN_SUSPEND` | An L1 reset was triggered while the HIF was suspended. `[L]` |
| 7 | `0x0000_0080` | `ERROR_DETECT_SER_DONE_IN_SUSPEND` / `SUBSYS_BUS_TIMEOUT` | Meaningful only with bit 6 set. `[L]` |
| 8 | `0x0000_0100` | `CP_LMAC_HANG_WORKAROUND_STEP1` | `[L]` |
| 9 | `0x0000_0200` | `CP_LMAC_HANG_WORKAROUND_STEP2` | `[L]` |
| 10 | `0x0000_0400` | `ERROR_DETECT_SER_BUS_HANG` | `[L]` |
| 11–15 | — | reserved | Unmasked by the `0xFFFF` enable value but undefined. `[U]` |
| 24 | `0x0100_0000` | `ERROR_DETECT_LMAC_ERROR` | `[L]` |
| 25 | `0x0200_0000` | `ERROR_DETECT_PSE_ERROR` | `[L]` |
| 26 | `0x0400_0000` | `ERROR_DETECT_PLE_ERROR` | `[L]` |
| 27 | `0x0800_0000` | `ERROR_DETECT_PDMA_ERROR` | `[L]` |
| 28 | `0x1000_0000` | `ERROR_DETECT_PCIE_ERROR` | `[L]` |

Acted-on error mask on this part: **`0x0000_003C`** (bits 2, 3, 4, 5) — identical to the
public `ERROR_DETECT_MASK`. `[C]`

### 3.2 Host → MCU

* Register: `CONNAC2X_WPDMA_HOST2MCU_SW_INT_SET`, at the **MCU-view** WFDMA0 base:
  **`0x5400_0108`**. (A WFDMA1 alias at `0x5500_0108` is defined by the family but is
  unreachable on this part because WFDMA1 is absent.) `[C]`
* Semantics: write-1-to-set; the host writes a bit mask, the MCU takes the interrupt.
  There is no host-readable status for this direction. `[C]`
* Command codes (CONNAC2 allocation):

| Bit | Mask | Name | Meaning |
|---|---|---|---|
| 0 | `0x01` | `MCU_INT_PDMA0_STOP_DONE` | Host has stopped its PDMA rings (SER handshake). `[L]` |
| 1 | `0x02` | `MCU_INT_PDMA0_INIT_DONE` | Host has re-initialised its PDMA rings. `[L]` |
| 2 | `0x04` | `MCU_INT_SER_TRIGGER_FROM_HOST` / `MCU_INT_NOTIFY_MD_CRASH` (CONNAC2 aliasing) | Host requests SER / notifies modem crash. `[L]` |
| 3 | `0x08` | `MCU_INT_PDMA0_RECOVERY_DONE` | Host completed SER recovery. `[L]` |
| 4 | `0x10` | `MCU_INT_DRIVER_SER` | Host-initiated ("driver") SER. `[L]` |

---

### 3.3 Acknowledgement obligations and their deadlines

Both software-interrupt directions are part of a lock-step protocol with the firmware, not a
notification the host may batch or defer.

**MCU → host.** The **obligations** (what the host writes, and when) are `[C]`; the **If not
met** column describes firmware behaviour and is `[L]` throughout.

| Obligation | Deadline | If not met |
|---|---|---|
| Write the read status value back to `0x7C02_41F0` | before any subsequent release of ownership | The firmware is understood to sample the host WFDMA interrupt state before parking and not to complete a sleep entry while it is non-zero. The release is fire-and-forget and reports success `[C]`, so the host does not learn that the part stayed awake. |
| Enter the error-recovery state machine when `status & 0x0000_003C` is non-zero | the host's own 10 s recovery timer | the firmware waits at that recovery checkpoint indefinitely; the MAC stays reset |
| Write exactly `0x0000_0001` back when the wake-by-RX-packet reason is consumed | before re-arming the low-power path | the wake source stays latched and the next wake is misattributed |
| Never write anything when the register reads `0xFFFF_FFFF` | — | the bus is gone (function-level reset in progress); the write would fault |

**Host → MCU.** There is **no status register for this direction** and no completion
indication: the write is the entire protocol. `[C]` Each recovery-phase code must be written
exactly once, after the corresponding host-side work has *completed*. Writing a phase code
early lets the firmware advance while the host is still re-programming rings; writing it
twice is indistinguishable from a spurious phase advance. `[L]` (firmware behaviour) The three
recovery phase codes (`0x01`, `0x02`, `0x08`) and the host-initiated recovery request (`0x10`)
are the only values used on this part. `[C]`

**No firmware-side timeout that advances either direction was found.** `[L]` The firmware
appears to announce each phase and then block. Every deadline on this page is a host-side
deadline `[C]`, and its expiry must escalate to the next recovery level; no self-healing path
is provided by the host `[C]`.

## 4. MSI / MSI-X

### 4.1 Capability and vector count

* The Wi-Fi function is an ordinary PCIe endpoint using **MSI** (the package also presents a
  separate Bluetooth function — see the chip-delta section §1.4). Nothing in the programming
  sequences requires MSI-X; the vector table below is an event→vector map, not a PCI MSI-X
  table. `[L]`
* The hardware-configuration record declares a **maximum of 8 MSI vectors**
  (`u4MaxMsiNum` = 8) and defines **two** vector layouts: an 8-entry multi-vector layout and a
  1-entry single-vector layout. `[C]` That the endpoint's MSI capability actually advertises 8
  is `[U]` (open question 5 and §1.4 of the PCIe section).

### 4.2 Multi-vector layout (8 vectors)

Each entry identifies one interrupt source by three fields: a TX-ring index bitmap, an
RX-ring index bitmap and a literal `HOST_INT_STA` bit mask. The two ring bitmaps expand
through the ring-to-interrupt map of §2.1 into a concrete `HOST_INT_STA` acknowledge mask
for that vector, together with a coarse "TX done / RX done" classification (TX bitmap
non-empty ⇒ TX-done; RX bitmap non-empty ⇒ RX-done). `[C]`

In the two tables below the bitmap and literal-mask **values**, and the acknowledge masks
derived from them, are `[C]`; the **Source name** and **Signals** columns are an
interpretation of those bitmaps and are `[L]`.

| Vector | Source name | TX-ring bitmap | RX-ring bitmap | Literal INT_STA bits | Expanded `HOST_INT_STA` ack mask | Signals |
|---|---|---|---|---|---|---|
| 0 | *(unused)* | — | — | — | `0x0000_0000` | no source mapped |
| 1 | *(unused)* | — | — | — | `0x0000_0000` | no source mapped |
| 2 | *(unused)* | — | — | — | `0x0000_0000` | no source mapped |
| 3 | **RX data** | — | `0x0000_0004` → RX ring 2 | — | `0x0000_0004` | Band-0 RX data ring done |
| 4 | **TX-free-done** | — | `0x0000_0008` → RX ring 3 | — | `0x0000_0008` | TX-free-done / MSDU-report ring done (data TX completion) |
| 5 | *(unused)* | — | — | — | `0x0000_0000` | no source mapped |
| 6 | *(unused)* | — | — | — | `0x0000_0000` | no source mapped |
| 7 | **lump** | `0x0007_0000` → TX rings 16, 17, 18 | `0x0000_01F1` → RX rings 0, 4, 5, 6, 7, 8 | `0x2000_0000` | `0x4C00_0000` \| `0x82C8_0001` \| `0x2000_0000` = **`0xEEC8_0001`** | Everything else: FWDL/CMD/MCU TX-done, MCU event rings, and the MCU→host software interrupt |

### 4.3 Single-vector layout (1 vector)

| Vector | Source name | TX-ring bitmap | RX-ring bitmap | Literal INT_STA bits | Expanded ack mask | Signals |
|---|---|---|---|---|---|---|
| 0 | **lump** | `0x0007_0000` → TX rings 16, 17, 18 | `0x0000_01FD` → RX rings 0, 2, 3, 4, 5, 6, 7, 8 | `0x2000_0000` | `0x4C00_0000` \| `0x82C8_000D` \| `0x2000_0000` = **`0xEEC8_000D`** | All WFDMA0 TX-done, RX-done and MCU software-interrupt sources |

The single-vector RX bitmap differs from the 8-vector "lump" bitmap by exactly the two
bits that the dedicated vectors 3 and 4 would otherwise own (RX rings 2 and 3). The
resulting single-vector ack mask `0xEEC8000D` is bit-identical to the `HOST_INT_ENA`
value the enable sequence programs, i.e. one vector acknowledges exactly the enabled set.
`[C]`

### 4.4 Mode selection

* Requesting **one** MSI vector selects the single-vector layout; on failure the host
  falls back to the legacy line interrupt (§7). Single-vector MSI is a valid operating
  mode even though the part declares support for 8 vectors, and is the mode for which the
  masks in §4.3 are given. `[C]` (Which vector count to request is a host choice.)
* When MSI is enabled the status need not be read from `HOST_INT_STA` at all: the host may
  OR together the acknowledge masks (`0xEEC8000D` for the single vector) of the pending
  vectors and write the result to `0x7C024200` in one W1C write. `[C]`

### 4.5 Device-side registers involved in MSI

| Register | Value written | Purpose |
|---|---|---|
| `0x7403_0188` (PCIe MAC interrupt enable) | `0x0000_01FF` on enable, `0x0000_0000` on disable | Device-side master unmask; must be re-written after every mask/service cycle and after resume. `[C]` |
| `0x7403_018C` (PCIe MAC interrupt clear) | caller-supplied bit mask | PCIe-MAC-level acknowledge / re-arm. `[C]` |
| `0x7C02_4204` (`HOST_INT_ENA`) | `0xEEC8_000D` typical | WFDMA-level unmask. `[C]` |
| `0x7C02_4298` (`WPDMA_INT_RX_PRI_SEL`) | `0x0000_000C` when MSI is enabled | Selects RX rings 2 and 3 onto the "priority" RX interrupt path. Written **only** when MSI is enabled `[C]`; the coalescing path separately read-modify-writes it (§6.2) |
| `0x7C02_7038` (ext conn-hif wrap CSR) | `0x0000_0013` when MSI is enabled | Undocumented ext-wrap CSR programmed only in the MSI path `[C]`. On CONNAC3 siblings the neighbouring ext-wrap CSRs at `+0x2C`/`+0x30`/`+0xF0..0xFC` hold MSI vector-count and per-source vector-routing fields, so this is plausibly the CONNAC2 equivalent routing/enable word `[L]`; `[C]` address+value, `[U]` field meaning |

**There is no per-source vector-routing register programmed on this part.** The CONNAC3
`WFDMA_MSI_CONFIG` (`ext-wrap + 0x2C`), `pcie0_msi_num`/`pcie1_msi_num`
(`ext-wrap + 0x30`, bits [1:0]/[5:4]) and `MSI_INT_CFG0..3`
(`ext-wrap + 0xF0..0xFC`) registers are **never written** on MT7932; the event→vector
association in §4.2 is purely a host-side software table over a fixed hardware routing.
`[C]`

---

## 5. Interrupt service discipline

The order below is the one the host observes. Each step is `[C]` as a description of what is
done; where the text says the hardware *requires* it, that is `[L]` unless the step names a
register property (write-1-to-clear, no set/clear alias) that is itself `[C]`.

1. **Top half (hard IRQ context): mask first.** The top half writes
   `HOST_INT_ENA = 0` **and** `PCIe MAC interrupt enable = 0`, then schedules the bottom
   half. It performs no status read. For the legacy line interrupt this masking is
   mandatory to de-assert the level-triggered line; for MSI it is what prevents an
   interrupt storm while the bottom half runs. `[C]`
2. **Bottom half: obtain status.** On this part the status register is **never read** here:
   (a) in MSI mode the status is synthesised from the pending-vector set, and
   (b) otherwise it is derived from the DMA descriptor DONE bits in host memory.
   `[C]` (Reading `HOST_INT_STA` once is the obvious third option and is what public `mt76`
   does; it is not exercised on this part. `[L]`)
3. **Acknowledge before processing.** Write the serviced bit mask back to
   `0x7C024200` (W1C) **before** draining the rings. Acknowledging after the drain would
   lose completions that arrived during the drain. `[C]`
4. **If bit 29 was set**, read `0x7C0241F0`, write the read value back, then act on the
   reason bits (§3.1). `[C]`
5. **Drain the rings**, then **re-enable**: write the full enable value to
   `HOST_INT_ENA` and `0x0000_01FF` to the PCIe MAC interrupt enable, in that order.
   `[C]`

Ordering / race notes:

* **The status register must be read (or synthesised) *after* masking, not before.**
  Masking is unconditional and must not be preceded by a status read.
  `[C]`
* **Mask/status race:** `HOST_INT_ENA` has no set/clear alias, so a partial re-enable is
  a read-modify-write on a register that the interrupt path itself also writes. The
  public mt76 driver handles the same silicon by keeping a software shadow of the mask
  under a lock and never doing a bare RMW from the interrupt path; writing only the full
  value or zero avoids the problem entirely.
  A driver that wants per-ring re-arm must maintain its own shadow. `[C]` (register
  behaviour), `[L]` (necessity of shadowing)
* A source whose status bit is set while masked stays latched; unmasking re-raises it
  without any further device event. `[U]` — same open point as above.
* **No read-back is performed in the interrupt service path.** A plain 32-bit posted MMIO
  write with no read-back and no barrier is used, including for the mask write in hard-IRQ
  context. `[C]` (Note that the *bring-up* and *disable* paths do read `HOST_INT_ENA` back to
  push the write — see §3.5 of the PCIe section. Whether a read-back is ever required is
  `[L]`.)
* The status must **not** be masked against the enable value before acknowledging;
  acknowledge exactly the bits that are to be serviced.
  `[C]`
* **Correction to a common misreading.** The alternate enable constant `0x6000_0000` is not
  selected by a "no-MMIO-read" option. It is used while the driver is still in its
  **initialisation / pre-firmware-ready state** (the flag is cleared at the end of a
  successful adapter start) `[C]`. In that state the host also substitutes a bounded retry
  poll of the RX descriptor DONE bit (5 attempts, 20 µs apart) for the normal service path
  `[C]`. Note that **neither** state reads `HOST_INT_STA` over MMIO: in MSI mode the pending
  set is reconstructed from which vector fired, and otherwise from the descriptors' DONE bits
  in host DRAM. `[C]` The only reads of `0x7C02_4200` anywhere are in two debug
  register-dump routines `[C]`.

---

## 6. Interrupt moderation

### 6.1 Priority (per-ring) delayed interrupt — `0x7C02_42F0`

`WPDMA_PRI_DLY_INT_CFG0`, CONNAC2 layout:

| Field | Bits | Unit | Applies to |
|---|---|---|---|
| `PRI0_MAX_PTIME` | [7:0] | 20 µs ticks; 0 disables the time check | RX ring 2 |
| `PRI0_MAX_PINT` | [14:8] | interrupt events (packet count) | RX ring 2 |
| `PRI0_DLY_INT_EN` | [15] | 1 = enable | RX ring 2 |
| `PRI1_MAX_PTIME` | [23:16] | 20 µs ticks | RX ring 3 |
| `PRI1_MAX_PINT` | [30:24] | interrupt events | RX ring 3 |
| `PRI1_DLY_INT_EN` | [31] | 1 = enable | RX ring 3 |

An interrupt is raised when either the pending count reaches `MAX_PINT` **or** the
pending time reaches `MAX_PTIME × 20 µs`. `[L]` for the layout (it is the public CONNAC2 CODA
definition, assumed to hold here — see open question 5 of the WFDMA section), `[C]` for the
values below.

Values used on this part:

* **Default, written during WFDMA setup when MSI is enabled:** `0x8032_800A`
  → RX ring 2: enabled, `MAX_PINT = 0`, `MAX_PTIME = 10` (**200 µs**);
    RX ring 3: enabled, `MAX_PINT = 0`, `MAX_PTIME = 0x32 = 50` (**1000 µs**). `[C]`
  The same constant is restored when the adaptive high-throughput mode stops `[C]`.
* **Adaptive high-throughput value:** the host runs a 1 s RX-throughput monitor; above
  the configured RX packet/byte thresholds it rewrites the register as
  `0x8032_0000 | (en << 15) | ((pint & 0x7F) << 8) | ptime`, i.e. it retunes **only**
  the RX-ring-2 half and leaves the RX-ring-3 half pinned at
  `enable, MAX_PINT = 0, MAX_PTIME = 50`. Below the thresholds it restores
  `0x8032_800A`. `[C]`
* `WPDMA_PRI_DLY_INT_CFG1` / `CFG2` (`0x7C02_42F4`, `0x7C02_42F8`), which on sibling
  CONNAC parts extend the same PRI0/PRI1 field pair to RX rings 4/5 and beyond, are
  **never written** on this part. `[C]` (unused), `[L]` (existence)

### 6.2 Priority-path ring select — `0x7C02_4298`

`WPDMA_INT_RX_PRI_SEL`. Bit *n* selects RX ring *n* onto the priority (delayed-interrupt)
path.

| Bits | Meaning | Value used |
|---|---|---|
| [2] | RX ring 2 priority select | set |
| [3] | RX ring 3 priority select | set |

Written as `0x0000_000C` during WFDMA setup when MSI is enabled `[C]`. The interrupt-coalescing
path separately read-modify-writes it: enabling sets both bits (OR `0xC`), disabling clears
**only bit 2** (AND `~0x4`) — so **bit 3 is never cleared once set**. `[C]`

### 6.3 Periodic (per-RX-ring) delayed interrupt — `0x7C02_42E8`

`HOST_PER_DLY_INT_CFG`:

| Field | Bits | Unit |
|---|---|---|
| `wpdma_per_max_ptime` | [7:0] | 20 µs ticks; 0 disables the pending-time check |
| reserved | [15:8] | — |
| `wpdma_per_dly_int_en` | [31:16] | one enable bit per RX ring, bit *n* = RX ring *n* |

Values used: `0x01FD_0032` when enabled, `0x0000_0000` when disabled. `[C]`
Decoded: `wpdma_per_max_ptime = 0x32 = 50` → **1000 µs**; `wpdma_per_dly_int_en = 0x01FD`
→ RX rings **0, 2, 3, 4, 5, 6, 7, 8** (exactly the set of present RX rings; rings 1 and 9
excluded). `[L]` — the field split is not in any public header; the decode is inferred from the
exact match between `0x01FD` and the populated-RX-ring set, and needs the trace of open
question 5 in the WFDMA section.

### 6.4 PCIe-level delayed RX interrupt — `0x7C02_7030`

`WFDMA_HOST_CONFIG` (ext conn-hif wrap). Bit 9 = `pcie_dly_rx_int_en`. Set when
interrupt coalescing is enabled, cleared when disabled; the rest of the register is
preserved by read-modify-write. `[C]`

### 6.5 Firmware-side coalescing

A firmware command (**command ID `0xB2`**, `CMD_PF_CF_COALESCING_INT`, set, 76-byte
payload) configures packet-parsing-based RX interrupt coalescing inside the MCU; the
firmware replies with event ID `0xB2`. Payload layout as used:

| Offset | Size | Field | Value |
|---|---|---|---|
| 0 | 1 | command version | 0 |
| 1 | 1 | action | 0 |
| 2 | 2 | command length | 0 |
| 4 | 1 | packet-count threshold enable | 1 when RX throughput ≥ threshold, else 0 |
| 5 | 1 | timer threshold enable | same value as above |
| 6 | 2 | max packets | host-configured (public default 50) |
| 8 | 2 | max time | host-configured, **milliseconds** (public default 1 ms) |
| 10 | 2 | filter mask | BIT0 IPv4/TCP, BIT1 IPv4/UDP, BIT2 IPv6/TCP, BIT3 IPv6/UDP |
| 12 | 64 | reserved | 0 |

Both threshold-enable bytes are driven together by a single "RX throughput above the
configured coalescing threshold" decision. `[C]`

### 6.6 Per-RX-ring pause thresholds

`MT_WFDMA0_WPDMA_PAUSE_RXQ_TH10/32/54/76` at `0x7C02_4260`, `0x7C02_4264`,
`0x7C02_4268`, `0x7C02_426C`, each holding a low threshold in bits [11:0] and a high
threshold in bits [27:16] for a pair of RX rings, are part of the public CONNAC2 map but
are **not programmed** on MT7932. `[C]` (unused), `[L]`
(layout, from public mt76)

---

## 7. Line-based (INTx) fallback

Supported and used whenever the single-MSI allocation fails. `[C]`

Differences from the MSI path:

* One shared, level-triggered handler is installed on the legacy interrupt line.
* The top half **must** mask (`HOST_INT_ENA = 0`, PCIe MAC interrupt enable `= 0`) before
  returning, otherwise the line stays asserted. `[C]`
* The bottom half cannot synthesise the status from a vector index; it runs in the
  derive-from-descriptor mode of §5 (reading `HOST_INT_STA` would be the alternative, and is
  what public `mt76` does). `[C]`
* No per-source separation: all sources funnel to the single line, exactly like the
  single-vector MSI layout of §4.3, but with the extra level-de-assert requirement.

---

## 8. Wake-on-WLAN and low-power interrupt path

### 8.1 Ownership handshake

| Register | Bit | Meaning |
|---|---|---|
| `0x7C06_0010` (`BN0_LPCTL`) | 0 | Host sets firmware ownership (device may enter low power). `[C]` |
| `0x7C06_0010` | 1 | Host clears firmware ownership (request the device back). `[C]` |
| `0x7C06_0010` | 2 | Owner-state-sync: 1 = firmware owns, 0 = host owns. Polled after writing bit 1. `[C]` |
| `0x7C06_0014` (`BN0_IRQ_STAT`) | 0 | Firmware-cleared-own latch: the device has handed ownership back. W1C. `[L]` — carried per-chip as `fw_own_clear_addr`/`fw_own_clear_bit` `[C]`, but **never read or written on MT7932** (below) |
| `0x7C06_0018` (`BN0_IRQ_ENA`) | 0 | Written `BIT(0)` during bring-up `[C]`; that this enables the latch above as an interrupt is `[L]` (public naming) |

Sequence to reacquire the device: write `BIT(1)` to `0x7C060010`, then read `0x7C060010`
and test bit 2 == 0. `[C]`

**Correction / MT7932 note.** The ownership-acquisition path on this part does **not** touch
`0x7C06_0014`. The write-back of `BIT(0)` is gated on a per-chip "check driver-own interrupt"
flag which is **false** for MT7932 (and for MT7922), so ownership is confirmed purely by
polling `LPCTL` bit 2. `[C]` The power/reset section §3.1 states the same. A driver should not
implement the latch acknowledge unless it first establishes that the latch is set at all.

### 8.2 Wake sources

| Source | Register / bit | Notes |
|---|---|---|
| RX packet matched a wake pattern while host was in low power | `HOST_INT_STA` bit 29 → `MCU2HOST_SW_INT_STA` bit 0 | The canonical wake-on-WLAN path. Host acknowledges by writing `0x1` to `0x7C0241F0`, then treats the band-0 RX data ring as pending. `[C]` |
| Firmware returned device ownership | `BN0_IRQ_STAT` bit 0, enabled by `BN0_IRQ_ENA` bit 0 | Signals completion of a host-own request. `[C]` |
| Sub-system error / SER while suspended | `MCU2HOST_SW_INT_STA` bits 2–5, plus bits 6/7 (`SER_TRIGGER_IN_SUSPEND`, `SER_DONE_IN_SUSPEND`) | Bits 6/7 exist specifically so the host can tell, after resume, that an L1 reset happened while the HIF was suspended. `[C]` bit definitions from public CONNAC2, `[L]` for MT7932 |
| Normal ring completion | `HOST_INT_STA` RX/TX-done bits | Only after DMA is re-enabled. |

### 8.3 Wake-reason reporting

* The **hardware** wake-reason latch is `MCU2HOST_SW_INT_STA` (`0x7C0241F0`) bit 0. There
  is no dedicated wake-reason CSR. `[C]`
* The **detailed** wake reason (which pattern, which protocol, which offload engine) is
  delivered as a firmware event (`EVENT_WOW_WAKEUP_REASON_INFO`), not through a register.
  `[C]`
* The device does not shadow `MCU2HOST_SW_INT_STA`, so the host must keep its own copy of
  the value it read before acknowledging it, or the wake reason is lost. `[C]`
* The PCIe MAC interrupt enable (`0x7403_0188`) is zeroed on suspend and must be
  re-written with `0x0000_01FF` on resume before any WFDMA interrupt can reach the host.
  `[C]`

### 8.4 MT7932-specific: ownership-IRQ-status clear is skipped

The bring-up path contains an ownership-interrupt-status clear — write `0x0000_0003` to
`0x7C00_1620`, then wait 2 ms — that is **gated on the PCI device ID being exactly
`0x7922`**. It is therefore **not executed for MT7932** (device ID `0x7932`) or for
device ID `0x7923`. This is one of the very few places in the whole HIF where behaviour
is conditional on the chip identity, and it is the clearest genuine MT7932 delta in the
interrupt/power domain. `[C]`

---

## 9. Interrupt enable / disable sequences (normative)

**Enable** `[C]`

1. If any RX ring is in the "no free receive buffer" state, wait ~1 ms first.
2. Write `0x0000_01FF` to `0x7403_0188` (PCIe MAC interrupt enable).
3. Compute `mask = (OR of the `HOST_INT_STA` bit of every present RX ring) & 0x93CF_FFFF`.
4. Write `mask | 0x6C00_0000` to `0x7C02_4204`
   (or `mask | 0x6000_0000` while the driver is still in its pre-firmware-ready state).
5. Write `0x0000_FFFF` to `0x7C02_41F4` (MCU→host software-interrupt enable).

**Disable** `[C]`

1. Write `0x0000_0000` to `0x7C02_4204`.
2. Write `0x0000_0000` to `0x7403_0188`.

The MCU→host software-interrupt enable is **not** cleared on disable; the level-1 gate
(bit 29 of `HOST_INT_ENA`) is what stops it reaching the host.

---

## 10. Chip-identity conditionals in the interrupt path

Exhaustive list of places where interrupt-related behaviour depends on the part:

| Condition | Effect | MT7932 outcome |
|---|---|---|
| PCI device ID `== 0x7922` | Ownership-IRQ-status clear at `0x7C001620` (§8.4) | **skipped** `[C]` |
| `is_support_wfdma1` (public gen4m field) | Selects `0x7C025xxx` / `0x5500_0xxx` software-interrupt aliases | false ⇒ WFDMA0 addresses only `[C]` |
| `is_support_asic_lp` | Gates the `MCU2HOST_SW_INT_MASK = 0xFFFF` write and the ownership register accesses | true `[C]` |

The MT7932 and MT7922 hardware-configuration records are byte-identical in every
interrupt-relevant field, including the PCIe MAC interrupt-enable value `0x1FF`, the
status/enable addresses, both aggregate masks, and both MSI layout tables. The **only**
difference is the chip-ID word, which is what makes the `0x7922`-gated ownership clear
above the sole behavioural delta. `[C]`

---

## 11. Deltas versus publicly documented MT7921 / MT7922 / MT7925 / MT7927

| Item | Public MT7921 (mt76 / gen4m "mt7961") | MT7932 (this part) |
|---|---|---|
| `HOST_INT_STA` / `HOST_INT_ENA` addresses | `0x7C024200` / `0x7C024204` | identical |
| **TX-done aggregate** | gen4m mt7961: TX rings 0–6, 16, 17 = `0x0C00_07F0`; mt76 `MT_INT_TX_DONE_ALL` = `BIT(27) \| BIT(4) \| GENMASK(18,4)` = `0x0807_FFF0` | **`0x4C00_0000`** — TX rings 16, 17, 18 only. **No data TX-done bit is used at all.** |
| Bit 30 | mt76 names it `HOST_TX_DONE_INT_ENA18` but never enables it | **used**: TX ring 18, and it is part of the TX-done aggregate |
| **RX-done aggregate** | gen4m mt7961: rings 0, 2, 3, 4, 5 = `0x00C0_000D`; mt76 `MT_INT_RX_DONE_ALL` = `BIT(0) \| BIT(2) \| BIT(22)` = `0x0040_0005` | **`0x00C0_000D`** — matches gen4m mt7961 exactly; mt76 omits bits 3 and 23 |
| Data TX completion reporting | `tx_done_int_sts_0..6` | **RX ring 3 (`rx_done_int_sts_3`, bit 3)** — a TX-free-done/MSDU-report RX ring; the vector that carries bit 3 is designated the TX-free-done source |
| RX rings 6, 7, 8 | not defined in public MT7921 headers | present, mapped to **bits 19, 25, 31** — a placement that appears in no public header |
| TX ring 7 (bit 11) | `HOST_TX_DONE_INT_ENA7` defined | ring exists and is enabled but this part's ring-to-interrupt map assigns it **no interrupt bit** |
| RX ring 1 (bit 1) | `HOST_RX_DONE_INT_ENA1` defined; used as WM2 on MT7925 | **not present** on this part |
| MT7927 comparison | MT7927 moves RX data/WM/WM2 to `BIT(12)`, `BIT(14)`, `BIT(15)` | MT7932 does **not** use the MT7927 allocation; it keeps bits 2, 0, 22 for those roles |
| PCIe MAC interrupt enable value | mt76 writes `0xFF` | **`0x1FF`** (bit 8 additionally set) |
| `MCU2HOST_SW_INT_ENA` value | mt76 sets only `MT_MCU_CMD_WAKE_RX_PCIE` (`BIT(0)`) | **`0x0000_FFFF`** (all 16 low reason bits unmasked) |
| MSI vectors | mt76 uses a single legacy/MSI interrupt, no vector table | **8-vector and 1-vector layouts** are both defined; hardware `u4MaxMsiNum = 8` |
| Delayed interrupt | mt76 defines `MT_WFDMA0_PRI_DLY_INT_CFG0` but never writes it | programmed to `0x8032_800A` and retuned adaptively; periodic per-ring delay at `0x7C0242E8` programmed to `0x01FD_0032` |
| `WPDMA_INT_RX_PRI_SEL` (`0x7C024298`) | unused by mt76 | written `0xC` (RX rings 2, 3) whenever MSI is in use |
| Ext-wrap CSR `0x7C027038` | not in any public header | written `0x13` in the MSI path |
| Extended status (`0x7C024118`) and WFDMA1 ext status (`0x7C027010/14`) | defined for CONNAC2 parts with WFDMA1 | **not used**; MT7932 has WFDMA0 only |

---

## 12. Open questions / needs hardware tracing

1. **TX ring 7 / bit 11.** The ring is present, enabled and has an `EXT_CTRL` register,
   yet its interrupt mask is zero. Determine whether bit 11
   is genuinely unimplemented on this silicon, is repurposed, or whether the omission is
   a host-side one. A trace that drives traffic through TX ring 7 and watches
   `HOST_INT_STA` would settle it.
2. **Bit 24 and bit 28.** Both are permitted by the per-ring enable mask (`0x93CFFFFF`)
   but no source maps to bit 24 and bit 28 (`CONNAC_SUBSYS_INT`) is never enabled.
   Confirm whether bit 24 is a ninth RX-done bit and whether bit 28 really is the
   sub-system error/reset-request indication on this part.
3. **Ext-wrap CSR `0x7C027038 = 0x13`.** Field meaning is unknown. On CONNAC3 siblings
   the adjacent CSRs carry MSI vector-count and per-source routing; confirm whether this
   register selects the number of MSI vectors or which sources lump onto vector 0. Note it is
   written only when MSI is enabled, so a legacy-interrupt driver never programs it.
4. **Multi-vector MSI.** The 8-vector layout is defined but not exercised in
   single-vector operation. Confirm that requesting 8 vectors actually yields per-source delivery
   (RX data on vector 3, TX-free-done on vector 4, everything else on vector 7) and
   identify what, if anything, must be programmed on the device to achieve that routing.
5. **MSI-X.** No MSI-X capability is used by this part. Read the endpoint's PCI
   capability list on real silicon to confirm whether an MSI-X capability exists.
6. **Status visibility while masked.** Confirm by trace that `HOST_INT_STA` latches bits
   whose `HOST_INT_ENA` bit is 0, and that unmasking re-raises the interrupt without a
   new device event. This is entirely unestablished: the analysed host never reads the
   status register in its service path, so it provides no evidence either way.
7. **Roles of RX rings 5–8.** Their interrupt bits are confirmed and the WFDMA section
   §3.2 assigns them (coredump, firmware log, low-latency data, management); confirm on
   hardware what traffic firmware actually steers to each, and hence whether a driver needs
   all four.
8. **RX ring 3 role.** The MSI vector map for this part designates the vector carrying bit 3
   as the TX-free-done source, while public gen4m for MT7921 labels RX ring 3 as
   "band-1 RX data". Confirm which is correct for MT7932.
9. **PCIe MAC `0x74030188 = 0x1FF`.** The ninth bit (bit 8) that this part sets and mt76
   does not is undocumented, and in fact the meaning of none of the nine bits is
   established. Determine what each unmasks.
12. **PCIe MAC interrupt status `0x7403_0184`.** Read only in the bus-failure dump, never in
    interrupt service; its role as an interrupt-status register comes from public
    sibling-chip debug code. Confirm what it reports on MT7932 and that `0x7403_018C`
    acknowledges those bits.
10. **`0x7C001620`.** Identify the register and what writing `0x3` does, and confirm that
    MT7932 genuinely does not need it (the skip is device-ID-gated, not
    capability-gated).
11. **Coalescing command defaults.** The packet/time thresholds and filter mask are host
    policy; the values best suited to MT7932 are not determined here.


---

# MT7932 — Firmware Boot Protocol

## Scope

This document specifies how the MT7932 combo Wi-Fi/BT part is brought from PCIe-visible
boot-ROM state to a running Wi-Fi RAM firmware over the PCIe host interface: the set of
firmware images the silicon requires, the byte layout of the two container formats it
accepts (ROM patch container and CONNAC2 RAM-code container), the integrity/crypto fields
in those containers, the dedicated firmware-download DMA path, the boot-ROM ("init")
command/event set that drives the transfer, the register-level ready signalling, the
mandatory post-boot exchange, and the firmware log / coredump interfaces the firmware
exposes to the host. Values labelled *observed* are the contents of the shipped MT7932
firmware artefacts; everything else is marked with a confidence level. The reader is assumed to know the MT7921/MT7922 (CONNAC2, `mt792x`)
boot flow as implemented in upstream `mt76` and in MediaTek's `gen4m`; **deltas are called
out in §11**.

Throughout, "chip address" means an address in the MT7932 internal address space, which
the host reaches either through the 1 MiB static PCIe-BAR window or through the
programmable remap window (see the PCIe/register-map section of this project).

---

## 1. Firmware images required by the silicon

The MT7932 boot ROM is capable of nothing but accepting a patch and a RAM image over the
firmware-download path; all Wi-Fi function lives in the downloaded RAM code.

| Image | Container | Mandatory | Purpose |
|---|---|---|---|
| MCU ROM patch | Patch container v2 (§2) | Yes, unless the on-chip patch semaphore reports the patch is already resident (§6.6) | Patches the MCU boot/mask ROM. Loaded to chip address `0x0090_0000`. Shipped size ~33 KB (33 568 B observed). |
| Wi-Fi RAM firmware ("N9" RAM code) | CONNAC2 tailer container (§3) | Yes | The operational Wi-Fi firmware. Shipped size ~1.14 MB (1 190 788 B observed), 5 regions. |
| Manufacturing/test ROM patch | Patch container v2, identical layout | Only in test/manufacturing mode | Larger patch (46 816 B observed, single section, same destination `0x0090_0000`). |
| Manufacturing/test RAM firmware | CONNAC2 tailer container, identical layout | Only in test/manufacturing mode | 787 328 B observed, 5 regions, release manifest string marks it `TEST_MODE`. |
| Index-log symbol file | Flat record list (§9.2) | No — diagnostics only | Translates the numeric index in firmware log records into `file:line:format`. 192 778 B observed, 3283 records. |
| EEPROM/efuse buffer image, per-rate-power blob, TX-power-limit tables | (out of scope here) | No | Consumed after firmware start, not part of boot. |

Notes:

* `[C]` The host does **not** need a separate CR4/WA co-processor image: MT7932 has
  *no CR4* and *no WA-CPU* (`is_support_cr4`/`is_support_wacpu` both false), so exactly one
  RAM image is downloaded (`IMG_DL_IDX_N9_FW`). This matches MT7921/MT7922.
* `[C]` There is no DSP firmware and no EMI-resident ROM image for this part.
* `[C]` MT7932's **hardware-configuration record** is byte-identical to MT7922's for the
  firmware-download procedure, bus parameters, TX-descriptor format and MCU handshake
  registers, and every behavioural difference is selected at run time from the PCI device ID
  (§11). `[L]` that the *silicon* therefore behaves identically in those respects — that is
  inferred from the record, not measured.

### 1.1 Variant selection

`[C]` The host picks images by three values, all read from the chip before download:

| Selector | Source | Observed MT7932 value |
|---|---|---|
| Chip ID | chip address `0x8002_1008` (bus top-config base `0x8002_0000` + `0x1008`), bits 15:0 | `0x7932` |
| Chip revision | chip address `0x8002_1000`, bits 3:0 | — |
| Hardware version (`TOP_HVR`) | chip address `0x7001_0204`, bits 7:0 | — |
| Factory version | `TOP_HVR` bits 11:8 | — |
| Software/ECO version (`TOP_FVR`) | chip address `0x8800_0004`, bits 7:0 | `1` |

The "firmware version" token embedded in file names is `TOP_FVR[7:0] + 1`, i.e. **2** for
the shipped MT7932 parts, corresponding to ECO **E2** — which agrees with the ECO code
carried in the RAM container's common tailer (§3.1).

`[C]` `TOP_HVR` and `TOP_FVR` are read through the **boot-ROM register-access command**
(§6.7), i.e. before any firmware image is downloaded and while the boot ROM is still the
only thing running. This ordering matters: the image file names cannot be constructed
until these values are known.

`[C]` The chip-ID/revision read at `0x8002_1008` / `0x8002_1000` is a direct register read
through the static BAR window. MT7932 sets `should_verify_chip_id` to **0**, so this
comparison is skipped for this part (it is performed on parts that enable it); the PCI
device ID is used instead.

`[C]` Name templates (chip id and version substituted in hex):

* ROM patch: `WIFI_MT<chipid>_patch_mcu_1_<fwver>_hdr.bin` → `WIFI_MT7932_patch_mcu_1_2_hdr.bin`
* Index log: `WIFI_RAM_CODE_MT<chipid>_1_<fwver>_idxlog.bin` → `WIFI_RAM_CODE_MT7932_1_2_idxlog.bin`
* RAM code: a priority list is generated from a prefix table as
  `<prefix><chipid><_fwver>.bin`, `<prefix><chipid>_<fwver>`, `<prefix><chipid>.bin`,
  `<prefix><chipid>` and tried in order → the shipped file matches `W7932_2.bin`.
* Manufacturing variants use a distinct prefix (`WIFI_MFG_MT<chipid>_<fwver>.bin`,
  `WIFI_MFG_MT<chipid>_patch_mcu_1_<fwver>_hdr.bin`).

`[C]` The per-chip patch destination address is `0x0090_0000`, used when the legacy (v1)
patch header format is presented; it is identical to the `dl_addr` carried inside the
shipped v2 container.

`[C]` The manufacturing images are size-capped by the host at 1 MiB (RAM) and 64 KiB
(patch); selection between production and manufacturing images is a host-side test-mode
decision, not a chip-visible one. The container format is byte-identical.

---

## 2. ROM patch container format

Endianness: the 32-byte outer header is a byte array plus (per `mt76`) big-endian version
words; **everything from offset `0x20` onward is big-endian** `[C]` — the host performs an
explicit 32-bit byte swap on every global-descriptor and section-map word.

### 2.1 Outer header (`PATCH_FORMAT_V2_T`, 32 bytes)

| Offset | Size | Field | Encoding | Observed (production MT7932 patch) |
|---|---|---|---|---|
| `0x00` | 16 | Build date/time string | ASCII, **not** NUL-terminated | `"20260331234651a\n"` |
| `0x10` | 4 | Platform tag | ASCII | `"ALPS"` |
| `0x14` | 4 | Combined SW/HW version | BE32, `hw<<16 \| sw` | `0x8A10_8A10` |
| `0x18` | 4 | Patch version / **format discriminator** | BE32 | `0xFFFF_FFFF` |
| `0x1C` | 2 | Reserved | — | `0x0000` |
| `0x1E` | 2 | CRC-16 over the image | LE16 | `0x0000` (not populated) |

`[C]` The value `0xFFFF_FFFF` at `0x18` is the magic that selects the **multi-address
(v2) container**. Any other value means the legacy v1 layout, in which the payload starts
at offset `0x1E` and is downloaded as one blob to the chip-support record's patch address.
The shipped MT7932 patch (production and manufacturing) is always v2.

`[C]` The trailing `a` in the build-date field is a flavour/branch letter, not part of the
timestamp. Manufacturing patch observed: `"20250821172242a\n"`, same `ALPS`, same
`0x8A108A10`, same `0xFFFFFFFF`.

### 2.2 Global descriptor (`PATCH_GLO_DESC`, 64 bytes, at `0x20`)

| Offset | Size | Field | Observed (production) | Observed (manufacturing) |
|---|---|---|---|---|
| `0x20` | 4 | Patch version (BE32) | `0x4433_2211` | `0x0207_0300` |
| `0x24` | 4 | Subsystem (BE32) | `0x0000_0004` | `0x0000_0004` |
| `0x28` | 4 | Feature bits (BE32) | `0x0000_0000` | `0x0000_0000` |
| `0x2C` | 4 | Section count (BE32) | `1` | `1` |
| `0x30` | 4 | CRC (BE32) | `0x0000_FFFF` | `0x0000_FFFF` |
| `0x34` | 44 | Reserved | zero | zero |

`[C]` The host reads the section count as the **least-significant byte** of the big-endian
word at `0x2C`, so the practical limit is 255 sections. `[L]` Subsystem `4` identifies the
Wi-Fi MCU subsystem. `[U]` The CRC word is not verified by the host and appears not to be
populated meaningfully (`0x0000FFFF` in both images).

`[C]` The word at `0x20` is **not** a format magic — it is a plain patch version number,
and the two shipped MT7932 patches carry different values (`0x4433_2211` production,
`0x0207_0300` manufacturing). The format discriminator is the `0xFFFF_FFFF` at `0x18`
(§2.1). The `0x20` word's raw byte sequence in the production file is `44 33 22 11`, i.e.
`0x1122_3344` if read little-endian; a parser must not key off it.

### 2.3 Section map (`PATCH_SEC_MAP`, 64 bytes each, array starts at `0x60`)

All fields BE32.

| Offset in entry | Field | Meaning |
|---|---|---|
| `+0x00` | `section_type` | Low 16 bits select the descriptor kind. `0x0002` = *binary info* (a downloadable section). Upper 16 bits are a sub-type/tag. |
| `+0x04` | `section_offset` | Byte offset of the payload from the **start of the file** |
| `+0x08` | `section_size` | Payload length in the file |
| `+0x0C` | `dl_addr` | Destination **chip address** |
| `+0x10` | `dl_size` | Number of bytes to transfer to `dl_addr` |
| `+0x14` | `sec_info` | Security/crypto descriptor (§4.2). `0xFFFF_FFFF` = "no security info" |
| `+0x18` | `align_len` | Padding the host must append to reach the transfer alignment the target requires |
| `+0x1C` | `bin_type` | Which subsystem the section belongs to: `0x0000_0100` = Wi-Fi patch, `0x0000_0002`/`0x0000_0003` = BT patch / BT cacheable patch, `0x0000_0080` = BT ILM text, `0x0000_1000` = ZigBee firmware, `0x0001_0000` = BT RAM position |
| `+0x20` | reserved[8] | 32 bytes; first word observed `0x0000_0001` |

Observed, production MT7932 patch (1 section):

| # | type | file off | file size | dest chip addr | dl size | sec_info | align | bin_type |
|---|---|---|---|---|---|---|---|---|
| 0 | `0x0004_0002` | `0x0000_00A0` | `0x0000_8280` | `0x0090_0000` | `0x0000_8280` | `0xFFFF_FFFF` | `0` | `0x0000_0100` (Wi-Fi patch) |

Observed, manufacturing MT7932 patch (1 section): identical except `file size = dl size =
0x0000_B640`.

`0xA0 + 0x8280 = 0x8320` = 33 568 B = exact file size: **the container has no trailer and
no appended CRC** `[C]`.

### 2.4 Per-section download rules

`[C]`

1. Iterate the section map. A section whose `section_type & 0xFFFF != 0x0002` is **not**
   a binary section and must be skipped (no transfer, no command).
2. For a binary section, derive the *download mode* word from `sec_info` (§4.2).
3. Issue a download-target command (`INIT_CMD_ID_PATCH_START`, §6.2) carrying
   `{dl_addr, dl_size, data_mode}`.
4. Stream `dl_size` bytes from `file_base + section_offset` as scatter packets of at most
   2048 bytes each (§5).
5. After **all** sections, issue exactly one patch-finish command (§6.5).
   The finish command is *not* per section.
6. `[C]` The container is capable of carrying Wi-Fi, Bluetooth and ZigBee patch sections in
   one file (see `bin_type`); a Wi-Fi host driver must download only `bin_type == 0x100`
   sections and must not assume a single-section file. The shipped MT7932 Wi-Fi patch
   happens to contain exactly one Wi-Fi section.
7. `[C]` The download-mode word must be computed **per section** from that section's own
   `sec_info`. With the shipped single-section patch this is moot; a multi-section,
   mixed-encryption patch would give a wrong result if one section's value were reused.

---

## 3. RAM firmware container format

This is the standard CONNAC2 *tailer* container: payload first, metadata at the end.
All tailer fields are **little-endian** `[C]`.

Layout of the file, from the end backwards:

```
[ region 0 payload ][ region 1 payload ] ... [ region N-1 payload ]
[ optional release-info block ]
[ region tailer 0 ][ region tailer 1 ] ... [ region tailer N-1 ]   (40 bytes each)
[ common tailer ]                                                  (36 bytes)
EOF
```

### 3.1 Common tailer (36 bytes, at EOF-36)

| Offset in tailer | Size | Field | Observed (production) |
|---|---|---|---|
| `+0x00` | 1 | Chip info / chip family code | `0x14` |
| `+0x01` | 1 | ECO code | `0x01` `[C]` (raw byte). Reported by the host as **E2** on the reading that this field is zero-based `[L]` — see the chip-delta section §1.2 |
| `+0x02` | 1 | Region count | `5` |
| `+0x03` | 1 | Format version | `2` |
| `+0x04` | 1 | Format flag (non-zero ⇒ a release-info block follows the payload) | `1` |
| `+0x05` | 2 | Reserved | `0x0000` |
| `+0x07` | 10 | RAM version string | `"____00000\0"` |
| `+0x11` | 15 | RAM build date string | `"20260331164939\0"` |
| `+0x20` | 4 | CRC32 | `0x5DB7_4326` |

`[C]` The host rejects the image if the region count exceeds **10** (`MAX_FWDL_SECTION_NUM`).
`[C]` The CRC32 word is stored but **not** verified by the host.

### 3.2 Region tailer (40 bytes each)

| Offset in entry | Size | Field |
|---|---|---|
| `+0x00` | 4 | Decompressed CRC (compression option only) |
| `+0x04` | 4 | Decompressed (real) size |
| `+0x08` | 4 | Compression block size |
| `+0x0C` | 4 | Reserved |
| `+0x10` | 4 | Destination **chip address** |
| `+0x14` | 4 | Length in the file / bytes to transfer |
| `+0x18` | 1 | Feature set (bitfield, §3.4) |
| `+0x19` | 1 | Type (`0` = none, `1` = index-log database appended to the RAM image) |
| `+0x1A` | 14 | Reserved |

`[C]` Byte-for-byte identical to upstream `mt76`'s `struct mt76_connac2_fw_region` — `mt76`
merges the type byte into its reserved array but the size and field placement match.

### 3.3 Observed region list — production MT7932 RAM image (1 190 788 B, 5 regions)

| # | Destination chip address | Length | Feature set | Type | decomp CRC / size / blk |
|---|---|---|---|---|---|
| 0 | `0x0090_7C60` | `0x0006_8350` (426 832) | `0x20` (valid RAM entry) | 0 | 0 / 0 / 0 |
| 1 | `0x0200_F020` | `0x0005_4FD0` (348 112) | `0x00` | 0 | 0 / 0 / 0 |
| 2 | `0x0040_4400` | `0x0000_3BD0` (15 312) | `0x00` | 0 | 0 / 0 / 0 |
| 3 | `0x0210_A000` | `0x0000_1BD0` (7 120) | `0x00` | 0 | 0 / 0 / 0 |
| 4 | `0xE020_0050` | `0x0005_FF90` (393 104) | `0x00` | 0 | 0 / 0 / 0 |

Sum of region lengths = `0x0012_2A50`; `+0x48` release-info block `+5×0x28` region tailers
`+0x24` common tailer = `0x0012_2B84` = file size. Regions are contiguous in the file, in
tailer order, starting at file offset 0 `[C]`.

Observed region list — manufacturing MT7932 RAM image (787 328 B, 5 regions), build date
`"20250821172420"`:

| # | Destination | Length | Feature |
|---|---|---|---|
| 0 | `0x0090_B000` | `0x0006_4F90` | `0x20` |
| 1 | `0x0200_F000` | `0x0005_4FD0` | `0x00` |
| 2 | `0x0040_4400` | `0x0000_3BD0` | `0x00` |
| 3 | `0x0210_A000` | `0x0000_1FD0` | `0x00` |
| 4 | `0xE020_0000` | `0x0000_0710` | `0x00` |

`[L]` Address-space reading of the destinations: `0x0090_xxxx` is MCU code/ILM (the same
window the ROM patch targets); `0x0040_xxxx` is statically mapped through the PCIe BAR
(BAR offset `0x0008_0000`, 64 KiB); `0x0200_xxxx` / `0x0210_xxxx` and `0xE020_xxxx` are
MCU data / secondary-core memories that are **not** in the static BAR map — they are only
reachable through the firmware-download engine, which is why a register-window download
path is not viable for a full RAM image on this part.

### 3.4 Region feature-set bits

Identical to public CONNAC/CONNAC2 (`FW_FEATURE_*` in `gen4m`, `DL_MODE_*` in `mt76`) `[C]`:

| Bit | Name | Effect |
|---|---|---|
| 0 | encrypted | Section is encrypted; host must set the encryption bits in the download-mode word |
| 2:1 | key index | AES key index, copied verbatim into the download-mode word |
| 3 | compressed image | Region carries a compressed payload (decompression start command) |
| 4 | encryption mode | `0` = AES, `1` = scramble |
| 5 | valid RAM entry | This region's destination address is the firmware entry point |
| 6 | not downloaded | Skip this region entirely |
| 7 | download to EMI | Region targets host/shared DRAM rather than chip memory |

`[C]` For the shipped MT7932 image only bit 5 is set, on region 0. Consequently the
firmware **entry address handed to the start command is `0x0090_7C60`** (production) /
`0x0090_B000` (manufacturing).

### 3.5 Release information block and firmware version reporting

`[C]` Present only when the common tailer's format flag is non-zero **and** the summed
region lengths leave a gap before the region-tailer array. Layout:

```
16 bytes  0x23 ('#') separator
 4 bytes  outer record header  { u16 length ; u8 padding_len ; u8 tag }
   ... nested records, each { u16 length ; u8 padding_len ; u8 tag } + length bytes
       + padding_len bytes, iterated until outer.length bytes are consumed ...
16 bytes  0x23 ('#') separator
```

Nested record tags: `0x01` = release manifest (preferred), `0x02` = release manifest
(fallback, used only when no tag-`0x01` record is present), anything else = ignore.

Observed, production MT7932 RAM image (72-byte block):

| Field | Value |
|---|---|
| leading separator | 16 × `'#'` |
| outer header | length `0x0024` (36), padding `0`, tag `0x00` |
| nested header | length `0x001E` (30), padding `0x02`, tag `0x01` (manifest) |
| manifest string | `"Sunrise_mt7932_FW_4.3020260209"` |
| trailing separator | 16 × `'#'` |

Observed, manufacturing image (132-byte block): outer length `0x0060`, nested length
`0x0059`, padding `0x03`, tag `0x01`, manifest
`"t-neptune-custom-mt7932-ccn9-2405-mfg-MT7932_2405_CCN9_x86_Gen4m_TEST_MODE-20250821172010"`.

`[C]` Firmware version information is available from three places:

1. Common tailer: `chip_info`, `ECO code + 1`, RAM version string, RAM build-date string.
2. Release manifest string (when present) — the human-readable release identity.
3. Patch container: platform tag (`ALPS`), patch version word and build-date string.

`[C]` There is **no** boot-time command that returns the firmware version; the host
the version must be taken from the containers themselves. When the patch is already
resident and therefore not downloaded, its header must still be read from the file if the
version is to be reported.

---

## 4. Cryptographic and integrity handling

### 4.1 What the observed images actually are

`[C]` **Both shipped MT7932 images are plaintext and unencrypted** — the encryption fields
say so (below). `[L]` that they are also **unsigned**: no signature block was found, but the
absence of one in a container this specification does not fully enumerate is not proof.

* Patch: the single section carries `sec_info = 0xFFFF_FFFF` ("no security info"), which
  maps to a download-mode word of `0x8000_0000` — no encryption bit, no key index.
  Payload byte entropy per 4 KiB block is 4.1–7.0 bits/byte, consistent with
  compiled code and tables, not with ciphertext.
* RAM image: every region's feature set has bit 0 clear, so no region is encrypted; the
  computed download-mode word is `0x8000_0000` for all five regions.
* Container CRC fields (patch 16-bit CRC at `0x1E`, patch global CRC at `0x30`, RAM
  common-tailer CRC32) are present but not populated with useful values and are **not**
  checked by the host.
* `[C]` The patch-finish command is issued with its *check-CRC* flag set to `0`.

### 4.2 Per-section security descriptor (`sec_info`, patch container)

`[C]` Encoding and the resulting download-mode word:

| `sec_info` | Meaning | Download-mode word produced |
|---|---|---|
| `0xFFFF_FFFF` | Security info not supported / plaintext | `0x8000_0000` |
| bits 31:24 = `0x00` | Plaintext | `0x8000_0000` |
| bits 31:24 = `0x01` | AES; bits 7:0 = AES key index (only bits 1:0 used) | `0x8000_0009 \| (key<<1)` |
| bits 31:24 = `0x02` | Scramble; bits 15:0 = scramble seed info | `0x8000_0049` |
| any other | unknown → treated as plaintext | `0x8000_0000` |

### 4.3 Download-mode word bit definitions

Identical to public CONNAC2 (`DOWNLOAD_CONFIG_*` / `DL_MODE_*`) `[C]`:

| Bit | Name | Meaning |
|---|---|---|
| 0 | encryption mode | Payload is encrypted |
| 2:1 | key index | Decryption key index for the on-chip decryption accelerator |
| 3 | reset security IV | Reset the decryption IV at the start of this section |
| 4 | working-PDA option | Target the secondary (CR4/DSP) patch-decryption accelerator instead of the MCU one |
| 5 | valid RAM entry | (feature-set only; never set in the transmitted mode word) |
| 6 | encryption mode select | `0` = AES, `1` = scramble |
| 7 | download to EMI | |
| 8 | index-log payload | |
| 31 | **ACK required** | Firmware must return a status event for the target command |

`[C]` Bit 31 is set on **every** MT7932 download-target command, i.e. the host always waits
for a per-section status event.

### 4.4 Where checking happens

`[C]` The host performs **no** decryption and no integrity check of its own. `[L]` that they
are performed on the chip by the boot ROM / patch-decryption accelerator — that follows from
the download-mode word's key-index and encryption-mode fields, not from an observation. The
host only:

* selects the decryption path and key index via the download-mode word;
* optionally asks the MCU to verify a CRC over the downloaded patch, via the check-CRC
  flag in the patch-finish command;
* reads back the resulting status code in the response event (§6.8).

`[C]` The MT7932/MT7923 boot-ROM status vocabulary explicitly includes
`sec boot check fail`, `region check fail`, `RAM entry check fail` and `Section check
fail`, i.e. the boot ROM performs region and entry-point validation independently of the
host. On MT7922 the vocabulary is smaller (`invalid crc`, `decrypt fail`, `sec boot fail`)
— see §11.

### 4.5 Secure-boot protection window

`[C]` The host acquires a **secure-boot semaphore** around the patch-finish and
firmware-start commands, on MT7932 and MT7922 alike. It is *not* held across the bulk data
transfer. Whether the silicon *requires* it is `[U]` — see open question 2.

Sequence, in order:

| Step | Access | Value |
|---|---|---|
| 1 | write chip address `0x7C00_E24C` | `0x1845_1807` |
| 2 (acquire) | poll BAR offset `0x0004_0060` | wait for bit 0 = 1; 5000 iterations spaced 1000 µs → 5 s timeout |
| 2' (release) | write BAR offset `0x0004_0260` | `0x0000_0001` |
| 3 | write chip address `0x7C00_E24C` | `0x1845_184F` |

`[C]` `0x7C00_E24C` is the **PCIe2AP remap control register** whose low 16 bits select the
chip page exposed at BAR window `0x40000` (see the PCIe/register-map section §5.2). Step 1
retargets that window from its default page `0x184F` (WF_MCU_CFG_LS, chip `0x8800_0000`) to
page `0x1807`, and step 3 restores the default; the upper half (`0x1845`, the BAR-`0x50000`
window) is written back unchanged because the register is written as a full 32-bit word.
The semaphore block therefore lives at **AP-bus `0x1807_0060` / `0x1807_0260`**, i.e.
host-view **`0x7C07_0060` / `0x7C07_0260`** (conn-infra semaphore block) — **not** in the
`0x8800_xxxx` range, which is what BAR `0x40000` decodes to only while the default remap
value is in place. `[C]` If the acquire poll times out, the host aborts patch or RAM
download.

`[C]` The host acquires the semaphore *after* the last data scatter packet of the image and
releases it *after* the finish/start command has completed, for both the ROM patch and the
RAM image.

---

## 5. Download transport

### 5.1 Rings

`[C]` MT7932 uses the standard CONNAC2 host WFDMA0 TX rings:

| Hardware TX ring | Use |
|---|---|
| **16** | Firmware download (PDA firmware-download path) |
| 17 | MCU command ring (direct to WM), also used for all boot-ROM commands |

`[C]` The host RX side for boot-ROM responses is host RX ring **0** (the MCU event ring).

`[C]` Both ring numbers are fixed for the PCIe/AXI transport (`cmd=17`, `fwdl=16`); on the
USB variant of the same family they become bulk endpoints instead. Ring 16/17 match
MT7921/MT7922 exactly.

### 5.2 Descriptor and payload framing

`[C]` A firmware-download packet is enqueued exactly like a command packet:

* One PDMA TX descriptor per packet. The descriptor's DW1 is written as
  `((len & 0x3FFF) << 16) | BIT(30)`, i.e. **SDLen0 = len, LastSec0 = 1**, single segment,
  `SDPtr1 = 0`, burst = 0, DMADONE = 0. DW0 = physical address bits 31:0.
* `[C]` DMA addressing is 32-bit; the descriptor's high-address extension field is zero on
  this part.
* `[C]` The buffer length written to the ring is rounded **up to a multiple of 4 bytes**.
* `[L]` The hardware per-descriptor limit is the 14-bit SDLen0 field, i.e. 16383 bytes;
  MediaTek's own re-download path caps at `0x3FF0` (16-byte aligned).

### 5.3 The header prepended to a scatter chunk — the key difference

`[C]` **A firmware-download scatter packet has no header at all.** The DMA descriptor
points directly at raw image bytes.

A normal boot-ROM command, by contrast, is prefixed with a **64-byte header** made of two
32-byte parts:

| Part | Offset | Size | Field |
|---|---|---|---|
| HW MAC TX descriptor | `0x00` | 4 | DW0: TX byte count (bits 15:0) = total packet length; packet-format field = `2` (command) or `3` (PDA firmware download); queue index |
| | `0x04` | 4 | DW1: header format = `1` (command), long format |
| | `0x08` | 24 | DW2..DW7 = 0 |
| Init command header | `0x20` | 2 | `u2TxByteCount` = total packet length − 32 |
| | `0x22` | 2 | `u2PQ_ID` — **transmitted as `0`** on the PCIe path: the header builder never writes this field and the buffer is zero-filled, so the destination is selected by the ring and by `PKT_FT`, not by this field. (The public CONNAC2 definition is `0x8000` = port 1/queue 0, and `0xF800` for the PDA firmware-download port; neither value is emitted here.) |
| | `0x24` | 1 | `ucCID` — boot-ROM command ID (§6) |
| | `0x25` | 1 | `ucPktTypeID` = **`0xA0`** |
| | `0x26` | 1 | reserved / set-query |
| | `0x27` | 1 | `ucSeqNum` — 8-bit sequence number, echoed in the response |
| | `0x28` | 4 | reserved |
| | `0x2C` | 20 | reserved (pads to a full 32-byte TXD shape) |
| Payload | `0x40` | n | command-specific |

`[C]` This 64-byte prefix is byte-identical in size and field placement to upstream
`mt76`'s `struct mt76_connac2_mcu_txd` (32-byte `txd[8]` + 32-byte MCU header). `[C]`
`mt76` reaches the same on-wire result for scatter packets by returning early from its
message builder for `MCU_CMD_FW_SCATTER`, never pushing a TXD. On MT7932 the equivalent is
`ucCID = 0`, for which no TXD is built. **The wire format is
the same: raw payload only.** The chip distinguishes firmware-download data from commands
by the *ring* (16 vs 17), not by any in-band field.

### 5.4 Maximum payload and chunking

| Property | Value |
|---|---|
| Max bytes per scatter packet used on MT7932 | **2048** (host choice) `[C]` |
| Max bytes per scatter packet, upstream `mt76` on PCIe for MT7921/MT7922 | 4096 |
| Alignment of the enqueued length | 4 bytes `[C]` |
| Alignment of the payload split | none required; the last chunk is short `[C]` |

`[C]` The 2048-byte cap is enforced host-side. `[L]` 4096 is also
accepted by this silicon, since the boot ROM is the MT7921/MT7922 one and the transport is
identical; 2048 is a conservative host choice, not a hardware limit.

`[C]` The number of scatter packets is `ceil(section_length / 2048)`; there is no
per-packet acknowledgement and no per-packet sequence checking. Flow control is by TX-ring
resource accounting only.

### 5.5 Alternative direct-write path

`[C]` MT7932 does **not** use one. The "download by dynamic memory map" method defined for
other parts of the family does not apply here, so no section is written through the
programmable register-remap window. This is consistent with §3.3: three of the
five RAM regions live at addresses that are not in the static BAR map at all.

`[C]` Register-window writes are used during boot for exactly two purposes: the
secure-boot semaphore block (§4.5) and the conn-infra remap sanity check (§8.4).

`[C]` The boot ROM does expose an EMI download command pair
(`INIT_CMD_ID_EMI_FW_DOWNLOAD_CONFIG` = `0x23`, `INIT_CMD_ID_EMI_FW_TRIGGER_AXI_DMA` =
`0x24`) and a dynamic-memory-map finish pair (`0x40`/`0x41`); none of these are used on
MT7932 over PCIe.

---

## 6. Download command sequence

All commands below are **boot-ROM ("init") commands**, a namespace entirely separate from
the runtime Wi-Fi command set. They are sent on TX ring 17 with the 64-byte header of
§5.3, `ucPktTypeID = 0xA0`. Every command whose payload is `n` bytes is enqueued with a
total length of `64 + n`.

### 6.0 Command / event ID map

| ID | Command | Payload size | Response event |
|---|---|---|---|
| `0x00` | *(firmware download scatter data — no header)* | ≤2048 | none |
| `0x01` | `INIT_CMD_ID_DOWNLOAD_CONFIG` (target address/length) | 12 | `0x01` command result |
| `0x02` | `INIT_CMD_ID_WIFI_START` (firmware start) | 8 | `0x01` command result |
| `0x03` | `INIT_CMD_ID_ACCESS_REG` | 12 | `0x02` register access |
| `0x04` | `INIT_CMD_ID_QUERY_PENDING_ERROR` | — | `0x03` |
| `0x05` | `INIT_CMD_ID_PATCH_START` | 12 | `0x01` command result |
| `0x07` | `INIT_CMD_ID_PATCH_FINISH` | 4 | `0x01` command result |
| `0x10` | `INIT_CMD_ID_PATCH_SEMAPHORE_CONTROL` | 4 | `0x04` patch-semaphore result |
| `0x21` | `INIT_CMD_ID_LOG_BUF_CTRL` | 20 | `0x08` — **public gen4m only**; neither this command nor event `0x08` is implemented by the analysed host `[U]` for MT7932 |
| `0x22` | `INIT_CMD_ID_QUERY_INFO` | 8 | `0x09` — **public gen4m only**, as above `[U]` |

`[C]` These IDs and the corresponding event IDs are identical to upstream `mt76`
(`MCU_CMD_TARGET_ADDRESS_LEN_REQ` = 1, `MCU_CMD_FW_START_REQ` = 2,
`MCU_CMD_INIT_ACCESS_REG` = 3, `MCU_CMD_PATCH_START_REQ` = 5,
`MCU_CMD_PATCH_FINISH_REQ` = 7, `MCU_CMD_PATCH_SEM_CONTROL` = 0x10; event
`MCU_EVENT_GENERIC` = 1, `MCU_EVENT_MT_PATCH_SEM` = 4). `mt76`'s `MCU_CMD_FW_SCATTER` =
`0xEE` never appears on the wire.

### 6.1 Response event framing

`[C]` A boot-ROM response is fetched from host RX ring 0 as a fixed-size read of
`RXD_size + init_event_size + 4` = `24 + 8 + 4` = **36 bytes**:

| Offset | Size | Field |
|---|---|---|
| `0x00` | 24 | RX descriptor |
| `0x18` | 2 | `u2RxByteCount` |
| `0x1A` | 2 | `u2PacketType` (`0xE000` = event) |
| `0x1C` | 1 | `ucEID` — event ID |
| `0x1D` | 1 | `ucSeqNum` — must equal the request's sequence number |
| `0x1E` | 2 | reserved |
| `0x20` | 4 | event payload (first dword) |

`[C]` The host validates `ucEID` and `ucSeqNum` before looking at the payload; a mismatch
of either is a hard failure.

`[C]` Per-command response timeout: **6000 ms** (a single constant used for every boot-ROM
command).

### 6.2 (a) Prepare a download target

Two different command IDs carry the *same* 12-byte payload:

| ID | Used for |
|---|---|
| `0x05` `INIT_CMD_ID_PATCH_START` | ROM patch sections |
| `0x01` `INIT_CMD_ID_DOWNLOAD_CONFIG` | RAM firmware regions |

Request payload (`INIT_CMD_DOWNLOAD_CONFIG`, 12 bytes, LE):

| Offset | Size | Field |
|---|---|---|
| `0x00` | 4 | Destination chip address |
| `0x04` | 4 | Length in bytes |
| `0x08` | 4 | Download-mode word (§4.3) |

Total enqueued length `0x4C` (76 = 64 + 12) `[C]`.

Response: event `0x01`, one status byte at payload offset 0 (§6.8).

`[C]` **Selection-rule delta:** on MT7932 the choice between `0x05` and `0x01` is made on
the *image kind* (patch vs RAM). Upstream `mt76` selects on the *destination address* (`0x900000` ⇒
`PATCH_START_REQ`). For MT7932 both rules produce the same result, because the patch
destination is `0x0090_0000`. Note that RAM region 0 lives at `0x0090_7C60`, i.e. **inside
the same `0x0090_xxxx` window**; a host implementing `mt76`'s address-based rule must keep
`mt76`'s exact equality test (`addr == 0x900000`) and not a range test.

### 6.3 (b) Stream the data

`[C]` `ceil(len / 2048)` scatter packets on TX ring 16, no header, no per-packet response
(§5). Offsets advance monotonically from the section's file offset.

### 6.4 (c) Finalise a section

`[C]` There is **no per-section finalise command**. The per-section status is delivered
by the response to the download-target command of the *next* section, or by the
patch-finish / firmware-start command at the end. In other words: target-command status
is checked immediately (it is acknowledged before any data is sent, because bit 31 of the
download-mode word is set), and the correctness of the transferred bytes is reported only
by the finish/start command.

### 6.5 (d) Start the patch — `INIT_CMD_ID_PATCH_FINISH` (`0x07`)

Request payload (4 bytes):

| Offset | Size | Field | Observed |
|---|---|---|---|
| `0x00` | 1 | `ucCheckCrc` — ask the MCU to CRC-verify the downloaded patch | `0x00` |
| `0x01` | 3 | reserved (zero) | `00 00 00` |

`[C]` The variant of this structure used on parts that download BT/ZigBee patches places a
patch-type selector in byte 1 (`0` = Wi-Fi, `1` = BT, `2` = Wi-Fi-for-modem, `3` =
ZigBee); on MT7932 the Wi-Fi host sends all-zero, i.e. type Wi-Fi.

Total enqueued length `0x44` (68 = 64 + 4) `[C]`.
Response: event `0x01`, status byte (§6.8). This command both finalises and activates the
patch.

`[C]` The secure-boot semaphore (§4.5) must be held across this command.

### 6.6 The patch semaphore — `INIT_CMD_ID_PATCH_SEMAPHORE_CONTROL` (`0x10`)

Purpose: arbitrate ownership of the ROM patch between the Wi-Fi host, the Bluetooth host
and any previous owner, and discover whether a patch is already resident (e.g. because the
BT stack downloaded it, or because the Wi-Fi subsystem was warm-reset).

Request payload (4 bytes):

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | operation |
| `0x01` | 3 | reserved (zero) |

Total enqueued length `0x44` `[C]`. Response: **event `0x04`**, one status byte at payload
offset 0.

`[C]` **The operation and status encodings are chip-dependent** — this is a genuine MT7932
delta:

| | MT7922 (and MT7921, `mt76`) | **MT7932 / MT7923** |
|---|---|---|
| Operation: release semaphore | `0` | `1` `[U]` |
| Operation: **get semaphore** | `1` | **`2`** `[C]` |
| Status: no semaphore, patch needed (retry) | `0` | `1` `[C]` |
| Status: **no patch needed — patch already downloaded and ready** | `1` | **`2`** `[C]` |
| Status: semaphore granted, patch needed | `2` | `3` `[U]` |
| Status: semaphore released | `3` | `4` `[U]` |

The two values the host actually uses are shifted by **+1** on MT7932. `[C]` The host sends
operation `2` on MT7932 (`1` on MT7922) and treats status `2` (`1` on MT7922) as "skip the
patch download entirely". **The remaining three values are guesses and are marked `[U]`, not
`[L]`:** the host never issues a release and never compares any other status, and the +1-shift
heuristic demonstrably does **not** hold for the command-result enumeration of §6.9 (which
drops `decrypt fail` and moves `unknown` from 4 to 2). Do not implement them from this table.

`[C]` Recommended host handling:

1. Send get-semaphore, wait ≤6000 ms for event `0x04` with matching sequence number.
2. Status = "no patch needed" ⇒ skip §6.2–§6.5 entirely and go straight to the RAM image.
3. Status = "granted, patch needed" ⇒ perform the patch download.
4. Status = "no semaphore, patch needed" ⇒ another owner holds it; retry. MediaTek's
   reference retries every 100 ms up to 50 times before giving up.
5. Any transport failure ⇒ assume the patch is needed.

`[C]` There are sibling command IDs `0x11` (BT patch semaphore) and `0x12` (ZigBee patch
semaphore) with a wider payload that additionally carries a remap address; the Wi-Fi host
on MT7932 does not use them.

### 6.7 Register access during boot — `INIT_CMD_ID_ACCESS_REG` (`0x03`)

Request payload (12 bytes):

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | set (`1`) / query (`0`) |
| `0x01` | 3 | reserved |
| `0x04` | 4 | chip address |
| `0x08` | 4 | data (write value / ignored on read) |

Response: event `0x02`, payload `{ u32 address ; u32 data }`.

`[C]` MT7932 uses this **before any download**, while only the boot ROM is running, to
read `TOP_HVR` (`0x7001_0204`) and `TOP_FVR` (`0x8800_0004`) and thereby fix the hardware,
factory and software/ECO versions (§1.1). Neither is reachable as a plain BAR offset:
`TOP_HVR` has **no static-map entry at all** (the `0x7000_0000` entry covers only
`0x7000_0000`–`0x7000_FFFF`, so it needs a remap-window write), and `TOP_FVR` is reachable
only at BAR `0x40004`, i.e. through the remappable slot-4 window while that window still
holds its default value. Using the boot-ROM command avoids both dependencies.

### 6.8 (e) Start the RAM firmware — `INIT_CMD_ID_WIFI_START` (`0x02`)

Request payload (8 bytes, LE):

| Offset | Size | Field |
|---|---|---|
| `0x00` | 4 | Override flags |
| `0x04` | 4 | Firmware entry chip address |

Override flag bits `[C]` (identical to `mt76`'s `FW_START_*`):

| Bit | Name | Set when |
|---|---|---|
| 0 | override start address | The RAM image declared a valid RAM entry (feature-set bit 5 on some region). Always set for the shipped MT7932 image. |
| 1 | delay calibration | Host wants calibration deferred |
| 2 | working PDA = CR4 | Starting the CR4 image (not applicable to MT7932) |
| 3 | CRC check | Ask the MCU to CRC the RAM image |
| 4 | change decompression temp address | Compressed-image option |

Observed MT7932 request: override = `0x0000_0001`, address = `0x0090_7C60`.

Total enqueued length `0x48` (72 = 64 + 8) `[C]`.
Response: event `0x01`, status byte. `[C]` The secure-boot semaphore (§4.5) must be held
across this command.

### 6.9 Command-result status codes (event `0x01`, payload byte 0)

`[C]` **A second genuine MT7932 delta:** the status vocabulary is different from
MT7921/MT7922. It is *not* a uniform +1 shift — `decrypt fail` disappears, `unknown` moves
from 4 to 2, and six structural checks are added — so re-map it entry by entry from the table
below rather than by adding 1.

| Code | MT7922 (= public `WIFI_FW_DOWNLOAD_*`) | **MT7932 / MT7923** |
|---|---|---|
| 0 | success | *(unused)* |
| 1 | invalid param | **success** |
| 2 | invalid crc | unknown |
| 3 | decrypt fail | invalid param |
| 4 | unknown | invalid crc |
| 5 | timeout | timeout |
| 6 | sec boot fail | sec boot check fail |
| 7 | — | region check fail |
| 8 | — | cmd size check fail |
| 9 | — | RAM entry check fail |
| 10 | — | section check fail |
| 11 | — | FW download flow check fail |
| 12 | — | FW download cmd logic check fail |

`[C]` The host treats **`0` as success on MT7922 and `1` as success on MT7932**; anything
else aborts the boot. Note that the MT7932 vocabulary drops `decrypt fail` and adds six
new structural checks, which is consistent with a boot ROM that validates region
descriptors and the RAM entry point before starting.

### 6.10 Complete ordered boot sequence

`[C]`

```
 1. driver-own handshake; adapter init; conn-infra remap sanity check   (§7.4)
 2. DMA-ring init (TX rings 16 and 17, RX ring 0 at minimum)
 3. publish host shared-memory descriptors to the MCU                   (§8.4)
 4. enable firmware download (no-op on PCIe)                            (§7.3)
 5. enable interrupts
 6. read TOP_HVR / TOP_FVR                            cmd 0x03 -> evt 0x02   (§6.7)
    -> hardware / factory / software(ECO) versions -> image file selection
 7. reset TX resource accounting
 8. patch semaphore: get                              cmd 0x10 -> evt 0x04   (§6.6)
    if status == "no patch needed": go to 12
 9. for each patch section with (section_type & 0xFFFF) == 2:
       target command                                 cmd 0x05 -> evt 0x01
       scatter data, <=2048 B per packet, ring 16
10. acquire secure-boot semaphore                                       (§4.5)
11. patch finish                                      cmd 0x07 -> evt 0x01
    release secure-boot semaphore
12. for each RAM region with feature bits 6 and 7 clear:
       target command                                 cmd 0x01 -> evt 0x01
       scatter data, <=2048 B per packet, ring 16
13. acquire secure-boot semaphore                                       (§4.5)
14. firmware start, entry = region with feature bit 5  cmd 0x02 -> evt 0x01
    release secure-boot semaphore
15. disable firmware download (no-op on PCIe)
16. poll the firmware-ready indication until it reads "ready"           (§7.1, §7.1.1)
    MT7932: PCI config 0x48C[19:16] == 2   (MT7922: chip 0x7C06_00F0 bits[1:0] == 3)
17. post-boot exchange                                                  (§8)
```

---

## 7. Boot-completion signalling

### 7.1 Firmware-ready register

`[C]`

| Property | Value |
|---|---|
| Register (chip address) | **`0x7C06_00F0`** (`CONN_CFG_ON` block base `0x7C06_0000` + `0xF0`, "conn-on misc") |
| Static BAR offset | `0x000E_00F0` (`0x7C06_0000` maps to BAR `0x000E_0000`, 64 KiB) |
| Ready field shift | `0` |
| Ready mask | `0x0000_0003` |
| Ready condition | `(value & 0x3) == 0x3` |
| Poll interval | **5 ms** |
| Timeout | **5000 ms** |
| Access width | 32-bit |

Bit meanings (public `ENUM_WIFI_FUNC`) `[C]`:

| Bit | Name |
|---|---|
| 0 | `WIFI_FUNC_INIT_DONE` |
| 1 | `WIFI_FUNC_N9_DONE` — the RAM firmware (N9 MCU) is running |
| 2 | `WIFI_FUNC_CR4_READY` — not used on MT7932 (no CR4) |
| 3 | `WIFI_FUNC_DUMMY_REQ` |

`[C]` The ready condition is bits 1:0 only (`WIFI_FUNC_NO_CR4_READY_BITS`), identical to
MT7921/MT7922. The **same register and mask, tested for the value being clear**, is the
power-off / Wi-Fi-function-off indication.

#### 7.1.1 MT7932 delta — the ready signal is read from configuration space, not this register

`[C]` **The register above is the MT7922 branch. On MT7932 the host must not use it.** The
readiness poll is gated at run time on the PCI device ID, and it selects a different source:

| Device ID | Ready source | Ready condition | Off condition |
|---|---|---|---|
| `0x7922` | chip register `0x7C06_00F0`, bits `[1:0]` | `(v & 0x3) == 0x3` | `(v & 0x3) == 0` |
| **`0x7932`** (and `0x7923`) | **PCI configuration dword `0x48C`, bits `[19:16]`** | **`== 2`** | `!= 2` (initial state reads `0`) |

Both branches use the **same cadence** — 5 ms poll interval, 5000 ms total budget — and sit
at the **same point in the boot sequence** (§6.10 step 16, i.e. after the firmware-start
command has been acknowledged). They are two encodings of one checkpoint, not two
checkpoints. `[C]`

The `0x7C06_00F0` values (`sw_sync0` = `0x7C06_00F0`, ready bits `0x3`, shift `0`) are still
present in this part's hardware-configuration record and are byte-identical to MT7922's;
they are simply not consulted on the MT7932 path. `[C]`

Practical consequence for a driver: the field is read **before ownership is taken** and
**before any MMIO mapping is established** — the post-FLR "MCU back in its initial state"
test (`bits [19:16] == 0`) uses it in exactly that state. `[C]` That it is also independent of
the Wi-Fi-domain clocks is `[L]`.

`[U]` Only values `0` and `2` of `0x48C[19:16]` have been observed; the intermediate values
presumably encode ROM / patch-loaded / RAM-code stages and would give a finer-grained boot
progress indication if decoded.

`[C]` On timeout the host reads the register once more, dumps the interrupt-status
register, and triggers a whole-chip reset.

### 7.2 Patch-running indication

`[C]` The host uses **no register bit for "the patch is running"**; the only indication it
takes is the success status in the response event to the patch-finish command (§6.9). `[L]`
that no such bit exists, and `[L]` that the firmware-ready indication stays clear until the
RAM firmware has started.

### 7.3 The "firmware download enable" control

`[C]` On MT7932 over PCIe this control performs **no register write**, so the assert/de-assert
around the download has no device-side effect `[L]`. (On the USB variant of CONNAC2 the
equivalent step programs the bulk-endpoint configuration.) The de-facto enable on PCIe is
simply that TX ring 16 exists and is serviced; that the boot ROM routes anything arriving on
ring 16 into the patch-decryption/download engine is `[L]`.

Consequently a PCIe host driver needs **no** explicit enable/disable step; it only needs
ring 16 configured before the first target command and may leave it configured afterwards.

### 7.4 Other boot-time polls

| Poll | Register | Condition | Cadence |
|---|---|---|---|
| Wi-Fi subsystem software-init done | chip `0x7C00_0140` | bit 4 set; `0xFFFF_FFFF` read = bus down | 3 attempts, 100 ms apart `[C]` |
| Secure-boot semaphore | BAR `0x40060` with remap slot 4 = `0x1807` (chip `0x7C07_0060`) | bit 0 set | 5000 × 1000 µs `[C]` |
| Conn-infra remap sanity | chip `0x7C00_E254` | write `0x1805_1848`, wait 2 µs, read back; bits 31:16 must read `0x1805` | once, before MCU access `[C]` |
| MCU-in-init / link recovery | PCI config space `0x48C` | bits 19:16 must be `0` | 50 × 50 µs `[C]` |

The `0x7C00_E24C` / `0x7C00_E254` pair are the conn-infra **PCIe2AP remap selector**
registers; their field layout **is** established — two 16-bit selectors per register, each
value being the target chip base shifted right by 16 (PCIe/register-map section §5.1). `[C]`
What remains open is only which blocks the sibling selector values `0x1845` and `0x1846`
reach, since those halves are written back unchanged and never used. `[U]`

---

## 8. Post-boot initial exchange

### 8.1 Unsolicited traffic from the firmware

`[C]` After the firmware-start command succeeds the host polls the readiness indication
(§7.1.1) until it reads "ready"; no unsolicited event is waited for. `[L]` that the firmware
sets the indication immediately on start. `[L]` Any capability event the firmware chooses
to push before the host's first query is tolerated by the normal event path but is not
part of the required handshake.

### 8.2 What the host must send before normal operation

`[C]` In order:

| Step | What | Notes |
|---|---|---|
| 1 | Reset TX resource accounting | The firmware's page/resource pool is re-advertised after start |
| 2 | **Dummy command** | A command carrying only the runtime 64-byte command header and no payload, sent on the command ring, used to prime the firmware's command path. Enqueued length equals the runtime command-header size (`0x40`). |
| 3 | **Query NIC capability (V2)** | Returns the firmware's capability TLV set: resource/page counts, feature bits, MAC address, DBDC capability, etc. A V1 query exists for older firmware; MT7932 firmware advertises V2 support. |
| 4 | Clear firmware-own (allow the MCU to sleep) | |
| 5 | Apply resource information, basic config, MLME config, network address | Derived from the capability reply |

`[C]` Until step 3 has completed the host does not know the firmware's TX resource
geometry and must not submit data traffic.

`[C]` The version reads of §6.7 happen *before* the firmware download, not after; the
post-boot exchange does not repeat them.

### 8.3 Runtime-command header (for contrast)

`[C]` After boot, commands use a 64-byte header of the same overall size as the boot-ROM
one but with the runtime field set (`cid`, `ext_cid`, `set_query`, `s2d_index`,
`ext_cid_ack`), sent on the same ring 17. Boot-ROM command IDs and runtime command IDs are
disjoint namespaces.

### 8.4 Host shared-memory publication (before download)

`[C]` Before the firmware download starts, the host publishes the physical addresses and
sizes of host-DRAM buffers to the MCU by writing a fixed block of conn-MCU configuration
registers. The register addresses are per-chip; MT7932 uses the same set as MT7922:

Write order and contents (32-bit DMA-mask case, which is the MT7932 case) `[C]`:

| # | Register (chip address) | Written with |
|---|---|---|
| 1 | `0x7C05_3C28` | `0x0000_0001` — enable / doorbell; written **first** |
| 2 | `0x7C05_3A38` | control flags: `BIT(0)` = 64-bit host addressing in use (**clear** on MT7932), `BIT(1)` = share-info valid |
| 3 | `0x7C05_3A3C` | **low 32 bits of the share-info block physical address** |
| 4 | `0x7C05_3A30` | low 32 bits of a second host buffer address |
| 5 | `0x7C05_3A54` | low 32 bits of a third host buffer address |
| — | `0x7C05_3A34`, `0x7C05_3A58` | high 32 bits of the pairs at `0x3A30` / `0x3A54`; written **only** when the host DMA mask is wider than 32 bits, so never on MT7932 |
| — | `0x7C05_3A50`, `0x7C05_3A5C` | present in the per-chip register list but never written `[C]`, purpose `[U]` |

`[C]` A further per-chip constant `0x0320_0300` accompanies these register addresses but is
not written to any register; it is a pair of 16-bit byte counts, `0x0300` and `0x0320`,
matching the two host buffer sizes the power/reset section documents (`[L]` for the
identification). `[C]` The DMA address is checked to fit in 32
bits; a 64-bit address is a host error on this part. `[L]` This block is what gives the firmware
somewhere to write its log stream and coredump (§9).

---

## 9. Firmware logging and coredump interfaces

### 9.1 Log delivery

`[C]` Three mechanisms exist:

1. **Runtime "firmware log to host" command** — runtime command ID `0xC5`, 4-byte payload,
   selects the log destination/level. This is the switch that makes the firmware emit log
   records to the host at all.
2. **Boot-ROM log buffer control** — `INIT_CMD_ID_LOG_BUF_CTRL` (`0x21`), 20-byte payload
   `{u32 mcu_addr, u32 wifi_addr, u32 bt_addr, u32 gps_addr, u8 type, u8 rsv[3]}`, reply
   event `0x08` `{u8 type, u8 status, u8 rsv[2], u32 address, u32 rsv}`. `type` bit 0 =
   fetch the log-buffer control-block base address; bits 1..4 = update the MCU / Wi-Fi /
   BT / GPS read pointers. This gives the host the address of a per-subsystem log ring and
   lets it advance the read pointer. MT7932's Wi-Fi host does not use it during boot.
3. **Host shared-memory region** published at `0x7C05_3A3C` (§8.4). `[C]` The structure the
   firmware writes there carries a status word whose **bit 0 is a "coredump pending"
   flag**; the host tests and clears it. `[L]` The same region carries the firmware log
   stream.

### 9.2 Index-log translation file

`[C]` Purpose: the firmware does not transmit format strings. Each log record carries a
32-bit **index**, which is a **byte offset into the index-log file**. The file is a flat
concatenation of records:

```
record := 0x24 0x26 ("$&") , "<source-file>:<line>:<printf-format>" , 0x00 , pad-to-4-bytes
file   := record* , 14-byte ASCII build timestamp
```

Observed for MT7932: 192 778 bytes, 3283 records, first record at file offset 0, trailing
timestamp `"20260331234844"` (no NUL). The index transmitted by the firmware points at the
**first byte after** the `"$&"` marker, i.e. at the source-file name. `[C]` The host parses
forward from that offset, splitting on the first two `:` characters to obtain file, line
and format string, then applies the record's arguments to the format string.

`[C]` The file must match the RAM image build; the host derives its name from chip ID and
ECO version (§1.1) and loads it whole into memory.

### 9.3 Firmware log record format (on the wire)

`[C]` Header:

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | STX marker |
| `0x01` | 1 | bits 2:0 = version type; bits 7:3 = argument count |

Three version types are decoded:

| Version type | Header length | Index field | Argument encoding | Record length |
|---|---|---|---|---|
| `0` | 12 | `u32` at `0x08` | fixed 32-bit | `12 + 4×argc` |
| `3` | 16 | `u32` at `0x0C` | fixed 32-bit | `16 + 4×argc` |
| `4` | 16 | `u32` at `0x0C` | variable-length, 1–5 bytes each, continuation bit `0x80` in every byte but the last | `16 + Σ(arg lengths)`, scanned |

`[C]` Any other version type is rejected. `[C]` Version type `4`'s varint packing is the
compact form and is what the MT7932 firmware emits; the scan is bounded at `5 × argc`
bytes.

### 9.4 Coredump

`[C]` Trigger (host-initiated assert): write `0x8000_0000` to chip address **`0x7403_1484`**
(PCIe MAC block `0x7403_0000 + 0x1484`; static BAR offset `0x0001_1484`). `[L]` that this
makes the firmware take an exception and produce a dump. `[C]` The host issues it only when
firmware is running and no dump is already in progress.

`[C]` Collection: the firmware raises the coredump-pending bit in the host shared-memory
status word (§9.1); the host drains the dump from that region. The dump stream is
delimited in the log by `;;[CONNSYS] coredump start` / `;coredump end` and contains
`====N9 ASSERT_END====` (and `====Cr4 ASSERT_END====` on parts with a CR4). A dump that
does not complete is reported as `[DUMP_N9] Core dump timeout!!!` and escalates to a whole
chip reset.

`[C]` A firmware-triggered path also exists: the MCU raises a software interrupt to the
host through the WFDMA software-interrupt mailbox at chip address **`0x7C02_41F0`**
(WFDMA0; `0x7C02_51F0` for WFDMA1). The host reads the register, writes the same value
back to clear it, and decodes the reason bits; bits 5:2 non-zero indicates a
firmware-signalled condition. The reset reasons that map to this path are firmware assert
done, firmware assert timeout and watchdog reset.

`[U]` The exact coredump record framing inside the shared region was not established.

### 9.5 Assert / coredump event — the mandatory host side of the handshake

The bulk of a firmware assertion is **not** delivered through the shared region: it arrives
as a stream of ordinary MCU events with `ucEID = 0xF0` (`EVENT_ID_ASSERT_DUMP`, §6 of the
MCU-protocol section) `[C]`. A driver that merely logs and discards these events will hang the
part, because the firmware does not appear to restart itself after an assertion — the host
must both acknowledge and, at the end, reset. `[L]` for the firmware behaviour; `[C]` for the
host actions listed below, which are the ones the analysed host performs.

**Event framing.**

| Field | Location | Meaning |
|---|---|---|
| CPU selector | event header byte `0x0B` | `0` = main Wi-Fi MCU dump; non-zero = second-CPU dump (present in the decoder, never produced by MT7932, which has one MCU) `[C]` |
| Payload | event body, i.e. from offset `0x0C` | **ASCII text**, not binary. Length = `u2Length` − event-header length `[C]` |

The payload is a plain text log stream carrying these in-band markers `[C]`:

| Marker | Meaning |
|---|---|
| `;;[CONNSYS] coredump start` | start of the binary-encoded dump body |
| `;more log added here` | end of the leading banner |
| `====N9 ASSERT_DUMPSTART====` | banner emitted around the first record |
| `;coredump end` | **last** record of the dump |

**Required host actions, in order** `[C]`:

1. **On the first `EVENT_ID_ASSERT_DUMP` of a dump**, write **`0x0000_0800`** to the
   host→MCU doorbell register (chip `0x7403_1484`, §9.6). This is the acknowledgement that
   the host is present and collecting; it is a single write, not repeated per chunk.
2. Stop the firmware health-monitor / watchdog timer for the duration of the dump, so the
   watchdog does not race the dump to trigger a different recovery level.
3. Open a sink for the dump and append every event payload to it in arrival order. The
   events are ordinary MCU events on the event RX ring and are subject to the same ring
   refill obligations as any other event: **if the host stops refilling the event ring the
   dump stalls and is lost.**
4. **Re-arm a 5 000 ms inactivity timer on every received chunk.** Expiry before
   `;coredump end` is the *assert-dump-timeout* condition (recovery reason 4 in §8.4 of the
   power/reset section). `[C]`
5. On the chunk containing `;coredump end`: stop the inactivity timer and **initiate a
   reset** — a whole-chip (L0) reset in the normal case, or the watchdog-reset path if a
   watchdog reset was already pending. This is recovery reason 3 (*firmware assert, dump
   complete*). `[C]`

**A dump is treated as not optional.** The host provides no "ignore and continue" path `[C]`.
That the firmware is halted in its exception handler from the first event onwards, that no
further commands are serviced, and that the only exit is a host-driven reset are `[L]` —
firmware behaviour inferred from the host's handling and from the firmware's own log sites.

**Host-initiated assert.** Writing `0x8000_0000` to the same doorbell (§9.4) forces the
firmware into the identical path, so the collection state machine above must already be in
place before that write is issued. `[C]`

### 9.6 Host→MCU doorbell register (`0x7403_1484`)

A single 32-bit write-only register in the PCIe-MAC block (static BAR offset
`0x0001_1484`) carries out-of-band host→firmware notifications that are **not** part of the
MCU command ring and are still delivered when the command path is dead. Observed bit
assignments `[C]`:

| Value written | Meaning |
|---|---|
| `0x0000_0800` | Host has begun collecting an assert/coredump (§9.5 step 1) |
| `0x0000_2000` | **Host is going away** — written immediately before a hibernate / whole-chip (L0) reset, after the host has stopped its TX/RX rings. Tells the firmware not to expect further host service. |
| `0x8000_0000` | Force a firmware exception (host-initiated assert, §9.4) |

No other bits of this register are used. `[U]`

Constraint: the register must not be written while the host is in the post-suspend state in
which MMIO is refused. `[C]`

---

## 10. Timing summary

| Item | Value | Confidence |
|---|---|---|
| Boot-ROM command response timeout | 6000 ms | `[C]` |
| Firmware-ready poll interval | 5 ms | `[C]` |
| Firmware-ready timeout | 5000 ms | `[C]` |
| Secure-boot semaphore acquire | 1000 µs × 5000 = 5 s | `[C]` |
| Wi-Fi subsystem software-init poll | 100 ms × 3 | `[C]` |
| Conn-infra remap read-back delay | 2 µs | `[C]` |
| MCU-in-init link poll | 50 µs × 50 | `[C]` |
| Patch-semaphore retry cadence (reference) | 100 ms, ≤50 attempts | `[L]` |

---

## 11. Delta summary versus public MT7921 / MT7922

### Identical (no delta) — implement exactly as `mt76`/`gen4m` do

* `[C]` Patch container v2 layout, magic `0xFFFF_FFFF`, big-endian global descriptor and
  section map, `PATCH_SEC_TYPE_BIN_INFO = 2`, `bin_type` values, patch destination
  `0x0090_0000`.
* `[C]` RAM container: 36-byte common tailer, 40-byte region tailer, region-count limit 10,
  feature-set bit meanings, release-info block with `'#'` separators and tag `0x01`/`0x02`
  manifest records.
* `[C]` Download-mode word bit definitions, including the always-set "ACK required" bit 31.
* `[C]` Boot-ROM command IDs `0x01`, `0x02`, `0x03`, `0x05`, `0x07`, `0x10` and event IDs
  `0x01`, `0x02`, `0x04`; the 64-byte command header (32-byte HW TXD + 32-byte init header,
  `ucPktTypeID = 0xA0`, `u2PQ_ID` transmitted as `0`); the headerless scatter packet.
* `[C]` Hardware TX ring 16 = firmware download, 17 = MCU command; host RX ring 0 = MCU
  events; 32-bit DMA; PDMA descriptor `SDLen0`/`LastSec0` framing.
* `[C]` Firmware-ready **cadence** — 5 ms poll, 5 s timeout — and the `0x7C06_00F0` /
  mask `0x3` / shift `0` values carried in the hardware-configuration record. The **source
  actually polled differs**; see delta 10 below and §7.1.1.
* `[C]` Firmware-start override flag layout; the RAM entry point taken from the region
  whose feature-set bit 5 is set.
* `[C]` Version registers `TOP_HCR 0x7001_0200`, `TOP_HVR 0x7001_0204`,
  `TOP_FVR 0x8800_0004`; chip-ID register at `top_cfg_base + 0x1008` with
  `top_cfg_base = 0x8002_0000`.

### Genuine MT7932 deltas

1. `[C]` **Patch-semaphore encoding differs.** Get-semaphore is operation **`2`**
   (not `1`); "patch already downloaded, skip" is status **`2`** (not `1`); "no semaphore,
   retry" is status **`1`** (not `0`). These three are established. This is selected at run
   time by PCI device ID: only device `0x7922` uses the legacy encoding; `0x7932` and `0x7923`
   use the other. The remaining operation and status values are `[U]` (§6.6).
   A driver reusing `mt76`'s `PATCH_NOT_DL_SEM_FAIL/PATCH_IS_DL/PATCH_NOT_DL_SEM_SUCCESS`
   constants unchanged **will misread the reply** and either re-download the patch
   needlessly or, worse, skip a needed download.
2. `[C]` **Command-result status vocabulary is different.** Success is **`1`**, not `0`.
   Six additional structural failure codes exist (region check, cmd-size check,
   RAM-entry check, section check, download-flow check, download-cmd-logic check),
   `decrypt fail` is gone and `unknown` moves to code 2 — so it is **not** a uniform +1 shift.
   Same device-ID selection.
3. `[C]` **Secure-boot semaphore around finish/start.** Not present in upstream `mt76` for
   MT7921/MT7922. The host must bracket the patch-finish and firmware-start commands with
   an acquire/release on the conn-infra semaphore block (`0x7C07_0060` / `0x7C07_0260`,
   reached at BAR `0x40060` / `0x40260`), framed by writes of
   `0x1845_1807` / `0x1845_184F` to the remap control register `0x7C00_E24C`. `[U]` Whether this is strictly required
   by MT7932 silicon or a defensive measure inherited from a newer platform needs hardware
   tracing.
4. `[C]` **Conn-infra remap sanity check** at `0x7C00_E254` (write `0x1805_1848`, read back
   `0x1805` in bits 31:16) before MCU access. Not in upstream `mt76`.
5. `[C]` **Host shared-memory publication** through `0x7C05_3A30..0x7C05_3A5C` +
   `0x7C05_3C28 = 1` before download. Not in upstream `mt76` for MT7921/MT7922.
6. `[C]` **Scatter chunk size 2048** rather than upstream's 4096 on PCIe. `[L]` Cosmetic;
   4096 should work.
7. `[C]` **Target-command selection rule** is by image kind rather than by destination
   address. Equivalent for MT7932 but see the `0x0090_7C60` caveat in §6.2.
8. `[C]` The shipped MT7932 RAM image has **5 regions** including one at `0xE020_0050`
   (393 KB) and one at `0x0040_4400`, a different region set from the MT7921/MT7922
   images; the container format is unchanged.
9. `[C]` **The firmware-ready signal moves out of MMIO into configuration space.** MT7922
   polls chip register `0x7C06_00F0` bits `[1:0]` for `0x3`; MT7932 polls **PCI
   configuration dword `0x48C` bits `[19:16]` for the value `2`** (§7.1.1). Same cadence,
   same point in the sequence, different register. `[L]` that a driver keeping the
   `CONN_ON_MISC` poll would hang for the full 5 s budget after every otherwise-successful
   firmware start and then escalate to a chip reset — the MT7932 behaviour of those bits has
   not been observed.
10. `[C]` Everything else — bus parameters, the firmware-download procedure, the
   TX-descriptor format and the MCU handshake registers — is byte-identical to MT7922's in
   the hardware-configuration record. `[L]` that no other boot-path difference therefore
   exists: a silicon-level difference that the record does not express would not show up here.

---

## 12. Open questions / needs hardware tracing

1. `[U]` The patch-semaphore values for "release" (operation) and for "granted, patch
   needed" / "semaphore released" (status) on MT7932 are **guesses** from a +1 shift that is
   known not to hold for the sibling enumeration; they do not occur on the normal boot path.
   Confirm by sending a release and by racing the BT stack for the semaphore.
2. `[U]` Whether the MT7932 boot ROM *requires* the secure-boot semaphore (§4.5), or
   whether patch-finish / firmware-start succeed without it. Test by omitting it.
3. `[U]` What the sibling selector values `0x1845` (BAR window `0x50000`) and `0x1846`
   (BAR window `0x60000`) reach. The *layout* of these words is established (§7.4); only
   these two halves, which are written back unchanged and never used, are unidentified.
4. `[U]` What the second and third host buffers published at `0x7C05_3A30` and
   `0x7C05_3A54` are for (the block at `0x7C05_3A3C` is the share-info block), and whether
   firmware boots at all without them being written.
5. `[U]` Whether the boot ROM enforces the 2048-byte scatter limit or accepts 4096 (and up
   to the 14-bit descriptor limit) on MT7932.
6. `[U]` Whether `0x0090_7C60` (RAM region 0) is accepted by `INIT_CMD_ID_PATCH_START` as
   well as by `INIT_CMD_ID_DOWNLOAD_CONFIG`, i.e. whether the boot ROM distinguishes the
   two commands by more than the address range.
7. `[U]` Whether the firmware pushes any unsolicited event between the ready-bit assertion
   and the host's first query, and whether skipping the dummy command is harmful.
8. `[U]` Coredump record framing inside the host shared-memory region, and the meaning of
   software-interrupt reason bits 5:2 at `0x7C02_41F0`.
9. `[U]` Whether the patch-finish check-CRC flag actually causes the MCU to verify
   anything, given that the shipped containers carry no populated CRC.
10. `[U]` Whether an encrypted or signed MT7932 image variant exists in the field; the
    shipped images are plaintext, so the AES/scramble paths and the key-index field are
    untested for this part.


---

# MT7932 — MCU Command / Event Protocol

## Scope

This document specifies the host↔firmware message interface of the MediaTek MT7932
Wi-Fi function: the exact byte layout of command frames placed on the MCU TX ring, the
exact byte layout of event frames returned on the RX rings, the command and event
identifier spaces the shipping firmware implements, the request/response correlation
rules, status codes, timeouts, flow control and ring routing, and the reduced command set
that is legal before the RAM firmware is running. MT7932 presents a CONNAC2 (mt792x-class)
MCU interface that is **byte-for-byte identical to MT7922**; every structural statement
below therefore also holds for MT7922/MT7921 unless a delta is called out. Deltas that are
specific to this part are flagged inline. Confidence markers: `[C]` confirmed (directly observed, unambiguous), `[L]` likely
(strongly implied and consistent with public silicon of the same family), `[U]` unverified
(needs hardware tracing).

Everything in this document concerns the **WM (Wi-Fi MCU)** processor. MT7932 has no
second ("WA") Wi-Fi offload CPU exposed to the host — see §7.

---

## 1. Command frame layout on the wire

### 1.1 Overall structure

A normal (post-boot) MCU command occupies one contiguous DMA buffer laid out as:

```
 offset   size   content
 0x00     32 B   CONNAC2 hardware TX descriptor (TXD), long format
 0x20     32 B   MCU command header
 0x40     N B    command payload (command-specific body)
```

Total header size = **0x40 = 64 bytes** `[C]`. This is the chip's "command TX header size"
and is what the host must reserve in front of every command body. The buffer is zero-filled before the header fields are written, so
every field not listed below is transmitted as 0 `[C]`.

### 1.2 TX descriptor prefix used for MCU traffic

Only three TXD fields are meaningful for MCU traffic; the remaining 30 bytes of the TXD
are transmitted as zero `[C]`.

| TXD word | Bits    | Field            | Value for MCU commands |
|----------|---------|------------------|------------------------|
| DW0      | 15:0    | `TX_BYTE_CNT`    | Total buffer length **including** the 32-byte TXD |
| DW0      | 22:16   | `ETH_TYPE_OFFSET`| 0 |
| DW0      | 24:23   | `PKT_FT` (packet format) | `2` = command packet; `3` = PDA firmware-download packet |
| DW0      | 31:25   | `Q_IDX`          | 0 |
| DW1      | 17:16   | `HDR_FORMAT`     | `1` = `HEADER_FORMAT_COMMAND` |

`PKT_FT` encoding (CONNAC2, same as public mt792x): `0` = cut-through data, `1` =
store-and-forward data, `2` = command, `3` = PDA/firmware-download. `HDR_FORMAT`
encoding: `0` = 802.3, `1` = command, `2` = 802.11, `3` = 802.11 extended. `[C]`

Note the deliberate redundancy: `DW0[15:0]` counts the whole buffer, whereas the length
field inside the MCU header (below) counts the whole buffer **minus 32** — i.e. it
excludes the TXD. Both are written from the same source value. `[C]`

### 1.3 MCU command header (legacy + extended schemes)

This is the public CONNAC2 `WIFI_CMD` / `mt76_connac2_mcu_txd` structure. All fields are
little-endian. Offsets are relative to the start of the whole buffer (i.e. the TXD
occupies 0x00–0x1F).

| Offset | Size | Field | Description | Written by host? |
|--------|------|-------|-------------|------------------|
| 0x20 | 2 | `u2Length` | Total buffer length **minus 32** (MCU header + payload) | yes `[C]` |
| 0x22 | 2 | `u2PqId` | Port/queue id. Transmitted as **0** on the PCIe/WFDMA path. Public Linux mt76 writes `MCU_PQ_ID(port,q)` here; the field is a don't-care on PCIe because the ring index selects the destination. | no (0) `[C]` |
| 0x24 | 1 | `ucCID` | Command ID (legacy command space, §5.1) | yes `[C]` |
| 0x25 | 1 | `ucPktTypeID` | Constant **0xA0** (`CMD_PACKET_TYPE_ID`) | yes `[C]` |
| 0x26 | 1 | `ucSetQuery` | Direction: `1` = SET, `0` = QUERY | yes `[C]` |
| 0x27 | 1 | `ucSeqNum` | Sequence number, 1…255 (see §1.5) | yes `[C]` |
| 0x28 | 1 | `ucD2B0Rev` | Reserved / hardware may overwrite | no (0) `[C]` |
| 0x29 | 1 | `ucExtenCID` | Extended command ID; `0` for pure legacy commands | yes `[C]` |
| 0x2A | 1 | `ucS2DIndex` | Source→destination routing index. Always **0** (`S2D_INDEX_CMD_H2N`, host→WM) on the PCIe host path | yes `[C]` |
| 0x2B | 1 | `ucExtCmdOption` | Response-required flag: non-zero ⇒ firmware must return an event even for a SET | yes `[C]` |
| 0x2C | 1 | `ucCmdVersion` | Command-structure version. Left at **0** | no (0) `[C]` |
| 0x2D | 3 | reserved | 0 | no |
| 0x30 | 16 | reserved | 0 (four reserved dwords) | no |
| 0x40 | N | payload | command body | yes |

`ucS2DIndex` encoding (public CONNAC2 definition, listed for completeness; only value 0 is
ever emitted on the PCIe host path `[C]`):

| Value | Meaning |
|-------|---------|
| 0 | `CMD_S2D_IDX_H2N` — host → WM |
| 1 | `CMD_S2D_IDX_C2N` — WA → WM |
| 2 | `CMD_S2D_IDX_H2C` — host → WA |
| 3 | `CMD_S2D_IDX_H2N_AND_H2C` — host → WM and WA |

**There is no checksum field in the legacy/extended command header.** The unified-command
header (§2.4) defines one, but that scheme is not used by this part's firmware. `[C]`

### 1.4 Alignment and padding

* The command **payload buffer is allocated rounded up to a multiple of 4 bytes** and the
  padding is zero-filled `[C]`.
* The DMA write of an initialisation-time command rounds the transferred length up to a
  multiple of 4 (`(len + 3) & ~3`) `[C]`.
* No other alignment constraint is imposed by the header. Individual command bodies are
  themselves 4-byte-aligned structures.
* Maximum total buffer (TXD + header + payload) is **0x640 = 1600 bytes**, enforced twice:
  once at command-composition time and again by a per-chip size check on the send path,
  which exempts only 802.11 management frames pushed through the command ring. Maximum
  command **payload** is therefore **0x600 = 1536 bytes** `[C]`.

### 1.5 Sequence number

* Width: **8 bits**, header offset 0x27 `[C]`.
* Allocation: a single global counter incremented by 1 per outgoing command, wrapping
  255 → **1**. **Value 0 is never allocated** and is reserved to mark firmware-initiated
  (unsolicited) events `[C]`.
* Collision avoidance: a candidate value that is already in use by an outstanding command
  must be skipped. (Retrying at most 20 times before failing is a host choice.) `[C]`
* The same counter is shared by initialisation-time commands, legacy commands and extended
  commands `[C]`.
* The firmware echoes `ucSeqNum` unchanged in the event header of the response `[C]`.

### 1.6 Set/query and response-required semantics

| `ucSetQuery` (0x26) | `ucExtCmdOption` (0x2B) | Firmware behaviour |
|---|---|---|
| 0 (QUERY) | any | Firmware always returns an event carrying the queried data |
| 1 (SET) | 0 | Fire-and-forget; no event is generated |
| 1 (SET) | ≠0 | Firmware returns a completion event |

The host arms its response timer exactly when `ucSetQuery == 0 || ucExtCmdOption != 0` `[C]`.

---

## 2. Command schemes

Three command schemes exist in the CONNAC2 family. This part's firmware implements the
first two; the third is not used.

### 2.1 Scheme A — legacy command (`ucCID` only)

`ucExtenCID = 0`. The command body starts at buffer offset 0x40. Command IDs are listed in
§5.1. Responses arrive as an event whose `ucEID` is the corresponding event ID (usually,
but not always, numerically different from the command ID). `[C]`

### 2.2 Scheme B1 — "Layer-0" extended command (`ucCID = 0xED`)

`ucCID = 0xED` (`CMD_ID_LAYER_0_EXT_MAGIC_NUM`), `ucExtenCID` = the extended command ID.
The body still starts at 0x40. Responses arrive as event `ucEID = 0xED`
(`EVENT_ID_LAYER_0_EXT_MAGIC_NUM`) with the extended event ID at event-header offset
0x08; the extended event ID normally equals the extended command ID. Extended command IDs
are listed in §5.2. `[C]`

### 2.3 Scheme B2 — "Layer-1" extended command (`ucCID = 0xEA`)

`ucCID = 0xEA`, `ucExtenCID` = a sub-command ID in a **separate** number space from
scheme B1. This family carries the firmware-resident MLME / connection-offload interface
(connect, disconnect, PMK cache, SAE/authentication offload, BSS transition management,
roaming cache, blacklist). Responses and notifications arrive as events `ucEID = 0xEA` or
`ucEID = 0xEE`; both event IDs are routed to the same handler. Extended event ID `0x11`
inside that family is the WPA/connection-status report; all other sub-event IDs are passed
to the host's firmware-supplicant interface. `[C]`

This 0xEA family is **not present in the public gen4m header set** (which reserves only
`0xEE` as "layer-1 magic"). It is a genuine extension of the CONNAC2 command space in the
firmware shipped for MT7922/MT7932. `[C]` — see §5.3.

### 2.4 Scheme C — unified ("UNI") command with TLV body — **not used by this part**

The CONNAC2 silicon/firmware family defines a second command header with a 16-bit command
ID and a tag-length-value body. **On MT7932 no unified-command descriptor is produced, no
unified-command header size is configured, and no unified command or unified event is
exchanged in either direction.** `[C]` Whether the RAM firmware would accept one is
unverified `[U]`.

For completeness, the public CONNAC2 unified header (`mt76_connac2_mcu_uni_txd` /
`CONNAC2X_WIFI_UNI_CMD`) is:

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0x00 | 32 | TXD | as §1.2, `PKT_FT = 2`, `HDR_FORMAT = 1` |
| 0x20 | 2 | `u2Length` | total length minus 32 |
| 0x22 | 2 | `u2CID` | **16-bit** unified command ID |
| 0x24 | 1 | reserved | 0 |
| 0x25 | 1 | `ucPktTypeID` | 0xA0 |
| 0x26 | 1 | `ucFragNum` | fragment number |
| 0x27 | 1 | `ucSeqNum` | sequence number |
| 0x28 | 2 | `u2Checksum` | 0 = no checksum |
| 0x2A | 1 | `ucS2DIndex` | as §1.3 |
| 0x2B | 1 | `ucOption` | bit0 = ACK requested, bit1 = unified command, bit2 = SET(1)/QUERY(0) |
| 0x2C | 4 | reserved | 0 |
| 0x30 | N | TLV body | |

Unified header size = **0x30 = 48 bytes**. Its TLV body uses a 4-byte tag header
(`u2Tag`, `u2Length`), where `u2Length` includes the 4-byte tag header, tags are packed
back-to-back with 4-byte alignment, and the number of tags is carried in the fixed part of
the body. `[public source, not confirmed for MT7932]`

### 2.5 TLV bodies that *are* used by this part

Even without the unified scheme, several legacy and extended commands/events carry
tag-length-value payloads. Two distinct TLV encodings are in use `[C]`:

**(a) 32-bit TLV** — used by the chip-capability query (command `0x8A`, event `0xEC`):

```
body:
  +0x00  u16  total element count
  +0x02  u16  reserved
  +0x04       first element
element:
  +0x00  u32  tag id
  +0x04  u32  body length (bytes, excludes this 8-byte element header)
  +0x08       element body
next element = current element + 8 + body length      (no extra padding)
```
`[C]` — the element count in the fixed part is authoritative; elements are walked with the
stride above and carry no inter-element padding.

**(b) 16-bit TLV** — used by the boot-time information query and by the pre-power-on PHY
action command:

```
tag:
  +0x00  u16  tag id
  +0x02  u16  length
  +0x04       body
```
with an outer header of either `{u16 total element count, u16 length}` (boot info query
event) or `{u32 magic 0x556789AA, u8 tag count, u8 version 0x01, u16 length}` (PHY action).
`[L]`

The extended commands that in other CONNAC2 firmware carry TLV station/BSS/device records
(`EXT_CMD_ID_STAREC_UPDATE 0x25`, `EXT_CMD_ID_BSSINFO_UPDATE 0x26`,
`EXT_CMD_ID_DEVINFO_UPDATE 0x2A`) are **not used** by this firmware; the flat legacy
commands `0x13` (update station record), `0x12` (set BSS info) and `0x11` (BSS activate)
are used instead `[C]`. This is the single largest structural delta versus the upstream
Linux mt7921/mt7925 drivers, which drive the same silicon through unified commands.

---

## 3. Event / response frame layout

### 3.1 RX descriptor prefix and packet-type discrimination

Every RX buffer begins with the CONNAC2 hardware RX descriptor (RXD). For this part the
RXD is **24 bytes (6 dwords)** `[C]`.

| RXD word | Bits | Field | Note |
|----------|------|-------|------|
| DW0 | 15:0 | `RX_BYTE_COUNT` | whole buffer including the 24-byte RXD |
| DW0 | 22:16 | `ETH_TYPE_OFFSET` | for software-defined packets the low nibble (bits 19:16) is reused as the software sub-type |
| DW0 | 26:25 | HW info | |
| DW0 | 31:27 | `PKT_TYPE` | 5-bit packet class |

`PKT_TYPE` values (public CONNAC2 enumeration; all of the following are decoded for this
part `[C]`):

| Value | Meaning | Routed to |
|-------|---------|-----------|
| 0 | TX status | — |
| 1 | RX vector | — |
| 2 | RX data | 802.11 data path |
| 3 | duplicate RFB | — |
| 4 | TM report | — |
| 6 | MSDU (TX token free) report | TX completion accounting |
| 7 | **software-defined** | see below |
| 11 (0x0B) | RX report | RX statistics |
| 12 (0x0C), 13 (0x0D) | ICS / PHY-ICS log | firmware capture log |

For `PKT_TYPE == 7` the discriminator is the 16-bit word at RXD byte offset 2
(i.e. `DW0[31:16]`), masked with **0x380F**:

| Masked value | Meaning |
|--------------|---------|
| **0x3800** | **MCU event** — parse as §3.2 |
| **0x3801** | 802.11 management / software-generated frame |
| other | dropped, counted as an unknown packet type |

`[C]` These three constants are the public CONNAC2 values
`CONNAC2X_RX_STATUS_PKT_TYPE_SW_BITMAP / _SW_EVENT / _SW_FRAME`.

**Events are recognised purely from the RXD, not from the ring they arrived on.** An MCU
event delivered on a data RX ring is parsed identically. `[C]`

### 3.2 Event header (normal, post-boot)

The event header begins immediately after the 24-byte RXD, i.e. at buffer offset 0x18. It
is **12 bytes (0x0C)**; the event payload begins at buffer offset 0x24. `[C]`

| Offset (from event header) | Size | Field | Description |
|---|---|---|---|
| 0x00 | 2 | `u2PacketLength` | Length of **event header + payload**, i.e. excludes the RXD. Minimum legal value 12 |
| 0x02 | 2 | `u2PacketType` | Echo of the software packet type (0x3800 family) |
| 0x04 | 1 | `ucEID` | Event ID (§5.4) |
| 0x05 | 1 | `ucSeqNum` | Sequence number echoed from the request; **0 for unsolicited events** |
| 0x06 | 1 | `ucOption` | Option/flags. Unused (always 0) in the legacy scheme on this part. In the unified scheme bit1 = "is event", bit2 = "unsolicited event" |
| 0x07 | 1 | reserved | |
| 0x08 | 1 | `ucExtenEID` | Extended event ID; meaningful when `ucEID` is 0xED / 0xEA / 0xEE |
| 0x09 | 2 | reserved | |
| 0x0B | 1 | `ucS2DIndex` | Source→destination index of the event (`S2D_INDEX_EVENT_N2H = 0`) |
| 0x0C | N | payload | |

There is **no status/return-code field in the generic event header**. Status is carried
inside the payload of the specific event (see §4).

Sanity constraints the host must apply, and that firmware is expected to honour `[C]`:
* `u2PacketLength >= 12`
* `u2PacketLength <= (RX buffer size − RXD size)`; with the 0x930-byte (2352 B) RX buffer
  used here that is `<= 0x918 = 2328` bytes.

### 3.3 Response ↔ request correlation

The host correlates strictly by the 8-bit sequence number `[C]`:

1. Read `ucEID` from event-header offset 0x04.
2. Look `ucEID` up in the table of separately-decoded event IDs (§5.4). If present:
   * stop the response timer of any outstanding command whose sequence number equals
     `ucSeqNum` (a no-op for unsolicited events, whose `ucSeqNum` is 0), and
   * decode the event from the start of the event header.
   The extended-event IDs (0xED, 0xEA/0xEE) require their own sequence-number
   lookup, because those IDs carry both solicited responses and asynchronous
   notifications.
3. If `ucEID` is **not** in that table, the event is the response to an
   outstanding command: match it against the outstanding command whose sequence number
   equals `ucSeqNum`, take the result from the event **payload** (offset 0x0C), and retire
   the command.
4. If no outstanding command matches, the event is discarded.

**Recognising an unsolicited event:** `ucSeqNum == 0`. Because the host never allocates
sequence number 0, an event carrying 0 can never match an outstanding command. This is the
same rule the public Linux driver applies to this silicon (`!rxd->seq ⇒ unsolicited`).
In addition, a fixed set of event IDs (§5.4, marked *U*) is always firmware-initiated. `[C]`

Out-of-order responses are tolerated: the pending list is searched by sequence number, not
popped in order. `[C]`

### 3.4 Event header (initialisation time)

Before the RAM firmware is running, events use a shorter header — **8 bytes** — again
placed immediately after the 24-byte RXD, with the payload at buffer offset 0x20 `[C]`:

| Offset | Size | Field |
|--------|------|-------|
| 0x00 | 2 | `u2RxByteCount` |
| 0x02 | 2 | `u2PacketType` (0x3800 family, as §3.1) |
| 0x04 | 1 | `ucEID` (initialisation event ID, §8.4) |
| 0x05 | 1 | `ucSeqNum` (echoed) |
| 0x06 | 2 | reserved |
| 0x08 | N | payload |

The host validates the packet type, then `ucEID`, then `ucSeqNum`, then the payload status
byte. `[C]`

---

## 4. Status / return codes

### 4.1 Generic command status (post-boot)

`CMD_STATUS_*` values used by the generic "command result" event
(`EVENT_ID_INIT_EVENT_CMD_RESULT = 0xFD`, payload `{u8 ucStatus; u8 ucCID; u8 rsv[2];}`):

| Value | Name | Meaning |
|-------|------|---------|
| 0x00 | `CMD_STATUS_SUCCESS` | accepted and executed |
| 0x01 | `CMD_STATUS_REJECTED` | rejected |
| 0x02 | `CMD_STATUS_UNKNOWN` | unknown command |
| 0xFE | — | command not supported by this firmware build |

`[L]` — this event ID is not separately decoded on this part, so such a frame is matched to
the issuing command by sequence number (§3.3 step 3).

### 4.2 Initialisation / firmware-download status codes

Two different enumerations exist, selected by the PCI device ID. **This is a real MT7932
delta.** `[C]`

**Enumeration 1 — used when the device ID is 0x7922:**

| Value | Meaning |
|-------|---------|
| 0 | success |
| 1 | invalid param |
| 2 | invalid crc |
| 3 | decrypt fail |
| 4 | unknown |
| 5 | timeout |
| 6 | sec boot fail |

**Enumeration 2 — used for every other device ID, i.e. including MT7932 (0x7932) and
MT7923 (0x7923):**

| Value | Meaning |
|-------|---------|
| 1 | **success** |
| 2 | unknown |
| 3 | invalid param |
| 4 | invalid crc |
| 5 | timeout |
| 6 | sec boot check fail |
| 7 | region check fail |
| 8 | cmd size check fail |
| 9 | RAM entry check fail |
| 10 | Section check fail |
| 11 | FW download flow check fail |
| 12 | FW download cmd logic check fail |

A host driver for MT7932 must therefore test the boot-command result byte against **1**,
not 0. `[C]` The public gen4m `INIT_CMD_STATUS_*` / `WIFI_FW_DOWNLOAD_*` constants
correspond to enumeration 1 only.

### 4.3 Patch semaphore status

Returned by the patch-semaphore control initialisation command. The two values the host
actually uses are one higher on MT7932 than on MT7922, and the operation code sent in the
request is too. The selection is by PCI device ID: only `0x7922` uses the legacy encoding.
**The three `[U]` rows below are extrapolations, not findings** — the host never issues a
release and never tests those statuses, and the analogous +1 reading is wrong for the
firmware-download enumeration of §4.2.

| Meaning | MT7922 (`0x7922`) value | **MT7932 / MT7923 value** | Conf. |
|---|---|---|---|
| Operation: release semaphore | 0 | **1** | `[U]` |
| Operation: get semaphore | 1 | **2** | `[C]` |
| Status: no semaphore obtained, patch still required (retry) | 0 | **1** | `[C]` |
| Status: patch already downloaded and ready — skip the download | 1 | **2** | `[C]` |
| Status: semaphore obtained, host must download the patch | 2 | **3** | `[U]` |
| Status: semaphore released | 3 | **4** | `[U]` |

The MT7922 column carries the public gen4m names
`PATCH_STATUS_NO_SEMA_NEED_PATCH` / `PATCH_STATUS_NO_NEED_TO_PATCH` /
`PATCH_STATUS_GET_SEMA_NEED_PATCH` / `PATCH_STATUS_RELEASE_SEMA`. The firmware-boot section
§6.6 carries the same table.

### 4.4 Pre-power-on PHY-action status

| Value | Meaning |
|-------|---------|
| 0 | success |
| 1 | fail |
| 2 | recalibration required |
| 3 | ePA/eLNA configuration |

`[L]`

---

## 5. Identifier tables

Legend for the "Src" column: **C** = confirmed for this part (the command is issued to this
silicon, or the event ID is separately decoded for it); **P** = carried
over from public gen4m / mt76 sources and not exercised on this part.

Legend for "Dir": **S** = SET, **Q** = QUERY, **S+R** = SET with response requested.

### 5.1 Legacy command IDs (`ucCID`, `ucExtenCID = 0`)

Body size is the payload size in bytes excluding the 64-byte header, where it is a
compile-time constant.

| CID | Public name | Function | Dir | Body | Src |
|-----|-------------|----------|-----|------|-----|
| 0x00 | `CMD_ID_DUMMY_RSV` | no-op / keepalive probe (header only, zero-length body) | S | 0 | C |
| 0x01 | `CMD_ID_TEST_CTRL` | enter/leave RF test (manufacturing) mode, test-mode control | S | 0x0C | C |
| 0x02 | `CMD_ID_BASIC_CONFIG` | basic NIC configuration pushed at start-up | S | var | C |
| 0x03 | `CMD_ID_SCAN_REQ_V2` | scan request (TLV-extended v2 form; v1 `0x1A` is deprecated) | S | 0x4D4 | C |
| 0x04 | `CMD_ID_NIC_POWER_CTRL` | NIC power on/off control | S | var | C |
| 0x05 | `CMD_ID_POWER_SAVE_MODE` | power-save profile | S | 4 | C |
| 0x06 | `CMD_ID_LINK_ATTRIB` | link attributes | S | — | P |
| 0x07 | `CMD_ID_ADD_REMOVE_KEY` | install / remove a security key | S | var | C |
| 0x08 | `CMD_ID_DEFAULT_KEY_ID` | set default (group) key index | S | var | C |
| 0x09 | `CMD_ID_INFRASTRUCTURE` | set infrastructure / operating mode | S | 0 | C |
| 0x0A | `CMD_ID_SET_RX_FILTER` | RX packet filter | S | 0x44 | C |
| 0x0B | `CMD_ID_DOWNLOAD_BUF` | generic buffer download | S | — | P |
| 0x0C | `CMD_ID_WIFI_START` | start Wi-Fi subsystem | S | — | P |
| 0x0D | `CMD_ID_CMD_BT_OVER_WIFI` | BT-over-Wi-Fi | S | — | P |
| 0x0F | `CMD_ID_SET_DOMAIN_INFO` | regulatory domain / channel list (also v2 form and passive-scan channel list) | S | 0x40 | C |
| 0x10 | `CMD_ID_SET_IP_ADDRESS` | ARP offload / IPv4 address list | S | 4 | C |
| 0x11 | `CMD_ID_BSS_ACTIVATE_CTRL` | activate / deactivate a BSS (network) | S | 0x0C | C |
| 0x12 | `CMD_ID_SET_BSS_INFO` | full BSS-info record update | S | 0x74 | C |
| 0x13 | `CMD_ID_UPDATE_STA_RECORD` | station-record create/update | S | 0xC8 | C |
| 0x14 | `CMD_ID_REMOVE_STA_RECORD` | station-record removal | S | 5 | C |
| 0x15 | `CMD_ID_INDICATE_PM_BSS_CREATED` | power-management: BSS created | S | 8 | C |
| 0x16 | `CMD_ID_INDICATE_PM_BSS_CONNECTED` | power-management: BSS connected | S | 0x0C | C |
| 0x17 | `CMD_ID_INDICATE_PM_BSS_ABORT` | power-management: BSS aborted | S | 4 | C |
| 0x18 | `CMD_ID_UPDATE_BEACON_CONTENT` | beacon template / IE update | S | var | C |
| 0x19 | `CMD_ID_SET_BSS_RLM_PARAM` | radio-resource (bandwidth/HT/VHT operation) sync | S | 0x16 | C |
| 0x1A | `CMD_ID_SCAN_REQ` | scan request v1 — **deprecated, use 0x03** | S | — | P |
| 0x1B | `CMD_ID_SCAN_CANCEL` | abort scan | S | 4 | C |
| 0x1C | `CMD_ID_CH_PRIVILEGE` | request / abort channel privilege (off-channel grant) | S | 0x18 | C |
| 0x1D | `CMD_ID_UPDATE_WMM_PARMS` | EDCA / WMM parameter update | S | 0x2C | C |
| 0x1E | `CMD_ID_SET_WMM_PS_TEST_PARMS` | WMM power-save test parameters | S | 4 | C |
| 0x1F | `CMD_ID_TX_AMPDU` | TX A-MPDU enable/parameters | S | 4 | C |
| 0x20 | `CMD_ID_ADDBA_REJECT` | reject incoming ADDBA | S | — | P |
| 0x24 | `CMD_ID_SET_TX_PWR` | TX power | S | — | P |
| 0x26 | `CMD_ID_P2P_ABORT` | P2P operating-mode switch / abort | S | 4 | C |
| 0x28 | `CMD_ID_SET_DBDC_PARMS` | DBDC (dual-band dual-concurrent) settings | S | 0x24 | C |
| 0x2A | `CMD_ID_SET_ACL_POLICY` | MAC ACL policy | S | — | P |
| 0x30 | `CMD_ID_ROAMING_TRANSIT` | roaming FSM transition | S | — | P |
| 0x32 | `CMD_ID_SET_NOA_PARAM` | P2P notice-of-absence | S | var | C |
| 0x33 | `CMD_ID_SET_OPPPS_PARAM` | P2P opportunistic power save | S | var | C |
| 0x3D | `CMD_ID_SET_GTK_REKEY_DATA` | GTK rekey offload material | S | var | C |
| 0x3E | `CMD_ID_ROAMING_CONTROL` | roaming enable / thresholds | S | var | C |
| 0x3F | `CMD_ID_RESET_BA_SCOREBOARD` | reset BA scoreboard | S | — | P |
| 0x48 | `CMD_ID_SET_NVRAM_SETTINGS` | push NVRAM/manufacturing data blob | S | 0x5DC / 0x800 | C |
| 0x4A | `CMD_ID_SET_WOWLAN` | WoWLAN configuration | S | 0xE4 | C |
| 0x4B | `CMD_ID_SET_IPV6_ADDRESS` | IPv6 NS offload address list | S | — | P |
| 0x4F | *(WoW configuration, extended)* | wake-on-WLAN feature configuration / query | S,Q | var | C |
| 0x50 | `CMD_ID_SET_SLTINFO` | SLT (system-level test) info | S | — | P |
| 0x53 | `CMD_ID_GET_CHIPID` | chip ID query | Q | — | P |
| 0x58 | `CMD_ID_SET_SUSPEND_MODE` | host suspend notification | S | 0x44 | C |
| 0x5A | `CMD_ID_SET_RRM_CAPABILITY` | 802.11k radio-measurement capability sync | S | 0x2C | C |
| 0x5B | `CMD_ID_SET_AP_CONSTRAINT_PWR_LIMIT` | maximum TX power limit | S | 0x28 | C |
| 0x5D | `CMD_ID_SET_COUNTRY_POWER_LIMIT_PER_RATE` | per-rate / SAR / SDB / 6 GHz / 2.4 GHz-path TX power limit tables | S | 0x3A, 0x3C, 0x214 | C |
| 0x5E | `CMD_ID_SET_TSM_STATISTICS_REQUEST` | traffic-stream measurement start/stop | S | 0x14 | C |
| 0x5F | `CMD_ID_GET_TSM_STATISTICS` | traffic-stream measurement result | Q | 0x48 | C |
| 0x61 | `CMD_ID_SET_SCAN_SCHED_ENABLE` | scheduled-scan enable | S | 4 | C |
| 0x62 | `CMD_ID_SET_SCAN_SCHED_REQ` | scheduled-scan request | S | var | C |
| 0x6A | `CMD_ID_UPDATE_AC_PARMS` | per-AC parameter sync | S | 0x10 | C |
| 0x6B | *(queue-management BSS/WMM update)* | queue-manager BSS + WMM parameter update | S | 0x20 | C |
| 0x6C | *(time-sync control)* | 802.1AS/TSF time-sync configuration, GPIO trigger, statistics | S,Q | 0x1C | C |
| 0x6E | `CMD_ID_SET_DROP_PACKET_CFG` | drop-packet configuration | S | — | P |
| 0x70 | `CMD_ID_GET_SET_CUSTOMER_CFG` | vendor/customer configuration key-value push | S | 0x11C | C |
| 0x71 | *(ICMP offload)* | ICMP/ping offload configuration | S | 0x108 | C |
| 0x74 | *(DMS offload enable)* | directed-multicast-service offload enable | S | 1 | C |
| 0x75 | `CMD_ID_TDLS_PS` | TDLS power save | S | — | P |
| 0x76 | *(scheduled scan, extended)* | extended scheduled-scan parameters | S | var | C |
| 0x77 | *(TX duty cycle)* | TX duty-cycle limit | S | 4 | C |
| 0x79 | `CMD_ID_GET_CNM` | concurrency-manager state query | Q | 0xA1 | C |
| 0x7A | *(set channel)* | direct channel/RF set command | S | 0x10 | C |
| 0x7B | *(coexistence profile)* | BT-coexistence profile configuration | S | 0x40 | C |
| 0x7C | `CMD_ID_COEX_CTRL` | coexistence control | S,Q | — | P |
| 0x7E | `CMD_ID_PERF_IND` | performance indication / RCPI report to firmware | S | 0x138 | C |
| 0x80 | `CMD_ID_GET_NIC_CAPABILITY` | chip capability query (v1) | Q | 0x74 | C |
| 0x81 | `CMD_ID_GET_LINK_QUALITY` | RSSI / link-quality / link-speed query | Q | 0 | C |
| 0x82 | `CMD_ID_GET_STATISTICS` | general statistics query | Q | 0x0C | C |
| 0x85 | `CMD_ID_GET_STA_STATISTICS` | per-station statistics, last TX rate | Q | 0x1C | C |
| 0x87 | `CMD_ID_GET_LTE_CHN` | LTE-safe channel query | Q | 0x14 | C |
| 0x88 | *(MIB / aggregate statistics)* | MIB counters, aggregation, per-BSS, per-STA, PHY-activity, LQM/CCA, frame counters | Q | 0x0C…0x1C4 | C |
| 0x89 | `CMD_ID_GET_BUG_REPORT` | bug report | Q | — | P |
| 0x8A | `CMD_ID_GET_NIC_CAPABILITY_V2` | chip capability query (TLV form, §2.5a) | Q | 0 | C |
| 0x8C | *(statistics update enable)* | enable periodic statistics reporting | S | 4 | C |
| 0x8D | `CMD_ID_LOG_UI_INFO` | firmware log level / log UI control | S | 0x0C | C |
| 0x8F | `CMD_ID_RDD_ON_OFF_CTRL` | radar-detection (DFS) start/stop, DFS channel switch | S | 8 | C |
| 0x91 | `CMD_ID_SET_REPORT_BEACON` | per-NSS data-count query | Q | 0x10 | C |
| 0x92 | *(secure timing / STOF)* | secure time-of-flight: enable, auth, offset, timing, start | S | 8…0x9C | C |
| 0x93 | `CMD_ID_SET_ICS_SNIFFER` | in-chip-sniffer / ICS filter control, time-sync reset | S | 0x54 | C |
| 0x95 | *(legacy 6 GHz power limit)* | 6 GHz legacy per-rate TX power limit | S | var | C |
| 0x96 | *(BSS/STA info dump)* | dump BSS and station tables | Q | 0x0C | C |
| 0x97 | *(chip counters, group 2)* | channel-switch counters, firmware info, A-MSDU stats, beacon-loss counters, management-frame counters | Q | 0x0C, 0x34 | C |
| 0x98 | *(chip counters, group 3)* | TX-rate counters, A-MPDU enable | Q,S | 0x34 | C |
| 0x99 | *(chip counters, group 4)* | all-STA dump, low-latency TX stats, RTS counters | Q | 8 | C |
| 0x9A | *(RMAC security)* | RMAC security key/BK/RS request | S | 8 | C |
| 0x9B | *(security parameters)* | security parameter block | S | 0x3C | C |
| 0x9D | *(SDB channel group)* | single-dual-band channel-group info | S | 2 | C |
| 0x9E | `CMD_ID_SET_SAP_SUS` | per-PHY block-ack window size | S | 4 | C |
| 0x9F | `CMD_ID_SET_SAP_RPS` | Bonjour/mDNS offload records | S,Q | var | C |
| 0xA0 | `CMD_ID_WFC_KEEP_ALIVE` | Wi-Fi-calling keepalive | S | — | P |
| 0xA1 | `CMD_ID_RSSI_MONITOR` | RSSI monitor thresholds | S | — | P |
| 0xA2 | `CMD_ID_PKT_OFLD` | packet offload | S | — | P |
| 0xA3 | *(ARP keepalive)* | ARP keepalive offload | S | 0x44 | C |
| 0xA5 | *(TCP/UDP keepalive)* | TCP/UDP keepalive offload | S | 0x4F8 | C |
| 0xA9 | *(L2 keepalive)* | layer-2 keepalive offload | S | 0x10 | C |
| 0xAB | *(WNM keepalive)* | 802.11v WNM sleep/keepalive offload | S | 0x44 | C |
| 0xAD | *(UWB coexistence)* | UWB coexistence mode and critical-protection window | S,Q | 4, 8, 0x20 | C |
| 0xAE | `CMD_ID_CAL_BACKUP_IN_HOST_V2` | calibration backup in host memory | S,Q | — | P |
| 0xAF | *(beacon IE parse result)* | push beacon-IE parse result to firmware | S | 4 | C |
| 0xB0 | `CMD_ID_MQM_UPDATE_MU_EDCA_PARMS` | MU-EDCA parameters | S | 0x48 | C |
| 0xB1 | `CMD_ID_RLM_UPDATE_SR_PARAMS` | spatial-reuse parameters | S | 0x3C | C |
| 0xB2 | `CMD_ID_PF_CF_COALESCING_INT` | interrupt-coalescing configuration | S | 0x4C | C |
| 0xB3 | `CMD_ID_LP_DBG_CTRL` | low-power debug / DFS-pause config | S | 0x15 | C |
| 0xB6 | *(chip sleep info)* | chip sleep statistics query | Q | 0x20 | C |
| 0xB7 | *(action-frame filter)* | action-frame filter configuration | S | 1 | C |
| 0xB8 | *(set station MAC)* | program a station MAC address | S | 8 | C |
| 0xB9 | *(get station MAC)* | read back a station MAC address | Q | 6 | C |
| 0xBA | *(WF TRX info set)* | Wi-Fi TX/RX information control | S | 4 | C |
| 0xBB | *(WF TRX info query)* | Wi-Fi TX/RX information read-back | Q | 0xE8 | C |
| 0xBD | *(motion statistics)* | motion/Doppler statistics | S+R | 4 | C |
| 0xBE | *(scan private MAC)* | randomised scan MAC query | S+R | 6 | C |
| 0xBF | *(runtime calibration config)* | firmware runtime-calibration configuration | Q | 0x14 | C |
| 0xC0 | `CMD_ID_ACCESS_REG` | MCR (chip register) read / write | S,Q | 8 | C |
| 0xC1 | `CMD_ID_MAC_MCAST_ADDR` | multicast address list | S | 0xC8 | C |
| 0xC2 | `CMD_ID_802_11_PMKID` | PMKID list | S | — | P |
| 0xC3 | `CMD_ID_ACCESS_EEPROM` | EEPROM access | S,Q | — | P |
| 0xC4 | `CMD_ID_SW_DBG_CTRL` | software debug control: CTIA modes, TP test, noise-floor / LQM queries, SW-CR read/write | S,Q | 0x108 | C |
| 0xC5 | `CMD_ID_FW_LOG_2_HOST` | firmware-log-to-host enable/level | S | 4 | C |
| 0xC6 | `CMD_ID_DUMP_MEM` | firmware memory dump | Q | 0x10 | C |
| 0xC7 | `CMD_ID_RESOURCE_CONFIG` | TX/RX resource configuration | S,Q | — | P |
| 0xC8 | `CMD_ID_ACCESS_RX_STAT` | RX statistics access | Q | — | P |
| 0xCA | `CMD_ID_CHIP_CONFIG` | generic chip-config string/blob interface; also coexistence statistics and NAN/AWDL parameter sub-commands | S,Q | 0x148 | C |
| 0xCB | `CMD_ID_TPUT_INFO` | throughput / statistics log trigger | S | 0x24 | C |
| 0xCD | `CMD_ID_WTBL_INFO` | WTBL (hardware station table) dump | Q | 0xA0 | C |
| 0xCE | `CMD_ID_MIB_INFO` | MIB counter dump | Q | 0x110 | C |
| 0xD0 | `CMD_ID_GET_TXPWR_TBL` | TX-power table dump | Q | 0x0C | C |
| 0xD3 | *(protocol offload query)* | protocol-offload configuration query | Q | 0x220 | C |
| 0xD4 | *(protocol offload counters)* | protocol-offload counter query | Q | 0xB0 | C |
| 0xD6 | *(one-time calibration)* | firmware calibration type config, one-time-calibration get/put | S,S+R | 0x14 | C |
| 0xD7 | *(6 GHz channel scan set)* | 6 GHz channel-scan configuration | S | 1 | C |
| 0xD8 | *(LPAS / WoW LPAS config)* | low-power always-sensing configuration | S,Q | 0x68 | C |
| 0xD9 | *(WoW test)* | wake-on-WLAN self-test trigger | S | var | C |
| 0xDA | *(channel-time accounting)* | get/set per-channel dwell time accounting | Q | 0x80 | C |
| 0xDB | *(DFS TX pause)* | pause TX for DFS | S | 2 | C |
| 0xDC | *(RTS threshold)* | protection / RTS threshold set and default | S,Q | 0x10 | C |
| 0xDF | *(OMI / action frame)* | HE OM-control NSS/OFDMA update, action-frame injection | S,Q | 0x20 | C |
| 0xE0 | *(antenna RSSI config)* | per-antenna RSSI reporting configuration | Q | 2 | C |
| 0xE2 | *(SW DRBG seed)* | push software DRBG seed material to firmware | S | 0x44 | C |
| 0xE4 | *(NDP offload)* | IPv6 neighbour-discovery offload | S | 0x114 | C |
| 0xE5 | *(DMS offload)* | directed-multicast-service offload configuration | S | 0x2C | C |
| 0xE7 | *(low-latency window)* | low-latency-window parameter set/query | S,Q | 0x30 | C |
| 0xEA | *(Layer-1 extended command)* | MLME / connection-offload family — see §5.3 | S,S+R | — | C |
| 0xEB | `CMD_ID_NAN_EXT_CMD` | NAN extended command family (≈45 distinct sub-operations) | S,Q | 0x0C…0x533 | C |
| 0xEC | *(Aware/RA update)* | Wi-Fi Aware rate-adaptation buffer update | S | 0xB8 | C |
| 0xED | `CMD_ID_LAYER_0_EXT_MAGIC_NUM` | Layer-0 extended command escape — see §5.2 | S,Q | — | C |
| 0xEF | `CMD_ID_INIT_CMD_WIFI_RESTART` | reload firmware | S | — | P |
| 0xF1 | `CMD_ID_SET_BWCS` | Bluetooth/Wi-Fi coexistence signalling | S | — | P |
| 0xF6 | `CMD_ID_HIF_CTRL` | host-interface control: PCIe pre-suspend and resume handshake | S | 0x24 | C |
| 0xF7 | *(AWDL command family)* | Apple Wireless Direct Link control (enable, channel sequence, election, BSSID, sync master, AF rate/mode, PSF period) | S | 0x11…0x92 | C |
| 0xF8 | `CMD_ID_GET_BUILD_DATE_CODE` | firmware build date | Q | — | P |
| 0xF9 | `CMD_ID_GET_BSS_INFO` | AIS/BSS info query | Q | 0 | C |
| 0xFB | `CMD_ID_SET_TDLS_CH_SW` | TDLS channel switch | S | — | P |
| 0xFC | `CMD_ID_SET_MONITOR` | monitor-mode configuration | S | 0x10 | C |
| 0xFE | `CMD_ID_SET_MDVT` | advanced/verification control | S | 0x2C | C |

### 5.2 Layer-0 extended command IDs (`ucCID = 0xED`, `ucExtenCID = …`)

| Ext CID | Public name | Function | Dir | Body | Src |
|---------|-------------|----------|-----|------|-----|
| 0x01 | `EXT_CMD_ID_EFUSE_ACCESS` | eFuse block read / write | Q, S+R | 0x18 | C |
| 0x02 | `EXT_CMD_ID_RF_REG_ACCESS` | RF register access | — | — | P |
| 0x03 | `EXT_CMD_ID_EEPROM_ACCESS` | EEPROM access | — | — | P |
| 0x04 | `EXT_CMD_ID_RF_TEST` | RF test / internal-capture start, status and raw-data read | S,Q | 0x58 | C |
| 0x05 | `EXT_CMD_ID_RADIO_ON_OFF_CTRL` | radio on/off | — | — | P |
| 0x07 | `EXT_CMD_ID_PM_STATE_CTRL` | power-management state | — | — | P |
| 0x08 | `EXT_CMD_ID_CHANNEL_SWITCH` | channel switch | — | — | P |
| 0x09 | `EXT_CMD_ID_NIC_CAPABILITY` | capability query | — | — | P |
| 0x10 | `EXT_CMD_ID_SECURITY_ADDREMOVE_KEY` | key install/remove (WA form) | — | — | P |
| 0x11 | `EXT_CMD_ID_SET_TX_POWER_CONTROL` | TX power control | — | — | P |
| 0x12 | `EXT_CMD_ID_SET_THERMO_CALIBRATION` | thermal calibration | — | — | P |
| 0x13 | `EXT_CMD_ID_FW_LOG_2_HOST` | firmware log to host | — | — | P |
| 0x19 | `EXT_CMD_ID_COEXISTENCE` | coexistence | — | — | P |
| 0x21 | `EXT_CMD_ID_EFUSE_BUFFER_MODE` | push an EEPROM/eFuse image page-by-page into firmware ("buffer mode"); used for the EEPROM image, the per-rate power blob and the calibration blob | Q,S+R | var | C |
| 0x22 | `EXT_CMD_ID_OFFLOAD_CTRL` | offload control | — | — | P |
| 0x23 | `EXT_CMD_ID_THERMAL_PROTECT` | thermal protection | — | — | P |
| 0x25 | `EXT_CMD_ID_STAREC_UPDATE` | TLV station record (WA form) — **not used** | — | — | P |
| 0x26 | `EXT_CMD_ID_BSSINFO_UPDATE` | TLV BSS record (WA form) — **not used** | — | — | P |
| 0x2A | `EXT_CMD_ID_DEVINFO_UPDATE` | TLV device record (WA form) — **not used** | — | — | P |
| 0x2C | `EXT_CMD_ID_GET_SENSOR_RESULT` | thermal sensor read | — | — | P |
| 0x32 | `EXT_CMD_ID_WTBL_UPDATE` | WTBL update | — | — | P |
| 0x33 | `EXT_CMD_ID_BCN_UPDATE` | beacon update | — | — | P |
| 0x3C | `EXT_CMD_ID_GET_MAC_INFO` | MAC information query (12-byte result) | Q | — | C |
| 0x4E | `EXT_CMD_ID_EFUSE_BUFFER_RD` | read back buffer-mode content | — | — | P |
| 0x4F | `EXT_CMD_ID_EFUSE_FREE_BLOCK` | number of free eFuse blocks | Q | 4 | C |
| 0x57 | `EXT_CMD_ID_DUMP_MEM` | memory / TX-descriptor dump (0x44-byte result) | Q | 0x44 | C |
| 0x58 | `EXT_CMD_ID_TX_POWER_FEATURE_CTRL` | TX-power feature control; per-rate manual power set and power-info query (0x135-byte result) | S,Q | 4, 8 | C |
| 0x81 | `EXT_CMD_ID_SER` | system-error-recovery trigger; also WFDMA re-allocation (0x250-byte result) | S+R | 4 | C |
| 0x83 | *(health monitor)* | firmware health-monitor configuration | S+R | 8 | C |
| 0x94 | `EXT_CMD_ID_TWT_AGRT_UPDATE` | TWT agreement update | — | — | P |
| 0xA5 | *(GPIO control)* | GPIO direction/level control | S,Q | 8 | C |
| 0xA8 | `EXT_CMD_ID_SR_CTRL` | spatial-reuse control (sub-command in first payload byte) | S+R | var | C |
| 0xA9 | *(firmware version)* | firmware version / build query | Q | 4 | C |
| 0xBC | *(secure eFuse access)* | secure (protected) eFuse block read / write | Q, S+R | 0x48 | C |
| 0xC1 | *(chip UID)* | read chip unique identifier (8-byte result) | Q | 0 | C |
| 0xC2 | *(one-time calibration / isolation)* | pre-power-on one-time calibration start/stop, antenna-isolation detect | S,S+R | 0x7C | C |
| 0xC8 | *(FE case control)* | front-end (ePA/eLNA) case selection | S,Q | 0x0C | C |
| 0xCA | *(smart CCA set)* | smart-CCA configuration; also generates an asynchronous host notification | S+R | var | C |
| 0xCB | *(smart CCA status)* | smart-CCA status read-back (0x7C-byte result) | Q | 0 | C |

### 5.3 Layer-1 extended command IDs (`ucCID = 0xEA`, `ucExtenCID = …`)

This is the firmware-resident MLME / connection-offload interface. Not documented in
public gen4m. All entries confirmed for this part `[C]`.

| Ext CID | Function | Dir | Body |
|---------|----------|-----|------|
| 0x08 | AP / hostapd offload configuration | S | var |
| 0x30 | RX authentication frame handed to the firmware-resident supplicant | S | var |
| 0x31 | SAE authentication request | S | var |
| 0x40 | **Connect request** (full association parameters) | S | 0x358 |
| 0x41 | Install PMK / PMKSA entry | S | var |
| 0x42 | **Disconnect / link-down** | S | var |
| 0x43 | Set vendor ("product") IE | S | 0x10 |
| 0x44 | Set RSSI→rate mapping table | S | 0x5C |
| 0x46 | Delete PMK / PMKSA entry | S | var |
| 0x47 | Release channel privilege after DHCP completion | S | var |
| 0x48 | Beacon-report request configuration (802.11k) | S | 0x90 |
| 0x49 | BSS-transition-management (802.11v) parameters | S | var |
| 0x4A | Record client IE information | S | var |
| 0x4B | Reassociation request | S | var |
| 0x4C | User roaming cache set / query | S+R | 0x58 |
| 0x4D | BSS blacklist | S | var |
| 0x4E | MLME configuration update | S | 4 |
| 0x4F | Read back PMK | S+R | 0x5C |
| 0x51 | Inform firmware of SDB (single/dual-band) switch | S | 4 |
| 0x60 | Push reduced-neighbour-report information | S | 0x14 |
| 0x61 | User roaming cache channel list | S+R | 0x84 |
| 0x62 | Roaming split-scan list query | S+R | 0x44 |

### 5.4 Event IDs (`ucEID`)

The 69 entries below are the complete set of event IDs that are separately decoded for this
part. Any event ID **not** in this table must be matched to the outstanding command whose
sequence number it echoes. "Kind": **U** = unsolicited (`ucSeqNum == 0`), **S** = solicited
(response, `ucSeqNum` echoes the request), **S/U** = both occur.

| EID | Public name | Function | Kind | Src |
|-----|-------------|----------|------|-----|
| 0x01 | `EVENT_ID_NIC_CAPABILITY` | chip capability v1 response (handled synchronously during boot, not via the table) | S | C |
| 0x02 | `EVENT_ID_LINK_QUALITY` | RSSI / link quality / link speed | S/U | C |
| 0x03 | `EVENT_ID_STATISTICS` | general statistics | S | C |
| 0x04 | `EVENT_ID_MIC_ERR_INFO` | TKIP MIC failure report | U | C |
| 0x07 | `EVENT_ID_SLEEPY_INFO` | firmware sleep-state notification | U | C |
| 0x0A | `EVENT_ID_RX_ADDBA` | inbound block-ack agreement established | U | C |
| 0x0B | `EVENT_ID_RX_DELBA` | inbound block-ack agreement torn down | U | C |
| 0x0D | `EVENT_ID_SCAN_DONE` | **scan completion** | U | C |
| 0x0F | `EVENT_ID_TX_DONE` | **TX status / completion report** (per-PID, with status, sequence, TID, retry count, flush flag) | U | C |
| 0x10 | `EVENT_ID_CH_PRIVILEGE` | channel-privilege grant/revoke | U | C |
| 0x11 | `EVENT_ID_BSS_ABSENCE_PRESENCE` | BSS absence/presence (queue gating) | U | C |
| 0x12 | `EVENT_ID_STA_CHANGE_PS_MODE` | peer entered/left power save | U | C |
| 0x13 | `EVENT_ID_BSS_BEACON_TIMEOUT` | **beacon loss** | U | C |
| 0x14 | `EVENT_ID_UPDATE_NOA_PARAMS` | P2P notice-of-absence update | U | C |
| 0x15 | `EVENT_ID_AP_OBSS_STATUS` | OBSS status for AP role | U | C |
| 0x16 | `EVENT_ID_STA_UPDATE_FREE_QUOTA` | per-station TX quota update | U | C |
| 0x18 | `EVENT_ID_ROAMING_STATUS` | roaming FSM status | U | C |
| 0x19 | `EVENT_ID_STA_AGING_TIMEOUT` | station aged out | U | C |
| 0x1B | `EVENT_ID_SEND_DEAUTH` | firmware requests/reports deauthentication | U | C |
| 0x1C | `EVENT_ID_UPDATE_RDD_STATUS` | radar-detection status | U | C |
| 0x1D | `EVENT_ID_UPDATE_BWCS_STATUS` | BT/Wi-Fi coexistence signalling status | U | C |
| 0x1E | `EVENT_ID_UPDATE_BCM_DEBUG` | coexistence debug | U | C |
| 0x20 | `EVENT_ID_DUMP_MEM` | firmware memory-dump payload | S | C |
| 0x23 | `EVENT_ID_SCHED_SCAN_DONE` | scheduled-scan completion | U | C |
| 0x24 | `EVENT_ID_ADD_PKEY_DONE` | pairwise key installation complete | U | C |
| 0x25 | `EVENT_ID_ICAP_DONE` | internal-capture complete | U | C |
| 0x27 | `EVENT_ID_DEBUG_MSG` | **firmware log / debug message** | U | C |
| 0x2E | `EVENT_ID_TX_ADDBA` | outbound block-ack agreement established | U | C |
| 0x3E | *(thermal notify)* | **thermal threshold crossing notification** | U | C |
| 0x4D | *(WoW RX packet info)* | wake-up packet contents | U | C |
| 0x50 | `EVENT_ID_RDD_SEND_PULSE` | radar pulse report | U | C |
| 0x5D | *(scan start report)* | scan has started (channel, type, source) | U | C |
| 0x5E | *(scan done report)* | detailed scan-done report (state, reason, hit count) | U | C |
| 0x60 | `EVENT_ID_RDD_REPORT` | radar detected | U | C |
| 0x61 | `EVENT_ID_CSA_DONE` | channel-switch-announcement complete | U | C |
| 0x62 | `EVENT_ID_WOW_WAKEUP_REASON` | wake-up reason | U | C |
| 0x63 | `EVENT_ID_OPMODE_CHANGE` | operating-mode change | U | C |
| 0x64 | `EVENT_ID_LTE_IDC_REPORT` | LTE in-device coexistence report | U | C |
| 0x6C | *(time-sync)* | time-sync sub-event family | U | C |
| 0x6D | *(time-sync trigger)* | time-sync capture trigger | U | C |
| 0x78 | `EVENT_ID_DBDC_SWITCH_DONE` | DBDC hardware switch complete | S/U | C |
| 0x79 | `EVENT_ID_GET_CNM` | concurrency-manager info | S | C |
| 0x7A | *(set-channel done)* | channel/RF set completed | S/U | C |
| 0x7C | `EVENT_ID_COEX_CTRL` | coexistence control response | S/U | C |
| 0x8C | *(channel switch)* | channel-switch notification | U | C |
| 0x90 | `EVENT_ID_UPDATE_COEX_PHYRATE` | coexistence PHY-rate update | U | C |
| 0x92 | *(secure ToF)* | secure time-of-flight sub-event family | S/U | C |
| 0x9A | *(RMAC security)* | RMAC security response | S | C |
| 0x9C | *(trigger capture)* | firmware-requested capture trigger | U | C |
| 0xA1 | `EVENT_ID_RSSI_MONITOR` | RSSI threshold crossed | U | C |
| 0xB2 | `EVENT_ID_PF_CF_COALESCING_INT_DONE` | interrupt-coalescing config applied | S | C |
| 0xCC | `EVENT_ID_CHECK_REORDER_BUBBLE` (0x2A upstream; **0xCC here**) | reorder-bubble timer | U | C |
| 0xCD | `EVENT_ID_WTBL_INFO` | WTBL dump | S | C |
| 0xCE | `EVENT_ID_MIB_INFO` | MIB counter dump | S | C |
| 0xD4 | *(protocol-offload counters)* | protocol-offload counter response | S | C |
| 0xD6 | *(one-time cal data)* | one-time calibration data from firmware | S/U | C |
| 0xD7 | *(one-time cal data from host)* | firmware requests calibration data held by the host | U | C |
| 0xD8 | *(LPAS)* | low-power always-sensing event | U | C |
| 0xD9 | *(one-time cal load status)* | calibration-data load result | U | C |
| 0xDA | *(channel-time report)* | per-channel dwell-time accounting | S | C |
| 0xDE | *(current FE loss)* | current front-end loss values | S | C |
| 0xDF | *(vendor request info)* | vendor request information | U | C |
| 0xEA | *(Layer-1 extended event)* | MLME/connection-offload event family — §5.6 | S/U | C |
| 0xEB | `EVENT_ID_NAN_EXT_EVENT` | NAN sub-event family | S/U | C |
| 0xEC | `EVENT_ID_NIC_CAPABILITY_V2` | **chip capability query response** (TLV, §5.5) | S | C |
| 0xED | `EVENT_ID_LAYER_0_EXT_MAGIC_NUM` | Layer-0 extended-event escape — §5.6 | S/U | C |
| 0xEE | *(Layer-1 extended event, alias)* | same handler as 0xEA | S/U | C |
| 0xF0 | `EVENT_ID_ASSERT_DUMP` | **firmware assertion / coredump** | U | C |
| 0xF6 | `EVENT_ID_HIF_CTRL` | host-interface control: PCIe pre-suspend completion. Payload ≥ 0x24 B; byte 0 = interface type (2 = PCIe), byte 1 and byte 2 = handshake state (both 2 ⇒ pre-suspend done), byte 3 = suspend-notify flag | U | C |
| 0xF7 | *(AWDL event family)* | Apple Wireless Direct Link sub-events | S/U | C |
| 0xF9 | `EVENT_ID_GET_AIS_BSS_INFO` | AIS/BSS info response | S | C |

Event IDs defined by the public gen4m/mt76 headers but **absent** from this part's
set of separately-decoded IDs (they would be matched by sequence number if the firmware
emitted them):
`0x05 ACCESS_REG`, `0x06 ACCESS_EEPROM`, `0x08 BT_OVER_WIFI`, `0x09 TEST_STATUS`,
`0x0C ACTIVATE_STA_REC`, `0x0E RX_FLUSH`, `0x17 SW_DBG_CTRL`, `0x1A SEC_CHECK_RSP`,
`0x1F RX_ERR`, `0x21/0x22 STA_STATISTICS`, `0x26 RESOURCE_CONFIG`, `0x28–0x2D RTT/batch`,
`0x2F LTE_SAFE_CHN`, `0x30–0x3B GSCAN`, `0x3C CSI_DATA`, `0x40–0x48 UART/SLT/chip-config`,
`0x51–0x5B PFMU/MU/AM/heartbeat`, `0x65–0x70`, `0x77 GC_CSA`, `0x80 TDLS`,
`0x8D LOG_UI_INFO`, `0x91 UPDATE_COEX_STATUS`, `0x94 BEACON_TSF_SYNC`, `0xA2 PKT_OFLD`,
`0xA3 FW_DROP_SSN`, `0xAE CAL_BACKUP`, `0xAF CAL_ALL_DONE`, `0xB3 LP_DBG_CTRL`,
`0xB5 DELAY_BAR`, `0xCF TX_MCS_INFO`, `0xD0 GET_TXPWR_TBL`, `0xD2 GET_DPD_CACHE`,
`0xD5 FAST_PATH`, `0xE3 P2P_LO_STOP`, `0xF3 VOLT_INFO`, `0xF8 BUILD_DATE_CODE`,
`0xFB DEBUG_CODE`, `0xFC RFTEST_READY`, `0xFD INIT_EVENT_CMD_RESULT`,
`0xFE RDD_OPMODE_CHANGE`. `[P]`

### 5.4a Host obligations for unsolicited events

An unsolicited event is not advisory. For the events below the firmware has already changed
its own state, or is waiting on the host, and losing or ignoring the event desynchronises
the two sides. This table states, for each, **when the firmware emits it** and **what the
host must do**. Events not listed here may be logged and discarded without consequence.

In the table below the **Host is obliged to** column is `[C]` — it is what the analysed host
does on each event. The **Emitted when** and **If ignored** columns describe firmware
behaviour and are `[L]` throughout, even where an individual cell is marked `[C]`; only the
event bodies and the host's actions were established.

| EID | Emitted when | Host is obliged to | If ignored |
|-----|--------------|--------------------|------------|
| `0x07` `SLEEPY_INFO` | The firmware has finished its work and wants to power down. Body byte 0 = "sleepy" flag; non-zero means *release ownership now*. | Run the release-ownership sequence (hand `SET_OWN` back to the firmware) as soon as the current interrupt service completes. `[C]` | The chip never enters low power; power consumption stays at the active level indefinitely. No retry event is sent. `[C]` |
| `0x0A` `RX_ADDBA` | A peer's ADDBA request has been accepted by the firmware. | **Create a host-side receive reorder window** for (station index, TID) with the window size and starting sequence number carried in the event (§5.4b). `[C]` | Received A-MPDU subframes arrive out of order and are passed up unordered; the host's own diagnostic path reports "reordering but no BA agreement". `[C]` |
| `0x0B` `RX_DELBA` | The BA agreement ended (peer DELBA, timeout, or firmware teardown). | Flush and destroy the reorder window for (station index, TID). `[C]` | Buffered frames are stranded in the reorder queue and the window never advances; the TID stalls. `[C]` |
| `0x2E` `TX_ADDBA` | An outbound BA agreement was established. | Record the negotiated buffer size and, per TID, the **A-MSDU-in-A-MPDU enable bitmap**, and rebuild the per-station TX descriptor template accordingly (§5.4b). `[C]` | The host may set `HW_AMSDU` on TIDs where the peer did not agree to A-MSDU, or fail to use it where it did. `[C]` |
| `0xCC` `CHECK_REORDER_BUBBLE` | A hole in a receive reorder window has persisted. | Advance the window past the hole and release the frames that have become in-order. `[C]` | The affected TID stops delivering frames permanently. `[C]` |
| `0x10` `CH_PRIVILEGE` | Answer to a channel-privilege request, **or** an unsolicited revocation. | Match on the token, act on the status (grant / reject / recover / remove-request) and, when finished with the channel, send the matching abort. See the MAC-control section. `[C]` | The firmware's channel arbiter keeps the channel assigned to a role that no longer wants it; other roles are starved. `[C]` |
| `0x11` `BSS_ABSENCE_PRESENCE` | The firmware has taken a BSS off-channel (absent) or brought it back (present). Body: `{u8 bss_index, u8 is_absent, u8 bss_free_quota, u8 rsv}`. | While `is_absent = 1`, **stop dequeuing frames for that BSS index**; resume on the present transition. Honour `bss_free_quota`. `[C]` | Frames queued for an off-channel BSS occupy firmware packet-buffer credits until they age out, which back-pressures every other BSS. `[C]` |
| `0x16` `STA_UPDATE_FREE_QUOTA` | The number of frames the firmware will accept for a power-saving peer has changed. Body: `{u8 sta_index, u8 update_mode, s8 quota}`. | Update the per-station credit: `update_mode` `0`/`1` = **set** to `quota`, `2` = **add** `quota`, `3` = **subtract** `quota`; then re-split the credit between the delivery-enabled and legacy AC groups and re-run the TX scheduler. `[C]` Any other `update_mode` value is a protocol violation. `[C]` | Frames are pushed to a sleeping peer beyond what the firmware buffered for it; they are dropped or stall the peer's service period. `[C]` |
| `0x12` `STA_CHANGE_PS_MODE` | A peer entered or left power save. | Gate the peer's TX queues accordingly. `[C]` | As above. |
| `0x13` `BSS_BEACON_TIMEOUT` | The firmware's link-monitor lost the beacon on a BSS index. | Treat the link as down and tear down / re-connect. `[C]` | The firmware has already stopped maintaining the link; the host keeps a dead association and its station entries leak. `[C]` |
| `0x19` `STA_AGING_TIMEOUT` | The firmware aged out a peer (reports station index and MAC). | **Free the station index host-side** and issue the delete-station command. `[C]` | The station index remains allocated in the host's 15-entry pool but is dead in the firmware; the pool leaks until reset. `[C]` |
| `0x1B` `SEND_DEAUTH` | The firmware sent (or wants the host to send) a deauthentication. | Complete the disconnect: delete keys, delete the station, deactivate the BSS. `[C]` | Same leak as above, plus stale key-table entries. `[C]` |
| `0x0D` `SCAN_DONE` / `0x5E` scan-done report | The scan the host requested has finished or was aborted. | Clear the "scan in progress" state and release the channel privilege the scan holds; only then may another scan or a channel change be issued. `[C]` for the state clear, `[L]` for the privilege release | A second scan request is rejected or silently merged; the radio may stay parked on the last scanned channel. `[L]` |
| `0x62` `WOW_WAKEUP_REASON` | The firmware woke the host. | Read and consume the reason; if the wake path used the software-interrupt mailbox bit 0, acknowledge that bit explicitly. `[C]` | The wake source stays latched and the next wake is misattributed. `[C]` |
| `0xD7` *(one-time cal data request)* | The firmware needs a calibration object that the host is holding. | **Reply** with the requested block through the calibration-data command path. `[C]` | Calibration provisioning never completes; the radio runs on firmware defaults. `[L]` |
| `0xF0` `ASSERT_DUMP` | The firmware has hit an assertion. | Acknowledge with the host→MCU doorbell, collect the dump, and reset the chip. See the firmware-boot section. `[C]` | The part is halted permanently. `[C]` |
| `0xF6` `HIF_CTRL` | Completion of the PCIe pre-suspend request. | Advance the suspend sequence; the handshake bytes must both read `2`. `[C]` | Suspend times out (~1 s) and the host must abandon the suspend. `[C]` |
| `0x27` `DEBUG_MSG` | Firmware log output, once logging has been enabled. | Consume and free the receive buffer. `[C]` | Receive buffers are consumed at the firmware's logging rate; the event ring under-runs. `[C]` |

Additionally, the **software-interrupt mailbox** (not an MCU event) carries the
error-recovery handshake and the low-power wake indication; both have hard host responses
and are specified in the interrupt and power/reset sections.

### 5.4b Layout of the block-acknowledgement events

These three events are the only mechanism by which the host learns the negotiated
aggregation parameters — **receive reordering on this part is performed by the host, not by
the hardware** (there is no RX-reorder offload), so the host cannot skip them. `[C]`

`EVENT_ID_RX_ADDBA` (`0x0A`) body, matching the public `EVENT_RX_ADDBA` layout `[C]`:

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | `ucStaRecIdx` — station-record index |
| `0x01` | 1 | `ucDialogToken` — copied from the peer's ADDBA request |
| `0x02` | 2 | `u2BAParameterSet` — bits `[5:2]` = **TID**, bits `[15:6]` = **buffer size** (reorder window) |
| `0x04` | 2 | `u2BATimeoutValue` |
| `0x06` | 2 | `u2BAStartSeqCtrl` — bits `[15:4]` = **starting sequence number** |

Only TIDs `0`–`7` are accepted; a TID outside that range is discarded. `[C]`

`EVENT_ID_RX_DELBA` (`0x0B`) body: `{u8 ucStaRecIdx, u8 ucTid, u8 rsv[2]}`. `[C]`

`EVENT_ID_TX_ADDBA` (`0x2E`) body `[C]`:

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | `ucStaRecIdx` |
| `0x01` | 2 | BA parameter set — bits `[5:2]` = **TID**, bits `[15:6]` = **buffer size**. (Note: this is a packed 16-bit field at an odd offset; the public `gen4m` `EVENT_TX_ADDBA` declares separate `ucTid`/`ucWinSize` bytes here. Decode as the packed form. `[C]`) |
| `0x03` | 1 | `ucAmsduEnBitmap` — bit *n* = A-MSDU-in-A-MPDU permitted on TID *n* |
| `0x06` | 1 | `ucMaxMpduCount` — maximum MPDUs per A-MSDU |
| `0x08` | 4 | `u4MaxMpduLen` — maximum A-MSDU length |
| `0x0C` | 4 | `u4MinMpduLen` — below this length a frame is not aggregated |

The host must apply `ucAmsduEnBitmap` by rebuilding the per-station, per-TID TX descriptor
template; the bitmap changes on every re-negotiation and only the delta is acted on. `[C]`
The event is only meaningful on a part whose capability record advertises hardware A-MSDU;
MT7932 does. `[C]`

### 5.5 Chip-capability TLV tag IDs (event 0xEC body, encoding §2.5a)

All confirmed present in this part's capability response `[C]`.

| Tag | Content |
|-----|---------|
| 0x01 | eFuse address / layout |
| 0x02 | coexistence feature bitmap |
| 0x03 | single-SKU regulatory power table from firmware |
| 0x05 | hardware version |
| 0x06 | software (firmware) version |
| 0x07 | MAC address |
| 0x08 | PHY capability |
| 0x09 | MAC capability |
| 0x0A | frame-buffer capability |
| 0x0B | beamforming capability |
| 0x0C | location (RTT/FTM) capability |
| 0x0D | MU-MIMO capability |
| 0x14 | A-die hardware version |
| 0x17 | WFDMA reallocation descriptor |
| 0x18 | 6 GHz capability |
| 0x1E | maximum RMAC quota |
| 0x1F | MLME-offload random-MAC capability |
| 0x20 | roaming / BSS-transition policy |
| 0x30 | AWDL 6 GHz capability |
| 0x31 | NAN capability |
| 0x32 | low-latency-window TX capability |
| 0x33 | low-latency-window RX capability |
| 0x34 | secure time-of-flight capability |
| 0x35 | UWB coexistence capability |
| 0x36 | external-PTA capability |
| 0x37 | time-sync improvement capability |
| 0x38 | runtime-calibration support flag |
| 0x39 | suspend-offload capability |
| 0x3A | per-antenna RSSI capability |
| 0x3B | PCIe link-speed capability |
| 0x3C | smart-CCA capability |

### 5.6 Extended event IDs

**Layer-0 (event `ucEID = 0xED`, extended event ID at event-header offset 0x08).** As a
rule the extended event ID equals the extended command ID that produced it; the handler
falls through to generic sequence-number correlation for any value it does not special-case.
Values with dedicated handling `[C]`:

| Ext EID | Function | Result size |
|---------|----------|-------------|
| 0x00 | generic extended-command result | 0x404 |
| 0x01 | eFuse access result (address/valid/data) | 0x18 |
| 0x04 | RF-test / internal-capture result | var |
| 0x3C | MAC information | 0x0C |
| 0x4C | per-station parameter update (indexed by WLAN index) | — (unsolicited) |
| 0x57 | memory / TX-descriptor dump | 0x44 |
| 0x58 | TX-power information | 0x135 |
| 0x81 | SER / WFDMA-realloc result | 0x250 |
| 0x8A | EEPROM/calibration TLV blob delivered to host (sub-type in payload byte 1; type 3 = calibration image) | var |
| 0xA6 | 1-byte status result | 8 |
| 0xA8 | spatial-reuse control result (sub-event in payload byte 0) | 6 |
| 0xAA | 32-bit scalar result | 4 |
| 0xBC | secure-eFuse result (≤32 bytes of key material) | 0x48 |
| 0xC1 | chip UID (64-bit) | 8 |
| 0xC2 | one-time-calibration / isolation-detect condition stop | var |
| 0xC8 | FE-case control result | 0x0C |
| 0xCA | smart-CCA host notification | var (unsolicited) |
| 0xCB | smart-CCA status (14 × 64-bit counters) | 0x7C |

Public gen4m/mt76 Layer-0 extended event IDs not specially handled here: `0x05 PS_SYNC`,
`0x13 FW_LOG_2_HOST`, `0x22 THERMAL_PROTECT`, `0x23 ASSERT_DUMP`, `0x3A RDD_REPORT`,
`0x4F CSA_NOTIFY`, `0x74 WA_TX_STAT`, `0x75 BCC_NOTIFY`, `0x9A WF_RF_PIN_CTRL`,
`0x9F MURU_CTRL`. `[P]`

**Layer-1 (event `ucEID = 0xEA` or `0xEE`).** Extended event ID `0x11` is the
WPA/connection status report; every other value carries an opaque payload (from
offset 0x0C) destined for the host's supplicant. `[C]`

---

## 6. Timeouts and flow control

| Property | Value | Note |
|----------|-------|------|
| Response timeout the host allows per command | **10 000 ms** | `[C]` One timer is armed per command that expects a response, keyed on its sequence number |
| Boot / firmware-download command timeout | **6 000 ms** (default; configurable) | `[C]` Used for `INIT_CMD_ID_*` and for the two synchronous capability queries |
| Command-buffer starvation escalation | **6 000 ms** | `[C]` If no command descriptor can be obtained for this long, the host escalates to a chip reset |
| Maximum command payload | **0x600 = 1536 bytes** | `[C]` (total buffer 0x640 = 1600 B including the 64-byte header) |
| Maximum firmware-download section per frame | **0x800 = 2048 bytes** | `[C]` |
| Hardware command-ring depth | **24 descriptors** (TX ring 17) | `[C]` This is the hard limit on commands simultaneously in flight to the MCU |
| Firmware-download ring depth | **256 descriptors** (TX ring 16) | `[C]` |
| Sequence-number space | 255 usable values (1…255) | `[C]` |
| Ordering | Commands are consumed from a single ring **in order**; responses are **not** required to be in order — the host matches by sequence number over the whole outstanding set | `[C]` |

**Flow control.** Before composing a command the host must confirm that the command TX
ring has at least one free descriptor; the free count is `(ring size − 1) − (cpu_idx −
dma_idx)`. When the ring is full the host must reclaim completed descriptors and retry.
`[C]`
Command traffic is additionally accounted against a dedicated traffic class (TC index 4)
in the PSE/PLE page-credit scheme; each command consumes `ceil(total_length / page_size)`
pages. `[C]`

**On timeout the host must:**
1. Fail the request the command was issued for.
2. Remove the command from the outstanding-response set, releasing its sequence number
   and its buffer. `[C]`
3. Treat a late-arriving event carrying that sequence number as unmatched and discard it.
4. On repeated failures, escalate to system-error recovery (`EXT_CMD_ID_SER`, ext CID
   0x81) or a full chip reset. `[C]`

A command that has been queued but not yet written to the ring may be cancelled
(e.g. on interface teardown); nothing is sent to the chip in that case. `[C]`

---

### 6.1 Firmware-side acceptance rules — what a malformed command does

The firmware validates every command before dispatching it. `[L]` **No** negative
acknowledgement, status event or error counter for a rejected command is decoded by the host
`[C]`, so as far as the host is concerned a rejected command is indistinguishable from a lost
one and the only feedback is its own response timeout (10 000 ms, §6). That the firmware
emits **nothing at all** in that case is `[L]` and is open question 9. A driver that fires
commands without response tracking will silently mis-configure the part. `[C]`

Checks the firmware performs. `[L]` for the whole table — these are read from the firmware's
own validation and log sites, not observed on silicon; the "dropped, no event" outcome is the
inference of the paragraph above:

| Check | Failure behaviour |
|---|---|
| **Exact payload length.** Each command ID has one fixed body length; the firmware compares the received length against it and rejects a mismatch — including a body that is *longer* than expected. The per-command lengths in §5.1–§5.3 are therefore normative, not advisory. | dropped, no event |
| **Set/query direction.** A command defined as "set" rejects a query and vice versa. | dropped, no event |
| **BSS index bound.** Any command carrying a BSS index is checked against the firmware's BSS-context count. | dropped, no event |
| **BSS activation state.** A command that programs a BSS is rejected if that BSS context has not been activated. | dropped, no event |
| **Station-record index / validity.** Commands referencing a station index check that the record exists and is valid. | dropped, no event |
| **Action / sub-type enumeration.** Sub-command and action bytes are range-checked. | dropped, no event |
| **Unknown command ID.** | dropped, no event |
| **Channel-privilege request type.** An out-of-range request type in a channel-privilege command makes the firmware **assert**, which halts the part and starts the coredump path. `[C]` | firmware assertion |

The same discipline applies to the events the host consumes: the host must range-check every
index in an event body before using it (BSS index against the context count, station index
against the record pool, MSDU token against the pool size), because the firmware does not
guarantee that a stale index will not appear after a recovery. `[C]`

### 6.2 Ordering and mutual-exclusion constraints

The command ring is consumed strictly in order, but **responses are not returned in order**
(§6). Ordering that the firmware requires must therefore be enforced by the host either by
send order (for fire-and-forget commands) or by waiting on the response (where one exists).
The constraints below are `[C]` as descriptions of the order the host observes; that the
*firmware* requires each of them is `[L]` unless the row cites a firmware check from §6.1.

**Allocation before reference.** Every index the host puts in a command body is a host-
allocated index into a firmware table; the firmware validates it but does not allocate it.
The dependency chain, in the order the objects must be created, is:

```
BSS context activated  ->  own-MAC (MUAR) slot bound  ->  station record created
      ->  station promoted to the associated state  ->  key installed
                                                    ->  data frames referencing that index
```

* A **BSS index** must be activated before any command that names it (channel set, BSS
  parameters, beacon template, EDCA, station create).
* A **station index** must exist before a key command, a BA-related command, a per-station
  power offset or a rate command names it. For group keys, the BSS's broadcast/multicast
  station row must be reserved first.
* A **key** must be installed before the station is promoted to the associated state, or
  the first protected frames are transmitted unencrypted.
* An **operating channel** must be set on a BSS before the radio is enabled for that band;
  enabling the radio first leaves it on an undefined channel.
* **Regulatory domain and power limits** must be pushed before the channel is set, because
  the channel legality check uses them.
* **Calibration objects** have their own required order (WCal → OCAL → IFCAL) — see the
  EEPROM/calibration section.

**Commands illegal in a given state:**

| Command class | Illegal when |
|---|---|
| Any runtime (post-boot) command | before the RAM firmware has signalled ready; only the initialisation command subset (§8.3) is accepted `[C]` |
| Initialisation commands (`INIT_CMD_ID_*`) | after the RAM firmware is running `[C]` |
| Any command at all | while an assertion/coredump is in progress, or between the "stop PDMA" and "MCU normal" phases of an error recovery — the firmware is not servicing the ring `[C]` |
| Power-off / NIC power control | while a coredump or a sub-system error recovery is outstanding `[C]` |
| A second scan request | before the scan-done event for the first `[C]` |
| A channel change on a band | while an off-channel privilege is outstanding on that band `[L]` |

**Must not be issued concurrently:**

* **TX/RX control (abort / disable / pause) commands are stateful and nest badly.** The
  firmware refuses a new TX-or-RX control command on a band while a previous one is still
  in effect: the previous command must be *restored* (the inverse action issued) before a
  different one is accepted. `[C]`
* **Only one channel-privilege request per (BSS, token) may be outstanding**, and an abort
  must carry the same token as the request it cancels.
* **Only one firmware-download stream** may be in flight; the patch semaphore serialises
  hosts and must be released even on the failure path.
* **The ownership handshake is not re-entrant**: a release-to-firmware must not be issued
  while a driver-own acquisition, a coredump, a driver-triggered error recovery or a
  pending subsystem reset is outstanding.

### 6.3 Command-path resource exhaustion

| Resource | Size | Exhaustion is visible as | Host must |
|---|---|---|---|
| Command TX ring | 24 descriptors | `(size − 1) − (cpu_idx − dma_idx) == 0` | reclaim completed descriptors and retry; escalate to a chip reset after 6 000 ms of continuous starvation `[C]` |
| Sequence-number space | 255 values (1…255), 0 reserved for unsolicited events | the counter wraps onto a still-outstanding sequence number | never allow more than 255 commands outstanding; in practice the 24-deep ring binds first `[C]` |
| Firmware packet/page credits (PSE/PLE) | accounted per traffic class; commands use TC 4 and consume `ceil(total_length / page_size)` pages | commands stop being consumed from the ring | drain and retry; there is **no** credit-return event for the command path — the ring indices are the only back-pressure signal `[C]` |
| Firmware event buffers | — | the firmware log carries an "EVENT packet alloc failed" site `[C]`; that it then **silently drops the event**, including responses, is `[L]` | tolerate a lost response: the response timeout must be a normal, recoverable path, not a fatal one `[C]` |

The last row is important and easy to miss: if the firmware can fail to allocate an
event buffer under memory pressure, **a command that was accepted and executed may still
never produce its response** `[L]`. The host must not treat a single response timeout as
evidence that the command did not take effect. `[C]` as host guidance.

---

## 7. Transport routing

### 7.1 TX direction

| Traffic class | Ring index | Ring depth | Descriptor prefix |
|---------------|------------|------------|-------------------|
| Firmware download (RAM code, ROM patch payload sections) | **16** (`TX_RING_FWDL`) | 256 | **none** — see below |
| Initialisation-time commands (`INIT_CMD_ID_*`) | **17** (`TX_RING_CMD`) | 24 | 32-byte TXD + 32-byte init header |
| Normal MCU commands (legacy, Layer-0 ext, Layer-1 ext) | **17** (`TX_RING_CMD`) | 24 | 32-byte TXD + 32-byte command header |
| Security / management frames pushed through the command path | 17 (or the "WA command" ring index when a second MCU is present) | 24 | 32-byte TXD + 32-byte command header |
| Ring index 18 | 18 | 16 | **alternate command-class ring**: the firmware capability TLV `0x17` (WFDMA reallocation) rewrites a command-class ring-index field from 17 to 18 `[C]`; which class moves is `[L]` (the WM-command and WA-command indices both default to 17 here). No traffic is steered there by default |

`[C]` for all of the above except the last row.

**Second-CPU ("WA") commands: not applicable.** The "WA command ring index" for this part
is the same value (17) as the WM command ring, and `is_support_wacpu` is false, so no
traffic is ever steered to a distinct WA ring.
MT7932, like MT7922, exposes a single Wi-Fi MCU to the host. `[C]`

**Firmware-download frames carry no descriptor at all.** For the firmware/patch payload
command (command ID 0, §8.2), the host writes the raw section bytes to ring 16 with **no
TX descriptor and no MCU header**; the PDA engine consumes them directly. The DMA length is
the section length rounded up to a multiple of 4. This matches the public Linux behaviour
for `MCU_CMD(FW_SCATTER)`, which likewise bypasses the MCU TXD on the FWDL queue. `[C]`
(The `PKT_FT = 3` "PDA firmware download" TXD encoding exists and applies only where a
descriptor is built for a CID-0 frame; the normal download path builds none. `[C]`)

### 7.2 RX direction

| Ring | Depth | Buffer | Role |
|------|-------|--------|------|
| 0 | 32 | 2352 B | **MCU event ring used before the RAM firmware signals ready** (boot/initialisation events) |
| 2 | 512 | 2352 B | 802.11 data |
| 3 | 32 | 2352 B | TX-free-done / MSDU-token return |
| **4** | **32** | **2352 B** | **MCU event ring used once the RAM firmware is running** |
| 5 | 224 | 2352 B | coredump events |
| 6 | 128 | 2352 B | firmware-log events |
| 7 | 512 | 2352 B | low-latency data |
| 8 | 64 | 2352 B | management-frame receive |
| 1, 9 | — | — | defined but disabled |

Ring roles other than 0 and 4 are as assigned by the WFDMA section §3.2, which is the
authority for the ring inventory.

`[C]` Rings 0 and 4 are the two MCU event rings; the host must switch its event source from
0 to 4 the moment the RAM firmware reports ready. Before that point events are
**polled** from ring 0 rather than taken from an interrupt. `[C]`

**Events on the data path.** Because the event/frame decision is taken from the RXD
`PKT_TYPE` field and the software sub-type nibble (§3.1), an MCU event delivered on any RX
ring is parsed correctly. The same classification applies to every ring.
`[C]`

**Not-events on the RX path.** Two other MCU-adjacent report classes share the RX rings but
are *not* MCU events and must not be fed to the event parser:
* `PKT_TYPE = 6` — MSDU / TX-token free report (per-packet TX completion accounting).
* `PKT_TYPE = 0x0B` — RX report; `PKT_TYPE = 0x0C/0x0D` — ICS / PHY-ICS capture log.

The higher-level per-frame TX status report is instead delivered as MCU event
`EVENT_ID_TX_DONE` (0x0F). `[C]`

---

## 8. Initialisation-time command subset

Before the RAM firmware is running, only a small, separately-encoded command set is legal.

### 8.1 Initialisation command frame layout

Same 64-byte prefix size as normal commands, but with a reduced field set — no extended
CID, no set/query, no S2D index, no response flag `[C]`:

```
 offset   size   content
 0x00     32 B   TXD: DW0[15:0] = total length incl. TXD;
                      DW0[24:23] = PKT_FT (2 = command);
                      DW1[17:16] = HDR_FORMAT (1 = command)
 0x20      2 B   u2TxByteCount   = total length − 32
 0x22      2 B   u2PQ_ID         = 0 (public definition: 0x8000 for MCU port,
                                     0xF800 for the PDA port; left 0 on PCIe)
 0x24      1 B   ucCID           initialisation command ID
 0x25      1 B   ucPktTypeID     = 0xA0
 0x26      1 B   reserved        = 0
 0x27      1 B   ucSeqNum        (same sequence-number space as normal commands, 1…255)
 0x28      4 B   reserved        = 0
 0x2C     20 B   reserved        = 0   (padding to the 32-byte normal-TXD footprint)
 0x40      N B   payload
```

The transferred length is `(total length + 3) & ~3` — i.e. 4-byte aligned `[C]`.
Initialisation commands are written to **TX ring 17** `[C]`.

### 8.2 Initialisation command IDs

| CID | Public name | Function | Payload | Src |
|-----|-------------|----------|---------|-----|
| **0x00** | *(firmware/patch payload)* | Raw image section data. **No TXD, no header** — see §7.1. Sent on **ring 16**. Section ≤ 2048 B | ≤0x800 | C |
| 0x01 | `INIT_CMD_ID_DOWNLOAD_CONFIG` | Announce the next RAM-code section: `{u32 address; u32 length; u32 data mode}` | 0x0C | C |
| 0x02 | `INIT_CMD_ID_WIFI_START` | Start the downloaded firmware: `{u32 override; u32 address}`. Override bits: bit0 = override start address, bit1 = delay calibration, bit2 = working-PDA option, bit3 = CRC check, bit4 = change decompression temp address | 8 | C |
| 0x03 | `INIT_CMD_ID_ACCESS_REG` | Chip-register read/write before firmware start: `{u8 setQuery; u8 rsv[3]; u32 address; u32 data}` | 0x0C | C |
| 0x04 | `INIT_CMD_ID_QUERY_PENDING_ERROR` | Query a pending download error | — | P |
| 0x05 | `INIT_CMD_ID_PATCH_START` | Announce the next ROM-patch section (same body as 0x01) | 0x0C | C |
| 0x06 | `INIT_CMD_ID_PATCH_WRITE` | Patch write | — | P |
| 0x07 | `INIT_CMD_ID_PATCH_FINISH` | Patch download complete: `{u8 checkCrc; u8 type; u8 rsv[2]}`; type 0 = Wi-Fi, 1 = BT, 2 = Wi-Fi/modem, 3 = 802.15.4 | 4 | C |
| 0x08 | `INIT_CMD_ID_PHY_ACTION` | Pre-power-on PHY action (calibration request / use backup), TLV body per §2.5b | var | P |
| 0x09 | `INIT_CMD_ID_LOG_TIME_SYNC` | Firmware log time sync | — | P |
| 0x10 | `INIT_CMD_ID_PATCH_SEMAPHORE_CONTROL` | Acquire/release the Wi-Fi patch semaphore: `{u8 operation; u8 rsv[3]}`. **On MT7932 the get-semaphore operation code is `2`, not `1`** (§4.3) | 4 | C |
| 0x11 | `INIT_CMD_ID_BT_PATCH_SEMAPHORE_CONTROL` | Same for the BT patch | — | P |
| 0x12 | `INIT_CMD_ID_ZB_PATCH_SEMAPHORE_CONTROL` | Same for the 802.15.4 patch | — | P |
| 0x13 | `INIT_CMD_ID_CO_PATCH_DOWNLOAD_CONFIG` | Combo-patch download config | — | P |
| 0x20 | `INIT_CMD_ID_HIF_LOOPBACK` | HIF loopback test | — | P |
| 0x21 | `INIT_CMD_ID_LOG_BUF_CTRL` | Firmware log-buffer base addresses / read-pointer update | — | P |
| 0x22 | `INIT_CMD_ID_QUERY_INFO` | Boot-time information query (TLV response, §2.5b) | 8 | P |
| 0x23 | `INIT_CMD_ID_EMI_FW_DOWNLOAD_CONFIG` | EMI-based download config | — | P |
| 0x24 | `INIT_CMD_ID_EMI_FW_TRIGGER_AXI_DMA` | EMI download trigger | — | P |
| **0x50** | *(eFuse read, boot-time)* | Read an eFuse word before the RAM firmware is running: `{u32 address}` | 4 | C |
| 0xFF | `INIT_CMD_ID_DECOMPRESSED_WIFI_START` | Start compressed firmware | — | P |

Note the presence of **0x50** — a boot-time eFuse read command that is not in the public
gen4m `ENUM_INIT_CMD_ID`. `[C]`

### 8.3 Commands legal before the RAM firmware runs

Only the initialisation set above uses the initialisation descriptor. Two **normal**
(64-byte-header) commands are also legal in the window between "RAM firmware started" and
"interface up", and are answered on the boot event ring (RX ring 0) rather than on the
run-time event ring `[C]`:

* `CMD_ID_GET_NIC_CAPABILITY` (0x80) → event `EVENT_ID_NIC_CAPABILITY` (0x01)
* `CMD_ID_GET_NIC_CAPABILITY_V2` (0x8A) → event `EVENT_ID_NIC_CAPABILITY_V2` (0xEC)

Both are QUERY commands (`ucSetQuery = 0`), both are polled from RX ring 0 with the
6 000 ms boot timeout, and both are validated by checking the RXD software packet type,
then the event ID. `[C]`

All other commands require the RAM firmware to have signalled ready. `[C]`

### 8.4 Initialisation event IDs

Header per §3.4 (8 bytes, payload at RXD+8).

| EID | Public name | Payload |
|-----|-------------|---------|
| 0x01 | `INIT_EVENT_ID_CMD_RESULT` | `{u8 status; u8 cid; u8 rsv[2]}` — status per §4.2 |
| 0x02 | `INIT_EVENT_ID_ACCESS_REG` | `{u32 address; u32 data}` |
| 0x03 | `INIT_EVENT_ID_PENDING_ERROR` | pending error code |
| 0x04 | `INIT_EVENT_ID_PATCH_SEMA_CTRL` | `{u8 status}` per §4.3 |
| 0x05 | `INIT_EVENT_ID_PHY_ACTION` | `{u8 event; u8 status; u8 rsv[2]; u32 emiAddress; u32 emiLength; u32 temperature}` |
| 0x06 | `INIT_EVENT_ID_BT_PATCH_SEMA_CTRL` | `{u8 status; u8 rsv[3]; u32 remapAddr; u8 rsv1[4]}` |
| 0x07 | `INIT_EVENT_ID_ZB_PATCH_SEMA_CTRL` | as 0x06 |
| 0x08 | `INIT_EVENT_ID_LOG_BUF_CTRL` | `{u8 type; u8 status; u8 rsv[2]; u32 address; u32 rsv}` |
| 0x09 | `INIT_EVENT_ID_QUERY_INFO_RESULT` | `{u16 totalElementNum; u16 length; TLVs}` (§2.5b) |

`[C]` for 0x01 (directly validated by the boot path); `[P]` for the remainder.

---

## 9. Summary of MT7932-specific deltas

1. **Boot status enumeration.** The firmware-download / `WIFI_START` result code uses the
   12-value enumeration in §4.2 in which **success = 1**, not the 7-value enumeration in
   which success = 0 that MT7922 uses. Host code that checks for 0 will treat every
   successful MT7932 boot as a failure. `[C]`
2. **Boot-time eFuse read command 0x50** exists in the initialisation command space. `[C]`
3. **Layer-1 extended command family at `ucCID = 0xEA`** (§5.3) carries a complete
   firmware-resident MLME/connection-offload interface (connect at ext CID 0x40,
   disconnect at 0x42, PMK cache at 0x41/0x46/0x4F, roaming cache at 0x4C/0x61/0x62,
   BTM at 0x49). This is not present in public gen4m headers. `[C]`
4. **No unified-command traffic**: no unified command descriptor, no unified header size,
   no unified command or event IDs are exchanged. The station-record,
   BSS-info and device-info updates go through the flat legacy commands 0x13/0x12/0x11,
   not through TLV extended or unified commands. `[C]` This is the opposite of the
   upstream Linux mt7921/mt7925 drivers for the same silicon family.
5. A number of event IDs outside the public gen4m enumeration are implemented:
   0x3E (thermal notify), 0x4D, 0x5D/0x5E (scan start/done report), 0x6C/0x6D (time sync),
   0x7A, 0x8C, 0x92, 0x9A, 0x9C, 0xCC, 0xD4, 0xD6–0xDA, 0xDE, 0xDF, 0xEA/0xEE, 0xF7. `[C]`
6. Extended command/event IDs 0x83, 0xA5, 0xA9, 0xBC, 0xC1, 0xC2, 0xC8, 0xCA, 0xCB are
   implemented and are not in public gen4m headers. `[C]`
7. The chip-capability TLV tag space (§5.5) extends to tag 0x3C, well beyond the public
   set. `[C]`

Everything else in this document — the 32-byte TXD, the 32-byte command header at 0x20,
the 24-byte RXD, the 12-byte event header, the 0x380F/0x3800/0x3801 event discriminator,
ring 16 for firmware download and ring 17 for commands, the 8-bit sequence number and the
0xA0 packet-type constant — is **identical to MT7922 and to the publicly documented
CONNAC2/mt792x MCU interface.** `[C]`

---

## Open questions / needs hardware tracing

1. Does the MT7932 RAM firmware also accept the **unified (UNI) command scheme** described
   in §2.4? No unified command is issued on this part, so acceptance is untested. `[U]`
2. **Ring index 18** (16 descriptors) is provisioned and initialised but carries no traffic
   by default; the capability TLV `0x17` selector that switches a command-class ring index
   from 17 to 18 was never observed being set by firmware, and which of the two ring-index
   fields (WM command vs. WA command, both 17 here) it targets is not established. Confirm
   on hardware. `[U]`
3. The exact meaning of Layer-0 extended event **0x00** (generic 0x404-byte result) and
   **0xAA** (32-bit scalar) — the payload is returned to the requester uninterpreted. `[U]`
4. `u2PqId` (command header 0x22) is transmitted as **0**; the public Linux driver writes
   `MCU_PQ_ID(port, queue)` there. Whether the firmware ignores the field entirely on the
   PCIe path, or whether a non-zero value changes routing, is untested. `[U]`
5. Whether the firmware ever emits `EVENT_ID_INIT_EVENT_CMD_RESULT` (0xFD) post-boot for
   rejected commands, and hence whether §4.1's status codes are reachable at run time. `[U]`
6. Whether the sequence number is validated by the firmware (e.g. rejected if 0), or merely
   echoed. `[U]`
7. Whether the 10 s host response timeout reflects a firmware guarantee or is simply a
   generous host policy; the worst-case firmware turnaround for the slowest commands
   (calibration, eFuse buffer-mode page pushes) has not been measured. `[U]`
8. The `ucCmdVersion` field (header 0x2C) is always 0; whether the firmware supports
   non-zero structure versions for any command is untested. `[U]`
9. Whether the firmware emits any negative acknowledgement at all for a command that fails
   its length or index validation, or whether the response timeout really is the only
   feedback. No such event is decoded on this part, but that does not establish that the
   firmware never emits one. `[U]`
10. Whether `EVENT_ID_STA_UPDATE_FREE_QUOTA` update modes `2`/`3` (add/subtract) are ever
    actually used by this firmware, or whether it only ever sends the absolute form. The
    host implements all three. `[U]`
11. The exact field split of the calibration-block identifier in event `0xD7`. `[U]`
12. Whether a lost `SLEEPY_INFO` is ever re-sent after a timeout, or whether the firmware
    genuinely waits forever. `[U]`


---

# MT7932 — MAC Transmit Descriptor (TMAC TXD) Specification

## Scope

This document specifies the transmit descriptor that MT7932 silicon consumes: the fixed 8-double-word
MAC TX descriptor ("TMAC TXD"), the 8-double-word host append block that follows it inside the same
64-byte descriptor page, the byte-count and length arithmetic the host must perform, the descriptor
forms used for MCU commands and firmware download, the hardware-enforced field constraints, and the
per-frame rate-override encoding. MT7932's TX descriptor operation set, descriptor geometry
(`txd_append_size = 32`, `pse_header_length = 8`, extra TX byte count = 0) and packet-format default
are byte-for-byte identical to MT7922's; both are **CONNAC2 / TXD v2**, the same generation as the
public MT7921 (`mt7961`) and MT7922 parts. Everything below therefore matches
`ref/wifi/gen4m/include/nic/nic_connac2x_tx.h` + `ref/wifi/gen4m/nic/nic_txd_v2.c` and upstream
`mt76/mt76_connac2_mac.h` + `mt76/mt76_connac_mac.c::mt76_connac2_mac_write_txwi()` unless a delta is
called out explicitly. Confidence markers: `[C]` confirmed, `[L]` likely, `[U]` unverified.

---

## 1. Descriptor generation and geometry

| Property | Value | Conf |
|---|---|---|
| Descriptor generation | CONNAC2 "TXD v2" (gen4m `nic_txd_v2`, mt76 `mt76_connac2`) | [C] |
| Public part with identical layout | MT7921 (`mt7961`), MT7922 — bit-for-bit | [C] |
| Long-format descriptor size | 8 DW = **32 bytes** | [C] |
| Short-format descriptor size | 3 DW = **12 bytes** (only DW0/DW1 carry defined fields) | [C] |
| Descriptor padding after TXD | 0 bytes (`NIC_TX_DESC_PADDING_LENGTH = 0`) | [C] |
| Host append block | 8 DW = **32 bytes**, immediately after the TXD at byte offset 0x20 | [C] |
| Total descriptor page | 16 DW = **64 bytes** (= 1 "TXD page"; see DW7[31:30]) | [C] |
| Frame payload position | Byte offset **+64** of the TX buffer, i.e. immediately after the append block | [C] |
| Alignment | 4-byte (DW) minimum; the descriptor page is the first 64 bytes of the DMA-mapped buffer and is DMA'd as one WFDMA scatter segment of length 0x40 | [C] |
| Endianness | Little-endian 32-bit words | [C] |
| Max descriptor pages | 2 (DW7[31:30], 1 page = 16 DW = 64 bytes). MT7932 always uses 1 page | [C] |

### 1.1 Where the descriptor sits

For PCIe (the only host interface on MT7932) the TX data path is **cut-through / token based**:

```
DMA buffer:  +0x00 .. +0x1F   TMAC TXD          (8 DW, host-written)
             +0x20 .. +0x3F   TXD append block  (8 DW, host-written: MSDU IDs + ptr/len pairs)
             +0x40 .. +N      frame body        (host-written, *not* DMA'd by the ring descriptor)
```

The WFDMA TX-data ring descriptor points at byte 0 of this buffer with `SDL0 = 0x40` and
`LAST_SEC0 = 1` [C]; **only the 64-byte descriptor page is therefore fetched over the ring**.
That the frame body is fetched separately by the packet engine using the address/length pairs
in the append block is [L] — it is the standard CONNAC2 cut-through model and is consistent
with `SDL0 = 0x40`, but the fetch itself was not observed.

For MCU-command and firmware-download traffic the whole buffer (TXD + command header + payload) is a
single contiguous DMA segment; `SDL0 = TXD length + payload length`. [C]

---

## 2. Normal data TX descriptor — word-by-word field map

**How to read the confidence markers in this section.** The descriptor *geometry* for this
part (32-byte TXD, 32-byte append block, 64-byte page, `pse_header_length = 8`, extra TX byte
count 0, default `PKT_FMT`) is carried per-chip and is `[C]`. The **bit positions and field
names below are the public CONNAC2 "TXD v2" definition**, assumed to hold on MT7932 because
this part's TX-descriptor operation set is byte-identical to MT7922's: a `[C]` on a row means
*this part writes or reads that field at that position*, which pins it; a field the host never
touches carries the public definition only and should be read as `[L]`. Statements about what
the **hardware does** with a field (overwrites it, derives a value from it, assigns a sequence
number) are inferences from the public definition and are marked `[L]` individually.

`X` = defined in short format too (DW0/DW1 only). All other words exist only when DW1[31] = 1.

### DW0 — byte count, header hint, queue

| Bits | Name | Semantics | Conf |
|---|---|---|---|
| 15:0 | `TX_BYTE_COUNT` | Total byte count, see §4. 16-bit unsigned. | [C] |
| 22:16 | `ETH_TYPE_OFFSET` | Offset of the EtherType/LLC field from the start of the descriptor page's frame, **in 16-bit words**. 7 bits (0–127 words = 0–254 B). Pre-parsed header pointer used for header translation and checksum offload. | [C] |
| 24:23 | `PKT_FMT` (packet format) | `0` cut-through, `1` store-and-forward, `2` command, `3` PDA firmware download. MT7932 data frames use **0** (PCIe default from the chip record). | [C] |
| 31:25 | `Q_IDX` (queue index, 7 bits) | Bits[5:0] = hardware queue; **bit 6 (= DW0[31]) = port select**: 0 = LMAC (air), 1 = MCU (frame is consumed as a TXCMD, never transmitted). | [C] |

`ETH_TYPE_OFFSET` computation performed by the host:
* 802.3/Ethernet-II frame: `(14 - 2 + pse_header_length) / 2` = `(12 + 8)/2` = **10 words**.
* 802.11 frame: `(pse_header_length + MAC-header-length + LLC-length) / 2`.
`pse_header_length` for MT7932 = **8** (`CONNAC2X_NIC_TX_PSE_HEADER_LENGTH`). [C]

LMAC queue-index encoding (Q_IDX[5:0]) — identical to mt76 `MT_LMAC_*`:

| Value | Queue |
|---|---|
| 0x00–0x03 | WMM set 0, AC0..AC3 |
| 0x04–0x07 | WMM set 1, AC0..AC3 |
| 0x08–0x0B | WMM set 2, AC0..AC3 |
| 0x0C–0x0F | WMM set 3, AC0..AC3 |
| 0x10 | Band0 ALTX0 |
| 0x11 | Band0 BMC0 |
| 0x12 | Band0 BCN0 (beacon) |
| 0x13 | Band0 PSMP0 |
| 0x14–0x17 | Band1 ALTX/BMC/BCN/PSMP (single-band part: unused) |

MCU port encoding (Q_IDX[6] = 1): `0x20`–`0x23` = MCU RX Q0..Q3, `0x3E` = MCU FWDL port
(mt76 `MT_TX_MCU_PORT_RX_Q0`, `MT_TX_MCU_PORT_RX_FWDL`). [L]
MT7932 is single-band; only WMM set 0 (0x00–0x03) plus 0x10–0x13 are produced. [C]

### DW1 — station index, header format, TID, own-MAC, descriptor format

| Bits | Name | Semantics | Conf |
|---|---|---|---|
| 9:0 | `WLAN_IDX` | Station / WTBL index (10 bits). Selects the WTBL entry that supplies cipher, keys, rate table, sequence counters. Valid range enforced by the host: ≤ 0x30 on this part, otherwise a default index is substituted. | [C] |
| 10 | `VTA` | TX-vector apply/report enable. **Left 0 on MT7932** (mt76 also skips this bit for connac2 — it ORs in `MT_TXD1_VTA` only for non-connac2 parts). Delta vs. gen4m mainline, which sets it to 1. | [C] |
| 15:11 | `HDR_INFO` | Meaning depends on `HDR_FORMAT`, see below. | [C] |
| 17:16 | `HDR_FORMAT` | `0` = non-802.11 (802.3 / Ethernet-II, hardware performs header translation); `1` = command frame; `2` = 802.11 normal mode (host supplies the full 802.11 header); `3` = 802.11 enhanced mode. | [C] |
| 18 | `HDR_PAD_LEN` | Header padding length: 0 = 0 bytes, 1 = 2 bytes. MT7932 host writes **0**. | [C] |
| 19 | `HDR_PAD_MODE` | 1 = padding inserted at the **head** of the payload, 0 = at the **tail**. | [L] |
| 22:20 | `TID` | Access class / TID, 0–7. Written from the frame's user priority. | [C] |
| 23 | `AMSDU` (UTXB A-MSDU) | Frame body already is an A-MSDU built by the host. | [C] |
| 29:24 | `OWN_MAC` | BSS / own-MAC (OMAC) index, 6 bits. Selects the transmit BSSID/address set. | [C] |
| 30 | `TGID` | Band / PHY group id. Single-band part → **0**. | [C] |
| 31 | `FORMAT` | 0 = short format (3 DW), 1 = long format (8 DW). MT7932 always **1**. | [C] |

`HDR_INFO` (DW1[15:11]) decode by header format:

| HDR_FORMAT | Bit 11 | Bit 12 | Bit 13 | Bit 14 | Bit 15 |
|---|---|---|---|---|---|
| 0 — non-802.11 | `MRD` more-data | `EOSP` | `RMVL` remove VLAN tag | `VLAN` VLAN tag present | `ETYP` Ethernet-II (1) vs 802.3-LLC (0) |
| 1 — command | reserved (0) | reserved | reserved | reserved | reserved |
| 2 — 802.11 normal | `HDR_LEN[0]` | `HDR_LEN[1]` | `HDR_LEN[2]` | `HDR_LEN[3]` | `HDR_LEN[4]` — MAC header length in **16-bit words**, 0–31 words (0–62 B) |
| 3 — 802.11 enhanced | reserved | `EOSP` | `AMS` (frame is an A-MSDU) | reserved | reserved |

Notes:
* "802.11 with header translation" = `HDR_FORMAT = 0` together with `ETYP`/`VLAN`/`RMVL`; the MAC
  builds the 802.11 header from the WTBL entry and the 802.3 header supplied by the host. [C]
* For `HDR_FORMAT = 2` the "more data" and "power management" bits live in the frame's own Frame
  Control field, not in the descriptor; `EMRD` (DW3[2]) is the descriptor override. [C]
* MT7932 emits `HDR_FORMAT = 0` for data and `HDR_FORMAT = 2` for management/802.11 frames. [C]

### DW2 — frame type, protection, lifetime, power offset, fixed-rate enable

| Bits | Name | Semantics | Conf |
|---|---|---|---|
| 3:0 | `SUB_TYPE` | 802.11 FC subtype (copy of `FC[7:4]`). For 802.3 frames the host writes the QoS-data / data subtype. | [C] |
| 5:4 | `FRAME_TYPE` | 802.11 FC type (copy of `FC[3:2]`). | [C] |
| 6 | `NDP` | Frame is a null data packet (sounding). | [C] |
| 7 | `NDPA` | Frame is an NDP announcement. | [C] |
| 8 | `SOUNDING` | Sounding frame. | [C] |
| 9 | `RTS` | Force RTS/CTS protection for this frame. | [C] |
| 10 | `BC_MC_PKT` | Broadcast/multicast frame. Set together with `NO_ACK`. | [C] |
| 11 | `BIP` | Frame is BIP-protected management (uses IGTK from the WTBL); mutually exclusive with `PROTECT_FRAME`. | [C] |
| 12 | `DURATION` | 1 = use the Duration field supplied by software, 0 = hardware computes it. | [C] |
| 13 | `HTC_VLD` | Frame carries an HT Control field (hardware will not insert one). | [C] |
| 15:14 | `FRAG` | Fragment position: 0 = not fragmented, 1 = first, 2 = middle, 3 = last. | [C] |
| 23:16 | `MAX_TX_TIME` / remaining lifetime | Remaining life time in units of **64 TU** (≈65.536 ms). `0` = no lifetime limit. MSB (bit 23) reserved for hardware, host must write 0 → effective range 0–127. Hardware replaces this field with an absolute "max TX time" once queued. | [C] for the value the host writes and the 0–127 range; [L] for the hardware rewrite |
| 29:24 | `POWER_OFFSET` | Signed 6-bit per-frame TX-power offset. | [C] |
| 30 | `FIXED_RATE_MODE` | Source of the fixed rate: **0 = from this descriptor (DW6)**, 1 = from a chip control register. | [C] |
| 31 | `FIX_RATE` | 1 = use fixed rate, bypass rate adaptation. 0 = firmware/hardware rate adaptation. | [C] |

Host lifetime arithmetic (2000 ms per traffic class on this part):
`units = ((ms * 1000) >> 10) >> 6`, saturated to 127, and forced to 1 if the computation rounds a
non-zero request down to 0. 2000 ms → **30** units. A build-time configuration bit selects a 2^4
(16 TU) unit instead of 2^6; the hardware default in this image is 2^6. [C] / [U] for the 2^4 mode.

### DW3 — ACK/protection, retry budget, sequence number

| Bits | Name | Semantics | Conf |
|---|---|---|---|
| 0 | `NO_ACK` | No ACK expected; also masks the retry bit in the transmitted FC. | [C] |
| 1 | `PROTECT_FRAME` (`PF`) | Encrypt with the key from the WTBL entry selected by `WLAN_IDX`. **`PF = 0` is the "no-encrypt" control** — there is no separate cipher-suite or key-index field in the TXD; cipher, key index and key material all come from the WTBL. | [C] |
| 2 | `EMRD` | Extended "more data" override (long format). | [C] |
| 3 | `EEOSP` | Extended EOSP override (long format). | [C] |
| 4 | `DAS` (`da_select`) | Destination-address source: 0 = from the MSDU, 1 = from the WTBL. | [C] |
| 5 | `TIMING_MEASURE` (`tm`) | Timing-measurement frame: hardware inserts the departure timestamp (FTM / 802.11 timing measurement). This is the **only timestamp-insertion control in the CONNAC2 TXD**; the timestamp-offset fields of CONNAC3 (`MT_TXD6_TIMESTAMP_OFS_*`) do not exist here. | [C] |
| 10:6 | `TX_COUNT` | Transmission attempts already made. Host writes 0; hardware updates. | [C] |
| 15:11 | `REM_TX_COUNT` | Remaining transmit-count limit (retry budget), 5 bits, 0–31. Host default on this part = **30**; broadcast/multicast is left unlimited. | [C] |
| 27:16 | `SEQ` | Host-supplied 12-bit sequence number. Only meaningful when `SN_VALID` = 1. | [C] |
| 28 | `BA_DISABLE` | Do not aggregate this MPDU into an A-MPDU / do not use Block Ack. | [C] |
| 29 | `SW_POWER_MGMT` | 1 = the PM bit of the transmitted frame is taken from software. **Left 0 on MT7932** (mt76 ORs in `MT_TXD3_SW_POWER_MGMT` only for non-connac2 parts); set only when the host explicitly requests the SW-PS option. | [C] |
| 30 | `PN_VALID` | 1 = use the host-supplied packet number in DW4/DW5[31:16]; 0 = hardware supplies the PN from the WTBL. | [C] |
| 31 | `SN_VALID` | 1 = use `SEQ` from DW3[27:16]; **0 = hardware assigns the sequence number from the per-TID counter in the WTBL.** | [C] for the bit and for the host writing 0; [L] for the hardware-assignment behaviour (the per-TID counters are, however, directly visible in UWTBL DW2–DW4, MAC-tables §B.3.3) |

### DW4 — packet number, low half

| Bits | Name | Semantics | Conf |
|---|---|---|---|
| 31:0 | `PN_LOW` | PN[31:0]. Only used when `PN_VALID` = 1. | [C] |

### DW5 — TX-status request, packet ID, packet number high half

| Bits | Name | Semantics | Conf |
|---|---|---|---|
| 7:0 | `PID` (packet ID) | **Completion/token identifier that matches a TX-status (TXS) report back to the submitted frame.** Allocated per (WLAN index, packet class). Value **0x7F is reserved** on this part and is used when a status report is wanted for logging only; allocation wraps at 0x7E or 0x7F. mt76 reserves 0/1/2 and starts real IDs at 3. | [C] |
| 8 | `TX_STATUS_FMT` | TXS report format: 0 = MPDU-based, 1 = PPDU-based. | [L] |
| 9 | `TX_STATUS_MCU` | Generate a TXS report and deliver it to the MCU (which may relay it to the host as an event). | [C] |
| 10 | `TX_STATUS_HOST` | Generate a TXS report and deliver it directly to the host RX path. | [C] |
| 13:11 | reserved | 0 | [C] |
| 14 | `ADD_BA` | Frame is an ADDBA request; hardware may pre-arm the BA session. | [C] |
| 15 | `MD` | Reserved/undocumented single-bit control (`CONNAC2X_TX_DESC_MD_MASK`, mt76 `MT_TXD5_MD`). Never written on this part. | [U] |
| 31:16 | `PN_HIGH` | PN[47:32]. Only used when `PN_VALID` = 1. | [C] |

**Two independent completion identifiers exist; do not conflate them.** [C]

| Identifier | Width | Where the host writes it | What returns it | What it closes |
|---|---|---|---|---|
| **PID** (packet ID) | 8 bits | TXD **DW5[7:0]** | the MAC TX-status (TXS) record, field `TXS3[31:24]` (RX-descriptor section §6.2) | the **MAC-level** loop: per-MPDU delivery result (ACK/BA, retries, final rate) |
| **MSDU token** | 15 bits | TXD **append block**, `MSDU_ID[0..3]` bits [14:0] (§3.1) | the TX-free / MSDU report, `PKT_TYPE = 6` (RX-descriptor section §6.3) | the **buffer-ownership** loop: the packet engine has finished with the buffer and the host may reclaim it |

A frame may carry both, one, or neither: the MSDU token is mandatory for every data frame
(the descriptor checker rejects a data descriptor without a valid token entry, §7), while
`PID` and the `TX_STATUS_*` bits are only set when a per-frame status report is wanted.

### DW6 — fixed-rate control (meaningful only when DW2[31] `FIX_RATE` = 1)

| Bits | Name | Semantics | Conf |
|---|---|---|---|
| 1:0 | `BW` | Fixed bandwidth: 0 = 20 MHz, 1 = 40, 2 = 80, 3 = 160/80+80. | [C] |
| 2 | `FIXED_BW` | Enable the fixed bandwidth in [1:0]. gen4m treats [2:0] as one 3-bit field. | [L] |
| 3 | `DYN_BW` | Dynamic-bandwidth RTS. | [C] |
| 7:4 | `ANT_ID` | Antenna index (4 bits). | [C] |
| 9:8 | reserved | 0 | [C] |
| 10 | `SPE_ID_IDX` (`spe idx sel`) | Spatial-extension index source: **0 = take the index from DW7[15:11]** (`ENUM_SPE_SEL_BY_TXD`), 1 = take it from the WTBL. MT7932 writes 0. | [C] |
| 11 | `LDPC` | 0 = BCC, 1 = LDPC. | [C] |
| 13:12 | `HE_LTF` | HE LTF type: 0 = 1x, 1 = 2x, 2 = 4x. | [C] |
| 15:14 | `GI` | Guard interval. Non-HE: 0 = long GI, 1 = short GI. HE: 0 = 0.8 µs (SGI), 1 = 1.6 µs (MGI), 2 = 3.2 µs (LGI). | [C] |
| 29:16 | `TX_RATE` | 14-bit rate code, see §8. | [C] |
| 30 | `TXE_BF` | Explicit beamforming. | [C] |
| 31 | `TXI_BF` | Implicit beamforming. | [C] |

### DW7 — offload requests, spatial extension, hardware debug overlay

| Bits | Name | Semantics | Conf |
|---|---|---|---|
| 9:0 | `TXD_ARRIVAL_TIME` | Written by hardware when the descriptor is admitted. Host writes 0. | [C] |
| 10 | `HW_AMSDU` | Allow hardware to aggregate this MSDU with others into an A-MSDU. Cleared whenever a per-frame TXS is requested (TXS is MPDU-based). | [C] |
| 15:11 | `SPE_IDX` | Spatial-extension index, 5 bits (0–31). Used when DW6[10] = 0 and `FIX_RATE` = 1. | [C] |
| 19:16 | `PP_SUB_TYPE` | Copy of the 802.11 FC subtype, consumed by the packet processor (cut-through path). | [C] |
| 21:20 | `PP_TYPE` | Copy of the 802.11 FC type, consumed by the packet processor. | [C] |
| 27:16 | `PSE_FID` | *Hardware-written overlay* after queuing: the 12-bit PSE frame ID of this descriptor (0x000–0xFFE; 0xFFF = fault). Overlaps `PP_SUB_TYPE`/`PP_TYPE`/`CTXD*`. | [C] that the host's debug path decodes this word that way; [L] that the hardware writes it |
| 25:23 | `CTXD_CNT` | Chained-TXD count (overwritten by the packet processor with PSE_FID). | [C] |
| 26 | `CTXD` | Chained-TXD flag (overwritten by the packet processor with PSE_FID). | [C] |
| 28 | `IP_CHKSUM_OFFLOAD` (`i`) | Request hardware IPv4 header checksum generation. | [C] |
| 29 | `TCP_UDP_CHKSUM_OFFLOAD` (`UT`) | Request hardware TCP/UDP checksum generation. | [C] |
| 31:30 | `TXD_LEN` | Descriptor length in 64-byte pages minus 1: `0` = 1 page (16 DW), `1` = 2 pages (32 DW). MT7932 uses **0**. | [C] |

**Checksum-offload delta:** although DW7[29:28] exist and are wired to the same bits as on
MT7921/MT7922, they are never set on MT7932 — TX checksum offload is left disabled. [C]

### 2.1 Additional per-frame controls not carried in the TXD

There is no cipher-suite selector, key index, "duplicate-detect" flag or A-MPDU density field in the
CONNAC2 TXD. Those live in the WTBL entry addressed by `WLAN_IDX`. The nearest TXD equivalents are:
`PF` (encrypt/don't encrypt), `BIP`, `PN_VALID`+PN (replay counter override), `BA_DISABLE` (A-MPDU
inhibit) and `HW_AMSDU`/`AMSDU`/`AMS` (A-MSDU control). The `ETH_TYPE_OFFSET` field is the
"pre-parsed header" hint. [C]

---

## 3. Extension / append blocks

CONNAC2 does not append optional descriptor *words* to the TMAC TXD itself — the TXD is always 8 DW
(long) or 3 DW (short). Extension happens in two orthogonal ways:

### 3.1 Host TXD-append block (PCIe) — 32 bytes at offset 0x20

Always present on the PCIe data path; `txd_append_size` = **0x20** for this part.
Zero-filled before use.

| Offset | Size | Field | Semantics | Conf |
|---|---|---|---|---|
| 0x20 | u16 | `MSDU_ID[0]` | bits[14:0] MSDU token id, bit15 = valid | [C] |
| 0x22 | u16 | `MSDU_ID[1]` | " | [C] |
| 0x24 | u16 | `MSDU_ID[2]` | " | [C] |
| 0x26 | u16 | `MSDU_ID[3]` | " | [C] |
| 0x28 | u32 | `PTR0` | DMA address bits[31:0] of buffer 0 | [C] |
| 0x2C | u16 | `LEN0` | bits[11:0] length; bits[14:12] = DMA address bits[34:32]; bit15 = `ML` (last segment of this MSDU) | [C] |
| 0x2E | u16 | `LEN1` | same encoding for buffer 1 | [C] |
| 0x30 | u32 | `PTR1` | DMA address bits[31:0] of buffer 1 | [C] |
| 0x34 | u32 | `PTR2` | buffer 2 | [C] |
| 0x38 | u16 | `LEN2` | buffer 2 | [C] |
| 0x3A | u16 | `LEN3` | buffer 3 | [C] |
| 0x3C | u32 | `PTR3` | buffer 3 | [C] |

Note the interleaved `{ptr0, len0, len1, ptr1}` 12-byte pair structure: entry *i* uses
`PTR(i&1 ? 1 : 0)` inside pair `i>>1`. Up to **4 buffers / 4 MSDU ids per descriptor page**. MT7932's
DMA address mask is 32 bits, so `LEN[14:12]` is always 0. [C]

* Single-MSDU submission fills slot 0 only, with `ML = 1`.
* Software A-MSDU aggregation fills slots 0..N-1, each with its own MSDU token, and the *first*
  descriptor's `TX_BYTE_COUNT` is rewritten to the aggregate size. [C]
* The MSDU token pool on this part holds **6000** entries, so token ids stay inside the 15-bit
  field. [C]

### 3.2 Second descriptor page (chained TXD)

`DW7[31:30] = 1` declares a two-page (128-byte) descriptor; `DW7[26] CTXD` and `DW7[25:23] CTXD_CNT`
describe the chain. MT7932 never uses this; a data descriptor whose page count is not 1 must be
rejected (§7 rule 6). [C]

### 3.3 Interface-specific append variants (not used on MT7932)

| Variant | Size | Used by | Conf |
|---|---|---|---|
| CR4 / WA-CPU append (`u2PktFlags`, `u2MsduToken`, `ucBssIndex`, `ucWtblIndex`, `ucBufNum`, 6×ptr, 6×len) | 44 B | parts with `is_support_cr4`/`is_support_wacpu`; MT7932 has both **FALSE** | [C] |
| USB/SDIO extension DW8..DW15 (`DW8[5:4] L_TYPE`, `DW8[3:0] L_SUB_TYPE`) | +32 B | USB/SDIO CONNAC2 parts. MT7932 is PCIe-only; the FC type/subtype copy goes to DW7[21:16] instead | [C] |

---

## 4. Byte-count and length accounting

### 4.1 `TX_BYTE_COUNT` (DW0[15:0]) for data frames

```
TX_BYTE_COUNT = 32                     /* the TMAC TXD itself, always the long format size */
              + frame_length           /* 802.3 or 802.11 frame as handed to the MAC */
              + extra_tx_byte_count    /* chip record value; MT7932 = 0 */
```

Therefore, on MT7932: **`TX_BYTE_COUNT = 32 + frame_length`**. [C]

It does **not** include:
* the 32-byte host append block,
* the FCS (added by the PHY),
* encryption expansion (IV/EIV/MIC/ICV added by the MAC),
* any header padding declared in DW1[18],
* the 802.11 header the MAC synthesises when `HDR_FORMAT = 0`.

`frame_length` is also what goes into `LEN0` of the append block (bits[11:0]) — so a single buffer
segment is limited to **4095 bytes**; longer frames must be split across append slots. [C]

Parts that need the append block counted (WA-CPU designs) add `txd_append_size` instead of
`extra_tx_byte_count`; MT7932 does **not**. [C]

### 4.2 Minimum frame length and padding

| Rule | Value | Conf |
|---|---|---|
| Minimum Ethernet frame accepted by the host path | `len > 14 + u4MinTxLen`; MT7932 `u4MinTxLen = 0` → **≥ 15 bytes** (14-byte 802.3 header + ≥1 payload byte) | [C] |
| MT7925 / CONNAC3 comparison | `u4MinTxLen = 2` → ≥ 17 bytes. **MT7932 matches MT7921/MT7922, not MT7925.** | [C] |
| Header padding | DW1[18]: 0 = none, 1 = 2 bytes; DW1[19] selects head/tail. MT7932 writes 0/0. Used on other CONNAC2 designs to 4-byte-align the payload after a QoS+HTC header. | [C] |
| Descriptor-page padding | none (`NIC_TX_DESC_PADDING_LENGTH = 0`); the append block starts at exactly +0x20 | [C] |
| DW padding of the DMA segment | not required on PCIe (a 64-byte page is inherently DW-aligned); the USB/SDIO paths pad the whole transfer to a DW boundary | [C] |

---

## 5. MCU command descriptor and firmware-download descriptor

All MCU traffic reuses the **same 32-byte TMAC TXD** followed by a command header. Only DW0 and DW1
carry meaningful values; DW2..DW7 are left zero.

### 5.1 Normal MCU command TXD (`PKT_FMT = 2`)

| Word/Bits | Value | Conf |
|---|---|---|
| DW0[15:0] `TX_BYTE_COUNT` | total buffer length **including** the 32-byte TXD and the 32-byte command header | [C] |
| DW0[22:16] `ETH_TYPE_OFFSET` | 0 (untouched) | [C] |
| DW0[24:23] `PKT_FMT` | **2** (command) | [C] |
| DW0[31:25] `Q_IDX` | 0 (untouched). Routing to the MCU is by the WFDMA command ring, not by `Q_IDX`. | [C] |
| DW1[17:16] `HDR_FORMAT` | **1** (command frame) | [C] |
| DW1, all other bits | 0 | [C] |
| DW2..DW7 | 0 | [C] |

Command header, immediately after the TXD (this is `struct CONNAC2X_WIFI_CMD` minus the TXD):

| Offset | Size | Field | Notes | Conf |
|---|---|---|---|---|
| 0x20 | u16 | `u2Length` | `TX_BYTE_COUNT − 32` (command header + payload) | [C] |
| 0x22 | u16 | `u2PqId` | left 0 on MT7932 (legacy CONNAC1 port/queue tag) | [C] |
| 0x24 | u8 | `ucCID` | command id | [C] |
| 0x25 | u8 | `ucPktTypeID` | must be **0xA0** (`CMD_PACKET_TYPE_ID`) for a command packet — the same constant as the initialisation-time header (§5.2) and as the MCU-protocol section §1.3. This byte and `ucSetQuery` at 0x26 are emitted together as one 16-bit field, so `0xA0` is fixed for every command. **Do not substitute `0x20`**: public CONNAC2 headers carry a stale `/* Must be 0x20 */` comment on this field, but `0x20` is not what the silicon is given. | [C] |
| 0x26 | u8 | `ucSetQuery` | 0 = query, 1 = set | [C] |
| 0x27 | u8 | `ucSeqNum` | monotonically increasing command sequence number, echoed in the response event | [C] |
| 0x28 | u8 | `ucD2B0Rev` | 0 (hardware may modify) | [C] |
| 0x29 | u8 | `ucExtenCID` | extended command id | [C] |
| 0x2A | u8 | `ucS2DIndex` | source-to-destination routing index | [C] |
| 0x2B | u8 | `ucExtCmdOption` | extended command option | [C] |
| 0x2C | u8 | `ucCmdVersion` | 0 | [C] |
| 0x2D–0x3F | 19 B | reserved | 0 | [C] |
| 0x40 | — | payload | command body starts here | [C] |

Total header before the payload: **64 bytes** (32 TXD + 32 command header). [C]

### 5.2 Initialisation-time (pre-firmware) command TXD

Used before the RAM firmware is running. Same 32-byte TXD; the header that follows is the smaller
`INIT_HIF_TX_HEADER` + `INIT_WIFI_CMD` pair, most of which is reserved padding whose only purpose is
to keep the payload at the same +0x40 offset as the full command form.

| Word/Bits | Value | Conf |
|---|---|---|
| DW0[15:0] | total buffer length including the 32-byte TXD | [C] |
| DW0[22:16] | preserved (0) | [C] |
| DW0[24:23] `PKT_FMT` | **2** when `ucCID != 0` (init command); **3** when `ucCID == 0` (PDA firmware download) | [C] |
| DW0[31:25] | preserved (0) | [C] |
| DW1[17:16] `HDR_FORMAT` | **1** (command frame) | [C] |
| DW2..DW7 | 0 | [C] |

| Offset | Size | Field | Notes | Conf |
|---|---|---|---|---|
| 0x20 | u16 | `u2TxByteCount` | `TX_BYTE_COUNT − 32` | [C] |
| 0x22 | u16 | `u2PQ_ID` | architecturally "must be 0x8000 (Port 1, Queue 0)"; **left 0 on CONNAC2** because DW0/DW1 already carry the routing | [C] |
| 0x24 | u8 | `ucCID` | init command id (0 ⇒ firmware-download data) | [C] |
| 0x25 | u8 | `ucPktTypeID` | must be **0xA0** for an init command packet | [C] |
| 0x26 | u8 | reserved | 0 | [C] |
| 0x27 | u8 | `ucSeqNum` | command sequence number | [C] |
| 0x28–0x3F | 24 B | reserved (`u4Reserved`, `au4D3toD7Rev[5]`) | 0 — present purely so the payload lands at +0x40 | [C] |
| 0x40 | — | payload | init-command body | [C] |

The meaningful part of the init header is therefore only **8 bytes** (0x20–0x27) versus 12 bytes of
`u2Length/u2PqId/CID/PktType/SetQuery/Seq` plus the extended fields in the full command form. There
is *no* set/query, extended-CID, S2D or version field before firmware is running. [C]

### 5.3 Firmware-download data frames

* `ucCID == 0` identifies a firmware-download payload.
* When the caller only needs the payload pointer, **no descriptor is written at all** — the buffer
  is handed back at offset 0 and the download payload is written from byte 0. [C]
* When a descriptor is written, `PKT_FMT = 3` (PDA firmware download), `HDR_FORMAT = 1`, and
  `TX_BYTE_COUNT` is the total buffer length. [C]
* Firmware-download traffic is posted on a **separate WFDMA command ring** from ordinary MCU
  commands, and is exempt from the command-size limit (§7). [C]

### 5.4 MCU-delivered "security frame" TXD template

A further TXD form exists that describes a frame the *firmware* will transmit on the host's behalf
(offloaded security / keep-alive frames). It is a long-format TXD with `PKT_FMT = 2` (delivered to
the MCU), `HDR_FORMAT = 0` with `ETYP = 1` (802.3 payload), `ETH_TYPE_OFFSET = pse_header_length/2 + 6`,
`TX_BYTE_COUNT = 32 + frame_length`, `Q_IDX` from the MCU-port traffic class, TID = 0, header padding
= 0, `FIX_RATE = 1` with a 14-bit rate code taken from the BSS configuration, and `PID`/`TX_STATUS_MCU`
set when a completion is required. This descriptor form is **not present in public gen4m**; it is an
MT7932/MT7922 addition. [C]

---

## 6. Host-interface descriptor prefix

**None on MT7932.** No transport header is prepended ahead of the TMAC TXD on the PCIe path;
the MAC TX descriptor is the first byte of the DMA buffer.
[C]

For completeness, the *transport-level* structure that carries the descriptor is the 16-byte WFDMA
TX ring descriptor (separate memory, not a prefix):

| Word | Bits | Field | Data-frame value | Command value |
|---|---|---|---|---|
| DW0 | 31:0 | `SDP0` | DMA address of the descriptor page | DMA address of the command buffer |
| DW1 | 13:0 | `SDL1` | 0 | 0 |
| DW1 | 14 | `LAST_SEC1` | 0 | 0 |
| DW1 | 15 | `BURST` | 0 | 0 |
| DW1 | 29:16 | `SDL0` | **0x40** (32-byte TXD + 32-byte append) | `TXD length + payload length` |
| DW1 | 30 | `LAST_SEC0` | 1 | 1 |
| DW1 | 31 | `DMA_DONE` | 0 | 0 |
| DW2 | 31:0 | `SDP1` | 0 | 0 |
| DW3 | 15:0 | `SDP0_EXT` (segment-0 address extension, only meaningful when the global-config address-extension bit is set — it is not on this part) | 0 | 0 |
| DW3 | 31:16 | `SDP1_EXT` | 0 | 0 |

`SDL0` for data frames is `txd_append_size + 32` and must not exceed 0x40. Designs with a WA-CPU
append use `txd_append_size + 0x68`. [C]

The SDIO/USB variants of this family *do* prepend a 4-byte transport header and use a 9–16 DW TXD;
neither applies to MT7932. [C]

---

## 7. Sanity constraints

Constraints that must hold before a descriptor is posted; a violation must abort the
submission:

| # | Condition | Applies to | Conf |
|---|---|---|---|
| 1 | `PKT_FMT` (DW0[24:23]) **must be 2** | command descriptors | [C] |
| 2 | `Q_IDX` (DW0[31:25]) **must equal the index of the WFDMA TX data ring** the descriptor is posted on | data descriptors | [C] |
| 3 | `Q_IDX` bit 6 (DW0[31]) **must be 0**. If set, the hardware treats the frame as a TXCMD and the station index is ignored. The checker forcibly clears it. | data descriptors | [C] |
| 4 | `PKT_FMT` **must be 0** (cut-through) | data descriptors | [C] |
| 5 | `HDR_FORMAT` (DW1[17:16]) **must be 0**; value 1 (command) is reported as an illegal header format for data | data descriptors on the checked path | [C] |
| 6 | Descriptor page count must be 1 (`DW7[31:30] = 0`) | data descriptors | [C] |
| 7 | A valid MSDU token entry must exist | data descriptors | [C] |

Other hard limits:

| Limit | Value | Conf |
|---|---|---|
| MCU command total size (TXD + header + payload) | **≤ 0x640 (1600) bytes**; exceeding it aborts the submission | [C] |
| Firmware-download packets | exempt from the 1600-byte limit | [C] |
| Scatter-gather segments per descriptor page | **4** (4 MSDU ids, 4 address/length pairs) | [C] |
| Bytes per scatter-gather segment | **≤ 4095** (12-bit length field) | [C] |
| DMA address width usable | 32 bits on MT7932 (the 3 extra address bits in `LEN[14:12]` stay 0) | [C] |
| `WLAN_IDX` | 10-bit field; host clamps to ≤ 0x30 and substitutes a default index above that | [C] |
| `REM_TX_COUNT` | 5 bits, 0–31 | [C] |
| `MAX_TX_TIME` | bit 23 reserved for hardware → host range 0–127 | [C] |
| `PID` | 0x7F reserved | [C] |
| PSE frame id (`DW7[27:16]`, hardware-written) | 0x000–0xFFE; 0xFFF = fault | [C] |
| MSDU token pool | 6000 entries | [C] |

Illegal / conflicting combinations:

* `BIP = 1` together with `PROTECT_FRAME = 1` — mutually exclusive; the BIP path clears `PF`. [C]
* `HW_AMSDU = 1` together with a per-frame TXS request — TXS is MPDU-based, so `HW_AMSDU` must be
  cleared whenever `PID`/`TX_STATUS_*` are used. [C]
* `HW_AMSDU = 1` together with a host-supplied sequence number (`SN_VALID`) — mt76 clears
  `HW_AMSDU` in that case. [C]
* `FIX_RATE = 1` on an 802.3-format descriptor: fixed rate is only defined for descriptors that
  describe an 802.11 frame; the fixed-rate path also sets `HTC_VLD` because hardware will not insert
  an HT Control field for management/control frames. [C]
* Writing DW6 requires at least 0x1C bytes of descriptor buffer; a shorter buffer skips the
  fixed-rate word. [C]

---

## 7A. MSDU token lifecycle, exhaustion and back-pressure

The 15-bit MSDU token in the host TXD-append block (§3.1) is the identity of a transmit
buffer for the whole time the hardware owns it. It is the only handle by which the buffer
comes back, so its accounting is load-bearing: a leaked token is a permanently lost buffer,
and a token returned twice frees a buffer that is still in use.

### 7A.1 Pool geometry

| Property | Value | Conf |
|---|---|---|
| Pool size | **6000** entries, ids `0`…`5999` | [C] |
| Field width in the TXD append block | 15 bits + a validity bit | [C] |
| Field width in the TX-free report | 15 bits (report version 3), 16 bits (versions 0–2) | [C] |
| Allocation discipline | LIFO free stack; the used count is the stack pointer | [C] |
| Per-token state | in-use flag, acquisition timestamp, owning descriptor cell, owning ring index | [C] |

The pool is *not* a hardware resource — the silicon only echoes the value — but its size is
bounded by the report field, and the host's choice of 6000 keeps every id inside the 15-bit
form so that report version 3 can be used. `[C]`

### 7A.2 Lifecycle and the rules that must not be broken

```
acquire  ->  write id into the TXD append block, record acquisition time
         ->  post the descriptor
         ...
TX-free report entry names the id
         ->  stop the frame's lifetime timer
         ->  decrement the owning ring's outstanding-token count
         ->  clear the descriptor cell's back-pointer
         ->  release the buffer, push the id back on the free stack
```

Rules `[C]`:

* **A token must not be returned twice.** The host must test the in-use flag before acting
  on a report entry; a report naming a token that is already free is a firmware or
  transport fault and the entry must be skipped, not processed.
* **A token id ≥ the pool size must be rejected**, not used as an index. An out-of-range id
  is a hard transport fault and should be treated as one. `[C]`
* **The free stack must not be pushed when nothing is outstanding** ("token pool full").
* **The pool must be reset — not incrementally repaired — after any reset.** Resetting the
  pool walks all 6000 entries, releases every buffer still marked in use, and rebuilds the
  free stack in index order. This is a required step of the L1 error-recovery sequence, of
  the resume-from-suspend path when a sub-system error occurred while suspended, and of
  every L0.5/L0 bring-up. Skipping it leaks every token that was in flight at the moment of
  the fault. `[C]`

### 7A.3 Exhaustion

Acquisition simply fails when the used count reaches the pool size, and the transmit
submission fails with it. **There is no firmware notification of token exhaustion and no
event that reports it** — the host learns only from its own accounting. `[C]` The host must therefore either queue the frame and retry, or drop
it; it must not spin, because the only thing that frees a token is a TX-free report arriving
on the RX path, which requires the receive path to keep running.

Two host-side limiters sit above the pool and exist to stop one queue from consuming it
`[C]`:

| Limiter | Value |
|---|---|
| Outstanding tokens per hardware TX queue | derived from that queue's descriptor count (observed: `5 ×` the count); lowered to a fixed **192** when the low-latency/concurrent-role condition is in force |
| Outstanding tokens tracked per queue | a per-queue counter incremented on acquire and decremented when the token is returned |

The per-queue caps are host policy, not a silicon limit, but some limiter is required: the
hardware will happily accept descriptors until the pool is empty, at which point *all*
queues stall together rather than just the offending one. `[L]`

### 7A.4 Transmit lifetime and the report timeout

Each frame carries an acquisition timestamp and a lifetime timer. Two independent deadlines
apply `[C]`:

* **Firmware-side transmit lifetime.** `[L]` The firmware is understood to age frames out of
  its packet buffer (`TX_DELAY_CHK`/lifetime fields, DW2) and to return the token with a
  failure indication, so a frame that is never transmitted still returns its token —
  *provided the firmware is alive*. This is inferred from the lifetime field and the host's
  handling of failed completions, not observed.
* **Host-side MSDU-report timeout.** If a token is not returned within the host's own
  deadline, the host declares a transmit fault. This is one of the recovery reasons ("TX
  error") in the graded-recovery table of the power/reset section. The deadline exists
  precisely because a firmware fault, a DMA-scheduler stall or a lost report produces no
  other symptom: the ring indices keep advancing, the interrupt keeps firing, and only the
  token accounting shows that frames are not completing.

A driver that does not implement a token-return deadline will hang on the first lost
TX-free report with no diagnostic at all. `[C]`

### 7A.5 Firmware-side acceptance of a host descriptor

The checks in §7 are the ones the *host* performs before posting `[C]`. The firmware performs
its own set on arrival and is understood to **drop the frame without any report** when one
fails — in particular the MSDU token may or may not come back, so the host's report timeout is
the backstop. Conditions the firmware rejects, read from its own validation and log sites and
inferred as to outcome — `[L]` for the whole table (see also §6.1 and open question 9 of the
MCU-protocol section):

| Condition | Firmware behaviour |
|---|---|
| Descriptor fails the generic TXD sanity check (a numbered check set) | frame dropped, firmware-side log only |
| `PID` outside the valid range | frame dropped ("HIF TX packet with invalid PID") |
| Frame references a BSS context that is inactive or invalid | frame dropped |
| Management frame references an invalid station record | frame dropped |
| Packet type / format field not one the firmware handles | frame dropped |
| A forwarding-path descriptor with the "for WFDMA" flag set | frame dropped |

Practical consequence: **the host must not post data frames for a BSS index between
deactivating it and reactivating it**, and must not post frames referencing a station index
it has already deleted. Both are ordinary races in a driver that tears interfaces down
asynchronously, and both are silent.

---

## 8. Rate control

**Rate selection on MT7932 is firmware/hardware-controlled by default.** With `FIX_RATE`
(DW2[31]) = 0 the MAC's rate-adaptation engine picks the rate from the rate table in the WTBL entry
addressed by `WLAN_IDX`; the host supplies no rate at all for ordinary data traffic. There is no
per-frame rate table in the descriptor. [C]

The host may override on a per-frame basis:

| DW2[31] `FIX_RATE` | DW2[30] `FIXED_RATE_MODE` | Meaning |
|---|---|---|
| 0 | x | Rate adaptation (default for data) |
| 1 | 0 | Fixed rate taken from **DW6 of this descriptor** |
| 1 | 1 | Fixed rate taken from a chip control register (global) |

In practice the override is used for management frames, broadcast/multicast frames and frames
flagged "use minimum rate". [C]

### 8.1 `TX_RATE` (DW6[29:16]) encoding — 14 bits

Bit numbering below is *within the 14-bit rate code*; add 16 for the DW6 bit position.

| Rate-code bits | DW6 bits | Field | Encoding |
|---|---|---|---|
| 5:0 | 21:16 | `RATE_IDX` | CCK/OFDM hardware rate index, or HT MCS 0–31. For VHT/HE only bits 3:0 are the MCS. |
| 4 | 20 | `DCM` | HE dual-carrier modulation (overlays `RATE_IDX` for HE modes) |
| 5 | 21 | `SU_EXT_TONE` | HE extended-range / 106-tone RU (overlays `RATE_IDX` for HE modes) |
| 9:6 | 25:22 | `RATE_MODE` (PHY mode) | see table below |
| 12:10 | 28:26 | `NSS` | number of spatial streams **minus 1** (NSTS) |
| 13 | 29 | `STBC` | 1 = space-time block coding |

PHY mode (`RATE_MODE`) values — identical to mt76 `enum mt76_phy_type`:

| Value | Mode |
|---|---|
| 0 | CCK |
| 1 | OFDM |
| 2 | HT mixed mode |
| 3 | HT green field |
| 4 | VHT |
| 5–7 | reserved (7 = PLR test mode) |
| 8 | HE SU |
| 9 | HE extended-range SU |
| 10 | HE trigger-based |
| 11 | HE MU |
| 13–15 | EHT (CONNAC3 only; not supported by MT7932) |

Companion fields, all in DW6 (see §2, DW6 table):

| Field | DW6 bits | Encoding |
|---|---|---|
| Bandwidth | 1:0 | 0 = 20 MHz, 1 = 40, 2 = 80, 3 = 160/80+80 |
| Fixed-BW enable | 2 | 1 = honour bits[1:0] |
| Dynamic-BW RTS | 3 | 1 = dynamic bandwidth signalling |
| Antenna index | 7:4 | 4-bit antenna selection |
| Spatial-extension select | 10 | 0 = index from DW7[15:11], 1 = index from WTBL |
| LDPC | 11 | 0 = BCC, 1 = LDPC |
| HE LTF | 13:12 | 0 = 1x, 1 = 2x, 2 = 4x |
| GI | 15:14 | non-HE: 0 = long, 1 = short. HE: 0 = 0.8 µs, 1 = 1.6 µs, 2 = 3.2 µs |
| Explicit BF | 30 | TXE BF |
| Implicit BF | 31 | TXI BF |

Spatial-extension index: **DW7[15:11]**, 5 bits (0–31), only consulted when `FIX_RATE = 1` and
DW6[10] = 0. On MT7932 it is derived from the active antenna-path preference with a
1-stream/1-transmitter regulatory adjustment applied. [C]

The fixed-rate word DW6 is composed as
`(rate_code << 16) | bandwidth[2:0]`, then ORing in `GI = 1` for short GI (bit 14), `LDPC` (bit 11),
`DYN_BW` (bit 3) and the antenna index in bits[7:4]. HE extended-range/DCM operation additionally
requires `HE_LTF = 1` (2x) and `GI = 1` (1.6 µs). [C]

---

## 9. Delta summary — MT7932 vs. public CONNAC2 parts

| Item | MT7921 / MT7922 (public) | MT7932 | Conf |
|---|---|---|---|
| TXD generation, all bit positions | CONNAC2 v2 | **identical** | [C] |
| TXD size / append size / page size | 32 / 32 / 64 B | **identical** | [C] |
| `pse_header_length` | 8 | 8 | [C] |
| Extra TX byte count | 0 | 0 | [C] |
| Default `PKT_FMT` (PCIe) | 0 (cut-through) | 0 | [C] |
| Minimum Ethernet TX length | > 14 B | > 14 B (unlike MT7925/CONNAC3, which need > 16) | [C] |
| `VTA` (DW1[10]) | left 0 on connac2 | left 0 | [C] |
| `SW_POWER_MGMT` (DW3[29]) | left 0 on connac2 | left 0 unless explicitly requested | [C] |
| TX checksum offload (DW7[29:28]) | supported, driver-dependent | wired but **never enabled** on this part | [C] |
| TX-status delivery | `TX_STATUS_HOST` in mt76 | both `TX_STATUS_MCU` and `TX_STATUS_HOST` used; MCU delivery is the default and host delivery is selected by configuration | [C] |
| Reserved PID | 0/1/2 reserved in mt76 | **0x7F** reserved; allocation wraps at 0x7E or 0x7F | [C] |
| Descriptor forms defined | data, MCU command, firmware download | **the same three**, plus the MCU-delivered security-frame form of §5.4 | [C] |
| Per-TC descriptor length | long format | long format (32 B) on all 6 traffic classes | [C] |
| Per-TC lifetime / retry defaults | chip/driver specific | 2000 ms (→ `MAX_TX_TIME = 30`) and 30 retries on all 6 traffic classes | [C] |
| CONNAC3 (MT7925/MT7927) layout | — | **not applicable**: CONNAC3 moves `OWN_MAC` to DW1[30:25], `TID` to DW1[24:21], `HDR_FORMAT` to DW1[15:14], `WLAN_IDX` to DW1[11:0], `HW_AMSDU` to DW3[5], `TX_RATE` to DW6[21:16] and adds timestamp-offset fields. A CONNAC2 descriptor is **not** accepted by CONNAC3 hardware and vice versa. | [C] |

---

## 10. Open questions / needs hardware tracing

1. **DW5[15] `MD`** — defined in both the vendor and upstream headers but never written on this
   part. Semantics (more-data override? MSDU-duplicate marker?) unconfirmed. [U]
2. **DW1[10] `VTA`** — left 0 on all CONNAC2 parts; the vendor tree has a build option
   "set VTA in accordance with fixed rate". The functional effect of setting it on MT7932 is
   unverified. [U]
3. **Software-A-MSDU "last MSDU" flag.** Multi-MSDU submission sets **DW1 bit 20** on the
   first descriptor of a software A-MSDU group. In the CONNAC2 map DW1[22:20] is the TID field, so
   this bit overlaps TID[0]. Either the bit is consumed and stripped by the HIF/PSE front-end before
   TMAC reads the TID, or software A-MSDU is only valid on CONNAC3 (where the flag does not overlap
   TID). Needs tracing before software A-MSDU is enabled. [U]
4. **Sanity rule 5** rejects `HDR_FORMAT = 2` (802.11 normal) on the data path, yet
   `HDR_FORMAT = 2` is what management frames use. Either management frames do not go through
   that check or the check is over-strict. Determine empirically whether 802.11-format
   descriptors are legal on the TX data rings. [U]
5. **Lifetime unit.** A configuration bit switches the `MAX_TX_TIME` unit from 2^6 (64 TU) to 2^4
   (16 TU). Public CONNAC2 documentation only defines 64 TU. Confirm whether MT7932 silicon actually
   honours a 16-TU unit or whether the bit selects a firmware-side behaviour. [U]
6. **`TX_STATUS_FMT` (DW5[8])** — single bit in the descriptor, but the TX-status report carries a
   2-bit format field (0 = MPDU, 2 = PPDU). Confirm the mapping of the descriptor bit to the report
   format. [U]
7. **`HDR_PAD_MODE` (DW1[19]) polarity.** The CONNAC2 accessor macros imply 1 = head, 0 = tail; the
   older CONNAC1 debug decoder prints the opposite. Never exercised on this part (padding length is
   always 0). [U]
8. **MCU-port queue indices.** `Q_IDX` values 0x20–0x23 / 0x3E for the MCU port are taken from
   upstream mt76; on MT7932 the command path leaves `Q_IDX` at 0 and relies on ring-based routing,
   so those encodings are untested here. [U]
9. **Two-page (chained) descriptors.** `DW7[31:30] = 1`, `CTXD`, `CTXD_CNT` are decoded by the debug
   path but never produced. Whether MT7932 silicon implements chained TXDs is unverified. [U]
10. **The host-side MSDU-report deadline.** A per-frame timer supervises token return and its
    expiry raises a transmit-error recovery, but the numeric deadline was not established.
    Whether it is derived from the descriptor's transmit-lifetime field or is an independent
    host constant needs tracing. `[U]`
11. **Whether the firmware always returns the token of a frame it drops** on one of the
    acceptance checks of §7A.5. If it does not, the host's report deadline is the only
    recovery and the pool leaks one entry per dropped frame until then. `[U]`
12. **The per-TX-queue outstanding-token cap.** The two observed values (a multiple of the
    queue's descriptor count, and a fixed 192) and the condition that selects between them
    are host policy; whether any hardware or firmware limit sits underneath them is
    unverified. `[U]`


---

# MT7932 — MAC RX Descriptor Formats

## Scope

This document specifies the receive descriptor that the MT7932 MAC writes in front of every
record it delivers to a host WFDMA RX ring: the fixed base descriptor, the five optional
extension groups, the PHY receive vectors carried in two of those groups, the alternative
record layouts produced by the same rings (TX status report, TX-free/MSDU report, RX report,
ICS log, MCU event), and the buffer, drop and filter behaviour a driver must assume. It
covers only the MAC-level descriptor; the WFDMA ring descriptor (buffer pointer, `SDL0`,
`LS0`, `DDONE`) and the interrupt/ring programming are the subject of the WFDMA section and
are referenced here only where the RX-descriptor semantics depend on them. Bit positions use
the public MediaTek naming from `gen4m` (`CONNAC2X_RX_STATUS_*`, `HW_MAC_RX_STS_GROUP_n`) and
from upstream Linux `mt76` (`MT_RXD*`, `MT_PRXV_*`, `MT_CRXV_*`, `MT_TXS*`, `MT_TX_FREE_*`).

**Headline result: MT7932's receive path is driven as the CONNAC2 "RXD v2" descriptor,
field-for-field identical to MT7921/MT7922 in every position that public `mt76`/`gen4m`
documents.** [C] for the descriptor size, the group presence/ordering walk and every field the
host actually reads — those pin their own bit positions; [L] for the positions of fields the
host never reads, which carry the public definition only. There is **no chip-ID-conditional
variation anywhere in the RX path** [C] (confirmed negative over the analysed RX code).
Deltas are collected in §10; the only substantive one is the reuse of
seven reserved/flag bits of DW0/DW1/DW3 as two 7-bit signed quantities (§3.8).

---

## 1. Descriptor generation and sizes

| Item | Value | Conf. |
|---|---|---|
| RX-descriptor generation | CONNAC2 "RXD v2" (`nic_rxd_v2` / `HW_MAC_CONNAC2X_RX_DESC`) | [C] |
| Base descriptor size | **24 bytes** (6 DW, DW0…DW5), fixed, same for every packet type | [C] |
| Maximum descriptor size incl. all groups | **144 bytes** (24 + 16 + 16 + 8 + 8 + 72) | [C] |
| Optional-group presence field | DW1[15:11], 5 bits, one per group | [C] |
| Group 5 (C-RXV) size for this part | **72 bytes** (18 DW, DW18…DW35) | [C] |
| MCU-event header after the descriptor, RAM firmware running | **12 bytes** | [C] |
| MCU-event header after the descriptor, initialisation phase (ROM/patch, before RAM firmware) | **8 bytes** | [C] |
| Initialisation-event header size constant defined for this part | **8 bytes** | [C] |
| Descriptor alignment in the RX buffer | descriptor starts at buffer offset 0; buffer must be 4-byte aligned | [C] |

Notes:

* The 24-byte base descriptor is identical in size and layout to MT7921/MT7922
  (`MT7961_RX_DESC_LENGTH = 24`, `CONNAC2X_RX_STATUS_FIXED_LEN = 24`). [C]
* During the pre-RAM-firmware phase (patch download, EFUSE access, register access via the
  init-command path, WiFi-function configuration) the *same* 24-byte MAC descriptor is
  produced; only the event payload header shrinks from 12 to 8 bytes. The event header is
  located at descriptor-size (24) + 0 in every case. [C]
  The 8-byte constant carried for this part beside the 24-byte descriptor size is the
  **initialisation-event header size**, not a second descriptor length: the boot-ROM
  response fetch is sized `descriptor (24) + init-event header (8) + 4` = **36 bytes**,
  which is exactly how the firmware-boot section (§6.1 there) and the MCU-protocol section
  (§3.4 there) describe it. [C]
* Group 5 is 18 DW here. Older CONNAC2 silicon documents 16 DW (DW18…DW33); the size is a
  per-chip parameter and MT7932 uses 18 DW, the same as MT7921/MT7922. [C]

---

## 2. Buffer layout of a complete RX record

```
offset 0                                                          byte-count (DW0[15:0])
+--------+---------+---------+---------+---------+----------+-----------------+---------+
| RXD    | GROUP 4 | GROUP 1 | GROUP 2 | GROUP 3 | GROUP 5  | header padding  | frame   |
| 24 B   | 16 B    | 16 B    | 8 B     | 8 B     | 72 B     | 0/2/4/6 B       | body    |
| DW0-5  | DW6-9   | DW10-13 | DW14-15 | DW16-17 | DW18-35  |                 |         |
+--------+---------+---------+---------+---------+----------+-----------------+---------+
   always   bit 14    bit 11    bit 12    bit 13    bit 15     DW2[15:14] * 2
            of DW1    of DW1    of DW1    of DW1    of DW1
```

Groups appear **in this fixed physical order**, independent of the order of the presence
bits, and only the enabled ones consume space. [C] The offset of the frame body is therefore

```
frame_offset = 24
             + 16 * DW1[14]      /* group 4 */
             + 16 * DW1[11]      /* group 1 */
             +  8 * DW1[12]      /* group 2 */
             +  8 * DW1[13]      /* group 3 */
             + 72 * DW1[15]      /* group 5 */
             +  2 * DW2[15:14]   /* header padding */
frame_length = DW0[15:0] - frame_offset
```

This is bit-for-bit the walk performed by `mt7921_mac_fill_rx()` in `mt76`. [C]

---

## 3. Base descriptor, word by word

### 3.1 DW0 — length, checksum results, packet type

| Bits | Name (gen4m / mt76) | Meaning | Conf. |
|---|---|---|---|
| 15:0 | `RX_BYTE_COUNT` / `MT_RXD0_LENGTH` | Total received byte count **including** the descriptor, all present groups and the header padding. Unsigned, 16 bit; this is the only length field. | [C] |
| 22:16 | `ETH_TYPE_OFFSET` / `MT_RXD0_NORMAL_ETH_TYPE_OFS` | Byte offset of the EtherType field within the translated Ethernet header, used by the checksum-offload engine. Valid only for `PKT_TYPE = 2`. | [L] |
| 19:16 | `MT_RXD0_PKT_FLAG` (alias of the above field) | For `PKT_TYPE = 7` only, this is the software-defined sub-type: `0` = MCU event, `1` = 802.11 management/SW frame. See §6.1. | [C] |
| 20:16 | record count | For `PKT_TYPE = 0` (TX status report) only: number of 32-byte TXS records that follow. 5 bits, max 31. | [C] |
| 25:16 | MSDU count | For `PKT_TYPE = 6` (TX-free/MSDU report), report version ≥ 3: number of entries. 10 bits. | [C] |
| 22:16 | MSDU count | For `PKT_TYPE = 6`, report version < 3: 7 bits. | [C] |
| 23 | `IP_CHKSUM` / `MT_RXD0_NORMAL_IP_SUM` | 1 = the IPv4 header checksum was verified correct by hardware. | [L] |
| 24 | `UDP_TCP_CHKSUM` / `MT_RXD0_NORMAL_UDP_TCP_SUM` | 1 = the UDP or TCP checksum was verified correct by hardware. | [L] |
| 26:25 | `DW0_HW_INFO` | Reserved in public MT7921/MT7922 documentation. On MT7932 these two bits carry bits [1:0] of the 7-bit signed field "A" of §3.8. | [C] that the host assembles the field from these bits; [U] for what the field means |
| 31:27 | `PKT_TYPE` / `MT_RXD0_PKT_TYPE` | Producer of this record; see §6. | [C] |

Both checksum bits must be set for the frame's checksums to be considered verified; this
matches `csum_mask = MT_RXD0_NORMAL_IP_SUM | MT_RXD0_NORMAL_UDP_TCP_SUM` in `mt76`. [L]
RX checksum offload is not used on this part, so their polarity is not established
here. [U]

### 3.2 DW1 — station index, group presence, security status, band

| Bits | Name | Meaning | Conf. |
|---|---|---|---|
| 9:0 | `WLAN_INDEX` / `MT_RXD1_NORMAL_WLAN_IDX` | Hardware station (WTBL) index that matched the frame. **10 bits** (0…1023). | [C] |
| 10 | — | reserved | [L] |
| 11 | `GROUP1_VALID` | Group 1 (crypto/PN, 16 B) present | [C] |
| 12 | `GROUP2_VALID` | Group 2 (timestamp/CRC, 8 B) present | [C] |
| 13 | `GROUP3_VALID` | Group 3 (P-RXV, 8 B) present | [C] |
| 14 | `GROUP4_VALID` | Group 4 (header translation, 16 B) present | [C] |
| 15 | `GROUP5_VALID` | Group 5 (C-RXV, 72 B) present | [C] |
| 20:16 | `SEC_MODE` | Cipher used to decrypt (see §3.9). 0 = none/plaintext. | [C] |
| 22:21 | `KEY_ID` | Key index taken from the frame's IV (0…3). | [C] |
| 23 | `CIPHER_MISMATCH` (`CM`) | Frame's protection state did not match the key entry (e.g. plaintext received on a protected link). | [C] |
| 24 | `CIPHER_LENGTH_MISMATCH` (`CLM`) | Cipher header/MIC length inconsistent with the frame length. | [C] |
| 25 | `ICV_ERROR` | ICV / CCMP-MIC / BIP-MIC / WPI-MIC check failed. | [C] |
| 26 | `TKIPMIC_ERROR` | TKIP Michael MIC failure. | [C] |
| 27 | `FCS_ERROR` | PPDU FCS check failed. | [C] |
| 28 | `BAND_IDX` | PHY band that received the frame (0 = band 0). MT7932 is single-band-at-a-time; a non-zero value is unexpected. | [C] |
| 31:29 | `DW1_HW_INFO`; in `mt76`: 29 = `SPP_EN`, 30 = `ADD_OM`, 31 = `SEC_DONE` | Bit 29 is not read on MT7932. Bits 31:30 carry bits [3:2] of the 7-bit signed field "A" of §3.8 — see the deviation note. | [C] that the host assembles the field from these bits; [U] for what the field means |

`ERROR_MASK` (the "hard error" set used to gate header parsing) is
`FCS_ERROR | ICV_ERROR | CIPHER_LENGTH_MISMATCH`, i.e. `0x0A00_0000`; TKIP MIC error is
deliberately excluded because it must be reported to the supplicant rather than silently
dropped. [C]

### 3.3 DW2 — header geometry, TID, frame-class flags

| Bits | Name | Meaning | Conf. |
|---|---|---|---|
| 5:0 | `BSSID` / `MT_RXD2_NORMAL_BSSID` | BSS (own-MAC) index that matched. 6 bits. | [C] |
| 6 | `CO_ANT` | co-antenna indication | [L] |
| 7 | `BF_CQI` | beamforming CQI report attached | [L] |
| 12:8 | `HEADER_LEN` / `MAC_HDR_LEN` | Length of the 802.11 MAC header **in units of 2 bytes** (0…62 B). Valid when header translation is off. | [C] |
| 13 | `HEADER_TRAN` | 1 = hardware replaced the 802.11 header with an 802.3/Ethernet header; the original 802.11 header fields are then in Group 4. | [C] |
| 15:14 | `HEADER_OFFSET` | Padding inserted between the end of the descriptor+groups and the start of the frame, **in units of 2 bytes** (0, 2, 4 or 6). This is the "2-byte offset" control. | [C] |
| 19:16 | `TID` | Traffic identifier / access class of the frame. | [C] |
| 20 | — | reserved | [L] |
| 21 | `MU_BAR` | frame is an MU-BAR | [L] |
| 22 | `SW_BIT` | Software-defined RX class error marker; firmware sets it to flag a frame that failed its own classification. | [C] |
| 23 | `DE_AMSDU_FAIL` (`DAF`) | Hardware A-MSDU de-aggregation failed for this frame. | [C] |
| 24 | `EXCEED_LEN` / `MAX_LEN_ERROR` | Frame exceeded the maximum length the MAC accepts. | [C] |
| 25 | `TRANS_FAIL` / `HDR_TRANS_ERROR` (`LLC_MIS`) | Header translation was requested but the LLC/SNAP header did not match, so the frame was **not** translated. | [C] |
| 26 | `INTF` / `INT_FRAME` | "Interested frame" — matched an interrupt/notify filter. | [L] |
| 27 | `FRAG` | Frame is a fragment (More-Fragments set or fragment number non-zero). | [C] |
| 28 | `NULL` / `NULL_FRAME` | Frame is a Null-Data frame. | [L] |
| 29 | `NDATA` | Frame is **not** a Data frame (management/control). | [C] |
| 30 | `NAMP` / `NON_AMPDU` | Frame is **not** part of an A-MPDU. Cleared ⇒ the MPDU is an A-MPDU subframe. | [C] |
| 31 | `BF_RPT` | beamforming report frame | [L] |

### 3.4 DW3 — RXV sequence, channel, address match

| Bits | Name | Meaning | Conf. |
|---|---|---|---|
| 7:0 | `RXV_SEQ_NO` | PPDU/receive-vector sequence number. All MPDUs of one A-MPDU carry the same value, so it is the aggregation reference. A value of 0 means "no RXV attached". | [C] |
| 15:8 | `CH_FREQ` | Hardware channel number (see §3.6). | [C] |
| 17:16 | `A1_TYPE` / `ADDR_TYPE` | 0 = other, 1 = unicast-to-me (`UC2ME`), 2 = multicast, 3 = broadcast. | [C] |
| 18 | `HTC` | HT-Control field present (Group 4 DW3 valid). | [L] |
| 19 | `TCL` | TSF-compare loss. | [C] |
| 20 | `BBM` / `BEACON_MC` | Beacon received on a multicast/broadcast match. | [L] |
| 21 | `BU` / `BEACON_UC` | Beacon with the buffered-unit bit for this STA. | [L] |
| 24:22 | `DW3_HW_INFO`; in `mt76`: 22 = `AMSDU`, 23 = `MESH`, 24 = `MHCP` | On MT7932 these carry bits [6:4] of the 7-bit signed field "A" of §3.8. | [C] that the host assembles the field from these bits; [U] for what the field means |
| 31:25 | `DW3_HW_INFO`; in `mt76`: 25 = `NO_INFO_WB`, 26 = `DISABLE_RX_HDR_TRANS`, 27 = `POWER_SAVE_STAT`, 28 = `MORE`, 29 = `UNWANT`, 30 = `RX_DROP`, 31 = `VLAN2ETH` | On MT7932 this whole 7-bit span is read as signed field "B" of §3.8. | [C] that the host assembles the field from these bits; [U] for what the field means |

### 3.5 DW4 — payload format, offload, wake-up and packet-filter status

| Bits | Name | Meaning | Conf. |
|---|---|---|---|
| 1:0 | `PF` / `PAYLOAD_FORMAT` | A-MSDU subframe position: 0 = whole MSDU (no A-MSDU), 3 = first subframe, 2 = middle subframe, 1 = last subframe. | [C] |
| 8:2 | — | reserved | [L] |
| 9 | `DP` / `PATTERN_DROP` | Frame matched a drop pattern but was still delivered (report-only mode). | [L] |
| 10 | `CLS` | Frame was classified by the packet-classifier. | [L] |
| 12:11 | `OFLD` | Offload type. Non-zero on a software-defined (`PKT_TYPE = 7`) record means the record must be treated as a data frame rather than an MCU event. | [C] |
| 13 | `MGC` / `MAGIC_PKT` | Frame is a wake-on-LAN magic packet. Delivered with the frame as a wake-reason indication. | [C] |
| 18:14 | `WOL` | Wake-on-WLAN match reason bitmap (5 patterns). | [L] |
| 28:19 | `CLS_BITMAP` | Packet-classifier match bitmap, low 10 bits. | [L] |
| 29 | `PF_MODE` | Packet-filter mode. | [L] |
| 31:30 | `PF_STS` | Packet-filter status. | [L] |

### 3.6 DW5 — classifier extension

| Bits | Name | Meaning | Conf. |
|---|---|---|---|
| 9:0 | `DW5_CLS_BITMAP` | Second packet-classifier match bitmap. | [L] |
| 30:10 | — | reserved | [L] |
| 31 | `MAC` | MAC-generated frame marker. | [L] |

### 3.7 Channel number and band derivation

`CH_FREQ` (DW3[15:8]) is a *hardware* channel index, not a frequency:

| `CH_FREQ` | Band | Real channel number | Conf. |
|---|---|---|---|
| 1 … 14 | 2.4 GHz | identical | [C] |
| 15 … 180 | 5 GHz | identical | [C] |
| 181 … 244 | 6 GHz | `(CH_FREQ − 181) × 4 + 1`, i.e. 1, 5, 9 … 253 | [C] |

The band decision is `CH_FREQ ≤ 14 → 2.4 GHz`, `CH_FREQ > 180 → 6 GHz`, else 5 GHz. [C]
The 6 GHz translation is arithmetically identical to public `gen4m`
(`((v − 181) << 2) + 1`); MT7932 additionally range-checks `CH_FREQ ≤ 244` (above that the
8-bit result would wrap) but drops `gen4m`'s special case that maps `CH_FREQ = 15` to
6 GHz channel 2. [C]

Centre frequency for radiotap is computed as `2407 + 5 × ch` (2.4 GHz), `5000 + 5 × ch`
(5 GHz), `5950 + 5 × ch` (6 GHz). [C]

### 3.8 The two scattered 7-bit signed fields — MT7932 deviation

MT7932 carries **two 7-bit two's-complement values** in bits that public MT7921/MT7922
documentation assigns to individual status flags:

```
A[6:4] = DW3[24:22]      A[3:2] = DW1[31:30]      A[1:0] = DW0[26:25]
B[6:0] = DW3[31:25]
```

Both are converted identically: a raw value ≥ 48 is treated as negative and becomes
`−((128 − value) & 0x7F)`, saturated to a minimum of **−16**; values 0…47 pass through
unchanged. The resulting range is therefore **[−16 … +47]**. [C]

Both values accompany the two per-antenna RCPI bytes in the per-frame receive metadata, one
per receive chain. The range and placement are consistent with a
**per-antenna SNR in dB**, but the identity is not proven. [U] This is the single genuine
functional deviation from MT7921/MT7922 in the RX descriptor; a driver that follows
`mt76` literally would decode DW1[31:30] as `ADD_OM`/`SEC_DONE`, DW3[24:22] as
`AMSDU`/`MESH`/`MHCP` and DW3[31:25] as `NO_INFO_WB … VLAN2ETH`, and those decodes are
mutually exclusive with the interpretation above. Needs hardware tracing.

### 3.9 Cipher (`SEC_MODE`) encoding

Same table as MT7921/MT7922 (`enum mt76_cipher_type`). [L] Values 2 (TKIP) and 4 (AES-CCMP)
are confirmed in use on MT7932. [C]

| Value | Cipher | Value | Cipher |
|---|---|---|---|
| 0 | none | 7 | WEP-128 |
| 1 | WEP-40 | 8 | WAPI (SMS4) |
| 2 | TKIP | 9 | CCMP-CCX |
| 3 | TKIP w/o MIC | 10 | CCMP-256 |
| 4 | AES-CCMP (CCMP-128) | 11 | GCMP-128 |
| 5 | WEP-104 | 12 | GCMP-256 |
| 6 | BIP-CMAC-128 | | |

---

## 4. Optional field groups

| Group | Presence bit | Size | Position | Contents |
|---|---|---|---|---|
| 4 — header translation | DW1[14] | 16 B | DW6…DW9 | see below |
| 1 — crypto / PN | DW1[11] | 16 B | DW10…DW13 | see below |
| 2 — timestamp | DW1[12] | 8 B | DW14…DW15 | see below |
| 3 — P-RXV | DW1[13] | 8 B | DW16…DW17 | §5.1 |
| 5 — C-RXV | DW1[15] | 72 B | DW18…DW35 | §5.2 |

### 4.1 Group 4 — header-translation shadow (16 B) [C]

Present when the MAC replaced the 802.11 header with an 802.3 header (`HEADER_TRAN` = 1);
it preserves the fields that would otherwise be lost.

| Offset | Size | Field |
|---|---|---|
| +0 | 2 | Frame Control (`MT_RXD6_FRAME_CONTROL`) |
| +2 | 6 | Address 2 / transmitter address (`MT_RXD6_TA_LO` + `MT_RXD7_TA_HI`) |
| +8 | 2 | Sequence Control (`MT_RXD8_SEQ_CTRL`): fragment number [3:0], sequence number [15:4] |
| +10 | 2 | QoS Control (`MT_RXD8_QOS_CTL`) |
| +12 | 4 | HT Control (`MT_RXD9_HT_CONTROL`), valid when DW3[18] is set |

The sequence number is `(SeqCtrl >> 4)` and the fragment number `(SeqCtrl & 0xF)`. [C]
When header translation is *off*, the same information is read from the real 802.11 header
at the frame offset, which requires `HEADER_LEN ≥ 24`. [C]

### 4.2 Group 1 — cipher packet number (16 B) [C]

16 raw bytes of the received cipher header. For CCMP/CCMP-256/CCMP-CCX/GCMP/GCMP-256/TKIP
the 48-bit packet number occupies the first 6 bytes **in reverse order**: PN[0] is byte 5,
PN[1] byte 4, … PN[5] byte 0 (identical to `mt7921_mac_fill_rx`). [L] For CCMP ciphers this
group must also be consulted when `FRAG` (DW2[27]) is set, because the CCMP header must be
re-inserted before the fragment is reassembled. [L]

### 4.3 Group 2 — timestamp and CRC (8 B) [C]

| Offset | Size | Field |
|---|---|---|
| +0 | 4 | MAC timestamp of the **start** of the PPDU, in TSF units (µs) |
| +4 | 4 | CRC / FCS value of the received frame |

All MPDUs of one A-MPDU carry the same timestamp; combined with `RXV_SEQ_NO` (DW3[7:0]) and
`NAMP` (DW2[30]) this is how a driver derives an A-MPDU reference number. [C]

### 4.4 Groups 3 and 5

See §5.

---

## 5. PHY receive vector

MT7932 delivers the receive vector in two pieces. Group 3 is the **P-RXV** (2 DW, "primary"),
Group 5 is the **C-RXV** (18 DW, "complete"). MT7932 uses the *V2* P-RXV encoding, in which
the P-RXV alone carries everything needed to reconstruct the rate; the older CONNAC2 encoding,
in which mode/bandwidth/GI live in C-RXV word 0, is not used on this part. [C]

### 5.1 Group 3 — P-RXV, V2 encoding (8 B)

**P-RXV word 0**

| Bits | Name | Meaning | Conf. |
|---|---|---|---|
| 6:0 | `MT_PRXV_TX_RATE` | Rate index. Interpretation depends on `TX_MODE`: CCK/OFDM hardware rate index; HT MCS (0…31); VHT MCS in [3:0] with NSS in `NSTS`; HE MCS in [3:0]. | [C] |
| 4 | `MT_PRXV_TX_DCM` | (alias inside the rate field) HE dual-carrier modulation. | [L] |
| 5 | `MT_PRXV_TX_ER_SU_106T` | (alias inside the rate field) HE extended-range SU on a 106-tone RU. | [L] |
| 9:7 | `MT_PRXV_NSTS` | Number of space-time streams **minus one**. | [C] |
| 10 | `MT_PRXV_TXBF` | Frame was beamformed. | [C] |
| 11 | `MT_PRXV_HT_AD_CODE` | 1 = LDPC, 0 = BCC. | [C] |
| 14:12 | `MT_PRXV_FRAME_MODE` | Bandwidth: 0 = 20 MHz, 1 = 40 MHz, 2 = 80 MHz, 3 = 160/80+80 MHz. Values > 3 are invalid. | [C] |
| 16:15 | `MT_PRXV_HT_SGI` | Guard interval. HT/VHT: 0 = long (0.8 µs), 1 = short (0.4 µs). HE: 0 = 0.8 µs, 1 = 1.6 µs, 2 = 3.2 µs. | [C] |
| 17 | `MT_PRXV_DCM` | Dual-carrier modulation. | [L] |
| 20:18 | `MT_PRXV_NUM_RX` | Number of active receive chains **minus one**. 0 ⇒ single chain. | [C] |
| 21 | `MT_PRXV_MU` | Frame is part of a MU (MU-MIMO / OFDMA) transmission. | [C] |
| 23:22 | `MT_PRXV_HT_STBC` | STBC. Non-zero ⇒ NSS = (NSTS+1)/2. | [C] |
| 27:24 | `MT_PRXV_TX_MODE` | PHY mode, see §5.4. | [C] |
| 31:28 | `MT_PRXV_HE_RU_ALLOC_L` | HE RU allocation, low nibble. | [C] |

**P-RXV word 1** — the RCPI word.

| Bits | Name | Meaning |
|---|---|---|
| 7:0 | `MT_PRXV_RCPI0` | RCPI, chain 0 |
| 15:8 | `MT_PRXV_RCPI1` | RCPI, chain 1 |
| 23:16 | `MT_PRXV_RCPI2` | RCPI, chain 2 |
| 31:24 | `MT_PRXV_RCPI3` | RCPI, chain 3 |
| 3:0 | `MT_PRXV_HE_RU_ALLOC_H` | (alias) HE RU allocation, high nibble |

`0xFF` in any RCPI byte means "measurement not available". [C]

### 5.2 Group 5 — C-RXV (72 B, 18 DW)

C-RXV words are numbered 0…17 from the start of Group 5.

| Word | Bits | Name | Meaning | Conf. |
|---|---|---|---|---|
| 0 | 1:0 | `MT_CRXV_HT_STBC` | STBC | [C] |
| 0 | 3:2 | `NESS` | number of extension spatial streams | [C] |
| 0 | 7:4 | `MT_CRXV_TX_MODE` | PHY mode (legacy encoding, see §5.4) | [C] |
| 0 | 10:8 | `MT_CRXV_FRAME_MODE` | bandwidth | [C] |
| 0 | 11 | `TXOP_PS_NOT_ALLOWED` | | [C] |
| 0 | 14:13 | `MT_CRXV_HT_SHORT_GI` | guard interval | [C] |
| 0 | 18:17 | `MT_CRXV_HE_LTF_SIZE` | HE LTF size, reported +1 | [C] |
| 0 | 20 | `MT_CRXV_HE_LDPC_EXT_SYM` | LDPC extra OFDM symbol | [C] |
| 0 | 23 | `MT_CRXV_HE_PE_DISAMBIG` | HE packet-extension disambiguity | [C] |
| 0 | 30:24 | `MT_CRXV_HE_NUM_USER` | number of users in an HE MU PPDU | [C] |
| 0 | 31 | `MT_CRXV_HE_UPLINK` | UL/DL bit | [C] |
| 1 | 7:0 / 15:8 / 23:16 / 31:24 | `MT_CRXV_HE_RU0…RU3` | HE SIG-B per-RU content channels | [C] |
| 2 | 27:22 | `MT_CRXV_GROUP_ID` | VHT group ID (0 and 0x3F mean SU) | [C] |
| 2 | 30:28 | `NUM_RX` | active receive chains − 1 | [C] |
| 3 | 18:13 | `MT_CRXV_SNR` | 6-bit signal-to-noise ratio | [L] |
| 3 | 31:19 | `MT_CRXV_FOE_LO` | frequency offset estimate, low 13 bits | [L] |
| 4 | 6:0 | `MT_CRXV_FOE_HI` | frequency offset estimate, high bits (`FOE = LO \| HI << 13`) | [L] |
| 5 | 30:20 | `MT_CRXV_HE_MU_AID` / `PART_AID` | VHT partial AID / HE MU AID | [C] |
| 6 | 7:0 / 15:8 / 23:16 / 31:24 | `RCPI0…RCPI3` | per-antenna RCPI (same encoding as P-RXV word 1) | [C] |
| 9 | 11:8 / 15:12 / 19:16 / 23:20 | `MT_CRXV_HE_SR_MASK…SR3_MASK` | HE spatial-reuse fields 1…4 | [C] |
| 12 | 5:0 | `MT_CRXV_HE_BSS_COLOR` | BSS colour | [C] |
| 12 | 12:6 | `MT_CRXV_HE_TXOP_DUR` | TXOP duration | [C] |
| 12 | 13 | `MT_CRXV_HE_BEAM_CHNG` | beam change | [C] |
| 12 | 15 | `MT_CRXV_DCM` | data DCM | [C] |
| 12 | 16 | `MT_CRXV_HE_DOPPLER` | Doppler | [C] |

### 5.3 RCPI, RSSI and noise

**RSSI conversion (exact):**

```
rssi_dBm = (RCPI >> 1) + 0x92   evaluated as a signed 8-bit value
         = RCPI / 2 - 110
valid only for RCPI < 221 (0xDD); RCPI >= 221 is reported as 0 dBm / invalid
RCPI == 0xFF means "measurement not available"
```

This is numerically identical to `mt76`'s `to_rssi(field, v) = (FIELD_GET(field,v) − 220) / 2`.
[C] So RCPI 0 ⇒ −110 dBm, RCPI 40 ⇒ −90 dBm, RCPI 120 ⇒ −50 dBm, in 0.5 dB steps.

**Which RCPI word to use.** If Group 5 is present, use C-RXV word 6. If only Group 3 is
present, use P-RXV word 1. If the information came from an RX report record (§7), use the
report's own C-RXV1. The chain count for the "single chain" shortcut comes from
`NUM_RX`: C-RXV word 2 bits [30:28] when reading Group 5, P-RXV word 0 bits [20:18] when
reading Group 3. [C]

**Combining rules.** The hardware provides four raw per-antenna values; the aggregation
modes a driver typically needs are: per-chain (0…3), arithmetic mean of chains 0 and 1,
maximum of chains 0 and 1, minimum of chains 0 and 1. If `NUM_RX` indicates a single chain,
chain 0 is used unless it reads `0xFF`, in which case chain 1 is used. [C]

**Antenna-gain compensation.** RCPI as delivered is uncompensated. A per-band, per-antenna
offset (one signed byte per (band, antenna) pair, bands 2.4/5/6 GHz × 2 antennas) must be
added to RCPI0/RCPI1 before conversion, skipping bytes that read `0xFF`. Compensation is
only meaningful for bands 1…3. [C]

**Noise floor is not in the RX descriptor.** It is obtained from the firmware statistics
path and applied by the host; the descriptor and receive vectors carry only RCPI, the C-RXV
SNR field and (on MT7932) the two scattered 7-bit fields of §3.8. [C]

### 5.4 PHY mode (`TX_MODE`) encoding

| Value | Mode | Value | Mode |
|---|---|---|---|
| 0 | CCK | 8 | HE SU |
| 1 | OFDM | 9 | HE ER-SU |
| 2 | HT mixed mode | 10 | HE TB (trigger-based) |
| 3 | HT Greenfield | 11 | HE MU |
| 4 | VHT | 13 | EHT SU |
| | | 14 | EHT trigger-based |
| | | 15 | EHT MU |

Modes 13…15 are defined by the family but MT7932 is an 802.11ax part and does not emit
them. [L] Only modes 0, 1, 2, 3, 4 and 8 need to be decoded on MT7932. [C]

### 5.5 Rate reconstruction (exact arithmetic)

Inputs, all from P-RXV word 0:

| Quantity | Field |
|---|---|
| `mode` | bits [27:24] |
| `nsts` | bits [9:7] |
| `bw` | bits [14:12] — 0 = 20, 1 = 40, 2 = 80, 3 = 160; a value > 3 is an error |
| `sgi` | bits [16:15] |
| `stbc` | bits [23:22] |

Spatial-stream count: `nss = 1` when `nsts == 0`; otherwise `nss = (nsts + 1) >> 1` when
`stbc` is non-zero and `nss = nsts + 1` when it is zero.

Rate, by `mode`:

| `mode` | Rate derivation |
|---|---|
| 0, 1 (CCK, OFDM) | `rate_500kbps` = hardware rate table indexed by bits [6:0] |
| 2 (HT mixed) | `phy_rate_100kbps` = HT/VHT rate table `[mcs = bits[2:0]][bw][sgi]`, multiplied by `nss` |
| 3 (HT greenfield) | as mode 2 but `mcs` = bits [6:0] |
| 4 (VHT) | `phy_rate_100kbps` = HT/VHT rate table `[bits[6:0]][bw][sgi]` |
| 8 (HE SU) | `phy_rate_100kbps` = HE rate lookup on (bits [6:0], `bw`, `sgi`, `nss`) |
| any other | unsupported; rate = 0 |

Finally `rate_500kbps = phy_rate_100kbps / 5`.

The HT/VHT rate table is indexed `[mcs 0..12][bw 0..3][gi 0..1]` with 4-byte entries in units
of 100 kbps. [C] For HT mixed mode the per-stream MCS is the low 3 bits of the rate field and
the result must be multiplied by NSS; for VHT/HE the MCS is taken directly and NSS is a
separate lookup dimension. [C]

For a VHT frame, the group ID (C-RXV word 2 bits [27:22]) distinguishes SU (0 or 0x3F) from
MU; for MU frames the reported `NSTS` is the *total* across users, so the per-user NSS is
`nsts + 2` in the legacy path and `nsts + 1` in the V2 path. [C]

### 5.6 Which receive-vector words are delivered by default

* **Group 3 (P-RXV, 2 DW)** is attached to received data and management frames by default and
  is what a driver must rely on for per-frame rate and RSSI. [L]
* **Group 5 (C-RXV, 18 DW)** is only attached when the full receive-vector report is enabled
  — i.e. in monitor/sniffer mode, or when the RX report (§7) / ICS capture paths are turned on.
  A driver must treat Group 5 as optional and fall back to Group 3. This matches `mt76`, whose
  comment states that monitor mode uses the RCPI in Group 5 instead. [L]
* **Group 2 (timestamp)** is required for radiotap and for A-MPDU reference numbering; a
  complete radiotap header requires Groups 2, 3 and 5 to all be present. [C]
* Beyond the descriptor groups, the MAC can be told to emit a **standalone receive-vector
  record** (`PKT_TYPE = 1`) and a **combined RX report** (`PKT_TYPE = 11`, §7). On MT7932
  `PKT_TYPE = 1` is left disabled and `PKT_TYPE = 11` is used instead. [C]

---

## 6. Discriminating the producer of a completed descriptor

### 6.1 The rule

For every buffer completed on **any** host RX ring, read DW0 and branch on
`PKT_TYPE = DW0[31:27]`:

| `PKT_TYPE` | `mt76` name | `gen4m` name | Meaning / action |
|---|---|---|---|
| 0 | `PKT_TYPE_TXS` | `RX_PKT_TYPE_TX_STATUS` | **TX status report** — a 2-DW header followed by *N* 32-byte TXS records (§6.2). |
| 1 | `PKT_TYPE_TXRXV` | `RX_PKT_TYPE_RX_VECTOR` | Standalone receive vector. Not enabled on MT7932; discard. |
| 2 | `PKT_TYPE_NORMAL` | `RX_PKT_TYPE_RX_DATA` | **802.11 / Ethernet frame.** Parse groups per §2. |
| 3 | `PKT_TYPE_RX_DUP_RFB` | `RX_PKT_TYPE_DUP_RFB` | **Duplicated receive buffer** — the hardware's duplicate indication. Discard. |
| 4 | `PKT_TYPE_RX_TMR` | `RX_PKT_TYPE_TM_REPORT` | Timing-measurement report. Discard unless FTM is in use. |
| 5 | `PKT_TYPE_RETRIEVE` | — | Reserved. Discard. |
| 6 | `PKT_TYPE_TXRX_NOTIFY` | `RX_PKT_TYPE_MSDU_REPORT` | **TX-free / MSDU report** — returns TX MSDU tokens (§6.3). |
| 7 | `PKT_TYPE_RX_EVENT` | `RX_PKT_TYPE_SW_DEFINED` | Software-defined; sub-classify, see below. |
| 8 | `PKT_TYPE_NORMAL_MCU` | — | Frame injected by the MCU. `mt76` handles it as `PKT_TYPE = 2`; on MT7932 it can be discarded, because an MCU-injected frame is already covered by `PKT_TYPE = 7` sub-type 1. |
| 11 | — | `RX_PKT_TYPE_RX_REPORT` | **RX report** (§7). |
| 12 | `PKT_TYPE_RX_FW_MONITOR` | `RX_PKT_TYPE_ICS` | MAC ICS (in-chip sniffer) log record. Only produced when ICS capture is enabled. |
| 13 | — | `RX_PKT_TYPE_PHY_ICS` | PHY ICS log record. Only when PHY ICS capture is enabled. |

**Sub-classification of `PKT_TYPE = 7`** (this is the "is it an MCU event or a management
frame?" decision):

Form `sw_type = DW0[31:16] & 0x380F`, then:

| `sw_type` | Meaning |
|---|---|
| `0x3800` | MCU event / command response |
| `0x3801` | 802.11 management or software-generated frame |
| anything else | malformed; discard |
`0x380F` selects `PKT_TYPE` bits [2:0] together with DW0[19:16]; because `PKT_TYPE` is already
known to be 7, the test degenerates to `DW0[19:16] == 0` ⇒ event, `== 1` ⇒ frame. [C]
This is exactly `MT_RXD0_SW_PKT_TYPE_MAP`/`_FRAME` in `mt76`, and exactly `mt7921`'s
`type == PKT_TYPE_RX_EVENT && flag == 1 → PKT_TYPE_NORMAL_MCU`. [C]

Two overrides apply to a `PKT_TYPE = 7` record before it may be treated as an event:
if `OFLD` (DW4[12:11]) is non-zero **or** `HEADER_TRAN` (DW2[13]) is set, the record is a
data frame regardless of the sub-type. [C]

**Ring dependence.** The `PKT_TYPE` test is authoritative on its own; the classification does
not depend on which ring the buffer arrived on. In practice MT7932 is programmed with a data
ring (types 0, 2, 6, 11, 12, 13), a dedicated MCU-event ring (type 7 sub-type 0) and a
separate management/MMPDU ring (type 7 sub-type 1) [L]. Up to 10 host RX ring indices are
supported [C]. A record of the "wrong" type on a ring is
still parsed correctly; only the buffer pool it is returned to differs. [C]

**MCU event body.** For an MCU event the event structure begins at buffer offset
`descriptor size` = 24, with no groups. The event header is:

| Offset | Size | Field |
|---|---|---|
| +0 | 2 | Event length in bytes, including this header. Must be ≥ 12 and ≤ 2352 − 24. |
| +2 | 2 | reserved / packet type |
| +4 | 1 | Event ID |
| +5 | 1 | Sequence number (echoes the command sequence number) |
| +6 | 6 | remainder of the 12-byte header |

During the initialisation phase the header is 8 bytes instead of 12; the Event ID and
sequence number remain at +4 and +5. [C]

### 6.2 TX status report record (`PKT_TYPE = 0`) — full layout

Container:

```
byte 0 : DW0  -- [15:0] total byte count, [20:16] number of TXS records, [31:27] = 0
byte 4 : DW1  -- reserved
byte 8 : first 32-byte TXS record
byte 40: second TXS record ...
```
Records are 32 bytes (8 DW) and start at byte offset 8. [C] `mt76` iterates the same way
(`for (rxd += 2; rxd + 8 <= end; rxd += 8)`), using the buffer end instead of the count
field; MT7932's count field in DW0[20:16] is authoritative and caps a report at 31 records. [C]

**TXS record, format 0 (MPDU-based) — the format MT7932 produces:**

| DW | Bits | Name | Meaning | Conf. |
|---|---|---|---|---|
| 0 | 13:0 | `MT_TXS0_TX_RATE` | Final transmit rate actually used, in the same encoding as the TXD rate field (mode/NSS/MCS/GI packed). | [L] |
| 0 | 14 | `MT_TXS0_TX_STATUS_MCU` | This report was also sent to the MCU. | [L] |
| 0 | 15 | `MT_TXS0_TX_STATUS_HOST` | This report was sent to the host. | [L] |
| 0 | 16 | `MT_TXS0_ACK_TIMEOUT` | No ACK/BA received. | [C] |
| 0 | 17 | `MT_TXS0_RTS_TIMEOUT` | No CTS received for RTS. | [C] |
| 0 | 18 | `MT_TXS0_QUEUE_TIMEOUT` | Frame aged out in the queue. | [C] |
| 0 | 18:16 | `MT_TXS0_ACK_ERROR_MASK` | Any of the three above. | [C] |
| 0 | 19 | `MT_TXS0_BIP_ERROR` | Management-frame protection (BIP) error. | [C] |
| 0 | 19:16 | — | **Success indication: all four bits zero ⇒ the MPDU was acknowledged.** | [C] |
| 0 | 20 | `MT_TXS0_TXOP_TIMEOUT` | TXOP expired. | [L] |
| 0 | 21 | `MT_TXS0_PS_FLAG` | Peer entered power save. | [L] |
| 0 | 22 | `MT_TXS0_BA_ERROR` | Block-Ack error. | [L] |
| 0 | 24:23 | `MT_TXS0_TXS_FORMAT` | 0 = MPDU-based (this layout), 2 = PPDU-based. | [L] |
| 0 | 25 | `MT_TXS0_AMPDU` | The MPDU was sent inside an A-MPDU. | [C] |
| 0 | 28:26 | `MT_TXS0_TID` | TID of the reported MPDU. | [C] |
| 0 | 30:29 | `MT_TXS0_BW` | Bandwidth used (0=20, 1=40, 2=80, 3=160). | [L] |
| 0 | 31 | `MT_TXS0_FIXED_RATE` | The frame was sent at a host-fixed rate. | [L] |
| 1 | 7:0 | `MT_TXS1_TX_POWER_DBM` | Transmit power actually used, dBm. | [L] |
| 1 | 15:8 | `MT_TXS1_RXV_SEQNO` | Receive-vector sequence number of the acknowledging frame. | [L] |
| 1 | 19:16 | `MT_TXS1_RESP_RATE` | Rate of the received ACK/BA. | [L] |
| 1 | 31:20 | `MT_TXS1_SEQNO` | **802.11 sequence number** of the reported MPDU (12 bits). | [C] |
| 2 | 15:0 | `MT_TXS2_TX_DELAY` | Queue-to-air delay of the MPDU. | [L] |
| 2 | 25:16 | `MT_TXS2_WCID` | Station (WTBL) index. MT7932 matches on the low 8 bits. | [C] |
| 2 | 26 | `MT_TXS2_SHARED_ANTENNA` | Transmitted while the antenna was shared with Bluetooth. | [L] |
| 2 | 29:27 | `MT_TXS2_LAST_TX_RATE` | Index of the rate entry finally used (0…7 of the rate table). | [L] |
| 2 | 31:30 | `MT_TXS2_BF_STATUS` | Beamforming status. | [L] |
| 3 | 2:0 | `LAST_TX_RATE` | (alternate placement) | [L] |
| 3 | 3 | `SHARED_ANTENNA` | (alternate placement) | [L] |
| 3 | 5:4 | `SRC` | Report source. | [L] |
| 3 | 6 | `FIXED_RATE` | | [L] |
| 3 | 7 | `RATE_STBC` | STBC used. | [L] |
| 3 | 8 | — | Read by MT7932 as a single status flag; not named in public sources. | [U] |
| 3 | 23:0 | `MT_TXS3_ANT_ID` | Antenna identifier bitmap (connac2 naming). | [L] |
| 3 | 31:24 | `MT_TXS3_PID` | **Packet ID (PID) — an echo of the 8-bit `PID` field the host wrote in TXD DW5[7:0].** This closes the *MAC-level* transmit-status loop (per-MPDU delivery result). It is **not** the DMA-level MSDU token: that is the 15-bit token carried in the TXD append block and returned by the TX-free / MSDU report of §6.3. The two identifiers are independent and different widths — see the TX-descriptor section §2 (DW5) and §3.1. | [C] |
| 4 | 31:0 | `MT_TXS4_TIMESTAMP` | MAC timestamp of the transmission, TSF units (µs). | [L] |
| 5 | 24:0 | `MT_TXS5_F0_FRONT_TIME` | Air/front time of the MPDU. | [L] |
| 5 | 29:25 | `MT_TXS5_F0_TX_COUNT` | **Number of transmit attempts (retry count + 1), 5 bits.** | [C] |
| 5 | 30 | `MT_TXS5_F0_QOS` | The MPDU was a QoS frame. | [L] |
| 5 | 31 | `MT_TXS5_F0_FINAL_MPDU` | Last MPDU of the reported burst. | [L] |
| 6 | 7:0 / 15:8 / 23:16 / 31:24 | `MT_TXS6_F0_NOISE_0…3` | Per-antenna noise measured during the transmission. | [L] |
| 7 | 7:0 / 15:8 / 23:16 / 31:24 | `MT_TXS7_F0_RCPI_0…3` | Per-antenna RCPI of the received ACK/BA. | [L] |

MT7932 matches a report to a pending frame with the triple
`(WCID = TXS2[25:16] low 8 bits, PID = TXS3[31:24], TID = TXS0[28:26])`. [C]
Failure cause is decoded in priority order: bit 16 → "ACK timeout", else bit 17 → "RTS
timeout", else bit 18 → "queue timeout", else bit 19 → "BIP error", else success. [C]

In **PPDU-based format** (`TXS_FORMAT = 2`) DW5…DW7 change meaning to
`MPDU_TX_CNT`/`MPDU_TX_BYTE`, `MPDU_FAIL_CNT`/`MPDU_FAIL_BYTES` and
`MPDU_RETRY_CNT`/`MPDU_RETRY_BYTE`. MT7932 does not produce this format. [L]

### 6.3 TX-free / MSDU report record (`PKT_TYPE = 6`) — token return

This is the record that returns TX MSDU tokens (buffer descriptors) to the host. Container:

```
DW0: [15:0]  total byte count of the report
     [25:16] MSDU count (report version >= 3)   /  [22:16] MSDU count (version < 3)
     [31:27] = 6
DW1: [18:16] report version
entries start at byte offset 8; entry count = (byte_count - 8) / 4 bounds the walk
```
[C]

| Version (DW1[18:16]) | Entry size | Token field | Extra |
|---|---|---|---|
| 0 | 2 B | whole u16 | — |
| 1 | 4 B | bits [15:0] | — |
| 2 | 4 B | bits [14:0] (`0x7FFF`) | — |
| 3 | 4 B | bits [30:16] (`MT_TX_FREE_MSDU_ID`) | bits [12:0] = `MT_TX_FREE_COUNT` (bytes freed); **bit 31 = `MT_TX_FREE_PAIR`: when set the entry is a WCID/pair record, not a token — skip it** |

[C] The `MT_TX_FREE_PAIR` skip rule is byte-identical to `mt76_connac2_mac_tx_free()`. [C]
The MSDU-ID field is 15 bits wide, so the hardware token space is 0…32767. A pool of 6000
tokens is used on this part — a host sizing choice, not a hardware limit. [C]

---

### 6.3a Parsing rules the host must enforce on a TX-free report

The report is the only path by which transmit buffers come back, and it is parsed from a
DMA'd buffer whose contents the host did not write. Every bound in it must be checked; a
violation must be treated as a transport fault rather than trusted. `[C]`

| Bound | Value | On violation |
|---|---|---|
| MSDU count field | ≤ **0x3FD** (1021) for report version 3; the field itself is 10 bits. Versions < 3 use a 7-bit count field | record rejected whole |
| Entry walk limit | `(DW0[15:0] − 8) / 4` entries for the 4-byte forms, `(DW0[15:0] − 5)` for the 2-byte form; the walk must stop at whichever of the count field or this byte-derived limit comes first | walk aborted |
| Entry index | must stay below the MSDU count *and* below 0x3FE | walk aborted |
| Token id | must be **< the host's token-pool size** (6000 on this part) | entry discarded, HIF debug dump, transport fault raised |
| Token state | the token must currently be marked in use | entry skipped |

`[C]` for all rows.

Two further rules `[C]`:

* **The pair/WCID entries interleave with token entries and must be skipped before the
  count is consumed** — in report version 3 the walk advances past every entry whose
  `MT_TX_FREE_PAIR` bit is set *without* decrementing the token counter, so the entry index
  and the token counter advance at different rates.
* **The whole report must be consumed in one pass, in order.** The freed buffers are chained
  into a single list and returned to the pool as a batch after the walk; a parser that
  processes entries out of order or abandons the walk part-way leaks the remaining tokens.

### 6.3b Receive-path obligations that keep the transmit path alive

Because token return travels on the receive path, the receive path is a hard dependency of
transmit. Specifically `[C]`:

* The host must keep the TX-free-report ring refilled with buffers at all times. If that
  ring starves, transmit stops within one token-pool's worth of frames and the only symptom
  is the host-side token-return timeout.
* The report ring must be drained **before** the host declares a transmit fault: a burst of
  reports pending in the ring looks identical to a stalled firmware if the ring is not
  serviced.
* During the error-recovery "stop DMA" phase the reports in flight are lost. That is why the
  recovery sequence resets the token pool wholesale rather than waiting for outstanding
  tokens to drain.
* Receive buffers are a shared pool with the reorder queues. Reorder queues must be flushed
  when the free-buffer count drops below a threshold, so that
  reordering cannot starve the ring refill. A driver that lets its reorder queues grow
  without bound will stall its own receive ring and, through it, its transmit path. `[C]`

---

## 7. RX report record (`PKT_TYPE = 11`)

A combined per-PPDU report that carries the full receive vectors without attaching them to a
frame. Layout:

```
DW0    : [15:0] byte count, [25] "RXV blocks present", [31:27] = 11
DW10   : [29:22] WLAN (WTBL) index
DW24   : block presence bits:
             bit 16 = C-RXV1 block present   (18 DW = 72 B)
             bit 17 = P-RXV1 block present   ( 2 DW =  8 B)
             bit 18 = P-RXV2 block present   ( 4 DW = 16 B)
             bit 19 = C-RXV2 block present   (26 DW = 104 B)
```
Expected total length:
```
hdr_dw  = 24 + 2 * DW0[25]
hdr_dw += 18 * DW24[16]
len     = (hdr_dw + 2*DW24[17] + 4*DW24[18]) * 4  +  104 * DW24[19]
```
and must equal `DW0[15:0]`; otherwise the record is malformed. [C] Blocks follow the header
in the order C-RXV1, P-RXV1, P-RXV2, C-RXV2, each present only if its bit is set. [C]

The block sizes differ from the values documented in public `gen4m`
(`RX_RPT_BLK_CRXV2_LEN` = 20 DW there, **26 DW = 104 B here**); the other three match
(`CRXV1` 18 DW, `PRXV1` 2 DW, `PRXV2` 4 DW). [C]

MT7932 uses this record as its preferred source of rate and RSSI information for statistics:
P-RXV1 word 0 supplies the rate (V2 encoding, §5.1) and C-RXV1 word 6 supplies the RCPI. [C]

---

## 8. Buffer requirements

| Item | Value | Conf. |
|---|---|---|
| RX buffer size | **2352 bytes** (`0x930`) per RX ring entry, for every ring (data, event, management) | [C] |
| Derivation | `28 + 2312 + 12` = maximum 802.11 MPDU (2312) + MAC descriptor headroom (28) + HIF header (12). This is the public gen4m host constant `CFG_RX_MAX_PKT_SIZE`, i.e. a **host/driver sizing choice, not a hardware or firmware requirement** — the engine writes whatever the descriptor's `SDL0` capacity allows. Upstream `mt76` uses 2048 for the same silicon. | [C] |
| Buffer alignment | **4 bytes minimum** — a buffer whose physical address is not 4-byte aligned is rejected | [C] |
| Descriptor position | descriptor at buffer offset 0; no leading pad | [C] |
| Frame-start offset control | `HEADER_OFFSET` = DW2[15:14] × 2 bytes (0/2/4/6). The classic "2-byte offset" that aligns the IP header on a 4-byte boundary after an 14-byte Ethernet header is `HEADER_OFFSET = 1`. | [C] |
| Extra 2-byte shift for A-MSDU | When header translation is **off** and the payload format indicates an A-MSDU subframe, the 802.11 header must additionally be moved forward 2 bytes to keep the payload aligned (`mt76` performs the same `memmove(data+2, data, hdrlen); skb_pull(skb, 2)`). | [L] |
| Maximum record length | `DW0[15:0]` is 16 bits, but the WFDMA segment-length field (`SDL0`) is 14 bits (max 16383) and the ring buffer size caps the practical maximum at 2352 bytes | [C] |
| Maximum frame body | 2352 − 24 (descriptor) − groups − padding; with all groups present, 2352 − 144 − 6 = **2202 bytes** | [C] |
| Scatter/gather | **Implemented on the host receive path** `[C]`; that the DMA engine will actually produce a split record on this part is `[L]` and untested (open question 9). A completed WFDMA RX descriptor with `LS0` (last-segment) **clear** means the record continues in the following descriptor; the host walks forward, summing each descriptor's `SDL0` (each fragment contributing at most the 2352-byte buffer size), until it reaches a descriptor with `LS0` set, and reassembles the fragments into one record. Reassembly is gated on a host-side option: when that option is off, a descriptor with `LS0` clear is dropped instead. A reassembled record longer than **`0xF80` = 3968 bytes** is abandoned. See the WFDMA section §7.3, which describes the same walk. | [C] |
| Ring count | up to 10 host RX ring indices are handled | [C] |
| Buffer pools | three independent free-buffer pools (data frames, MCU events, management frames) keep an event storm from starving the data path — a host choice, not a hardware requirement | [C] |

---

## 9. Drop / filter behaviour

### 9.1 Delivered with an error flag (host must decide)

These conditions do **not** stop delivery; the frame arrives with the corresponding
descriptor bit set:

| Condition | Bit | Typical host action |
|---|---|---|
| FCS error | DW1[27] | drop; count; monitor mode may keep it |
| ICV / CCMP-MIC / BIP-MIC failure | DW1[25] | drop; monitor-only |
| TKIP Michael MIC failure | DW1[26] | drop **and** report a MIC failure to the supplicant |
| Cipher mismatch (plaintext on a protected link) | DW1[23] | drop, except EAPOL (EtherType 0x888E), WAPI and, on NAN links, broadcast/multicast |
| Cipher length mismatch | DW1[24] | drop |
| A-MSDU de-aggregation failure | DW2[23] | drop |
| Header-translation (LLC/SNAP) mismatch | DW2[25] | drop, **unless** the EtherType at header offset 12 is 0x8100 (VLAN) and the header is ≥ 14 bytes, or the frame is the first subframe of an A-MSDU |
| Maximum-length exceeded | DW2[24] | drop |
| Fragmented broadcast/multicast | DW2[27] + DW3[17:16] = 2 or 3 | drop (fragment-aggregation attack mitigation) |
| Software class error | DW2[22] | reported only; host policy decides |
| Duplicate | delivered as `PKT_TYPE = 3` | discard the record |

The complete drop predicate implemented on MT7932 is:

```
drop =  FCS_ERROR
     |  TKIP_MIC_ERROR
     |  ICV_ERROR
     |  DE_AMSDU_FAIL
     | (HDR_TRANS_ERROR & !(VLAN EtherType) & !(first A-MSDU subframe))
     | (CIPHER_MISMATCH & data-frame & !(EAPOL | WAPI | NAN-BMC))
     | ((BC | MC) & FRAG)
```
[C]

The frame classification that survives the error test is:
`NAMP == 0` ⇒ eligible for the reorder buffer; else `NDATA == 1` ⇒ not a data frame;
else `FRAG == 1` ⇒ a fragment. [C]

### 9.2 Dropped by hardware (never delivered)

Everything the MAC's address/BSSID/type filters reject is dropped in silicon and never
produces a descriptor [L]; the RX filter configuration (promiscuous, BSSID match, control-frame
pass, probe-request pass, etc.) is programmed by MCU command and is outside the descriptor [C].
The descriptor-visible consequences a driver must plan for are:

* Because FCS-error, ICV-error and de-AMSDU-failure frames **are** delivered, a driver that
  does not want them must either drop them in software or disable the corresponding
  "pass error frames" RX-filter bits. [L]
* Because duplicates are delivered as a separate packet type (3) rather than as a flag,
  hardware duplicate suppression cannot be relied on: the driver must do its own duplicate
  detection using the Retry bit plus a per-TID sequence-control cache (16 entries + one
  non-QoS entry). [C]
* Wake-on-WLAN matches (`MGC` DW4[13], `WOL` DW4[18:14]) are reported on the frame that
  caused the wake, so the RX filter must be left permissive enough for those frames to
  arrive. [C]
* Packet-classifier drops are reported rather than silent when `PATTERN_DROP` (DW4[9]) is
  set, i.e. the classifier can be run in "mark" mode. [L]

---

## 10. Deltas versus public MT7921 / MT7922

**Identical (no delta):** descriptor generation and 24-byte size; every bit position in
DW0…DW5 that `mt76`/`gen4m` documents; the five group presence bits and their physical
ordering; group sizes 16/16/8/8/72; Group 4 and Group 1 contents; Group 2 timestamp
semantics; the V2 P-RXV bit layout; the C-RXV bit layout; RCPI→dBm arithmetic
`(RCPI−220)/2`; the packet-type enumeration; the `0x380F`/`0x3800`/`0x3801` software-packet
sub-type rule; the 32-byte TXS record layout and its start at byte offset 8; the TX-free
report versions 0…3 and the `MT_TX_FREE_PAIR` skip rule; RX buffer size 2352; the
`LS0` last-segment semantics. [C]

**Deltas found:**

1. **Two 7-bit signed fields in otherwise-flag-bearing bits** (§3.8): DW0[26:25] + DW1[31:30]
   + DW3[24:22] form one value, DW3[31:25] the other, each sign-converted with a −16
   saturation and stored per receive chain next to RCPI0/RCPI1. On MT7921/MT7922 those bits
   are `ADD_OM`, `SEC_DONE`, `AMSDU`, `MESH`, `MHCP`, `NO_INFO_WB`, `DISABLE_RX_HDR_TRANS`,
   `POWER_SAVE_STAT`, `MORE`, `UNWANT`, `RX_DROP`, `VLAN2ETH`. **This is the one place where
   copying `mt76` verbatim would give a different answer.** [C] / semantics [U]
2. **`BSSID` (DW2[5:0]) is available on the v2 descriptor.** Public `gen4m` exposes it only
   from the v3 (CONNAC3) descriptor onwards; on MT7932 it is usable from the v2 descriptor.
   The bit position is unchanged. [C]
3. **RX-report C-RXV2 block is 26 DW (104 B), not 20 DW (80 B)** as `RX_RPT_BLK_CRXV2_LEN`
   states in public `gen4m`. [C]
4. **6 GHz channel translation drops the `CH_FREQ == 15 → channel 2` special case** present
   in public `gen4m`, and adds an upper range check at `CH_FREQ ≤ 244`. The main formula
   `(CH_FREQ − 181) × 4 + 1` is unchanged. [C]
5. **Rate/RSSI source selection is fixed to the "P-RXV/V2" variant** for this part (a chip
   capability query returns "use P-RXV word 0"), whereas the family also supports reading
   mode/bandwidth/GI from C-RXV word 0 bits [14:4]. Both encodings exist; MT7932 uses the
   former, like MT7922. [C]
6. **Per-frame receive metadata carries RCPI for two chains and the two fields of item 1**,
   where public `gen4m` carries only RCPI0. Also, the HE RU allocation is assembled as the full
   8 bits `P-RXV0[31:28] | P-RXV1[3:0] << 4`, whereas public `gen4m` uses
   `(RU_ALLOC1 >> 1) | (RU_ALLOC2 << 3)` (7 bits). [C]
7. **The maximum-length error bit (DW2[24]) is usable as a per-frame monitor-mode flag**;
   public `gen4m` only uses it to drop. Cosmetic. [C]
8. *(withdrawn — the 8-byte constant is the initialisation-event **header** size, which
   public `gen4m` also sets to 8 for CONNAC2 parts; there is no delta here. See §1.)*

No other RX-path behaviour is conditional on the chip ID or on a chip revision. [C]

---

## Open questions / needs hardware tracing

1. **Identity of the two 7-bit signed fields of §3.8.** Range [−16, +47], one per chain,
   stored beside RCPI. Per-antenna SNR in dB is the best hypothesis. Trace: receive a frame
   at a known SNR on a shielded link and compare against the C-RXV `SNR` field
   (word 3, bits [18:13]).
2. **Whether MT7921/MT7922's DW1[31:30], DW3[24:22] and DW3[31:25] flag meanings still hold
   on MT7932.** If they do, items 1 and this are in conflict and one of the two decodes is
   unused. Trace: receive a VLAN-tagged, header-translated frame and check whether
   DW3[31] behaves as `VLAN2ETH`.
3. **Checksum-offload polarity.** DW0[23] and DW0[24] are not used on this part.
   Trace: send frames with deliberately corrupted IPv4/TCP checksums and observe the bits.
4. **Whether MT7932 can emit PPDU-format TXS records** (`TXS_FORMAT = 2`) and, if so, whether
   DW5…DW7 then follow the connac2 `MPDU_TX_CNT`/`MPDU_FAIL_CNT`/`MPDU_RETRY_CNT` layout.
5. **TXS3 bit 8.** Read as a single boolean; not named in any public source.
6. **Exact conditions under which the MAC attaches Group 5.** Assumed to be monitor/sniffer
   mode and the RX-report/ICS capture paths; the enabling MCU command was not identified in
   this section's scope.
7. *(resolved — see §1: the 8-byte constant is the initialisation-event header size, and
   the initialisation phase uses the same 24-byte MAC descriptor as every other record.)*
8. **RX-report DW0 bit 25 and DW24 bits [19:16]** are inferred from the length-consistency
   relation; the field names are not public. Confirm against a captured report.
9. **How often the MAC actually splits a record across RX ring entries.** With a
   2352-byte buffer and a 2352-byte maximum record, `LS0` should always be set in practice;
   the reassembly path exists but was not observed running. Confirm whether the MAC ever splits a frame across two RX ring entries (e.g. with a smaller ring
   buffer size) is untested.


---

# MT7932 — EEPROM / NVRAM, Calibration and RF Provisioning

## Scope

This document specifies how the MediaTek MT7932 combo Wi‑Fi/BT part obtains its RF
provisioning data: the on-die efuse, the host-supplied EEPROM binary image, the per-rate
power (PPR) overlay, the one-time-calibration cache, the regulatory / TX-power-limit
tables, and the MCU commands that carry each of these into firmware. Everything is stated
as a property of the silicon + firmware interface, in vendor-neutral / public-`mt76`
terminology. Confidence markers: `[C]` confirmed by direct observation of shipped data
files or of the command encodings, `[L]` likely (strongly implied, consistent with public
MT7921/MT7922 silicon), `[U]` unverified (needs hardware tracing). The MT7932 is a
CONNAC2 (mt792x-class) part and the great majority of this interface is bit-identical to
the public MT7921/MT7922 definitions; deltas are called out explicitly.

Shipped provisioning payloads referenced throughout, under their generic MediaTek names.
(A platform may add a vendor-specific prefix to the on-disk file names; the names the
firmware-request path uses are the un-prefixed ones given here.)

| File | Size | Role |
|---|---|---|
| `EEPROM_MT7932_1.bin` | 2560 B | full EEPROM image (host-supplied NVRAM substitute) |
| `PPR_MT7932.bin` | 412 B | per-rate power overlay, `BLOB` container |
| `WCal_MT7932.bin` | (not shipped as a file; delivered as an opaque platform-supplied blob) | one-time-calibration result blob |
| `TxPwrLimit_MT79x1.dat` | 20 104 839 B | 2.4/5 GHz per-country per-rate TX power limits |
| `TxPwrLimit6G_MT79x1.dat` | 14 728 357 B | 6 GHz per-country per-power-mode limits |
| `TxPwrLimit_SAR.dat` | 153 930 B | per-country per-sub-band SAR caps |
| `TxPwrLimit_AntGain.dat` | 638 B | 6 GHz antenna-gain groups |
| `TxPwrLimit_SDB.dat` | 1 119 B | shared-antenna dual-band backoff |
| `TxPwrLimit_2gCommonPath.dat` | 48 957 B | 2.4 GHz common-path backoff |
| `db.dat` | 133 959 B | regulatory channel matrix (2.4/5/6 GHz) |
| `wifi.cfg`, `MFG_wifi.cfg` | 1 696 / 1 179 B | provisioning selectors |

---

## 1. Where calibration data comes from

### 1.1 The three sources

| Source | Description |
|---|---|
| On-die efuse (OTP) | Byte-addressable one-time-programmable array inside the chip. Readable/writable through MCU commands in 16-byte blocks `[C]`. The three fields the host actually reads (module identity at `0x07A`, 2.4 GHz TX target power at `0x164`, 64-byte OTP region at `0x200`) are `[C]`; that a fully burned module additionally holds the whole RF calibration record is `[L]`, inferred from the board-type classification of §1.2 |
| Host EEPROM binary | A 2560-byte image supplied by the host and pushed into the MCU's EEPROM shadow buffer. Replaces the efuse content wholesale. `[C]` |
| Host overlay binaries | Two further host binaries pushed through the *same* command but with distinct source-mode codes: a per-rate-power overlay (`PPR`) and a one-time-calibration result blob (`WCal`). These *patch* selected regions of whatever the firmware already holds rather than replacing it. `[C]` |

There is **no external serial EEPROM/flash on this design**. The 2560-byte file is a
software substitute for one; the part itself only has the efuse. `[L]`

### 1.2 Board-type auto-detection

Before any provisioning decision the host reads three efuse bytes and classifies the
board. This is the mechanism that decides whether the efuse alone is trustworthy: `[C]`

| Efuse byte address | Field | Meaning |
|---|---|---|
| `0x07A` | module enable / module version | bit7 = module-enable, bits[3:0] = module/board revision |
| `0x164` | WF0 2.4 GHz TX target power | 0.5 dBm units; non-zero once RF cal has been burned |
| `0x200` | first byte of the OTP region | first byte of a 64-byte host-readable OTP region |

Classification: `[C]`

| Condition | Board type | Provisioning behaviour |
|---|---|---|
| module version `[3:0] != 0` **and** 2.4 GHz TX target power `!= 0` | 1 — development board, efuse burned | use efuse only |
| module version `[3:0] == 0` | 2 — development board, efuse blank | upload the EEPROM binary image |
| module version `[3:0] != 0` **and** 2.4 GHz TX target power `== 0` | 3 — production ("FF") module | efuse carries identity only; RF numbers come from the PPR overlay and the one-time-calibration cache |

The 64-byte OTP region at efuse `0x200` is exposed to the host only when board type == 3
or when the byte at `0x200` reads `0x15`; otherwise the reported OTP length is 0. `[C]`

### 1.3 "Efuse buffer mode" selector

Which provisioning source is used is a host selection (public gen4m carries it as the
`EfuseBufferModeCal` setting; the value for a production board is **3**, for a
manufacturing board **0**). It maps onto the `ucSourceMode` byte of
`EXT_CMD_ID_EFUSE_BUFFER_MODE` as follows: `[C]`

| Selection | `ucSourceMode` sent | Behaviour |
|---|---|---|
| 0 | 0 (`EE_MODE_EFUSE`) | efuse only. The command is still issued, paged, with `u2Count = 0` and an empty payload, to tell the firmware to (re)load its shadow from efuse. |
| 1 | 1 (`EE_MODE_BUFFER`) | upload `EEPROM_MT<chipid>_<n>.bin` (2560 B) in 1024-byte pages, wholly replacing the efuse-derived shadow |
| 3 | 2 (**MT7932 extension**) | efuse provides identity; upload `PPR_MT<chipid>.bin` as a per-rate-power overlay; one-time-calibration blobs are uploaded later with `ucSourceMode = 3` |
| 4 | — | no EEPROM/PPR upload at all; only the one-time-calibration flow runs. Leaves the firmware entirely on efuse content while still enabling the one-time-cal machinery. `[L]` |

**Delta vs public parts.** Upstream `mt76` and public `gen4m` define only
`EE_MODE_EFUSE = 0` and `EE_MODE_BUFFER = 1`. MT7932 firmware additionally accepts
**`ucSourceMode = 2` (per-rate-power overlay)** and **`ucSourceMode = 3`
(one-time-calibration result blob)** on the same `EXT_CMD_ID_EFUSE_BUFFER_MODE` (ext CID
`0x21`). Both extension modes are single-shot (payload must be ≤ 1024 B, no paging).
`[C]`

### 1.4 If no host image is supplied

Nothing breaks: the firmware falls back to the efuse shadow it loaded at boot. On a
production module (board type 3) the efuse carries the PCI identity, the module version
and the OTP region but **not** the RF calibration, so without the PPR overlay the part
will associate but transmit at firmware default power, and the per-rate power curve is
uncalibrated. The host is expected to treat "production board and no PPR overlay" as a
configuration error. `[C]`

---

## 2. EEPROM image layout (2560 bytes)

### 2.1 Size and addressing

* Image size is **0xA00 = 2560 bytes** — bit-identical to the upstream `mt76`
  `mt7921`/`mt792x` EEPROM address space (`__MT_EE_MAX = 0x9ff`). `[C]`
* The firmware's EEPROM shadow buffer accepts up to **0x E00 = 3584 bytes**
  (`MAX_EEPROM_BUFFER_SIZE` in public `gen4m`); the extra space above 0xA00 is unused on
  this part. `[C]`
* Upload granularity is **0x400 = 1024 bytes** (`BUFFER_BIN_PAGE_SIZE`). `[C]`
* Efuse read/write granularity is **16 bytes** (`MT7921_EEPROM_BLOCK_SIZE`). `[C]`
* All multi-byte scalars are **little-endian**. `[C]`

### 2.2 Field table

Offsets that match the public MT7921/MT7922 definitions are marked "= mt792x".

| Offset | Size | Field | Observed value in the shipped image | Notes |
|---|---|---|---|---|
| `0x000` | 2 | Chip ID (`MT_EE_CHIP_ID`) | `0x7932` | = mt792x. Firmware/host sanity check. `[C]` |
| `0x002` | 1 | EEPROM format version (`MT_EE_VERSION`) | `0x01` | = mt792x `[C]` |
| `0x003` | 1 | reserved | `0x00` | `[C]` |
| `0x004` | 6 | Station MAC address (`MT_EE_MAC_ADDR`) | **all zero** | = mt792x offset. Byte order is wire order, octet[0] first (i.e. OUI first). In the shipped image the field is a zero placeholder — the host always overrides (§4). `[C]` |
| `0x00A` | 6 | reserved | zero | `[C]` |
| `0x010` | 16 | **PCI identity record, Wi-Fi function** | see below | **MT7932-specific region, not present in public mt7921 layout** `[C]` |
| `0x020` | 16 | **PCI identity record, Bluetooth function** | see below | idem `[C]` |
| `0x045` | 3 | RF/clock trim word A | `a0 93 11` | purpose not determined `[U]` |
| `0x049` | 3 | RF/clock trim word B | `c0 4d 10` | `[U]` |
| `0x04C` | 2 | RF/clock trim word C | `08 51` | `[U]` |
| `0x075` | 1 | RF front-end configuration | `0xC0` | `[U]` |
| `0x07A` | 1 | **Module enable / module version** | `0x89` → enable = 1, version = 9 | bit7 = module enabled; bits[3:0] = revision. Read from *efuse* on a burned board, from *this offset* when the binary image is used. `[C]` |
| `0x07B` | 1 | module sub-config | `0x20` | `[U]` |
| `0x07C` | 4 | `MT_EE_WIFI_CONF` (mt792x) | zero | = mt792x offset; not populated on this part `[C]` |
| `0x138` | 1 | RF/PA option | `0x47` | `[U]` |
| `0x145` | 3 | front-end loss / path option | `0c 0c 0f` | `[U]` |
| `0x14D` | 1 | front-end loss / path option | `0x11` | `[U]` |
| `0x154` | 1 | feature mask | `0xFF` | `[U]` |
| `0x15B` | 9 | per-band maximum TX power ceiling | `3c 30 2c 00 3c 3c 3c 3c 00` | 0.5 dBm units (0x3C = 30 dBm) `[L]` |
| `0x164` | 36 | **Per-sub-band TX target power** (WF0 2.4 GHz target starts here) | `23` (17.5 dBm) for 2.4 GHz groups, `1e` (15 dBm) / `23` for 5 GHz groups | signed 0.5 dBm. The byte at `0x164` alone is the board-type discriminator of §1.2 `[C]` |
| `0x18E` | 16 | per-sub-band front-end/path loss | `04 04 04 04 05 05 05 04 …` | 0.5 dB `[L]` |
| `0x19E` | 94 | **2.4 GHz per-rate power offsets** | see §7.1 | `[C]` |
| `0x1FC` | 138 | **5 GHz per-rate power offsets** | see §7.1 | `[C]` |
| `0x2A1` | 49 | Temperature-compensation curves, 7 curves × 7 points | `80 80 eb f8 fe 08 16 …` | signed 8-bit deltas `[L]` |
| `0x300` | 10 | Temperature threshold table, 2 chains × 5 points | `0d 1a 27 34 41` ×2 | thresholds 13/26/39/52/65 `[L]` |
| `0x325` | 21 | Temperature-compensation curves, 3 × 7 | `d7 e3 ef fb 04 09 17 …` | `[L]` |
| `0x373` | 10 | Temperature threshold table, 2 × 5 | `0d 1a 27 34 41` ×2 | `[L]` |
| `0x400` | 25 | RF/PLL/coexistence configuration | `be a8 00 81 …` | `[U]` |
| `0x440` | 18 | **Per-chain / per-band RF path record: 6 × 3 bytes, arranged as 2 chains × 3 bands** | `02 24 01 / 02 50 03 / 02 50 03` then the identical triple repeated | The exact repetition of the 3-band triple confirms **2 transmit/receive chains and 3 bands** `[C]` for the structure, `[L]` for the field meanings |
| `0x55A` | 1 | | `0x01` | `[U]` |
| `0x55B` | 1 | `MT_EE_HW_TYPE` (mt792x) | `0x00` | = mt792x offset. bit0 = `MT_EE_HW_TYPE_ENCAP` (hardware Ethernet-mode TX encapsulation). Cleared → 802.11 native TXD. Upstream reads this one via the efuse-access command, not from the image. `[C]` |
| `0x55D` | 37 | per-channel gain/backoff table | `d5` then `d0` ×36 | `[U]` |
| `0x860`, `0x884` | 1 each | | `0x0B` | `[U]` |
| `0x8A8` | 3 | | `1c 1c 1c` | `[U]` |
| `0x943` | 32 | 6 GHz / miscellaneous RF configuration | `83 83 0f 0f 00 00 78 00 …` | `[U]` |
| `0x97F` | 10 | Temperature threshold table, 2 × 5 | `0d 1a 27 34 41` ×2 | `[L]` |
| `0x9B9` | 10 | Temperature threshold table, 2 × 5 | `0d 1a 27 34 41` ×2 | `[L]` |
| — | — | everything else | zero | |

### 2.3 The PCI identity records

Both records have the same 16-byte layout. The **byte values** below are `[C]` (read straight
out of the shipped image); the reading of them as **the PCI configuration-space identity
dwords** is `[L]` — the vendor/device pairs at +0x0 and +0x8 are unambiguous, the class-code
and configuration words are inferred from their shape. The chip-delta section §1.4 marks the
same table the same way.

| Rel. offset | Size | Field | Wi-Fi record (`0x010`) | Bluetooth record (`0x020`) |
|---|---|---|---|---|
| +0x0 | 4 | `(vendor << 16) \| device` | `0x14C3_7932` | `0x14C3_793B` |
| +0x4 | 4 | `(class_code << 8) \| revision_id` | `0x0280_0000` → class `02:80:00` (Network controller / Other), revision 0 | `0x0280_0000` |
| +0x8 | 4 | `(subsystem_vendor << 16) \| subsystem_device` | `0x14C3_7932` | `0x14C3_793B` |
| +0xC | 4 | device configuration word | `0x0000_A210` | `0x0000_0000` |

**Answer to "what is each pair for":** the first pair (`14C3:7932`) programmes the
identity of the **Wi-Fi PCIe function**; the second pair (`14C3:793B`) programmes the
identity of the **Bluetooth PCIe function of the same combo die**. The family follows a
consistent `79xx`/`79xB` pairing between the Wi-Fi and Bluetooth device IDs of one package
(7922 ↔ 792A, 7923 ↔ 792B, 7932 ↔ 793B). `[C]`
The subsystem IDs are set equal to the primary IDs, i.e. this design does not use
subsystem IDs to distinguish SKUs. `[C]`
The trailing word `0xA210` in the Wi-Fi record is an unidentified per-function
configuration value; the Bluetooth record leaves it zero. `[U]`

**Delta:** public MT7921/MT7922 EEPROM layouts do not define this region; the identity
records at `0x010`/`0x020` are an MT7932-specific (or at least mt792x-combo-specific)
addition. Everything else that is defined publicly (`0x000` chip id, `0x002` version,
`0x004` MAC, `0x07C` WIFI_CONF, `0x55B` HW_TYPE, total size `0xA00`) is at the identical
offset. `[C]`

---

## 3. Getting the image into the firmware

### 3.1 EEPROM / buffer-bin upload

Command: `CMD_ID_LAYER_0_EXT_MAGIC_NUM` (`0xED`) with extended CID
`EXT_CMD_ID_EFUSE_BUFFER_MODE = 0x21`, set direction. Payload (public
`CMD_EFUSE_BUFFER_MODE_CONNAC_T`): `[C]`

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `ucSourceMode` — 0 = efuse, 1 = buffer/binary, 2 = per-rate-power overlay, 3 = one-time-cal blob |
| 1 | 1 | `ucContentFormat` |
| 2 | 2 | `u2Count` — payload byte count in this message |
| 4 | ≤1024 | `aBinContent[]` |

`ucContentFormat` bit layout for paged buffer-mode uploads — **identical to the public
`mt7915` flash buffer-mode encoding**: `[C]`

| Bits | Field |
|---|---|
| [1:0] | format: `1` = `EE_FORMAT_WHOLE` |
| [4:2] | page index `i` (0-based) |
| [7:5] | `total_pages − 1` |

For the shipped 2560-byte image, `total_pages = ceil(2560/1024) = 3`, so exactly three
commands are issued: `[C]`

| # | `ucSourceMode` | `ucContentFormat` | `u2Count` | payload |
|---|---|---|---|---|
| 0 | 1 | `0x41` | 1024 | image[0x000…0x3FF] |
| 1 | 1 | `0x45` | 1024 | image[0x400…0x7FF] |
| 2 | 1 | `0x49` | 512 | image[0x800…0x9FF] |

For `ucSourceMode = 0` (efuse-only) the identical page loop is executed with
`u2Count = 0` and a zeroed payload; the page count is 1. `[C]`

For `ucSourceMode = 2` (PPR) and `ucSourceMode = 3` (WCal) the message is **not** paged:
`ucContentFormat = 0`, `u2Count` = whole file length, which must be ≤ 1024 bytes.
The 412-byte PPR blob therefore goes in one command. `[C]`

The host-side staging buffer is 3584 bytes and a file longer than that is rejected. `[C]`

### 3.2 Position in the initialisation order

`[C]` unless noted:

1. Chip power-on, WFDMA/HIF bring-up.
2. ROM-patch download, RAM-code download, MCU start / handshake.
3. **Board-type detection** — two efuse block reads (`0x070`, `0x160`) via
   `EXT_CMD_ID_EFUSE_ACCESS`.
4. **EEPROM / efuse / PPR provisioning** — the `EXT_CMD_ID_EFUSE_BUFFER_MODE`
   sequence of §3.1.
5. Module-version read-back → sets the one-time-calibration block geometry (§8).
6. Firmware version query → yields the one-time-calibration format version.
7. On a production board: **one-time-calibration replay** — `WCal` (source mode 3), then
   the TLV calibration caches (§8).
8. Calibration-type negotiation with firmware (§8.4).
9. Normal adapter start; MAC address applied; regulatory domain and TX-power-limit tables
   pushed (§7, §9).

Steps 3–8 all occur before the interface is registered.

### 3.3 Reading efuse back

| Purpose | Command | Payload |
|---|---|---|
| Read one 16-byte efuse block | `0xED` / ext CID `EXT_CMD_ID_EFUSE_ACCESS = 0x01`, **query** | `struct CMD_ACCESS_EFUSE` = `{ u32 u4Address; u32 u4Valid; u8 aucData[16] }` — 24 bytes. `u4Address` must be 16-byte aligned; the response returns the block in `aucData[]` and the host indexes `addr % 16`. `[C]` |
| Write one 16-byte efuse block | same, ext CID `0x01`, **set** | same 24-byte structure `[C]` |
| Free-block query | `0xED` / ext CID `EXT_CMD_ID_EFUSE_FREE_BLOCK = 0x4F`, query | `struct CMD_EFUSE_FREE_BLOCK` = `{ u8 ucGetFreeBlock = 0; u8 ucVersion = 1; u8 ucDieIndex; u8 rsv }` — 4 bytes. Response reports free-block count and total block count. **Block size is 16 bytes.** `[C]` |
| Read one efuse byte during initialisation (before the normal command path is up) | initialisation-command CID `0x50`, 68-byte command carrying a 32-bit efuse byte address; the single byte arrives in the completion event `[C]` | The command is implemented by this host and is **not** in the public `gen4m` init-command enumeration `[C]`; whether MT7932 firmware answers it is `[U]` (open question 9) |
| Bulk OTP read | repeated single-byte reads from efuse `0x200`, length capped at 64 bytes | `[C]` |

`EXT_CMD_ID_EFUSE_BUFFER_RD = 0x4E` (read back the firmware's EEPROM shadow) exists in the
command space but is not required by the provisioning flow. `[L]`

### 3.4 Write path (secure efuse)

A separate manufacturing-only path programmes a **secure efuse** region from a
`SEC_EEPROM.bin` container. The container has: a header carrying two magic numbers and a
chip-ID field (validated against the running chip ID), followed by a sequence of blocks,
each with its own magic and a size field, **maximum 256 bytes per block**. Programming is
refused unless the manufacturing firmware is running. `[C]` for the container
shape, `[U]` for the block encoding.

---

## 4. MAC address

### 4.1 Sources and precedence

`[C]`

The chip itself has one MAC-address source: the EEPROM/efuse field at `0x004`, which
firmware will use if it is populated. In the shipped image that field is **zero**, so the
address must come from the host and be applied with the "set network address" command.
`[C]`

Where the host obtains the address is host policy. The available sources, in the order a
driver would normally try them: `[C]`

| Priority | Source | Notes |
|---|---|---|
| 1 | An explicit configuration override (gen4m carries this as `MacOverride` + `MacAddr` in `wifi.cfg`) | ASCII, colon-separated, 17 characters + NUL, strictly validated |
| 2 | A platform-supplied 6-byte address property | the production source on a platform that provides one |
| 3 | A locally generated address | e.g. a fixed OUI plus 3 varying bytes; only when neither of the above yields a usable address |

An address whose octet[0] has the multicast bit set must be rejected and treated as absent.
`[C]`

### 4.2 Byte order

Wire order throughout: octet[0] (the OUI's first byte) is the lowest address, in the
EEPROM field and in every host-supplied form. `[C]`

### 4.3 Derived interface addresses

Additional concurrent interfaces need distinct addresses. One workable derivation from a
single base address toggles the locally-administered bit group of octet[0]: `base`,
`base[0] ^ 0x02`, `base[0] | 0x02`, `(base[0] & ~0x02) ^ 0x06`. This is host policy, not a
hardware constraint. `[C]`

---

## 5. Antenna and chain configuration

`[C]` unless noted.

* **2 transmit chains and 2 receive chains** (WF0/WF1), 2 spatial streams. `[L]` — the
  evidence is `[C]` but the conclusion is drawn from it, and the firmware-reported `ucNss` was
  not observed. Evidence: every per-antenna table shipped for this part is two-wide
  (`siso_wf0/siso_wf1`, `mimo_wf0/mimo_wf1`, `cck_siso_wf0/wf1`, …); the EEPROM per-chain
  RF-path record at `0x440` contains exactly two identical 3-band triples.
* The spatial-stream count actually used is min-combined with the firmware-reported
  `ucNss` (§11); 2 is the ceiling for this part. Per-role reductions are a host choice.
* Per-rate power tables carry **three antenna configurations** per PHY mode:
  * `s_` — SISO (single chain)
  * `c_` — CDD (one stream transmitted from both chains with cyclic delay diversity)
  * `m_` — MIMO (two spatial streams)
  A per-channel "CDD not supported" bitmap and a "1SS/1T" special-path index exist and are
  pushed to firmware separately (§7.7).
* **Antenna sharing / SDB.** The part supports Simultaneous Dual Band operation on a
  shared antenna pair. The configurable modes are:

  | Code | Mode |
  |---|---|
  | 0 | non-SDB (both chains on one band) |
  | 1 | SDB (both bands concurrently, chains split) |
  | 2 | Core #0 1SS, non-SDB |
  | 3 | **Antenna #0 shared, SDB** |
  | 4 | Core #1 1SS, non-SDB |
  | 5 | **Antenna #1 shared, SDB** |

  The channel-group axis for SDB is `{all, 2G4, 5G-low, 5G-high, 6G-low, 6G-high}` — which
  independently confirms 6 GHz capability. A dedicated TX-power backoff table
  (`TxPwrLimit_SDB.dat`, §7.4) applies while any shared-antenna SDB mode is active.
* **Bluetooth coexistence.** The Bluetooth function shares the same die, the same efuse
  and the same antennas; its PCI identity is programmed from the same EEPROM image
  (§2.3). Coexistence/antenna-sharing controls are issued through the chip's coexistence
  configuration channel (see also §7.6, time-averaged SAR, which is a coexistence-domain
  setting). `[L]`

---

## 6. Supported bands and channel plan

`[C]`

* **2.4 GHz** — channels 1…14 (limit tables carry ch001…ch013; the regulatory matrix has a
  2g14 column).
* **5 GHz** — channels 36…181 including the 2 MHz-spaced "duplicate" centres used for
  wide-bandwidth entries (ch038, ch042, … and the explicit `chAAA-BBB` 40/80/160 MHz
  centre/primary pairs).
* **6 GHz** — channels 1…233 in the 4-channel grid used by the limit tables
  (ch001, ch005, …, ch233), and the full 2 MHz grid in the regulatory matrix. 6 GHz
  support is gated by a runtime capability flag; when it is clear, the 6 GHz section of
  the domain-info command is emitted with a zero channel count and the 6 GHz power tables
  are skipped.

Evidence for 6 GHz: a dedicated 6 GHz limit table with LPI/VLP/SP power modes, a 6 GHz
antenna-gain table covering 5925–7125 MHz, a `6gCC` section in the regulatory matrix, the
6 GHz SDB channel groups, and a 6 GHz-specific power-mode negotiation command. `[C]`

There is no per-band chain restriction in the shipped data: all three bands carry 2-chain
entries. `[C]` The EEPROM per-rate region is provisioned only for 2.4 GHz and 5 GHz; the
6 GHz per-rate section of the PPR overlay is present but essentially empty in the shipped
build (§7.1). `[C]`

---

## 7. Transmit power provisioning

### 7.1 Per-rate power (`PPR`) blob

#### 7.1.1 Container

`PPR_MT7932.bin`, 412 bytes. `[C]`

```
+0x00  4   magic  "BLOB"
+0x04  4   header length            = 0x4C (76)
+0x08  2   container version        = 2
+0x0A  2   section count            = 3
+0x0C  4   reserved                 = 0
+0x10  n*20  section descriptors
```

Section descriptor, 20 bytes: `[C]`

```
+0x00  1   type        = 1
+0x01  1   section id  = 1, 2, 3
+0x02  1   section version = 1
+0x03  1   reserved    = 0
+0x04  4   byte offset of the section body from file start
+0x08  4   section body length, *including* the body header
+0x0C  4   checksum: 8-bit-wide sum of all body bytes, zero-extended to 32 bits
+0x10  4   reserved    = 0
```

Section body: `[C]`

```
+0x00  4   the descriptor's 4 tag bytes, repeated verbatim
+0x04  4   length, repeated verbatim
+0x08  n   payload
```

Shipped instance (all three checksums verified as plain byte sums over the body):

| id | offset | length | checksum | payload bytes | band |
|---|---|---|---|---|---|
| 1 | `0x04C` | `0x068` | `0x29B2` | 96 | 2.4 GHz `[L]` |
| 2 | `0x0B4` | `0x0B8` | `0x51CD` | 176 | 5 GHz `[L]` |
| 3 | `0x16C` | `0x030` | `0x00B6` | 40 | 6 GHz, unprovisioned in the shipped blob `[L]` |

Sections are contiguous: `offset(n) + length(n) == offset(n+1)`, and the last section ends
exactly at end-of-file. `[C]`

#### 7.1.2 Payload encoding

Every per-rate byte uses MediaTek's classic **enable/sign/magnitude** power-delta encoding
(the same one public `mt76` implements as `sign_extend_optional(val, 7)`): `[C]`

| Bit | Meaning |
|---|---|
| 7 | entry valid. `0x00` means "no offset" |
| 6 | sign: 1 = positive, 0 = negative |
| [5:0] | magnitude |

Unit: **0.5 dB**. Examples from the shipped blob: `0xC8` = +4.0 dB, `0xC4` = +2.0 dB,
`0x83` = −1.5 dB, `0x00` = 0 dB. `[C]` The values decrease monotonically with MCS index `[C]`;
that they are **offsets applied to the per-sub-band target power** rather than absolute
powers, and that the shape is the expected EVM-driven per-rate backoff, is `[L]`.

#### 7.1.3 Per-rate group layout

The per-rate arrays are laid out identically in the PPR sections and in the EEPROM image
(the PPR overlay is a drop-in replacement for the EEPROM's per-rate region): `[C]`

**2.4 GHz block — 94 bytes** (EEPROM `0x19E`…`0x1FB`, PPR section 1 payload bytes 1…94):

| Rel. | Count | Group |
|---|---|---|
| +0 | 4 | CCK 1 / 2 / 5.5 / 11 Mb/s |
| +4 | 8 | OFDM 6 / 9 / 12 / 18 / 24 / 36 / 48 / 54 Mb/s |
| +12 | 10 | VHT20 MCS0…9 |
| +22 | 10 (+2 pad) | VHT40 MCS0…9 |
| +34 | 12 | HE RU26 MCS0…11 |
| +46 | 12 | HE RU52 |
| +58 | 12 | HE RU106 |
| +70 | 12 | HE RU242 |
| +82 | 12 | HE RU484 |

**5 GHz block — 138 bytes** (EEPROM `0x1FC`…`0x285`, PPR section 2 payload bytes 35…172):

| Rel. | Count | Group |
|---|---|---|
| +0 | 8 | OFDM ×8 |
| +8 | 10 | VHT20 MCS0…9 |
| +18 | 10 | VHT40 |
| +28 | 10 | VHT80 |
| +38 | 10 (+6 pad) | VHT160 |
| +54 | 12 | HE RU26 |
| +66 | 12 | HE RU52 |
| +78 | 12 | HE RU106 |
| +90 | 12 | HE RU242 |
| +102 | 12 | HE RU484 |
| +114 | 12 | HE RU996 |
| +126 | 12 | HE RU996×2 |

`[L]` for the group *labels*; `[C]` for the group *boundaries and sizes* (they are forced
by the observed 10-valid-then-2-zero versus 12-valid patterns, and by the fact that the
2.4 GHz array stops one RU short of the 5 GHz array).

PPR section 1's payload additionally has a leading byte (`0x22` = 17.0 dBm target power)
and a trailing byte (`0x1B`); PPR section 2's payload has a leading 35-byte per-sub-band
target-power array (0.5 dBm; observed values `0x22`/`0x1F`/`0x1C`/`0x21`/`0x19`) and 3
trailing bytes. `[C]` for the bytes, `[L]` for "target power".

PPR section 3 (6 GHz) is 40 bytes with only its first byte non-zero (`0x81`, i.e.
−0.5 dB); 6 GHz per-rate power is not provisioned by the shipped overlay. `[C]`

#### 7.1.4 Command

Pushed with `EXT_CMD_ID_EFUSE_BUFFER_MODE`, `ucSourceMode = 2`, `ucContentFormat = 0`,
`u2Count` = 412, payload = the whole file including its `BLOB` header. The firmware parses
the container. `[C]`

### 7.2 TX-power-limit tables (2.4/5 GHz)

`TxPwrLimit_MT79x1.dat`, plain 8-bit ASCII, CRLF-agnostic. `[C]`

```
{Ver:<build string> Date:<YYYY-MM-DD hh:mm:ss>}
<Ver:03>

[<CC>]                       # regulatory domain, ISO-3166 alpha-2 or a private code
<cck,  c1, c2, c5, c11>
ch001, 30, 30, 30, 30
...
</cck>
<ofdm, o6, o9, o12, o18, o24, o36, o48, o54>
...
</ofdm>
...
```

* **Axes**: regulatory domain (outer `[CC]` block) → PHY mode / bandwidth (`<section>`
  tag) → channel or channel pair (row) → antenna configuration and MCS (column).
* **Sections**, in file order (15): `cck`, `ofdm`, `ht20`, `ht40`, `vht20`, `vht40`,
  `vht80`, `vht160`, `ru26`, `ru52`, `ru106`, `ru242`, `ru484`, `ru996`, `ru996X2`.
  This is exactly the public `gen4m` 15-section HE set. `[C]`
* **Columns**: `cck` → `c1,c2,c5,c11`; `ofdm` → `o6…o54`; every MCS-indexed section
  carries **three antenna groups** — `s_m0…`, `c_m0…`, `m_m0…` (SISO, CDD, MIMO) — with
  MCS0-7 for HT, MCS0-9 for VHT, MCS0-11 for HE/RU.
  **Delta:** public `gen4m` version-2 tables have a single `m0…m11` column group. The
  `s_`/`c_`/`m_` triplication is what `<Ver:03>` denotes and is specific to this
  generation. `[C]`
* **Rows**: `chNNN` for a plain channel, and `chAAA-BBB` for a wide-bandwidth entry where
  `AAA` is the centre channel and `BBB` the primary. `[C]`
* **Units**: signed 8-bit, **0.5 dBm**, valid range 0…63 for absolute values
  (`TX_PWR_LIMIT_MAX_VAL = 63` in public `gen4m`); e.g. `40` = 20.0 dBm. `[C]`
* **Tokens**: `X` = not applicable for this combination; `Y` = no independent value (the
  entry inherits / is a duplicate centre). `[L]`
* The shipped file carries **252 regulatory-domain blocks**, including the private codes
  `XZ` (world-wide default), `X0`, `X2`, `X3` and the `A0`/`A1`/`A2`/`A4`/`A5`/`A6`
  pseudo-domains. `XZ` is the fallback when the requested country is absent. `[C]`

**How a driver applies it:** for the active country, load every section, build a
per-channel array of 432 values laid out as
`cck[4] | ofdm[8] | ht20[3][8] | ht40[3][8] | vht20[3][10] | vht40[3][10] | vht80[3][10] |
vht160[3][10] | ru26[3][12] | ru52[3][12] | ru106[3][12] | ru242[3][12] | ru484[3][12] |
ru996[3][12] | ru996x2[3][12]` (the `[3]` being SISO/CDD/MIMO), then compress it to the
121-byte firmware vector described in §7.7. `[C]`

### 7.3 Antenna-gain table

`TxPwrLimit_AntGain.dat`. `[C]`

```
<ant_gain, siso, cdd, mimo>
group1, 16, 16, 10
...
group6, 15, 16, 10
</ant_gain>
```

* Six frequency groups, **6 GHz only**, documented in the file itself:
  group1 5925–6105 (UNII-5_1), group2 6106–6265, group3 6266–6425, group4 6426–6525,
  group5 6526–6875, group6 6876–7125 MHz.
* Three values per group: SISO, CDD, MIMO antenna gain.
* **Unit 0.5 dB**, signed. Used to convert the regulatory EIRP limits of the 6 GHz table
  into conducted per-chain limits and to fill the Transmit Power Envelope report.
* Adjusts: the 6 GHz maximum-power computation only. `[C]`

### 7.4 SDB (shared-antenna dual-band) table

`TxPwrLimit_SDB.dat`. `[C]`

```
<sdb, cck_siso_wf0, cck_siso_wf1, ofdm_siso_wf0, ofdm_siso_wf1,
      ofdma_siso_wf0, ofdma_siso_wf1, cck_mimo_wf0, cck_mimo_wf1,
      ofdm_mimo_wf0, ofdm_mimo_wf1, ofdma_mimo_wf0, ofdma_mimo_wf1>
subband1, 42, 42, 42, 42, 42, 42, X, X, X, X, X, X
... subband17
</sdb>
```

* 17 sub-bands × 12 values (2 chains × {CCK, OFDM, OFDMA} × {SISO, MIMO}).
* Sub-bands 1–3 are the CCK-capable (2.4 GHz) ones; 4–17 carry only OFDM/OFDMA.
* `X` = not applicable. Unit **0.5 dBm** (`42` = 21 dBm). `[C]`
* Adjusts: the maximum power while a shared-antenna SDB mode is active. `[L]`
* The shipped table is not country-partitioned (a single block). `[C]`

### 7.5 2.4 GHz common-path backoff table

`TxPwrLimit_2gCommonPath.dat`. `[C]`

```
[<CC>]
<common_path_backoff, backoff>
ch001, 9
... ch013
</common_path_backoff>
```

* Per country, 13 rows (2.4 GHz channels 1–13), one **backoff** value each.
* Unit 0.5 dB; typical value 8 (= 4 dB) with 9 at channel 1 in the Americas and 5 at
  channel 13 for CN; the `XZ`/`X0`/`X2`/`X3` pseudo-domains are all zero.
* Adjusts: an additional backoff applied to the shared/common RF path in 2.4 GHz. `[L]`
* Fallback: if the country is absent, the `XZ` block is loaded. `[C]`

### 7.6 SAR table and dynamic SAR

`TxPwrLimit_SAR.dat`. `[C]`

```
{Ver:<build string> Date:...}
<Ver:01>

[<CC>]
<tab, siso_wf0, siso_wf1, mimo_wf0, mimo_wf1>
subband1, 39, 38, 39, 38
... subband20
</tab>
```

* **259 country blocks**, each 20 sub-bands × 4 values (2 chains × {SISO, MIMO}).
* Unit **0.5 dBm** absolute cap (39 = 19.5 dBm). `[C]`
* Selection: by the active regulatory country code; falls back to a default block when
  the country is absent. `[C]`
* The 20 sub-bands span 2.4 GHz (sub-bands 1–5 in the shipped data hold the highest caps)
  through 5/6 GHz. `[L]`

**Dynamic SAR / time-averaged SAR.** Two orthogonal controls exist on top of the static
table: `[C]`

| Control | Carried as | Effect |
|---|---|---|
| SAR enable | a chip-configuration string command (`CMD_ID_CHIP_CONFIG`, `0xCA`) | globally arms/disarms SAR capping |
| SAR level | a chip-configuration string command carrying an index and a value | selects the applied SAR entry |
| Time-averaged SAR window | a coexistence-configuration string command, parameter 13, values 0..3 | selects one of **four** time-averaging windows |

The selected level modulates the caps that are then sent to firmware with the SAR
sub-command of §7.7. The currently applied level can be queried back. `[C]`

### 7.7 The TX-power-limit commands

All of the tables above are pushed with the **same** command:
`CMD_ID_SET_COUNTRY_POWER_LIMIT_PER_RATE = 0x5D`
(upstream `mt76`: `MCU_CE_CMD_SET_RATE_TX_POWER`). `[C]`

Common 44-byte header — byte-compatible with upstream's
`struct mt76_connac_tx_power_limit_tlv`: `[C]`

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0 | 1 | `ver` | table format version — `3` for the shipped `<Ver:03>` tables |
| 1 | 1 | flags | MT7932 delta: upstream pads this. Bit1 is set while the record array is being filled and bit3 additionally before transmission `[U]` |
| 2 | 2 | `len` | total command length |
| 4 | 1 | `n_chan` | number of records in this message |
| 5 | 1 | `band` | 1 = 2.4 GHz, 2 = 5 GHz, 3 = 6 GHz |
| 6 | 1 | `last_msg` | 1 on the final fragment |
| 7 | 1 | sub-table selector | MT7932 delta: upstream pads this. **0 = per-rate limit, 1 = SAR, 2 = SDB, 4 = 2.4 GHz common-path** |
| 8 | 4 | `alpha2[4]` | regulatory country code |
| 12 | 32 | auxiliary parameter block | MT7932 delta: upstream pads this with zeros; here it carries a 32-byte per-command parameter block `[U]` |
| 44 | … | record array | |

Per sub-table:

| Sub-table | Record size | Record content | Records per command |
|---|---|---|---|
| per-rate limit (`0`) | **122 B** | `u8 channel` + **121** signed 0.5 dBm values | **8** (fragmented; a chip-level constant) |
| SAR (`1`) | 5 B | `u8 sub-band index` + 4 values | all 20 in one command |
| SDB (`2`) | 13 B | `u8 sub-band index` + 12 values | all 17 in one command |
| 2.4 GHz common path (`4`) | 1 B | one backoff value per channel | 13 in one command |
| unsupported-rate bitmap | — | fixed 532-byte command carrying a per-channel not-supported-rate bitmap and a CDD-not-supported bitmap | one command |
| Transmit Power Envelope | — | fixed 60-byte command: enable, band, primary channel, per-bandwidth maximum TX power limits (`0x7F` = unset) plus the SISO/CDD/MIMO antenna gains from §7.3 | per channel switch |

**Delta on the per-rate record.** Upstream `mt76` defines
`MT_SKU_POWER_LIMIT = 161` values per channel (`struct mt76_connac_sku_tlv` = 162 B).
MT7932 uses **121** values per channel (record = 122 B). The 121 values are: `[C]`

| # | Group | Values |
|---|---|---|
| 1 | CCK | 1 (representative of 1/2/5.5/11 Mb/s) |
| 3 | OFDM | 3 (6–18, 24–36, 48–54 Mb/s) |
| 9 each, ×13 | `ht20, ht40, vht20, vht40, vht80, vht160, ru26, ru52, ru106, ru242, ru484, ru996, ru996x2` | 3 antenna configurations (SISO, CDD, MIMO) × 3 MCS groups |

Total = 1 + 3 + 13×9 = 121.
Within each antenna group the three MCS-group representatives are taken from the parsed
table at relative element indices **{0, 3, 5}** for VHT and HE/RU sections (i.e. MCS0,
MCS3, MCS5 → the "L / M / H" grouping), and at **{1, 5, 7}** for HT sections when the
table version is 3 (**{0, 3, 5}** for version 2). The OFDM triple is taken from elements
`o6`, `o24`, `o48`. `[C]`

**Ordering of the whole power-limit push:** `[C]`

1. load the 2.4/5 GHz limit table for the active country (fallback `XZ`), mark
   unsupported channels;
2. load the SAR table for the active country;
3. load the SDB table;
4. load the 2.4 GHz common-path table (fallback `XZ`);
5. send: per-rate limit (fragmented, 8 channels per command, 2.4 GHz then 5 GHz), then
   SAR, then 2.4 GHz common path, then SDB (only if the SDB table loaded);
6. if 6 GHz is supported: initialise the antenna-gain table, then load and send the 6 GHz
   tables. `[C]`

### 7.8 6 GHz specifics

`TxPwrLimit6G_MT79x1.dat`. `[C]`

* Block key is a **pair**: `[<CC>,<power mode>]` where the power mode is one of
  **`LP`** (low-power indoor), **`VLP`** (very low power) and **`SP`** (standard power).
  The shipped file has 374 such blocks.
* Sections: `ofdm`, `ru26`, `ru52`, `ru106`, `ru242`, `ru484`, `ru996`, `ru996X2` only —
  no `cck`, no `ht*`, no `vht*`.
* Rows are the 4-channel-spaced 6 GHz grid `ch001, ch005, …, ch233`.
* Columns use the same `s_`/`c_`/`m_` triplication with MCS0…11.
* Units: signed, 0.5 dB. Observed magnitudes: `SP` ≈ +35…+37, `LP` ≈ −1…+1,
  `VLP` ≈ −7…−16 — the LPI/VLP entries are low because they express the regulatory
  EIRP/PSD ceilings after the 6 GHz antenna gain of §7.3 has been removed. `[L]`
* Within one row the SISO / CDD / MIMO columns differ by fixed steps (observed
  SISO → MIMO = −6 units = −3 dB, SISO → CDD = −12 units = −6 dB), i.e. the per-chain
  split of a common EIRP budget. `[C]`
* Power-mode selection is negotiated at runtime from the AP's HE 6 GHz regulatory info and
  pushed per-BSS; a separate command reports the P2P-VLP-allowed channel list to firmware
  and resets it when VLP is disallowed. `[C]`
* **1SS/1T handling.** A dedicated "1SS/1T special-path index" update exists: for
  1-spatial-stream AWDL/NAN traffic the special-path index is overridden (only the
  1SS scenario is affected; other scenarios keep their original index). A helper
  enumerates the channels for which the 1SS/1T variant applies. `[C]` for existence,
  `[U]` for the index encoding.
* An unsupported-rate consistency pass runs over the assembled 6 GHz table and produces
  the not-supported-rate / CDD-not-supported bitmaps of §7.7. `[C]`

---

## 8. Runtime and one-time calibration

### 8.1 What the firmware does at start

The RF calibration proper (RX DCOC, TX DPD, LOFT, TSSI/thermal) runs inside the firmware
during the "PHY action" stage of start-up. On a production module the host does **not**
re-run it every boot; instead it replays a cached result. `[L]`

An integrity check reads five chip registers after calibration and fails if any reads
back zero: `[C]`

| Address |
|---|
| `0x830A_D418` |
| `0x830A_D424` |
| `0x830A_D42C` |
| `0x8300_7A14` |
| `0x8301_7A14` |

(The last two are the same register in the two per-band PHY instances — `0x8300_xxxx`
and `0x8301_xxxx`.) `[L]`

### 8.2 The one-time-calibration ("OCAL") cache

Whether the one-time-calibration machinery runs at all, and which on-disk format
generation is expected, are host selections. `[C]`

Five distinct cached calibration objects exist. Each is an opaque blob that the host holds
and replays; none of them is produced or interpreted by the host. `[C]`

| Type | Object | Delivery to firmware |
|---|---|---|
| 0 | **WCal** | `EXT_CMD_ID_EFUSE_BUFFER_MODE`, `ucSourceMode = 3`, ≤1024 B, one command |
| 1 | **OCA2** | parsed as TLV by the host, then pushed with the calibration-data command (§8.3) |
| 2 | **OCAL** | idem |
| 3 | **IFCAL** (in-field calibration) | idem |
| 4 | **runtime calibration** | idem |

Ordering constraints imposed by the firmware: **WCal must be loaded before OCAL**,
and **OCAL must be loaded before IFCAL**. `[C]`
A parallel Bluetooth calibration blob (**BCal**) exists for the BT
function of the same die. `[C]`
The in-field re-calibration flow additionally produces "saved / triggered / result /
FDR / disconnect" state that the host must persist. `[C]`

### 8.3 Storage format and sizes

* The host-side working buffer for a calibration file is **0x64000 = 409 600 bytes**;
  a single object may not exceed it. `[C]`
* Objects of type 1–4 are **TLV files**. The TLV header carries a big-endian format
  version that is compared against the version reported by the running firmware; a
  mismatch aborts the load. Version 2 is the first that carries the version field; a
  file declaring version `0x0200` is accepted as version 2 when the firmware reports 3.
  `[C]`
* The calibration data is organised as numbered **blocks**. Each calibration type maps to
  a block range and a total byte size: `[C]`

  | Type | First block | Last block | Size (bytes) |
  |---|---|---|---|
  | 0 | 0 | 2 | 5 508 |
  | 1 | 0 | 0 | 3 276 |
  | 2 | 1 | 17 | 55 692 (17 × 3 276) |
  | 3 | 18 | 34 | 55 692 (17 × 3 276) |
  | 4 | 0 | 13 | `N × 14 × 624`, where `N` is the 2.4 GHz block count of §8.5 |
  | 5 | 0 | 25 | 32 448 (26 × 1 248) |
  | 6 | 26 | 52 | 33 696 (27 × 1 248) |
  | 7–10 | `27·t + 0x78` | `27·t + 0x92` (type 10: `0xA2`) | `(last−first+1) × 1 248` |
  | 11 | 0 | 176 | 5 664 |

  The recurring element sizes are **3 276**, **1 248**, **624** and **5 508** bytes. `[C]`
* On disk each type is stored as a separate file whose name is built as
  `<prefix><type-name><index>.bin`. `[C]`
* Chunk copies during the file↔data conversion use element sizes of **0x438 (1 080)**,
  **0x25C (604)** and **16** bytes depending on the block class. `[C]`

### 8.4 Trigger and completion

* **Calibration-type negotiation.** A 20-byte command (sub-type 9) is exchanged with the
  firmware; the response is validated against the magic value **`0xBF`** at byte offset
  12 and returns the firmware's calibration type, its preload flag and a preload version.
  If driver and firmware disagree on calibration type, provisioning is aborted. On a
  production board the firmware **must** report "preload enabled", otherwise the cached
  OCAL cannot be applied. A legacy firmware that does not implement the query is detected
  by the magic mismatch and the command is re-sent in a fire-and-forget form for backward
  compatibility. `[C]`
* **Verification.** A "set one-time-cal data to verify" command (command `0xD6`, header 20
  bytes plus a length field at offset 4) pushes a block for the firmware to check. `[C]`
* **Trigger/stop.** A one-time-calibration start/stop pair with a supervising timer
  (3-tick option) drives the flow; an "external condition" (isolation-detect) start/stop
  pair exists as a separate sub-flow, and both report elapsed duration on completion.
  Commands are queued and executed one at a time. `[C]`
* **Completion** is signalled by firmware events; the in-field flow additionally tracks a
  count of empty responses. `[U]`

### 8.4a The firmware asks the host for calibration data

Calibration provisioning is not purely host-driven. The firmware can request a cached
calibration block from the host at any time after it starts, using the unsolicited event
`ucEID = 0xD7`. `[C]`

| Property | Value |
|---|---|
| Direction | firmware → host, unsolicited (`ucSeqNum = 0`) |
| Minimum body length | **16 bytes**; a shorter event is discarded `[C]` |
| Body | three 32-bit words at body offsets `0x04`, `0x08` and `0x0C` that the host combines into the identifier of the block being requested (composition observed but the field split is not established) `[C]` / `[U]` |
| Required host response | push the named block back with the one-time-calibration data command path (§8.4), in "preload" mode `[C]` |

**The host answers this event.** `[C]` No retry and no error event was found: `[L]` if the
host does not reply, the firmware's preload presumably never completes and the radio runs on
defaults for the rest of the session. The companion event `0xD9` reports the load result and
is the only confirmation the host gets. `[C]`

A driver that treats the calibration cache as write-only — pushing objects at bring-up and
then ignoring the event stream — will provision correctly on a cold boot and then fail
silently after any event-driven re-load (for example after an in-field re-calibration). `[L]`

### 8.5 Calibration block geometry (module-dependent)

Derived at start-up from the module-enable/version byte (§1.2/§2.2) and the chip ID: `[C]`

| Condition | 2.4 GHz band-0 blocks | 2.4 GHz band-1 blocks |
|---|---|---|
| default | 2 | 1 |
| module enabled **and** chip ID ∈ {`0x7932`, `0x7923`} **and** module version == 1 | 4 | 2 |
| module enabled **and** chip ID ∈ {`0x7932`, `0x7923`} **and** module version ≠ 1 | 8 | 4 |

The total 2.4 GHz block count `N` is the sum, and feeds the type-4 cache size in §8.3.
With the shipped image (module enabled, version 9) `N = 12`. `[C]`
This is one of the few places where behaviour is explicitly conditional on the chip ID,
and MT7932 shares it with MT7923. `[C]`

### 8.6 What the host must persist across reboots

`[C]`

* the five Wi-Fi calibration objects (WCal, OCA2, OCAL, IFCAL, runtime) and the
  Bluetooth BCal object, verbatim;
* the in-field-calibration state flags (`saved`, `triggered`, `result`);
* nothing else — the EEPROM image, PPR blob and power tables are static assets that ship
  with the driver.

If the cached objects are absent the part still comes up, but it runs on firmware-default
calibration until a one-time calibration is triggered and its result written back. `[L]`

---

## 9. Regulatory

### 9.1 Setting the country

`[C]`

* The country is an ISO-3166 alpha-2 packed into a 16-bit little-endian word
  (`'X','Z'` → `0x5A58`); a 32-bit form is used where four characters are carried.
* A local regulatory database file (`db.dat`) is consulted first; if the country is not in
  it, the world-wide default (`XZ`/`WW`) is used.
* Setting the country triggers, in order: the domain-info command (§9.2) and then the
  whole TX-power-limit push of §7.7.

### 9.2 Domain-info command

`CMD_ID_SET_DOMAIN_INFO = 0x0F`. Total length = `12 + 8 × n_channels`. `[C]`

| Offset | Size | Field |
|---|---|---|
| 0 | 4 | country code (packed alpha-2/alpha-4) |
| 4 | 2 | regulatory-domain / channel-list index |
| 6 | 1 | 6 GHz channel-list index (written only when 6 GHz is enabled) |
| 7 | 1 | flags: bit0 and bits[7:4] carry a DFS/regulatory-region selector; **bit3 = 6 GHz channel list present** |
| 8 | 1 | number of 2.4 GHz channel entries |
| 9 | 1 | number of 5 GHz channel entries |
| 10 | 1 | number of 6 GHz channel entries (0 when 6 GHz is disabled) |
| 11 | 1 | `0x0F` when the country is US or a US territory (`US, CA, PR, VI, GU, MP, AS, UM`), else 0 |
| 12 | 8×n | channel entries: `u16 channel`, `u16 reserved`, `u32 per-channel attributes` |

The host walks fixed per-band channel arrays of **14** (2.4 GHz), **53** (5 GHz) and
**110** (6 GHz) entries and emits only the entries the current regulatory state marks
active. `[C]`

### 9.3 Regulatory channel matrix (`db.dat`)

Plain ASCII, two sections. `[C]`

```
{Ver:<name> Date:<...>}

<2gCC,2g1..2g14,5g36..5g181,11ac,txbf,vht40,vht80,vht160,ercrs,red,11ax,dsa>
US, 1, 1, ... , 1, 1, 1, 1, 1, 0, 0, 1, 1,
</end>

<6gCC,6g2,6g1,6g3..6g233,vht40,vht80,vht160,c2clpi,vlpp2p,flag>
...
</end>
```

* Per-country rows; per-channel cells hold a small integer or a `/`-separated set
  (e.g. `2/3`, `1/7`, `2/7/3`) encoding the channel's regulatory attributes
  (active/passive/DFS/indoor-only/…). `0` = channel not permitted. `[C]` for the syntax,
  `[L]` for the individual code meanings.
* Per-country capability columns: `11ac`, `txbf`, `vht40`, `vht80`, `vht160`, `ercrs`,
  `red`, `11ax`, `dsa` (2.4/5 GHz section) and `vht40`, `vht80`, `vht160`, `c2clpi`,
  `vlpp2p`, `flag` (6 GHz section). `[C]`
* The 6 GHz column order starts `6g2, 6g1, 6g3, 6g5, …` — channel 2 first, then the odd
  grid. `[C]`

### 9.4 How power limits are communicated

Channel *lists* go out with the domain-info command (§9.2); channel *power limits* go out
with `CMD_ID_SET_COUNTRY_POWER_LIMIT_PER_RATE` (§7.7), which itself carries the alpha-2
so the firmware can cross-check. The per-channel maximum-power figure reported to the
host stack is additionally derived and pushed with the Transmit Power Envelope
sub-command whenever the operating channel changes. `[C]`

---

## 10. Summary of deltas vs public MT7921/MT7922

| Area | MT7932 | Public mt792x |
|---|---|---|
| EEPROM size | 2560 B (`0xA00`) | identical |
| `MT_EE_CHIP_ID`, `MT_EE_VERSION`, `MT_EE_MAC_ADDR`, `MT_EE_WIFI_CONF`, `MT_EE_HW_TYPE` | identical offsets | — |
| EEPROM `0x010`/`0x020` | **PCI identity records for the Wi-Fi (`14C3:7932`) and Bluetooth (`14C3:793B`) functions** | not defined |
| Buffer-bin page encoding | `format[1:0]=1`, `[4:2]`=page, `[7:5]`=pages−1 | identical to public `mt7915` flash mode |
| `ucSourceMode` | 0, 1, **2 (PPR overlay)**, **3 (one-time-cal blob)** | 0, 1 only |
| Per-rate power-limit record | **122 B** (1 + 121) | 162 B (1 + `MT_SKU_POWER_LIMIT`=161) |
| Power-limit command header | 44 B, but bytes 1, 7 and 12…43 carry a flag byte, a sub-table selector and a 32-byte parameter block | same 44 B, those fields are padding |
| Power-limit sub-tables on cmd `0x5D` | per-rate, **SAR**, **SDB**, **2.4 GHz common path**, unsupported-rate bitmap, TPE | per-rate only |
| Limit-table text format | `<Ver:03>` with `s_`/`c_`/`m_` (SISO/CDD/MIMO) column triplication | single `m0…` column group |
| 6 GHz limit table | keyed by `[CC,LP\|VLP\|SP]` | not in upstream |
| Efuse access | ext CID `0x01`, 16-byte blocks; free-block ext CID `0x4F` | identical |
| One-time-calibration cache | 5 object types, TLV, ≤400 KiB working buffer | not in upstream |
| Chip-ID-conditional behaviour | calibration block geometry (`0x7932` and `0x7923` behave alike) | — |

---

## Open questions / needs hardware tracing

1. The meaning of the EEPROM words at `0x045`, `0x049`, `0x04C`, `0x075`, `0x07B`,
   `0x138`, `0x145`, `0x14D`, `0x154`, `0x400`, `0x943` — plausibly crystal trim, PA/LNA
   type, and coexistence settings, but unconfirmed.
2. The trailing 32-bit word `0xA210` of the Wi-Fi PCI identity record.
3. Whether the 2-byte / 6-byte reserved gaps between the VHT and HE areas of the per-rate
   power blocks are true padding or carry a 1SS-delta field.
4. Whether PPR section 3 (6 GHz, 40 bytes) uses the same group layout as sections 1 and 2
   — it is unprovisioned in the shipped build, so the layout cannot be inferred.
5. The exact semantics of the 32-byte auxiliary parameter block at offset 12 of the
   `0x5D` command header, and of the flag bits in byte 1.
6. The per-channel attribute encoding (`u32`) in the domain-info command entries, and the
   numeric codes used in `db.dat` cells (`1`, `2`, `3`, `5`, `7`, and their `/`
   combinations).
7. The exact per-record layout inside a one-time-calibration TLV block (the 3 276 / 1 248 /
   624-byte elements), and which RF quantity each block class holds.
8. What each bitfield within the five calibration-integrity registers of §8.1 means. (The
   five addresses themselves, and the "any word reads back zero ⇒ failure" test, are
   established; only the field semantics are open.)
9. Whether the initialisation-phase single-byte efuse read (init CID `0x50`) is reachable
   on MT7932 or is inherited unused from another chip in the family.
10. Whether `EXT_CMD_ID_EFUSE_BUFFER_RD` (`0x4E`) is implemented by MT7932 firmware.
11. The scaling of the 6 GHz limit values (whether they are conducted-per-chain absolute
    limits, as assumed here, or offsets relative to the standard-power block).
12. The mapping between the 20 SAR sub-bands / 17 SDB sub-bands and actual frequency
    ranges.


---

# MT7932 — MAC/PHY Control Surface and Hardware Tables

## Scope

This document specifies (A) the minimum set of MAC/PHY control operations a host driver must
be able to issue to an MT7932 to bring the radio up, join or run a BSS, scan, encrypt and
filter traffic; and (B) the on-chip tables that back those operations — the station table
(WTBL) in its LMAC and UMAC halves, the key table, the BSS/own-MAC binding, the per-station
rate and counter state, and the MIB/statistics counter block — together with the exact
register-level access mechanism the host uses to reach them. MT7932 presents a CONNAC2
(mt792x-class) MAC. Everything in Part B that is not called out as a delta is bit-for-bit
the same as MT7921/MT7922 and the reader should treat public `mt76` (`mt792x_regs.h`,
`mt7921/`) and `gen4m` (`cmm_asic_connac2x.h`) naming as authoritative. The genuine deltas
are the **table-entry version selection** (MT7932 uses the "v2"/ECO≥2 field layouts
unconditionally) and a small number of **chip-ID-conditional** firmware-command behaviours.

> **Correction.** The WTBL/UWTBL data-unit control register addresses `0x820D_4200` and
> `0x820C_4094` are **not** an MT7932 delta, although the generic CONNAC2 macros in
> `cmm_asic_connac2x.h` are `0x820D_4000` / `0x820C_4000`. Public gen4m already defines
> exactly `MT7961_WIFI_LWTBL_BASE = 0x820d4200` and `MT7961_WIFI_UWTBL_BASE = 0x820c4094`
> for MT7921, and MT7932 uses the same pair. `[C]` The generic macros are what the SoC parts
> use. A driver ported from MT7921 needs no change here.

Confidence markers: `[C]` confirmed, `[L]` likely, `[U]` unverified.

---

## 0. Summary of deltas versus public MT7921/MT7922

| Item | Public MT7921/MT7922 reference | MT7932 |
|---|---|---|
| LMAC WTBL data-unit control register (WDUCR) | **`0x820D_4200`** — public gen4m `MT7961_WIFI_LWTBL_BASE`, i.e. the MT7921 value | **`0x820D_4200`**, identical `[C]`. **Not a delta** (see note below) |
| UMAC WTBL/key data-unit control register (WDUCR) | **`0x820C_4094`** — public gen4m `MT7961_WIFI_UWTBL_BASE` | **`0x820C_4094`**, identical `[C]`. **Not a delta** |
| LMAC WTBL entry size | 33 DW (132 B) | identical, 33 DW `[C]` |
| UMAC WTBL entry size | 8 DW (32 B) | identical, 8 DW `[C]` |
| LWTBL DW3 / DW5 / DW9 field variant | ECO-dependent (`field` vs `field_v2`) | **`field_v2` unconditionally** — `BEAM_CHG`+`BA_MODE`+`ULPF_IDX`/`ULPF`/`IGN_FBK` in DW3, 3-bit `SR_R` and no `BEAM_CHG` in DW5, `PRITX_SW_MODE`/`PRITX_PLR` in DW9 `[C]` |
| Per-station response RCPI | DW29 (pre-ECO2) or DW30 (ECO2+) | **DW30** `[C]` |
| MAC-block chip→BAR fixed map | `mt7921_reg_map` | identical entry-for-entry **for the MAC-block rows** (`0x820C_xxxx` … `0x820F_xxxx`) `[C]`. The map as a whole is *not* identical to public MT7921 — see §4.1 of the PCIe/register-map section, which lists four changed or new slots |
| Command set | `mt76` unified/CE split | legacy (non-unified) `gen4m` CONNAC2 command set; **no UNI_CMD framing at all** `[C]` |
| DBDC enable command body | n/a | field at byte 4 is `0` only when the PCI device ID is `0x7922`; for `0x7932` (and `0x7923`) it is `2` `[C]` — a real device-ID-conditional divergence |
| Additional command IDs not in public `gen4m` enum | — | `0x88` (statistics-item query), `0x99` (all-station query), `0x9D` (SDB channel-group info) `[C]` |
| 802.11be / 320 MHz / preamble puncturing | not present | **not present** — the channel/bandwidth and station-record command bodies carry no EHT or puncturing fields `[C]` |
| WTBL entry count | `MT792x_WTBL_SIZE` = 20 (mt76 software limit) | reported by firmware in the NIC-capability response; the accepted range is **1 .. 49** `[C]` |
| Station-record index space | — | **15** indices; 0–1 reserved for one special network type, 2–14 for peers `[C]` |
| Hardware BSS contexts | 4 | **4**, plus one pseudo-index equal to the reported BSS count used for the P2P-device role `[C]` |

---

# Part A — Minimum MAC/PHY control operations

## A.0 Framing and completion conventions

All MAC/PHY control on MT7932 is performed by **MCU commands**, not by host register writes.
The host has direct MMIO access to the MAC sub-blocks (see §B.1) and uses it for
table inspection and debug, but every operation in this section is expressed as a command.

Every command is submitted with five control attributes `[C]`:

| Attribute | Meaning |
|---|---|
| command ID (8 bit) | selects the operation (tables below) |
| set / query (1 bit) | `1` = set, `0` = query |
| needs-response (1 bit) | `1` = firmware must return a response event carrying the same sequence number |
| is-OID (1 bit) | the request is synchronous: the issuer waits for the response and must be failed on timeout |
| payload length | fixed per command ID (tables below) |

Completion rules `[C]`:

* **set, needs-response = 0** — fire-and-forget. The only completion signal is consumption of
  the descriptor from the MCU command ring. Almost all of the BSS/STA/channel/EDCA commands
  are issued this way.
* **query, needs-response = 1** — firmware returns an event whose event ID equals the command
  ID for the `0x80`-and-above query range (e.g. command `0xCD` → event `0xCD`), or a dedicated
  event ID for the low range (e.g. command `0x82` → `EVENT_ID_STATISTICS` = `0x03`,
  command `0x85` → `EVENT_ID_STA_STATISTICS` = `0x21`). The sequence number matches the request.
* **set, is-OID = 1** — the command completes when the firmware acknowledges it; otherwise it
  must be failed when the per-command timeout expires.
* Asynchronous state changes are reported by unsolicited events (§A.17.2).

Recommended ordering for a cold bring-up `[L]`:

```
firmware download / MCU handshake        (outside this document's scope)
CMD 0x04  NIC_POWER_CTRL(mode = 0)       -> radio subsystem on
CMD 0x8A  GET_NIC_CAPABILITY_V2 (query)  -> learn BSS count / WTBL count / WMM sets / bands
CMD 0x28  SET_DBDC_PARMS                 -> band/DBDC configuration
CMD 0x0F  SET_DOMAIN_INFO (+0x5B/0x5D)   -> regulatory domain and power limits
CMD 0x11  BSS_ACTIVATE_CTRL(active=1)    -> allocate BSS context + own-MAC (MUAR) slot
CMD 0x19  SET_BSS_RLM_PARAM              -> channel / bandwidth
CMD 0x12  SET_BSS_INFO                   -> BSSID, operating mode, rates, security mode
CMD 0x1D  UPDATE_WMM_PARMS               -> EDCA
CMD 0x13  UPDATE_STA_RECORD              -> peer station entries
CMD 0x07  ADD_REMOVE_KEY                 -> keys
CMD 0x0A  SET_RX_FILTER                  -> RX filter
```

---

## A.1 Radio start / stop, per-band enable

### A.1.1 Radio (NIC) power control

| Field | Value |
|---|---|
| Command ID | `0x04` (`CMD_ID_NIC_POWER_CTRL`) `[C]` |
| Direction | set, needs-response = 0 `[C]` |
| Payload | 4 bytes `[C]` |
| Payload layout | `[0] = ucPowerMode` (0 = radio/MAC on, 1 = off), `[1..3]` reserved, zero `[C]` |

This command may be written straight into the command ring, bypassing any host-side queue,
so it can be issued while that queue is quiesced; a free ring slot must be waited for
(5 s bound, 5 ms poll granularity — host choice) `[C]`. No confirmation event is produced —
completion is ring consumption `[C]`. **No host-writable PHY on/off register is used**; the
radio is started and stopped through this command plus per-BSS activation (§A.5) `[C]`
(absence). That no such register exists is `[L]`.

### A.1.2 Per-band / DBDC enable

| Field | Value |
|---|---|
| Command ID | `0x28` (`CMD_ID_SET_DBDC_PARMS`) `[C]` |
| Direction | set, needs-response = 0 `[C]` |
| Payload | 36 bytes (`0x24`) `[C]` |

Observed body layout `[C]`:

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | DBDC enable (0 = disable, non-zero = enable) |
| `0x01` | 1 | WMM-set bitmap for the band being enabled. Set to `1` when the primary BSS is on the 2.4 GHz band, otherwise `((1 << ucWmmSetNum) - 1) & 0xFE` |
| `0x04` | 1 | mode/action selector — **`0` if the PCI device ID is `0x7922`, `2` otherwise (MT7932 → `2`)** |
| `0x05` | 1 | secondary-enable flag |
| `0x06` | 2 | `0x0024` (constant) |
| `0x08` | 2 | reserved/state word |
| `0x0A` | 1 | reserved/state byte |
| `0x0C` | 2 | OR of the per-BSS TX-queue bitmaps of all active BSSes currently on the 2.4 GHz band (populated only on the non-`0x7922` path) |
| `0x0E..0x23` | 22 | zero |

This is the only place in the MAC control surface where behaviour is conditional on the PCI
device ID. On MT7932 the `2` selector is taken and the 2.4 GHz TX-queue bitmap word at `0x0C`
is populated; on MT7922 neither happens `[C]`. The practical effect on silicon needs tracing `[U]`.

An adjacent command exists for sub-band/channel-group (SDB) selection:

| Field | Value |
|---|---|
| Command ID | `0x9D` (SDB channel-group info) `[C]` |
| Payload | 2 bytes: `[0]` = channel group, `[1]` = smart-CCA mode `[C]` |
| Notes | Not present in the public `gen4m` command enumeration. Failure is reported but non-fatal `[C]`. |

---

## A.2 Set operating channel

| Field | Value |
|---|---|
| Command ID | `0x19` (`CMD_ID_SET_BSS_RLM_PARAM`) `[C]` |
| Direction | set, needs-response = 0 `[C]` |
| Payload | 22 bytes (`0x16`) `[C]` |

Body (2-byte aligned, exactly the public CONNAC2 `CMD_SET_BSS_RLM_PARAM`) `[C]`:

| Offset | Size | Field | Encoding |
|---|---|---|---|
| `0x00` | 1 | `ucBssIndex` | 0 .. `ucHwBssIdNum` |
| `0x01` | 1 | `ucRfBand` | 1 = 2.4 GHz, 2 = 5 GHz, 3 = 6 GHz `[L]` |
| `0x02` | 1 | `ucPrimaryChannel` | IEEE channel number in the band |
| `0x03` | 1 | `ucRfSco` | secondary-channel offset: 0 = none (SCN), 1 = above (SCA), 3 = below (SCB) `[L]` |
| `0x04` | 1 | `ucErpProtectMode` | |
| `0x05` | 1 | `ucHtProtectMode` | |
| `0x06` | 1 | `ucGfOperationMode` | |
| `0x07` | 1 | `ucTxRifsMode` | |
| `0x08` | 2 | `u2HtOpInfo3` | |
| `0x0A` | 2 | `u2HtOpInfo2` | |
| `0x0C` | 1 | `ucHtOpInfo1` | |
| `0x0D` | 1 | `ucUseShortPreamble` | |
| `0x0E` | 1 | `ucUseShortSlotTime` | |
| `0x0F` | 1 | `ucVhtChannelWidth` | see §A.3 |
| `0x10` | 1 | `ucVhtChannelFrequencyS1` | centre-frequency segment 0, channel number |
| `0x11` | 1 | `ucVhtChannelFrequencyS2` | centre-frequency segment 1 (80+80 only) |
| `0x12` | 2 | `u2VhtBasicMcsSet` | |
| `0x14` | 1 | `ucTxNss` | 1..2 |
| `0x15` | 1 | `ucRxNss` | 1..2 |

Ordering: the BSS index must already have been activated (§A.5) before the channel is set,
because the RLM body is addressed by BSS index and the firmware binds the channel to the BSS
context, not to a global "current channel". The same 22-byte block is also embedded verbatim
at offset `0x44` inside the `SET_BSS_INFO` body (§A.5), so a combined
"activate → set channel → set BSS info" sequence sets it twice; that is harmless `[C]`.

Completion: none (fire-and-forget). A channel actually becoming usable is observable
indirectly through `EVENT_ID_CH_PRIVILEGE` (`0x10`) when the channel-privilege/grant
mechanism (`CMD_ID_CH_PRIVILEGE` = `0x1C`) is used to arbitrate the radio between roles `[L]`.

---

## A.2a Channel-privilege handshake (firmware concurrency manager)

Setting a channel (§A.2) programs the RF for a BSS; it does **not** give that BSS the right
to use the radio. When more than one role is active — or for any off-channel excursion (scan,
remain-on-channel, an off-channel action frame, a channel-switch dwell) — the radio is
arbitrated by a concurrency manager inside the firmware, and the host takes part in a
**request / grant / abort** handshake `[C]`. Ignoring it is expected to leave the radio
assigned to a role that has finished with it, starving every other role until the next reset
`[L]` — that consequence is inferred from the handshake, not observed.

**Request / abort — `CMD_ID_CH_PRIVILEGE` (`0x1C`), 24-byte body** `[C]`. Layout is
byte-for-byte the public `CMD_CH_PRIVILEGE`:

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | `ucBssIndex` |
| `0x01` | 1 | `ucTokenID` — **host-allocated**, see below |
| `0x02` | 1 | `ucAction`: `0` = request, `1` = abort |
| `0x03` | 1 | `ucPrimaryChannel` |
| `0x04` | 1 | `ucRfSco` |
| `0x05` | 1 | `ucRfBand` |
| `0x06` | 1 | `ucRfChannelWidth` |
| `0x07` | 1 | `ucRfCenterFreqSeg1` |
| `0x08` | 1 | `ucRfCenterFreqSeg2` |
| `0x09` | 1 | `ucReqType` — purpose of the request (join, scan, off-channel TX, …). **An out-of-range value makes the firmware assert**, §6.1 of the MCU-protocol section `[C]` |
| `0x0A` | 1 | `ucDBDCBand` |
| `0x0B` | 1 | reserved |
| `0x0C` | 4 | `u4MaxInterval` — requested duration, milliseconds |
| `0x10` | 8 | reserved |

The command is fire-and-forget (no response is matched to it); the grant arrives as a
separate unsolicited event.

**Grant / revoke — `EVENT_ID_CH_PRIVILEGE` (`0x10`), 24-byte body** `[C]`, the public
`EVENT_CH_PRIVILEGE`:

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | `ucBssIndex` — **range-checked by the host against the BSS-context count; out of range is discarded** |
| `0x01` | 1 | `ucTokenID` — echoes the request |
| `0x02` | 1 | `ucStatus`: `0` = grant, `1` = reject, `2` = recover, `3` = remove-request |
| `0x03` … `0x0A` | 8 | channel description, same fields and order as the request |
| `0x0C` | 4 | `u4GrantInterval` — granted duration, milliseconds (may be shorter than requested) |

Body shorter than 24 bytes ⇒ the event is discarded. `[C]`

**Rules the host must obey** `[C]`:

1. **The token space is the host's.** `ucTokenID` is allocated by the host and only echoed
   by the firmware. Allocate monotonically; a grant whose token is **older than or equal to**
   the last token the host abandoned must be discarded, because the firmware may still be
   answering a request the host has already cancelled.
2. **A grant is mandatory before use.** The host must not begin the off-channel activity
   until the grant for its own token arrives. A non-grant status
   (`1`/`2`/`3`) at this point in the flow must be treated as a hard error.
3. **Every granted request must be aborted.** When the activity finishes — normally,
   on failure, or because the grant interval expired — the host must send the same command
   with `ucAction = 1` and the **same BSS index and token**. If a matching request is still
   queued host-side it is cancelled locally instead; if it has already been sent, the abort
   must go on the wire. `[C]` No firmware-side timeout that reclaims a forgotten grant was
   found — `[L]`.
4. **The grant expires.** `u4GrantInterval` is treated as a deadline, not a hint; the firmware
   is expected to revoke the channel when it elapses (status `3`, remove-request) and the host
   must then stop using it immediately. `[C]` for the field and the four status values, `[L]`
   for the revocation behaviour.
5. **The BSS must be active before the request.** If the target BSS context is not yet
   activated, the host activates it (and issues the network-activation command) as part of
   the request path.
6. Only one request per (BSS index, token) may be outstanding.

Because a scan is an off-channel activity, the scan-request / scan-done flow (§A.4) sits
inside a channel-privilege request/abort pair; the abort is part of scan teardown and must
be issued even when the scan is aborted or fails. `[L]` — the arbitration mechanism itself
is `[C]`, its use by the scan path specifically is inferred.

---

## A.3 Bandwidth and puncturing

Bandwidth is carried in `ucVhtChannelWidth` of the RLM parameter block (§A.2) together with
`ucRfSco` and the two centre-frequency segment fields `[C]`:

| `ucVhtChannelWidth` | Meaning | Additional fields |
|---|---|---|
| 0 | 20 MHz or 40 MHz | 40 MHz selected by `ucRfSco` = 1/3; S1 = 0 |
| 1 | 80 MHz | S1 = 80 MHz centre channel |
| 2 | 160 MHz | S1 = 160 MHz centre channel |
| 3 | 80+80 MHz | S1, S2 = the two 80 MHz centres |

**Maximum bandwidth expressible here is 160 MHz (or 80+80).** There is no 320 MHz encoding and
**no channel puncturing / preamble-puncturing field anywhere in the MAC control surface** —
neither in the RLM parameter block, nor in the station-record body, nor in the monitor-mode
body `[C]` (confirmed negative over those command bodies). That the *silicon* cannot exceed
160 MHz follows from this plus the chip-delta section's PHY-generation analysis, `[L]`.
Per-station bandwidth capability is limited independently by the LMAC WTBL `FCAP` field
(§B.2, DW9: bit0 = 20/40 MHz capable, bit1 = 20-to-80 MHz, otherwise 20-to-160 MHz) `[C]`.

---

## A.4 Scan

### A.4.1 Scan request

| Field | Value |
|---|---|
| Command ID | `0x03` (`CMD_ID_SCAN_REQ_V2`) `[C]` |
| Direction | set, needs-response = 0 `[C]` |
| Payload | **1236 bytes (`0x4D4`)** `[C]` |

The legacy `CMD_ID_SCAN_REQ` (`0x1A`) is present but deprecated and must not be used `[C]`.
Body layout `[C]`:

| Offset | Size | Field |
|---|---|---|
| `0x000` | 1 | `ucSeqNum` — echoed in the scan-start/scan-done events |
| `0x001` | 1 | `ucBssIndex` |
| `0x002` | 1 | `ucScanType` (passive / active / …) |
| `0x003` | 1 | `ucSSIDType` |
| `0x004` | 1 | `ucSSIDNum` (0..4 for the primary list) |
| `0x005` | 1 | `ucNumProbeReq` — probe requests per channel per SSID |
| `0x006` | 1 | `ucScnFuncMask` (see below) |
| `0x007` | 1 | version |
| `0x008` | 144 | `arSSID[4]` — 4 × { u32 length, 32-byte SSID } |
| `0x098` | 2 | `u2ProbeDelayTime` (ms) |
| `0x09A` | 2 | `u2ChannelDwellTime` (ms) — **max per-channel dwell** |
| `0x09C` | 2 | `u2TimeoutValue` (ms) — whole-scan timeout |
| `0x09E` | 1 | `ucChannelType` (full / 2.4-only / 5-only / specified) |
| `0x09F` | 1 | `ucChannelListNum` (0..32) |
| `0x0A0` | 64 | `arChannelList[32]` — 2 bytes each: `{ ucBand, ucChannelNum }` |
| `0x0E0` | 2 | `u2IELen` |
| `0x0E2` | 600 | `aucIE[600]` — **probe-request IE template** appended to every probe request |
| `0x33A` | 1 | `ucChannelListExtNum` (0..32) |
| `0x33B` | 1 | `ucSSIDExtNum` (0..6) |
| `0x33C` | 2 | `u2ChannelMinDwellTime` (ms) — **min per-channel dwell** |
| `0x33E` | 64 | `arChannelListExtend[32]` |
| `0x37E` | 2 | padding to 4-byte alignment |
| `0x380` | 216 | `arSSIDExtend[6]` — 6 × { u32 length, 32-byte SSID } |
| `0x458` | 6 | `aucBSSID` — directed-scan BSSID |
| `0x45E` | 6 | `aucRandomMac` — source address for probe requests |
| `0x464` | 60 | `aucExtBSSID[10][6]` — **10** out-of-band/6 GHz RNR BSSID slots |
| `0x4A0` | 1 | `ucShortSSIDNum` |
| `0x4A1` | 10 | `ucBssidMatchCh[10]` |
| `0x4AB` | 10 | `ucBssidMatchSsidInd[10]` |
| `0x4B5` | 3 | padding |
| `0x4B8` | 4 | `u4ScnFuncMaskExtend` |
| `0x4BC` | 1 | `ucScnSourceMask` |
| `0x4BD` | 2 | `u2OpChStayTimeMs` — time to return to the operating channel between scan hops |
| `0x4BF` | 1 | `ucDfsChDwellTimeMs` |
| `0x4C0` | 1 | `ucPerScanChannelCnt` — channels visited before returning to the operating channel |
| `0x4C1` | 19 | padding; when the "use padding as BSSID" function bit is set, bytes `0x4C4..0x4C9` carry an additional 6-byte address `[C]` |

`ucScnFuncMask` bits `[C]` (identical to public `gen4m`):

| Bit | Meaning |
|---|---|
| 0 | randomised MAC in use (`aucRandomMac` valid) |
| 1 | disable DBDC scan |
| 2 | DBDC scan type 3 |
| 3 | interpret the trailing padding as an extra BSSID |
| 4 | randomise probe-request sequence numbers |
| 5 | split scan |
| 6 | fence scan |
| 7 | OCE scan |

The probe-request **template is supplied inline** in `aucIE`; there is no separate template
command for scanning. Dwell configuration is per scan, not per channel: `u2ChannelMinDwellTime`
/ `u2ChannelDwellTime` bound each channel visit, `ucDfsChDwellTimeMs` overrides for DFS
channels, and `ucPerScanChannelCnt` / `u2OpChStayTimeMs` control interleaving with the
operating channel `[C]`.

### A.4.2 Scan abort

| Field | Value |
|---|---|
| Command ID | `0x1B` (`CMD_ID_SCAN_CANCEL`) `[C]` |
| Payload | 4 bytes: `[0] = ucSeqNum` (must match the outstanding request), `[1] = ucIsExtChannel`, `[2..3]` reserved `[C]` |

### A.4.3 Scan events

| Event | ID | Contents |
|---|---|---|
| Scan start (per channel) | — | sequence number, channel number, scan type, scan source `[C]` |
| Scan done | `0x0D` (`EVENT_ID_SCAN_DONE`) | sequence number, request type, scan type, source, completion state, reason code, "hit" count, channel count `[C]` |
| Scheduled-scan done | `0x23` (`EVENT_ID_SCHED_SCAN_DONE`) | `[C]` |

Beacons and probe responses arrive as ordinary RX frames; the scan-done event is the sole
completion signal. Aborting produces the same scan-done event with a cancel reason `[C]`.

### A.4.4 Scheduled scan

| Operation | Command ID |
|---|---|
| Enable/disable scheduled scan | `0x61` (`CMD_ID_SET_SCAN_SCHED_ENABLE`) `[L]` |
| Scheduled-scan request | `0x62` (`CMD_ID_SET_SCAN_SCHED_REQ`) `[L]` |

---

## A.5 Create / delete a BSS context

Two commands are involved and the order matters.

### A.5.1 BSS activation (context allocation + own-MAC binding)

| Field | Value |
|---|---|
| Command ID | `0x11` (`CMD_ID_BSS_ACTIVATE_CTRL`) `[C]` |
| Direction | set, needs-response = 0 `[C]` |
| Payload | 12 bytes `[C]` |

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | `ucBssIndex` |
| `0x01` | 1 | `ucActive` — 1 = create/activate, 0 = delete/deactivate |
| `0x02` | 1 | `ucNetworkType` — role (infra STA / AP / P2P / NAN / …) |
| `0x03` | 1 | `ucOwnMacAddrIndex` — **index into the on-chip own-MAC (MUAR) table**; also becomes the `MUAR_Idx` written into the LMAC WTBL entries of that BSS |
| `0x04` | 6 | `aucBssMacAddr` — the BSS's own MAC address |
| `0x0A` | 1 | `ucBMCWlanIndex` — WTBL index reserved for this BSS's broadcast/multicast (group-key) traffic |
| `0x0B` | 1 | `ucMldLinkIdx` — 1 on activation, 1 on deactivation `[C]` |

Ordering constraints `[C]`:

* The broadcast/multicast WTBL index at `0x0A` must be allocated **before** the command is
  sent (see §B.2.5 for the allocation rule); group keys may only be programmed after
  activation.
* All pending TX for the BSS index is flushed before activation and before deactivation.
* Deactivation is followed by: release of the BSS's BC/MC WTBL entry, release of all station
  records bound to the BSS, flush of the BSS's absence queue, and release of the WMM-set index.
* `ucBssIndex` values `0 .. ucHwBssIdNum-1` map to hardware BSS contexts. The index equal to
  `ucHwBssIdNum` is a **pseudo BSS** used for the P2P-device role and has no hardware context `[C]`.

### A.5.2 BSS parameter programming

| Field | Value |
|---|---|
| Command ID | `0x12` (`CMD_ID_SET_BSS_INFO`) `[C]` |
| Direction | set, needs-response = 0 `[C]` |
| Payload | **116 bytes (`0x74`)** `[C]` |

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | `ucBssIndex` |
| `0x01` | 1 | `ucConnectionState` |
| `0x02` | 1 | `ucCurrentOPMode` (infra / AP / IBSS / …) |
| `0x03` | 1 | `ucSSIDLen` (≤ 32) |
| `0x04` | 32 | `aucSSID` |
| `0x24` | 6 | `aucBSSID` |
| `0x2A` | 1 | `ucIsQBSS` |
| `0x2B` | 1 | `ucVersion` |
| `0x2C` | 2 | `u2OperationalRateSet` |
| `0x2E` | 2 | `u2BSSBasicRateSet` |
| `0x30` | 1 | `ucStaRecIdxOfAP` |
| `0x31` | 1 | reserved |
| `0x32` | 2 | `u2HwDefaultFixedRateCode` |
| `0x34` | 1 | `ucNonHTBasicPhyType` (drives slot time and CWmin) |
| `0x35` | 1 | `ucAuthMode` |
| `0x36` | 1 | `ucEncStatus` |
| `0x37` | 1 | `ucPhyTypeSet` |
| `0x38` | 1 | `ucWapiMode` |
| `0x39` | 1 | `ucIsApMode` |
| `0x3A` | 1 | `ucBMCWlanIndex` |
| `0x3B` | 1 | `ucHiddenSsidMode` |
| `0x3C` | 1 | `ucDisconnectDetectThreshold` |
| `0x3D` | 1 | `ucIotApAct` |
| `0x3E` | 2 | reserved |
| `0x40` | 4 | `u4PrivateData` |
| `0x44` | 22 | embedded `CMD_SET_BSS_RLM_PARAM` (§A.2) |
| `0x5A` | 1 | `ucDBDCBand` |
| `0x5B` | 1 | `ucWmmSet` |
| `0x5C` | 1 | `ucDBDCAction` |
| `0x5D` | 1 | `ucNss` |
| `0x5E` | 2 | reserved |
| `0x60` | 3 | `ucHeOpParams[3]` |
| `0x63` | 1 | `ucBssColorInfo` |
| `0x64` | 2 | `u2HeBasicMcsSet` |
| `0x66` | 1 | `ucMaxBSSIDIndicator` |
| `0x67` | 1 | `ucMBSSIDIndex` |
| `0x68` | 12 | padding |

Setting `ucConnectionState` to the "disconnected" value additionally causes the host to
re-initialise the BSS client list, free all station records of the BSS and flush its queues `[C]`.

---

## A.6 Create / delete a station entry

### A.6.1 Create / update

| Field | Value |
|---|---|
| Command ID | `0x13` (`CMD_ID_UPDATE_STA_RECORD`) `[C]` |
| Direction | set; needs-response is caller-selectable (used for the transition into the associated state) `[C]` |
| Payload | **200 bytes (`0xC8`)** `[C]` |

Leading fields (public CONNAC2 `CMD_UPDATE_STA_RECORD`) `[C]`:

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | `ucStaIndex` — host station-record index (0..14) |
| `0x01` | 1 | `ucStaType` |
| `0x02` | 6 | `aucMacAddr` — fixed at creation, must not change on update |
| `0x08` | 2 | `u2AssocId` |
| `0x0A` | 2 | `u2ListenInterval` |
| `0x0C` | 1 | `ucBssIndex` — fixed at creation |
| `0x0D` | 1 | `ucDesiredPhyTypeSet` |
| `0x0E` | 2 | `u2DesiredNonHTRateSet` |
| `0x10` | 2 | `u2BSSBasicRateSet` |
| `0x12` | 1 | `ucIsQoS` |
| `0x13` | 1 | `ucIsUapsdSupported` |
| `0x14` | 1 | `ucStaState` — 0/1/2 (see §A.7) |
| `0x15` | 1 | `ucMcsSet` |
| `0x16` | 1 | `ucSupMcs32` |
| `0x17` | 1 | `ucVersion` |
| `0x18` | 10 | `aucRxMcsBitmask` |
| `0x22` | 2 | `u2RxHighestSupportedRate` |
| `0x24` | 4 | `u4TxRateInfo` |
| `0x28` | 2 | `u2HtCapInfo` |
| `0x2A` | 2 | `u2HtExtendedCap` |
| `0x2C` | 4 | `u4TxBeamformingCap` |
| `0x30` | 1 | `ucAmpduParam` |
| `0x31` | 1 | `ucAselCap` |
| `0x32` | 1 | `ucRCPI` |
| `0x33` | 1 | `ucNeedResp` |
| `0x34` | 1 | `ucUapsdAc` (b0–3 trigger-enabled, b4–7 delivery-enabled) |
| `0x35` | 1 | `ucUapsdSp` (0 = all, 1 = max 2, 2 = max 4, 3 = max 6) |
| `0x36` | 1 | **`ucWlanIndex` — the hardware WTBL index; fixed at creation** |
| `0x37` | 1 | `ucBMCWlanIndex` — fixed at creation |
| `0x38` | 4 | `u4VhtCapInfo` |
| `0x3C` | 2 | `u2VhtRxMcsMap` |
| `0x3E` | 2 | `u2VhtRxHighestSupportedDataRate` |
| `0x40` | 2 | `u2VhtTxMcsMap` |
| `0x42` | 2 | `u2VhtTxHighestSupportedDataRate` |
| `0x44` | 1 | `ucRtsPolicy` (0 auto, 1 static BW, 2 dynamic BW, 3 legacy, 7 no-RTS) |
| `0x45` | 1 | `ucVhtOpMode` — VHT operating mode: bit 7 = RX-NSS type, bits 6:4 = RX NSS, bits 1:0 = channel width |

Remaining fields (`0x46` … `0xC7`). The whole 200-byte body is populated; the offsets below
are `[C]` and the field *names* follow public CONNAC2 (`CMD_UPDATE_STA_RECORD`) `[L]` where
noted.

| Offset | Size | Field |
|---|---|---|
| `0x46` | 1 | `ucTrafficDataType` |
| `0x47` | 1 | `ucTxGfMode` — HT greenfield: 0 = disabled, 1 = enabled, 2 = force-enabled |
| `0x48` | 1 | `ucTxSgiMode` — short GI, same 0/1/2 encoding |
| `0x49` | 1 | `ucTxStbcMode` — same encoding (not used on this part) |
| `0x4A` | 2 | `u2HwDefaultFixedRateCode` — 14-bit rate word (§B.6 encoding) used when no rate adaptation applies |
| `0x4C` | 1 | `ucTxAmpdu` — TX A-MPDU enable |
| `0x4D` | 1 | `ucRxAmpdu` — RX A-MPDU enable |
| `0x4E` | 2 | padding (alignment), zero |
| `0x50` | 4 | `u4FixedPhyRate` — `0` = disabled; otherwise bit 31 = 1 with the rate code in bits 15:0 |
| `0x54` | 2 | `u2MaxLinkSpeed` — `0` = unlimited, else units of 0.5 Mb/s |
| `0x56` | 2 | `u2MinLinkSpeed` — same units |
| `0x58` | 4 | `u4Flags` — vendor/"synergy" flag word, copied verbatim from the host station record `[U]` |
| `0x5C` | 4 | **`rBaSize` — block-ack window sizes, a union whose interpretation follows the peer's PHY** `[C]`: for an HE peer, `u16 TxBaSize` at `0x5C` and `u16 RxBaSize` at `0x5E`; for an HT/VHT peer, `u8 TxBaSize` at `0x5C`, `u8 RxBaSize` at `0x5D`, 2 bytes reserved |
| `0x60` | 2 | `u2PfmuId` — beamforming profile-matrix-unit index; **`0xFFFF` = no PFMU allocated** |
| `0x62` | 1 | `fgSU_MU` — 0 = SU beamforming, 1 = MU |
| `0x63` | 1 | `fgETxBfCap` — 0 = implicit TxBF, 1 = explicit TxBF |
| `0x64` | 1 | `ucSoundingPhy` |
| `0x65` | 1 | `ucNdpaRate` |
| `0x66` | 1 | `ucNdpRate` |
| `0x67` | 1 | `ucReptPollRate` |
| `0x68` | 1 | `ucTxMode` |
| `0x69` | 1 | `ucNc` — number of columns (sounding) |
| `0x6A` | 1 | `ucNr` — number of rows (sounding) |
| `0x6B` | 1 | `ucCBW` — sounding bandwidth: 0 = 20, 1 = 40, 2 = 80, 3 = 80+80 |
| `0x6C` | 1 | `ucTotMemRequire` — total PFMU memory required |
| `0x6D` | 1 | `ucMemRequire20M` |
| `0x6E`…`0x75` | 8 | `ucMemRow0/ucMemCol0` … `ucMemRow3/ucMemCol3` — PFMU memory allocation, four row/column pairs |
| `0x76` | 2 | `u2SmartAnt` |
| `0x78` | 1 | `ucSEIdx` — spatial-extension index for this peer |
| `0x79` | 1 | `uciBfTimeOut` |
| `0x7A` | 1 | `uciBfDBW` |
| `0x7B` | 1 | `uciBfNcol` |
| `0x7C` | 1 | `uciBfNrow` |
| `0x7D` | 1 | reserved in public CONNAC2; **must be written as `1`** `[C]` / meaning `[U]` |
| `0x7E` | 1 | `ucMlrMode` |
| `0x7F` | 1 | `ucMlrState` — only meaningful when MLR is in use |
| `0x80` | 1 | `ucTxAmsduInAmpdu` |
| `0x81` | 1 | `ucRxAmsduInAmpdu` |
| `0x82` | 1 | padding, zero |
| `0x83` | 1 | reserved in public CONNAC2; **carries a per-station byte on this part** `[C]` / meaning `[U]` |
| `0x84` | 4 | `u4TxMaxAmsduInAmpduLen` — maximum A-MSDU-in-A-MPDU length in octets (8192 is a workable host value) |
| `0x88` | 6 | `ucHeMacCapInfo[6]` — the peer's HE MAC Capabilities Information field, verbatim |
| `0x8E` | 11 | `ucHePhyCapInfo[11]` — the peer's HE PHY Capabilities Information field, verbatim |
| `0x99` | 3 | padding (2 declared + 1 alignment), zero |
| `0x9C` | 2 | `u2HeRxMcsMapBW80` |
| `0x9E` | 2 | `u2HeTxMcsMapBW80` |
| `0xA0` | 2 | `u2HeRxMcsMapBW160` |
| `0xA2` | 2 | `u2HeTxMcsMapBW160` |
| `0xA4` | 2 | `u2HeRxMcsMapBW80P80` |
| `0xA6` | 2 | `u2HeTxMcsMapBW80P80` |
| `0xA8` | 2 | `u2He6gBandCapInfo` — the peer's HE 6 GHz Band Capabilities element |
| `0xAA` | 30 | padding, zero (`0xAA`–`0xC7`) |

Two structural consequences `[C]`:

* **`ucVersion` (`0x17`) is always written as `1`.** Version 1 is what selects the
  802.11ax-extended body — i.e. the fields from `0x88` onward exist only when version 1 is
  declared. A version-0 body would end at `0x88`.
* **The body is exactly 200 bytes with the HE block present and no 802.11be block.** The
  CONNAC2 802.11be variant of this structure appends EHT MAC/PHY capability bytes and
  20/80/160/320 MHz MCS maps after `0x88`+, which would make the body longer than 200 bytes.
  The 200-byte body length is therefore independent confirmation that this part has no
  EHT support (§0, and the PHY-generation analysis in the chip-delta section).

Other encodings referenced by the body `[L]` (public CONNAC2):

* `ucStaType` (`0x01`) is a **bitmask**, not an enumeration: bits [3:0] select the network
  type (bit 0 = legacy/infrastructure, bit 1 = P2P, bit 2 = BT-over-Wi-Fi, bit 3 = NAN) and
  bits [7:4] select the peer's role (bit 4 = ad-hoc peer, bit 5 = client, bit 6 = AP,
  bit 7 = DLS/TDLS peer). A legacy infrastructure AP is therefore `0x41`.
* `ucUapsdAc` (`0x34`) packs delivery-enabled in bits [7:4] and trigger-enabled in
  bits [3:0], i.e. `trigger | (delivery << 4)` `[C]`.
* `ucBMCWlanIndex` (`0x37`) is written as **`0xFF`** on every station-record
  update — the peer's own WTBL row carries its keys, and the BSS's broadcast/multicast row
  is bound through `BSS_ACTIVATE_CTRL` instead (§A.5.1) `[C]`.
* `ucRtsPolicy` (`0x44`) is derived rather than configured directly:
  from two per-adapter bandwidth-mode bytes, yielding `3` (legacy) when the first is outside
  1..2, `2` (dynamic bandwidth) when both are in 1..2, and `1` (static bandwidth)
  otherwise `[C]`.

The station index and the WTBL index are **separate namespaces**. The station index selects a
host/firmware record; `ucWlanIndex` selects the hardware WTBL row that the LMAC uses for
address matching, cipher, rate and aggregation state (§B.2). Both are allocated at creation
and must not change for the life of the entry `[C]`.

### A.6.2 Delete

| Field | Value |
|---|---|
| Command ID | `0x14` (`CMD_ID_REMOVE_STA_RECORD`) `[C]` |
| Payload | 5 bytes `[C]` |

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | `ucActionType` — **what to remove**: `0` = this one station record; `1` = every station record of the BSS; `2` = every station record of the BSS *except* the one named in `ucStaIndex` `[C]` value / `[L]` names |
| `0x01` | 1 | `ucStaIndex` — host station-record index (0..14); ignored for action type 1 `[C]` |
| `0x02` | 1 | `ucBssIndex` `[C]` |
| `0x03` | 1 | response-requested flag; set only when the named station record is in the associated state `[C]` |
| `0x04` | 1 | reserved, written 0 `[C]` |

Note the field order: the **action type comes first and the station index second** — the
reverse of the `UPDATE_STA_RECORD` body, which starts with the station index. `[C]`

Notes `[C]`: needs-response and is-OID are both set to the same value, and that value is
derived from the station's own state, so a delete of an associated peer is synchronous while
a delete of an unassociated one is fire-and-forget. A station index ≥ 15 is accepted but
never generates a completion.

### A.6.3 Index allocation

* **Station records**: pool of 15 `[C]`. Indices 0–1 are reserved for one special network
  type; regular peers are allocated sequentially from index 2 upward, first free wins `[C]`.
  This is a **host-side software pool** and is unrelated to the hardware WTBL depth (which the
  firmware reports, accepted range 1..49) — do not read 15 as a silicon limit.
  On allocation the record must be zeroed, a TX descriptor template prepared, a WTBL index
  reserved (§B.2.5) and an initial `UPDATE_STA_RECORD` issued `[C]`.
* **WTBL indices**: see §B.2.5.

---

## A.6a Host-owned index spaces, firmware credits, and teardown

### A.6a.1 Index spaces the host allocates and the firmware only validates

Every table index that appears in a command body or an event body is **allocated by the
host**. The firmware keeps a parallel copy of the table, range-checks the index it is given,
and silently drops anything out of range or referring to an inactive entry (see §6.1 of the
MCU-protocol section). Nothing in the interface tells the host what the firmware currently
believes; keeping the two in sync is entirely the host's job. `[C]`

| Space | Size | Allocated by | Firmware validation | Freed by |
|---|---|---|---|---|
| BSS context index | **5** valid indices (`0`…`4`): four hardware BSS contexts plus the pseudo-index used for the P2P-device role (§0) | host | hard bound check in the command path *and* in the event path; an index ≥ 5 is dropped | BSS deactivation command |
| Own-MAC (MUAR) slot | one per active BSS | host | bound to the BSS index at activation | BSS deactivation |
| Station-record index | **15** (`0`–`1` reserved for one special network type, `2`–`14` for peers) | host, first free from 2 upward | existence + validity check | delete-station command, **or** a `STA_AGING_TIMEOUT` / `SEND_DEAUTH` event |
| WTBL (hardware station-table) row | see §B.2.5 | host security layer | index `0xFF` = the reserved invalid row | key removal / station delete |
| Key index | per station / per BSS | host | bound to a WTBL row | key-remove command |
| MCU command sequence number | 1…255 (`0` reserved to mark unsolicited events) | host | echoed only | response, or the host's 10 s timeout |
| Channel-privilege token | 8-bit, host-allocated, monotonic | host | echoed only | matching abort |
| TX MSDU token | 6000 entries, 15-bit field | host | firmware only echoes it back in the TX-free report | TX-free report, or a host-side reset of the pool |
| TX packet identifier (`PID`) | 8-bit, `0x7F` reserved | host | **firmware rejects an out-of-range PID and drops the frame** | TX-status report |

Two consequences a driver must design for:

* **An event can free an index without the host asking.** `STA_AGING_TIMEOUT` (`0x19`) and
  `SEND_DEAUTH` (`0x1B`) both mean the firmware has stopped maintaining a peer. The host
  must release the station index (and its WTBL row and keys) on receipt, otherwise the
  15-entry pool leaks and new peers cannot be added. `[C]`
* **After any reset that reloads the firmware** (L0.5 or L0), every one of these spaces is
  empty on the firmware side while the host still holds its allocations. The host must
  discard and re-create all of them; re-using a stale index is not detected as an error, it
  simply programs the wrong entry. After an L1 (sub-system) recovery, by contrast, BSS and
  station context **survives** and must *not* be re-created — only the DMA rings, the MSDU
  token pool and the beacon templates are re-initialised. `[C]`

### A.6a.2 Credits the firmware hands back

Two unsolicited events are the only transmit back-pressure the firmware provides above the
DMA-ring level. Both are per-object counters that the host must apply exactly; there is no
"current value" query and no re-transmission if one is lost. `[C]`

**Per-station free quota — `EVENT_ID_STA_UPDATE_FREE_QUOTA` (`0x16`)**, 3 significant bytes:

| Offset | Field |
|---|---|
| `0x00` | `ucStaRecIdx` |
| `0x01` | `ucUpdateMode` |
| `0x02` | `ucFreeQuota` (signed) |

| `ucUpdateMode` | Operation on the host's stored quota |
|---|---|
| `0`, `1` | **set** to `ucFreeQuota` |
| `2` | **add** `ucFreeQuota` |
| `3` | **subtract** `ucFreeQuota` |
| other | protocol violation — must be treated as fatal `[C]` |

The quota counts frames the firmware is willing to buffer for a peer that is in power save.
On each update the host re-splits the quota between the peer's delivery-enabled and
legacy access-category groups (a half/half split, biased by one to whichever
group was served last, and the split collapses to zero when the quota reaches zero) and then
re-runs its transmit scheduler. A quota of zero means *stop transmitting to this peer*. `[C]`

**Per-BSS absence/presence — `EVENT_ID_BSS_ABSENCE_PRESENCE` (`0x11`)**, body
`{u8 ucBssIndex, u8 ucIsAbsent, u8 ucBssFreeQuota, u8 rsv}` `[C]`:

* `ucIsAbsent = 1` — the BSS is off its operating channel (an off-channel excursion, a P2P
  notice-of-absence period, a DBDC hand-over). The host must stop dequeuing frames for that
  BSS index.
* `ucIsAbsent = 0` — back on channel; the host restarts its transmit-queue service.
* `ucBssFreeQuota` is the BSS-level equivalent of the per-station quota.
* The BSS index is bound-checked against the context count and an out-of-range event is
  discarded. `[C]`

Ignoring either event does not produce an error report: frames simply pile up in the
firmware's packet buffer until they hit their transmit lifetime, consuming page credits that
every other BSS and station shares. `[C]`

### A.6a.3 Teardown order

Before any reset that reloads firmware, before an interface is removed, and before the
driver unloads, the objects must be released **from the leaves inwards**, because the
firmware validates references and a parent that is torn down first orphans its children
`[C]`:

```
1. stop the host transmit queues for the BSS
2. move each station out of the associated state (station-state update)
3. remove the pairwise and group keys of each station
4. delete each station record   (frees the WTBL row)
5. stop beaconing / clear the beacon template          (AP-role BSS only)
6. abort any outstanding channel privilege (matching token)
7. deactivate the BSS context   (frees the own-MAC slot)
8. disable the radio / per-band enable
9. stop and drain the DMA rings, then release the MSDU token pool
```

If the host skips straight to the reset, nothing is reported and the reset still succeeds
`[C]` — but the firmware may transmit deauthentication or beacon frames on behalf of contexts
the host has already forgotten, up to the moment the subsystem is held in reset. `[L]`
On the teardown path the same order applies, with the addition that the interrupt must be
disabled and all packet servicing stopped before the rings are freed, so that nothing is
still running against freed ring memory. `[C]`

---

## A.7 Association / connection state transitions

The firmware tracks a three-state station machine. Transitions are communicated by re-issuing
`CMD_ID_UPDATE_STA_RECORD` (`0x13`) with the new `ucStaState` `[C]`:

| `ucStaState` | Name | Meaning |
|---|---|---|
| 0 | STA_STATE_1 | not authenticated |
| 1 | STA_STATE_2 | authenticated, not associated |
| 2 | STA_STATE_3 | associated — data path enabled |

Rules `[C]`:

* Entering state 2 sets the command's **needs-response** flag so the host learns when the
  firmware has committed the transition; all other transitions are fire-and-forget.
* Leaving state 2 (to 0 or 1) first deactivates the station's TX queues host-side, then
  sends the update.
* A station whose network type is the reserved special type does **not** get an update
  command on state change.
* For an AP-role BSS whose operating mode is "AP", any station state change additionally
  triggers a recompute-and-resend of the BSS RLM parameters (protection modes, slot time).
* BSS-level connection state is carried separately in `SET_BSS_INFO` byte `0x01` (§A.5.2).

Relevant unsolicited events `[L]`:

| Event | ID | Meaning |
|---|---|---|
| `EVENT_ID_ACTIVATE_STA_REC` | `0x0C` | station record committed |
| `EVENT_ID_STA_CHANGE_PS_MODE` | `0x12` | peer entered/left power save |
| `EVENT_ID_STA_AGING_TIMEOUT` | `0x19` | peer aged out (reports station index and MAC) `[C]` |
| `EVENT_ID_BSS_BEACON_TIMEOUT` | `0x13` | beacon loss on a BSS index `[C]` |
| `EVENT_ID_SEND_DEAUTH` | `0x1B` | firmware sent a deauth |
| `EVENT_ID_RX_ADDBA` / `RX_DELBA` / `TX_ADDBA` | `0x0A`/`0x0B`/`0x2E` | block-ack agreement changes |

---

## A.8 Install / remove an encryption key

| Field | Value |
|---|---|
| Command ID | `0x07` (`CMD_ID_ADD_REMOVE_KEY`) `[C]` |
| Direction | set, is-OID = 1, dedicated completion and timeout handlers `[C]` |
| Payload | **64 bytes (`0x40`)** `[C]` |

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | `ucAddRemove` — 1 = add, 0 = remove |
| `0x01` | 1 | `ucTxKey` — this key is the default TX key |
| `0x02` | 1 | `ucKeyType` — 0 = group, 1 = pairwise |
| `0x03` | 1 | `ucIsWapi` |
| `0x04` | 6 | `aucPeerAddr` — peer MAC for pairwise, BSSID/`ff:ff:ff:ff:ff:ff` for group |
| `0x0A` | 1 | `ucBssIdx` |
| `0x0B` | 1 | `ucAlgorithmId` — cipher, see table below |
| `0x0C` | 1 | `ucKeyId` — key index, see table below |
| `0x0D` | 1 | `ucKeyLen` (≤ 32) |
| `0x0E` | 1 | `ucWlanIndex` — **target WTBL row** |
| `0x0F` | 1 | `ucMgmtProtection` |
| `0x10` | 32 | `aucKeyMaterial` |
| `0x30` | 16 | `aucKeyRsc` — receive sequence counter / PN seed |

### A.8.1 Cipher enumeration (`ucAlgorithmId`, and the LMAC WTBL `CIPHER_SUITE` field)

The command's cipher identifier and the 5-bit `CIPHER_SUITE` field of LMAC WTBL DW2 use the
**same enumeration** `[C]`:

| Value | Cipher |
|---|---|
| 0 | none / open |
| 1 | WEP-40 |
| 2 | TKIP with MIC |
| 3 | TKIP without MIC |
| 4 | CCMP-128 (PMF-capable) |
| 5 | WEP-104 |
| 6 | BIP-CMAC-128 |
| 7 | WEP-128 |
| 8 | WPI-128 (SMS4) |
| 9 | CCMP-128 (CCX/DFP variant) |
| 10 | CCMP-256 |
| 11 | GCMP-128 |
| 12 | GCMP-256 |
| 13 | GCM-WPI-128 |
| 14 | BIP-CMAC-256 `[L]` |
| 15 | beacon-protection CMAC-128 `[L]` |
| 16 | beacon-protection CMAC-256 `[L]` |
| 17 | BIP-GMAC-256 `[L]` |
| 18/19 | beacon-protection GMAC-128 / GMAC-256 `[L]` |

The 2-bit `CIPHER_SUITE_IGTK` field of LMAC WTBL DW2 selects the integrity-key cipher for
management-frame protection independently of the data cipher `[C]`.

### A.8.2 Key-index encoding

| `ucKeyId` | Use |
|---|---|
| 0..3 | pairwise / group data keys (WEP: the four WEP key slots) |
| 4..5 | IGTK (management-frame protection) |
| 6..7 | BIGTK (beacon protection) |

The LMAC WTBL DW0 `KID` field is only 2 bits wide and holds the *data* key index `[C]`; the
integrity/beacon keys are reached through the separate `key_loc1` pointer in UMAC WTBL DW5
(§B.4) `[C]`. Where the peer has no dedicated WTBL row (e.g. a group key installed before the
AP station record exists), the host allocates a broadcast/multicast WTBL entry for the BSS and
puts its index in `ucWlanIndex` (§B.2.5) `[C]`.

### A.8.3 Default (transmit) key selection

| Field | Value |
|---|---|
| Command ID | `0x08` (`CMD_ID_DEFAULT_KEY_ID`) `[C]` |
| Direction | set, is-OID = 1 `[C]` |
| Payload | 4 bytes: `[0] = ucKeyId`, `[1] = ucWlanIndex`, `[2] = ucBssIdx`, `[3] = flags` `[U]` |
| Notes | When no key is present the WTBL index is written as `0xFF` (the reserved/invalid entry) `[C]` |

### A.8.4 Completion and error events

| Event | ID | Meaning |
|---|---|---|
| `EVENT_ID_ADD_PKEY_DONE` | `0x24` | pairwise key installed; carries BSS index and peer MAC `[C]` |
| `EVENT_ID_MIC_ERR_INFO` | `0x04` | TKIP MIC failure; carries flags and station address `[C]` |

Ordering: the target station record must exist (or a BC/MC WTBL row must be reserved) before
the key command is sent, and the key command must precede the transition of the station to
state 2 for the data path to be protected from the first frame `[C]`.

---

## A.9 RX filter, promiscuous and monitor mode

### A.9.1 RX filter — it is a **command**, not a register write

| Field | Value |
|---|---|
| Command ID | `0x0A` (`CMD_ID_SET_RX_FILTER`) `[C]` |
| Direction | set, is-OID selectable `[C]` |
| Payload | **68 bytes (`0x44`)** `[C]` |
| Layout | `[0..3] = u4RxPacketFilter` (32-bit filter word), `[4..67]` reserved, zero `[C]` |

Filter word bits, as accepted by the firmware `[C]`:

| Bit | Mask | Meaning |
|---|---|---|
| 0 | `0x0000_0001` | accept frames directed to our address |
| 1 | `0x0000_0002` | accept multicast frames that pass the multicast address filter (§A.15) |
| 2 | `0x0000_0004` | accept **all** multicast (bypass the address filter) |
| 3 | `0x0000_0008` | accept broadcast |
| 5 | `0x0000_0020` | promiscuous (defined by the encoding; **not settable through this command on this part**) |
| 7 | `0x0000_0080` | accept all locally addressed |
| 28 | `0x1000_0000` | forward association requests to the host (P2P/AP role) |
| 29 | `0x2000_0000` | forward authentication frames to the host |
| 30 | `0x4000_0000` | forward action frames to the host |
| 31 | `0x8000_0000` | forward probe requests to the host |

Behaviour `[C]`:

* Only bits 0–3 are set from the generic filter path; a value with any other low bit set must
  be rejected. Bits 28–31 are per-role forwarding controls and must be preserved across
  updates (the top nibble `0xF000_0000` is merged from the previously programmed value).
* **There is therefore no promiscuous bit reachable through this command**; promiscuous
  reception is only available through monitor mode (§A.9.3).
* Setting bit 2 makes the multicast address table (§A.15) irrelevant, so the table need not
  be programmed while it is set.
* Some host modes require the filter to be forced to `(filter & ~0x6) | 0x2` — i.e. drop
  "all multicast" and use filtered-multicast instead.

### A.9.2 Underlying hardware filter (RMAC)

The firmware implements the above by programming the RMAC frame-control registers, which are
also directly reachable from the host (§B.1). MT7932 is register-compatible with MT7921/MT7922
here `[L]`:

`MT_WF_RFCR` = `MT_WF_RMAC_BASE(band) + 0x000`, RMAC base = `0x820E_5000` (band 0) /
`0x820F_5000` (band 1). All bits are **drop-enables** (1 = discard):

| Bit | Name |
|---|---|
| 0 | `DROP_STBC_MULTI` |
| 1 | `DROP_FCSFAIL` |
| 3 | `DROP_VERSION` (bad protocol version) |
| 4 | `DROP_PROBEREQ` |
| 5 | `DROP_MCAST` |
| 6 | `DROP_BCAST` |
| 7 | `DROP_MCAST_FILTERED` (failed the multicast address filter) |
| 8 | `DROP_A3_MAC` |
| 9 | `DROP_A3_BSSID` |
| 10 | `DROP_A2_BSSID` |
| 11 | `DROP_OTHER_BEACON` |
| 12 | `DROP_FRAME_REPORT` |
| 13 | `DROP_CTL_RSV` (reserved control subtypes) |
| 14 | `DROP_CTS` |
| 15 | `DROP_RTS` |
| 16 | `DROP_DUPLICATE` |
| 17 | `DROP_OTHER_BSS` |
| 18 | `DROP_OTHER_UC` (unicast not addressed to us) |
| 19 | `DROP_OTHER_TIM` |
| 20 | `DROP_NDPA` |
| 21 | `DROP_UNWANTED_CTL` |

`MT_WF_RFCR1` = RMAC base + `0x004`: bit 4 `DROP_ACK`, bit 5 `DROP_BF_POLL`, bit 6 `DROP_BA`,
bit 7 `DROP_CFEND`, bit 8 `DROP_CFACK` `[L]`.

Note for implementers: upstream `mt76` sends `CMD_ID_SET_RX_FILTER` with a *different*
interpretation of the same 68-byte body — 4 reserved bytes, a `mode` byte (1 = "set fif",
2 = "clear"), 3 pad, a 32-bit `fif` word and a 32-bit `bit_map`/`bit_op` pair mapping directly
onto the RFCR drop bits. The firmware shipped with MT7932 accepts the layout in §A.9.1 (filter
word at offset 0) `[C]`; whether it also accepts the `mode`-based layout needs tracing `[U]`.

### A.9.3 Monitor mode

| Field | Value |
|---|---|
| Command ID | `0xFC` (`CMD_ID_SET_MONITOR`) `[C]` |
| Direction | set, is-OID selectable `[C]` |
| Payload | **16 bytes (`0x10`)** `[C]` |

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | `ucEnable` |
| `0x01` | 1 | `ucBand` |
| `0x02` | 1 | `ucPriChannel` |
| `0x03` | 1 | `ucSco` (secondary-channel offset) |
| `0x04` | 1 | `ucChannelWidth` |
| `0x05` | 1 | `ucChannelS1` |
| `0x06` | 1 | `ucChannelS2` |
| `0x07` | 1 | `ucBandIdx` |
| `0x08` | 2 | `u2Aid` — restrict capture to one AID, 0 = all |
| `0x0A` | 1 | `fgDropFcsErrorFrame` |
| `0x0B` | 5 | reserved |

When monitor mode is enabled the channel fields carry the capture channel; disabling sends
the same command with the channel fields zeroed `[C]`. Enabling monitor mode also requires a
pre-load calibration step before the command is issued `[C]`.

---

## A.10 Transmit power

Four independent mechanisms exist; they compose as *min(...)* in firmware `[L]`.

### A.10.1 Regulatory domain and per-channel legality

| Field | Value |
|---|---|
| Command ID | `0x0F` (`CMD_ID_SET_DOMAIN_INFO`) `[C]` |
| Payload (passive-scan / channel-attribute variant) | 64 bytes (`0x40`) `[C]` |
| Payload (domain variant) | `12 + 8 × N` bytes, where N is the number of channel-list sub-bands `[C]` |

### A.10.2 Absolute per-BSS power cap (AP constraint / local maximum)

| Field | Value |
|---|---|
| Command ID | `0x5B` (`CMD_ID_SET_AP_CONSTRAINT_PWR_LIMIT`) `[C]` |
| Direction | set, is-OID = 1 `[C]` |
| Payload | **40 bytes (`0x28`)** `[C]` |

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | version/tag, constant `1` |
| `0x05` | 1 | enable (0 = release the constraint) |
| `0x06` | 1 | maximum power, **units of 0.5 dBm** (value = dBm × 2) |
| `0x07` | 1 | constant `0x10` (mode/selector) |
| `0x08..0x27` | 32 | zero |

The requested maximum is clamped to **8 dBm ≤ P ≤ 20 dBm** before encoding; out-of-range
requests are silently clamped and logged `[C]`. This is the "local maximum transmit power"
constraint, not a per-rate backoff.

### A.10.3 Per-rate / per-channel country power limits (the backoff table)

| Field | Value |
|---|---|
| Command ID | `0x5D` (`CMD_ID_SET_COUNTRY_POWER_LIMIT_PER_RATE`) `[L]` |
| Legacy variant | `0x49` (`CMD_ID_SET_COUNTRY_POWER_LIMIT`) `[L]` |
| Record size | **547 bytes (`0x223`) per channel entry** for the per-rate table; a second, smaller **145-byte (`0x91`)** record form is used for the legacy per-channel table `[C]` |

> **Wire format: see the EEPROM/calibration section, which is authoritative here.** That
> section specifies the command as a **44-byte header** (byte-compatible with the public
> `CMD_SET_TXPOWER_COUNTRY_TX_POWER_LIMIT_PER_RATE`: `ucCmdVer`, flags, `u2CmdLen`, `ucNum`,
> `eBand`, `bCmdFinished`, `eLimitType` sub-table selector, `u4CountryCode`, 32-byte
> auxiliary block) followed by **122-byte per-channel records** (1 channel byte + 121 signed
> 0.5 dBm values), 8 records per command. The 547/145 figures above are **host-side buffer
> sizes, not wire record sizes** `[L]`: a 547-byte record does not correspond to any public
> `CMD_SKU_TABLE_TYPE` variant (CONNAC2 = 161 values ⇒ 162 B, CONNAC3 = 449 ⇒ 450 B), and the
> 121-value form is the one derived from the shipped table's `s_`/`c_`/`m_` column structure.
> A driver must emit the 122-byte record. `[U]` — the origin of 547/145 needs re-derivation.

The limit tables themselves ship as plain-text data files (per-rate
limits, SAR limits, SDB sub-band limits, 2.4 GHz common-path backoff, antenna gain) and must
be merged per country/channel/rate before being pushed to firmware; the merge takes the
**minimum** of the regulatory limit, the SAR limit, the SDB limit and the common-path backoff `[C]`.
A per-rate power (`PPR`) binary blob and a wireless-calibration blob are additionally pushed
through the EFUSE-buffer-mode path at init `[C]`.

Additional related command IDs present in the enumeration `[L]`: `0x24` `SET_TX_PWR`,
`0x25` `SET_PWR_PARAM`, `0x27` `SET_TX_EXTEND_PWR`, `0x36`/`0x40` edge-power limits 2.4/5 GHz,
`0x38` `SET_TXPWR_CTRL`, `0x41` `SET_CHANNEL_PWR_OFFSET`, `0x42` `SET_80211AC_TX_PWR`,
`0x43` `SET_PATH_COMPASATION`, `0xD0` `GET_TXPWR_TBL` (query).

### A.10.4 Per-station power offset

The LMAC WTBL entry carries a **6-bit signed `TX_POWER_OFFSET`** field in DW5 that biases the
transmit power for that station relative to the band setting `[C]` (§B.2).

---

## A.11 Antenna, spatial streams and chain selection

There is no host-visible antenna-switch register in the MAC control surface. Chain and stream
selection is expressed at three levels `[C]`:

1. **Per-BSS**: `ucNss` in `SET_BSS_INFO` (byte `0x5D`) and `ucTxNss`/`ucRxNss` in the RLM
   parameter block (bytes `0x14`/`0x15`).
2. **Per-station**: LMAC WTBL DW4 carries eight 3-bit `ANT_ID_STS0..7` fields — the antenna
   (chain) identifier used for each space-time stream index — plus `LDPC_HT`/`LDPC_VHT`/`LDPC_HE`
   enables; DW7 carries a 5-bit `SPE_IDX` (spatial-extension / antenna-pattern index) and
   `DBNSS_EN` (dynamic-bandwidth Nss) `[C]`.
3. **Per-packet**: the TX descriptor carries a `spe_idx` field and a fixed-rate override that
   includes Nsts `[C]`.

An additional TX-descriptor/WTBL selector bit exists in the WTBL init control register
(`MT_WTBL_SPE_IDX_SEL`, bit 6 of the WTBL init-target control register at WTBLON_TOP + `0x3B0`)
which chooses whether the SPE index comes from the WTBL or the descriptor `[L]`.

MT7932 is a 2×2 part; the reported `Nss` capability is delivered in the NIC-capability response
along with the LDPC/STBC TX and RX capability flags and the DBDC flag `[C]`.

---

## A.12 Set MAC address / BSSID

* **Own MAC address** of a BSS is programmed by `CMD_ID_BSS_ACTIVATE_CTRL` (`0x11`, §A.5.1):
  the 6-byte address plus the **own-MAC-address index**. That index is the MUAR
  (multi-user address register) table slot; the same value is written into the `MUAR_Idx`
  field (6 bits, DW0 bits 21:16) of every LMAC WTBL row belonging to the BSS, so the receiver
  can match address 1 against the right own-MAC `[C]`. A 6-bit index implies **64 own-MAC
  slots** in the MUAR table `[L]`.
* **BSSID** is programmed by `CMD_ID_SET_BSS_INFO` (`0x12`, §A.5.2) at body offset `0x24` `[C]`.
* **Randomised / scan MAC** is supplied per scan in the scan request body (`aucRandomMac`,
  offset `0x45E`) and needs no separate command `[C]`. A pool of randomised addresses is
  managed host-side and each address in concurrent use consumes a WTBL row, so the WTBL
  allocation policy must reserve rows for them `[C]`.
* There is no command to change the permanent (EEPROM) address; that is read from the EEPROM
  image at init.

---

## A.13 EDCA / WMM parameter programming

| Field | Value |
|---|---|
| Command ID | `0x1D` (`CMD_ID_UPDATE_WMM_PARMS`) `[C]` |
| Direction | set, needs-response = 0 `[C]` |
| Payload | **44 bytes (`0x2C`)** `[C]` |

Body: four packed 10-byte AC parameter records followed by 4 bytes of BSS binding `[C]`:

| Offset | Size | Field |
|---|---|---|
| `0x00` | 10 | AC_BE: `u2CWmin`, `u2CWmax`, `u2TxopLimit`, `u2Aifsn`, `ucGuardTime`, `ucIsACMSet` |
| `0x0A` | 10 | AC_BK (same layout) |
| `0x14` | 10 | AC_VI |
| `0x1E` | 10 | AC_VO |
| `0x28` | 1 | `ucBssIndex` |
| `0x29` | 1 | `fgIsQBSS` |
| `0x2A` | 1 | `ucWmmSet` — which of the hardware WMM parameter sets this BSS uses |
| `0x2B` | 1 | reserved |

AC index order is **BE = 0, BK = 1, VI = 2, VO = 3** `[C]`. `u2CWmin`/`u2CWmax` are the
*expanded* contention-window values (not the exponents). `ucGuardTime` is the stop/flush guard.
The number of independent hardware WMM parameter sets is reported by firmware in the
NIC-capability response (`ucWmmSetNum`, minimum 1) and each BSS is bound to one of them `[C]`.

### A.13.1 MU-EDCA (802.11ax)

| Field | Value |
|---|---|
| Command ID | `0xB0` (`CMD_ID_MQM_UPDATE_MU_EDCA_PARMS`) `[C]` |
| Payload | **72 bytes (`0x48`)** `[C]` |
| Body | version/length header, BSS index, then four 8-byte per-AC records `{ucECWmin, ucECWmax, ucAifsn, ucIsACMSet, ucMUEdcaTimer, 3 pad}` `[L]` |

### A.13.2 Read-back

There is no command to read EDCA parameters back from hardware; the host must keep its own
copy of the four AC records (CWmin, CWmax, TXOP limit, AIFSN, guard time) per BSS `[C]`.

Related: `0x6A` `CMD_ID_UPDATE_AC_PARMS`, `0x34` `CMD_ID_SET_UAPSD_PARAM`,
`0x1E` `CMD_ID_SET_WMM_PS_TEST_PARMS` `[L]`.

---

## A.14 Beacon and probe-response template programming

| Field | Value |
|---|---|
| Command ID | `0x18` (`CMD_ID_UPDATE_BEACON_CONTENT`) `[C]` |
| Direction | set, needs-response = 0, no completion handler `[C]` |
| Payload | `8 + u2IELen` bytes for an update; **6 bytes** for a delete `[C]` |
| IE limit | **600 bytes**; a longer template is rejected `[C]` |

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | `ucUpdateMethod` |
| `0x01` | 1 | `ucBssIndex` |
| `0x02` | 2 | reserved |
| `0x04` | 2 | `u2Capability` — the capability-information field firmware places in the beacon |
| `0x06` | 2 | `u2IELen` |
| `0x08` | n | IE body, starting immediately after the fixed beacon fields |

`ucUpdateMethod` accepted values `[C]`:

| Value | Meaning | Payload |
|---|---|---|
| 0 | update (randomised-address variant) | `8 + IE len` |
| 1 | update all — **the value used for normal beacon content** | `8 + IE len` |
| 2 | delete all | 6 |
| ≥3 | **rejected** ("unknown update method") | — |

Consequence: **this firmware exposes no separate probe-response template method.** In AP/GO
roles the firmware derives probe responses from the beacon template; only the beacon body is
uploaded `[C]`. Probe responses that must differ from the beacon have to be transmitted by the
host as ordinary management frames.

The IE body is the element sequence that follows the fixed beacon header: CSA, HT/VHT/HE
capability and operation, ERP, extended capabilities, vendor/MTK OUI, TX-power envelope and
so on, in that order. The fixed beacon header is 36 bytes and is **not** part of the command
payload; the payload starts immediately after it `[C]`.

Deleting the template (method 2) is issued on BSS teardown/disconnect `[C]`.

---

## A.15 Multicast / broadcast address filter

| Field | Value |
|---|---|
| Command ID | `0xC1` (`CMD_ID_MAC_MCAST_ADDR`) `[C]` |
| Direction | set, is-OID = 1, standard completion/timeout handlers `[C]` |
| Payload | **200 bytes** `[C]` |

| Offset | Size | Field |
|---|---|---|
| `0x00` | 4 | `u4NumOfGroupAddr` — number of valid addresses (computed as byte-length ÷ 6, capped to 6 bits ⇒ ≤ 63) |
| `0x04` | 1 | `ucBssIndex` |
| `0x05` | 3 | reserved — written as `0x22 0x22 0x22` when the list is non-empty and `0x00 0x00 0x00` when it is empty `[U]` |
| `0x08` | 192 | `arAddress[32][6]` — up to **32** group addresses |

The list is bounded to 192 bytes of address data (32 entries) `[C]`. An empty list combined
with RX-filter bit 1 set means "no multicast accepted"; RX-filter bit 2 bypasses this table
entirely `[C]`. The hardware side of this filter is the RMAC `DROP_MCAST_FILTERED` bit (§A.9.2) `[L]`.

A second command (`CMD_ID_SET_AM_FILTER` = `0x55`) exists for the address-match offload used
in suspend `[L]`.

---

## A.16 Power-save configuration

| Field | Value |
|---|---|
| Command ID | `0x05` (`CMD_ID_POWER_SAVE_MODE`) `[C]` |
| Direction | set; needs-response = 0; is-OID and handlers attached only when the caller wants a synchronous result `[C]` |
| Payload | **4 bytes** `[C]` |

| Offset | Size | Field |
|---|---|---|
| `0x00` | 1 | `ucBssIndex` |
| `0x01` | 1 | `ucPsProfile` — power-save profile (CAM / fast-PS / legacy-PS) |
| `0x02` | 2 | reserved |

Rules `[C]`:

* The BSS index must be within `ucHwBssIdNum`; out-of-range must be rejected.
* Repeat requests for an unchanged profile need not be sent.

Related commands `[L]`: `0x21` `CMD_ID_SET_PS_PROFILE_ADV` (advanced PS profile),
`0x33` `CMD_ID_SET_OPPPS_PARAM`, `0x32` `CMD_ID_SET_NOA_PARAM`,
`0x34` `CMD_ID_SET_UAPSD_PARAM`, `0x58` `CMD_ID_SET_SUSPEND_MODE`,
`0x4A` `CMD_ID_SET_WOWLAN`. Per-station power-save state is mirrored in the LMAC WTBL
(`POWER_SAVE`/`TX_PS` in DW2 and `PSM`/`I_PSM`/`DONOT_UPDATE_I_PSM`/`TXOP_PS_CAP` in DW5) `[C]`.
Peer power-save transitions are reported by `EVENT_ID_STA_CHANGE_PS_MODE` (`0x12`) `[L]`.

---

## A.17 Command / event ID reference

### A.17.1 Commands confirmed in use on MT7932

| ID | Name | Dir | Payload | Section |
|---|---|---|---|---|
| `0x03` | `SCAN_REQ_V2` | set | 1236 | A.4 |
| `0x04` | `NIC_POWER_CTRL` | set | 4 | A.1.1 |
| `0x05` | `POWER_SAVE_MODE` | set | 4 | A.16 |
| `0x07` | `ADD_REMOVE_KEY` | set (OID) | 64 | A.8 |
| `0x08` | `DEFAULT_KEY_ID` | set (OID) | 4 | A.8.3 |
| `0x0A` | `SET_RX_FILTER` | set | 68 | A.9.1 |
| `0x0F` | `SET_DOMAIN_INFO` | set | 64 / `12+8N` | A.10.1 |
| `0x11` | `BSS_ACTIVATE_CTRL` | set | 12 | A.5.1 |
| `0x12` | `SET_BSS_INFO` | set | 116 | A.5.2 |
| `0x13` | `UPDATE_STA_RECORD` | set | 200 | A.6.1 |
| `0x14` | `REMOVE_STA_RECORD` | set | 5 | A.6.2 |
| `0x18` | `UPDATE_BEACON_CONTENT` | set | `8+IE` / 6 | A.14 |
| `0x19` | `SET_BSS_RLM_PARAM` | set | 22 | A.2 |
| `0x1B` | `SCAN_CANCEL` | set | 4 | A.4.2 |
| `0x1D` | `UPDATE_WMM_PARMS` | set | 44 | A.13 |
| `0x28` | `SET_DBDC_PARMS` | set | 36 | A.1.2 |
| `0x5B` | `SET_AP_CONSTRAINT_PWR_LIMIT` | set (OID) | 40 | A.10.2 |
| `0x82` | `GET_STATISTICS` | query | 12 | B.7 |
| `0x85` | `GET_STA_STATISTICS` | query | 28 | B.7 |
| `0x88` | statistics-item query (**not in public enum**) | query | 12 … 452 | B.7 |
| `0x99` | all-station query (**not in public enum**) | query | 8 | B.7 |
| `0x9D` | SDB channel-group info (**not in public enum**) | set | 2 | A.1.2 |
| `0xB0` | `MQM_UPDATE_MU_EDCA_PARMS` | set | 72 | A.13.1 |
| `0xC1` | `MAC_MCAST_ADDR` | set (OID) | 200 | A.15 |
| `0xC4` | `SW_DBG_CTRL` | query | 264 | B.7 |
| `0xFC` | `SET_MONITOR` | set | 16 | A.9.3 |

### A.17.2 Events referenced above

| ID | Name |
|---|---|
| `0x02` | `LINK_QUALITY` |
| `0x03` | `STATISTICS` |
| `0x04` | `MIC_ERR_INFO` |
| `0x0A`/`0x0B`/`0x2E` | `RX_ADDBA` / `RX_DELBA` / `TX_ADDBA` |
| `0x0C` | `ACTIVATE_STA_REC` * |
| `0x0D` | `SCAN_DONE` |
| `0x0F` | `TX_DONE` (per-packet TX status: WTBL index, PID, status, SN, TID, retry count, flush flag) |
| `0x10` | `CH_PRIVILEGE` |
| `0x12` | `STA_CHANGE_PS_MODE` |
| `0x13` | `BSS_BEACON_TIMEOUT` |
| `0x17` | `SW_DBG_CTRL` * |
| `0x19` | `STA_AGING_TIMEOUT` |
| `0x21`/`0x22` | `STA_STATISTICS` / `STA_STATISTICS_UPDATE` * |
| `0x23` | `SCHED_SCAN_DONE` |
| `0x24` | `ADD_PKEY_DONE` |
| `0xA1` | `RSSI_MONITOR` |
| `0xCD` | `WTBL_INFO` |
| `0xCE` | `MIB_INFO` |

\* These IDs are the public CONNAC2 event numbers for the corresponding query responses.
They are **not** in this part's set of separately-decoded event IDs (see the MCU-protocol
section §5.4): a frame carrying one of them is matched to the outstanding command by its
echoed sequence number instead of being dispatched by ID. Every other row above **is**
separately decoded.

---

# Part B — Hardware tables

## B.1 How the host reaches the on-chip tables

All MAC sub-blocks used by the tables in this part live in the chip address range
`0x820C_0000 – 0x820F_FFFF` and are reachable through the **static PCIe BAR window**
(1 MiB, no programmable remap needed) `[C]`. For these MAC-block rows the chip→BAR fixed map
is entry-for-entry identical to public MT7921 `[C]` (the map as a whole is not — see §4.1 of
the PCIe/register-map section). The rows that matter here:

| Chip base | BAR offset | Size | Block |
|---|---|---|---|
| `0x820C_0000` | `0x00_8000` | `0x4000` | PLE |
| `0x820C_4000` | `0x0A_8000` | `0x4000` | **WF_UWTBL (UMAC station table + key table)** |
| `0x820C_8000` | `0x00_C000` | `0x2000` | PSE |
| `0x820C_A000` | `0x02_6000` | `0x2000` | MU control (band 0) |
| `0x820C_C000` | `0x00_E000` | `0x2000` | packet processor |
| `0x820C_E000` | `0x02_1C00` | `0x0200` | **WF_SEC (security engine)** |
| `0x820C_F000` | `0x02_2000` | `0x1000` | WF_PF (packet filter) |
| `0x820D_0000` | `0x03_0000` | `0x10000` | **WF_WTBLON (LMAC station table region)** |
| `0x820E_0000` | `0x02_0000` | `0x0400` | band 0 **WF_CFG** |
| `0x820E_1000` | `0x02_0400` | `0x0200` | band 0 **WF_TRB** |
| `0x820E_2000` | `0x02_0800` | `0x0400` | **band 0 AGG** |
| `0x820E_3000` | `0x02_0C00` | `0x0400` | **band 0 ARB** |
| `0x820E_4000` | `0x02_1000` | `0x0400` | **band 0 TMAC** |
| `0x820E_5000` | `0x02_1400` | `0x0800` | **band 0 RMAC (RX filter, airtime MIB)** |
| `0x820E_7000` | `0x02_1E00` | `0x0200` | band 0 DMA |
| `0x820E_9000` | `0x02_3400` | `0x0200` | **band 0 WTBLOFF (RCPI update control)** |
| `0x820E_A000` | `0x02_4000` | `0x0200` | band 0 ETBF |
| `0x820E_B000` | `0x02_4200` | `0x0400` | band 0 LPON (TSF) |
| `0x820E_C000` | `0x02_4600` | `0x0200` | band 0 |
| `0x820E_D000` | `0x02_4800` | `0x0800` | **band 0 MIB counter block** |
| `0x820F_0000 … 0x820F_D000` | `0x0A_0000 … 0x0A_4800` | — | band 1 mirrors of the above |

Derived addresses used below `[C]`:

| Item | Chip address | BAR offset |
|---|---|---|
| LMAC WTBL data-unit control register (WDUCR) | `0x820D_4200` | `0x03_4200` |
| LMAC WTBL data window base | `0x820D_8000` | `0x03_8000` |
| UMAC WTBL / key data-unit control register (WDUCR) | `0x820C_4094` | `0x0A_8094` |
| UMAC WTBL / key data window base | `0x820C_6000` | `0x0A_A000` |

---

## B.2 Station table — LMAC half ("LWTBL")

### B.2.1 Geometry

| Property | Value |
|---|---|
| Data window base | `0x820D_8000` `[C]` |
| Index field in the address | bits 14:8 → **7 bits (128 entries per group)** `[C]` |
| Word field in the address | bits 7:2 → 6 bits (64 DW of address space per entry) `[C]` |
| Entry stride | **256 bytes** `[C]` |
| Entry size actually defined | **33 DW = 132 bytes (DW0 … DW32)** `[C]` — this is the host's read window, not an address-decode limit: the 256-byte stride makes DW0–DW63 addressable, and public gen4m reads one DW more on the pre-ECO2 layout `[L]` |
| Group select | 3 bits in WDUCR `[C]` → up to 8 groups ⇒ **1024 addressable entries architecturally** `[L]` (inferred from the field width; the implemented depth is open question 2) |
| Entries in use | reported by firmware in the NIC-capability response; accepted range **1 .. 49** `[C]`. Public mt76 assumes 20 for MT7921/MT7922. |
| Reserved/invalid index | `0xFF` `[C]` |
| Default TX index | `entryCount - 1` — used for unencrypted management transmission `[C]` |

### B.2.2 Access sequence (read or write)

| Step | Action |
|---|---|
| 1 | Write WDUCR (`0x820D_4200`) = `(wlanIdx >> 7) & 0x7` — group select, bits [2:0] |
| 2 | Form the address `0x820D_8000 \| ((wlanIdx & 0x7F) << 8) \| ((DW & 0x3F) << 2)` |
| 3 | 32-bit read or write at that address |
| 4 | Repeat step 3 for further DWs of the same entry; the group select is sticky |

No busy/ready handshake is required for the data-unit window; the read returns the current
entry word directly `[C]`. The WDUCR itself can be read back and reflects the selected group `[C]`.
(A separate indirect path exists via the WTBL init-target control/data registers at
WTBLON_TOP + `0x3B0` / `0x3B8` / `0x3BC` with an execute bit 31 and a write bit 16, used for
bulk initialisation; the busy indication is bit 31 of the WTBL update register `[L]`.)

### B.2.3 Entry field layout (MT7932 uses the "v2" variants throughout `[C]`)

The v2 selection is genuinely **unconditional** in this build — the WTBL decode paths contain
no ECO test and no call to the ECO-version accessor, and the fixed 33-DW read size is the
ECO≥2 size `[C]`. Note what that does and does not establish: it shows the shipping
MT7932/MT7922 silicon in this platform is ECO≥2, not that an older field layout cannot exist
on some other stepping `[L]`.

**DW0 — peer address low / control**

| Bits | Field | Notes |
|---|---|---|
| 7:0 | `ADDR_4` | peer MAC byte 4 |
| 15:8 | `ADDR_5` | peer MAC byte 5 |
| 21:16 | `MUAR_IDX` | own-MAC (MUAR) table index for address-1 matching |
| 22 | `RC_A1` | receive-check address 1 |
| 24:23 | `KID` | data key index (0..3) |
| 25 | `RC_ID` | receive-check ID |
| 26 | `FROM_DS` | expected From-DS |
| 27 | `TO_DS` | expected To-DS |
| 28 | `RV` | receive valid |
| 29 | `RC_A2` | receive-check address 2 |
| 30 | `WPI_FLAG` | WAPI |
| 31 | reserved | — |

**DW1** — peer MAC bytes 0..3 (`ADDR_0`, 32 bits) `[C]`

**DW2 — capability / security**

| Bits | Field |
|---|---|
| 11:0 | `AID12` |
| 12 | `GID_SU` |
| 13 | `SPP_EN` |
| 14 | `WPI_EVEN` |
| 15 | `AAD_OM` |
| 20:16 | `CIPHER_SUITE` (§A.8.1) |
| 22:21 | `CIPHER_SUITE_IGTK` |
| 23 | reserved |
| 24 | `SW` |
| 25 | `UL` |
| 26 | `TX_PS` / `POWER_SAVE` |
| 27 | `QOS` |
| 28 | `HT` |
| 29 | `VHT` |
| 30 | `HE` |
| 31 | `MESH` |

**DW3 — queueing / beamforming (v2 layout)**

| Bits | Field |
|---|---|
| 1:0 | `WMM_Q` |
| 3:2 | `RXD_DUP_MODE` |
| 4 | `VLAN2ETH` |
| 5 | `BEAM_CHG` |
| 7:6 | `BA_MODE` |
| 15:8 | `PFMU_IDX` |
| 23:16 | `ULPF_IDX` |
| 24 | `RIBF` (implicit BF) |
| 25 | `ULPF` |
| 26 | `IGN_FBK` |
| 28:27 | reserved |
| 29 | `TEBF` |
| 30 | `TEBF_VHT` |
| 31 | `TEBF_HE` |

**DW4 — antenna / coding**

| Bits | Field |
|---|---|
| 2:0 … 23:21 | `ANT_ID_STS0` … `ANT_ID_STS7` (eight 3-bit chain IDs) |
| 24 | `CASCAD` |
| 25 | `LDPC_HT` |
| 26 | `LDPC_VHT` |
| 27 | `LDPC_HE` |
| 28 | `DIS_RHTR` |
| 29 | `ALL_ACK` |
| 30 | `DROP` |
| 31 | `ACK_EN` |

**DW5 — TX behaviour / power / power-save (v2 layout)**

| Bits | Field |
|---|---|
| 2:0 | `AF` (A-MPDU factor) |
| 4:3 | `AF_HE` |
| 5 | `RTS` |
| 6 | `SMPS` |
| 7 | `DYN_BW` |
| 10:8 | `MMSS` (min MPDU start spacing) |
| 11 | `USR` |
| 14:12 | `SR_R` (spatial reuse, 3 bits in v2) |
| 15 | `SR_ABORT` |
| 21:16 | **`TX_POWER_OFFSET`** (signed, per-station power bias) |
| 23:22 | `MPDU_SIZE` |
| 25:24 | `PE` (packet extension) |
| 26 | `DOPPL` |
| 27 | `TXOP_PS_CAP` |
| 28 | `DONOT_UPDATE_I_PSM` |
| 29 | `I_PSM` |
| 30 | `PSM` |
| 31 | `SKIP_TX` |

**DW6 — aggregation**: eight 4-bit `BA_WIN_SIZE_TID0..7` fields `[C]`.

**DW7 — sounding / GI / spatial extension**

| Bits | Field |
|---|---|
| 2:0 | `CBRN` |
| 3 | `DBNSS_EN` |
| 4 | `BAF_EN` |
| 5 | `RDG_BA` |
| 6 | `R` |
| 11:7 | `SPE_IDX` |
| 12–15 | `G2`, `G4`, `G8`, `G16` |
| 17:16 … 23:22 | `G2_LTF`, `G4_LTF`, `G8_LTF`, `G16_LTF` (2 bits each) |
| 25:24 … 31:30 | `G2_HE`, `G4_HE`, `G8_HE`, `G16_HE` (2 bits each) |

**DW8 — RTS counters / partial AID**

| Bits | Field |
|---|---|
| 4:0 / 9:5 / 14:10 / 19:15 | `RTS_FAIL_CNT_AC0..AC3` (5 bits each, saturating) |
| 28:20 | `PARTIAL_AID` |
| 30:29 | reserved |
| 31 | `CHK_PER` |

**DW9 — RX size average / format capability / rate-selection state (v2 layout)**

| Bits | Field |
|---|---|
| 13:0 | `RX_AVG_MPDU_SIZE` |
| 15:14 | reserved |
| 16 | `PRITX_SW_MODE` |
| 17 | `PRITX_PLR` |
| 18 | `PRITX_DCM` |
| 19 | `PRITX_ER160` |
| 20 | `PRITX_ERSU` |
| 22:21 | **`FCAP`** — bandwidth capability: bit 0 = 20/40 MHz, bit 1 = 20→80 MHz, `0` = 20→160 MHz |
| 25:23 | `MPDU_FAIL_CNT` |
| 28:26 | `MPDU_OK_CNT` |
| 31:29 | **`RATE_IDX`** — index of the rate entry currently selected by the hardware rate controller |

**DW10 – DW13 — the per-station rate table** (see §B.6).

**DW14 – DW18 — auto-rate counters** (see §B.7.1).

**DW19 — PPDU counters**: `DATA_RETRY_CNT` (15:0), `MGNT_RETRY_CNT` (31:16) `[C]`.

**DW20 – DW27 — admission control** (per-AC medium-time budgets) — layout not enumerated by
the host `[U]`.

**DW28 — OM/duplicate control (v2 layout)**: `OM_INFO` (11:0), `RXD_DUP_OM_CHG` (12), rest
reserved `[C]`.

**DW29 — user RSSI/SNR (v2 layout)**: `USR_RSSI` (8:0), `USR_SNR` (14:9), reserved (15),
`RAPID_REACTION_RATE` (26:16), reserved (29:27), `HT_AMSDU` (30), `AMSDU_CROS_LG` (31) `[C]`.

**DW30 — response RCPI (v2 layout, MT7932)**: `RESP_RCPI_0..3`, 8 bits each — the RCPI measured
on the last response frame from this peer, per receive chain. **This is the word a driver reads
for per-station RSSI on MT7932**; on pre-ECO-2 CONNAC2 silicon the same information lives in
DW29 `[C]`. Conversion: dBm = (RCPI / 2) − 110 `[L]`.

**DW31 / DW32 — per-chain SNR**: six 6-bit fields per word, `SNR_RX0..3` in DW31 and
`SNR_RX4..7` in DW32 `[C]`.

### B.2.4 Reading per-station RSSI without a full dump

A single 32-bit read of DW30 of the entry (address
`0x820D_8000 | ((idx & 0x7F) << 8) | (30 << 2)`, after selecting the group) yields all four
response RCPI values `[C]`. The mode and averaging parameters for those fields are set in the
WTBLOFF block: `MT_WTBLOFF_TOP_RSCR` = `0x820E_9000 + 0x008` (band 0) /
`0x820F_9000 + 0x008` (band 1), with `RCPI_MODE` in bits 31:30 and `RCPI_PARAM` in bits 25:24 `[L]`.

### B.2.5 WTBL index allocation policy

No hardware allocation policy is visible `[L]`. The partitioning below is the host policy the
analysed driver uses `[C]`; it is consistent with what the MT7932 firmware interface expects
and is a safe model for a new driver:

* Index `entryCount - 1` is the **default TX index** for unencrypted management frames.
* Broadcast/multicast (group-key) rows are drawn from a per-BSS partition of the low indices:
  BSS 0 uses indices `0 … min(entryCount-2, 7)`; every other BSS uses
  `min(entryCount-1, 8) … entryCount-2`.
* A BC/MC row is reused for a BSS if it is already owned by that BSS and its cipher matches or
  is "unset"; open/WEP/WPI BSSes always reuse the same row.
* Peer (unicast) rows are allocated from the remaining space at station-record creation.
* Randomised-MAC operation consumes one additional row per concurrently-used address.
  The per-row state that must be tracked outside the chip is: in-use flag, owning BSS index,
  cipher and key index.

---

## B.3 Station table — UMAC half ("UWTBL")

### B.3.1 Geometry

| Property | Value |
|---|---|
| Data window base | `0x820C_6000` `[C]` |
| Index field in the address | bits 12:6 → **7 bits (128 entries per group)** `[C]` |
| Word field | bits 5:2 → 4 bits (16 DW of address space per entry) `[C]` |
| Entry stride | **64 bytes** `[C]` |
| Entry size actually defined | **8 DW = 32 bytes (DW0 … DW7)** `[C]` — again the host's read window; the 64-byte stride makes 16 DW addressable `[L]` |
| Group select | 4 bits in WDUCR `[C]` → up to 16 groups ⇒ **2048 addressable entries architecturally** `[L]` (inferred from the field width) |
| Target select | WDUCR bit 31: `0` = station half, `1` = key table (§B.4) `[C]` |

### B.3.2 Access sequence

| Step | Action |
|---|---|
| 1 | Write WDUCR (`0x820C_4094`) = `(wlanIdx >> 7) & 0xF` — bit 31 = 0 selects the UWTBL |
| 2 | Form the address `0x820C_6000 \| ((wlanIdx & 0x7F) << 6) \| ((DW & 0xF) << 2)` |
| 3 | 32-bit read or write |

The station index used here is the **same** `ucWlanIndex` as for the LMAC half — the two halves
are two views of one station entry, not two tables `[C]`.

### B.3.3 Entry field layout

| DW | Bits | Field |
|---|---|---|
| 0 | 31:0 | `PN0` — packet-number low 32 bits |
| 1 | 15:0 | `PN1` — packet-number high 16 bits (48-bit PN total) |
| 1 | 27:16 | `COM_SN` — common sequence number |
| 1 | 31:28 | reserved |
| 2 | 11:0 | `AC0_SN` |
| 2 | 23:12 | `AC1_SN` |
| 2 | 31:24 | `AC2_SN` (low 8) |
| 3 | 3:0 | `AC2_SN` (high 4) |
| 3 | 15:4 | `AC3_SN` |
| 3 | 27:16 | `AC4_SN` |
| 3 | 31:28 | `AC5_SN` (low 4) |
| 4 | 7:0 | `AC5_SN` (high 8) |
| 4 | 19:8 | `AC6_SN` |
| 4 | 31:20 | `AC7_SN` |
| 5 | 10:0 | **`KEY_LOC0`** — key-table index for the primary key |
| 5 | 15:11 | reserved |
| 5 | 26:16 | **`KEY_LOC1`** — key-table index for the secondary key (IGTK/BIGTK or the rekey partner) |
| 5 | 27 | `QOS` |
| 5 | 28 | `HT` |
| 5 | 31:29 | reserved |
| 6 | 9:0 | `HW_AMSDU_CFG` — hardware A-MSDU aggregation configuration |
| 6 | 31:10 | reserved |
| 7 | 31:0 | reserved / key-table shadow |

`KEY_LOC0` = all-ones (`0x7FF`) means "no key bound" `[C]`.

Eight per-TID/AC sequence-number counters plus one common SN are maintained here; the 48-bit PN
is the replay counter used by the security engine `[C]`.

---

## B.4 Key / security table

### B.4.1 Geometry

| Property | Value |
|---|---|
| Data window | shared with the UWTBL: base `0x820C_6000` `[C]` |
| Selection | WDUCR bit 31 = 1 (`TARGET`) `[C]` |
| Index field | bits 12:6 → 7 bits per group; group = bits 3:0 of WDUCR `[C]` → **2048 key slots architecturally** `[L]` (inferred from the field widths; the implemented depth is open question 3) |
| Entry stride | **64 bytes** `[C]` |
| Entry size read by the host | **8 DW = 32 bytes** (one 256-bit key) `[C]` |
| Keys per station | **2** (`KEY_LOC0`, `KEY_LOC1` in UWTBL DW5) `[C]` |

### B.4.2 Access sequence

| Step | Action |
|---|---|
| 1 | Read UWTBL DW5 of the station to obtain `key_loc0` / `key_loc1` |
| 2 | If `key_loc == 0x7FF` there is no key bound — stop |
| 3 | Write WDUCR (`0x820C_4094`) = `0x8000_0000 \| ((key_loc >> 7) & 0xF)` |
| 4 | Form the address `0x820C_6000 \| ((key_loc & 0x7F) << 6) \| ((DW & 0xF) << 2)`, DW = 0..7 |
| 5 | 32-bit reads give the 32-byte key material |

### B.4.3 Binding

* Cipher selection is **not** in the key entry; it is in LMAC WTBL DW2
  (`CIPHER_SUITE`, `CIPHER_SUITE_IGTK`) `[C]`.
* Data key index (0..3) is in LMAC WTBL DW0 `KID` `[C]`.
* The key slot itself is allocated by firmware in response to `CMD_ID_ADD_REMOVE_KEY`; the host
  never writes `KEY_LOC*` directly. The host-visible contract is
  `(ucWlanIndex, ucKeyId, ucAlgorithmId, key material)` → firmware picks the slot `[C]`.
* The security engine block (`WF_SEC`, `0x820C_E000`, 512 bytes, BAR `0x02_1C00`) holds the
  cipher datapath control registers `[L]`.

---

## B.5 BSS table and own-MAC (MUAR) table

| Property | Value |
|---|---|
| Hardware BSS contexts | **4** (`ucHwBssIdNum`, validated to the range 1..4) `[C]` |
| Extra pseudo-index | index `== ucHwBssIdNum` is a software-only P2P-device BSS with no hardware context `[C]` |
| Own-MAC (MUAR) slots | 64 (6-bit `MUAR_IDX` in LMAC WTBL DW0) `[L]` |
| Independent WMM parameter sets | `ucWmmSetNum`, reported by firmware, minimum 1 `[C]` |

Binding chain `[C]`:

```
BSS index  --(CMD 0x11)-->  own-MAC index (MUAR slot) + own MAC address
                            + BC/MC WTBL index
BSS index  --(CMD 0x12)-->  BSSID, op mode, rates, security, DBDC band, WMM set, Nss
BSS index  --(CMD 0x19)-->  RF band, primary channel, SCO, bandwidth, centre freqs, Nss
station    --(CMD 0x13)-->  station index + BSS index + WTBL index
WTBL row   --(DW0 MUAR_IDX)-->  the BSS's own-MAC slot   (address-1 match)
WTBL row   --(DW2 AID12)-->     the BSS's association ID space
```

There is no host-writable BSS-descriptor table in MMIO; the BSS context is materialised by
firmware into the RMAC own-MAC comparators, the TSF/LPON block (`0x820E_B000`) and the beacon
engine `[L]`. The `ucDBDCBand` field of `SET_BSS_INFO` is what binds a BSS index to a **band**
(and therefore to the band-0 vs band-1 register mirrors) `[C]`.

---

## B.6 Rate table and per-station rate-selection state

The per-station rate table lives in **LMAC WTBL DW10 – DW13** and is host-readable and
host-writable `[C]`:

| DW | Bits 13:0 | Bits 29:16 |
|---|---|---|
| 10 | `RATE1` | `RATE2` |
| 11 | `RATE3` | `RATE4` |
| 12 | `RATE5` | `RATE6` |
| 13 | `RATE7` | `RATE8` |

Each 14-bit rate word is encoded as `[C]`:

| Bits | Field | Encoding |
|---|---|---|
| 5:0 | MCS / rate index | CCK: 0..3 (1, 2, 5.5, 11 Mb/s); OFDM: index into the 8-entry OFDM table; HT/VHT/HE: MCS number |
| 9:6 | TX mode | 0 = CCK, 1 = OFDM, 2 = HT-mixed, 3 = HT-greenfield, 4 = VHT, 8 = HE-SU, … |
| 12:10 | Nsts − 1 | 0 ⇒ 1 stream |
| 13 | STBC | |

Rate-selection state visible to the host `[C]`:

| Location | Field | Meaning |
|---|---|---|
| DW9 bits 31:29 | `RATE_IDX` | which of `RATE1..RATE8` the hardware rate controller is currently using |
| DW9 bits 28:26 / 25:23 | `MPDU_OK_CNT` / `MPDU_FAIL_CNT` | 3-bit saturating short-term success/failure trackers driving the up/down decision |
| DW14 | `RATE_1_TX_CNT` (15:0), `RATE_1_FAIL_CNT` (31:16) | attempts and failures at rate 1 |
| DW15 | `RATE_2_OK_CNT` (15:0), `RATE_3_OK_CNT` (31:16) | |
| DW16 | `CURRENT_BW_TX_CNT` (15:0), `CURRENT_BW_FAIL_CNT` (31:16) | |
| DW17 | `OTHER_BW_TX_CNT` (15:0), `OTHER_BW_FAIL_CNT` (31:16) | |
| DW18 | `RTS_OK_CNT` (15:0), `RTS_FAIL_CNT` (31:16) | |
| DW29 bits 26:16 | `RAPID_REACTION_RATE` | rate the fast-fallback logic drops to |

Firmware also exposes a higher-level view through the statistics path — link speed, "rate table"
identifier, 2.4 GHz 256-QAM enable, train-up/train-down thresholds, forced-stream and
forced-SE overrides, a running counter, a status word and the SPE index `[C]`. A per-packet
fixed-rate override is available through the TX descriptor `[C]`.

---

## B.7 Per-station and per-BSS counters

### B.7.1 Counters the hardware keeps in the station entry (host-readable by MMIO)

All of these are in the LMAC WTBL row and are read with the sequence of §B.2.2 `[C]`. They are
treated as **free-running and not clear-on-read**, so the host differences successive samples
`[C]`; that the narrow saturating fields (`MPDU_OK_CNT`, `MPDU_FAIL_CNT`, `RTS_FAIL_CNT_ACn`)
are maintained and reset by hardware for rate control is `[L]`.

| DW | Width | Counter |
|---|---|---|
| 8 | 4 × 5 bit | per-AC RTS failure count (saturating) |
| 9 | 3 + 3 bit | MPDU OK / MPDU fail (saturating) |
| 14 | 2 × 16 bit | rate-1 TX attempts / failures |
| 15 | 2 × 16 bit | rate-2 OK / rate-3 OK |
| 16 | 2 × 16 bit | current-bandwidth TX / fail |
| 17 | 2 × 16 bit | other-bandwidth TX / fail |
| 18 | 2 × 16 bit | RTS OK / RTS fail |
| 19 | 2 × 16 bit | data retry count / management retry count |
| 30 | 4 × 8 bit | response RCPI per chain (a measurement, not a counter) |
| 31/32 | 8 × 6 bit | per-chain SNR |

### B.7.2 Counters read through the MCU

The aggregate MIB registers are reachable over MMIO (§B.9) but are not used that way on this
part; the practical way to get per-station and per-BSS counters is the statistics command
family `[C]`.

**Per-station (`CMD_ID_GET_STA_STATISTICS` = `0x85`, query, 28-byte request,
`EVENT_ID_STA_STATISTICS` = `0x21`)** `[C]`. The response is organised into selectable groups:

| Group mask | Contents |
|---|---|
| `0x01` | STA stat: current temperature, association ID, TX total count, TX fail count, rate-1 TX count, rate-1 fail count, with derived PER |
| `0x02` | MIB info, per DBDC band: RX success, RX with CRC error, RX dropped for FIFO full |
| `0x04` | Last RX info: per-antenna RX RSSI, TX-response RSSI, beacon RSSI |
| `0x08` | Last TX info: per-antenna output TX power (0.1 dBm resolution) |
| `0x10` | RX reorder: miss / within-window / ahead / behind (64-bit) |
| `0x20` | Rate-adaptation info: link speed, rate table, 2.4 GHz 256-QAM TX, train-down, train-up, forced TX stream, forced SE off, running count, status, SPE index |
| `0x40` | Aggregation histogram: **16 TX bins and 16 RX bins per DBDC band**, plus the 16 programmable range boundaries |

**Per-BSS and per-band (`0x88` statistics-item query, 452-byte request)** `[C]`. The response
carries, as 64-bit counters:

*Per band*: TX time count, TX duration count, TX duration backoff time, A-MPDU count, BA count,
A-MPDU MPDU count, A-MPDU acked count, SU TX OK count, spatial-reuse A-MPDU MPDU count,
spatial-reuse A-MPDU acked count, TX bandwidth counts for 20/40/80/160 MHz, MPDU retry-drop
count, RTS drop count, lifetime-drop count, beacons transmitted, RX duration count, RX duration
backoff time, RX CCK MDRDY time, RX FCS error, RX FCS OK, MDRDY count, A-MPDU RX count,
RX total bytes, RX MPDU count, RX FIFO overflow, intra-BSS PPDU count, inter-BSS PPDU count,
beacons received, EIFS slot count, channel-idle time, CCA-NAV TX time, NAV time, primary
ED time, EIFS CCK count, EIFS OFDM count, primary CCA time, secondary CCA time, total CCA time,
secondary 20/40/80 MHz CCA time, secondary primary-20 ED time for sub-bands 0..7, OBSS airtime,
non-Wi-Fi airtime.

*Per BSS* (four BSS instances per counter): TX data count, TX count, TX fail count, TX OK count,
TX byte count, RX data count, RX MSDU count, unicast-to-me data NSS count, RX OK count,
RX byte count.

*Per AC* (four ACs): TX retry count, TX drop count, RX count (+1 further counter).

**Per peer (`0x99` all-station query, 8-byte request, 644-byte response)** `[C]`, for each
active peer: peer ID, total TX count, total TX fail count, management retry count, data retry
count, RTS fail count, RTS OK count, RSSI, RCPI0, SNR0/SNR1, RX toss count, firmware TX fail,
firmware TX retransmit, firmware TX frame count.

**Aggregate/legacy statistics (`CMD_ID_GET_STATISTICS` = `0x82`, query, 12-byte request,
`EVENT_ID_STATISTICS` = `0x03`)** `[C]`; per-BSS statistics are accumulated by the host in
276-byte (`0x114`) blocks of seven 28-byte per-AC records `[C]`.

**Other statistics items reachable via `0x88`** `[C]`:

| Item | Request size | Contents |
|---|---|---|
| Chip-counter MIB | 284 B | rate info (STBC/Nsts/mode/rate/SGI/BW, TX rate), TX retries/error/frame counts, RX PHY errors (good PLCP, PLCP header parity error, PLCP header CRC error, FCS error, unicast RTS), RX MAC counts (unicast, multicast, management/control matching RA, management/control/data other RA, multicast management/control, unicast CTS/ACK, unicast RTS/CTS other RA), RX error counts (frame count, retries, duplicate error, A-MPDU duplicate error, overflow), RX security errors (unicast/multicast TKIP replay and MIC errors, unicast/multicast CCMP replay and CCMP errors), TX/RX stat (RX RTS, RX unicast BSS, RX PLCP, RX no delimiter), TX A-MPDU count, RX BA count, TX BA count, RX A-MPDU/MPDU counts, RX A-MPDU SGI/STBC, RX BAR, RX out-of-window, RX out-of-sequence, RX duplicate, RX stuck, RX A-MPDU single HT/legacy, RX sequence hole, RX queue, RX no-BA-session, RX/TX DELBA, RX/TX density, RX unexpected |
| A-MPDU/rate MIB | 132 B | success/attempt TX counts per HT MCS (17 entries), VHT MCS (12), HE MCS (12), each also in SGI and HE 0.8/1.6 µs variants; TX non-A-MPDU counts for CCK (4), OFDM (8), HT (17), VHT (12), HE (12); BA timeout count; RX rate counts per HT/VHT/HE MCS and SGI variants; **`u4TxAggregationCnt[16]` and `u4RxAggregationCnt[16]`** |
| A-MSDU | — | MSDU count, A-MSDU count, MSDU-per-A-MSDU histogram (**8 bins**) |
| Station state | 36 B | RSSI0/RSSI1, noise, CCA, CCA self total, CCA other total, CCA interference total; beacon received, beacons missed (per reason: NAN/AWDL, scan, coexistence, other); QBSS load station count and channel utilisation (current/min/max), available admission capacity |
| Frame counter | 12 B | per-BSS management/data frame counters |
| PHY activity | 28 B | PHY TX duration, PHY RX duration |
| LQM CCA | 36 B | CCA state |
| LQM RSSI / SNR | — | RSSI0/1, SNR0/1 |
| TRX per-AC counts | 68 B | per-AC TX/RX counts |
| TX RTS | — | TX RTS fail count, TX RTS OK count |
| TRX hardware delay | — | TX and RX hardware-delay histograms |
| Channel switch | — | old channel (primary channel, sub channel, bandwidth, band), new channel, time, dwell time, reason, PHY channel-switch latency |

**Noise floor** is retrieved differently: `CMD_ID_SW_DBG_CTRL` (`0xC4`, query) with a
264-byte (`0x108`) request whose first 32-bit word is the selector `0xB126_0001`; the response
carries the noise value `[C]`.

**Weighted-average link quality** (weighted-average SNR[0..1], RSSI[0..1], TX rate, RX rate,
Bluetooth antenna duration, eSCO count) is a further `0x88` item `[C]`.

**Clear-on-read**: none of the MCU-delivered counters are clear-on-read; the firmware returns
running totals, so successive samples must be differenced. A separate "clean" query exists to
zero the accumulated aggregate statistics `[C]`. The register-level MIB counters (§B.9) *do* have
clear-on-read and clear-enable bits `[L]`.

---

## B.8 Address / header-translation table

MT7932 performs **Ethernet ↔ 802.11 header translation in hardware**, on both TX and RX `[C]`:

* **RX**: the receive descriptor carries a "header translated" indicator; when set, the frame
  has already been converted to an Ethernet-II header by the MAC. Frames whose EtherType the
  translator does not recognise arrive untranslated and are reported as such `[C]`.
* **TX**: the transmit descriptor carries a header-format field selecting
  {802.11 native, "command", Ethernet with translation, Ethernet with translation and
  VLAN removal}, plus `RMVL` (remove VLAN), `VLAN`, `ETYP` (EtherType present) and `MRD` flags
  and an explicit header length in words `[C]`.
* The **translation blacklist / EtherType table** is *not* programmed by the host. There is no
  host-visible register or command in the MAC control surface that writes translation-table
  entries; the table is configured by firmware from the station record (the `TX_PROC`/header-
  translation attributes of the station-record command) and by the per-station `VLAN2ETH` bit
  in LMAC WTBL DW3 `[C]`.
* The block that implements it is the packet processor at `0x820C_C000` (BAR `0x00_E000`,
  8 KiB) `[L]`. Its register layout is not established and would need tracing `[U]`.

---

## B.9 MIB / statistics counter block

Register block, register-compatible with MT7921/MT7922 `[L]`:

| Band | Chip base | BAR offset | Size |
|---|---|---|---|
| 0 | `0x820E_D000` | `0x02_4800` | **2 KiB (`0x0800`)** |
| 1 | `0x820F_D000` | `0x0A_4800` | **2 KiB (`0x0800`)** |

> **Correction.** An earlier version of this section gave the block size as `0x2000` (8 KiB).
> It is `0x0800`: that is what entries 25 and 43 of the fixed map (§4 of the PCIe/register-map
> section) declare, every offset listed below fits under `0x7F8`, and a `0x2000` window from
> BAR `0x024800` would run into BAR `0x026000`, which the map assigns to WF_MUCOP. `[C]`

Notable offsets (all relative to the band base) `[L]`:

| Offset | Name | Contents |
|---|---|---|
| `0x004` | `MIB_SCR1` | bit 8 `TXDUR_EN`, bit 9 `RXDUR_EN` — enable TX/RX duration accumulation |
| `0x02C` | `MIB_SDR9` | bits 23:0 **channel-busy time** |
| `0x048` | `MIB_SDR16` | bits 23:0 busy time (second accumulator) |
| `0x054` | `MIB_SDR36` | bits 23:0 **TX airtime** |
| `0x058` | `MIB_SDR37` | bits 23:0 **RX airtime** |
| `0x090` | `MIB_SDR34` | bits 15:0 MU-beamforming TX count |
| `0x0B0 + 4n` (n = 0..3) | `MIB_ARNG` | **aggregation-range boundaries**, four 8-bit ranges per register ⇒ 16 bins |
| `0x0C0` / `0x0C4` / `0x0CC` | `MIB_DR8` / `DR9` / `DR11` | RX MPDU / byte counters |
| `0x100 + 16n` | `MIB_MB_SDR0` | per-AC: bits 31:16 RTS retry count |
| `0x108 + 16n` | `MIB_MB_SDR2` | per-AC: bits 15:0 frame retry count |
| `0x518` | `MIB_MB_BSDR2` | bits 15:0 BA failure count |
| `0x520` | `MIB_MB_BSDR3` | bits 15:0 ACK failure count |
| `0x558` / `0x55C` / `0x564` / `0x568` | `MIB_SDR12` / `SDR31` / `SDR14` / `SDR15` | A-MPDU and MPDU transmit counters |
| `0x688` | `MIB_MB_BSDR0` | bits 15:0 RTS count |
| `0x690` | `MIB_MB_BSDR1` | bits 15:0 RTS failure count |
| `0x698` | `MIB_SDR3` | bits 31:16 **FCS error count** |
| `0x770` / `0x774` | `MIB_SDR22` / `SDR23` | RX counters |
| `0x780` | `MIB_SDR5` | |
| `0x7A8` | `MIB_SDR32` | bits 31:16 implicit-BF count, bits 15:0 explicit-BF count |
| `0x7DC + 4n` (n = 0..3) | `TX_AGG_CNT` | **aggregation histogram bins 0..7** (two 16-bit bins per register) |
| `0x7EC + 4n` (n = 0..3) | `TX_AGG_CNT2` | **aggregation histogram bins 8..15** |

Adjacent airtime/CCA counters live in the RMAC block (`0x820E_5000` band 0 / `0x820F_5000`
band 1) `[L]`:

| Offset | Name | Contents |
|---|---|---|
| `0x380 + 4n` | `RMAC_MIB_AIRTIME0..` | per-sub-band airtime accumulators |
| `0x3B8` | `RMAC_MIB_AIRTIME14` | bits 23:0 **OBSS airtime** |
| `0x3C4` | `RMAC_MIB_TIME0` | bit 30 `RXTIME_EN`, bit 31 `RXTIME_CLR` — enable and **clear** the RX-time accumulators |

Beamforming feedback counters are in the ETBF block (`0x820E_A000` band 0) `[L]`:
`ETBF_TX_APP_CNT` at `+0x150` (bits 31:16 implicit-BF TX, 15:0 explicit-BF TX) and
`ETBF_RX_FB_CNT` at `+0x158` (all / HE / VHT / HT feedback counts, 8 bits each).

**Clear-on-read behaviour**: the airtime/duration accumulators are cleared by an explicit
clear bit (`RXTIME_CLR`) rather than by reading, and the aggregation and per-AC counters are
free-running `[L]`. **Direct host reads of the MIB block are not required** — all statistics
can be obtained through the MCU (§B.7), which is the recommended path because firmware also
owns the clear/enable bits `[C]`.

---

## Open questions / needs hardware tracing

1. **WDUCR offsets.** `0x820D_4200` (LMAC) and `0x820C_4094` (UMAC/key) are the MT7932
   values. Whether other bits of those registers (beyond GROUP and
   TARGET) are meaningful on this silicon — e.g. an auto-increment or a lock bit — is unknown `[U]`.
2. **Real WTBL entry count.** The architectural address decode allows 1024 LMAC entries; the
   firmware-reported value is clamped to 49 here, and public mt76 assumes 20. The number the
   MT7932 firmware actually reports needs to be read from the NIC-capability response on live
   hardware `[U]`.
3. **Key-table depth.** `KEY_LOC` is 11 bits and the decode allows 2048 slots; the number
   actually implemented is unknown `[U]`.
4. **DBDC device-ID conditional.** MT7932 takes the "selector = 2" branch of the DBDC command
   and populates the 2.4 GHz TX-queue bitmap word that MT7922 leaves zero. Whether this
   disables DBDC, selects a different band-pairing mode, or is a workaround is unknown `[U]`.
5. **Commands `0x88`, `0x99`, `0x9D`.** These are not in any public MediaTek header. The
   `0x88` request body is a 4-byte header (`[0]` DBDC band index, `[1]` item count) followed by
   8-byte `{u32 group, u32 item-id}` records with group `0x0008_0000` and item IDs including
   `0x02`, `0x03`, `0x63`, `0x6E`, `0x7A`, `0x7B`, `0x7C`, `0x84` and `0xD1 + bssIndex`; the
   full item-ID space and the exact response encodings need tracing `[U]`.
6. **RX-filter body dialect.** This firmware is driven with the filter word at offset 0;
   upstream mt76 drives the same command ID with a `mode`/`fif`/`bit_map` body. Which forms the
   MT7932 firmware accepts, and whether the RFCR drop bits can be set individually through it,
   needs tracing `[U]`.
7. **Multicast-filter reserved bytes.** Bytes `0x05..0x07` of the multicast command are written
   as `0x22` when the list is non-empty and `0x00` when it is empty; their meaning is unknown `[U]`.
8. **Probe-response templates.** Update method 3 (probe response) is not accepted by this
   firmware. Whether some firmware build would accept it, or whether MT7932 firmware
   genuinely derives probe responses from the beacon template only, needs tracing `[U]`.
9. **Admission-control words.** LMAC WTBL DW20–DW27 are readable as raw data; their
   field layout is not established `[U]`.
10. **Header-translation table.** There is no host-visible programming interface. Whether the
    packet-processor block at `0x820C_C000` exposes a writable EtherType/blacklist table needs
    tracing `[U]`.
11. **MIB clear semantics.** Which of the MIB counters are genuinely clear-on-read on this
    silicon, as opposed to cleared by `RXTIME_CLR` or a global MIB clear, is undetermined —
    the block is only reached through the MCU statistics path `[U]`.
12. **Own-MAC (MUAR) table.** The 64-slot depth is inferred from the 6-bit `MUAR_IDX` field;
    the register block that holds the own-MAC comparators and any per-slot mask was not
    identified `[U]`.


---

# MT7932 — Chip Identity and Delta versus Publicly-Documented MediaTek Parts

## Scope

This section establishes what the MT7932 *is*: how it identifies itself to a host, how its
silicon revision is read, what its two PCIe functions are, what radio and MAC capabilities it
has, and — the central question — exactly which parts of the CONNAC2 (mt792x-class) host
interface described in the rest of this document are MT7932-specific. The remainder of this
specification documents a register-compatible MT7922-class PCIe host interface; this section
enumerates, exhaustively, every point at which behaviour must diverge for MT7932, and states
plainly where nothing diverges. Confidence markers: `[C]` confirmed, `[L]` likely, `[U]`
unverified.

---

## 1. Identity

### 1.1 Chip ID

| Item | Value | Conf. |
|---|---|---|
| Chip ID | `0x7932` | [C] |
| Source of the chip ID used by the host | PCI configuration space Device ID of the Wi-Fi function, used directly as the chip ID | [C] |
| Chip-ID verification register (`should_verify_chip_id`) | Present but **disabled** for this part | [C] |
| Optional chip-ID readback register | WF_TOP_CFG `top_cfg_base + 0x1008`, i.e. **`0x8002_1008`**; low 16 bits = chip ID | [C] |
| Adjacent IP/version register | `0x8002_1000`; bits [3:0] are taken as a chip-version nibble | [C] |
| `top_cfg_base` for this part | `0x8002_0000` (WF_TOP_MISC_OFF / WF_TOP_CFG) | [C] |

The chip-ID readback path is identical to MT7922 and to public CONNAC2 parts; because
`should_verify_chip_id` is false for both MT7922 and MT7932, a driver need not read it. It is
nevertheless a valid identity register.

There is a legacy Wi-Fi PCI device ID `0x0616` that the host maps to chip ID `0x7922`
(this is the upstream `14c3:0616` MT7922). No such alias exists for MT7932. [C]

### 1.2 Hardware / ROM / factory version registers and the revision table

Version discovery uses two registers, read through the **boot-ROM (initialisation) register-access
command** `INIT_CMD_ID_ACCESS_REG` (ID `0x03`, reply event `0x02`) rather than through the static
PCIe BAR window. `TOP_HVR` has no static-map entry at all (the `0x7000_0000` entry stops at
`0x7000_FFFF`) and `TOP_FVR` is reachable only through the remappable BAR slot at `0x40000`
while that slot still holds its default value, so neither is a plain BAR-offset read. The
reads happen **before any firmware image is downloaded**, because the image file names are
derived from the result.

| Register | Chip address | Field | Meaning | Conf. |
|---|---|---|---|---|
| `TOP_HCR` | `0x7001_0200` | — | present in the hardware-configuration record, not read during version discovery | [C] |
| `TOP_HVR` | `0x7001_0204` | `[7:0]` | **HW version** (`ucHwVer`) | [C] |
| `TOP_HVR` | `0x7001_0204` | `[11:8]` | **Factory version** (`ucFactoryVer`) | [C] |
| `TOP_FVR` | `0x8800_0004` | `[7:0]` | **ROM version** (`ucRomVer`) | [C] |

`TOP_HCR`/`TOP_HVR` match public CONNAC2 (`CONNAC2X_TOP_HCR`/`CONNAC2X_TOP_HVR` in gen4m).
`TOP_FVR` is **`0x8800_0004`**, not `0x7001_0208`: this build carries the MT7961-header value
of `CONNAC2X_TOP_FVR`, which public gen4m also defines as `0x8800_0004` for that family. [C]

**Revision (ECO) table.** The `ECO_INFO` table is shared verbatim between the MT7922 and
MT7932 hardware-configuration records — i.e. it is an MT792x-family table, not an
MT7932-specific one. Entries are 4 bytes, `{ucHwVer, ucRomVer, ucFactoryVer, ucEcoVer}`,
terminated by an all-zero `{HwVer, RomVer, FactoryVer}` triple.

| `ucHwVer` (HVR[7:0]) | `ucRomVer` (FVR[7:0]) | `ucFactoryVer` (HVR[11:8]) | → `ucEcoVer` | Stepping | Conf. |
|---|---|---|---|---|---|
| `0x00` | `0x00` | `0x0A` | `0x01` | E1 | [C] |
| `0x01` | `0x01` | `0x0A` | `0x02` | E2 | [C] |
| `0x00` | `0x00` | `0x00` | — | end of table | [C] |

Lookup semantics are the public gen4m ones: walk the table, match all three of
HwVer/RomVer/FactoryVer; on reaching the terminator without a match, fall back to the last
real entry (E2). `ucEcoVer` is then stored in the hardware-configuration record.

**Delta vs. public MT7961/MT7921:** the public `mt7961_eco_table` E2 entry is
`{0x10, 0x01, 0x0A, 0x02}`; here E2 is `{0x01, 0x01, 0x0A, 0x02}`. The E1 entry is identical.
A driver that reuses the MT7961 table verbatim will mis-detect an E2 MT7932/MT7922 part and
fall through to the "last entry" path (which happens to yield the right answer, ECO 2, by
accident). [C]

The shipped RAM firmware trailer declares `eco_code = 0x01` [C] (raw byte, read from the
container). The stepping is **E2**: the ECO table above maps this part's
`{0x01, 0x01, 0x0A}` to `ucEcoVer = 2`, and the firmware-image name token is `_2` [C]. The
reconciliation — that the trailer's `eco_code` field is **zero-based**, so that the host
reports `E(eco_code + 1)` — is [L]; the raw `0x01` and the derived `E2` come from two
different fields and only that reading makes them agree. Do not read the raw `0x01` as "E1"
without checking which field you are holding.

### 1.3 A-die / companion-die version

There is **no A-die version in the static hardware-configuration record**: the field that
public gen4m uses for this (`u4ADieVer` / `u2ADieChipVersion`) is zero-initialised for both
MT7922 and MT7932. [C]

The A-die product ID and ECO version are obtained at run time from the firmware capability
query, TLV **`TAG_CAP_HW_ADIE_VERSION` (tag `0x14`)**, whose payload is the public
`struct CAP_HW_ADIE_VERSION` `{u2ProductID, u2EcoVersion, u32 reserved[4]}`. The host stores
only `u2ProductID`. [C] The actual value reported by MT7932 firmware is not observable
statically. [U]

The MAC / baseband / TOP IP versions and the configuration ID likewise come from
`TAG_CAP_HW_VERSION` (tag `0x05`, `struct CAP_HW_VERSION`) at run time. [C]

### 1.4 PCIe identity — both functions

The MT7932 package presents **two PCIe functions**.

| Function | Vendor ID | Device ID | Class code | Conf. |
|---|---|---|---|---|
| Wi-Fi | `0x14C3` | **`0x7932`** | `0x02` / `0x80` / `0x00` (Network controller, other) | [C] IDs, [L] class code (read out of the EEPROM identity record, §1.4 below, whose field roles are inferred) |
| Bluetooth | `0x14C3` | **`0x793B`** | `0x02` / `0x80` / `0x00` | [C] IDs, [L] class code |

Sibling parts handled by the same driver family, for context:

| Package | Wi-Fi device ID | Bluetooth device ID | Conf. |
|---|---|---|---|
| MT7922 | `0x7922` (also `0x0616`) | `0x792A` | [C] |
| MT7923 | `0x7923` | `0x792B` | [C] |
| MT7932 | `0x7932` | `0x793B` | [C] |

**What `0x793B` in the calibration image denotes.** The shipped 2560-byte EEPROM/eFuse image
begins with the chip ID `0x7932` at offset 0, then contains a PCIe-identity programming region
at offsets `0x10`–`0x2F` holding two 8-byte records per function:

| EEPROM offset | Contents (16-bit LE fields) | Interpretation | Conf. |
|---|---|---|---|
| `0x00` | `0x7932` | chip ID | [C] |
| `0x10` | `0x7932`, `0x14C3`, then dword `0x0280_0000` | Wi-Fi function: device ID, vendor ID, PCI class-code register (rev `0x00`, prog-if `0x00`, subclass `0x80`, class `0x02`) | [C] value / [L] field roles |
| `0x18` | `0x7932`, `0x14C3`, then dword `0x0000_A210` | Wi-Fi function: subsystem identity record + one further configuration dword | [C] value / [L] field roles |
| `0x20` | `0x793B`, `0x14C3`, then dword `0x0280_0000` | Bluetooth function: device ID, vendor ID, class-code register | [C] value / [L] field roles |
| `0x28` | `0x793B`, `0x14C3`, then dword `0x0000_0000` | Bluetooth function: subsystem identity record | [C] value / [L] field roles |

So `0x793B` is **the PCIe Device ID of the Bluetooth function of the same package**, not a
Wi-Fi variant, not an A-die, and not a second Wi-Fi SKU. The calibration image programs the
identity of both functions because both are fed from the same eFuse/EEPROM. [C]

No subsystem vendor/device pair distinct from `14C3:7932` / `14C3:793B` exists; a driver
should match on vendor+device only. [C]

### 1.5 Bluetooth identity of the package

The Bluetooth side is a separate PCIe function of the same die, of the class handled by
MediaTek's public `btmtk` PCIe support. The evidence and what it implies:

* The Bluetooth device IDs in this family are `14C3:792A`, `14C3:792B` and `14C3:793B`,
  mapping to internal BT chip IDs `0x7922`, `0x7923` and `0x7932` respectively. [C]
* Every chip-conditional distinction on the Bluetooth side is of the form "`0x7922` → one
  behaviour; `0x7932` and `0x7923` → the other". **`0x7932` is treated identically to
  `0x7923` throughout, and never identically to `0x7922`.** [C]
* Only two Bluetooth RAM images ship: one built for MT7922 and one built for MT7923. The
  MT7923 image is the **IPC-transport** variant (its patch modules are the PCIe
  MMIO-doorbell/MSI inter-processor-communication set); the MT7922 image is not. [C]
* Bluetooth firmware file names follow the template
  `BT_RAM_CODE_MT<chipid>_<n>_<m>_hdr.bin`; no `MT7932` Bluetooth image is shipped. [C]
* Both Bluetooth images and the MT7932 Wi-Fi manufacturing image carry build tags from the
  same MediaTek "neptune"/CCN9 combo-firmware line:
  `t-neptune-custom-mt7923-ccn9-2242-MT7922_2245_CCN9_x86_Gen4m-…`,
  `t-neptune-custom-mt7923-ccn9-2242-MT7923_2247_CCN9_x86_Gen4m-…`,
  `t-neptune-custom-mt7932-ccn9-2405-mfg-MT7932_2405_CCN9_x86_Gen4m_TEST_MODE-…`. [C]
* The MT7923 Bluetooth image's build path names the BT subsystem `btsys19` / `a7972`. [C]

**Conclusion.** The MT7932 package's Bluetooth controller is an **MT7923-class BT function**:
same BT/BGF core generation as MT7923, same IPC-over-PCIe transport, and it runs the MT7923
Bluetooth RAM image. `[L]` — the chip-ID mapping and the uniform "`0x7932` ≡ `0x7923`" branch
structure are `[C]`; the specific firmware-image selection is inferred from the fact that no
MT7932 Bluetooth image exists and the MT7923 image is the only IPC-capable one. The BT
subsystem was evidently *not* revised when the Wi-Fi side moved from MT7923 to MT7932.
Practically: an MT7932 Bluetooth driver should be an MT7923 Bluetooth driver with the device
ID added. [L]

The Wi-Fi side mirrors this grouping exactly: on the Wi-Fi function too, every chip-conditional
branch is "`0x7922` vs. everything else", and MT7932 always takes the `0x7923` path. [C]

### 1.6 Firmware image identity

| Image | Identity strings | Conf. |
|---|---|---|
| RAM code (`W7932_2.bin`, 1 190 788 B) | trailer: `chip_id = 0x14`, `eco_code = 0x01`, `n_region = 5`, `format_ver = 2`, `format_flag = 1`, `fw_ver = "____00000"`, `build_date = "20260331164939"`; embedded release string `Sunrise_mt7932_FW_4.30` + `20260209`; second date `20260331234721` | [C] |
| ROM patch (`…MT7932_patch_mcu_1_2_hdr.bin`, 33 568 B) | build date `20260331234651a`, platform tag `ALPS`, HW/SW version word `0x8A10_8A10` (BE32 at offset `0x14`), format discriminator `0xFFFF_FFFF` (BE32 at offset `0x18`), global-descriptor patch-version word `0x4433_2211` (BE32 at offset `0x20`), **1 section** (destination `0x0090_0000`, length `0x8280`) | [C] |
| Manufacturing RAM code (`WIFI_MFG_MT7932_2.bin`) | build tag `t-neptune-custom-mt7932-ccn9-2405-mfg-MT7932_2405_CCN9_x86_Gen4m_TEST_MODE-20250821172010`; source tree `build/csp/**7923**/asic2.0/projects/wifi_mobile_ram_rf_test_mode_2/…` and `wifi/base/**7923**/prj/exthal/cal/…` | [C] |
| MFG ROM patch | build date `20250821172242a` | [C] |
| Firmware log index file | firmware source file list; the ASIC-specific layer is named `*_falcon.c` (`wh_hif_falcon.c`, `wh_rxd_falcon.c`, `wh_txd_falcon.c`, `wh_sys_falcon.c`, `wh_phy_falcon.c`, `wh_ser_falcon.c`, `wh_hwcfg_falcon.c`, …) | [C] |

Two things follow. First, **the MT7932 firmware is built from the MT7923 project tree**
(`csp/7923/asic2.0`, `base/7923`) — the MAC/PHY IP is the MT7923 one, "ASIC 2.0". [C] Second,
the manufacturing image still contains the string `DBDC band :%d not support in MT7961`,
confirming the direct MT7961/MT7921 codebase lineage. [C]

The bundle also carries a second, development-labelled copy of the RAM code and ROM patch;
these are separate builds from the production images, not duplicates. [C]

---

## 2. Chip-ID-conditional behaviour — exhaustive enumeration

### 2.1 Scope of the enumeration

The list below is exhaustive for the host interface: every point at which MT7932 behaviour
differs from MT7922 is either a run-time test on the PCI device ID or a comparison against
the chip-ID field, and both sets are enumerated in full. The chip identities `0x7961`,
`0x7925`, `0x7927`, `0x793B` and `0x7663` play no part in the MT7932 host interface. The
device ID `0x0616` is only an alias for chip ID `0x7922` and does not affect MT7932.

### 2.2 The deltas

Every branch is binary: MT7922 on one side, MT7923 **and** MT7932 on the other. There is no
branch anywhere that distinguishes MT7932 from MT7923.

| # | What is being selected | MT7922 behaviour | **MT7932 behaviour** | Conf. |
|---|---|---|---|---|
| 1 | **Wi-Fi function power/ready state source** (checked at adapter start, at power-off, and on wake) | Poll chip register `sw_sync0` = `0x7C06_00F0` (`CONN_CFG_ON` `CONN_ON_MISC`), ready bits `0x3` at shift 0 | Poll **PCI configuration space dword `0x48C`, field bits `[19:16]`; value `2` = Wi-Fi function on/ready**. Poll interval 5 ms, timeout 5000 ms; a timeout must escalate to a chip reset | [C] |
| 2 | **PCIe fabric sanity check** before/around bus access | Not performed | Read PCI config dword **`0x488`** and require: bit0 = 1, bit1 = 1, bit4 = 0, bit6 = 1, bit9 = 1, bit25 = 1, bit26 = 1 ("Fabric 1.1/1.2"). In one additional mode also require bit10 = 0, bit13 = 1, bit14 = 1, bit15 = 1 ("Fabric 2") | [C] |
| 3 | **MTCMOS hardware-mode sequence** at PCIe power transition | Not performed | Performed, after taking driver ownership. Two variants, selected by whether the permanent CB-TOP window has already been established and verified (`0x7C00_E250[31:16] == 0x7000`, see §2 of the power/reset section): (a) **window established** — write `0x0000_0000` to chip address `0x7000_3020` through the static map, wait 1000 µs; (b) **window not established** — write `0x1845_7000` to `0x7C00_E24C`, wait 2 µs, write `0x0000_0000` to BAR offset `0x43020` (the same register through the borrowed window), wait 1000 µs, write `0x1845_184F` to `0x7C00_E24C` to restore the default remap | [C] |
| 4 | **"Clear own" IRQ-status write** during MCU init | Write `0x0000_0003` to chip address `0x7C00_1620`, then sleep 2 ms | **Skipped entirely** | [C] |
| 5 | **ROM-patch semaphore request value** (`INIT_CMD_ID_PATCH_SEMAPHORE_CONTROL`, ID `0x10`, 68-byte command) | `ucGetSemaphore = 1` (public `PATCH_GET_SEMA_CONTROL`) | **`ucGetSemaphore = 2`** | [C] |
| 6 | **ROM-patch "already downloaded" status value** in `INIT_EVENT_ID_PATCH_SEMA_CTRL` | `ucStatus == 1` (`PATCH_STATUS_NO_NEED_TO_PATCH`) | **`ucStatus == 2`** | [C] |
| 7 | **Init-event success status code** for the WIFI-function start/stop command | `ucStatus == 0` means success | **`ucStatus == 1` means success** | [C] |
| 8 | **Init-event status-code enumeration** (see §2.3) | 7 codes, `0`-based | **12 codes, `1`-based, different meanings** | [C] |
| 9 | **DBDC setting command** (`CMD_DBDC_SETTING`, CMD ID `0x28`, 36-byte payload) | **payload byte `0x04` = `0`**; when enabling, `ucWmmBandBitmap` (payload byte 1) is derived from the single infrastructure BSS | **payload byte `0x04` = `2`**; when enabling, a 16-bit per-BSS band bitmap is written at payload offset `0x0C` (OR of the band masks of all connected BSSes), and `ucWmmBandBitmap` is left zero. The selector is a **payload** byte, not the command header's `ucCmdVersion` field, which stays `0` for every command on this part | [C] |
| 10 | **One-time-calibration 2.4 GHz channel-group block counts** (also taken by MT7923) | band0 = 2 blocks, band1 = 1 block | If the calibration data's module-enable byte has bit 7 set: band0 = **4** (module version 1) or **8** (otherwise); band1 = **2** (module version 1) or **4** (otherwise). Otherwise the MT7922 defaults apply | [C] |
| 11 | **6 GHz capability TLV** (`TAG_CAP_6G_CAP`, tag `0x18`) | Honoured | Honoured (**this is where MT7932 differs from MT7923**: the TLV is discarded outright when chip ID == `0x7923`) | [C] |
| 12 | **PCIe config-space diagnostic dump** | Not performed | Dumps config dwords starting at `0x488` | [C] |

Two further chip-ID comparisons exist and produce **no** MT7932-specific behaviour:

* The legacy Wi-Fi device ID `0x0616` is an alias for chip ID `0x7922`; MT7932 is
  unaffected. [C]
* A host-side transmit-deadline quantiser uses a coarser divisor for device ID `0x7923`;
  MT7932 takes the default (MT7922) path. [C]

### 2.3 Init-event status-code enumeration

This is a firmware-interface delta that will silently break a ported driver, so it is given in
full.

| Status | MT7922 meaning | MT7932 / MT7923 meaning |
|---|---|---|
| 0 | success | *(unused)* |
| 1 | invalid param | **success** |
| 2 | invalid crc | unknown |
| 3 | decrypt fail | invalid param |
| 4 | unknown | invalid crc |
| 5 | timeout | timeout |
| 6 | sec boot fail | sec boot check fail |
| 7 | — | region check fail |
| 8 | — | cmd size check fail |
| 9 | — | RAM entry check fail |
| 10 | — | Section check fail |
| 11 | — | FW download flow check fail |
| 12 | — | FW download cmd logic check fail |

[C]. The MT7932 ROM performs materially more validation on downloaded firmware (region, RAM
entry point, section, download flow and command-logic checks) than MT7922 does.

### 2.4 What does *not* differ — the confident negative

Beyond the twelve items above, **nothing in the host interface is conditional on the chip ID.**
Specifically, all of the following are byte-for-byte identical between the MT7922 and MT7932
hardware-configuration records, and are therefore shared:

the bus/HIF parameters; the firmware-download procedure; the TX- and RX-descriptor formats;
the ATE/test and debug interfaces; the MCU handshake registers; `should_verify_chip_id`
(false); `sw_sync0` (`0x7C06_00F0`); `sw_ready_bits` (`0x3`); `sw_ready_bit_offset` (0);
patch load address (`0x0090_0000`); CR4/WA-CPU support (both absent); TXD append size (32 B);
RXD size (24 B); PSE header length (8 B); init-event size (8 B); event header size (12 B);
NIC-capability V1 flag (false → V2 query always used); eFuse support (true);
`TOP_HCR`/`TOP_HVR`/`TOP_FVR`; ARB AC-mode register address (`0x820E_315C`); the ECO table;
the buffer-bin/PPR/calibration file-name templates; the SER and health-monitor interfaces;
the MCU register-map programming; the command size limit; the WFDMA address ranges and
re-initialisation procedure; the DMASHDL configuration; the LMAC/UMAC WTBL data-unit control
addresses (`0x820D_4200` / `0x820C_4094`, which are also the public MT7921 values —
see the MAC-tables section §0); the PCIe→chip static address map; the per-chip
**feature word, which is `0x0000_0000` on both parts**; hardware A-MSDU support (true);
TX-power-limit table name and batch size (8); ASIC low-power support (true); WFDMA1 support
(false); DMA-shaper support (false); the MMIO fabric-check, mapping and all-ones-whitelist
rules; the TXD sanity rules; the secure-FWDL protection sequence. [C]

The only *semantic* difference between the two records is the chip-ID word itself. [C]
A byte-level comparison shows one further difference, in the low bits of one function pointer
(the per-rate-power file-name constructor): that is a Mach-O chained-fixup link field, an
artefact of the two records lying at different offsets within their pages, and the pointer
target and its authentication attributes are identical. [C] Do not mistake it for a
behavioural delta.

---

## 3. Capabilities

### 3.1 Spatial streams and antenna chains

| Item | Value | Evidence | Conf. |
|---|---|---|---|
| Spatial streams | **2** (2×2) | host-side ceiling of 2, min-combined with the firmware-reported `ucNss` | [C] that the host ceiling is 2; [L] that the silicon is 2×2 — the value firmware reports is listed as unknown in §5.2 |
| RF chains | **2** (core 0, core 1) | the shipped secure-ToF group-delay calibration table has exactly `Core0`/`Core1` entries for 2.4 GHz and for 5 GHz UNII-1/2/3; the antenna-gain table has `siso`/`cdd`/`mimo` columns | [C] |
| Per-band RF path mask | reported at run time in `CAP_PHY_CAP.ucWifiPath` (BIT0 2G4_WF0, BIT1 5G_WF0, BIT2 2G4_WF1, BIT3 5G_WF1) and, for 6 GHz, `CAP_6G_CAP.ucHwWifiPath` (BIT0 6G_WF0, BIT1 6G_WF1) | capability query | [C] |
| DBDC (dual-band dual-concurrent) | supported; enabled/disabled by `CAP_PHY_CAP.ucDbdc`; A+A (5 GHz + 6 GHz simultaneous) support and its minimum frequency separation are reported by `CAP_6G_CAP` | capability query; firmware log string reporting `u2WifiDBDCAwithA` and `MinimumFrqInterval` | [C] |
| Antenna swap | not present (`TAG_CAP_ANTSWP` is not handled) | capability table | [C] |

### 3.2 Bands and maximum bandwidth

| Band | Supported | Max bandwidth | Evidence | Conf. |
|---|---|---|---|---|
| 2.4 GHz | yes | **40 MHz** | power tables carry `cck`, `ofdm`, `ht20`, `ht40` for channels 1–13 and nothing wider | [C] band, [L] 40 MHz ceiling (inferred from the shipped tables) |
| 5 GHz | yes | **160 MHz** | power table sections `vht20/vht40/vht80/vht160` | [C] band, [L] 160 MHz ceiling (inferred from the shipped tables) |
| 6 GHz | **yes** | **160 MHz** | dedicated 6 GHz power-limit table covering UNII-5 through UNII-8 (5925–7125 MHz), LP/SP/VLP power classes, RU sizes up to `ru996X2`; antenna-gain table groups span 5925–7125 MHz; the 6 GHz capability TLV is *not* suppressed for chip ID `0x7932` | [C] band, [L] 160 MHz ceiling |
| Channel 14 | 802.11b/CCK only | — | OFDM/HT/VHT/HE explicitly disabled by the regulatory channel matrix | [C] |

The firmware-reported `CAP_PHY_CAP.ucMaxBandwidth` (0=20, 1=40, 2=80, 3=160, 4=80+80, 5=320)
is min-combined with the host's per-band setting, so it can only *reduce* these. [C]

### 3.3 PHY generation — this is a Wi-Fi 6E part, not Wi-Fi 7

| PHY | Supported | Conf. |
|---|---|---|
| 802.11a/b/g/n (HT) | yes | [C] |
| 802.11ac (VHT) | yes, incl. VHT160 | [C] |
| 802.11ax (HE), 2.4 / 5 / 6 GHz | yes | [C] |
| **802.11be (EHT)** | **no** | [C] |

Evidence, separated by source:

**Silicon/firmware evidence (dispositive):**
1. The 6 GHz and 5 GHz TX-power-limit tables shipped for this part contain **only** HE-era
   sections: `cck`, `ofdm`, `ht20/40`, `vht20/40/80/160`, and RU sizes `ru26 … ru996X2`. There
   is **no** `ru4x996`/320 MHz section and **no** MCS 12/13 column anywhere; every MCS column
   set runs `m0…m11`. A Wi-Fi 7 part's SKU tables cannot be expressed in this schema. [C]
2. The firmware log-symbol file for the shipped RAM code lists the complete firmware source
   file set. It contains `he_rlm.c` and HE HTC/OM handling, and contains **no** EHT, 802.11be,
   MLO, multi-link, or 320 MHz strings or source files at all. [C]
3. The manufacturing firmware's own certification string is `WiFi6 Certification: …`. [C]
4. The firmware ASIC layer is the MT7923 project tree (`csp/7923/asic2.0`) — MT7923 is a
   Wi-Fi 6 part; MT7932 adds 6 GHz to it. [C]
5. The `CAP_PHY_CAP` record for this part ends at `ucHe`; there is **no** `ucEht` field, and
   **no** `TAG_CAP_MLO_CAP` (`0x22`) tag is defined for it. [C]
6. The chip presents a CONNAC2 host interface (TXD v2 / RXD v2 / legacy CMD-EVENT). No
   shipping MediaTek Wi-Fi 7 part uses CONNAC2. [C]

**Non-evidence, explicitly discounted:** EHT and 320 MHz identifiers appear in the shipped
third-party user-space supplicant/AP components (`eht_enabled`, `EHT20/40/80/160/80+80/320`,
and similar). Those are generic 802.11be-aware user-space components and say nothing about
this silicon. An isolated 802.11be-labelled rate-mapping entry likewise exists on the host
side; it is not backed by any EHT capability field, EHT power table, EHT firmware module or
EHT capability TLV, and is treated here as forward-looking, unused. [C]

**Verdict: MT7932 is a 2×2 Wi-Fi 6E (802.11ax, tri-band, ≤160 MHz) part.** [L] — each of the
six evidence items above is [C], but the verdict itself is the conclusion drawn from them,
not a value that was read out. The **negative** half of it (no EHT/320 MHz/MLO anywhere in
this part's capability set, power tables, firmware module list or command bodies) is [C].

### 3.4 Multi-link operation

Not supported. There is no `TAG_CAP_MLO_CAP` tag for this part, the firmware
contains no MLO/multi-link module, and there is no `CHIP_CAP_MLO`-equivalent bit. [C]
(The unrelated `IDS_MLO` token in the firmware is an internal identifier in a different
subsystem and is not an 802.11be multi-link reference. [L])

### 3.5 BSS, station, aggregation and A-MSDU limits

All of these are ultimately reported by firmware at run time; the values below are the host's
accepted ranges and configured ceilings.

| Item | Value | Source | Conf. |
|---|---|---|---|
| Hardware BSSID count | **1–4**; values outside this range must be ignored | `CAP_MAC_CAP.ucHwBssIdNum`, bounded by public `MAX_BSSID_NUM` = 4 | [C] |
| WMM set count | reported by `CAP_MAC_CAP.ucWmmSet` (1 = AC0–3, 2 = AC0–3 plus AC10–13, …), clamped to ≥ 1 | capability query | [C] |
| WTBL / station-record entries | **1–49** accepted (`CFG_STA_REC_NUM` = 49 for this part; public gen4m default is 27, upstream mt76 `MT792x_WTBL_SIZE` is 20) | `CAP_MAC_CAP.ucWtblEntryNum` | [C] range, [U] actual value reported |
| TX A-MSDU build | **hardware** (`is_support_hw_amsdu` true; the TXD v2 hardware A-MSDU form is used); software A-MSDU sub-frame count is 0 | per-chip parameters | [C] |
| Max TX A-MSDU-in-A-MPDU length | **8192 octets** | host ceiling | [C] |
| RX max MPDU length | setting `2` → **11454 octets** (VHT max-MPDU encoding) | host setting | [C] |
| Frame-buffer capability (`TAG_CAP_FRAME_BUF_CAP`, tag `0x0A`) | TLV is returned but not acted on — TX A-MSDU sub-frame count, RX A-MSDU size, TXD count and packet-buffer size reported by firmware are not used | capability query | [C] |
| Block-ack window sizes | separately configurable for TX, RX-HT, RX-VHT, RX-HE and TX-HE | host settings | [C] |
| MCU command size limit | **`0x640` = 1600 octets for the whole buffer** (32-byte TXD + 32-byte command header + payload), i.e. a maximum payload of `0x600` = 1536 octets; the firmware-download port is exempt | command size check | [C] |
| Beamforming / MU-MIMO / location capability TLVs | returned but not acted on | capability query | [C] |

### 3.6 Hardware offloads

| Offload | Present | Notes | Conf. |
|---|---|---|---|
| RX header translation (802.11 → 802.3) | **yes** | RXD v2 carries a header-translated indicator; TKIP MIC handling has a header-translation variant | [C] |
| Hardware A-MSDU build (TX) | **yes** | see §3.5 | [C] |
| TX/RX checksum offload | RXD v2 carries the CONNAC2 checksum-status fields | the run-time capability tag for checksum offload (`TAG_CAP_CSUM_OFFLOAD`, `0x04`) is **not** part of this part's capability set, so the feature is assumed rather than negotiated | [C] tag absent, [L] feature present |
| RRO / hardware RX-reordering offload | **no** | no RRO capability and no RRO parameters for this part; RRO is a CONNAC3/Filogic-8xx feature | [C] |
| Packet classifier / packet filter | **yes** | firmware packet-filter module; a "set packet filter" command exists | [C] |
| Wake-on-WLAN | **yes**, enabled by default | GPIO wake pin 13, trigger level `0x5`, 1 wake pin, detect-type mask `0x114`, HIF wake mask `0x3`, wake on multi-DTIM | [C] |
| ARP / IPv6 NS offload | **yes** | firmware `arp_ns.c`; per-BSS IPv4/IPv6 address programming with a version field | [C] |
| Keep-alive offload | **yes** | firmware `keep_alive.c`; per-BSS, per-protocol, with protected-frame support | [C] |
| Bonjour / mDNS offload | **yes** | firmware `mdns.c`; host has enable/query/set entry points and a dedicated event | [C] |
| Directed Multicast Service (DMS) | **yes** | dedicated DMS offload enable and configuration commands (`0x74`, `0xE5`) | [C] |
| Suspend-mode offload | **yes**, four independent sub-capabilities negotiated at run time | capability tag `0x39` | [C] |
| Beacon-loss / link-detection offload, roaming offload, scheduled scan | **yes** | firmware modules `linkdt_*.c`, `roaming*.c`, `scan_cmd_sched_scan.c` | [C] |
| NAN / Wi-Fi Aware, AWDL | **yes**, incl. 6 GHz variants, negotiated at run time | capability tags `0x30`, `0x31` | [C] |
| Secure ToF / 802.11az ranging | **yes**, two sub-flags | capability tag `0x34`; firmware `tm_stof*.c`, `hal_toae.c` | [C] |
| Low-latency TX / RX paths | **yes**, independently negotiated | capability tags `0x32`, `0x33` | [C] |
| UWB coexistence, external PTA | **yes**, negotiated | capability tags `0x35`, `0x36` | [C] |
| SmartCCA | **yes**, negotiated | capability tag `0x3C`; firmware `smartcca.c` | [C] |
| Runtime (as opposed to one-time) calibration | negotiated | capability tag `0x38` | [C] |
| Per-antenna RSSI reporting | negotiated | capability tag `0x3A` | [C] |

### 3.7 Run-time capability query — the authoritative mechanism

This is how a driver discovers everything above.

**Request.** After firmware is ready, the host sends the CONNAC2 **initial-phase (INIT) query
command for NIC capability V2** on the MCU command ring, using the CONNAC2 init-command TXD.
Nothing in the request is chip-specific. The V1 capability query is not used on this part
(`isNicCapV1` is false). [C]

**Response.** A single INIT event with event ID **`0xEC`**, delivered on the init-event path;
the host validates the RX packet type against the CONNAC2 SW-packet mask
(`u2RxSwPktBitMap = 0x380F`, event value `0x3800`) and the event ID before parsing. Response
buffer size allowed: **2352 octets** (`0x930`). Wait timeout is the firmware-download timeout. [C]

**Payload format** (public gen4m `EVENT_NIC_CAPABILITY_V2`):

```
u16  u2TotalElementNum
u16  reserved
[ per element:
    u32 tag_type
    u32 body_len          ; length of body only
    u8  body[body_len]    ; element stride = body_len + 8
] × u2TotalElementNum
```
[C]

**Tags defined for this part** (31 entries). Tags not listed may be skipped.

| Tag | Public gen4m name | Payload / effect | Conf. |
|---|---|---|---|
| `0x01` | `TAG_CAP_TX_EFUSEADDRESS` | eFuse start address / size | [C] |
| `0x02` | `TAG_CAP_COEX_FEATURE` | 32-bit coexistence feature word | [C] |
| `0x03` | `TAG_CAP_SINGLE_SKU` | regulatory single-SKU table delivered by firmware | [C] |
| `0x05` | `TAG_CAP_HW_VERSION` | `CAP_HW_VERSION`: product ID, ECO version, MAC IP ID, BB IP ID, TOP IP ID, configuration ID | [C] |
| `0x06` | `TAG_CAP_SW_VERSION` | `CAP_SW_VERSION`: FW version, build number, 4-byte branch tag, 16-byte date code | [C] |
| `0x07` | `TAG_CAP_MAC_ADDR` | factory MAC address | [C] |
| `0x08` | `TAG_CAP_PHY_CAP` | `CAP_PHY_CAP`; see below | [C] |
| `0x09` | `TAG_CAP_MAC_CAP` | `CAP_MAC_CAP`: HW BSSID count, WMM set, WTBL entry count | [C] |
| `0x0A` | `TAG_CAP_FRAME_BUF_CAP` | accepted, **discarded** | [C] |
| `0x0B` | `TAG_CAP_BEAMFORM_CAP` | accepted, **discarded** | [C] |
| `0x0C` | `TAG_CAP_LOCATION_CAP` | accepted, **discarded** | [C] |
| `0x0D` | `TAG_CAP_MUMIMO_CAP` | accepted, **discarded** | [C] |
| `0x14` | `TAG_CAP_HW_ADIE_VERSION` | A-die product ID (16-bit) stored; ECO version discarded | [C] |
| `0x17` | *(gen4m assigns `TAG_CAP_P2P` here)* | **WFDMA reallocation descriptor**: byte 1 = "firmware supports WFDMA reallocation", dword at offset 4 = reallocation parameters; if byte 2 is non-zero the host overwrites one 32-bit **ring-index** field of the per-chip bus descriptor, whose static value is **`17`**, with **`18`** `[C]`. It is a ring *index*, not a ring count. The field's neighbour also holds `17`, and on this part the MCU-command ring index and the "WA command" ring index are both `17`, so the effect is to move a command class from hardware TX ring 17 to hardware TX ring 18 (16 descriptors, done-interrupt bit 30) `[L]` | |
| `0x18` | `TAG_CAP_6G_CAP` | `CAP_6G_CAP` `{ucIsSupport6G, ucHwWifiPath, ucWifiDBDCAwithA, ucWifiDBDCAwithAMinimumFrqInterval}`; the last field is in units of 5 MHz. **Ignored for chip ID `0x7923`; honoured for `0x7932`** | [C] |
| `0x1E` | *(gen4m assigns `TAG_CAP_REDL_INFO`)* | 1-byte maximum RMAC quota | [C] |
| `0x1F` | *(gen4m assigns `TAG_CAP_HOST_SUSPEND_INFO`)* | MLME-offload address-randomisation capability | [C] |
| `0x20` | *(gen4m assigns `TAG_CAP_MLR_CAP`)* | roaming / BSS-transition policy | [C] |
| `0x30` | vendor | 6 GHz support for the peer-to-peer discovery mode | [C] |
| `0x31` | vendor | 32-bit NAN feature bitmap (bits 0–6 individually consumed) | [C] |
| `0x32` | vendor | low-latency TX path | [C] |
| `0x33` | vendor | low-latency RX path | [C] |
| `0x34` | vendor | secure-ToF, 2 sub-flags | [C] |
| `0x35` | vendor | UWB coexistence | [C] |
| `0x36` | vendor | external PTA | [C] |
| `0x37` | vendor | time-sync accuracy improvement | [C] |
| `0x38` | vendor | runtime-calibration indicator | [C] |
| `0x39` | vendor | suspend-mode offload, 4 sub-flags | [C] |
| `0x3A` | vendor | per-antenna RSSI in reports | [C] |
| `0x3B` | vendor | negotiated PCIe link speed | [C] |
| `0x3C` | vendor | SmartCCA | [C] |

**`CAP_PHY_CAP` field consumption** (this is the authoritative source for §3.1–§3.3):

| Offset | Field | Consumed as | Conf. |
|---|---|---|---|
| 0 | `ucHt` | *not consumed* | [C] |
| 1 | `ucVht` | ANDed into the STA/AP/P2P-GO/P2P-GC VHT enables; if 0, VHT is disabled | [C] |
| 2 | `uc5gBand` | if 0, 5 GHz is disabled | [C] |
| 3 | `ucMaxBandwidth` | min-combined into the per-band maximum bandwidth | [C] |
| 4 | `ucNss` | min-combined into the spatial-stream count | [C] |
| 5 | `ucDbdc` | if 0, DBDC mode is forced off | [C] |
| 6–9 | `ucTxLdpc`,`ucRxLdpc`,`ucTxStbc`,`ucRxStbc` | ANDed into the corresponding enables | [C] |
| 10 | `ucWifiPath` | stored as the per-band RF-path mask | [C] |
| 11 | `ucHe` | if 0, HE is disabled | [C] |
| 12 | `ucEht` | **field does not exist in this interface** | [C] |

If the capability query is not answered, the host must fall back to a fixed default profile. [C]

---

## 4. Relationship to public parts

Rows marked **confirmed** are directly established; rows marked *inferred* follow from the
confirmed identity of the MT7932 and MT7922 hardware-configuration records but are not
independently established.

| Axis | MT7921 / MT7961 | MT7922 | MT7925 | MT7927 | **MT7932** | Match? |
|---|---|---|---|---|---|---|
| CONNAC generation | CONNAC2 | CONNAC2 | CONNAC3 | CONNAC3 (Filogic 380) | **CONNAC2** | = MT7922 **confirmed** |
| PHY class | Wi-Fi 6 (2.4/5) | Wi-Fi 6E (2.4/5/6) | Wi-Fi 7 | Wi-Fi 7, EHT 320 MHz | **Wi-Fi 6E, ≤160 MHz** | = MT7922 **confirmed** |
| Spatial streams | 2×2 | 2×2 | 2×2 | 2×2 | **2×2** | = **confirmed** |
| TX descriptor generation | `mt76_connac2` TXD, 32-byte MAC-TXD append | same | `mt76_connac3` TXD | connac3 (mt7925 TXP format) | **v2, 32-byte append**, v2 TXD operation set incl. hardware-A-MSDU template | = MT7921/22 **confirmed** |
| RX descriptor generation | `mt76_connac2` RXD, 24 bytes | same | `mt76_connac3` RXD | connac3 | **v2, 24 bytes**, v2 RXD operation set (header-translation, offload, TCL, BSSID, radiotap, wake-reason accessors) | = MT7921/22 **confirmed** |
| MCU command scheme | legacy CONNAC2 CMD/EVENT (`mt76_connac_mcu`) | same | **unified/TLV** (UNI commands) | unified/TLV | **legacy CONNAC2 CMD/EVENT**; *no* unified-command traffic in either direction | = MT7921/22 **confirmed** |
| Init/FWDL command set | CONNAC2 INIT commands | same | connac3 | connac3 | **CONNAC2 INIT commands**, but with the shifted status enumeration and patch-semaphore values of §2.2/§2.3 | ≈ MT7922 with deltas, **confirmed** |
| Command header size | 64 B (`CONNAC2X_WIFI_CMD`) | same | larger (UNI) | UNI | **64 B** | = **confirmed** |
| RX SW packet-type mask / event / frame | `0x380F` / `0x3800` / `0x3801` | same | connac3 values | connac3 | **`0x380F` / `0x3800` / `0x3801`** | = **confirmed** |
| WFDMA ring assignment (host TX) | FWDL = ring 16, MCU CMD = ring 17 | same | different | different | **FWDL = ring 16, MCU CMD = ring 17** | = **confirmed** |
| WFDMA1 present | no | no | yes | yes | **no** (`is_support_wfdma1` false) | = MT7921/22 **confirmed** |
| DMA scheduler (DMASHDL) base | host view `0x7C02_6000` (BAR `0x0D6000`); the bus descriptor additionally carries the CONNAC2 `HIF_DMASHDL` constant `0x5200_0000`, which is the MCU/AXI-internal-bus view of the same block | same | different | different | **host view `0x7C02_6000`; `0x5200_0000` carried but never used by a host access** | = **confirmed** |
| Interrupt-enable poke at cap-init | write BIT(0) to `CONNAC2X_BN0_IRQ_ENA` `0x7C06_0018` | same | different | different | **write `0x1` to `0x7C06_0018`** | = **confirmed** |
| Interrupt bit assignment | CONNAC2 host-int layout; MSI layout table | same | connac3 layout | connac3 + MT7927-specific IRQ map | **CONNAC2 layout; the MT7922 MSI layouts apply unchanged** | = MT7922 *inferred from record identity, high confidence* |
| Static PCIe-BAR → chip-address window | `< 0x100000` (1 MiB) in upstream `mt76`, but `0x000F_0000` in gen4m `mt7961` | **`0x0010_0000`** (1 MiB) | `< 0x200000` (2 MiB) | 2 MiB + CBTOP remap | **`0x0010_0000`** (1 MiB) | = MT7922 **confirmed**; larger than gen4m `mt7961`'s `0x000F_0000` |
| Static map table | MT7961 `bus2chip` table | MT7922 table | much larger mt7925 table | mt7925 table + MT7927 entries | **the MT7922 table verbatim** | = MT7922 **confirmed** |
| Remap mechanism | `MT_HIF_REMAP_L1`-style HIF remap inherited from MT7915; **no conn-infra PCIe2AP remap descriptor** | conn-infra **PCIe2AP** remap array at `0x7C00_E24C` / `0x7C00_E250` / `0x7C00_E254` (16-bit page selectors, two per register, 64 KiB granularity) | L1/L2 + extras | + `PCIE2AP_REMAP_WF` | **the same PCIe2AP remap array; `0x7C00_E24C` default `0x1845_184F`, `0x7C00_E250` programmed `0x7000_1846`, `0x7C00_E254` programmed `0x1805_1848`** | = MT7922 **confirmed**; a real delta versus public MT7921 |
| DMA address width | 32-bit | 32-bit | 34-bit (32-bit fallback) | 34-bit | **32-bit** | = MT7921/22 **confirmed** |
| Patch load address | `0x0090_0000` | same | different | different | **`0x0090_0000`** | = **confirmed** |
| `sw_sync0` / ready bits | `0x7C06_00F0`, bits `0x3`, shift 0 | same | different | different | **the same values are defined, but MT7932 does not use them for the readiness check** (see §2.2 item 1) | definition = MT7922 **confirmed**; use differs **confirmed** |
| eFuse support | yes | yes | yes | yes | **yes** | = **confirmed** |
| CR4 / WA co-processor | no / no | no / no | no | no | **no / no** | = **confirmed** |
| RRO / MAWD / SDO host offloads | none | none | none (mt7925) | none | **none** | = **confirmed** |
| Chip-ID readback register | `TOP_HCR`-family | same | connac3 | connac3 | **`0x8002_1008`** (verification disabled) | = MT7922 **confirmed** |
| ECO table | `{0x00,0x00,0x0A,0x01}`, `{0x10,0x01,0x0A,0x02}` | shared MT792x table | `{0x00,0x00,0x0A,0x01}` only | — | **`{0x00,0x00,0x0A,0x01}`, `{0x01,0x01,0x0A,0x02}`** | differs from MT7961 in the E2 `ucHwVer` **confirmed** |
| Firmware image set | `WIFI_RAM_CODE_MT7961_1.bin` + `WIFI_MT7961_patch_mcu_1_2_hdr.bin` | MT7922 equivalents | MT7925 set | MT6639 set (`WIFI_RAM_CODE_MT6639_2_1.bin`) | **own set**: RAM `W7932_2.bin`, patch `WIFI_MT7932_patch_mcu_1_2_hdr.bin`, plus MFG variants, a 2560-byte EEPROM image, a per-rate-power blob, plain-text SKU/SAR/antenna-gain tables and a log index file | MT7932-specific **confirmed** |
| Bluetooth function | USB (`0e8d:7961`) | PCIe `14C3:792A` | PCIe | PCIe (`0489:e1xx` USB-over-PCIe modules) | **PCIe `14C3:793B`, MT7923-class BT core, IPC transport** | ≈ MT7923 **[L]** |

**Summary:** on every axis that matters to a driver's DMA, descriptor, interrupt and MCU
plumbing, MT7932 is an MT7922. It is *not* related to MT7925/MT7927 in any of these respects —
those are CONNAC3 parts with a different descriptor format, a different command scheme, a
2 MiB window and 34-bit DMA. Anyone tempted to start from the `mt7925` driver because "7932 >
7925" would be starting from the wrong place.

---

## 5. Driver-author delta summary

### 5.1 If you have a working MT7921/MT7922 driver, change this and only this

1. **Match** PCI `14C3:7932` (Wi-Fi) and, separately, `14C3:793B` (Bluetooth). Set the internal
   chip ID from the device ID.
2. **Firmware paths.** Load the MT7932 ROM patch and RAM code; the RAM image is ~1.19 MB with a
   standard CONNAC2 5-region trailer (`format_ver = 2`, `format_flag = 1`, `eco_code = 1`, i.e. stepping **E2**).
   Load the 2560-byte EEPROM/calibration image, the per-rate-power blob and the plain-text
   TX-power-limit/SAR/antenna-gain tables. Do **not** expect an MT7922 firmware to work.
3. **Wi-Fi function ready/on detection.** Stop polling `CONN_ON_MISC` (`0x7C06_00F0`) ready
   bits. Poll **PCI config dword `0x48C`, bits `[19:16]`**; `2` means the Wi-Fi function is on.
   Use this both when waiting for "on" after firmware start and when confirming "off" after
   power-down. 5 ms poll, 5 s timeout.
4. **Add the MTCMOS hardware-mode sequence** at PCIe power transitions (item 3 of §2.2). It is
   absent from MT7921/MT7922 drivers and is required here.
5. **Remove the "clear own" IRQ-status write** to `0x7C00_1620` from the MCU-init path — MT7932
   must not perform it.
6. **Optionally add the PCIe fabric check** on PCI config dword `0x488` (bits 0,1,6,9,25,26
   must be 1; bit 4 must be 0). Treat a failure as "bus is dead", not as a chip error.
7. **ROM-patch semaphore:** send `ucGetSemaphore = 2`, and treat reply `ucStatus == 2` (not 1)
   as "patch already downloaded, skip download".
8. **Init-event status codes:** success is `1`, not `0`. Re-map the whole enumeration per §2.3
   or you will misreport every firmware-download failure.
9. **DBDC command:** set **payload byte `0x04` to `2`** (not the header's `ucCmdVersion`,
   which stays 0) and put the 16-bit union of per-BSS band masks at payload offset `0x0C`;
   leave `ucWmmBandBitmap` zero.
10. **One-time calibration:** if you implement it, use the larger 2.4 GHz block counts
    (4/8 for band 0, 2/4 for band 1, selected by the calibration module version) whenever the
    calibration blob's module-enable byte has bit 7 set.
11. **Enable 6 GHz.** Honour `TAG_CAP_6G_CAP`; register a 6 GHz band with HE, up to 160 MHz,
    and load the 6 GHz SKU table. (This is the one thing MT7932 has that MT7923 does not.)
12. **Do not** add EHT/320 MHz/MLO advertisement.

Everything else — WFDMA ring layout and prefetch, interrupt mask and status bits, TXD v2 and
RXD v2 formats and every field in them, the token/ID scheme, the legacy CMD/EVENT MCU
protocol, the DMA scheduler, the static address map and the L1/L2 remap, SER/L0.5/L1 error
recovery, the debug/health-monitor register sets, the 1 MiB window and 32-bit DMA mask —
is bit-identical to MT7922. Do not re-derive it.

### 5.2 Still unknown; must be established by hardware tracing

* The values MT7932 firmware actually reports in the capability query: `ucNss`,
  `ucMaxBandwidth` per band, `ucWifiPath`, `ucHwBssIdNum`, `ucWmmSet`, `ucWtblEntryNum`,
  the A-die product ID and ECO version, the MAC/BB/TOP IP IDs and the configuration ID, and
  whether 5+6 GHz DBDC A+A is advertised and with what minimum frequency separation.
* The maximum bandwidth on 6 GHz. 160 MHz is inferred from the RU sizes present in the 6 GHz
  SKU table; it has not been read out of firmware.
* The real WTBL depth. Up to 49 station records are accepted here — larger than the public
  gen4m (27) and mt76 (20) bounds — which hints at a bigger table, but the hardware value is
  unconfirmed.
* Whether TX/RX checksum offload is actually enabled in MT7932 firmware. The capability tag
  that would negotiate it is not part of this part's capability set.
* The full field semantics of PCI config registers `0x488` and `0x48C` beyond the bits checked
  here, and the meaning of the remaining bits of `0x488`.
* The exact semantics of capability tag `0x17` (WFDMA reallocation) beyond the ring-index
  substitution described above, and whether MT7932 firmware ever sets it.
* Whether the MT7932 silicon actually exists in an E1 stepping, and what `TOP_HVR`/`TOP_FVR`
  read on shipping parts. Only the E1/E2 table entries are known.
* The second identity dword in the EEPROM PCIe-identity records (`0x0000_A210` on the Wi-Fi
  function, `0x0000_0000` on the Bluetooth function).
* Whether the Bluetooth function of a real MT7932 accepts the MT7923 IPC RAM image, and what
  its HCI local-version / chip-ID readback reports.
* Whether any MT7932 stepping exists on which the `0x7923` behaviours (6 GHz suppression, the
  coarse TX-deadline quantiser) apply.

---

## 6. Open questions / needs hardware tracing

Consolidated from §5.2; the highest-value traces, in order:

1. Capture the NIC-capability-V2 event (INIT event `0xEC`) from a live MT7932 and decode every
   TLV. This single trace resolves spatial streams, bandwidth per band, RF paths, BSS and WTBL
   counts, A-die identity, IP versions, 6 GHz A+A support, and every negotiated offload.
2. Read `TOP_HVR` (`0x7001_0204`) and `TOP_FVR` (`0x8800_0004`) via the MCU register-access
   command on shipping silicon to pin the stepping.
3. Dump PCI config space `0x480`–`0x4A0` in both the Wi-Fi-off and Wi-Fi-on states to fully
   characterise the fabric-status (`0x488`) and function-state (`0x48C`) registers.
4. Confirm that omitting the `0x7C00_1620` "clear own" write and adding the MTCMOS sequence is
   sufficient for a clean cold start and for D3→D0 resume.
5. Attach the Bluetooth function and read its HCI local version / chip-ID to confirm the
   MT7923-class identity.


---

# MT7932 — Initialisation State Machine

**Scope.** The ordering constraints the silicon imposes on bring-up, expressed as a state
machine. Each state is defined by a *hardware precondition* that is true on entry, the
*hardware actions* that advance it, and the *observable signal* that confirms the
transition. Register addresses, bit encodings and timeouts are specified in the sections
referenced from each state; they are not repeated here.

This section deliberately describes **only** what the hardware requires. It does not
prescribe how a driver structures the work (threads, work queues, callbacks, retry
policy); any ordering not stated here is free.

Confidence: the **sequence** below is `[C]` — it is the order the analysed bring-up actually
performs, and the exit signals are the conditions it actually tests. That each ordering
constraint is *imposed by the silicon* rather than being host convention is `[L]` unless the
referenced section establishes it for that step; the two places where the hardware constraint
is explicit are S2 (ownership before Wi-Fi-domain MMIO) and S6 (prefetch and ring registers
before the DMA enables). Individual register details carry their own markers in the
referenced sections.

---

## 1. State diagram

```
                    ┌──────────────────────────────────────────────┐
                    │                                              │
                    ▼                                              │ L0 (full re-probe)
   ┌───────────────────────────────┐                               │
   │ S0  FUNCTION_PRESENT          │                               │
   │  PCI function enumerated      │                               │
   └───────────────┬───────────────┘                               │
                   │ memory space + bus master enabled, D0,        │
                   │ 32-bit DMA mask, BAR0 mapped, MSI allocated   │
                   ▼                                               │
   ┌───────────────────────────────┐                               │
   │ S1  BUS_VALIDATED             │  §1 §2                        │
   │  fabric/clock status good,    │                               │
   │  MMIO returns sane reads      │                               │
   └───────────────┬───────────────┘                               │
                   │ driver-own requested and granted              │
                   ▼                                               │
   ┌───────────────────────────────┐◄──────────────────┐           │
   │ S2  HOST_OWNED                │  §2               │           │
   │  WF register domain           │                   │           │
   │  accessible to the host       │                   │           │
   └───────────────┬───────────────┘                   │           │
                   │ remap windows programmed + verified│           │
                   ▼                                    │           │
   ┌───────────────────────────────┐                    │           │
   │ S3  WINDOWS_PROGRAMMED        │  §1.5              │           │
   └───────────────┬───────────────┘                    │           │
                   │ host WFDMA stopped, idle confirmed │           │
                   ▼                                    │           │
   ┌───────────────────────────────┐                    │           │
   │ S4  DMA_QUIESCED              │  §3.9              │           │
   └───────────────┬───────────────┘                    │           │
                   │                                    │           │
        ┌──────────┴───────────┐                        │           │
        │ subsystem already    │ no                     │           │
        │ running?             ├────────────┐           │           │
        └──────────┬───────────┘            │           │           │
                   │ yes                    │           │           │
                   ▼                        │           │           │
   ┌───────────────────────────────┐        │           │           │
   │ S5  SUBSYSTEM_RESET           │  §2.5  │           │           │
   │  WFSYS reset, sw-init-done    │────────┘           │           │
   └───────────────────────────────┘  re-acquire own ───┘           │
                   │                                                │
                   ▼                                                │
   ┌───────────────────────────────┐                                │
   │ S6  RINGS_READY               │  §3                            │
   │  rings + prefetch + scheduler │                                │
   │  programmed, TX/RX DMA on     │                                │
   └───────────────┬───────────────┘                                │
                   │ interrupt masks programmed                     │
                   ▼                                                │
   ┌───────────────────────────────┐                                │
   │ S7  INTERRUPTS_ARMED          │  §4                            │
   └───────────────┬───────────────┘                                │
                   │ revision read and resolved                     │
                   ▼                                                │
   ┌───────────────────────────────┐                                │
   │ S8  IDENTITY_KNOWN            │  §11.1                         │
   └───────────────┬───────────────┘                                │
                   │ patch + RAM image accepted, firmware started   │
                   ▼                                                │
   ┌───────────────────────────────┐                                │
   │ S9  FIRMWARE_RUNNING          │  §5                            │
   └───────────────┬───────────────┘                                │
                   │ function-ready indication observed             │
                   ▼                                                │
   ┌───────────────────────────────┐                                │
   │ S10 MCU_READY                 │  §5.7 §6                       │
   │  command/event channel live   │                                │
   └───────────────┬───────────────┘                                │
                   │ capability response parsed                     │
                   ▼                                                │
   ┌───────────────────────────────┐                                │
   │ S11 CAPABILITY_KNOWN          │  §11.3                         │
   └───────────────┬───────────────┘                                │
                   │ calibration + regulatory data accepted         │
                   ▼                                                │
   ┌───────────────────────────────┐                                │
   │ S12 RF_PROVISIONED            │  §9                            │
   └───────────────┬───────────────┘                                │
                   │ MAC address, BSS contexts, RX filter, EDCA     │
                   ▼                                                │
   ┌───────────────────────────────┐                                │
   │ S13 MAC_CONFIGURED            │  §10                           │
   └───────────────┬───────────────┘                                │
                   │ band/channel/bandwidth set, radio enabled      │
                   ▼                                                │
   ┌───────────────────────────────┐                                │
   │ S14 RADIO_READY               │  §10                           │
   │  scan / associate / transmit  │                                │
   └───────────────┬───────────────┘                                │
                   │                                                │
                   ├─── L1 (subsystem error recovery) ──► S6        │
                   ├─── L0.5 (Wi-Fi subsystem reset) ───► S4        │
                   └─── L0 (whole-chip reset) ──────────────────────┘
```

---

## 2. States

### S0 — FUNCTION_PRESENT

**Precondition.** The PCI function is enumerated. Configuration space is accessible; the
register aperture is not yet usable.

**Actions.** Enable memory space and bus mastering; place the function in D0; set a 32-bit
DMA mask; map BAR 0; allocate at least one MSI vector.

**Exit signal.** BAR 0 mapped. No device-side signal is involved. `[C]`

**Notes.** The register aperture is 1 MiB and only BAR 0 is used (§1.2). The DMA mask is
32-bit; this part has no address-extension mode (§1.2, §2).

### S1 — BUS_VALIDATED

**Precondition.** Configuration space is readable.

**Actions.** Read the internal fabric/clock status word and check the required bit pattern;
read the Wi-Fi function state field. Probe MMIO liveness with a read of a known-good
register.

**Exit signal.** The fabric bit pattern is satisfied and the liveness read returns neither
the bus-failure sentinel nor the no-response sentinel (§1.3.3).

**Notes.** The fabric check and the configuration-space function-state field are MT7932
behaviours; the sibling MT7922 uses a chip register for the equivalent state (§11.2). If
the function reports itself already running here, the subsystem must be brought back to its
initial state before bring-up continues — this is what selects the S5 branch below.

### S2 — HOST_OWNED

**Precondition.** The bus is validated.

**Actions.** Request driver ownership and poll the own-synchronisation bit until it clears.
Re-issue the request on the retry interval until the total timeout expires.

**Exit signal.** The own-synchronisation bit reads clear.

**Notes.** This is the gate on *all* Wi-Fi-domain MMIO `[C]` — the host refuses such access
until it is passed. That register reads in the WF domain are physically not meaningful before
it, including the ASIC revision registers, is `[L]`. Ownership is re-acquired after every
reset that can hand the chip back to firmware, so several later transitions re-enter this
state `[C]`.

### S3 — WINDOWS_PROGRAMMED

**Precondition.** Host owns the chip.

**Actions.** Program the remap window selectors for the chip regions that the fixed BAR map
does not cover but that bring-up needs. Read each selector back and honour the settle
delay before using the corresponding aperture.

**Exit signal.** Each selector reads back the intended value.

**Notes.** Failure here is not fatal if the host instead borrows a window per
access, but the windows must be programmed before any access to the regions concerned
(§1.5). The default value of any half-register that is not being changed must be preserved.

### S4 — DMA_QUIESCED

**Precondition.** Host owns the chip; register access is reliable.

**Actions.** Clear the global TX and RX DMA enables; poll the corresponding busy
indications until both are clear; assert and release the host-interface reset.

**Exit signal.** Both DMA busy bits read clear within the idle timeout.

**Notes.** Entering this state is mandatory before ring registers are reprogrammed. It is
also the re-entry point for the L0.5 recovery path.

### S5 — SUBSYSTEM_RESET (conditional)

**Entered when** the Wi-Fi subsystem is already running at S1, or when recovery
demands it.

**Actions.** Perform the Wi-Fi-subsystem reset sequence: assert the subsystem reset, hold
for the required interval, release, then poll the software-init-done indication. On this
part the power-domain (MTCMOS) sequence is part of this path, and the ownership-interrupt
status clear that the sibling part performs must be **omitted** (§2.5, §11.2).

**Exit signal.** The software-init-done bit is observed set.

**Transition.** Ownership must be re-acquired afterwards — return to S2, then proceed to
S6.

**Notes.** If a Bluetooth function is present in the same package, it must be notified
before a function-level reset of the Wi-Fi function (§2.5).

### S6 — RINGS_READY

**Precondition.** DMA is quiesced and the subsystem is in its initial state.

**Actions.** Allocate descriptor rings and receive buffers in DMA-coherent memory; program
each ring's descriptor base, entry count and initial indices; program the per-ring prefetch
windows so that no two overlap; program the DMA scheduler groups; fill the receive rings
and publish their producer indices; then set the global TX and RX DMA enables together with
the required global-configuration bits.

**Exit signal.** The global configuration register reads back with TX and RX DMA enabled.

**Notes.** The firmware-download ring and the MCU command ring must be live before S9. The
receive ring that carries pre-firmware events must be live before any boot-ROM command is
issued.

### S7 — INTERRUPTS_ARMED

**Precondition.** Rings are programmed and DMA is enabled.

**Actions.** Program the PCIe-MAC interrupt enable, then the WFDMA host interrupt enable
mask. Read the enable register back to push the write.

**Exit signal.** The enable register reads back the programmed mask.

**Notes.** Interrupts may equally be armed before enabling DMA `[L]` — see open question 1;
what is established is that both are done before firmware download begins, because download
progress is reported through the event ring `[C]`.

### S8 — IDENTITY_KNOWN

**Precondition.** The boot ROM is running and the command/event rings are live, or the
remap windows are programmed.

**Actions.** Read the hardware-version, ROM-version and factory-version fields — on this
part through the **boot-ROM register-access command**, before any firmware image is
downloaded, because the image file names are derived from the result (§5.6, §5.7) — and
resolve them to a revision (ECO) value through the revision table.

**Exit signal.** A revision value is resolved.

**Notes.** The chip-ID verification register exists but is not required on this part; the
chip identity is already known from the PCI device ID (§11.1). The revision selects
firmware image variants and a small number of calibration parameters.

### S9 — FIRMWARE_RUNNING

*(Ordering `[C]`; the "success status" caveat below is `[C]` and is the single most
implementation-critical item in this state.)*

**Precondition.** The firmware-download ring and the event ring are live; interrupts are
armed.

**Actions, in order.**
1. Acquire the ROM-patch semaphore (§5.6). If the reply says the patch is already resident,
   skip to step 5. **Both the request value and the "already resident" reply value differ
   from the public sibling parts** (§11.2). The host does not release this semaphore on the
   normal path.
2. For the ROM patch: declare each section's destination and length, then stream the
   section bytes over the firmware-download ring.
3. Acquire the secure-boot gate (§5.4) — required on this part, absent on public siblings —
   issue patch-finish, then release the gate.
4. For the RAM image: declare each region's destination, length and options, then stream
   the region bytes.
5. Acquire the secure-boot gate again, issue the firmware-start request naming the entry
   point, then release the gate.

The secure-boot gate is held only across the patch-finish and firmware-start commands, not
across the bulk data transfer, and it is taken twice — once for each of those two commands
(§5.4).

**Exit signal.** The firmware-start request is answered with the success status. **The
success status code on this part is not the public value** — see §11.2; a driver that
tests for the public value will read every successful start as a failure.

### S10 — MCU_READY

**Precondition.** Firmware has been started.

**Actions.** Poll the function-ready indication on the interval and budget given in §5.7.
On this part that indication is read from **configuration space**, not from the chip
register the sibling part uses; the source is selected by device identity and the cadence is
the same either way (§5.7, §11.2). Then switch the ownership handshake to the post-boot
path.

**Exit signal.** The ready indication is observed. Because the configuration-space field is
readable without ownership, without an MMIO mapping and without Wi-Fi-domain clocks, this
poll imposes no register-access precondition on this part.

**Notes.** From this point the command/event channel is the primary control interface, and
direct register writes to MAC/PHY blocks are largely superseded by commands (§10).

### S11 — CAPABILITY_KNOWN

**Actions.** Issue the capability query and parse the returned records.

**Exit signal.** The capability response is received and parsed.

**Notes.** This is the authoritative source for spatial streams, supported bands, maximum
bandwidth, station-table size, hardware BSSID count and the offloads the firmware will
honour (§11.3). Values configured later must be clamped to what is reported here. If the
query is not answered the part cannot be safely configured.

### S12 — RF_PROVISIONED

**Actions.** Supply the calibration/EEPROM image and any overlay blobs; supply the cached
one-time-calibration result if one exists; set the regulatory domain; push the
transmit-power limit tables.

**Exit signal.** Each upload is acknowledged.

**Notes.** This must precede any radio enable. Ordering among the calibration objects is
constrained — see §9.

### S13 — MAC_CONFIGURED

**Actions.** Program the MAC address; create the BSS context(s); program the receive filter;
program the channel-access (EDCA) parameters.

**Exit signal.** Each command is acknowledged.

### S14 — RADIO_READY

**Actions.** Select band, channel and bandwidth; enable the radio.

**Exit signal.** The channel-set command is acknowledged.

**Result.** Scanning, station creation, key installation and data transmission are
permitted (§10).

---

## 3. Recovery transitions

The part exposes three graded recovery levels. They differ in what is reset and therefore
in how far back the state machine must rewind.

| Level | Reset scope | Driven by | Rewind to | Host must re-do |
|---|---|---|---|---|
| **L1** — subsystem error recovery | MAC/PHY datapath and the DMA rings; firmware keeps running | Firmware, announced to the host; host acknowledges each phase | **S6** | Re-initialise rings and the parts of the DMA scheduler that firmware does not restore (§3.8); re-arm interrupts |
| **L0.5** — Wi-Fi subsystem reset | Whole Wi-Fi subsystem including the MCU; PCIe link and function survive | Host | **S4** (then S5, S2) | Full re-download of patch and firmware; all of S6–S14 |
| **L0** — whole-chip reset | Function-level reset; everything | Host | **S0** | Everything, including re-enumeration side effects |

Selection between them is driven by the fault indication observed (§2.6, §2.8). A
sub-system error latch, a DMA hang, a bus no-response sentinel and a firmware watchdog each
map to a different level.

**Ownership across recovery.** Any reset that reaches the MCU returns the chip to firmware
ownership. Ownership must be re-acquired (S2) before register access resumes.

**Bluetooth coordination.** A function-level reset of the Wi-Fi function must be preceded
by notifying the Bluetooth function in the same package (§2.5). Subsystem-level resets
affect only the Wi-Fi path.

---

## 4. Runtime power states (post-S14)

Once the radio is ready, the part moves between two steady ownership states rather than
returning through bring-up:

* **Host-owned** — the host may access registers and the DMA rings are active.
* **Firmware-owned** — the host has released ownership; register access is illegal and the
  firmware manages low-power behaviour autonomously. The host re-enters host-owned via the
  ownership handshake before any register access or ring update.

Wake sources that can return the part to host-owned are listed in §4.8. Entering and
leaving these states does **not** rewind the initialisation state machine.

---

## 5. Standing obligations — true in every state from S9 onwards

The state machine above describes how to *reach* the operating state. Once the firmware is
running, a set of obligations becomes continuous, and failing any of them stalls or kills
the part without rewinding the state machine. They are listed here because they belong to no
single state.

The **Obligation** and **Deadline** columns are `[C]` (they are what the host does and the
budgets it uses). The **Failure mode** column describes firmware behaviour and is `[L]`
throughout — see the corresponding sections.

| Obligation | Triggered by | Deadline | Failure mode |
|---|---|---|---|
| Acknowledge the host interrupt status by writing the read value back | every interrupt | before the next release of ownership | firmware refuses to enter low power `[L]` |
| Answer each error-recovery checkpoint on the host→MCU software-interrupt register | firmware sets a bit in the MCU→host software-interrupt status | host's own 10 s recovery timer; the firmware waits forever | MAC stays reset; no traffic `[L]` |
| Refill every active receive ring | ring consumption | continuous | events and, through the TX-free report, the whole transmit path stall `[C]` |
| Return every MSDU token the TX-free report gives back | each report | — | transmit stops after one pool's worth of frames `[C]` |
| Detect a token that never returns | per-frame timer | host-chosen; a timeout should raise a "TX error" recovery reason | silent transmit stall `[C]` |
| Release ownership when the firmware asks (`EVENT_ID_SLEEPY_INFO`) | unsolicited event, sent once | none enforced | part never sleeps `[L]` |
| Acknowledge and collect an assert dump, then reset | `EVENT_ID_ASSERT_DUMP` | 5 000 ms of dump inactivity | part halted permanently `[L]` |
| Track command responses and time them out | every command with a response | 10 000 ms | the response may legitimately never arrive (firmware event-buffer exhaustion) `[L]` |
| Apply per-station and per-BSS transmit credits | `STA_UPDATE_FREE_QUOTA`, `BSS_ABSENCE_PRESENCE` | immediate | firmware packet buffer fills; all traffic classes back-pressure `[L]` |
| Maintain host-side receive reordering from the block-ack events | `RX_ADDBA` / `RX_DELBA` / reorder-bubble event | immediate | per-TID receive stall or out-of-order delivery `[L]` |
| Free a station index when the firmware reports it aged out or deauthenticated | `STA_AGING_TIMEOUT`, `SEND_DEAUTH` | immediate | the 15-entry pool leaks `[C]` (the pool is host-side) |
| Abort every channel privilege that was granted | activity completion, failure, or grant expiry | grant interval | the radio stays assigned to a dead role `[L]` |

---

## 6. Teardown (S14 → S0) and re-entry

Teardown is not the bring-up sequence reversed: several steps exist only on the way down.
The required order is `[C]`:

```
 1. stop offering frames to the transmit path; quiesce the host queues
 2. release all 802.11 state, leaves first:
        station state -> keys -> station records -> beacon template
        -> outstanding channel privileges (abort with the matching token)
        -> BSS contexts -> per-band radio disable
 3. if the firmware is alive and no coredump or recovery is outstanding:
        send the NIC power-off command and poll for the power-off indication
 4. disable interrupts (PCIe-MAC enable, WFDMA host enable, MCU->host software interrupt)
 5. stop host WFDMA and wait for the idle indication
 6. release the interrupt and stop all packet servicing, so that nothing can still be
    running against ring memory
 7. release the MSDU token pool (walking and releasing every entry still in use)
 8. free the rings and the descriptor templates
 9. invalidate the share-info block (clear its "valid" flag) so firmware stops writing
    into host memory
10. for a hibernate or whole-chip reset: write the "host is going away" doorbell value
11. release ownership to firmware
```

Steps 6, 7 and 9 are the ones with no visible reason in the register map. Their omission is
expected to be silently fatal, because the firmware writes into the share-info block and into
ring buffers asynchronously, so freeing that memory while the block is still marked valid, or
while an interrupt can still fire, corrupts host memory rather than producing an error. `[L]`
— the teardown order itself is `[C]`; this rationale for it is inferred.

**Re-entry.** Nothing persists across a teardown that reaches the MCU. On the next probe the
state machine starts at S0. The only host state that must survive a teardown is the
persisted calibration cache; see the EEPROM/calibration section.

---

## Open questions / needs hardware tracing

1. Whether S7 (interrupt arming) may legally precede S6 (DMA enable) on this silicon, or
   whether the order given here is required. `[U]`
2. Whether the remap-window programming in S3 is required before the subsystem reset in S5
   or only before the first access to a remapped region. `[U]`
3. The minimum settling time between firmware start and the first capability query. `[U]`
4. Whether L1 recovery can ever require re-programming the prefetch windows, or whether
   firmware restores them. `[U]`


---

# MT7932 — Appendix: Address Map, Register Index and Quick Reference

**Scope.** This appendix is a lookup aid, not a specification. It consolidates, in one place,
every directly addressable register that §1–§12 of this document specify: the chip address,
the BAR-0 offset a host must actually issue, the register name, its access type, a one-line
purpose, and the section that specifies it in full. It is preceded by a block-level
address-space map ("where am I") and followed by the PCI configuration-space registers the
host must touch, dense cross-reference tables for the MCU command / event / status-code and
capability-tag spaces, and a single ordered bring-up sequence a driver author can work
through on day one. Nothing here is new: every row cites the section that establishes it.
Where two sections disagree, that is recorded in §6 of this appendix rather than silently
resolved.

## Conventions used in this appendix

* **Chip address** — an address in the MT7932 internal (flat) address space.
* **BAR-0 offset** — the byte offset within the 1 MiB BAR-0 aperture that the host actually
  issues. Computed from the fixed map of §1 §4 by the translation rule of §1 §3.2:
  `0x0000_2000 < A < 0x0010_0000` ⇒ offset = `A`; otherwise the first matching fixed-map
  entry gives `bar_offset + (A − chip_base)`.
  Offsets in this appendix are **computed** unless a section quotes them verbatim; the
  arithmetic is over a `[C]` map and `[C]` chip addresses and therefore carries no separate
  marker. Where the *chip address itself* is `[L]`/`[U]` in its source section, the marker is
  reproduced.
* **remap slot N** — the register is not reachable at a fixed BAR offset. The 1 MiB aperture
  is sixteen 64 KiB slots; slot *N* occupies BAR `N × 0x10000 … N × 0x10000 + 0xFFFF` and its
  chip base is the 16-bit selector field in the PCIe2AP remap array (§1 §5). "remap slot 4 ←
  `0x7001`" means: program `0x7C00_E24C[15:0] = 0x7001`, wait ≥ 2 µs, read back, then access
  BAR `0x40000 + (A & 0xFFFF)`, then restore.
* **Outside the static window** — flagged **`⚑`** in the BAR-offset column: the chip address
  has no fixed-map entry at all and *always* needs a remap slot.
* **Slot-dependent** — flagged **`†`**: the address *is* in the fixed map, but only decodes
  while the controlling remap slot holds its default/bring-up value (§1 §5.5).
* Access types: `RO` read-only · `WO` write-only · `RW` read/write · `W1C` write-1-to-clear ·
  `W1S` write-1-to-set · `W1` write-1-to-act (self-clearing) · `RTC` read-to-clear.
* Confidence: `[C]` confirmed · `[L]` likely · `[U]` unverified. Markers appear only where
  they add information — chiefly on access types that no section states outright and on
  addresses their source section already qualified.
* Array rows give the **element-0 address**, the **stride** and the **element count** in place
  of expanding every element.

---

# 1. Address-space map (part B)

## 1.1 Functional blocks

The fixed BAR-0 → chip decode has exactly 50 entries (§1 §4). Grouped by function:

| Block base | Size | BAR-0 offset | What lives there | Sections |
|---|---|---|---|---|
| `0x0040_0000` | `0x10000` | `0x080000` † | WF MCU SYSRAM (MCU code/data window; RAM-image region 2 target) | §1 §4, §2 §1.2, §5 §3.3 |
| `0x5400_0000` | `0x1000` | `0x002000` | WFDMA PCIE0 **MCU DMA0** — MCU-side view of host WPDMA0: host→MCU software interrupt, WFDMA re-init dummy CR, MCU-view status aliases | §3 §1.2, §4 §1, §4 §3.2, §2 §8.1 |
| `0x5500_0000` | `0x1000` | `0x003000` | WFDMA PCIE0 MCU DMA1 — defined, not populated (no WFDMA1 on this part) | §3 §1.2, §4 §1 |
| `0x5600_0000` | `0x1000` | `0x004000` | WFDMA **reserved** instance — no register in this document. (§2 §1.2 previously called BAR `0x4000` a "legacy `PCIE_HIF`" software-interrupt alias; that has been **withdrawn** — the software interrupts are at chip `0x5400_0108` and `0x7C02_41F0`.) | §1 §4, §2 §1.2 |
| `0x5700_0000` | `0x1000` | `0x005000` | WFDMA MCU wrap CSR | §1 §4, §3 §1.2 |
| `0x5800_0000` | `0x1000` | `0x006000` | WFDMA PCIE1 MCU DMA0 (MEM_DMA) | §1 §4, §3 §1.2 |
| `0x5900_0000` | `0x1000` | `0x007000` | WFDMA PCIE1 MCU DMA1 | §1 §4, §3 §1.2 |
| `0x7000_0000` | `0x10000` | `0x070000` † | **CB-TOP / CONN2AP**: RGU (WF subsystem reset) at `+0x2600`, MTCMOS `VLP_UDS_CTRL` at `+0x3020`, CB-TOP debug CRs. Requires slot 7 ← `0x7000` (§1 §5.5) | §1 §4/§7.4, §2 §5.1/§5.6, §11 §2.2 |
| `0x7001_0000` | — | ⚑ remap slot ← `0x7001` | **CB-TOP chip-ID / HW-version / GALS page.** Outside the `0x7000_0000` entry, which stops at `0x7000_FFFF` | §1 §6.2, §2 §6.6, §5 §6.7, §11 §1.2 |
| `0x7403_0000` | `0x10000` | `0x010000` | **PCIe MAC internal (`PCIE_MAC_IREG`)**: interrupt enable/status/clear, PM, debug mux, and the configuration-space MMIO aliases (`+0x1000 + cfg` = Wi-Fi function, `+0x9000 + cfg` = Bluetooth function) | §1 §7.1, §2 §1.1/§6.1, §4 §1, §5 §9.6 |
| `0x7C00_0000` | `0x10000` | `0x0F0000` | **CONN_INFRA off-domain**: WFSYS reset / sw-init-done, own-IRQ status, **PCIe2AP remap selector array at `+0xE244`** | §1 §4/§5.2/§7.4, §2 §1.3/§5.1 |
| `0x7C02_0000` | `0x10000` | `0x0D0000` | **CONN_INFRA host WFDMA CSR aperture** — host WPDMA0 `+0x4000`, host WPDMA1 `+0x5000` (unused), DMASHDL `+0x6000`, WFDMA ext-wrap CSR `+0x7000` | §3 §1.1, §4 §1 |
| `0x7C05_0000` | `0x10000` | `0x090000` † | **CONN_INFRA SYSRAM** — SW-defined-CR mailbox through which the host publishes the share-info block. Requires slot 9 ← `0x1805` (§1 §5.5) | §1 §4, §2 §3.5, §5 §8.4 |
| `0x7C06_0000` | `0x10000` | `0x0E0000` | **`conn_host_csr_top`** — ownership (`CONN_ON_LPCTL`), own-IRQ, `CONN_ON_MISC`, conn-infra diagnostics | §1 §7.3/§7.5, §2 §3.1, §4 §1/§8.1 |
| `0x7C07_0000` | — | ⚑ remap slot ← `0x1807` | **CONN_INFRA semaphore block** — secure-boot / firmware-download gate | §1 §5.3/§7.4, §2 §4.5, §5 §4.5 |
| `0x8002_0000` | `0x10000` | `0x0B0000` | **WF_TOP_MISC_OFF (`TOP_CFG`)** — chip ID, HW version, chip-ID mirror / bus-liveness canary | §1 §6.1, §2 §2.3, §5 §1.1, §11 §1.1 |
| `0x8102_0000` | `0x10000` | `0x0C0000` | WF_TOP_MISC_ON — no register in this document | §1 §4, §2 §1.2 |
| `0x820B_0000` | `0x1000` | `0x0AE000` | [APB2] WFSYS_ON — no register in this document | §1 §4 |
| `0x820C_0000` | `0x4000` | `0x008000` | **WF_UMAC_TOP (PLE)** — queue/group empty status (all-ones whitelist) | §1 §3.4, §2 §2.2 |
| `0x820C_4000` | `0x4000` | `0x0A8000` | **WF_UWTBL** — UMAC station-table + key-table data-unit control (`+0x94`) and data window (`+0x2000`) | §10 §B.1/§B.3/§B.4 |
| `0x820C_8000` | `0x2000` | `0x00C000` | WF_UMAC_TOP (PSE) | §1 §4, §10 §B.1 |
| `0x820C_A000` | `0x2000` | `0x026000` | WF_MUCOP / MU control, band 0 | §1 §4, §10 §B.1 |
| `0x820C_C000` | `0x2000` | `0x00E000` | WF_UMAC_TOP (PP) **and** WF_MDP_TOP (`0x820C_D000`) — the packet processor that performs header translation. Second aperture at BAR `0x0A5000` (unreachable by chip-address lookup) | §1 §4, §10 §B.8 |
| `0x820C_E000` | `0x0200` | `0x021C00` | **WF_SEC** — cipher datapath control | §1 §4, §10 §B.4.3 |
| `0x820C_F000` | `0x1000` | `0x022000` | WF_PF (packet filter) | §1 §4, §10 §B.1 |
| `0x820D_0000` | `0x10000` | `0x030000` | **WF_WTBLON** — LMAC station table: data-unit control `+0x4200`, data window `+0x8000`, init-target CRs `+0x3B0…0x3BC`. Whole page is on the all-ones whitelist | §1 §3.4, §10 §B.1/§B.2 |
| `0x820E_0000` | `0x0400` | `0x020000` | WF_LMAC_TOP BN0 — WF_CFG | §1 §4 |
| `0x820E_1000` | `0x0200` | `0x020400` | WF_LMAC_TOP BN0 — WF_TRB | §1 §4 |
| `0x820E_2000` | `0x0400` | `0x020800` | WF_LMAC_TOP BN0 — WF_AGG | §1 §4 |
| `0x820E_3000` | `0x0400` | `0x020C00` | WF_LMAC_TOP BN0 — **WF_ARB** (AC-mode CR at `+0x15C`) | §1 §4, §11 §2.4 |
| `0x820E_4000` | `0x0400` | `0x021000` | WF_LMAC_TOP BN0 — WF_TMAC | §1 §4 |
| `0x820E_5000` | `0x0800` | `0x021400` | **WF_LMAC_TOP BN0 — WF_RMAC**: `RFCR`/`RFCR1` hardware receive filter, airtime/CCA MIB | §1 §4, §10 §A.9.2/§B.9 |
| `0x820E_7000` | `0x0200` | `0x021E00` | WF_LMAC_TOP BN0 — WF_DMA | §1 §4 |
| `0x820E_9000` | `0x0200` | `0x023400` | **WF_LMAC_TOP BN0 — WF_WTBLOFF** (per-station RCPI update control) | §1 §4, §10 §B.2.4 |
| `0x820E_A000` | `0x0200` | `0x024000` | **WF_LMAC_TOP BN0 — WF_ETBF** (beamforming feedback counters) | §1 §4, §10 §B.9 |
| `0x820E_B000` | `0x0400` | `0x024200` | WF_LMAC_TOP BN0 — WF_LPON (TSF) | §1 §4, §10 §B.5 |
| `0x820E_C000` | `0x0200` | `0x024600` | WF_LMAC_TOP BN0 — WF_INT | §1 §4 |
| `0x820E_D000` | `0x0800` | `0x024800` | **WF_LMAC_TOP BN0 — WF_MIB counter block** (size disputed — §6.3 of this appendix) | §1 §4, §10 §B.9 |
| `0x820F_0000 … 0x820F_D000` | — | `0x0A0000 … 0x0A4800` | **WF_LMAC_TOP BN1** — band-1 mirrors of every BN0 block above | §1 §4, §10 §B.1 |
| `0x8300_xxxx / 0x8301_xxxx / 0x830A_xxxx` | — | ⚑ remap slot ← `0x8300`/`0x8301`/`0x830A` `[L]` | **PHY / RF instance registers** — only the five calibration-integrity CRs of §9 §8.1 are named | §9 §8.1 |
| `0x8800_0000` | `0x10000` | `0x040000` † | **WF_MCU_CFG_LS** — `TOP_FVR` at `+0x0004`, WFSYS bus-status CRs at `+0x0430/0444/044C/0450`. Decodes only while slot 4 holds its default `0x184F` | §1 §4/§5.3/§7.5, §5 §1.1, §11 §1.2 |
| `0x5200_0000` | — | not mapped | CONNAC2 `HIF_DMASHDL` constant: the **MCU/AXI-internal-bus view** of the DMA scheduler that the host reaches at `0x7C02_6000`. No host access is ever made through it | §3 §8.2, §11 §4 |

**Band-1 mirror rule.** For every WF_LMAC_TOP block except WF_TRB, band 1 is at
chip + `0x0001_0000` and BAR + `0x08_0000`. WF_TRB is the exception: BN0 `0x820E_1000` →
BAR `0x020400`, BN1 `0x820F_1000` → BAR `0x0A0600`. `[C]` (derived from the §1 §4 map.)

**Unmapped BAR ranges** (§1 §4): `0x00000–0x01FFF` (also excluded from direct BAR-offset
addressing), `0x050000–0x06FFFF`, `0x0A6000–0x0A7FFF`, `0x0AF000–0x0AFFFF`.

## 1.2 Remap slot map

Array base **`0x7C00_E244`** `[L]` (host view) = AP-bus `0x1800_E244`, eight 32-bit
registers, two 16-bit selector fields each, 16 slots, 64 KiB granularity. Selector value =
target chip address bits [31:16]. Conn-infra targets are expressed on the AP bus
(`host 0x7C0n_xxxx ≡ AP 0x180n_xxxx`); CB-TOP and PCIe-MAC targets are used unmodified.
(§1 §5.)

| Slot | BAR window | Control register | Field | Selector value | Reaches |
|---|---|---|---|---|---|
| 0 | `0x00000` | `0x7C00_E244` | [15:0] | — `[U]` | composite WFDMA/UMAC decode |
| 1 | `0x10000` | `0x7C00_E244` | [31:16] | `0x7403` `[L]` | PCIe MAC IREG |
| 2 | `0x20000` | `0x7C00_E248` | [15:0] | — `[U]` | composite LMAC BN0 decode |
| 3 | `0x30000` | `0x7C00_E248` | [31:16] | `0x820D` `[L]` | WF_WTBLON |
| **4** | `0x40000` | **`0x7C00_E24C`** | **[15:0]** | default **`0x184F`**; host temporarily writes `0x7000`, `0x7001`, `0x1807` | WF_MCU_CFG_LS / CB-TOP / CB-TOP HW-ID / conn-infra semaphore `[C]` |
| 5 | `0x50000` | `0x7C00_E24C` | [31:16] | `0x1845` (written back unchanged) `[C]` | unidentified `[U]` |
| 6 | `0x60000` | `0x7C00_E250` | [15:0] | `0x1846` (written back unchanged) `[C]` | unidentified `[U]` |
| **7** | `0x70000` | **`0x7C00_E250`** | **[31:16]** | host **must program `0x7000`** `[C]` | CB-TOP / CONN2AP |
| 8 | `0x80000` | `0x7C00_E254` | [15:0] | `0x1848` `[C]` | WF_MCU_SYSRAM (`0x0040_0000`) |
| **9** | `0x90000` | **`0x7C00_E254`** | **[31:16]** | host **must program `0x1805`** `[C]` | CONN_INFRA SYSRAM (`0x7C05_0000`) |
| 10 / 11 | `0xA0000` / `0xB0000` | `0x7C00_E258` | [15:0]/[31:16] | — / `0x8002` `[L]` | LMAC BN1 composite / WF_TOP_MISC_OFF |
| 12 / 13 | `0xC0000` / `0xD0000` | `0x7C00_E25C` | [15:0]/[31:16] | `0x8102` / `0x1802` `[L]` | WF_TOP_MISC_ON / conn-infra |
| 14 / 15 | `0xE0000` / `0xF0000` | `0x7C00_E260` | [15:0]/[31:16] | `0x1806` / `0x1800` `[L]` | `conn_host_csr_top` / CONN_INFRA off-domain |

Confirmed slot-4 values and what they reach (§1 §5.3): `0x184F` → `0x8800_0000`;
`0x7000` → `0x7000_0000`; `0x7001` → `0x7001_0000`; `0x1807` → `0x7C07_0000`.
Composite default word for `0x7C00_E24C` is **`0x1845_184F`** and must be restored after
every borrow (§1 §5.4, §2 §1.3).

---

# 2. Consolidated chip-register index (part A)

Sorted by chip address. Values, masks, command IDs, descriptor field offsets, EEPROM offsets
and firmware-image offsets are **not** in this table; see §4 and §5 of this appendix, and the
sections cited.

| Chip address | BAR-0 offset | Name | Access | Purpose | Specified in |
|---|---|---|---|---|---|
| `0x5400_0108` | `0x002108` | `HOST2MCU_SW_INT_SET` | W1S | Host→MCU software interrupt; carries the four SER handshake phase codes and the host-initiated-SER request | §4 §3.2, §2 §8.1 |
| `0x5400_0120` | `0x002120` | `MT_WFDMA_DUMMY_CR` | RW | Bit 1 = `NEED_REINIT`: host sets it after programming the rings, tests it after an L1/deep-sleep event | §3 §1.2, §2 §5.3c |
| `0x5400_0200` | `0x002200` | `HOST_INT_STA` (MCU view) | RO | Diagnostic alias of `0x7C02_4200` | §4 §1 |
| `0x5400_0204` | `0x002204` | `HOST_INT_ENA` (MCU view) | RO | Diagnostic alias of `0x7C02_4204` | §4 §1 |
| `0x5400_0208` | `0x002208` | MCU `WPDMA_GLO_CFG` | RW `[L]` | Global config of the MCU-side DMA instance | §3 §1.2/§2 |
| `0x5400_0320` | `0x002320` | WM TX ring 2 control block | RW `[L]` | MCU→air data/management ring; layout as §2's host ring block | §3 §1.2 |
| `0x5400_0510` | `0x002510` | WM RX ring *N* control block — **element 0 = ring 1, stride `0x10`, 5 elements** (rings 1–5: host→MCU command, management, data, TX-free-done, RX report) | RW `[L]` | MCU-side receive rings, observable by the host debug path | §3 §1.2 |
| `0x7000_0100` | `0x070100` † | CB-TOP debug CR | RO `[L]` | Read in the CB-TOP debug dump (BAR `0x40100` when slot 4 ← `0x7000`) | §1 §5.3 |
| `0x7000_0108` | `0x070108` † | CB-TOP debug CR | RO `[L]` | as above | §1 §5.3 |
| `0x7000_0148` | `0x070148` † | CB-TOP debug CR | RO `[L]` | as above | §1 §5.3 |
| `0x7000_2010` | `0x072010` † | CB-TOP debug CR | RO `[L]` | as above | §1 §5.3 |
| `0x7000_2014` | `0x072014` † | CB-TOP debug CR | RO `[L]` | as above | §1 §5.3 |
| `0x7000_2030` | `0x072030` † | CB-TOP debug CR | RO `[L]` | as above | §1 §5.3 |
| `0x7000_2600` | `0x072600` † | `CBTOP_RGU_WF_SUBSYS_RST` | RW | `BIT(0)` `WF_WHOLE_PATH_RST` (1 = assert, hold 50 ms), `BIT(6)` `BYPASS_WFDMA_SLP_PROT` (not used) | §2 §5.1, §1 §7.4 |
| `0x7000_300C` | `0x07300C` † | CB-TOP debug CR | RO `[L]` | CB-TOP debug dump | §1 §5.3 |
| `0x7000_3014` | `0x073014` † | CB-TOP debug CR | RO `[L]` | CB-TOP debug dump | §1 §5.3 |
| `0x7000_3020` | `0x073020` † | `VLP_UDS_CTRL` (MTCMOS) | RW | MTCMOS hardware-mode power control; written `0`, then wait 1 ms. **MT7932/MT7923 only** | §2 §5.6, §11 §2.2 item 3 |
| `0x7000_3100` | `0x073100` † | CB-TOP debug CR | RO `[L]` | CB-TOP debug dump | §1 §5.3 |
| `0x7000_320C` | `0x07320C` † | CB-TOP debug CR | RO `[L]` | CB-TOP debug dump | §1 §5.3 |
| `0x7001_0020` | ⚑ slot ← `0x7001` → `0x040020` | `MT_HW_BOUND` | RO | Public MT7921 A-die selector (`BIT(7)`). **Not used on MT7932** — A-die identity comes from capability tag `0x14` | §1 §6.1/§6.4 |
| `0x7001_0200` | ⚑ slot ← `0x7001` → `0x040200` | `TOP_HCR` (`MT_HW_CHIPID`) | RO | Chip ID. Present in the record; not read during version discovery | §1 §6.1, §11 §1.2 |
| `0x7001_0204` | ⚑ slot ← `0x7001` → `0x040204` | `TOP_HVR` (`MT_HW_REV`) | RO | `[7:0]` HW version, `[11:8]` factory (ECO) version. **Read through boot-ROM cmd `0x03`, not by MMIO** | §1 §6.1/§6.2, §5 §6.7, §11 §1.2 |
| `0x7001_0208` | ⚑ slot ← `0x7001` → `0x040208` | `TOP_FVR` (public gen4m alternative location) | RO | **Not used on this part** — `TOP_FVR` is at `0x8800_0004` | §11 §1.2 |
| `0x7001_3008` | ⚑ slot ← `0x7001` → `0x043008` | CB-TOP CR | RO `[L]` | CB-TOP debug dump | §1 §5.3 |
| `0x7001_3018` | ⚑ slot ← `0x7001` → `0x043018` | CB-TOP CR | RO `[L]` | CB-TOP debug dump | §1 §5.3 |
| `0x7001_3090` | ⚑ slot ← `0x7001` → `0x043090` | CB-TOP CR | RO `[L]` | CB-TOP debug dump | §1 §5.3 |
| `0x7001_3100` | ⚑ slot ← `0x7001` → `0x043100` | CB-TOP **GALS status** | RO | Bus-bridge status; DMA-hang predicate is `(v & 0x0080_00C0) == 0x0080_0080` | §2 §6.6, §1 §5.3 |
| `0x7001_3120` | ⚑ slot ← `0x7001` → `0x043120` | CB-TOP CR | RO `[L]` | CB-TOP debug dump | §1 §5.3 |
| `0x7001_3124` | ⚑ slot ← `0x7001` → `0x043124` | CB-TOP CR | RO `[L]` | CB-TOP debug dump | §1 §5.3 |
| `0x7001_360C` | ⚑ slot ← `0x7001` → `0x04360C` | CB-TOP CR | RO `[L]` | CB-TOP debug dump | §1 §5.3 |
| `0x7403_002C` | `0x01002C` | PCIe MAC debug **status read-back** | RO | **Resolved:** it is read after each selector write; the "written `0`" reading in §1 §7.1 has been corrected | §1 §7.1, §2 §6.1 |
| `0x7403_0150` | `0x010150` | PCIe MAC diagnostic CR | RO `[L]` | Read wholesale in the bus-failure dump | §1 §7.1, §2 §6.1 |
| `0x7403_0154` | `0x010154` | PCIe MAC diagnostic CR | RO `[L]` | as above | §1 §7.1, §2 §6.1 |
| `0x7403_0164` | `0x010164` | PCIe MAC debug **signal** select | WO | 4 × 8-bit debug-signal selector; ten selector values are swept | §2 §6.1, §1 §7.1 |
| `0x7403_0168` | `0x010168` | PCIe MAC debug **group/mode** select | WO | **Resolved:** written `0xCCCC_0100` / `0x9999_0100`; the "read-only probe data" reading in §1 §7.1 has been corrected | §2 §6.1, §1 §7.1 |
| `0x7403_0184` | `0x010184` | PCIe MAC interrupt status | RO | PCIe-MAC-level interrupt status; also in the diagnostic dump | §4 §1, §1 §7.1 |
| `0x7403_0188` | `0x010188` | `MT_PCIE_MAC_INT_ENABLE` | RW | Device-side master interrupt unmask. `0x0000_01FF` to enable, `0` to disable. Nine sources (mt76 writes `0xFF`) | §4 §1/§4.5/§9, §1 §7.1, §2 §4.4 |
| `0x7403_018C` | `0x01018C` | PCIe MAC interrupt clear | W1C `[L]` | Acknowledge / re-arm the serviced PCIe-MAC vectors | §4 §1/§4.5, §1 §7.1 |
| `0x7403_0194` | `0x010194` | `MT_PCIE_MAC_PM` | RW `[L]` | `BIT(8)` = L0s disable. **Never written on MT7932** | §1 §7.1, §2 §7 |
| `0x7403_01D4` | `0x0101D4` | PCIe MAC diagnostic CR | RO `[L]` | bus-failure dump | §1 §7.1, §2 §6.1 |
| `0x7403_01D8` | `0x0101D8` | PCIe MAC diagnostic CR | RO `[L]` | bus-failure dump | §1 §7.1, §2 §6.1 |
| `0x7403_01E0` | `0x0101E0` | PCIe MAC diagnostic CR | RO `[L]` | bus-failure dump | §1 §7.1, §2 §6.1 |
| `0x7403_03C8` | `0x0103C8` | PCIe MAC diagnostic CR | RO `[L]` | bus-failure dump | §1 §7.1, §2 §6.1 |
| `0x7403_03CC` | `0x0103CC` | PCIe MAC diagnostic CR | RO `[L]` | bus-failure dump | §1 §7.1, §2 §6.1 |
| `0x7403_0E00` | `0x010E00` | PCIe MAC diagnostic CR | RO `[L]` | bus-failure dump | §1 §7.1, §2 §6.1 |
| `0x7403_0E04` | `0x010E04` | PCIe MAC diagnostic CR | RO `[L]` | bus-failure dump | §1 §7.1, §2 §6.1 |
| `0x7403_0E08` | `0x010E08` | PCIe MAC diagnostic CR | RO `[L]` | bus-failure dump | §1 §7.1, §2 §6.1 |
| `0x7403_1204` | `0x011204` | MMIO alias of **Wi-Fi** cfg `0x204` (AER uncorrectable status) | RW1C `[L]` | Read in the bus-failure dump | §2 §6.1, §1 §7.1 |
| `0x7403_1210` | `0x011210` | MMIO alias of Wi-Fi cfg `0x210` (AER correctable status) `[L]` | RO `[L]` | Link-register dump | §2 §6.1 |
| `0x7403_121C` | `0x01121C` | Wi-Fi function link register | RO `[L]` | Link-register dump | §2 §6.1 |
| `0x7403_1220` | `0x011220` | Wi-Fi function link register | RO `[L]` | Link-register dump | §2 §6.1 |
| `0x7403_1224` | `0x011224` | Wi-Fi function link register | RO `[L]` | Link-register dump | §2 §6.1 |
| `0x7403_1228` | `0x011228` | Wi-Fi function link register | RO `[L]` | Link-register dump | §2 §6.1 |
| `0x7403_1480` | `0x011480` | PCIe MAC diagnostic CR (vendor cfg alias region) | RO `[L]` | bus-failure dump | §2 §6.1, §1 §7.1 |
| `0x7403_1484` | `0x011484` | **Host→MCU doorbell** (MMIO alias of cfg `0x484`) | WO | Out-of-band notifications that bypass the command ring: `0x800` dump-collection ack, `0x2000` host-going-away, `0x8000_0000` force firmware assert | §5 §9.6, §2 §8.5, §1 §7.1 |
| `0x7403_1488` | `0x011488` | MMIO alias of cfg `0x488` (fabric status) | RO `[L]` | Fabric/power-domain status, MMIO view | §2 §1.1 |
| `0x7403_148C` | `0x01148C` | MMIO alias of cfg `0x48C` (function state) | RO `[L]` | Wi-Fi function state, MMIO view | §2 §1.1 |
| `0x7403_818C` | `0x01818C` | PCIe MAC CR read in the bus-failure dump | RO `[L]` | **Resolved:** it is inside the PCIe MAC window; the §1 §7.1 note placing it in LMAC BN0 has been corrected | §2 §6.1, §1 §7.1 |
| `0x7403_9204` | `0x019204` | MMIO alias of **Bluetooth** cfg `0x204` (AER uncorrectable status) | RO `[L]` | Read alongside the Wi-Fi function's on any bus failure | §2 §5.5/§6.1 |
| `0x7403_9210` | `0x019210` | BT function link register | RO `[L]` | Shared-domain diagnosis | §2 §5.5/§6.1 |
| `0x7403_921C` | `0x01921C` | BT function link register | RO `[L]` | Shared-domain diagnosis | §2 §5.5/§6.1 |
| `0x7403_9220` | `0x019220` | BT function link register | RO `[L]` | Shared-domain diagnosis | §2 §5.5/§6.1 |
| `0x7403_9224` | `0x019224` | BT function link register | RO `[L]` | Shared-domain diagnosis | §2 §5.5/§6.1 |
| `0x7403_9228` | `0x019228` | BT function link register | RO `[L]` | Shared-domain diagnosis | §2 §5.5/§6.1 |
| `0x7C00_0140` | `0x0F0140` | WFSYS software reset / init-done (`MT_WFSYS_SW_RST_B` equivalent; AP-bus `0x1800_0140`) | RW | `BIT(0)` `WFSYS_SW_RST_B`, `BIT(4)` `WFSYS_SW_INIT_DONE` (RO status, polled 3 × 100 ms) | §2 §5.1, §1 §7.4, §5 §7.4 |
| `0x7C00_1000` | `0x0F1000` | CONN_INFRA off-domain diagnostic CR | RO `[L]` | Conn-infra diagnostic read | §1 §7.5 |
| `0x7C00_1620` | `0x0F1620` | CONN_INFRA own-IRQ status | W1C | Written `0x3` + 2 ms settle **on MT7922 only**; MT7932 must skip it | §2 §5.2, §4 §8.4, §11 §2.2 item 4 |
| `0x7C00_E244` | `0x0FE244` `[L]` | **PCIe2AP remap selector array** — element 0, **stride `0x04`, 8 elements** (16 × 16-bit slot selectors) | RW | Programs each 64 KiB BAR slot's chip base; value = target address >> 16. Read-back + ≥ 2 µs settle required. Slot map in §1.2 of this appendix | §1 §5.2/§5.4/§5.5, §2 §1.3, §5 §4.5 |
| `0x7C02_4100` | `0x0D4100` | `WPDMA_HIF_RST` (`MT_WFDMA0_RST`) | RW `[L]` | Write `0` then `0x30` (bit 4 logic reset, bit 5 DMASHDL reset) | §3 §7.6/§9.2, §2 §5.3a |
| `0x7C02_4118` | `0x0D4118` | `HOST_INT_STA_EXT` | R/W1C | Extended host interrupt status. Defined by CONNAC2, **never accessed on this part** | §4 §1 |
| `0x7C02_413C` | `0x0D413C` | `BUSY_ENA` | RO | Bit 0 TX FIFO 0, bit 1 TX FIFO 1, bit 2 RX FIFO. Not needed for the idle check | §3 §9.3 |
| `0x7C02_41F0` | `0x0D41F0` | `MCU2HOST_SW_INT_STA` (`MT_MCU_CMD`) | R/W1C | MCU→host software-interrupt reason register: SER handshake, wake-by-RX, error latches. Write the read value back to acknowledge; never write when it reads `0xFFFF_FFFF` | §4 §3.1, §2 §8.1, §5 §9.4 |
| `0x7C02_41F4` | `0x0D41F4` | `MCU2HOST_SW_INT_MASK/ENA` | RW | Programmed `0x0000_FFFF` (all 16 reason bits unmasked) | §4 §3.1/§9, §3 §7.6 |
| `0x7C02_4200` | `0x0D4200` | `HOST_INT_STA` | R/W1C | Primary host interrupt status; acknowledge **before** draining the rings. Full bit map in §4 §2 | §4 §2/§5, §1 §7.2, §3 §7.6 |
| `0x7C02_4204` | `0x0D4204` | `HOST_INT_ENA` | RW | Primary interrupt mask. Typical value `0xEEC8_000D`. No set/clear alias — partial re-arm is a host-serialised RMW | §4 §2/§9, §2 §4.4, §3 §7.6 |
| `0x7C02_4208` | `0x0D4208` | `WPDMA_GLO_CFG` | RW (bits 1, 3 RO) | Global DMA configuration and the TX/RX enable and busy bits. Complete field map in §3 §2 | §3 §2/§9, §2 §5.3b |
| `0x7C02_420C` | `0x0D420C` | `WPDMA_RST_DTX_PTR` | WO `[L]` | Write `0xFFFF_FFFF` to reset all TX descriptor pointers at the end of prefetch programming | §3 §5/§7.6 |
| `0x7C02_4260` | `0x0D4260` | `WPDMA_PAUSE_RXQ_TH10/32/54/76` — element 0, **stride `0x04`, 4 elements** | RW `[L]` | Per-RX-ring-pair low/high pause thresholds. **Not programmed on MT7932** | §4 §6.6 |
| `0x7C02_4280` | `0x0D4280` | `WPDMA_RST_DRX_PTR` | WO `[L]` | Reset RX DMA pointers. **Not written on MT7932** | §3 §7.6 |
| `0x7C02_4298` | `0x0D4298` | `WPDMA_INT_RX_PRI_SEL` | RW | Bit *n* puts RX ring *n* on the priority (delayed-interrupt) path. Written `0x0000_000C` (rings 2, 3) in the WPDMA-config path **when MSI is enabled**; the coalescing path read-modify-writes it (OR `0xC` / AND `~0x4`, so bit 3 is never cleared) | §4 §6.2, §3 §7.6, §1 §7.2 |
| `0x7C02_42E8` | `0x0D42E8` | `HOST_PER_DLY_INT_CFG` | RW | Periodic per-RX-ring delayed interrupt: `[7:0]` max pending time in 20 µs ticks, `[31:16]` per-ring enable. Written `0x01FD_0032` | §4 §6.3, §3 §7.6 |
| `0x7C02_42F0` | `0x0D42F0` | `WPDMA_PRI_DLY_INT_CFG0` | RW | Priority delayed interrupt for RX rings 2 and 3 (count + time + enable per half). Written `0x8032_800A` **when MSI is enabled**, retuned adaptively and restored to the same constant | §4 §6.1, §3 §7.6, §1 §7.2 |
| `0x7C02_42F4` | `0x0D42F4` | `WPDMA_PRI_DLY_INT_CFG1` | RW `[L]` | Same layout for the next RX-ring pair. **Never written on this part** | §4 §6.1 |
| `0x7C02_42F8` | `0x0D42F8` | `WPDMA_PRI_DLY_INT_CFG2` | RW `[L]` | As above | §4 §6.1 |
| `0x7C02_4300` | `0x0D4300` | TX ring `CTRL0` — descriptor base — element 0, **stride `0x10`, 20 elements `[L]`** (20/10 is the host ring-table dimension, not a register-file read-out — §3 §3) | RW | Physical address of descriptor 0 (32 bits) | §3 §4.1 |
| `0x7C02_4304` | `0x0D4304` | TX ring `CTRL1` — count / base extension — **stride `0x10`, 20 elements** | RW | `[11:0]` `MAX_CNT`, `[19:16]` descriptor-base bits 35:32 (written 0 on this part) | §3 §4.1/§4.4 |
| `0x7C02_4308` | `0x0D4308` | TX ring `CTRL2` — CPU index (`CIDX`) — **stride `0x10`, 20 elements** | RW | 12-bit entry index; **the write is the doorbell** | §3 §4.1/§7.1 |
| `0x7C02_430C` | `0x0D430C` | TX ring `CTRL3` — DMA index (`DIDX`) — **stride `0x10`, 20 elements** | RO | 12-bit entry index advanced by the engine | §3 §4.1/§4.3 |
| `0x7C02_4500` | `0x0D4500` | RX ring `CTRL0` — descriptor base — element 0, **stride `0x10`, 10 elements** | RW | Physical address of descriptor 0 | §3 §4.2 |
| `0x7C02_4504` | `0x0D4504` | RX ring `CTRL1` — count / base extension — **stride `0x10`, 10 elements** | RW | As TX `CTRL1` | §3 §4.2 |
| `0x7C02_4508` | `0x0D4508` | RX ring `CTRL2` — CPU index — **stride `0x10`, 10 elements** | RW | Initialised to `MAX_CNT − 1`; batched refill doorbell | §3 §4.2/§7.3 |
| `0x7C02_450C` | `0x0D450C` | RX ring `CTRL3` — DMA index | RO | Engine's fill pointer | §3 §4.2/§4.3 |
| `0x7C02_4600` | `0x0D4600` | TX ring `EXT_CTRL` (prefetch) — element 0, **stride `0x04`, 20 elements `[L]`** | RW | `(DISP_BASE_PTR[31:16] << 16) \| DISP_MAX_CNT[7:0]`. Must be programmed for **every** ring of **every** instance before either DMA enable bit is set | §3 §5, §4 §2.1 |
| `0x7C02_4680` | `0x0D4680` | RX ring `EXT_CTRL` (prefetch) — element 0, **stride `0x04`, 10 elements `[L]`** | RW | As above | §3 §5, §4 §2.1 |
| `0x7C02_51F0` | `0x0D51F0` | `MCU2HOST_SW_INT_STA`, WPDMA1 instance | R/W1C | WFDMA1 is absent on this part; never accessed | §1 §7.2, §5 §9.4 |
| `0x7C02_51F4` | `0x0D51F4` | `MCU2HOST_SW_INT_MASK`, WPDMA1 instance | RW | As above | §1 §7.2, §3 §7.6 |
| `0x7C02_600C` | `0x0D600C` | DMASHDL scheduler control | RW | Host clears bits 17:16 and **sets bit 16**. Public CONNAC2 names bit 16 `GROUP_SEQUENCE_ORDER_TYPE` and bit **17** `SLOT_TYPE_ARBITER_CONTROL` `[L]`, so the slot arbiter is left **clear** | §3 §8.2/§8.4 |
| `0x7C02_6010` | `0x0D6010` | DMASHDL refill control | RW | Bit `16+n` = refill **disable** for group *n*. Groups 0–14 enabled, group 15 disabled | §3 §8.2/§8.4 |
| `0x7C02_601C` | `0x0D601C` | DMASHDL packet max page | RW | `[11:0]` PLE max page (`0x001`), `[27:16]` PSE max page (`0x000`); write mask `0x0FFF_0FFF` | §3 §8.2/§8.4 |
| `0x7C02_6020` | `0x0D6020` | DMASHDL group quota — element 0, **stride `0x04`, 16 elements** | RW | `[11:0]` minimum (reserved) pages, `[27:16]` maximum pages; `[31:28]` preserved | §3 §8.2/§8.4 |
| `0x7C02_6060` | `0x0D6060` | DMASHDL queue→group map — element 0, **stride `0x04`, 4 elements** | RW | 32 LMAC queues, 4 bits each, 8 per register. MT7932 programs an identity map for queues 0–15 | §3 §8.2/§8.3 |
| `0x7C02_6070` | `0x0D6070` | DMASHDL priority→group map — element 0, **stride `0x04`, 2 elements** | RW | 16 priority slots, 4 bits each. Identity map | §3 §8.2/§8.4 |
| DMASHDL status/counters | `0x0D6100`, `0x0D6140 + 4n`, `0x0D6180 + 4k` `[L]` | Per-group page / reserved / source / packet counters | RO | Read by the diagnostic path. Offsets are the public gen4m `STATUS_RD` / `STATUS_RD_GP0..15` / `RD_GP_PKT_CNT_*` (base + `0x100` / `0x140` / `0x180`), `[L]` for this part. They do **not** lie inside the quota array at base + `0x20`…`0x5C` | §3 §8.2 |
| `0x7C02_7010` | `0x0D7010` | `WPDMA_EXT_INT_STA` | R/W1C | Second-level WFDMA1 aggregate status. **Never accessed** (WFDMA1 absent) | §4 §1 |
| `0x7C02_7014` | `0x0D7014` | `WPDMA_EXT_INT_MASK` | RW | As above | §4 §1 |
| `0x7C02_7030` | `0x0D7030` | `WFDMA_HOST_CONFIG` | RW | Bit 9 `pcie_dly_rx_int_en`, set/cleared with interrupt coalescing; rest preserved by RMW | §4 §6.4, §3 §7.6 |
| `0x7C02_7038` | `0x0D7038` | WFDMA ext-wrap host configuration | RW | Written `0x0000_0013` in the MSI/DMA-enable path (only when MSI is enabled). **Field meaning `[U]`**; plausibly the CONNAC2 equivalent of the CONNAC3 MSI routing word `[L]` | §4 §4.5, §3 §7.6, §1 §7.2 |
| `0x7C02_70F0` | `0x0D70F0` | Ext-wrap prefetch/MSI configuration — element 0, **stride `0x04`, 4 elements** `[L]` | RW `[L]` | `MSI_INT_CFG0..3` on CONNAC3 siblings. **Never written on MT7932** | §3 §5, §4 §4.5 |
| `0x7C05_3A30` | `0x093A30` † | Share-info mailbox — second host buffer address, low 32 bits | WO `[L]` | Published to the MCU before firmware download | §2 §3.5, §5 §8.4 |
| `0x7C05_3A34` | `0x093A34` † | Share-info mailbox — second buffer address, high 32 bits | WO `[L]` | Written only in 64-bit addressing mode ⇒ **never on MT7932** | §2 §3.5, §5 §8.4 |
| `0x7C05_3A38` | `0x093A38` † | Share-info mailbox — control flags | WO `[L]` | `BIT(0)` 64-bit host addressing (clear here), `BIT(1)` share-info valid. Cleared to invalidate the block | §2 §3.5, §5 §8.4 |
| `0x7C05_3A3C` | `0x093A3C` † | Share-info mailbox — **share-info block address, low 32 bits** | WO `[L]` | The host-DRAM block firmware writes ownership grant, SER latches, health monitor and coredump flags into | §2 §3.5, §5 §8.4/§9.1 |
| `0x7C05_3A50` | `0x093A50` † | Share-info mailbox slot | — `[U]` | Present in the per-chip register list, **never written**; purpose unknown | §2 §3.5, §5 §8.4 |
| `0x7C05_3A54` | `0x093A54` † | Share-info mailbox — third host buffer address, low 32 bits | WO `[L]` | Published before firmware download | §2 §3.5, §5 §8.4 |
| `0x7C05_3A58` | `0x093A58` † | Share-info mailbox — third buffer address, high 32 bits | WO `[L]` | 64-bit mode only ⇒ never on MT7932 | §2 §3.5, §5 §8.4 |
| `0x7C05_3A5C` | `0x093A5C` † | Share-info mailbox slot | — `[U]` | Never written; purpose unknown | §2 §3.5, §5 §8.4 |
| `0x7C05_3C28` | `0x093C28` † | Share-info mailbox — enable / doorbell | WO `[L]` | Written `1` **first**, before the address words | §2 §3.5, §5 §8.4 |
| `0x7C06_0000` | `0x0E0000` | Conn-infra off-domain control | RW | Written `1` then read back in the conn-infra diagnostic path | §1 §7.5 |
| `0x7C06_0010` | `0x0E0010` | `CONN_ON_LPCTL` (`CONNAC2X_BN0_LPCTL`) | W1 (bits 0,1) / RO (bit 2) | `BIT(0)` set-firmware-own, `BIT(1)` clear-own (take driver own), `BIT(2)` own-sync status (`0` = host owns). **The gate on all Wi-Fi-domain MMIO** | §2 §3.1–§3.3, §1 §7.3, §4 §8.1 |
| `0x7C06_0014` | `0x0E0014` | `CONN_ON_IRQ_STAT` (`BN0_IRQ_STAT`) | R/W1C | `BIT(0)` firmware-cleared-own latch. Not used in the normal MT7932 driver-own path | §2 §3.1, §4 §1/§8.1 |
| `0x7C06_0018` | `0x0E0018` | `CONN_ON_IRQ_ENA` (`BN0_IRQ_ENA`) | RW | Written `BIT(0)` during bring-up `[C]`; that it enables the latch above is `[L]` (public naming — the latch itself is never read on this part) | §4 §1/§8.1, §1 §7.3, §11 §4 |
| `0x7C06_00F0` | `0x0E00F0` | `CONN_ON_MISC` (`sw_sync0`) | RO | `BIT(0)` FW power on, `BIT(1)` FW N9 on. The **MT7922** readiness gate; defined but **not consulted on MT7932** | §5 §7.1/§7.1.1, §2 §2.4, §1 §7.3, §11 §2.2 item 1 |
| `0x7C06_0138` | `0x0E0138` | Conn-infra signal-status **select** | RW `[L]` | 11 selector values swept in the diagnostic path | §1 §7.5 |
| `0x7C06_0150` | `0x0E0150` | Conn-infra signal-status **result** | RO `[L]` | Paired with `0x7C06_0138` | §1 §7.5 |
| `0x7C06_015C` | `0x0E015C` | Conn-infra debug **select** | RW `[L]` | 6 selector values swept | §1 §7.5 |
| `0x7C06_0204` | `0x0E0204` | WFSYS CPU program counter | RO `[L]` | Diagnostic | §1 §7.3 |
| `0x7C06_0208` | `0x0E0208` | WFSYS low-power state | RO `[L]` | Diagnostic | §1 §7.3 |
| `0x7C06_0294` | `0x0E0294` | Conn-infra strap pins | RO | Boot strap sampling | §1 §7.5 |
| `0x7C06_02C8` | `0x0E02C8` | Conn-infra debug **status** | RO `[L]` | Paired with `0x7C06_015C` | §1 §7.5 |
| `0x7C06_02D4` | `0x0E02D4` | Conn-infra off-domain bus alive | RO | `BIT(0)` clear ⇒ the off-domain bus is not responding | §1 §7.5 |
| `0x7C07_0060` | ⚑ slot 4 ← `0x1807` → `0x040060` | Secure-boot / firmware-download semaphore, acquire | RO (read-to-acquire) `[L]` | Poll `BIT(0)` = 1; 1000 µs tick, 5000 iterations. Held across patch-finish and firmware-start | §5 §4.5, §2 §4.5, §1 §5.3 |
| `0x7C07_0260` | ⚑ slot 4 ← `0x1807` → `0x040260` | Secure-boot semaphore, release | WO | Write `1` | §5 §4.5, §2 §4.5, §1 §5.3 |
| `0x8002_1000` | `0x0B1000` | `TOP_HW_VERSION` | RO `[L]` | `[3:0]` = revision / ECO nibble. **Not** `0x8002_1008[19:16]` | §1 §6.1/§6.2, §2 §2.3, §11 §1.1 |
| `0x8002_1008` | `0x0B1008` | `TOP_HW_CONTROL` (`WCIR`) | RO `[L]` | `[15:0]` = chip ID (`0x7932`). Verification is **disabled** on this part | §1 §6.1/§6.4, §5 §1.1, §11 §1.1 |
| `0x8002_1010` | `0x0B1010` | *(identity `[U]`)* | RO | The **bus-liveness canary** — a second `0xDEAD_FEED` here means the chip is dead `[C]`. The public-macro name `CONN_CFG_CHIP_ID` is not established for this part (gen4m's CONNAC1 map calls the same offset `STRAP_STA`) | §1 §3.3, §2 §6.4 |
| `0x820C_0600` | `0x008600` | PLE queue/group empty status | RO | All-ones is a legal data value (§1 §3.4 whitelist) | §1 §3.4, §2 §2.2 |
| `0x820C_0604` | `0x008604` | PLE queue/group empty status | RO | as above | §1 §3.4, §2 §2.2 |
| `0x820C_0680` | `0x008680` | PLE queue/group empty status | RO | as above | §1 §3.4, §2 §2.2 |
| `0x820C_0684` | `0x008684` | PLE queue/group empty status | RO | as above | §1 §3.4, §2 §2.2 |
| `0x820C_0700` | `0x008700` | PLE queue/group empty status | RO | as above | §1 §3.4, §2 §2.2 |
| `0x820C_0704` | `0x008704` | PLE queue/group empty status | RO | as above | §1 §3.4, §2 §2.2 |
| `0x820C_0780` | `0x008780` | PLE queue/group empty status | RO | as above | §1 §3.4, §2 §2.2 |
| `0x820C_0784` | `0x008784` | PLE queue/group empty status | RO | as above | §1 §3.4, §2 §2.2 |
| `0x820C_4094` | `0x0A8094` | **UMAC WTBL / key-table `WDUCR`** | RW | `[3:0]` group select (128 entries per group), `BIT(31)` target: 0 = UWTBL, 1 = key table. **Not a delta** — this is public gen4m `MT7961_WIFI_UWTBL_BASE`; only the *generic* CONNAC2 macro is UWTBL base + 0 | §10 §0, §10 §B.3.2/§B.4.2 |
| `0x820C_6000` | `0x0AA000` | **UMAC WTBL / key-table data window** — element 0, **stride `0x40` per entry, 128 entries per group** | RW | Address = base \| `((idx & 0x7F) << 6)` \| `((DW & 0xF) << 2)`; 8 DW defined per entry | §10 §B.3, §B.4 |
| `0x820D_03B0` | `0x0303B0` | WTBL init-target **control** | RW `[L]` | Bulk WTBL initialisation: bit 31 execute, bit 16 write, bit 6 `MT_WTBL_SPE_IDX_SEL` | §10 §B.2.2, §A.11 |
| `0x820D_03B8` | `0x0303B8` | WTBL init-target data 0 | RW `[L]` | Bulk-initialisation data | §10 §B.2.2 |
| `0x820D_03BC` | `0x0303BC` | WTBL init-target data 1 | RW `[L]` | Bulk-initialisation data | §10 §B.2.2 |
| `0x820D_4200` | `0x034200` | **LMAC WTBL `WDUCR`** | RW | `[2:0]` group select (128 entries per group, 8 groups). **Not a delta** — this is public gen4m `MT7961_WIFI_LWTBL_BASE`; only the *generic* CONNAC2 macro is WTBLON base + 0 | §10 §0, §10 §B.2.2 |
| `0x820D_8000` | `0x038000` | **LMAC WTBL data window** — element 0, **stride `0x100` per entry, 128 entries per group** | RW | Address = base \| `((idx & 0x7F) << 8)` \| `((DW & 0x3F) << 2)`; 33 DW defined per entry | §10 §B.2 |
| `0x820E_315C` | `0x020D5C` | WF_ARB access-class mode register | RW `[L]` | Named in the per-chip record; identical MT7922/MT7932 | §11 §2.4 |
| `0x820E_5000` | `0x021400` | `MT_WF_RFCR` (band 0) | RW `[L]` | Hardware receive filter — 22 drop-enable bits. Programmed by firmware from `CMD_ID_SET_RX_FILTER` | §10 §A.9.2 |
| `0x820E_5004` | `0x021404` | `MT_WF_RFCR1` (band 0) | RW `[L]` | Control-frame drop enables (ACK, BF-poll, BA, CF-End, CF-Ack) | §10 §A.9.2 |
| `0x820E_5380` | `0x021780` | `RMAC_MIB_AIRTIME0..` (band 0) — element 0, **stride `0x04`** | RO `[L]` | Per-sub-band airtime accumulators | §10 §B.9 |
| `0x820E_53B8` | `0x0217B8` | `RMAC_MIB_AIRTIME14` (band 0) | RO `[L]` | `[23:0]` OBSS airtime | §10 §B.9 |
| `0x820E_53C4` | `0x0217C4` | `RMAC_MIB_TIME0` (band 0) | RW `[L]` | Bit 30 `RXTIME_EN`, bit 31 `RXTIME_CLR` — enable and clear the RX-time accumulators | §10 §B.9 |
| `0x820E_9008` | `0x023408` | `MT_WTBLOFF_TOP_RSCR` (band 0) | RW `[L]` | `[31:30]` `RCPI_MODE`, `[25:24]` `RCPI_PARAM` — averaging for the per-station response RCPI in LMAC WTBL DW30 | §10 §B.2.4 |
| `0x820E_A150` | `0x024150` | `ETBF_TX_APP_CNT` (band 0) | RO `[L]` | `[31:16]` implicit-BF TX count, `[15:0]` explicit-BF TX count | §10 §B.9 |
| `0x820E_A158` | `0x024158` | `ETBF_RX_FB_CNT` (band 0) | RO `[L]` | All / HE / VHT / HT feedback counts, 8 bits each | §10 §B.9 |
| `0x820E_D004` | `0x024804` | `MIB_SCR1` (band 0) | RW `[L]` | Bit 8 `TXDUR_EN`, bit 9 `RXDUR_EN` | §10 §B.9 |
| `0x820E_D02C` | `0x02482C` | `MIB_SDR9` (band 0) | RO `[L]` | `[23:0]` channel-busy time | §10 §B.9 |
| `0x820E_D048` | `0x024848` | `MIB_SDR16` (band 0) | RO `[L]` | `[23:0]` busy time, second accumulator | §10 §B.9 |
| `0x820E_D054` | `0x024854` | `MIB_SDR36` (band 0) | RO `[L]` | `[23:0]` TX airtime | §10 §B.9 |
| `0x820E_D058` | `0x024858` | `MIB_SDR37` (band 0) | RO `[L]` | `[23:0]` RX airtime | §10 §B.9 |
| `0x820E_D090` | `0x024890` | `MIB_SDR34` (band 0) | RO `[L]` | `[15:0]` MU-beamforming TX count | §10 §B.9 |
| `0x820E_D0B0` | `0x0248B0` | `MIB_ARNG` (band 0) — element 0, **stride `0x04`, 4 elements** | RW `[L]` | Aggregation-range boundaries, four 8-bit ranges per register ⇒ 16 bins | §10 §B.9 |
| `0x820E_D0C0` | `0x0248C0` | `MIB_DR8` (band 0) | RO `[L]` | RX MPDU counter | §10 §B.9 |
| `0x820E_D0C4` | `0x0248C4` | `MIB_DR9` (band 0) | RO `[L]` | RX byte counter | §10 §B.9 |
| `0x820E_D0CC` | `0x0248CC` | `MIB_DR11` (band 0) | RO `[L]` | RX counter | §10 §B.9 |
| `0x820E_D100` | `0x024900` | `MIB_MB_SDR0` (band 0) — element 0, **stride `0x10`, 4 elements (per AC)** | RO `[L]` | `[31:16]` RTS retry count | §10 §B.9 |
| `0x820E_D108` | `0x024908` | `MIB_MB_SDR2` (band 0) — element 0, **stride `0x10`, 4 elements (per AC)** | RO `[L]` | `[15:0]` frame retry count | §10 §B.9 |
| `0x820E_D518` | `0x024D18` | `MIB_MB_BSDR2` (band 0) | RO `[L]` | `[15:0]` BA failure count | §10 §B.9 |
| `0x820E_D520` | `0x024D20` | `MIB_MB_BSDR3` (band 0) | RO `[L]` | `[15:0]` ACK failure count | §10 §B.9 |
| `0x820E_D558` | `0x024D58` | `MIB_SDR12` (band 0) | RO `[L]` | A-MPDU / MPDU transmit counter | §10 §B.9 |
| `0x820E_D55C` | `0x024D5C` | `MIB_SDR31` (band 0) | RO `[L]` | A-MPDU / MPDU transmit counter | §10 §B.9 |
| `0x820E_D564` | `0x024D64` | `MIB_SDR14` (band 0) | RO `[L]` | A-MPDU / MPDU transmit counter | §10 §B.9 |
| `0x820E_D568` | `0x024D68` | `MIB_SDR15` (band 0) | RO `[L]` | A-MPDU / MPDU transmit counter | §10 §B.9 |
| `0x820E_D688` | `0x024E88` | `MIB_MB_BSDR0` (band 0) | RO `[L]` | `[15:0]` RTS count | §10 §B.9 |
| `0x820E_D690` | `0x024E90` | `MIB_MB_BSDR1` (band 0) | RO `[L]` | `[15:0]` RTS failure count | §10 §B.9 |
| `0x820E_D698` | `0x024E98` | `MIB_SDR3` (band 0) | RO `[L]` | `[31:16]` FCS error count | §10 §B.9 |
| `0x820E_D770` | `0x024F70` | `MIB_SDR22` (band 0) | RO `[L]` | RX counter | §10 §B.9 |
| `0x820E_D774` | `0x024F74` | `MIB_SDR23` (band 0) | RO `[L]` | RX counter | §10 §B.9 |
| `0x820E_D780` | `0x024F80` | `MIB_SDR5` (band 0) | RO `[L]` | RX counter | §10 §B.9 |
| `0x820E_D7A8` | `0x024FA8` | `MIB_SDR32` (band 0) | RO `[L]` | `[31:16]` implicit-BF count, `[15:0]` explicit-BF count | §10 §B.9 |
| `0x820E_D7DC` | `0x024FDC` | `TX_AGG_CNT` (band 0) — element 0, **stride `0x04`, 4 elements** | RO `[L]` | Aggregation histogram bins 0–7, two 16-bit bins per register | §10 §B.9 |
| `0x820E_D7EC` | `0x024FEC` | `TX_AGG_CNT2` (band 0) — element 0, **stride `0x04`, 4 elements** | RO `[L]` | Aggregation histogram bins 8–15 | §10 §B.9 |
| `0x820F_5000` | `0x0A1400` | `MT_WF_RFCR` and the whole RMAC block, **band 1** | as band 0 | Band-1 mirror; every band-0 RMAC/WTBLOFF/ETBF/MIB row above repeats at chip + `0x0001_0000`, BAR + `0x08_0000` | §10 §A.9.2/§B.9, §1 §4 |
| `0x820F_9008` | `0x0A3408` | `MT_WTBLOFF_TOP_RSCR`, band 1 | RW `[L]` | Band-1 mirror | §10 §B.2.4 |
| `0x820F_D000` | `0x0A4800` | MIB counter block, **band 1** | as band 0 | Band-1 mirror of every `0x820E_Dxxx` row | §10 §B.9 |
| `0x8300_7A14` | ⚑ slot ← `0x8300` `[L]` → `0x047A14` | Calibration-integrity CR, PHY instance 0 | RO `[L]` | Must read non-zero after calibration | §9 §8.1 |
| `0x8301_7A14` | ⚑ slot ← `0x8301` `[L]` → `0x047A14` | Calibration-integrity CR, PHY instance 1 | RO `[L]` | Same register in the second per-band PHY instance | §9 §8.1 |
| `0x830A_D418` | ⚑ slot ← `0x830A` `[L]` → `0x04D418` | Calibration-integrity CR | RO `[L]` | Must read non-zero after calibration | §9 §8.1 |
| `0x830A_D424` | ⚑ slot ← `0x830A` `[L]` → `0x04D424` | Calibration-integrity CR | RO `[L]` | Must read non-zero after calibration | §9 §8.1 |
| `0x830A_D42C` | ⚑ slot ← `0x830A` `[L]` → `0x04D42C` | Calibration-integrity CR | RO `[L]` | Must read non-zero after calibration | §9 §8.1 |
| `0x8800_0004` | `0x040004` † | `TOP_FVR` | RO | `[7:0]` = ROM/SW version. Feeds the ECO lookup. Read through boot-ROM cmd `0x03` in practice | §1 §6.1, §5 §1.1/§6.7, §11 §1.2 |
| `0x8800_0430` | `0x040430` † | WFSYS bus-status CR | RO `[L]` | Conn-infra/WFSYS bus diagnostics | §1 §5.3/§7.5 |
| `0x8800_0444` | `0x040444` † | WFSYS bus-status CR | RO `[L]` | as above | §1 §5.3/§7.5 |
| `0x8800_044C` | `0x04044C` † | WFSYS bus-status CR | RO `[L]` | as above | §1 §5.3/§7.5 |
| `0x8800_0450` | `0x040450` † | WFSYS bus-status CR | RO `[L]` | as above | §1 §5.3/§7.5 |

**Registers that fall outside the 1 MiB static window** (⚑ above), collected for emphasis:
the whole CB-TOP `0x7001_xxxx` page (`TOP_HCR`, `TOP_HVR`, GALS status and seven further
debug CRs), the conn-infra semaphore pair `0x7C07_0060` / `0x7C07_0260`, and the five
PHY-domain calibration-integrity CRs at `0x8300_7A14`, `0x8301_7A14`, `0x830A_D418`,
`0x830A_D424`, `0x830A_D42C`. Everything else in the table is reachable at a fixed BAR
offset, subject only to the three slot-dependent regions marked †
(`0x0040_0000`, `0x7000_0000`, `0x7C05_0000`, `0x8800_0000`).

---

# 3. PCI configuration-space register summary (part C)

Every configuration-space offset the host must touch. Accesses are 32-bit dword reads/writes
except `COMMAND`, `Device Control` and `Device Status`, which are 16-bit (§1 §3.1).
Configuration reads stay valid in more device states than MMIO — they are used precisely to
decide whether MMIO is usable — except after the suspend sequence has completed (§1 §3.6).

## 3.1 Standard header and capabilities

| Offset | Register | Access | Meaning / host action | Specified in |
|---|---|---|---|---|
| `0x00` | Vendor ID / Device ID | RO | `0x14C3` / `0x7932`. The **device ID is used directly as the chip ID**; no MMIO chip-ID compare is performed | §1 §1.1, §11 §1.1 |
| `0x04` | `COMMAND` (16-bit) | RMW16 | Set **Bus Master Enable** and **Memory Space Enable** before any DMA. Also dumped on a bus failure | §1 §1.3, §2 §1.1 |
| `0x08` | Class code / revision | RO | `02:80:00` (Network controller, other), revision `0x00`; programmed from the EEPROM identity record | §9 §2.3, §11 §1.4 |
| `0x10` | BAR 0, low dword | RW | Register aperture base. Only the first `0x0010_0000` is **used**; the size the BAR decodes is `[U]`. BAR 0 is the only BAR touched. Read back for diagnostics | §1 §1.2/§1.3, §2 §1.1 |
| `0x14` | BAR 0, high dword | RW | High half of the BAR-0 base | §1 §1.3, §2 §1.1 |
| `0x2C` | Subsystem vendor / device | RO | Equal to the primary IDs (`14C3:7932`); **not** used for matching | §1 §1.1, §9 §2.3 |
| `0xFC` | vendor status | RO | Read into the bus-failure dump. Contents **`[U]`** | §1 §1.3, §2 §1.1 |
| PM capability (offset discovered at run time) | Power Management Control/Status | RW | Device is placed in **D0** via the standard PM capability before use | §1 §1.3 |
| PCIe cap + `0x08` | Device Control (16-bit) | RW16 | **Initiate Function Level Reset = bit 15**. The capability offset is discovered at run time, never hard-coded | §1 §1.3, §2 §5.4 |
| PCIe cap + `0x0A` | Device Status (16-bit) | RO16 | **Transactions Pending = bit 5**; must be clear before an FLR. Poll budget ≈ 1 s | §1 §1.3 |
| MSI capability | Message Control | RW | Up to **8 vectors** declared; the observed configuration allocates **1**. No vendor programming is needed beyond `0x7403_0188` | §1 §1.4, §4 §4 |

## 3.2 Advanced Error Reporting — extended capability at `0x200`

| Offset | Register | Access | Meaning / host action | Specified in |
|---|---|---|---|---|
| `0x200` | AER extended-capability header | RO | Fixes the AER block at configuration offset `0x200` | §1 §1.3, §2 §1.1 |
| `0x204` | AER Uncorrectable Error **Status** | RW1C | Write `0xFFFF_FFFF` to clear all before re-arming AER; read and cleared in the error handler and on the way out of every reset | §1 §1.3, §2 §5.4/§7 |
| `0x208` | AER Uncorrectable Error **Mask** | RW | **Arm**: `0x0040_0000` (unmask bit 22 only). **Disarm around every FLR and WFSYS reset**: `0x0057_F010` | §1 §1.3, §2 §5.4/§7 |
| `0x210` | AER Correctable Error Status | RW1C | Read and cleared in the error handler; cleared when swallowing errors raised by a reset in progress | §1 §1.3, §2 §5.4/§7 |

## 3.3 Vendor-specific dwords — not present in any public MediaTek driver

All three are also aliased into MMIO at chip `0x7403_1000 + offset` for the Wi-Fi function
and `0x7403_9000 + offset` for the Bluetooth function (§2 §1.1).

| Offset | Register | Access | Meaning / host action | Specified in |
|---|---|---|---|---|
| `0x484` | **PCIe doorbell push** | WO | One bit at a time. `BIT(0)` = set-firmware-own / **notify Bluetooth ≥ 1 ms before a Wi-Fi FLR**; `BIT(1)` = clear own `[L]`; `BIT(11)` (`0x800`) = host is collecting an assert dump; `BIT(13)` (`0x2000`) = host entering hibernate / going away; `BIT(31)` = force a firmware assert. Reset paths use the configuration cycle, runtime paths the MMIO alias | §2 §2.5/§5.5/§8.5, §5 §9.6, §1 §7.1 |
| `0x488` | **Fabric / power-domain status** | RO | General bus/fabric-liveness guard, **MT7932/MT7923 only** (skipped for `0x7922`); invoked from the DMA-hang detector, error recovery and the debug dumps, not only at probe. Level 1.1/1.2: bits 0, 1, 6, 9, 25, 26 = 1 and bit 4 = 0. Level 2 additionally: bit 10 = 0 and bits 13, 14, 15 = 1. A read of `0xFFFF_FFFF` means configuration access itself failed. Remaining bits `[U]` | §1 §1.3, §2 §6.2, §11 §2.2 item 2 |
| `0x48C` | **Wi-Fi function state**, bits `[19:16]` | RO | `0` = function off / held in reset / MCU in its initial state; `2` = function ready / firmware running. **The MT7932 readiness and power-off gate** (5 ms poll, 5000 ms budget) and the post-FLR "MCU back to initial state" test. Values other than 0 and 2 `[U]` | §1 §1.3, §2 §2.4/§6.3, §5 §7.1.1, §11 §2.2 item 1 |
| `0x490`+ | further vendor dwords | RO `[U]` | The diagnostic dump starts at `0x488` and continues; only `0x488`/`0x48C` are decoded | §11 §2.2 item 12, §11 §6 |

**Not programmed on this part** (§2 §7): Link Control / ASPM, the L1 PM Substates capability,
LTR, Max Payload Size and Max Read Request are all left at platform defaults; there is no
ASPM-disable or link-keep-awake write anywhere in the bring-up, runtime or reset paths.

---

# 4. Command, event and status-code quick reference (part D)

Cross-references only. Every row names the section that specifies the payload in full.
Column conventions: **Dir** — S set, Q query, S+R set with response required.
**Src** — C confirmed in use on this part, P carried over from public gen4m/mt76 and not
exercised here (§6 §5).

## 4.1 Boot / initialisation command IDs (`INIT_CMD_ID_*`)

A namespace **entirely separate** from the runtime command set. Sent on TX ring 17 with the
64-byte boot header (`ucPktTypeID = 0xA0`); responses are fetched from RX ring 0 with the
8-byte initialisation event header. Per-command response timeout 6000 ms. (§5 §6, §6 §8.)

| ID | Name | Payload | Response | Purpose | Payload defined in |
|---|---|---|---|---|---|
| `0x00` | *(firmware / patch payload)* | ≤ 2048 B | none | Raw image bytes on **ring 16**, **no TXD and no header at all** | §5 §5.3, §6 §7.1 |
| `0x01` | `INIT_CMD_ID_DOWNLOAD_CONFIG` | 12 | evt `0x01` | Announce a RAM-code region: `{addr, len, download-mode word}` | §5 §6.2 |
| `0x02` | `INIT_CMD_ID_WIFI_START` | 8 | evt `0x01` | Start the RAM firmware: `{override flags, entry address}` | §5 §6.8 |
| `0x03` | `INIT_CMD_ID_ACCESS_REG` | 12 | evt `0x02` | Chip-register read/write while only the boot ROM runs — how `TOP_HVR`/`TOP_FVR` are read | §5 §6.7 |
| `0x04` | `INIT_CMD_ID_QUERY_PENDING_ERROR` | — | evt `0x03` | Query a pending download error (P) | §5 §6.0 |
| `0x05` | `INIT_CMD_ID_PATCH_START` | 12 | evt `0x01` | Announce a ROM-patch section; same body as `0x01` | §5 §6.2 |
| `0x06` | `INIT_CMD_ID_PATCH_WRITE` | — | — | (P) | §6 §8.2 |
| `0x07` | `INIT_CMD_ID_PATCH_FINISH` | 4 | evt `0x01` | Finalise **and activate** the patch: `{checkCrc, type}`; sent once, not per section | §5 §6.5 |
| `0x08` | `INIT_CMD_ID_PHY_ACTION` | var | evt `0x05` | Pre-power-on PHY action / calibration request (P) | §6 §8.2, §6 §4.4 |
| `0x09` | `INIT_CMD_ID_LOG_TIME_SYNC` | — | — | (P) | §6 §8.2 |
| `0x10` | `INIT_CMD_ID_PATCH_SEMAPHORE_CONTROL` | 4 | evt `0x04` | Patch ownership between Wi-Fi/BT hosts. **MT7932: get = `2`, not `1`** | §5 §6.6, §6 §4.3 |
| `0x11` | `INIT_CMD_ID_BT_PATCH_SEMAPHORE_CONTROL` | — | evt `0x06` | Bluetooth patch semaphore (P) | §5 §6.6 |
| `0x12` | `INIT_CMD_ID_ZB_PATCH_SEMAPHORE_CONTROL` | — | evt `0x07` | 802.15.4 patch semaphore (P) | §5 §6.6 |
| `0x13` | `INIT_CMD_ID_CO_PATCH_DOWNLOAD_CONFIG` | — | — | Combo-patch download config (P) | §6 §8.2 |
| `0x20` | `INIT_CMD_ID_HIF_LOOPBACK` | — | — | HIF loopback test (P) | §6 §8.2 |
| `0x21` | `INIT_CMD_ID_LOG_BUF_CTRL` | 20 | evt `0x08` | Firmware log-buffer base addresses / read-pointer update | §5 §9.1 |
| `0x22` | `INIT_CMD_ID_QUERY_INFO` | 8 | evt `0x09` | Boot-time information query, 16-bit TLV response | §5 §6.0, §6 §2.5b |
| `0x23` | `INIT_CMD_ID_EMI_FW_DOWNLOAD_CONFIG` | — | — | EMI download config — **not used on PCIe** | §5 §5.5 |
| `0x24` | `INIT_CMD_ID_EMI_FW_TRIGGER_AXI_DMA` | — | — | EMI download trigger — not used | §5 §5.5 |
| **`0x50`** | *(boot-time eFuse read)* | 4 | completion event | Read one eFuse word before the RAM firmware runs. **Not in the public gen4m enumeration — an MT7932-generation addition** | §6 §8.2, §9 §3.3 |
| `0xFF` | `INIT_CMD_ID_DECOMPRESSED_WIFI_START` | — | — | Start a compressed image (P) | §6 §8.2 |

**Initialisation event IDs** (8-byte header, payload at RXD + 8; §6 §8.4):

| EID | Name | Payload |
|---|---|---|
| `0x01` | `INIT_EVENT_ID_CMD_RESULT` | `{u8 status; u8 cid; u8 rsv[2]}` — status per §4.5 of this appendix |
| `0x02` | `INIT_EVENT_ID_ACCESS_REG` | `{u32 address; u32 data}` |
| `0x03` | `INIT_EVENT_ID_PENDING_ERROR` | pending error code |
| `0x04` | `INIT_EVENT_ID_PATCH_SEMA_CTRL` | `{u8 status}` — status per §4.5 |
| `0x05` | `INIT_EVENT_ID_PHY_ACTION` | `{u8 event; u8 status; u8 rsv[2]; u32 emiAddr; u32 emiLen; u32 temperature}` |
| `0x06` | `INIT_EVENT_ID_BT_PATCH_SEMA_CTRL` | `{u8 status; u8 rsv[3]; u32 remapAddr; u8 rsv1[4]}` |
| `0x07` | `INIT_EVENT_ID_ZB_PATCH_SEMA_CTRL` | as `0x06` |
| `0x08` | `INIT_EVENT_ID_LOG_BUF_CTRL` | `{u8 type; u8 status; u8 rsv[2]; u32 address; u32 rsv}` |
| `0x09` | `INIT_EVENT_ID_QUERY_INFO_RESULT` | `{u16 totalElementNum; u16 length; TLVs}` |

Two **runtime** commands are also legal before the interface is up and are answered on RX
ring 0: `CMD_ID_GET_NIC_CAPABILITY` (`0x80`) → event `0x01`, and
`CMD_ID_GET_NIC_CAPABILITY_V2` (`0x8A`) → event `0xEC` (§6 §8.3).

## 4.2 Runtime legacy command IDs (`ucCID`, `ucExtenCID = 0`)

Body size excludes the 64-byte header. Full table and field semantics: §6 §5.1;
MAC/PHY-control bodies: §10 part A; power/regulatory bodies: §9 §7/§9.

| CID | Name | Purpose | Dir | Body | Src |
|---|---|---|---|---|---|
| `0x00` | `CMD_ID_DUMMY_RSV` | header-only no-op; primes the command path after boot | S | 0 | C |
| `0x01` | `CMD_ID_TEST_CTRL` | enter/leave RF test mode | S | 0x0C | C |
| `0x02` | `CMD_ID_BASIC_CONFIG` | basic NIC configuration at start-up | S | var | C |
| `0x03` | `CMD_ID_SCAN_REQ_V2` | scan request (§10 §A.4.1) | S | 0x4D4 | C |
| `0x04` | `CMD_ID_NIC_POWER_CTRL` | radio/MAC on-off (§10 §A.1.1) | S | 4 | C |
| `0x05` | `CMD_ID_POWER_SAVE_MODE` | per-BSS power-save profile (§10 §A.16) | S | 4 | C |
| `0x06` | `CMD_ID_LINK_ATTRIB` | link attributes | S | — | P |
| `0x07` | `CMD_ID_ADD_REMOVE_KEY` | install/remove a key (§10 §A.8) | S | 0x40 | C |
| `0x08` | `CMD_ID_DEFAULT_KEY_ID` | default (transmit) key index | S | 4 | C |
| `0x09` | `CMD_ID_INFRASTRUCTURE` | operating-mode set | S | 0 | C |
| `0x0A` | `CMD_ID_SET_RX_FILTER` | RX packet filter word (§10 §A.9.1) | S | 0x44 | C |
| `0x0B` | `CMD_ID_DOWNLOAD_BUF` | generic buffer download | S | — | P |
| `0x0C` | `CMD_ID_WIFI_START` | start Wi-Fi subsystem | S | — | P |
| `0x0D` | `CMD_ID_CMD_BT_OVER_WIFI` | BT-over-Wi-Fi | S | — | P |
| `0x0F` | `CMD_ID_SET_DOMAIN_INFO` | regulatory domain + channel list (§9 §9.2) | S | 0x40 / `12+8N` | C |
| `0x10` | `CMD_ID_SET_IP_ADDRESS` | ARP-offload IPv4 address list | S | 4 | C |
| `0x11` | `CMD_ID_BSS_ACTIVATE_CTRL` | allocate/free a BSS context + own-MAC slot (§10 §A.5.1) | S | 0x0C | C |
| `0x12` | `CMD_ID_SET_BSS_INFO` | full BSS record (§10 §A.5.2) | S | 0x74 | C |
| `0x13` | `CMD_ID_UPDATE_STA_RECORD` | create/update a station record (§10 §A.6.1) | S | 0xC8 | C |
| `0x14` | `CMD_ID_REMOVE_STA_RECORD` | delete station record(s) (§10 §A.6.2) | S | 5 | C |
| `0x15` | `CMD_ID_INDICATE_PM_BSS_CREATED` | power management: BSS created | S | 8 | C |
| `0x16` | `CMD_ID_INDICATE_PM_BSS_CONNECTED` | power management: BSS connected | S | 0x0C | C |
| `0x17` | `CMD_ID_INDICATE_PM_BSS_ABORT` | power management: BSS aborted | S | 4 | C |
| `0x18` | `CMD_ID_UPDATE_BEACON_CONTENT` | beacon template / IE update (§10 §A.14) | S | `8+IE` / 6 | C |
| `0x19` | `CMD_ID_SET_BSS_RLM_PARAM` | channel / bandwidth / protection (§10 §A.2) | S | 0x16 | C |
| `0x1A` | `CMD_ID_SCAN_REQ` | scan v1 — **deprecated** | S | — | P |
| `0x1B` | `CMD_ID_SCAN_CANCEL` | abort a scan (§10 §A.4.2) | S | 4 | C |
| `0x1C` | `CMD_ID_CH_PRIVILEGE` | request/abort an off-channel grant (§10 §A.2a) | S | 0x18 | C |
| `0x1D` | `CMD_ID_UPDATE_WMM_PARMS` | EDCA parameters (§10 §A.13) | S | 0x2C | C |
| `0x1E` | `CMD_ID_SET_WMM_PS_TEST_PARMS` | WMM power-save test parameters | S | 4 | C |
| `0x1F` | `CMD_ID_TX_AMPDU` | TX A-MPDU enable | S | 4 | C |
| `0x20` | `CMD_ID_ADDBA_REJECT` | reject incoming ADDBA | S | — | P |
| `0x24` | `CMD_ID_SET_TX_PWR` | TX power | S | — | P |
| `0x26` | `CMD_ID_P2P_ABORT` | P2P mode switch / abort | S | 4 | C |
| `0x28` | `CMD_ID_SET_DBDC_PARMS` | DBDC / per-band enable (§10 §A.1.2) — **byte `0x04` is device-ID-conditional** | S | 0x24 | C |
| `0x2A` | `CMD_ID_SET_ACL_POLICY` | MAC ACL policy | S | — | P |
| `0x30` | `CMD_ID_ROAMING_TRANSIT` | roaming FSM transition | S | — | P |
| `0x32` | `CMD_ID_SET_NOA_PARAM` | P2P notice of absence | S | var | C |
| `0x33` | `CMD_ID_SET_OPPPS_PARAM` | P2P opportunistic power save | S | var | C |
| `0x3D` | `CMD_ID_SET_GTK_REKEY_DATA` | GTK rekey offload material | S | var | C |
| `0x3E` | `CMD_ID_ROAMING_CONTROL` | roaming enable / thresholds | S | var | C |
| `0x3F` | `CMD_ID_RESET_BA_SCOREBOARD` | reset the BA scoreboard | S | — | P |
| `0x48` | `CMD_ID_SET_NVRAM_SETTINGS` | push NVRAM / manufacturing blob | S | 0x5DC / 0x800 | C |
| `0x4A` | `CMD_ID_SET_WOWLAN` | WoWLAN configuration | S | 0xE4 | C |
| `0x4B` | `CMD_ID_SET_IPV6_ADDRESS` | IPv6 NS-offload address list | S | — | P |
| `0x4F` | *(WoW configuration, extended)* | wake-on-WLAN feature config / query | S,Q | var | C |
| `0x50` | `CMD_ID_SET_SLTINFO` | system-level-test info | S | — | P |
| `0x53` | `CMD_ID_GET_CHIPID` | chip-ID query | Q | — | P |
| `0x58` | `CMD_ID_SET_SUSPEND_MODE` | host suspend notification | S | 0x44 | C |
| `0x5A` | `CMD_ID_SET_RRM_CAPABILITY` | 802.11k capability sync | S | 0x2C | C |
| `0x5B` | `CMD_ID_SET_AP_CONSTRAINT_PWR_LIMIT` | absolute per-BSS power cap (§10 §A.10.2) | S | 0x28 | C |
| `0x5D` | `CMD_ID_SET_COUNTRY_POWER_LIMIT_PER_RATE` | per-rate / SAR / SDB / common-path power tables (§9 §7.7) | S | 0x3A, 0x3C, 0x214 | C |
| `0x5E` | `CMD_ID_SET_TSM_STATISTICS_REQUEST` | traffic-stream measurement start/stop | S | 0x14 | C |
| `0x5F` | `CMD_ID_GET_TSM_STATISTICS` | traffic-stream measurement result | Q | 0x48 | C |
| `0x61` | `CMD_ID_SET_SCAN_SCHED_ENABLE` | scheduled-scan enable | S | 4 | C |
| `0x62` | `CMD_ID_SET_SCAN_SCHED_REQ` | scheduled-scan request | S | var | C |
| `0x6A` | `CMD_ID_UPDATE_AC_PARMS` | per-AC parameter sync | S | 0x10 | C |
| `0x6B` | *(queue-manager BSS/WMM update)* | queue-manager BSS + WMM update | S | 0x20 | C |
| `0x6C` | *(time-sync control)* | 802.1AS/TSF time sync, GPIO trigger, statistics | S,Q | 0x1C | C |
| `0x6E` | `CMD_ID_SET_DROP_PACKET_CFG` | drop-packet configuration | S | — | P |
| `0x70` | `CMD_ID_GET_SET_CUSTOMER_CFG` | vendor key-value configuration push | S | 0x11C | C |
| `0x71` | *(ICMP offload)* | ICMP/ping offload configuration | S | 0x108 | C |
| `0x74` | *(DMS offload enable)* | directed-multicast-service enable | S | 1 | C |
| `0x75` | `CMD_ID_TDLS_PS` | TDLS power save | S | — | P |
| `0x76` | *(scheduled scan, extended)* | extended scheduled-scan parameters | S | var | C |
| `0x77` | *(TX duty cycle)* | TX duty-cycle limit | S | 4 | C |
| `0x79` | `CMD_ID_GET_CNM` | concurrency-manager state query | Q | 0xA1 | C |
| `0x7A` | *(set channel)* | direct channel/RF set | S | 0x10 | C |
| `0x7B` | *(coexistence profile)* | BT-coexistence profile | S | 0x40 | C |
| `0x7C` | `CMD_ID_COEX_CTRL` | coexistence control | S,Q | — | P |
| `0x7E` | `CMD_ID_PERF_IND` | performance / RCPI report to firmware | S | 0x138 | C |
| `0x80` | `CMD_ID_GET_NIC_CAPABILITY` | capability query v1 (**not used**; V2 is) | Q | 0x74 | C |
| `0x81` | `CMD_ID_GET_LINK_QUALITY` | RSSI / link quality / link speed | Q | 0 | C |
| `0x82` | `CMD_ID_GET_STATISTICS` | general statistics (§10 §B.7.2) | Q | 0x0C | C |
| `0x85` | `CMD_ID_GET_STA_STATISTICS` | per-station statistics, last TX rate (§10 §B.7.2) | Q | 0x1C | C |
| `0x87` | `CMD_ID_GET_LTE_CHN` | LTE-safe channel query | Q | 0x14 | C |
| `0x88` | *(statistics-item query)* | MIB / per-band / per-BSS / per-AC counters. **Not in any public enum** (§10 §B.7.2) | Q | 0x0C…0x1C4 | C |
| `0x89` | `CMD_ID_GET_BUG_REPORT` | bug report | Q | — | P |
| `0x8A` | `CMD_ID_GET_NIC_CAPABILITY_V2` | **capability query, TLV form** → event `0xEC` | Q | 0 | C |
| `0x8C` | *(statistics update enable)* | enable periodic statistics reporting | S | 4 | C |
| `0x8D` | `CMD_ID_LOG_UI_INFO` | firmware log level / UI control | S | 0x0C | C |
| `0x8F` | `CMD_ID_RDD_ON_OFF_CTRL` | radar detection (DFS) start/stop | S | 8 | C |
| `0x91` | `CMD_ID_SET_REPORT_BEACON` | per-NSS data-count query | Q | 0x10 | C |
| `0x92` | *(secure ToF / STOF)* | secure time-of-flight enable/auth/offset/timing/start | S | 8…0x9C | C |
| `0x93` | `CMD_ID_SET_ICS_SNIFFER` | in-chip-sniffer filter, time-sync reset | S | 0x54 | C |
| `0x95` | *(legacy 6 GHz power limit)* | 6 GHz legacy per-rate limit | S | var | C |
| `0x96` | *(BSS/STA info dump)* | dump BSS and station tables | Q | 0x0C | C |
| `0x97` | *(chip counters, group 2)* | channel-switch, firmware info, A-MSDU, beacon-loss, management counters | Q | 0x0C, 0x34 | C |
| `0x98` | *(chip counters, group 3)* | TX-rate counters, A-MPDU enable | Q,S | 0x34 | C |
| `0x99` | *(all-station query)* | per-peer counters. **Not in any public enum** (§10 §B.7.2) | Q | 8 | C |
| `0x9A` | *(RMAC security)* | RMAC security key/BK/RS request | S | 8 | C |
| `0x9B` | *(security parameters)* | security parameter block | S | 0x3C | C |
| `0x9D` | *(SDB channel group)* | single-/dual-band channel-group info. **Not in any public enum** | S | 2 | C |
| `0x9E` | `CMD_ID_SET_SAP_SUS` | per-PHY block-ack window size | S | 4 | C |
| `0x9F` | `CMD_ID_SET_SAP_RPS` | Bonjour/mDNS offload records | S,Q | var | C |
| `0xA0` | `CMD_ID_WFC_KEEP_ALIVE` | Wi-Fi-calling keepalive | S | — | P |
| `0xA1` | `CMD_ID_RSSI_MONITOR` | RSSI monitor thresholds | S | — | P |
| `0xA2` | `CMD_ID_PKT_OFLD` | packet offload | S | — | P |
| `0xA3` | *(ARP keepalive)* | ARP keepalive offload | S | 0x44 | C |
| `0xA5` | *(TCP/UDP keepalive)* | TCP/UDP keepalive offload | S | 0x4F8 | C |
| `0xA9` | *(L2 keepalive)* | layer-2 keepalive offload | S | 0x10 | C |
| `0xAB` | *(WNM keepalive)* | 802.11v WNM sleep/keepalive offload | S | 0x44 | C |
| `0xAD` | *(UWB coexistence)* | UWB coexistence mode / critical window | S,Q | 4, 8, 0x20 | C |
| `0xAE` | `CMD_ID_CAL_BACKUP_IN_HOST_V2` | calibration backup in host memory | S,Q | — | P |
| `0xAF` | *(beacon IE parse result)* | push beacon-IE parse result to firmware | S | 4 | C |
| `0xB0` | `CMD_ID_MQM_UPDATE_MU_EDCA_PARMS` | MU-EDCA parameters (§10 §A.13.1) | S | 0x48 | C |
| `0xB1` | `CMD_ID_RLM_UPDATE_SR_PARAMS` | spatial-reuse parameters | S | 0x3C | C |
| `0xB2` | `CMD_ID_PF_CF_COALESCING_INT` | firmware-side RX interrupt coalescing (§4 §6.5) | S | 0x4C | C |
| `0xB3` | `CMD_ID_LP_DBG_CTRL` | low-power debug / DFS-pause config | S | 0x15 | C |
| `0xB6` | *(chip sleep info)* | chip sleep statistics | Q | 0x20 | C |
| `0xB7` | *(action-frame filter)* | action-frame filter configuration | S | 1 | C |
| `0xB8` | *(set station MAC)* | program a station MAC address | S | 8 | C |
| `0xB9` | *(get station MAC)* | read back a station MAC address | Q | 6 | C |
| `0xBA` | *(WF TRX info set)* | Wi-Fi TX/RX information control | S | 4 | C |
| `0xBB` | *(WF TRX info query)* | Wi-Fi TX/RX information read-back | Q | 0xE8 | C |
| `0xBD` | *(motion statistics)* | motion / Doppler statistics | S+R | 4 | C |
| `0xBE` | *(scan private MAC)* | randomised scan MAC query | S+R | 6 | C |
| `0xBF` | *(runtime calibration config)* | firmware runtime-calibration configuration | Q | 0x14 | C |
| `0xC0` | `CMD_ID_ACCESS_REG` | MCR (chip register) read/write at run time | S,Q | 8 | C |
| `0xC1` | `CMD_ID_MAC_MCAST_ADDR` | multicast address list (§10 §A.15) | S | 0xC8 | C |
| `0xC2` | `CMD_ID_802_11_PMKID` | PMKID list | S | — | P |
| `0xC3` | `CMD_ID_ACCESS_EEPROM` | EEPROM access | S,Q | — | P |
| `0xC4` | `CMD_ID_SW_DBG_CTRL` | software debug: CTIA, TP test, **noise floor**, SW-CR access (§10 §B.7.2) | S,Q | 0x108 | C |
| `0xC5` | `CMD_ID_FW_LOG_2_HOST` | firmware-log-to-host enable / level (§5 §9.1) | S | 4 | C |
| `0xC6` | `CMD_ID_DUMP_MEM` | firmware memory dump | Q | 0x10 | C |
| `0xC7` | `CMD_ID_RESOURCE_CONFIG` | TX/RX resource configuration | S,Q | — | P |
| `0xC8` | `CMD_ID_ACCESS_RX_STAT` | RX statistics access | Q | — | P |
| `0xCA` | `CMD_ID_CHIP_CONFIG` | generic chip-config string interface; **SAR enable/level** (§9 §7.6); coexistence statistics; NAN/AWDL parameters | S,Q | 0x148 | C |
| `0xCB` | `CMD_ID_TPUT_INFO` | throughput / statistics log trigger | S | 0x24 | C |
| `0xCD` | `CMD_ID_WTBL_INFO` | WTBL dump | Q | 0xA0 | C |
| `0xCE` | `CMD_ID_MIB_INFO` | MIB counter dump | Q | 0x110 | C |
| `0xD0` | `CMD_ID_GET_TXPWR_TBL` | TX-power table dump | Q | 0x0C | C |
| `0xD3` | *(protocol offload query)* | protocol-offload configuration query | Q | 0x220 | C |
| `0xD4` | *(protocol offload counters)* | protocol-offload counter query | Q | 0xB0 | C |
| `0xD6` | *(one-time calibration)* | calibration type config, one-time-cal get/put (§9 §8.4) | S,S+R | 0x14 | C |
| `0xD7` | *(6 GHz channel scan set)* | 6 GHz channel-scan configuration | S | 1 | C |
| `0xD8` | *(LPAS config)* | low-power always-sensing configuration | S,Q | 0x68 | C |
| `0xD9` | *(WoW test)* | wake-on-WLAN self-test trigger | S | var | C |
| `0xDA` | *(channel-time accounting)* | per-channel dwell-time accounting | Q | 0x80 | C |
| `0xDB` | *(DFS TX pause)* | pause TX for DFS | S | 2 | C |
| `0xDC` | *(RTS threshold)* | protection / RTS threshold | S,Q | 0x10 | C |
| `0xDF` | *(OMI / action frame)* | HE OM-control NSS/OFDMA update, action-frame injection | S,Q | 0x20 | C |
| `0xE0` | *(antenna RSSI config)* | per-antenna RSSI reporting configuration | Q | 2 | C |
| `0xE2` | *(SW DRBG seed)* | push DRBG seed material | S | 0x44 | C |
| `0xE4` | *(NDP offload)* | IPv6 neighbour-discovery offload | S | 0x114 | C |
| `0xE5` | *(DMS offload)* | directed-multicast-service configuration | S | 0x2C | C |
| `0xE7` | *(low-latency window)* | low-latency-window parameters | S,Q | 0x30 | C |
| **`0xEA`** | *(Layer-1 extended command escape)* | MLME / connection-offload family — §4.4 below | S,S+R | — | C |
| `0xEB` | `CMD_ID_NAN_EXT_CMD` | NAN family, ≈45 sub-operations | S,Q | 0x0C…0x533 | C |
| `0xEC` | *(Aware/RA update)* | Wi-Fi Aware rate-adaptation buffer update | S | 0xB8 | C |
| **`0xED`** | `CMD_ID_LAYER_0_EXT_MAGIC_NUM` | Layer-0 extended command escape — §4.3 below | S,Q | — | C |
| `0xEF` | `CMD_ID_INIT_CMD_WIFI_RESTART` | reload firmware | S | — | P |
| `0xF1` | `CMD_ID_SET_BWCS` | BT/Wi-Fi coexistence signalling | S | — | P |
| `0xF6` | `CMD_ID_HIF_CTRL` | **PCIe pre-suspend / resume handshake** (§2 §4.3) | S | 0x24 | C |
| `0xF7` | *(AWDL command family)* | Apple Wireless Direct Link control | S | 0x11…0x92 | C |
| `0xF8` | `CMD_ID_GET_BUILD_DATE_CODE` | firmware build date | Q | — | P |
| `0xF9` | `CMD_ID_GET_BSS_INFO` | AIS/BSS info query | Q | 0 | C |
| `0xFB` | `CMD_ID_SET_TDLS_CH_SW` | TDLS channel switch | S | — | P |
| `0xFC` | `CMD_ID_SET_MONITOR` | monitor-mode configuration (§10 §A.9.3) | S | 0x10 | C |
| `0xFE` | `CMD_ID_SET_MDVT` | advanced/verification control | S | 0x2C | C |

## 4.3 Layer-0 extended command IDs (`ucCID = 0xED`, ID in `ucExtenCID`)

Responses arrive as event `ucEID = 0xED` with the extended event ID at event-header offset
`0x08`. Specified in §6 §5.2; results in §6 §5.6.

| Ext CID | Name | Purpose | Dir | Body | Src |
|---|---|---|---|---|---|
| `0x01` | `EXT_CMD_ID_EFUSE_ACCESS` | read/write one 16-byte eFuse block (§9 §3.3) | Q, S+R | 0x18 | C |
| `0x02` | `EXT_CMD_ID_RF_REG_ACCESS` | RF register access | — | — | P |
| `0x03` | `EXT_CMD_ID_EEPROM_ACCESS` | EEPROM access | — | — | P |
| `0x04` | `EXT_CMD_ID_RF_TEST` | RF test / internal-capture start, status, raw read | S,Q | 0x58 | C |
| `0x05` | `EXT_CMD_ID_RADIO_ON_OFF_CTRL` | radio on/off | — | — | P |
| `0x07` | `EXT_CMD_ID_PM_STATE_CTRL` | power-management state | — | — | P |
| `0x08` | `EXT_CMD_ID_CHANNEL_SWITCH` | channel switch | — | — | P |
| `0x09` | `EXT_CMD_ID_NIC_CAPABILITY` | capability query (WA form) | — | — | P |
| `0x10` | `EXT_CMD_ID_SECURITY_ADDREMOVE_KEY` | key install/remove (WA form) | — | — | P |
| `0x11` | `EXT_CMD_ID_SET_TX_POWER_CONTROL` | TX power control | — | — | P |
| `0x12` | `EXT_CMD_ID_SET_THERMO_CALIBRATION` | thermal calibration | — | — | P |
| `0x13` | `EXT_CMD_ID_FW_LOG_2_HOST` | firmware log to host | — | — | P |
| `0x19` | `EXT_CMD_ID_COEXISTENCE` | coexistence | — | — | P |
| **`0x21`** | `EXT_CMD_ID_EFUSE_BUFFER_MODE` | **push the EEPROM image, the PPR overlay or a one-time-cal blob** (§9 §3.1) | Q, S+R | var | C |
| `0x22` | `EXT_CMD_ID_OFFLOAD_CTRL` | offload control | — | — | P |
| `0x23` | `EXT_CMD_ID_THERMAL_PROTECT` | thermal protection | — | — | P |
| `0x25` | `EXT_CMD_ID_STAREC_UPDATE` | TLV station record — **not used**; `0x13` is used instead | — | — | P |
| `0x26` | `EXT_CMD_ID_BSSINFO_UPDATE` | TLV BSS record — **not used**; `0x12` is used | — | — | P |
| `0x2A` | `EXT_CMD_ID_DEVINFO_UPDATE` | TLV device record — **not used**; `0x11` is used | — | — | P |
| `0x2C` | `EXT_CMD_ID_GET_SENSOR_RESULT` | thermal sensor read | — | — | P |
| `0x32` | `EXT_CMD_ID_WTBL_UPDATE` | WTBL update | — | — | P |
| `0x33` | `EXT_CMD_ID_BCN_UPDATE` | beacon update | — | — | P |
| `0x3C` | `EXT_CMD_ID_GET_MAC_INFO` | MAC information query | Q | — | C |
| `0x4E` | `EXT_CMD_ID_EFUSE_BUFFER_RD` | read back the firmware EEPROM shadow | — | — | P |
| `0x4F` | `EXT_CMD_ID_EFUSE_FREE_BLOCK` | free-eFuse-block count (§9 §3.3) | Q | 4 | C |
| `0x57` | `EXT_CMD_ID_DUMP_MEM` | memory / TX-descriptor dump | Q | 0x44 | C |
| `0x58` | `EXT_CMD_ID_TX_POWER_FEATURE_CTRL` | TX-power feature control, manual per-rate power, power-info query | S,Q | 4, 8 | C |
| `0x81` | `EXT_CMD_ID_SER` | **system-error-recovery trigger**; also WFDMA re-allocation | S+R | 4 | C |
| `0x83` | *(health monitor)* | firmware health-monitor configuration (§2 §6.7) | S+R | 8 | C |
| `0x94` | `EXT_CMD_ID_TWT_AGRT_UPDATE` | TWT agreement update | — | — | P |
| `0xA5` | *(GPIO control)* | GPIO direction / level | S,Q | 8 | C |
| `0xA8` | `EXT_CMD_ID_SR_CTRL` | spatial-reuse control (sub-command in payload byte 0) | S+R | var | C |
| `0xA9` | *(firmware version)* | firmware version / build query | Q | 4 | C |
| `0xBC` | *(secure eFuse access)* | protected eFuse block read/write (§9 §3.4) | Q, S+R | 0x48 | C |
| `0xC1` | *(chip UID)* | read the 64-bit chip unique identifier | Q | 0 | C |
| `0xC2` | *(one-time cal / isolation)* | pre-power-on one-time calibration start/stop, antenna-isolation detect | S,S+R | 0x7C | C |
| `0xC8` | *(FE case control)* | front-end (ePA/eLNA) case selection | S,Q | 0x0C | C |
| `0xCA` | *(smart CCA set)* | smart-CCA configuration; also raises an async notification | S+R | var | C |
| `0xCB` | *(smart CCA status)* | smart-CCA status read-back | Q | 0 | C |

## 4.4 Layer-1 extended command IDs (`ucCID = 0xEA`, ID in `ucExtenCID`)

The firmware-resident MLME / connection-offload interface. **Not present in public gen4m.**
Responses and notifications arrive as event `0xEA` or `0xEE`. All confirmed for this part.
Specified in §6 §5.3.

| Ext CID | Purpose | Dir | Body |
|---|---|---|---|
| `0x08` | AP / hostapd offload configuration | S | var |
| `0x30` | RX authentication frame handed to the firmware supplicant | S | var |
| `0x31` | SAE authentication request | S | var |
| `0x40` | **Connect request** (full association parameters) | S | 0x358 |
| `0x41` | Install PMK / PMKSA entry | S | var |
| `0x42` | **Disconnect / link-down** | S | var |
| `0x43` | Set vendor ("product") IE | S | 0x10 |
| `0x44` | Set the RSSI→rate mapping table | S | 0x5C |
| `0x46` | Delete PMK / PMKSA entry | S | var |
| `0x47` | Release channel privilege after DHCP completion | S | var |
| `0x48` | Beacon-report request configuration (802.11k) | S | 0x90 |
| `0x49` | BSS-transition-management (802.11v) parameters | S | var |
| `0x4A` | Record client IE information | S | var |
| `0x4B` | Reassociation request | S | var |
| `0x4C` | User roaming cache set / query | S+R | 0x58 |
| `0x4D` | BSS blacklist | S | var |
| `0x4E` | MLME configuration update | S | 4 |
| `0x4F` | Read back PMK | S+R | 0x5C |
| `0x51` | Inform firmware of an SDB (single/dual-band) switch | S | 4 |
| `0x60` | Push reduced-neighbour-report information | S | 0x14 |
| `0x61` | User roaming cache channel list | S+R | 0x84 |
| `0x62` | Roaming split-scan list query | S+R | 0x44 |

Extended event `0x11` inside this family is the WPA/connection-status report; every other
sub-event carries an opaque payload for the host's supplicant (§6 §5.6).

## 4.5 Runtime event IDs (`ucEID`)

The complete set of event IDs that are **separately decoded** on this part. Any other event ID
is matched to the outstanding command whose 8-bit sequence number it echoes. Kind: **U** =
unsolicited (`ucSeqNum == 0`), **S** = solicited response, **S/U** = both occur.
Specified in §6 §5.4; host obligations in §6 §5.4a; the ★ rows are the ones whose loss
desynchronises host and firmware.

| EID | Name | Purpose | Kind |
|---|---|---|---|
| `0x01` | `EVENT_ID_NIC_CAPABILITY` | capability response v1 (handled synchronously at boot) | S |
| `0x02` | `EVENT_ID_LINK_QUALITY` | RSSI / link quality / link speed | S/U |
| `0x03` | `EVENT_ID_STATISTICS` | general statistics (response to cmd `0x82`) | S |
| `0x04` | `EVENT_ID_MIC_ERR_INFO` | TKIP MIC failure report | U |
| ★ `0x07` | `EVENT_ID_SLEEPY_INFO` | firmware asks the host to release ownership; body byte 0 non-zero = release now, sent **once** | U |
| ★ `0x0A` | `EVENT_ID_RX_ADDBA` | inbound BA agreement established → create the host reorder window (§6 §5.4b) | U |
| ★ `0x0B` | `EVENT_ID_RX_DELBA` | inbound BA agreement torn down → flush and destroy the window | U |
| ★ `0x0D` | `EVENT_ID_SCAN_DONE` | scan completion; the sole completion signal | U |
| `0x0F` | `EVENT_ID_TX_DONE` | per-PID TX status: WTBL index, PID, status, SN, TID, retry count, flush flag | U |
| ★ `0x10` | `EVENT_ID_CH_PRIVILEGE` | channel grant / reject / recover / remove-request (§10 §A.2a) | U |
| ★ `0x11` | `EVENT_ID_BSS_ABSENCE_PRESENCE` | BSS off/on channel + BSS free quota → gate that BSS's queues | U |
| ★ `0x12` | `EVENT_ID_STA_CHANGE_PS_MODE` | peer entered/left power save | U |
| ★ `0x13` | `EVENT_ID_BSS_BEACON_TIMEOUT` | beacon loss on a BSS index — the link is already down | U |
| `0x14` | `EVENT_ID_UPDATE_NOA_PARAMS` | P2P notice-of-absence update | U |
| `0x15` | `EVENT_ID_AP_OBSS_STATUS` | OBSS status, AP role | U |
| ★ `0x16` | `EVENT_ID_STA_UPDATE_FREE_QUOTA` | per-station transmit credit: `{sta, update mode, quota}` | U |
| `0x18` | `EVENT_ID_ROAMING_STATUS` | roaming FSM status | U |
| ★ `0x19` | `EVENT_ID_STA_AGING_TIMEOUT` | peer aged out → free the station index host-side | U |
| ★ `0x1B` | `EVENT_ID_SEND_DEAUTH` | firmware sent / wants a deauthentication → complete the disconnect | U |
| `0x1C` | `EVENT_ID_UPDATE_RDD_STATUS` | radar-detection status | U |
| `0x1D` | `EVENT_ID_UPDATE_BWCS_STATUS` | BT/Wi-Fi coexistence signalling status | U |
| `0x1E` | `EVENT_ID_UPDATE_BCM_DEBUG` | coexistence debug | U |
| `0x20` | `EVENT_ID_DUMP_MEM` | firmware memory-dump payload | S |
| `0x23` | `EVENT_ID_SCHED_SCAN_DONE` | scheduled-scan completion | U |
| `0x24` | `EVENT_ID_ADD_PKEY_DONE` | pairwise key installed | U |
| `0x25` | `EVENT_ID_ICAP_DONE` | internal capture complete | U |
| ★ `0x27` | `EVENT_ID_DEBUG_MSG` | firmware log record — consume and free the buffer | U |
| ★ `0x2E` | `EVENT_ID_TX_ADDBA` | outbound BA established: buffer size + **A-MSDU-in-A-MPDU bitmap** | U |
| `0x3E` | *(thermal notify)* | thermal threshold crossed | U |
| `0x4D` | *(WoW RX packet info)* | contents of the wake-up packet | U |
| `0x50` | `EVENT_ID_RDD_SEND_PULSE` | radar pulse report | U |
| `0x5D` | *(scan start report)* | scan started: channel, type, source | U |
| `0x5E` | *(scan done report)* | detailed scan-done: state, reason, hit count | U |
| `0x60` | `EVENT_ID_RDD_REPORT` | radar detected | U |
| `0x61` | `EVENT_ID_CSA_DONE` | channel-switch-announcement complete | U |
| ★ `0x62` | `EVENT_ID_WOW_WAKEUP_REASON` | wake-up reason detail | U |
| `0x63` | `EVENT_ID_OPMODE_CHANGE` | operating-mode change | U |
| `0x64` | `EVENT_ID_LTE_IDC_REPORT` | LTE in-device coexistence report | U |
| `0x6C` | *(time sync)* | time-sync sub-event family | U |
| `0x6D` | *(time-sync trigger)* | time-sync capture trigger | U |
| `0x78` | `EVENT_ID_DBDC_SWITCH_DONE` | DBDC hardware switch complete | S/U |
| `0x79` | `EVENT_ID_GET_CNM` | concurrency-manager info | S |
| `0x7A` | *(set-channel done)* | channel/RF set completed | S/U |
| `0x7C` | `EVENT_ID_COEX_CTRL` | coexistence control response | S/U |
| `0x8C` | *(channel switch)* | channel-switch notification | U |
| `0x90` | `EVENT_ID_UPDATE_COEX_PHYRATE` | coexistence PHY-rate update | U |
| `0x92` | *(secure ToF)* | secure time-of-flight sub-event family | S/U |
| `0x9A` | *(RMAC security)* | RMAC security response | S |
| `0x9C` | *(trigger capture)* | firmware-requested capture trigger | U |
| `0xA1` | `EVENT_ID_RSSI_MONITOR` | RSSI threshold crossed | U |
| `0xB2` | `EVENT_ID_PF_CF_COALESCING_INT_DONE` | coalescing configuration applied | S |
| ★ `0xCC` | `EVENT_ID_CHECK_REORDER_BUBBLE` | reorder-window hole persisted → advance the window (upstream numbers this `0x2A`; **`0xCC` here**) | U |
| `0xCD` | `EVENT_ID_WTBL_INFO` | WTBL dump | S |
| `0xCE` | `EVENT_ID_MIB_INFO` | MIB counter dump | S |
| `0xD4` | *(protocol-offload counters)* | protocol-offload counter response | S |
| `0xD6` | *(one-time cal data)* | calibration data from firmware | S/U |
| ★ `0xD7` | *(one-time cal data request)* | **firmware asks the host for a cached calibration block** — the host must answer (§9 §8.4a) | U |
| `0xD8` | *(LPAS)* | low-power always-sensing event | U |
| `0xD9` | *(one-time cal load status)* | calibration-load result | U |
| `0xDA` | *(channel-time report)* | per-channel dwell-time accounting | S |
| `0xDE` | *(current FE loss)* | current front-end loss values | S |
| `0xDF` | *(vendor request info)* | vendor request information | U |
| `0xEA` | *(Layer-1 extended event)* | MLME / connection-offload family (§4.4) | S/U |
| `0xEB` | `EVENT_ID_NAN_EXT_EVENT` | NAN sub-event family | S/U |
| `0xEC` | `EVENT_ID_NIC_CAPABILITY_V2` | **capability response, TLV** (§4.8) | S |
| `0xED` | `EVENT_ID_LAYER_0_EXT_MAGIC_NUM` | Layer-0 extended-event escape (§4.6) | S/U |
| `0xEE` | *(Layer-1 extended event, alias)* | same handler as `0xEA` | S/U |
| ★ `0xF0` | `EVENT_ID_ASSERT_DUMP` | **firmware assertion / coredump stream** — acknowledge, collect, then reset (§5 §9.5) | U |
| ★ `0xF6` | `EVENT_ID_HIF_CTRL` | PCIe pre-suspend completion; handshake bytes 1 and 2 both `2` | U |
| `0xF7` | *(AWDL event family)* | Apple Wireless Direct Link sub-events | S/U |
| `0xF9` | `EVENT_ID_GET_AIS_BSS_INFO` | AIS/BSS info response | S |

Public event IDs **absent** from this part's separately-decoded set (they would be matched by
sequence number if emitted) are listed in §6 §5.4; they include `0x0C ACTIVATE_STA_REC`,
`0x17 SW_DBG_CTRL`, `0x21/0x22 STA_STATISTICS`, `0xFD INIT_EVENT_CMD_RESULT`.

## 4.6 Layer-0 extended event IDs (event `ucEID = 0xED`, ID at event-header offset `0x08`)

As a rule the extended event ID equals the extended command ID that produced it (§6 §5.6).
Values with dedicated handling:

| Ext EID | Purpose | Result size |
|---|---|---|
| `0x00` | generic extended-command result | 0x404 |
| `0x01` | eFuse access result (address / valid / data) | 0x18 |
| `0x04` | RF-test / internal-capture result | var |
| `0x3C` | MAC information | 0x0C |
| `0x4C` | per-station parameter update, indexed by WLAN index | — (unsolicited) |
| `0x57` | memory / TX-descriptor dump | 0x44 |
| `0x58` | TX-power information | 0x135 |
| `0x81` | SER / WFDMA-realloc result | 0x250 |
| `0x8A` | EEPROM/calibration TLV blob to host (sub-type in payload byte 1; type 3 = calibration image) | var |
| `0xA6` | 1-byte status result | 8 |
| `0xA8` | spatial-reuse control result (sub-event in payload byte 0) | 6 |
| `0xAA` | 32-bit scalar result | 4 |
| `0xBC` | secure-eFuse result (≤ 32 bytes of key material) | 0x48 |
| `0xC1` | chip UID (64-bit) | 8 |
| `0xC2` | one-time-calibration / isolation-detect condition stop | var |
| `0xC8` | FE-case control result | 0x0C |
| `0xCA` | smart-CCA host notification | var (unsolicited) |
| `0xCB` | smart-CCA status, 14 × 64-bit counters | 0x7C |

## 4.7 Status and return-code enumerations

### 4.7.1 Firmware-download / boot command result — event `0x01`, payload byte 0

**Selected at run time by PCI device ID.** This is the enumeration most likely to break a
ported driver silently. (§5 §6.9, §6 §4.2, §11 §2.3.)

| Code | MT7922 (`0x7922`) — public `WIFI_FW_DOWNLOAD_*` | **MT7932 / MT7923 (`0x7932`, `0x7923`)** |
|---|---|---|
| 0 | **success** | *(unused)* |
| 1 | invalid param | **success** |
| 2 | invalid crc | unknown |
| 3 | decrypt fail | invalid param |
| 4 | unknown | invalid crc |
| 5 | timeout | timeout |
| 6 | sec boot fail | sec boot check fail |
| 7 | — | region check fail |
| 8 | — | cmd size check fail |
| 9 | — | RAM entry check fail |
| 10 | — | section check fail |
| 11 | — | FW download flow check fail |
| 12 | — | FW download cmd logic check fail |

A host must test the boot result byte against **`1`**, not `0`. `[C]`

### 4.7.2 Patch semaphore — command `0x10` operation byte, event `0x04` status byte

Also shifted by +1 on MT7932/MT7923. (§5 §6.6, §6 §4.3, §11 §2.2 items 5–6.)

| Meaning | MT7922 value | **MT7932 value** | Conf. |
|---|---|---|---|
| Operation: release semaphore | 0 | **1** | `[L]` |
| Operation: **get semaphore** | 1 | **2** | `[C]` |
| Status: no semaphore, patch still needed (retry) | 0 | **1** | `[C]` |
| Status: **patch already resident — skip the download** | 1 | **2** | `[C]` |
| Status: semaphore obtained, host must download | 2 | **3** | `[L]` |
| Status: semaphore released | 3 | **4** | `[L]` |

### 4.7.3 Generic runtime command status (`CMD_STATUS_*`, event `0xFD` payload)

`0x00` success · `0x01` rejected · `0x02` unknown command · `0xFE` not supported by this
build. Not separately decoded on this part, so such a frame is matched by sequence number.
(§6 §4.1, `[L]`.)

### 4.7.4 Other enumerations

| Enumeration | Values | Specified in |
|---|---|---|
| Pre-power-on PHY-action status | `0` success · `1` fail · `2` recalibration required · `3` ePA/eLNA configuration `[L]` | §6 §4.4 |
| Channel-privilege status (`EVENT_ID_CH_PRIVILEGE` byte `0x02`) | `0` grant · `1` reject · `2` recover · `3` remove-request | §10 §A.2a |
| `STA_UPDATE_FREE_QUOTA` update mode | `0`,`1` set · `2` add · `3` subtract · anything else = protocol violation | §6 §5.4a, §10 §A.6a.2 |
| Station state (`ucStaState`) | `0` not authenticated · `1` authenticated · `2` associated (data path on) | §10 §A.7 |
| Beacon-template update method | `0` update (randomised-address variant) · `1` update all · `2` delete all · `≥3` rejected | §10 §A.14 |
| `ucSourceMode` (EFUSE buffer mode) | `0` efuse · `1` binary image · **`2` PPR overlay** · **`3` one-time-cal blob** — modes 2 and 3 are MT7932 extensions (public parts define 0 and 1 only) | §9 §1.3 |
| Board type (from efuse `0x07A` / `0x164`) | `1` dev board, efuse burned · `2` dev board, efuse blank · `3` production "FF" module | §9 §1.2 |
| Cipher / `CIPHER_SUITE` / `SEC_MODE` | `0` none · `1` WEP-40 · `2` TKIP · `3` TKIP-no-MIC · `4` CCMP-128 · `5` WEP-104 · `6` BIP-CMAC-128 · `7` WEP-128 · `8` WPI/SMS4 · `9` CCMP-CCX · `10` CCMP-256 · `11` GCMP-128 · `12` GCMP-256 · `13` GCM-WPI-128 · `14`–`19` BIP/beacon-protection variants `[L]` | §10 §A.8.1, §8 §3.9 |
| TX-free report version (`DW1[18:16]`) | `0` 2-byte entries · `1` token in `[15:0]` · `2` token in `[14:0]` · `3` token in `[30:16]`, `BIT(31)` = pair record, **skip it** | §8 §6.3 |
| MMIO read sentinels | `0xFFFF_FFFF` bus failure (unless whitelisted) · `0xDEAD_FEED` conn-infra "no response" · `0xDEAD_0001` host-bus timeout `[L]` | §1 §3.3, §2 §6.4 |
| `MCU2HOST_SW_INT_STA` reason bits (`0x7C02_41F0`) | 0 wake-by-RX · 1 stop-PDMA-with-reload · 2 stop-PDMA · 3 reset-done · 4 recovery-done · 5 MCU-normal · 6 SER-in-suspend · 7 SER-done-in-suspend · 8/9 LMAC-hang workaround · 10 SER bus hang · 24–28 LMAC/PSE/PLE/PDMA/PCIe error. **Acted-on mask `0x0000_003C`** | §4 §3.1, §2 §8.1 |
| `HOST2MCU_SW_INT_SET` codes (`0x5400_0108`) | `0x01` PDMA stopped · `0x02` PDMA re-initialised · `0x08` SER handling finished · `0x10` host-initiated SER | §4 §3.2, §2 §8.1 |
| Recovery reason codes (0–22) | 2 driver-own failure · 3/4 firmware assert done/timeout · 8 CR access failure · 13 watchdog · 14/15 SER L1/L0.5 failed · 16 FLR · 17 PCIe access failure · 18/19 FLR requested by SER / health monitor — full list and the reason→action table | §2 §8.4 |
| Recovery action flags | `BIT(0)` core dump · `BIT(1)` prevent power-off · `BIT(2)` L0 · `BIT(3)` L0.5 · `BIT(4)` L1 | §2 §8.4 |
| Firmware health-monitor codes (shared block `+0x164`) | `0x000001` clock · `0x02` thermal · `0x04` power · `0x08` radio · `0x10` channel switch · `0x20` internal state machine · `0x40` radio probe · `0x100000` security | §2 §6.7 |
| `sec_info` → download-mode word | `0xFFFF_FFFF` / `[31:24]=0x00` ⇒ `0x8000_0000`; `0x01` AES ⇒ `0x8000_0009 \| (key<<1)`; `0x02` scramble ⇒ `0x8000_0049` | §5 §4.2 |
| RXD `PKT_TYPE` (`DW0[31:27]`) | `0` TX status · `1` RX vector · `2` data · `3` duplicate RFB · `4` TM report · `6` **TX-free/MSDU report** · `7` software-defined (`0x3800` event / `0x3801` frame) · `8` MCU-injected frame · `11` RX report · `12`/`13` ICS / PHY-ICS log | §8 §6.1, §6 §3.1 |
| ECO table `{hw_ver, rom_ver, factory_ver}` → stepping | `{0x00,0x00,0x0A}` → **E1**; `{0x01,0x01,0x0A}` → **E2**; no match ⇒ use the previous entry. **Public MT7961 uses `0x10` for E2's hw_ver** | §1 §6.3, §11 §1.2 |

## 4.8 Capability-query record tags (event `0xEC` body, 32-bit TLV)

Element stride = `body_len + 8`; the element count in the fixed part is authoritative
(§6 §2.5a). 31 tags are defined for this part. Payload structures and how each field is
consumed: §11 §3.7; the same list appears as §6 §5.5.

| Tag | Public gen4m name | Content / effect |
|---|---|---|
| `0x01` | `TAG_CAP_TX_EFUSEADDRESS` | eFuse start address and size |
| `0x02` | `TAG_CAP_COEX_FEATURE` | 32-bit coexistence feature word |
| `0x03` | `TAG_CAP_SINGLE_SKU` | regulatory single-SKU power table supplied by firmware |
| `0x05` | `TAG_CAP_HW_VERSION` | product ID, ECO version, MAC / BB / TOP IP IDs, configuration ID |
| `0x06` | `TAG_CAP_SW_VERSION` | firmware version, build number, branch tag, date code |
| `0x07` | `TAG_CAP_MAC_ADDR` | factory MAC address |
| `0x08` | `TAG_CAP_PHY_CAP` | **the authoritative PHY profile**: VHT, 5 GHz, max bandwidth, `ucNss`, DBDC, LDPC/STBC, `ucWifiPath`, HE. **No `ucEht` field exists** |
| `0x09` | `TAG_CAP_MAC_CAP` | hardware BSSID count (1–4), WMM set count, **WTBL entry count (1–49 accepted)** |
| `0x0A` | `TAG_CAP_FRAME_BUF_CAP` | accepted, **discarded** |
| `0x0B` | `TAG_CAP_BEAMFORM_CAP` | accepted, **discarded** |
| `0x0C` | `TAG_CAP_LOCATION_CAP` | accepted, **discarded** |
| `0x0D` | `TAG_CAP_MUMIMO_CAP` | accepted, **discarded** |
| `0x14` | `TAG_CAP_HW_ADIE_VERSION` | **A-die product ID** (the only source on this part — no A-die register) |
| `0x17` | *(gen4m: `TAG_CAP_P2P`)* | **WFDMA reallocation descriptor** — when its selector byte is non-zero, a command-class TX **ring index** is rewritten from `17` to `18` |
| `0x18` | `TAG_CAP_6G_CAP` | 6 GHz support, RF paths, DBDC A+A and its minimum frequency separation. **Honoured on `0x7932`; discarded on `0x7923`** |
| `0x1E` | *(gen4m: `TAG_CAP_REDL_INFO`)* | 1-byte maximum RMAC quota |
| `0x1F` | *(gen4m: `TAG_CAP_HOST_SUSPEND_INFO`)* | MLME-offload address-randomisation capability |
| `0x20` | *(gen4m: `TAG_CAP_MLR_CAP`)* | roaming / BSS-transition policy |
| `0x30` | vendor | 6 GHz support for the peer-to-peer discovery mode (AWDL) |
| `0x31` | vendor | 32-bit NAN feature bitmap (bits 0–6 consumed) |
| `0x32` | vendor | low-latency TX path |
| `0x33` | vendor | low-latency RX path |
| `0x34` | vendor | secure ToF, two sub-flags |
| `0x35` | vendor | UWB coexistence |
| `0x36` | vendor | external PTA |
| `0x37` | vendor | time-sync accuracy improvement |
| `0x38` | vendor | runtime-calibration support flag |
| `0x39` | vendor | suspend-mode offload, four sub-flags |
| `0x3A` | vendor | per-antenna RSSI in reports |
| `0x3B` | vendor | negotiated PCIe link speed |
| `0x3C` | vendor | SmartCCA |

Tags not listed may be skipped. If the query goes unanswered the host must fall back to a
fixed default profile (§11 §3.7).

---

# 5. Quick-start register sequence (part E)

A single ordered path from "PCI function appears" to "firmware running and answering
commands". It is a **condensation of §1–§5 and is consistent with the state machine of
§12** — the state each step reaches is named in the last column. Nothing here is new; every
value is specified in the section cited by the corresponding row of §2, §3 or §4 of this
appendix. `cfg` = PCI configuration space; everything else is a chip address, with the
BAR-0 offset a host actually issues in brackets.

| # | Register / command | Value | Expected result | State |
|---|---|---|---|---|
| 1 | `cfg 0x04` COMMAND | RMW: set Memory Space + Bus Master | BAR-0 decodes; DMA permitted | S0 |
| 2 | PM capability | put the function in **D0** | device out of low power | S0 |
| 3 | `cfg 0x10` / `0x14` | read | BAR-0 base; map the first `0x10_0000` only | S0 |
| 4 | host DMA mask | `0xFFFF_FFFF` | 32-bit descriptors throughout; no address extension | S0 |
| 5 | MSI capability | allocate **1** vector (fall back to INTx) | single-vector layout, ack mask `0xEEC8_000D` | S0 |
| 6 | `cfg 0x488` | read | bits 0,1,6,9,25,26 = 1 and bit 4 = 0, else the fabric is down. `0xFFFF_FFFF` ⇒ configuration access itself failed | S1 |
| 7 | `cfg 0x48C` | read `[19:16]` | `0` = MCU in its initial state. If non-zero, loop 50 × (read, write `0x7C06_0010 = BIT(0)`, wait 50 µs); still non-zero ⇒ jump to step 14 | S1 |
| 8 | `0x7C06_0010` [`0x0E0010`] | write `BIT(1)` (`HOST_CLR_OWN`), then poll bit 2 == 0; 1 ms tick, re-issue the write every 200 ms, 2048 ms budget | **host owns the chip** — all Wi-Fi-domain MMIO is now legal | S1→S2 |
| 9 | `0x8002_1010` [`0x0B1010`] | read | neither `0xFFFF_FFFF` nor `0xDEAD_FEED` ⇒ the WF bus is alive | S2 |
| 10 | `0x7C00_E250` [`0x0FE250`] | write `0x7000_1846` | after ≥ 2 µs, read back `[31:16] == 0x7000`; failure is fatal. CB-TOP now at BAR `0x70000` | S3 |
| 11 | `0x7C00_E254` [`0x0FE254`] | write `0x1805_1848` | after ≥ 2 µs, read back `[31:16] == 0x1805`. CONN_INFRA SYSRAM now at BAR `0x90000` | S3 |
| 12 | `0x7C02_4208` [`0x0D4208`] | RMW `&= 0xE7DF_7FFA`, then poll `& 0x0000_000A == 0`; 1000 µs × 101 | TX/RX DMA disabled and `TX_DMA_BUSY`/`RX_DMA_BUSY` clear — engine idle | S3→S4 |
| 13 | `0x7C02_4100` [`0x0D4100`] | write `0`, then `0x30` | HIF logic + DMASHDL reset (instance 0 only; WFDMA1 is absent) | S4 |
| 14 | `0x7C00_E24C` [`0x0FE24C`] then BAR `0x043020` (chip `0x7000_3020`) | write `0x1845_7000`, wait 2 µs; write `0x0000_0000`; wait 1000 µs | slot 4 borrowed for CB-TOP; MTCMOS hardware-mode sequence done — **required on MT7932, absent on MT7922** | S5 |
| 15 | BAR `0x042600` (chip `0x7000_2600`) | write `0x0000_0001`; hold **50 ms**; write `0x0000_0000`; then `0x7C00_E24C = 0x1845_184F` | `WF_WHOLE_PATH_RST` asserted and released; slot 4 restored so `0x8800_xxxx` addressing works again | S5 |
| 16 | `0x7C00_0140` [`0x0F0140`] | poll `BIT(4)`; 3 attempts, 100 ms apart | `WFSYS_SW_INIT_DONE` — subsystem back in its initial state. **Do not** write `0x7C00_1620` here (MT7922 only) | S5 |
| 17 | repeat step 8 | — | ownership re-acquired after the reset | S2→S6 |
| 18 | TX ring *N* `CTRL0`/`CTRL2`/`CTRL1` [`0x0D4300 + N*0x10`] | `desc_pa_lo` / `0` / `((pa_hi & 0xF) << 16) \| MAX_CNT` | 18 populated TX rings: 0–14 data, 16 FWDL (256), 17 CMD (24), 18 alt-CMD (16) | S6 |
| 19 | RX ring *N* `CTRL0`/`CTRL2`/`CTRL1` [`0x0D4500 + N*0x10`] | `desc_pa_lo` / `MAX_CNT − 1` / as above; clear `DMA_DONE` in every descriptor | 8 populated RX rings: 0, 2–8; 2352-byte buffers, 4-byte aligned | S6 |
| 20 | `0x7C02_4208` [`0x0D4208`] | RMW clear `BIT(15)` | `CSR_DISP_BASE_PTR_CHAIN_EN` off so manual prefetch bases are honoured | S6 |
| 21 | RX then TX `EXT_CTRL` [`0x0D4680 + N*4`, then `0x0D4600 + N*4`] | `(base << 16) \| depth`, ascending, one shared accumulator | 26 windows, `0x000`–`0x840`; RX 2 depth 8, TX 4/8/12 depth 12, the rest 4 | S6 |
| 22 | `0x7C02_420C` [`0x0D420C`] | write `0xFFFF_FFFF` | all TX DMA pointers reset | S6 |
| 23 | `0x7C02_601C` [`0x0D601C`], `0x7C02_600C` [`0x0D600C`] | `0x0000_0001` (mask `0x0FFF_0FFF`); then clear bits 17:16 and set bit 16 | DMASHDL PLE max page 1 / PSE max page 0; scheduler-control bit 16 set (public CONNAC2 calls it `GROUP_SEQUENCE_ORDER_TYPE`; the slot arbiter at bit 17 stays clear — §3 §8.2) | S6 |
| 24 | `0x7C02_6060 + 4k` / `0x7C02_6070 + 4k` | identity maps | queue *q* → group *q* for 0–15; priority *i* → group *i* | S6 |
| 25 | `0x7C02_6020 + 4n` [`0x0D6020 + 4n`], then `0x7C02_6010` [`0x0D6010`] | groups 0–3 max 512/min 40; 4–14 max 256, min 20 on 4/8/12 else 40; group 15 zero. Then set only bit 31 | per-group page credits programmed; refill on for groups 0–14, off for 15 | S6 |
| 26 | `0x7C02_4208` [`0x0D4208`] | RMW `\|= 0x5020_9040` | `CLK_GATE_DIS`, `OMIT_TX_INFO`, `OMIT_RX_INFO_PFET2`, chain-en, FIFO-LE, `TX_WB_DDONE` | S6 |
| 27 | `0x7C02_4208` [`0x0D4208`] | RMW `\|= 0x0000_0005` | **TX and RX DMA enabled**; the register reads back with both set | S6 |
| 28 | `0x7C02_4298` [`0x0D4298`] | write `0x0000_000C` | RX rings 2 and 3 on the priority interrupt path | S6 |
| 29 | `0x7C02_7038` [`0x0D7038`] | write `0x0000_0013` | ext-wrap CSR programmed (field meaning `[U]`) | S6 |
| 30 | `0x7C02_42F0` [`0x0D42F0`] | write `0x8032_800A` | RX 2 coalesced at 200 µs, RX 3 at 1000 µs | S6 |
| 31 | `0x7C02_42E8` [`0x0D42E8`] | write `0x01FD_0032` | periodic delayed interrupt on all eight present RX rings, 1000 µs | S6 |
| 32 | `0x5400_0120` [`0x002120`] | set `BIT(1)` | WFDMA "initialised" flag set for the L1/deep-sleep re-init test | S6 |
| 33 | `0x7C05_3C28`, then `0x7C05_3A38`, `0x7C05_3A3C`, `0x7C05_3A30`, `0x7C05_3A54` [`0x093C28`, `0x093A38`, …] | `1`; then `BIT(1)`; then the three buffer addresses, low 32 bits | share-info block published to the MCU (order matters: doorbell first) | S6 |
| 34 | — | wait **1 ms** | mandatory pre-interrupt-enable delay (`[U]` root cause) | S6→S7 |
| 35 | `0x7403_0188` [`0x010188`] | write `0x0000_01FF` | PCIe-MAC master interrupt enable (nine sources; mt76 writes `0xFF`) | S7 |
| 36 | `0x7C02_4204` [`0x0D4204`] | write `(OR of present RX-ring bits & 0x93CF_FFFF) \| 0x6C00_0000` = **`0xEEC8_000D`** | WFDMA host interrupt enable; read back to push the write | S7 |
| 37 | `0x7C02_41F4` [`0x0D41F4`], `0x7C06_0018` [`0x0E0018`] | `0x0000_FFFF`; then `0x0000_0001` | all 16 MCU→host software-interrupt reasons unmasked; firmware-cleared-own interrupt enabled | S7 |
| 38 | boot cmd `0x03` (query `0x7001_0204`), then `0x03` (query `0x8800_0004`) | ring 17, 64-byte boot header, `ucPktTypeID = 0xA0` | event `0x02` twice → `TOP_HVR` and `TOP_FVR` → ECO stepping → firmware file names | S8 |
| 39 | boot cmd `0x10` | operation = **`2`** | event `0x04`: status **`2`** ⇒ patch already resident, skip to step 42; status `3` ⇒ download it; status `1` ⇒ retry | S8→S9 |
| 40 | boot cmd `0x05`, then the section data | `{0x0090_0000, section length, 0x8000_0000}` | event `0x01` status **`1`**; then stream the section on **ring 16**, ≤ 2048 B per packet, **no header** | S9 |
| 41 | `0x7C00_E24C` = `0x1845_1807`; poll BAR `0x040060` bit 0 (1000 µs × 5000); boot cmd `0x07` `{ucCheckCrc = 0}`; BAR `0x040260` = `1`; `0x7C00_E24C` = `0x1845_184F` | — | patch finalised **and activated** under the secure-boot semaphore, which is then released and slot 4 restored | S9 |
| 42 | boot cmd `0x01` per RAM region, then stream on ring 16 | `{region address, length, 0x8000_0000}` | event `0x01` status `1` per region; five regions for the shipped image | S9 |
| 43 | semaphore acquire (as step 41); boot cmd `0x02`; semaphore release + slot-4 restore | `{override = 0x0000_0001, entry = 0x0090_7C60}` | event `0x01` status **`1`** ⇒ firmware started | S9 |
| 44 | `cfg 0x48C` | poll `[19:16] == 2`; 5 ms tick, **5000 ms** budget | **firmware running.** Do *not* poll `0x7C06_00F0` — that is the MT7922 branch | S9→S10 |
| 45 | event ring switch | RX ring 0 → **RX ring 4** | MCU events now arrive on ring 4; ring 0 goes idle | S10 |
| 46 | runtime cmd `0x00` on ring 17 | header only, `0x40` bytes | command path primed | S10 |
| 47 | runtime cmd `0x8A` (query) | 0-byte body | event `0xEC` with the capability TLVs → §4.8 | S10→S11 |

From S11 onward the interface is command-driven: RF provisioning (§9), MAC configuration
(§10 part A) and radio enable, in the order given by §12 S12–S14.

---

# 6. Address conflicts and defects found in §1–§12

Building this index required computing every BAR-0 offset from the fixed map of §1 §4. That
exercise exposed the following disagreements between sections.

**Status: all eight have since been resolved and the affected sections corrected in place.**
Each subsection below records the defect and the resolution, so that a reader who has an
older copy of a section can tell which reading was wrong.

## 6.1 BAR `0x4000` — "legacy PCIe HIF alias" versus the fixed map

§2 §1.2 states that "the legacy `PCIE_HIF` block at BAR offset `0x4000` aliases the host
WFDMA0 CSR page; BAR `0x4108` is `HOST2MCU_SW_INT_SET` and BAR `0x41F0` is
`MCU2HOST_SW_INT_STA`", and §2 §8.1 repeats BAR `0x4108` as an alias of the host→MCU
doorbell.

This cannot hold together with the rest of the document:

* §1 §4 entry 5 maps BAR `0x004000` to chip **`0x5600_0000`** ("WFDMA reserved"), so BAR
  `0x4108` decodes to `0x5600_0108` — a WFDMA instance §1 and §3 both describe as unused.
* §4 §3.2 and §2 §8.1 itself both place `HOST2MCU_SW_INT_SET` at chip **`0x5400_0108`**,
  which is BAR **`0x002108`**.
* `MCU2HOST_SW_INT_STA` is at chip `0x7C02_41F0` = BAR **`0x0D41F0`** (§4 §1) — nowhere near
  BAR `0x41F0`.

**Resolved — the alias claim is withdrawn.** The bare offset `0x4108` does occur in the
generic code, but only as the **fallback taken when the part supplies no chip-specific
host→MCU software-interrupt hook**. MT7932's bus descriptor *does* supply that hook, so the
fallback is dead code here and the write goes to chip `0x5400_0108`. No `0x41F0` bare-offset
form exists at all. §2 §1.2 and §2 §8.1 have been corrected; this appendix indexes only the
`0x5400_0108` / `0x7C02_41F0` forms. `[C]`

## 6.2 PCIe-MAC debug mux — `0x7403_0164` / `0x7403_0168` / `0x7403_002C` roles

The two sections that describe the PCIe-MAC debug read-out assign different roles to the same
three registers:

| Register | §1 §7.1 | §2 §6.1 |
|---|---|---|
| `0x7403_0164` | "debug-probe **select** — write a probe selector" | "4 × 8-bit debug-**signal** select" (written the same ten values) |
| `0x7403_0168` | "debug-probe **data** — read-only" | debug **group/mode** select — **written** `0xCCCC_0100` / `0x9999_0100` |
| `0x7403_002C` | "debug latch/clear — written `0` between probe reads" | the **status read-back** after each selector write |

**Resolved in favour of §2.** The debug-mux descriptor table is a list of
{address, value} records in the order: write `0x7403_0168` = `0xCCCC_0100`, write
`0x7403_0164` = `0x4F4E_4D4C`, **read** `0x7403_002C`; then write `0x7403_0168` =
`0x9999_0100`, write `0x7403_0164` = `0x5756_5553`, read `0x7403_002C`; then eight further
`0x7403_0164` selector writes each followed by a read of `0x7403_002C`. `[C]` So `0x0168` is
the group/mode select, `0x0164` the signal select, and `0x002C` the read port — which is also
what the DMA-hang predicate of §2 §6.6 requires. §1 §7.1's three rows have been corrected.

## 6.3 WF_MIB block size — `0x0800` versus `0x2000`

* §1 §4 entry 25: `0x820E_D000 → 0x024800`, size **`0x0800`** (and entry 43 the same for
  band 1).
* §10 §B.1 and §B.9: the same entry, size **`0x2000`** ("8 KiB").

**Resolved in favour of §1.** Every MIB offset §10 §B.9 itself enumerates lies below `0x7F8`,
so the block as documented fits in `0x800`; and a `0x2000` window from BAR `0x024800` would
run into BAR `0x026000`, which §1 §4 assigns to WF_MUCOP. §10 §B.1 and §B.9 have been
corrected to `0x0800`. `[C]`

## 6.4 §1 §7.1's location of BAR `0x1818C`

§1 §7.1 lists BAR `0x1818C` in the PCIe-MAC diagnostic set and then notes it "is a BAR offset
inside the LMAC BN0 window, not the PCIe MAC block". BAR `0x010000`–`0x01FFFF` **is** the
PCIe-MAC window (§1 §4 entry 1, `0x7403_0000`, size `0x10000`); the LMAC BN0 windows start at
BAR `0x020000`. BAR `0x1818C` therefore decodes to chip `0x7403_818C` — exactly what §2 §6.1
lists in the same dump. The addresses in the two sections agree; only §1's parenthetical was
wrong, and it has been corrected. `[C]`

## 6.5 DMASHDL status registers overlap the group-quota array

§3 §8.2's register table lists, in adjacent rows, "`0x7C02_6020 + 4*n` (n = 0..15) — group *n*
quota" (i.e. `0x6020`–`0x605C`) and "`0x7C02_6024`.. (read-only status) — per-group page /
reserved / source counters", with the note that the exact status offsets are `[U]`. The two
claims occupy the same addresses. **Resolved:** the quota array is right (public gen4m
`MT_HIF_DMASHDL_GROUP0_CTRL` = DMASHDL base + `0x20`), and the read-only status registers are
not in it — public gen4m places them at base + `0x100` (`STATUS_RD`), base + `0x140`…`0x17C`
(`STATUS_RD_GP0..15`) and base + `0x180`… (`RD_GP_PKT_CNT_*`). §3 §8.2 has been corrected to
those offsets, marked `[L]` because they are public values not independently confirmed for
this part.

## 6.6 §10 §B.1's LMAC BN0 block labels are shifted by one entry

§10 §B.1 labels `0x820E_0000 → 0x02_0000` as "band 0 TRB" and `0x820E_1000 → 0x02_0400` as
just "band 0". §1 §4 entries 12 and 13 label them `WF_CFG` and `WF_TRB` respectively, which
also matches the public MT7921 map. The addresses and BAR offsets in both sections agree;
only the two names were displaced, and they have been corrected. `[C]`

## 6.7 One value, three different conditions for `0x6C00_0000` → `0x6000_0000`

The `HOST_INT_ENA` constant term drops the two FWDL/CMD TX-done bits under a condition each
section states differently:

* §1 §7.2 — "when the datapath is suspended";
* §2 §4.4 — "in the FLR/reset-pending mode";
* §3 §7.6 and §4 §2.2/§5 — in a "derive-from-ring / no-MMIO-read" operating mode.

**Resolved: none of the three was right.** The reduced constant is used while the driver is
still in its **pre-firmware-ready (initialisation) state**; the controlling flag is cleared at
the end of a successful adapter start. `[C]` In that state the host also substitutes a bounded
retry poll of the RX descriptor DONE bit for its normal service path. Note that the
"no-MMIO-read" framing was misleading in a second way: **neither** state reads `HOST_INT_STA`
over MMIO — in MSI mode the pending set comes from the vector that fired, otherwise from the
descriptors' DONE bits. §1 §7.2, §2 §4.4, §3 §7.6 and §4 §2.2/§5 have all been corrected.

## 6.8 Cross-checks that are now consistent

For completeness, the following previously-flagged items were re-checked while building this
index and are consistent across sections: the ROM-patch section count (§5 and §11 both say
one section, destination `0x0090_0000`, length `0x8280`); the RAM-image ECO code (`0x01`
zero-based ⇒ **E2** in both §5 and §11); the DMA-scheduler base (`0x7C02_6000` host view,
`0x5200_0000` MCU/AXI view — §3 §8.2 and §11 §4 now say so explicitly); the
firmware-ready checkpoint (§1, §5 §7.1.1 and §11 agree it is one checkpoint with two
device-ID-selected encodings); the interrupt bit map and the MSI ring-index bitmaps (§1 §1.4,
§3 §3 and §4 §2 agree, including that no data-TX-done bit is used); the host TX/RX ring
inventory (§3 and §4 §2.1 agree ring-for-ring); the RX buffer size as a host choice (§3 and
§8); the two TX completion identifiers (§7 §2 and §8 §6.2 use one vocabulary); the MSI vector
capability versus the single-vector mode (§1 §1.4 and §4 §4); and the chip-identity registers
`0x8002_1000` / `0x8002_1008` / `0x7001_0200` / `0x7001_0204` / `0x8800_0004` (§1 §6, §2 §2.3,
§5 §1.1 and §11 §1.2 agree on address and field positions).

---

# 7. Open questions / needs hardware tracing

Raised by this appendix specifically — the per-topic lists at the end of §1–§12 still apply.

1. **The six defects of §6.** Items 6.1 and 6.2 are the only ones where a driver could issue
   the wrong access; both need a trace. Items 6.3–6.6 are documentation errors with no
   behavioural consequence, and 6.7 needs the correct condition established.
2. **Access types marked `[L]` throughout §2 of this appendix.** The sections state what the
   host *does* to each register, not what the silicon permits. Every `RO`/`WO` derived from
   "only ever read" or "only ever written" — the whole PCIe-MAC diagnostic set, the CB-TOP
   debug CRs, the MIB and ETBF counters, the share-info mailbox and the WFSYS bus-status CRs —
   should be confirmed by attempting the opposite direction.
3. **`0x7C07_0060` acquire semantics.** The sections say "poll bit 0"; whether the read itself
   takes the semaphore (the usual conn-infra hardware-semaphore convention) or a separate
   write is implied is not established. `[U]`
4. **Reachability of the five calibration-integrity CRs** (`0x8300_7A14`, `0x8301_7A14`,
   `0x830A_D418/D424/D42C`). §9 §8.1 names them but no section says how the host reaches
   them. They are outside every fixed-map entry, so either a remap slot must be programmed to
   `0x8300`/`0x8301`/`0x830A` — the BAR offsets given in §2 assume slot 4 and are **derived,
   not read** — or they are read through the runtime register-access command `0xC0`. `[U]`
5. **DMASHDL status/counter offsets** (§6.5). Enumerate `0x7C02_6000`–`0x7C02_60FF` on live
   silicon and separate the quota array from the counters.
6. **`0x7403_1488` / `0x7403_148C`.** §2 §1.1 establishes that configuration dwords `0x488`
   and `0x48C` are aliased into MMIO at `0x7403_1000 + offset`, but no section reads them
   there; the existence of the two aliases is `[L]` by construction from the `0x484` case.
7. **The remap-array base and the eleven slots that are never written.** `0x7C00_E244` is
   `[L]` (anchored by three confirmed fields), and slots 0, 2, 3, 10–15 carry `[L]`/`[U]`
   selector values. Reading `0x7C00_E240 … 0x7C00_E264` once on live hardware would confirm
   the whole array and the composite-window question of §1's open item 4.
8. **Whether `0x7001_0208` decodes at all on this part.** It is the public gen4m `TOP_FVR`
   location; MT7932 uses `0x8800_0004` instead, and nothing establishes that the CB-TOP copy
   exists or is stale. `[U]`
9. **Band-1 register mirrors.** Every `0x820F_xxxx` row in §2 of this appendix is derived from
   the band-0 row plus the §1 §4 map; MT7932 is single-band-at-a-time, so no section observes
   a band-1 access. Whether band 1 is populated in this silicon at all is `[U]`.
