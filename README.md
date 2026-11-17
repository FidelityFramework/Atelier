# Atelier

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![License: Commercial](https://img.shields.io/badge/License-Commercial-orange.svg)](Commercial.md)
[![Status: Design](https://img.shields.io/badge/Status-Preliminary%20Design-yellow.svg)](./docs/)

Atelier is a planned development environment for the Fidelity ecosystem and the commercial home of the Clef toolchain. The project is in preliminary design, and the features described here have yet to be built.

Atelier is intended to run on Windows, Linux and macOS.

Atelier is designed to provide a native app experience with responsive REPL and hot module replacement (HMR) modes. Differential compilation is intended to deliver near-immediate feedback as code changes, including code resolution diagnostics and proof obligations. Atelier will also help developers explore the Clef Compiler Service's Program Semantic Graph (PSG), inspect compilation, and debug Clef applications. The compiler supplies the semantic information Atelier displays.

## Architecture and direction

Atelier follows the [WREN Stack](https://speakez.tech/blog/wren-stack/) (**W**ebView, **R**eactive, **E**mbedded, **N**ative): reactive views and embedded assets paired with native services. The long-term goal is to use Clef to build Atelier itself.

- **Reactive views:** Atelier will use [Ranvier](https://github.com/ranvier-org/Ranvier), Shayan Habibi's reactive computation library. Its native path accesses WebCore directly, without compiling the application to JavaScript. [Fickl](https://github.com/shayanhabibi/fickl), also developed by Habibi, is the selected compiler where JavaScript is needed.
- **Native host:** WrenHello demonstrates a Clef host compiled by Composer that runs without a .NET runtime. The initial Ranvier Native investigation uses a .NET NativeAOT application and a packaged WebCore backend. Connecting it to Clef services and building a Clef-native frontend are still to come.
- **Compiler services:** ClefAutoComplete (CAC) is a .NET language server that Atelier can access over LSP. Atelier will also read PSG data and build artifacts produced by the compiler. The BAREWire-generated PSG is the authoritative graph representation; JSON phase files provide views for inspection.
- **Self-hosting:** Future work includes compiling the Clef Compiler Service with Clef and writing views in Clef.

WrenHello provides a reference for the host and its communication bridge. See the [reactive frontend design](docs/06_reactive_frontend.md) for more detail.

```mermaid
flowchart TB
    subgraph atelier["Atelier views"]
        editor["Editor and diagnostics"]
        repl["REPL and HMR"]
        graph["PSG and compilation views"]
        debug["Debugger and tools"]
    end

    native["Native views: Ranvier → WebCore"]

    subgraph services["Compiler and runtime services"]
        cac["ClefAutoComplete · LSP"]
        ccs["Clef Compiler Service"]
        composer["Composer"]
        runtime["Application runtime"]
    end

    editor & repl & graph & debug --> native
    editor <-->|"Edits and diagnostics"| cac
    repl -.->|"Interactive evaluation and updates"| runtime
    cac --> ccs
    ccs -->|"PSG, resolution and obligations"| graph
    composer -->|"Build artifacts"| graph
    runtime -->|"Runtime information"| debug
```

The diagram shows the intended flow. The native views, service connections and design-time feedback still need to be integrated and validated.

## Planned capabilities

- **Responsive editing:** Support differential compilation and surface code resolution diagnostics and proof obligations with near-immediate feedback as the developer edits. CAC provides language services over LSP. CodeMirror 6 is a candidate editor; a Lezer grammar would provide coloring, while structural and semantic information comes from the compiler.
- **REPL and HMR:** Provide interactive evaluation and hot module replacement with responsive design-time feedback in the native app.
- **PSG and compilation views:** Explore the graph, nanopass outputs, obligations, and related MLIR and LLVM artifacts supplied by the compiler.
- **Proof obligations:** Show compiler-produced obligations and solver results alongside the relevant graph nodes.
- **Delimited continuation debugging:** Inspect suspension, frames and control flow using graph and layout information from the compiler. This depends on compiler and runtime support that is still to come.
- **Visualization and tooling:** Explore coordinated views for editing, graphs, debugging and terminals. Measure responsiveness and resource costs on each host, including comparisons with native editors such as Zed.

CodeMirror, Dockview, D3.js and xterm.js are candidates from earlier designs. Using them with native WebCore views would require integration work or shared implementations. BAREWire is planned for typed communication between components; each integration still needs a defined delivery and lifetime contract.

## Documentation

These documents cover the design in more detail. Some describe earlier frontend and WebGPU experiments; the direction above reflects the choice of Ranvier and Fickl and Ranvier's direct native WebCore path.

- [Architecture](docs/00_architecture.md) and [WebView abstraction](docs/01_webview_abstraction.md)
- [Reactive frontend](docs/06_reactive_frontend.md), [earlier component design](docs/02_solid_components.md), and [Lezer parsing](docs/07_lezer_parsing.md)
- [Unique features](docs/03_unique_features.md), [multi-WebView architecture](docs/04_multi_webview.md), and [tooling integration](docs/08_tooling_integration.md)
- [Graphics and visualization](docs/05_webgpu.md), [hypergraph visualization](docs/12_hypergraph_visualization.md), [integrity](docs/13_visualization_integrity.md), and [roadmap](docs/15_visualization_roadmap.md)
- [Flow loss analysis](docs/09_flow_loss_analysis.md), [WREN WebGPU bindings](docs/14_wren_webgpu_bindings.md), and [Xantham binding playbook](docs/16_xantham_gpu_binding_playbook.md)

## Related projects

- [Ranvier](https://github.com/ranvier-org/Ranvier) — reactive computation and view infrastructure.
- [Fickl](https://github.com/shayanhabibi/fickl) — the selected compiler where JavaScript is needed.
- [Composer](https://github.com/FidelityFramework/Composer) — Clef AOT compiler; Alex witnesses the PSG into MLIR.
- [ClefAutoComplete](https://github.com/FidelityFramework/ClefAutoComplete) — the Clef language server.
- [BAREWire](https://github.com/FidelityFramework/BAREWire) — schema-driven binary serialization and platform description.
- [Fidelity.WebKit](https://github.com/FidelityFramework/Fidelity.WebKit) — generated bindings for platform WebView integration.

## License

Dual-licensed under Apache 2.0 and a commercial license. See [LICENSE](LICENSE) and [Commercial.md](Commercial.md).
