# 15 - WREN Graphics and Visualization: Delivery Roadmap

WREN aims to support responsive applications across desktop, browser and mobile hosts. Ranvier's native WebCore backend and a separate Fickl-compiled browser frontend can share visual and data contracts while using target-specific DOM and graphics integrations. Xantham's TypeScript bindings serve the browser path, and a native Clef backplane supplies application services where needed. Atelier is the in-house example that brings together an editor, multiple execution domains, compiler evidence and a dense interactive hypergraph. The same framework should also serve financial dashboards, scientific instruments, observability tools, design applications and other products.

The browser experiments in this roadmap are one deployment track. Establish the native track alongside them: pin Ranvier Native and its WebCore ABI, mount and update a view, present pixels and input on the chosen host, and then demonstrate the desired native graph renderer and GPU interface. Compare the two tracks against the same data, selection and responsiveness cases without assuming their rendering APIs or language runtimes are identical.

This roadmap turns the architecture into testable increments. It is an opportunity to build a lighter and more responsive alternative for suitable Electron applications. The scale of that advantage must be measured for the selected application, features and deployment; a system WebView or native host does not by itself establish a performance result.

## The first two applications

Use two bounded examples with a shared technical foundation and distinct domain contracts:

1. **A financial chart.** Begin with exact amounts and currencies, a dataset revision, a BAREWire update and a coherent Ranvier snapshot. Draw bounded display coordinates and retain exact labels. The acceptance case exercises correct data, rounding and selection without claiming that a visual approximation is an authoritative monetary calculation.
2. **An Atelier hypergraph inspector.** Begin with a real bounded graph projection or a clearly labeled fixture containing node identities, one joint hyperedge, incidence roles and an obligation. Render, select and inspect that relation across a graph revision. This case exercises dense graph geometry, graph/evidence identity and editor interaction.

The applications can share WebGPU and D3 bindings, renderer lifecycle machinery and host-capability probes. Their domain meaning, accessible presentation and exactness requirements remain explicit. A finance application should not need to import Atelier's compiler graph model in order to render a chart.

## Horizon 1: establish the vertical slice

### A. Bind and run the actual frontend

Pin the Fickl compiler, Ranvier packages, Xantham generator and selected TypeScript declaration sources. Exercise existing D3 bindings before designing a replacement. Generate or establish the minimal WebGPU binding surface needed for device acquisition, buffers, a shader, a render pipeline and canvas presentation. Record widened, unsupported and hand-authored mappings. Compile the F# consumer with the exact Fickl build and run the emitted JavaScript in the tested host.

For the browser track, start from WrenHello's existing F#/Partas.Solid, Fable and WebKitGTK integration. Its current script-message bridge is an implementation baseline with documented message bounds. Migrate that frontend toward Fickl and Ranvier, retaining a view adapter only after testing its batching, event and disposal behavior. Establish the chosen BAREWire projection and transport as an integrated path rather than inferring that path from the cross-target BAREWire libraries alone.

**Acceptance:** identify the exact source, generated bindings, packages, compiled output, host and dependencies. Demonstrate one real reactive update in a regular browser and in WrenHello's WebView, or report the exact failing host capability. This is the first cross-platform signal; a Node-based unit test cannot by itself establish WebView or GPU behavior.

### B. Make one graph relationship inspectable

Use three buffers, each requiring four units, under a capacity of ten. Retain one joint capacity obligation with its participant identities, incidence roles, liveness premises, source and proof status. The view shows the complete predicate and associates every visible curve with that one relationship. Selecting the obligation or a participant opens the corresponding typed fact and evidence.

Add an orbiting camera and a bounded 2.5D view with explicitly labeled layers. Include a flat or structured inspection mode. Camera movement changes presentation state and must not rewrite the graph or imply that its position is semantic evidence. The graph producer remains authoritative for facts; CCS/Baker graph-export and stable identity/delta behavior require their own owning contract.

**Acceptance:** a reviewer can identify all three participants, the joint `4 + 4 + 4 <= 10` condition, its premises and its actual status. Hiding a participant must be shown as a filtered or incomplete view, never as a different binary obligation. A missing source or evidence reference is reported as missing.

### C. Compare rendering and layout choices

Render the same visual snapshot through Canvas 2D and at least one available GPU path. Evaluate WebGPU and WebGL 2 on named hosts. Compare D3's standard 2D layout with a suitable 3D or layered layout. A Three.js-based prototype and a WebGL graph library are controls, with exact versions and interoperability tested. Test the required shader, picking, labels, camera controls and resource behavior rather than assuming that an available API or fallback supports the entire scene.

GPU compute for layout is a separate experiment from GPU rendering. D3's force layout is an engineering baseline with its own approximation and scheduling choices. Record setup, transfers, layout iterations, synchronization and readback when comparing it with a GPU kernel. Preserve positions on the GPU when appropriate; retrieve them only when a real consumer requires them. Any numerical layout output is presentation state and cannot change the underlying graph.

**Acceptance:** the tested renderer identifies every visible object by the correct graph and revision, supports a useful interaction cycle, and shows an accessible or structured alternative. Report unavailable capabilities and fallback behavior. Make no generalized node-count or speedup claim from an unmeasured example.

### D. Exercise the difficult boundaries

Use bounded scenarios that distinguish the contracts:

