# Multi-Touch Processor (MTP) — Firmware ABI Specification
## MacBook Neo (Mac17,5 / board J700AP / SoC t8140)

**Status:** Draft specification for independent driver implementation.
**Platform baseline:** the factory-shipped system software of this machine as of 2026-06.
**Audience:** implementers writing a Linux driver from this document plus hardware tracing.

---

## 1. Scope and conventions

This document specifies the interface exposed by the Multi-Touch Processor (MTP), a coprocessor
that owns the built-in keyboard and trackpad of this machine and presents them to the host as
HID devices.

**In scope:** the wire protocol between host and MTP; the command surface; firmware provisioning;
the touch device's report and configuration surface; power management; reset and recovery.

**Out of scope:** anything already implemented by existing open-source support for this family of
coprocessors. Where this machine matches existing knowledge, the text says so and does not restate
it. See Appendix A.

### 1.1 Conventions

- All multi-byte fields are **little-endian** unless a field is explicitly marked big-endian.
- Byte offsets are relative to the start of the structure being described.
- "Host" is the application processor. "Device" is the MTP, or a logical interface behind it.
- Numeric identifiers (report identifiers, command values, state values) are given in hexadecimal.
  These are interface constants that an implementation must reproduce exactly.
- Confidence is annotated where it is not uniform:
  **[C]** established, **[P]** strongly supported but not settled, **[U]** uncertain.
  Unannotated statements are **[C]**.

**Report identifiers are scoped, not global.** A report identifier is meaningful only in the
combination *(interface number, report class, identifier)*. The same numeric value therefore recurs
with unrelated meanings across this document, and an implementation must not key on the identifier
alone. The collisions that occur on this machine are:

| Value | One meaning | Another meaning |
|---|---|---|
| `0x02` | Input report on the touch interface: compound wrapper (§8.2) | Feature report on the touch interface: host-association state (§8.7). Also a *status value* inside report `0x60` (§8.5) |
| `0x43`, `0x44` | Input reports on the touch interface: touch frames (§8.2) | Control-interface commands: power-cycle request and memory-capture request (§6) |
| `0x73` | Input report on the touch interface: touch frame (§8.2) | Feature report on the touch interface: residency descriptor (§9.9) |
| `0xA1` | Feature report on the touch interface: sensor region parameters (§8.7) | Control-interface configuration response (§6.2) |
| `0xF0` | Device-to-host descriptor delivery (§6.1) | Host-to-device descriptor request (§6.2) |

### 1.2 What this interface is not

Several structural properties are stated up front because they invalidate models carried over from
other devices in this class:

- The host-to-MTP link is a **hardware byte channel**, not a host-driven SPI bus. No host-driven
  SPI boot interface is selected on this machine. **[C]**
- The MTP is a **full coprocessor with its own operating system and address-translation unit**, not
  a masked-ROM device awaiting raw memory writes. There is no fixed-opcode handshake, chip-info,
  memory-write, memory-read or jump-to-entry packet protocol on the host-facing link. **[C]**
- There *is* an SPI link in the system, but it is **internal to the MTP** — the coprocessor's own
  downstream connection to the touch sensor front-end. It is not visible to, or driven by, the host.
  Its parameters are supplied to the coprocessor inside the provisioning image (§7.2). **[C]**
- Firmware is not pushed word-by-word by the host. The host supplies a **declarative provisioning
  script** which the coprocessor executes against the touch controller (§7). **[C]**

---

## 2. System topology

Four elements matter to a driver:

| Element | Role |
|---|---|
| Byte channel | Bidirectional hardware channel carrying host-to-device traffic and most device-to-host traffic. |
| Coprocessor mailbox | Separate control path used to boot, sleep, and supervise the MTP itself. |
| Address translation | Translation for all coprocessor memory access. Shared-memory buffers must be mapped through it. |
| Logical interfaces | The MTP multiplexes several HID devices over the byte channel, addressed by an interface number. |

A **second, device-to-host-only** data path exists alongside the byte channel: shared-memory rings
in DRAM carrying identical framing (§3.7). Both paths must be serviced.

The MTP therefore has **two distinct control surfaces**: the coprocessor mailbox (start it, stop it,
sleep it) and the in-band HID transport protocol carried over the byte channel (everything else).
These are independent; a driver needs both.

**Device population on this machine.** The coprocessor's shipped configuration declares **three**
interfaces: a **control interface** (number 0) carrying the management command set, a **touch
interface** (number 1), and a third interface (number 2) whose role is a coprocessor-level
management and telemetry channel rather than a user-input device. **[C]**

A **keyboard interface** is declared in platform data as a child of this transport, but does **not**
appear in the coprocessor's own interface list. Whether the keyboard is reached through this
transport or attaches elsewhere is **an open question** (§12.2). **[U]**

Unlike earlier machines in this family, there is **no separate sensor-microcontroller interface**;
the sensor front-end is driven by the MTP internally and is not addressable by the host. **[C]**

A transport-multiplexer node is declared in platform data. Nothing in the interface described here
consumes it, and its purpose is **undetermined**. One reading is that it mediates which processor
owns the touch stream while the host is asleep, which would make ownership handover a first-class
driver concern; another is that it is inert on this board. It should not be relied upon, and equally
should not be assumed inert. **[U]**

The multiplexing that demonstrably exists is the interface-number field in the frame header, plus an
interface *type* reported by the device during discovery (§4). **[C]**

---

## 3. Transport framing

### 3.1 Byte channel

The channel is a byte stream with a hardware-reported free-space count and a receive interrupt.
The channel hardware reports how many channels it implements; on this machine the MTP path uses
**channel number 0** and no second channel is in use. The channel's depth on this machine is
2 KiB. This layer — register access, interrupt handling and byte-level read/write — is already
understood and implemented elsewhere; see Appendix A.

Two facts matter above that layer:

- Free space must be checked before writing, and a writer must not begin a frame it cannot
  complete. Poll cadence is an implementation choice; a 1 ms poll interval with a multi-second
  ceiling is sufficient in practice. **[C]**
- A frame's total length is derivable from its own header: header length at byte 0, body length at
  bytes 2 and 3. Frames are **contiguous byte runs**; there is no fragmentation, no reassembly
  protocol and no escaping. The channel depth bounds a single read request, not a frame — a frame
  larger than the channel depth arrives in several reads. **[C]**

### 3.2 Frame layout

Every frame, in both directions, has the same shape:

| Region | Size | Notes |
|---|---|---|
| Transport header | 8 bytes | Always present. Declares its own length. |
| Application header | 8 bytes | Present when the body length is at least 8. |
| Payload | variable | HID report bytes. First byte is the report identifier. |
| Padding | 0–3 bytes | Rounds the body to a 4-byte boundary. |
| Checksum | 4 bytes | Covers the header and body. |

Derived quantities:

- Body length = (payload length + 8), rounded up to a multiple of 4.
- Total frame length = 8 + body length + 4.
- Minimum legal frame = 12 bytes (header plus checksum, no body). Used for status-only frames.

**Padding content is unspecified.** The rounding bytes are not required to be zero, but they **are**
covered by the checksum, so a receiver must checksum them as received and a transmitter must
checksum exactly the bytes it sends. Do not assume zero. **[C]**

The length field permits a theoretical maximum frame of 65 544 bytes, but a frame is also bounded by
the receiver's message-buffer capacity (§5.4), which is smaller. A frame that would exceed the
receiver's capacity is a fatal framing error.

There is no preamble, no sync word, no byte stuffing and no escaping.

### 3.3 Transport header

| Offset | Size | Field | Meaning |
|---|---|---|---|
| 0 | 1 | Header length | Always 8. A larger value is a protocol error. |
| 1 | 1 | Message type | `0x11` control, `0x12` input. No other value is valid on the wire. |
| 2 | 2 | Body length | Application header plus payload plus padding. Always a multiple of 4. |
| 4 | 1 | Transfer identifier | Per-interface sequence counter used to match responses. |
| 5 | 1 | Interface number | Selects the logical device. |
| 6 | 1 | Byte 6 | **Control frames only.** Populated by the device; the host does not write it. Purpose undetermined. **[U]** |
| 6 | 2 | Bytes 6-7 | **Input frames only.** A 16-bit field overlaying both bytes. Purpose undetermined. **[U]** |
| 7 | 1 | Status class | **Control frames only.** See below. |

Bytes 6 and 7 are **type-overloaded**: their interpretation depends on the message type in byte 1.
An implementation must branch on message type before reading them. On input frames byte 7 is
physically the high byte of the 2-byte data-rate field and carries no independent meaning.

The field *positions* are established; the *meaning* of the retry-count and data-rate fields is not.
A host implementation may safely write zero to byte 6 on the control frames it originates. **[U]**

**Status class (control frames, byte 7).** This is the transport's coarse error channel and it
selects where the real status lives:

| Value | Meaning |
|---|---|
| `0x00` | No transport-level error. **The transfer's status is the 4-byte field in the application header (§3.4).** |
| `0x80`–`0x85` | Six distinguished device-signalled error classes. The transfer failed. The distinction *between* these six values is not established. **[U]** |
| any other non-zero | Generic error. The transfer failed. |

A receiver that ignores byte 7 will misread failed transfers as successful, because a frame carrying
a non-zero class need not carry a meaningful application-header status. **[C]**

### 3.4 Application header

| Offset | Size | Field | Meaning |
|---|---|---|---|
| 0 | 1 | Report type and direction | Bitfield, see below. |
| 1 | 1 | Unused by the host | The host writes zero. What the device does with it is undetermined. **[U]** |
| 2 | 2 | Payload length | Length of the HID report that follows. |
| 4 | 4 | Status | Completion status for the transfer, supplied by the device. |

The report type byte encodes both the HID report class and the transfer direction:

- **Bits 7:6** — report class: `0` input, `1` output, `2` feature.
- **Bit 0** — direction: `1` = get (read from device), `0` = set (write to device).

Consequently `0x01`, `0x41` and `0x81` are get operations on input, output and feature reports
respectively; `0x40` and `0x80` are set operations; `0x00` marks an unsolicited input report
delivered by the device. Input reports may not be *set*.

**Status encoding.** The status field is a 32-bit value drawn from the host platform's standard
driver status space. **Zero means success; any non-zero value means the transfer failed.** The
non-zero values a driver will see include generic failure, overrun, bad argument and timeout, plus a
distinct value used for the byte-7 error classes above. An independent implementation should map
non-zero to a local error code rather than attempt to reproduce the value space; only the
zero/non-zero distinction is load-bearing. **[C]**

The payload begins immediately after this header and its **first byte is the HID report
identifier**.

### 3.5 Checksum

The checksum is the **one's complement of the 32-bit little-endian word sum** over the header and
body, transmitted immediately after the body. When the covered length is less than 4 bytes the value
is all-ones. It is **not** a CRC.

Omitting the complement produces a value the device rejects, and a rejected frame is fatal
(§3.6). This algorithm is already implemented in existing open-source support and matches it
exactly; see Appendix A.

### 3.6 Error handling and the absence of retransmission

There is **no acknowledgement, no negative acknowledgement, no retransmission and no resynchronisation
mechanism** in this protocol. A malformed frame — bad header length, declared length exceeding
buffer capacity, a boundary overrun, or a checksum mismatch — is a **fatal, unrecoverable condition**.
There is no defined path back to a good state.

**Implementation guidance.** A Linux driver must not model this as a lossy link. It should treat a
framing error as a reason to reset the coprocessor (§10) rather than attempt recovery in
place, and must be careful never to write a partial frame.

The sequence field is a per-interface 8-bit counter, matched on the control response only. At most
one control transfer per interface may be outstanding. The counter free-runs and wraps; it is not
reset across a device reset, and the device is expected to echo whatever it receives.

### 3.7 Shared-memory rings

A second data path carries **identical framing** through DRAM ring buffers mapped for coprocessor
access, rather than through the byte channel. Rings are registered during the startup handshake
(§6.3, command `0x91`).

**Ring layout.** Each ring is a fixed 8-byte header followed by the data area:

| Offset | Size | Field |
|---|---|---|
| 0 | 4 | Write index, advanced by the producer |
| 4 | 4 | Read index, advanced by the consumer |
| 8 | total size − 8 | Data area |

Usable capacity is the total registered size minus 8. Both indices are byte offsets into the data
area and advance modulo the capacity. **One slot is always left free** so that a full ring is
distinguishable from an empty one.

- Bytes available to the consumer = *write − read* when *write* is not less than *read*, otherwise
  *capacity + write − read*.
- Space available to the producer = *read − write − 1* when *write* is less than *read*, otherwise
  *read + capacity − write − 1*.
- An ordering barrier is required after copying data and before publishing the updated index.
- A reset zeroes both indices.

**Direction.** The rings created on this machine are **device-to-host only**. The configuration
declares one ring, 2 MiB of device-to-host capacity, and a host-to-device size of zero; no
host-to-device ring is created. A driver that builds a host-to-device ring will find nothing reads
it. **[C]**

**There is no doorbell or interrupt for the rings.** They are drained after **every** byte-channel
read completion, plus once at power-on. A driver that waits for a ring interrupt will never see ring
traffic, and a driver that does not drain them on resume will stall (§9.6). **[C]**

Oversized *host-to-device* transfers do **not** use these rings; they use a separately registered
one-shot buffer (§5.4).

---

## 4. Interface model and discovery

### 4.1 Addressing

Logical devices are addressed by the **interface number** in the transport header. The control
interface is a normal interface whose report identifiers form a private command set (§6).

Interface numbers run **0 through 31**; the maximum count is **32**. An identifier of 32 or above is
rejected as invalid. On this machine the **control interface is number 0** and the **touch interface
is number 1**. The numbering is nominally discovered rather than fixed, so an implementation should
read it from discovery rather than hard-coding it. **[C]**

Note: existing open-source support for earlier machines assumes a maximum of 16 interfaces. That
limit is too low for this device.

### 4.2 Interface descriptor

The host retrieves a descriptor blob per interface with a two-step feature exchange:

1. **Set** feature report `0xF0` with payload `{0xF0, interface identifier}`.
2. **Get** feature report `0xF0`. The response may be large; the receive buffer must accommodate
   up to 64 KiB.

The returned blob is a 22-byte header followed by a sequence of type-length-value sections.

**Blob header (22 bytes):**

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | Report identifier (`0xF0`); not otherwise used |
| 1 | 2 | Format version; must be 1 or less, otherwise the descriptor must be refused |
| 3 | 1 | Interface identifier |
| 4 | 16 | Interface name; the last byte is forced to a terminator, so at most 15 characters are usable |
| 20 | 2 | Continuation. **Zero means the descriptor is complete**; non-zero means more is to follow |

The 2-byte version field at offset 1 is unaligned. Whether it is genuinely a single 16-bit field or
two single-byte fields is not settled; a live descriptor capture would resolve it. **[P]**

The continuation field at offset 20 is the only signal that a descriptor has been fully delivered. A
driver must check it before treating the descriptor as complete.

**Section format:** each section is a 2-byte type, a 2-byte length, then that many bytes. **A section
length must be less than 2048.** An unknown section type must be **skipped, not treated as fatal**
(§6.5).

| Type | Content |
|---|---|
| 0 | HID report descriptor, verbatim |
| 1 | Configuration entries (below) |
| 2 | Transport information (below) |
| 3 | Serial number |
| 4 | Ready status: 2 bytes, `0` = ready, `1` = inactive (still requires provisioning) |
| 5 | List of available interfaces: a raw byte array of interface identifiers |
| 6 | Location identifier, 4 bytes |
| 7 | Product name string, terminated |

**Configuration entries (section type 1).** An array of fixed **36-byte** entries. Each entry begins
with a 2-byte type identifier and carries a 4-byte value at offset 32. Type identifiers are
`0` = calibration, `1` = external resources, `2` = shared resources. The calibration entry supplies
the **report identifier used for the configuration and calibration push** described in §6.2 — that
command has no fixed identifier and cannot be constructed without this entry. **[C]**

**Transport information (section type 2).** A 4-byte version that must be zero, a 1-byte interface
type, and a 1-byte flags field:

- Interface type: `0` = HID device, `2` = multiplexer.
- Flags **bit 0 = reject control requests**: when set, the host **must not** issue get or set report
  operations on that interface. This is the mechanism by which the device declares an interface that
  exists but is not the host's to drive. **[C]**

The HID report descriptor is therefore **fetched from the device**, not statically known. A driver
must parse it at runtime and must not assume a fixed descriptor.

Interfaces whose type is *multiplexer* are addressable over the transport but are **not** HID
devices and must not be published as such. **[C]**

**Other per-interface configuration supplied at discovery.** The descriptor carries, in addition to
the above, a set of per-interface policy and sizing values that exist only at runtime and cannot be
obtained from platform data. A driver depends on several of them and must read them rather than
assume defaults:

- **Maximum input, output and control report lengths.** These bound every transfer on that
  interface independently of the frame and message capacity (§5.4).
- **Get and set report allowlists.** When present, a report identifier not on the corresponding list
  must be refused by the host before it reaches the wire.
- Flags marking the interface unavailable, either administratively or because it is in use.
- The **power method** to use for this interface (§9.2), and which power-transition and reset events
  the device expects to be notified about.
- **Minimum off-time** enforced between powering the interface off and back on (§9.2).
- **Power, sleep and reset sequences** — platform operations to be run locally around power
  transitions (§9.8).
