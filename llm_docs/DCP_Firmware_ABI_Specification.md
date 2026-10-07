# Display CoProcessor (DCP) Firmware ABI Specification

**Silicon:** Apple `t8132` (M4-class SoC; kernel identifier *mac16g*) and `t8140`
(A18 Pro-class SoC; kernel identifier *mac17p*, board *j700ap*).
**Host OS baseline:** macOS 26.6 (build 25G72).
**DCP firmware baseline:** `AppleDCP-1041.120.7~2508` (RELEASE) over Apple RTKit release
`3255.160.4`. Both silicon variants carry the identical firmware and RTKit release; the
DCP RPC application ABI is identical across them except where explicitly noted in §12.

---

## 1. Introduction

### 1.1 Scope

This document specifies the functional Application Binary Interface (ABI) exposed by the
Display CoProcessor (DCP) firmware to a host Application Processor (AP). The DCP is a
dedicated coprocessor inside the display engine that runs its own real-time operating
system (RTKit) and executes the majority of the display-driver logic — DisplayPort link
training, memory-bandwidth arithmetic, mode enumeration, HDMI conversion, color and
brightness pipelines, and content protection — on behalf of the host. The host and the DCP
communicate primarily by remote procedure call (RPC): each side invokes methods that are
serviced by objects resident on the other side.

The specification covers, exhaustively:

1. The transport layer — the coprocessor mailbox, endpoints, channels, and shared memory
   (§3).
2. The RPC method interface — the call primitive, directional pointers, and argument
   serialization (§4), followed by the full method enumeration for both directions
   (§5, §6) and the reentrancy model (§7).
3. The four key-value / parameter mechanisms (§8).
4. The peripheral display sub-services: DisplayPort transmitter, DP/AV, HDMI, output
   crossbar/mux, HDCP, backlight, and timing controller (§9).
5. Data structures carried across the interface (§10).
6. Silicon-specific differences and the firmware container formats (§11, §12).

### 1.2 Naming and notation

Interface elements are named by **observed function**. Where an element has a stable
wire-level identifier, that identifier is authoritative and is given verbatim:

- **Method tags** are four-character (FourCC) selectors carried in the RPC transport. Tags
  beginning `A` denote host→DCP calls; tags beginning `D` denote DCP→host callbacks. Tag
  values are quoted, e.g. `A407`.
- **Service names** are the literal ASCII registration strings used by the endpoint service
  layer, e.g. `dcpav-controller`.
- **Property keys** (parameter-store keys, host-property keys, published registry keys, and
  runtime-property keys) are literal ASCII strings and are quoted verbatim, because they are
  part of the wire interface.

Functional method, structure, and field names are descriptive and are not tied to any host
driver's internal identifiers.

### 1.3 Confidence annotations

Every non-trivial claim is annotated:

- **[Confirmed]** — directly evidenced in the analyzed firmware/host images for this build.
- **[Inferred]** — derived from strong indirect evidence.
- **[Uncertain]** — plausible but weakly evidenced.

Numeric values (tags, IDs, sizes, indices) are provided only where recoverable; where a
value is an ordinal inference rather than a directly recovered constant, it is marked
accordingly. No numeric value in this document is invented.

---

## 2. Architecture overview

The DCP presents itself to the host as an Apple Silicon Coprocessor (ASC): a mailbox-driven
core running RTKit. Two application transports are layered over the one mailbox:

```
  Application services
    ├─ DCP RPC "link"            (one endpoint) ─ framebuffer, display pipeline,
    │                                             power, property/service/memory relays
    └─ AFK + EPIC services       (many endpoints) ─ DisplayPort/AV/HDMI sink services,
    │                                               DPTX ports, HDCP, CEC, backlight-control
    │                                               bus, timing controller, power logging
    │
  RTKit endpoint layer          (management + system services:
    │                            syslog, crashlog, kdebug, ioreport, oslog, tracekit)
    │
  ASC mailbox                   (hardware doorbell + one 64-bit data word + endpoint tag)
    │
  Shared memory                 (DART/IOMMU-mapped device-virtual memory:
                                 RPC payload heap, on-demand mapped buffers, AFK rings)
```

Two distinct RPC families ride this stack:

- The **DCP RPC "link"** is a bidirectional, reentrant, name-dispatched RPC that carries the
  framebuffer, display-pipeline, power, and host-relay method surfaces. It uses the flat
  FourCC-tagged Call primitive (§4) and the stream/channel model (§3.3). This is the
  interface described by §4–§8.
- The **AFK + EPIC** family carries the DisplayPort/AV/HDMI/HDCP/CEC/backlight-control
  services (§9). AFK provides ring-buffer messaging endpoints; EPIC multiplexes named
  services onto an endpoint and provides announce/open/close/command/report/response
  semantics.

Both silicon variants implement the same architecture and the same RTKit and firmware
releases; the differences are confined to the set of application endpoints exposed, the
per-silicon display-pipe method numbering, the firmware container format, and a trusted-IPC
layer present only on `t8140` (§12).
---

## 3. Transport layer

### 3.1 The 64-bit mailbox message

The physical transport is the standard ASC doorbell mailbox. Each transfer carries a single
**64-bit data word** plus an **endpoint tag** (a small integer) that routes the word to the
registered handler for that endpoint. RTKit multiplexes many logical endpoints over the one
hardware mailbox; the endpoint tag associates each 64-bit word with an endpoint. Each side
exposes a primitive to enqueue one 64-bit word and to dequeue one 64-bit word. [Confirmed]

The 64 bits are **not** interpreted uniformly; the owning endpoint defines the meaning of
the word. Two grammars are relevant:

**(a) RTKit management / system-service words** — a small opcode in the low bits with inline
arguments in the upper bits (the conventional RTKit management grammar). Receivers validate a
message *type* sub-field and a *sequence* sub-field and reject unknown values. [Inferred]

**(b) DCP RPC "link" words** — the grammar used by the framebuffer/pipeline RPC endpoint. The
word packs an inline header in the low half and a shared-memory payload descriptor in the
upper half:

| Bits    | Field                | Meaning | Confidence |
|---------|----------------------|---------|------------|
| [15:0]  | header               | Combined message-class / stream-index / direction selector (below) | [Confirmed] |
| within [15:0] | message class  | A masked flag group distinguishing command vs. reply/ack classes; a word not matching a valid data-message class is rejected | [Confirmed] |
| within [15:0] | stream index   | Selects one of a small pool of concurrent stream control blocks (§3.3); out-of-range values are rejected | [Confirmed] |
| within [15:0] | direction/context bit | Distinguishes the two ends' ownership of a stream; compared against the endpoint's stored role | [Inferred] |
| [47:16] | payload descriptor   | 32-bit offset/length descriptor addressing this stream's slice of shared memory | [Confirmed] |
| [63:48] | trailing tag         | 16-bit auxiliary tag stored with the stream state | [Inferred] |

Malformed words are rejected with distinct internal error classes: bad message class, bad
stream index (out-of-range / no matching stream), bad sequence, and unknown message type.
[Confirmed] The exact bit boundaries within the low 16 bits are validated by combined masks
rather than discrete field extracts; the partition into {class, stream-index, direction} is
directly evidenced, the precise widths are [Inferred].

### 3.2 Endpoints

Endpoints fall into three groups: a **management** endpoint, a set of **RTKit system-service**
endpoints, and the **application** endpoints (the DCP RPC link and the AFK/EPIC services).
At boot the management endpoint enumerates and starts endpoints; low system endpoints use the
fixed RTKit-wide convention, while application endpoints are enumerated by name. Every endpoint
carries a human-readable name of the form `<role>-ep`; each EPIC service additionally carries a
name of the form `<role>-epic`. [Confirmed for names; assignment mechanism Inferred]

Endpoints are discovered by name in these images rather than by a code-embedded tag constant,
so the numeric tags below follow the established RTKit-wide convention and are marked
[Inferred]; the endpoint *names* are confirmed in-image. The DCP RPC "link" application endpoint
uses tag **`0x37`** [Inferred] (the established value; not independently derivable as a code
constant from these images).

| Tag (hex) | Endpoint (name in image) | Role | Direction | Confidence |
|-----------|--------------------------|------|-----------|------------|
| 0x00 | `management`/`system` | Endpoint enumeration/start; AP↔DCP power-state negotiation; boot handshake | AP↔DCP | ID [Inferred], name [Confirmed] |
| 0x01 | `crashlog` | Crash-region announce + crash-dump delivery | DCP→AP | ID [Inferred], name [Confirmed] |
| 0x02 | `syslog` | RTKit text log stream | DCP→AP | ID [Inferred], name [Confirmed] |
| 0x03 | `kdebug` | Trace-event stream (capacity/mask/threshold tunables) | DCP→AP | ID [Inferred], name [Confirmed] |
| 0x04 | `report` (ioreport) | IOReporting counter channels | AP↔DCP | ID [Inferred], name [Confirmed] |
| ~0x06–0x08 | `oslog` | Binary os_log stream | DCP→AP | ID [Uncertain], name [Confirmed] |
| ~0x0a | `tracekit` | Ring-buffered tracing (buffer size + control tunables) | DCP→AP | ID [Uncertain], name [Confirmed] |
| **0x37** | DCP RPC "link" (serves `disp0`) | Framebuffer / pipeline / power / property/service/memory relays | AP↔DCP | ID [Inferred], role [Confirmed] |
| dynamic (by name) | `service`, `interface`, `device`, `controller`, `port`, `power`, `disp0`, `dcpexpert`, `hdcp`, `arc`, `sac`, `md` | AFK/EPIC application endpoints (§3.6–§3.7, §9) | AP↔DCP | names [Confirmed], tags dynamic [Inferred] |

**Application endpoint name inventory (both silicon):** `management`, `system`, `service`,
`interface`, `device`, `controller`, `port`, `power`, `disp0`, `dcpexpert`, `hdcp`, `arc`,
`sac`, `md`, `report`, `syslog`, `crashlog`, `kdebug`, `oslog`, `tracekit`. [Confirmed]
`t8140` additionally exposes `mipi`, `mipitool` (internal MIPI/DSI panel), `tb`
(Thunderbolt/USB4 display), and `comms`, `cbservice`, `monitor`, `test`. [Confirmed]

### 3.3 Channels (streams)

The DCP RPC link is a bidirectional, reentrant RPC. Its concurrency unit is a **stream**: a
fixed-size control block held in a small pool; a message's stream-index field (§3.1) selects
one. Streams map onto the five channel classes as follows.

| Channel | Shortcode | Stream realization | Direction / semantics | Confidence |
|---------|-----------|--------------------|-----------------------|------------|
| **CMD** | `C` | Local stream, command class | Synchronous **host→DCP** call: host allocates a local stream, writes arguments to shared memory, rings the doorbell, and waits for the reply on the same stream | mechanism [Confirmed], shortcode [Inferred] |
| **CB** | `d` | Remote stream, reentrant | Synchronous **DCP→host** callback issued *while a host→DCP command is outstanding*; serviced on a nested callee path so the DCP can query the host mid-call | mechanism [Confirmed], shortcode [Inferred] |
| **ASYNC** | `a` | Async remote stream | **DCP→host** notification not tied to any outstanding command (vblank, swap-complete, hotplug, …) | mechanism [Confirmed], shortcode [Inferred] |
| **OOBCMD** | `O` | Out-of-band command stream | Host→DCP command on the out-of-band stream set, used when in-band streams are busy or for priority traffic (governed by an out-of-band threshold tunable) | OOB existence [Confirmed], shortcode [Inferred] |
| **OOBCB** | `o` | Out-of-band callback stream | DCP→host reentrant callback on the out-of-band set | [Inferred] |

Stream mechanics [Confirmed unless noted]:

- **Identity.** Each stream carries an owner thread id and a stream UID; the transport
  resolves streams by owning thread id or by UID, and tracks host-owned ("local") and
  DCP-owned ("remote") streams separately.
- **Independent execution.** Each stream has its own work loop, command gate, and event source
  on the host side, so streams run concurrently — this is the basis of **multi-channel
  concurrency**: multiple outstanding operations proceed on different stream offsets at once.
- **Pool size.** A small fixed pool (on the order of four in-band per direction) plus the
  out-of-band set; the stream-index field is only a few bits wide. [Inferred]
- **Method selection.** Within a stream the RPC method is selected by the FourCC tag (§4.1);
  the callee resolves the handler by tag.
- **Payload framing.** Each RPC carries an input blob and an output blob; both sizes are
  bounds-checked. Payloads exceeding the inline budget are placed in shared memory (§3.4).

### 3.4 Shared memory

All bulk RPC payloads and all AFK rings live in memory mapped through the DCP's DART (its
IOMMU). The DCP-visible address is a **device-virtual address (dva)**. [Confirmed]

**RPC heap.** At initialization the host grants the DCP a shared heap; the firmware reports
the heap size and grows it from a pool. Two dva values describe the shared region (distinct
host-owned and DCP-owned mapping views). Per-call payloads are copied between caller buffers
and the shared region by shared-copy helpers, with overflow/underflow guards. A stream
"assumes" and "releases" control of its shared slice around each call. [Confirmed]

**On-demand mapped buffers.** For framebuffers/surfaces and other large blobs, the host maps
caller memory into the DCP DART on demand and passes the resulting dva in the RPC. This is
mediated by the **Memory-Descriptor Relay** (a DCP→host callback group, §6): the host keeps an
id-indexed registry of buffers/descriptors, and the DCP refers to a buffer by numeric id which
the host resolves to a dva. Operations include: allocate a buffer (type, size, alignment →
id, dva, physical), map a physical range (→ dva), resolve a descriptor from an id, and complete
(release) a buffer. DART is powered up around use. [Confirmed]

**Setup / teardown.** The RTKit "shmem setup/destroy" contract is realized through the
endpoint's memory-mapping API: the host binds a coprocessor slave-memory view at a given
IOVA/size ("assume control") and tears it down ("release control"); per-buffer lifecycle is
allocate/map → use → complete. [Confirmed] A separate small shared-variable region exists for
cheaply-polled scalar state. [Inferred]

### 3.5 RTKit base services

Each base service is a distinct endpoint (§3.2); message sets are the RTKit-standard per-service
grammars. Presence and tunables are confirmed in-image; message-level detail is [Inferred]
except where noted.

| Service | Endpoint | Function |
|---------|----------|----------|
| Management | `management`/`system` | Endpoint enumeration/start; power-state request/ack; boot/epoch handshake |
| Syslog | `syslog` | DCP text log: buffer announce + line records |
| Crashlog | `crashlog` | Crash-buffer registration + crash-dump delivery. On `t8140` this is split into kernel/user/message compartments plus a coredump server (§12) |
| Debug/kdebug | `kdebug` | Trace-event stream; capacity/mask/threshold tunables |
| IOReport | `report` | Configure/update IOReporting counter channels |
| OSLog | `oslog` | Binary os_log stream (built-in OSLOG mailbox interface) |
| Tracekit | `tracekit` | Ring-buffered tracing; buffer-size and control tunables |
| Power / EP mgmt | `management` / `power` | Host↔DCP power-state coordination: remote power-state set/handler, wait-for-idle |
| Time | `management` | RTKit time-base exchange [Uncertain] |

