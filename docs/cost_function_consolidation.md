---
type: concept
title: Consolidation penalty
status: draft
stale_after: 2026-11-06T00:00:00Z
---

# 🤖 Consolidation penalty

A privacy liability term of the [cost function](optimization_framework.md). Spending several own coins together links them, and loss from [intersection attacks](collaborative_txn_privacy.md) compounds with each one. It's tolerated freely only when the subset sums are very dense *and* the coins are already impervious to intersections. Both are required for any attenuation, and neither is sufficient alone. Each has a threshold, and the penalty is fully discharged once both meet theirs.

- **Why unbounded.** Spending a whole wallet at once, e.g. in a hurry and without regard for privacy, would normally reveal its coins as one user's, and let all of its origin funds be traced. Under the right conditions, even such a transaction, with no other users' coins, can be safe.
- Density alone isn't enough. Comparing the ancestor sets of pairs of inputs narrows down an intersection efficiently, even in a large transaction.

It's related to the term of the [Max flow based counterfactual path estimation](cost_function_counterfactual_paths.md) chapter, which deals with the flows to the coins' ancestry. Its labels attenuate this penalty.

## Definition

- $I$: the own inputs of the transaction under consideration, each with a label $\iota(a) \in [0, 1]$, the degree to which $a$ is already impervious to intersections.
  - The labels come from the [Max flow based counterfactual path estimation](cost_function_counterfactual_paths.md) chapter, and are taken as given here.
- **Links.** $\nu_0 = |I| - 1$.
- **Linked count.** $\nu = \sum_{a \in I} (1 - \iota(a)) - \max_{a \in I} (1 - \iota(a))$.
  - A single coin gives $\nu = 0$, and so does one coin that isn't impervious spent with ones that are.
- **Routes.** $q = 1 - \nu / \nu_0$, and $q = 1$ for a single coin.
- **Density.** $\eta = \min_{a \in I} f_a$.
  - $f_a$ is the size-corrected fraction of steps 3–6 of the [subset sum ambiguity](cost_function_subset_sum_ambiguity.md#definition) term, with inputs and outputs swapped, as for [inputs absorbed into outputs](cost_function_subset_sum_ambiguity.md#after-confirmation).
  - The targets are the non-empty $S \subseteq I$, with $r_S = \sum_{a \in S} e_a$, the sum of their effective values. The ways are the subsets of the other users' outputs summing into $[r_S - \varepsilon, r_S]$.
  - $p = 1 - \text{clamp}(\log_2(\varphi_c / \eta) / \theta_c, 0, 1)$, with $\varphi_c$ well above that term's $\varphi$.
- **Liability.** $\ell_c = (b^{\nu_0} - 1) \cdot (1 - p q)$, with $b > 1$ the factor per intersection.
  - **Thresholds.** $p = 1$ once $\eta \ge \varphi_c$. $q = 1$ once every input but one has $\iota = 1$, i.e. $K$ routes.
  - It's 0 only when both meet their thresholds.
  - The relief from each is proportional to the other: $\partial \ell_c / \partial p \propto -q$. With $q = 0$, density relieves nothing, and with $p = 0$, routes relieve nothing.

## In the decision tree

- It's a privacy-related term.
- The labels are fixed when the inputs are chosen.
- **Worst case:** $\eta$ counts only the other users' outputs known so far. Before any are known, $p = 0$ and the full penalty applies, whatever the routes.
- **Best case:** $\ell_c = 0$, which is used for pruning.
- $\ell_c \ge 0$ is unbounded, so it can exceed the remaining budget. The action is then taken only if other terms, e.g. a payment deadline, outweigh it.
- It applies to every own output of the transaction. The discount scales with value, not with the number of outputs.

## After confirmation

- $\eta$ is computed from the complete sets.
- $\ell_c \ge 0$ stays unbounded, and isn't capped at 1 once the action is taken.

## Evaluating the wallet state

- The term can evaluate a wallet state too. Imagine a unilateral transaction spending all of its coins right after, and take that transaction's consolidation penalty.
- That transaction's cost includes its objective costs, e.g. the fees for each coin, so this also prices a wallet split over many coins.
- It depends on the [open question](#open-questions) about transactions with only the wallet's own coins. Until that's settled, $p = 0$ there, and every state with more than one coin gets the full penalty.

## Monotonicity

- $\ell_c$ can only fall as $\eta$ rises, or as the labels rise.

## Open questions

- Should siblings also count towards $\nu$?
- A transaction with only the wallet's own coins has no other users' outputs, so $p = 0$ and the penalty can't be discharged. Which density applies there? For example, the [output overlap](cost_function_subset_sum_ambiguity.md#sumset-overlap) $u_o$ of the transactions that created the inputs.
- How should $p$ and $q$ be scaled?
- Should $p$ use the [input overlap](cost_function_subset_sum_ambiguity.md#sumset-overlap) $u_a$, instead of or as well as $f_a$? It's known during coalition formation, and measures how easily the inputs can be clustered.