- A ceiling on **consecutive failed bring-up attempts** (§7.6).
- **Memory-capture policy** around reset, and a capture timeout (§6.2, `0x44`).
- A **periodic counter report** identifier, and whether it continues during sleep (§9.9).
- An **initial report** — a get whose response the host compares against an expected value each time
  the interface reports ready. A mismatch indicates the interface did not come up correctly.
- **Country code**, and whether registration completes asynchronously.

None of these values can be stated here; they arrive from the device. Capturing one descriptor
exchange yields all of them at once, which is why §12.2 places that capture first.

### 4.3 Discovery flow

Interfaces become known in one of two ways: the device pushes a descriptor unsolicited (§6.1),
or the host enumerates and fetches descriptors explicitly.

The explicit path is:

- Fetch the control interface's own descriptor (`0xF0` set then get).
- Read the available-interfaces list (section type 5) from it.
- For each listed identifier other than the control interface, fetch that interface's descriptor.
- **Signal completion per interface with feature set `0xB4`** (2 bytes: `{0xB4, interface
  identifier}`). This terminates per-interface enumeration and must not be omitted.

An interface is only usable once the device has reported it **ready** (§6.1, `0xF1`); a driver must
wait for that signal before publishing the device or issuing traffic to it. An interface whose ready
status is *inactive* still requires provisioning (§7).

### 4.4 Retrievable identity

The following identity comes **from the device**, per interface: interface name, serial number,
product name string, location identifier, country code, and the HID report descriptor.

The following are **not** device-supplied and must be provided by the host from platform data or
chosen by the implementation: vendor identifier, product identifier, version number and
manufacturer string. A driver that expects to read these off the wire will not find them. **[C]**

On this machine the transport identifies itself by a short transport-type string denoting the byte-channel FIFO.

---

## 5. Report request and response

### 5.1 Get

A get request carries a **one-byte payload**: the report identifier being requested. No length is
sent; the device decides how much to return.

The response payload's first byte is the report identifier. This is symmetric with the set
direction, but it has not been confirmed against a live device and is cheap to settle by tracing
(§12.2). **[P]**

### 5.2 Set

A set request carries the **entire report including its leading report identifier byte**. The
report identifier appears both in the report-type encoding path and as the first payload byte;
these must agree.

### 5.3 Matching, ordering and timeouts

Responses are matched on the **transfer identifier together with the interface number**. Only one
control transfer per interface may be outstanding; a driver must serialise per-interface requests.

The request timeout is **5 seconds**. There is **no retransmission** — a timeout is a hard failure,
not a prompt to retry. A timeout should be treated as a device fault and escalated to a reset scope
(§10).

Requests may be cancelled by the host; a cancelled request completes with a cancellation status.

### 5.4 Length limits and the oversized path

Three separate limits apply to every transfer, and all three must be respected:

1. **Per-interface maximum report length**, supplied by the interface descriptor (§4.2), separately
   for input, output and control (feature) reports. An over-length set must be refused by the host
   before transmission. **[C]**
2. **Message-buffer capacity.** On this machine the shipped configuration sets both the read and the
   write message capacity to **16 KiB**. A frame that would exceed the receiver's capacity is a
   fatal framing error (§3.6). Conflicting figures exist for the default in the
   absence of configuration — 4 KiB, and a much larger value — so an implementation should use the
   configured value and never the default. **[P]**
3. **The frame length field**, which tops out at 65 544 total bytes (§3.2).

A transfer that will not fit is handled by registering a shared-memory buffer and referencing it,
rather than by fragmenting the frame: command `0x95` with **use-type 3** (§6.2). There is no
fragmentation at the framing layer.

The oversized-transfer buffer is 64 KiB and may be cached and reused across transfers with an
expiry of the order of 30 seconds. **Shared-memory registrations do not survive a reset** — the
device's view of them is lost and they must be re-registered (§10.3).

---

## 6. Control interface command set

Byte 0 of every control payload repeats the report identifier. Byte 1, where present, is a
**per-message version byte** whose expected value is `1` for most commands; for the power and reset
commands it also serves to select the operation. There is no protocol-wide version handshake — see
§6.5.

### 6.1 Device to host

Delivered as input reports on the control interface.

| ID | Name (functional) | Length | Payload |
|---|---|---|---|
| `0xA0` | Configuration request | 20 (see note) | Byte 1 interface identifier; bytes 2–3 resource; bytes 4–7 action; bytes 8–11 reason; bytes 12–19 timestamp. Fields are unaligned. The response is not required to be immediate. |
| `0xA2` | Reset request | ≥2 | Byte 1 interface identifier. The device is asking the host to reset it. |
| `0xA3` | Memory-capture request | ≥3 | A version value of `1` and the interface identifier. Which of bytes 1 and 2 carries which is not settled. **[U]** |
| `0xA4` | Memory-capture response | 8 exactly | Byte 1 version, byte 2 interface identifier, byte 3 capture identifier, bytes 4–7 status. Completes a host-issued `0x44`. |
| `0xB5` | Host-assisted operation request | ≥28 plus arguments | See §6.4. |
| `0xF0` | Descriptor delivery | ≥22 | Interface descriptor (§4.2); feeds discovery. |
| `0xF1` | Ready | 4 | Byte 1 interface identifier, bytes 2–3 status: `0` = ready, `1` = inactive (still requires provisioning). |
| `0xF2` | Wake | ≥10 | Byte 1 interface identifier, bytes 2–9 timestamp. Identifies the wake source. |

**`0xA0` length.** This is unresolved: the request is variously bounded at exactly 20 bytes, and at
an extended form of 24 bytes or more, carrying a 4-byte byte-count at offset 20 followed by that
many bytes. An implementation should accept both and be prepared to ignore the trailing region.
**[U]**

**Timestamps.** The 8-byte timestamps in `0xF2`, `0xA0` and `0xB5` are device-supplied and are
echoed back verbatim where a response carries them. Their tick rate and epoch are **not established**
and no conversion is defined by the interface. **[U]**

Four further device-to-host reports are handled at the transport level rather than as
control-interface traffic:

| ID | Name (functional) | Length | Meaning |
|---|---|---|---|
| `0x43` | Power-cycle request | ≥3 | Byte 2 is the interface identifier. Device asks the host to power-cycle an interface (§9.4). |
| `0x96` | Release shared memory | ≥16 | Byte 1 use-type, byte 2 version, byte 3 interface identifier, bytes 4–11 device address, bytes 12–15 size. The host matches the address and size pair to identify which registered buffer to free. Use-type 3 does not occur here. |
| `0x97` | Memory-capture report | ≥15 | Carries a version, and the address and size of a capture the device has produced in a previously registered buffer. Field offsets are not established. **[U]** |
| `0x9A` | Provisioning message, other family | ≥9 | Carries a version value of `1`. Belongs to a **different provisioning family that this machine does not use** (see `0x99` in §6.2). It is listed only because the transport recognises the identifier; it is **not** this machine's provisioning notification. |

### 6.2 Host to device

All are **feature** reports on the control interface.

| ID | Length | Payload | Function |
|---|---|---|---|
| `0x40` | 4 | `{0x40, 0x01, interface identifier, power state}` | Set interface power, immediate |
| `0x40` | 9 | `{0x40, 0x02, interface identifier, power state, phase, 4-byte status}` | Power transition notify; phase `0` = will change, `1` = has changed |
| `0x41` | 4 | Set `{0x41, 0x01, interface identifier, 0}`, then get, which returns `{0x41, 0x01, interface identifier, power state}` | Get interface power |
| `0x42` | 3 | `{0x42, 0x01, interface identifier}` | **Reset interface** |
| `0x44` | 18 | See below | Request a memory capture from the device |
| `0x91` | 15 | `{0x91, 0, 0, 8-byte device address, 4-byte total size}` | Register a shared-memory ring |
| `0x95` | 16 | `{0x95, use-type, 0, interface identifier, 8-byte device address, 4-byte size}` | Register a shared-memory buffer. **Use-types: `2` = provisioning payload, `3` = oversized set-report payload, `4` = memory-capture buffer.** Values `0` and `1` are unassigned as far as this interface is concerned. **[U]** |
| `0x99` | — | — | Provisioning table-of-contents push for the **other** provisioning family. Not used on this machine; listed so the identifier is not mistaken for something else. |
| `0xA1` | 25, or 4 plus payload | See below | Configuration response, replying to `0xA0` |
| `0xB4` | 2 | `{0xB4, interface identifier}` | Signal that descriptor retrieval for that interface is complete |
| `0xB6` | 40 plus return data | See §6.4 | Report completion of a host-assisted operation |
| `0xC1` | 2 | `{0xC1, state}`; state `1` = system asleep, `2` = system awake | Announce host power state |
| `0xF0` | 2 | `{0xF0, interface identifier}` | Request interface descriptor (§4.2) |
| *(from descriptor)* | ≤ 64 KiB | `{report identifier, interface identifier, 2-byte tag, blob}` | **Configuration and calibration push.** The report identifier is not fixed: it is supplied by the interface descriptor's calibration configuration entry (§4.2). An over-length blob is rejected. Required after a reset (§10.3). |

Addresses passed in `0x91`, `0x95` and `0x44` are **translated device addresses**, not physical
addresses. The buffer must be mapped for coprocessor access before the command is sent. Note the
fields are **unaligned**; an implementation must not assume natural alignment.

**Memory capture request (`0x44`), 18 bytes:**

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `0x44` |
| 1 | 1 | Version / sub-opcode, value `1` |
| 2 | 1 | Interface identifier |
| 3 | 1 | Capture identifier, matched by the `0xA4` response |
| 4 | 8 | Device address of the capture buffer |
| 12 | 4 | Capture buffer size |
| 16 | 1 | Capture level |
| 17 | 1 | Build-variant flag |

The buffer must first be registered with `0x95` use-type 4. The transaction completes with `0xA4`
carrying the matching capture identifier; a response bearing a stale identifier must be discarded.
The timeout is the per-interface capture timeout from the descriptor (§4.2).

**Configuration response (`0xA1`) — two incompatible descriptions exist and this is unresolved.**

- **Reading A (the one to implement).** A **fixed 25-byte** report: `{identifier, 4-byte result,
  20-byte echo of the originating request}`, sent **only when the originating request's action value
  is 3, 4, 5, 7 or 9**. For every other action the host sends nothing.
- **Reading B.** A variable report `{identifier, interface identifier, 2-byte resource, payload}`,
  4 bytes plus payload, with the payload capped just below 64 KiB, sent in reply to any request.

These cannot both be right. Reading A is the form actually sent in reply to `0xA0`; the form in
reading B is not exercised on that path. **Implement reading A**, and in particular **do not
answer requests whose action is outside that set** — answering a request the device is not waiting
on is worse than not answering one it is. Settle this by tracing (§12.2). **[U]**

The report identifier used for the configuration response defaults to `0xA1` but is **overridable by
configuration**; a driver should not hard-code it if it can read the override. Action values 6
through 9 form an "external resource" family; the meaning of individual action and resource values
is not established. **[U]**

### 6.3 Startup handshake

The device signals liveness simply by transmitting. Before ordinary traffic is valid, two
requirements must be met, in this order:

- The host must announce system awake — feature set `0xC1` with state `2`.
- Every shared-memory ring the host intends to use must be registered with feature set `0x91`.

Provisioning of any interface that requires firmware follows registration, and interface traffic is
valid only once that interface reports ready (§6.1).

A settling delay of **100 ms**, measured from the device's first transmission, applies before the
first of these. **[C]**

### 6.4 Host-executed platform operations

The firmware cannot reach certain platform resources itself and asks the host to act for it. Two
mechanisms exist: the configuration request/response pair (`0xA0` / `0xA1`, §6.1 and §6.2) and the
host-assisted operation pair (`0xB5` / `0xB6`).

**Request (`0xB5`), at least 28 bytes plus an argument region:**

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `0xB5` |
| 1 | 1 | Version |
| 2 | 1 | Interface identifier |
| 4 | 8 | **Operation identifier**, an 8-character name |
| 12 | 4 | Reason |
| 16 | 8 | Timestamp (see §6.1) |
| 24 | 4 | Size of the argument region that follows |
| 28 | variable | Arguments |

The 8-character operation identifier is the field that says *which* platform operation to perform.
The host resolves it against the machine's platform description, which names the available
operations and their providers. Without this field the request cannot be serviced, so a driver that
cannot resolve the name must fail the request explicitly rather than silently ignore it.

**Completion (`0xB6`), 40 bytes plus return data:**

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `0xB6` |
| 1 | 1 | Version, value `1` |
| 2 | 1 | Interface identifier |
| 3 | 1 | Zero |
| 4 | 12 | First return descriptor: an 8-byte field followed by a 4-byte length |
| 16 | 12 | Second return descriptor, same shape |
| 28 | 12 | Third return descriptor, same shape |
| 40 | variable | Up to three return blobs, in order, of the declared lengths |

The total length is 40 plus the sum of the three declared lengths. The meaning of the 8-byte field
in each descriptor is not established; a host that returns no data should declare three zero lengths.
**[P]**

On this machine, the touch device's power-sequencing and analog-front-end reset controls are
**not host GPIOs** — they are routed through the system power-management controller. A driver must
be able to service these requests to bring the touch device up. **[C]**

This is a substantive difference from earlier machines in this family, where the equivalent
operations were direct host GPIO manipulations.

### 6.5 Versioning and forward compatibility

**There is no protocol-wide version handshake.** Versioning is per message, in the byte immediately
after the report identifier, and independently in several descriptor structures. The values expected
on this machine:

| Structure | Version field | Expected |
|---|---|---|
| Frame | Header length, byte 0 of the transport header | 8 or less |
| Descriptor blob | 2 bytes at offset 1 | 1 or less |
| Descriptor transport section | 4 bytes at offset 0 | 0 |
| Descriptor section type | 2-byte type field | 0 through 7 |
| Descriptor ready status | 2-byte status | 0 or 1 |
| Configuration entry type | 2-byte type identifier | 0, 1 or 2 |
| Interface type | 1 byte | 0 or 2 |
| Shared memory (`0x95` / `0x96`) | byte 2 | 1 |
| Memory-capture report (`0x97`) | version field | 1 |
| Memory-capture request (`0xA3`) | version field | 1 |
| Memory-capture response (`0xA4`) | byte 1 | must match the host's expectation |
| Power (`0x41`), reset (`0x42`), capture (`0x44`), completion (`0xB6`) | byte 1 | 1 |

**Recommended handling of an unexpected value.** These rules keep a driver working against a newer
device rather than failing opaquely:

- A header length greater than 8 means a newer framing; stop and do not guess.
- A descriptor version above 1 means a newer descriptor format; refuse to parse it.
- A non-zero transport-section version means a newer section format; do not create a device for that
  interface.
- An **unknown section type or configuration entry type must be skipped and parsing continued**, not
  treated as fatal.
- An unknown interface type means: do not publish a HID device for that interface.
- Any other structure whose version byte exceeds the known value should fail that one transaction
  only.

**Capability discovery is entirely descriptor-driven.** Which sections a device emits, and the
available-interfaces list, define what exists. There is no feature bitmap anywhere in this
interface.

---

## 7. Firmware provisioning

### 7.1 Division of responsibility

Two separate firmware questions exist and must not be confused.

**The MTP's own program.** Platform data marks the coprocessor as not requiring a host-supplied
firmware service, and the coprocessor node carries no firmware name and no memory region for an
image. On that basis nothing in this interface pushes an operating system to the MTP. **However, a
program image for the coprocessor domain is shipped with the platform** (§7.4), and how that image
reaches the coprocessor is **not established** — whether an earlier boot stage loads it, or whether
some other path hands it over. **This is unresolved and it gates whether a Linux driver can boot
the MTP at all** (§12.2). A driver author must settle it before assuming there is nothing to load.
**[U]**

**The touch controller's program** is host-supplied, but not as a raw binary transfer. The host
hands the MTP a **declarative provisioning script**; the MTP interprets it and drives the touch
controller over its internal link. The host never addresses the touch controller directly. **[C]**

The keyboard path **requires no firmware**: platform data for this machine contains no input-device
firmware component for it, and it is marked as needing no provisioning. **[C]** Whether the keyboard
enumerates as an interface behind this transport at all is a separate and open question (§2, §12.2);
"usable immediately" holds only if it does. **[U]**

The touch interface **requires provisioning on every start**. This matches the behaviour already
known for earlier machines in this family. **[C]**

### 7.2 Provisioning script model

The script is a **CBOR-encoded array of instruction records**. Each record is a map whose `Type`
key selects the operation. The type values are wire constants and must be reproduced exactly,
including the terminating NUL that the encoding requires (§7.3).

**Operations:**

| `Type` value | Purpose |
|---|---|
| `Binary` | Deliver a payload to a target address. Keys: `Address`, `Description`, `MaxSize` (required); `Payload`, `Tag` (optional). The payload length must not exceed `MaxSize`. |
| `Config` | Carries the configuration record for the target (below). At most one per script. Key: `Config`. |
| `Property` | Late-bound payload sourced from platform storage rather than carried inline. |
| `ReadModifyWrite` | Read a 32-bit location, clear the bits in `Mask`, set `Value`, write it back. Keys: `Address`, `Mask`, `Value`; `SkipCheck` suppresses read-back verification. |
| `Poll` | Read `Address`, apply `Mask`, wait until it equals `Value`. Keys: `TimeoutMs`, `DelayMs`. |
| `ReadProperty` | Read `Size` bytes from `Address` into a named property. |
| `RequestCalibration` | Ask the controller to produce calibration data. `DelayMs` optional. |
| `Metadata` | Non-executable descriptive content. |

