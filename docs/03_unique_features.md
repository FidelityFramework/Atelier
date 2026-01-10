# 03 - Unique Features: Beyond VSCode and NeoVim

## The Opportunity

VSCode and NeoVim are general-purpose editors. They excel at breadth but cannot provide depth for specialized domains. WRENEdit, built specifically for the Fidelity ecosystem, can offer capabilities that general editors fundamentally cannot.

## Feature 1: Delimited Continuation Debugging

### The Problem with Traditional Debuggers

Traditional debuggers operate on a stack-based mental model:

```
Thread 1:
├─ main()
│  ├─ processRequest()
│  │  ├─ parseInput()
│  │  │  └─ [BREAKPOINT] ← Here
│  │  └─ handleParsed()
│  └─ sendResponse()
```

This model breaks down with delimited continuations. A continuation captures "the rest of the computation" and can be:
- Stored for later
- Invoked multiple times
- Passed across threads
- Serialized and transmitted

```fsharp
// Fidelity delimited continuation
let handler = effect {
    let! input = shift (fun k ->
        // k is the continuation "the rest of processRequest"
        // We might invoke it 0, 1, or many times
        // We might store it and invoke later
        storeForLater k
        defaultResponse
    )
    // ... rest of computation
}
```

### WRENEdit's Continuation Inspector

WRENEdit provides specialized debugging for delimited continuations:

```
┌─────────────────────────────────────────────────────────────┐
│  Continuation Inspector                              [x]    │
├─────────────────────────────────────────────────────────────┤
│  Active Continuations (3)                                   │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ [k1] processRequest continuation                       ││
│  │      Created: handleEffect @ line 42                   ││
│  │      Status: SUSPENDED                                 ││
│  │      Captures: {input, config, tempBuffer}            ││
│  │      Invocation count: 0                               ││
│  │      [Invoke] [Step Into] [Visualize]                 ││
│  ├─────────────────────────────────────────────────────────┤│
│  │ [k2] retry continuation                                ││
│  │      Created: withRetry @ line 78                      ││
│  │      Status: INVOKED (2x)                              ││
│  │      Captures: {attempt, maxRetries}                   ││
│  │      [Invoke] [Step Into] [Visualize]                 ││
│  ├─────────────────────────────────────────────────────────┤│
│  │ [k3] transaction continuation                          ││
│  │      Created: atomically @ line 156                    ││
│  │      Status: STORED (pending commit)                   ││
│  │      Captures: {connection, statements}                ││
│  │      [Invoke] [Step Into] [Visualize]                 ││
│  └─────────────────────────────────────────────────────────┘│
│                                                             │
│  Continuation Graph                                         │
│  ┌─────────────────────────────────────────────────────────┐│
│  │         [main]                                         ││
│  │            │                                           ││
│  │            ▼                                           ││
│  │      [handleEffect]──creates──>[k1]                   ││
│  │            │                     │                     ││
│  │            ▼                     │ (suspended)         ││
│  │      [withRetry]───creates───>[k2]                    ││
│  │            │                  /   \                    ││
│  │            │          invoke(1)  invoke(2)            ││
│  │            ▼                                           ││
│  │      [atomically]──creates──>[k3]                     ││
│  │                               │                        ││
│  │                          (stored)                      ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### Implementation Architecture

```fsharp
// Native debug engine integration
module ContinuationDebugger

type ContinuationInfo = {
    Id: ContinuationId
    CreatedAt: SourceLocation
    Status: ContinuationStatus
    CapturedBindings: Map<string, Value>
    InvocationHistory: InvocationRecord list
    ParentContinuation: ContinuationId option
}

type ContinuationStatus =
    | Suspended
    | Invoked of count: int
    | Stored of location: string
    | Completed

// Debug events sent to frontend
type DebugEvent =
    | ContinuationCreated of ContinuationInfo
    | ContinuationInvoked of ContinuationId * InvocationRecord
    | ContinuationStored of ContinuationId * string
    | ContinuationCompleted of ContinuationId

