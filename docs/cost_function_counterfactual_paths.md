---
type: concept
title: Max flow based counterfactual path estimation
status: draft
stale_after: 2026-11-05T00:00:00Z
---

# 🤖 Max flow based counterfactual path estimation

A privacy term of the [cost function](optimization_framework.md). It counts how many times the sink's amount could have travelled from a source coin to the sink coins, along disjoint counterfactual paths. This is the [problem definition](collaborative_txn_privacy.md)'s robust connectivity, measured with one max flow.

The paths start at any coin linked to the source within its transactions, and end at any coin linked to the sinks. These links are what make more than one route possible. A source and a sink with nothing linked to them give at most one.

Its main use is to label the wallet's own coins. A coin with enough counterfactual routes from its own origins is impervious to intersection attacks. The same counts tell when spending coins together is safe from them.

## Network

Built from the transaction graph alone, following Maurer et al.[^maurer] It knows nothing of clustering, and it's acyclic.

- **Coin edges.** Each coin $c$ has two vertices: where it's created, and where it's spent. An edge joins them, with capacity $v(c)$.
- **Flow edges.** For each transaction, an edge from each spent coin to each created coin.
  - Capacity: $P_{I,O}(i, o) \cdot \min(v(i), v(o))$.
  - $P_{I,O}(i, o)$ is the probability that $i$ and $o$ are in the same sub-transaction.
- **Link edges.** These carry no flow. They only define the neighbourhoods below.
  - Between the inputs of a transaction, with probability $P_{I,I}$.
  - Between its outputs, with probability $P_{O,O}$.
- **Neighbourhoods.** The super-source of a coin $s$ connects to $s$ and to every coin linked to it on either side, with capacity equal to the link probability times that coin's value. The super-sink of the sink coins $T$ is built the same way.
- **Own edges.** When source and sink are both the wallet's coins, only counterfactual paths should remain. Two kinds of edges are set to 0:
  - flow edges between two of the wallet's coins in the same transaction;
  - coin edges of the wallet's coins on the way from the source to the sink.

## Definition

For a single source coin $s$ and sink coins $T$:

1. **Amount.** $d = \min(v(s), v(T))$, where $v(T)$ is the sum of the sinks' values.
2. **Caps.** Every coin edge and every flow edge is capped at $d$.
   - Uncapped: the coin edges of the source and the sinks, and the super-source and super-sink edges.
   - Coins linked to the source or the sinks are capped like any other, so each carries at most one route.
3. **Flow.** $F$ is the max flow from the super-source to the super-sink.
4. **Routes.** $k(s, T) = \lfloor F / d \rfloor$.

For several sources, $k(S, T) = \min_{s \in S} k(s, T)$.

- A single super-source over all of $S$ would add up the routes of different sources. Two linked coins with one route each would score 2, hiding the intersection.

### Why this counts routes

- **Theorem.** An acyclic flow of value $v$ in which no edge carries more than $v/k$ is a $k$-route flow.[^kishimoto][^aggarwal-orlin][^chekuri-idleman]
  - That is, it's a sum of bundles. Each bundle is $k$ paths that share no edge, carrying equal amounts.
- **Guarantee.** With every capped edge at $d$ and $F \ge k d$, removing any $j \lt k$ capped edges still leaves a flow of at least $(k - j) d$.
  - So no $k - 1$ capped edges cut the source off from the sinks. This is the problem definition's "no small cut".
- **Uncapped edges.** These behave like $k$ parallel copies, so routes may share them.

### Split routes

A route may be spread over several smaller coins, as long as no two routes share a capped edge. For example, two disjoint coins of $d/2$ each count as one route of $d$.

The floor makes $k$ conservative. Three disjoint coins of $d/2$ give $k = 1$, yet any one of them can be removed and $d$ still gets through.

### Max flow alone isn't enough

- Without caps, a single large coin can carry all of $F$, so $F$ can't tell one route from several.
- Scaling the demand to $k d$ doesn't help, for the same reason.

## Labels

For each own coin $c$:

- **Origins.** $O(c)$ is the set of coins by which the wallet received the funds $c$ descends from.
- **Routes.** $k_c = \min_{o \in O(c)} k(o, \{c\})$, with the wallet's own edges set to 0.
- **Label.** $\iota(c) \in [0, 1]$ rises with $k_c$ and reaches 1 at $K$, the number of routes at which a coin counts as impervious to intersections. For example, $\iota(c) = \min(1, k_c / K)$.
- **Use.** $\iota$ is the label the consolidation term, in the [Consolidation penalty](cost_function_consolidation.md) chapter, takes as given. At $\iota(c) = 1$, $c$ adds nothing to that term's linked count $\nu$.

Everything $k_c$ depends on exists once $c$ confirms: the coins between its origins and $c$, the other coins in their transactions, and their link probabilities. So $k_c$ is fixed from then on, and labels are cached per coin.

### Consolidation

Spending own coins $I$ together links them. An intersection attack then intersects their candidate origins.

- **Union.** $U = \bigcup_{i \in I} O(i)$: every coin by which the wallet received funds that any of them descends from.
- **Routes matrix.** $M_{i,u} = k(u, \{i\})$, for each $i \in I$ and $u \in U$.
  - It's a bipartite graph from $U$ to $I$, with each edge labelled by its routes, written as a matrix.
  - Each entry is one max flow. Entries already computed for a coin's own origins carry over: $k_c$ is the minimum of row $c$ over $O(c)$.
- **Reading it.** Row $i$ shows which origins $i$ has many counterfactual routes from. When every row is high in almost every column, each input's candidate origins cover almost all of $U$. The sets the adversary can intersect then have a small symmetric difference, so intersecting them reveals little.
  - "Almost every" is read off $M$, not fixed in advance. For example, the columns where one row is at least $K$ and another isn't are what an intersection of those two inputs can reveal.
- **Attenuation.** The routes factor is the degree to which the consolidation penalty is attenuated.
  - Without attenuation, the penalty can exceed the budget, so actions open to intersection attacks are avoided.
  - Consolidation is tolerated freely only when both hold: the subset sums are dense enough that the inputs can't easily be clustered, and the counterfactual routes are in place.
  - Both are required for any attenuation. Neither is sufficient on its own.
  - Each has a threshold. When both meet their thresholds, the penalty is fully discharged.
  - Dense subset sums aren't enough: even in a large transaction, comparing the ancestor sets of pairs of inputs efficiently narrows down an intersection attack.
- **Counterfactual superconcentrator.** The graph of other users' edges only, with the wallet's own edges set to 0.
  - Sources: the earliest ancestor coins, $U$.
  - Sinks: the inputs under consideration for consolidation, $I$.
  - The aim is $K$ counterfactual routes across it, making it a superconcentrator: for every $j$, any $j$ sources and any $j$ sinks are joined by $j$ disjoint paths.

## In the decision tree

- **Fixed per coin.** Once a coin confirms, no later block adds to its ancestry. So $\iota$, and the entries of $M$ for confirmed inputs, are computed once and reused in every leaf. Over time, they're constant.
- **A leaf's inputs.** A leaf fixes the own inputs $I$, and with them $U$ and the rows of $M$. Nothing other users add to the transaction changes them, so the best and worst cases are equal.
- **Outputs.** The transaction's own outputs get labels only after it confirms.
- **Order book coins.** The own inputs' coin edges are set to 0, so routes from their origins into the transaction's own outputs enter only through other users' inputs.
  - An order book coin that descends from those origins adds routes, so selecting it is valuable.
  - A co-spend proposal from an own input that names such a coin links the two, unless such proposals are already common. An outcome may still prefer the selection, but the cost function should penalize proposals that leak the link this way.
