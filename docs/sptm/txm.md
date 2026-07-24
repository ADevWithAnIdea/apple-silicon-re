# TXM SPTM Dispatch

This document describes the minimal permissive TXM shim tested with macOS 26.x
on T8140. The configuration reached WindowServer and the login screen.

Unlike the other SPTM tables, we make no attempt to reproduce or describe
TXM's security functionality. The goal is only to provide the minimal ABI and
state required for macOS to boot. The shim disables the code-signing monitor
and leaves code-signature validation and policy enforcement to XNU and AMFI.

Understanding of how TXM is used can be found in the open XNU source.
Information about the XNU-visible TXM data structures is available in the KDK.

- **Domain 2, table 0** `TXM/XNU` — selectors 0..51

## 1. Calling Convention

The packed dispatch word is described in the top-level README. TXM table 0 has
one additional convention: `x0` is not a selector argument. It is the physical
address of a 16 KiB TXM thread-stack page. Selector arguments begin in `x1`.
The constant handlers described here ignore their arguments.

XNU reads the result from `TXMSharedContextData_t` at the last 1 KiB of the
stack page. Given `shared = x0 + 0x3c00`, write:

| offset from `shared` | size | value |
|---:|---:|---|
| `0x08` | 8 | TXM return code |
| `0x10` | 1 | return-data type, always 0 for inline words |
| `0x18` | 8 | number of return words, 0..6 |
| `0x20` | 48 | six 64-bit word slots; zero unused slots |

Clean the written cache lines to the point of coherency, issue the normal
barrier, and wake waiters before returning to XNU.

The shim uses these TXM return codes:

| name | value |
|---|---:|
| `SUCCESS` | 0 |
| `GENERIC` | 1 |
| `NOT_FOUND` | 8 |
| `NOT_PERMITTED` | 38 |
| `NOT_SUPPORTED` | 41 |

## 2. Boot Handoff

Some TXM information is passed in the XNU bootstrap structure:

| object | required contents |
|---|---|
| thread-stack pointer array | one VA per boot CPU, each naming a thread-stack page |
| thread-stack pages | one zeroed 16 KiB page per boot CPU |

Pass the thread-stack pointer array and boot CPU count through the corresponding
XNU bootstrap fields. XNU translates an acquired stack VA to a PA and supplies
that PA in `x0` on each call.

No endpoint-specific TXM data page is included in the bootstrap structure.

## 3. Endpoint Gating

Selector 2 reports that monitor-backed code signing is disabled while
separately reporting developer mode enabled. This disables the code signing
monitor paths in XNU.  The public CSM wrappers consequently return locally
without issuing selectors 17..25, 32..34, or 38..44. Other endpoints are also
disabled:

- no monitor provisioning-profile object exists for selectors 18..21;
- no monitor code-signature object exists for its validation, association, or
  cleanup paths;
- no monitor entitlements context exists for selector 43.

The other endpoint families remain independently reachable, but hardware
testing showed that they also need only constant responses.

## 4. Selector Table

`[a, b, ...]` denotes the exact inline return words written to the stack record.
A successful response without brackets has zero return words. An error
response also has zero return words.

The `trivial` column has three values:

- `yes`: the handler always returns the listed constant response;
- `no`: selector 2 returns the only structured object;
- `not called`: the current XNU call graph does not issue the selector under
  this configuration; the listed response is defensive.

XNU checks the successful word count for selectors 1, 2, 3, 4, 12, 15, 19,
23, 25, 27, 32, 35, and 43, so preserve the tuples exactly. Selector 12 must
return `SUCCESS [1]`; `NOT_SUPPORTED` fails a required trust-cache load. The
word is only a nonzero reclaim indication. Selector 35 must likewise return a
nonzero word; XNU retains the sentinel but the minimized call graph never
dereferences it.

