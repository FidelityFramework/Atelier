# 04 - Multi-WebView Architecture

A WREN application can place its editor, graph visualizations, debugger and other tools in WebViews suited to their different interaction needs. Atelier is the in-house example: a developer can edit on one display while exploring a program hypergraph on another. The same approach can serve applications that combine interactive forms, dense charts, live data and native services.

The architectural goal is a responsive application in two complementary senses. **Ranvier makes related state changes coherent and propagates them to the views that depend on them.** Work scheduling, rendering and GPU acceleration keep input and presentation responsive as the amount of work grows. Both need deliberate boundaries and measurement on each host.

## From WrenHello to multiple surfaces

WrenHello demonstrates the foundation on Linux: a native Clef host uses Farscape-generated WebKitGTK bindings to embed a WebView. Its existing frontend uses F# compiled with Fable and Partas.Solid, with a bounded compact ASCII message bridge. The WREN direction standardizes the F# frontend on **Fickl and Ranvier**. Xantham generates JavaScript-facing bindings, with D3 and graphics APIs as targets for visualizations.

BAREWire already supplies typed encoding, decoding and framing across JavaScript, .NET and native Clef. Integrating it with a particular WREN host transport is an application integration step. WrenHello describes BAREWire over a local WebSocket as a direction for its bridge. Each host must establish its own transport lifecycle and access policy. The [reactive frontend chapter](06_reactive_frontend.md) describes the frontend contract and integration sequence.

A WebView is a separate visual and document context with its own frontend state and lifetime. Whether WebViews use separate operating-system processes, JavaScript threads, GPU processes or devices is decided by the host and engine. Some resources may be shared. A separate surface therefore gives the application a useful ownership and presentation boundary, while isolation, concurrency, crash recovery and performance must be established for the actual host.

```mermaid
flowchart TB
    service["Compiler and application services"]
    host["Native host and coordinator"]
    editor["Editor WebView<br/>Fickl + Ranvier"]
    graph["Visualization WebView<br/>Ranvier + D3 + renderer"]
    debug["Debugger WebView<br/>Fickl + Ranvier"]
    worker["Layout worker where supported"]
    gpu["WebGPU or WebGL device<br/>where available"]
    service <--> host
    host <--> editor
    host <--> graph
    host <--> debug
    graph <--> worker
    graph --> gpu
```

The diagram expresses logical boundaries. It makes no promise about operating-system processes or physical GPU allocation. An application can also put several panels in one WebView or run a browser-only frontend when its services are available there.

## Surfaces and their responsibilities

### Editor

The primary editor surface manages text editing, selection, navigation, file and panel presentation, and the user's immediate interaction. CodeMirror and a docking library can be integrated through typed adapters. Local text and selection feedback should remain prompt while diagnostics and other service results arrive asynchronously. Each result must identify the document version it describes so that a late response does not attach old diagnostics to newer text.

### Debugger and terminal

A debugger surface presents a session's breakpoints, call stack, suspension frames and other published facts. The compiler and debug services supply the meaning of those facts. The view manages selection, filters and presentation. Separate placement can make a session easier to use across monitors, while the host's actual process and failure model determines how much fault isolation it provides.

A terminal surface connects a terminal renderer to a native pseudoterminal through a bounded, lifecycle-managed stream. Continuous output needs flow control and a policy for retained scrollback. Its rendering may use an accelerated path when supported, but GPU and memory resources are shared with other activity according to the host. Closing the surface releases its subscriptions and follows the application's explicit policy for the shell session.

### Program hypergraph visualization

The graph surface renders an authorized projection of a compiler revision. The projection retains the identity of each relationship and its participants, roles, scope and published evidence. Layout, camera, filters and selection belong to the view. Moving a glyph does not change a graph fact or establish a proof. The [hypergraph chapter](12_hypergraph_visualization.md) develops the 2.5D layered and spatial views, including how to select one joint hyperedge and inspect all of its incidences.

Use a stable layered view for tracing source, semantic, evidence and realization relationships, and an exploratory spatial view for navigating a selected neighborhood. An orbiting camera, curved connections, labels, search, focus and an accessible textual view make dense structures useful. A visual grouping or bundle must still let the user inspect the individual source relationships it contains. Cross-revision selection follows the identity contract supplied by the compiler; similar labels or positions alone do not establish identity.

