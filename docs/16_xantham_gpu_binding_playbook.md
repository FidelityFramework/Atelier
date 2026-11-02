# 16 - Building the D3 and WebGPU Bindings

This playbook covers the **browser deployment track**: TypeScript declarations to a Fickl-compiled F# consumer in a browser or conventional WrenHello WebView. It is intended for reusable Fidelity libraries and product applications. Atelier's graph inspector is a demanding first consumer. Financial charts, scientific visualizations and other WREN applications can use the resulting browser bindings without depending on the editor.

For this browser track, use Xantham for the F# binding surface, Fickl for F#-to-JavaScript compilation and Ranvier for reactive frontend state. Package a prospective Fidelity.WebGPU browser binding and keep generated declarations separate from authored layout, lifecycle and integrity helpers. BAREWire carries shared application data across admitted boundaries. Clef-to-JavaScript through JSIR is a later browser realization with its own gates.

Ranvier Native is a separate track: application code reaches WebCore's DOM through native interop without Fickl or a JavaScript application runtime. The TypeScript bindings and D3 runtime described here do not automatically provide native WebCore, D3 or GPU functionality. Use the native ABI and renderer integration appropriate to that backend, and compare the resulting behavior against the same visual and integrity contracts.

This is an implementation guide, not a report that WebGPU generation, browser execution or WrenHello integration already passes. Confirm the checked-out tool versions and the current owning documentation before running a generation or release gate.

## What exists and what must be established

At the inspected Xantham revision `d8486b4de3b328e637be255ac7316f666f3bf0b0`, Xantham is rebuilding its TypeScript-to-F# generator around TypeScript 7's `tsc --api` server. Its generated bindings target Fable 5.x. The checked-in root `package.json` pins TypeScript `7.1.0-beta`, and `.config/dotnet-tools.json` pins Fickl `0.7.2`. These are repository-specific observations, not permanent version requirements. The [Xantham working agreements](../../Xantham/AGENTS.md) and [consumer binding guide](../../Xantham/site/content/xantham-cli/guide/bindings.md) describe the current build and compatibility rules.

D3 was Xantham's original binding target. Establish which D3 declarations and generated packages are currently maintained, what the latest generator emits, and which selected APIs compile and run through Fickl. Historical interest is a reason to start there; it is not a substitute for a pinned result.