| ID | selector | minimal response | trivial | shim behavior |
|---:|---|---|:---:|---|
| 0 | ResumeThread | `GENERIC` | not called | Reserved SPTM-facing selector. |
| 1 | GetLogInfo | `SUCCESS [0, 0, 0]` | yes | Disables TXM log draining. |
| 2 | GetCodeSigningInfo | `SUCCESS [txm_info_page, txm_info_page + 0x318, 0, 0, 0, txm_info_page]` | no | Returns the page described in section 5. |
| 3 | GetTrustCacheInfo | `SUCCESS [0, 0, 0, 0]` | yes | Advertises no static trust-cache capabilities. |
| 4 | GetBuildVariant | `SUCCESS [0]` | yes | Reports a release build. |
| 5 | EnterLockdownMode | `SUCCESS` | yes | |
| 6 | EnableRestrictedMode | `SUCCESS` | yes | |
| 7 | GetSecureChannelAddr | `NOT_SUPPORTED` | yes | No secure-channel resource is provided. |
| 8 | UpdateDeviceState | `SUCCESS` | yes | |
| 9 | CompleteSecurityBootMode | `SUCCESS` | yes | |
| 10 | AddFreeListPage | `SUCCESS` | yes | Acknowledge and ignore the donated page. |
| 11 | GetFreeListPage | `NOT_SUPPORTED` | not called | No callsite exists in the current XNU tree. |
| 12 | LoadTrustCache | `SUCCESS [1]` | yes | Report a nonzero reclaim indication; ignore inputs. |
| 13 | UnloadTrustCache | `SUCCESS` | yes | |
| 14 | QueryTrustCache | `NOT_FOUND` | yes | Report every supplied CDHash absent. |
| 15 | QueryTrustCacheForREM | `SUCCESS [0]` | yes | Reports zero REM permissions. |
| 16 | CheckTrustCacheRuntimeForUUID | `NOT_FOUND` | yes | |
| 17 | RegisterProvisioningProfile | `NOT_SUPPORTED` | not called | Gated by disabled CSM. |
| 18 | TrustProvisioningProfile | `SUCCESS` | not called | No monitor profile lifecycle. |
| 19 | UnregisterProvisioningProfile | `SUCCESS [0, 0]` | not called | No monitor profile lifecycle. |
| 20 | AssociateProvisioningProfile | `SUCCESS` | not called | No monitor profile lifecycle. |
| 21 | DisassociateProvisioningProfile | `SUCCESS` | not called | No monitor profile lifecycle. |
| 22 | RegisterCodeSignature | `NOT_SUPPORTED` | not called | Gated by disabled CSM. |
| 23 | UnregisterCodeSignature | `SUCCESS [0, 0]` | not called | No monitor signature lifecycle. |
| 24 | ValidateCodeSignature | `NOT_SUPPORTED` | not called | Gated by disabled CSM. |
| 25 | ReconstituteCodeSignature | `SUCCESS [0, 0]` | not called | No monitor signature lifecycle. |
| 26 | SetLocalSigningPublicKey | `SUCCESS` | yes | |
| 27 | GetLocalSigningPublicKey | `SUCCESS [0]` | yes | No key is fabricated. |
| 28 | AuthorizeLocalSigningCDHash | `SUCCESS` | yes | |
| 29 | AuthorizeCompilationServiceCDHash | `SUCCESS` | yes | |
| 30 | MatchCompilationServiceCDHash | `NOT_FOUND` | yes | Always reports no match. |
| 31 | DeveloperModeToggle | `SUCCESS` | yes | Does not change the developer-mode byte. |
| 32 | AcquireSigningIdentifier | `SUCCESS [0]` | not called | Gated by disabled CSM. |
| 33 | AssociateKernelEntitlements | `SUCCESS` | not called | Gated by disabled CSM. |
| 34 | AccelerateEntitlements | `NOT_SUPPORTED` | not called | XNU uses its non-monitor entitlements parser. |
| 35 | RegisterAddressSpace | `SUCCESS [1]` | yes | Return a constant nonzero opaque handle. |
| 36 | UnregisterAddressSpace | `SUCCESS` | yes | |
| 37 | SetupNestedAddressSpace | `SUCCESS` | yes | |
| 38 | AssociateCodeSignature | `NOT_SUPPORTED` | not called | Gated by disabled CSM. |
| 39 | AllowJITRegion | `SUCCESS` | not called | XNU returns success locally while CSM is disabled. |
| 40 | AssociateJITRegion | `SUCCESS` | not called | Gated by disabled CSM. |
| 41 | AllowInvalidCode | `NOT_SUPPORTED` | not called | Gated by disabled CSM. |
| 42 | AssociateDebugRegion | `SUCCESS` | not called | Gated by disabled CSM. |
| 43 | GetEntitlementsContext | `SUCCESS [0]` | not called | No monitor entitlements context is created. |
| 44 | ResolveKernelEntitlementsAddressSpace | `NOT_FOUND` | not called | Gated by disabled CSM. |
| 45 | Image4Dispatch | `SUCCESS` | yes | Acknowledge without performing the operation. |
| 46 | Image4GetExports | `NOT_PERMITTED` | yes | Deliberately refused. |
| 47 | Image4SetReleaseType | `NOT_PERMITTED` | yes | Deliberately refused. |
| 48 | Image4SetBootNonceShadow | `NOT_PERMITTED` | yes | Deliberately refused. |
| 49 | Image4SetNonce | `NOT_PERMITTED` | yes | Deliberately refused. |
| 50 | Image4RollNonce | `NOT_PERMITTED` | yes | Deliberately refused. |
| 51 | Image4GetNonce | `NOT_PERMITTED` | yes | Deliberately refused. |
| other | unsupported selector | `NOT_SUPPORTED` | yes | |

