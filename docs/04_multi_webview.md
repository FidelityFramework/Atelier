# 04 - Multi-WebView Architecture

## The Single-WebView Problem

Traditional Electron apps and simple WebView wrappers use a single WebView for the entire application:

```
┌─────────────────────────────────────────┐
│           Single WebView                │
│  ┌─────────────────────────────────────┐│
│  │         Application UI              ││
│  │  ┌──────┐ ┌──────┐ ┌──────┐        ││
│  │  │Editor│ │Debug │ │Terminal       ││
│  │  │      │ │Panel │ │      │        ││
│  │  └──────┘ └──────┘ └──────┘        ││
│  └─────────────────────────────────────┘│
│        Single JavaScript Thread         │
└─────────────────────────────────────────┘
```

**Problems:**

1. **Single JavaScript thread** - Heavy operations block the entire UI
2. **Shared memory space** - Memory leaks in one component affect everything
3. **No isolation** - A crash in the debugger crashes the editor
4. **Limited parallelism** - Web Workers help but have communication overhead

## The Multi-WebView Solution

WRENEdit uses multiple WebView instances, each running in its own process:

```
┌─────────────────────────────────────────────────────────────┐
│                    F# Native Coordinator                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Editor    │  │   Debug     │  │  Resource   │  ...    │
│  │   Manager   │  │   Engine    │  │  Monitor    │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
└─────────┼────────────────┼────────────────┼────────────────┘
          │                │                │
          │ IPC            │ IPC            │ IPC
          ▼                ▼                ▼
    ┌───────────┐    ┌───────────┐    ┌───────────┐
    │  Editor   │    │  Debug    │    │  Monitor  │
    │  WebView  │    │  WebView  │    │  WebView  │
    │  (pid 1)  │    │  (pid 2)  │    │  (pid 3)  │
    └───────────┘    └───────────┘    └───────────┘
```

**Benefits:**

1. **Process isolation** - Each WebView runs in its own OS process
2. **True parallelism** - Multiple JavaScript engines run simultaneously
3. **Crash isolation** - A WebView crash doesn't take down others
4. **Memory isolation** - Each WebView has its own heap
5. **Security** - Compromised WebView can't access others' memory

## WebView Roles

### Primary WebView: Editor

The main editor WebView handles:
- Code editing (CodeMirror)
- Panel docking (Dockview)
- File tree
- Search UI
- Most user interactions

```fsharp
type EditorWebView = {
    Handle: WebViewHandle
    Dockview: IDockviewApi
    OpenEditors: Map<string, EditorInstance>
    ActiveFile: string option
}

let createEditorWebView () =
    let webview = WebView.create {
        Title = "WRENEdit"
        Width = 1200
        Height = 800
        DevTools = true
        Transparent = false
    }

    webview.LoadUrl "wrendit://app/editor.html"
    webview.AddMessageHandler "editor" handleEditorMessage

    { Handle = webview; Dockview = None; OpenEditors = Map.empty; ActiveFile = None }
```

### Debug WebView

A separate WebView for debugging:

```fsharp
type DebugWebView = {
    Handle: WebViewHandle
    ActiveSession: DebugSession option
    Breakpoints: Set<SourceLocation>
    Continuations: Map<ContinuationId, ContinuationInfo>
}

let createDebugWebView () =
    let webview = WebView.create {
        Title = "WRENEdit - Debugger"
        Width = 800
        Height = 600
        DevTools = true
        Transparent = false
    }

    webview.LoadUrl "wrendit://app/debugger.html"
    webview.AddMessageHandler "debug" handleDebugMessage

    { Handle = webview; ActiveSession = None; Breakpoints = Set.empty; Continuations = Map.empty }
```

**Why separate?**

- Debugging involves heavy computation (stepping, evaluating expressions)
- Real-time updates (variable watches, call stacks)
- Potential for runaway debug targets
- Should never freeze the editor

### PSG Visualization WebView

D3-based graph rendering in its own WebView:

```fsharp
type PSGWebView = {
    Handle: WebViewHandle
    CurrentPSG: PSG option
    CurrentPhase: int
    SelectedNode: NodeId option
}

let createPSGWebView () =
    let webview = WebView.create {
        Title = "WRENEdit - PSG Viewer"
        Width = 1000
        Height = 700
        DevTools = true
        Transparent = false
    }

    webview.LoadUrl "wrendit://app/psg-viewer.html"
    webview.AddMessageHandler "psg" handlePSGMessage

    { Handle = webview; CurrentPSG = None; CurrentPhase = 0; SelectedNode = None }
```

**Why separate?**

