---
type: concept
title: Concrete cost function
status: draft
stale_after: 2026-11-06T00:00:00Z
---

# Concrete cost function

The concrete terms of the [cost function](cost_function_abstract.md), and a procedure for constructing the decision tree it evaluates. Each term is specified in its own chapter. This one summarizes them.

## Terms

Objective costs, such as fees and blockspace, are concrete. The terms below are privacy liabilities and preferences.

### Randomization

Two weak preferences, defined only on outpoints. $t$ is a block hash, used as a public randomness beacon, and $x_i = H(t, i)$ for each input outpoint $i$. Each preference is worth less than 1 sat, so it only breaks ties.

- **Individual:** $x_i$, preferring "lucky" hashes. This is a total order on coins.
- **Pairwise:** $x_i \oplus x_j$, preferring pairs with a small XOR metric.

Both are summed over the transaction's inputs, through a monotone scaling function. They bias co-spending through market choices. See the *Randomized selection and Sybil resistance* chapter.

### Sybil resistance

The provenance term estimates the size of the order book at the time of a coin's funding transaction, from that transaction's input set. So it can be seen as a multiplier for the liquidity (coindays) and the transaction cost (fees) of the observed transaction. A discount rate reduces them to one cost. It can be repeated for ancestors, subject to data availability and computational limits.

That's why it's conservative: the cost to the adversary can be reported as a minimum cost and a predicted maximum. The estimate can also be used for degree sequence estimation in the Karrer–Newman model of the *Max flow based counterfactual path estimation* chapter.

The Sybil resistance index $L$ is a lower bound on the number $N$ of candidates, computed from joint sums of spans:

1. Sort the input hashes $x_i$ on a circle of $2^{64}$ points.
2. $T_{m,r}$ is the smallest total width of $r$ spans with disjoint interiors, each from a hash to its $m$-th successor, plus one grid unit per span. Let $\tau_{m,r} = T_{m,r} / 2^{64}$.
3. For a hypothetical pool of $M$ candidates:
   $$
   q_{m,r}(M) = \min\left(1, \frac{M^{r(m+1)} \, \tau_{m,r}^{rm}}{r! \, (rm)!}\right)
   $$
4. The score is:
   $$
   S(M) = \sum_{m,r} \frac{q_{m,r}(M)^{-1/2} - 1}{m(m+1)r(r+1)}
   $$
5. $L$ is the smallest $M$, at least the number of inputs, with $S(M) \lt 1/\delta$.

Someone examining the transaction doesn't know $t$. So they take the largest $L$ over the $W$ candidate block hashes, each at $\delta / W$.

Joint sums are used because the estimate:

- **is symmetric:** rotating or reflecting the hashes doesn't change $L$;
- **uses all input values:** every input's hash enters through the spans;
- **can be bounded computationally:** caps on $m$ and $r$ keep the guarantees, and sums of single gaps screen each block hash;
- **is simple to understand how it's monotone:** adding an input splits a gap, so the sums only shrink and $L$ only grows;
- **has a controllable error:** $P(L \gt N) \le \delta$ for every selection strategy.

See the *Randomized selection and Sybil resistance* chapter for the terms, and the *Sybil resistance index* chapter for the proofs and measurements.

### Subset sum ambiguity

A liability $0 \le \ell(o) \le 1$ for each output $o$ of a proposed decomposition.

- It scores how well subset sums of the other users' inputs' effective values cover the complete sumset of the decomposition from above, within a slop $\varepsilon$.
- The denser these subset sums, the more explanations, and the lower $\ell(o)$.
- Counting stops once there are enough explanations, and $\ell(o)$ only falls as inputs and outputs are learned.

See the *Subset sum ambiguity* chapter, and the *Subset sum density* chapter for how density is estimated and optimized.

### Max flow based counterfactual path estimation

Disjoint flows: the number of counterfactual routes $\lfloor F / d \rfloor$ from a source coin to a set of sink coins.

- $F$ is a max flow over the transaction graph, with every edge capped at the amount $d$.
- Each own coin gets a label $0 \le \iota(c) \le 1$ from the routes from its own origins. At $\iota(c) = 1$ it's impervious to intersection attacks.
- For coins spent together, the routes matrix, with the routes from each origin to each input, shows what an intersection can reveal.

See the *Max flow based counterfactual path estimation* chapter.

### Consolidation

Spending own coins together links them, and loss from intersection attacks compounds with each one.

- $\ell_c = (b^{\nu_0} - 1) \cdot (1 - p q)$, where $\nu_0$ is the number of inputs less one, and:
  - $q = 1 - \nu / \nu_0$ measures the routes, where $\nu$ sums $1 - \iota$ over the inputs, less the largest one;
  - $p$ measures the density of their subset sums.