### 3.6 AFK transport

The peripheral services do not ride the DCP RPC link; they use **AFK**, a ring-buffer
messenger over RTKit, with **EPIC** service semantics on top (§3.7). [Confirmed]

- Each AFK endpoint is an ordinary RTKit endpoint whose 64-bit words carry AFK protocol
  opcodes and ring pointers rather than inline data. [Confirmed]
- **Ring bring-up handshake:** the two ends exchange a *READY* and a *READY-ACK* control
  opcode, each carrying a `protocol` version and an `options` bitmap; this negotiates and
  starts the shared ring buffers. Ring geometry (slot counts/sizes) is negotiated at runtime,
  not fixed in the image. [Confirmed]
- **Rings** are DART-mapped shared memory (§3.4). Larger transfers are carried by **memory
  descriptors** referenced from ring messages; block (de)allocation is by explicit
  block-request messages, and freeing by a descriptor-free message. [Confirmed]
- The host send primitive takes a message-info header, an array (span) of AFK message
  elements, and send options — a single send may enqueue multiple envelope elements. Each ring
  message word carries a command/opcode and an argument/pointer, driven by a producer/consumer
  state machine guarded by a ring magic value. [Confirmed]

### 3.7 EPIC service layer

Within one AFK endpoint, **EPIC** multiplexes multiple named logical **services** (each
addressed by an interface number, `ifNum`) and provides announce/discovery, open/close, and a
command/report/response exchange. [Confirmed]

**Packet header** (self-describing) [Confirmed]:

| Field | Meaning |
|-------|---------|
| `pktLen` | Total packet length |
| `pktCat` | Packet category (command / report / response / internal) |
| `pktType` | Message type within the category |
| `commandID` | Correlates a command with its response |
| `result` | Status returned in a response |
| `ifNum` | Target EPIC interface within the endpoint |

**Packet classes** [Confirmed]:

- **Command** — enqueued with `(pktType, commandID)`, delivered to a command handler.
- **Report** — enqueued with `(pktType)`; unsolicited/async (publish, open, close, terminate,
  hotplug, EDID, …).
- **Response** — enqueued with `(pktType, commandID, result)`; matched to a command by
  `commandID`. An unmatched response is logged as unexpected.

**Service lifecycle** [Confirmed]: service creation → name discovery (the service's functional
name and its EPIC name are resolved) → register → publish (name + index) → open (by a client)
→ close → terminate. Each announced service carries a small property dictionary for matching,
with keys `EPICName`, `EPICProviderClass`, `EPICLocation`, `EPICUnit`, `endpoint-name`. A
report may be forwarded to another coprocessor (cross-IOP proxying). Within a service an
individual command is addressed by a 64-bit **selector** plus an argument count; the full
per-service numeric selector tables are not stringified and are [Uncertain]. EPIC payloads are
flat/inline — the descriptor/scatter mechanism used by some AFK services is explicitly not
supported by EPIC. [Confirmed]

The EPIC service inventory is enumerated in §9.
---

## 4. RPC method interface

### 4.1 The Call primitive and method tags

Every typed method call on the DCP RPC link uses one transport primitive:

```
Call(method_tag, in_buffer, in_len, out_buffer, out_len) -> Status
```

- `method_tag` — a 32-bit FourCC method selector (§1.2).
- `in_buffer` / `in_len` — a flat, contiguous binary blob holding all input argument data.
- `out_buffer` / `out_len` — a flat, contiguous binary blob the callee fills with all output
  data.

Both lengths are fixed for a given method and are computed from the method signature. The
transport delivers the input blob to the DCP, the DCP executes the method against its copy,
and the transport returns the output blob. The effective return of every call is a **Status**
word (a signed 32-bit result code); query results are delivered through output arguments. A few
trivial accessors return a scalar inline instead. [Confirmed]

**Tag namespaces.** Host→DCP calls use an `A…` tag; DCP→host callbacks use a `D…` tag. The
last three characters of a tag are decimal digits acting as a per-endpoint method index. Tags
are per-endpoint, per-build ordinals — stable for the core framebuffer/pipeline surfaces
across these two silicon variants, but not global constants (see §12). [Confirmed]

**In/out split.** A Call is unidirectional at the blob level but carries data both ways:

- The **in-buffer** holds every field read by the DCP: all by-value scalars, all inline input
  pointer payloads (InPtr/InOutPtr data), and the null-mask bytes for *every* pointer argument.
- The **out-buffer** holds every field the DCP produces: OutPtr payloads, returned (modified)
  InOutPtr payloads, and — for methods that return a status — a trailing status word as the
  final 4-byte field. Methods returning no status place only data in the out-buffer.
[Confirmed]

### 4.2 Directional pointer types

Pointer-typed arguments are classified by data-flow direction. All three classes pass their
payload **inline inside the flat blob**; the DCP never dereferences a host virtual address for
these arguments. The marshaller copies the pointee into the buffer on the way in and copies
returned bytes back out into the caller's object on the way out. [Confirmed]

| Class | Direction | In-buffer content | Out-buffer content |
|-------|-----------|-------------------|--------------------|
| **InPtr** | host→DCP only | inline copy of the pointee | (none) |
| **OutPtr** | DCP→host only | null-mask byte only | inline slot the DCP writes |
| **InOutPtr** | bidirectional | inline copy of the pointee | inline slot with the modified value |

Object/handle arguments (e.g. a surface) are **not** copied byte-for-byte; they are reduced to
an identifier/handle carried in the in-buffer. [Inferred] Large out-of-line buffers backed by
their own DMA/shared mapping are passed by reference to a separately mapped region (described by
task + address + length) rather than inlined. [Inferred]

### 4.3 Flat-struct argument serialization

The marshaller is a fixed-width, little-endian, **4-byte-granular** packer. [Confirmed]

- **Field order** follows source argument order: by-value scalars and inline pointer payloads
  in sequence, then the pointer null-mask bytes.
- **Alignment / padding.** Each field is written at its natural position; the *total* buffer
  length is rounded up to a multiple of 4. (Example: a method taking a 32-bit index followed by
  a 3×3 array of 64-bit values lays the index at offset 0, the 72-byte array inline at offset 4,
  one null-mask byte after it, then pads the total to a multiple of 4. A lone boolean argument
  yields a 4-byte in-buffer: one value byte plus three pad bytes.)
- **Size computation.** `in_len` = Σ(value fields + inline input payloads + one null-mask byte
  per pointer), rounded to a 4-byte multiple. `out_len` = Σ(output payloads + optional trailing
  status word), rounded to a 4-byte multiple.
- **Defensive fill.** Buffers are pre-filled with a sentinel byte pattern before assembly;
  padding and the data slot of a NULL pointer therefore carry sentinel content, not zeros. The
  DCP reads only the defined fields.

**Nullable pointers.** Every pointer argument — In/Out/InOut — is accompanied by exactly **one
boolean byte** recording whether the caller's pointer was NULL (1 = NULL, 0 = present). These
bytes are grouped into a **contiguous parallel mask array** within the in-buffer (one byte per
pointer, in order), after which the buffer is padded to a 4-byte multiple. When a pointer is
NULL its inline data slot is left at the sentinel fill and the DCP skips it based on the mask.
This is the mechanism for optional arguments. [Confirmed]

**Scalar / aggregate type system** [Confirmed unless noted]:

| Type | Wire encoding | Field width |
|------|---------------|-------------|
| int8 / uint8 | 1 byte | 1 byte (buffer padded to 4) [Inferred] |
| int16 / uint16 | 2 bytes LE | 2 bytes [Inferred] |
| int32 / uint32 | 4 bytes LE | 4 bytes |
| int64 / uint64 | 8 bytes LE | 8 bytes |
| Float32 | 4 bytes IEEE-754 LE | 4 bytes |
| Float64 | 8 bytes IEEE-754 LE | 8 bytes [Inferred] |
| Bool | 1 value byte (0/1) in a 4-byte-padded field | value byte + pad |
| FourCC | 4 bytes; ASCII as a 32-bit value | 4 bytes |
| Array(N,T) | N elements of T laid inline, contiguously | element alignment of T |
| string(N) | fixed N-byte field, null-padded | 1-byte; (64-byte fixed name fields observed) |
| SizedBytes / Bytes(N) | fixed N-byte inline region (+ a separate length for SizedBytes) | 1-byte [Inferred] |
| pointer (In/Out/InOut) | inline payload + one null-mask byte | payload alignment; mask grouped (§ above) |

Note on **Bool**: the value occupies a single byte within a 4-byte field; the upper three bytes
are undefined padding. A "bool" that is deliberately transmitted as a fully-defined 32-bit 0/1
is indistinguishable from a `uint32`. [Confirmed for the 1-byte-in-4 encoding; the fully-defined
32-bit form is [Uncertain].]

### 4.4 Object serialization (dictionaries)

Schema-less, nested arguments — property dictionaries, mode/capability blobs, EDID and HDR
metadata — are not flat-packed. They are serialized to a self-describing IOKit-derived object
stream. One **object grammar** is used (dictionary / array / number / boolean / string / symbol
/ data, with back-references for shared objects), in two **representations**:

**(a) Binary object form (primary).** Produced/consumed by the standard binary object
serializer. Value kinds exercised on the control path: dictionary, array, number, boolean,
string, data. On the framebuffer control channel a serialized dictionary is embedded in a
**fixed 4096-byte (0x1000) region** of the message record. The DCP→host property-set callbacks
build a record of the form:

| Offset | Field |
|--------|-------|
| +0x000 | property/record id (uint32) |
| +0x004 | property name, `string(0x40)` (64-byte, null-padded) |
| +0x044 | serialized dictionary blob, **0x1000 bytes** |
| +0x1044 | null-mask byte for the dictionary pointer |

The receiver deserializes the 0x1000-byte blob back into a dictionary (subject to the mask
byte) and applies it. The fixed-region dictionary and array blobs are what the firmware labels
*AFKDictionary* / *AFKArray*. The 0x1000 padding size is identical on both silicon. On the
framebuffer channel this traffic flows DCP→host (the DCP asks the host to set registry
properties). [Confirmed]

**(b) Textual (XML) object form.** The same object grammar rendered as UTF-8 XML text. Used for
registry/property and diagnostic serialization rather than the hot control path. [Confirmed]

**Object transport over AFK/EPIC.** The AV/HDMI/DP sink services exchange whole objects with the
DCP by converting them to/from the *binary* object form and carrying them as EPIC command/report/
response payloads (with a variable-bytes helper for oversize payloads). This is a transport
wrapper over the same binary object grammar, not a third encoding. [Confirmed]

Summary of the coders: one object grammar; two representations (binary, XML); two transports for
the binary representation (the fixed 0x1000 region inside a flat Call record, and variable-length
AFK/EPIC message payloads).
---

## 5. RPC calls (host → DCP)

The host→DCP call surface is partitioned into six method-tag ranges, each served by a distinct
DCP-side object (named here by function). Return type is **Status** unless a scalar return is
noted. Pointer arguments are annotated In/Out/InOut per §4.2. Parameter types are given
functionally: *u32/u64* = unsigned integer; *surface* = surface handle; *dva* = device-virtual
address; *task* = client task handle; *client* = user-client handle. Structure names are defined
functionally in §10.

| Tag range | Service (functional) | Domain | # tags (t8132) |
|-----------|----------------------|--------|:--:|
| `A000`–`A046` | SoC Display-Pipe | Per-pipe boot, color, run-mode, real-time bandwidth, frame/default-framebuffer, CRC | 41 |
| `A100`–`A132` | Shared Pipe | Gamma/matrix, timing, data blocks, color, CRC, test-generator, PMU/backlight matching | 33 |
| `A200`–`A206` | Property Relay | Push/pull DCP runtime property values | 7 |
| `A350`–`A393` | Display Pipeline | Mode/timing enumeration, DSC, tiling, display-wall, bandwidth | 44 |
| `A401`–`A480` | Framebuffer | Framebuffer/swap/surface/power/color/brightness/digital-out surface | 74 |
| `A500`–`A501` | Power Manager | Power-state handshake | 2 |

Total distinct host→DCP call tags on `t8132`: **~200** (the sum of the per-service counts above;
the exact figure varies with the numbering gaps noted below) [Confirmed]. Small numbering gaps
(e.g. `A003`, `A105`, `A402`) are tags not observed at any call site in this build (reserved,
retired, or issued only from an orchestration path that fans out to other tags).

### 5.1 Framebuffer service (`A401`–`A480`)

**Lifecycle / client state**

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A401` | start_signal | () | — | Signal the framebuffer service has started; hand control to the DCP boot flow |
| `A458` | last_client_close | (u32* outState) | Out | Notify that the last user client closed; returns residual state |
| `A465` | set_display_refresh_properties | () | — | Push refresh-related properties after a mode/timing change |
| `A468` | flush_supports_power | (bool) | — | Publish whether the display currently supports powering |
| `A474` | is_keep_on_screen | () → bool | — | Query whether the current framebuffer must be kept on screen |

A "first client open" host orchestration entry point drives several of these and the power
group; it is not itself a single tagged call. [Confirmed]

**Power management**

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A473` | set_power_state | (u64 state, bool, bool, u32* outResult, bool, bool) | Out | Drive a display power-state transition; flags select sync/deferred behavior |
| `A500` | link_shutdown_signal | (bool, bool) | — | (Power Manager) Signal link shutdown during power-down |
| `A501` | power_state_will_change_to | (u64 newState, u64) | — | (Power Manager) Advance-notice of an impending power-state change |

