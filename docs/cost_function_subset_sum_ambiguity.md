---
type: concept
title: Own inputs or outputs subset sum ambiguity
status: draft
stale_after: 2026-11-05T00:00:00Z
---

# 🤖 Own inputs or outputs subset sum ambiguity

<!-- cspell:words Salas -->

A privacy liability term of the [cost function](optimization_framework.md). Given the wallet state and a transaction under consideration, it estimates how ambiguous a proposed output decomposition is, relative to the other users' coins known so far. It's cheap enough to evaluate for many decompositions.

It implements [subset sum density](subset_sum_density.md) for the outputs: a bitmap or sparse convolution of the other users' inputs' effective values, with lookups for the complete sumset of the outputs, approximated from above.

## Inputs

- $f$: the feerate of the [coalition formation proposal](coalition_formation.md). Effective values are [fixed](subset_sum_density.md#effective-values) at $f$.
- $X$: effective values $a$ of the other users' inputs known so far, positive values only.
- $Y$: effective costs of the other users' outputs known so far.
- $I$: the own inputs, as effective values $e_a$.
- $O$: the proposed decomposition, as effective costs $c_o$, with $m = |O|$.
- $\beta$: the wallet's balance, as effective value.
- Parameters:
  - $\varepsilon$: the slop;
  - $A$: the number of explanations that counts as enough for one output;
  - $\varphi$: the explained fraction at which the liability reaches 0;
  - $\theta$: the width of the liability ramp, in bits.

## Definition

1. **Targets.** For each non-empty $S \subseteq O$: $s_S = \sum_{o \in S} c_o \le \beta$.
2. **Companions.** $C$ is the set of subset sums of $Y$, including 0.
3. **Ways.** $w(S, y)$, for $y \in C$, is the number of subsets of $X$ that sum into $[s_S + y, s_S + y + \varepsilon]$.
   - Each one is a reading in which another user's inputs fund $S$ together with other users' outputs summing to $y$.
4. **Per-pair cap.** $\kappa = \lceil A / 2^{m-1} \rceil$. Each pair contributes at most $\kappa$ ways (satisficing).
   - A large decomposition needs one explanation per combination.
   - A single output needs $A$ ways for its only target.
5. **Count.** $n_o = \sum_{(S, y) : o \in S} \min(\kappa, w(S, y))$.
6. **Size correction.** $f_o = \min(1, n_o / (\kappa \cdot 2^{m-1}))$.
   - Adding an output doesn't raise $f_o$ by itself. It has to pay for its blockspace through the objective cost.
7. **Liability.** $\ell(o) = \text{clamp}(\log_2(\varphi / f_o) / \theta, 0, 1)$.
   - It's 0 once $n_o \ge \varphi \kappa 2^{m-1}$.
   - It falls by $1/\theta$ per doubling of $n_o$.

## Sumset overlap

How well combinations of the user's coins match combinations of the other users' coins on the same side. Each match is a reading in which another user's coins take the place of the user's.

- **Inputs, during coalition formation.** For each non-empty $S \subseteq I$, with $r_S = \sum_{a \in S} e_a$, $v(S)$ is the number of subsets of $X$ summing into $[r_S - \varepsilon, r_S + \varepsilon]$.
  - These are lookups in $D$ or $T$, which already cover $X$.
  - Steps 4–6 apply with $I$ in place of $O$ and $v$ in place of $w$, giving an overlap fraction $u_a$ per own input.
- **Outputs, in hindsight.** The same, with $O$ and the other users' outputs, once they're all known, giving $u_o$.
  - The lookups are in $C$, or in a capped count table over $Y$ like $T$ when $\kappa > 1$.
- [Same side](#one-by-one) one by one matching is the case $|S| = 1$.

## Implementation

### Tables

Built once per transaction and shared by all decompositions:

- $D$: a bitmap with $D[i] = 1$ if some subset of $X$ sums into $[i, i + \varepsilon]$. It's used when $\kappa = 1$.
  - Start with $D[-\varepsilon..0] = 1$.
  - Then apply $D[i] \gets D[i] \lor D[i - a]$ per input, from the top down.
  - Where the distinct sums are far fewer than the range, [Salas](https://arxiv.org/abs/2507.15511)'s enumerator can replace it.
- $T$: $\min(A, \text{number of subsets summing to } i)$, as u16. It's used when $\kappa > 1$.
  - Apply $T[i] = \min(A, T[i] + T[i - a])$ per input, from the top down.
  - Window sums stop at $\kappa$.
- $C$: a bitmap of companion sums.
- **Range:** $[-\varepsilon, \beta + y_{\max}]$, where $y_{\max}$ is the largest companion sum visited. Double it and rebuild if needed. Skip inputs that are too large to fit.

### Updates

- Each auxiliary input or concurrent output is one more pass over $D$, $T$ or $C$. The order doesn't matter.
- Deduplicate inputs by outpoint and outputs by output id.
- Removing a value needs uncapped counts, or a rebuild.

### Per decomposition

- For each $S$, add $\min(\kappa, w)$ to $n_o$ for each $o \in S$, for $y = 0$ first and then companions in increasing order.
- **Satisficing stops the work early:**
  - a window, at $\kappa$;
  - an output, once $\ell(o) = 0$;
  - a decomposition, once all of its outputs have.

### One by one

Each own coin is matched against single coins of other users, using sorted arrays and binary search.

- **Across sides:** a single input in $[c_o, c_o + \varepsilon]$, or a single output in $[e_i - \varepsilon, e_i]$.
- **Same side:** swaps within $\varepsilon$, and input and output swaps whose differences cancel.
- **Fast path:** these can satisfice an output before any table is built.

### Radix denominations

These are inputs with nominal value $c \cdot b^i$:

- $b = 2$, $c = 1$;
- $b = 3$, $c \in \{1, 2\}$;
- $b = 10$, $c \in \{1, 2, 5\}$.

They're grouped by effective value $v$, with multiplicity $m_v$.

- **Multiplicity:** one pass per group, as a sliding window along each residue class mod $v$.
- **Contiguity:** sort the coins and keep a running sum $R$. Accept each next coin while it's $\le R + \varepsilon$, starting from a coin $\le \varepsilon$.
  - Every target $\le R$ is then explained without the tables.
  - Keeping $m_v - r$ coins per value gives robustness to $r$ removals per value.

## In the decision tree

- It's a privacy-related term.
- **Worst case:** only the coins known so far count. $X$ and $Y$ are lower bounds, and auxiliary inputs and outputs are learned later. Before any other user's outputs are known, $C = \{0\}$.
- **Best case:** $\ell(o) = 0$, which is used for pruning.
- Every participant, including the aggregator in that role, evaluates [co-spend proposals](subset_sum_density.md#co-spend-proposals) with this term, deciding whether to propose a generated one or to accept an incoming one.
  - A generated proposal can carry density challenges, with a reward in the surplus.
  - Perturbing a proposal adds values to the tables, one pass each.
- $\ell(o) \in [0, 1]$.

## After confirmation

This is the analysis that contributes privacy gains, and that uses the budget.

- All inputs and outputs are known.
- **Coalition formation proposal known:** evaluate at its $f$, which continues the decision tree's chain. Also evaluate at the final feerate, which is a second scenario.
- **Recovery from seed:** evaluate at the final feerate only.
- **Inputs absorbed into outputs:** the mirror image.
  - Targets are subsets of the wallet's inputs.
  - Subsets of the other users' outputs must sum into $[r_S - \varepsilon, r_S]$.
  - In the decision tree, only the outputs known so far count, which before then means none.

## Monotonicity

- The decision tree's estimate is the same definition restricted to the lower bounds $X$ and $Y$.
- Learning inputs or outputs only adds ways and pairs, and the denominator depends only on $m$ and $A$. So $\ell(o)$ can only fall.
- This requires the same $f$, the same realized $O$, and the same parameters.

## Open questions

- Is $\varepsilon$ absolute, or relative to the target? Does it include adjustments?
- Should ways count input subsets, or distinct sums in the window?
- For one by one: across sides, same side, or both?
- How do $u_a$ and $u_o$ enter the liability: as their own term, or as extra ways in $n_o$?
