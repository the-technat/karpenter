# Multi-Node Consolidation

## Purpose

Single-node consolidation evaluates each node independently, asking whether one node can be removed or replaced with a cheaper alternative. This works well for individual inefficiencies but cannot recognize situations where removing several underutilized nodes together and replacing them with fewer, better-sized nodes would reduce cost. Multi-node consolidation fills this gap by evaluating groups of nodes as a unit, enabling moves that no single-node evaluation could discover.

The canonical example is a cluster with three nodes each running a handful of small pods. No single node's pods can be absorbed by the remaining two, so single-node consolidation does nothing. Multi-node consolidation can recognize that all three nodes' pods fit on one properly sized replacement, eliminating two nodes worth of cost. By handling these cases, multi-node consolidation captures savings that are structurally invisible to single-node evaluation.

## When It Runs

Multi-node consolidation runs as part of Karpenter's disruption reconciliation loop, which evaluates disruption methods in a fixed order. Emptiness, static drift, and drift are evaluated first. If any of those methods produce an action, the loop exits immediately and multi-node consolidation does not run for that cycle. If none of the earlier methods act, multi-node consolidation is evaluated. If it produces no action, control passes to single-node consolidation.

Multi-node consolidation is skipped entirely when the cluster's consolidation state has not changed since the last time it was marked consolidated. This prevents redundant scheduling simulations when nothing in the cluster has changed. If multi-node consolidation finds no action but some candidates were excluded due to disruption budgets, it does not mark itself as consolidated, because a future cycle with different budget availability may yield a valid consolidation.

## Candidate Selection

Candidates are nodes that Karpenter manages and that pass the disruption eligibility checks. A node must have a known instance type, capacity type, and availability zone. Its NodePool must have consolidation enabled with the `WhenEmptyOrUnderutilized` policy, and a non-nil `consolidateAfter` duration. The node must not be owned by a static NodePool (one with explicit replicas). The node must have its consolidatable status condition set to true. Nodes that are already in the disruption queue or that have pods blocking eviction (accounting for PDBs and do-not-disrupt annotations) are excluded.

Empty nodes -- those with no reschedulable pods -- are explicitly filtered out before multi-node evaluation. If an empty node has not already been handled by the emptiness method, it is assumed to be constrained by a disruption budget. Including it in multi-node consolidation could circumvent the operator's intent to limit empty-node disruptions. Only nodes carrying at least one reschedulable pod participate in multi-node consolidation.

## Candidate Ordering

Candidates are sorted by disruption cost in ascending order, placing the least costly nodes to disrupt first. Disruption cost is a composite value derived from the rescheduling cost of the pods on the node (which accounts for pod priority and pod count) multiplied by the fraction of the node's expected lifetime remaining under its NodePool's configuration.

This ordering is significant because the evaluation algorithm always considers contiguous prefixes of the sorted list. Placing the cheapest-to-disrupt nodes first means the algorithm preferentially consolidates nodes whose removal causes the least operational impact. Ties in disruption cost result in a stable but arbitrary ordering among the tied candidates.

## Move Evaluation

Multi-node consolidation requires at least two candidates. It evaluates contiguous prefixes of the sorted candidate list using a binary search: it starts by testing the group of all eligible candidates, then narrows or widens the prefix based on whether each attempt yields a valid consolidation. The goal is to find the largest prefix that can be consolidated in a single move.

For each prefix under consideration, a scheduling simulation removes all candidates in the prefix from the cluster model and attempts to place their pods on the remaining nodes. A valid outcome is either a pure deletion (all pods fit on existing nodes with no new node required) or a replacement (pods require exactly one new node). If the simulation requires two or more new nodes, the prefix is too large and the search narrows. If it succeeds, the search widens to try including more candidates.

The binary search operates under a one-minute timeout. If the timeout expires before the search completes, the algorithm returns the last valid consolidation command it found, if any. If no valid command has been found by timeout, the method returns no action. The candidate set is also capped at 100 nodes to bound computation.

## Cost Comparison

A replacement is only valid if the new node is strictly cheaper than the combined cost of the candidates it replaces. The cost of each candidate is derived from its current instance type's cheapest compatible offering in its zone and capacity type. The cost of the replacement is the cheapest offering among its eligible instance types.

