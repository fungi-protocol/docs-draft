---
type: concept
title: Cost function structure
status: draft
stale_after: 2026-11-06T00:00:00Z
---

# 🤖 Cost function structure

The evaluation of pure strategies is a family of cost functions, sharing the structure set out in the [Optimization framework](optimization_framework.md#preliminaries) chapter: a linear combination of terms, each labelled as privacy related or not, and each a piecewise linear function of time. This chapter describes that structure: how terms depend on time, how they are combined into a single amount in satoshis, and how incomplete information is handled by bracketing. The chapters that follow define the concrete terms.

## Time dependence

Every term of the cost function and every intent in the queue may depend on time, and how it does may depend on what is being evaluated. This time dependence is modeled as piecewise linear offsets to values computed independently of time, so the cost function returns a family of piecewise linear functions of time, one per term. Evaluating them at a given time is just a matter of indexing them and taking a sum, but treating them explicitly as piecewise linear functions makes it possible to solve for intercepts.

## Linear combination

The [global privacy budget](optimization_framework.md#global-privacy-budget), with its budget rate $p$ and remaining budget $B$, normalizes the privacy terms, so that all terms can be summed into a single amount in satoshis.

Each coin has a *privacy liability* $\lambda$. Concrete privacy terms each start from their own notions, for example quantifying the entropy of a coin's anonymity set and how close it is to some threshold, potentially non-linearly, and are normalized into a single ensemble liability. Coins received at external addresses start at the baseline $\lambda = 1$, since the wallet has no prior knowledge about them, and change outputs of unilateral payments are treated the same way. Their descendants may have a lower liability when they are produced by automated actions, reaching 0 when the coins are deemed fully private. $\lambda$ is stored per coin, but it is derived information.

A coin's *subjective value* is its nominal value $x$ discounted by $\lambda p x$, effectively devaluing non-private wallet states. This is what turns abstract thresholds on privacy metrics into a penalty in satoshis. It simplifies the cost function so it only outputs a satoshi amount, allowing subjective and objective costs to be reduced to a single number, and therefore comparisons can be made between the wallet states predicted to result from taking one action vs. another. Since the discount depends only on the coin itself, this applies regardless of how many inputs and outputs a transaction has.

A transaction spending coins with total discount $d_{in}$ and creating coins with total discount $d_{out}$ at an objective cost of $y$ changes the wallet's total subjective value by

$$(d_{in} - d_{out}) - y,$$

so on privacy grounds alone, an action is only worth taking if the reduction in discounts dominates its cost. Privacy-motivated costs must also fit within the remaining budget. Other terms, such as the waste metric or the utility of completing a payment, also drive the decision. However, the utility of a payment is not part of the subjective value of the resulting state: a payment's utility rising over time doesn't mean that waiting longer to make it would leave the wallet with a higher subjective balance.

For example, suppose Alice sets her budget rate to 1% and receives a coin of 100,100 sats, adding 1,001 sats to her remaining budget. The coin is not yet private, so its subjective value is 99,099 sats. The wallet considers CoinJoin transactions costing 100 sats in fees, each replacing it with a coin of 100,000 sats:

| | nominal | $\lambda$ | discount | subjective value | change |
|---|---|---|---|---|---|
| before | 100,100 | 1 | 1,001 | 99,099 | |
| large improvement | 100,000 | 0.5 | 500 | 99,500 | +401 |
| small improvement | 100,000 | 0.8 | 800 | 99,200 | +101 |
| not worth it | 100,000 | 0.95 | 950 | 99,050 | −49 |

Either of the first two outcomes is worth taking, and either leaves 901 sats in the remaining budget. The last reduces the discount by only 51 sats while costing 100, so the wallet would not make it.

The same accounting applies when coins are split or consolidated. Splitting the same coin into two outputs whose liabilities improve by different amounts:

| | nominal | $\lambda$ | discount | subjective value |
|---|---|---|---|---|
| input | 100,100 | 1 | 1,001 | 99,099 |
| output A | 60,000 | 0.5 | 300 | 59,700 |
| output B | 40,000 | 0.9 | 360 | 39,640 |
| outputs total | 100,000 | | 660 | 99,340 |

Discounts fall by 341, which dominates the fee, for a net change of +241. Consolidating two non-private coins of 60,000 and 40,100 sats (discounts of 600 and 401) into a single CoinJoin output of 100,000 sats with $\lambda = 0.5$ gives exactly the same result as the single coin, +401, since only the totals on each side matter.

When evaluating a candidate action, the cost function may assign $\lambda > 1$ to the resulting coins. For example, if the privacy metrics assess the output of consolidating the same two coins without a CoinJoin, at the same fee, at $\lambda = 1.2$, its discount is 1,200 and its subjective value 98,800, a change of −299 before any other terms. Such an increase counts against the action, which is only taken if other terms outweigh it, for example the utility of a payment with a hard deadline. Such states never materialize as the result of an automated action: once the action is taken, the resulting coins have their liability capped at the baseline of 1. The same applies if the user manually takes an action the wallet would not have taken: the loss it implies is written off as an external adjustment authorized by the user, reflecting a valuation the cost function fails to capture.

In general, a transaction which is worth taking on privacy grounds alone, spending coins of nominal value $x$ with total discount $d$ at a cost of $y$, results in coins whose subjective value is greater than $x - d$ and at most $x - y$ (when no liability remains).

## Bracketing

We model privacy liabilities as terms that only reduce the evaluation of a particular state, and can only be reduced as more information is revealed. This means that for every state we can derive a best case and worst case cost. The worst case is based on the information already available: it assumes that no other party contributes anything of value to the wallet, apart from the adjustments of any co-spend proposals in effect, which are guaranteed if the leaf materializes. The best case assumes they act optimally for the wallet, providing perfect cover, and is obtained by simply omitting any privacy related penalties from the sum. Its use is pruning: a leaf whose best case is still unacceptable can be discarded, as in branch and bound.

The most likely outcome lies in between, with other parties behaving the way they typically behave. An expected value for each leaf could in principle be estimated empirically, by comparing how a coalition was evaluated when the wallet committed to it with the actual wallet state, and the reduction in liabilities it generated, once it was finalized in a block. However, peers that consistently behave well could push the wallet towards optimism, and then exploit it in a coalition where they don't. How much risk to accept is up to the user, but this document only describes the pessimistic version, in which a leaf's expected value is its worst case. An expected value can always be introduced later as another term.

During the critical section of transaction construction, i.e. after the inputs are revealed and before signatures are posted, knowledge of the other users' inputs may improve the worst case bracket. Preparing the decision tree ahead of time and computing these brackets makes it possible to respond to this new information efficiently, finalizing the decision of which outputs to add to the transaction in order to obtain better outcomes.