- **Bounds.** Leaves use $\iota$ from $k^-$. Tightening $k^-$, with more walks or an exact $k$ on a pruned graph, can run between iterations of the decision loop in the [Mixed strategy](mixed_strategy.md) chapter. It can only raise $\iota$.
- **Score.** The routes factor attenuates the consolidation penalty. See [Consolidation](#consolidation).
- **Strategy.** Building the routes compels a strategy:
  1. While the subset sums on the inputs are dense, CoinJoin with low fan-in.
  2. Once the counterfactual routes exist and the funds are well absorbed into the CoinJoin graph, consolidate their descendants, subject to the counterfactual superconcentrator condition.

## Scope

For one source and one sink, only the coins that descend from the source's neighbourhood and are ancestors of the sink's neighbourhood matter. These lie between the two confirmations. One pair never needs the full chain.

### Pruning

Any restriction of the transaction graph gives a lower bound on $k$, since removing coins or edges can't raise a max flow. Some examples:

- the last $n$ blocks;
- only CoinJoins meeting some conditions;
- only the flow edges of specific coins in them, such as equal amount outputs.

A lower bound is safe for labels: a coin is never labelled impervious when it isn't.

Restricting to CoinJoins undercounts, though: payments and PayJoins intertwined with CoinJoins also carry counterfactual routes.

### Cost

- One max flow per source. Sources are independent, so they parallelize.
- A transaction with $n$ inputs and $m$ outputs has $n m$ flow edges.
  - Cache $P_{I,O}$ per transaction. It has a closed form for equal amounts, and needs subset sum enumeration otherwise.
  - Leaving out edges with small $P_{I,O}$ gives a lower bound.
- A new block only adds coins and edges, so it never lowers $k$.

## Approximation

This avoids computing $k$ on the full graph.

### Data access

How much of the chain a wallet sees depends on its constraints:

- With limited bandwidth, it samples blocks.
- With limited storage, it fetches every block in a range, but keeps only a subgraph in a compact encoding. See [Pruning](#pruning).
- If it can only fetch transactions adjacent to ones it knows, it uses the [bounds](#bounds).

### Layers

The approximation estimates each layer of the network separately. Each layer depends only on the ones before it.

1. **Topology.** Transactions, their numbers of inputs and outputs, and how long coins stay unspent.
2. **Amounts.** Coin values. Relative to $d$, they set each coin's capacity.
3. **Links.** $P_{I,O}$, $P_{I,I}$ and $P_{O,O}$, computed from the amounts within each transaction. They set the flow edges and the neighbourhoods.
4. **Knowledge.** The wallet's own edges.

### Sampled blocks

- Sample blocks, and estimate each layer's distributions from their transactions, separately.
  - For the topology: the numbers of inputs and outputs per transaction, and the spend rate $p$, the fraction of unspent coins spent per block.
- Base the estimate on randomized selection, as in the [Randomized selection and Sybil resistance](cost_function_randomized_selection.md) chapter. When coins are selected at random from the candidates, the transaction graph can be treated as a random acyclic network with those degree distributions.[^karrer-newman-acyclic][^karrer-newman-models]
- **Number of candidates $N$.** A larger $N$ gives a smaller $k$, so the conservative choice is an upper bound, such as the number of unspent coins.
  - Randomized selection gives a lower bound $L \le N$. Using it in place of $N$ would overstate $k$.
  - In the pool model below, doubling $N$ from 1000 to 2000 lowers $E[k]$ from 1.20 to 0.73 at $n = 4$, $R = 12$.

### Time slices

Every route crosses each block boundary on a coin that's unspent at that boundary.

- A **slice** is the set of coins unspent at a boundary that descend from the source's neighbourhood and are ancestors of the sinks' neighbourhood, excluding the neighbourhoods themselves.
- Each slice is a cut, so $k$ is at most the smallest slice's capacity.
- In the prototype below, $k$ equals the smallest slice in 95% to 100% of runs.

### Analytic estimate

This covers the topology layer only: every coin's capacity is $d$, and every $P_{I,O}$ is 1.

- **Growth.** $D_r$ is the number of unspent coins after block $r$ that descend from the source's neighbourhood.
  - $D_{r+1} = (1 - p) D_r + \frac{p N \bar{o}}{\bar{\imath}} \left(1 - G(1 - D_r / N)\right)$.
  - $\bar{\imath}$ and $\bar{o}$ are the mean numbers of inputs and outputs, and $G$ is the generating function of the number of inputs. The two numbers are assumed independent.
  - Under random selection, a transaction joins the descendants with probability $1 - G(1 - D_r / N)$, the probability that any of its inputs is a descendant.
  - $A_r$, the ancestors of the sinks' neighbourhood, grows the same way backwards from the sink, with inputs and outputs swapped.
- **Slices.** Under random selection, an unspent coin after block $r$ is an ancestor of the sinks with probability $A_r / N$, independently of whether it descends from the source.
- **Ends.** The slices next to the source and the sinks are the narrowest, and nearly independent of each other.
  - $X \sim \text{Binomial}(n_s, A_0 / N)$, where $n_s$ is the number of coins the source's neighbourhood first reaches.
  - $Y \sim \text{Binomial}(n_T, D_{R-1} / N)$, where $n_T$ is the number of coins that first reach the sinks' neighbourhood, and $R$ is the number of blocks between them.
  - $k \approx \min(X, Y)$, so $P(k \ge K) \approx P(X \ge K) \cdot P(Y \ge K)$.
- Using only the single narrowest slice, $\text{Binomial}(D_r, A_r / N)$, overestimates $k$.

The estimate is neither a lower nor an upper bound.

### Monte Carlo

- Generate transaction graphs from the estimated layers, in order: topology, amounts, links, knowledge.
- Compute $k$ on each with the max flow, restricted to the coins between source and sink.
- Estimate $P(k \ge K)$ from the results.
- Unlike the analytic estimate, this covers every layer.

**Prototype, analytic and Monte Carlo.** A pool of $N$ candidate coins of equal value. In each block, a fraction $p = 0.2$ of the pool is selected at random into transactions of $n$ inputs and $n$ outputs. The source's neighbourhood is the inputs of a transaction in the first block, and the sinks' is the outputs of one $R$ blocks later. Each Monte Carlo row is 30 to 40 runs.

| $n$ | $N$ | $R$ | $E[k]$, Monte Carlo | $E[k]$, analytic | $P(k \ge n/2)$, Monte Carlo | $P(k \ge n/2)$, analytic |
|---|---|---|---|---|---|---|
| 4 | 1000 | 8 | 0.23 | 0.11 | 0.03 | 0.00 |
| 4 | 1000 | 12 | 1.20 | 0.97 | 0.30 | 0.23 |
| 4 | 1000 | 20 | 3.08 | 3.20 | 0.97 | 0.99 |
| 8 | 2000 | 6 | 1.43 | 1.04 | 0.03 | 0.00 |
| 8 | 2000 | 8 | 2.83 | 2.90 | 0.30 | 0.30 |
| 8 | 2000 | 12 | 5.33 | 5.58 | 0.93 | 0.97 |

Counting only whether the first coins at each end are spent or created in time predicts $E[k] = 2.8$ at $n = 4$, $R = 8$. The graph between the ends is what limits $k$ there.

### Bounds

- **Upper bound.** For $k^+$, merge every coin $r$ transactions away from the source into the super-sink, and compute $k$. Do the same from the sink backwards, and take the smaller of the two.
  - Keep $d$ as computed from $s$ and $T$. Merged coins raise the sink's value, and recomputing $d$ from it would break the bound.
  - Merging only adds capacity, so each is an upper bound.
  - The cost depends on $r$ and the local degrees, not on the chain.
- **Lower bound.** For $k^-$, take random walks forward from the source's neighbourhood and backward from the sink's. Follow $f$ randomly chosen edges per step, up to the other end's confirmation height. Compute $k$ on their union.
  - The union is a subgraph, so its $k$ is a lower bound.
  - Walks that cross share edges, and the max flow counts a shared edge once. Overlapping paths aren't double counted.

**Randomization.** With randomized selection, as in the [Randomized selection and Sybil resistance](cost_function_randomized_selection.md) chapter, in the prototype below, this puts the smallest cut next to one of the ends, so $k = k^+$. That isn't guaranteed in general.

- The selection bias bounds the number of candidates from below. It doesn't show that both ends draw from the same candidates: two groups that never transact with each other can each select at random.
- The evidence that the ends are connected is $k^-$ itself, as it approaches $k^+$.

**Labels from bounds.**

- $\iota$ uses $k^-$, so it's never overstated.
- If $k^- \ge K$, $c$ is impervious.
- If $k^+ \lt K$, it isn't, and more sampling won't change that.
- Otherwise, take more walks, or compute $k$ exactly on a pruned graph.

**Prototype, bounds.** A layered model: 7 layers of 256 coins of equal value, joined by 6 rounds of transactions with 8 inputs and 8 outputs, 6 seeds each.

- The source's neighbourhood is the 8 inputs of its transaction, and the sink's is the 8 outputs of its transaction. So 8 is the most $k$ can be.
- **Random selection:** $k = k^+ = 8$ in every seed, so no route was lost in between. With 4 walks branching 2 ways per step, $k^- = 8$ in every seed.
- **Two halves that never transact with each other:** $k = 0$, while $k^+ = 8$.
- **Two halves, 10% of transactions mixed:** $k = 8$ in 5 seeds, and 7 in the other.

## Limitations

- **Sub-transactions.** Flow relaxes sub-transaction mappings: it can split across them in ways no single mapping allows.
- **Unknown link probabilities.** Taking $P_{I,O} = 0$ when it's unknown is conservative.
- **Independent transactions.** Each transaction's probabilities are computed on their own. Intersections across transactions enter only through the minimum over sources and the [consolidation](#consolidation) condition.
- **What the adversary removes.** Routes are counted as disjoint in coins and flow edges, while an adversary deanonymizes users, removing all of a user's coins at once.
- **Timing and hops.** Route length and timing aren't modelled.

## Open questions

- With several sink coins, should each sink get its own share of $d$?
- What link probabilities apply to the transaction under construction? They must hold in every completion.
- Which term penalizes a co-spend proposal that leaks a link to an order book coin that adds routes, and how is "already common" measured?
- How is the routes factor computed: from each input's $\iota$, as the [consolidation term](cost_function_consolidation.md) does now, or from the counterfactual superconcentrator condition?
- How is the counterfactual superconcentrator condition checked? It ranges over sets of sources and sinks of equal size, while $M$ holds only single pairs.
- Should a privileged counterparty's own coins also be set to 0, since it knows where they went?
- Should $\iota$ ramp up to $K$, or switch from 0 to 1 at $K$? What should $K$ be?
- May $\iota$ use an estimate ($k^+$, the analytic estimate or Monte Carlo) when the randomization evidence is strong, or only $k^-$?
- **Known transactions.** The wallet already has the transactions next to its own coins. Should the estimates use those as they are, such as the actual $n_s$ and $n_T$, and model only the transactions it hasn't fetched?
- **Amounts and links in the analytic estimate.** Can they enter as expected capacities per coin, or only through Monte Carlo?
- How should competing spends of one coin be handled?
- What label does an own input get before it confirms, such as change from a transaction still in flight?

[^maurer]: F. K. Maurer, T. Neudecker, M. Florian, [*Anonymous CoinJoin Transactions with Arbitrary Values*](https://doi.org/10.1109/Trustcom/BigDataSE/ICESS.2017.280), TrustCom 2017.

[^kishimoto]: W. Kishimoto, [*A method for obtaining the maximum multiroute flows in a network*](https://doi.org/10.1002/(SICI)1097-0037(199607)27:4%3C279::AID-NET3%3E3.0.CO;2-D), Networks 27(4), 1996.

[^aggarwal-orlin]: C. C. Aggarwal, J. B. Orlin, [*On multiroute maximum flows in networks*](https://doi.org/10.1002/net.10008), Networks 39(1), 2002.

[^chekuri-idleman]: C. Chekuri, M. Idleman, [*Congestion Minimization for Multipath Routing via Multiroute Flows*](https://doi.org/10.4230/OASIcs.SOSA.2018.3), SOSA 2018, Theorem 1.

[^karrer-newman-acyclic]: B. Karrer, M. E. J. Newman, [*Random acyclic networks*](https://arxiv.org/abs/0902.4013), 2009.

[^karrer-newman-models]: B. Karrer, M. E. J. Newman, [*Random graph models for directed acyclic networks*](https://arxiv.org/abs/0907.4346), 2009.