- Normally the penalty can be larger than the budget, to avoid actions vulnerable to intersection attacks.
- Routes and density each relieve it only in proportion to the other. With subset sums dense enough that the inputs can't easily be clustered, and with counterfactual routes, consolidation is tolerated freely: $\ell_c = 0$ once the density reaches its threshold and every input but one is impervious. Density alone isn't enough, because comparing the ancestor sets of pairs of inputs narrows down an intersection attack efficiently, even in a large transaction.
- So the cost function compels a strategy. First CoinJoin with low fan-in while the subset sums on the inputs are dense. Once the counterfactual routes are built and the funds are well absorbed into the CoinJoin graph, consolidate their descendants, subject to the counterfactual superconcentrator condition: $K$ routes over the other users' edges only, from the earliest ancestor coins to the inputs under consideration.

See the *Consolidation* chapter.

### Sibling avoidance

Spending outputs of the same transaction together is penalized heavily.

- $\ell_s = \sigma \sum_T \max(0, j_T - 1)$, where $j_T$ is the number of inputs created by transaction $T$, and $\sigma$ is large.
- Neither $\iota$ nor density reduces it.

See the *Siblings* chapter.

### UTXO pool diversity

A wallet also values the diversity of its own UTXO pool:

1. some radix denominations;
2. some coins with high Hamming weight;
3. a good size distribution, possibly adapted to the wallet's payment distribution.

How to represent this mathematically is an open question.

### Other ad hoc terms

Other ad hoc terms, such as fingerprint avoidance, are yet to be specified.

### In the decision tree

- **Subset sum ambiguity** can only fall as other users' inputs and outputs are learned. This gives the best and worst cases in [step 5](#5-bracket-and-back-up-values).
- **Labels** $\iota$, and the routes matrix entries for confirmed inputs, are fixed once a coin confirms, so they're cached per coin. Nothing other users add changes them.
- **Siblings and consolidation** are fixed when the inputs are chosen, except that $\ell_c$ can only fall as density rises.
- **Best cases** of the privacy liabilities are 0, used for pruning.
- **Randomization** changes during coalition formation, as candidates appear. The provenance term is fixed once a coin's funding transaction confirms. Both are events that trigger [re-evaluation](#re-evaluation).

## Decision tree

The rest of this chapter gives a procedure for constructing the decision tree. The input is the current wallet state. The output is a tree whose edges are actions and whose nodes are successor wallet states.

### Inputs and output

The wallet state contains the spendable UTXO set, inflight transactions, current protocol observations and beliefs, policy parameters, and an ordered queue $Q=(q_1,\ldots,q_n)$ of pending actions. An entry may be a PSBT fragment, a lone `TxOut` for an intended payment, or an arbitrary transaction fragment. This procedure calls every such entry an *intent* without assuming that it already defines a complete transaction.

The edges of the tree are the actions listed in the *cost function* chapter, together with constructing, signing, broadcasting, or replacing a unilateral transaction, and deferring or skipping an intent.

Each path must be executable from the current wallet state. Each leaf denotes the resulting wallet state after a mutually exclusive plan, including its remaining queue and unilateral fallbacks.

### Bottom-up construction

Construct the tree from the leaves toward the root. First enumerate candidate outcomes: complete allocations of queued intents to transaction batches or unilateral fallbacks, together with concrete inputs, outputs, change structures, and protocol contingencies. Then derive action sequences that realize those outcomes from the current wallet state. While working upward, merge reverse paths whenever they share the same predecessor action. Equivalently, factor common prefixes of the root-to-leaf action sequences so that shared, less-committal actions appear closer to the root.

#### 1. Seed partitions of the pending-action queue

A partition $\pi$ of $Q$ groups intents that the wallet plans to realize together. Each part is one batch; distinct parts are separate transactions or sessions. The partition-refinement order captures every way the same intents may be realized together or separately. Its bottom element is the discrete partition of singleton batches, and its top element is the indiscrete partition containing the entire queue in one batch.

Time-dependent payoff curves constrain this lattice. If an intent is elapsing and its expected multiparty delay would make it irrational, move it to an executable unilateral fallback or skip it. The residue remains eligible for batching. This avoids the false dichotomy between doing the whole queue unilaterally and putting the whole queue into one multiparty transaction: one leaf may contain several multiparty batches, several unilateral actions, and deferred intents.

Seed candidate partitions as follows:

