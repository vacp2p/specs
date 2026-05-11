# Provider Record Spillover

| Lifecycle Stage | Maturity      | Status | Latest Revision |
|-----------------|---------------|--------|-----------------|
| 1A              | Working Draft | Active | r0, 2026-05-11  |

Authors: [@gmelodie]

Interest Group: []

[@gmelodie]: https://github.com/gmelodie

See the [lifecycle document][lifecycle-spec] for context about the maturity level
and spec status.

[lifecycle-spec]: https://github.com/libp2p/specs/blob/master/00-framework-01-spec-lifecycle.md

---

## Overview

This document specifies an optional extension to the [libp2p Kademlia DHT
specification][kad-spec] that addresses provider record hotspots. When a key is
popular, the `k` closest nodes concentrate all `ADD_PROVIDER` traffic for it
and can become overloaded. This extension lets nodes enforce per-key provider
limits and signal rejection to advertisers, and lets advertisers spill over to
progressively farther peers from their lookup path rather than repeatedly
hammering the same overloaded nodes.

The extension is opt-in and backward compatible. Nodes that have not enabled it
behave exactly as before.

## Definitions

**Provider record capacity**: the maximum number of distinct providers a node
is willing to store for a single key.

**Rejection**: a node signalling to an advertiser that it will not store the
provider record for the given key.

**Spillover**: the behavior of an advertiser that, upon receiving rejections
from the closest peers, continues advertising to progressively farther peers
discovered during the iterative lookup.

**Spillover round**: one batch of `ADD_PROVIDER` requests sent to the next
group of candidates during a spillover.

## Motivation

In the base Kademlia DHT protocol, provider advertisement targets the `k`
closest peers to the content key unconditionally. For popular keys, these peers
accumulate unbounded provider records and act as a permanent bottleneck. The
base spec provides no mechanism for a peer to decline an `ADD_PROVIDER` or for
the advertiser to store the record elsewhere.

This extension solves both sides of the problem:

1. **Server-side protection**: nodes can enforce a per-key limit and reject
   `ADD_PROVIDER` requests once the limit is reached.
2. **Client-side resilience**: advertisers react to rejections by backtracking
   through the lookup path, widening the set of peers that store the record
   until the replication target is met.

## Provider Record Limits

A node MAY enforce a maximum number of distinct providers per key,
`maxProvidersPerKey`. When set, the node counts the distinct provider peer IDs
already stored for the key. If that count is at or above `maxProvidersPerKey`
and the `ADD_PROVIDER` sender is not already a known provider for that key
(i.e., it is not a re-advertisement), the node MUST reject the request.

Re-advertisements from a provider that is already stored for the key are always
accepted, regardless of the limit, so that existing providers can refresh their
records.

Nodes SHOULD also enforce coarser bounds such as total provider records stored
(`providerRecordCapacity`) and total distinct keys for which records are held
(`providedKeyCapacity`), but those are outside the scope of the rejection
signalling defined here.

## ADD_PROVIDER Rejection

### Response signalling

Support for this extension is optional. A node that supports it MUST include a
`providerStatus` field (field 11, see [Protobuf](#protobuf)) in its
`ADD_PROVIDER` response:

- `accepted (0)` — the record was stored.
- `rejected (1)` — the record was not stored due to capacity limits.

An absent `providerStatus` field — whether because the responding node does not
support this extension or due to a timeout — MUST be interpreted as `accepted`
by the advertiser.

## Spillover Algorithm

### Overview

When an advertiser sends `ADD_PROVIDER` to a batch of peers and all peers in
that batch reject, it performs a **spillover round**: it moves to the next
group of peers that are farther from the key, as discovered during the initial
iterative lookup. This continues until either:

- the replication target `k` accepting peers has been reached, or
- the advertiser has exhausted all peers discovered during the lookup.

### Lookup phase

Before advertising, the advertiser performs a full iterative lookup (using
`FIND_NODE`) for the content key as usual, collecting all peers encountered.
The resulting candidate set is sorted in ascending order of XOR distance to
the key.

### Advertisement phase

The advertiser splits the sorted candidate list into chunks of size `α` (the
concurrency parameter). It then iterates over these chunks from closest to
farthest:

1. Send `ADD_PROVIDER` to all peers in the current chunk concurrently.
2. Collect responses. Count accepting peers (absent `providerStatus` or
   `providerStatus = accepted`) towards the replication target.
3. If the replication target `k` accepting peers has been reached, stop.
4. If **all** peers in the chunk rejected (a unanimous rejection), continue to
   the next chunk (spillover round). Otherwise stop, whether the target was
   met or not, to avoid unnecessary advertisement.
5. If no more chunks remain, stop.

**Note:** The timeout per peer in a spillover round SHOULD be slightly larger
than the base timeout to account for dial overhead to less-familiar peers.

### Relationship to base advertisement

This algorithm is a generalisation of the base `ADD_PROVIDER` procedure. When
no peer rejects, only the closest chunk (or the closest `k` peers) is used and
behaviour is identical to the base spec. Spillover only occurs when unanimous
rejections are encountered.

## Protobuf

The following changes extend the `Message` type defined in the [kad-dht
spec][kad-spec]:

```protobuf
syntax = "proto2";

message Record {
    bytes key = 1;
    bytes value = 2;
    string timeReceived = 5;
}

message Message {
    enum MessageType {
        PUT_VALUE = 0;
        GET_VALUE = 1;
        ADD_PROVIDER = 2;
        GET_PROVIDERS = 3;
        FIND_NODE = 4;
        PING = 5;
    }

    enum ConnectionType {
        NOT_CONNECTED = 0;
        CONNECTED = 1;
        CAN_CONNECT = 2;
        CANNOT_CONNECT = 3;
    }

    // Added by this extension.
    enum AddProviderStatus {
        accepted = 0;
        rejected = 1;
    }

    message Peer {
        bytes id = 1;
        repeated bytes addrs = 2;
        ConnectionType connection = 3;
    }

    MessageType type = 1;
    bytes key = 2;
    Record record = 3;
    repeated Peer closerPeers = 8;
    repeated Peer providerPeers = 9;
    int32 clusterLevelRaw = 10; // NOT USED

    // Added by this extension. Absent field MUST be treated as accepted.
    optional AddProviderStatus providerStatus = 11;
}
```

Field 11 is optional; older implementations that do not know about it ignore
it, maintaining full wire-level backward compatibility.

## Backward Compatibility

- Nodes that have not enabled `providerRejection` never write `providerStatus`
  and are unaffected by receiving it.
- Advertisers that do not implement spillover continue to advertise to the `k`
  closest peers; absent `providerStatus` is treated as acceptance.
- No change is made to `GET_PROVIDERS` or any lookup message.

## Security Considerations

**False rejections**: an adversary controlling the closest peers to a key could
reject all `ADD_PROVIDER` requests to suppress its advertisement. Spillover
defeats this: unanimous rejection causes the advertiser to route around those
peers and store the record on nodes beyond the adversary's controlled set.
However, an adversary that mixes accepts and rejects strategically can stay
below the unanimous-rejection threshold and reduce replication without
triggering spillover. This is a known limitation of the unanimous-rejection
trigger.

---

## References

[0]: libp2p Kademlia DHT specification – https://github.com/libp2p/specs/blob/master/kad-dht/README.md

[kad-spec]: https://github.com/libp2p/specs/blob/master/kad-dht/README.md