**Conditional execution.** A record may carry a `Requirements` map whose single key is a predicate
operator, allowing one image to serve several hardware variants. The operators are `Not`, `And`,
`Or`, `VarEqual`, `VarLessThan`, `VarByteArrayEqual`, `PropertyEqual` and
`BootloaderPropertyEqual`.

**Optional per-record integrity check.** A record may carry a `Checksum` or `Validation` map with
`HeaderSize`, `PayloadSize` and `FooterSize`. Each of the three may be a plain integer or a
`{Offset, Size}` reference (size at most 4) read out of the payload itself. The check is an
**8-bit two's-complement additive sum** over the payload bytes from `HeaderSize` for
`PayloadSize + FooterSize` bytes, which must total zero when taken as a signed 8-bit value. A
related range check supports `Value`, `MinValue`, `MaxValue`, `BitShift` and `Mask` applied at a
given `Offset` and `Size` (at most 8). **Neither appears in this machine's touch script**, but an
implementation that re-encodes a script must carry them through. **[C]**

**Script shape for this machine.** The touch provisioning script is an array of thirteen records:
six read-modify-write pokes, two binary payload segments at distinct target addresses, two
calibration blobs (distinguished by their `Tag` values), a calibration request, a final
read-modify-write carrying `SkipCheck`, and a configuration record. **[C]** The records are executed
in array order and the final read-modify-write is what starts execution; both of those readings are
strongly supported but not directly established, because the interpreter runs on the coprocessor.
**[P]**

**Configuration record contents for this machine.** The `Config` record delivered to the touch
controller carries, among others:

| Key | Value on this machine | Significance |
|---|---|---|
| `Always Powered` | true | The controller is not expected to be powered down between uses. |
| `Protocol` | a short generation string | Names the touch controller generation. Supplied by platform data; a driver never constructs it. |
| `Continue On Failure` | true | **A failed touch provisioning must not tear down the transport.** This is firmware-visible policy an independent implementation should match. |
| `Boot Timeout` → `Normal Boot Ms` | 500 | The provisioning deadline for the touch controller. |
| `Bootloader SPI Config` | 12 MHz, phase 0, polarity 0, chip-select preamble 200, postamble 1000 | Parameters for the **coprocessor's own downstream link**, not the host's (§1.2). |
| `SPI Config` | 16 MHz, phase 0, polarity 0, chip-select preamble 100, postamble 100 | As above, for normal operation. |
| `Interface Config` | one entry for the touch interface | Declares the interface number, name and type, and the periodic counter report of §9.9. |

No `Bootload Timeout Ms` key is present in this machine's touch script, so completion is reported
asynchronously by the ready report rather than awaited synchronously (§7.5).

### 7.3 CBOR profile

The encoding is standard CBOR with important restrictions and one extension. An implementation must
match these exactly:

- Integers, byte strings, text strings, arrays, maps and booleans are used. The decoder accepts
  negative integers, but the encoder never emits them.
- **Text strings include their terminating NUL in the encoded length.** A conventional CBOR encoder
  will produce strings one byte short. This is the single most likely interoperability defect.
- Map keys are always text strings.
- Indefinite-length items, floating-point values and null are **rejected** by the decoder.
- **Byte strings are aligned so that their first payload byte falls on a 4-byte boundary.** Filler
  bytes with the value `0xD3` (a CBOR tag) are emitted **before the byte-string header**, not after
  the payload. The count of filler bytes is
  `(alignment − 1) AND (−current_offset − encoded_length_header_size)` with an alignment of 4 — that
  is, enough filler that the current offset plus the filler plus the encoded length header lands the
  payload on the boundary. A reader skips any run of `0xD3` bytes before reading an item; a writer
  must emit them ahead of the header, and must include them when computing the encoded size.
  Padding *after* the payload — the natural misreading — produces a stream the device cannot
  consume and a wrong encoded size.

### 7.4 Container format

The provisioning image is delivered inside the platform's signed image container. Two separate
images exist on this machine: one for the transport/coprocessor domain and one for the touch device.

**Outer container.** A DER sequence of four elements: a fixed 4-character type string, a
4-character content tag, a version string, and an octet string holding the payload. On this machine
the payloads are uncompressed, unencrypted and carry no embedded manifest. **The content tag in the
container is the byte-reversed form of the tag in platform data** — a driver that looks up an image
by the platform-data tag must reverse it before matching. **[C]**

**Sub-file table.** The payload may itself be a table of tagged sub-files. It is little-endian:

| Offset | Size | Field |
|---|---|---|
| 0 | 4 | Zero |
| 4 | 4 | All-ones |
| 8 | 24 | Zero |
| 32 | 8 | Magic, an 8-character ASCII signature |
| 40 | 4 | Entry count |
| 44 | 4 | Reserved |
| 48 | 16 × entry count | Entries |

Each entry is a 4-character tag, a 4-byte offset from the start of the table, a 4-byte size and a
4-byte reserved field. **If the magic does not match, the payload is used verbatim.** Exactly one
sub-file tag carries the driver configuration; the rest are ignored by the host.

On this machine the transport image contains such a table with two entries — a program image for
the coprocessor domain, and the configuration record — while the touch image has no table and is
used verbatim.

The literal magic signature and the tag values are readable directly from a shipped image and are
not reproduced here.

**Inner record.** The consumed sub-file is a **serialised structured record** keyed by a
configuration name: either a table of instruction records, or a single configuration record which is
treated as a one-element script. Selection is by the four-character tag drawn from platform data,
then by the configuration name, also drawn from platform data.

The serialised form is **not standard**: it is a vendor dialect using identity and back-reference
attributes and explicit integer widths, and a conventional parser for that format will not read it.
This is an interoperability hazard of the same class as the string-length rule in §7.3. An
implementation must either convert the file ahead of time or implement the dialect. **[C]**

The transport image's configuration record supplies the message capacities, the ring configuration
and the statically-declared interface list quoted elsewhere in this document.

### 7.5 Handoff and completion

Provisioning is **not chunked**. The host places the entire script in one buffer mapped for
coprocessor access. There is no per-chunk addressing, no read-back-and-verify command, and no
host-written entry address.

Two host actions are required, and **omitting the second is the classic failure mode** — the device
never reports ready and the driver waits forever:

- Register the buffer with `0x95` using **use-type 2**.
- **Power-cycle the interface** — an off transition followed by an on transition (§9.2). This is
  what makes the coprocessor re-run its provisioning against the newly registered script.

**The ordering of those two actions is not settled**: registration-then-power-cycle and
power-cycle-then-registration are both consistent with what is known. That the
power transitions are off-then-on is itself strongly supported rather than established. An
implementation should be prepared to try both orders. **[U]**

Completion is reported asynchronously by the device with a **ready** report (`0xF1`), whose status
distinguishes *ready* from *still requiring provisioning*. A ready report with status *inactive*
means the interface is waiting for provisioning and the sequence above should be repeated.

### 7.6 Failure handling

On failure the device produces a memory capture into a second registered buffer (`0x95` use-type 4),
encoded in the same CBOR form. The host is expected to have registered that buffer **before**
starting provisioning.

Note that the generic memory-capture flow — the pre-reset and post-reset capture power states, and
the `0x96`/`0x97` pair — is **not exercised for the transport's own domain on this machine**; crash
capture there uses the coprocessor's own facilities. The CBOR capture described here belongs to the
touch interface. **[C]**

A device that fails to provision will keep asking to be reset. **An implementation must bound
consecutive attempts per interface and stop honouring device-initiated reset and capture requests
once the bound is reached**, or a failing device can hold the system in a reset loop indefinitely.
The bound itself is supplied per interface by the descriptor (§4.2).

Because this machine's touch configuration sets *continue on failure* (§7.2), a failed touch
provisioning must not be escalated to a transport teardown.

### 7.7 Re-provisioning

The touch provisioning script is **re-pushed after every handshake** — that is, after every
transport restart, resume from a deep sleep state, or reset. It is not persistent in the touch
controller. A driver must retain the image in memory for the lifetime of the device. **[C]**

This is established from the mechanism that performs it rather than from a live device, and one
description of the interface leaves it open. The statement above is the one to implement. **[C]**

---

## 8. Touch device ABI

### 8.1 Device class

This trackpad is a **mechanical-switch device with no force sensing and no haptic actuator**.
Click is a physical switch closure reported as a bit in the touch report header. There is no
strain-gauge force channel, no click-suppression control (the host click-control feature report
`0x21` used by actuator-bearing devices is absent here) and no waveform output path. Any code
ported from a force-sensing Apple trackpad must have those paths removed rather than stubbed.
**[P]**

The touch interface presents its top-level HID usage as a **pointer device**, not a digitizer.
Multi-touch data rides on vendor-defined report identifiers inside that interface. A driver must
not expect a standard HID digitizer collection. **[P]**

**Provenance of the contact-format material.** The per-contact record formats (§8.4), the coordinate
and unit model (§8.10), the contact identity and lifecycle model (§8.11) and the amplitude, density
and force semantics (§8.12) specify the **record formats themselves** — what a decoder must accept
for a given report identifier — rather than the coprocessor interface that the rest of this document
covers. A reader should keep that division in mind. Those sections are firm about **format given
identifier**, and correspondingly precise. They do **not** establish **which** report identifier this
machine's trackpad emits; that question is not answerable from format definitions and needs a
capture (§8.4.1, §8.4.14). **[C]**

### 8.2 Accepted input report identifiers

The following input report identifiers carry touch data or touch-adjacent status:

| ID | Meaning |
|---|---|
| `0x28`, `0x29` | Touch frame, legacy generations |
| `0x31` | Touch frame with **compact header** (§8.3) |
| `0x43`, `0x44`, `0x45` | Touch frame |
| `0x50` | Critical error frame (§8.6) |
| `0x60` | Status frame (§8.5) |
| `0x73`, `0x74`, `0x75`, `0x76` | Touch frame; `0x75` uses the **extended header** (§8.3) |

Report `0x02` is a **compound wrapper**: an 8-byte pointer-style prefix followed, at offset 8, by a
nested report whose first byte is the nested report identifier. The nested report must be
re-dispatched.

**Which identifier this trackpad actually emits is not established.** The plausible candidates are
`0x31` and `0x75`, or `0x02` wrapping one of them. This is a primary target for hardware tracing.
See §8.4.14, which states the case for each candidate and how one captured frame settles it.
**[U]**

Identifiers outside the set above are **not** accepted by this interface, which rules out several of
the contact formats specified in §8.4 for this machine. The full format set is nevertheless
specified there, because the format is chosen by the firmware rather than by the hardware
generation and a firmware revision may begin using a different one.

### 8.3 Touch report headers

Two header formats are described here, selected by report identifier. Every other touch-frame
identifier carries its own header; the full set is tabulated in §8.4.2.

**Compact header — 4 bytes, report `0x31`:**

| Byte | Bits | Field |
|---|---|---|
| 0 | 7:0 | Report identifier |
| 1 | 0 | **Button / click state** |
| 1 | 1 | Flag. Two readings exist and are not reconciled: a surface-orientation indication, or an unrelated spare. **[P]** |
| 1 | 2 | Flag: level-triggered state whose edges are significant to the host. One reading is "report is not valid as relative pointer motion"; the flag's meaning is not settled. **[U]** |
| 1 | 7:3 | Timestamp, bits 4:0 |
| 2 | 7:0 | Timestamp, bits 12:5 |
| 3 | 7:0 | Timestamp, bits 20:13 |

The timestamp is a **21-bit value spanning bytes 1 through 3**. **Its tick is one millisecond**
(§8.10.6). The counter free-runs and wraps roughly every 35 minutes; the wrap-extension rule is in
§8.10.6. **[C]**

The compact header is exactly 4 bytes, and **per-contact data begins at byte 4**. **[C]**

**Extended header — 32 bytes, report `0x75`:**

The full field layout of this header is given in §8.4.11. The two fields previously singled out are:

| Byte | Size | Field |
|---|---|---|
| 12 | 2 | Bit 0: flag, same role as the compact header's byte 1 bit 2. Bit 1: contact data present. Bit 2: image data present |
| 23 | 1 | **Button / click state**, full byte |

The remaining bytes are **no longer undetermined** — see §8.4.11, which specifies all 32. Note in
particular that **the header declares its own length in byte 2** (normally 32) and that the contact
array does **not** necessarily begin at byte 32: it begins at the declared header length plus the
length of an auxiliary block declared in bytes 14 and 15. **[C]**

### 8.4 Per-contact records

This section specifies the contact array for every touch-frame format this interface can carry.

This section specifies the **record formats themselves** — what a decoder must accept for a given
report identifier — rather than the coprocessor interface covered elsewhere (§8.1). It does **not**
by itself establish which identifier this machine's trackpad emits; that remains open (§8.4.14).

Related material: coordinate system, units and scaling in §8.10; contact identity and lifecycle in
§8.11; amplitude, density and reported force in §8.12.

#### 8.4.1 Format selection

**The contact format is selected by the input report identifier and by nothing else.** There is no
negotiation, no capability bit and no format field consulted before the contact array is read. The
report identifier is the first byte of the report and is part of the frame. **[C]**

Two traps:

- **A published device property appears to select a parse format. It selects nothing.** It
  is not consulted when choosing a format and must not be used as a selector. A driver that keys on
  it will pick the wrong layout on any device whose firmware sends a different generation than the
  property suggests. **[C]**
- Report `0x02` is a compound wrapper (§8.2). When a frame arrives wrapped, **the nested report
  identifier is the selector**, not `0x02`. **[C]**

A device may also declare a list of report identifiers that are to be delivered to the host verbatim
as opaque messages rather than parsed as touch frames. Such identifiers are removed from the
dispatch before format selection happens, so a driver that implements that list must apply it first.
**[P]**

#### 8.4.2 Format summary

"Length" below means the length of the whole report **including** the leading identifier byte.

| Report id | Generation | Header | Contact record | Max contacts | Contact count |
|---|---|---|---|---|---|
| `0x24`, `0x26` | 1 (base) | 8 | 16 or more, unpacked | 32 | explicit, header byte 2; record size = (length − 8) ÷ count |
| `0x25` | 1 (base) | 8 | 12, packed | 32 | as above |
| `0x27` | 3 | 6 | 8 | 32 | derived: (length − 6) ÷ 8 |
| `0x28` | 4, compact | 4 | 9 | 32 | derived: (length − 4) ÷ 9 |
| `0x29` | 5 | 6 | 8 | 32 | derived: (length − 6) ÷ 8 |
| `0x31` | 7 | 4 | 9 | 32 | derived: (length − 4) ÷ 9 |
| `0x32` | 8 | 4 | 7 | **2** | derived: (length − 4) ÷ 7, clamped to 2 |
| `0x33` | 9 | 7 | 13 | **2** | explicit, header byte 1 bits 1:0 |
| `0x34` | 10 | 17 | 19 | **15** | explicit, header byte 13 bits 7:4 |
| `0x75` | 4, extended header | declared, normally 32 | declared, 20 to 30 | 32 | explicit, header byte 22; record size = contact-payload length ÷ count |
| `0x76` | extended header | as above | 48, floating-point | 32 | explicit, header byte 22; payload length must equal 48 × count |
| `0x77` | extended header | as above | 60, floating-point | **2** | explicit, header byte 22; payload length must equal 60 × count |

Rules that apply to all of them:

- The **contact-count ceiling is 32**, except for report `0x77` where it is 2. A frame declaring more
  contacts than its ceiling is invalid and **the whole frame must be discarded** — not truncated.
  **[C]**
- Where the count is derived from the length, a partial trailing record is ignored rather than
  treated as an error. **[C]**
- Where the count is explicit, the record size may still be implied by the payload length; for the
  extended-header family the contact-payload length must divide exactly by the count. **[C]**
- Reports `0x43`, `0x44`, `0x45`, `0x73` and `0x74` are older frame families with their own headers.
  They are **not specified here**. **[U]**

**Not every identifier above can occur on this machine.** The touch interface's accepted input
report set is given in §8.2 and does not include `0x24`–`0x27`, `0x32`, `0x33`, `0x34` or `0x77`.
Those generations are specified so that the dispatch is unambiguous and so that a driver written
against this document does not mis-handle a firmware revision that starts using one. **[C]**

#### 8.4.3 The uncompressed contact record — 30 bytes

Every compact generation is a compressed encoding of one common field set. That field set is also
the **wire record of the extended-header family at its maximum size**: a `0x75` contact record **is**
this structure on the wire, with no bit packing anywhere in it, each 16-bit field byte-swapped when
the device declares big-endian (§8.7). Smaller `0x75` record sizes are this structure truncated
(§8.4.11).