| Case | Expected result |
|---|---|
| R1 layout completes after R2 is current | R1 cannot replace R2's visual snapshot. |
| A completion from device generation G1 arrives after G2 is mounted | It cannot publish into G2 or authorize reuse of G2's resources. |
| A delta arrives against the wrong base revision | Request a replacement or follow the declared recovery policy. |
| A joint hyperedge is filtered to two of three participants | The view indicates the omission and preserves access to the complete fact. |
| Two `vec3<f32>` values are packed without the required WGSL array stride | The layout check rejects or performs the declared conversion. |
| An exact monetary amount is converted to a display coordinate | The chosen scaling/conversion is checked; the exact amount and currency remain available for its label. |
| A buffer is still in flight when its owner disposes | Retirement follows the actual GPU and host completion contract. |
| WebGPU is unavailable or a device is lost | The view follows its declared fallback or reports the unsupported capability, and can recover under a new generation where supported. |

Also test malformed and truncated BAREWire data, unsupported message versions, resource limits, shader compilation failure and dataset/selection changes during preparation. Verify input, picking and labels against the same admitted revision. The actor/mailbox serializes renderer decisions; Ranvier batching establishes a coherent local update; GPU queue completion governs resource retirement. Each needs separate evidence.

## Horizon 2: productize across hosts

Develop the reusable **Fidelity.WebGPU** browser-facing library from Xantham-generated declarations plus authored capability, buffer-layout, conversion and lifecycle contracts. Retain a clean boundary between the faithful low-level API and stronger helpers. An application can choose the operations and checks it needs, and uses outside UI can share the same library.

Extend the fidelity of graph updates where the producing services provide it. Compare revision-aware snapshots and validated deltas. Define bounds, queueing, coalescing and recovery so that an overloaded or disconnected frontend never silently loses structural facts. Preserve unrelated reactive and visual work only when its dependencies support reuse.

Qualify a small named host matrix:

- Linux WebKitGTK in WrenHello and a regular browser on the same machine.
- Windows WebView2 and its supported browser configuration.
- Apple WKWebView on supported desktop and mobile versions.
- Android System WebView on selected devices, alongside the browser as a separate test.

For every host, record OS, architecture, WebView version, device/GPU and driver, origin and secure-context treatment, adapter, enabled features, limits, shader compilation, device loss and lifecycle behavior. Check worker and optional OffscreenCanvas support independently. Browser functionality is a lead for the embedded-host test, not its result. Report behavior and fallbacks per profile rather than promising identical GPU capabilities on every platform.

Separate reusable WREN frontend assets from packaging and native integrations. A browser-only visualization can serve data available to it. A browser-plus-service product adds a network protocol, authentication and availability requirements. A desktop or mobile WREN package adds a platform-specific host and, when needed, its native Clef backplane and Farscape/platform bindings. Shipping one source frontend does not remove the need to build and qualify those native artifacts.

## Measure responsiveness and computational cost

Benchmark named application builds under a published workload. Include WREN and at least one named Electron-based editor or application with comparable visible work and features. Record source and dependency revisions, configuration, host runtime, OS, CPU, memory, GPU/driver, display resolution, device-pixel ratio and power mode. Include all processes and shared services under a consistent accounting policy.

Measure cold and warm startup, installed/download size, idle and active CPU, resident and GPU memory, input-to-visible-response latency and frame-time distribution. Include median, p95 and p99 where meaningful, dropped or missed frames, and the time from an accepted graph update to a coherent stable view. Measure layout, projection, encode/decode, data transfer, upload and GPU work separately enough to locate a bottleneck. Record graph node, hyperedge and incidence counts, labels, geometry, camera movement and update frequency.

Run idle editing, typing during a layout, sustained graph updates, camera orbit, selection/picking, resize, renderer disposal and recovery from device loss. Compare bounded main-thread preparation with a Web Worker and with native work where available. Record copying or transfer, queueing and backpressure costs. A worker, WebView or GPU may improve responsiveness, but its existence does not establish thread isolation or a frame-time benefit.

Graph topology and interaction matter as much as node count. Use a small inspectable fixture, a realistic compiler projection and larger synthetic topologies with stated arity and connectivity. For approximate layouts record the algorithm, seed or reproducibility policy, convergence criterion and the tolerated positional variation. Compare semantic correctness separately from geometric differences. Publish both wins and regressions; avoid predicting a fixed multiple of Electron performance or a universal GPU speedup.

## Horizon 3: Clef-native frontend and retained evidence

As Clef's JavaScript/JSIR and JSX pathways become available, compile selected shared frontend behavior and bindings through Clef. Compare its emitted artifact with the Fickl baseline on the same BAREWire messages, reactive updates, visual identity, numeric conversions, device lifecycle and target hosts. Carry forward Fickl's established application contracts while identifying any changed language semantics and representation assumptions.

For proof-bearing GPU work, specify what Tier 1 can establish structurally: selected graph participants and types, declared buffer layout, bounds and resource-use premises where the language can express them. Numeric accuracy, WGSL operational behavior, device features and host/target correspondence require their appropriate additional evidence. Bind each result to the generated JavaScript, shader, graph revision and platform configuration. A correct source proof or API type does not automatically prove the delivered GPU operation.

The intended outcome is a reusable WREN/Fidelity capability that supports ordinary applications and demanding compiler tooling. Atelier supplies a stringent end-to-end case. Wider product claims follow from tests of the relevant application, deployment profile and delivered artifacts.

## Related documents

- [Visualization strategy](05_webgpu.md)
- [Hypergraph visualization and interaction](12_hypergraph_visualization.md)
- [Visualization integrity and coordination](13_visualization_integrity.md)
- [WREN WebGPU bindings and host integration](14_wren_webgpu_bindings.md)
