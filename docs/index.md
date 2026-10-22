---
okf_version: "0.2"
---

* [Collaborative transaction privacy](collaborative_txn_privacy.md) - Under review in #7.

# Protocol Suite

* [Market-based coalition formation](coalition_formation.md) - Under review in #18.

# Privacy Optimization Strategy

* [Cost function framework for Bitcoin wallets](optimization_framework.md) - Under review in #15.
* [🤖 Others' inputs subset sum density](subset_sum_density.md) - Outline of subset sum density, how it's estimated, and how co-spend proposals and the aggregator optimize it.
* [🤖 Own inputs or outputs subset sum ambiguity](cost_function_subset_sum_ambiguity.md) - Outline of a concrete cost function term for the subset sum ambiguity of output values.
* [🤖 Consolidation penalty](cost_function_consolidation.md) - Outline of a concrete cost function term penalizing consolidation unless the subset sums are very dense and the coins are impervious to intersections.
* [🤖 Sibling output penalty](cost_function_siblings.md) - Outline of a concrete cost function term penalizing spending sibling outputs together.

# BIP Drafts

* [PSBT Scrubber](bip-drafts/scrubber.md) - Which PSBT fields can be shared with trusted and untrusted parties such that privacy is preserved.
* [Concurrent PSBT Construction](psbt-concurrent.md) - Under review in #16.