## 5. Selector 2: GetCodeSigningInfo

Selector 2 is the only nontrivial endpoint. Its job is to return configuration
information to XNU, which uses this information to perform future calls.

Before selector 2 can be called, allocate one zeroed, XNU-readable 16 KiB page
named `txm_info_page` and keep it mapped for the lifetime of XNU. Write the
page's own VA as a 64-bit value at offset `0x190`, and write the byte value 1
at offset `0x318`. Leave every other byte zero.

Return exactly six words:

```text
SUCCESS [txm_info_page, txm_info_page + 0x318, 0, 0, 0, txm_info_page]
```

### 5.1 Context on What These Words Mean

XNU interprets the return words as:

| word | XNU use | minimal value |
|---:|---|---|
| 0 | `TXMReadWriteData_t` and metrics base | `txm_info_page` |
| 1 | persistent developer-mode boolean pointer | `txm_info_page + 0x318` |
| 2 | obsolete pointer, unused by the current XNU build | 0 |
| 3 | managed code-signature size | 0 |
| 4 | obsolete configuration pointer, unused by the current XNU build | 0 |
| 5 | `TXMReadOnlyData_t` and configuration base | `txm_info_page` |

The relevant type definitions are beneath
`<KDK>/System/Library/Frameworks/Kernel.framework/Versions/A/PrivateHeaders/`:

Only the following XNU-visible locations matter. Every unlisted byte remains
zero:

| offset from `txm_info_page` | size | XNU-visible purpose | value |
|---:|---:|---|---|
| `0x05` | 1 | research build flag | 0 |
| `0x06` | 1 | extended-research build flag | 0 |
| `0x190` | 8 | `CSConfiguration.systemPolicy` | VA of `txm_info_page` |
| `0x288` | `0x88` | metrics derived from return word 0 | all zero |
| `0x318` | 1 | `codeSigningDisabled` | 1 |

Pointing `CSConfiguration.systemPolicy` back to `txm_info_page` gives XNU a
valid policy object whose feature bytes are all zero. In particular, restricted
execution mode remains disabled, so XNU does not follow the unused
restricted-mode state pointer.

The byte at `txm_info_page + 0x318` serves two independent XNU interfaces. As
`codeSigningDisabled`, it disables monitor-backed code signing. Return word 1
also points to it as the developer-mode state, so developer mode remains
enabled. Selector 31 is only a constant-success notification in this shim and
does not update the byte; it must remain equal to 1.

Return words 0 and 5 deliberately share the same page. XNU derives its zeroed
metrics from word 0 and its configuration from word 5; the required ranges do
not conflict.

If the goal is simply a working emulator, all of this information is
irrelevant.