1. Always add the bottom and top elements after applying feasibility and time filters.
2. If $n$ is small enough, add the whole partition-refinement lattice. The number of partitions is the Bell number $B_n$, so exhaustive enumeration is reserved for small queues.
3. Otherwise, seed at least one maximal path from bottom to top. Starting at the bottom, repeatedly merge the pair of parts with the greatest priority according to objective cost reduction, coin-selection feasibility, and time constraints. Never backtrack along this path. Its $n$ partitions provide $O(n)$ partition seeds and a linear-space baseline for this stage. Later coin-selection and change alternatives may expand each seed into several outcomes and leaves.
4. Prefer several paths when resources allow. Vary priority order or retain several near-best merges at selected levels so the seed is a tree rather than only a list.
5. Spend remaining precomputation budget on additional partitions. Simulated annealing, randomized merge and split moves, beam search, or another combinatorial search may sample the omitted lattice. During a live protocol session, repeat this search with newly revealed information and score the sampled partitions directly.

#### 2. Allocate coin selections within each partition

For every seeded partition, allocate inputs jointly across all of its batches. A simple implementation processes the batches successively with an ordinary coin-selection algorithm while removing each selected UTXO from the set available to later batches. The order is part of the candidate. No UTXO may appear in two batches of the same partition.

Repeat allocation with different batch orders, coin-selection randomization, or near-optimal solutions so that each batch can have several candidate coin selections. Reject an allocation unless every batch is balanced: its selected input value must exceed the effective cost of its intended outputs and other mandatory transaction fields.

Positive balance establishes feasibility, not rationality. Before expanding an allocation, reject any batch whose excess implies an unreasonable feerate or whose objective cost can never be offset by its payoff curve. Also reject any choice that the internalized privacy model marks as prohibited, encoded as an infinite privacy liability: by construction such a choice is never rational, regardless of nominal fee savings. Finite privacy harms remain priced terms in the wallet-state valuation and may trade off against other costs.

#### 3. Recover surplus with change construction

Coin selection generally leaves a positive balance that would otherwise be overpaid in fees. Change construction recovers that value and therefore minimizes the batch's objective cost.

In the Radix CoinJoin setting, expand each batch into a decision tree of Radix CoinJoin decompositions. Keep at least $k$ acceptable decompositions selected uniformly at random when at least $k$ exist. A partial decomposition may also be retained when later information is needed to finish it.

In a net-settlement setting, the residual balance may require only an ordinary self-spend output. That produces a degenerate decision tree with one choice and no decomposition decision.

#### 4. Expand leaves into action sequences

For each surviving outcome, construct one or more sequences of atomic actions that lead from the current wallet state to that leaf. A multiparty batch may require initiating a session, sending co-spend proposals, adding inputs, observing other participants' inputs, and adding outputs. A fallback batch may instead require constructing and broadcasting a unilateral transaction. Attach the estimated duration and information revealed by each edge.

When several leaves share an action, factor that action into a common ancestor. Choose the factoring and edge order to preserve options: given each leaf's bracket, put the least committal and least revealing common actions nearest the root. For example, if one or two coins appear in almost every partition, using them in co-spend proposals can leave more of the generated forest addressable than revealing outputs or committing rarer inputs first.

Time-dependent payoff curves participate in this construction. An action sequence is admissible only if its cumulative estimated duration leaves every dependent branch rational at the time it would be taken. A branch may end in a unilateral fallback when coalition formation outlives an intent's multiparty window.

#### 5. Bracket and back up values

Evaluate every leaf under currently available information. A leaf is a pure strategy. The *cost function* chapter scores it as a family of piecewise linear time-dependent curves, one per term, each flagged as privacy-related or not, evaluated at a point and summed. Objective costs, including blockspace, fees, and change, are normally concrete. A leaf's wallet state can itself be valued by imagining a unilateral transaction spending all of its coins right after, and taking that transaction's consolidation penalty and objective costs, as in the *Consolidation* chapter.

Terms that depend on other participants' future input or output choices are bracketed:

- **Worst case:** no other party contributes value, apart from co-spend adjustments in effect.
- **Best case:** every privacy liability is 0. It's used for pruning.

Back the leaf values up through each action, shifting successor payoff curves by the action's estimated duration, and adding the immediate action cost at each edge. These backups internalize future consequences, so the action at the root isn't myopic. The wallet plays a mixed strategy, uniform among the $k$ best leaves. A parent's value is a tree level expectation: the mean of the worst cases of the $k$ least worst leaves below it. This internalizes the optionality of the leaves into their parent.

Once the wallet commits an action, prune its incompatible siblings. Every action discards parts of the forest, so early actions that preserve many acceptable descendants are preferred.

### Re-evaluation

The decision loop is specified in the *cost function* chapter. The tree is extended or rebuilt within the available compute budget, and re-evaluated with current brackets and action-duration estimates, whenever the pending-action queue changes or other new information arrives.

Relevant events include new co-spend proposals, changes to randomization terms during coalition formation, fee or time updates, and observing additional inputs before committing to particular output choices. Event-driven re-evaluation allows the wallet to exploit new information without requiring exhaustive precomputation of the partition lattice.