| Offset | Size | Sign | Field | Notes |
|---|---|---|---|---|
| 0 | 1 | u | **Contact identifier** | Tracking slot key; §8.11 |
| 1 | 1 | u | **Lifecycle state** | 0–7; §8.11 |
| 2 | 1 | u | **Classification code** | Small enumeration; §8.11 |
| 3 | 1 | s | **Grouping code** | Constant per generation; §8.11 |
| 4 | 2 | s | **X position** | Hundredths of a millimetre |
| 6 | 2 | s | **Y position** | Hundredths of a millimetre |
| 8 | 2 | s | **X velocity** | Eighths of a millimetre per second |
| 10 | 2 | s | **Y velocity** | As above. An all-ones pair at offsets 8–11 is the "no usable previous sample" sentinel, not a velocity |
| 12 | 2 | u | **Ellipse major-axis radius** | Hundredths of a millimetre; a radius, not a diameter |
| 14 | 2 | u | **Ellipse minor-axis radius** | As above |
| 16 | 2 | s | **Ellipse orientation** | Binary angle, full scale ±π; §8.10.4 |
| 18 | 2 | u | **Amplitude** | 8.8 fixed point; §8.12 |
| 20 | 2 | u | **Density** | 8.8 fixed point; §8.12 |
| 22 | 2 | u | Second density slot | Duplicates offset 20 except in the 24-byte-and-larger extended-header record, where it is a distinct field. No use for it is established **[U]** |
| 24 | 2 | s | Second orientation angle | Binary angle. Only the extended-header family carries it; its meaning is not established **[U]** |
| 26 | 2 | u | **Force-like scalar** | Grams; §8.12. Read the warning there before using it |
| 28 | 2 | u | **Per-contact flags** | Bitmask. Individual bit meanings are not established **[U]** |

The compact generations assemble their fields byte by byte and are therefore **endianness-neutral**;
the declared endianness affects only the base generation's 32-bit timestamp, the `0x24`/`0x26`
unpacked records and the extended-header family. **[C]**

#### 8.4.4 Generation 1 — base compact (`0x24`, `0x25`, `0x26`)

**Header, 8 bytes:**

| Byte | Size | Field |
|---|---|---|
| 0 | 1 | Report identifier |
| 1 | 1 | **Button / click state**, whole byte |
| 2 | 1 | **Contact count**, 1–32 |
| 3 | 1 | Frame flags; carried, meaning not established **[U]** |
| 4 | 4 | **Timestamp**, 32-bit, 1 ms tick. Byte-swapped when the device declares big-endian. No wrap extension is applied |

Contacts start at byte 8. The record size is **derived**: (length − 8) ÷ contact count. The frame is
valid only when 8 + size × count is within the report.

**Packed record (`0x25`), 12 bytes.** Source bytes are numbered 0–11 within the record.

| Source | Field | Width | Sign | Scaling |
|---|---|---|---|---|
| 0 all, 1 bits 5:0 | X position | 14 | signed | none; hundredths of a millimetre |
| 1 bits 7:6, 2 all, 3 bits 3:0 | Y position | 14 | signed | none |
| 3 bits 7:4, 4 bits 5:0 | X velocity | 10 | signed | ×4 → eighths of a millimetre per second |
| 4 bits 7:6, 5 all | Y velocity | 10 | signed | ×4 |
| 6 all, 7 bits 3:0 | Major radius | 12 | unsigned | none |
| 7 bits 7:4, 8 all | Minor radius | 12 | unsigned | none |
| 9 bits 5:0 | Amplitude | 6 | unsigned | ×32 → 8.8 fixed point |
| 9 bits 7:6, 10 bits 1:0 | Contact identifier | 4 | unsigned | — |
| 10 bits 7:2 | Orientation | 6 | unsigned | ×1024 → full-turn binary angle, 5.625° steps, read back signed |
| 11 bits 3:0 | Lifecycle state | 4 | unsigned | only 0–7 are defined |
| 11 bits 7:4 | Classification code | 4 | unsigned | — |

All 96 bits are assigned. No force-like field; no per-contact flags; density is not on the wire.

**Unpacked record (`0x24`, `0x26`), 16 bytes or more.** The same fields, byte-aligned, subject to the
declared endianness:

| Offset | Size | Field |
|---|---|---|
| 0 | 2 | X position, signed |
| 2 | 2 | Y position, signed |
| 4 | 2 | X velocity, signed |
| 6 | 2 | Y velocity, signed |
| 8 | 2 | Major radius |
| 10 | 2 | Minor radius |
| 12 | 2 | Low 12 bits: amplitude. Top 4 bits: contact identifier |
| 14 | 1 | Orientation code; ×256 to reach the binary angle |
| 15 | 1 | Low nibble: lifecycle state. High nibble: classification code |
| 16 | 2 | **Density**, present for `0x26` only when the record size is 18 or more |

The grouping code is not on the wire in this generation and is fixed at 1.

#### 8.4.5 Generations 3 and 5 (`0x27`, `0x29`)

Both use a **6-byte header** and an **8-byte contact record**, and both derive the count as
(length − 6) ÷ 8, bounded to 1–32. Contacts start at byte 6. The two generations differ only in how
header byte 3 is divided.

**Generation 3 header (`0x27`):**

| Byte | Bits | Field |
|---|---|---|
| 0 | 7:0 | Report identifier |
| 1 | 7:0 | Signed 8-bit delta, consumed as pointer motion **[P]** |
| 2 | 7:0 | Signed 8-bit delta, consumed as pointer motion **[P]** |
| 3 | 1:0 | **Button / click state**, 2 bits |
| 3 | 7:2 + 4 + 5 | **Timestamp**, 22 bits, 1 ms tick |

**Generation 5 header (`0x29`):**

| Byte | Bits | Field |
|---|---|---|
| 0 | 7:0 | Report identifier |
| 1 all + 3 bits 3:2 | — | Relative-pointer X delta, **10-bit signed** |
| 2 all + 3 bits 5:4 | — | Relative-pointer Y delta, **10-bit signed** |
| 3 | 1:0 | **Button / click state**, 2 bits |
| 3 | 7:6 + 4 + 5 | **Timestamp**, 18 bits, 1 ms tick |

Generation 5 trades four timestamp bits for four bits of relative-pointer precision. Neither header
carries a contact count.

**Contact record, 8 bytes:**

| Source | Field | Width | Sign | Scaling |
|---|---|---|---|---|
| 0 all, 1 bits 3:0 | X position | 12 | signed | ×2, no bias |
| 1 bits 7:4, 2 all | Y position | 12 | signed | ×2, then **+4095** |
| 3 | Major radius code | 8 | unsigned | companded, §8.4.12 |
| 4 | Minor radius code | 8 | unsigned | companded, §8.4.12 |
| 5 bits 5:0 | Amplitude | 6 | unsigned | ×32 → 8.8 fixed point |
| 5 bits 7:6, 6 bits 1:0 | Contact identifier | 4 | unsigned | — |
| 6 bits 7:2 | Orientation | 6 | unsigned | ×512 → half-turn binary angle, 2.8125° steps, 0° to 180° |
| 7 bits 3:0 | Classification code | 4 | unsigned | — |
| 7 bits 6:4 | Lifecycle state | 3 | unsigned | — |
| 7 bit 7 | No-prior-sample flag | 1 | — | sets the velocity sentinel of §8.4.3 |

All 64 bits are assigned. No velocity, no force-like field, no per-contact flags, no density on the
wire. Grouping code fixed at 1.

#### 8.4.6 Generation 4, compact (`0x28`)

**Header, 4 bytes:**

| Byte | Bits | Field |
|---|---|---|
| 0 | 7:0 | Report identifier |
| 1 | 0 | **Button / click state** |
| 1 | 1 | Surface-orientation indication **[P]** |
| 1 | 7:2 + 2 + 3 | **Timestamp**, 22 bits, 1 ms tick |

Contacts start at byte 4; record size 9; count = (length − 4) ÷ 9, bounded to 1–32.

**Contact record, 9 bytes:**

| Source | Field | Width | Sign | Scaling |
|---|---|---|---|---|
| 0 all, 1 bits 4:0 | X position | 13 | signed | ×2, no bias |
| 1 bits 7:5, 2 all, 3 bits 1:0 | Y position | 13 | signed | ×2, then **+5000** |
| 3 bits 7:2 | Spare field | 6 | unsigned | Carried on the wire and **not interpreted**. It is what makes this record nine bytes rather than eight, so the device does populate it. Meaning not established **[U]** |
| 4 | Major radius code | 8 | unsigned | companded, §8.4.12 |
| 5 | Minor radius code | 8 | unsigned | companded, §8.4.12 |
| 6 bits 5:0 | Amplitude | 6 | unsigned | ×32 → 8.8 fixed point |
| 6 bits 7:6, 7 bits 1:0 | Contact identifier | 4 | unsigned | — |
| 7 bits 7:2 | Orientation | 6 | unsigned | ×512 → half-turn binary angle, 2.8125° steps |
| 8 bits 3:0 | Classification code | 4 | unsigned | — |
| 8 bits 6:4 | Lifecycle state | 3 | unsigned | — |
| 8 bit 7 | No-prior-sample flag | 1 | — | sets the velocity sentinel |

All 72 bits are assigned. No velocity, no force-like field, no per-contact flags, no density on the
wire. Grouping code fixed at 1.

This generation is generation 3 and 5 with the coordinates widened from 12 to 13 bits, the Y bias
changed from 4095 to 5000, and the spare 6-bit field added.

#### 8.4.7 Generation 7 (`0x31`)

Header as §8.3, 4 bytes. Contacts start at byte 4; record size 9; count = (length − 4) ÷ 9, bounded
to 1–32.

**Contact record, 9 bytes:**

| Source | Field | Width | Sign | Scaling |
|---|---|---|---|---|
| 0 all, 1 bits 4:0 | X position | 13 | signed | ×2, no bias |
| 1 bits 7:5, 2 all, 3 bits 1:0 | Y position | 13 | signed | ×2, then **+5000** |
| 3 bits 4:2 | Classification code | 3 | unsigned | the value 7 is re-mapped to **12** |
| 3 bits 7:5 | Lifecycle state | 3 | unsigned | — |
| 4 | Major radius code | 8 | unsigned | companded, §8.4.12 |
| 5 | Minor radius code | 8 | unsigned | companded, §8.4.12 |
| 6 | Amplitude code | 8 | unsigned | companded, §8.4.12 |
| 7 | **Force-like code** | 8 | unsigned | companded to grams, 0–1008; §8.12 |
| 8 bits 3:0 | **Contact identifier** | 4 | unsigned | — |
| 8 bit 4 | No-prior-sample flag | 1 | — | sets the velocity sentinel |
| 8 bits 7:5 | Orientation | 3 | unsigned | ×4096 → 22.5° steps, 0° to 157.5° |

All 72 bits are assigned. No velocity, no per-contact flags and no density on the wire; grouping code
fixed at 1.

**A naming ambiguity worth stating.** This record contains two small identity-like fields: byte 3
bits 4:2 and byte 8 bits 3:0. Descriptions of this format disagree about which is "the identifier".
The field that acts as the **tracking slot key** — the one a driver must use for per-contact state
and for `ABS_MT_SLOT` — is **byte 8 bits 3:0**. Byte 3 bits 4:2 is the classification code (§8.11).
**[C]**

Generation 7 is the only compact generation that carries **both** a companded amplitude byte and a
force-like byte.

#### 8.4.8 Generation 8 (`0x32`)

**Header, 4 bytes:**

| Byte | Bits | Field |
|---|---|---|
| 0 | 7:0 | Report identifier |
| 1 | 0 | **Button / click state** |
| 1 | 7:1 + 2 | **Timestamp**, 15 bits, 1 ms tick |
| 3 | 3:0 | Per-contact side channel for contact 0 |
| 3 | 7:4 | Per-contact side channel for contact 1 |

Header byte 3 is **not part of the timestamp**: it carries one 4-bit side channel per contact, which
the contact record needs in order to be decoded. Contacts start at byte 4; record size 7; count =
(length − 4) ÷ 7, **clamped to 2**. A full frame is 18 bytes.

**Contact record, 7 bytes, plus the 4-bit side channel:**

| Source | Field | Width | Sign | Scaling |
|---|---|---|---|---|
| 0 all, 1 bits 3:0 | X position | 12 | signed | ×2, then **+2000** |
| 1 bits 7:4, 2 all | Y position | 12 | signed | ×2, then **+2000** |
| 3 | Major radius code | 8 | unsigned | companded, §8.4.12 |
| 4 | Minor radius code | 8 | unsigned | companded, §8.4.12 |
| 5 | Amplitude code | 8 | unsigned | companded, §8.4.12 |
| 6 bits 2:0 | Lifecycle state | 3 | unsigned | — |
| 6 bit 3 | Contact identifier | 1 | — | set → identifier 1; clear → identifier 2 |
| 6 bit 4 | No-prior-sample flag | 1 | — | sets the velocity sentinel |
| 6 bits 7:5 | Orientation | 3 | unsigned | ×4096 → 22.5° steps |
| side channel bits 1:0 | Classification code | 2 | unsigned | selects from the set {1, 2, 6, 12} in that order |
| side channel bits 3:2 | Per-contact flags | 2 | unsigned | value 0 → flags 0; 1 → 1; 2 → 32; 3 → 33 |

All 56 record bits are assigned. No velocity, no force-like field and no density on the wire;
grouping code fixed at 1. The identifier is a single bit because the format admits only two
contacts.

#### 8.4.9 Generation 9 (`0x33`)

**Header, 7 bytes:**

| Byte | Bits | Field |
|---|---|---|
| 0 | 7:0 | Report identifier |
| 1 | 1:0 | **Contact count**, 0–3 |
| 1 | 7:2 | 6-bit field, purpose undetermined **[U]** |
| 2 | — | **Timestamp**, 32-bit little-endian spanning bytes 2–5, 1 ms tick, used as received with no wrap extension |
| 6 | 7:0 | Carried; meaning not established **[U]** |

This header carries **no button bit, no orientation flags and no frame sequence number**. A driver
using generation 9 will see the click state reported as absent.

Contacts start at byte 7; record size 13. **One record is decodable when the length is at least 20;
two when the length is at least 33.** The header count field can encode 3, but a third record is not
decodable — a driver must bound the count by what the length supports rather than trusting the field.
**[C]**

**Contact record, 13 bytes:**

| Source | Field | Width | Sign | Scaling |
|---|---|---|---|---|
| 3 all, 4 bits 3:0; **sign bit from byte 0 bit 7** | X position | 13 | signed | none; hundredths of a millimetre. The sign is carried out of line: when byte 0 bit 7 is set, bits 12–15 of the value are set |
| 4 bits 7:4, 5 all; **sign bit from byte 2 bit 7** | Y position | 13 | signed | as above |
| 6 all, 7 bits 3:0 | Major radius | 12 | unsigned | none; linear |
| 7 bits 7:4, 8 all | Minor radius | 12 | unsigned | none; linear |
| 9 | Orientation | 8 | unsigned | **whole degrees**; multiply by 65536 and divide by 360 to reach the binary angle |
| 10 all, 11 bits 3:0 | Amplitude | 12 | unsigned | 8.8 fixed point |
| 11 bits 7:4, 12 all | **Density** | 12 | unsigned | 8.8 fixed point, **on the wire** |
| 0 bits 1:0 | Contact identifier | 2 | unsigned | the value 0 is re-mapped to **4**, giving 1–4 |
| 0 bits 4:2 | Lifecycle state | 3 | unsigned | — |
| 0 bit 5 | Per-contact flags bit 0 | 1 | — | — |
| 0 bit 6 | Per-contact flags bit 4 | 1 | — | — |
| 1 bits 3:0 | Classification code | 4 | unsigned | raw, no re-mapping |
| 1 bits 7:4, 2 bits 6:0 | **Force-like value** | 11 | unsigned | linear, no companding; §8.12 |

All 104 bits are assigned. No velocity on the wire. Grouping code fixed at 0, not 1.

Generation 9 abandons companding entirely: every magnitude is a plain linear integer, positions are
unbiased, orientation resolution rises to one degree, and both density and the force-like value
become real wire fields.

#### 8.4.10 Generation 10 (`0x34`)

**Header, 17 bytes:**

| Byte | Size | Field |
|---|---|---|
| 0 | 1 | Report identifier |
| 1 | 4 | **Timestamp**, 28 bits spanning bytes 1–4 (byte 4 contributes its low nibble as the top nibble). **The tick is 0.3125 ms**, not 1 ms — see §8.10.6 |
| 5 | 8 | Carried; meaning not established **[U]** |
| 13 | 1 | Bit 1: contact data present. Bit 2: image data present. Bits 7:4: **contact count**, 0–15 |
| 14 | 1 | Carried; meaning not established **[U]** |
| 15 | 1 | **Image payload length in bytes** |
| 16 | 1 | Carried; meaning not established **[U]** |

The **image payload immediately follows the header**, and the contact array follows the image
payload. Contacts therefore start at 17 + image length, not at 17. A frame is valid only when the
image length plus 36 is within the report, and when the contact count is at least 1. **[C]**

**Contact record, 19 bytes.** Unlike every other compact generation, nothing is bit-packed: every
field is a byte-aligned little-endian value, and no byte swap is applied regardless of declared
endianness.

| Offset | Size | Sign | Field | Units |
|---|---|---|---|---|
| 0 | 2 | s | X position | Hundredths of a millimetre |
| 2 | 2 | s | Y position | Hundredths of a millimetre |
| 4 | 2 | s | X velocity | Eighths of a millimetre per second |
| 6 | 2 | s | **Y velocity** | Eighths of a millimetre per second — **see the defect note below** |
| 8 | 2 | u | Major radius | Hundredths of a millimetre |
| 10 | 2 | u | Minor radius | Hundredths of a millimetre |
| 12 | 2 | u | Amplitude | 8.8 fixed point |
| 14 | 2 | u | **Density** | 8.8 fixed point, on the wire |
| 16 | 1 | u | Low nibble: contact identifier. High nibble: lifecycle state | — |
| 17 | 2 | u | Per-contact flags | Bitmask |

