---
type: Specification
title: Peer and broadcast channels
description: Generic message interfaces, transport interactivity, privacy profiles and anonymous replies.
status: draft
stale_after: 2027-04-15T00:00:00Z
---

<!-- cspell:words Mixnet mixnet's nothingmuch OHTTP onwloqro SURB surbs Zaddyy's Zaddyy -->

# 🤖 Peer and broadcast channels

## Scope and status

Channels deliver messages for the [protocol suite](overview.md). Peer channels carry payment negotiation and neighbor reconciliation; broadcast channels disseminate contributions to a group. This draft describes generic interfaces and their required characteristics. It leaves wire formats, concrete Rust trait signatures and transport selection open.

The interfaces separate message direction, addressing, interactivity, delivery and privacy. A bidirectional channel need not be connection based; a broadcast channel need not identify its sender. An asynchronous programming interface alone establishes neither property.

At the protocol boundary, Fungi consumes capabilities meeting those requirements. The application's [contact and identity integrations](contacts.md#capability-boundary-and-application-integration) handle preferred identifiers, discovery, authentication and revocation, and supply or withdraw usable capabilities. The protocol need not implement an identity system to use an authenticated channel.

## Message interfaces and composition

A send capability accepts a message of type `M`; a receive capability yields a message of type `M`. Each defines its own error type. A bidirectional interface combines send and receive capabilities, possibly with different message types in each direction. These operations suffice for both peer and broadcast communication; the endpoint's semantics determine which messages reach whom.

The following notation describes capabilities, rather than prescribing Rust declarations:

| Interface | Operation | Contract |
| --- | --- | --- |
| `SendChannel<M>` | `send(M)` | Submit one message under the channel's declared delivery and privacy profile. |
| `RecvChannel<M>` | `recv()` | Receive one message under that profile. |
| `Bidirectional<I, O>` | Both operations | Receive `I` and send `O`; no requirement that either operation completes first. |
| `ChannelBuilder` | Build from configuration or addressing data | Produce a capability, performing setup interactions where required. |

A successful send must specify what it acknowledges: acceptance into a local queue, storage at a relay, or receipt by the destination. These events do not establish application acceptance or inclusion in a completed transaction phase. Error types can describe transport-specific failures without requiring every transport to have a concept such as a broken connection or an unreachable peer.

Adapters provide composition without imposing it on every implementation:

* `Duplex<S, R>` pairs independent send and receive capabilities. Two simplex mailbox sequences can provide a bidirectional channel this way; a Tor stream is already bidirectional.
* Splitting a native bidirectional implementation into independently driven halves is a separate adapter when supported. Splitting and shared ownership are not requirements of the message interfaces.
* A conversion `f: T -> U` adapts a sender of `U` into a sender of `T`, or a receiver of `T` into a receiver of `U`. Fallible conversions extend the error type. Encoding and decoding are instances of this composition; byte serialization stays at the transport boundary.

Exclusive ownership of a send capability provides a natural place for mutable sequence state. Sharing it among concurrent writers is an additional policy. In particular, pairing two simplex channels does not authorize two writers to reuse one sequence of mailbox slots.

## Peer addressing

A peer channel targets one logical counterparty. Peer management associates the underlying capabilities with the expected peer identity, authentication evidence, trust policy and session lifetime. Possession of a transport address alone need not authenticate a payment counterparty.

The message interfaces do not require a `peer_id()` operation. A peer wrapper can hold an identity together with the channel; a receive endpoint may instead accept messages from many senders without a separate accept operation. Nym raw messaging is an example of the latter shape. Demultiplexing and authentication then establish which logical exchange a message belongs to.

A long-lived contact can have addresses on several transports. Its identity and trust relationship need not change when its transport changes. Contacts can represent the user's other devices, trusted collaborators or peers trusted only for liveness. These relationships differ from the coin-associated validator identities used in Byzantine construction; [peer introduction issue #60](https://github.com/fungi-protocol/fungi/issues/60) keeps them separate.