// Frontend subscribes to events
WREN.subscribe "debug_event" (function
    | ContinuationCreated info -> updateContinuationList (Add info)
    | ContinuationInvoked (id, record) -> updateInvocationGraph id record
    | _ -> ()
)
```

### Why This Matters

No existing debugger provides this capability because:

1. **Traditional runtimes don't have delimited continuations** - .NET, JVM, V8 don't expose continuation primitives
2. **Stack-based mental models are embedded** - Debugger UIs assume call stacks
3. **General editors can't integrate deeply** - VSCode's DAP assumes traditional debugging

WRENEdit, built for Fidelity:
- Understands continuation semantics natively
- Visualizes non-linear control flow
- Tracks continuation lifecycle (creation → suspension → invocation → completion)
- Shows captured closures explicitly

## Feature 2: PSG Visualization with D3

### The Program Semantic Graph

Firefly's PSG (Program Semantic Graph) is the compiler's intermediate representation:

```
F# Source → FCS → PSG → Nanopasses → Alex → MLIR → Native
```

The PSG captures:
- Syntax structure (from SynExpr)
- Type information (from FSharpExpr)
- Def-use relationships
- Reachability information
- SRTP resolutions

### Interactive PSG Viewer

```
┌─────────────────────────────────────────────────────────────┐
│  PSG Viewer: HelloWorld.fs                          [x]     │
├─────────────────────────────────────────────────────────────┤
│  Filters: [x] Values [ ] Types [x] Calls [ ] Unreachable   │
│  Phase: [▼ Phase 4: Typed Tree Overlay]                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                     ┌─────────┐                             │
│                     │ Module  │                             │
│                     │ Program │                             │
│                     └────┬────┘                             │
│                          │                                  │
│            ┌─────────────┼─────────────┐                   │
│            │             │             │                    │
│            ▼             ▼             ▼                    │
│       ┌────────┐   ┌────────┐   ┌────────┐                │
│       │ let    │   │ let    │   │ entry  │                │
│       │ name   │   │ greet  │   │ main   │                │
│       │:string │   │:string │   │:unit   │                │
│       └────┬───┘   └───┬────┘   └───┬────┘                │
│            │           │            │                       │
│            │      ┌────┴────┐       │                       │
│            │      │ App     │       │                       │
│            │      │ sprintf │◄──────┤ uses                 │
│            │      └────┬────┘       │                       │
│            │           │            │                       │
│            └───────────┼────────────┘                       │
│                        │                                    │
│                        ▼                                    │
│                   ┌─────────┐                              │
│                   │ App     │                              │
│                   │ writeln │                              │
│                   │ :unit   │                              │
│                   └─────────┘                              │
│                                                             │
│  Selected: [let greet]                                     │
│  Type: string                                              │
│  Range: HelloWorld.fs:3:5-3:42                             │
│  Symbol: greet (FSharpMemberOrFunctionOrValue)             │
│  [Jump to Source] [Show MLIR] [Show Typed Tree]            │
└─────────────────────────────────────────────────────────────┘
```

### D3-Based Rendering

[D3.js](https://d3js.org/) provides the visualization engine:

```typescript
// PSG Graph rendering with D3
import * as d3 from 'd3'

interface PSGNode {
  id: string
  kind: string
  label: string
  type?: string
  range: SourceRange
  isReachable: boolean
}

interface PSGEdge {
  source: string
  target: string
  kind: 'ChildOf' | 'DefUse' | 'Call' | 'Type'
}

function renderPSG(container: HTMLElement, nodes: PSGNode[], edges: PSGEdge[]) {
  const svg = d3.select(container).append('svg')

  // Force-directed layout
  const simulation = d3.forceSimulation(nodes)
    .force('link', d3.forceLink(edges).id(d => d.id))
    .force('charge', d3.forceManyBody().strength(-300))
    .force('center', d3.forceCenter(width / 2, height / 2))

  // Render edges
  const link = svg.selectAll('.edge')
    .data(edges)
    .join('line')
    .attr('class', d => `edge edge-${d.kind}`)

  // Render nodes
  const node = svg.selectAll('.node')
    .data(nodes)
    .join('g')
    .attr('class', d => `node node-${d.kind} ${d.isReachable ? '' : 'unreachable'}`)
    .call(d3.drag())

  node.append('rect')
    .attr('width', 80)
    .attr('height', 40)

  node.append('text')
    .text(d => d.label)

  // Click to select
  node.on('click', (event, d) => {
    WREN.send('psg_node_selected', d.id)
  })
}
```

### Bidirectional Navigation

The PSG Viewer enables bidirectional navigation:

```
Source Code                     PSG Graph
────────────                    ─────────
let greet name =          ◄──►  [let greet] node
    sprintf "Hello, %s!" name       │
                                    ▼
                              [App sprintf] node
```

Clicking a node highlights the corresponding source. Clicking source highlights the corresponding node.

```fsharp
// Source → PSG navigation
WREN.onMessage "source_cursor_moved" (fun pos ->
    let node = findNodeAtPosition psg pos
    highlightNode node.Id
)

// PSG → Source navigation
WREN.onMessage "psg_node_selected" (fun nodeId ->
    let node = psg.GetNode nodeId
    jumpToSource node.Range
)
```

### Phase Comparison

The PSG evolves through nanopass phases. The viewer allows comparing phases:

```
Phase 3 (Reachability)          Phase 4 (Typed Tree)
──────────────────────          ────────────────────
[let x] (reachable)             [let x] (reachable)
    │                               │
    ▼                               ▼