- D3 force simulations are CPU-intensive
- Large PSGs can have thousands of nodes
- Layout computation shouldn't block editing
- Can be updated in background after compilation

### Terminal WebView

xterm.js in its own WebView:

```fsharp
type TerminalWebView = {
    Handle: WebViewHandle
    PTY: nativeint  // Master PTY fd
    Pid: int        // Shell process
}

let createTerminalWebView () =
    let webview = WebView.create {
        Title = "WRENEdit - Terminal"
        Width = 800
        Height = 400
        DevTools = false
        Transparent = false
    }

    // Create PTY
    let (masterFd, childPid) = PTY.fork "/bin/bash" 24 80

    webview.LoadUrl "wrendit://app/terminal.html"
    webview.AddMessageHandler "terminal" (handleTerminalMessage masterFd)

    // Start reading from PTY
    startPTYReader masterFd webview

    { Handle = webview; PTY = masterFd; Pid = childPid }
```

**Why separate?**

- Terminal I/O is continuous and high-volume
- WebGL rendering (xterm.js) benefits from dedicated resources
- Shell processes may produce output at any rate
- Terminal crash shouldn't affect editor

## WebView Coordination

### Central Coordinator

The F# Native backend coordinates all WebViews:

```fsharp
type WRENEditApp = {
    Editor: EditorWebView
    Debug: DebugWebView option
    PSG: PSGWebView option
    Terminals: TerminalWebView list
    LSPClient: LSPClient
    Compiler: CompilerInstance
}

module Coordinator =
    let mutable app: WRENEditApp option = None

    let start () =
        let editor = createEditorWebView ()

        app <- Some {
            Editor = editor
            Debug = None
            PSG = None
            Terminals = []
            LSPClient = LSPClient.connect "fsnac"
            Compiler = Compiler.create ()
        }

    let openDebugger () =
        match app with
        | Some a when a.Debug.IsNone ->
            let debug = createDebugWebView ()
            app <- Some { a with Debug = Some debug }
        | _ -> ()

    let openPSGViewer () =
        match app with
        | Some a when a.PSG.IsNone ->
            let psg = createPSGWebView ()
            app <- Some { a with PSG = Some psg }
        | _ -> ()

    let openTerminal () =
        match app with
        | Some a ->
            let terminal = createTerminalWebView ()
            app <- Some { a with Terminals = terminal :: a.Terminals }
        | None -> ()
```

### Cross-WebView Communication

WebViews communicate through the native coordinator:

```
┌─────────────────┐         ┌─────────────────┐
│  Editor WebView │         │  Debug WebView  │
└────────┬────────┘         └────────┬────────┘
         │                           │
         │ "set_breakpoint"          │
         ▼                           │
┌─────────────────────────────────────────────┐
│              F# Native Coordinator           │
│                                             │
│  handleMessage msg =                        │
│      match msg.Type with                    │
│      | "set_breakpoint" ->                  │
│          Breakpoints.add msg.Location       │
│          // Notify debug WebView            │
│          Debug.send "breakpoint_added" loc  │──────►
│      | ...                                  │
└─────────────────────────────────────────────┘
```

```fsharp
// Message routing
let handleEditorMessage (msg: Message) =
    match msg.Type with
    | "set_breakpoint" ->
        let loc = msg.Payload |> deserialize<SourceLocation>
        Breakpoints.add loc
        // Forward to debug WebView if open
        match app.Debug with
        | Some debug -> debug.Send "breakpoint_added" loc
        | None -> ()

    | "open_psg_viewer" ->
        openPSGViewer ()
        // Send current PSG
        match app.Compiler.CurrentPSG with
        | Some psg ->
            app.PSG.Value.Send "load_psg" psg
        | None -> ()

    | "compile" ->
        let file = msg.Payload |> deserialize<string>
        async {
            let! psg = app.Compiler.Compile file
            // Update PSG viewer if open
            match app.PSG with
            | Some viewer -> viewer.Send "load_psg" psg
            | None -> ()
            // Send diagnostics to editor
            app.Editor.Send "diagnostics" psg.Diagnostics
        } |> Async.Start

    | _ -> ()
```

### Event Broadcasting

Some events should reach all WebViews:

```fsharp
let broadcast eventType payload =
    app.Editor.Send eventType payload
    app.Debug |> Option.iter (fun d -> d.Send eventType payload)
    app.PSG |> Option.iter (fun p -> p.Send eventType payload)
    app.Terminals |> List.iter (fun t -> t.Send eventType payload)

// Example: Theme change affects all WebViews
let changeTheme theme =
    broadcast "theme_changed" theme
```