An additional filter addresses the case where the replacement's eligible instance types overlap with the instance types being removed. If the replacement could be launched as the same instance type as one of the candidates, that would not reduce the number of nodes -- it would be equivalent to just deleting the other candidates. To prevent this, when an instance type appears in both the candidate set and the replacement options, the filter removes that instance type and any instance type at or above its price from the replacement options. If no replacement options remain after filtering, the move is rejected.

## Interaction with Single-Node

Multi-node consolidation always runs before single-node consolidation. If multi-node produces a command, single-node does not run in that disruption cycle. This ordering exists because multi-node moves are strictly more efficient when they apply: consolidating N nodes into one move causes less pod churn than N sequential single-node moves.

Single-node consolidation catches cases that multi-node structurally cannot. Because multi-node only evaluates contiguous prefixes of the sorted candidate list, a node in the middle or tail of the list that could be individually consolidated may not appear in any winning prefix. Single-node also handles spot-to-spot consolidation with its minimum instance type flexibility requirement (ensuring at least 15 cheaper instance type alternatives to avoid consolidation loops), which multi-node does not enforce. Finally, single-node consolidation can act when fewer than two candidates are eligible, a situation where multi-node cannot operate.

## Interaction with Disruption Budgets

Before multi-node consolidation begins its evaluation, candidates are filtered against the disruption budget for each NodePool. Candidates are processed in disruption-cost order. For each candidate, if its NodePool's budget still has remaining capacity, the candidate is included and the budget is decremented. If the budget is exhausted, the candidate is skipped. This means the disruption budget acts as a per-NodePool cap on how many nodes from that pool can participate in a single multi-node consolidation.

Because candidates are processed in order, budget exhaustion removes candidates from the tail of the eligible list (the most expensive to disrupt). If any candidate was skipped due to budget constraints, multi-node consolidation does not mark itself as consolidated even if it finds no valid action, because the next evaluation cycle may have different budget availability.

## Known Limitations

The most significant limitation is the prefix-only evaluation strategy. Multi-node consolidation only considers contiguous prefixes of the sorted candidate list, starting from the lowest-disruption-cost node. It cannot discover consolidation opportunities that exist among arbitrary subsets of candidates. For example, if nodes 3, 5, and 7 in the sorted list could be consolidated together but no prefix containing all three also forms a valid consolidation, that opportunity is missed entirely. This is a known trade-off favoring computational tractability over exhaustive search.

The single-replacement constraint means multi-node consolidation can only produce moves where N candidates are replaced by zero or one new nodes. Valid consolidations that would require two replacement nodes (for example, consolidating five nodes into two) are not considered. This simplifies cost comparison and avoids combinatorial complexity in replacement evaluation but leaves some savings on the table.

The 100-candidate cap means that in very large clusters, only the 100 cheapest-to-disrupt nodes are considered. Consolidation opportunities among higher-cost nodes beyond this cap are invisible to multi-node evaluation. The one-minute timeout can also cause the algorithm to return a suboptimal result or no result at all in clusters with many candidates and expensive scheduling simulations.

Spot-to-spot consolidation in multi-node mode does not enforce the minimum instance type flexibility requirement that single-node consolidation uses. This means multi-node spot-to-spot replacements could theoretically result in consolidation loops if the launched instance type is not among the cheapest options, though in practice the multi-node case is less susceptible because it consolidates multiple nodes simultaneously.

## Steady-State Behavior

Multi-node consolidation maintains a consolidation cache based on the cluster's consolidation state timestamp. After a full evaluation that finds no valid action and was not constrained by budgets, the method records the current cluster state. Subsequent disruption cycles skip multi-node evaluation entirely as long as the cluster state has not changed. Any change to the cluster -- a node being added, removed, or modified; a pod being scheduled or evicted; a NodePool being updated -- advances the consolidation state and causes multi-node consolidation to re-evaluate on the next cycle.

This caching is critical for steady-state performance. In a well-consolidated cluster, multi-node evaluation is the most expensive disruption method due to its scheduling simulations. Without the cache, every 10-second disruption cycle would re-run these simulations for no benefit. The cache ensures that multi-node consolidation imposes near-zero cost when the cluster is stable, and only re-engages when there is a material change that could create a new consolidation opportunity.