[GPUWeb's TypeScript definitions](https://github.com/gpuweb/types) are a candidate declaration source for WebGPU. Current GPUWeb guidance says the completeness of built-in DOM declarations varies by TypeScript version, including some TypeScript 7 releases. Its proposed `@types/web` replacement and `@webgpu/types` supplement need to be assessed against the actual pinned Xantham compiler and Core.TS catalog. Do not load overlapping ambient DOM providers without checking duplicate declarations and the identity of the generated F# types. Record the package version, source hash and the exact declaration files selected.

The observed Xantham pin `7.1.0-beta` is not covered by a blanket conclusion about TypeScript 7.0.2 or earlier. Inspect its actual installed DOM declarations and catalog inputs. Xantham's `tests/fixtures/globals-lab/package.json` is an ambient-global-only fixture worth studying as a generation precedent. It does not establish that the complete WebGPU augmentation, its overloads or a particular Core.TS profile already work. The pinned package fixture list inspected for this guide did not name D3 or WebGPU; that does not prove there are no other relevant tests or packages.

The browser provides the WebGPU implementation. Its `navigator.gpu` entry is a browser global, not an npm module whose JavaScript implementation is provided by a type-only declaration package. Likewise, a D3 binding still needs the matching actual D3 runtime packages and import paths. A generated `.fs` file provides access to an API; it does not install the browser feature or implement GPU operations.

## Select a narrow first surface

Bind a complete vertical slice instead of attempting every WebGPU API at once:

| Slice | Declarations and behavior to establish |
|---|---|
| Browser entry | `navigator.gpu` availability in a secure context; canvas lookup and `getContext("webgpu")`; adapter acquisition and nullable results. |
| Device | `requestAdapter`, `requestDevice`, promises, required features and limits, failures and device-loss notification. |
| Data | GPU buffer descriptors, sizes and usage flags; typed-array/ArrayBuffer input; the selected queue upload call and bounds. |
| Drawing | WGSL source as shader module input, minimal render pipeline, canvas configuration, command encoding, queue submission and a triangle or bounded chart. |
| Lifetime | asynchronous operations, ownership of buffers and canvas, cleanup, `destroy`, unmap and device-loss behavior. |
| D3 | the exact selected scales, 2D force/layout and curve functions plus callbacks, data shapes, runtime imports and module subpaths. |

An F# wrapper can make those operations pleasant, but it must match the emitted binding names and shapes rather than assuming names in advance. In particular, descriptors include required and optional fields, unions, literal strings, nested records and callbacks. WebGPU also uses bitmask flags and JavaScript numeric sizes. Preserve flag composition and check integer/range conversions at the boundary. `GPUSize64` does not turn an arbitrary F# `int64` into a safe JavaScript `Number`; the wrapper needs a checked conversion consistent with the selected range and device limit.

The API binding controls host calls and descriptors. It does not compile F# or Clef into WGSL. Begin with a small, versioned WGSL shader and explicit input layout. A future shader compiler path needs separate source-to-WGSL and numeric-preservation evidence.

## Reproduce generation

Use the primary Xantham checkout. Put any new test lab and scratch artifacts in its documented `tests/fixtures/<feature>-lab/` and `tests/.scratch/` locations. Follow the current checkout's [generator workflow](../../Xantham/site/content/dev/generator.md), [usage](../../Xantham/site/content/xantham-cli/guide/usage.md), [configuration](../../Xantham/site/content/xantham-cli/guide/configuration.md) and [dependency/catalog guidance](../../Xantham/site/content/xantham-cli/guide/dependencies.md). The owner's working agreement supersedes historical instructions that mention secondary worktrees or copied repositories.

1. Pin the generator revision, TypeScript release and source, package/declaration version, package-manager lockfile, Fickl version, support-package versions and target host. Restore dependencies according to the selected checkout and verify the matching compiler is available.
2. Choose the ownership of the ECMAScript/DOM declarations. `Xantham.Fable.Core.TS` supplies a standard-library binding and a declaration catalog. Reuse the exact matching package and compiler/profile where compatible; establish which generated package owns `Navigator`, `HTMLCanvasElement`, `ArrayBuffer` and the WebGPU interfaces. Do not generate a second independently owned type merely because the source name is similar.
3. Make an npm fixture whose input directory has `package.json`. For D3, install the actual JavaScript runtime plus the declaration provider where separate. Select the specific package, entry or subpath and its real runtime import. For WebGPU globals, test the selected ambient declaration provider and output; do not invent an import path or claim an ambient-augmentation shape is supported until generation and the consumer both demonstrate it.
4. Store `xantham.json` outside `node_modules`. `--config` and `--out` are relative to the command's working directory; declaration `entry` paths inside the config are relative to the input package directory. Generate and retain the outputs and reports.

The documented local CLI entry point is:

```sh
dotnet run --project src/Xantham.Cli -- generate <package-dir> --config <path-to-xantham.json> -o <output-dir>
```

The installed CLI exposes `xantham generate <package-dir> --config <path> -o <output-dir>`. Select a config appropriate to the actual package, and confirm the supported options with its `generate --help`. Do not treat the following generic configuration as a prevalidated WebGPU recipe:

```json
{
  "module": "Fidelity.WebGPU.Bindings",
  "lib": ["esnext", "dom"],
  "types": []
}
```

`types: []` disables automatic ambient-provider discovery. It does not prevent explicit imports or reference directives from resolving. A package that needs an ambient provider may need that provider named or referenced. If using a replacement DOM provider, its library profile and Core.TS ownership must be reconciled before choosing `lib` and `types`.

Xantham writes F# files, `manifest.json` and `symbols.jsonl`; a configured declaration catalog can also be emitted. `manifest.json` reports aggregate findings, while `symbols.jsonl` associates findings with individual symbols. Review all used APIs at `Exact`, `Ergonomic`, `Widened` and `Escape` grades. A generated `obj`, erased union, missing overload or permissive callback is a design decision to inspect, not a silently accepted proof of parity. Keep the emitted signatures, diagnostics and findings beside the reproducibility record.

## Preserve shared type identities

An initial GPU slice will likely touch DOM types, typed arrays, promises and callbacks shared with other Xantham output. Choose a single producer for each shared type. Xantham can `ship`, `reference`, `map` or `widen` dependency groups, and supports declaration catalogs for reuse across generation runs. A reference still needs its producer binding or compiled package in the consumer. Compile shipped groups and producer files before dependent bindings.

The packaged Core.TS catalog is at `xantham/declarations.json.br` within the restored package. The [dependency guide](../../Xantham/site/content/xantham-cli/guide/dependencies.md) explains `declarationReferences`, owner ordering and authenticated compatibility. Its documented Core.TS reuse example requires `anyReturnAsUnit: "^on"` to match that catalog's inference profile. Install the exact TypeScript release/source and support package required by the catalog. A catalog mismatch is a diagnostic to resolve, not a reason to cast types until the consumer compiles.

Pay particular attention to global augmentation, union alternatives, overloaded `getContext`, nullable adapter returns, promises, callable values, string literals, numeric flags, `ArrayBuffer` and typed-array generics. When generation cannot represent one declaration faithfully, reproduce it in a minimal lab fixture, fix the owning generator pass when appropriate, and retain a narrowly scoped authored wrapper with an explicit runtime check where necessary. Keep generated files disposable and regenerated; do not edit them by hand.

## Compile and test the bindings

The target consumer must use the supported Xantham support packages and Fable 5-compatible APIs. The inspected guide specifies `Fable.Core` 5.5.0, `Xantham.Fable.Core`, `Xantham.Fable.Core.TS`, and a `net8.0` or newer F# project. It identifies Fickl support for generated inline/callable interfaces and erased union predicates; the repository's own pinned Fickl is 0.7.2. Verify the actual consumer package versions, transitive `Fickl.Core` dependency and compiler output together.

Use the existing Xantham test workflow when changing the generator. The documented focused loop is:

```sh
dotnet fsi build.fsx -- test --quick --update --filter "<suite>"
```

`--update` rewrites goldens, so review the complete diff. The documented full suite is `dotnet fsi build.fsx -- test`; add `--run-gate` for the JavaScript runtime gate. Xantham's instructions specifically use its build script rather than `dotnet test` for the Expecto suites. Run the Fickl-compiled consumer in addition to the .NET compile gate: .NET type checking, successful JavaScript imports and behavior in a real WebView establish different facts. No gate in this playbook has been run by writing it.

### Fickl options that change the boundary

Record the chosen Fickl flags in each test and delivered build. The inspected [Fickl README](../../fickl/README.md) documents options with materially different JavaScript representations:

- `--clampByteArrays` emits F# byte arrays as `Uint8ClampedArray`, useful for APIs such as canvas pixel data. Out-of-range assignments clamp instead of wrapping. A BAREWire or WebGPU byte stream should use the representation and conversion behavior its own contract requires; do not enable a flag globally merely to simplify one canvas path. Test out-of-range values, including negative values and values above 255, at the actual conversion or assignment boundary.
- `--strictOptions` preserves the distinction between `Some null` and `None` for supported nullable reference options. Value options remain erased. Test absence, JavaScript `null` and any generated adapter representation separately, especially for `navigator.gpu`, `requestAdapter()` and optional descriptor fields. The selected TypeScript-to-F# mapping and runtime API determine which distinctions are meaningful; a compiler flag does not make a nonexistent browser object available.
- `[<Fickl.Core.Curried>]` changes how annotated function parameters or record fields preserve curried calling conventions. D3's callback invocation must be tested with the real JavaScript library and the generated signature, including argument order and number. Do not apply an annotation without matching the API's actual callback contract.

Compare the emitted JavaScript and observed values under the chosen configuration. A successful compile establishes neither the byte representation expected by BAREWire nor the behavior of callbacks called from JavaScript.

### Small fixtures that discriminate real errors

| Fixture | Positive case | Failure or distinct behavior to exercise |
|---|---|---|
| D3 scale and curve | Pinned D3 declarations bind a scale and a selected 2D curve; emitted JS calls the actual runtime. | Wrong subpath, callback shape or unavailable runtime import is reported. |
| D3 state | A small force-layout consumer uses stable input identities and records its algorithm and seed policy. | Mutating a layout input must not silently change the authoritative graph; 2D output does not become a 3D layout. |
| Browser globals | Canvas context and `navigator.gpu` resolve through the selected declaration owner. | Missing API or a null adapter is handled; no fabricated module import is assumed. |
| Async callbacks | Adapter/device promises and loss or rejection paths run in JavaScript. | Rejection, cancellation and device loss do not appear as successful initialization. |
| Descriptors and flags | Required fields and legal flags reach JavaScript with their intended values. | Missing field, invalid flag combination, widened union or incorrect overload must be observable in compile findings or runtime validation. |
| Buffers and layout | Distinct values in two `vec3<f32>` array elements use a 16-byte stride and match the shader. | A 24-byte packed pair must not be accepted as a 32-byte stride layout. |
| Reactive revisions | A coherent R2 snapshot remains current when earlier R1 work completes. | Late R1 and pre-restart device-generation G1 cannot overwrite current state or publish into G2. |
| Resource disposal | One owner retires resources when the admitted producer and GPU uses complete. | Unmapping detaches mapped views; disposal or cancellation alone cannot prove in-flight GPU work has finished. |
| Numeric boundary | Exact minor units remain the source for labels while checked bounded coordinates feed the shader. | An out-of-contract integer conversion, size overflow, mismatched currency or stale dataset must be refused or reported. |

For the vector fixture, the values are explicit: two tightly packed triples occupy 24 bytes; two WGSL `array<vec3<f32>>` elements use two 16-byte slots, 32 bytes total. A particular BAREWire wire layout may coincide with a WGSL layout only when its schema and conversion contract establish that fact. [WGSL host-shareable layout](https://www.w3.org/TR/WGSL/#memory-layout).

## Build the reusable package around the generated core

Keep three layers distinguishable:

1. **Generated browser surface:** declaration-derived WebGPU and selected D3 APIs, regenerated by Xantham with reproducibility reports. No new global browser APIs or scheduler guarantees are invented.
2. **Authored Fidelity.WebGPU helpers:** explicit capability requirements, checked byte counts/offsets and conversions, host-to-WGSL layout, resource generation and disposal, and error handling. Their F# source and tests survive regeneration. Their error and versioning contract is a library design decision.
3. **Domain and visual adapters:** Ranvier graph/scene state, camera and selection, application schemas and accessibility semantics. Atelier owns its compiler-graph projection; a finance application owns money, currency and source identities. General GPU bindings do not embed those domains.

The browser API validates substantial resource and usage requirements. It cannot determine that bytes describe the correct account, graph revision or hyperedge. The wrapper must retain those associations where the application contract requires them. Nor does an F# type wrapper automatically ensure one-shot GPU ownership in the presence of aliases. Design resource states and runtime checks to make the supported behavior clear, and reserve stronger Clef resource claims for their own implementation and proof.

## Run the real WebView experiment

Use a minimal visual with one changing parameter. On initialization, verify the actual page is a secure context; inspect `navigator.gpu`, request an adapter, inspect its features and limits, then request the smallest device capability the visual needs. Configure a canvas, create a pipeline, draw, observe errors and dispose. Handle a missing API, null adapter, failed device request, shader/pipeline error and lost device explicitly. Do not convert an unavailable feature into a success state or assume hardware acceleration merely because an API exists. [WebGPU specification](https://www.w3.org/TR/webgpu/).

Run the same frontend in a named browser, then inside WrenHello's Linux WebKitGTK host. WrenHello currently uses a bounded compact ASCII script-message bridge and documents BAREWire-over-WebSocket as its intended later integration. Its inbound message limit is 4,096 bytes. The established BAREWire support for .NET, JavaScript and native Clef is important, while that particular transport and graph projection must be validated before treating it as a large-data stream.

After one local render works, send a small typed update across the available host bridge, apply it through one Ranvier batch, and change the visual without a host round trip for every animation frame. Record the exact WebView/runtime version, OS, GPU/driver, origin, adapter/device result, required limits/features, shader bytes, binding and frontend hashes, and observed errors. Test close/remount, device loss and R1/R2 or G1/G2 late completions.

Apple's WebKit documents WebGPU in Safari 26 for iOS and iPadOS, with [WKWebView guidance](https://developer.apple.com/forums/thread/770862) identifying default availability on iOS 26. Chromium has documented Android and WebView as targets for its [WebGPU compatibility mode](https://groups.google.com/a/chromium.org/g/blink-dev/c/N3RlLGCOTJ4/m/loneRTlPBwAJ). These are test leads. A browser result does not establish a specific Android WebView or WKWebView configuration, and the WrenHello Linux host remains a separate test. Mobile app hosts, packaging, lifecycle, platform bindings and native backplanes each need their own admission evidence.

## Release and later Clef realization

The binding handoff should contain: generator and compiler revisions; declaration source, package versions and lockfile; configuration and generated outputs; catalog owners and hashes; findings for every used symbol; F# compile and Fickl runtime results; the actual browser/WebView, device and feature matrix; positive and negative case results; known unsupported shapes; and reproduction commands. Bind claims to the exact output and host exercised. New generator behavior belongs in Xantham's existing passes and fixture workflow; authored helper contracts belong with the prospective Fidelity.WebGPU package.

Fidelity.UI defines UI-facing identity, behavior and lifetime expectations. Atelier defines its graph scene and evidence interaction. The same Fidelity.WebGPU binding should be reusable for a browser-only visualization, WREN desktop/mobile application or a frontend using a remote service. A native GPU backend and zero-copy native/WebView texture sharing are independent host-specific capabilities with their own binding, synchronization, lifetime and presentation contracts.

When Clef's JSIR/JavaScript and JSX paths support the required source and libraries, compile the same admitted frontend behavior through that route and compare it with the Fickl baseline. A later proof-bearing target must connect source values, BAREWire encoding, checked conversion, GPU layout, WGSL operations and the delivered artifact under declared numeric, platform and resource assumptions. Xantham's current TypeScript-to-Fable binding generation does not by itself establish that Clef compilation or proof.

## Related design

- [Graphics and visualization strategy](05_webgpu.md)
- [Hypergraph visualization](12_hypergraph_visualization.md)
- [Visualization integrity](13_visualization_integrity.md)
- [WREN WebGPU integration](14_wren_webgpu_bindings.md)
- [Delivery and benchmark roadmap](15_visualization_roadmap.md)