## WebView Lifecycle

### Lazy Creation

WebViews are created on demand:

```fsharp
// User clicks "Debug" button
handleEditorMessage { Type = "start_debug"; Payload = file }
    |> function
    | "start_debug" ->
        // Create debug WebView if not exists
        if app.Debug.IsNone then
            openDebugger ()

        // Start debug session
        let session = Debug.createSession file
        app.Debug.Value.Send "session_started" session
```

### Graceful Shutdown

```fsharp
let shutdown () =
    // Close all terminals (kill shells)
    app.Terminals |> List.iter (fun t ->
        PTY.kill t.Pid
        t.Handle.Close()
    )

    // Close optional WebViews
    app.Debug |> Option.iter (fun d -> d.Handle.Close())
    app.PSG |> Option.iter (fun p -> p.Handle.Close())

    // Close editor last
    app.Editor.Handle.Close()
```

### Crash Recovery

```fsharp
let monitorWebView (webview: WebViewHandle) onCrash =
    webview.OnProcessTerminated (fun exitCode ->
        if exitCode <> 0 then
            onCrash webview exitCode
    )

// Example: Recover from debug WebView crash
monitorWebView debugWebView.Handle (fun _ _ ->
    // Log the crash
    Logger.warn "Debug WebView crashed, recreating..."

    // Recreate the WebView
    let newDebug = createDebugWebView ()
    app <- Some { app.Value with Debug = Some newDebug }

    // Restore state if possible
    newDebug.Send "restore_breakpoints" app.Value.Editor.Breakpoints
)
```

## Performance Considerations

### IPC Overhead

Multiple WebViews mean more IPC. Minimize overhead:

```fsharp
// BAD: Sending every keystroke
editor.OnKeyPress (fun key ->
    coordinator.Send "key_pressed" key  // Too chatty!
)

// GOOD: Batch updates
let mutable pendingChanges = []
let flushInterval = 16  // ~60fps

editor.OnChange (fun change ->
    pendingChanges <- change :: pendingChanges
)

setInterval flushInterval (fun () ->
    if not (List.isEmpty pendingChanges) then
        coordinator.Send "changes" (List.rev pendingChanges)
        pendingChanges <- []
)
```

### Selective Updates

Don't send data to WebViews that don't need it:

```fsharp
let compilationComplete psg diagnostics =
    // Always: Editor needs diagnostics
    app.Editor.Send "diagnostics" diagnostics

    // Only if open: PSG viewer
    match app.PSG with
    | Some viewer when viewer.IsVisible ->
        viewer.Send "load_psg" psg
    | _ -> ()

    // Only if debugging: Debug WebView
    match app.Debug with
    | Some debug when debug.ActiveSession.IsSome ->
        debug.Send "symbols_updated" psg.Symbols
    | _ -> ()
```

## Window Management

### Floating WebViews

WebViews can be floating windows:

```fsharp
let createFloatingWebView title content =
    let webview = WebView.create {
        Title = title
        Width = 400
        Height = 300
        DevTools = false
        Transparent = false
        Decorations = true  // Window chrome
        AlwaysOnTop = false
    }

    webview.LoadHtml content
    webview
```

### Multi-Monitor Support

Dockview's popout feature combined with separate WebViews enables true multi-monitor:

```
Monitor 1                      Monitor 2
┌─────────────────────┐       ┌─────────────────────┐
│  Editor WebView     │       │  PSG WebView        │
│  ┌───────┬────────┐ │       │                     │
│  │ Code  │Terminal│ │       │    [PSG Graph]      │
│  │       │        │ │       │                     │
│  └───────┴────────┘ │       │                     │
└─────────────────────┘       └─────────────────────┘

Monitor 3
┌─────────────────────┐
│  Debug WebView      │
│  ┌────────────────┐ │
│  │ Continuations  │ │
│  │ Variables      │ │
│  │ Call Stack     │ │
│  └────────────────┘ │
└─────────────────────┘
```

## Summary

Multi-WebView architecture provides:

| Benefit | Mechanism |
|---------|-----------|
| UI responsiveness | Heavy operations in separate processes |
| Stability | Crash isolation between components |
| Parallelism | Multiple JavaScript engines |
| Memory efficiency | Separate heaps, independent GC |
| Flexibility | On-demand WebView creation |
| Multi-monitor | True separate windows |

The F# Native coordinator manages lifecycle, routing, and state synchronization, enabling the benefits of multiple processes without the complexity leaking into the frontend code.

## Next Steps

- [05_webgpu.md](./05_webgpu.md) - WebGPU compute integration
