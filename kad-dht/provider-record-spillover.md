# Provider Record Spillover

| Lifecycle Stage | Maturity      | Status | Latest Revision |
|-----------------|---------------|--------|-----------------|
| 1A              | Working Draft | Active | r1, 2026-08-31  |

Authors: [@gmelodie]

Interest Group: [@mxinden, @guillaumemichel, @MarcoPolo, @richard-ramos]

[@gmelodie]: https://github.com/gmelodie
[@mxinden]: https://github.com/mxinden
[@guillaumemichel]: https://github.com/guillaumemichel
[@MarcoPolo]: https://github.com/MarcoPolo
[@richard-ramos]: https://github.com/richard-ramos

See the [lifecycle document][lifecycle-spec] for context about the maturity level
and spec status.

[lifecycle-spec]: https://github.com/libp2p/specs/blob/master/00-framework-01-spec-lifecycle.md

---

## Overview

This document specifies an extension to the [libp2p Kademlia DHT
specification][kad-spec]. The extension solves the problem of provider record
hotspots. When a key is popular, the `k` closest nodes receive all the
`ADD_PROVIDER` traffic for that key, and they can overload. The extension lets a
node set a per-key provider limit, and lets it tell an advertiser that it
rejects a record. The advertiser then spills over to peers that are farther
along its lookup path, instead of a retry against the same overloaded nodes.

## Definitions

**Provider record capacity**: the maximum number of distinct providers a node is
willing to store for a single key.

**Rejection**: a node signalling to an advertiser that it will not store the
provider record for the given key.

**Placement**: an `ADD_PROVIDER` request that the advertiser counts towards the
replication target `k`.

**Spillover**: the behavior of an advertiser that, upon failing to reach the
replication target `k` from a batch of peers, continues advertising to
progressively farther peers discovered during the iterative lookup.

**Spillover round**: one batch of `ADD_PROVIDER` requests sent to the next group
of candidates during a spillover.

**Legacy node**: a node that supports `kad/1.0.0` only.

## Motivation

In the base Kademlia DHT protocol, provider advertisement targets the `k`
closest peers to the content key unconditionally. For popular keys, these peers
accumulate unbounded provider records and act as a permanent bottleneck. The
base spec provides no mechanism for a peer to decline an `ADD_PROVIDER` or for
the advertiser to store the record elsewhere.

This extension solves both sides of the problem:

1. **Server-side protection**: nodes can enforce a per-key limit and reject
   `ADD_PROVIDER` requests once the limit is reached.
2. **Client-side resilience**: when a node rejects a request, the advertiser
   continues along its lookup path. More peers store the record, until the
   advertiser reaches the replication target.

## Protocol Identifier

A new major version of the Kademlia protocol identifier carries this extension.
A network that runs the base protocol as `/ipfs/kad/1.0.0` runs this extension
as `/ipfs/kad/2.0.0`.

`kad/2.0.0` and `kad/1.0.0` are identical, except for the `ADD_PROVIDER`
exchange below. `PUT_VALUE`, `GET_VALUE`, `GET_PROVIDERS`, `FIND_NODE` and
`PING` keep their base semantics, message formats and requirements.

Rules:

- A node in server mode that implements this extension MUST announce both
  `kad/2.0.0` and `kad/1.0.0` through the [identify protocol][identify-spec]. It
  MUST accept an incoming stream on both versions.
- If the remote peer announces `kad/2.0.0`, the advertiser MUST open the
  `ADD_PROVIDER` stream on that version. If the peer announces `kad/1.0.0` only,
  the advertiser MUST open the stream on `kad/1.0.0`.
- On a `kad/1.0.0` stream, the advertiser MUST NOT wait for an `ADD_PROVIDER`
  response. On that version the request stays fire and forget, as in the base
  spec.
- A node that answers on a `kad/1.0.0` stream MUST NOT write `providerStatus`.

Protocol negotiation thus tells the advertiser which version the peer supports,
before the advertiser sends the request. The advertiser spends no timeout to
find a legacy node. A peer that drops `kad/1.0.0` at a later date only changes
the list that it announces, and the network needs no flag day.

## Provider Record Limits

