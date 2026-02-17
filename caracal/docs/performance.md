# Performance Architecture

## Target: 2000 Draw Calls at 60 fps on Modern Phones

This is the primary performance target. Here's a detailed breakdown of every optimization strategy Caracal employs.

## Bottleneck Analysis

At 60 fps, each frame has a **16.6 ms budget**. For 2000 draw calls:

| Phase | Budget | Strategy |
|-------|--------|----------|
| JS overhead (per draw call) | < 3 µs × 2000 = 6 ms | Minimize per-draw JS work |
| State changes | < 2 ms | State sorting, minimize switches |
| Uniform uploads | < 2 ms | UBOs, batch uploads |
| GPU draw submission | < 3 ms | Multi-draw (WebGL), render bundles (WebGPU) |
| Shadow pass | < 3 ms | Depth-only shader, frustum cull per cascade |
| Post-processing | < 2 ms | Reduced-resolution bloom, minimal passes |
| **Total** | **< 16.6 ms** | |

For comparison, Three.js spends roughly **15-30 µs per draw call** in JS overhead (material validation, uniform setup, state management). At 2000 draws, that's 30-60 ms — already over budget before any GPU work.

Caracal targets **< 3 µs per draw call** in JS overhead through the strategies below.

## Strategy 1: State Sorting

The single most impactful optimization. Draw calls are sorted to minimize GPU state changes:

```
Sort order (most expensive → least expensive state change):
1. Shader program / render pipeline
2. Material (textures, uniforms)
3. Geometry (VAO / vertex buffers)
4. Depth (front-to-back for early-Z rejection)
```

### Sort Key Encoding

Each draw call is assigned a 32-bit sort key that encodes all four criteria:

```typescript
// Encode sort key: 8 bits shader | 8 bits material | 8 bits geometry | 8 bits depth
const encodeSortKey = (
  shaderIndex: number,    // 0-255 (realistically < 20 shader variants)
  materialIndex: number,  // 0-255 (materials hashed to compact indices)
  geometryIndex: number,  // 0-255 (geometries hashed)
  depth: number           // 0-255 (quantized from near to far)
): number => {
  return (shaderIndex << 24) | (materialIndex << 16) | (geometryIndex << 8) | depth
}
```

### Radix Sort

Sort keys are sorted using a **radix sort** (O(n), stable, allocation-free):

```typescript
// Single-pass radix sort on 32-bit keys
// Uses pre-allocated scratch arrays — zero allocation
const radixSort = (
  keys: Uint32Array,
  indices: Uint32Array,
  count: number,
  scratch: Uint32Array,
  scratchIndices: Uint32Array
) => {
  // 4 passes, 8 bits each (256 buckets per pass)
  for (let shift = 0; shift < 32; shift += 8) {
    // Count occurrences
    const counts = _radixCounts // pre-allocated Uint32Array(256)
    counts.fill(0)
    for (let i = 0; i < count; i++) {
      counts[(keys[i] >> shift) & 0xFF]++
    }

    // Prefix sum
    let total = 0
    for (let i = 0; i < 256; i++) {
      const c = counts[i]
      counts[i] = total
      total += c
    }

    // Scatter
    for (let i = 0; i < count; i++) {
      const bucket = (keys[i] >> shift) & 0xFF
      const dest = counts[bucket]++
      scratch[dest] = keys[i]
      scratchIndices[dest] = indices[i]
    }

    // Swap
    keys.set(scratch.subarray(0, count))
    indices.set(scratchIndices.subarray(0, count))
  }
}
```

This sorts 2000 draw calls in roughly **0.1 ms** — negligible.

## Strategy 2: Uniform Buffer Objects (UBOs)

### Per-Frame Data (uploaded once)

```
CameraUBO (binding 0): 256 bytes
├── viewMatrix (64 bytes)
├── projectionMatrix (64 bytes)
├── viewProjectionMatrix (64 bytes)
├── cameraPosition (16 bytes)
├── near, far, time, padding (16 bytes)
└── viewport (16 bytes)

LightsUBO (binding 1): 320 bytes
├── directional light data
├── ambient light data
├── shadow matrices (3 cascades)
└── cascade split distances
```

Total per-frame upload: **576 bytes** — trivial.

### Per-Draw-Call Data

Only the model transform is uploaded per draw call:

```
ModelUBO (binding 2): 128 bytes
├── worldMatrix (64 bytes)
└── normalMatrix (64 bytes)   // inverse transpose of upper 3×3
```

For 2000 draws: 2000 × 128 = **256 KB** of uniform data per frame. On modern GPUs, this bandwidth is negligible.

### WebGL 2: bufferSubData with Ring Buffer

To avoid stalls when updating the ModelUBO every draw call, use a ring buffer approach:

```typescript
// Allocate a large UBO that can hold N model matrices
const MODEL_UBO_SIZE = 128        // bytes per model
const MAX_DRAWS_PER_FRAME = 4096
const ringBuffer = gl.createBuffer()
gl.bindBuffer(gl.UNIFORM_BUFFER, ringBuffer)
gl.bufferData(gl.UNIFORM_BUFFER, MODEL_UBO_SIZE * MAX_DRAWS_PER_FRAME * 3, gl.DYNAMIC_DRAW)
// 3× for triple buffering

let ringOffset = 0

const uploadModelMatrix = (worldMatrix: Float32Array, normalMatrix: Float32Array) => {
  const offset = ringOffset * MODEL_UBO_SIZE
  gl.bufferSubData(gl.UNIFORM_BUFFER, offset, worldMatrix)
  gl.bufferSubData(gl.UNIFORM_BUFFER, offset + 64, normalMatrix)
  gl.bindBufferRange(gl.UNIFORM_BUFFER, 2, ringBuffer, offset, MODEL_UBO_SIZE)
  ringOffset = (ringOffset + 1) % (MAX_DRAWS_PER_FRAME * 3)
}
```

### WebGPU: Dynamic Uniform Buffer

WebGPU supports dynamic offsets in bind groups, allowing a single large buffer with per-draw offsets:

```typescript
const modelBuffer = device.createBuffer({
  size: 256 * MAX_DRAWS,  // 256-byte aligned per draw
  usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
})

// Per frame: write all model matrices at once
device.queue.writeBuffer(modelBuffer, 0, allModelMatrices)

// Per draw call: set dynamic offset
renderPass.setBindGroup(2, modelBindGroup, [drawIndex * 256])
```

## Strategy 3: Minimize Per-Draw-Call JS Work

### What Caracal Does Per Draw Call (WebGL 2)

```
1. Check if shader changed → if yes: gl.useProgram (expensive)
2. Check if material changed → if yes: bind textures + update material UBO
3. Check if geometry changed → if yes: gl.bindVertexArray
4. Update model UBO offset (always)
5. gl.drawElements (always)
```

With state sorting, steps 1-3 are skipped for most consecutive draws. The effective per-draw cost is:
- 1 bufferSubData (128 bytes)
- 1 bindBufferRange
- 1 drawElements

This is approximately **2-3 µs per draw call** in JS.

### What Three.js Does Per Draw Call (for comparison)

```
1. Validate material needsUpdate flags
2. Compile/upload uniforms one-by-one (not UBO)
3. Set up each texture individually
4. Check and set each GL state bit
5. Upload model matrix via gl.uniformMatrix4fv
6. Compute and upload normal matrix
7. Multiple redundant state checks
8. gl.drawElements
```

This is approximately **15-30 µs per draw call** in JS.

## Strategy 4: WebGPU Render Bundles

For static geometry (which in a typical game is 80-90% of the scene), render bundles eliminate nearly all per-frame CPU cost:

```typescript
// Built once (or when static scene changes)
const staticBundle = buildRenderBundle(device, staticMeshes, camera)

// Each frame: just execute
renderPass.executeBundles([staticBundle])

// Only dynamic objects need per-frame draw calls
for (const mesh of dynamicMeshes) {
  drawMesh(renderPass, mesh)
}
```

For a scene with 2000 objects where 1800 are static:
- Static: 0 CPU cost per frame (bundle execution is a single GPU command)
- Dynamic: 200 individual draw calls ≈ 0.6 ms

## Strategy 5: WEBGL_multi_draw

On WebGL 2, when available, batch draws that share the same shader and VAO:

```typescript
const ext = gl.getExtension('WEBGL_multi_draw')

// If 50 objects share the same material and geometry buffer:
// Instead of 50 × gl.drawElements = 50 driver calls
// → 1 × ext.multiDrawElementsWEBGL = 1 driver call
ext.multiDrawElementsWEBGL(
  gl.TRIANGLES,
  countsArray, 0,           // triangle count per draw
  gl.UNSIGNED_SHORT,
  offsetsArray, 0,          // index buffer offset per draw
  50                         // number of draws in batch
)
```

This is particularly effective for scenes with many objects sharing the same material (common in low-poly games).

## Strategy 6: Zero Allocation in the Render Loop

All temporary objects used during rendering are pre-allocated at module scope:

```typescript
// Pre-allocated temporaries — never GC'd
const _tempVec3 = new Vec3()
const _tempMat4 = new Mat4()
const _tempQuat = new Quat()
const _frustumPlanes = new Float32Array(24)      // 6 planes × 4 components
const _sortKeys = new Uint32Array(MAX_DRAW_CALLS)
const _sortIndices = new Uint32Array(MAX_DRAW_CALLS)
const _scratchKeys = new Uint32Array(MAX_DRAW_CALLS)
const _scratchIndices = new Uint32Array(MAX_DRAW_CALLS)
```

**No `new` calls in the render loop. No array creation. No object creation. No closures.**

