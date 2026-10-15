---
type: Specification
title: Contacts, devices and discovery
description: Introduction, TOFU, trust policy, revocation, application interoperability across transports, contact channels and payment negotiation.
status: draft
stale_after: 2027-04-15T00:00:00Z
---

<!-- cspell:words Nostr Nunchuk Nunchuk's pkarr SURB -->

# 🤖 Contacts, devices and discovery

## Scope and identity model

The goal is to authenticate a peer once, then maintain that relationship as its devices, addresses and supported transports change. A contact can initially be reachable over one transport and later advertise another without requiring the user to repeat the original identity check. Authentication of the update and of the new endpoint carries the established binding forward.

This chapter describes application-level contact management above the [message channels](channels.md). It develops the transport-independent contact model in [peer introduction issue #60](https://github.com/fungi-protocol/fungi/issues/60). At the Fungi protocol boundary, the caller supplies a communication capability meeting the role's authentication, delivery and privacy requirements. The surrounding application and its identity integrations establish and maintain that capability. Owner-controlled keys, device authorization and a web of trust between connected peers can supply the bindings. There is no required external identity authority, registration service or globally trusted introducer.

| Concept | Meaning |
| --- | --- |
| Contact ID | A local handle for the intended counterparty and its authentication history, devices and trust policy. It need not be a global public identifier or a transport address. |
| Contact authentication binding | Evidence accepted under local policy that associates credentials with this relationship, such as a contact-owner key or an introduction through connected peers. Authorized changes can preserve the local contact handle. |
| Device ID | An authenticated identity for one device or installation acting for the contact, with its own keys and permitted actions. |
| Transport peer ID or address | A transport-scoped identifier or delivery capability. Its namespace includes the transport and network; its binding records the device or contact authorized to use it. |
| Channel instance | A particular usable send/receive capability. Replacing it need not change the contact, device or transport identity. |

A contact may have several devices, and each device may support several transport endpoints. Some transports combine addressing and authentication in one identifier; others require separate authentication material or offer only a consumable reply capability. The contact layer records the relevant binding rather than forcing every transport into a persistent `PeerId` API. Display names and user labels do not authenticate any of these identities.

## Capability boundary and application integration

Fungi uses the capability supplied to it. It need not resolve names, interpret identity records or implement device enrollment. The application chooses how to turn a preferred identifier and local trust policy into an appropriate capability. Contact management and identity backends can expose these integration operations:

| Operation | Information supplied to the contact/channel layer |
| --- | --- |
| Resolve a contact | Devices and supported transport endpoints currently accepted for that contact. |
| Assess an introduction or update | The contact/device association, verification provenance, permitted actions and any unresolved conflict. |
| Authenticate a route | Evidence that the opened channel is bound to the expected contact and device under the accepted identity scheme. |
| Observe identity changes | Accepted endpoint additions, device changes, key transitions or revocations, with enough context to preserve continuity. |

These are conceptual application-integration operations, not additional Fungi protocol requirements or fixed Rust signatures. A backend may authenticate an endpoint update using an owner key, a device authorized by the owner, or an introduction accepted through the local web of trust. The application selects the behavioral trust assumptions; cryptographic verification alone does not supply them. Record storage, key-management procedures and the evidence format remain behind the capability boundary.

## Application interoperability and transport negotiation

Applications need a more explicit common contract than the Fungi protocol itself. A versioned **contact exchange protocol** can specify how applications exchange transport support, authenticated endpoint information and lifecycle updates. An implementation can use its preferred identity backend internally, while a shared exchange profile gives another application enough information to interpret and authenticate its records.

### Bootstrap requirements

Peers must share at least one usable transport and a compatible way to establish an authenticated channel over it. Sharing a transport name alone is insufficient: framing, transport profile and authentication must also be compatible. The initial introduction or existing contact configuration supplies the candidate routes and authentication information needed to establish that first channel. Transport discovery after authentication cannot bootstrap a connection when no initial route exists.

Once the channel is authenticated under the locally accepted contact binding, peers negotiate a contact-protocol version and exchange their supported transports. A bidirectional control exchange can use a native bidirectional channel, paired simplex channels or an authenticated reply mechanism. It need not require a continuously open stream or simultaneous availability.

### Exchange semantics

The following message roles form a draft interoperability contract. Their names describe semantics; canonical encoding remains to be specified.

| Message role | Required information and effect |
| --- | --- |
| Capabilities | Supported contact-protocol versions, optional features, and transport/authentication profile identifiers and versions. Establish which profiles both peers understand. |
| Endpoint advertisement | Record identity, contact/device binding under a named identity profile, transport profile, peer ID or delivery address, and the authentication and update context required by that profile. Supplies a candidate additional route to the established contact. |
| State request | The desired scope and any previously accepted update checkpoint. Requests current endpoint/device information or missing updates after an offline period. |
| State response | An authenticated snapshot or update sequence with its scope and freshness context, including withdrawals needed to avoid reviving revoked bindings. |
| Withdrawal or revocation | The affected endpoint, device, credential or permission; authorizing evidence and update context under the named profile. Triggers the receiving application's revocation handling. |
| Update result | Which records were accepted, rejected, retained as unsupported or left unresolved. Distinguishes receipt from acceptance under local policy. |

Each exchange is bound to its authenticated channel context and intended contact relationship. A profile must define how record issuers are authorized, how device bindings are verified and how updates are ordered or checked for replay. Authentication of the immediate channel alone does not authorize every claimed device enrollment or third-party introduction.

An application can retain an authenticated advertisement for an unsupported transport without being able to use that transport yet. When both sides support the profile, it can construct another channel using the advertised information and verify the expected endpoint binding. Selecting that route preserves the established contact and assurance level. An update's receipt or a claimed privacy property does not by itself prove that the route satisfies a Fungi role's requirements.

Transport support, endpoint availability and permission to use an endpoint are distinct. Peers can support the same transport but have no currently usable, authorized route over it. A withdrawn device or endpoint cannot be revived merely by retrying an old advertisement. If no shared usable route remains, the application reports the contact unavailable; a later compatible route can restore communication without discarding the contact history.

### Specification boundary

Cross-application interoperability requires a canonical envelope, message encoding, namespaced profile identifiers, version-negotiation rules and error/resource limits. Each supported identity/transport profile must specify the authentication evidence, address interpretation and revocation/freshness semantics its adapter uses. Unknown optional profiles should not prevent exchange over an already supported profile; unsupported required features must produce an explicit incompatibility result.

These are application-level wire-protocol requirements. They allow distinct applications to exchange usable contact capabilities without standardizing their databases, UI, owner-key management or web-of-trust policy. The base encoding and concrete profiles remain specification work; this draft establishes their responsibilities rather than claiming byte-level interoperability already exists.

## Preferred identifiers and identity integrations

Users may prefer an existing identifier and its ecosystem rather than create a new Fungi-specific identity. An application can support several such choices while presenting the same channel capabilities to the protocol. Identifier kind, namespace and verification provenance remain attached to the contact record; translating to a local contact handle does not discard them.

| Integration | Existing identity or discovery model | Application responsibility |
| --- | --- | --- |
| Owner-controlled key or direct introduction | A key or peer binding selected through local verification, TOFU or the web of trust. | Apply the owner's authorization and local introduction policy. |
| Matrix | Existing Matrix user and device identities, verification and cross-signing. | Preserve the distinction between account identifiers and verified cryptographic device/user bindings. |
| pkarr | Public-key-addressed, signed DNS records resolved through a DHT or relays. | Verify records under the accepted key and interpret their endpoint information under the selected profile. |
| Nostr | Public-key identities and signed events, with optional NIP-05 names. | Preserve the accepted key binding when resolving names or interpreting identity updates. |

[Matrix cross-signing](https://spec.matrix.org/latest/client-server-api/#cross-signing) supports verification across a user's devices. Nunchuk's [earlier Matrix-based collaborative-wallet design](https://nunchuk.io/blog/group-wallet) is an example of reusing an existing communications ecosystem. [pkarr](https://github.com/pubky/pkarr) supplies signed resource-record discovery, while [Nostr NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) defines signed events. [NIP-05](https://github.com/nostr-protocol/nips/blob/master/05.md) explicitly says a changed name-to-key mapping must not silently replace a followed public key.

Identity integration and message transport are independent choices. A user can prefer a Nostr identifier while reaching the contact over another authenticated transport. An accepted binding can associate identifiers from different systems with one contact, but matching names alone cannot establish that association. The protocol consumes the resulting capability without selecting Matrix, pkarr, Nostr or a new identity system for everyone.

## Introduction, TOFU and trust

An introduction supplies candidate authentication material and enough addressing information to begin communication. It can arrive through a manual exchange, an existing authenticated contact or a discovery mechanism. The record retains how the binding was established.

With **trust on first use (TOFU)**, the application pins the authentication material accepted on first contact. Subsequent sessions must demonstrate continuity with that pin. TOFU does not independently establish that the first endpoint belonged to the person the user intended; it preserves the identity first accepted. Later fingerprint comparison or another independent verification procedure can strengthen the recorded assurance without creating a second contact. TOFU is one supported bootstrap policy; an existing owner-key binding or accepted introduction can provide another.

Authentication status and behavioral trust are separate:

| Dimension | Examples of recorded policy or evidence |
| --- | --- |
| Identity assurance | Discovered but unaccepted; TOFU-pinned; verified through an independent procedure; introduced by a named contact under local policy. |
| Privacy trust | Whether this contact or device may receive information whose disclosure would harm privacy. |
| Liveness trust | Whether the application assumes cooperation for the proposed honest or semi-honest session. |
| Permitted identity updates | Whether a credential or accepted introduction can advertise endpoints, enroll devices, rotate keys or introduce other contacts. |

The existing peer-message draft distinguishes the user's own devices, peers trusted with privacy and liveness, peers trusted only with liveness, and stored-but-unused peer records. These are useful local policies, not a single authentication score. A successful handshake or an introduction does not automatically grant all of them. An introducer's permissions and any trust inherited through an introduction must be explicit; introducing another contact is distinct from announcing another endpoint for oneself.

## Discovery and authenticated endpoint updates

Discovery answers where a peer might be reachable and which capabilities it advertises. Authentication determines whether that information belongs to the established contact. Candidates can come from manual exchange, contact introductions or a chosen directory or rendezvous mechanism. A directory response, familiar display name or address match alone does not authorize merging a candidate into a contact.

An established contact can announce new endpoints over its authenticated contact channel. Alternatively, it can supply an authenticated binding record that can be relayed or retrieved elsewhere. The backend interprets its issuer, permitted actions, device, transport namespace, peer ID or address, required authentication material, capabilities and update context under the chosen scheme. It checks the freshness and replacement information needed by that scheme. Unsupported transport records can be retained as authenticated information and used when local support becomes available.

The desired flow is:

1. Alice authenticates Bob through one supported channel, or records a TOFU pin, and stores the resulting contact binding.
2. Bob advertises another transport endpoint through credentials authorized by that binding. Alice verifies the update and records it under the existing contact and appropriate device.
3. When Alice can use that transport, she connects or exchanges messages with the endpoint and verifies the expected endpoint credentials or an authenticated exchange bound to this contact and channel context.
4. Alice exposes the resulting capability as another route to Bob. The original assurance and trust policy remain attached to the relationship; no fresh human identity check is needed for an authorized update.

For example, a device first reached over Tor can later announce a Nym endpoint or paired mailbox setup over the existing authenticated relationship. The new transport need not carry a universal contact identifier publicly. Where the transport supports proof of endpoint-key possession, that proof is checked; otherwise the profile must specify an equivalent authenticated binding and exchange. Merely copying an advertised peer ID is insufficient.

TOFU continuity extends through these authenticated updates, but its initial uncertainty also extends: learning another endpoint does not upgrade an unverified contact to independently verified. An unexplained key change remains a separate event requiring resolution, even if the previous transport has become unavailable.

## Device enrollment, rotation and recovery

An endpoint update for an existing device and enrollment of a new device are different operations. The identity backend decides whether the supplied evidence authorizes either operation under the owner's configuration and local trust policy. For an owner-key scheme, an owner can authorize device keys and their permitted actions. A web-of-trust scheme can instead express which connected peers' introductions are accepted for which purposes. Fungi consumes the resulting bindings without prescribing a device-enrollment hierarchy or introducing a trusted third party.

Enrollment binds the new device's credentials and permitted actions to the established contact. Knowing one device's ID does not require sharing its private key with every other device. The contact record can retain per-device trust and capabilities, including whether a device may manage other devices or only communicate.

The backend handles replay and conflicts within the chosen scheme and exposes unresolved identity changes to the application. Sequence numbers, epochs or explicit predecessor references are possible record-profile choices, not a required Fungi-wide identity log. A delayed record cannot silently undo an accepted revocation or replace a newer binding. Conflicting authenticated updates remain visible for resolution rather than choosing whichever arrived last.

Routine key rotation can preserve continuity through an authorized transition. Loss of every trusted credential requires a recovery or renewed verification procedure; a replacement key learned solely through TOFU is not evidence of continuity with the old contact. Revocation changes which subsequent actions are accepted under the contact policy. Offline peers may learn it later, so a profile must define how it treats delayed messages and records.

## Revocation and capability lifetime

Revocation is a required lifecycle concern for the contact integration. Its scope must be explicit: withdrawing an endpoint, removing a device, invalidating a credential or withdrawing an introducer's permission are different events. Local removal of a privacy or liveness trust assumption is another reason to stop supplying a capability for a particular role, even if the peer's key remains valid.

The application verifies revocation evidence according to the selected identity scheme, records the changed binding and invalidates affected cached routes and authorizations. A capability already handed to a running protocol must have a defined lifecycle response: its wrapper can stop further use and surface the loss of authorization, or the driver can withdraw and replace it. Merely deleting an address from future discovery results leaves existing channels and cached permissions unaddressed.

Selecting another route must preserve the required authentication and privacy profile. Revoking one device must not silently authorize a newly discovered device, and losing an authenticated route must not silently trigger a weaker identity check. The contact handle can persist while particular devices and endpoints cease to be usable.

An identity profile must describe how revocation is disseminated, what freshness can be established, and how offline or stale views are treated. Matrix, pkarr and Nostr integrations need not have identical mechanisms or guarantees. If current authorization cannot be established, the integration reports that uncertainty and local policy decides whether a capability may be supplied. Asynchronous discovery cannot promise instantaneous revocation knowledge everywhere. Handling already-received messages and an active session's commitments follows the relevant session rules.

## Session and privacy boundaries

Contact management supplies identity continuity for the [long-lived authenticated channel](#long-lived-authenticated-contact-channels) and the [manual introduction flow](#introductions-and-manual-session-initiation). Both can initiate honest or semi-honest sessions, including sessions operating within a broader BFT construction. Authentication of a counterparty does not confer trust in all participants of the enclosing session.

The permissionless alternative is coalition formation with Byzantine strangers over privacy-preserving transports. That path can use its own coin-associated authentication and protocol admission rules without enrolling the strangers as persistent contacts or establishing a web-of-trust relationship. Contact discovery and permissionless coalition discovery therefore share channel machinery while retaining their distinct trust assumptions.

Associating a contact's transport endpoints deliberately links them in the local contact record. Dissemination of that association is therefore a privacy decision. Endpoint updates should reach their intended audience under the chosen confidentiality policy; discovery does not require publishing a global contact-to-device-to-address directory. Pairwise contact credentials and scoped device identifiers are possible profiles.

Anonymous contribution paths retain their own privacy requirements. They must not acquire contact IDs, device IDs or stable endpoint-update tags merely because the same application also manages authenticated contacts. A return capability such as a SURB can support an exchange without enrolling a persistent device or creating a long-lived contact.

## Long-lived authenticated contact channels

The contact-management layer separates contact and device identity from transport peer IDs. It records first-contact verification or TOFU, discovers candidate endpoints and authenticates later endpoint updates, so an established relationship can acquire new transport routes without repeating human authentication.

Trusted contacts need a durable authenticated messaging relationship spanning multiple transaction-construction sessions. A **contact channel**, provisionally `ContactChannel<I, O>`, supplies bidirectional typed messaging bound to an established contact identity. Its lifetime spans transport connections, address changes and periods when either party is offline. It can use Tor streams, paired mailboxes or another delivery mechanism underneath.

An established contact channel provides a way to initiate an honest or semi-honest session with that contact. Manual introduction, described below, provides another. In either case, the resulting session can stand alone or operate within a broader Byzantine fault-tolerant (BFT) session. The inner session's trust assumptions apply to its participants and interactions; they do not require trusting every participant in the enclosing session.

The abstraction separates three kinds of state:

| State | Lifetime and responsibility |
| --- | --- |
| Contact record | Established identity, authentication keys, authorized transport addresses and local trust policy. Persists independently of any channel instance. |
| Authenticated messaging state | Binding to the contact, channel context, active key material and any sequence, replay or delivery bookkeeping required by the chosen profile. Can span multiple construction sessions and transport connections. |
| Transport capability | Current send/receive mechanism, connection or mailbox resources. Can fail or be replaced while the contact relationship persists. |

Establishment binds the remote authentication key to the intended contact through an introduction or verification procedure. The normal trusted-contact profile authenticates both parties and protects message confidentiality and integrity. Authentication establishes who controls the accepted credentials; the local trust policy determines whether that contact is trusted with privacy, liveness or both. Neither property follows merely from successful authentication.

The logical interface provides send and receive operations for that contact, together with lifecycle operations to establish or resume messaging, update authenticated addressing information, rotate keys and close the relationship's channel state. These operations need not perform a fresh handshake on every transport connection or message. Their concrete API and cryptographic protocol remain open.

The contact-channel contract requires:

* **Continuity:** resuming or switching transports preserves the authenticated counterparty binding. Pairing send and receive halves must bind both to the same intended contact and channel context. A newly observed address cannot replace that binding without authentication.
* **Durability:** persistence covers the key and sequence state needed for safe resumption. Reopening a channel must not reuse encryption nonces or mailbox slots. When safe resumption cannot be established, the profile specifies how to establish fresh messaging state.
* **Replay and delivery handling:** messages are bound to their channel context and, where applicable, construction session. Retries, duplicates and delayed messages across reconnects or transport changes have explicit handling; authentication alone supplies neither ordered nor exactly-once application delivery.
* **Key lifecycle:** the profile defines how authorized key rotation preserves contact continuity, how old in-flight messages are handled and when renewed contact verification is required. A long-lived relationship does not require one permanent traffic key; forward secrecy and recovery after compromise are separate profile choices.
* **Availability:** pending messages, retention, acknowledgements and expiry have defined bounds. An offline contact can delay progress without terminating the relationship or implying that a transaction phase has completed.

Messages within this relationship are intentionally attributable to the contact at the receiving endpoint. Transport metadata protection can still hide network addresses and activity from other observers, but contact authentication does not provide message unlinkability relative to that contact. Anonymous transaction contributions therefore use a separate submission capability rather than inheriting the contact channel's identity or session tags.

This extends the contact model in [peer introduction issue #60](https://github.com/fungi-protocol/fungi/issues/60). It also places [long-lived key rotation above the lower-level mailbox delivery abstraction](https://github.com/fungi-protocol/fungi/pull/107#discussion_r4186398006), allowing a finite construction session to use simpler messaging state while a persistent contact relationship manages renewal across sessions.

## Introductions and manual session initiation

[Contact introductions](#introduction-tofu-and-trust) establish identity and trust provenance; a session invitation binds a particular session's context. They can be exchanged together, but retain their distinct scopes.

An **introduction** supplies the information needed to establish communication and verify the intended counterparty or session context. A user can exchange it manually, for example by scanning a QR code, following an invitation link or importing a shared file. An existing authenticated contact channel can carry the same logical information. The representation is independent of the transport used afterward.

Both the established-contact path and the manual-introduction path initiate honest or semi-honest sessions. Such a session may itself be part of a BFT session: for example, payment counterparties can establish an inner session for delegation and confirmation while participating in broader BFT transaction construction. The invitation identifies the inner session's context and, when applicable, its relationship to the enclosing session. Accepting it establishes neither trust in all outer participants nor completion of the outer protocol's agreement conditions.

Introductions have an explicit scope: establish a persistent contact, join one construction session, or both. Joining a session does not automatically save a trusted contact, and saving a contact does not authorize participation in future sessions. Contact introductions carry authentication material and transport addressing information; the user chooses the contact's local trust policy through the introduction procedure.

A session invitation identifies a common construction context and supplies the capabilities needed to reach it:

| Information | Purpose |
| --- | --- |
| Protocol version and session identifier | Distinguish this construction attempt from other attempts and interpret its parameters. |
| Initial construction state and parameters, or their verifiable commitment | Bind the initial PSBT, applicable constraints and any agreed deterministic-shuffling seed. |
| Broadcast namespace and access information | Establish the same logical domain across transports, including any secret topic material. |
| Bootstrap addressing and authentication or reply material | Reach an introducer or initial neighbors and verify the expected binding where authentication is required. |
| Scope and validity conditions | State whether the invitation is session-only, whether it also introduces a contact, and any use or expiry limits. |

The descriptor binds these fields together so an accepted invitation cannot silently substitute a different session, topic or initial state. Encoding, commitment construction and invitation authentication remain profile choices. Manually transferring a descriptor does not itself authenticate its source; verification uses the established contact relationship or the verification procedure of the chosen out-of-band exchange.

Manual initiation proceeds as follows:

1. A user explicitly starts an honest or semi-honest session, either independently or within an enclosing BFT session. Their peer creates the initial context and an invitation for it, including broadcast and bootstrap information.
2. The user shares the invitation with selected peers, directly or through existing contact channels. Each recipient inspects the proposed context and independently accepts, declines or relays it under local policy.
3. An accepting peer verifies the descriptor's binding, establishes the required peer capabilities and joins the advertised broadcast domain. If only a commitment was supplied, it obtains and verifies the initial state before acting on it.
4. The peer enters the construction protocol for that context. Receiving an invitation, opening a channel and subscribing to a topic are distinct from accepting transaction terms or establishing phase agreement. The consuming protocol determines the evidence needed to proceed.

This realizes the manual path in [session initiation issue #61](https://github.com/fungi-protocol/fungi/issues/61): any peer can initiate, while others choose whether to relay or participate. The introducer supplies a starting point; introducing peers does not confer a permanent coordinator role or transfer one peer's trust decisions to another.

Repeated receipt of the same invitation identifies the same attempt rather than starting another construction automatically. A changed initial context requires a distinguishable descriptor and renewed acceptance where the change affects participation. Delayed, canceled and expired invitations need explicit handling in the initiation profile. Session access material is shared only with its intended recipients, and long-lived contact identifiers need not appear in anonymous contributions or public broadcast metadata.

## Payment negotiation across the session lifecycle

Payment negotiation uses the honest or semi-honest relationship established through a contact channel or manual introduction. It begins before transaction construction: counterparties first plan a payment and coordinate how to reach a suitable construction session, then exchange payment confirmations within that session. The construction session can use the BFT protocol while their bilateral negotiation retains its own trust assumptions.

This is also a separation of state: counterparties maintain their private negotiation state, while the construction session synchronizes the shared contributions and proofs needed for its transaction. The [state-sharding discussion](optimistic-track.md#sharded-state-and-synchronization-scope) distinguishes that partitioning from replication and validation within each shared scope.

### Notice and planning

A payer tells the intended payee that it intends to pay them. They can negotiate the amount, timing and other payment terms. Advance notice lets the payee account for the expected incoming payment when planning its own payments, funding needs and participation in transaction construction. The negotiation records an expected payment; its later confirmation refers to a concrete transaction state.

The relationship can therefore outlive any one coalition-formation attempt. A changed payment plan can be communicated before construction starts, and an unsuccessful attempt can lead to another agreed attempt without losing the payment context.

### Coordinating coalition formation

The peers agree which coalition-formation opportunities to pursue together. For example, Alice intends to pay Bob; they agree on a maker, then Alice and Bob each independently submit a co-spend proposal to that maker. Their proposals can be aggregated into a coalition that includes both of them, allowing the payment to be settled within its transaction.

Their private agreement coordinates the target of the attempts. Each peer still evaluates and accepts the actual proposal terms under the coalition-formation protocol. Agreement to approach the same maker does not itself establish a coalition or start transaction construction. Once a coalition is formed and its session commences, the counterparties associate their payment negotiation with that session's context.

This also connects manual initiation with permissionless discovery: the counterparties can establish their honest or semi-honest relationship first, then arrange to participate together in a broader BFT construction session with other peers.

### Confirmation during construction

Once construction is underway, the counterparties exchange payment confirmations tied to the [block DAG's Merkle set commitments](optimistic-track.md#payment-confirmation-commitments). A confirmation identifies the construction instance, phase and relevant committed message sets. These sets cover transaction contributions and applicable proof messages, allowing counterparties to confirm the protocol state on which the payment depends. They establish that the proposed construction satisfies the payment agreement before input signing; they are distinct from confirmation of a transaction on the Bitcoin chain.

The [concurrent PSBT specification](https://github.com/fungi-protocol/docs/blob/concurrent-psbt/psbt-concurrent.md#confirmation-of-successful-payment-prior-to-signing) supplies the payment-graph dependency logic. Here its confirmation target is generalized from a PSBT unique ID to the DAG's committed message sets. Proofs can be committed as protocol messages alongside transaction data without first defining PSBT fields for those proofs. Deriving the same transaction data alone does not establish agreement on those proof messages.

The confirmation dependencies follow the payment graph:

* A receiver whose net balance is non-negative can confirm once the committed state includes all of its intended outputs and the relevant proof obligations have been checked under its accepted validation policy.
* A receiver that sends more than it receives waits for matching confirmations from all of its own payment receivers before confirming to its senders.
* Before signing, every sender obtains confirmation from every one of its receivers for the required committed sets in the same instance and phase, and checks that the confirmed state matches the state it accepts locally.

If relevant transaction contributions or proof messages change the confirmation's committed scope, the counterparties reconcile their views and obtain confirmations for the updated scope before signing. Different DAG frontiers can represent the same confirmed message sets; acknowledgement-only blocks need not change the confirmation target. The exact selection and encoding of the required set commitments remain part of the confirmation profile.

An earlier agreement on a payment or maker is not a substitute for these confirmations. The bilateral exchange remains scoped to the payment and the committed protocol state, while the enclosing construction protocol supplies its own validity and phase-agreement rules.

## Open profile choices

The application integration needs operations to import a candidate introduction, pin or verify an identity, resolve a contact to authorized devices and supported endpoints, authenticate a newly opened route, and apply authenticated updates or revocations. Trust policy and verification provenance persist alongside those records. The shared contact exchange needs its own versioned specification; Fungi receives the resulting capabilities and their lifecycle outcomes.

Backend profiles can choose preferred identifier systems, owner-key schemes and pairwise identity support, introduction verification, device enrollment and recovery, binding-record encoding and freshness, discovery and update dissemination, conflict resolution and persistence across the user's own devices. They must expose the scope and practical limits of revocation and capability invalidation. The protocol leaves those mechanisms abstract while preserving the central rule: an authenticated route can teach us new authorized routes to the same contact without repeating human authentication, while discovery alone cannot change who that contact is.
