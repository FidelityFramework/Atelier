# 04 - The Temporal Ledger

A graph revision shows the relationships that hold for a particular compilation. The **temporal ledger** connects revisions so that Atelier can explain how a program, its obligations and its target realizations develop over time. It gives a developer a way to follow a design decision, investigate a regression, and return to the premises behind an earlier result.

The ledger is an Atelier knowledge-layer projection. CCS and Baker establish source semantics and the facts published in each `Fidelity.PSG` revision. The ledger records observations and comparisons over those authorized facts, together with their provenance. It can identify a change to a published obligation or its premises; the compiler and proof-checking pipeline determine what the new state means.

## From one revision to the next

Each ledger entry identifies the source and compiler revisions, the checked graph revision and scope, the platform and target configuration, and the evidence used for the comparison. It records the operation that led to the observation where that history is available, including uncommitted working states when the preservation system can identify them. A build that fails still has a history: retain its diagnostics and the last eligible checked revision without presenting the failed build as a successful graph publication.

A sequence can begin with a complete authorized snapshot of the selected scope. Later entries can carry source-authorized additions, replacements, withdrawals and retirements against the exact revision they update. An attachment need only include the demanded live scope and the facts needed to interpret it. The [indexed revision architecture](../../../Fidelity.PSG/docs/Indexed_Revision_Architecture.md) describes the underlying delivery direction. The ledger retains the identities, support and scope needed to explain its own history while respecting the source's authorization and dependency boundaries.

The compiler revision, project configuration and scope are part of an observation's meaning. The same source may elaborate differently under a changed platform description or compiler version. Equally, a text edit can change source locations while leaving an admitted semantic judgment unchanged. Atelier should be able to show both the source-history change and the narrower semantic comparison.

## A worked sequence

Consider a source operation that reads from a declared buffer. At revision R1, the graph contains a buffer capacity, an input bound and an obligation connecting them. The proof record identifies the claim, its premises, its checking result and the scope to which that result applies.

The platform description changes at R2. The source operation is still present, but its capacity premise has changed. The ledger records the changed declaration and the affected obligation as related events. If the prior evidence is no longer eligible, the UI shows that withdrawal and follows the replacement result when it is available. Unrelated facts can remain associated with their established evidence when source analysis authorizes their reuse.

An engineer can ask: “Why did this proof become pending?” The answer links the current obligation to the changed capacity premise, the previous and current revisions, and the applicable source and platform locations. A similar question about a measured slowdown can bring in a benchmark result with its artifact and environment, separately identified as an observation. Neither the chart nor a historical graph comparison turns an unmeasured performance hypothesis into a measurement.

## Entries and comparisons

The ledger can express different kinds of change through a shared event record: the subject and its scope, before and after observations, their revision and source references, the reason or supporting correspondence, and the event's provenance. The following names are a design vocabulary for [missions](05_missions.md), with each event grounded in the source facts that support it:

| Event | What it records | Required basis |
|---|---|---|
| **Added / Withdrawn** | A fact or obligation enters or leaves the authorized scope. | The revisions and their scope/delivery contracts. |
| **EvidenceChanged** | An obligation or a premise changes, or its result is established, invalidated or replaced. | The claim, evidence status, dependencies and checking provenance. |
| **Reclassified** | A recorded category, placement or admissible choice changes. | The corresponding source-owned classifications and a valid subject correspondence. |
| **TargetBitChanged** | Reachability or eligibility for a named target changes. | An admitted per-target representation and an identified target. |
| **ContractRevised** | A declared boundary contract or version changes. | Both contract identities and the affected endpoints or uses. |
| **SourceChanged** | A source state, location or editorial provenance changes. | The preservation system's source-state and operation identity. |
| **MeasurementAdded** | A performance or runtime observation is associated with a build and environment. | The artifact, workload, measurement procedure and recorded result. |

The event vocabulary lets Atelier answer useful questions without forcing every change into one category. A change in source position, a changed inference result and a changed benchmark result have different causes and warrant. If a required correspondence or source fact is unavailable, the ledger can retain both observations and report the comparison as unresolved. It should not match unrelated live instances merely because their values or printed names happen to agree.

## Following decisions through time

A useful question for heterogeneous development is: “Was this computation shaped by an inherent dependency, or by the targets available at the time?” The ledger can preserve the answer when the compiler records the admissible alternatives, relevant constraints and settled choice at the decision point. Later, a changed platform description can motivate a new source-owned evaluation, and Atelier can compare the two results and show the supporting factors.

This **counterfactual history** is a design goal for the [capability-arrival advisor](05_missions.md). An earlier graph can establish what was recorded, and a later what-if analysis can establish what is admitted under a changed configuration. The ledger connects those results. It should distinguish a documented earlier choice from a retrospective hypothesis about an intent that was never recorded.

## Identity, correction and provenance

Within a revision, use the graph's published identities and their declared scope. Across revisions, use an admitted source correspondence or explicitly identify the basis for a proposed match. Symbol paths, source ranges and hashes can help locate candidates, but each has change and ambiguity cases. [The preservation interface](07_preservation_interface.md) develops those choices.

Keep provenance with each comparison: the participating revisions, projection, matching rule, producer and evidence status. A user-authored association, an automatic candidate and a compiler-authorized correspondence are different records. Later correction should preserve the earlier record's history and make the corrected interpretation available; the source system's retention policy determines the extent of that audit trail.

A reverse view can reconstruct or compare earlier recorded states when the retained snapshots and change information are sufficient. Reversing a display change is not by itself a proof that a source edit, foreign effect or running computation has an inverse. The ledger should make the actual preservation and replay guarantees of its selected substrate explicit.

## Interface to missions and the editor

The [mission layer](05_missions.md) names desired conditions and checks them against selected revisions and evidence. The ledger supplies the trajectory: when a predicate began to hold, which supporting facts changed, and which unresolved or external criteria remain. Atelier can show a timeline, an affected-scope view or an answer with source links. A developer can follow the evidence back to the relevant graph and documentation through the [weave](03_weave.md).

The first end-to-end case should retain two compatible scoped revisions, one changed platform premise and its affected obligation, and a separately recorded outcome such as a benchmark. Exercise source movement, an evidence withdrawal, a mismatched update base and an ambiguous identity. The result should let a reader distinguish what the compiler established, what the ledger compared and what an external measurement observed.