This eliminates GC pauses, which on mobile can cause 5-10 ms frame spikes.

## Strategy 7: Frustum Culling

Before any draw call submission, each mesh is tested against the camera frustum:

```typescript
const frustumCull = (frustum: Float32Array, aabb: AABB): boolean => {
  // Test AABB against 6 frustum planes
  for (let i = 0; i < 6; i++) {
    const px = frustum[i * 4]
    const py = frustum[i * 4 + 1]
    const pz = frustum[i * 4 + 2]
    const pw = frustum[i * 4 + 3]

    // Find the AABB vertex most in the direction of the plane normal
    const testX = px > 0 ? aabb.maxX : aabb.minX
    const testY = py > 0 ? aabb.maxY : aabb.minY
    const testZ = pz > 0 ? aabb.maxZ : aabb.minZ

    if (px * testX + py * testY + pz * testZ + pw < 0) {
      return false // Outside this plane → culled
    }
  }
  return true // Inside all planes → visible
}
```

For a 200×200 m world with 2000 objects, only the objects in the camera's view (typically 30-60%) are submitted for rendering. This can cut draw calls by 40-70%.

## Strategy 8: Efficient World Matrix Updates

World matrices are only recomputed for dirty subtrees:

```
Frame 1: 2000 nodes, all dirty → 2000 matrix multiplies (initial)
Frame 2: 10 nodes moved → 10 matrix multiplies + descendants
Frame 3: Nothing moved → 0 matrix multiplies
```

The dirty flag propagation ensures that static objects (the vast majority) have zero per-frame CPU cost for transforms.

## Strategy 9: Flat Typed Array Storage

Critical per-frame data is stored in contiguous typed arrays, not object properties:

```typescript
// Instead of: node.worldMatrix → Mat4 object with .elements Float32Array
// We use: worldMatrices → one big Float32Array for ALL nodes

const worldMatrices = new Float32Array(MAX_NODES * 16)
const worldAABBs = new Float32Array(MAX_NODES * 6)
const sortKeys = new Uint32Array(MAX_DRAW_CALLS)

// Access: worldMatrices.subarray(nodeIndex * 16, nodeIndex * 16 + 16)
```

This improves cache locality for batch operations (culling, sorting, uploading).

## Strategy 10: Shadow Pass Optimization

Shadow rendering is optimized because:

1. **Depth-only shader**: Minimal fragment work (no lighting, no textures)
2. **Reduced geometry**: Only shadow-casting meshes are rendered
3. **Per-cascade culling**: Each cascade frustum-culls independently
4. **Front-face culling**: Render back-faces into shadow map to reduce acne (allows removing bias in many cases)

## Mobile-Specific Optimizations

### Tile-Based GPU Awareness

Mobile GPUs (Adreno, Mali, Apple) are tile-based. Caracal is designed to work with, not against, this architecture:

- **Minimize render target switches**: Each switch forces a tile flush. Caracal's pass structure minimizes target changes.
- **On-chip MSAA**: Tile-based GPUs resolve MSAA for free when the resolve target is the next pass's input. The render pass structure is designed for this.
- **Don't read back**: Never read from framebuffer or textures mid-frame (forces tile resolve)

### Power Consumption

- **RequestAnimationFrame only**: No polling, no setInterval
- **Skip frames when nothing changed**: If no animations, no input, no camera movement → skip the render pass entirely
- **Adaptive quality**: Expose hooks for reducing shadow resolution, bloom levels, or draw distance based on frame time

## Performance Monitoring

Caracal provides built-in performance counters (debug mode only):

```typescript
interface PerfCounters {
  frameTime: number              // Total frame time (ms)
  jsTime: number                 // JS execution time (ms)
  gpuTime: number                // GPU time (ms, if available via EXT_disjoint_timer_query)
  drawCalls: number              // Number of draw calls submitted
  triangles: number              // Total triangles rendered
  stateChanges: {
    shaders: number              // Shader program switches
    materials: number            // Material/texture switches
    geometries: number           // VAO/vertex buffer switches
  }
  culled: number                 // Meshes culled by frustum
  shadowDrawCalls: number        // Draw calls in shadow pass
  nodeCount: number              // Total scene graph nodes
}
```

Counters are collected with near-zero overhead (simple integer increments) and can be disabled entirely in production builds.

## Benchmark Expectations

| Scenario | Target (Phone) | Target (Desktop) |
|----------|----------------|-------------------|
| 2000 static cubes, 1 material | 60 fps | 60 fps |
| 2000 mixed meshes, 10 materials | 60 fps | 60 fps |
| 2000 meshes + shadows (3 cascades) | 45-60 fps | 60 fps |
| 2000 meshes + shadows + bloom | 30-45 fps | 60 fps |
| 500 skinned characters + 1500 static | 45-60 fps | 60 fps |

These targets assume a mid-range 2023+ phone (Snapdragon 8 Gen 1 / Apple A15 equivalent).