**Swap / flip pipeline**

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A406` | swap_start | (u32* outSwapID, u64) | Out | Open a swap transaction; DCP returns an allocated swap id |
| `A407` | swap_submit | (SwapRequestRecord*, surface[] layers, dva[] surfaceDVAs, dva[], surface[], dva[], bool, double, dva, bool, u32, bool* , u32*, u32*) | In/Out | Submit a fully-specified swap (surfaces, DVAs, timing/present params); DCP validates and queues it |
| `A408` | swap_submit_main | (SwapRequestRecord*, surface[], dva[], dva[], bool, double, dva, bool, u32, bool*, u32*, u32*) | In/Out | Core swap-submit path shared by the public variants |
| `A409` | swap_submit_blitter | (u32, u32, surface[], dva[]) | In | Blitter-assisted swap submit |
| `A410` | swap_signal | (u32 swapID, u32) | — | Signal/arm a queued swap |
| `A431` | temp_queue_swap_cancel | (u32 swapID) | — | Cancel a swap held in the temporary queue |
| `A432` | swap_cancel | (u32 swapID) | — | Cancel a specific in-flight swap |
| `A433` | swap_cancel_all | (u64) | — | Cancel all queued/in-flight swaps |
| `A441` | swap_set_color_matrix | (FixedPointColorMatrix*, MatrixFunction, u32) | In | Attach a color matrix to the pending swap |
| `A461` | io_fence_notify | (u32, u32, u64, Status) | — | Notify of an I/O-fence result tied to a swap |
| `A462` | swap_wait | (bool, u32, u32, u32) | — | Block until a swap reaches a requested state |
| `A464` | announce_next_swap_pts | (u64 presentationTimestamp) | — | Pre-announce the presentation timestamp of the next swap |
| `A479` | set_has_frame_swap_function | (bool) | — | Declare that a client frame-swap callback is registered |

**Surface / default framebuffer**

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A404` | rotate_surface | (u32, u32, u32) | — | Rotate the displayed surface |
| `A434` | surface_is_replaceable | (u32, bool* out) | Out | Query whether a surface may be replaced in place |
| `A446` | create_default_frame_buffer | () | — | Create the DCP-side default framebuffer surface |
| `A471` | update_default_fb | (surface, u32, u32, u64) | — | Update default-framebuffer contents (parameterized) |
| `A472` | update_default_fb | (surface) | — | Update default-framebuffer contents (simple) |

Surface DVA mapping/unmapping is handled through the Memory-Descriptor Relay (§6), not a
framebuffer tag. [Inferred]

**Data blocks & parameters**

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A438` | set_block | (task, u32, u32, u64[] keys, u32, u8[] data, u64, u64, u32, bool) | In | Write a typed parameter/data block to the DCP |
| `A439` | get_block | (task, u32, u32, u64[] keys, u32, u8[] out, u64) | In/Out | Read a typed data block from the DCP |
| `A440` | get_buf_block | (task, u32, u32, u64[], u32, u8[], u64, u64, u32, bool) | In | Read a buffer-backed data block |
| `A442` | set_parameter | (ParameterName, u64[] values, u32 count) | In | Set a named scalar-vector parameter (Parameter Store, §8.1) |
| `A466` | export_property | (u32, u32) | — | Export a numeric property to the DCP |
| `A467` | apply_property | (u32, u32) | — | Apply a numeric property on the DCP |

**Color / gamma / matrix**

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A420` | get_gamma_table | (GammaTable* out) | Out | Read the active gamma/transfer table |
| `A421` | set_gamma_table | (GammaTable*) | In | Program the gamma/transfer table |
| `A422` | get_matrix | (u32, u64[3][3]* out) | Out | Read a 3×3 fixed-point color matrix |
| `A423` | set_matrix | (u32, u64[3][3]*) | In | Program a 3×3 fixed-point color matrix |
| `A424` | set_contrast | (float*) | In | Set display contrast |
| `A425` | set_white_on_black_mode | (u32) | — | Enable/select inverted (white-on-black) mode |
| `A426` | set_color_remap_mode | (ColorRemapMode) | — | Select a color-remap mode |
| `A427` | get_color_remap_mode | (ColorRemapMode* out) | Out | Query the active color-remap mode |
| `A455` | set_rendering_angle | (float*) | In | Set the rendering rotation angle |

(A large InOut transfer such as the gamma table moves on the order of ~0xC10 bytes in each
direction. [Confirmed]) Frame-CRC readout lives on the pipe services (`A034`/`A106`/`A107`);
CRC delivery to the host is a callback (§6).

**Brightness**

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A428` | set_brightness_correction | (u32) | — | Apply a brightness-correction value |
| `A436` | set_panel_brightness | (u32) | — | Set panel backlight brightness (system-power/lighting control path) |
| `A437` | get_panel_brightness | (u32* out) | Out | Read panel backlight brightness |

**Digital-out / TV-out / mirroring**

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A411` | set_display_device | (u32) | — | Select the active display device |
| `A412` | is_main_display | () → bool | — | Query whether this is the main display |
| `A413` | set_digital_out_mode | (u32 colorMode, u32 timingMode) | — | Set the digital output mode (timing id + color id) |
| `A414` | get_digital_out_state | (u32* out) | Out | Read the current digital-out state |
| `A415` | get_display_area | (DisplayArea* out) | Out | Query the active display area/geometry |
| `A416` | set_tvout_mode | (u32) | — | Set TV-out mode |
| `A417` | set_tvout_signaltype | (u32) | — | Set TV-out signal type |
| `A418` | set_wss_info | (u32, u32) | — | Set wide-screen-signalling info |
| `A419` | set_content_flags | (u32) | — | Set content-descriptor flags |
| `A449` | set_underrun_color | (u32) | — | Set the underrun fill color |
| `A451` | set_video_dac_gain | (u32) | — | Set analog video DAC gain |
| `A452` | set_line21_data | (u32) | — | Set line-21 (closed-caption) data |
| `A453` | enable_internal_to_external_mirroring | (bool) | — | Enable internal→external mirroring |
| `A454` | get_external_mirroring_capability | (MirroringCapability* out) | Out | Query external-mirroring capability |
| `A456` | set_overscan_safe_region | (OverscanSafeRect*) | In | Set the overscan-safe rectangle |

**Display info / geometry**

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A405` | get_framebuffer_id | () → u32 | — | Return the framebuffer id |
| `A443` | display_width | () → u32 | — | Return display width in pixels |
| `A444` | display_height | () → u32 | — | Return display height in pixels |
| `A445` | get_display_size | (u32* w, u32* h) | Out | Query physical/display size |
| `A463` | get_link_quality | () → u32 | — | Query a link-quality metric |
| `A478` | get_shared_disp_state | () → u32 | — | Query shared display state |
| `A480` | get_performance_stats | (u32* a, u32* b) | Out | Read framebuffer performance counters |

**Gain maps (HDR)**

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A429` | create_gain_map | (task, client, GainMapDescriptor*, u32* outID, u64, u64, bool) | In/Out | Create an HDR gain map; DCP returns an id |
| `A430` | delete_gain_map | (u32 id) | — | Delete a gain map by id |
| `A470` | remove_gain_maps | (client) | — | Remove all gain maps owned by a client |

**Clamshell**

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A476` | set_clamshell_state | (u32) | — | Push clamshell (lid) state to the DCP |
| `A477` | send_clamshell_state_to_controller | () | — | Forward clamshell state to the display controller |

**Debug / test / misc**

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A435` | kernel_tests | (KernelTestArgs*) | InOut | Run in-firmware test routines |
| `A448` | enable_disable_dithering | (u32) | — | Enable/disable dithering |
| `A450` | enable_disable_video_power_savings | (u32) | — | Toggle video power-saving |
| `A459` | get_debug_info_buffer | (u32*, u32*) | Out | Obtain debug-info buffer descriptors |
| `A460` | flush_debug_flags | (u32) | — | Flush/apply debug flags |
| `A469` | abort_swaps | (u64) | — | Abort outstanding swaps (error recovery) |

HDCP host operations (send request, get reply, downstream-state, encryption-status) are
exchanged through the block/parameter and notification paths rather than dedicated framebuffer
tags in this build; the host entry points exist but fan out to `set_block`/`get_block` and
notifications. [Inferred]

### 5.2 Display Pipeline service (`A350`–`A393`)

