---
type: concept
title: Randomized selection and Sybil resistance
status: draft
stale_after: 2026-11-06T00:00:00Z
---

# Randomized selection and Sybil resistance

<!-- cspell:words coindays msats rivalrous Karrer superconcentrator superconcentrators bhatt Bhatt -->

Three terms of the [cost function](optimization_framework.md). Two weak preferences randomize the choice of which UTXOs to co-spend with, using the blockchain as a randomness beacon. The provenance term values coins by how well that randomization can be observed in their history.

Together they:

- make Sybil attacks more costly;
- promote a robustly connected transaction graph, with similarities to [interconnection networks](https://en.wikipedia.org/wiki/Multistage_interconnection_networks);
- establish a Schelling point among takers with regards to which makers to interact with;
- reduce temporal fingerprinting of makers' UTXOs' availability;
- provide useful information to light clients with limited access to the blockchain.

They compose with arbitrary other terms. The setting is non-cooperative, so they are a component of a strategy, not a mechanism. If widely adopted by those paying for CoinJoin blockspace, privacy related payoffs are more strongly positive sum.

Makers announce UTXOs in an order book, and takers propose to spend them in CoinJoin transactions, as in [coalition formation](coalition_formation.md). The roles are used loosely: a single participant may perform both.

## Inputs

- $t$: a block hash, e.g. 144 blocks from the best known chain tip.
- $I$: the outpoints being considered for spending in a possible transaction, with $n = |I|$.
- $x_i = H(t, i)$ for each $i \in I$, where $H$ is a tagged hash. Hash outputs are interpreted as rational numbers in $[0, 1]$.
- $f_1, f_2$: monotone scaling functions, e.g. log or sqrt, possibly with a bias.

## Per input term

$$
f_1\left( \sum_{i \in I} x_i \right)
$$

- How "lucky" a maker's coin is at a point in time, with many leading 1's in the hash output being considered lucky. This is a total ordering over inputs.
- A weak preference: more than 0 but less than 1 sat, so it only breaks ties.
- It lets takers converge on the same makers' coins, especially when the number of active takers is small enough that a grand coalition can construct a single transaction. All else being equal, takers concentrate on the same subset of makers, reducing their shared costs.

## Pairwise term

$$
f_2\left( \sum_{i,j \in I} x_i \oplus x_j \right)
$$

- An affinity between inputs, using the XOR metric. Valuing it promotes randomized assortativity between inputs.
- A weak preference, like the per input term. It scales quadratically in the number of inputs, which $f_2$ must account for.
- **Tipping point.** Due to the CoinJoin liveness model, success probability is roughly exponential in the number of inputs. This limits how far the total order of the per input term can be optimized. Where these interactivity costs dominate, this term dominates over the first, avoiding more rivalrous competition for lucky coins to all be spent in the same transaction.
- **Partitions.** When the candidate UTXO set is larger than the feasible transaction size, this term promotes partitioning of the candidate UTXO set, with each part being a reasonable estimator for its size.
- **Why XOR.** For uniform $x_i$, $x_i \oplus x_j$ is also uniform. Preferring small values also biases the $x_i$ themselves: a pair whose XOR has $b$ leading zero bits shares its $b$ most significant bits, so both lie in a random subinterval of size $2^{-b}$. This simplifies the density estimation. The Hamming distance or the arithmetic difference would change the distribution, and the scaling functions can simulate such effects.

## Provenance term

For a coin under consideration, a fidelity bond like analysis of the transaction that created it, its funding transaction:

1. **Order book size.** Estimate the size of the order book at the time of the funding transaction, from its input set. This is the Sybil resistance index $L$, a lower bound on the number of candidate UTXOs $N$, computed from [joint sums](#joint-sums).
2. **Cost.** $L$ is a multiplier for the liquidity, in coindays, and the transaction cost, in fees, of the observed transaction. A discount rate reduces them to one cost.
3. **Ancestors.** Repeat for earlier transactions, subject to data availability and computational limits.

The term rises with this cost: an inherited "luck" from previously lucky coins, or from coins that had a high affinity to each other at some previous point in time.

- **Why it's a cost.** For a single adversary to construct a transaction exhibiting such a bias, it must control the entire set of candidate UTXOs, not just the coins that end up being selected. Then takers would have to select the outputs of this transaction.
- **Conservative.** $L$ is a lower bound, so the cost to the adversary can be reported as a minimum cost and a predicted maximum.
- **Only with the preferences.** The term can only be optimized in the presence of a market preference like the per input and pairwise terms, or no bias would be detectable.
- **Costless for honest users.** Assuming takers and makers weakly prefer the bias, it's costless, at least on average. With market demand for it, makers can be profitable. This promotes a wider CoinJoin graph, with less temporal assortativity.
- **Waiting.** The longer a UTXO is unspent, the more chances it has of being lucky.
- **Intersection attacks.** Randomization promotes a high number of paths between coins, even for bounded degree graphs, as in superconcentrator graphs.[^bhatt] It doesn't guarantee them: it draws from the non-random set of available candidate UTXOs at a point in time. The estimate can also be used for degree sequence estimation in the Karrer–Newman model of the *Counterfactual routes* chapter.

See also Belcher's analysis of fidelity bonds.[^belcher]

### Joint sums

For the $n$ input hashes of the funding transaction and a confidence parameter $\delta$:

1. **Spans.** Truncate the hashes to 64 bits, and sort them around a circle of $2^{64}$ points. An $m$-span runs from a hash to its $m$-th successor. Its width is the distance between them, plus one point.
2. **Sums.** For each $1 \le m \lt n$ and $1 \le r \le n/m$, $T_{m,r}$ is the smallest total width of $r$ spans with disjoint interiors. Spans may share endpoints. Let $\tau_{m,r} = T_{m,r} / 2^{64}$.
   - For $m = 1$, this is the sum of the $r$ smallest spacings.
   - Larger $m$ detects groups of nearby hashes.
3. **Tail bound.** For a hypothetical order book of $M$ candidates:
   $$
   q_{m,r}(M) = \min\left(1, \frac{M^{r(m+1)} \, \tau_{m,r}^{rm}}{r! \, (rm)!}\right)
   $$
4. **Score.**
   $$
   S(M) = \sum_{m,r} \frac{q_{m,r}(M)^{-1/2} - 1}{m(m+1)r(r+1)}
   $$
5. **Index.** $L$ is the smallest integer $M \ge n$ with $S(M) \lt 1/\delta$.

Joint sums are used because they are:

- **symmetric:** rotating or reflecting the hashes on the circle doesn't change $L$, so it doesn't matter which hashes count as lucky;
- **based on all input values:** every input's hash enters through the spans;
- **computationally bounded:** leaving out the terms above a fixed $m$ keeps the guarantees, and the $m = 1$ terms, which only need sorting, screen each candidate block hash;
- **simple to understand as monotone:** adding an input splits a spacing, so every $T_{m,r}$ can only shrink, $S$ can only grow, and $L$ can only grow;
- **of controllable error:** $P(L \gt N) \le \delta$ for every selection strategy.

The [Sybil resistance index](sybil_resistance_index.md) chapter gives the proofs, the scan over block hashes, and measurements.

## In the decision tree

- **Per input and pairwise terms** change during coalition formation, as candidates appear.
- **Provenance term** of a coin under consideration is fixed once its funding transaction confirms, so it's computed once per coin and cached.

## After confirmation

This is the analysis that contributes privacy gains, and that uses the budget.

- All inputs are known, but $t$ isn't. It can be guessed from the confirmation time, or the nLockTime. $L$ is the maximum over the candidate block hashes mined after the inputs confirmed, each at $\delta / W$ for $W$ candidates. See [Finding the block hash](sybil_resistance_index.md#finding-the-block-hash).
- The provenance term of the transaction's outputs is then fixed.

## Open questions

- What should $\delta$ be?
- How are the minimum cost and the predicted maximum computed from $L$?
- How far back should the provenance term go, and how should the costs of several ancestors combine? The *Counterfactual routes* chapter walks ancestors too.
- Can light clients estimate $L$ for transactions they only sample?

[^belcher]: C. Belcher, [*Financial mathematics of JoinMarket fidelity bonds*](https://gist.github.com/chris-belcher/87ebbcbb639686057a389acb9ab3e25b#cost-of-sybil-attacks).

[^bhatt]: S. N. Bhatt, [*On Concentration and Connection Networks*](https://hdl.handle.net/1721.1/149007), MIT, 1981.
