# Atelier

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![License: Commercial](https://img.shields.io/badge/License-Commercial-orange.svg)](Commercial.md)
[![Status: Design](https://img.shields.io/badge/Status-Preliminary%20Design-yellow.svg)](./docs/)

Atelier is the planned development environment for the Fidelity ecosystem and the commercial home of the Clef toolchain. It is in preliminary design; the capabilities described here are goals, not implemented Atelier features.

Atelier is intended to help developers explore the Clef Compiler Service's Program Semantic Graph (PSG), inspect compilation and proof obligations, and debug Clef applications. The IDE presents facts supplied by the compiler and runtime; it does not establish language semantics itself.

## Architecture and direction

Atelier follows the [WREN Stack](https://speakez.tech/blog/wren-stack/) (**W**ebView, **R**eactive, **E**mbedded, **N**ative): reactive views and embedded assets paired with native services. The long-term goal is to use Clef to build Atelier itself.

- **Reactive views:** [Ranvier](https://github.com/ranvier-org/Ranvier), Shayan Habibi's reactive computation library, is the selected model. Its native WebCore backend is being investigated for native views and DOM interop. The browser direction uses [Fickl](https://github.com/shayanhabibi/fickl), Habibi's performance-focused compiler fork, to compile shared views to JavaScript.
- **Native host:** WrenHello demonstrates a Composer-compiled Clef host without a .NET runtime. The initial Ranvier Native investigation uses a .NET NativeAOT application and packaged WebCore backend. Connecting it to Clef services and establishing a Clef-native frontend remain integration work.
- **Compiler services:** ClefAutoComplete (CAC) is a .NET language server accessed over LSP. Atelier is intended to consume compiler-produced PSG data and build artifacts across explicit boundaries. The BAREWire-generated PSG is the authoritative graph representation; JSON phase files are inspection projections.
- **Self-hosting:** Compiling the Clef Compiler Service with Clef and supporting Clef views are future steps. Composer's JSIR pathway is a later browser target requiring its own conformance work.

The earlier WrenHello frontend remains a host and bridge reference. Its frontend stack is historical context, not Atelier's selected direction. See the [reactive frontend design](docs/06_reactive_frontend.md) for the native and browser paths.

## Planned capabilities

- **Compiler-aware editing:** Use CAC over LSP for diagnostics and language services. CodeMirror 6 is a candidate editor; a Lezer grammar would provide coloring, while structural and semantic facts come from the compiler.
- **PSG and compilation views:** Explore the authoritative graph, nanopass projections, obligations, and related MLIR and LLVM artifacts. Atelier witnesses these facts rather than recomputing them.
- **Proof obligations:** Present compiler-produced obligations, solver results and their association with graph nodes.
- **Delimited continuation debugging:** Inspect suspension, frames and control flow using compiler-supplied graph and layout facts. This depends on upstream compiler and runtime support.
- **Visualization and tooling:** Explore coordinated views for editing, graphs, debugging and terminals, including GPU-assisted visualization where appropriate. Isolation, performance and resource costs must be measured for each host.

CodeMirror, Dockview, D3.js and xterm.js are candidates for the browser/WebView path. Their availability in a native WebCore view requires an integration or shared implementation. BAREWire is intended for typed communication where components cross a boundary; the delivery and lifetime contracts still need to be established for each integration.

## Documentation

The design documents contain more detail; some retain earlier frontend explorations and should be read in light of the Ranvier and Fickl direction above.

- [Architecture](docs/00_architecture.md) and [WebView abstraction](docs/01_webview_abstraction.md)
- [Reactive frontend](docs/06_reactive_frontend.md), [earlier component design](docs/02_solid_components.md), and [Lezer parsing](docs/07_lezer_parsing.md)
- [Unique features](docs/03_unique_features.md), [multi-WebView architecture](docs/04_multi_webview.md), and [tooling integration](docs/08_tooling_integration.md)
- [Graphics and visualization](docs/05_webgpu.md), [hypergraph visualization](docs/12_hypergraph_visualization.md), [integrity](docs/13_visualization_integrity.md), and [roadmap](docs/15_visualization_roadmap.md)
- [Flow loss analysis](docs/09_flow_loss_analysis.md), [WREN WebGPU bindings](docs/14_wren_webgpu_bindings.md), and [Xantham binding playbook](docs/16_xantham_gpu_binding_playbook.md)

## Related projects

- [Ranvier](https://github.com/ranvier-org/Ranvier) — reactive computation and view infrastructure.
- [Fickl](https://github.com/shayanhabibi/fickl) — the selected compiler for the browser frontend.
- [Composer](https://github.com/FidelityFramework/Composer) — Clef AOT compiler; Alex witnesses the PSG into MLIR.
- [ClefAutoComplete](https://github.com/FidelityFramework/ClefAutoComplete) — the Clef language server.
- [BAREWire](https://github.com/FidelityFramework/BAREWire) — schema-driven binary serialization and platform description.
- [Fidelity.WebKit](https://github.com/FidelityFramework/Fidelity.WebKit) — generated bindings for platform WebView integration.

## License

Dual-licensed under Apache 2.0 and a commercial license. See [LICENSE](LICENSE) and [Commercial.md](Commercial.md).