Generation 10 carries **no orientation, no force-like field, no classification code and no grouping
code**; the classification code is fixed at 2 and the grouping code at 0. It does carry velocity and
density directly, so neither has to be derived.

**Caution — the Y-velocity field.** Every other multi-byte field in this record is a clean adjacent
pair, which places the Y velocity at bytes **6 and 7**. Decode it from bytes 6 and 7. That the device
populates that pair is inferred from the regularity of the record rather than established
directly. **[P]**

A decoding is known to exist that takes the low half from byte 5 instead — byte 5 being the high
half of the X velocity — leaving byte 6 unread. An implementation that must interoperate
bit-for-bit with such a decoder would see a different Y velocity than the record implies. This is
recorded only as a compatibility hazard; nothing here establishes it as intended. **[U]**

#### 8.4.11 The extended-header family (`0x75`, `0x76`, `0x77`)

**Header, 32 bytes.** All multi-byte fields are byte-swapped when the device declares big-endian
(§8.7). The header is valid only when the report is at least 32 bytes and the declared header length
is at least 16.

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | Report identifier — also the payload-format selector within this family |
| 1 | 1 | **Frame sequence number** |
| 2 | 1 | **Header length in bytes**, normally 32, minimum 16 |
| 3 | 1 | **Format version** |
| 4 | 4 | **Timestamp**, 32-bit, 1 ms tick, used as received |
| 8 | 1 | Carried; meaning not established **[U]** |
| 9 | 1 | Carried; meaning not established **[U]** |
| 10 | 1 | Flags. Bits 1:0 select a contact record size from the set {6, 8, 12, 16}; this applies to the older `0x73`/`0x74` family, and `0x75` takes its record size from the payload length instead **[P]**. Bit 7, when set, means the frame's payloads are not to be processed. Bits 2, 3 and 6 are carried; meanings not established **[U]** |
| 11 | 1 | Carried; meaning not established **[U]** |
| 12 | 2 | Flags. Bit 0: the flag of §8.3. Bit 1: **contact data present**. Bit 2: **image data present** |
| 14 | 2 | **Auxiliary block length in bytes.** The block sits immediately after the header. When the length is exactly 6 the block is three 16-bit values |
| 16 | 2 | **Contact payload length in bytes** |
| 18 | 2 | **Image payload length in bytes** |
| 20 | 2 | Carried; meaning not established **[U]** |
| 22 | 1 | **Contact count**, 1–32 |
| 23 | 1 | **Button / click state**, whole byte |
| 24 | 4 | 32-bit field, purpose undetermined **[U]** |
| 28 | 2 | 16-bit field, purpose undetermined **[U]** |
| 30 | 2 | 16-bit field, purpose undetermined **[U]** |

**Contact array placement.** The contact array begins at *(header length) + (auxiliary block
length)*. The **record size is the contact payload length divided by the contact count**, and the
division must be exact. Valid record sizes are 20, 22, 24, 26, 28 and 30 bytes. **[C]**

**Progressive truncation.** A record is the 30-byte structure of §8.4.3 cut short:

| Record size | Fields carried |
|---|---|
| under 20 | nothing usable |
| 20 | identifier, state, classification, grouping, X, Y, X velocity, Y velocity, major radius, minor radius, orientation, amplitude |
| 22 | the above plus **density** |
| 24 | plus the second density slot as a distinct field |
| 26 | plus the second orientation angle |
| 28 | plus the **force-like scalar** |
| 30 | plus the **per-contact flags** |

A device without force sensing would naturally declare a 26-byte record, or a 30-byte record with a
zero force field. **[P]**

**Reports `0x76` and `0x77`.** These use the same header and carry fixed-size floating-point contact
records instead — 48 bytes for `0x76`, 60 bytes for `0x77` — in which the same quantities appear
scaled by 1000, that is in micrometres and micrometres per second. `0x77` is limited to two contacts.
Neither is relevant to a mechanical-switch trackpad; they are listed so that their identifiers are
not mistaken for something else. **[C]**

#### 8.4.12 Companding curves

Generations 3, 4, 5, 7 and 8 transmit the radii, and generations 7 and 8 also the amplitude, as
8-bit codes that expand to 16-bit values through fixed monotone piecewise-quadratic curves. The
curves are **identical constants across those generations**. Generation 7 additionally uses a fourth
curve for its force-like byte.

All arithmetic below is integer. "÷1024" means truncating integer division by 1024. A code of 0 is a
distinguished "absent" value in the radius and amplitude curves, not a small value.

**Major-axis radius**, result in hundredths of a millimetre:

| Code range | Value |
|---|---|
| 0 | 0 |
| 1–49 | (742400 − 40 × (code − 75)²) ÷ 1024 |
| 50–169 | 600 + 2 × code |
| 170–255 | (256 × (code − 166)² + 958464) ÷ 1024 |

**Minor-axis radius**, result in hundredths of a millimetre:

| Code range | Value |
|---|---|
| 0 | 0 |
| 1–49 | (640000 − 40 × (code − 75)²) ÷ 1024 |
| 50–149 | 500 + 2 × code |
| 150–255 | ((((1024 × code − 141552)² ÷ 1024) × 87) ÷ 1024 + 807152) ÷ 1024 |

**Amplitude**, result in 8.8 fixed point:

| Code range | Value |
|---|---|
| 0 | 0 |
| 1–149 | (code × 10485) ÷ 1024 |
| 150–255 | ((((1024 × code − 114670)² ÷ 1024) × 138) ÷ 1024 + 1373544) ÷ 1024 |

**Force-like scalar** (generation 7 only), result in grams:

| Code range | Value |
|---|---|
| 0 | 0 |
| 1–127 | 2 × code |
| 128–255 | (((1024 × code − 98939)² ÷ 32768) + 230011) ÷ 1024 |

Check values, for validating an implementation:

| Code | Major | Minor | Amplitude | Force-like |
|---|---|---|---|---|
| 0 | 0 | 0 | 0 | 0 |
| 1 | 511 | 411 | 10 | 2 |
| 49 | 698 | 598 | 501 | 98 |
| 50 | 700 | 600 | 511 | 100 |
| 149 | 898 | 798 | 1525 | 310 |
| 150 | 900 | 799 | 1536 | 313 |
| 169 | 938 | 868 | 1779 | 388 |
| 170 | 940 | 873 | 1794 | 392 |
| 255 | 2916 | 1946 | 4097 | 1008 |

Resulting ranges: major radius 5.11 mm to 29.16 mm, minor radius 4.11 mm to 19.46 mm, amplitude 0.04
to 16.00 after the 8.8 conversion, force-like scalar 2 g to 1008 g. All four curves are monotone
non-decreasing for codes 1 and above. A 256-entry lookup table generated once from the formulas above
is the simplest faithful implementation. **[C]**

Generations 1, 9 and 10 and the extended-header family do **not** compand: their radii and
amplitudes are linear.

#### 8.4.13 Warning: the generation number is not a discriminator

**Do not key any parsing decision on a generation number. Key on the report identifier.** Two
independent reasons:

1. **Two unrelated formats carry the same generation number.** Report `0x28` (§8.4.6) is a 4-byte
   header with 9-byte bit-packed records. Report `0x75` (§8.4.11) is a 32-byte self-describing header
   with 20-to-30-byte byte-aligned records. They share a generation number and a common decoded field
   set, yet have **zero field offsets in common**. A driver that dispatches on the generation number
   will mis-parse one of them completely. The mapping from generation number to format is therefore
   not invertible.
2. **The compact formats do not carry a generation number on the wire at all.** Only the
   extended-header family has a format-version byte (byte 3). Any generation number associated with
   a compact frame comes from whatever chose the format in the first place — the report identifier.

The consequence is a **silent** mis-parse: both candidate layouts are plausible-looking byte
sequences, so the failure surfaces as wrong coordinates rather than as a length or checksum error.
**[C]**

#### 8.4.14 Which format this trackpad emits

**Not established. [U]**

What is known:

- The candidates are already narrowed to **`0x31` and `0x75`** (§8.2), possibly delivered inside the
  `0x02` compound wrapper. **[C]**
- Nothing in the material behind this section decides between them: format selection is by report
  identifier only (§8.4.1), and no device property maps a product to a format.
- Generation 7 (`0x31`) is the better fit on shape: a 4-byte header carrying exactly **one** button
  bit matches a single mechanical switch; a 9-byte record minimises report size on a constrained
  link; and the format's per-contact force-like byte is optional in practice because a device
  without force sensing can transmit zero. **[P]**
- The counter-argument is that `0x75` can declare a 26-byte record and so drop the force field
  entirely, and its whole-byte button field is equally consistent with one switch. **[P]**

**This must be settled by capture** — it is the single most valuable remaining measurement for the
touch path (§12.2). The header formats differ in length (4 versus 32 bytes) and the record sizes
differ (9 versus 20–30), so one captured frame of known length with one finger down resolves it
immediately.

### 8.5 Status frames (report `0x60`)

Exactly 2 bytes: the report identifier and a status value. **The values below are status values
inside report `0x60`, not report identifiers** — in particular `0x02` here is unrelated to the
input and feature reports numbered `0x02` (§1.1).

| Value | Meaning |
|---|---|
| `0x01` | Reset occurred |
| `0x02` | Device initialised; host should re-read cached device properties |
| `0x04` | Orderly shutdown notification |
| `0x05` | Sensor controller watchdog expired |
| `0x10` | Externally triggered reset, initialisation complete |
| `0x11` | Power-on reset, initialisation complete |
| `0x12` | Watchdog reset, initialisation complete |
| `0x13` | Firmware-requested reset, initialisation complete |

Valid range is `0x00`–`0x13`. Values `0x10`–`0x13` indicate the device has completed
re-initialisation and is usable again. A driver should treat these as the authoritative
"device is back" signal after any reset.

Values `0x01`, `0x02`, `0x04` and `0x10`–`0x13` are also the trigger for re-evaluating the
environment-dependent tuning state of §8.7 (`0xA8`).

### 8.6 Critical error frames (report `0x50`)

Three encodings, distinguished by length and a sub-type byte:

| Length | Sub-type | Layout after the 2-byte prefix |
|---|---|---|
| 8 | `0x01` | 2-byte error bits, 4-byte timestamp |
| 12 | `0x02` | 2-byte pad, 4-byte error bits, 4-byte timestamp |
| 16 | `0x03` | 2-byte pad, 4-byte error bits, 8-byte timestamp |

Error bits are a set of independent flags and should be accumulated. Individual bit meanings are
not specified by the interface. **[U]**

### 8.7 Feature reports: identity, geometry, configuration

All are HID **feature** reports on the touch interface. Buffer sizing: individual reports fit within
1 KiB; the bundle report below requires 520 bytes. On this machine every feature get is issued with
a **512-byte** buffer and no length pre-negotiation, because report `0x01` is not available (below);
512 bytes is therefore the effective maximum payload.

| ID | Direction | Payload | Function |
|---|---|---|---|
| `0x00` | Get | 1 byte | Device-initialised flag. Not required on this machine; treat as unsupported. |
| `0x01` | Set then Get | Target identifier; returns validity and a 2-byte length | Per-report metadata query. **Not available on this machine**; a driver must size buffers without it. |
| `0x02` | Set | 1 byte, `0` or `1` | Host-association state; `1` = associated |
| `0x7F` | Get | 2 or 4 bytes | Status and critical-error bits, accumulated by the host |
| `0xA1` | Get | Opaque | Sensor region parameters |
| `0xA8` | Set | 1 byte | Environment-dependent tuning state, values `0`, `5`, `6` (below) |
| `0xD0` | Get | Opaque | Sensor region descriptor |
| `0xD1` | Get | ≥1 byte | Device family identifier (below) |
| `0xD3` | Get | ≥5 bytes | Basic device information (below) |
| `0xD9` | Get | ≥16 bytes | Surface geometry (below) |
| `0xDB` | Get | ≤520 bytes | Bundle of several of the above (below) |
| `0xDC` | Set | 1 byte, `0` or `2` | Surface orientation. The device rotates; the host does not (§8.10.1) **[P]** |
| `0xDD` | Set | 1 byte, `1` or `2` | Surface-orientation mode **[P]** |
| `0x71`, `0x72`, `0x73` | — | — | Power telemetry and host state hint; see §9.9 |
| `0x7E` | Get | 129 bytes | Periodic counter report; see §9.9 |

**Device family identifier (`0xD1`):** the identifier is taken from **payload byte 0**. Whether the
field is wider than one byte is not established; the value is published host-side as a 32-bit
quantity. Read at least one byte and do not assume four. **[P]**

**Basic device information (`0xD3`):**

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | Endianness indicator; value `2` means the device reports big-endian |
| 1 | 1 | Sensor rows |
| 2 | 1 | Sensor columns |
| 3 | 2 | Version, **big-endian** |
| 8 | 4 | Maximum frame size (present when length ≥ 12). **Byte order is conditional**: the field is byte-swapped when, and only when, the endianness indicator at offset 0 equals `2` |
| 12 | 1 | Extended feature flags (present when length ≥ 14) |

The maximum frame size is a value that is only ever **raised**, never lowered: the effective value is
the larger of any value the host already holds and the value the device reports. When the field is
absent the default is 1024 bytes. The opposite direction is a plausible misreading; the rule given
here is the one to implement. **[C]**

**Surface geometry (`0xD9`):** requires a payload of at least **16 bytes**, and the first 16 bytes
are fully specified. **The unit throughout is one hundredth of a millimetre** (§8.10.1). **[C]**

| Offset | Size | Sign | Field |
|---|---|---|---|
| 0 | 4 | u | Surface width |
| 4 | 4 | u | Surface height |
| 8 | 2 | s | Minimum X |
| 10 | 2 | s | Minimum Y |
| 12 | 2 | s | Maximum X |
| 14 | 2 | s | Maximum Y |

All six fields are byte-swapped unless the device declares little-endian (`0xD3`, offset 0). The
four bounds are the **normalisation rectangle**; see §8.10.3, which also records the two
unresolved points about this payload — the ordering of the four bounds, and whether the length is
"exactly 16" or "at least 16". The descriptor is still, as before, **the entire payload starting at
offset 0**, overlapping the width and height fields — not a slice beginning at offset 8.

**Default dimensions.** When a device supplies neither the bounds nor the width and height, the
values to assume are **5000 × 7500**, that is **50.00 mm × 75.00 mm**. This is a fallback, not a
measurement of this machine's trackpad; a driver should prefer whatever the device reports. **[P]**

**Bundle report (`0xDB`):** a single fetch returning several of the above in one transfer.

- Byte 0: report identifier.
- Byte 1: **validity flag; it must equal 1**, and any other value means the response must be
  discarded rather than parsed.
- From byte 2: entries, each a 2-byte length, a 1-byte report identifier, then (length − 1) payload
  bytes. The stride to the next entry is the length field plus 2. The length field must be non-zero.

Any report present in the bundle need not be fetched individually; the ones not present must be.

**Sensor region descriptor (`0xD0`):** a 1-byte region count followed by that many **7-byte**
entries:

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | Region type. A **bit index**, not a plain enumeration: index 1 identifies the multi-touch region. Other indices designate other sensing regions and are not enumerated here **[P]** |
| 1 | 1 | First row |
| 2 | 1 | Row count |
| 3 | 1 | Row stride; a value of 0 must be treated as 1 |
| 4 | 1 | First column |
| 5 | 1 | Column count |
| 6 | 1 | Hardware column offset |

Row and column counts are each bounded by **64**. When no multi-touch region is supplied, the
fallback is first row 0, rows = sensor rows, stride 1, first column 0, columns = sensor columns,
offset 0. **[P]**

**Region parameters `0xA1` remain opaque** — a calibration blob passed through without
interpretation. Its internal structure is undetermined. **[U]**

**Environment-dependent tuning (`0xA8`):** a 1-byte set whose value is derived by the host from the
attached power adapter's family: two specific adapter families select the values `5` and `6`, and
anything else (including no adapter) selects `0`. It is re-evaluated on power-source changes and on
the status-frame values listed in §8.5. On this machine the platform data that arms this path is
absent, so it is unlikely to be exercised here — but the report exists in the interface. **[C]**

### 8.8 Configuration surface: what is and is not there

There is **no** feature report for enabling or disabling touch reporting, sensitivity, palm
rejection or report rate.

**Enabling touch reporting is a transport-level concern** — it is achieved by powering the interface
on (§9), not by a touch-specific command. This is an important negative result: implementers
should not search for a general enable command. **[C]**

Three qualifications keep this from being over-broad:

- Feature report `0xA8` (§8.7) *is* a device-side tuning control, even though it is environmental
  rather than behavioural.
- The host-association feature report `0x02` is written on some enable paths.
- **Orientation is an exception and this document previously stated otherwise.** Two 1-byte set
  feature reports, `0xDC` and `0xDD`, do control surface orientation, and they act on the **device**:
  the frame contents change, the host applies no rotation of its own (§8.10.1). Their value
  semantics are not established. **[P]**

**No device-side raw-capacitance or per-electrode diagnostic mode is reachable at this level.**
Nothing in the specified surface enables one. This establishes only that no such mode is exposed
through the interface described here; it does not establish that the device lacks one. **[U]**

### 8.9 Protocol generation

