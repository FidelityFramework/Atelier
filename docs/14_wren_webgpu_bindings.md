# 14 - WREN, Xantham and WebGPU Bindings

WREN applications need a shared reactive frontend that can draw detailed visualizations in a desktop WebView and in a browser. The selected F#-to-JavaScript path uses **Fickl and Ranvier**. Xantham supplies the route from TypeScript declarations to F# bindings, beginning with D3 and extending to WebGPU. A native Clef host supplies platform integration and application services; BAREWire supplies the typed communication contract. Atelier is the in-house exemplar for the complete design and adds compiler-aware graph inspection.

This chapter describes how these pieces fit across reusable Fidelity libraries and independently packaged products, what must be implemented, and what each target must demonstrate. Beancounter charts, scientific visualization and industrial control panels may share bindings, transport and lifecycle contracts while defining their own data and visual semantics. It is an integration design. A typed binding does not establish that a host exposes WebGPU, that the proposed package has been generated, or that a browser shader is a compiled Clef program.

For the editor's graph vocabulary and interaction design see [Hypergraph visualization](12_hypergraph_visualization.md). For byte layouts, identities and evidence across the renderer boundary see [Visualization integrity](13_visualization_integrity.md).

## The selected arrangement

```text
CCS/Baker: settle source semantics and graph facts
      ↓
published PSG revision and an admitted projection
      ↓
native Atelier host and its platform resources
      ↓                 BAREWire contract
Fickl-compiled frontend: decode → Ranvier state → selected visual snapshot
      ↓                                          ↓
D3 layout and curve tools                 renderer ownership and lifecycle
      └──────────────────────┬───────────────────┘
                             ↓
          Xantham-generated browser API bindings
                             ↓
      Canvas 2D / WebGL 2 / WebGPU in browser or WebView
```

The native host, the frontend reactive graph, and the GPU command queue are separate execution and ownership contexts. A compiler result or application update crosses the explicit communication boundary. Camera motion and repeated drawing can remain local to the frontend. The graph visualizer presents published facts and evidence; it does not supply a competing source of compiler semantics.

For a shared visual, the model contains stable-in-that-projection object identities, incidences, selection, camera, visible properties and the revisions on which they depend. Which identities survive a new compilation and whether a delta is available must follow the graph-export contract actually provided by CCS. A snapshot works as an initial input even when a cross-revision delta has not been admitted.

## Binding generation and library boundaries

