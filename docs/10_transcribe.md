# 10 - Transcribe: Polyglot Library Ingestion

> **Status:** Placeholder. This document establishes the directional intent for the Transcribe feature. Detailed design and implementation specifics will be authored as the feature lands.
>
> **Naming.** Composer's JavaScript-targeting docs use the same name for a different feature: source absorption, lifting a JavaScript implementation into Clef source (`javascript-targeting/05_supply_chain_and_transcribe.md`, also followed by Fidelity.CloudEdge). This document and the clef-lang-site Xantham page use it for binding ingestion. Which feature keeps the name is not settled here; until it is, read "Transcribe" in this document as the binding-ingestion feature.

## What Transcribe Is

Transcribe is Atelier's developer-facing IDE feature for ingesting external libraries from other languages into a Clef project. The developer points Transcribe at a library — a TypeScript `.d.ts` file, a Python module with type hints, a Rust crate's public API, a Go package, a C/C++ header, an F# assembly — and Transcribe produces Clef binding declarations that join the project's graph as first-class typed citizens, carrying the binding-strategy annotations Alex's rule library reads when it witnesses them.

The defining property of Transcribe is **language-agnosticism**. The same IDE workflow handles every supported source language. The bindings the developer ends up with look the same in their Clef code regardless of whether the underlying library is TypeScript, Rust, or Go. Alex's rule library witnesses each declaration into the portable form (func, scf, arith, memref, index); the pathway realizes it for the target (JSIR for V8 isolate targets; LLVM, CIRCT, or MLIR-AIE for native targets).

## Supported Source Languages