Controls mode discovery and multi-DCP / display-wall / DSC configuration.

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A350` | display_height | () → u32 | — | Pipeline display height |
| `A351` | display_width | () → u32 | — | Pipeline display width |
| `A352` | apply_property | (u32, u32) | — | Apply a pipeline numeric property |
| `A353` | get_min_frame_period_ns | () → u64 | — | Minimum frame period (ns) |
| `A354` | set_fast_preset_in_progress | (FastPresetData*) | In | Mark a fast preset-switch in progress |
| `A355` | get_tiled_modes | (TimingParameters*, u32*, u32, ColorParameters*, u32*, u32) | In/Out | Enumerate tiled timing+color modes |
| `A356` | remove_tiled_modes | () | — | Remove previously inserted tiled modes |
| `A357` | insert_tiled_modes | (TimingParameters*, u32, u32) | In | Insert tiled modes |
| `A358` | remove_display_wall_mode_list | () | — | Remove the display-wall mode list |
| `A359` | build_display_wall_mode_list | (u32, u32, u32, u32*) | Out | Build a display-wall mode list |
| `A360` | get_master_swap_id | () → u32 | — | Get the master swap id (multi-DCP sync) |
| `A361` | set_master_swap_id | (u32) | — | Set the master swap id (multi-DCP sync) |
| `A362` | manage_elements | (bool) | — | Enable/disable pipeline element management |
| `A363` | manage_properties | (bool) | — | Enable/disable pipeline property management |
| `A364` | get_video_data_for_ids | (VideoTimingData*, VideoColorData*, u32, u32, bool*) | Out | Resolve timing+color data for mode ids |
| `A365` | insert_merge_modes | (VideoTimingData*, VideoColorData*, u32, u32) | In | Insert merged (AV) modes |
| `A366` | setup_peer_dcp | (bool) | — | Configure the peer-DCP relationship |
| `A367` | get_dsc_capabilities | (DSCCapabilities* out) | Out | Query DSC capabilities |
| `A368` | set_dsc_capabilities | (DSCCapabilities*) | In | Set DSC capabilities |
| `A369` | get_requires_dsc | () → bool | — | Whether the current mode requires DSC |
| `A370` | get_dsc_bits_per_pixel | (u16* out) | Out | Query DSC bits-per-pixel |
| `A371` | set_dsc_bits_per_pixel | (u16*) | In | Set DSC bits-per-pixel |
| `A372` | get_dsc_slice_size | (DSCSliceHints* out) | Out | Query DSC slice-size hints |
| `A373` | set_dsc_slice_size | (DSCSliceHints*) | In | Set DSC slice-size hints |
| `A374` | get_matching_mode | (TimingParameters*, u32*, u32, ColorParameters*, u32*, u32) | In/Out | Find the best-matching timing+color mode |
| `A375` | get_display_product_id | (char*, u32*, u32, u32*) | Out | Read display product id / name |
| `A376` | get_display_wall_mode_size | (u32, u32*, u32*) | Out | Get display-wall mode dimensions |
| `A377` | get_system_type | () → u32 | — | Query system/display type |
| `A378` | get_display_wall_params | () | — | Get display-wall parameters |
| `A379` | is_display_wall_supported | () → bool | — | Whether display-wall is supported |
| `A380` | headless | () → bool | — | Whether the pipeline is headless |
| `A381` | are_timings_enabled | () → bool | — | Whether timings are enabled |
| `A382` | get_apt_frames_completed | () → u32 | — | Adaptive-panel-timing frames-completed counter |
| `A383` | export_idle_method | (u32) | — | Export/select an idle-caching method |
| `A384` | get_timing_attributes | (RefreshTimingAttributes* out, bool) | Out | Query refresh timing attributes |
| `A386` | set_temperature_hint | () | — | Push a temperature hint into the pipeline |
| `A387` | allocate_bandwidth_callback | (bool* out) | Out | Bandwidth-allocation gated call |
| `A388` | get_supports_panel_replay | () → bool | — | Whether panel self-refresh/replay is supported |
| `A390` | get_is_display_x1 | () → bool | — | Query the display-X1 flag |
| `A391` | set_is_display_x1 | (bool) | — | Set the display-X1 flag |
| `A393` | get_ext_disp_fast_preset_switch_supported | () → bool | — | Whether ext-display fast preset switch is supported |

### 5.3 Shared Pipe service (`A100`–`A132`)

| Tag | Method | Parameters | Ptr | Description |
|-----|--------|------------|-----|-------------|
| `A100` | get_gamma_table | (GammaTable* out) | Out | Read gamma table (gated) |
| `A101` | set_gamma_table | (GammaTable*) | In | Write gamma table (gated) |
| `A102` | test_control | (TestControlCmd, u32) | — | Issue a test-control command |
| `A103` | get_config_frame_size | (u32* w, u32* h) | Out | Query configured frame size |
| `A104` | set_config_frame_size | (u32 w, u32 h) | — | Set configured frame size |
| `A106` | read_blend_crc | () → u32 | — | Read blend-stage CRC |
| `A107` | read_config_crc | () → u32 | — | Read config-stage CRC |
| `A108` | disable_wpc_calibration | (bool) | — | Disable white-point-correction calibration |
| `A109` | test_generator_is_running | (RegisterStream*) | In | Whether the video test generator is running |
| `A110` | test_generator_debug | (RegisterStream*, u32) | In | Test-generator debug control |
| `A111` | test_generator_set_color_channels | (u32, u32, u32) | — | Set test-generator output color channels |
| `A112` | set_color_filter_scale | (int) | — | Set color-filter scale |
| `A113` | set_corner_temps | (int*) | In | Set corner-temperature compensation values |
| `A115` | always_on_time_enabled | () → bool | — | Whether always-on-time is enabled |
| `A116` | always_on_time_active | () → bool | — | Whether always-on-time is active |
| `A117` | set_timings_enabled | (RegisterStream*, bool) | In | Enable/disable timings |
| `A118` | get_frame_size | (RegisterStream*, u32* w, u32* h) | In/Out | Query current frame size |
| `A119` | set_block | (u64, u32, u32, u64[], u32, u8[], u64, u64, u32, bool, bool) | In | Write a data block via the pipe |
| `A120` | get_block | (u64, u32, u32, u64[], u32, u8[], u64) | In/Out | Read a data block via the pipe |
| `A121` | get_buf_block | (u64, u32, u32, u64[], u32, u8[], u64, u64, u32, bool, bool) | In | Read a buffer-backed block via the pipe |
| `A122` | get_matrix | (MatrixLocation, FixedPointColorMatrix* out) | Out | Read a color matrix at a pipeline location |
| `A123` | set_matrix | (MatrixLocation, FixedPointColorMatrix*) | In | Write a color matrix at a pipeline location |
| `A124` | get_internal_timing_attributes | (RefreshTimingAttributes* out) | Out | Query internal timing attributes |
| `A125` | display_edr_factor_changed | (float) | — | Notify that the EDR headroom factor changed |
| `A126` | set_contrast | (float) | — | Set contrast on the shared pipe |
| `A127` | p3_to_display_color_space | (float*, float[2]*) | In | Program P3→display color-space conversion |
| `A128` | max_panel_brightness | () → u32 | — | Query max panel brightness |
| `A130` | init_analytics_pmu | () | — | Initialize the analytics/PMU coupling |
| `A131` | pmu_service_matched | () | — | Notify that the PMU service matched |
| `A132` | backlight_service_matched | () | — | Notify that the backlight service matched |

### 5.4 SoC Display-Pipe service (`A000`–`A046`)

Per-silicon pipe: boot, color mode, run mode, real-time bandwidth, frame/default-framebuffer,
CRC. **Tag numbering differs between silicon** (§12): `t8132` tags shown.

| Tag (t8132) | Method | Parameters | Ptr | Description |
|------|--------|------------|-----|-------------|
| `A000` | late_init_signal | (bool) | — | Signal completion of DCP late-init |
| `A001` | init_ipa | (u64, u64) | — | Initialize the image-processing/pipe-address region |
| `A002` | ambient_light_supported | () → bool | — | Whether ambient-light sensing is supported |
| `A004` | display_edr_factor_changed | (float) | — | EDR-factor change (pipe level) |
| `A005` | set_contrast | (float) | — | Set contrast (pipe level) |
| `A006` | set_csc_mode | (CSCMode) | — | Set color-space-conversion mode |
| `A007` | set_op_mode | (CSCMode) | — | Set output-processing CSC mode |
| `A008` | set_op_gamma_mode | (TransferMode) | — | Set output-processing gamma/transfer mode |
| `A009` | set_video_out_mode | (OutputMode) | — | Set video output mode |
| `A010` | set_meta_allowed | (bool) | — | Allow/deny metadata pass-through |
| `A011` | set_tunneled_color_mode | (bool) | — | Enable tunneled color mode |
| `A012` | set_bwr_line_time_us | (double) | — | Set bandwidth-reservation line time (µs) |
| `A013` | performance_feedback | (double) | — | Deliver performance feedback to the DCP |
| `A014` | notify_swap_complete | (u32) | — | Notify the pipe of swap completion |
| `A015` | is_run_mode_change_pending | () → bool | — | Whether a run-mode change is pending |
| `A016` | ready_for_run_mode_change | (RegisterStream*) | In | Signal readiness for a run-mode change |
| `A017` | set_thermal_throttle_cap | (u32) | — | Set the thermal-throttle cap |
| `A019` | set_target_run_mode | (RegisterStream*) | In | Set the target run mode |
| `A020`* | rt_bandwidth_setup | () | — | Set up real-time bandwidth (**t8132 only**) |
| `A021`* | rt_bandwidth_update | (RegisterStream*, float, float, bool, bool) | In | Update real-time bandwidth (**t8132 only**) |
| `A022`* | rt_bandwidth_update_downgrade | (RegisterStream*) | In | Downgrade real-time bandwidth (**t8132 only**) |
| `A023`* | rt_bandwidth_write_update | (RegisterStream*, WritebackBlock, bool) | In | Real-time-bandwidth writeback update (**t8132 only**) |
| `A024` | blending_eco_present | () → bool | — | Whether the blending ECO is present |
| `A025` | request_backlight_update | () | — | Request a brightness/backlight update |
| `A027` | get_max_frame_size | (u32* w, u32* h) | Out | Query the maximum frame size |
| `A028` | shadow_fifo_empty | (RegisterStream*) | In | Whether the shadow FIFO is empty |
| `A030` | can_program_swap | () → bool | — | Whether a swap may be programmed now |
| `A031` | in_auto_mode | () → bool | — | Whether the pipe is in auto mode |
| `A032` | is_piodma_used | () → bool | — | Whether PIO-DMA is in use |
| `A034` | read_crc | (NotifyInfoIndex, u32) | — | Read a CRC value by index |
| `A035` | update_notify_clients | (u32*) | In | Update the notify-client tag set |
| `A036` | is_dual_rate | () → bool | — | Whether hi/lo (dual-rate) mode is active |
| `A037` | apt_supported | () → bool | — | Whether adaptive-panel-timing is supported |
| `A038` | get_default_fb_info | (u32*, u64*, u32*) | Out | Query default-framebuffer info |
| `A039` | get_default_fb_compression_info | (u32*) | Out | Query default-framebuffer compression info |
| `A040` | get_frame_done_time | () → u64 | — | Query the last frame-done timestamp |
| `A041` | get_performance_headroom | () → u32 | — | Query performance headroom |
| `A042` | are_stats_active | () → bool | — | Whether statistics collection is active |
| `A043` | supports_odd_h_blanking | () → bool | — | Whether odd horizontal-blanking is supported |
| `A044` | is_first_hw_version | () → bool | — | Whether this is the first HW revision |
| `A046` | update_ext_disp_clock_info | (u64) | — | Update external-display clock info |

\* The four real-time-bandwidth calls (`A020`–`A023`) exist on `t8132` but are **absent on
`t8140`**; their removal compacts the remaining `t8140` numbering (§12). [Confirmed]

### 5.5 Property Relay (`A200`–`A206`)

Host→DCP direction of the numeric runtime-property mechanism (§8.3). Both directions share the
RuntimeProperty id space.

| Tag | Method | Parameters | Description |
|-----|--------|------------|-------------|
| `A200` | set_bool | (RuntimeProperty, bool) | Set a boolean runtime property on the DCP |
| `A201` | set_int | (RuntimeProperty, u32) | Set an integer runtime property |
| `A202` | set_fx | (RuntimeProperty, int) | Set a fixed-point runtime property |
| `A203` | set_prop_dynamic | (RuntimeProperty, u32) | Set a dynamic property |
| `A204` | get_bool | (RuntimeProperty) → bool | Read a boolean runtime property |
| `A205` | get_int | (RuntimeProperty) → u32 | Read an integer runtime property |
| `A206` | get_fx | (RuntimeProperty) → int | Read a fixed-point runtime property |
---

## 6. RPC callbacks (DCP → host)

DCP→host callbacks are an ordered, numerically-tagged dispatch surface running over the same
name/tag-dispatched link (§3.3), in the reverse direction. Each callback receives the link/stream
context plus an input blob (DCP-provided data) and produces an output blob (host reply, if any):

```
D<nnn>(link, stream, in_buffer, in_len, out_buffer, out_len) -> Status
```

### 6.1 Callback tag ranges

The tag space is partitioned into contiguous per-service ranges. **Ranges and counts are
directly recovered [Confirmed];** the binding of an individual tag *within* a range to a specific
handler is only partially recoverable and is treated as [Uncertain] (§6.7).

| Tag range | Service (functional) | Count (t8132 / t8140) | Domain |
|-----------|----------------------|:--:|--------|
| `D000`–`D008` / `D000`–`D007` | SoC Display-Pipe | 9 / 8 | Control-mailbox events, ambient-light/backlight config, adaptive-panel-timing changes |
| `D100`–`D134` | Display Pipeline | 35 / 35 | Host-service, timing, power-gating, tiling callbacks |
| `D200`–`D209` | Shared Pipe | 10 / 10 | Per-pipe callbacks |
| `D300` | Property Relay | 1 / 1 | Numeric property publish |
| `D400`–`D424` | Host Service Relay | 25 / 25 | Host property/service queries (§8.2) |
| `D450`–`D454` | Memory-Descriptor Relay | 5 / 5 | DART-buffer mapping/unmapping (§3.4) |
| `D550`–`D599` | Framebuffer | 50 / 50 | Swap, hotplug, CRC, power, HDCP, idle, property publish (§8.4) |
| `D700` | Power Manager | 1 / 1 | Power-manager callback |

**Total numbered DCP→host callbacks: 136 (t8132) / 135 (t8140).** The single difference is one
extra SoC-pipe callback on `t8132`. [Confirmed] The Pipeline/Shared-Pipe/Property-Relay ranges
reside in the per-silicon display module; the Framebuffer/Service-Relay/Memory-Descriptor/
Power-Manager ranges reside in the shared graphics module and are common to both silicon.
[Confirmed]

The DisplayPort/DPTX/AV/CEC/HDCP callbacks (§9) do **not** use this `D<nnn>` space; they arrive
over the separate AFK/EPIC transport and are dispatched by typed message handlers. [Confirmed]

### 6.2 Hotplug / connection events

| Callback | Sync/Async | Parameters | Description |
|----------|-----------|------------|-------------|
| hotplug_notify | ASYNC (gated) | (u64 token, TiledDisplayInfo* info, bool connected) | Display connect/disconnect; carries tiled/multi-pipe topology so the host rebuilds the mode table |
| force_hotplug_detect | ASYNC | (port, AV-IPC message) | Request a forced HPD re-detect on a port |
| hotplug_detect_change_occurred | ASYNC | (port, bool state) | Report an HPD level change up the stack |
| set_hpd_state | ASYNC | (port, AV-IPC message) | DCP sets host-visible HPD state (low-power port) |

### 6.3 Swap / present / vblank notifications

| Callback | Sync/Async | Parameters | Description |
|----------|-----------|------------|-------------|
| swap_complete | ASYNC (gated) | (u32 swapID, bool success, SwapCompletionRecord*, SwapInfoRecord*, u32 flags, bool) | Primary flip-done callback: a submitted swap has been presented |
| batched_swap_complete | ASYNC (gated) | (u32[] swapIDs, u32 count, bool, bool[] perSwapSuccess, SwapCompletionRecord*) | Coalesced completion of several swaps |
| swap_complete_intent | ASYNC (gated) | (u32 swapID, bool success, SwapIntent, u32, u32) | Completion carrying the swap's requested intent class |
| swap_complete_head_of_line | ASYNC (gated) | (u32 swapID, bool, u32, bool) | Head-of-line swap retired (ordering/queue accounting) |
| swap_notify | ASYNC (gated) | (u64, u64, u64) | Per-frame swap/present notification (present timestamps / vblank counters) |
| swap_info_notify | ASYNC (gated) | (SwapInfoRecord*) | Publish a swap-info record to registered notify clients |
| need_swap_notify | ASYNC | () | DCP asks the compositor for a fresh swap (e.g. ESD recovery) |
| announce_next_swap_pts | ASYNC | (u64 presentationTimestamp) | Announce the presentation time of the next scheduled swap |
| frame_swap_function | ASYNC | (u32, surface, PresentationTimeTransaction*) | Registered per-frame swap callback (frame-driven present) |
| notify_event | ASYNC | (SharedEvent*, u64 value, u64) | Signal an IOSurface shared event (fence) tied to a presented surface |
| release_buffer_notify | ASYNC (gated) | (u32 id, u64, u64) | Early-release/buffer-release notification (source buffer is free) |

### 6.4 Idle-fence / surface lifecycle

| Callback | Sync/Async | Parameters | Description |
|----------|-----------|------------|-------------|
| idle_fence_create | SYNC (reentrant) | (IdleCachingState) | DCP asks the host to create an idle fence on entering an idle-caching state |
| idle_fence_complete | ASYNC | () | Idle fence has completed |
| idle_surface_release / …_nolock / verify_… | ASYNC | () | Request/validate release of the idle (self-refresh) surface |
| io_fence_notify | ASYNC | (u32, u32, u64, Status) | Generic I/O-fence completion/error notification |
| io_fence_callback | ASYNC | (ctx, ctx, surface, int status) | Lower-level fence callback carrying the target surface and status |
| shared_event_signal / …_abort | ASYNC | (SharedEvent*, surface, u32, u32, bool[, bool]) | Drive a host-side shared event to signal/abort a waiter tied to a surface |

### 6.5 Power / display-config / CRC / statistics

| Callback | Sync/Async | Parameters | Description |
|----------|-----------|------------|-------------|
| powerstate_notify | ASYNC | (bool, bool) | Notify a display/controller power-state transition |
| set_start_complete | SYNC (handshake) | (bool complete) | Signal firmware start/boot completion; host waits on it |
| start_signal / stop_signal | ASYNC | () | Start/stop lifecycle signals to the host client state machine |
| power_manager_callback (`D700`) | SYNC (reentrant) | transport buffer | DCP power manager queries/notifies host power via the registered power handler |
| enable_backlight_message | ASYNC (gated) | (bool enable) | Request the host to enable/disable backlight messaging |
| edid_callback | ASYNC (AFK) | (interface, AV-IPC message) | Deliver updated EDID/display-identification data |
| report_callback (AFK endpoint) | ASYNC | (endpoint, u32, u64, data, len, u32) | Generic AFK report (timing/telemetry-changed) |
| response_callback (AFK endpoint) | SYNC (reply) | (endpoint, commandCtx, int, u64, data, len) | AFK response completing a pending host command |
| crc_notify | ASYNC | (u32, u32, u32) | Deliver frame CRC values to the host CRC client |
| do_crc_notify | ASYNC | (SwapInfoRecord*) | CRC notification driven from a swap-info record |
| ambient_histogram_stats_ready / light-adaptive_stats_ready | ASYNC | endpoint buffer | Ambient-histogram / light-adaptive stats-ready |
| mapper_stats_ready (PL / RX-TC) | ASYNC | endpoint buffer | Panel/pixel-mapper statistics-ready |
| report_gp_stats / report_simple_value | ASYNC | pipe/value | General-purpose statistic/telemetry reporting |
| send_flipbook_analytics | ASYNC (timer) | timer ctx | Flipbook/present analytics reporting |
| analytics_send_event | SYNC (reentrant) | (name, payload dict) | DCP-triggered analytics event forwarded by the host |

### 6.6 HDCP / authentication and backlight/ambient callbacks

| Callback | Sync/Async | Parameters | Description |
|----------|-----------|------------|-------------|
| hdcp_send_request | SYNC (reentrant, gated) | (u8[] req, u64 len, NotificationRequestArgs*) | DCP asks the host to transmit an HDCP message toward the sink and register a reply notification |
| hdcp_get_reply | SYNC | (memoryMap, u64* outLen) | Host retrieves the HDCP reply payload for the DCP |
| get_hdcp_downstream_state | SYNC (reentrant, gated) | (HDCPDownstreamState* out) | DCP queries downstream/topology HDCP state from the host |
| update_hdcp_encryption_status | ASYNC (gated) | (u64 status) | Notify a change in HDCP encryption status |
| hdcp_receiver_connected | ASYNC | (session, AV-IPC message) | HDCP authentication: receiver connected |
| hdcp_receiver_id_list_available | ASYNC | (session, AV-IPC message) | HDCP repeater receiver-ID list available |
| hdcp_read_random | SYNC | (session, AV-IPC message) | HDCP auth random-value read |
| backlight_config_callback | ASYNC | (framebuffer message, config) | Backlight-control-subsystem configuration callback from the DCP framebuffer channel |
| ambient_light_config_callback | ASYNC | (framebuffer message, config) | Ambient-light-sensor-subsystem configuration/notification callback |

### 6.7 SoC-pipe callbacks (`D000` range)

| Callback | Sync/Async | Parameters | Description |
|----------|-----------|------------|-------------|
| control_mailbox_event | ASYNC (gated) | (event descriptor) | SoC control-mailbox event callback |
| control_mailbox_info | ASYNC (gated) | (info type, payload) | SoC control-mailbox info callback (typed payload) |
| apt_default_gray_value_changed | ASYNC | (RegisterStream*) | Adaptive-panel-timing default-gray-value change |
| apt_fixed_rr_changed | ASYNC | (RegisterStream*) | Adaptive-panel-timing fixed-refresh-rate change |
| apt_normal_mode_changed | ASYNC | (RegisterStream*) | Adaptive-panel-timing normal-mode change |

The SoC range also carries the transport-level registrations for the backlight-control and
ambient-light-sensor configuration callbacks. On `t8132` the range is `D000`–`D008` (9); on
`t8140` it is `D000`–`D007` (8); the entry absent on `t8140` is `D008`. [Confirmed]

### 6.8 Numeric tag → callback binding (recovery status)

- Ranges and per-service counts are [Confirmed].
- The individual number-to-handler binding within a range is [Uncertain]: the dispatch entries
  are ordinal and share one transport signature, and the mapping is not expressed in recoverable
  identifiers. No individual `D<nnn>` numbers are asserted for the framebuffer-notification block
  beyond the confirmed range `D550`–`D599`. An ordering hint — the highest-frequency swap-completion
  callbacks appear earliest in the framebuffer block — is [Inferred, low confidence].

---

## 7. Callback / reentrancy model

### 7.1 Synchronous reentrant callbacks

During a single host→DCP Call the DCP can issue one or more **synchronous callbacks** to the
host and block on their replies, effectively extending the call stack across both processors.
These are serviced inline on the RPC receive path and carry a reply buffer. The reentrant set:
[Confirmed]

- **Host-service queries** — the Host Service Relay (`D400`–`D424`), Property Relay (`D300`), and
  Memory-Descriptor Relay (`D450`–`D454`). While the DCP services a host modeset/swap Call it
  synchronously reads host properties, maps device memory, or enables clocks/power, and must
  receive replies to make progress.
- **HDCP data path** — downstream-state query, send-request, get-reply, read-random.
- **Idle-fence creation** — returns a fence the DCP then waits on. [Inferred]
- **Startup / link handshake** — start-complete signalling, link-init callbacks, link power-state
  handler, and the Power Manager callback (`D700`).
- **AFK response** — completes a specific pending host command synchronously.

### 7.2 Asynchronous callbacks

These arrive as standalone DCP→host messages, queued for host processing, carrying no reply the
DCP blocks on: the swap/present family (swap-complete, batched-swap-complete, swap-intent,
head-of-line, swap-notify, swap-info-notify, need-swap-notify, announce-next-swap-pts,
release-buffer-notify, notify-event, io-fence-notify); event notifications (hotplug, powerstate,
CRC, HDCP-encryption-status, EDID/HPD/frame-receive over AFK, stats-ready, backlight/ambient
config). [Confirmed]

### 7.3 Work-loop / command-gate serialization

Asynchronous callbacks that touch shared framebuffer state are marshalled onto the framebuffer
work loop through its command gate before the host acts. This is evidenced by paired handlers: a
transport-level entry plus a gated variant that runs the work under the gate. A dedicated suffix
consistently marks the host-side handler for a DCP-initiated callback, distinct from the suffix
used on host→DCP call variants. [Confirmed]

### 7.4 Multi-channel concurrency and ordering

- Multiple outstanding operations proceed simultaneously on different stream offsets (§3.3);
  each stream has its own work loop/gate/event source. [Confirmed]
- Swap completions are delivered in submission order for the head-of-line case and may be
  batched. [Inferred]
- A **notification-mask** mechanism (set/resend notify-tag mask over a NotificationType space)
  selects which asynchronous notification classes are delivered, so async delivery is gated by
  host-registered subscriptions. [Confirmed]
- Hotplug events arriving during power-down are defensively dropped. [Confirmed]
---

## 8. Key-value / parameter mechanisms

Four distinct key-value mechanisms cross the interface. They differ in direction, in how the
target is named, and in how the value is typed.

| Mechanism | Direction | Target addressing | Value typing |
|-----------|-----------|-------------------|--------------|
| **Parameter Store** (§8.1) | host→DCP | `ParameterName` enum | packed `u64[]` (≤5) + count |
| **Host Service Relay** (§8.2) | DCP→host | numeric service id (host object) + string key | typed accessor per call |
| **Property Relay** (§8.3) | bidirectional | `RuntimeProperty` enum id | Int / Bool / Fx per property |
| **Registry property publish** (§8.4) | DCP→host | registry key string on the framebuffer node | typed setter (int/bool/str/dict) |

### 8.1 Parameter Store (host → DCP)

A keyed vector-of-`u64` channel by which the host pushes scalar display parameters to the DCP.

- Entry point: `set_parameter(ParameterName key, u64[] values, u32 count)` (call `A442`, §5.1).
  A user-client trampoline forwards from user space. [Confirmed]
- Most keys are forwarded verbatim to the DCP, which handles them in a parameter-set dispatch
  (alongside mode-set, brightness-set, …). A few keys are handled host-side before/instead of
  forwarding. [Confirmed]
- **Value-vector limit: at most 5 `u64` per call.** The count is clamped/rejected at 5. [Confirmed]

`ParameterName` is an enumeration. One member is directly recovered by name; the numeric index of
each member is otherwise not individually recoverable from strings.

| Parameter (functional / key) | Value | Meaning | Enum index | Conf |
|------------------------------|-------|---------|:--:|------|
| `adaptive_sync` | 3× u64: min-refresh, media-target-rate, fractional-rate flag | VRR / adaptive-sync activation and min-refresh window (invalid media rates forced to 0) | pass-through (index [Uncertain]) | [Confirmed] |
| user power assertion | bool | Hold/release a DCP kernel power assertion (count = 1) | `0x0B` (11) | [Confirmed] |
| boolean pipeline toggle | bool | Routes a boolean to a pipeline handler (count = 1) | `0x13` (19) | [Inferred] |
| early pre-handled parameter | — | Routed to a framebuffer sub-object, then forwarded | `0x00` (0) | [Inferred] |
| host-consumed no-op | — | Accepted and returns success with no action | `0x09` (9) | [Inferred] |
| dithering enable | bool/u32 | Enable/disable output dithering | — | [Inferred] |
| panel brightness | u32 | Panel backlight level | — | [Inferred] |
| brightness correction | u32 | Brightness-correction factor | — | [Inferred] |
| contrast | float(s) | Output contrast | — | [Inferred] |
| all others | ≤5× u64 | Forwarded unmodified to the DCP parameter-set dispatch | — | [Confirmed] |

Several display attributes (adaptive/AOD refresh windows, ambient brightness, discrete media
refresh, panel-drive-compensation controls) appear as key strings but may be delivered via the
Property Relay (§8.3) or the block/property paths rather than this enum; their store assignment
is [Uncertain]. The high-confidence members of this store are `adaptive_sync`, the power
assertion, dithering, brightness, brightness-correction, and contrast.

### 8.2 Host Service Relay (DCP → host)

The DCP names a host object by a **numeric service id** (a 32-bit, FourCC-style object tag) and
issues typed get/set calls against a `(service_id, key)` pair, where `key` is a NUL-terminated
C-string (a registry/device-tree property name); some overloads accept a string-object key
instead. Host objects are registered with the relay before use; the relay resolves the id to the
registered proxy and performs the registry access or the clock/power operation. Direction is
DCP→host (synchronous reentrant, §7.1). Fixed-point ("fx") values are 32-bit signed. This group
is the callback range `D400`–`D424`. [Confirmed]

**Entry points**

| Method | Parameters | Returns | Meaning |
|--------|------------|---------|---------|
| get_property | svc, key, u8* buf, u32* len (in/out) | status | Read an arbitrary-length property blob (EDID, calibration arrays, …); `len` is buffer size in / bytes out |
| get_uint_prop (32 / 64) | svc, key, u32*/u64* out | status | Read a scalar property as unsigned 32/64-bit |
| set_uint_prop (32 / 64) | svc, key, u32/u64 val | status | Write a scalar 32/64-bit property (materialized as a number object) |
| set_bool_prop | svc, key, bool | status | Write a boolean property |
| get_fx_prop / set_fx_prop | svc, key, int32* / int32 | status | Read/write a fixed-point property |
| set_property (typed) | svc, key (C-string or string-object), value | status | Set a property to a specific typed container; value types: bool, number, string, boolean-object, array, dictionary (6 value types × 2 key forms) |
| remove_property | svc, key (C-string or string-object) | status | Remove a property (2 key-form overloads) |
| map_device_memory_with_index | svc, u32 index, u32 flags, u64* addr, u64*, u64* length | status | Map the Nth device-memory (register) aperture of the host object into a DCP mapping |
| enable_device_clock | svc, u32 clockIndex, u32 enable | status | Enable/disable an indexed clock of the host object |
| enable_device_power | svc, u32 domain, u32* state, u32 enable | status | Enable/disable a power domain; returns resulting state |
| get_clock_frequency | svc, u32 clockIndex | u64 Hz | Query an indexed clock's frequency (index 0 is the pixel/video clock fallback) |
| register_service | proxy, svc | — | (Host-internal) Bind a host proxy object to a numeric service id |

Housekeeping: init, free, service lookup by id, release of established device-memory mappings.
[Confirmed]

**Addressed host objects.** The service id is assigned at registration; per-object numeric values
were not recovered ([Uncertain]). The *set* of addressable objects: power manager (power domains,
gates, clocks); display clock manager (clock ids, pixel/display clock frequencies); PMU/backlight
PMIC (enable, sequencing delays); system management controller (sensor/telemetry); backlight/
brightness service (calibration, level); panel/embedded-display-timing node (timing, panel id,
bit depth, color space, EDID); DART/IOMMU (device-memory apertures); and the downstream AV/
DisplayPort topology objects. [Confirmed set; ids Inferred]

**Property keys** (C-strings passed to the get/set calls; all [Confirmed] as present):

*Clocks / power / frequency:* `clock-ids`, `max-pixel-clock`, `minimum-frequency` (t8132),
`vid-clock-to-disp-clock-factor`, `power-gates`, `power-gates-disp[0]`,
`power-gates-dispext[0/1]`, `power-gates-len-*`, `power-gates-stg2-*`, `power-gate-dbe-disp`,
`power-gate-dbe-dispext`, `power-gate-dsg-disp`, `control-rails`, `agile-clocking-data`.

*Backlight / brightness:* `backlight-enable`, `backlight-pmic-enable`, `backlight-calibration`,
`backlight-calibration-parameters`, `OLED-backlight-calibration`, `bl-pmic-off-to-bl-delay`,
`bl-pmic-off-to-lcd-delay`, `current-for-max-backlight`, `current-for-mid-backlight`,
`display-backlight-compensation`, `display-backlight-compensation-v1`, `twod-backlight` (t8132),
`force-dbv` (t8132).

*Panel / EDID / calibration:* `lcd-panel-id` (t8132), `panel_id`, `panel-overrides`,
`board-overrides`, `display-panel-bit-depth`, `display-panel-temperature-data`,
`display-temp-compensation`, `display-eeprom-compensation`, `gamma-calibration-lut`,
`panel-gamma`, `primary-calibration-matrix`, `display-ean-prst-data`, `coverglass-serial-number`,
`coverglass-serial-number-key`, `display-data-adcl`, `alpm-data`, `aod-data`, `dbsr-lut`,
EDID read-back (via `get_property`).

*Timing / geometry / attributes:* `display-timing-info`, `flexible-v-timing`,
`average-refresh-rate`, `refresh-rate`, `max-active-pixel-rate`, `max-total-pixel-rate`,
`max-pixel-width-ext`, `display-color-space`, `display-default-color`, `display-lead-time-nclks`,
`display-coex-prioritize-display`, `display-coex-prioritize-prox`, `display-vsh-comp`.

*External-display (t8132 only):* `external-display-limit-compress`, `external-display-limit-gps`,
`external-display-limit-scale`, `disable-tcon-2dira`, `target-require-2dira-wa`,
`ean-mode-update-thesh`.

*Internal-panel / brightness-health (t8140 only):* `brightness-service-lcd-2`,
`brightness-monitor-service`, a `brightness-health-*` telemetry key group,
`controller-pmu-frontend`, `controller-lp8549`, `backlight-level-encoding`, `async-mipi-cmd`.
These reflect the internal-panel PMIC/backlight hardware and the added panel-health monitoring on
that silicon; the mechanism is unchanged. [Confirmed]

### 8.3 Property Relay (bidirectional; `RuntimeProperty` enum)

A keyed registry indexed by the `RuntimeProperty` enum. Each property has one value class — **Int**
(u32), **Bool**, or **Fx** (fixed-point int32). The relay publishes DCP values to the host and
lets either side read/write the store. This is callback `D300` (publish) plus the host→DCP setters
`A200`–`A206` (§5.5). [Confirmed]

**Methods:** `publish(id, u32 val32, u64 val64)` (DCP→host announce; the used width depends on the
property's type); `get_int/get_bool/get_fx`; `set_int/set_bool/set_fx`; `set_prop_dynamic(id, u32)`
(runtime-dynamic subset). Firmware helpers map an id to a name and to a value class, and map a
subset of properties to **idle-detector** slots. [Confirmed]

**Idle-detector sub-enum** (12 entries; each `_enabled` + `_threshold`): `System`, `VRR`,
`VRRBurnInPrevention`, `VRRBurnInPreventionCooldown`, `PowerGateFrontEnd`, `DisplayWall`.
[Confirmed]

**`RuntimeProperty` enumeration — 270 entries, identical set and order on both silicon.**
[Confirmed set/order] The **id** column is the position in the firmware's runtime-property name
table; identical ordering across silicon is strong evidence this is the wire id, but the value is
an ordinal inference, not a recovered constant ([Inferred]). The **type** column is inferred from
naming and the accessor split ([Inferred]).

| Id | Name | Type | Id | Name | Type |
|--:|------|------|--:|------|------|
| 0 | blendOutCSCMethod | Int | 135 | maxCompressedSrcSurfaceWidth | Int |
| 1 | CMDegammaMethod | Int | 136 | asyncSwapDisabled | Bool |
| 2 | DisplayWallFBs | Int | 137 | isFlipDisabled | Bool |
| 3 | DisplayWallConfig | Int | 138 | sclCursorSwapFailed | Bool |
| 4 | DisplayWallTiming | Int | 139 | support2DBL | Bool |
| 5 | SupportsHeadOfLineSwaps | Bool | 140 | supportGPLite | Bool |
| 6 | requestPixelBacklightModulation | Bool | 141 | sharedDisplay | Int |
| 7 | pixelBacklightModulationForceState | Int | 142 | supportHDR | Bool |
| 8 | DPBDriverOverrideLPFControls | Int | 143 | enablePCCTrinity | Bool |
| 9 | DPBDriverLPFControlValue | Int | 144 | enable0DBLCTrinity | Bool |
| 10 | DPBDriverLPFControl2Value | Int | 145 | enablePCC | Bool |
| 11 | DPBDriverOverrideMaxSlopes | Int | 146 | enablePCC0DBLC | Bool |
| 12 | DPBDriverLPFMaxSlopeValue | Int | 147 | enablePCC2D | Bool |
| 13 | DPBDriverMaxSlopeValue | Int | 148 | enableRCA | Bool |
| 14 | DPBDriverOverrideBacklight | Int | 149 | disableRCAWL | Bool |
| 15 | DPBDriverBacklightValue | Int | 150 | bypassPCC2DLed | Bool |
| 16 | enableGammaCorrection | Bool | 151 | PCC2DLedAccelOut | Int |
| 17 | brightnessCorrection | Fx | 152 | PCC2DLedLog | Int |
| 18 | brightnessCorrectionB | Fx | 153 | disablePCC2DBrc | Bool |
| 19 | brightnessLevel | Int | 154 | GCPEnabled | Bool |
| 20 | indicatorBrightnessNits | Int | 155 | GCPManuallyControlled | Int |
| 21 | secureContentFactor | Fx | 156 | GCPStrength | Fx |
| 22 | secureIndicatorFactor | Fx | 157 | ExternalAppleLook | Int |
| 23 | secureIndicatorSDRFactor | Fx | 158 | GCPRangeMapGamma | Fx |
| 24 | indicatorNitsCap | Int | 159 | GCPDisablesTwilightLumaComp | Int |
| 25 | brightnessLevelMA | Int | 160 | GCPBypassesEDRPixels | Int |
| 26 | brightnessLevelIDAC | Int | 161 | disableTempComp | Bool |
| 27 | contrastEnhancerStrength | Fx | 162 | doPODBoost | Bool |
| 28 | brightnessCompensationEnable | Bool | 163 | PODBoost | Fx |
| 29 | temperatureCompensationEnable | Bool | 164 | UIVibrancyBoost | Fx |
| 30 | enableDither | Bool | 165 | UIVibrancy | Fx |
| 31 | enableDarkEnhancer | Bool | 166 | edrScaleinGP | Int |
| 32 | enableADBEColorManager | Bool | 167 | modeBlm | Int |
| 33 | enableWhitePointCorrection | Bool | 168 | PCCNormBrightOut | Int |
| 34 | enable2DUniformityCorrection | Bool | 169 | BLMVLEDManual | Int |
| 35 | enable2DTemperatureCorrection | Bool | 170 | BLMAHOutputFreq | Int |
| 36 | enableDefaultTemperatureCorrection | Bool | 171 | BLMAHMode | Int |
| 37 | defaultTemperatureCorrectionValue | Int | 172 | BLMPLimitCfg | Int |
| 38 | use0DSensorFor2DTemp | Bool | 173 | enableBLMSloper | Bool |
| 39 | TemperatureStructVersion2D | Int | 174 | enableLAC | Bool |
| 40 | digitalDimmingLevel | Int | 175 | BLMAHUPCount | Int |
| 41 | set0DTempSensorValue | Int | 176 | M3DiagsTimeout | Int |
| 42 | uniformity2D | Int | 177 | DisableBConBoot | Bool |
| 43 | enableAmbientLightSensorStatistics | Bool | 178 | BLMPowergateEnable | Bool |
| 44 | enableSubPixelLayoutCompensation | Bool | 179 | allowAnnouncingPTS | Bool |
| 45 | splxMapAwareMode | Int | 180 | enableLinearToPanel | Bool |
| 46 | enablePartialUpdate | Bool | 181 | enableAPT_PDC | Bool |
| 47 | enableLatLeakComp | Bool | 182 | enableAPT_PDC_PM | Bool |
| 48 | enableHighGrayOD | Bool | 183 | enableAPT_PDC_Repeat | Bool |
| 49 | ambientBrightness | Int | 184 | PDCGlobalTemp | Int |
| 50 | enableGamutMapper | Bool | 185 | PDCSettleCount | Int |
| 51 | enableStats | Bool | 186 | PDCSaveLongFrames | Int |
| 52 | maxAvgBpp | Int | 187 | PDCSaveRepeatUnstablePCC | Int |
| 53 | maxPeakBpp | Int | 188 | PDCEntryTime | Int |
| 54 | clockRatio | Fx | 189 | PDCExitTime | Int |
| 55 | debugUInt32 | Int | 190 | PDCAlwaysOnNits | Int |
| 56 | DisplayWallAvailFBs | Int | 191 | PDCContentSwitchCount | Int |
| 57 | vrrDivisor | Int | 192 | pccLumaGammaFactor | Fx |
| 58 | vrrVersion | Int | 193 | PDCCutoffLux | Int |
| 59 | darkBoot | Int | 194 | PDCExitCount | Int |
| 60 | wideGamutPassthrough | Int | 195 | overdriveCompCutoff | Int |
| 61 | enableFiltersNoRewriteMode | Bool | 196 | hgodCutoff | Int |
| 62 | enablePLCMode | Bool | 197 | enableAOT | Bool |
| 63 | enableABG | Bool | 198 | vshHistVal | Int |
| 64 | enableABGDynamicMap | Bool | 199 | contrastEnhancerCorrectionFactor | Fx |
| 65 | selectABGDynamicMap | Bool | 200 | proxScanPlan | Int |
| 66 | disableDBSR | Bool | 201 | proxScanPosition | Int |
| 67 | enableDBM | Bool | 202 | SWSTemp | Int |
| 68 | enableBIC | Bool | 203 | forceAOTARMode | Bool |
| 69 | enableBICUpdates | Bool | 204 | emissionFrequency | Int |
| 70 | enableSBIM | Bool | 205 | supportsAOTPowerSaving | Bool |
| 71 | enableTBIC | Bool | 206 | supportsAOT1HzOptimizations | Bool |
| 72 | enableGPDeringing | Bool | 207 | aotmultiRefresh | Int |
| 73 | enableGPAlphaDiv | Bool | 208 | enableSPUC | Bool |
| 74 | twilightStrength | Fx | 209 | enableQETC | Bool |
| 75 | ammoliteStrength | Fx | 210 | enableTLSC | Bool |
| 76 | ccParamsVersion | Int | 211 | clearUSPUCData | Bool |
| 77 | apriTempOverride | Int | 212 | USPUCDefaultData | Int |
| 78 | apriBrightOverride | Int | 213 | enableBLMAHOutputLog | Bool |
| 79 | enableAPri | Bool | 214 | enableBLMAHStatsLog | Bool |
| 80 | allocateDefaultFramebuffer | Int | 215 | enableBLMStandby | Bool |
| 81 | HardPowerEvent | Int | 216 | enableVUC | Bool |
| 82 | ESDThresholdMS | Int | 217 | enableIRDC | Bool |
| 83 | enableKernelTests | Bool | 218 | IRDCFrameReset | Int |
| 84 | disableDisplayOptimize | Bool | 219 | IRDCFrameMaxResetCount | Int |
| 85 | ConfigureQMSVRR | Bool | 220 | IRDCFrameMinResetCount | Int |
| 86 | enableEven60FPSFrames | Bool | 221 | IRDCEMPEnable | Bool |
| 87 | panicOnHungSwap | Bool | 222 | limit_max_physical_brightness | Int |
| 88 | fakeTconESDEvent | Bool | 223 | enablePTUC | Bool |
| 89 | registerTraceEnable | Bool | 224 | PTUCTempOverride | Int |
| 90 | frameInfoTraceEnable | Bool | 225 | enableRTPLC | Bool |
| 91 | QoSDebug | Int | 226 | enableRTPLCRT | Bool |
| 92 | enablePowerGateDCS | Bool | 227 | enableRTPLCFD | Bool |
| 93 | enableVideoCaching | Bool | 228 | enableRTPLCNitsCap | Bool |
| 94 | dummySystemWantsCaching | Bool | 229 | enableRTPLCPTC | Bool |
| 95 | disableSystemCachingInput | Bool | 230 | enableRTPLCFixedKnee | Bool |
| 96 | enableMockALSSCapture | Bool | 231 | enableRTPLCFDBicMult | Bool |
| 97 | APTFixedRR | Int | 232 | usePodBoostAtRTPLCFDLUT | Bool |
| 98 | AODFixedRR | Int | 233 | enableRTPLCFDPixelModAfterRTTrig | Bool |
| 99 | AODWaitForWalkdown | Int | 234 | RTPLCRecoveryCoeff | Fx |
| 100 | enablePCCLogs | Bool | 235 | enableSWS | Bool |
| 101 | enableAPTLogs | Bool | 236 | enableReplayInVMAOD | Bool |
| 102 | enableAPT_CDFD | Bool | 237 | enablePixelCapture | Bool |
| 103 | enableAPT_PRC | Bool | 238 | PixelCaptureConfig | Int |
| 104 | enableAPTConfigExpired | Bool | 239 | PixelCaptureBlockMask | Int |
| 105 | enableAPT_CA | Bool | 240 | GP0PixelCaptureLocation | Int |
| 106 | BLNitsCap | Int | 241 | GP1PixelCaptureLocation | Int |
| 107 | RTPLCBLNitsCap | Int | 242 | applyBrightnessProps | Bool |
| 108 | RTPLCBLNitsScaler | Fx | 243 | BlendPixelCaptureLocation | Int |
| 109 | RTPLCNitsThresh | Int | 244 | CMPixelCaptureLocation | Int |
| 110 | enableAPTEvents | Bool | 245 | PCCBLCPixelCaptureLocation | Int |
| 111 | APTEventsMask | Int | 246 | WPCPixelCaptureLocation | Int |
| 112 | enableNormalMode | Bool | 247 | BICSPixelCaptureLocation | Int |
| 113 | enableAPTDefaultGray | Bool | 248 | TBICSPixelCaptureLocation | Int |
| 114 | APTDefaultGrayValue | Int | 249 | RTPLCPixelCaptureLocation | Int |
| 115 | StatsTapPoint | Int | 250 | CalCLutPixelCaptureLocation | Int |
| 116 | enableFrameInfoForRepeats | Bool | 251 | CCPixelCaptureLocation | Int |
| 117 | resetAPT_CA | Bool | 252 | GPLiteMaxSrcSz | Int |
| 118 | publishChargeValues | Bool | 253 | disableBLKSFComp | Bool |
| 119 | resetChargeValues | Bool | 254 | enableDisplayTMDpc | Bool |
| 120 | panicOnChargeOOB | Bool | 255 | LTHSaveDispPerfBoostEnable | Bool |
| 121 | panicOnChargeParity | Bool | 256 | enablePowerTrace | Bool |
| 122 | panicOnStuckPolarity | Bool | 257 | AoD1HzPanicThreshold | Int |
| 123 | limitRefreshRate | Int | 258 | enableCalibrationCapture | Bool |
| 124 | midporchRegion | Int | 259 | CalibrationCaptureSubblock | Int |
| 125 | exportCRCAtVBI | Bool | 260 | CalibrationCaptureBlock | Int |
| 126 | IdleCachingMethod | Int | 261 | ADCLLoadAll | Int |
| 127 | enablePCCCabal | Bool | 262 | forceEnableACSS | Bool |
| 128 | pccStabilityThreshold | Int | 263 | supportHWSecureAnimation | Bool |
| 129 | pccStabilityThresholdAOT | Int | 264 | forceSecureAnimationUpdate | Bool |
| 130 | supportICCProfile | Bool | 265 | enablePulseWidthMaximization | Bool |
| 131 | isRGBMultiPlaneSupportEnabled | Bool | 266 | FIRST_GAMUT_CONVERSION_LOCATION | Int |
| 132 | supportsCursorOnSCLregion | Bool | 267 | LAST_GAMUT_CONVERSION_LOCATION | Int |
| 133 | isYUVSupportEnabled | Bool | 268 | dynamicSageLogging | Int |
| 134 | isBlendDisabled | Bool | 269 | supportLFC | Bool |

`FIRST_/LAST_GAMUT_CONVERSION_LOCATION` are range sentinels, not knobs; the `*Location` family
selects a pipeline tap-point (block index) for pixel/calibration capture.

### 8.4 Registry property publish (DCP → host)

The DCP publishes display metadata and capability flags into the host registry by invoking the
framebuffer callback surface (`D550`–`D599`), which resolves on the host to typed setters
(`set_property_int` / `set_property_bool` / `set_property_str` / `set_property_dict`, plus a
whole-table setter). Values are materialized as the matching container (number / boolean / string
/ data / dictionary / array). Addressing is by **registry key string** on the framebuffer node
(no numeric service id). The byte-blob and bit-width forms are backed by `(key, void*, len)` and
`(key, u64, bits)` setters. [Confirmed]

**Published keys** (all [Confirmed] as present; publication via a setter callback [Confirmed] for
the capability/metadata keys, [Inferred] for the remainder):

*Geometry / mode / power:* `DisplayHeight` (int), `DisplayWidth` (int), `DisplayLuminance` (int),
`DisplayClock` (int), `DisplayAttributes` (dict), `DisplayAsleep` (bool), `DisplayPower`
(int/bool), `DisplayRequest`/`DisplayRelease`/`DisplayAllocation` (dict/int), `DisplayTargets`
(array/dict), `DisplayContainerID`/`DisplayModuleID` (int/str), `DisplayPipePlaneBaseAlignment`
(int), `DisplayPipeStrideRequirements` (dict), `IOMFBDisplayRefresh` (int), `IOMFBRect`
(dict/data), `IOMFBNumLayers` (int), `IOMFBUUID` (str/data), `IOMFBStatus` (int).

*Capability flags* (bool unless noted): `IOMFBSupportsHDR10Plus`, `IOMFBSupportsICC`,
`IOMFBSupportsLFC`, `IOMFBSupportsRotation`, `IOMFBSupportsXFlip`, `IOMFBSupportsYFlip`,
`IOMFBSupportsHeadOfLineSwaps`, `IOMFBSupports2DBL`, `IOMFBSupportsGPLite`,
`IOMFBSupportHWSecureAnimation`, `IOMFBSupportRelbufInfoCb`, `IOMFBCompressionSupport`,
`IOMFBAsyncRGBCompressionSupport`, `IOMFBMaxCompressedSizeInBytes` (int),
`IOMFBCompressedSourceSurfaceWidth` (int), `IOMFBRGBMultiPlaneSupportKey`, `IOMFBYUVSupportKey`,
`IOMFBRGBSupportsCursorOnSCLregionKey`, `IOMFBBlendDisabledKey`, `IOMFBAsyncSwapDisabledKey`,
`IOMFBSCLCursorSwapFailedKey`, `IOMFBDisableFlip`, `IOMFBPPASupported`, `IOMFBWPASupported`,
`IOMFBSwapAppleLookSupported`, `IOMFBWideGamutPassthrough`, `IOMFBAllowAnnouncingPTS`,
`IOMFBMaxSrcPixels` (int), `IOMFBMaxVTPowerData` (int), `IOMFBScalingLimits` (int/dict),
`IOMFBHotplugKeysChangedKey`, `IOMFBIntDcpUsedForExtWhenLidClose`.

*Brightness / color / calibration state:* `IOMFBBrightnessLevel`, `IOMFBBrightnessLevelMA`,
`IOMFBBrightnessLevelIDAC` (int), `IOMFBBrightnessCompensationEnable`,
`IOMFBTemperatureCompensationEnable` (bool), `IOMFBDigitalDimmingLevel` (int),
`IOMFBContrastEnhancerStrength` (int/fx), `IOMFBIndicatorBrightnessNits`, `IOMFBIndicatorNitsCap`
(int), `IOMFBSecureContentFactor`, `IOMFBSecureIndicatorFactor`, `IOMFBSecureIndicatorSDRFactor`
(fx), `IOMFBDoPODBoost` (bool), `IOMFBPODBoost` (fx), `IOMFBApplyBrightnessProps` (bool),
`IOMFBStatsTapPoint`, `IOMFBHGODLocation` (int), `IOMFBTestBacklightDimValue` (int),
`IOFBSAGammaLUT` (data).

*Configuration blobs* (data/dict): `IOMFBELVSLUTs`, `IOMFBPRCLUTs`, `IOMFBPtucTLSLuts`,
`IOMFBUPCLData`, `IOMFBIRDCData`, `IOMFBLatLeakConfig`, `IOMFBALSSConfig`,
`IOMFB_2D_Temp_State_Data`, `IOMFBTemperatureMonitorContext[_v2/_v3]`,
`IOMFBPCC2DLEDData_Flow_in`, `IOMFBPCC2DLEDData_Flow_out`, `IOMFBFrameInfoBuffer`,
`IOMFBBICSType`, `IOMFBParameter_adaptive_sync`.

*Idle-detector keys:* `DisplayWallIdleEnabled`, `DisplayWallIdleThreshold`, `DisplayWallConfig`,
`DisplayWallTiming`, `DisplayWallFBs`, `DisplayWallAvailFBs`.
---

## 9. Peripheral display sub-services (AFK / EPIC)

The peripheral services ride the AFK/EPIC transport (§3.6–§3.7) rather than the DCP RPC link.
Each is a DCP-side EPIC service with a matching host-side proxy. Three message layers are used:
the RTKit command RPC (synchronous/asynchronous gated commands); the AFK/EPIC endpoint
(command/report/response); and an **AV-IPC** request/reply-plus-callback protocol used by the
host proxies that mirror DCP-hosted AV/DP/DPTX objects. A shared proxy base provides open/close,
send/validate/handle-message, forwarded-message routing, and an event-log control surface; large
property blobs transfer in chunks via a start/chunk/end sequence. On `t8140` an additional typed
trusted-IPC layer is present (§12). [Confirmed]

### 9.1 EPIC service inventory

Service registration names (literal `-epic` tokens). Presence: ● = present, ○ = absent.

| Service (registration name) | Role | t8132 | t8140 |
|-----------------------------|------|:--:|:--:|
| `dcpexpert` | Central configuration/health "expert": timing, health-stats, backlight-calibration/compensation store, board/chip config | ● | ● |
| (display service, on `disp0`) | Top-level display service (mode/power/HPD surface to host) | ● | ● |
| framebuffer proxy service | Framebuffer proxy service | ● | ● |
| AOP display manager | Always-on-processor display management | ● | ○ |
| `dcpdsb` | Display SPI backlight bus relay (READ/WRITE/SEQ/STATUS packet ops, plus test variants) | ● | ○ |
| MIPI/DSI panel controller | Internal MIPI/DSI panel controller | ○ | ● |
| `cb-ap-to-dcp-service` | Backlight/brightness control-bus bridge (nits↔brightness-value↔mA, calibration load, PLC control, PMIC access) | ● | ● |
| CB service / CB-AOP service (iOS variants), `cbrootservice` | Control-bus service variants and root anchor | ○ | ● |
| `dcpav-epic-response` | Base AV EPIC client / response channel | ● | ● |
| `dcpav-controller` | AV controller: link/sink lifecycle, dispatch root | ● | ● |
| `dcpav-service` | Per-sink AV service (EDID, DisplayID parse, VRR/QMS caps) | ● | ● |
| `dcpav-device` | AV device abstraction | ● | ● |
| `dcpav-video-interface` | Video interface: mode/color/timing element arrays (simple / slice / split-display flavors) | ● | ● |
| `dcpav-audio-interface` | Audio interface: channel layout, EDID UUID, product attributes, audio DMA | ● | ● |
| `dcpav-audio-arc` | HDMI Audio-Return-Channel RX device | ● | ● |
| `dcpav-cec-interface` | HDMI-CEC interface | ● | ● |
| `dcpav-power` | AV rail/power controller | ● | ● |
| `dcpav-sac` | Remote speaker/audio-amp calibration controller | ● | ● |
| `dcpdp-controller` / `dcpdp-service` / `dcpdp-device` | DisplayPort protocol family | ● | ● |
| `dcpdptx-port` / `dcp-lpdptx-port` | DisplayPort transmitter remote port / low-power port | ● | ● |
| DPTX HDCP interface / auth session | HDCP interface + authentication session over DPTX | ● | ● |
| power service / `powerlog` / `system` / `static` / `test` | Power, power-logging, system, static-property, and diagnostics services | ● | ● |
| `aop-test-control` | AOP test-control channel | ○ | ● |

The peripheral host proxy modules themselves are identical across silicon; silicon differences
are expressed by which subclasses instantiate and by the `t8140`-only secure/trusted additions
(§12). [Confirmed]

### 9.2 DisplayPort transmitter (DPTX)

A DCP-facing remoting layer represents each DPTX port to the DCP (upstream-facing and
low-power/internal variants) and executes DCP requests against a transmitter hardware driver;
hardware events are pushed back to the DCP. The firmware also carries **Multi-Stream Transport
(MST)** support and an adaptive ("agile") link-rate feature. [Confirmed]

**Port lifecycle / connection state:** activate/deactivate, connect-to (bind to a downstream
connection), validate-connection, set-resource-available, inactive-sink-detected,
force-hotplug-detect, hotplug-detect-change-occurred, interrupt-request-occurred (sink IRQ /
DPCD ESI), device-not-responding / device-busy-timeout / device-not-started, will/did-change-
link-configuration, set-display-hints, get/set virtual-device-mode, synchronous DPTX power
transition. [Confirmed]

**Link configuration:** get/set max/min/current link rate and lane count; get/set active lane
count; per-lane drive settings (voltage swing / pre-emphasis) with min/max query; spread-spectrum
(downspread) enable/query; enhanced-framing enable/query; lane mapping/polarity; link-bandwidth
accounting; and the adaptive ("agile") link-rate table/range operations. Link rates span the
RBR/HBR/HBR2/HBR3 and UHBR classes. [Confirmed]

**Link training:** train / retrain / will-train / did-train; a training state machine driving
clock-recovery then channel-equalization phases with per-phase interval delays (from the DPCD
training AUX interval); training-pattern selection (TPS1–TPS4 class) and combined
pattern+drive-settings programming; link-training-data negotiation (rate + lanes + settings) with
clamping and recommendation; fast-link-training capability (eDP); need-for-training evaluation and
link validation; lane/alignment status readback; PHY quality/compliance test patterns; scrambling
inhibit (compliance); and next-lower-rate / min-agile-rate fallback selection. [Confirmed]

**AUX / DPCD / DDC:** native AUX DPCD read/write; I2C-over-AUX read/write (EDID/DDC); DDC
transaction start; low-level AUX transaction setup/commit (with retries)/abort; AUX engine init,
reply-timeout, and access-window control (ALPM-aware); AUX arbitration (wait/signal for
availability); secure-AUX address filtering (HDCP). [Confirmed]

**Secondary-data packets / video stream:** start/stop info-frame and video-info-frame; SDP slot
management; enable/disable video stream; configure video mode / main-stream attributes / M-N link
clock ratio / frame interval / color / DSC / BIST; DSC picture-parameter-set (PPS) programming;
video-link lifecycle (prepare/start/stop/complete with will/did variants); vsync handling. [Confirmed]

**FEC / PHY power / low-power:** Forward Error Correction support/enable/ready/symbol/error-count
(required with DSC); Panel Self Refresh enter/exit; Advanced Link Power Management (ALPM)
enable/disable/start/stop with parameter validation and wakeup/standby timing calculation; Type-C
lane orientation; PHY calibration load; Alternate Scrambler Seed Reset (eDP). [Confirmed]

DPTX hardware/transport variants: a main register/link/AUX engine; a converged-IO
(Thunderbolt/USB4) tunneled variant; a low-power variant for the internal/eDP panel (ALPM, panel
calibration); a family-specific low-power variant with explicit PHY/PLL/AUX power gating; an
AUX-only variant (no main-link video); and crossbar upstream-facing-port shims. [Confirmed]

### 9.3 DP / AV family (mode / EDID / color / audio / HDR / DSC)

Requests are host→DCP unless noted as a DCP→host callback. Link operations are parameterized by a
link-source (which pipe/source) and a link-type (video/audio/…). [Confirmed]

**Service level:** start/stop link; get link data; copy EDID; copy CEC physical address (from
EDID); set virtual EDID; direct DDC (I2C) read/write; start/stop info-frame (AVI/audio/HDR);
set HDR static metadata (ST 2086 / CTA-861 HDR info-frame); get/set content-protection
capabilities and policy; query chosen protection / authenticated content-type; query encryption
and protection status. [Confirmed]

**Video interface:** video-link lifecycle (prepare/start/update/stop/complete with will/did
variants); enumerate supported timing elements, color elements, and display attributes; select
active timing/color element (by index, per source); set link mode; query link status/transport/
data; color-dither removal; geometry/white-point/test overrides (bounds, rotation, virtual
temperature, test mode). [Confirmed]

**Audio interface:** audio-link lifecycle; transfer audio samples via a memory descriptor with a
completion callback (DMA delegate); enumerate audio caps, channel layouts, product attributes,
EDID UUID; A/V-sync latency and safety-offset accounting; supported audio coding types (PCM/
compressed); audio port id. [Confirmed]

**Device / controller:** per-device link/protection queries; DDC read/write; power control
(get/set power, sleep/wake display, synchronous power transition); force hotplug detect; frame-CRC
capture (compliance/self-test); virtual/headless device mode. [Confirmed]

**DSC (Display Stream Compression):** read sink DSC capabilities from DPCD; query whether DSC is
usable for a color mode; enable/monitor DSC and clear DSC errors; program DSC + PPS in the
transmitter; handle a DSC status-change IRQ. [Confirmed]

**Color / colorimetry / HDR:** extended (VSC-SDP) colorimetry support; downstream max
bit-depth; RGB/YCbCr color-format conversion (configure/validate/support-query) in the converter
path; output bit-depth conversion; dither-removal handling; HDR static metadata info-frame.
[Confirmed]

### 9.4 HDMI conversion (DP → HDMI)

The DP branch protocol-converter (PCON) path fronts an HDMI 2.x sink through a converter, modeled
by a DP1.3+ service surfaced through the AV service and the HDMI port controller; it is configured
over DPCD. The firmware includes drivers for external DP→HDMI converter/retimer silicon
(a MegaChips-class converter with a full DPCD register set including HDMI TMDS PHY swing/
pre-emphasis/edge-rate/termination and an HDCP message transport, and Parade-class converters with
an HDCP key-status register) beneath a generic DP-to-HDMI layer (HDMI PHY configuration, Apple-
HDMI-extension detection). [Confirmed]

**PCON configuration / scrambling:** initialize the converter (defaults); enable/disable the
protocol converter; HDMI port enable / HPD / recovery; HDMI 2.0 scrambling control (incl.
autonomous-scrambling-disable) with capability query; pass-through EDID handling; HDMI-over-tunnel
capability query. [Confirmed]

**FRL (HDMI 2.1 Fixed Rate Link):** get/set FRL max/min rate; get negotiated FRL params; FRL
retrain and autonomous recovery; TMDS/FRL character-error telemetry and rate monitoring; HDMI
link-status monitoring enable/query and link-status-change IRQ. The FRL rate ladder is:
FRL-disabled (TMDS), 9 Gbps (3G×3), 18 Gbps and 24 Gbps (6G), 32 Gbps (8G×4), 40 Gbps (10G×4),
48 Gbps (12G×4). The FRL training state machine and autonomous-mode timeout handling reside in the
DCP-side AV service; lane-scramble-seed control governs FRL/TMDS scrambling. [Confirmed]

### 9.5 Output crossbar / mux and USB-C DP-alt mode

An output crossbar routes DPTX outputs (upstream-facing ports, **UFP**) to physical output ports
(downstream-facing ports, **DFP**). The switch fabric establishes/tears down UFP↔DFP routes; a
connection manager holds a persistent connection map. [Confirmed]

**Routing / switch:** connect/disconnect, apply/get current state, display-allocation done; port
registry (register/unregister/get, endpoint-name query); candidate selection and routing policy
(iterate available/connected UFPs, iterate peer-DFP candidates, make inactive UFP available, score
a connection); UFP allocation including peer/paired UFP for split-display; pipe configuration
(current/expected pipe config, enable display allocation, main-port query); internal-panel pipe
topology queries; Type-C DP mux-selector programming; role/rate constraints; persistent
connection mapping (create/get/set/serialize); tiled (multi-tile) display handling. [Confirmed]

**Output port taxonomy (DFP):** native DP over a Type-C/ATC port; USB-C **DisplayPort Alt-Mode**
port; HDMI-via-ATC (converter path, §9.4); Thunderbolt/USB4 **DP-tunnel** in/out adapter ports
(tunnel active/inactive/state-change handling, tunnel-use refcounting, destination CIO port/
product queries); silicon-specific DPTX (UFP) port shims; and the HDMI port controller with its
PCON. [Confirmed]

The USB-C DP-alt-mode mux is the **ATC DP crossbar** (with per-silicon subclasses), which programs
the lane mux and coordinates with the Type-C controller stack and the power-mux hardware
abstraction; charger muxing is a separate function. Per-port control (activate/deactivate,
display request/release, bring connection up / take down, port-message handling,
hotplug-detect-change, interrupt-request, link-rate, lane count, drive settings, downspread,
switch sleep/wake, connection validation, initial-HPD check, force-hotplug-detect, native-HPD
use) mirrors the DPTX port surface. [Confirmed]

### 9.6 HDCP

Three cooperating layers: host transmitter controllers (a base plus HDCP 1.x and HDCP 2.x); a DCP
remoting proxy carrying the auth session and interface over AV-IPC; and a message-transport /
auth-session framework (DP HDCP message transport, HDCP 2 local transport, a converter-chip HDCP
transport, and the auth-session engines for HDCP 1 transmitter, HDCP 2 transmitter/DP-transmitter/
receiver), plus a crypto interface parameterized by protocol and device-role. The **Secure Enclave
(SEP)** holds HDCP secrets and performs cryptographic verification, reached through a dedicated SEP
HDCP endpoint. [Confirmed]

**Link protection (transmitter):** protect/unprotect link; enable/disable HDCP block;
negotiate/abort/validate authentication; HW-block power (startup/shutdown); encryption-active
query and enable-change IRQ; HDCP 2 content-type (Type-0/Type-1) capability and query; HDCP 2
error condition; mode bracketing / policy gate. [Confirmed]

**HDCP 1.x key/auth data:** transmitter/receiver KSVs; session random; R0/R0′ verification;
M0 secret; V′ (KSV-list signature) for repeaters; device-key load; downstream KSV-list wait.
[Confirmed]

**HDCP 2.x auth-session flow** (proxy ↔ DCP ↔ SEP callbacks): open/close session; state
transitions (negotiating/authenticated); content-protection-desired; receiver-connected;
read-random (rtx/rn); H′ availability (AKE); pairing-info availability (AKE km); receiver-ID-list
availability (repeater); validate KSV / KSV-list (SRM/revocation); content-type capability/query.
The session runs on a gated event source; capability is gated by protocol + device-role.
[Confirmed]

**HDCP 2.x message building / crypto / SEP:** AKE init / send-cert / no-stored-km; session-key
exchange (Eks); content-type over DP; repeater auth (send-ack, stream-manage, receiver-ID-list,
stream-ready) with downstream topology; verification-value compute/verify (H/L/M/V); key material
generation/exchange (encrypted km/ks, rtx/rrx/rn/riv, send ks-riv over DP); pairing-store save/
load/clear and certificate read/write; SEP request dispatch with out-of-line buffer transfer to/
from the SEP; KSV-list/topology validation. Relevant enumerations: protocol type, message-transport
type, transmitter version, topology, auth-error cause, and max content-type; tunables select the
auth policy and a protocol mask. The firmware distinguishes HDCP 1.x-capable and HDCP 2.2-capable
sinks (no 2.3 evidence). On `t8140` a fuse mask fuse-gates HDCP, tied to the Secure Monitor
subsystem (§12). HDCP is identical across silicon. [Confirmed]

### 9.7 Backlight

Backlight is not part of the DP/DPTX peripheral services. The DCP delivers brightness through the
framebuffer pipeline down to a backlight function object, which drives the panel over **SPMI**;
PMU involvement is via the charger, and the system management controller is consulted only to
decide the initial backlight state. [Confirmed]

**DCP↔host bridge (framebuffer pipeline):** create/get/match the backlight service link;
enable backlight messaging (with gated variants); emit brightness to the PMU backlight nub over
SPMI; read/write brightness via the system-power/lighting-control path; brightness-correction
factor; panel-brightness limits. [Confirmed]

**Host backlight controller / PMU delivery:** apply level / enable (gated); pixel-clock-
compensation update and DCP-power-assertion coupling; unit conversions (nits↔DAC, nits↔milliamps,
nits↔slider, calibrated-nits) from a calibration table; transport-specific delivery selected by
platform (SPMI PMIC, LM, PWM, DWI); PMU-side backlight path. Firmware-side the DCP exposes
brightness-set, brightness-config-set, swap-brightness-set, and swap-brightness-limit-set, plus a
nits↔brightness-value↔mA conversion utility with a calibration table targeting an external
backlight PMIC. Per-die/per-pipe brightness/LED-compensation calibration constants (brightness-
response curve; LED-accelerator brightness clip/round/out-mask) are applied inside the pipeline,
distinct from the SPMI delivery path. [Confirmed]

On the `t8140` internal-panel laptop a secure display-pipe plus an ambient-light/brightness
controller form a trusted/exclave-backed stack (privacy indicators, secure-session health, NVRAM
display-post state, MMIO access, telemetry; UI/ambient brightness, display cross-talk calibration,
ambient-light-sensor calibration) reached over the trusted-IPC layer (§12). This corresponds to
the DCP firmware's AOP-driven auto/ambient-brightness additions on `t8140`. [Confirmed]

### 9.8 Timing controller (TCON)

The internal-panel timing-controller family is present and identical on both silicon (exercised on
the laptop, which has an internal panel). It comprises a base TCON, a DCP-side TCON, a
DisplayPort-attached TCON, and Parade-class timing-controller drivers (with I2C / I2C-PLS / SPI
transport variants, some with a rail controller). TCON commands ride an AFK endpoint to the DCP
(enqueue command with a packet type + component; handle response). The device-access surface:
read/write, memory read/write, register read/write, bridged I2C and SPI read/write/erase, I2C
transactions, EEPROM read/write/erase (with block-protection modify), access prepare/complete,
device protection, TCON-API read/write, descriptor query, read/write-protection query, and
device-tree-nub matching. [Confirmed]

### 9.9 Typed identifiers used across the sub-services

The following typed identifiers gate the named operations; the types exist [Confirmed], but their
value sets are not fully recoverable from symbols alone:

link-training pattern; PHY quality/compliance pattern; link-training-data bundle (rate + lanes +
drive); per-lane drive settings; lane map/polarity; sink/branch device id and IEEE OUI; link-type
and link-source selectors; negotiated video/audio/link data blocks; info-frame descriptor; HDMI
FRL rate and FRL training data; content-protection capabilities and policy options; encryption and
protection status; frame-CRC type and data; HDCP protocol and device-role; DPTX port attributes and
port address (mux addressing); display hints and display allocation.
---

## 10. Data structures

Described functionally; field offsets and layouts are intentionally omitted. All are
[Confirmed] to exist.

- **SwapRequestRecord** — one atomic display update: participating source surface(s) per layer,
  source/destination rectangles, per-layer blend/alpha and transform, background/fill color,
  swap flags, and the requested swap id. Carried by the swap-submit calls.
- **TimingParameters** — a display timing mode: active/blank pixel and line counts, pixel clock,
  sync polarities, interlace and related attributes. Used in mode enumeration/matching.
- **ColorParameters** — the color configuration paired with a timing: pixel encoding, bit depth,
  colorimetry/range, dynamic-range attributes.
- **VideoTimingData / VideoColorData** — AV-layer views of a mode's timing and color descriptors,
  resolved by id and merged during mode insertion.
- **DSCCapabilities / DSCSliceHints** — Display Stream Compression capability set and slice/rate
  hints.
- **GammaTable** — the per-channel gamma / transfer-function table (a large InOut transfer).
- **FixedPointColorMatrix** — a fixed-point color-transform matrix; the 3×3 form also appears as a
  9-element 64-bit array in the framebuffer matrix get/set. **MatrixLocation** names the pipeline
  stage a matrix applies to; **MatrixFunction** selects its role in a swap.
- **DisplayArea / OverscanSafeRect / MirroringCapability** — geometry and capability descriptors
  for the digital-out group.
- **ParameterName** — the enumeration keying the Parameter Store (§8.1); the payload is a `u64[]`.
- **RuntimeProperty** — the enumeration keying the Property Relay (§8.3); value class Int/Bool/Fx.
- **GainMapDescriptor** — HDR gain-map descriptor for gain-map creation.
- **TiledDisplayInfo** — tiled/multi-pipe display topology (tile layout and per-tile parameters),
  delivered on the hotplug callback so the host can build tiled mode entries.
- **SwapCompletionRecord** — per-swap completion record: swap identity, completion status/error,
  and fields the host uses to retire the swap and release resources.
- **SwapInfoRecord** — per-swap info/statistics record (frame timing / CRC / statistics) for
  notify clients and the CRC path.
- **NotificationRequest / NotificationRequestArgs** — a registered notification-client handle and
  the argument payload for a delivered notification (used by add/remove-notification, the
  notify-mask, and HDCP send/reply).
- **NotificationType** — enumeration selecting a notification class for masking/subscription
  (swap-info, frame-info, CRC, …). Value set [Uncertain].
- **IdleCachingState / IdleCachingMethod** — the idle/self-refresh caching state (passed to
  idle-fence create) and the caching method in effect.
- **SwapIntent** — enumeration classifying a swap's intent, delivered with swap-complete-intent.
  Value set [Uncertain].
- **PresentationTimeTransaction** — present-timing transaction record for the frame-swap /
  presentation-time path.
- **HDCPDownstreamState** — HDCP downstream/topology state returned to the DCP on request.
- **SharedEvent** — shared fence/event object the DCP signals via the event/shared-event callbacks.
- **RegisterStream** — an opaque register/command stream handle passed to several pipe methods.
- **Status** — the signed 32-bit result code used by the RPC transport and most callbacks.
- **AV-IPC message** — the request/reply/callback envelope for all DPTX/AV/CEC/HDCP proxy traffic.
- **Backlight / ambient-light configuration records** — configuration structures delivered to the
  backlight-control and ambient-light-sensor configuration callbacks.

---

## 11. Firmware container formats

The DCP firmware ships as a signed image payload (LZFSE-compressed). Two container formats occur
across the two silicon variants:

**Flat preload executable (`t8132`).** The decompressed payload is a position-independent, no-
undefined-symbols preload executable image mapped directly by the loader — a monolithic RTKit
image. [Confirmed]

**Bundle container (`t8140`).** The decompressed payload is a packed multi-segment bundle:

- A fixed header identified by a 4-character magic (`"BUND"`, stored byte-reversed), a format
  kind, and a segment count.
- **11 segments**, each described by a fixed-size descriptor carrying a flags word, a
  4-character segment tag (stored byte-reversed), and the segment's location and length within
  the payload. A separate header table carries per-segment load virtual addresses.
- Segment tags decode to a kernel/RTKit/user privilege split: kernel text+data (`ktxt`/`kdat`),
  RTKit + DCP application text+data (`rtxt`/`rdat`, the bulk of the image), user text+data
  (`utxt`/`udat`), shared read-only and read-write data (`dsro`/`dsrw`), an embedded secondary
  preload image (`nold`), user boot data (`ubdl`), and a runtime log buffer (`oslg`).

The bundle packages a microkernel-based runtime (a root task compartment over the microkernel)
rather than a single flat image; this is a packaging/OS-model difference, not an RPC ABI
difference. [Confirmed]

**Firmware identification.** Both images report the same RTKit release (`3255.160.4`, RELEASE) and
the same DCP application version (`AppleDCP-1041.120.7~2508`). `t8140` additionally reports a
microkernel root-task build. [Confirmed]

---

## 12. Silicon-specific differences (`t8132` vs `t8140`)

The RPC application ABI — the transport, the Call primitive, serialization, the framebuffer /
pipeline / property / service / memory-relay method surfaces, and all RTKit base services — is
**identical** across the two silicon variants except as listed here.

| Aspect | `t8132` (M4, mac16g) | `t8140` (A18 Pro, mac17p) |
|--------|----------------------|---------------------------|
| Firmware container | Flat preload executable (monolithic RTKit) | Bundle container over a microkernel (§11) |
| RTKit / DCP version | `3255.160.4` / `AppleDCP-1041.120.7~2508` | identical (plus a microkernel root-task build) |
| DCP RPC endpoint (0x37), channels, shmem, AFK/EPIC | present | present (identical) |
| Extra endpoints | — | `mipi`, `mipitool` (internal MIPI/DSI panel), `tb` (Thunderbolt/USB4 display), `comms`, `cbservice`, `monitor`, `test` |
| Extra EPIC services | — | internal-display service, CB (iOS-variant) services + CB root, AOP test-control; **no** SPI backlight-bus (`dcpdsb`) and **no** AOP display-manager service |
| SoC Display-Pipe call range | `A000`–`A046`; includes 4 real-time-bandwidth calls (`A020`–`A023`) | drops the 4 real-time-bandwidth calls, compacting the numbering below `A024` (e.g. blending-ECO-present is `A024` on `t8132`, `A020` on `t8140`) |
| SoC-pipe callback range | `D000`–`D008` (9) | `D000`–`D007` (8) |
| Shared framebuffer/pipeline call & callback surfaces | — | numerically identical to `t8132` |
| Trusted / typed IPC | — | an additional typed trusted-IPC layer (trusted/untrusted/exclave endpoints) with generated message/result structs; interfaces include a trusted display-power/run-mode notifier, a DPTX secure-monitor and DPTX debug channel, a brightness-utility PMU controller, a display-health monitor, and secure display-pipe interfaces |
| DPTX security | — | a Secure Monitor subsystem (secure GPIO, secure-DPTX-enable), an HDCP fuse mask, and a cover-glass-serial query |
| Secure display pipe / brightness | — | a secure display-pipe (privacy indicators, secure-session health, NVRAM post-state, MMIO, telemetry) and an ambient-light/brightness controller (UI/ambient brightness, cross-talk calibration, ALS calibration) with a calibration loader |
| Backlight | external backlight PMIC register control | additionally AOP-driven auto/ambient brightness, ambient/color-sensor stats, always-on-display ramps, flip-book brightness curves, EDR handling, cross-talk color/health telemetry |
| Panel / topology | external displays only | internal MIPI/eDP panel present → low-power DPTX + timing-controller path exercised |

The peripheral host proxy modules (DPTX, DP/AV, crossbar, HDCP, timing controller, SPMI backlight)
are byte-identical across silicon; the differences above are the extra `t8140` endpoints/services,
the compacted SoC-pipe numbering, the container format, and the `t8140`-only trusted/secure stack.
[Confirmed]

---

## Appendix A — Confidence summary and open items

**Confirmed (directly evidenced in this build):**
the transport layering; the five channel classes and the
concurrent-stream model; the shared-memory/DART model and the memory-descriptor registry; the AFK
ring handshake and the EPIC packet header and service lifecycle; the Call primitive and the FourCC
tag namespaces; the complete host→DCP call enumeration with per-endpoint tags; the DCP→host
callback tag *ranges* and counts; the reentrancy/gating model; 4-byte marshalling granularity, the
inline directional-pointer model, the per-pointer null-mask array, and the type table; the
0x1000-byte serialized-dictionary region and the binary + XML object representations; all four KV
mechanisms including the Host Service Relay entry points/keys, the 270-entry RuntimeProperty name list
(set and ordering), and the published registry keys; the peripheral sub-service inventory and
operation surfaces; the two firmware container formats and versions; and the silicon differences.

**Inferred (strong indirect evidence):**
the RTKit system-endpoint numeric tags and the DCP RPC link endpoint tag `0x37` (names/roles
confirmed; numeric tags are the established convention, not code constants in these images); the precise bit
widths inside the 16-bit link header; the in-band stream count (~4/direction); the RuntimeProperty
numeric ids (name-table ordinals) and value classes (Int/Bool/Fx from naming); object/handle
arguments being reduced to identifiers; large out-of-line buffers passed by mapped reference;
several higher-level host entry points fanning out to other tags rather than issuing their own.

**Uncertain (plausible, weak evidence):**
the individual number-to-handler binding *within* each callback range; per-service EPIC numeric
selector tables; per-object Host Service Relay numeric service-id values; the specific omitted
SoC-pipe callback on `t8140`; the value sets of several enumerations (notification type, swap
intent); whether any method transmits a Bool as a fully-defined 32-bit value; and the exact store
boundary for a subset of refresh/brightness display keys (Parameter Store vs. Property Relay vs.
block/property paths).

No binary offsets, addresses, or numeric values were fabricated; every quantitative claim is
either recovered or explicitly marked as an inference.
