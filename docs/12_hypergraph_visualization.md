# 12 - Seeing the Program Hypergraph

WREN applications should be able to make complex relationships explorable through a shared visual and interaction contract. An engineer examining a production process, a financial planner comparing scenarios or a researcher inspecting a simulation may all need to select a joint relationship and understand its participants, provenance and changes. The meaning of each relationship comes from its application domain.

Atelier is a demanding in-house example. A Clef developer can select an expression and see the values, operations, resource uses, obligations and evidence that participate in its meaning. They can follow a relationship through elaboration and realization, inspect a pending proof and see the effects of an edit. The visual is an interactive view of an authorized graph revision.

The first implementation should support two complementary presentations: a **stable, layered 2.5D view** for tracing relationships and an **interactive spatial graph** for exploring a selected neighborhood. Both use the same source projection, identities and inspection controls. Canvas 2D can supply a flat reference; GPU renderers can support dense scenes and camera movement. Their role is to make the same facts legible.

This document defines the reusable visualization behavior and uses Atelier to make its requirements concrete. Graph export, cross-revision identity, browser bindings and renderer capabilities remain separate implementation gates. See the [visualization overview](05_webgpu.md) for the WREN integration and linked topic chapters.

## A joint constraint is one object

The program hypergraph relates all participants in a judgment. A visual connection must therefore retain the identity of the relationship and of each participant's incidence, including its role, scope and any ordering supplied by the graph. A normal line between two nodes can represent a binary relationship when the source provides one. A multi-participant relationship needs a visual object of its own.

One initial glyph is a small diamond with a curve to each participant:

```text
   buffer A ──────┐
                  │
   buffer B ──────◇ capacity obligation
                  │
   buffer C ──────┘
```

The diamond denotes the original hyperedge. The curves are its incidences. Selecting the diamond opens the relation; selecting a curve identifies the participant and its role in that relation. The user should also be able to select a participant and find every incident relationship in the admitted projection. A ribbon, hull, bundle or surface can be offered where it makes an important relationship easier to see. Each representation must retain the same selectable identity and a route to the complete participant list.

This presentation also distinguishes **one joint relationship** from a visual grouping of several relationships. A bundle may join curves to reduce clutter, but selecting it should expose the individual source relationships it contains. The grouping is view state. The underlying hyperedges remain individually inspectable.

### The three-buffer case

Suppose buffers A, B and C each require four units of the same memory resource. A platform supplies ten units. If the three are live simultaneously, the applicable joint condition is:

```text
A + B + C <= capacity
4 + 4 + 4 <= 10
```

Every pair fits. The three-way requirement does not. The initial fixture should retain one obligation identity, the three buffer identities, their participation roles, the capacity declaration, the applicable scope and the liveness premises. The view can show a single obligation glyph with three incidences. Its details panel presents the condition, the basis for the overlapping lifetimes, the source or platform location and the obligation's actual status.

The display must not manufacture a successful or failed proof from the glyph color. If the evidence is pending, the panel says what is pending. If a checked result is published, it identifies that result and its assumptions. If a schedule change establishes non-overlapping lifetimes, a new authorized graph revision can carry that result. The viewer then shows the new fact and its provenance. Moving the glyphs apart on screen has no effect on storage or proof.

This is a useful integrity discriminator: a visual that shows three pairwise lines and hides the joint predicate can make every visible pair appear safe while failing to communicate the actual constraint.

## From source expression to realized operation

A layered view can help explain a relationship across representations. As an example, select a source operation that reads a measured field from a buffer and uses it in a computation. An admitted projection could expose:

1. **Source:** the declaration, expression and source location.
2. **Semantic graph:** the operation, field, buffer and their dimensional and access relationships.
3. **Evidence:** applicable range, lifetime, layout and access obligations with their premises and status.
4. **Realization:** the corresponding operation and artifact evidence where those links are available.

The user can rotate the view to distinguish crossings, click a line or ribbon, and open the exact correspondence behind it. The details view names the source and target objects and the particular relationship that supports the connection. When an item has no published correspondence, its path ends with an explicit unavailable or unresolved state.

These planes are a **chosen view**, not a claim that the whole hypergraph is a pipeline. A given hyperedge can connect several facts within one stage or cross a boundary in a more complicated way. Provide alternate groupings by source scope, operation, resource, proof premise or target only when the projection supports them. Every axis and visual encoding needs an explanation available in the interface.

The same mechanism can support other editor questions:

