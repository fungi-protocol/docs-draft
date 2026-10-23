---
type: Specification
title: Transaction construction
description: Content and state synchronization, the phases, the optimistic and recovery tracks, and the block DAG they share.
status: draft
stale_after: 2027-04-23T00:00:00Z
---

<!-- cspell:words antichain Ehud Idit mathcal mathrm Oded operatorname Ouri Pieter Wuille -->

# 🤖 Transaction construction

Transaction construction starts from an accepted coalition formation proposal and ends with a signed transaction. This chapter describes it in the abstract and defines the block DAG that its tracks share. The chapters that follow describe each track and the validity proofs.

## Content and state

Construction synchronizes two different things:

- **Content**: the transaction-level messages, such as inputs, outputs, fee contributions and their proofs. The [validity proofs](validity-proofs.md) constrain which content is valid.
- **State**: which admitted content is in the agreed state of each phase. State synchronization uses agreement-level messages, the validator blocks. Validity proofs do not cover blocks. Blocks have their own mechanisms, such as the [nonce commitments](#nonce-commitments-and-compact-equivocation-evidence) that expose the key of a validator that equivocates.

Transmitting either between peers is described in [peer set reconciliation](set-reconciliation.md).

## Phases

1. **Kickoff**: the validators acknowledge the coalition formation proposal, which lists the initial inputs.
2. **Auxiliary inputs** are added anonymously.
3. **Destination commitments**: knowing the complete input set, each input commits to its payments and outputs, keeping the balance fair. How output values are chosen is a matter of strategy, not of this protocol.
4. **Outputs** are added anonymously.
5. **Proven auxiliary inputs and proven outputs**, only if the outputs overspend. Missing auxiliary inputs and outputs are posted with proofs.
6. **Signatures** are collected, one per input.

Each phase closes on a stable state before the next one starts. The [optimistic track](optimistic-track.md#phase-closure) says why. The [overview](overview.md) describes the honest, semi-honest and Byzantine settings these phases run in.

## Tracks and validity proofs

- [**Optimistic track**](optimistic-track.md), with threshold $t = n$. It is asynchronous, and phases advance by unanimous acknowledgement. On its own, it is sufficient in the honest and semi-honest settings.
- [**Recovery track**](recovery-track.md), with $t \lt n$. It is optional, and assumes synchrony or partial synchrony. It turns a stalled phase into a recovery coalition formation proposal.
- [**Validity proofs**](validity-proofs.md) are needed only once overspending is detected. They reduce the validity of transaction-level messages to omission faults: a participant can still withhold a message, but cannot get an invalid one accepted.

## Block DAG

Both tracks use the block DAG defined here.

### Objects

A construction instance identifies a fixed validator set and its permitted inputs. A phase identifies the interpretation and admission rules for contributions within that instance. Recovery proposals can establish successor instances with different membership.

A **contribution** is an application message, such as a transaction fragment, proof, acknowledgement or coalition proposal. Anonymous submission prevents a validator's inclusion of a contribution from identifying its author.

A **block** is a product type

$$
b=(M_b, B_b, s_b, R_b^{\mathrm{opt}}, R_b^{\mathrm{rec}}, \kappa_b, \phi_b, E_b),
$$

with the following components:

| Component | Meaning |
| --- | --- |
| $M_b$ | Set of included messages; may be empty for an acknowledgement block. |
| $B_b$ | Set of parent blocks. |
| $s_b$ | Optional self-parent block, from the same validator. |
| $R_b^{\mathrm{opt}}$ | Nonce point committed for signing the validator's next optimistic block. |
| $R_b^{\mathrm{rec}}$ | Nonce point committed for signing the validator's next recovery block. |
| $\kappa_b$ | Block type: optimistic ($t=n$) or pessimistic recovery ($t<n$). |
| $\phi_b$ | Phase within that type's protocol. |
| $E_b$ | Optional extensions encoded as type-length-value (TLV) fields. |

Type and phase may be implicit in the protocol context rather than separately encoded. The optional self-parent is a distinct component from the parent set. Causal ancestry follows both the parent-set references and the self-parent, when present. The block is authenticated by its validator; the signature encoding remains to be specified. Every validator block commits separate nonce points for its next optimistic and recovery signatures.

Optimistic blocks coordinate transaction construction. Recovery blocks coordinate fault-tolerant, permissioned coalition formation so that a new transaction construction instance can start. The recovery type describes the fault model being accommodated, not evidence that a particular participant is faulty.

### Optional block extensions

Blocks carry optional protocol extensions as type-length-value (TLV) fields. A reader can skip an unknown field using its length. Ignoring an extension leaves the core block interpretation unchanged; an extension cannot silently redefine parent references, messages, thresholds or phase-transition rules. Extensions can supply additional evidence that a peer elects to use under its own validation policy.

One extension carries a trusted execution environment (TEE) attestation. It binds the claimed validation result to the relevant block contents, parent-set commitment, instance and phase, and identifies the executed validation code. A peer may rely on that evidence under its hardware-vendor and attestation trust assumptions, while another peer ignores it and validates independently. The attested statement must exclude the attestation bytes themselves to avoid a circular commitment. TLV identifiers, canonical encoding and the precise attested statement remain to be specified.

### Nonce commitments and compact equivocation evidence

Let $P_i=x_iG$ be validator $i$'s public signing key in a group of prime order $q$. Each block commits two independently generated nonce points, $R_b^{\mathrm{opt}}$ and $R_b^{\mathrm{rec}}$. A successor signature must use the point committed for its track in the preceding signing block. The two commitments allow an optimistic successor and a recovery successor without treating that permitted branching as equivocation. Each individual commitment may authorize only one distinct signed block.

For a Schnorr signature with committed nonce $R=rG$, response $z$ and challenge $e$, verification checks

$$
zG=R+eP_i,\qquad z=r+ex_i\pmod q.
$$

The challenge binds the public key, nonce point and complete signed block context, including the instance, track and predecessor commitment.[^schnorr] If the validator signs two distinct blocks using the same committed nonce, yielding distinct challenges $e_1\neq e_2$, their responses reveal

$$
x_i=(z_1-z_2)(e_1-e_2)^{-1}\pmod q.
$$

Anyone who obtains both signatures can extract this scalar. Other peers need only receive $x_i$ and check $x_iG=P_i$; they do not need to obtain the equivocated blocks. The validator private key therefore serves as compact, independently verifiable equivocation evidence. Repeating an identical signature is not equivocation, and using the two distinct track commitments as intended does not expose the key.

Strictly, the scalar proves that the validator key is exposed: deliberate disclosure or compromise produces the same evidence. The protocol treats this as sufficient to disqualify the key without requiring attribution to a particular pair of conflicting blocks. Once exposed, signatures under that key cannot establish who produced subsequent messages. Validator keys must be separate from Bitcoin spending keys and other secrets whose exposure would violate theft resistance or privacy.

The validator key is the online key endorsed by the coin owner's BIP-322 proof during coalition formation. The revealed scalar, together with that endorsement or a reference to it, forms a coin-associated fraud proof. A recipient checks both $x_iG=P_i$ and the endorsement binding $P_i$ to the coin. The recipient need not obtain the conflicting blocks or participate in the construction instance where the key was exposed.

This fraud proof can be gossiped through the permissionless coalition-formation order book using the same peer reconciliation mechanism as advertisements and proposals. Peers that validate it can ban the endorsed coin from subsequent coalition formation. Replacing the online key does not remove evidence against that coin. This is a protocol admission decision, not a restriction on spending the coin under Bitcoin consensus. The endorsement scope determines which coin or coins the evidence applies to.

The encoding must unambiguously identify the predecessor nonce commitment consumed by each signature, including the first signature on each track and transitions to successor instances. Nonce normalization and challenge construction must preserve the extraction equation. These details remain part of the signature profile.

### Genesis

The coalition formation proposal serves as the instance's genesis, denoted by $g$. In the honest and semi-honest settings, an equivalent initiation message serves the same role. It establishes the construction context, including the initial inputs and validator membership $N$, with $n=|N|$. Genesis is the common root within this instance; its representation need not be identical to an ordinary validator block. A recovery coalition proposal is also a genesis, but has parent references to blocks in the preceding session's recovery DAG. It begins a new construction context without discarding that causal ancestry.

### Optimistic and recovery tracks

Genesis and the two acknowledgement layers of [kickoff](optimistic-track.md#kickoff) have instance-local heights 0, 1 and 2. From height 3 onward, progress has two branches: the optimistic or unanimous branch with $t=n$, and a pessimistic recovery branch with $t<n$ and improved liveness under its chosen recovery assumptions. This does not require producing two blocks for every step. Until a reduced-threshold block is published, the recovery overlay contains no blocks; publication of such blocks activates it. The recovery track is described in the [recovery track](recovery-track.md) chapter.

### Causal order and block properties

Write $a\preceq b$ when $a=b$ or $a$ is reachable from $b$ through parent-set or self-parent references. Define

$$
D(b)=\{a:a\preceq b\},\qquad
C(b)=\bigcup_{a\in D(b)}M_a.
$$

$D(b)$ is the block's inclusive causal history. For independent validation, a block is causally delivered only after its required contents and ancestors have been obtained and validated. Receiving a root or block hash is not causal delivery. The [optional attested-validation profile](optimistic-track.md#optional-attested-validation-and-acknowledgement-relay) permits peers to accept certified validation of that history without independently retrieving and checking all of it; this does not assert that they locally possess the history. The following properties are derived from authenticated block data and its causal history:

| Property | Definition |
| --- | --- |
| Author $\operatorname{author}(b)$ | Validator authenticating the block. |
| Instance-local height $h(b)$ | Zero for this instance's genesis; otherwise one plus the maximum height of its parent-set and self-parent references within the instance. Cross-instance recovery ancestry remains part of causal history. |
| Context | Construction instance, block type $\kappa_b$ and phase $\phi_b$, whether explicit or inferred. |
| Observed messages $C(b)$ | Union of included messages throughout $D(b)$. |
| Phase state $V_\phi(b)$ | Application interpretation of the messages for phase $\phi$ and the construction instance in $D(b)$. Ancestors from predecessor instances provide recovery evidence, not new contributions to this instance. |
| Support | Distinct validators whose qualifying acknowledgements occur in $D(b)$, for a specified state and predicate. |

Height measures causal depth; it is not elapsed time or by itself a threshold logical clock. Support counts validators, not blocks. Producing additional blocks cannot multiply a validator's weight.

The analogue of Cordial Miners' $\tau$ is a deterministic interpretation function

$$
\mathcal I(b)=\mathcal I(D(b)),
$$

which derives the phase state, supporting evidence and permitted transitions. Cordial Miners uses $\tau$ to extract a total order while excluding equivocations.[^cordial] Here the interpretation is set-valued: compatible admitted contributions are joined without assigning them a total order. The phase rules determine how invalid or conflicting evidence affects admission. Observation of a message and acceptance into application state remain distinct.

For the compatible admitted contributions of a phase, extending causal history preserves inclusion. Block authentication identifies the validator, not the authors of anonymous payloads. Incompatible self-parent histories by the same validator within the same branch provide equivocation evidence. Explicitly separate recovery branches are different contexts, not automatically equivocations. Evidence is retained rather than resolved by choosing whichever block arrived first.

### Frontiers

A finite causally closed block set $D$ has frontier

$$
F(D)=\max_{\preceq}D.
$$

The frontier is the antichain of its maximal blocks. It represents the entire causal cut because

$$
D=\bigcup_{b\in F(D)}D(b).
$$

For a frontier $F$, write $D(F)=\bigcup_{b\in F}D(b)$ and $C(F)=\bigcup_{b\in F}C(b)$. Frontier extension is ordered by $F\sqsubseteq G$ iff $D(F)\subseteq D(G)$. The join of two compatible causal views is represented by $F(D(F)\cup D(G))$.

A frontier is not a selection of one block per validator. It can omit a validator whose blocks are ancestors of another frontier block, or contain several incomparable blocks by an equivocating validator. A local current frontier represents all causally delivered blocks in its scope. A historical frontier represents a specific earlier cut and must be identified as such.

Different frontiers may contain exactly the same phase contributions. Phase agreement therefore compares interpreted phase states, not equality of block hashes or frontier sets. Acknowledgement-only blocks can advance the frontier without changing the transaction state.

## References

[^schnorr]: Pieter Wuille, Jonas Nick and Tim Ruffing. [BIP 340: Schnorr Signatures for secp256k1](https://bips.dev/340/). Reference for the Schnorr verification equation and key-prefixed challenges; the predecessor nonce commitments and equivocation-evidence policy are part of this protocol.

[^cordial]: Idit Keidar, Oded Naor, Ouri Poupko and Ehud Shapiro. [Cordial Miners: Fast and Efficient Consensus for Every Eventuality](https://arxiv.org/abs/2205.09174).
