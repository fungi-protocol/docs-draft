---
okf_version: "0.2"
---

* [Collaborative transaction privacy](collaborative_txn_privacy.md) - Under review in #7.

# Protocol Suite

* [Overview](overview.md) - Collaborative transaction construction, privacy, fault tolerance and coalition formation.
* [🤖 Abstract transport interfaces](transport-interfaces.md) - Send and receive channels, privacy tags, and message unlinkable broadcast by gossip.
* [🤖 Peer and broadcast channels](channels.md) - Generic message interfaces, transport interactivity, privacy profiles and anonymous replies.
* [🤖 Peer set reconciliation](set-reconciliation.md) - Naive reconciliation, quiescence before block authoring, sparse Merkle sets, mutual knowledge, and range reconciliation and rateless streams as future optimizations.
* [🤖 Contacts, devices and discovery](contacts.md) - Introduction, TOFU, trust policy, revocation, application interoperability across transports, contact channels and payment negotiation.
* [Market-based coalition formation](coalition_formation.md) - Under review in #18.
* [🤖 Transaction construction](transaction-construction.md) - Content and state synchronization, the phases, the optimistic and recovery tracks, and the block DAG they share.
  * [🤖 Optimistic track: blocks and phase states](optimistic-track.md) - Unanimity and phase closure on the optimistic track, phase by phase.

# Privacy Optimization Strategy

* [Cost function framework for Bitcoin wallets](optimization_framework.md) - Under review in #15.

# BIP Drafts

* [PSBT Scrubber](bip-drafts/scrubber.md) - Which PSBT fields can be shared with trusted and untrusted parties such that privacy is preserved.
* [Concurrent PSBT Construction](psbt-concurrent.md) - Under review in #16.
