# 11 - Native Reflection: The Semantic Graph as Metadata System

> **Status:** Preliminary design. This document establishes the directional intent for Fidelity's answer to reflection. It was prompted by the adoption of compile-time static reflection into C++26 ([P2996](https://isocpp.org/files/papers/P2996R13.html)), which independently validates an architectural bet the Fidelity toolchain already made. Detailed API design will be authored as the CCS observation surface, and the resident compiler process that would host it, land; both are design-stage.

## What This Document Is

"Native reflection" in Fidelity proposes an observation surface over existing compiler machinery: the Program Semantic Graph (PSG), quotations as typed semantic carriers, active patterns as classification, and the Huet-style zipper as the traversal cursor. A resident compiler service would retain versioned graph snapshots between keystrokes. Composer's [interactive compiler workbench plan](../../Composer/docs/Interactive_Compiler_Workbench.md) owns that service and the native execution bridge; both remain planned. CCS owns the graph inside whatever process hosts it. A reflection API and metaprogram execution require their own contracts.

One framing matters enough to fix before anything else: **reflection here is not a query engine.** "Query" stands to reflection as "emit" stands to lowering — an imperative verb imported from someone else's architecture, dragging that architecture's execution model behind it. Composer does not emit; it witnesses, and the code is the residual. Nothing here queries; consumers observe, and everything is pull: coeffects are placed, codata is observed. C++26 ships reflection as a flat list of functions to call; Fidelity's reflection is the observation surface of the same traversal that compiles the program, made available to consumers other than the code generator.

## The Question Reflection Actually Asks

Runtime reflection — `System.Reflection`, `java.lang.reflect` — is a request for information the compiler possessed and threw away. The compiler resolved every type, computed every layout, knew every member; then it discarded most of that knowledge and emitted a partial reconstruction into metadata tables so the program could ask diminished versions of the same questions later, at runtime cost, with runtime failure modes.