## Ranvier across WebViews

Each WebView owns its local Ranvier graph and its UI lifecycle. Signals hold the current projection and interaction state, memos derive the visible subset and other presentation values, and effects connect accepted changes to the DOM or renderer. A window that closes disposes its owners, subscriptions, pending work and graphics resources. Reopening reconstructs its state from the coordinator and the relevant service.

A coordinator or service actor can order commands and serialize changes within its own mailbox. That ordering gives the application a place to manage shared state and route messages. A mailbox is not itself a reactive transaction: when a WebView accepts a related set of state changes, it applies them in a **local Ranvier batch** so dependent computations and effects see a coherent update. An event, such as a click or completed request, remains an occurrence; the current selected node or current graph revision is state. Separate WebViews do not share one implicit reactive graph.

For example, a user selects a relationship in the graph view:

1. The graph view resolves the pick against its displayed revision and batches its local selection and detail-panel state.
2. It sends a typed selection message containing the revision and relationship identity to the coordinator. The coordinator routes it to interested surfaces under the application's access policy.
3. The editor checks the identity and source correspondence supplied for that revision. It batches the accepted navigation and highlight changes in its own Ranvier graph.
4. If an editor has moved to another revision or the correspondence is unavailable, it requests or displays the appropriate current information instead of silently highlighting a guessed match.

A theme change can likewise be broadcast as a versioned state update, with each view applying it locally. A newly opened view receives current state, rather than depending on messages sent before it existed. Typed messages, endpoint validation, size limits, backpressure and shutdown handling belong to the selected host transport. A native handle or JavaScript object is local to its execution context unless an explicit protocol establishes a corresponding remote resource.

The coordinator should retain an explicit registry of open surfaces, their subscriptions and the current source or session each is authorized to observe. A request carries a correlation identity so its response returns to the intended view. A subscription has its own lifetime: closing a visualization retires that subscription without automatically stopping a compiler session or background observer that another owner still needs. Reopening obtains an admitted snapshot and then resumes from its declared revision. An event such as “open this file” is delivered as an occurrence; the currently selected file and current theme are state that can be reconstructed.

The following sketch gives that coordinator a concrete shape. The names describe the intended protocol; they are not declarations of an existing runtime API:

```fsharp
type SurfaceKind =
    | Editor
    | Graph
    | Debugger
    | Terminal

type SurfaceRegistration = {
    Id: SurfaceId
    Kind: SurfaceKind
    Generation: SurfaceGeneration
    Subscriptions: SubscriptionId list
}

type CoordinatorMessage =
    | OpenSurface of SurfaceKind
    | CloseSurface of SurfaceId * SurfaceGeneration
    | SelectGraphFact of GraphRevision * GraphFactId
    | NavigateToSource of GraphRevision * GraphFactId
    | ThemeChanged of ThemeRevision * Theme
```

The host maps each registration to its platform window and WebView through its admitted bindings. A graph fact identifier is interpreted in the supplied graph revision. A generation distinguishes an open surface from an earlier instance with the same role. The concrete wire schema and platform handles belong to their respective owners, and the first integration can use fewer message cases.

For example, the editor may accept source revision R2 while the graph view still displays the last complete projection for R1. The editor stays interactive and the graph identifies the revision on screen. When an authorized R2 projection arrives, the graph applies it as a local batch. Once R2's scene is ready, the renderer installs its geometry and picking table together. Until then, R1 remains the displayed revision. A request to navigate from the R1 scene carries R1 and its subject identity. The coordinator can resolve the request against that revision, ask for a supported correspondence or tell the user that the selection has changed. A matching label is insufficient to establish a match across revisions.

### Live revisions and asynchronous work

A complete newer snapshot can replace an older pending snapshot under a defined policy. A delta is applied only to its declared base revision. Dependent deltas cannot simply be dropped because a later one arrived; a missing base calls for recovery, such as a fresh authorized snapshot. Deliver only the scope and support facts needed by an active view when the service's projection contract permits it.