Generation is selected by **input report identifier**, and by nothing else — not by negotiation,
not by a capability bit, and not by any device property. §8.4.1 states the rule and names the two
traps; §8.4.13 states why a generation *number* is not a usable substitute for the identifier.
**[C]**

The other generation-like values a device exposes do **not** select a format, and none of them may be
used to choose one:

| Value | Where it comes from | What it does **not** do |
|---|---|---|
| Version field | Feature report `0xD3`, offset 3 | Does not select a contact format |
| Device family identifier | Feature report `0xD1` | Selects sensor-geometry defaults host-side; does not select a contact format **[C]** |
| Protocol identifier string | Provisioning configuration record (§7.2) | Names the touch controller generation; does not reach the frame path **[P]** |
| The format-selecting device property | Device property set | Nothing — format is chosen by report identifier alone (§8.4.1) **[C]** |
| Format-version byte | Extended-header family only, header byte 3 | Present only in that family; the compact formats carry no generation number on the wire **[C]** |

The version field and the family identifier are device-supplied at runtime and are not fixed by the
platform image; the protocol identifier string is supplied by platform data and a driver never needs
to construct it. The generation identity also appears in the device's initialisation status
reporting. **[P]**

### 8.10 Coordinate system, units and scaling

This section applies to every contact format in §8.4 and to the surface geometry reports of §8.7.

#### 8.10.1 The positional unit and the axes

- **The positional unit is one hundredth of a millimetre.** It is used uniformly: contact X and Y,
  the ellipse radii, the surface width and height, and the surface bounds are all expressed in it.
  A value of 100 is one millimetre. **[C]**
- **X is the side-to-side axis. Y is the top-to-bottom axis.** **[C]**
- **The X origin is the horizontal centre of the electrode array.** X is signed and is negative over
  half the surface. **[C]**
- **The Y origin is the first electrode row.** Y is non-negative across the surface. **[C]**
- The origin is therefore neither a corner nor the geometric centre: it is centred in X and
  zero-based in Y.
- **Which physical direction +Y points** — toward the user or away from them — is fixed by the
  sensor's own electrode numbering and is **not established** here. It must be determined by
  measurement. **[U]**
- **No rotation, mirroring, axis swap or scaling is applied between the wire and the physical
  surface.** Where surface orientation is changed, it is changed **in the device**: two 1-byte
  feature reports exist for it, `0xDC` (values `0` or `2`) and `0xDD` (values `1` or `2`). Their
  exact semantics are not established, and §8.8's statement that no orientation control exists is
  corrected by this. **[P]**

#### 8.10.2 Per-format coordinate ranges and biases

Each generation packs its coordinates differently and applies its own fixed scale and bias before
the value reaches the common record of §8.4.3. The scale is applied first, then the bias.

| Generation | Width | Scale | X bias | Y bias | X range | Y range |
|---|---|---|---|---|---|---|
| 1, packed (`0x25`) | 14 signed | ×1 | 0 | 0 | −8192 … 8191 | −8192 … 8191 |
| 1, unpacked (`0x24`, `0x26`) | 16 signed | ×1 | 0 | 0 | full signed 16-bit | full signed 16-bit |
| 3, 5 | 12 signed | ×2 | 0 | **+4095** | −4096 … 4094 | −1 … 8189 |
| 4 compact, 7 | 13 signed | ×2 | 0 | **+5000** | −8192 … 8190 | −3192 … 13190 |
| 8 | 12 signed | ×2 | **+2000** | **+2000** | −2096 … 6094 | −2096 … 6094 |
| 9 | 13 signed | ×1 | 0 | 0 | −4096 … 4095 | −4096 … 4095 |
| 10, extended header | 16 signed | ×1 | 0 | 0 | full signed 16-bit | full signed 16-bit |

Ranges are in hundredths of a millimetre. Where the scale is ×2, the **resolution on the wire is
0.02 mm**, not 0.01 mm.

Consequences worth noting: generation 7 addresses ±81.92 mm in X about the centre and
−31.92 mm … +131.90 mm in Y, which comfortably covers a laptop trackpad. Generation 8 addresses only
81.90 mm per axis with a 20.00 mm bias on both. Generation 9 addresses ±40.96 mm per axis.

#### 8.10.3 The normalisation rectangle

Positions are absolute, not normalised. A host that wants a normalised position needs a rectangle,
and the rectangle comes from the device, not from the wire format.

**Source.** Feature report `0xD9` (§8.7). Its 16-byte payload is:

| Offset | Size | Sign | Field |
|---|---|---|---|
| 0 | 4 | u | Surface width, hundredths of a millimetre |
| 4 | 4 | u | Surface height, hundredths of a millimetre |
| 8 | 2 | s | Minimum X |
| 10 | 2 | s | Minimum Y |
| 12 | 2 | s | Maximum X |
| 14 | 2 | s | Maximum Y |

All six fields are byte-swapped unless the device declares little-endian (§8.7). **The order of the
four bounds is not fully settled** — one description gives minimum X, minimum Y, maximum X, maximum
Y as above, another gives maximum X, minimum X, maximum Y, minimum Y for the same quantities held
host-side. A driver should sanity-check that its minima are below its maxima and swap if not. **[P]**
Similarly, one description requires the payload to be **exactly** 16 bytes while §8.7 records a
minimum of 16; accept 16 or more and read the first 16. **[P]**

**Use.**

- When the four bounds are supplied and are not all zero, **they are the rectangle**.
- Otherwise the rectangle must be synthesised from the surface width and height: X from
  −width ÷ 2 to +width ÷ 2, Y from 0 to height. This matches the origin convention of §8.10.1. It is
  an approximation of the rectangle a device that does report bounds would give, not an equivalent of
  it. **[P]**
- Normalised position on each axis is *(value − minimum) ÷ (maximum − minimum)*, giving 0 to 1.
- **Positions from generations 1, 3, 4, 5, 7 and 8 should be clamped into the rectangle before
  use. Generations 9 and 10 are not clamped.** **[C]**
- The rectangle is deliberately slightly larger than the electrode extent, by a margin of roughly a
  quarter of an electrode pitch on each side, so that a contact centroid extrapolated past the
  outermost electrode still lands inside it. **[P]**

For Linux input axes, report `ABS_MT_POSITION_X` and `ABS_MT_POSITION_Y` in hundredths of a
millimetre with the rectangle's minima and maxima as the axis limits, and set the axis resolution
accordingly.

#### 8.10.4 Radii and orientation

- The two ellipse axis values are **radii (semi-axes)**, in hundredths of a millimetre. Linux
  `ABS_MT_TOUCH_MAJOR` and `ABS_MT_TOUCH_MINOR` are **diameters**, so a driver must double them.
  **[C]**
- Derived quantities in common use: mean radius is the **geometric** mean, the square root of
  major × minor. Eccentricity is major ÷ minor with the divisor floored to guard against a vanishing
  minor axis; contacts below the floor are reported circular. **[P]**
- **One caveat on the radius unit.** It is given as hundredths of a millimetre on the strength of the
  positions in the same record sharing that unit and being converted identically. The eccentricity
  guard above is a bare value of 6, which fits a millimetre-scale guard but would be an implausible
  0.06 mm guard — which is consistent with the radii being a conventional sensor magnitude rather
  than a strict length. Treat a *physical* reading of the radii as **[P]**; the numeric conversion
  itself is **[C]**.
- **Orientation is a signed 16-bit binary angle with full scale ±π**: degrees = value × 180 ÷ 32768.
  It is measured from the +X axis, counter-clockwise, in the usual mathematical convention. **[C]**
- Per-generation angular resolution differs sharply:

| Generation | Wire width | Step | Range |
|---|---|---|---|
| 1 | 6 bits | 5.625° | full turn, −180° to +180° |
| 3, 4 compact, 5 | 6 bits | 2.8125° | half turn, 0° to 180° |
| 7, 8 | 3 bits | 22.5° | 0° to 157.5° |
| 9 | 8 bits | 1° | whole degrees; only 0° to 180° is meaningful for an ellipse axis |
| 10 | absent | — | orientation is not carried |
| extended header | 16 bits | 180 ÷ 32768 degrees | full scale ±180° |

A driver mapping to `ABS_MT_ORIENTATION` should scale from the binary angle rather than from the
per-generation code, so that the same code path serves every generation.

#### 8.10.5 Velocity

- **The velocity unit is one eighth of a millimetre per second**, signed, per axis. **[C]**
- Velocity is **on the wire** only for generation 1 (as a 10-bit signed value scaled by 4, so its
  wire resolution is 0.5 mm/s), generation 10, and the extended-header family. **[C]**
- Generations 3, 4, 5, 7, 8 and 9 do **not** carry velocity. It must be **derived by the host** from
  the change in position and the change in the frame timestamp. **[C]**
- The sentinel for "no usable previous sample" is an all-ones pair in the two velocity fields of the
  common record (§8.4.3). Generations 3, 4, 5, 7 and 8 set it from a dedicated per-contact bit. A
  driver must treat the sentinel as *absence*, not as a velocity of −1. **[C]**
- A derived velocity should only be computed while a contact is in states 4, 5 or 6 (§8.11.2); in
  any other state it should be reported as zero. **[C]**
- A derived velocity in the same units is *(position change in millimetres) × 8 ÷ (elapsed
  seconds)*. Normalised velocity, if wanted, is the millimetre-per-second value divided by the
  corresponding rectangle extent in millimetres, giving surface fractions per second.

#### 8.10.6 Timestamps

Every touch frame carries a free-running counter in its header. **The tick is one millisecond** for
every generation except generation 10. **[C]**

| Generation | Width | Location | Tick | Wrap period |
|---|---|---|---|---|
| 1 | 32 bits | header bytes 4–7 | 1 ms | about 49.7 days |
| 3 | 22 bits | header byte 3 bits 7:2, bytes 4, 5 | 1 ms | about 70 minutes |
| 4 compact | 22 bits | header byte 1 bits 7:2, bytes 2, 3 | 1 ms | about 70 minutes |
| 5 | 18 bits | header byte 3 bits 7:6, bytes 4, 5 | 1 ms | about 4.4 minutes |
| 7 | 21 bits | header byte 1 bits 7:3, bytes 2, 3 | 1 ms | about 35 minutes |
| 8 | 15 bits | header byte 1 bits 7:1, byte 2 | 1 ms | about 33 seconds |
| 9 | 32 bits | header bytes 2–5 | 1 ms | about 49.7 days |
| 10 | 28 bits | header bytes 1–4 | **0.3125 ms** | about 23.3 hours after scaling |
| extended header | 32 bits | header bytes 4–7 | 1 ms | about 49.7 days |

- **Epoch is undefined.** The counter is a free-running device counter with no stated zero point.
  Only differences are meaningful. **[C]**
- **Generation 10 is scaled**: multiply the 28-bit raw value by 3125 and divide by 10000 to obtain
  milliseconds, giving a wire tick of 0.3125 ms (3200 ticks per second). The scaled value wraps at
  83 886 080 ms. **[C]**
- **Wrap extension** is required for the narrow counters and is uniform: keep the high bits of the
  previously extended value, substitute the newly received low bits, and if the new low bits are
  numerically **less** than the previous low bits, add one modulus. Generations 1, 9 and the
  extended-header family use their 32-bit value as received. **[C]**
- A timestamp that moves **backwards** after extension indicates the device restarted its counter;
  a driver should discard its per-contact tracking state rather than compute a negative interval.
  **[C]**
- **There is no frame sequence number on the wire for any compact generation.** Only the
  extended-header family carries one (byte 1). A host that needs a frame counter must synthesise it
  by incrementing on each accepted frame. **[C]**

### 8.11 Contact identity and lifecycle

#### 8.11.1 The identifier

**The contact identifier is byte 0 of the common record (§8.4.3).** It is the key a driver uses for
per-contact state and it is the natural source for `ABS_MT_SLOT`.

- **The device assigns it.** Nothing on the host side allocates, recycles or renumbers identifiers;
  it is a pure lookup key. **[C]**
- **The usable range is 1 to 31.** The value 0, and any value of 32 or above, are not distinguishable
  from one another — they collapse onto a single shared slot with no per-contact isolation. A driver
  should treat identifier 0 as "the device assigned no identifier" rather than as contact zero.
  **[C]**
- **Identity is not positional.** The array index a contact occupies within a frame carries no
  meaning, and the same contact may appear at different indices in consecutive frames. **[C]**
- Per-generation width, which bounds practical simultaneity below the 32-contact ceiling:

| Generation | Identifier width | Values |
|---|---|---|
| 1 | 4 bits | 0–15 |
| 3, 4 compact, 5 | 4 bits | 0–15 |
| 7 | 4 bits | 0–15 |
| 8 | 1 bit | 1 or 2 |
| 9 | 2 bits, 0 re-mapped to 4 | 1–4 |
| 10 | 4 bits | 0–15 |
| extended header | 8 bits | 0–255, usable 1–31 |

- **No reuse guarantee is established.** Nothing states how long a device waits before re-issuing an
  identifier that has just been released. A driver must not assume a cool-down. **[U]**

#### 8.11.2 Lifecycle states

**The lifecycle state is byte 1 of the common record and is carried on the wire**, per contact, in
every generation. It is a 3- or 4-bit field depending on generation (§8.4).

| Value | Meaning |
|---|---|
| 0 | Not tracked — the slot is idle, or the contact is gone |
| 1 | Contact appears: the first frame of a newly tracked contact |
| 2 | Detected in proximity, not touching |
| 3 | Touch-down transition |
| 4 | In contact |
| 5 | Lift-off transition |
| 6 | In proximity again after lift-off |
| 7 | Leaving range: the final frame for this contact |

Values of 8 and above are invalid. **[C]**

**The device does not validate the order.** Any value may follow any other; a host must not assume a
well-formed sequence. **[C]**

Useful predicates:

- A contact is **active** when its amplitude is above zero **or** its state is in the range 1 to 6.
  State 7 alone does not make a contact active. **[C]**
- A contact should be **reported to clients** when its amplitude is above zero or its state is in the
  range 1 to 7, so that the terminating state-7 frame is delivered. **[C]**
- **A real touch** corresponds to states 3, 4 and 5. States 1, 2 and 6 are proximity without
  contact and must not raise a touch. **[C]**

#### 8.11.3 What is on the wire and what the host must derive

**On the wire:** the identifier, the state value, and every transition the device chooses to emit.

**Derived by the host — two transitions that never appear on the wire:**

1. **Stale-contact substitution.** A contact that reports state 0 within **40 milliseconds** of
   having last been in state 3 or 4 should be presented as state **7** rather than state 0, so that
   clients see an orderly end of touch. A driver may skip this and simply end the touch; it changes
   presentation, not meaning. **[C]**
2. **Ageing out a vanished contact.** A contact whose identifier simply **stops appearing** in
   frames receives no terminating record from the device. The host must synthesise one: on the first
   frame in which the identifier is absent, present state **7**; on the next, state **0**; then the
   slot is quiet. Amplitude should be forced to zero at the same time. **[C]**

Note that the second rule interacts with the whole-frame drop of §8.4.2: a frame discarded for
declaring too many contacts must **not** advance the ageing logic, or contacts will be aged out on a
frame that was never parsed. **[C]**

#### 8.11.4 Minting a stable tracking identifier

The device's identifier is a slot key, not a tracking identifier: the same value is reused for
unrelated contacts over time. To produce a stable `ABS_MT_TRACKING_ID` a driver must:

- Map the device identifier to the input slot.
- **Allocate a fresh tracking identifier whenever state 1 is seen for an identifier, and whenever an
  identifier reappears after having been absent from one or more frames** — not merely when the
  numeric identifier changes. Both conditions are necessary; either alone will merge two distinct
  touches. **[C]**
- Retire the tracking identifier on state 5, 7 or 0, or when the identifier disappears (§8.11.3).
- Carry, per identifier: the previous state, the previous timestamp, the previous position in
  millimetres if velocity is being derived, and the last frame in which it was seen.

#### 8.11.5 The classification and grouping codes

Bytes 2 and 3 of the common record are two further small identity-like fields.

- **Classification code** (byte 2). A small enumeration, not a plain index. Generation 7 transmits
  three bits and re-maps the value 7 to **12**; generation 8 selects from the set {1, 2, 6, 12};
  generations 3, 4, 5 and 9 transmit a raw 4-bit value; generation 10 fixes it at 2. The recurring
  value **12** is clearly a distinguished code, but **the enumeration is not established** and no
  meaning should be attached to individual values. **[U]**
- **Grouping code** (byte 3), signed. It is a per-generation constant rather than a wire field:
  **1** for generations 1, 3, 4, 5, 7 and 8; **0** for generations 9 and 10. It carries information
  only in the extended-header family, where it is a real record byte. Its meaning is not
  established. **[U]**

**There is no palm bit and no confidence bit in any contact record.** The per-contact flags word
(§8.4.3, offset 28) is the only candidate and its bit meanings are not established. Palm rejection
and resting-contact suppression are host-side concerns applied above this interface (§11), and no
part of them is negotiated with the device. **[C]**

### 8.12 Amplitude, density and reported force

Three distinct scalars travel with each contact. They are frequently confused; they are not the same
quantity and only one of them is nominally a force.

#### 8.12.1 Amplitude

The **total capacitive signal** of the contact patch. It is a 16-bit **8.8 fixed-point** value in
the common record, so its full range after conversion is **0.00 to 16.00**. It is **dimensionless** —
it is a sum of capacitive coupling, not a force. **[C]** Generations that transmit it as a 6-bit
code scaled by 32 reach only **0.00 to 7.875**, in steps of 0.125; the companded and linear
generations use the full range.

