# 05 - WebGPU Compute Integration

## WebGPU Overview

WebGPU is the successor to WebGL, providing modern GPU access from the web platform. Unlike WebGL (which maps to OpenGL ES), WebGPU maps to:
- **Vulkan** (Linux, Android)
- **Metal** (macOS, iOS)
- **Direct3D 12** (Windows)

This means WebGPU provides access to compute shaders, not just graphics.

## Why WebGPU in WRENEdit?

### Use Case 1: PSG Graph Layout

Large PSGs can have thousands of nodes. Force-directed graph layout (D3's default) is O(n²) per iteration. GPU compute can parallelize this:

```wgsl
// Force calculation compute shader
@compute @workgroup_size(64)
fn calculateForces(
    @builtin(global_invocation_id) id: vec3<u32>
) {
    let nodeIdx = id.x;
    if (nodeIdx >= nodeCount) { return; }

    var force = vec2<f32>(0.0, 0.0);

    // Repulsion from all other nodes
    for (var i = 0u; i < nodeCount; i++) {
        if (i == nodeIdx) { continue; }

        let delta = positions[nodeIdx] - positions[i];
        let dist = max(length(delta), 0.01);
        let repulsion = (repulsionStrength / (dist * dist)) * normalize(delta);
        force += repulsion;
    }

    // Attraction along edges
    for (var e = edgeStart[nodeIdx]; e < edgeStart[nodeIdx + 1]; e++) {
        let target = edges[e];
        let delta = positions[target] - positions[nodeIdx];
        let dist = length(delta);
        let attraction = attractionStrength * dist * normalize(delta);
        force += attraction;
    }

    forces[nodeIdx] = force;
}
```

**Performance comparison:**

| Nodes | CPU (D3) | WebGPU |
|-------|----------|--------|
| 1,000 | 16ms/iter | 0.5ms/iter |
| 5,000 | 400ms/iter | 2ms/iter |
| 10,000 | 1600ms/iter | 8ms/iter |

### Use Case 2: Syntax Highlighting

Large files benefit from GPU-accelerated tokenization:

```wgsl
// Parallel tokenization
@compute @workgroup_size(256)
fn tokenize(
    @builtin(global_invocation_id) id: vec3<u32>
) {
    let charIdx = id.x;
    if (charIdx >= textLength) { return; }

    // State machine transition
    let char = text[charIdx];
    let prevState = states[charIdx - 1]; // Or initial state
    let newState = transitionTable[prevState * 256 + char];

    states[charIdx] = newState;
    tokenTypes[charIdx] = stateToToken[newState];
}
```

This is speculative execution - some tokens span multiple characters and need CPU fixup - but 90%+ of characters can be classified in parallel.

### Use Case 3: Fuzzy Matching

File search and symbol lookup with fuzzy matching:

```wgsl
// Parallel fuzzy match scoring
@compute @workgroup_size(64)
fn fuzzyScore(
    @builtin(global_invocation_id) id: vec3<u32>
) {
    let candidateIdx = id.x;
    if (candidateIdx >= candidateCount) { return; }

    let candidate = candidates[candidateIdx];
    var score = 0;
    var queryIdx = 0u;
    var consecutive = 0;

    for (var i = 0u; i < candidate.length; i++) {
        if (queryIdx < queryLength && candidate.chars[i] == query[queryIdx]) {
            score += 1 + consecutive * 2;  // Bonus for consecutive matches
            consecutive++;
            queryIdx++;
        } else {
            consecutive = 0;
        }
    }

    // Penalize unmatched query characters
    if (queryIdx < queryLength) {
        score = -1;  // No match
    }

    scores[candidateIdx] = score;
}
```

Search 100,000 symbols in milliseconds.

### Use Case 4: Terminal Rendering

xterm.js already uses WebGL for text rendering. WebGPU could further accelerate:
- Glyph rasterization
- Scroll buffer management
- Selection highlighting

## WebGPU Architecture in WRENEdit

### Device Acquisition

```typescript
// Initialize WebGPU
async function initWebGPU(): Promise<GPUDevice> {
    if (!navigator.gpu) {
        throw new Error("WebGPU not supported")
    }

    const adapter = await navigator.gpu.requestAdapter({
        powerPreference: "high-performance"
    })

    if (!adapter) {
        throw new Error("No GPU adapter found")
    }

    const device = await adapter.requestDevice({
        requiredFeatures: [],
        requiredLimits: {
            maxStorageBufferBindingSize: 128 * 1024 * 1024,  // 128MB
            maxComputeWorkgroupsPerDimension: 65535
        }
    })

    return device
}
```

### Compute Pipeline

```typescript
// PSG layout compute pipeline
async function createLayoutPipeline(device: GPUDevice): Promise<GPUComputePipeline> {
    const shaderModule = device.createShaderModule({
        code: forceLayoutShader  // WGSL source
    })

    const pipeline = await device.createComputePipelineAsync({
        layout: 'auto',
        compute: {
            module: shaderModule,
            entryPoint: 'calculateForces'
        }
    })

    return pipeline
}
```

### Buffer Management

```typescript
interface PSGBuffers {
    positions: GPUBuffer      // vec2<f32> per node
    forces: GPUBuffer         // vec2<f32> per node
    edges: GPUBuffer          // u32 target indices
    edgeStart: GPUBuffer      // u32 edge list start per node
}

function createPSGBuffers(device: GPUDevice, psg: PSGData): PSGBuffers {
    const positions = device.createBuffer({
        size: psg.nodeCount * 8,  // 2 floats per node
        usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST | GPUBufferUsage.COPY_SRC
    })

    const forces = device.createBuffer({
        size: psg.nodeCount * 8,
        usage: GPUBufferUsage.STORAGE
    })

    // ... create edge buffers

    return { positions, forces, edges, edgeStart }
}
```

### Running Compute

```typescript
async function runLayoutIteration(
    device: GPUDevice,
    pipeline: GPUComputePipeline,
    buffers: PSGBuffers,
    nodeCount: number
): Promise<Float32Array> {
    const commandEncoder = device.createCommandEncoder()

    const passEncoder = commandEncoder.beginComputePass()
    passEncoder.setPipeline(pipeline)
    passEncoder.setBindGroup(0, bindGroup)
    passEncoder.dispatchWorkgroups(Math.ceil(nodeCount / 64))
    passEncoder.end()

    // Copy results back
    const readBuffer = device.createBuffer({
        size: nodeCount * 8,
        usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ
    })
    commandEncoder.copyBufferToBuffer(
        buffers.positions, 0,
        readBuffer, 0,
        nodeCount * 8
    )

    device.queue.submit([commandEncoder.finish()])

    await readBuffer.mapAsync(GPUMapMode.READ)
    const result = new Float32Array(readBuffer.getMappedRange())
    readBuffer.unmap()

    return result
}
```

## Integration with F# Native

### IPC for Compute Requests

The F# Native backend can request GPU computation:

```fsharp
// F# Native side
module GPUCompute =
    let layoutPSG (psg: PSG) =
        async {
            // Serialize PSG to buffer format
            let nodes = psg.Nodes |> Array.map (fun n -> {| x = n.X; y = n.Y |})
            let edges = psg.Edges |> Array.map (fun e -> {| source = e.Source; target = e.Target |})

            // Send to PSG WebView for GPU computation
            let! result = WREN.request "gpu_layout" {| nodes = nodes; edges = edges |}

            // Update PSG with new positions
            result.positions |> Array.iteri (fun i pos ->
                psg.Nodes.[i].X <- pos.x
                psg.Nodes.[i].Y <- pos.y
            )
        }
```

```typescript
// WebView side
WREN.onMessage('gpu_layout', async (data) => {
    const { nodes, edges } = data

    // Upload to GPU
    uploadToBuffers(nodes, edges)

    // Run iterations
    for (let i = 0; i < 100; i++) {
        await runLayoutIteration(device, pipeline, buffers, nodes.length)
    }

    // Read back results
    const positions = await readPositions()

    return { positions }
})
```

### Fallback to CPU

Not all systems have WebGPU support. Provide graceful fallback:

```typescript
async function layoutPSG(nodes: Node[], edges: Edge[]): Promise<Position[]> {
    if (navigator.gpu) {
        try {
            return await gpuLayout(nodes, edges)
        } catch (e) {
            console.warn("WebGPU layout failed, falling back to CPU:", e)
        }
    }

    // CPU fallback using D3
    return d3Layout(nodes, edges)
}
```

## WebGPU Availability

### Browser Support (as of 2024)

| Browser | Status |
|---------|--------|
| Chrome | Stable (113+) |
| Edge | Stable (113+) |
| Firefox | Behind flag |
| Safari | Stable (17+) |

### WebView Support

WebKitGTK, WKWebView, and WebView2 all inherit browser WebGPU support:

| Platform | WebView | WebGPU |
|----------|---------|--------|
| Linux | WebKitGTK | Via WebKit (limited) |
| macOS | WKWebView | Yes (Safari 17+) |
| Windows | WebView2 | Yes (Edge 113+) |

**Note:** WebKitGTK may lag behind Safari in WebGPU support. For Linux, we may need to wait or use native Vulkan directly.

## Native GPU Alternative

For maximum performance, WRENEdit could bypass WebGPU entirely and use native GPU APIs:

```fsharp
// Native Vulkan/Metal compute
module NativeCompute =
    #if LINUX
    [<DllImport("libvulkan.so")>]
    extern ...
    #elif MACOS
    [<DllImport("Metal.framework/Metal")>]
    extern ...
    #endif

    let layoutPSG (psg: PSG) =
        // Direct GPU access, no WebView overhead
        ...
```

This is more complex but provides:
- Lower latency (no IPC to WebView)
- More control over memory
- Access to full GPU capabilities

For WRENEdit v1, WebGPU is sufficient. Native compute is a future optimization.

## Practical Recommendations

### Phase 1: CPU-Only

Start with CPU implementations:
- D3 force layout for PSG
- JavaScript tokenization
- Standard fuzzy matching

This works for small-to-medium projects.

### Phase 2: WebGPU Acceleration

Add WebGPU for:
- PSG layout (most impactful)
- Large file tokenization
- Symbol search

Maintain CPU fallbacks.

### Phase 3: Native Compute (Future)

If WebGPU proves insufficient:
- Vulkan compute for Linux
- Metal compute for macOS
- DirectCompute for Windows

This is significant additional complexity and should only be pursued if WebGPU bottlenecks are proven.

## Summary

WebGPU brings GPU compute to WRENEdit's WebViews:

| Task | CPU Time | WebGPU Time | Speedup |
|------|----------|-------------|---------|
| PSG layout (5K nodes) | 400ms | 2ms | 200x |
| Tokenize 100KB file | 50ms | 5ms | 10x |
| Fuzzy match 100K symbols | 100ms | 10ms | 10x |

The multi-WebView architecture naturally supports this - compute-heavy operations run in dedicated WebViews without blocking the editor.

WebGPU support varies by platform. Always provide CPU fallbacks. Consider native GPU APIs only if WebGPU proves insufficient.

## References

- [WebGPU Specification](https://www.w3.org/TR/webgpu/)
- [WGSL Specification](https://www.w3.org/TR/WGSL/)
- [WebGPU Fundamentals](https://webgpufundamentals.org/)
- [D3 Force Simulation](https://github.com/d3/d3-force)