Tag a layout request with the graph revision and view generation. Tag GPU work with the renderer or device generation as well. When a worker or GPU completion arrives, accept it only if its owner and relevant generations are still current. For example, if revision R2 replaces R1 while R1's layout is running, R1's late result cannot overwrite R2's picture or picking table. If device generation G2 replaces G1 after device loss, a G1 completion cannot populate G2's resources. The displayed geometry, picking table, labels and details must all identify the same accepted revision.

Cancellation saves resources when available. Generation checks remain necessary because cancellation and asynchronous completion can race. A view may keep the last complete picture visible while preparing an update, provided the interface identifies the displayed revision and does not present its selection or evidence as belonging to a newer one.

## D3, GPU acceleration and a responsive interface

D3 and the graphics renderer play complementary roles. D3 provides scales, transformations, force layouts and shape generation through Xantham-generated bindings. Its standard force simulation is two-dimensional and mutates node coordinates, so run it on a view-owned copy of the projection. Its many-body force uses a Barnes–Hut approximation. A genuinely three-dimensional layout or spatial spline requires an appropriate 3D implementation; rotating a layered 2D layout is a different and often useful 2.5D presentation.

Canvas 2D is a useful flat reference and a portable rendering option. WebGL 2 can accelerate graphics on supported hosts. **WebGPU is the strategic path for dense scenes, interactive camera movement and compute-assisted visual work.** The renderer may use GPU buffers, shaders and instanced geometry for nodes and connections, while D3 continues to supply scales, layout or curves. A D3 force simulation does not automatically move to the GPU because its output is drawn with WebGPU. A GPU layout requires an explicit compute implementation and comparison of its behavior and results with the chosen reference.

Use the GPU where it helps the actual workload. A small graph may be faster and simpler on the CPU; large transfers, buffer allocation or synchronous readback can erase a rendering gain. Keep the editor's text and controls responsive while graph work proceeds:

- Run expensive layout in a worker where the host supports it. Transfer bounded projection data or suitable buffers, and keep the authoritative graph and UI state with their owners. A separate WebView alone does not guarantee that long-running JavaScript cannot contend with other surfaces.
- Incrementally prepare visual data, bound work per update, and use level of detail or scoped views to limit the visible scene. Camera movement should normally reuse settled geometry rather than rerun the layout.
- Schedule rendering around presentation opportunities and coalesce superseded *visual work*. Preserve ordered commands and revision-dependent graph changes. Keep selection and input feedback independent of a long layout.
- Treat GPU submission and completion as asynchronous. Manage buffers and their lifetime explicitly, avoid blocking readbacks on the input path, and restore the renderer after device loss using the accepted scene state.
- Keep the graph's exact identifiers and evidence in the application model. GPU-friendly numeric positions and picking buffers are presentation data. A pick resolves through the current revision's identity table before the interface opens source or proof details.

An initial D3 path can compute positions on the CPU in a worker and send a versioned layout result to the graph view. The renderer uploads the positions and incidence geometry once they are admitted. A camera-only update changes the view transform and draws the retained geometry. A changed graph may require new positions, while a proof-status change may only require updated visual attributes. These are optimization opportunities to measure against the selected renderer's actual dependencies.

A WebGPU compute layout is a separate implementation. It needs a defined input projection, shader, iteration and convergence policy, numeric representation, resource limits and a way to associate every output position with its source identity. The GPU can retain intermediate positions when the renderer consumes them there. If the CPU needs the positions for search or picking, an explicit readback and its synchronization cost enter the design. Compare its results and full elapsed cost against the selected D3 or other CPU algorithm, with the same visual and semantic requirements. The [binding playbook](16_xantham_gpu_binding_playbook.md) identifies the API and layout checks for that path.

For an interactive layered spline view, a common scene contract can include graph revision, hyperedge identity, participant incidences and roles, chosen layer, curve geometry, camera and the current selection. WebGPU, WebGL 2 and Canvas 2D can realize supported subsets. The user can rotate the scene without re-saturating the compiler graph, select one hyperedge and see every participant in a detail panel, and return to an accessible flat or textual view. A fallback should describe any visual capability it cannot reproduce while preserving the underlying relationship and inspection controls.

The host must test the actual WebView, origin and security context for `navigator.gpu`, adapter and device availability, required limits, and any optional worker or offscreen-canvas path. A custom application URL is not automatically a secure context. The application selects a supported WebGPU, WebGL 2 or Canvas 2D path and communicates the available experience. Browser and mobile support depend on the embedded engine and platform configuration, so exercise the real desktop and mobile hosts rather than assuming parity with a standalone browser.