A node MAY set a maximum number of distinct providers per key,
`maxProvidersPerKey`. The node counts the distinct provider peer IDs that it
stores for the key. If that count reaches `maxProvidersPerKey`, and the key
holds no record yet from the sender of the `ADD_PROVIDER`, the node MUST reject
the request or evict a stored record. [Eviction](#eviction) gives the rules.

Re-advertisements from a provider that is already stored for the key are always
accepted, regardless of the limit, so that existing providers can refresh their
records.

`maxProvidersPerKey` has no default value. A node that sets it MUST keep the
value at the replication factor `k` or above. With a value below `k`, the
closest peers alone cannot resolve even an unpopular key. The RECOMMENDED value
is 1000. A reader stops after a few dozen usable providers. A limit three orders
of magnitude above `k` thus caps the storage cost and the CPU cost of a hotspot,
and a popular key stays fully resolvable. This value is provisional.
Implementers SHOULD measure the live network and correct the value, as they do
for the republish interval and the expiration interval of the base spec.

Nodes SHOULD also enforce coarser bounds such as total provider records stored
(`providerRecordCapacity`) and total distinct keys for which records are held
(`providedKeyCapacity`).

A node MAY reject an `ADD_PROVIDER` for a different reason than capacity, for
example a local policy. The signals below apply to every rejection, for each
reason.

### Eviction

A node always accepts a re-advertisement. Therefore the first
`maxProvidersPerKey` providers of a key can hold their slots forever. They only
have to refresh their records. The stored set then freezes around the providers
that arrived first. A node MAY add an eviction policy to `maxProvidersPerKey`.
The policy lets the set rotate.

Eviction and rejection are exclusive. Eviction takes precedence:

- If the node evicts a record, it MUST store the new record, and it MUST answer
  `accepted`.
- If the node evicts no record, it MUST keep every stored record, and it MUST
  answer `rejected`.

A node MUST NOT evict a record and answer `rejected`. That combination drops a
provider and stores no replacement.

A policy that evicts the record with the oldest `timeReceived` is unsafe on its
own. An attacker rotates its peer ID. On each request the attacker looks like a
new provider. The attacker then flushes every honest provider out of the `k`
closest nodes, at a cost of one request per eviction. The base protocol has no
such censorship vector, because it removes no record early.

If a node evicts, it MUST select the eviction candidates only from the records
whose `timeReceived` is older than the republish interval of the network (22
hours in the IPFS DHT). Such a record is close to its expiration. An attacker
that rotates its peer ID thus gains no more than it gains when it waits. A
provider that republishes on schedule keeps its slot.

## ADD_PROVIDER Response

### Response message

On a `kad/2.0.0` stream, a node that receives an `ADD_PROVIDER` MUST write one
response message. It MUST then close its side of the stream. The response is a
new message, not a copy of the request. It contains these fields:

| Field            | Presence | Value                        |
|------------------|----------|------------------------------|
| `type`           | MUST     | `ADD_PROVIDER`               |
| `key`            | MUST     | the `key` from the request   |
| `providerStatus` | MUST     | see below                    |

The responder MUST leave every other field empty. If a field is present, the
advertiser MUST ignore it. The responder returns no `providerPeers` and no
`closerPeers`.

`providerStatus` is a response-only field. Advertisers MUST NOT set it in
`ADD_PROVIDER` requests. Receiving nodes MUST ignore `providerStatus` if it is
present in an incoming request, to avoid interoperability ambiguity.

### Status values

- `accepted (0)`: the node stored the record.
- `rejected (1)`: the node did not store the record, because of its capacity or
  its local policy. The request is well formed, thus the same request MAY
  succeed at another peer.
- `invalid (2)`: the request is malformed, and no other peer accepts it either.
  A node MUST answer `invalid` when `key` is not a valid multihash, when
  `providerPeers` is empty, or when no entry of `providerPeers` matches the peer
  ID of the sender. The base spec already sets these validity rules for
  `ADD_PROVIDER`.

### Outcome classification

An advertiser classifies each `ADD_PROVIDER` attempt as exactly one outcome.
Only the first two outcomes count towards the replication target `k`.

| Outcome                                              | Counts towards `k` |
|------------------------------------------------------|--------------------|
| `kad/2.0.0`, response with `providerStatus = accepted` | yes              |
| `kad/1.0.0`, request written successfully              | yes              |
| `kad/2.0.0`, response with `providerStatus = rejected` | no               |
| `kad/2.0.0`, response with `providerStatus = invalid`  | no, see below    |
| `kad/2.0.0`, response without `providerStatus`         | no               |
| dial failure, stream reset, write failure              | no               |
| stream closed before a response arrived                | no               |
| response timeout                                       | no               |

A `kad/1.0.0` request counts as a placement as soon as the write succeeds,
because the base protocol gives the advertiser no other information. This is the
only request that counts without an answer.

A response without `providerStatus` on a `kad/2.0.0` stream violates this
document. The advertiser MUST count the peer as a failure for this request. It
MUST NOT assume that the peer accepted the record.

A transport failure MUST NOT count as a placement. Silence tells the advertiser
nothing about storage. If silence counted, a peer that drops streams would
absorb placements and hold no record.

If a peer answers `invalid`, the advertiser SHOULD stop the advertisement for
that key, and it SHOULD report the error to the caller. Every other peer applies
the same validity rules, and answers `invalid` too. A spillover round therefore
gains nothing.

## Spillover Algorithm

### Overview

When an advertiser sends `ADD_PROVIDER` to a batch of peers and the replication
target `k` has not been reached after that batch, it performs a **spillover
round**: it moves to the next group of peers that are farther from the key, as
discovered during the initial iterative lookup. This continues until either:

- the advertiser reaches the replication target `k`, or
- the advertiser has exhausted all peers discovered during the lookup.

### Lookup phase

Before advertising, the advertiser performs a full iterative lookup (using
`FIND_NODE`) for the content key as usual, collecting all peers encountered. The
resulting candidate set is sorted in ascending order of XOR distance to the key.

### Advertisement phase

The advertiser splits the sorted candidate list into chunks of size `α` (the
concurrency parameter, default 10). It then iterates over these chunks from
closest to farthest:

1. Send `ADD_PROVIDER` to all the peers of the current chunk at the same time.
   Use `kad/2.0.0` or `kad/1.0.0`, as each peer announces.
2. Classify each attempt with [Outcome
   classification](#outcome-classification). Add the placements to the count.
3. If the count reaches `k`, stop.
4. If the count stays below `k`, continue with the next chunk. This is a
   spillover round.
5. If no chunk remains, stop. The advertiser made fewer than `k` placements, and
   it SHOULD report the number of placements.

**Note:** The timeout per peer in a spillover round SHOULD be slightly larger
than the base timeout to account for dial overhead to less-familiar peers.

### Relationship to base advertisement

The first `⌈k/α⌉` chunks hold the `k` closest peers, which is the set that the
base spec targets. With the default `k` = 20 and `α` = 10, these are the first
two chunks. If those peers accept every record, the advertiser stops there and
behaves as in the base spec. A spillover round starts only after that.

## Read Path

A spilled record sits on a peer outside the `k` closest peers to the key. A
reader that queries only the `k` closest peers thus never finds it.
`GET_PROVIDERS` stays unchanged. On a network with spillover, a reader adapts as
follows.

A reader that does a normal iterative lookup already moves outwards from the
closest peers. Only its stop rule changes. If the reader holds fewer providers
than it wants, it MUST continue past the `k` closest peers. It queries the next
chunk of `α` candidates in ascending distance order, as the advertisement phase
does. It stops when it has sufficient providers, or when no candidate remains.

Some readers skip the iterative lookup and take the `k` closest peers directly
from a full routing table. The accelerated DHT client works this way. Such a
reader MUST extend its query set in the same manner. The `k` closest peers
return the providers that arrived first, and they hide every provider that
spilled over.

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
        ACCEPTED = 0;
        REJECTED = 1;
        INVALID = 2;
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

    // Added by this extension. Set on kad/2.0.0 ADD_PROVIDER responses only.
    optional AddProviderStatus providerStatus = 11;
}
```

Both protocol versions use the same wire format. Field 11 is optional, thus a
legacy node that receives it ignores it.

## Backward Compatibility

This extension does not change the base `ADD_PROVIDER` exchange. It does not
change `GET_PROVIDERS`, `FIND_NODE` or any other message type, on either
version.

Current implementations, for example [go-libp2p-kad-dht][go-add-provider], do
not write and do not read an `ADD_PROVIDER` response. The table gives the
behaviour of each combination.

| Advertiser | Responder | Behaviour                                                                                              |
|------------|-----------|--------------------------------------------------------------------------------------------------------|
| legacy     | legacy    | Fire and forget on `kad/1.0.0`. The node stores the record.                              |
| legacy     | new       | The advertiser opens `kad/1.0.0`, the only version that it knows. While the limits stay disabled, the responder MUST accept the record on that version. See [Deployment](#deployment). After an operator enables the limits, the responder can drop the record, and the advertiser cannot learn this. |
| new        | legacy    | The responder does not announce `kad/2.0.0`. The advertiser thus opens `kad/1.0.0`, and waits for no response. It spends no timeout.                                             |
| new        | new       | The peers negotiate `kad/2.0.0`. Rejection and spillover work as this document specifies.                                   |

One combination loses records: a legacy advertiser against a responder that
enforces a limit. The deployment order below keeps that combination safe until
most advertisers support `kad/2.0.0`.

## Deployment

If a node enforces a limit before the advertisers can read a rejection, a legacy
advertiser loses placements. It receives no signal, and it has no fallback.
Implementations SHOULD deploy the extension in this order.

1. **Advertisers first.** Add `kad/2.0.0` support: the response reader, the
   outcome classification and the spillover algorithm. Keep `maxProvidersPerKey`
   unset. Only the negotiated version changes on the wire, and every peer still
   accepts every record.
2. **Readers next.** Add the read path change. A reader must find a spilled
   record before the first record spills over.
3. **Servers last.** Set `maxProvidersPerKey` only when most advertisers that a
   node sees use `kad/2.0.0`. Adoption on this scale takes months. An
   implementation SHOULD measure the share of incoming `ADD_PROVIDER` requests
   on `kad/2.0.0`, and decide from that share.
4. **Legacy requests.** While many advertisers still use `kad/1.0.0`, a node
   that enforces `maxProvidersPerKey` SHOULD apply the limit to `kad/2.0.0`
   requests only. It SHOULD accept every `kad/1.0.0` request, because it cannot
   tell those advertisers that it dropped the record.

## Security Considerations

**An acknowledgment is not proof of storage.** With `accepted`, the responder
claims that it stored the record. The protocol cannot prove the storage of any
record. It also cannot detect a peer that answers `accepted` and stores nothing.
Such a responder absorbs placements and stops spillover, as a peer that drops
records under load does. If `k` of the closest candidates collude, they suppress
an advertisement completely. The base protocol has the same weakness, because
the same peers can discard the record. An advertiser that needs a stronger
assurance checks the placement out of band, for example with a `GET_PROVIDERS`
to each peer that it advertised to.

**False rejections.** An adversary that controls the closest peers to a key can
reject every `ADD_PROVIDER`, and thus suppress an advertisement. Spillover
limits a unanimous rejection. Each shortfall below `k` moves the advertiser
outwards, to nodes outside the set that the adversary controls. An adversary
that mixes accepts and rejects across the chunks reduces the replication.
Rejections alone cannot stop the storage of the record, because spillover
continues while the count stays below `k`.

**Eviction and peer ID rotation.** A policy that always drops the oldest record
lets an attacker flush the honest providers. The attacker only rotates its peer
ID. The base protocol has no such censorship vector. [Eviction](#eviction)
removes the advantage: the candidates are the records that are older than the
republish interval, and those slots are close to their expiration.

**Slot monopolisation.** With no eviction policy, the first `maxProvidersPerKey`
providers of a key hold their slots while they republish. Spillover moves each
later provider outwards. This is the safe default. A later provider loses
proximity to the key, and it keeps reachability, because the read path finds a
spilled record.

---

## References

[kad-spec]: https://github.com/libp2p/specs/blob/master/kad-dht/README.md
[identify-spec]: https://github.com/libp2p/specs/blob/master/identify/README.md
[go-add-provider]: https://github.com/libp2p/go-libp2p-kad-dht/blob/10e0adf9859ef86ba08d8493d8313869a2e83d8a/handlers.go#L278