- Present in **every** generation.
- Carried as an 8-bit companded code in generations 7 and 8 (§8.4.12), as a 6-bit code scaled by 32
  in generations 1, 3, 4 and 5, and linearly in generations 9 and 10 and the extended-header family.
- It is the primary "is this contact real" discriminator: a contact is active when amplitude is
  above zero even if its state says otherwise (§8.11.2).
- **Fully meaningful on a device with no strain gauge.** **[C]**

#### 8.12.2 Density

**Amplitude normalised by contact area** — signal per unit contact radius. Also 8.8 fixed point.

- On the wire in **generations 9 and 10**, and in extended-header records of 22 bytes or more.
- **In every other generation it is not transmitted and must be derived by the host** from the
  amplitude and the two contact radii, as amplitude divided by a mean-radius term. **[C]**
- **The exact normalisation is not established** — it depends on device-supplied constants that are
  not available here, so density cannot be reproduced numerically from this document. A driver that
  needs it on a generation that does not transmit it must calibrate against a device, or use
  amplitude alone. **[U]**
- Interpretation: a firm fingertip has high density; a resting palm has high amplitude and *low*
  density. This is the classic capacitive discriminator between the two.
- Derived entirely from capacitance and geometry, so **fully meaningful with no strain gauge**.
  **[C]**

#### 8.12.3 The force-like scalar

A 16-bit value whose unit is **grams**.

- On the wire in **generation 7 only** among the small compact formats (byte 7, companded to
  0–1008 g, §8.4.12), in **generation 9** (an 11-bit linear field), and in extended-header records of
  **28 bytes or more**. **[C]**
- **Zero in every other generation** — the field is structurally absent from generations 1, 3, 4, 5,
  8 and 10 and is filled with zero. **[C]**

> **Warning. This field carries a value regardless of whether the device can measure force.**
> Its presence in a record is a property of the format, and says nothing about the device.
> On this machine the trackpad has **no strain gauge** (§8.1), so whatever
> appears here cannot be a force measurement. It can only be a capacitance-derived estimate produced
> by the touch controller, or a constant, or zero. **A driver on this machine must not advertise
> this field as force or pressure.** **[C]**

Whether this machine's touch controller populates the field at all is **not established** and needs
a capture. **[U]**

Also distinct, and not part of the contact record: the extended-header family carries a 6-byte
auxiliary block of three 16-bit values (§8.4.11). It is a frame-level quantity, not per-contact,
and no compact generation carries it. Its meaning is **not established**. **[U]**

#### 8.12.4 Recommendation

Map **amplitude** — or **density**, if a palm discriminator is wanted — onto `ABS_MT_PRESSURE`, and
document the axis as a dimensionless capacitive quantity rather than a force. Do not expose grams,
and do not expose a force axis at all on this machine, unless a capture proves the field is
populated and a calibration against known masses is available. **[C]**

---

## 9. Power management

### 9.1 Three independent state models

A driver must keep three power notions distinct.

**Per-interface power** — the state of one logical HID device, carried in the control command set.
Six values are accepted:

| Value | State |
|---|---|
| `0` | Off |
| `1` | Sleep |
| `2` | On |
| `3` | Reset |
| `4` | Pre-reset memory capture |
| `5` | Post-reset memory capture |

Transitions are constrained: **Sleep may only be entered from On**, and post-reset capture may only
be entered from Reset. Off, On, Reset and pre-reset capture may be requested from any state, and a
request equal to the current state (other than Off) has no effect. A value of 6 or above is invalid.
An implementation must reject or re-order requests that violate this.

States `4` and `5` are **not used on this machine** for the transport's own domain (§7.6), but they
remain part of the accepted enumeration.

**Coprocessor power** — the state of the MTP itself, managed over the mailbox path rather than the
in-band protocol. The meaningful states are On, Sleep, Sleep-without-memory-retention, and Off.
This layer is shared with other coprocessors on this platform and is already understood; see
Appendix A.

**Touch device power** — the touch controller's own state, an enumeration of three values in which
`2` means fully on. Which of `0` and `1` means off and which means idle is **not established**.
**[U]**

### 9.2 Setting interface power

Two methods exist, selected per interface by the descriptor:

- **Method 1, atomic** — a single set-power command (`0x40` sub-opcode `0x01`).
- **Method 2, two-phase** — a will-change notification, then the local action, then a has-changed
  notification carrying the result status (`0x40` sub-opcode `0x02`, phase `0` then phase `1`).

An implementation must support both. **On this machine the touch interface uses method 2.** Its
descriptor also requests notification on power-state changes but not on every reset, and completes
registration asynchronously. **[C]**

Current state is read back with `0x41`. The response echoes the interface identifier and the current
state; both should be checked by the requester, and a state value of 6 or above is invalid.

**Minimum off-time.** A configurable minimum interval is enforced between powering an interface off
and powering it back on. The value is **supplied by the interface descriptor** (§4.2) and is not
available from platform data. It is a different constraint from the 5 ms device-requested toggle of
§9.4, and both apply. A driver must record the time of every power-off and honour the interval on
the next power-on. **[C]**

Local actions accompany each transition: powering off runs the interface's power sequence in
reverse; powering on from Off runs it forward after the minimum off-time has elapsed; entering or
leaving Sleep runs the sleep sequence; entering Reset runs the reset sequence (§9.8). Powering an
interface on also re-provisions it if it has a provisioning requirement (§7.7).

### 9.3 Idle and low-power entry

The practical idle mechanism is **per-interface power state**, not a device-specific idle command.

To idle the trackpad while keeping the system responsive:

1. Place the touch interface in **Sleep** (from On, per the transition constraint).
2. Leave the transport and coprocessor running so that other interfaces continue to deliver input.

To reduce power further, the whole transport may be powered down, at the cost of losing all
interfaces and requiring re-provisioning of the touch device on the way back up (§7.7).

A separate **host operational-state hint** is available as feature report `0x71` on the touch
interface, one byte: **`0x01` = normal operation with touch active** (sent when the display comes
on), **`0x02` = host sleeping** (sent when the display goes off). This is advisory — it informs the
device of host intent rather than commanding a state. The mechanism and the values are established;
on this machine the platform data that arms it is absent, so it is unlikely to be sent here. **[C]**

The host also announces system-level sleep and wake with `0xC1` (`1` = asleep, `2` = awake). This
should be sent on every system power transition, and is the first act of the startup handshake.

### 9.4 Device-initiated power cycling

The device may ask the host to power-cycle an interface (`0x43`). The required host response is an
off/on cycle with a **minimum off-time of 5 milliseconds**. This is a mandated timing parameter;
shorter cycles are not known to be safe. **[C]**

### 9.5 Wake

**Wake sources are the transport's own interrupts** — the byte-channel receive interrupt and the
coprocessor **outbox-not-empty** interrupt are both registered as wake sources at the platform
level. **[C]**

Notably, the MTP's GPIO block declares **no wake events**: the property exists but is empty. Wake
does not arrive over a dedicated touch interrupt line, and a driver must not wait on a GPIO for
wake. **[C]**

The system power controller separately declares touch and keyboard wake bits. That these bits are
what wakes a sleeping system before the transport is running is a plausible reading of how the
pieces fit together, but the routing is **not established** — nothing in the interface described
here consumes them by name. **[U]**

On wake, the device sends a **wake report** (`0xF2`) of at least 10 bytes, carrying the interface
identifier that caused the wake and an 8-byte timestamp (see the timestamp caveat in §6.1). A driver
should use this to attribute the wake and to decide what to restore first.

### 9.6 Sleep and hibernation

Two depths must be distinguished.

**System sleep.** The transport and coprocessor may retain state. On resume the host re-announces
awake and resumes traffic. **The shared-memory rings must be drained on resume** — the coprocessor
buffers input reports while the host is asleep, and because the rings have no interrupt (§3.7) a
driver that does not drain them explicitly will appear to work and then stall.

**Hibernation is a full teardown and cold re-enumeration, not a save/restore.** On the way down the
transport is torn down; on the way up a completely new transport instance is created, interfaces are
rediscovered, and the handshake runs again from the beginning. **Touch firmware is always
re-provisioned.** **[C]**

No *device* state survives. Two pieces of *host* state are deliberately carried across and it is
sensible to do the same: the address-translation and shared-memory mapping context, and the cached
firmware images (which is also required by §7.7). **[P]**

Platform data marks the coprocessor as requiring a cold boot after hibernation and as participating
in a managed sleep handshake. The address-translation unit is marked as not sleeping, so its
mappings are neither torn down nor restored across a suspend.

### 9.7 Clock and power domains

The MTP block owns a tree of internal clock and power domains, all fed from an always-on rail and
sequenced by the coprocessor itself.

**The platform description declares no host-operable power gates or clock gates for this block, and
no clock or frequency control for it is exposed anywhere in this interface.** A driver must not
attempt to gate MTP clocks or domains; the host's only levers are the coprocessor mailbox and the
per-interface power commands. **[C]** The stronger statement that the domains are not host-operable
*at the silicon level* rests on a reading of platform flags that is not verified against hardware.
**[P]**

This is a meaningful simplification relative to coprocessors where the host owns the gating.

### 9.8 Touch device power sequencing

The touch device's power-sequence control and analog-front-end reset are routed through the
**system power-management controller**, not through host GPIOs (§6.4). They are driven as named
sequences: applied forward on assert and in reverse on deassert.

On this machine, for the touch interface:

- The **power sequence** consists of a single analog-front-end reset operation.
- The **reset sequence** consists of a single platform operation whose disable step
  asserts one level with **no delay**, and whose enable step asserts the other level followed by a
  **100 000 microsecond (100 ms) delay**. **[C]**
- The **minimum off-time** and any further per-step delays come from the interface descriptor
  (§4.2, §9.2) and cannot be stated here.

A driver must be prepared to service device-originated requests for these operations (`0xB5`,
§6.4) rather than performing them directly on GPIO lines.

### 9.9 Telemetry

Two independent telemetry surfaces exist, and their status on this machine differs. The residency
reports below are switched off by host configuration here and may be treated as optional. The
periodic counter reports further below are **enabled** and are the telemetry path that actually
runs on this machine.

**Residency reporting on the touch interface.** Three feature reports:

| ID | Direction | Function |
|---|---|---|
| `0x73` | Get | Residency descriptor: names the available channels and states |
| `0x72` | Get, then Set | Residency data. **Read-and-clear**: after reading, the host writes the report back zeroed |
| `0x71` | Set | Host operational-state hint (§9.3) |

*Descriptor (`0x73`)*: a 1-byte version which must be `1`; a 1-byte reporter count which must be
non-zero; then, per reporter, a 1-byte subgroup index (0 or 1), a 1-byte channel index (0 or 1), a
1-byte tick-rate index (0 to 2), a 1-byte flags field, a 1-byte state count which must be non-zero,
and that many 1-byte state indices (each 0 to 5). The tick-rate index selects from **2048, 32768 and
1000 ticks per second**.

*Data (`0x72`)*: for each reporter, for each of its states except a synthesised final one, a 4-byte
tick count followed by a 4-byte entry count. Residency in microseconds is
`ticks × 1 000 000 / ticks_per_second`. When a reporter's flags field is zero, the final state's
residency is not reported and must be derived host-side from elapsed wall-clock time minus the sum
of the reported states, with a tolerance of the order of 5000 microseconds.

The channel and state identities are supplied by the descriptor at runtime and must not be
hard-coded. On this machine there are two channels and six states, all of them residency counters.
Buffer 520 bytes for both reports.

**Periodic counter reports per interface.** Separately from the above, an interface's configuration
may declare a **periodic counter report** that the host polls and exposes as telemetry: a report
identifier, a report length, and a channel count. On this machine:

| Interface | Report | Length | Contents |
|---|---|---|---|
| Touch (1) | `0x7E` | 129 bytes | 16 counters of 8 bytes each, in milliseconds |
| Management/telemetry (2) | `0xEA` | 855 bytes | 197 counters |

These identifiers are **not fixed by the interface** — they come from configuration, and a driver
should read them rather than hard-code the values above. Whether a given report continues to be
polled while the interface sleeps is also a per-interface configuration item. **[C]**

**There is no thermal or current telemetry exposed through this interface.** Temperature and power
draw are not reported by this device on any of the paths described here. **[C]**

### 9.10 Watchdog

The coprocessor is supervised by a **periodic host-originated liveness exchange on the mailbox
path**, not by anything in the in-band protocol. The timeout is a platform-supplied per-processor
value; **20 seconds** is the default and no other value is declared for this machine. **[P]**
The host must:

- Quiesce the watchdog before entering sleep, blocking until any outstanding exchange completes.
- Re-arm it on wake, resetting the sequence counters.

Failure to quiesce before sleep will produce spurious watchdog resets. **[C]**

Separately, the in-band control timeout of 5 seconds (§5.3) is the practical hang detector for the
transport itself: no other liveness timer applies to this transport type, and the protocol carries
no heartbeat (§10.4). **[C]**

---

## 10. Reset and recovery

This section addresses a procedure that existing open-source support explicitly lacks, which
prevents both bootloader-to-OS handoff and runtime re-probing.

### 10.1 Reset scopes

| Scope | Mechanism |
|---|---|
| Single interface, in-band | Reset command `0x42` with the interface identifier |
| Single interface, power-cycle | Off → 5 ms → On, per §9.4 |
| Whole transport | Tear down and re-create the transport, then re-run discovery and the handshake |
| Coprocessor | Mailbox-level stop and start |

Which per-interface method applies is selected by the same descriptor property that selects the
power method (§9.2): method 1 uses a power transition followed by the `0x42` command, method 2 uses
a plain off/on power cycle. **This machine's touch interface uses method 2, so its per-interface
reset is a power cycle and not command `0x42`.** **[C]**

One description has the method-2 power cycle carried by report `0x41` rather than `0x40`; every
other has `0x40` as the set-power command and `0x41` as the read-back. Use `0x40`. **[P]**

There is **no host-side hardware reset line for the transport itself**; a transport reset is a
teardown and re-create, not a line toggle. **[C]**

### 10.2 Device-initiated reset

The device may request its own reset (`0xA2`, carrying the interface identifier) or a power cycle
(`0x43`). Both are advisory requests that the host executes asynchronously.

These requests **must be rate-limited**. Once the consecutive-failure ceiling for an interface is
reached, the host must stop honouring them (§7.6). An independent implementation without this guard
is vulnerable to a reset loop.

### 10.3 What must be rebuilt after a reset

After any reset that reaches the transport, all of the following must be re-established. Only the
first two have a required relative order; the rest follow from it.

1. The transport handshake runs again — announce awake (`0xC1`), then register every shared-memory
   ring (`0x91`). Ring indices start from zero.
2. Interfaces are rediscovered and their descriptors re-fetched (`0xF0`), terminated per interface
   with `0xB4`. Whether a full re-enumeration is required on every reset, or only when the device
   pushes a descriptor, is not settled; re-enumerating is the safe choice. **[P]**
3. Calibration blobs and configuration are re-pushed with the descriptor-supplied configuration
   report (§6.2).
4. The touch device is **re-provisioned** (§7.7), including the interface power cycle of §7.5.
5. Any shared-memory buffers (`0x95`) are re-registered — the device's view of them is lost.
6. The device signals readiness per interface (`0xF1`) before traffic may resume.
7. The shared-memory rings are drained (§3.7, §9.6).
8. Cached device properties are re-read; the device signals this need explicitly with status value
   `0x02` inside report `0x60` (§8.5).

After a single-interface reset, only that interface's portion of the above applies.

**The MTP's own program is not reloaded by the host on any of these paths** — subject to the
unresolved question of where that program comes from in the first place (§7.1, §12.2).

### 10.4 Detecting a hung device

There is no heartbeat on the in-band protocol. Detection relies on:

- The 5-second control-transfer timeout.
- The coprocessor watchdog (§9.10).
- Status frames reporting that a reset has already occurred (`0x60`, values `0x10`–`0x13`).

A driver should treat a control timeout as a device fault and escalate to transport reset, since the
protocol offers no lesser recovery (§3.6).

### 10.5 Host fault notification

The device can ask to be told when the **host** has failed. The interface descriptor may supply a
register address and a 32-bit value; on a host panic the host performs a single 32-bit write of that
value to that address, telling the coprocessor the host is gone. The mechanism is established; the
address and value for this machine are **not known**, because they are supplied at runtime rather
than by platform data. A driver should implement the write if the descriptor supplies the pair, and
otherwise do nothing. **[U]**

---

## 11. How a host driver uses this interface

This section describes the **shape** of host-side responsibility, at the level of concerns and
division of labour. It is deliberately not a procedure and not an implementation description.

**The host is a service provider to the coprocessor, not merely its master.** The unusual property
of this interface is that traffic is genuinely bidirectional in initiative. The device asks the host
to reset it, to power-cycle it, to perform platform operations it cannot reach, and to supply
configuration on demand. A host implementation that models the device purely as a target it
commands will fail to bring the hardware up, because parts of the bring-up depend on the host
answering the device's requests. Servicing device-originated requests is a first-class
responsibility, not an error path. Note the converse as well: the host must answer *only* the
requests the device is waiting on (§6.2, `0xA1`).

