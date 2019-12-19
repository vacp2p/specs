# Waku

> Version 0.2.0
>
> Authors: Adam Babik <adam@status.im>, Dean Eigenmann <dean@status.im>, Kim De Mey <kimdemey@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

## Table of Contents

- [Abstract](#abstract)
- [Motivation](#motivation)
- [Definitions](#definitions)
- [Underlying Transports and Prerequisites](#underlying-transports-and-prerequisites)
    - [Use of DevP2P](#use-of-devp2p)
    - [Gossip based routing](#gossip-based-routing)
- [Wire Specification](#wire-specification)
    - [Use of RLPx transport protocol](#use-of-rlpx-transport-protocol)
    - [ABNF specification](#abnf-specification)
    - [Packet Codes](#packet-codes)
    - [Packet usage](#packet-usage)
    - [Payload Encryption](#payload-encryption)
    - [Packet code Rationale](#packet-code-rationale)
- [Additional capabilities](#additional-capabilities)
    - [Light node](#light-node)
    - [Accounting for resources (experimental)](#accounting-for-resources-experimental)
- [Backwards Compatibility](#backwards-compatibility)
    - [Waku-Whisper bridging](#waku-whisper-bridging)
- [Forwards Compatibility](#forwards-compatibility)
- [Appendix A: Security considerations](#appendix-a-security-considerations)
    - [Scalability and UX](#scalability-and-ux)
    - [Privacy](#privacy)
    - [Spam resistance](#spam-resistance)
    - [Censorship resistance](#censorship-resistance)
- [Appendix B: Implementation Notes](#appendix-b-implementation-notes)
    - [Implementation Matrix](#implementation-matrix)
    - [Recommendations for clients](#recommendations-for-clients)
    - [Node discovery](#node-discovery)
- [Footnotes](#footnotes)
- [Changelog](#changelog)
    - [Differences between waku/0 and waku/1 (WIP)](#differences-between-waku0-and-waku1-wip)
    - [Differences between shh/6 waku/0](#differences-between-shh6-waku0)
- [Acknowledgements](#acknowledgements)
- [Copyright](#copyright)

## Abstract

This specification describes the format of Waku messages within the ÐΞVp2p Wire Protocol. This spec substitutes [EIP-627](https://eips.ethereum.org/EIPS/eip-627). Waku is a fork of the original Whisper protocol that enables better usability for resource restricted devices, such as mostly-offline bandwidth-constrained smartphones. It does this through (a) light node support, (b) historic messages (with a mailserver) (c) `topic-interest` for better bandwidth usage and (d) basic rate limiting.

## Motivation

Waku was created to incrementally improve in areas that Whisper is lacking in, with special attention to resource restricted devices. We specify the standard for Waku messages in order to ensure forward compatibility of different Waku clients, backwards compatibility with Whisper clients, as well as to allow multiple implementations of Waku and its capabilities. We also modify the language to be more unambiguous, concise and consistent.

## Definitions

| Term            | Definition                                            |
| --------------- | ----------------------------------------------------- |
| **Light node**   | A Waku node that does not forward any messages.       |
| **Envelope**    | Messages sent and received by Waku nodes.              |
| **Node**        | Some process that is able to communicate for Waku.    |

## Underlying Transports and Prerequisites

### Use of DevP2P

For nodes to communicate, they MUST implement devp2p and run RLPx. They MUST have some way of connecting to other nodes. Node discovery is largely out of scope for this spec, but see the appendix for some suggestions on how to do this.

### Gossip based routing

In Whisper, messages are gossiped between peers. Whisper is a form of rumor-mongering protocol that works by flooding to its connected peers based on some factors. Messages are elgible for retransmission until their TTL expires. A node SHOULD relay messages to all connected nodes if an envelope matches their PoW and bloom filter settings. If a node works in light mode, it MAY choose not to forward envelopes. A node MUST NOT send expired envelopes, unless the envelopes are sent as a [mailserver](./wms.md) response. A node SHOULD NOT send a message to a peer that it has already sent before.

## Wire Specification

### Use of RLPx transport protocol

All Waku messages are sent as devp2p RLPx transport protocol, version 5<sup>[1](https://github.com/ethereum/devp2p/blob/master/rlpx.md)</sup> packets. These packets MUST be RLP-encoded arrays of data containing two objects: packet code followed by another object (whose type depends on the packet code).  See [informal RLP spec](https://github.com/ethereum/wiki/wiki/RLP) and the [Ethereum Yellow Paper, appendix B](https://ethereum.github.io/yellowpaper/paper.pdf) for more details on RLP.

Waku is a RLPx subprotocol called `waku` with version `0`. The version number corresponds to the major version in the header spec.

### ABNF specification

Using [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234) we have the following format:

```abnf
; Packet codes 0 - 127 are reserved for Waku protocol
packet-code = 1*3DIGIT

; rate limits
limit-ip     = 1*DIGIT
limit-peerid = 1*DIGIT
limit-topic  = 1*DIGIT

rate-limits = "[" limit-ip limit-peerid limit-topic "]"

light-node = BIT

status = "[" 
	 version pow-requirement 
         [ bloom-filter ] [ light-node ] 
	 [ confirmations-enabled ] [ rate-limits ]
	 "]"

; version is "an integer (as specified in RLP)"
version = DIGIT

confirmations-enabled = BIT

; pow is "a single floating point value of PoW. 
; This value is the IEEE 754 binary representation 
; of a 64-bit floating point number. 
; Values of qNAN, sNAN, INF and -INF are not allowed.
; Negative values are also not allowed."
pow             = 1*DIGIT "." 1*DIGIT
pow-requirement = pow

; bloom filter is "a byte array"
bloom-filter = *OCTET

waku-envelope = "[" expiry ttl topic data nonce "]"

; List of topics interested in
topic-interest = "[" *1000topic "]"

; 4 bytes (UNIX time in seconds)
expiry = 4OCTET

; 4 bytes (time-to-live in seconds)
ttl = 4OCTET

; 4 bytes of arbitrary data
topic = 4OCTET

; byte array of arbitrary size 
; (contains encrypted message)
data = OCTET

; 8 bytes of arbitrary data 
; (used for PoW calculation)
nonce = 8OCTET

messages = 1*waku-envelope

; mail server / client specific
p2p-request = waku-envelope
p2p-message = 1*waku-envelope

; packet-format needs to be paired with its 
; corresponding packet-format
packet-format = "[" packet-code packet-format "]"

required-packet = 0 status / 
                  1 messages /
		  2 pow-requirement /
		  3 bloom-filter
		  
optional-packet = 126 p2p-request / 127 p2p-message / 20 rate-limits / 21 topic-interest

packet = "[" required-packet [ optional-packet ] "]"
```

All primitive types are RLP encoded. Note that, per RLP specification, integers are encoded starting from `0x00`.

### Packet Codes

The message codes reserved for Waku protocol: 0 - 127.

Messages with unknown codes MUST be ignored without generating any error, for forward compatibility of future versions.

The Waku sub-protocol MUST support the following packet codes:

| Name                       |     Int Value |
| -------------------------- | ------------- |
| Status                     |     0         |
| Messages                   |     1         |
| PoW Requirement            |     2         |
| Bloom Filter               |     3         |

The following message codes are optional, but they are reserved for specific purpose.

| Name                       | Int Value | Comment |
|----------------------------|-----------|---------|
| Rate limits                |     20    |         |
| Topic interest             |     21    | Experimental in v0 |
| P2P Request                |    126    | |
| P2P Message                |    127    | |

### Packet usage

**Status**

The bloom filter paramenter is optional; if it is missing or nil, the node is considered to be full node (i.e. accepts all messages).

The Status message serves as a Waku handshake and peers MUST exchange this
message upon connection. It MUST be sent after the RLPx handshake and prior to
any other Waku messages.

A Waku node MUST await the Status message from a peer before engaging in other Waku protocol activity with that peer.
When a node does not receive the Status message from a peer, before a configurable timeout, it SHOULD disconnect from that peer.

Upon retrieval of the Status message, the node SHOULD validate the message
content and decide whether it is compatible with the Waku version and mode
its peer is advertising. The handshake is completed when the node has sent,
received and validated the Status message. Note that its peer might not be in
the same state.

When a node is receiving other Waku messages from a peer before a Status
message is received, the node MUST ignore these messages and SHOULD disconnect
from that peer.
Status messages received after the handshake is completed MUST also be ignored.

The fields `bloom-filter`, `light-node`, `confirmations-enabled` and `rate-limits` are OPTIONAL. However if an optional field is specified, all subsequent fields MUST be specified in order to be unambiguous.

**Messages**

This packet is used for sending the standard Waku envelopes.

**PoW Requirement**

This packet is used by Waku nodes for dynamic adjustment of their individual PoW requirements. Recipient of this message should no longer deliver the sender messages with PoW lower than specified in this message.

PoW is defined as average number of iterations, required to find the current BestBit (the number of leading zero bits in the hash), divided by message size and TTL:

	PoW = (2**BestBit) / (size * TTL)

PoW calculation:

	fn short_rlp(envelope) = rlp of envelope, excluding env_nonce field.
	fn pow_hash(envelope, env_nonce) = sha3(short_rlp(envelope) ++ env_nonce)
	fn pow(pow_hash, size, ttl) = 2**leading_zeros(pow_hash) / (size * ttl)

where size is the size of the RLP-encoded envelope, excluding env_nonce field (size of `short_rlp(envelope)`).

**Bloom Filter**

This packet is used by Waku nodes for sharing their interest in messages with specific topics.

The Bloom filter is used to identify a number of topics to a peer without compromising (too much) privacy over precisely what topics are of interest. Precise control over the information content (and thus efficiency of the filter) may be maintained through the addition of bits.

Blooms are formed by the bitwise OR operation on a number of bloomed topics. The bloom function takes the topic and projects them onto a 512-bit slice. At most, three bits are marked for each bloomed topic.

The projection function is defined as a mapping from a 4-byte slice S to a 512-bit slice D; for ease of explanation, S will dereference to bytes, whereas D will dereference to bits.

	LET D[*] = 0
	FOREACH i IN { 0, 1, 2 } DO
	LET n = S[i]
	IF S[3] & (2 ** i) THEN n += 256
	D[n] = 1
	END FOR

**P2P Request**

This packet is used for sending Dapp-level peer-to-peer requests, e.g. Waku Mail Client requesting old messages from the Waku Mail Server.

**P2P Message**

This packet is used for sending the peer-to-peer messages, which are not supposed to be forwarded any further. E.g. it might be used by the Waku Mail Server for delivery of old (expired) messages, which is otherwise not allowed.

**Rate Limits**

This packet is used for informing other nodes of their self defined rate limits.

In order to provide basic Denial-of-Service attack protection, each node SHOULD define its own rate limits. The rate limits SHOULD be applied on IPs, peer IDs, and envelope topics.

Each node MAY decide to whitelist, i.e. do not rate limit, selected IPs or peer IDs.

If a peer exceeds node's rate limits, the connection between them MAY be dropped.

Each node SHOULD broadcast its rate limits to its peers using the rate limits packet. The rate limits MAY also be sent as an optional parameter in the handshake.

Each node SHOULD respect rate limits advertised by its peers. The number of packets SHOULD be throttled in order not to exceed peer's rate limits. If the limit gets exceeded, the connection MAY be dropped by the peer.

**Topic interest** (experimental)

This packet is used by Waku nodes for sharing their interest in messages with specific topics. It does this in a more bandwidth considerate way, at the expense of metadata protection. Peers MUST only send envelopes with specified topics.

This feature will likely stop being experimental in v1.

It is currently bounded to a maximum of 1000 topics. If you are interested in more topics than that, this is currently underspecified and likely requires updating it. The constant is subject to change.

### Payload Encryption

Asymmetric encryption uses the standard Elliptic Curve Integrated Encryption Scheme with SECP-256k1 public key.

Symmetric encryption uses AES GCM algorithm with random 96-bit nonce.

### Packet code Rationale

Packet codes `0x00` and `0x01` are already used in all Waku / Whisper versions.

Packet code `0x02` will be necessary for the future development of Whisper. It will provide possibility to adjust the PoW requirement in real time. It is better to allow the network to govern itself, rather than hardcode any specific value for minimal PoW requirement.

Packet code `0x03` will be necessary for scalability of the network. In case of too much traffic, the nodes will be able to request and receive only the messages they are interested in.

Packet codes `0x7E` and `0x7F` may be used to implement Waku Mail Server and Client. Without P2P messages it would be impossible to deliver the old messages, since they will be recognized as expired, and the peer will be disconnected for violating the Whisper protocol. They might be useful for other purposes when it is not possible to spend time on PoW, e.g. if a stock exchange will want to provide live feed about the latest trades.

## Additional capabilities

Waku supports multiple capabilities. These include light node, rate limiting and bridging of traffic. Here we list these capabilities, how they are identified, what properties they have and what invariants they must maintain.

Additionally there is the capability of a mailserver which is documented in its on [specification](./wms). 

### Light node

The rationale for light nodes is to allow for interaction with waku on resource restricted devices as bandwidth can often be an issue.

Light nodes MUST NOT forward any incoming messages, they MUST only send their own messages. When light nodes happen to connect to each other, they SHOULD disconnect. As this would result in messages being dropped between the two.

Light nodes are identified by the `light_node` value in the status message.

### Accounting for resources (experimental)

Nodes MAY implement accounting, keeping track of resource usage. It is heavily inspired by Swarm's [SWAP protocol](https://www.bokconsulting.com.au/wp-content/uploads/2016/09/tron-fischer-sw3.pdf), and works by doing pairwise accounting for resources.

Each node keeps track of resource usage with all other nodes. Whenever an envelope is received from a node that is expected (fits bloom filter or topic interest, is legal, etc) this is tracked.

Every epoch (say, every minute or every time an event happens) statistics SHOULD be aggregated and saved by the client:

| peer  | sent | received |
|-------|------|----------|
| peer1 | 0    | 123 |
| peer2 | 10   | 40  |

In later versions this will be amended by nodes communication threshholds, settlements and disconnect logic.

## Backwards Compatibility

Waku is a different subprotocol from Whisper so it isn't directly compatible. However, the data format is the same, so compatibility can be achieved by the use of a bridging mode as described below. Any client which does not implement certain packet codes should gracefully ignore the packets with those codes. This will ensure the forward compatibility. 

### Waku-Whisper bridging

`waku/0` and `shh/6` are different DevP2P subprotocols, however they share the same data format making their envelopes compatible. This means we can bridge the protocols naively, this works as follows.

**Roles:**
- Waku client A, only Waku capability
- Whisper client B, only Whisper capability
- WakuWhisper bridge C, both Waku and Whisper capability

**Flow:**
1. A posts message; B posts message.
2. C picks up message from A and B and relays them both to Waku and Whisper.
3. A receives message on Waku; B on Whisper.

**Note**: This flow means if another bridge C1 is active, we might get duplicate relaying for a message between C1 and C2. I.e. Whisper(<>Waku<>Whisper)<>Waku, A-C1-C2-B. Theoretically this bridging chain can get as long as TTL permits.

## Forwards Compatibility

It is desirable to have a strategy for maintaining forward compatibility between `waku/0` and future version of waku. Here we outline some concerns and strategy for this.

## Appendix A: Security considerations

There are several security considerations to take into account when running Waku. Chief among them are: scalability, DDoS-resistance and privacy. These also vary depending on what capabilities are used. The security considerations for extra capabilities such as [mailservers](./wms.md#security-considerations) can be found in their respective specifications.

### Scalability and UX

**Bandwidth usage:**

In version 0 of Waku, bandwidth usage is likely to be an issue. For more investigation into this, see the theoretical scaling model described [here](https://github.com/vacp2p/research/tree/dcc71f4779be832d3b5ece9c4e11f1f7ec24aac2/whisper_scalability).

**Gossip-based routing:**

Use of gossip-based routing doesn't necessarily scale. It means each node can see a message multiple times, and having too many light nodes can cause propagation probability that is too low. See [Whisper vs PSS](https://our.status.im/whisper-pss-comparison/) for more and a possible Kademlia based alternative.

**Lack of incentives:**

Waku currently lacks incentives to run nodes, which means node operators are more likely to create centralized choke points.

### Privacy

**Light node privacy:**

The main privacy concern with light nodes is that directly connected peers will know that a message originates from them (as it are the only ones it sends). This means nodes can make assumptions about what messages (topics) their peers are interested in.

**Bloom filter privacy:**

By having a bloom filter where only the topics you are interested in are set, you reveal which messages you are interested in. This is a fundamental tradeoff between bandwidth usage and privacy, though the tradeoff space is likely suboptimal in terms of the [Anonymity](https://eprint.iacr.org/2017/954.pdf) [trilemma](https://petsymposium.org/2019/files/hotpets/slides/coordination-helps-anonymity-slides.pdf).

**Privacy guarantees not rigorous:**

Privacy for Whisper / Waku haven't been studied rigorously for various threat models like global passive adversary, local active attacker, etc. This is unlike e.g. Tor and mixnets.

**Topic hygiene:**

Similar to bloom filter privacy, if you use a very specific topic you reveal more information. See scalability model linked above.

### Spam resistance

**PoW bad for heterogenerous devices:**

Proof of work is a poor spam prevention mechanism. A mobile device can only have a very low PoW in order not to use too much CPU / burn up its phone battery. This means someone can spin up a powerful node and overwhelm the network.

### Censorship resistance

**Devp2p TCP port blockable:**

By default Devp2p runs on port `30303`, which is not commonly used for any other service. This means it is easy to censor, e.g. airport WiFi. This can be mitigated somewhat by running on e.g. port `80` or `443`, but there are still outstanding issues. See libp2p and Tor's Pluggable Transport for how this can be improved.

## Appendix B: Implementation Notes

### Implementation Matrix

| Client | Version |
| ------ | ------- |
| go-ethereum (geth) | [v1.9.7](https://github.com/ethereum/go-ethereum/tree/v1.9.7) |
| status whisper | [25321](https://github.com/status-im/whisper/tree/25321b2c035b6e03dbae85a2f54cf89f9f873dd9) |
| nimbus | [9c19f](https://github.com/status-im/nim-eth/tree/9c19f1e5b17b36ebcf1c7513428818f585a3cb16) |
| status-go | [ed5a5](https://github.com/status-im/status-go/commit/ed5a5c154daf5362cdf0c35fd1bc204e6a6d49ae) |

| | Light mode | Mail Client | Mail Server | shh/6 | waku/0 |
| -: | :--------: | :---------: | :---------: |  :-: | :-: |
| **geth** | x | x           | x           | x | - |
| **status whisper** | x | x           | -           | x | - |
| **nimbus** | x | -           | -           | x | - |
| **status-go** | x | x           | x           |x | - |

### Recommendations for clients

Notes useful for implementing Waku mode.

 1. Avoid duplicate envelopes
 
	To avoid duplicate envelopes, only connect to one Waku node. Benign duplicate envelopes is an intrinsic property of Whisper which often leads to a N factor increase in traffic, where N is the number of peers you are connected to.

 2. Topic specific recommendations
 
	Consider partition topics based on some usage, to avoid too much traffic on a single topic.

### Node discovery

[Discovery v4](https://github.com/ethereum/devp2p/wiki/Discovery-Overview) SHOULD NOT be used, because it doesn't distinguish between capabilities. It will thus have a hard time finding Waku/Whisper nodes.

[Discovery v5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md) MAY be used. However, it is quite bandwidth heavy for resource restricted devices. Thus, some lighter discovery mechanism is used. For some ad hoc ideas, see the current ad hoc implementation used in the [Status spec](https://github.com/status-im/specs/blob/master/status-client-spec.md#discovery).

This is an ongoing area of research and will likely change in future versions.

Known static nodes MAY also be used.

## Footnotes

1. <https://github.com/ethereum/devp2p/blob/master/rlpx.md>

## Changelog

| Version | Comment |
| :-----: | ------- |
| 0.2.0 (current) | See [CHANGELOG](https://github.com/vacp2p/specs/releases/tag/waku-0.2.0) for more details. |
| [0.1.0](https://github.com/vacp2p/specs/blob/b59b9247f2ac1bf45c75bd3227a2e5dd87b6d7b0/waku.md) | Initial Release |


### Differences between waku/0 and waku/1 (WIP)

Features considered for waku/1:

- `topic-interest` packet code

### Differences between shh/6 waku/0

Summary of main differences between this spec and Whisper v6, as described in [EIP-627](https://eips.ethereum.org/EIPS/eip-627):

- RLPx subprotocol is changed from `shh/6` to `waku/0`.
- Light node capability is added.
- Optional rate limiting is added.
- Status packet has following additional parameters: light-node,
confirmations-enabled and rate-limits
- Mail Server and Mail Client functionality is now part of the specification.
- P2P Message packet contains a list of envelopes instead of a single envelope.

## Acknowledgements
 - Andrea Maria Piana

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).