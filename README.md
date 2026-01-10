# WRENEdit

**A Bespoke Editor for the Fidelity Ecosystem**

WRENEdit is a native code editor built on the WREN Stack (WebView + Reactive + Embedded + Native), designed specifically for F# Native development with the Fidelity framework.

## Vision

Modern development environments are computationally profligate. Electron apps bundle entire browser runtimes, consuming gigabytes of memory to display text. VSCode, while feature-rich, carries the weight of universal compatibility at the cost of specialized excellence.

WRENEdit takes a different path: **a lean, native foundation with a reactive web frontend**, purpose-built for compiler development and the unique demands of the Fidelity ecosystem.

### What Makes WRENEdit Different

| Capability | VSCode/Electron | NeoVim | WRENEdit |
|------------|-----------------|--------|----------|
| Memory footprint | ~500MB+ | ~50MB | ~80MB (native + WebView) |
| Startup time | 2-5 seconds | <100ms | <200ms |
| PSG visualization | Plugin (limited) | Not practical | Native D3 integration |
| Delimited continuation debugging | Not supported | Not supported | First-class support |
| Multi-WebView architecture | Single process | N/A | Multiple isolated WebViews |
| WebGPU compute | Limited | N/A | Direct access |

### Core Principles

1. **Computational Responsibility** - Every byte of memory, every CPU cycle is justified
2. **Compiler-Aware Tooling** - Deep integration with Firefly's PSG and MLIR pipeline
3. **Native Performance** - F# Native backend, no managed runtime
4. **Web-Class UI** - SolidJS reactivity with CodeMirror 6's proven editor foundation
5. **Multi-WebView Architecture** - Thread isolation for debugging, monitoring, visualization

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     WRENEdit Application                      │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Main       │  │  Debug      │  │  PSG        │         │
│  │  WebView    │  │  WebView    │  │  WebView    │  ...    │
│  │             │  │             │  │             │         │
│  │ CodeMirror  │  │ Continuation│  │ D3 Graph    │         │
│  │ + Dockview  │  │ Inspector   │  │ Renderer    │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                 │
│         └────────────────┼────────────────┘                 │
│                          │ BAREWire IPC                     │
├──────────────────────────┼──────────────────────────────────┤
│                    F# Native Core                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ LSP Client   │  │ PSG Engine   │  │ Debug Engine │      │
│  │ (FSNAC)      │  │ (Firefly)    │  │ (Delim Cont) │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
├─────────────────────────────────────────────────────────────┤
│                  Platform Abstraction                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ WebKitGTK    │  │ WKWebView    │  │ WebView2     │      │
│  │ (Linux)      │  │ (macOS)      │  │ (Windows)    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

## Technology Stack

### Frontend (WebView)
- **SolidJS** - Fine-grained reactivity, components run once
- **CodeMirror 6** - Modern editor with Lezer parsing, LSP support
- **solid-dockview** - VS Code-style panel docking with floating windows
- **D3.js** - PSG graph visualization
- **xterm.js** - Terminal emulation with WebGL renderer

### Backend (F# Native)
- **Firefly-compiled** - No .NET runtime dependency
- **BAREWire IPC** - Binary typed messaging, no JSON overhead
- **FSNAC LSP** - F# Native language server
- **PTY integration** - Native terminal support

### Platform Layer
- Inspired by WRY/Tauri abstraction patterns
- Conditional compilation for platform-specific WebView implementations
- Script message handlers for bidirectional communication

## Documentation

See the [docs/](./docs/) folder for detailed documentation:

- [00_architecture.md](./docs/00_architecture.md) - WREN architecture deep dive
- [01_webview_abstraction.md](./docs/01_webview_abstraction.md) - Platform WebView lessons from WRY
- [02_solid_components.md](./docs/02_solid_components.md) - Partas.Solid bindings for CodeMirror and Dockview
- [03_unique_features.md](./docs/03_unique_features.md) - Delimited continuations, PSG visualization
- [04_multi_webview.md](./docs/04_multi_webview.md) - Multi-WebView architecture
- [05_webgpu.md](./docs/05_webgpu.md) - WebGPU compute integration

## Building

*Coming soon - requires Firefly compiler and WREN Stack infrastructure*

```bash
# Build the native backend
firefly compile WRENEdit.fidproj

# Build the frontend
cd frontend && npm run build

# Run
./WRENEdit
```

## Related Projects

| Project | Description |
|---------|-------------|
| [Firefly](https://github.com/user/Firefly) | F# Native AOT compiler |
| [WRENHello](https://github.com/user/WRENHello) | WREN Stack proof of concept |
| [FSNAC](https://github.com/user/FsNativeAutoComplete) | F# Native language server |
| [Alloy](https://github.com/user/Alloy) | Native standard library |

## License

MIT