[Xantham](https://github.com/shayanhabibi/Xantham) translates TypeScript declarations into F# bindings intended for Fable-family JavaScript compilation. D3 was its original binding target. Re-exercise the current generator against selected D3 modules before treating their historical coverage as current: the useful first surface includes scales, graph layout and curve generation, event callbacks and any data-join or selection operations actually needed by the chosen prototype.

WebGPU offers a second demanding target. The [GPUWeb TypeScript definitions](https://github.com/gpuweb/types) describe devices, buffers, textures, descriptors, features and limits. Choose and pin a declaration source compatible with Xantham's TypeScript compiler and DOM ownership. The GPUWeb project notes that declaration completeness differs among TypeScript releases; adding a second ambient WebGPU declaration set without checking the existing DOM set can create conflicts. Record the exact declaration inputs and generator version.

The first binding slice should cover:

- canvas context lookup, adapter and device acquisition, features and limits;
- buffer, texture and pipeline descriptors needed by one scene;
- command encoding, queue submission and the selected upload operation;
- validation errors, asynchronous completion, device loss and cleanup;
- the callable, promise, nullable and typed-array forms used by those operations.

Inspect Xantham's per-symbol findings and classify widened or unsupported mappings. Compile the actual generated package with the pinned Fickl frontend and run it in the target WebView. A generated signature can type-check while passing a malformed descriptor, omitting a runtime conversion or using an unsupported feature. Conversely, a generator failure should identify the declaration shape it cannot represent, so the owning Xantham pass can be improved or a narrow, documented adapter can bridge it.

### Fidelity.WebGPU

The proposed **Fidelity.WebGPU** package is the browser-facing WebGPU surface. Keep its generated bindings and authored helpers visibly distinct:

1. **Generated interface:** faithful F# access to the selected WebGPU declarations, including the asynchronous and nullable cases the browser defines.
2. **Authored integrity helpers:** declared GPU buffer layouts, checked size and range conversions, capability checks, device/resource lifecycle and error reporting. These helpers can use the generated interface without changing its ownership.
3. **Application and UI adapters:** Atelier-specific graph scene, camera, selection and visual semantics. Fidelity.UI can express shared UI behavior and lifetime expectations independently of the chosen renderer.

The package can serve non-UI WebGPU compute too. Fidelity.UI need not become the authority for generic buffers or shader dispatch. Likewise Xantham need not encode Atelier's graph or accounting contracts in a general bindings generator. A future native WebGPU package would require a separately justified native API, binding and interoperability contract; a JavaScript API binding does not automatically provide it.

The generated layer should retain the faithful browser API for advanced use, and the authored checked path should make its conditions clear. In F#, lifecycle wrappers and runtime checks help prevent misuse but cannot alone guarantee unique ownership across all aliases. A future Clef-native implementation can explore stronger resource judgments without overstating the current Fickl/F# guarantees.

## Ranvier, drawing and asynchronous work

Ranvier manages frontend state and dependencies. A graph projection update can be decoded and applied as one coherent batch. Derived values can prepare a scene, filter a region, select an obligation or update a camera. A renderer owner accepts work associated with a graph revision and device generation, prepares or uploads changed resources, and schedules presentation.

This is a design connection, not a requirement that every Ranvier node be an actor. In the current Fable target, Ranvier documents single-threaded graph dispatch and later promise completions. Node.js tests of the shared engine are useful package evidence; the selected Fickl build and actual WebView need their own integration tests. A worker or native service is a separate execution context with messages and ownership rules. [Ranvier's Fable target](https://ranvier-org.github.io/Ranvier/fable/).

The renderer actor/mailbox idea has a specific job: serialize decisions about what to accept, supersede and retire. Applying a Ranvier batch avoids observing a half-applied local update. A mailbox orders accepted messages. GPU work already submitted remains asynchronous and has its own resource-completion conditions. Graph revision R1 finishing after R2 must not overwrite R2; work created on lost device generation G1 must not publish to G2. [Visualization integrity](13_visualization_integrity.md) specifies those checks and the discriminating cases.

## The current host and the bridge

[WrenHello](../../WrenHello/README.md) demonstrates a Linux x86-64 native host using WebKitGTK and a frontend authored with F#/Partas.Solid, compiled through Fable and bundled with Vite. Its existing counter exchanges typed command/event values through a compact ASCII script-message protocol. Its inbound binding preserves the UTF-8 byte count and rejects messages beyond 4,096 bytes before copying. The host and its lifecycle are useful evidence for WREN, while the selected Fickl/Ranvier frontend is the direction to implement and validate.

BAREWire already has shared encoding and decoding, schema and framing components, with .NET, JavaScript and native Clef project paths. Its envelope includes a message kind, correlation identifier and BARE payload, with a length prefix for stream transports and a schema epoch in a Hello frame. The project's recorded gates cover the three compilation paths within their stated snapshots. Its [implementation status](../../BAREWire/docs/Implementation%20Status.md) distinguishes those built components from particular transport and shared-memory integrations.

WrenHello's README describes BAREWire-encoded messages over localhost WebSocket as the intended integrated route. That project must connect the selected existing codec and framing to the actual host and frontend, then exercise handshake, frame size, origin, shutdown behavior and transport before a graph stream depends on it. The current 4,096-byte script channel should not be silently promoted to a large-scene transport. This is a WrenHello integration gate, not a proposal to rebuild BAREWire's encoding foundation.

Send meaningful versioned snapshots or admitted deltas across the boundary, with bounded queues and recovery. The native host need not receive a request for every animation frame or camera movement. Within the WebView, the renderer can retain GPU resources and update the parts affected by accepted frontend data. When a delta's base revision is absent or obsolete, request the declared recovery action rather than rendering a partial graph as a complete one.

Farscape-generated bindings and Fidelity.WebKit describe the native WebView and desktop boundary. They establish how the native process calls the host platform. Xantham's bindings describe JavaScript-facing WebGPU and D3 APIs. The two binding systems meet through the application's messages, not by giving a JavaScript object the identity or lifetime of a native handle.

## Canvas, WebGL and WebGPU

Use the same scene and interaction contract across renderers where its capabilities permit:

| Path | Suitable first use | What must be checked |
|---|---|---|
| **Canvas 2D** | Small graphs, charts, labels and a flat reference view | Drawing cost and interactions on the actual host. The browser may accelerate it; the API does not expose programmable GPU compute. |
| **WebGL 2** | Dense 2D or 3D graphics on a host where it is the available GPU path | Context creation, extensions, GLSL shaders, resource lifetime, picking and device/context loss. |
| **WebGPU** | Explicit GPU rendering and compute for the strategic 3D and data-processing path | Secure context, adapter/device, features and limits, WGSL validation, buffers, queue completion and device loss. |

The renderer is a realization choice, while layout is a separate calculation. D3's standard force simulation and 2D curves can feed any renderer after an appropriate geometry conversion. A spatial layout needs a 3D-capable algorithm or a constrained projection. [D3 force](https://d3js.org/d3-force) and [D3 curves](https://d3js.org/d3-shape/curve) describe those separate jobs.

[3D Force-Directed Graph](https://github.com/vasturiano/3d-force-graph) is a useful reference for combining 3D layout, camera interaction and Three.js/WebGL. It does not itself preserve the semantics of a Fidelity hyperedge. [Three.js WebGPURenderer](https://threejs.org/manual/pages/webgpurenderer) attempts WebGPU and can fall back to WebGL 2, with documented feature and material differences. Pin and test the actual packages together: using a 3D force graph built on a WebGL renderer does not by itself establish compatibility with that WebGPU renderer or its fallback.

Canvas output needs complementary accessible controls or a structured text view for graph navigation and evidence. A fallback must preserve declared identities, selection and the actual joint relationship, even where its picture or visual density differs. It must expose the current graph and evidence state truthfully; a visually appealing approximation cannot authorize a new graph fact.

## Work scheduling and rendering

Evaluate layout independently from drawing. A modest 2D view may use a CPU layout and Canvas. A dense 3D view may use CPU layout with GPU rendering. WebGPU compute is a candidate when layout or data preparation is sufficiently large to benefit after setup, upload, scheduling and any required readback. Do not assume an all-pairs shader beats D3's approximated many-body force. D3 documents its [Barnes–Hut approximation](https://d3js.org/d3-force/many-body); benchmark the actual algorithms with equivalent visual requirements.

For editor responsiveness, compare main-thread work with bounded layout or geometry preparation in a Web Worker where the particular WebView supports it. An `OffscreenCanvas` or worker-owned WebGPU device is an additional capability to probe, not a universal property of WREN. Message buffers have explicit transfer, copy and ownership consequences. Cancellation must retire the result safely even if the worker or GPU has not yet completed its work. The owner validates the revision and generation again when a result returns.

Keep static or slowly changing geometry on the GPU when that is supported and useful. A camera update can often change a small uniform or scene transform without rebuilding the entire graph. That optimization must not leave labels, picking or proof details tied to a different accepted revision. Device loss, a resized surface, an updated layout or changed shader capability can require recreation or invalidation.

## Deployment choices

| Form | Frontend | Backplane and distribution |
|---|---|---|
| **WREN desktop** | Shared Fickl/Ranvier visual in the system WebView | Build and package a native host for each supported platform. A local backend can reach compiler services and platform resources under its contracts. |
| **Browser with a service** | The same admitted visual in a browser | Deploy a compatible service and explicit authenticated transport. Availability and latency are part of that deployment. |
| **Browser-only visual** | The shared frontend with locally available data and logic | No native host for the chosen workload. Wider Clef application logic needs an implemented and validated browser target. |
| **Mobile application** | The shared frontend inside an Android WebView or Apple WKWebView | Build and validate each platform's host, app package, lifecycle, native bindings and selected backplane route. |

The shared frontend avoids maintaining entirely different visual code for each host. It does not remove the native packaging work when an application chooses a local native backplane. A browser-only deployment and a browser connected to a service are useful comparisons for deciding which parts of an application actually need a local host.

An independently packaged application uses the same boundary with its own app identity, assets, capability profile, update policy and lifecycle. A financial application may select a bounded 2D chart while an engineering tool selects an orbiting graph; neither needs to package Atelier. A migration from an Electron application can retain suitable web components and adapt native operations behind typed WREN contracts. Inventory Node or Electron-specific APIs, host integrations, filesystem assumptions and update mechanisms before promising portability. Compare startup, end-to-end interaction and frame latency, CPU and GPU memory, installer size and workload behavior on the same devices. The system WebView can reduce bundled runtime duplication, while its availability and behavior remain deployment dependencies. Publish the measured result for the named versions and workload.

### Actual WebView capabilities

WrenHello's initial desktop target is Linux WebKitGTK. Check the actual library version, GPU/driver, WebView settings and origin used by the embedded application. WebKitGTK accelerating ordinary page composition or supporting WebGL does not establish WebGPU support in that WebView.

Windows WebView2, Apple WKWebView and Android WebView each have distinct host APIs and runtime configurations. WebKit announced WebGPU in Safari 26 for iOS and iPadOS, and Apple WebKit guidance identifies default WKWebView support on iOS 26. Chromium's Android WebGPU and compatibility-mode work includes WebView. These are starting points for supported-platform tests, not a guarantee for every installed device and driver. [WebKit](https://webkit.org/blog/17333/webkit-features-in-safari-26-0/), [WKWebView guidance](https://developer.apple.com/forums/thread/770862), [Chromium compatibility-mode intent](https://groups.google.com/a/chromium.org/g/blink-dev/c/N3RlLGCOTJ4/m/loneRTlPBwAJ).

WebGPU is a secure-context API. Test the actual embedded URL or custom scheme rather than assuming it is treated as secure. Probe `navigator.gpu`, request an adapter and device, inspect the needed limits and features, validate a shader and render a small scene. Test mobile suspension/resumption, desktop window close/reopen, resize, device loss and a fallback that preserves the visual's essential meaning. Record the exact WebView build and selected GPU where that information is available. [WebGPU specification](https://www.w3.org/TR/webgpu/).

### Native GPU work

Some workloads justify native GPU access: a native simulation may already own its data and compute resources, or a target may need capabilities unavailable to its WebView. The native host can send a bounded result or visualization projection to the frontend through an admitted BAREWire contract. If the visual needs to display a native GPU texture directly, establish a host-specific sharing mechanism, device and adapter compatibility, ownership, synchronization, presentation, security and resource-retirement rules. WebGPU's browser API does not promise that a native allocation is automatically a WebGPU texture.

Native GPU work is a useful comparison path rather than a prerequisite for a WebGPU canvas. The first goal is to prove the frontend path in the actual WebView. Later measurements can decide whether a native computation or shared-texture integration solves a concrete workload problem.

## Clef and JSIR horizon

Fickl and Ranvier are the selected F#-to-JavaScript baseline for WREN. Xantham's current generated surface targets Fable-family JavaScript; the selected Fickl and package versions must be demonstrated together. Do not attribute a native Clef compiler target to the generator merely because the resulting F# looks familiar.

The later Clef path is to establish the needed language surface, JavaScript/JSX lowering and platform bindings through Composer's JSIR direction, then compare the same frontend contract and scenes with the Fickl baseline. A first port can retain the browser's WebGPU implementation and explicit WGSL source. Compiling a Clef computation into a shader is a distinct GPU compiler target with its own numeric, memory and semantic correspondence obligations.

The proof-bearing horizon should connect an admitted source value and its constraints through serialization, JavaScript conversion, WebGPU upload, WGSL layout and the generated operation. Each layer contributes a specific premise. The evidence for a native Clef source program reaching a GPU operation is an implementation and verification goal, not a consequence of generating TypeScript bindings.

## Integration acceptance

1. Pin the Xantham generator and TypeScript declaration inputs, Fickl, Ranvier and the actual WebView. Generate the minimum D3 and WebGPU surface and record findings and hand-authored adapters.
2. In a regular browser, request an adapter and render a minimal canvas. Exercise failed acquisition and a missing required capability. Then run the same frontend in WrenHello's Linux WebView.
3. Send one typed graph snapshot from the native host through the available bridge. Retain the current script-message limit until the selected BAREWire-based WrenHello transport is integrated and tested. Confirm revision and participant identities at the frontend.
4. Apply the update as one Ranvier batch, render a selectable hyperedge, and distinguish the graph fact from D3's layout. Rotate the camera without requesting a new graph or recomputing the layout.
5. Delay a layout result, accept a newer revision, then deliver the old result. Repeat across a device restart. Verify stale work cannot replace the accepted scene and that resources are retired safely.
6. Compare Canvas, WebGL 2 and WebGPU on the same admitted scene where available. Report the selected path and any unsupported capability. Measure UI input and frame latency while layout or preparation runs, including a worker comparison where supported.

The resulting evidence is bound to the versions, device, driver and host actually tested. The wider architecture and proof-carrying claims advance through their own gates.
