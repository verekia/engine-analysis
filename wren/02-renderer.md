# 02 — Sort-Based Renderer

The renderer is the heart of Wren's performance story. Instead of issuing draw calls during scene traversal, it collects them into a flat array, sorts by a 64-bit key that encodes optimal state-change ordering, then submits draws in sorted order through the HAL.

## Why Sort-Based Rendering

Three.js traverses the scene graph and issues draw calls in traversal order, with ad-hoc sorting for transparent objects. This means:
- Frequent shader switches (expensive)
- Frequent texture rebinds (expensive)
- Poor early-z utilization for opaque geometry
- Transparent objects require manual `renderOrder` management

A sort-based renderer solves all of these by construction.

## Render Key Encoding

Each drawable produces a 64-bit sort key packed into two 32-bit integers (JS doesn't have native uint64, but we can sort by high word then low word):

### Opaque Objects (sorted front-to-back within material groups)

```
Bits 63-62: Layer (0=world, 1=effects, 2=overlay, 3=HUD)
Bits 61-60: Pass (0=shadow, 1=depth-prepass, 2=opaque, 3=post)
Bit  59:    Transparency flag (0 = opaque)
Bits 58-48: Pipeline ID (11 bits → 2048 unique pipelines)
Bits 47-32: Material ID (16 bits → 65536 unique materials)
Bits 31-0:  Depth (32-bit float quantized to uint32, front-to-back)
```

For opaque geometry, we sort by pipeline first (minimizing the most expensive state change), then material (minimizing texture/uniform rebinds), then depth (for early-z rejection).

### Transparent Objects (sorted back-to-front)

```
Bits 63-62: Layer
Bits 61-60: Pass (set to OIT pass)
Bit  59:    Transparency flag (1 = transparent)
Bits 58-27: Depth (32-bit float quantized, back-to-front / inverted)
Bits 26-16: Pipeline ID
Bits 15-0:  Material ID
```

For transparent geometry with WBOIT, strict sort order is not required (OIT handles it). But we still sort by depth for best visual results with the weighted approximation, with material as a secondary criterion to reduce state changes.

## Draw Call Collection

During scene traversal, each visible `Mesh` node pushes a `DrawCommand` into a flat array:

```ts
interface DrawCommand {
  sortKeyHigh: number        // Upper 32 bits of sort key
  sortKeyLow: number         // Lower 32 bits of sort key
  pipelineId: number         // HAL pipeline handle index
  bindGroups: BindGroupHandle[] // [perFrame, perMaterial, perObject]
  vertexBuffers: GPUBufferHandle[]
  indexBuffer: GPUBufferHandle | null
  indexFormat: IndexFormat
  indexCount: number
  vertexCount: number
  firstIndex: number
  baseVertex: number
}
```

The array is pre-allocated to a fixed capacity (e.g., 4096) and reused each frame. A write index tracks the current count. Zero allocations per frame.

## Sorting

**Radix sort** on the 64-bit key, implemented as two passes over 32-bit words (high word first for primary sort, low word for secondary). Radix sort is:

- O(n) for fixed-width keys
- Cache-friendly (sequential memory access)
- Stable (preserves insertion order for equal keys)
- Predictable performance (no worst cases like quicksort)

For 2000 draw calls, radix sort completes in < 0.1ms.

```ts
// Simplified radix sort for draw commands by 64-bit key
const radixSortDrawCommands = (commands: DrawCommand[], count: number): void => {
  // Pass 1: sort by sortKeyLow (lower 32 bits)
  radixSort32(commands, count, 'sortKeyLow')
  // Pass 2: sort by sortKeyHigh (upper 32 bits) — stable sort preserves Pass 1 order
  radixSort32(commands, count, 'sortKeyHigh')
}
```

## State Caching

After sorting, consecutive draw calls often share the same pipeline and/or bind groups. The submission loop tracks current state and skips redundant calls:

```ts
const submitDrawCalls = (encoder: RenderPassEncoder, commands: DrawCommand[], count: number): void => {
  let currentPipeline = -1
  let currentBindGroup0 = null
  let currentBindGroup1 = null
  let currentVB = null
  let currentIB = null

  for (let i = 0; i < count; i++) {
    const cmd = commands[i]

    if (cmd.pipelineId !== currentPipeline) {
      encoder.setPipeline(pipelines[cmd.pipelineId])
      currentPipeline = cmd.pipelineId
      // Pipeline change invalidates all bind groups
      currentBindGroup0 = null
      currentBindGroup1 = null
    }

    if (cmd.bindGroups[0] !== currentBindGroup0) {
      encoder.setBindGroup(0, cmd.bindGroups[0])
      currentBindGroup0 = cmd.bindGroups[0]
    }

    if (cmd.bindGroups[1] !== currentBindGroup1) {
      encoder.setBindGroup(1, cmd.bindGroups[1])
      currentBindGroup1 = cmd.bindGroups[1]
    }

    // Per-object bind group always changes
    encoder.setBindGroup(2, cmd.bindGroups[2])

    if (cmd.vertexBuffers[0] !== currentVB) {
      encoder.setVertexBuffer(0, cmd.vertexBuffers[0])
      currentVB = cmd.vertexBuffers[0]
    }

    if (cmd.indexBuffer && cmd.indexBuffer !== currentIB) {
      encoder.setIndexBuffer(cmd.indexBuffer, cmd.indexFormat)
      currentIB = cmd.indexBuffer
    }

    if (cmd.indexBuffer) {
      encoder.drawIndexed(cmd.indexCount, 1, cmd.firstIndex, cmd.baseVertex)
    } else {
      encoder.draw(cmd.vertexCount)
    }
  }
}
```

## Bind Group Strategy

Three bind group slots, ordered by update frequency:

| Slot | Name | Update Frequency | Contents |
|------|------|-----------------|----------|
| 0 | Per-Frame | Once per frame | Camera matrices (view, projection, viewProjection), time, ambient light, CSM matrices, CSM textures |
| 1 | Per-Material | Once per material | Albedo color, texture, AO texture, sampler, material-specific uniforms |
| 2 | Per-Object | Once per object | World matrix, normal matrix, bone matrices (skinned), material index color table |

This hierarchy means sorting by pipeline+material naturally groups objects that share bind groups 0 and 1, minimizing rebinds.

## Frame Render Pipeline

A single frame executes these passes in order:

```
1. Scene graph traversal
   ├── Update dirty world matrices
   ├── Frustum cull (populate visible list)
   └── Collect DrawCommands from visible nodes

2. Sort DrawCommands by render key

3. Shadow pass (for each CSM cascade)
   ├── Filter opaque draw commands
   ├── Set light-space camera
   └── Submit depth-only draws

4. Opaque pass
   ├── Set main camera
   ├── Bind shadow map textures
   └── Submit opaque draw commands (front-to-back)

5. Transparency pass (WBOIT)
   ├── Switch to accumulation/revealage render targets
   └── Submit transparent draw commands

6. WBOIT composite pass
   └── Fullscreen quad composites transparency over opaque

7. Post-processing
   ├── Bloom (threshold → downsample → upsample → composite)
   └── MSAA resolve (if applicable)

8. Present to screen
```

## WebGPU Render Bundles Integration

On WebGPU, static geometry (meshes whose transform, material, and visibility don't change frame-to-frame) can be recorded into render bundles:

1. After sorting, identify runs of consecutive static draw commands
2. Record each run into a `GPURenderBundle`
3. Cache the bundle, keyed by the set of included objects
4. On subsequent frames, if the set hasn't changed, replay the bundle instead of re-submitting individual draws
5. Invalidation triggers: object added/removed from scene, material changed, frustum culling result changed

This optimization is transparent — the renderer tries bundles first and falls back to individual draw calls for dynamic content.

## Performance Characteristics

For 2000 draw calls on mobile:

| Phase | Budget |
|-------|--------|
| Traversal + culling | ~0.3ms |
| Sort (radix) | ~0.1ms |
| Draw submission (with state cache) | ~2ms |
| GPU work | ~12ms |
| **Total** | **~14.4ms** (within 16.6ms budget) |

The key insight: by sorting, we reduce GPU state changes from O(n) worst case to O(unique_pipelines + unique_materials), which is typically 10-50 state changes for 2000 objects. This is what enables high draw-call counts on mobile where driver overhead per state change is severe.