Peer identity and anonymous initiation can coexist. An initiator may authenticate a payment recipient while withholding its own identity. A reply capability can let the recipient answer without learning a stable return address. Authentication in the reverse direction is a separate requirement and may deliberately be absent.

Contact channels, introductions and payment negotiation build on peer channels, and are described in the [contacts chapter](contacts.md#long-lived-authenticated-contact-channels).

## Broadcast addressing

A broadcast channel targets a domain, such as a transaction-construction instance or an order-book topic. A sender submits a contribution to that domain; a receiver obtains contributions admitted by its subscription and protocol rules. Domains need explicit membership or subscription rules, namespace separation and retention policies. A secret topic can restrict discovery, but does not itself hide authorship or prevent members from sharing it.

Broadcast can be implemented by gossip over peer channels or by a specialized dissemination service. Neither implementation inherently supplies Byzantine reliable broadcast, agreement, a total order, or evidence that every participant has received a message. Those properties require additional protocol rules and assumptions. The [block DAG](optimistic-track.md) provides causal evidence for construction and recovery; its interpretation remains separate from transport delivery.

In the honest and semi-honest setting, a peer can initiate a session in its connected component and other peers decide whether to relay and participate. The initiation message binds the broadcast topic to a common construction context, including initial PSBT state and parameters; see [session initiation issue #61](https://github.com/fungi-protocol/fungi/issues/61). This transport-independent binding must survive bridging between different delivery mechanisms.

A broadcast endpoint need not return the original author's identity. Anonymous contributions can be received without an attribution field, while validator blocks can authenticate the validator that acknowledges a set of contributions. The validator that relays or includes an output is not thereby its owner.

### Gossip and reconciliation

A minimal gossip exchange announces a content identifier, requests missing content and returns the requested object. [Peer set reconciliation](set-reconciliation.md) compares inventories to suppress redundant announcements and find missing identifiers; fetching preimages remains a separate operation. The same mechanism serves permissionless order-book gossip and permissioned construction.

Anonymous submission also needs a way to inject a previously unannounced object. The existing peer-message notes propose unsolicited push, followed by forwarding to a randomly selected neighbor and announcements to other neighbors. This is a candidate dissemination policy, not an established anonymity guarantee. An announce-then-pull exchange with an identified origin can disclose who first held a contribution, even if later gossip hides its route.

A peer channel authenticates an immediate neighbor where required. That authentication need not identify the author of a relayed contribution. Conversely, stripping the author field cannot undo attribution already revealed by the submission channel.

## Channel characteristics

Each usable channel profile answers the following questions. Direction and addressing are independent of the remaining properties.

| Characteristic | Questions the profile must answer |
| --- | --- |
| Direction | Send only, receive only, or bidirectional? Are the two directions independently usable? |
| Addressing | One logical peer, one reply capability, or a broadcast domain? Can one receiver accept many senders? |
| Interactivity | Does setup or continued use require timely participation by the destination peer? What state expires if it pauses? |
| Delivery | What does send success mean? Can messages be lost, duplicated or reordered? What are retention and retry limits? |
| Authentication and confidentiality | Which endpoint or message author is authenticated? Who can read or modify a payload? |
| Unlinkability | Which messages or connections should an observer be unable to associate, and under what assumptions? |
| Metadata protection | Which observers can learn addresses, timing, sizes, volumes, topics or session associations? |
| Resource limits | What bounds message size, queued data, outstanding work and reply capacity? |

### Interactivity relative to the data layer

Here, **connection based** means that setup or continued delivery requires interactivity on a shorter timescale than the data layer's communication model permits peers to assume. A live stream can fail while a construction instance remains valid and is still waiting for delayed messages. This property concerns operational requirements; it does not impose synchrony on the transaction protocol.

Tor onion-service streams require interactive introduction and rendezvous before stream use, and depend on live circuits.[^tor] A builder may therefore connect or accept, and the resulting channel can become unusable after connection loss. Rebuilding that transport need not restart construction or erase previously acknowledged inventory state.

Nym's raw mixnet messaging API instead routes individual messages to an address without establishing a stream to the destination peer.[^nym-messaging] Local client registration, infrastructure connections, delivery retries and finite retention still have their own availability requirements. Connectionless peer messaging does not imply stateless implementation, unlimited offline delivery or a bounded delivery time. A stream adapter over a mixnet introduces additional session semantics and must be assessed separately.

Linked store-and-forward mailboxes likewise separate a sender's submission from a recipient's retrieval. Both endpoints may interact with a storage service without being simultaneously available to each other. An initial key exchange can still require a reply before an established symmetric channel is usable; distinguish this handshake from each subsequent message delivery.

| Delivery mechanism | Peer interaction | Native shape and lifetime |
| --- | --- | --- |
| Tor onion-service stream | Interactive setup; both endpoints participate in a live connection. | Bidirectional byte stream with message framing; rebuild after connection loss. |
| Nym raw mixnet messages | Addressed messages without a destination-peer stream handshake. | Message receive endpoint and addressed sends; replies can use SURBs. |
| Linked mailboxes over OHTTP | Independent submission and retrieval through infrastructure; any channel handshake is separate. | One writer and one reader per directional sequence; pair two sequences for bidirectional use. |
| Broadcast by gossip | Inherits each neighbor channel's interaction requirements. | A logical domain spanning multiple peer exchanges; it can outlive any one connection. |

An `async fn` can wrap any of these mechanisms. Calling a transport "synchronous" would conflate its interactivity with the synchrony assumptions of the agreement protocol, so profiles state the actual setup, availability and expiry requirements.

### Privacy and type separation

**Connection unlinkability** concerns associations between separate connections. Messages on one connection can remain linkable to each other. **Message unlinkability** concerns associations between individual messages. Both require an observer and scope: privacy relative to a destination peer differs from privacy against relays or a network observer.

An isolated Tor connection can hide the initiator's network address from the responder while exposing the association among messages sent on that stream. Opening a fresh stream for each contribution is insufficient to assert message unlinkability: circuit sharing, isolation configuration, timing, volume and application fingerprints also matter. A mixnet's message-oriented interface alone is likewise insufficient; its traffic model and the application's contents and reply identifiers affect the claim.

Privacy tags on capabilities and builders can prevent structurally compatible channels from being used in the wrong role. A code path requiring anonymous output submission should require the corresponding declared message-unlinkability and metadata profile. Naming the property directly avoids an ambiguous "anonymous" tag that might mean only that no explicit sender identifier was attached. Such tags express assumptions and implementation obligations; they are not proofs that a transport deployment satisfies them.

Adapters must preserve these obligations. A serialization adapter cannot manufacture unlinkability; adding a stable sender key, cookie or session tag can destroy it. Combining send and receive halves does not automatically give both directions the same privacy guarantees.

## Single-use reply blocks

A single-use reply block (SURB) gives the responder one anonymous return capability. The initiator prepares a protected return route and the material needed to construct a reply. The responder can use it without learning the initiator's destination address. Nym exposes this mechanism for anonymous replies.[^nym-surbs]

At the interface boundary, a SURB can be modeled as a consumable send capability with bounded payload capacity. It supplies a return path without requiring a persistent bidirectional connection. The surrounding adapter can pool capabilities, fragment replies and request replenishment, but those operations have visible resource and latency costs.

The reply contract must account for:

* Single use: consumption must prevent accidental reuse. A failed or canceled send needs an explicit rule for whether the capability may already have been spent.
* Capacity: one logical reply may consume several SURBs. Reserve enough capacity for control traffic or define how exhaustion recovers.
* Validity: route availability, key changes and local storage policy can invalidate a reply capability. Profiles must specify expiry behavior without assuming a universal Nym lifetime.
* Replenishment: obtaining more capabilities introduces further communication. It must respect the data layer's tolerance for delay and must be bounded against resource exhaustion.
* Linkability: the request and reply are associated by the responding application. Nym sender tags can also group reply capabilities into a session. Hiding a return address does not make all messages in that session unlinkable.

A SURB routes a reply; the application must separately establish any required counterparty authentication, confidentiality and integrity. Interactive payment delegation can use a reply path while keeping the initiator anonymous to the recipient. Public output submission should not acquire a stable session identifier merely because the transport supports replies.

## Browser clients, mailboxes and bridging

The [OHTTP priorities discussion](https://github.com/fungi-protocol/fungi/issues/73#issuecomment-5816227367) identifies browser and mobile web clients as an immediate target. Its minimal approach is a simplex peer channel with one writer and one reader, then ordinary epidemic gossip over those channels. A specialized multiparty broadcast service is a separate optimization path. Nym's browser messaging client is another candidate; browser deployment does not inherently require OHTTP.

For the existing Payjoin directory path, compatibility is with directory infrastructure, rather than with BIP 77 payment clients. That path must preserve the directory's fixed-size opaque framing and write-once/read-once traffic profile. A Fungi logical message may need framing across several such payloads. Treating one mailbox as a contended multiparty broadcast destination changes its privacy and censorship properties; it is not a consequence of implementing the send and receive interfaces.

A future service designed for multiple writers and readers can choose a different storage and dissemination model. Its profile must specify request and response size protection, key discovery and consistency, relay selection and retry policy, and confidentiality against stored-traffic decryption. The discussion calls for evaluating post-quantum protection at the payload, OHTTP and TLS layers. These are design requirements to evaluate, not claims that the existing directory path provides them.

The [Tor/OHTTP bridge plan](https://github.com/fungi-protocol/fungi/issues/10) forwards logical messages across transports. Its [identity and deduplication rules](https://github.com/fungi-protocol/fungi/issues/8) belong to the message envelope and inventory layer. A bridge must preserve the construction namespace and content identity while evaluating what new metadata its operators can observe. Bridging transports neither transfers their privacy guarantees automatically nor establishes anonymity of the resulting broadcast path.

## Applying the interfaces

| Use | Addressing and capabilities | Additional requirements |
| --- | --- | --- |
| Initiation through a trusted contact | Long-lived authenticated contact channel over replaceable transport capabilities. | Initiate an honest or semi-honest session, potentially within a BFT session; persist the contact binding and safe messaging state. |
| Manual introduction and session initiation | Out-of-band descriptor or invitation over an existing contact channel, then peer and broadcast capabilities. | Initiate an honest or semi-honest session, potentially within a BFT session; verify its context and scope, with independent local acceptance. |
| Payment planning, coalition coordination and confirmation | Honest or semi-honest peer session over a contact channel or manual introduction. | Notify the payee, agree which coalition attempts to pursue, then confirm the DAG's message-set commitments, including proofs, before signing; the construction session may be BFT. |
| Anonymous transaction contributions | Send to the construction broadcast domain. | Message unlinkability and metadata protection appropriate to the fault model; dissemination must preserve required inclusion. |
| Inventory and content exchange | Neighbor send and receive. | Scope requests to advertised roots, validate contents, bound work and tolerate delayed or repeated messages. |
| Order-book dissemination | Broadcast subscriptions and neighbor exchanges. | Permissionless admission, retention and resource policies; relayed proofs remain independently verifiable. |

Reconciliation exchanges name their immutable root and scope, so a delayed response remains interpretable after a newer advertisement or a transport rebuild. Retaining a peer's acknowledged baseline can avoid repeating work, provided the peer binding and evidence remain valid. A newly created channel to a different or anonymous counterparty must not inherit another peer's knowledge claims merely because it uses the same transport.

Streaming reconciliation describes a sequence of messages, not a requirement for a live byte stream. A narrowing range reply stops further parent-scope transmission when processed; cells already in transit can arrive later and must remain distinguishable from child-scope cells. For store-and-forward or mixnet delivery, scheduling and outstanding-data limits must account for that delay.

Neither a successful send, an exhausted receive queue nor an expired connection establishes reconciliation quiescence or phase completion. The consuming protocol interprets evidence of completion under its communication and fault assumptions.

## Design provenance and open choices

This draft develops the channel and gossip portions of the existing `peer-messages` draft (`onwloqro`, snapshot `1f68a962`). That branch also contains transaction, block and introduction notes; those remain separate. The newer DAG and reconciliation specifications supply the synchronization context here.

Relevant feedback in the Fungi monorepo includes:

* Cindy's [channel abstraction review](https://github.com/fungi-protocol/fungi/pull/17#discussion_r3994764107): builders can perform interactive connection setup or construct capabilities from an address; [typed adapters](https://github.com/fungi-protocol/fungi/pull/17#discussion_r4020203904) separate message conversion from transport.
* Cindy's [typed channel review](https://github.com/fungi-protocol/fungi/pull/32#pullrequestreview-5223909618): privacy tagging, interactivity and generic adapters are distinct concerns. Later feedback [names unlinkability directly](https://github.com/fungi-protocol/fungi/pull/32#discussion_r4106065317) and [places peer identity with peer management](https://github.com/fungi-protocol/fungi/pull/32#discussion_r4106122178).
* The [Cap'n Proto bridge review](https://github.com/fungi-protocol/fungi/pull/35#discussion_r4021384973), posted as `nothingmuch`: keep message and error types generic instead of assuming connection-specific failures at the RPC boundary.
* The [in-memory channel review](https://github.com/fungi-protocol/fungi/pull/34#pullrequestreview-5217326680): typed message channels, connection setup and store-and-forward simulation can be composed as separate layers.
* Cindy's [bidirectional adapter follow-up](https://github.com/fungi-protocol/fungi/pull/83#discussion_r4232570038): `Bidirectional` names the capability and `Duplex` names the pair adapter.
* 0xZaddyy's [linked mailbox review](https://github.com/fungi-protocol/fungi/pull/107#discussion_r4186249336): separate pure derivation, mutable sequence state and I/O. A [handshake can establish a subsequent symmetric channel](https://github.com/fungi-protocol/fungi/pull/107#discussion_r4186442342); this is design direction, not a specified cryptographic profile.
* 0xZaddyy's [simplex transport review](https://github.com/fungi-protocol/fungi/pull/111#pullrequestreview-5417305690): expose exclusive ownership of each simplex sequence, then pair opposite directions into a bidirectional channel.
* The [transport tracking issue #6](https://github.com/fungi-protocol/fungi/issues/6), [OHTTP discussion #73](https://github.com/fungi-protocol/fungi/issues/73#issuecomment-5816227367), and [naming follow-up #137](https://github.com/fungi-protocol/fungi/issues/137): anonymous broadcast over interoperating transports is the goal; simplex peer channels plus gossip provide the initial implementation path.

Open choices include concrete trait signatures, representation of addressing and privacy tags, contact introduction and key-management profiles, persistence and resumption rules, builder roles, receive demultiplexing, delivery acknowledgements, reconnect policy, SURB budgets and a transport-specific threat model for each anonymous submission path.

## References

[^tor]: Tor Project. [Onion-service protocol overview](https://spec.torproject.org/rend-spec/protocol-overview.html), including interactive rendezvous and stream establishment.

[^nym-messaging]: Nym. [Mixnet messaging module](https://nym.com/docs/developers/rust/mixnet), distinguishing individually routed messages from the stream interface.

[^nym-surbs]: Nym. [Anonymous replies with SURBs](https://nym.com/docs/network/traffic/anonymous-replies), including sender tags, replenishment and deployment-dependent lifetimes.
