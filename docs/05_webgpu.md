# 05 - WREN Graphics and Interactive Visualization

WREN is a foundation for responsive applications across desktop, browser and mobile hosts. A shared web frontend can communicate with a native Clef backplane through BAREWire and use the host's WebView for presentation. The same visual can also run in a regular browser when its required data and services are available there. Each native deployment still needs its own host, packaging and platform contracts.

Atelier is a demanding in-house example: an editor that can make a live Program Semantic Graph and its proof evidence inspectable. A financial application, scientific dashboard or engineering tool can use the same binding, reactive, graphics and communication foundations with its own domain model. These documents live in Atelier because it brings the pieces together, while the intended capability serves WREN and Fidelity applications more broadly.

## The selected direction

While WREN's frontend is F# compiled to JavaScript, **standardize on Fickl and Ranvier**. Fickl compiles the frontend. Ranvier manages changing state and derived values. Xantham generates F# bindings from the selected TypeScript declarations. BAREWire supplies a shared typed representation across its supported .NET, JavaScript and native Clef compilation paths. Farscape and the desktop packages connect the native host to platform services.

Use Xantham's original D3 binding target for data, layout, scales and curves. Pursue WebGPU as the strategic graphics and compute API, with a proposed **Fidelity.WebGPU** browser-facing library containing faithful generated bindings and a distinct authored layer for resource, numeric and layout contracts. Evaluate WebGL 2 as a graphics compatibility path and Canvas 2D as a reference and practical renderer for bounded views. Choose each based on the product's capability and behavior requirements, rather than promising that every renderer supports every operation.

```text
Application data or CCS/Baker-published graph facts
                         ↓
             BAREWire communication contract
                         ↓
Fickl frontend → Ranvier state → D3 / layout / visual snapshot
                                          ↓
                         Xantham browser bindings
                                          ↓
                          Canvas 2D | WebGL 2 | WebGPU
                                          ↓
                               Browser or WebView
```

The native host and its Farscape bindings provide the platform boundary when a deployment needs them. GPU commands for a WebView visual can remain on the frontend, without a native round trip for each frame. Native GPU computation is a separate path for workloads that justify it. Native GPU texture sharing with a WebView needs explicit host, device, synchronization and lifetime support.

## What makes the design distinctive

Atelier's graph must remain **truthful to the semantic relationships it displays**. A hyperedge is one relationship with its own identity and multiple participants. A 3D glyph with splines, a layered 2.5D view and a textual inspector can all expose the same relationship, its roles, premises and actual evidence status. Layout and animation help a person explore the graph; they do not produce compiler facts or prove obligations.

The integrity boundary extends from source data through BAREWire, reactive updates, host-to-GPU conversion and shader interpretation to the picture and its labels. A chart may deliberately approximate a coordinate while preserving an exact financial amount. A renderer owner can reject a late graph revision or work from an expired device generation. Those are application contracts to implement and test; typed bindings and a valid GPU command do not establish them automatically.

These properties matter beyond Atelier. An independently packaged financial planner, observability tool or engineering application should be able to use the bindings, communication and lifecycle foundations without importing a compiler-specific graph model.

## Documents

1. [Hypergraph visualization and interaction](12_hypergraph_visualization.md) develops the visual model: multi-participant relationships, evidence inspection, D3, 3D camera movement, rotating 2.5D spline views, layout and accessibility.
2. [Visualization integrity and coordination](13_visualization_integrity.md) follows exact values, wire and GPU layouts, Ranvier batches, mailbox ownership, revisions, resource retirement and proof scope through worked cases.
3. [WREN, Xantham and WebGPU bindings](14_wren_webgpu_bindings.md) defines library boundaries, renderer choices, native versus WebView GPU work, deployment profiles and host capability checks.
4. [Delivery roadmap and measurement](15_visualization_roadmap.md) sets the first financial and hypergraph examples, acceptance cases, host matrix, Electron comparison and later Clef/JSIR and proof-bearing horizons.
5. [Xantham binding implementation playbook](16_xantham_gpu_binding_playbook.md) records the declaration inputs, generator workflow, implementation risks and executable checks needed to make the D3 and WebGPU bindings usable.

## Current baseline and next step

BAREWire already provides shared encoding, schema and framing across its supported targets. WrenHello currently demonstrates an F#/Partas.Solid frontend compiled through Fable, bundled into a Linux WebKitGTK host, with a bounded script-message bridge. The selected Fickl/Ranvier frontend, integration of the existing BAREWire foundation with WrenHello's chosen transport, graph export and GPU renderer each need their own acceptance evidence. The future Clef-to-JavaScript/JSIR path is a separate horizon. It does not follow automatically from existing F# bindings.

Start with a bounded financial chart and a genuine multi-participant hyperedge. Generate and compile the minimum D3/WebGPU bindings, run the same frontend in a regular browser and the actual WrenHello WebView, and exercise late revisions, layout conversion, picking, disposal and device loss. Then test a layered spline view and an orbiting camera against the same graph and evidence contract.

Atelier aims to be lighter and more responsive than comparable Electron-based editors. Establish that with named applications, equivalent workloads, pinned builds and measurements of startup, memory, input latency, frame pacing and graph-update time. WebGPU, a system WebView and background workers create opportunities to improve those results; their presence alone is not a benchmark.

See [Multi-WebView architecture](04_multi_webview.md) and [Reactive frontend](06_reactive_frontend.md) for the surrounding editor context.
