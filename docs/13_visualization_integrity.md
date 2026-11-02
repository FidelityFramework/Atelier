# 13 - Integrity from Graph to GPU

WREN applications should let a user trust what a picture refers to. That means preserving the input revision, the identities and relationships in the selected data, the evidence attached to it, and the association between a displayed result and the data that produced it. A valid GPU command is one step in that chain. Atelier's compiler graph is a demanding in-house case for a capability that should also support product charts, engineering views and other visualizations.

The Atelier case uses CCS/Baker's published graph facts, an admitted BAREWire graph view, Ranvier state and a renderer appropriate to the deployment. Ranvier Native reaches WebCore through native interop. A browser deployment can use a Fickl-compiled frontend and supported Canvas, WebGL or WebGPU APIs. Other products can supply their own authoritative data and domain contracts. The [visualization strategy](05_webgpu.md) describes the separate target experiments. The same integrity questions apply across them, but a browser API or JavaScript binding does not establish its native counterpart.

## What a trustworthy view must preserve

| Claim | Evidence the viewer needs |
|---|---|
| This is the selected graph fact | Its source revision, node or hyperedge identity, source link where available and the identity rules for this projection. |
| This relationship has these participants | Every included incidence with its participant and role, ordering where meaningful, and an indication if a projection is incomplete. |
| This obligation has this status | The published proposition, premises, checking result and applicable revision. A created obligation and a discharged obligation are distinct states. |
| This image represents the selected update | A coherent model and matching layout, labels, selection, picking data and renderer generation. |
| These bytes have the intended GPU meaning | A specified wire schema, conversion, buffer size, layout, stride, value representation and shader interface. |
| This computation establishes a result | The arithmetic or semantic contract, its premises, the performed operation and evidence connecting it to the realized target. |

Atelier reads and presents the graph. CCS/Baker remains responsible for source semantics, inference, obligations and the settled facts published in the PSG. Layout, grouping, camera position and spline geometry are viewer state. They can aid comprehension without becoming graph facts. A capability or proof status the projection does not supply stays unavailable until its owner supplies the corresponding evidence.

## A joint constraint worth preserving

Suppose buffers A, B and C are all live at the same time and each requires four units from a memory space with capacity ten. The proposition is `4 + 4 + 4 <= 10`. Every pair fits; the three together do not. The full conclusion depends on the declared capacity, each allocation and the evidence that their lifetimes overlap.

The projection should bind one obligation or joint relation to all three buffer identities and their participant roles. A central glyph and three incident curves can make that relationship visible. Clicking the glyph should reveal the proposition, scope, source, all participants, liveness premises and the actual proof status. Clicking a curve should identify the particular incidence and still lead to the same joint relationship.

If the layout displays only A and B, the view must say that C is omitted. It must not present the visible pair as the whole capacity fact. A different schedule might allow memory reuse, but that depends on a new lifetime argument. When a revised PSG supplies that argument, the view can show the revised conclusion and the premises that changed. An attractive spatial arrangement cannot establish non-overlap.

The layout may internally use a star graph, force simulation or GPU geometry to position a joint glyph. Those auxiliary lines are a view of the incidence structure. Their identity must still map back to the one hyperedge and its participant roles, not become new pairwise source facts.

## Exact financial data and approximate pictures