| Source language | Analysis substrate | Status |
|:----------------|:-------------------|:-------|
| TypeScript / JavaScript (`.d.ts`) | [Xantham](https://github.com/shayanhabibi/Xantham) — schema-driven extractor producing JSON for downstream consumers | Analysis substrate exists; Transcribe integration anticipated |
| C / C++ (headers) | [Farscape](https://speakez.tech) — clang-based two-pass parsing with XParsec post-processing | Analysis substrate exists (Farscape CLI); Transcribe integration anticipated |
| F# (assemblies / source) | F# Compiler Services or equivalent reflection-based analysis | TBD |
| Python (modules with type hints / typeshed) | Python AST + typeshed + library-specific type stubs | TBD |
| Rust (crate public APIs) | rustc/`syn`-based analysis or rust-analyzer's type info | TBD |
| Go (package public APIs) | `go/types` and `gopls` integration | TBD |

The list reflects intent, not commitment to a delivery order. Languages will be added as their analysis substrates mature and consumer demand surfaces.

## What Transcribe Produces

For each ingested library, Transcribe produces one thing: **Clef binding declarations**, the type signatures and module structure user code calls. These look like idiomatic Clef regardless of source language. A TypeScript `interface` becomes a Clef interface declaration; a C struct becomes a Clef record; a Rust trait becomes a Clef SRTP-style abstraction; etc. The mapping is per-source-language but the *output shape* is consistent.

A declaration is a citizen of the graph, not a signature beside it. For a C binding it is the extern form ffi-boundary §3.2 fixes: an external `func.func` plus `func.constant` on the declaration, resolved through `PlatformBinding`, with the `ExternCall`-style binding-strategy annotation (static or dynamic) attached before witnessing. For a TypeScript binding it is the same kind of declaration, annotated for the JSIR pathway. Transcribe does not generate lowering metadata: the witnesses live in Alex's rule library (the Library of Alexandria), keyed by the shape of the declaration, and Alex emits the five portable dialects only; JSIR and LLVM IR are pathway realizations below the witness boundary.

What Transcribe would guarantee by construction is narrower and sturdier than a generated witness: every declaration it emits is one the rule library has a rule for, checked at ingestion time, so a binding cannot join the project without a lowering. This mirrors the [witnessing pattern from C/C++ library binding](https://github.com/speakeztech/clef-lang-site/blob/main/hugo/content/docs/design/interop/library-binding.md), generalized to multiple source languages. Where a shape has no rule yet, that is a Composer/Alex contract item, and Transcribe reports it at design time rather than inventing one.

## Why an IDE Feature, Not a CLI Tool

Each per-language analysis substrate already has (or will have) a CLI form for batch use. Xantham runs from the command line; Farscape has its own CLI; future substrates will too. Transcribe is the **interactive design-time surface** that brings these batch tools into the developer workflow.

The design-time framing matters because library bindings are decisions, not just translations. A given TypeScript surface might map to multiple plausible Clef shapes; the right mapping depends on how the developer intends to use it. Transcribe lets the developer:

- See the proposed bindings before they join the project
- Annotate or override per-symbol mappings
- Decide which types cross BAREWire boundaries (wire-typed) versus stay at the JS interop boundary (interop-typed)
- Pin to a source library version and regenerate when that version changes
- Review what Alex's rule library will witness for each declaration on each target before committing

This is consistent with the broader [design-time-spec philosophy](https://github.com/speakeztech/clef-lang-site/blob/main/hugo/content/docs/design/javascript-targeting/design-time-spec-runtime-reliability.md) — surface decisions to the developer at design time, where the cost of changing them is low, rather than burying them in build-pipeline configuration.

## Relationship to the Per-Language Analysis Substrates

Transcribe doesn't replace the analysis substrates. It consumes them.

- For TypeScript, Transcribe invokes Xantham's encoder to produce the analysis JSON, then runs a TypeScript-specific binding generator over that JSON to produce Clef binding declarations annotated for the JSIR pathway. See [TypeScript Binding via Xantham](https://github.com/speakeztech/clef-lang-site/blob/main/hugo/content/docs/design/interop/typescript-binding-via-xantham.md) for the persistent-architecture rationale.
- For C/C++, Transcribe invokes Farscape's analysis and runs the existing C/C++ binding generator, whose declarations resolve to the extern form ffi-boundary §3.2 fixes (`PlatformBinding`; an external `func.func` plus `func.constant` on the declaration) with the `ExternCall`-style binding-strategy annotation attached before witnessing. See [Library Binding for C/C++](https://github.com/speakeztech/clef-lang-site/blob/main/hugo/content/docs/design/interop/library-binding.md).
- For other languages, the same pattern: a language-specific analysis substrate produces structural type information; a language-specific binding generator consumes it and produces binding declarations the rule library witnesses.

The substrates remain independently maintained. Xantham is Shayan Habibi's project; Farscape is SpeakEZ's; future substrates will have their own ownership stories. Transcribe coordinates them at the IDE level without taking maintenance ownership of any of them.

## What This Document Does Not Cover

This is a placeholder. It does not specify:

- The Transcribe UI (how the developer interacts with it)
- The on-disk format for stored binding declarations and their annotations
- The version-tracking and regeneration workflow
- Per-language binding-generator implementation details
- The Alex witnessing rule library structure (covered by Composer's internal docs as the Library of Alexandria stabilizes)
- The query mechanism for matching incoming source-language analysis against the rule library (likely property-graph-shaped given the multi-cardinality relationships involved — types ↔ names ↔ exports ↔ packages ↔ patterns; SQL/PGQ via DuckDB or equivalent is a candidate, but the choice belongs to the implementation phase, not this placeholder)

Each of these will be authored as the relevant work lands. This placeholder establishes the **directional intent**: Transcribe is the polyglot IDE-side coordination point for consuming per-language analysis substrates and producing binding declarations that join Clef projects as graph citizens.

## Research reference: pattern families for source migration

Wesley Attrot, Luciano Zago, Marcio Pereira, Vinícius Couto, Hervé Yviquel, and Guido Araujo. **[A Pattern Generation Language for MLIR Compiler Matching and Rewriting](https://doi.org/10.1145/3777905)**. *ACM Transactions on Architecture and Code Optimization* 23(1), Article 6, March 2026.

The paper's Pattern Generation Language (PGL) and compiler (PGC) generate families of source patterns for the Source Matching and Rewriting (SMR) tool. Their relevant contribution is reducing the manual effort of enumerating variations of a computational idiom in C and Fortran.

For the source-transposition/implementation-absorption work distinguished above, this technique could help authors recognize foreign idioms and route them to established Clef translation recipes. It could also generate migration regression cases. Language-specific analysis would supply the source semantics; translated Clef would enter Composer's normal elaboration and verification path. Pattern recognition itself supplies no proof of translation correctness.

**Disposition:** retain as a research reference for migration-rule authoring and test generation. Adoption of the PGL/PGC/SMR toolchain is not planned; the paper does not presently justify a new dependency or a change to Alex's MiddleEnd.

### Tree-sitter's supporting role

[Tree-sitter](https://tree-sitter.github.io/tree-sitter/) provides incremental concrete-syntax trees and [structural queries](https://tree-sitter.github.io/tree-sitter/using-parsers/queries/1-syntax.html). For source migration, it could support navigation, source-range selection and identification of candidate idioms while a developer edits foreign code, including incomplete code. Such queries could help present a shortlist of regions for transposition.

Tree-sitter's syntax information does not establish resolved types, ownership, effects or behavioral equivalence. Language-specific semantic analysis would still determine whether a candidate fits a translation recipe, and Composer would check the resulting Clef through its normal pipeline. Its prospective value is in the interactive source-selection surface; adoption would depend on whether it adds useful coverage beyond the parsers and compiler APIs already available to Atelier.

### Concrete example: reading a pointer's lifetime contract

Farscape's `docs/10_Boundary_Marshaling_Spec.md`, particularly **Representation** and **The Composer-Side Contract**, supplies a small example worth carrying into Transpose/Transcribe research. A callback pointer, its `userdata` and its destroy hook together describe a lifecycle. Recognizing that relationship could recover an existing binding contract for a migration recipe: the callback's ABI shape, the environment it accesses, who owns that environment and which event releases it.

The documented callback tiers give three distinct readings:

- **With `userdata` and a destroy hook:** marshal a Clef closure with an environment whose lifetime spans registration; wire the hook to its release thunk.
- **With `userdata` but no destroy hook:** associate the environment with a registration handle whose consumption releases it.
- **Without `userdata`:** require a closed function with no captured environment.

This is a candidate for reading semantic relationships across declarations and uses, beyond matching isolated syntax. Tree-sitter could locate the relevant source regions; language-specific analysis and the library's lifecycle contract would establish the relationship. A function named `destroy` alone does not establish ownership transfer, release timing or whether callbacks have finished. Missing evidence should remain an unresolved contract requirement.

For binding ingestion, that reading selects the appropriate marshaling contract. For implementation absorption, it informs the corresponding Clef lifetime structure, retaining the boundary contract wherever foreign calls remain. Composer's existing joint constraint over Clef type, C ABI and lifetime owner is the intended destination. This records a focused integration example from Farscape's design, not a claim that automatic lifecycle recovery or all callback-tier enforcement is implemented.

## Cross-references

- [TypeScript Binding via Xantham](https://github.com/speakeztech/clef-lang-site/blob/main/hugo/content/docs/design/interop/typescript-binding-via-xantham.md) — the TypeScript analysis substrate and its persistent role
- [Library Binding for C/C++](https://github.com/speakeztech/clef-lang-site/blob/main/hugo/content/docs/design/interop/library-binding.md) — the Farscape pattern that Transcribe extends to other languages
- [JSIR: JavaScript as an MLIR Backend](https://github.com/speakeztech/clef-lang-site/blob/main/hugo/content/docs/design/javascript-targeting/jsir-javascript-as-mlir-backend.md) — the pathway that realizes TypeScript-derived bindings below the witness boundary
- [Design-Time Specification for Runtime Reliability](https://github.com/speakeztech/clef-lang-site/blob/main/hugo/content/docs/design/javascript-targeting/design-time-spec-runtime-reliability.md) — the broader design-time philosophy Transcribe embodies
- Atelier docs `00_architecture.md`, `08_tooling_integration.md` — the IDE substrate Transcribe runs within