[App f] (reachable)             [App f] (reachable)
    │                               │
    ▼                               ▼  + Type: int -> string
[App g] (unreachable)           [App g] (unreachable)
                                    │
                                    ▼  + SRTP: Resolved to String.Length
```

Switching phases shows how the PSG gains information:
- Phase 1-2: Structure only
- Phase 3: Reachability marks
- Phase 4: Type information, SRTP resolution
- Phase 5+: Def-use edges, operation classification

## Feature 3: Compilation Pipeline Inspector

### Seeing the Full Picture

WRENEdit can display the entire compilation pipeline for a selection:

```
┌─────────────────────────────────────────────────────────────┐
│  Pipeline Inspector: `greet name`                    [x]    │
├─────────────────────────────────────────────────────────────┤
│  F# Source                                                  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ let greet name = sprintf "Hello, %s!" name            ││
│  └─────────────────────────────────────────────────────────┘│
│                          │                                  │
│                          ▼                                  │
│  PSG Node                                                   │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Let { Name = "greet"; Body = App(...) }               ││
│  │ Type: string -> string                                 ││
│  └─────────────────────────────────────────────────────────┘│
│                          │                                  │
│                          ▼                                  │
│  MLIR                                                       │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ func.func @greet(%arg0: !fir.string) -> !fir.string { ││
│  │   %0 = fir.constant "Hello, %s!" : !fir.string        ││
│  │   %1 = fir.call @sprintf(%0, %arg0) : ...             ││
│  │   return %1 : !fir.string                              ││
│  │ } loc("HelloWorld.fs":3:5)                             ││
│  └─────────────────────────────────────────────────────────┘│
│                          │                                  │
│                          ▼                                  │
│  LLVM IR                                                    │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ define %string @greet(%string %name) {                 ││
│  │   %fmt = getelementptr ...                             ││
│  │   %result = call @sprintf(%fmt, %name)                 ││
│  │   ret %string %result                                  ││
│  │ }                                                       ││
│  └─────────────────────────────────────────────────────────┘│
│                          │                                  │
│                          ▼                                  │
│  Assembly (x86_64)                                          │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ greet:                                                  ││
│  │   push rbp                                             ││
│  │   mov rbp, rsp                                         ││
│  │   lea rdi, [rip + .Lstr.hello]                        ││
│  │   mov rsi, %arg0                                       ││
│  │   call sprintf                                         ││
│  │   pop rbp                                              ││
│  │   ret                                                  ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

This allows developers to:
- Understand how F# maps to native code
- Debug code generation issues
- Learn compiler internals
- Verify optimization decisions

## Feature 4: Effect System Awareness

### Visualizing Effects

Fidelity's effect system tracks computational effects:

```fsharp
// Effect-annotated code
let readFile path : IO<string> = ...
let parseJson text : Parse<Json> = ...
let validate json : Validation<Config> = ...

// Effect composition visible in editor
let loadConfig path =
    effect {
        let! text = readFile path       // IO
        let! json = parseJson text       // Parse
        let! config = validate json      // Validation
        return config
    }
// Inferred effect: IO + Parse + Validation
```

WRENEdit displays effect annotations inline:

```
┌──────────────────────────────────────────────────────────────┐
│  let loadConfig path =                    [IO+Parse+Valid]  │
│      effect {                                               │
│          let! text = readFile path        [IO]             │
│          let! json = parseJson text       [Parse]          │
│          let! config = validate json      [Validation]     │
│          return config                                      │
│      }                                                      │
└──────────────────────────────────────────────────────────────┘
```

### Effect Mismatch Diagnostics

When effects don't compose correctly:

```
┌──────────────────────────────────────────────────────────────┐
│  ⚠ Effect mismatch at line 15                               │
│                                                              │
│  Expected: Pure                                              │
│  Got: IO + Parse                                            │
│                                                              │
│  The function `compute` is marked [<Pure>] but calls:       │
│    - readFile (IO) at line 17                               │
│    - parseJson (Parse) at line 18                           │
│                                                              │
│  [Show Effect Graph] [Suggested Fix]                        │
└──────────────────────────────────────────────────────────────┘
```

## Summary

WRENEdit's unique features arise from deep integration with the Fidelity ecosystem:

| Feature | Enabled By |
|---------|-----------|
| Continuation debugging | Fidelity's delimited continuation runtime |
| PSG visualization | Direct access to Firefly's IR |
| Pipeline inspection | Integration with all compilation phases |
| Effect awareness | FNCS effect system |

These capabilities are impossible in general-purpose editors because they require knowledge of specific compiler internals that only purpose-built tooling can provide.

## Next Steps

- [04_multi_webview.md](./04_multi_webview.md) - Multi-WebView architecture
- [05_webgpu.md](./05_webgpu.md) - WebGPU compute integration