This is the same displacement pattern described in [ByRef Resolved](https://github.com/speakeztech/clef-lang-site/blob/main/hugo/content/docs/design/types/byref-resolved.md) for memory safety: when a capability lives in the wrong place, the host's limitations become the language's limitations. Runtime reflection lives in the wrong place. The right place is the compiler's own semantic state — kept alive, enriched, and observable.

## What C++26 Conceded

P2996 brings compile-time static reflection to C++26. Stripped to its core, it is three primitives forming a closed loop, confined to constant evaluation:

| Primitive | C++26 form | What it does |
|:----------|:-----------|:-------------|
| **Lift** | `^^entity` | Produces a `std::meta::info` handle to a language entity |
| **Query** | `std::meta::*` consteval functions | Interrogates the handle: identity, structure, layout, predicates |
| **Splice** | `[: r :]` | Injects the denoted entity back into the program |

Three design decisions matter for Fidelity:

1. **Reflections are handles into compiler-internal semantic state.** Not a metadata format, not a runtime table — an opaque pointer into the compiler's own data structures, valid only during translation. Zero runtime overhead is definitional: reflections cannot survive translation unless explicitly reified into static data.

2. **`info` is a single opaque type.** The committee rejected a strongly-typed reflection class hierarchy, reasoning that if C++03 had standardized a `std::meta::variable` type, C++11's references would have broken it. Evolvability won; static typing of the reflection surface lost.

3. **Synthesis is structured or deferred.** `define_aggregate` completes a type from a member description list — synthesis by description. Arbitrary token injection (P3294) was explicitly deferred to a future standard because it is unstructured.

C++26 is the mainstream's admission that reflection done right is an API over the compiler's live semantic state. But C++ ships the idea fractured by forty years of batch-compilation constraints — and shaped as a query API, because in a batch compiler there is no walk for a consumer to join.

## The Asymmetry Nobody Closed

Every mainstream ecosystem gives programs and tools **different doors** into semantic information:

| Ecosystem | Programs use | Tools use | Same state? |
|:----------|:-------------|:----------|:------------|
| C# / .NET | `System.Reflection` (runtime tables) | Roslyn (compiler API) | No — two metadata models |
| F# / .NET | Quotations + runtime reflection | FSharp.Compiler.Service symbols | No — split by phase |
| C++26 | `std::meta` (inside the TU being compiled) | libclang / Clang AST | No — the standard API is unavailable to tools |
| **Fidelity (proposed shared service)** | **PSG observation surface** | **PSG observation surface** | **Intended: versioned observations of the same compiler-owned graph** |

The C++26 irony is sharpest: programs finally get the compiler-state API, but clangd and clang-tidy still enter through a different one. FCS had the inverse problem for a decade: tooling got a superb compiler-state API (`FSharpEntity`, `FSharpSymbol` — recognizably structure and identity observations avant la lettre), but programs could not observe themselves.

The proposed shared service would let these consumers observe the same compiler-owned snapshots. Keeping a process alive supplies the hosting opportunity; snapshot identity, invalidation and legal observations still need explicit contracts.

## One Mechanism, Four Residuals

Composer's workbench plan starts with a possible SageFS/.NET bootstrap host and a shared versioned compiler/proof service, then adds LLVM ORC execution. It does not select the topology previously sketched here: a Prospero supervisor, separate PSG/LSP/proof actors, BAREWire shared memory and a lazy evaluation worker. Those remain proposals for a later host, not prerequisites to the bootstrap workbench. Full native self-hosting is also separate. CCS/Baker own graph construction, elaboration, saturation and semantic facts. Alex observes those facts through ctx pull and witnesses the admitted MLIR; no host or reflection consumer may supply missing semantics.

Reflection is then not an API bolted onto that state. It is the compilation mechanism itself, serving consumers beyond the code generator. The Huet-style zipper is the attention mechanism: something directs its focus; the joint constraints already resolved into the graph meet at that focus; a witness observes them; and the residual is elided. What differs per consumer is only *what directs attention* and *what the residual is*:

| Consumer | What directs attention | Residual of observation |
|:---------|:-----------------------|:------------------------|
| Alex traversal | the compilation walk itself | MLIR |
| Metaprogram actor (proposal) | the metaprogram's own navigation | compile-time values; synthesized PSG structure |
| Lattice service | the developer's cursor and edits | hovers, diagnostics, code lenses |
| Atelier panes | the viewport | rendered views: PSG pane, layout maps, escape diagnostics |

```mermaid
flowchart TB
    subgraph host["Resident compiler process (proposed)"]
        psg["CCS: the PSG<br/>(live graph, saturated:<br/>types · SRTP · affinity · residence · obligations · layout)"]
        zipper[["Zipper focus<br/>(attention) → witnesses observe"]]
        psg --- zipper
    end

    zipper --> alex["Alex walk<br/>residual: MLIR"]
    zipper --> meta["Metaprogram actors (proposal)<br/>residual: values,<br/>synthesized structure"]
    zipper --> lsp["Lattice service<br/>residual: hovers,<br/>diagnostics"]
    zipper --> atelier["Atelier panes<br/>residual: rendered views"]
```

Design-time tooling is another elision target. A hover card is the residual of the same observation that, on the compilation walk, would have become an MLIR op. Nothing is computed on demand for the editor's benefit; the codata principle holds across all four columns: the enrichments were placed at saturation (mise-en-place), and every consumer *observes*.

Two requirements connect this proposal to the workbench:

- **Attention can prioritize work; it cannot establish freshness.** Cursor and viewport focus can guide scheduling. Compiler-owned dependencies, source/target/compiler generations and proof-query identity determine whether observations remain valid. A warm graph or focused node does not establish incremental recomputation. The [proof responsiveness contract](../../Composer/docs/Interactive_Compiler_Workbench.md#proof-responsiveness) applies equally to Atelier, Lattice and agent consumers.
- **Clef execution would be native.** The selected interactive name is `clefx`, matching `.clefx` script files. Composer plans the CCS/Baker → PSG → Alex → admitted MLIR → LLVM/ORC path, including while the compiler itself remains .NET-hosted. FSI may host that compiler implementation; it cannot evaluate Clef in place of the native path. This does not settle whether user metaprograms execute during elaboration. Quotation metaprogramming requires no runtime evaluation; any synthesized graph structure must enter through Baker's recipe channel.

## The Triad Is Already Two-Thirds Owned

Mapping P2996's primitives onto Clef's standing art:

| P2996 | Clef | Status |
|:------|:-----|:-------|
| Lift: `^^entity` → `info` | Quotations: `<@ expr @>` → `Expr<'T>` | **Exists — and typed**, where `info` is monotyped |
| Splice: `[: r :]` | Quotation splicing at the language level; **witness elision** at the compiler level | **Exists** — injection is structured by construction |
| The semantic handle: `info` denotes a *resolved entity* | A reference into the PSG — a place the zipper can focus | **The missing third piece** |

Quotations carry *syntax*; P2996 handles denote *resolved semantics*. Native reflection extends the quotation machinery from syntax carriers to semantic handles — references into the saturated PSG, where symbol resolution, type resolution, SRTP witness selection, coeffect annotation, and escape classification have already happened. A handle is not a ticket for asking questions; it is a place to stand. Everything downstream of it is observation at a focus.

## Active Patterns: Classification Without Commitment

P2996's monotype compromise is a real dilemma honestly resolved: a typed reflection hierarchy freezes the language's ontology into a standardized API; an opaque handle preserves evolvability but surrenders static typing of the reflection surface.

Clef dissolves the dilemma with a feature C++ does not have. A single opaque PSG handle, classified through **active patterns**, yields typed views without a frozen hierarchy:

```fsharp
// Illustrative sketch — names provisional
let describe (t: NodeRef) =
    match t with
    | RecordShape fields ->
        fields |> Seq.map (fun f -> $"{f.Name}: {display f.FieldType}")
    | UnionShape cases ->
        cases |> Seq.map (fun c -> $"| {c.Name}")
    | FunctionShape (domain, range) ->
        Seq.singleton $"{display domain} -> {display range}"
    | Opaque ->
        Seq.singleton "no structural view available"
```

The handle stays open — new node kinds, new classifications, new pattern banks can be added without breaking any existing match site, exactly the evolvability the C++ committee bought with `info`'s opacity. But every arm binds *typed* results, exactly the static discipline they gave up. Partial active patterns compose into total views; parameterized active patterns admit platform- or phase-conditioned classification.

This is also not a new idiom invented for the occasion. It is the same mechanism the compiler uses internally: Alex's semantic lenses are active patterns classifying PSG nodes during traversal, and the codata principle — *witnesses observe and return, never build or compute* — is precisely the contract an observation must satisfy. Promoting active patterns to the user-facing reflection surface means metaprograms, the LSP, and the compiler's own internals speak one idiom over one graph.

Of the three Don Syme designs that anchor the Fidelity architecture, two already had their defining role: quotations as semantic carriers (the Fidelity.Platform descriptor spec), computation expressions with `MailboxProcessor` as sugared delimited continuations (the concurrency model). Native reflection is where active patterns take the same rank: **classification without commitment is the reflection story.** And a fourth piece of standing art joins them from a different lineage — Huet's zipper (1997), thirty years a data-structure curiosity, here the attention mechanism of a graph compiler.

## The Vocabulary and the Semantic Projections

Two terms carry this design, and they name different things.

The **vocabulary** is the contract: the enumerable, versioned set of observation signatures a focus supports. Its edition travels with CCS — one can point at a CCS version and say precisely which observations exist and no others.

The **Semantic Projections** are proposed compiler-owned views of versioned graph facts. Their incremental maintenance must follow the current [PSG/PHG plan](../../clef/docs/fidelity/phg/PSG_to_PHG_Plan.md) and [incremental provenance contract](../../Composer/docs/Nanopass_Incremental_Contract_Direction.md). This document does not establish a completed incremental driver, one universal settlement lattice or a latency guarantee. A consumer observes the compiler's facts; retaining a derived view does not give that consumer semantic authority or its own invalidation policy.

CCS already owns range facts and escape codata. Reusing a projection after an edit additionally requires an explicit dependency contract: source/type relationships, captures, selected target, compiler generation and proof premises can all affect validity. A width-only change may preserve another answer only when the compiler establishes that answer's independence. The observation groups below describe desired views; they are not implemented incremental partitions.

On both the batch walk and the proposed interactive path, owning compiler passes establish the facts before a consumer relies on them. A request may schedule compiler work without moving that work's semantic authority into the consumer. Observation, transport and formatting still have costs to measure. Baker's settlement and Alex's passive witnessing remain the integrity boundary regardless of scheduling policy.

Structural dependency information can eventually reduce the work invalidated by an edit. Until that reuse is established, conservative invalidation remains valid. Attention can prioritize pending checks and solver jobs, while source/target/compiler identity and exact proof-query correspondence determine freshness. Self-adjusting computation and demand-driven memoization are research precedents; neither is selected by this reflection proposal.

Seed the vocabulary from P2996's battle-tested taxonomy rather than inventing one — every question their function list answers is an observation this surface must support — then extend it with what only this compiler's enrichments contain.

**Inherited shape (P2996 parity):**

| Group | Representative observations |
|:------|:----------------------------|
| Identity | name, source location, enclosing scope, declaring module |
| Structure | record fields, union cases, function signature, members, type parameters |
| Layout | size, alignment, field offsets |
| Predicates | is-function, is-record, is-mutable, is-inline, is-complete |

**Fidelity-only extensions (beyond any C++ capability):**

| Group | Representative observations | Why C++ cannot have it |
|:------|:----------------------------|:-----------------------|
| Coeffects | `ArenaAffinity`, `Resides` and `Constrains` edges, obligation anchors at the focus | No coeffect system |
| Memory | compiler-owned escape classification (`Graph.Codata.Escapes`), arena affinity, residence in a declared space | No escape analysis surface |
| Resolution | SRTP witness selection, active dispatch targets | SRTP does not exist |
| Dimensional | units-of-measure carried by a type, dimensional constraints | No dimensional types |
| Platform | **platform-conditioned layout** — offsets under NTU deferral, indexed by platform context | C++ layout queries only answer post-concretization |
| Graph | reachability, def-use edges, nanopass provenance of an enrichment | No persistent semantic graph |

The platform row deserves emphasis. Because the Native Type Universe defers widths to platform context, a layout observation in Clef is not a scalar — it is a function of the compilation target. Reflection over a deferred type algebra is a capability class C++26 cannot express, and it is exactly what Atelier's cross-target views (FPGA vs. CPU realizations of the same source) need to render. The carrier is the layout hyperedge's per-target map (`Map<TargetId, LayoutAnnotation>` in `Layout_As_Joint_Constraint.md`), pending in CCS.

## Synthesis and Reification Discipline

P2996 validated a line Fidelity already draws: **synthesis by description is in; token pasting is out.** `define_aggregate` made C++26; arbitrary token injection did not.

The Clef analog of generative reflection is derive-style synthesis through the existing structured channels:

- **BAREWire schema derivation** — a record's wire schema generated from its PSG structure, replacing runtime serializer reflection
- **`%A` formatting, equality, comparison** — the classic `System.Reflection` consumers in .NET, regenerated as compile-time SRTP/witness synthesis (the obj-elimination table's "compile-time SRTP" column, given its general mechanism)
- **Platform descriptor expansion** — quotation-carried constraints elaborated against observed structure

These are intended applications of structured synthesis. Baker must elaborate and saturate any synthesized PSG structure before Alex observes it. Alex's witnesses do not synthesize source semantics, and this proposal does not establish an implemented general derive API.

**Reification** follows the same rule as C++: nothing survives to runtime unless explicitly frozen. Where P3491 adds `define_static_string`/`define_static_object` to give reflection results a runtime address, Composer already places program-lifetime values in static storage via `memref.global`. An observation whose result the program keeps becomes a static constant elided by a witness; an observation consumed entirely at compile time leaves no residue. "What you don't use, you don't pay for" holds by construction.

## Annotations

P3394 lets C++ attach user-defined compile-time metadata to declarations and query it through reflection. Clef is already there in substance: attributes and quotation-carried constraints (memory regions, peripheral descriptors, alignment demands) ride the PSG as first-class semantic information. The design task is only to make them uniformly observable at a focus — one annotation group in the vocabulary, not a parallel mechanism.

## Delivery Surfaces

The four consumers share the vocabulary; they differ in transport and in how attention is directed:

1. **Metaprogram actors** (proposal, contingent on `clefx`): full vocabulary, in-process BAREWire, compile-time only. This would be Clef's `consteval`: a metaprogram is an actor that navigates its own focus through the graph and observes, with its lifetime bounded by elaboration; anything it synthesizes enters the graph only through the Baker recipe channel.
2. **Lattice service**: the design-time subset serialized over LSP as `clef/*` extension methods. The general reflection methods (`clef/memoryLayout`, `clef/srtpResolution`, `clef/platformBindings`, and successors) remain proposals. Each would be a versioned focus-and-observe pair: the request identifies a source position or node in a checked snapshot, and the response presents compiler-owned facts. The earlier `clef/runVerification` sketch in [08 - Tooling Integration](./08_tooling_integration.md) is not the current proof protocol; [Composer's Lattice integration](../../Composer/docs/Lattice_Integration.md) owns the implemented surface and its acceptance gates.
3. **Atelier panes**: direct BAREWire views over shared memory, no LSP serialization. The PSG visualizer is, in this framing, simply *observation made visible*: the D3 pane renders the same focus a metaprogram would stand at. Hover cards, cache-line maps, escape diagnostics, and cross-target comparisons are all residuals of observation. With a remote host, the same views would arrive over BAREWire-over-WebSocket unchanged. This needs a BAREWire-described graph export from CCS; today the only export is the JSON phase artifacts, so the export is a CCS contract item (`Lattice_Consumer_Contract.md` §3).
4. **Alex itself** — the first and load-bearing consumer. Nothing in this document adds machinery to the compilation walk; the point is that the walk's machinery already is the reflection system.

Transcribe ([10 - Transcribe](./10_transcribe.md)) composes naturally: ingested bindings join the PSG and therefore support the same observations — a focus on a TypeScript-derived or C-derived binding is indistinguishable from a focus on hand-written Clef.

## What We Deliberately Do Not Copy

| C++26 constraint | Fidelity position |
|:-----------------|:------------------|
| Reflection as a query API (a list of functions to call) | Reflection as observation at a focus on a live walk |
| Translation-unit scope | The compiler owns the program graph; persistence and sound reuse across edits belong to the planned shared service |
| Constant-evaluator interpretation | Native metaprogram execution is a separate proposal, contingent on the planned `clefx` path and an elaboration-phase execution contract |
| Monotyped `info` with a flat function list | Opaque handle + active-pattern classification: typed views, open ontology |
| Vocabulary frozen by the standard | Vocabulary versioned with CCS; there is exactly one compiler, so compiler-specific semantics are exposable |
| Programs-only access (tools use libclang) | One observation surface for metaprograms, LSP, Atelier, and Alex itself |

## Answering "What About Reflection?"

The wrong answer is "Clef doesn't need reflection" — it concedes that reflection is a runtime facility one either needs or forgoes, and invites the follow-up "so how do you serialize?"

The grounded answer is that CCS retains semantic information in the PSG, and this design proposes exposing it through compiler-owned observations and structured Baker synthesis. Serializers, formatters, comparers, dependency wiring and schema generation are intended applications. Their general APIs, complete semantic coverage and session persistence are not established by this document or by a future JIT alone. Required facts must be settled before use; unresolved observations must remain explicit.

## Open Questions

- **Naming.** The user-facing handle type (`NodeRef` above is a placeholder), and whether the vocabulary surfaces primarily as a module of active patterns, a module of functions, or both.
- **Phase discipline.** Which observations are legal at which elaboration phases — a layout observation against a not-yet-saturated graph must fail honestly (`TRError`-style), not answer approximately.
- **LSP subset.** Which observation groups serialize over `clef/*` versus remain daemon-local; where the line sits between Lattice-served editors and Atelier's direct views.
- **Vocabulary versioning.** The vocabulary is a public contract of CCS; how its editions are versioned against the language and against saved PSG snapshots.
- **Projection slices and eagerness.** CCS already owns ranges and escape codata; sound dependency-directed reuse across edits remains work. The compiler must establish which changed inputs invalidate each fact and its proof evidence before attention-based scheduling can reuse it. See the [incremental contract direction](../../Composer/docs/Nanopass_Incremental_Contract_Direction.md).
- **Metaprogram authority.** A metaprogram's focus can wander the whole program's graph; whether library-published metaprograms need a capability gate on where they may stand is unresolved.
- **Metaprogram execution.** Whether user metaprograms execute during elaboration, and under what authority and lifetime rules. Composer owns the planned `clefx` native bridge, but its delivery does not answer this question. Quotation metaprogramming requires no runtime evaluation; metaprogram actors remain a proposal and synthesis stays on the Baker recipe channel.

## Related Documents

- [Interactive compiler workbench](../../Composer/docs/Interactive_Compiler_Workbench.md): hosting, compiler integrity, native execution and proof responsiveness; all milestones remain planned
- [08 - Tooling Integration](./08_tooling_integration.md): the `clef/runVerification` sketch this surface generalizes
- [10 - Transcribe](./10_transcribe.md) — ingested bindings as observable citizens of the graph
- [CCS Architecture](https://github.com/FidelityFramework/Composer/blob/main/docs/CCS_Architecture.md) — "PSG as Design-Time Resource," the contract this document builds on
- [ByRef Resolved](https://github.com/speakeztech/clef-lang-site/blob/main/hugo/content/docs/design/types/byref-resolved.md) — the displacement argument, applied to memory safety
- [Opining Upon Reflection](https://github.com/speakeztech/clef-lang-site/blob/main/hugo/content/blog/opining-upon-reflection.md) — the public essay form of this document's position
- [Standing Art: Clef Metaprogramming](https://github.com/speakeztech/clef-lang-site/blob/main/hugo/content/docs/design/language/standing-art-clef-metaprogramming.md) — quotations, active patterns, computation expressions as the architectural backbone
- Gérard Huet, "The Zipper" (JFP, 1997) — the functional pearl behind the traversal cursor
- [P2996R13 — Reflection for C++26](https://isocpp.org/files/papers/P2996R13.html) and [bloomberg/clang-p2996](https://github.com/bloomberg/clang-p2996) — the external validation