- **Why does this value have this dimension?** Trace the relevant source types and dimensional relationships, retaining the distinction between dimension and selected numeric representation.
- **Why was this region or layout selected?** Show the relevant source use, lifetime, platform description and settled layout premises.
- **Why is this proof pending?** Highlight the obligation, its source operation, available premises, missing premises and dependent claims.
- **What changed after my edit?** Compare admitted revisions and show affected facts and evidence alongside retained facts, provided the compiler supplies the corresponding identity and authorization.
- **Which continuation owns this resource?** Follow the retained invocation, captured frame, owned resource and completion relationship where those facts have been published.

The view remains useful when a relationship is not yet available. It can say what the current projection contains and navigate the facts it has. It cannot infer a missing compiler judgment from visual proximity.

### Beyond the compiler

The same interaction vocabulary can serve different domains when each supplies its own data and semantics:

- **Financial planning:** a scenario relates accounts, rates, assumptions and projected results. Select a result to inspect the contributing inputs and distinguish a forecast from a confirmed transaction. Currency, exact amount and effective date remain application facts.
- **Industrial operations:** a process stage relates equipment, materials, sensor observations and operating limits. A selected joint constraint can show the complete participants and the source of each threshold, rather than merely coloring a machine in a diagram.
- **Scientific and engineering work:** a model relates variables, dimensions, parameters and experimental observations. Layers can show model inputs, derived quantities and measured results, with uncertainty and provenance preserved under the domain's contract.
- **Distributed services:** a request relates its caller, service, continuation, resource and reply. A live view can make that relationship inspectable when the runtime supplies the relevant identities and observation contract.

These examples motivate a shared facility, not a universal data model for every domain. Each application defines what its nodes and relationships mean, which source is authoritative, and which conclusions the interface may claim. Graph layout and renderer contracts can then be reused without turning a visual cluster or spline into domain evidence.

## Spatial views and interaction

### Stable 2.5D

In a 2.5D view, elements can occupy planes or ordered bands while the user changes camera position, zoom or tilt. Curves and ribbons may pass between planes to make a chosen relationship visible. This gives the user an opportunity to separate crossing paths while preserving a recognizable organization.

Use a stable arrangement when the question is a trace: the user should be able to keep source, graph, evidence and target positions in mind while selecting or comparing them. Camera movement changes the projection, not the graph or proof. Offer an orthographic or otherwise readable flat view, a reset camera, focus on selection, keyboard navigation and a textual or tabular representation of the same selected facts.

### Exploratory 3D

An exploratory scene can place nodes and relation glyphs in a volume. A force layout, clustering policy or explicit domain grouping may propose positions. The camera can orbit, zoom and pan without recomputing those positions. The interface must name the layout and show which grouping and filters are active.

Force-directed geometry is a way to explore structure. Proximity can reflect the forces or constraints used by the layout; it is not evidence that two facts are semantically equivalent. Clustering is also a derived visual operation. Offer the underlying participant and relationship lists so a user can check the interpretation.

As the scene becomes dense, progressively reveal details. A user can expand a scope, isolate one hyperedge, focus on a source expression, filter pending obligations or ask for the immediate participants of a selected relationship. Display labels as the view and zoom permit; keep them accessible through selection and search even when hidden in the picture. A bounded or aggregated view should visibly say what was omitted and let the user request a supported expansion.

### Curves and visual encodings

Splines can distinguish crossings and show related paths as a bundle. Thick ribbons or translucent surfaces can emphasize an important multi-participant relationship. Each visual encoding must have a stated meaning in the selected view: color, line pattern, thickness, depth, animation and glyph shape cannot acquire a universal interpretation by accident.

Use more than color to distinguish statuses and selected objects. Offer reduced motion. Animation may show how a selection or graph revision changes; it represents an execution trace only when tied to an actual recorded trace. Explain clipping and obscured items. For a misleading or ambiguous pick, offer a list of candidates rather than silently choosing the wrong fact.

## D3, Xantham and the renderer

Xantham's original target was D3.js. That history makes D3 the first binding surface to revisit for this visualization. Inspect the current generated declarations and findings and compile an actual consumer under the selected Fickl toolchain. Record which D3 packages and versions work and which declarations need generator or adapter changes. A historical target is a strong starting point; it does not substitute for the current test.

