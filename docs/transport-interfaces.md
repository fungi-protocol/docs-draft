---
type: Specification
title: Abstract transport interfaces
description: Send and receive channel traits, their privacy tags, and how message unlinkable broadcast is built from them by gossip.
status: draft
stale_after: 2027-04-22T00:00:00Z
---

# 🤖 Abstract transport interfaces

Protocol code sends and receives messages through two traits, `SendChannel` and `RecvChannel`. Each send channel declares in its type what its messages reveal to their destination. The compiler can then reject a channel that would leak where unlinkability is required. A broadcast channel is built out of peer channels, by gossip.

## Scope

The transport layer takes care of message delivery. View consistency and the communication model belong to other layers. If the communication model is synchronous, protocol timeouts are more or less independent of transport timeouts. Wire encodings and framing are out of scope for this document.

Good protocol design should allow progress even if a channel handles only one message at a time. The network adversary can already impose this in the asynchronous and partially synchronous models.

## Channels

```rust
pub trait SendChannel<M = Vec<u8>>: Send {
    /// Privacy properties for this sender in relation to its destination peer.
    type Privacy: Privacy;
    type SendError: Error + Send + Sync + 'static;
    fn send(&mut self, message: M) -> impl Future<Output = Result<(), Self::SendError>> + Send;
}

pub trait RecvChannel<M = Vec<u8>>: Send {
    type RecvError: Error + Send + Sync + 'static;
    fn recv(&mut self) -> impl Future<Output = Result<M, Self::RecvError>> + Send;
}

pub trait Bidirectional<I = Vec<u8>, O = Vec<u8>>: RecvChannel<I> + SendChannel<O> {}
```

These match `crates/transport` in the Fungi repository at `5c0f08e`.

* **One message per call.** `send` submits one message. `recv` yields one message.
* **Exclusive ownership.** Both operations take `&mut self`. Normally one consumer reads a channel and one writes it. If concurrent writes are ever needed, a layer on top wraps the channel in an `Arc` with interior mutability, such as a mutex.
* **Channel-defined errors.** Each channel defines its own error types. Failures such as "closed", "too large" or "unreachable" are specific to connection based channels, so the traits do not name them.
* **Typed messages.** The message type defaults to bytes. Transports provide byte channels at the boundary. Conversion adapters turn them into typed channels of the same shape.
* **Either direction alone.** An implementation may provide sending, receiving, or both. A consumer requires only the direction it uses. Adapters bridge the two shapes.
* **Many senders.** A receive channel need not belong to one sender. Some transports deliver every message for an address to a single receive interface.
* **Minimal `Bidirectional`.** It adds no methods. Its input and output message types may differ.

## Privacy tags

`Privacy` is a sealed trait. Its implementations are uninhabited types, used only as tags.

| Tag | Messages linkable to each other | Messages linkable to a sender identifier |
| --- | --- | --- |
| `MessageUnlinkability` | No | No |
| `ConnectionUnlinkability` | Yes | No |
| `Unspecified` | Not declared | Not declared |

* A tag describes the sending side. It states whether submitting a message on the channel leaks information to the destination peer.
* A missing sender field does not mean anonymity. Identity can still leak, or be inherent to the transport. For example, a connecting peer may authenticate to the accepting peer. Authenticated channels, and channels whose properties are not yet verified, are `Unspecified`.
* Code that writes messages that must be unlinkable requires a channel with the matching tag. The compiler checks the bound. This prevents privacy leaks that the structural types alone would allow. Concurrent PSBT construction tracks modifiability in types similarly.
* The tags name each property directly. The words "anonymous" and "pseudonymous" invite confusion: an unlinkable connection is easily read as anonymity of the connection, and a linkable one as pseudonymity of a long-term identity.
* Metadata hiding is always with respect to the initiator: the sender, or the connecting side of a stream. The responder is identified by its peer ID or address.
* `RecvChannel` carries no tag.

More properties are not tags yet:

