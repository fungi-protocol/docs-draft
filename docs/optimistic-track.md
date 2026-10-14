---
type: Specification
title: "Optimistic track: blocks and phase states"
description: Unanimity and phase closure on the optimistic track, phase by phase.
status: draft
stale_after: 2027-04-15T00:00:00Z
---

<!-- cspell:words antichain APSA APSA's Chondros Christos counterpropose CRBB Croman Diamantopoulos Dolev Duan Ehud Garay Guerraoui Haibin Haifeng Idit Ittai Kichidis Maneas mathcal mathrm Mema Minghang Nikos Oded operatorname Ouri Patsonakis Pieter Rachid Reiter Roussopoulos Ruomu semilattice Sisi Skoumios Stathis Tasos Tonkikh Vassilis Wonseok Wuille Yuval Zhuolun Zikas -->

# 🤖 Optimistic track: blocks and phase states

<!-- cspell:words Baudet Brugeres Danezis Kelkar Kuznetsov Languille Mahimna Maxence paxpay precommitted Sonnino Zarfaoui -->

## Scope and status

This specification describes how Fungi transaction construction obtains consistency on the optimistic track, with threshold $t=n$. The [recovery track](recovery-track.md), with $t<n$, is described separately. The block DAG it uses is defined in [transaction construction](transaction-construction.md#block-dag). Transmitting that history between peers is described in the [peer set reconciliation](set-reconciliation.md) chapter.

This is a draft technical specification. The data model and reconciliation rules below describe the intended construction. Exact encodings, cryptographic parameters, quiescence scheduling and phase-specific agreement predicates remain to be specified.

The construction retains the causal delivery and validation semantics of Byzantine Eventual Consistency (BEC).[^bec] Discovering missing data only needs BEC's [naive reconciliation](set-reconciliation.md#naive-reconciliation); more efficient algorithms are future optimizations. It does not introduce a total order on transaction fragments. Compatible fragments combine by set union; arbitrary PSBT updates are not assumed to form a join semilattice. The transaction protocol determines admission, compatibility and the interpretation of its state.

## Design objectives

The unit of progress is a completed transaction-construction phase, not a newly produced block. Unlike a blockchain processing a continuing transaction stream, this protocol coordinates a finite set of contributions toward a stable phase boundary. Its principal performance objectives are end-to-end phase latency, lightweight validation and low communication cost. Faster block production has no independent value.

For a given phase latency and agreement guarantee, fewer blocks are preferable. They reduce authentication, validation, storage and dissemination overhead, and expose fewer intermediate causal views from which an observer could infer propagation order or network topology. Block count is therefore both a resource metric and a privacy consideration, although it is not itself a measure of privacy leakage. Batching must be evaluated against the delay it adds to completing the phase.

The relevant latency measurements are time to reconciliation quiescence, time to a stable common phase state, and time until the evidence permits the next phase. The optimistic target is to reach stability in approximately two communication round trips once the phase's contributions and local prerequisites are available. This is a design target, not an established bound: the precise exchanges, starting condition and completion predicate remain to be derived from the phase rules. DAG depth alone does not count round trips, and communication rounds have no fixed wall-clock duration under asynchrony.

Evaluation therefore measures blocks and bytes per completed phase, per-peer validation work, and the communication needed to establish and disseminate closing evidence. Block throughput and time to the next block do not substitute for these measurements.

## Sharded state and synchronization scope

State sharding partitions the state and work touched by operations so unrelated operations can progress independently. Partitioning and replication are separate dimensions: the state of one shard may still be replicated across the authorities responsible for validating it.

Zef provides a concrete example. Within each authority, account identifiers determine which shard handles a request; an operation affecting another account can issue an asynchronous cross-shard query.[^zef] PaxPay likewise proposes routing requests by ranges of spent-coin serial numbers to shards within each validator; its published evaluation does not implement that sharding.[^paxpay] The relevant connection is the scope of state access and synchronization, rather than a requirement to replicate one globally ordered state machine.

Fungi's finite construction uses a precommitted flow that can bind inputs to their intended destinations. This makes it possible to enforce the agreed allocation within the construction instance, avoiding the need to adopt these papers' continuing coin lifecycle and spent-serial-number machinery for that purpose. The destination binding must preserve the protocol's unlinkability requirements; it is not a public input-to-output mapping. Exact proof encodings and the efficiency gain remain to be evaluated for the chosen construction.

The sharding comparison also does not select anonymous credential issuance as the validation mechanism. With roughly 500 validators, issuance and its associated proof work may impose an impractical cost on each validator, depending on the scheme, threshold and number of contributions. This is a feasibility concern requiring measurements, not an established performance result. Evaluation must distinguish per-validator work, aggregate work and end-to-end latency, and compare credential issuance against the precommitted input-to-destination flow.

For Fungi, contact state and bilateral payment negotiations are scoped to their counterparties, while construction instances synchronize the shared contributions and evidence needed for their own transaction. An honest or semi-honest payment session can therefore maintain private state within a broader BFT construction session. Its payment confirmations connect that local relationship to the relevant DAG commitments, including proof messages; the private negotiation transcript need not become shared validator state.

Separate construction instances need not synchronize unrelated state. Within an instance, the dependencies of its admitted contributions and phase rules still determine what must be shared and agreed. A Merkle root binds a set but does not establish possession or validation of its contents. Further sharding of storage or validation within one instance requires an explicit assignment, availability and verification protocol; the current DAG interface leaves that mechanism open.

## Unanimity

### State acknowledgement and unanimity

The following is a candidate agreement interface for closing a phase; [phase closure](#phase-closure) gives the concrete rules. Let $N$ be the fixed validator set for the instance. Let $S$ be a candidate phase state and $\operatorname{Ready}_\phi(i,S)$ the phase-specific condition under which validator $i$ can close the phase at $S$.

A block $a$ acknowledges closure at $(\phi,S)$ only if its interpretation establishes both

$$
V_\phi(a)=S
\quad\text{and}\quad
\operatorname{Ready}_\phi(\operatorname{author}(a),S).
$$

The acknowledgement can be implicit in block type and phase; it need not be an additional payload message. Mere inclusion of $S$ in a larger state, or receipt of a block from every validator, does not establish this acknowledgement. The readiness predicate expresses completion of that validator's obligations and acceptance of closing admission at $S$. It must be defined separately for each phase.

For a causal cut $D$, define

$$
A_\phi(S,D)=\{i\in N:\exists a\in D,\ 
\operatorname{author}(a)=i\ \land\ a\text{ acknowledges closure at }(\phi,S)\}.
$$

Then

$$
\operatorname{Unanimous}_\phi(S,D)
\iff A_\phi(S,D)=N.
$$

This is unanimity on the same interpreted state within the same instance and phase. The witnessing blocks need not have equal height or identical parents. A verifier checks the complete causal evidence rather than substituting an arbitrary collection of historical blocks for its current view.

A closing acknowledgement commits an honest validator not to acknowledge a different closing state for that instance and phase. Before issuing it, the validator must have included its required contributions and checked the phase's readiness conditions. This commitment is a proposed rule for making closure well-defined; it is stronger than ordinary acknowledgement by inclusion. Ordinary cordial blocks can continue to extend the view without claiming phase closure. The readiness rules must prevent honest validators from prematurely closing at different states; otherwise the commitment preserves consistency but can stall the branch. Defining those rules is part of the phase protocol, not a consequence of the certificate definition.

Unanimity is evidence available at a causal cut. It does not imply that every validator has already received that evidence. If $D\subseteq D'$, an existing certificate remains present in $D'$, although further evidence may expose a validator's misbehavior. Conflicting unanimous certificates require every honest validator in the instance to have violated the closing-acknowledgement rule; with at least one honest validator, that rule excludes them. This is a consistency argument, not a liveness guarantee.

### Phase transitions

Every phase closes as described in [phase closure](#phase-closure): each validator echoes once the phase's condition holds in its view, and the phase closes when all $n$ validators acknowledge the same $n$ echoes. The condition for each phase is given under [phases](#phases). Validation of an individual contribution does not itself establish a phase's completion.

A participant advances when the relevant evidence becomes available; others catch up through causal delivery. Local phase changes need not be simultaneous.

The exact successor phase is a deterministic function of the completed state and the transaction protocol's rules. A late message cannot silently amend an already closed phase: its treatment must follow those rules or belong to a new attempt. A claimed transition without the required evidence is not accepted as a valid transition.

No phase transition can infer readiness solely from local quiescence. Conversely, obtaining a closing certificate does not require proving that no further network traffic exists. The finite agreement is on $S$ and its readiness conditions.

### Payment confirmation commitments

Payment confirmations bind the relevant Merkle message-set commitments in this DAG, together with their construction instance and phase. The committed messages include transaction contributions and the proofs required to validate them. Proof objects can be carried as content-addressed protocol messages; committing them does not require defining dedicated PSBT fields. Two views that yield identical transaction data can still differ in their committed proof messages.

The [payment confirmation exchange](contacts.md#confirmation-during-construction) uses these commitments as its target while retaining the concurrent PSBT protocol's payment-graph dependencies. The confirmation profile must specify which committed sets are being confirmed and how they determine the accepted payment state. Equality of entire DAG frontiers is not required: acknowledgement-only blocks can advance a frontier without changing those sets. A commitment binds contents; validation and the phase's completion evidence remain separate obligations.

## Phase closure

Every phase ends with two blocks per validator, on the optimistic track with $t = n$. A stall is then always a missing per-input message, or a missing block from a named validator, which the [recovery track](recovery-track.md) handles.

Closing every phase with an echo and a closing acknowledgement keeps each phase's state stable for the next. This matters most for the input set, which must be stable before outputs:

- **Validity proofs and their privacy.** Proving an output before a late input arrives would need a more complicated proof statement and would harm privacy, so it is not supported.
- **The balance condition.** With only some of the inputs and some of the outputs, the two could sum to the same amount, and the balance would reach zero too early.
- **Choosing output values.** Most importantly, output values are chosen on the basis of the input set, including the auxiliary inputs. This is a privacy requirement.

### Hashes and identifiers

- $H(\mathit{tag}, m) = \mathrm{SHA256}(\mathrm{SHA256}(\mathit{tag}) \Vert \mathrm{SHA256}(\mathit{tag}) \Vert m)$, the tagged hash of BIP 340.[^schnorr] Tags begin with `fungi/`.
- Instance id: $\mathit{sid} = H(\texttt{fungi/session}, \mathit{genesis})$.
- Outpoint: $\mathit{op}$ = txid (32 bytes, internal byte order) followed by vout (u32, little-endian).
- Validator: its index in $N$, in genesis order.

### Input lists

The initial inputs $I$ are listed in the genesis, and each owner signs with BIP-322. The auxiliary inputs $A$ are admitted anonymously in the auxiliary input phase.

The per-input phases have the lists below. Each is sorted by outpoint bytes, ascending.

| Phase | List $L_\phi$ |
| --- | --- |
| kickoff | the initial inputs $I$ |
| destination commitments | $I \cup A$, with $A$ the admitted auxiliary inputs |
| signatures | the inputs of the final unsigned transaction |

The auxiliary input, output and proven phases are anonymous. They have no list.

### Per-input messages

A per-input message is signed by, or bound to, one input. These are the only messages that can blame an input.

| Phase | Message | Valid if |
| --- | --- | --- |
| kickoff | $\mathrm{ACCEPT}(i)$ | BIP-322 signature by $i$ over $H(\texttt{fungi/accept}, \mathit{sid})$ |
| destination commitments | input proof of $i$ | verifies under the validity proofs, bound to $\mathit{sid}$ and $\mathit{op}_i$ |
| signatures | signature of $i$ | a valid witness for $i$ over the final unsigned transaction |

An invalid message counts as absent.

### Slots

Each pair $(\phi, i)$ with $i \in L_\phi$ is a slot. The first valid per-input message a validator holds for it fills the slot, and the slot stays filled.

Presence must never be withdrawn. Otherwise a faulty input can send a second, conflicting message to some honest validators, so that they never close the phase while others have, and the honest validators get blamed for it.

- Two valid input proofs of $i$ for the same $\mathit{sid}$ with different destination commitments are **per-input equivocation**. Both are bound to $i$, so the pair is evidence against $i$ that anyone can check.
- Two different valid signatures over the same transaction, or two valid ACCEPT messages, are not equivocation. Either one fills the slot.
- As the [overview](overview.md#consistency-liveness-and-termination-in-the-presence-of-byzantine-faults) requires, conflicting contributions are both removed from the phase state. [Recovery](recovery-track.md#evidence-that-excludes-at-once) additionally excludes $i$.

### Block kinds

Each block has a kind: ordinary, echo, closing acknowledgement, or recovery track block. Its payload holds the messages not yet in its causal history, except the author's own anonymous contributions. Including those could link them to the input that endorses the author's validator key, so a validator relies on other validators to include them.

Echoes and closing acknowledgements belong to the optimistic track. Recovery track blocks belong to the [recovery track](recovery-track.md#recovery-track-blocks) and are signed with the recovery nonce commitment. So a recovery block never counts as equivocation against the same validator's optimistic blocks.

At most one echo and one closing acknowledgement per validator per $(\mathit{sid}, \phi)$, and one recovery track block per $(\mathit{sid}, \phi, r)$. A second one reuses a committed nonce and exposes the key.

### Entry

- A validator enters phase $\phi$ when it holds closing acknowledgements for $\phi - 1$ from all of $N$, and all $n$ of them name the same $n$ echoes, one per validator. Kickoff is entered on holding the genesis.
- If two acknowledgements name different echoes by the same validator $w$, then $w$ signed two echoes. That exposes $w$'s key. The validator does not enter $\phi$, and an evidence-only [recovery proposal](recovery-track.md#recovery-proposal) excludes $w$.

### Echo

A validator publishes its echo for phase $\phi$ once that phase's condition holds in its view. The conditions are given with each of the [phases](#phases). The echo's payload holds every phase message the validator holds that is not yet in its causal history, apart from its own anonymous contributions.

### Phase state

The phase state $S_\phi$ is the set of admitted phase contributions in $\bigcup_{w \in N} C(e_w(\phi))$, the union below the $n$ echoes. Every validator that enters $\phi + 1$ holds the same $n$ echoes, by the entry rule, so all compute the same $S_\phi$. A message that arrives after an echo enters $S_\phi$ only if another echo has it in its history.

### Closing acknowledgement

A validator publishes its closing acknowledgement once it holds and has validated all $n$ echoes. Its parents include all $n$.

## Phases

Each phase below admits its content, then closes by the rules above. The phases run in the order given in [transaction construction](transaction-construction.md#phases).

### Kickoff

Kickoff is the validators' acknowledgement of the coalition formation proposal, the genesis. Its slots are the initial inputs, each filled by an $\mathrm{ACCEPT}$ message, and a validator echoes once every slot is filled in its view. Kickoff consists of two rounds of validator acknowledgements:

1. Each validator $i\in N$ publishes an echo block $e_i$ acknowledging genesis. Its only parent is $g$; it has no self-parent. These echoes are independent and can be produced concurrently.
2. After obtaining and validating echoes from all $n$ validators for the same genesis, each validator publishes a second-round acknowledgement $a_i$. Its causal history includes those echoes, including its own $e_i$. The threshold is $t=n$.

Once a peer has obtained and validated the second-round acknowledgements from all $n$ validators, kickoff is complete for that peer. It can begin anonymously disseminating auxiliary inputs and wait for the auxiliary-input phase to conclude. The second round establishes that every validator has acknowledged the complete first-round echo set; merely receiving all first-round echoes does not complete kickoff.

These rounds are causal exchanges, not wall-clock intervals. Peers enter construction as the required evidence becomes available locally. Kickoff opens the auxiliary-input phase; it does not establish that this phase's contributions are complete. A missing acknowledgement can stall kickoff. The [recovery overlay](recovery-track.md#recovery-overlay) begins after the genesis acknowledgement layers; this rule does not itself provide a recovery mechanism for incomplete kickoff.

### Auxiliary inputs

Auxiliary inputs are admitted anonymously. The phase closes when all validators acknowledge the same set of them.

A validator echoes once its own auxiliary inputs, if any, are in its view, included by another validator. A validator whose own auxiliary inputs are not included eventually echoes without them; they are then not part of this transaction.

### Destination commitments

Knowing the complete input set $I \cup A$, each input posts its input proof, which commits to its payments and outputs. A validator echoes once every slot of $I \cup A$ is filled in its view.

### Outputs

Outputs and fee contributions are admitted anonymously. The balance in a validator's view is the total of $I \cup A$ minus the outputs and fee contributions in that view.

A validator echoes once its own outputs are in its view, included by other validators, and the balance is zero or negative. The censorship exception below is the only other case.

- **Balanced.** At a balance of zero with closing acknowledgements from all $n$ validators, every validator's outputs are present. Honest outputs cannot be told apart from others, but none is missing, so all outputs are honest. The phase continues to signatures.
- **Overspent.** If the balance is negative, at least one output or fee contribution is not covered. The phase continues with the proven phases, as the [validity proofs](validity-proofs.md) require. In the Byzantine setting anyone can force this. For example, a faulty party posts an output for exactly the remaining amount before an honest output arrives; the honest output then makes the balance negative.
- **Underspent.** Only the censorship exception closes the phase with a positive balance. The phase continues to signatures. Honest participants whose outputs are present still sign. An owner whose outputs are missing does not, and its missing signature names its input.

A participant that runs no validator has no echo of its own to hold back, so the balanced case covers only participants that run one. See the recovery track's [open questions](recovery-track.md#open-questions).

**Censorship exception.** A validator's anonymous outputs may not get included, even though its peers accept them. This may indicate an $n-1$ Sybil deanonymization attack. Including the outputs in its own block could link them to the input that endorses its validator key. So a validator that suspects such an attack eventually echoes with some of its outputs missing. In the signatures phase it then withholds the signature of one input: the one whose missing signature is least damaging to it.

### Proven auxiliary inputs and proven outputs

These phases run only after an overspent outputs phase. Auxiliary inputs and outputs are posted again with proofs, including any that were missing. A validator echoes as in the auxiliary input and output phases. As the [validity proofs](validity-proofs.md) describe, the proofs that depend on the repeated auxiliary inputs are then made again.

### Signatures

Each input of the final unsigned transaction posts its signature. A validator echoes once every slot is filled in its view. Once every validator holds every signature, the transaction is complete.

## Dissemination interface

A peer that obtains a block whose parents or included messages it does not have requests their preimages by hash from the peer that sent it. It repeats this for each obtained block until it reaches blocks it already has. This is the [naive reconciliation](set-reconciliation.md#naive-reconciliation) of BEC, and it is sufficient: causal delivery only needs the preimages of the hashes a block refers to.

The [peer set reconciliation](set-reconciliation.md) chapter also describes future optimizations, which synchronize sparse Merkle sets using mutual and asymmetric knowledge to avoid redundant transfers. The DAG supplies the parent and message sets committed by blocks, plus causal evidence of which contents a neighbor has acknowledged. Reconciliation supplies obtained contents and completion status for a specified scope. It does not decide block validity, unanimity or phase transitions.

Beluga formalizes a modular block synchronizer for incremental retrieval and resource-aware exchange, including protection against redundant pulls.[^beluga] It is a reference for the boundary between dissemination and consensus, and for evaluating recovery traffic; the reconciliation mechanism here is a separate design.

## Optional attested validation and acknowledgement relay

A validator with a TEE can perform validation and relay the other validators' acknowledgements. Peers choosing this profile retain their own validator keys and produce their own blocks. Their acknowledgement blocks may have empty message sets, $M_b=\varnothing$: their authenticated parent references supply the acknowledgement without repeating the underlying contributions.

The attested validator supplies the parent set needed for a peer to issue its acknowledgement of the preceding block. After checking the attestation and its context, the peer can use those parent commitments in its own acknowledgement block, retaining its self-parent and nonce commitments. Thus constrained peers need not reconstruct the parent set by independently processing every relayed contribution and acknowledgement.

The exchange is interleaved:

1. A TEE-backed validator publishes an attested block containing the parent-set evidence needed to acknowledge the preceding block.
2. Other validators issue their individual acknowledgement blocks using that evidence; their message sets can be empty.
3. The next TEE-backed block references the preceding TEE-backed block and the other validators' acknowledgements, relaying them and supplying the evidence for the next acknowledgements.

The validator relays acknowledgements rather than signing on behalf of their authors. Support still counts the distinct validators whose acknowledgements are represented. Peers can independently validate the same DAG, and multiple validators may provide attestations; the profile does not create a privileged protocol leader.

The attestation can certify validation and satisfaction of the applicable agreement predicate. It does not authorize a change to that predicate: unanimous phases still require all required acknowledgements, and transaction signing still requires the input owners. Private allocation information and spending keys are not supplied to the TEE. Peers retain local checks needed to bind their own contributions, proofs and signatures to the intended instance and privacy context. The optional trust assumption reduces validation work and can support stronger liveness under the selected recovery policy.

## Block publication and quiescence

Morpheus adapts between quiescent low-throughput operation and a high-throughput DAG protocol.[^morpheus] It motivates attention to idle traffic and workload-dependent behavior. Its quiescence property is distinct from the reconciliation-before-publication policy defined here.

Reconciliation between neighbors precedes the release of a new cordial block. The objective is to publish a consolidated view rather than a sequence of intermediate views that exposes the order in which contributions reached the validator. This reduces observable correlations between the block contents and the network's propagation topology.

For a fixed reconciliation scope, local quiescence means that its advertised sets have been reconciled and there is no outstanding reconciliation work for that scope. It is a local condition: it neither proves that no further message exists nor closes admission to a phase. A newly advertised root can create more work.

Waiting for neighbors' Merkle sets to quiesce applies even when the parent threshold is already met; see [quiescence before block authoring](set-reconciliation.md#quiescence-before-block-authoring).

A publication rule must specify which neighbors and advertised roots belong to the scope, how concurrent advertisements affect it, and how stalled neighbors are handled. Those choices are not yet fixed. Waiting for all future advertisements is not a finite condition; waiting for every neighbor can indefinitely delay publication. Any cutoff or recovery policy must be evaluated together with the intended privacy and liveness properties.

Cordial parent-inclusion rules additionally limit block production according to progress by other validators.[^cordial] Quiescence batches the available contributions before a permitted block is released; it does not replace the parent threshold. On a hub-and-spoke topology, a well-connected validator can assemble most payloads while others acknowledge them. This is workload concentration, without requiring a privileged leader.

The privacy objective is to avoid additional attribution beyond the transaction, authorized counterparty knowledge and the transport's guarantees. Channel-randomized transfer schedules reduce redundant traffic; quiescent publication reduces intermediate public views. Neither alone establishes a topology-hiding theorem. The remaining leakage includes publication timing, neighbor failures and the selected quiescence policy.

## Phase agreement

Dissemination makes causal contents available. A phase-specific predicate determines when the available DAG evidence establishes agreement and permits the next phase. Reconciliation completion, quiescence and phase completion are distinct events.

For compatible admitted contributions, the desired completed state is their join, represented by set union. Inclusion protects honest participants' ability and incentive to complete: an agreement cannot discard an acknowledged required output or signature. The exact admission boundary and the evidence establishing completeness are determined by the transaction phase.

Prefix Consensus formalizes compatible lower and upper outputs; Strong Prefix Consensus additionally requires agreement on the upper output.[^prefix] A prospective set-based analogue uses lower and upper states $L_i$ and $U_i$, with

$$
L_i\subseteq U_j,\qquad U_i=U_j
$$

for honest peers in the same instance. Requiring each honest peer's admitted contribution set $X_i$ to satisfy $X_i\subseteq U_i$ additionally implies $\bigcup_iX_i\subseteq U$. Equality of lower and upper states can express a finalized boundary only when their definitions include the phase's admission and completeness conditions. This draft calls the objective **agreement on a common upper state**; it does not claim a new consensus construction from these equations alone.

The optimistic branch uses $t=n$ and remains asynchronous. Every phase closes by unanimous acknowledgement of the same phase state. Cooperative participants can advance immediately upon receiving sufficient evidence; late messages remain useful while a phase is open. A silent participant can stall progress.

## Resource bounds and validation

The transaction protocol bounds admitted contributions through its resource accounting and its limits on uncertified submissions. These bounds do not automatically bound block headers, root advertisements, speculative sketches, duplicate delivery or competing recovery proposals.

The peer specification covers bounds on exchanges, speculative sketches and unresolved roots. DAG rules must additionally bound block production and recovery proposals. Invalid contributions do not enter validated application state.

Cordial parent inclusion is intended to constrain progress and effort on unsupported branches. A bound on total work across concurrent branches remains to be established. Reconciliation cannot convert an unavailable root into available data or force a withholding participant to sign.

## Open specification choices

The remaining choices are:

* Canonical block encodings and the interface to peer-layer committed sets.
* Quiescence scope and publication scheduling, including handling of continuing traffic and stalled neighbors.
* Concrete phase readiness predicates, confirmation of the proposed closing-acknowledgement rule, admission boundaries and inclusion guarantees for each recovery mode.
* Bounds on block production and branch proliferation.

The DAG interpretation and phase rules are independent of the peer layer's choice of reconciliation algorithm.

## References

[^zef]: Mathieu Baudet, Alberto Sonnino, Mahimna Kelkar and George Danezis. [Zef: Low-latency, Scalable, Private Payments](https://arxiv.org/pdf/2201.05671), Section 3, "Sharding and cross-shard queries."

[^paxpay]: Maxence Brugeres, Victor Languille, Petr Kuznetsov and Hamza Zarfaoui. [Fast, Private and Regulated Payments in Asynchronous Networks](https://drops.dagstuhl.de/storage/00lipics/lipics-vol354-aft2025/LIPIcs.AFT.2025.3/LIPIcs.AFT.2025.3.pdf), Section 7.2, discussion of serial-number-range sharding within validators. Full version: [ePrint 2025/098](https://eprint.iacr.org/2025/098).

[^bec]: Martin Kleppmann and Heidi Howard. [Byzantine Eventual Consistency and the Fundamental Limits of Peer-to-Peer Databases](https://arxiv.org/abs/2012.00472).

[^schnorr]: Pieter Wuille, Jonas Nick and Tim Ruffing. [BIP 340: Schnorr Signatures for secp256k1](https://bips.dev/340/). Reference for the Schnorr verification equation and key-prefixed challenges; the predecessor nonce commitments and equivocation-evidence policy are part of this protocol.

[^cordial]: Idit Keidar, Oded Naor, Ouri Poupko and Ehud Shapiro. [Cordial Miners: Fast and Efficient Consensus for Every Eventuality](https://arxiv.org/abs/2205.09174).

[^beluga]: Tasos Kichidis et al. [Beluga: Block Synchronization for BFT Consensus Protocols](https://arxiv.org/abs/2511.15517).

[^morpheus]: Andrew Lewis-Pye and Ehud Shapiro. [Morpheus Consensus: Excelling on Trails and Autobahns](https://arxiv.org/abs/2502.08465).

[^prefix]: Zhuolun Xiang, Andrei Tonkikh and Alexander Spiegelman. [Prefix Consensus For Censorship Resistant BFT](https://arxiv.org/abs/2602.02892).