The [graphics overview](05_webgpu.md) compares the rendering choices. [Visualization integrity](13_visualization_integrity.md) covers identity, revisions and evidence. The [WREN WebGPU bindings](14_wren_webgpu_bindings.md) and [Xantham binding playbook](16_xantham_gpu_binding_playbook.md) cover API generation, resource lifetimes and validation. In particular, BAREWire's wire encoding and WGSL's GPU buffer layout are distinct contracts and require an explicit conversion.

## Coordination and lifecycle

The native coordinator connects surfaces to the application's services and manages their lifetimes. In Atelier, the compiler-service connection supplies diagnostics and graph projections for the editor and visualizer. The intended CCS-backed service provides the published semantic facts; each view consumes an appropriate projection rather than deriving or repairing those facts. CAC's present FSharp.Compiler.Service-based implementation and the intended Lattice/CCS integration have different interfaces, so adapters should name the service and revision contract they actually use.

Open optional surfaces on demand and subscribe them to the state they need. Route a command to the appropriate service, then deliver the accepted result to interested views. On close, stop subscriptions, release local reactive owners and GPU resources, and settle or cancel outstanding operations. If a WebView terminates, restore its state from the coordinator when the host exposes a recovery mechanism. A shared process failure may affect several surfaces, so recovery must follow the platform's actual failure boundary.

Keep the renderer's lifetime distinct from the WebView's and the application service's lifetime. A resized canvas may require new presentation resources while preserving the selected graph. Device loss retires one renderer generation; rebuilding it from the latest accepted scene should preserve the view's selection when the identities remain valid. Closing a window disposes its local owner and retires its outstanding work, while the coordinator continues other windows and any independently owned service. Cancellation and shutdown must still account for callbacks and GPU work already in flight.

Floating windows and multiple monitors are valuable for editing and visualization together. Focus, keyboard shortcuts, accessibility, window placement and state restoration are part of this experience. A browser-only deployment can use the same frontend architecture with browser-supported windowing and an available service connection.

## Demonstrating responsiveness

Use a repeatable workload on each named host rather than assuming a fixed frame rate or an inherent advantage over another application framework. The [visualization roadmap](15_visualization_roadmap.md) describes the wider acceptance sequence. For the multi-surface case, exercise these behaviors together:

1. Compile the Fickl/Ranvier frontend and verify local batching, derived values, owner disposal and asynchronous completion in the actual WebView. Record the versions and host configuration.
2. Edit and navigate continuously while a bounded graph projection is delivered and its D3 layout runs. Compare main-thread and supported worker placement. Measure input-to-feedback latency, layout time, frame-time percentiles, CPU and memory.
3. Render the same graph and interactions with Canvas 2D, supported WebGL 2 and WebGPU paths. Record graph size, incidence count, labels, viewport, GPU, driver and engine. Measure update-to-stable time, frame pacing, transfer costs and selection accuracy, including camera movement.
4. Deliver R1's layout after R2 is accepted, test a missing delta base, and complete old G1 work after a G2 device recreation. Confirm that the visible geometry, picking and details remain coherent with the accepted revision.
5. Open, close and restore a visualization while the editor remains active. Test sustained terminal output, transport backpressure, device loss and the actual host's process termination behavior. Compare any Electron reference under the same workload and measurement conditions before making comparative claims.

These tests establish the behavior of a named frontend, renderer and host. They also direct optimization: Ranvier coherence, worker scheduling, transport, layout and GPU presentation each have distinct costs and responsibilities.

## Related documents

- [Reactive frontend: Fickl, Ranvier and WREN](06_reactive_frontend.md)
- [WREN graphics and interactive visualization](05_webgpu.md)
- [Seeing the program hypergraph](12_hypergraph_visualization.md)
- [Visualization integrity](13_visualization_integrity.md)
- [WREN, Xantham and WebGPU bindings](14_wren_webgpu_bindings.md)
- [Visualization roadmap](15_visualization_roadmap.md)
- [Xantham GPU binding playbook](16_xantham_gpu_binding_playbook.md)
- Previous: [Atelier's unique features](03_unique_features.md)