* **Interactivity.** A connection based channel requires the peer to be available. The working adjective for this is "synchronous". It affects how channels are built and how long they live. It probably need not be a type, but may prove useful.
* **Destination.** A channel reaches either a single peer or many (multicast). See [open questions](#open-questions).
* **Reliability.** Whether delivery is reliable or unreliable.
* **Metadata hiding.** Whether the channel hides transport metadata from the responder.

## Adapters

Composition lives in adapters, not in the core traits.

**`Duplex<S, R>`** pairs a send channel and a receive channel into a `Bidirectional` channel. `into_parts` recovers them, so no borrowed halves are needed. The send destination and the receive source may differ. Its `Privacy` is that of `S`. Pairing is the right shape for talking to one particular peer, as net settlement requires.

**Split** is the dual of `Duplex`. It is not implemented yet. A native bidirectional channel `T` moves into an `Arc<T>`. Each half holds a clone and implements one direction. This assumes `T` is safe to share. Splitting consumes the channel. It is not a required trait method, since it may not be practical for some transports.

**Conversion** changes the message type. Given `f: T -> S`:

* a `SendChannel<S>` becomes a `SendChannel<T>` by applying `f` before each send (contramap);
* a `RecvChannel<T>` becomes a `RecvChannel<S>` by applying `f` to each received message (map);
* a bidirectional channel takes a pair of functions, one per direction.

Conversion adapters are not implemented yet either. One adapter struct per underlying trait suffices. A fallible conversion extends the underlying channel's error type. Invalid data from a peer is then handled generically. Encoding and decoding are conversions.

A paired codec type is deferred. When it is added, it should capture that round trips preserve information: `f ∘ g = id = g ∘ f`.

A `Stream` adapter for receive channels is deferred until a consumer needs it.

## Builders

```rust
pub trait ChannelBuilder<M = Vec<u8>>: Send {
    type Input: Send + Sync;
    type Privacy: Privacy;
    type Channel: SendChannel<M, Privacy = Self::Privacy>;
    type BuildError: Error + Send + Sync + 'static;
    fn build(
        &mut self,
        input: &Self::Input,
    ) -> impl Future<Output = Result<Self::Channel, Self::BuildError>> + Send;
}
```

* A builder declares the `Privacy` of the channels it builds.
* Some builders interact. They open a connection. Their channels may become unusable when the connection drops.
* Other builders are stateless. They build a channel from an address alone.
* A connection based transport has two builders for the same channel type: one connects and one accepts. A connectionless transport needs only one builder per peer, if peers authenticate mutually.
* The main shared-secret builder derives a simplex channel and returns only one end. Both peers can use the same builder. Two writers on one simplex channel can only conflict, which harms privacy and censorship resistance. It is better to make that case impossible to construct. A bidirectional channel is a pair of simplex channels derived from the same secret, one per direction.

## Message unlinkability from connection unlinkability

A `ConnectionUnlinkability` channel is message unlinkable if it carries only one message. It may instead carry one set of inherently related messages, such as a transaction input and its signature. There are caveats: timing metadata can still leak, for example.

A sketch, not from the reviews: an adapter could hold a builder whose `Privacy` is `ConnectionUnlinkability`, build a fresh channel for each message, and implement `SendChannel` with `Privacy = MessageUnlinkability`.

## Broadcast channels from peer channels

A broadcast channel is built out of peer channels. It implements the same two traits, so protocol code that broadcasts sees only `SendChannel` and `RecvChannel`. Tagging is about how a channel is used, and the destination is an [open question](#open-questions).

* **Send.** `send` submits a message to the group. The caller does not choose which peers the gossip reaches.
* **Receive.** `recv` yields messages from the group. It need not authenticate or distinguish anonymous senders.

Gossip always happens within a namespace or topic. Some namespaces are ephemeral, derived from a shared secret with a tagged hash. Others are well known, such as a public order book. If the namespace includes a block hash, message IDs can be short.

A broadcast channel uses two kinds of peer channel:

* **Neighbor channels** connect a peer to each of its gossip neighbors. They are bidirectional, and may have any tag, including authenticated ones.
* **Submission channels** carry the peer's own messages into the group anonymously. A builder tagged `MessageUnlinkability` builds them.

### Synchronization

Convergence is the point of a transaction construction session. Each peer therefore synchronizes the set of all session messages with each of its neighbors. Order does not matter: the result is the union of the sets. This includes the blocklace, the session's block DAG. A peer's broadcast `recv` yields each message when it first enters the peer's set.

Every message carries a state commitment: the Merkle roots of the sender's sets, with height and set sizes. Fetching an ever-growing set of sets is a resource exhaustion risk. Blocks are necessary for progress, and cordiality rate limits them.

Honest peers should converge on the messages to include before producing blocks. A peer should be eager to gossip, but not too eager to produce blocks.

The goal is that all participants, even those with asymmetric capabilities such as limited bandwidth, arrive at the same set of messages. This repeats over multiple epochs.

### Gossip messages

These travel over neighbor channels. The minimal set suffices for the honest setting:

* **announce** a message by its hash;
* **pull** a message by its hash;
* **push** a message that a neighbor pulled.

The semi-honest and Byzantine settings add **unsolicited push**, used for anonymous broadcast. A peer that receives a new unsolicited message pushes it to one randomly selected neighbor and announces it to its other neighbors.

### Anonymous submission

1. The sender pushes its message, unsolicited, to a group member over a submission channel.
2. That member handles it as any new unsolicited message. From then on it is an ordinary element of the synchronized set. A message that is not linked to its owner can be relayed even on a channel that is not anonymous.
3. The sender watches synchronization for its own message. It keeps submitting anonymously until it is satisfied that the message has been disseminated.

So the broadcast send channel's unlinkability rests on the submission channels. It does not rest on the neighbor channels.

### Dissemination methods

**Epidemic gossip** is the minimal approach: announce new messages to neighbors, which pull the ones they lack. It works over simplex peer channels. It avoids write contention and multiple readers per destination, and the potential censorship these allow. It will likely win on latency. But announcements are sent for every message to every neighbor, and a peer may hear of the same message many times.

**Set reconciliation** is a later improvement of blocklace dissemination efficiency. Two neighbors compare their sets and learn which messages each is missing. It throttles or suppresses announcements. Communication then grows with the difference between the sets, not with their size. It will likely win on communication complexity. Range based reconciliation and rateless invertible Bloom lookup tables (RIBLT) are candidates. Multiparty set reconciliation has not been studied as far as we know; recent work covers two parties. A peer can still reconcile with each neighbor in turn.

**Network coding**, such as algebraic gossip on top of the same structure, offers a whole spectrum of further possibilities, with some additional complications.

The same building blocks may not suit both open broadcast (coalition formation) and closed group broadcast (transaction construction). Message counts and sizes also vary: small messages on the happy path, larger ones for validity proofs.

### Topology

* **Hub and spoke.** Some peers are better connected, and many others connect to them.
* **Clique**, or multicast. Every peer is connected to every other peer, with no distinction between them. A transport that can send to any peer at any time, without a connection, behaves like this.
* **Gossip.** In the random telephone model, any peer may be reachable, and each round a peer shares new information with one neighbor. In the local model, each peer is connected to only part of the network. Fungi mainly targets the local model, with a limited number of connections per peer.

Connection based channels limit a peer to a more or less static local topology. A transport that can send a message to any peer at any time behaves more like a low capacity clique, or the random telephone model.

## Uses

Each message type states which channel suits it. All these uses are message based. A stream transport becomes message based by framing.

| Use | Direction | Sender |
| --- | --- | --- |
| Session initiation with known peers (honest and semi-honest settings) | Bidirectional | Authenticated |
| Interactive payments: notification and confirmation for net settlement | Bidirectional, reliable, to one peer | The receiver is authenticated and can reply. The sender need not be authenticated, and game theory suggests it shouldn't be. |
| Offering co-spend proposals in coalition formation | Bidirectional | Sender anonymous |
| Anonymous broadcast within a session, such as transaction outputs in the semi-honest and Byzantine settings | Unidirectional, to the broadcast domain | Message unlinkable and metadata hiding |

## Open questions

From the reviews and issues:

1. **Destination tag.** An associated type on `SendChannel` and `RecvChannel` could say whether a channel reaches a single peer or uses multicast, like `Privacy`. A builder cannot express a separate `PeerChannel` trait generically without trait objects. The lean is to keep peer identity in a wrapper type for connection management. `PeerChannel` is deferred to a later PR; the decision is still open.
2. **Interactivity.** A type tag, or only a concern of construction and lifetime. The lean is no tag.
3. **Broadcast topic.** Bind a session's messages to a common root without depending on the transport, perhaps with a hash tag.

Added by this draft, not from the reviews:

4. **Name.** The code says `ConnectionUnlinkability`. Should the spec say "channel unlinkability"?
5. **Tag order.** `MessageUnlinkability` implies `ConnectionUnlinkability`, but the sealed tags have no order. Should a bound that requires `ConnectionUnlinkability` also accept `MessageUnlinkability`?
6. **Related messages.** The one-message adapter above is a sketch. How does a caller group related messages onto one channel?
7. **Own messages in the synchronized set.** If a sender adds its own message to its set before submission, synchronization shows that it held the message first. Should a sender keep its own message out of its set until it arrives through synchronization?
