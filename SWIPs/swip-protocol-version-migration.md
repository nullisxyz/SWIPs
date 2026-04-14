---
swip: <to be assigned>
title: Protocol Version Migration
author: mfw78 (@mfw78)
status: Draft
type: Standards Track
category: Networking
created: 2026-01-21
requires: swip-bzzaddress-signature-v1
---

## Simple Summary

Introduce a protocol version digest mechanism that enables nodes to identify compatible peers during network upgrades via the handshake and Hive peer gossip. Additionally, clean up the underlay encoding by using proper protobuf repeated fields instead of custom serialization.

## Abstract

A protocol version digest is a 4-byte identifier derived from the network configuration and current protocol version. By including this digest in both the handshake protocol and Hive peer advertisements, nodes can efficiently filter incompatible peers before connection and avoid gossiping peers that recipients cannot use.

This proposal also replaces the custom underlay serialization format (magic `0x99` prefix with varint-length encoding) with idiomatic protobuf `repeated` fields, simplifying implementations and improving interoperability.

## Motivation

Swarm lacks a standardised mechanism for:

1. **Peer compatibility detection.** Nodes cannot determine protocol compatibility before establishing connections.
2. **Efficient peer gossip.** Hive broadcasts all known peers regardless of protocol compatibility, wasting bandwidth.
3. **Graceful upgrades.** During network upgrades, incompatible nodes waste resources attempting failed connections.
4. **Clean underlay encoding.** The current protocol uses a custom serialization format for multiple underlay addresses: a magic `0x99` prefix byte followed by varint-length-prefixed multiaddr bytes. This deviates from idiomatic protobuf usage, complicates implementations, and conflates wire encoding with application logic.

## Specification

### Protocol Version Digest Calculation

```
protocol_version_digest = keccak256(genesis_hash || current_protocol_version)[0:4]
```

Where:

- `genesis_hash`: 32-byte hash uniquely identifying the network
- `current_protocol_version`: 4-byte version for the active protocol

#### Genesis Hash

The genesis hash MUST uniquely identify a Swarm network. An illustrative example:

```
genesis_hash = keccak256(network_id || genesis_timestamp || ...)
```

The exact composition of the genesis hash requires further discussion. Candidates for inclusion:

- `network_id` (required)
- `genesis_timestamp`
- Contract addresses (postage stamp, staking, redistribution)
- Other network-specific parameters

Feedback is solicited on what should constitute the genesis hash for mainnet and testnet.

### Protocol Versions

| Version | Identifier | Activation |
|---------|------------|------------|
| Homestead | `0x00000000` | Genesis |

### Version Activation Condition

Protocol versions use timestamp-based activation:

```
version_active = current_timestamp >= activation_timestamp
```

### Handshake Integration

The handshake protocol is updated with the protocol version digest and proper underlay encoding:

```protobuf
syntax = "proto3";

package handshake;

message Syn {
    bytes observed_underlay = 1;
}

message Ack {
    BzzAddress address = 1;
    uint64 network_id = 2;
    bool full_node = 3;
    bytes nonce = 4;
    string welcome_message = 99;
}

message SynAck {
    Syn syn = 1;
    Ack ack = 2;
}

message BzzAddress {
    repeated bytes underlays = 1;  // Multiple multiaddr bytes (was: single bytes with custom encoding)
    bytes signature = 2;
    bytes overlay = 3;
    bytes protocol_version_digest = 4;  // 4 bytes
}
```

Key changes:

1. **`underlays` becomes `repeated bytes`** — Each multiaddr is a separate element. No custom serialization (no `0x99` prefix, no varint length encoding). Protobuf handles the wire format.
2. **`protocol_version_digest` is added** — 4-byte protocol version identifier.

Nodes MUST reject connections where `peer.protocol_version_digest != local.protocol_version_digest`.

### Hive Protocol Integration

The Hive protocol uses the same `BzzAddress` message defined above. Gossip filtering rules:

- When receiving peers, ignore those with an incompatible protocol version digest.
- When sending peers, only advertise those matching the recipient's protocol version digest.

This prevents nodes from filling their address books with unreachable peers and reduces unnecessary connection attempts across the network.

### BzzAddress Signature

When the `protocol_version_digest` field is present, the v1 signature scheme (as defined in the BzzAddress Signature Scheme v1 SWIP) MUST be used with the protocol version digest appended:

```
underlays_concat = underlays[0].bytes || underlays[1].bytes || ... || underlays[n].bytes
data = underlays_concat || overlay || network_id || protocol_version_digest
signature = eip191_sign(data)
```

Including the protocol version digest in the signed data binds the signature to a specific protocol version, preventing replay of old addresses on new protocol versions.

### Grace Period

During protocol version transitions (a one-hour window around activation), nodes MAY accept both pre-transition and post-transition digests to accommodate clock skew.

## Rationale

**4-byte digest.** A 4-byte value is compact yet sufficient (2^32 possible values) for network and protocol version disambiguation. This matches Ethereum's approach with fork digests.

**Timestamp activation.** Swarm has no block consensus, making timestamps the natural coordination mechanism.

**Hive integration.** Without version-aware gossip, nodes accumulate stale peer lists during upgrades, degrading connectivity.

**Digest in signature.** Including the protocol version digest in the signature binds the address to a specific protocol version, preventing replay of old addresses after upgrades.

**Repeated bytes for underlays.** The legacy custom encoding (magic `0x99` prefix + varint length prefixes) was a workaround for backward compatibility with single-underlay nodes. This conflates wire encoding with application logic and complicates implementations. Using protobuf's native `repeated bytes` field:

- Leverages protobuf's built-in length-prefixed encoding for wire format
- Simplifies parsing — no custom deserialization logic needed
- Improves interoperability — standard protobuf tooling works correctly
- Separates concerns — wire encoding is handled by protobuf, signature construction is application logic

## Backwards Compatibility

This proposal introduces breaking changes to the handshake and Hive protocols:

1. **Protocol version digest field** — New required field in BzzAddress
2. **Underlay encoding** — Changes from `bytes underlay` (custom encoding) to `repeated bytes underlays` (native protobuf)

Migration follows a two-release plan:

1. **Release N.** Protocol version digest is optional. BzzAddress accepts both:
   - Legacy format: `bytes underlay` with custom encoding, no digest
   - New format: `repeated bytes underlays` with protocol version digest
   Nodes generate the new format but accept both.

2. **Release N+1.** Only the new format is accepted. Legacy format is rejected.

Once Release N is deployed, the new format will propagate through Hive gossip as nodes exchange peer information. By the time Release N+1 is deployed, the network should be predominantly using the new format.

Nodes that have not upgraded by Release N+1 will be unable to connect.

## Test Cases

### Protocol Version Digest

| Scenario | Expected |
|----------|----------|
| Same network, same protocol version | Connection accepted |
| Different networks | Connection rejected |
| Pre/post transition during grace period | Connection accepted |
| Pre/post transition outside grace period | Connection rejected |
| Hive gossip with matching digest | Peer accepted |
| Hive gossip with mismatched digest | Peer ignored |

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