**Transport ownership is separate from device ownership.** One component owns the byte channel,
framing and multiplexing; other components own individual logical devices. This separation matters
because the transport must remain functional while individual interfaces are reset, powered down or
re-provisioned independently. Collapsing the two makes per-interface recovery impossible.

**Almost everything that shapes behaviour arrives at runtime.** Report lengths, allowlists, power
method, minimum off-time, power and reset sequences, failure ceilings, calibration report
identifiers and telemetry report identifiers all come from the interface descriptor, not from
platform data or from this document. A driver structured around compile-time constants will have to
be restructured; one that treats the descriptor as the source of truth will not.

**Provisioning is a data-driven concern, not a code concern.** Because the touch controller is
programmed by a declarative script that the coprocessor interprets, the host's role is to hold that
image, hand it over, trigger execution with a power cycle, and observe the outcome. The host does
not need to understand the touch controller's memory map, and should not encode knowledge of it. The
image is versioned platform data and belongs alongside other firmware assets rather than inside
driver logic.

**State is not durable across power transitions.** The touch device retains nothing meaningful
across a deep power transition, so every path that can lose power converges on the same
re-establishment work. Treating resume, reset and initial start as one shared path is simpler and
less error-prone than special-casing each, and matches what the interface actually requires.

**Failure handling is coarse by design.** The framing layer offers no retransmission or
resynchronisation, and the request layer offers no retry. Consequently the host's error strategy is
necessarily escalation to a reset scope rather than in-place recovery. The corresponding risk is a
reset loop, so a failure ceiling that stops honouring device-initiated reset requests is a necessary
part of a correct implementation, not a refinement.

**Policy belongs above this interface.** The device exposes no behavioural tuning surface — no
sensitivity, palm-rejection, orientation or rate controls. Gesture interpretation, palm rejection and
pointer acceleration are host-side concerns applied to decoded contact data, and none of them are
negotiated with the device. An implementation should not search this interface for behaviour that is
not there.

**Contact decoding is the host's job, and it is specified here.** The interface delivers touch
frames and interprets nothing above the header; §8.4 gives the record layout for every format the
interface can carry, §8.10 the units and scaling, §8.11 the identity and lifecycle model, and §8.12
the amplitude and force semantics. Three properties of that work shape a driver's structure:

- **Dispatch on the report identifier, once, at the top.** Every other discriminator on offer is
  either absent or wrong (§8.4.1, §8.4.13).
- **A significant part of the contact model is host-derived, not received.** Velocity for six of the
  nine generations, density for most of them, the end of a touch when a contact simply stops being
  reported, and the tracking identifier itself are all things the host must produce. A driver that
  models the device as the sole source of contact state will emit touches that never end.
- **Decode into one common record and convert once.** Every generation is a compressed encoding of
  the same field set (§8.4.3), so per-generation code should end at that record and the units,
  clamping and normalisation should be shared.

---

## 12. Confidence summary and open questions

### 12.1 Well established

The framing layout, checksum and status encoding; the get/set encoding; the control command set and
the payload layouts given in §6; the interface and discovery model and the descriptor blob format;
the provisioning model, encoding profile, container format and handoff sequence; the touch header
formats; the touch feature-report set with field layouts; the status and error frame encodings; the
power state models, transitions and wire commands; the ring layout, index arithmetic and the absence
of a ring doorbell; the reset scopes and the re-establishment requirements; the mechanical
(non-force, non-haptic) nature of the trackpad.

Added to that list by the contact work: the report-identifier dispatch rule, and the fact that
nothing else selects a format (§8.4.1); the bit-level record layout of every contact format and the
extended-header field table (§8.4.3 to §8.4.11); the companding curves and their check values
(§8.4.12); the positional unit, axis origins, per-format biases and ranges (§8.10.1, §8.10.2); the
radius, orientation, velocity and timestamp units (§8.10.4 to §8.10.6); the identifier's usable
range and the eight lifecycle states with the two host-derived transitions (§8.11); and the
distinction between amplitude, density and the force-like scalar (§8.12).

Three caveats on that list. The `0xA1` configuration response is **not** settled (§6.2); several
descriptor-supplied values that the model depends on can only be read from a live device (§4.2); and
the contact material is established **per report identifier**, which is not the same as knowing
which identifier this machine emits (§8.1, §8.4.14).

### 12.2 Open questions, in priority order

The order is by what blocks a driver first, not by intellectual interest. Items 1 to 5 must be
answered before a device will ever produce a touch frame; item 6 must be answered before a frame can
be parsed at all; items 7 onward refine what is done with the frames once they are parsed.

Several questions that appeared in earlier revisions of this list have been answered and are gone:
the per-contact record layout, the coordinate and surface units, the touch timestamp tick, the
pressure semantics, the contact array start offsets, and the previously undecoded remainder of the
extended header. They are now specified in §8.4, §8.10, §8.11 and §8.12.

**Tier 1 — blocks bring-up.**

1. **Capture the descriptor exchange (`0xF0` set then get) for every interface.** This single
   capture yields the true interface list, the HID report descriptors, the power method, the power,
   sleep and reset sequences, the minimum off-time, the maximum report lengths, the allowlists, the
   calibration configuration entry and its report identifier, the periodic counter report
   identifiers, and the initial-report value. It also settles the descriptor header layout (§4.2)
   and, as a by-product, whether a get response includes the report-identifier byte (§5.1). It
   answers more open questions than everything below it combined and should be done first.
2. **The interface population, and whether the keyboard is behind this transport at all.** The
   coprocessor's configuration declares three interfaces and no keyboard, while platform data
   declares a keyboard child of the transport (§2). The available-interfaces list from item 1
   settles it. This gates the whole discovery model.
3. **Where the coprocessor's own program comes from, and whether the host must supply it** (§7.1).
   If the host must load it, a Linux driver has no boot path for the MTP until this is answered. If
   it does not, the driver's start-up is much simpler. Nothing about the touch path can be tested
   until the coprocessor runs.
4. **The provisioning trigger sequence.** Capture a full bring-up: the `0x95` use-type 2
   registration, the interface power cycle, and the `0xF1` that follows — including which order the
   first two occur in (§7.5).
5. **The `0xA1` configuration response: shape and which requests get one** (§6.2). Servicing `0xA0`
   is mandatory for bring-up, and answering a request the device is not waiting on is itself a
   fault. Capture the `0xA0` traffic during bring-up and observe which requests are answered, and
   with what.

**Tier 2 — blocks correct decoding of a frame that has arrived.**

6. **Which input report identifier this trackpad emits** (§8.2, §8.4.14). This is now the *only*
   thing standing between a captured frame and a fully specified parse, which promotes it well above
   its previous rank. The candidates are `0x31` and `0x75`, possibly wrapped in `0x02`. Their
   headers are 4 and 32 bytes and their records 9 and 20–30 bytes, so a single captured frame of
   known length with one finger on the surface resolves it. Nothing short of a capture can.
7. **The surface bounds actually reported by this device** — the four signed values of feature
   `0xD9` (§8.7, §8.10.3), plus the sensor rows and columns and the family identifier. Without them
   a driver cannot set its axis limits or normalise, and it must fall back on defaults that are not
   this machine's. Read them once at bring-up. Two small ambiguities in that payload are settled by
   the same capture: the ordering of the four bounds, and whether its length is exactly 16 or at
   least 16. Until then, a driver should validate that each minimum is below its maximum and swap
   the pair if not.
8. **The exact density normalisation** (§8.12.2). It depends on device-supplied constants that are
   not available here, so for any generation that does not transmit density it cannot be reproduced
   numerically. Only its general form — amplitude over a mean-radius term — is known. Calibration
   against a device would settle it.
9. **Whether this machine's touch controller populates the force-like byte at all** (§8.12.3).
    The field's presence is a property of the record format rather than of the device, so a non-zero
    value proves nothing about the hardware; a capture with known applied loads is the only way to
    learn whether it carries information. Until then the field must not be exposed.
10. **The physical direction of +Y** — toward the user or away (§8.10.1). One drag with a known
    direction settles it. Everything else about the axes is established.

**Tier 3 — needed for a working, correct driver.**

11. **The semantics of the configuration request's action and resource values** (§6.1), and of the
    8-character operation identifiers carried by `0xB5` (§6.4). The transport carries them; their
    meaning is device-defined and must be recovered by tracing.
12. **Meanings of the six transport status classes `0x80`–`0x85`, and of other non-zero values in
    that byte** (§3.3). A driver can treat them all as failures, but cannot distinguish
    retry-worthy from fatal without them.
13. **The per-contact flags word, bit by bit** (§8.4.3, offset 28). Different generations set
    different bits of the same word, so it is a per-format bitfield rather than one shared
    enumeration. This is the most likely home for a palm or confidence indication, and no such
    indication is identified anywhere else (§8.11.5).
14. **The classification-code enumeration** (§8.11.5) — in particular the recurring distinguished
    value 12 and generation 8's set {1, 2, 6, 12}. The field is plainly an enumeration and not a
    plain index, but its members are not named anywhere available.
15. **Whether the device guarantees any delay before reusing a contact identifier** (§8.11.1). No
    such guarantee is established; a driver must be defensive, and knowing the real policy would let
    it be less so.
16. **Timestamp epoch, and the 8-byte management timestamps.** The touch-frame tick is settled
    (§8.10.6) but the epoch is a free-running counter with no defined zero. Separately, the tick
    rate and epoch of the 8-byte timestamps in `0xF2`, `0xA0` and `0xB5` remain undefined (§6.1).

**Tier 4 — refinements and small unknowns.**

17. **The generation-10 Y-velocity anomaly** (§8.4.10). That the device places the field at record
    bytes 6 and 7 is inferred from the regularity of the rest of the record, not established. A
    capture of a generation-10 device would settle it — but no such device is expected on this
    machine.
18. **The generation-4-compact spare 6-bit field** (§8.4.6, record byte 3 bits 7:2). It is what
    makes that record nine bytes rather than eight, so it is populated, but nothing interprets it.
19. **The second density slot and the second orientation angle** (§8.4.3, offsets 22 and 24). Both
    are real wire fields in the larger extended-header records and neither has an identified
    consumer or meaning.
20. **Two undetermined header fields** — 6 bits in the generation-9 header and 32 bits in the
    extended header (§8.4.9, §8.4.11). Both are real wire fields; neither's purpose is established.
21. **The undetermined header bytes of generations 9, 10 and the extended-header family** (§8.4.9
    byte 6; §8.4.10 bytes 5–12, 14 and 16; §8.4.11 bytes 8, 9, 11 and 20–21).
22. **The meaning of the compact header's two flag bits** (§8.3, byte 1 bits 1 and 2). Two readings
    of bit 1 exist — a surface-orientation indication or an unrelated spare — and bit 2 is a
    level-triggered state whose meaning is unresolved.
23. **The semantics of transport header bytes 6 and 7 on their respective message types** (§3.3) —
    and of application header byte 1 (§3.4).
24. **Shared-memory use-types `0` and `1`** (§6.2), which are unassigned as far as this interface
    goes.
25. **Which of touch-device power values `0` and `1` is off and which is idle** (§9.1).
26. **Meanings of the individual critical-error bits** (§8.6) and the internal structure of the
    opaque region-parameter blob `0xA1` (§8.7).
27. **The value semantics of the orientation feature reports `0xDC` and `0xDD`** (§8.7, §8.10.1) —
    which value means which rotation, and what the "mode" report selects.
28. **The older `0x43`/`0x44`/`0x45` and `0x73`/`0x74` frame families** (§8.4.2), whose headers and
    contact records are not specified here. They are in this interface's accepted report set, so a
    device could in principle emit one.
29. **The host-fault register address and value** for this machine (§10.5).
30. **Whether a device-side raw-sensor diagnostic mode exists.** Nothing in this interface exposes
    one; that is not the same as the device not having one (§8.8).
31. **The transport-multiplexer node's role**, and whether ownership of the touch stream can move to
    another processor while the host sleeps (§2). If it can, ownership handover becomes a driver
    concern.

### 12.3 Notable negative results

These are stated explicitly because they save implementation effort. Each is scoped to what is
actually supported.

- **No host-driven SPI boot interface is used on this machine.** Such an interface exists in this
  family of devices; this machine selects the declarative-script path instead (§1.2, §7).
- **No enable/disable or general tuning command for touch reporting is reachable through this
  interface** (§8.8). Enabling touch is a power operation.
- **No GPIO wake line for the touch device.** The wake-events property on the relevant GPIO block is
  present and empty (§9.5).
- **No host-operable MTP clock or power gates are declared by the platform** (§9.7).
- **No thermal or current telemetry is exposed through this interface** (§9.9).
- **No separate sensor-microcontroller interface on this machine** (§2) — stated alongside the
  unresolved keyboard-attachment question of §12.2.
- **No acknowledgement, retransmission or resynchronisation in the framing layer** (§3.6).
- **No interrupt or doorbell for the shared-memory rings**; they are polled after every byte-channel
  read completion (§3.7).
- **No host-to-device shared-memory ring** is created; bulk host-to-device transfers use one-shot
  registered buffers instead (§3.7, §5.4).
- **No protocol-wide version handshake** exists; versioning is per message (§6.5).
- **No transport-multiplexer consumer exists in the interface described here.** This is *not* the
  same as the node being unused: it is declared in platform data, and its consumer — if any — lies
  outside this interface (§2, §12.2).
- **No device property, capability bit or version field selects the contact format.** The report
  identifier is the only selector, and a property that appears to name a parser is never consulted
  (§8.4.1). A generation number is likewise not a usable key (§8.4.13).
- **No palm bit, resting bit or confidence bit exists in any contact record** (§8.11.5). Palm
  rejection is entirely a host concern and is not negotiated with the device.
- **No frame sequence number is carried on the wire by any compact contact format** (§8.10.6). Only
  the extended-header family has one; a host that needs a frame counter must synthesise it.
- **No contact format carries a strain-gauge force reading.** Where a force-like field exists it is
  filled in without regard to whether the device supports force, so on this machine it carries no
  force information (§8.12.3).
- **No terminating record is guaranteed at the end of a touch.** A contact can simply stop appearing;
  the end of touch must be synthesised (§8.11.3).

---

## Appendix A. Relationship to existing open-source knowledge

Existing open-source support for this coprocessor family on earlier machines already covers, and
this document does not restate:

the byte-channel hardware layer and its register interface; the coprocessor mailbox and its boot
sequence; the outer and inner header field layouts; the control and input message-type values; the
checksum algorithm; the interface model; the descriptor, ready and platform-operation event kinds;
the reset, enable, provisioning-handoff and platform-operation acknowledgement commands; the general
request/response and shared-buffer approach.

**Where this document corrects existing knowledge:**

- The maximum interface count is **32**, not 16. Existing support's limit is too low for this
  device.
- A single byte channel serves this path, and its number is 0.

**What this document adds for this machine:**

- The device population differs: three declared interfaces, with no separate sensor-microcontroller
  interface, and an unresolved question about where the keyboard attaches.
- The provisioning path is a declarative script in a constrained CBOR profile, replacing the
  earlier container-and-transfer approach. The string-length and byte-string-padding rules in
  §7.3, the container tag reversal and the non-standard serialised dialect in §7.4 are
  interoperability-critical.
- The provisioning handoff requires an interface power cycle to trigger execution.
- Platform operations route through the system power controller rather than host GPIOs.
- The shared-memory ring path: layout, index arithmetic, direction, and the absence of a doorbell.
- A complete power-management model, which existing support does not cover.
- A reset and recovery model, including what must be rebuilt — the specific gap that currently
  prevents handoff and re-probing.
- The touch device is mechanical-switch, without force or haptics, unlike every other machine in
  this family.
- The touch feature-report surface, including identity, geometry, status and telemetry.
- The transport status-class encoding and the application-header status semantics — meaning for
  header bytes that existing support treats as padding or unknown.
- The per-message versioning model and forward-compatibility rules.

**What the contact-format material adds beyond existing open-source knowledge.**

The open-source transport support for this coprocessor family **decodes no contact data at all**: its
touch path ends when the HID input report is delivered, and the contact array is passed on
uninterpreted. Everything in §8.4, §8.10, §8.11 and §8.12 is therefore new relative to it.

A separate and partial overlap does exist elsewhere in open-source: mainline Linux decodes an 8- or
9-byte contact record for externally connected Apple trackpads, which corresponds to generations 3, 5
and one of the 9-byte generations here. That code covers a fraction of what this section specifies
and does not state the surrounding model. Specifically, none of the following is available from it:

- the report-identifier dispatch table across **all** contact formats, and the explicit rule that
  nothing else selects a format (§8.4.1, §8.4.2);
- the two traps in §8.4.13 — the two unrelated formats sharing generation 4, and the non-injective
  generation number;
- the exact companding curves with check values (§8.4.12), rather than approximations;
- the coordinate origin convention, the per-generation biases and the unit statement (§8.10.1,
  §8.10.2);
- the normalisation rectangle and where it comes from (§8.10.3);
- the timestamp widths, ticks and wrap-extension rule per generation (§8.10.6);
- the full eight-state lifecycle, the two host-derived transitions, and the rule for minting a
  stable tracking identifier (§8.11);
- the three-way distinction between amplitude, density and the force-like scalar, and the warning
  that the last of these is populated regardless of force support (§8.12).

Where this document and that code overlap they should be cross-checked against each other; **this
document has not been reconciled against it**, and a disagreement should be resolved by capture
rather than by assuming either is right. **[P]**