Beancounter provides a concrete integrity reference. Its monetary contract keeps currency and integer minor units together. For USD, `$19.99` is represented as 1,999 cents. Its checked money operation rejects currency mismatch and results outside its admitted signed 64-bit range. Rates and rounding have their own contracts. [Beancounter's JavaScript substrate notes](https://forge.spkez.dev/Braidpoint/beancounter/wiki/22_javascript_substrate) explain the intended controls and their evidence.

For a chart, preserve the authoritative amount and deliberately derive a display coordinate:

1. Receive a versioned amount of 1,999 cents, its currency and record identity through the admitted protocol.
2. Validate the schema, range and revision. Retain the exact amount in the application state.
3. Apply a declared scale to calculate a bounded display coordinate. Check every conversion and any integer intermediate required by that scale.
4. Convert the coordinate to the selected GPU representation, including `f32` where the shader contract calls for it. Draw the bar or point.
5. Show `$19.99` and any accessible value from the exact amount associated with the selected record and revision.

The GPU coordinate is a visual approximation. Several exact amounts may land on the same pixel, and a minimum visible width may be a deliberate presentation choice. Picking that mark must still recover the right record, amount and currency. Do not use the rounded or clamped display coordinate as an authoritative accounting amount.

JavaScript `BigInt` can represent integers exactly, but a boundary converting a signed 64-bit amount to JavaScript `Number` must check the selected contract. Beancounter refuses integer magnitudes beyond `2^53 - 1` on its checked conversion path. A number outside that range may happen to be representable, but adjacent integer precision is no longer guaranteed. The conversion rule must be explicit; a visually plausible bar does not justify silently changing an amount.

This distinction also governs GPU computation. A shader may calculate a display coordinate or an intentionally approximate visual effect. If it calculates an authoritative total, allocation, engineering result or model update, define and validate the required arithmetic and error contract for that use. Currency separation, operation order and intermediate capacity remain application obligations.

WGSL specifies permitted numeric behavior rather than making every operation a bit-for-bit substitute for the host. Its floating-point rules specify no single rounding mode for every intermediate operation, and its built-in operations can have properties different from corresponding CPU operations. Choose a contract compatible with those rules and the target implementation. [WGSL floating-point evaluation](https://www.w3.org/TR/WGSL/#floating-point-evaluation).

## From BAREWire to a shader buffer

BAREWire supplies shared encoding and decoding across its supported .NET, JavaScript and native Clef paths. A specific WREN bridge still needs its actual framing, version, size, ownership and endpoint integration checked. The GPU receives a second representation: the layout expected by the shader. Wire decoding and GPU upload therefore need an explicit correspondence even if a particular schema is designed so that their byte layouts coincide.

WGSL makes the risk visible. An array of `vec3<f32>` has a 16-byte element stride, although each vector has three four-byte scalar components. Two tightly packed triples occupy 24 bytes; two array elements at that stride occupy 32. Copying the tightly packed bytes into an otherwise valid GPU buffer gives the shader the wrong second element.

```text
Wire values:       x1 y1 z1 | x2 y2 z2
WGSL array slots:  x1 y1 z1 pad | x2 y2 z2 pad
Byte offsets:       0  4  8  12 | 16 20 24 28
```

The exact wire format, endianness and interpretation come from its declared schema, not from this illustration. A layout helper should compute field offsets, alignment, array stride, total extent and any padding for the admitted shader declaration. It should check lengths, integer overflow in size calculations, conversion range, device limits and required buffer usage before upload. The shader declaration, pipeline binding and host writer must use matching layouts. [WGSL memory layout](https://www.w3.org/TR/WGSL/#memory-layout).

API validation can reject invalid resource usage or an incompatible declared binding. It does not know whether the correctly sized bytes represent the intended node, field, amount or revision. Shader-access safety and meaningful application results have separate requirements. Use deliberately distinguishable values and padding in tests so that an incorrectly packed second or later element cannot accidentally look correct.

## Coherent reactive updates

A BAREWire message can be well formed and still arrive for an old selection or graph revision. A coherent picture must associate the accepted graph data, layout, labels, picking information and evidence details with compatible versions.

The proposed update path is:

```text
versioned BAREWire message
    → validate framing, schema, identity and revision
    → apply one coherent Ranvier batch
    → derive a visual snapshot and its dependencies
    → renderer owner admits preparation and GPU work
    → present and inspect matching data and labels
```

Signals represent changing state. Commands, completion and failure are occurrences with their own identity. A batch can keep observers from seeing a half-applied local update. The renderer still needs a policy for asynchronous layout, resource creation, upload, submission and presentation. Do not infer whole-scene consistency from equality of one field, or assume that a stable object reference proves its mutable bytes have not changed.

### A renderer owner and its mailbox

A renderer actor or equivalent serialized owner is a candidate for coordinating the lifecycle. It may own multiple areas and batches of resources. One visual area does not require its own actor, thread or GPU surface. The schematic messages below name the decisions without claiming these exact actor declarations or opaque GPU values already compile under Fickl or Clef:

```fsharp
type RendererMessage =
    | ApplySnapshot of revision: int64 * snapshot: VisualSnapshot
    | ApplyDelta of baseRevision: int64 * revision: int64 * delta: VisualDelta
    | LayoutReady of revision: int64 * generation: int64 * layout: Layout
    | DeviceLost of generation: int64
    | Dispose
```

The owner accepts a snapshot, records the current revision, and starts preparation. A completion returns as a message carrying the graph and layout identities it used. The actor decides whether it is still eligible before installing its results. The mailbox serializes those decisions. A WebGPU queue and any worker execute asynchronously, with their own completion and resource contracts.

The visual snapshot is the renderer's coherent input: the admitted data revision and scope, scene generation, visible node and incidence data, labels, camera and selection, together with the mapping from rendered instance or pick identifier to its source object. A pick target distinguishes a node, a relationship and an incidence of that relationship. Install geometry, labels and the matching pick table as one accepted scene. A pick from a retired scene is resolved against that scene only while its mapping remains available, or discarded and retried against the current scene. The exact projection schema belongs to the graph's publishing contract; this is the information the renderer needs to preserve.

Classify updates by their actual dependencies. A camera-only change can normally reuse scene geometry. A selection or proof-status change may update attributes and labels while retaining positions. A changed projection or layout may require new geometry and a matching pick table. A replacement device needs resources recreated for its new generation. These are scheduling opportunities, and each renderer must verify which resources its implementation actually depends on.

**A discriminating sequence:** R1 arrives and begins layout. R2 arrives before R1 finishes and becomes current. R2 completes and is presented. R1 then completes. Its result remains a valid calculation about R1, but the current-view policy rejects it instead of replacing R2. The labels, pick map and proof details presented for R2 must come from R2's admitted snapshot as well.

Now a device loss retires generation G1 and a replacement device creates generation G2. An old G1 completion arrives carrying the otherwise-current R2 identity. It is ineligible for G2: device generation and data revision answer different questions. Retire G1 resources only under their completion and host contracts. A cancellation request alone does not establish that a producer, worker or GPU has stopped accessing its buffers.

### Snapshots, deltas and backpressure

A complete snapshot can supersede an older unpresented snapshot when the application explicitly selects that policy and no required occurrence is lost. A delta has a base revision and an order. Apply it only to the base for which it was generated. If its base is missing or out of order, follow the protocol's recovery rule, such as requesting a full snapshot. Do not discard an intermediate structural change merely because a newer message arrived; it might be necessary to interpret the later delta.

Bound queued work and data transfer. The policy must say which complete snapshots may be coalesced, which commands or completions must be retained, and how the UI reports a temporarily incomplete or stale view. When cross-revision identity has not been established by the graph projection's owner, preserve selection within a known snapshot and identify any cross-revision mapping as provisional. A hash, label or visual position alone is not an identity guarantee.

## GPU resource and error boundaries

At initialization, check the actual WebView's secure origin, WebGPU availability, adapter and device acquisition, limits and features required by the chosen rendering path. Report whether a capability is unavailable, unsupported, fails validation or is lost during use. An ordinary GPU-accelerated WebView does not by itself prove that its embedded page can acquire WebGPU.

The selected renderer records which buffers and textures belong to which device generation, usage flags, submitted work and owning visual scope. On update or disposal, it releases resources only after their relevant users have retired. The implementation may keep data GPU-resident and batch updates; correctness does not require waiting for all GPU work after every frame.

Track the users separately: submitted GPU work follows the graphics API's lifetime rules, while mapped views, transferred worker buffers, pending promises and application references have their own lifetimes. Disposal stops new work for the retired owner and makes late completions ineligible. Device recovery rebuilds the scene from retained admitted application data and the current visual snapshot, rather than relying on resources from the lost device.

For a CPU readback, mapping has a pending period. A successful `mapAsync` establishes that the mapped buffer's requested range is ready; it is not a blanket completion signal for every independent operation. `getMappedRange()` returns views on mapped data. Copy any data needed afterward before `unmap()`, because unmapping detaches those views. Device loss and mapping failure need their own paths. [WebGPU buffer mapping and promise ordering](https://www.w3.org/TR/webgpu/#buffer-mapping).

F# wrappers can encode useful resource states and centralize checks. Mutable aliases or foreign references can still violate a protocol unless the implementation controls them. Clef's future resource and use judgments may make stronger claims possible, subject to their own formal rules and target correspondence. A generated TypeScript binding alone provides API access, not a proof of one-shot consumption or lifetime safety.

## Proof expression and the realized artifact

The selected visualizer should make Tier 1 structural facts tangible. When the graph supplies a field's identity and dimension, an incidence role, or a permitted use under an admitted rule, the interface can expose that established relationship and its remaining obligations. It must identify the graph operation and revision to which the evidence applies.

A checkable end-to-end claim for a future Clef path might state: the decoded record carries the same selected value and unit as the source projection; the admitted layout conversion writes it to the declared shader slot; the realized operation uses that slot for the corresponding mark; and the label resolves to the same record and revision. Such a claim needs its actual premises, a checker or proof at each admitted boundary, and a preservation or validation argument for the delivered JavaScript, WGSL, runtime and device behavior it covers.

An exact value, a matching buffer layout, a valid WebGPU submission and a visual inspection establish different facts. Fickl's existing F# tests and compiled-JavaScript tests can provide valuable executable evidence today. They do not by themselves establish Clef's proposed tiered proof system or a verified GPU compiler. Bind each reported result to the build, schema, shader, browser/WebView, GPU and observation actually checked.

## Discriminating acceptance cases

Use one small end-to-end case before scaling the workload. Each negative case should fail at the boundary responsible for the rejected claim, with a useful explanation and no silently substituted value.

| Case | Positive result | Discriminator |
|---|---|---|
| Joint relation | A, B and C select the same capacity hyperedge with their roles and the `12 <= 10` obligation. | Omit C or change the role; the view reports an incomplete or different projection. |
| Exact money | 1,999 USD cents survives encoding and decoding and labels the correct record. | Change currency or send a malformed value; reject before it becomes a valid amount. |
| Conversion | A declared, bounded display scale produces an admitted coordinate. | Cross the accepted integer-to-`Number` boundary or overflow a checked intermediate; refuse the conversion. |
| Buffer layout | Two distinct vectors occupy the declared `vec3<f32>` slots and read back or render as expected. | Send tightly packed triples without the required array stride; the layout check or comparison fails. |
| Reactive snapshot | A complete update changes geometry, labels and picking under the same revision. | Interleave old labels with new geometry; reject or retain the earlier coherent view. |
| Delayed completion | R2 remains current when R1 completes late. | Deliver R1 after R2 and attempt to install it as current. |
| Device replacement | G2 owns its new resources after G1 is lost. | Deliver a G1 result after restart, including one for the current graph revision. |
| Delta order | A delta applies to its declared base and produces the stated revision. | Drop or reorder a prerequisite delta; request recovery rather than applying it to another base. |
| Resource lifetime | A buffer retires after its admitted users finish. | Dispose during worker or GPU use; verify no reuse while an admitted user can still access it. |
| Target capability | A supported host obtains the requested adapter and renders. | Deny an essential feature or simulate device loss; surface the condition and follow the declared alternative. |

Run the same bounded case through a regular browser and WrenHello's actual WebView, then add named mobile hosts as their integrations exist. Record the toolchain, JS and shader artifacts, target capabilities, graph/input revision, observations and failure results. The performance roadmap can then measure larger workloads without losing the small case that makes each integrity claim inspectable.

## References and related design

- [Atelier visualization strategy](05_webgpu.md)
- [Beancounter: Integrity in the JavaScript substrate](https://forge.spkez.dev/Braidpoint/beancounter/wiki/22_javascript_substrate)
- [WebGPU specification](https://www.w3.org/TR/webgpu/) and [WGSL specification](https://www.w3.org/TR/WGSL/)
- [Carrying Proofs into JavaScript](https://clef-lang.com/blog/carrying-proofs-into-javascript/)