D3 supplies reusable transformations, scales, layout tools and shape generation. Its standard force simulation produces node positions and can drive the graphics system of choice. Its standard curves construct two-dimensional shapes; genuinely spatial splines need their own construction. D3's many-body force uses a Barnes–Hut approximation, so compare against the real algorithm and its parameters rather than an assumed all-pairs quadratic implementation. [D3 force](https://d3js.org/d3-force), [many-body force](https://d3js.org/d3-force/many-body), [D3 curves](https://d3js.org/d3-shape/curve).

A 3D-capable layout is a separate choice. [3D Force-Directed Graph](https://github.com/vasturiano/3d-force-graph) combines a Three.js/WebGL visual with `d3-force-3d` or `ngraph`; it is a useful reference for camera controls, curved links and dynamic updates. Its pairwise graph abstraction needs an explicit projection to show a Fidelity hyperedge and all its incidences. Record the library version, graph model and conversion in the experiment.

The renderer takes the resulting visual model and draws it. A Canvas 2D reference can check a bounded flat view. WebGL 2 can provide GPU graphics where the host supports it. WebGPU is the strategic path for dense scenes and possible compute-assisted layout. A given graph-layout package does not acquire WebGPU merely because a WebGPU renderer exists; demonstrate the combination and its shader/material compatibility on the target. See [Three.js WebGPURenderer](https://threejs.org/manual/pages/webgpurenderer).

## Live graph changes

Fidelity's differential compilation direction makes a live view especially valuable. The compiler can revalidate affected relationships and evidence after an edit, and Atelier can update the visible projection when an authorized delivery contract is available. The visualization should make retained, replaced and withdrawn facts distinguishable without asserting that an unchanged-looking drawing has retained proof evidence.

The initial attachment should request the live scope needed by the active view, together with its required support and boundary facts. Later scoped additions, replacements, withdrawals and retirements need an exact resident base and source authorization. A visualizer should not demand the whole retained compiler corpus merely to draw a small neighborhood. The current indexed-revision architecture describes this delivery direction and identifies the scoped integration as work still to accept. [Indexed revision architecture](../../Fidelity.PSG/docs/Indexed_Revision_Architecture.md).

For a snapshot-based first experiment, associate the displayed scene, labels, picking table and selected details with the snapshot revision. When a new snapshot is accepted, replace or reconcile the view according to the available identity contract. If an identity is retired or no longer present, clear or explicitly mark its selection. When identity across revisions is not provided, do not treat a similar label or location as the same fact.

For an admitted delta, require its declared base revision. A missing or mismatched base calls for the defined recovery path, such as obtaining a fresh authorized snapshot. Do not drop an intermediate delta merely because a later delta looks newer. A complete later snapshot can replace an older pending snapshot under an explicit policy. Layout work may be cancelled or superseded, but its eventual completion still needs a revision and owner-generation check before it can affect the current picture.

## First experiments and acceptance

1. Create a bounded fixture with named source locations, three buffers, one capacity hyperedge and explicitly represented participant roles and obligation status. Verify that the view reconstructs the full relation from the projection.
2. Offer a flat list and a simple Canvas 2D view with selection of the hyperedge and each incidence. Both paths must resolve to the same identities, premises and revision.
3. Render the same fixture in a GPU-backed spatial scene with an orbiting camera and curved incidences. Test a WebGL reference and WebGPU where the actual host supports them. Keep the evidence and inspection behavior stable across the two.
4. Add the layered 2.5D source-to-realization view for a case with real correspondence data. Identify its layers, unavailable facts and visual grouping. A view transformation must not create a compiler fact.
5. Exercise a late layout completion, a changed proof status, participant retirement and a mismatched delta base. Verify that picking and details always identify the displayed revision. Provide a textual alternative and reduced-motion interaction.
6. Increase the graph size under pinned layouts and measure layout time, update-to-stable latency, frame-time percentiles, selection accuracy and memory. Record node count, hyperedge count, total incidences, labels, filters, renderer, WebView, GPU and driver. Compare the 2D and 3D treatments on the questions a developer can answer as well as raw frame rate.

This is a design and implementation plan. It does not establish a complete graph-export interface, stable cross-revision identity, GPU renderer or proof of compiler correctness.

## Related documents

- [Interactive graph visualization and GPU strategy](05_webgpu.md)
- [Atelier's graph visualization direction](03_unique_features.md)
- [The Program Hypergraph](https://arxiv.org/abs/2603.17627)
- [D3 force simulation](https://d3js.org/d3-force) and [D3 curves](https://d3js.org/d3-shape/curve)
