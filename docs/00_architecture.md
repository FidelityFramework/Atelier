# 00 - WREN Architecture

## The WREN Stack

**W**ebView + **R**eactive + **E**mbedded + **N**ative

WREN represents a fundamental rethinking of desktop application architecture. Rather than embedding a full browser runtime (Electron) or constraining ourselves to native-only UI toolkits (GTK widgets, Qt), WREN leverages the platform's existing WebView infrastructure with a native F# backend.

## Why WREN?

### The Electron Problem

Electron applications bundle Chromium (~150MB) and Node.js (~30MB) into every application. A "Hello World" in Electron ships as a 180MB+ binary that consumes 300-500MB of RAM at runtime.

This is computationally irresponsible.

### The Native-Only Problem

Pure native toolkits (GTK, Qt, native F#/WinForms) offer efficiency but lack the rich, expressive UI capabilities that web technologies provide. Building a sophisticated code editor UI in pure GTK is possible but requires enormous effort.

### The WREN Solution

Every modern operating system ships with a WebView component:
- **Linux**: WebKitGTK (GTK4 integration)
- **macOS**: WKWebView (WebKit)
- **Windows**: WebView2 (Chromium-based, ships with Windows 11, downloadable for 10)

These WebViews are already present, already maintained, and already consuming memory for other applications. WREN uses them directly, avoiding the Electron tax while gaining web UI capabilities.

## Architecture Layers

### Layer 1: Platform WebView

The lowest layer provides raw WebView access:

```
┌─────────────────────────────────────────────────────┐
│                Platform WebView                      │
├─────────────────┬─────────────────┬─────────────────┤
│   WebKitGTK     │   WKWebView     │   WebView2      │
│   (Linux)       │   (macOS)       │   (Windows)     │
├─────────────────┴─────────────────┴─────────────────┤
│              Common Capabilities:                    │
│  - Load HTML/URL                                    │
│  - Execute JavaScript                               │
│  - Receive script messages                          │
│  - Custom protocol handlers                         │
│  - DevTools access                                  │
└─────────────────────────────────────────────────────┘
```

Each platform provides these capabilities through different APIs, but the semantics are remarkably similar.

### Layer 2: F# Native Abstraction

WRENEdit abstracts platform differences using F# discriminated unions and conditional compilation:

```fsharp
type WebViewHandle =
    | WebKitGTK of nativeint  // GtkWidget*
    | WKWebView of nativeint  // WKWebView*
    | WebView2 of nativeint   // ICoreWebView2*

type WebViewConfig = {
    Title: string
    Width: int
    Height: int
    DevTools: bool
    Transparent: bool
}

// Platform-specific initialization
#if LINUX
let create config = WebKitGTK (webkitgtk_create config)
#elif MACOS
let create config = WKWebView (wkwebview_create config)
#elif WINDOWS
let create config = WebView2 (webview2_create config)
#endif
```

### Layer 3: IPC Protocol (BAREWire)

Communication between F# Native backend and WebView frontend uses BAREWire - a binary typed protocol:

```
┌──────────────┐                    ┌──────────────┐
│   F# Native  │   BAREWire IPC     │   WebView    │
│   Backend    │◄──────────────────►│   Frontend   │
│              │                    │              │
│  - LSP       │  Binary messages   │  - UI State  │
│  - PSG       │  Type-safe         │  - Editors   │
│  - Debug     │  No JSON overhead  │  - Panels    │
└──────────────┘                    └──────────────┘
```

Why not JSON?
- Parsing overhead on every message
- No type safety (runtime errors)
- Verbose encoding (strings for everything)

BAREWire provides:
- Zero-copy deserialization where possible
- Compile-time type checking
- Minimal wire format

### Layer 4: Reactive Frontend (SolidJS)

The WebView runs a SolidJS application with fine-grained reactivity:

```typescript
// SolidJS - components run ONCE, only signals update
const [fileContent, setFileContent] = createSignal("")
const [diagnostics, setDiagnostics] = createSignal<Diagnostic[]>([])

// This component body runs once
function Editor() {
  // Only the signal reads re-execute
  return (
    <CodeMirror
      value={fileContent()}
      extensions={[lspExtension(diagnostics())]}
    />
  )
}
```

This is fundamentally different from React's re-render model:
- **React**: Component functions re-execute on every state change
- **SolidJS**: Component functions run once; signal reads are tracked and update surgically

For an editor that receives thousands of updates per second (keystrokes, LSP responses, debug events), this efficiency matters.

## Data Flow

```
User Action (keystroke, click, etc.)
        │
        ▼
┌───────────────────┐
│  SolidJS Signal   │  (Frontend state update)
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  BAREWire Encode  │  (Serialize to binary)
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Script Message   │  (WebView → Native)
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  F# Native Core   │  (Process: LSP, PSG, Debug)
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  BAREWire Encode  │  (Serialize response)
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Execute Script   │  (Native → WebView)
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  SolidJS Signal   │  (Frontend state update)
└───────────────────┘
```

## Memory Model

WRENEdit's memory footprint:

| Component | Approximate Size |
|-----------|-----------------|
| F# Native binary | ~5MB |
| WebKitGTK shared libs | Already loaded by system |
| WebView process | ~30-50MB |
| SolidJS app | ~2MB |
| CodeMirror | ~3MB |
| **Total** | **~40-60MB** |

Compare to:
- VSCode: 300-500MB
- Atom (RIP): 400-700MB
- Sublime Text: 80-100MB

## Thread Model

The multi-WebView architecture enables true thread isolation:

```
┌─────────────────────────────────────────────────────┐
│                    Main Process                      │
│                    (F# Native)                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │   LSP       │  │   Debug     │  │   Build     │ │
│  │   Thread    │  │   Thread    │  │   Thread    │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘ │
└─────────┼────────────────┼────────────────┼────────┘
          │                │                │
          ▼                ▼                ▼
    ┌───────────┐    ┌───────────┐    ┌───────────┐
    │  Editor   │    │  Debug    │    │  Build    │
    │  WebView  │    │  WebView  │    │  Output   │
    │           │    │           │    │  WebView  │
    └───────────┘    └───────────┘    └───────────┘
```

Each WebView runs in its own process (WebKit/Chromium architecture). Heavy operations (debugging, build monitoring) cannot freeze the editor UI.

## Next Steps

- [01_webview_abstraction.md](./01_webview_abstraction.md) - Platform abstraction patterns from WRY
- [02_solid_components.md](./02_solid_components.md) - CodeMirror and Dockview integration
