# 05 - Missions

A mission gives a developer or agent a checkable objective and a way to follow its progress. Atelier should express the objective as predicates over a particular revision of the solution, then use the [temporal ledger](04_temporal_ledger.md) to show how those predicates change over time. Measurements and human decisions can accompany those predicates with their own provenance.

Long-running changes require more context than a single test result provides. A mission brings the intended outcome, the supporting compiler facts, and the history of the work into one view. [SWE-EVO](https://arxiv.org/abs/2512.18470) examines related challenges across repository evolution tasks.

## A Mission Has an Objective and Evidence

A mission identifies the solution revision, the predicates to check, and the evidence needed to evaluate them. For example:

- **Target readiness:** every subgraph selected for an inference pipeline is reachable on an NPU. CCS supplies the per-target reachability facts for the selected configuration.
- **Verification:** the crypto module's designated obligations are discharged at the required tiers, including properties supported by a Rocq-checked rule library. The objective names the obligation families, applicable premises, proof artifacts, and verification status.
- **Contract migration:** both endpoints of every identified BAREWire crossing have adopted a chosen contract version. The predicate uses the published crossing inventory and resolved contract versions. A `ContractRevised` ledger event tracks an observed change.

The dashboard should show satisfied, failing, and unknown conditions. A missing compiler fact or unavailable proof is unknown, with a link to the missing evidence. A graph comparison can identify a changed predicate as soon as the relevant facts are available, but it does not replace tests or establish every behavioral property.

A latency objective illustrates the division of evidence. Structural facts may identify a transfer boundary or a candidate optimization. A benchmark supplies the measured latency, environment, and revision. Atelier can display both and evaluate a budget against the measurement. The [PHG paper](https://arxiv.org/abs/2603.17627) explains why program properties cannot all be decided from structure alone.

## The Capability-Arrival Advisor

A useful first application compares a solution built for a CPU with a candidate configuration that adds an FPGA or NPU. The advisor should help the developer find eligible work, understand the boundaries, and decide what to investigate. Its results depend on facts published by the compiler and on measurements where performance is at issue.

1. **Establish eligibility.** CCS settles target reachability for the relevant graph and configuration. The Live/Latent/Fresh and reachability design in the Clef PSG-to-PHG plan provides a foundation for retaining alternatives as targets change.
2. **Compare configurations.** CCS re-evaluates the same source against a different `PlatformDescription` set. The advisor compares the resulting published revisions. `TargetBitChanged` and `Reclassified` ledger events identify the underlying facts and their revision provenance.
3. **Explain the history where known.** CCS can record admissible lowerings and the factors behind a choice, allowing the ledger to preserve their changes. Where no earlier choice record exists, the advisor shows current eligibility and identifies historical intent as unknown.
4. **Describe the crossing.** A migration candidate creates boundaries whose types, layouts, and transfer requirements must be settled before a BAREWire contract can be generated or checked. The advisor shows the candidate cut and its supporting boundary facts, including any gaps in coverage.
5. **Support the decision.** Show the developer the evidence, unknowns, estimated costs, and measurements. Accepting a candidate can open a migration mission whose checks and proof obligations evolve with the implementation.

[HelloArty](https://github.com/FidelityFramework/HelloArty) and [HelloNappy](https://github.com/FidelityFramework/HelloNappy) illustrate the kind of target-sensitive inference that motivates this advisor. Its acceptance cases should demonstrate comparative re-evaluation, historical choice records and boundary contracts for selected targets. Determinism, completeness and type correctness are properties to establish for each supported fragment.

## Deterministic Checks and Learned Assistance

Build the mission engine around reproducible predicates and revision-bound evidence first. A learned assistant can later rank candidates or suggest questions, with its suggestions clearly identified and checked against the same facts. Research on temporal graph learning, including [TGB](https://arxiv.org/abs/2307.01026) and [T-GRAB](https://arxiv.org/abs/2507.10183), makes strong baselines and task-specific evaluation worthwhile. Model selection should follow measured usefulness on Atelier's own work.
