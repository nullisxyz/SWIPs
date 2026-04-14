---
swip: <to be assigned>
title: BzzAddress Signature and Underlay Encoding v1
author: mfw78 (@mfw78)
status: Draft
type: Standards Track
category: Networking
created: 2026-01-21
---

## Simple Summary

Replace the legacy BzzAddress signature scheme with a simplified v1 format that removes redundant prefixes and uses straightforward concatenation of underlay bytes. Additionally, replace the custom underlay serialization format with idiomatic protobuf `repeated` fields.

## Abstract

The current BzzAddress signature (v0) includes a redundant `"bee-handshake-"` prefix and relies on custom underlay serialization. This proposal defines a v1 signature scheme that removes the prefix (EIP-191 already provides domain separation), uses simple concatenation of multiaddr bytes, and establishes a clean foundation for future protocol extensions such as protocol version fields.

This proposal also replaces the custom underlay serialization format (magic `0x99` prefix with varint-length encoding) with idiomatic protobuf `repeated` fields, simplifying implementations and improving interoperability.

## Motivation

The legacy v0 signature scheme has several issues:

1. **Redundant prefix.** The `"bee-handshake-"` prefix duplicates the domain separation already provided by EIP-191's `"\x19Ethereum Signed Message:\n<length>"` envelope.
2. **Custom serialization in signatures.** The v0 scheme signs over a custom-encoded underlay format (`0x99` prefix + varint length encoding), coupling the signature to a specific wire encoding rather than the logical data.
3. **No extensibility.** The v0 scheme has no mechanism to include additional authenticated fields (such as a protocol version identifier) without breaking existing signatures.
4. **Custom underlay encoding.** The current protocol uses a magic `0x99` prefix byte followed by varint-length-prefixed multiaddr bytes. This deviates from idiomatic protobuf usage, complicates implementations, and conflates wire encoding with application logic.

## Specification

### Underlay Encoding

The `BzzAddress` protobuf message is updated to use idiomatic repeated fields:

```protobuf
message BzzAddress {
    repeated bytes underlays = 1;  // Multiple multiaddr bytes (was: single bytes with custom encoding)
    bytes signature = 2;
    bytes overlay = 3;
}
```

**`underlays` becomes `repeated bytes`** — Each multiaddr is a separate element in the repeated field. No custom serialization (no `0x99` prefix, no varint length encoding). Protobuf handles the wire format natively.

### Legacy Signature (v0)

```
serialized_underlay = custom_serialize(underlays)  // 0x99 prefix + varint lengths, or raw for single
data = "bee-handshake-" || serialized_underlay || overlay || network_id
signature = eip191_sign(data)
```

### New Signature (v1)

```
underlays_concat = underlays[0].bytes || underlays[1].bytes || ... || underlays[n].bytes
data = underlays_concat || overlay || network_id
signature = eip191_sign(data)
```

Key changes:

1. **Simple concatenation for underlays.** The signature is computed over the concatenation of all underlay multiaddr bytes in order. No length prefixes, no magic bytes, no padding. The multiaddr self-describing format and the fixed-length `overlay` (32 bytes) provide implicit framing.

2. **No "bee-handshake-" prefix.** EIP-191 personal sign already provides domain separation via `"\x19Ethereum Signed Message:\n<length>"`. The legacy prefix was redundant.

### Extensibility

The v1 signature scheme is designed to accommodate future additional fields by appending them after `network_id`. For example, a future protocol version field would be signed as:

```
data = underlays_concat || overlay || network_id || protocol_version
```

This is possible because `network_id` is fixed-length, providing implicit framing for any subsequent fields.

### Migration

During migration, nodes MUST support verifying both signature formats:

1. If the BzzAddress contains fields only present in the new protocol (e.g. a protocol version identifier), verify using the v1 scheme.
2. Otherwise, verify using the legacy v0 scheme (with custom underlay deserialization).

Nodes SHOULD generate v1 signatures when creating new BzzAddress entries once support is enabled.

## Rationale

**Removing the "bee-handshake-" prefix.** EIP-191 personal sign already prefixes messages with `"\x19Ethereum Signed Message:\n<length>"`, providing sufficient domain separation. Removing the legacy prefix simplifies the protocol without reducing security.

**Concatenation for signature.** The signature is over the simple concatenation of multiaddr bytes. No additional framing is required because:

- Multiaddrs are self-describing (each component includes its protocol code and length)
- The overlay address is fixed-length (32 bytes)
- EIP-191 includes the total message length, providing overall framing

This provides sufficient domain separation without introducing complexity.

**Repeated bytes for underlays.** The legacy custom encoding (magic `0x99` prefix + varint length prefixes) was a workaround for backward compatibility with single-underlay nodes. This conflates wire encoding with application logic and complicates implementations. Using protobuf's native `repeated bytes` field:

- Leverages protobuf's built-in length-prefixed encoding for wire format
- Simplifies parsing — no custom deserialization logic needed
- Improves interoperability — standard protobuf tooling works correctly
- Separates concerns — wire encoding is handled by protobuf, signature construction is application logic

## Backwards Compatibility

This is a breaking change to the signature scheme and underlay encoding. A two-release migration is required:

1. **Release N.** Both v0 and v1 signatures are accepted. Both legacy `bytes underlay` (with custom encoding) and new `repeated bytes underlays` are accepted. Nodes generate the new format but verify both.
2. **Release N+1.** Only v1 signatures and `repeated bytes underlays` are accepted. Legacy formats are rejected.

Nodes that have not upgraded by Release N+1 will be unable to connect.

## Test Cases

### Signature Verification

| Scenario | Expected |
|----------|----------|
| v0 signature (legacy underlay encoding) | Accepted (Release N only) |
| v1 signature (concatenated underlays) | Accepted |
| v0 signature after Release N+1 | Rejected |
| v1 signature with underlays in different order | Rejected (signature mismatch) |

### Underlay Encoding

| Scenario | Expected |
|----------|----------|
| Single underlay in repeated field | Valid |
| Multiple underlays in repeated field | Valid |
| Empty underlays (zero elements) | Rejected |
| Legacy 0x99-prefixed encoding in bytes field | Accepted (Release N only) |
| Raw multiaddr in bytes field (single underlay) | Accepted (Release N only) |

## Implementation

Reference: [vertex](https://github.com/nxm-rs/vertex)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
