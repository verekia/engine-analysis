# Performance Strategy

## Target

**2000+ draw calls at 60 fps on recent mobile phones** (iPhone 14+, Pixel 7+,
Samsung S23+). This means completing all CPU work + GPU work within a 16.67 ms
frame budget.

### Frame Budget Breakdown

```
Total budget:         16.67 ms
─────────────────────────────
CPU work:             ~6 ms
  User logic:           1.0 ms
  Animation update:     0.5 ms
  Scene graph update:   0.3 ms  (50 dirty of 2000, SoA + dirty flags)
  Frustum culling:      0.15 ms (BVH-accelerated)
  Draw key sort:        0.05 ms (radix sort, 2000 keys)
  Ring buffer write:    0.4 ms  (2000 × 208 bytes sequential write)
  Command execution:    3.0 ms  (2000 draw calls, ~1.5 µs each with sorting)
  Pool reset + misc:    0.1 ms

GPU work:             ~8 ms
  Shadow maps (3 cas):  2.5 ms  (depth-only, ~1500 objects)
  Opaque geometry:      3.0 ms  (2000 draws, simple shaders)
  OIT pass:             0.5 ms  (transparent objects, typically few)
  MSAA resolve:         0.3 ms
  Bloom (5 levels):     1.2 ms  (half-res chain)
  OIT composite + blit: 0.5 ms

Headroom:             ~2.67 ms
```

## Performance Pillars

### 1. Command Buffer + Radix Sort

Instead of issuing draw calls as objects are encountered, the renderer collects
draw commands into a flat pre-allocated array, sorts them, then executes.

**Draw Sort Key (64-bit)**

```
Bits 63–56 (8 bits):   Pipeline ID        (up to 256 unique pipelines)
Bits 55–44 (12 bits):  Material ID         (up to 4096 materials)
Bits 43–32 (12 bits):  Texture hash         (up to 4096 texture combos)
Bits 31–0  (32 bits):  Depth (front-to-back for opaque)
```

Sorting by this key minimizes GPU state changes:
- Pipeline switches (most expensive) happen first in sort order
- Material switches (moderate cost) happen next
- Depth sorting within the same material maximizes early-Z rejection

**Radix Sort** operates on the 64-bit keys in O(n) time. For 2000 keys, this
completes in < 0.05 ms. A traditional comparison sort (O(n log n)) would be
~2× slower and have worse cache behavior.

```typescript
// 8-bit radix sort on 64-bit keys, 8 passes
const radixSort = (keys: BigUint64Array, indices: Uint32Array, count: number) => {
  const histogram = new Uint32Array(256)
  const tempKeys = new BigUint64Array(count)
  const tempIndices = new Uint32Array(count)

  for (let pass = 0; pass < 8; pass++) {
    const shift = BigInt(pass * 8)

    // Histogram
    histogram.fill(0)
    for (let i = 0; i < count; i++) {
      histogram[Number((keys[i] >> shift) & 0xFFn)]++
    }

    // Prefix sum
    let sum = 0
    for (let i = 0; i < 256; i++) {
      const count = histogram[i]
      histogram[i] = sum
      sum += count
    }

    // Scatter
    for (let i = 0; i < count; i++) {
      const bucket = Number((keys[i] >> shift) & 0xFFn)
      const dest = histogram[bucket]++
      tempKeys[dest] = keys[i]
      tempIndices[dest] = indices[i]
    }

    // Swap
    keys.set(tempKeys.subarray(0, count))
    indices.set(tempIndices.subarray(0, count))
  }
}
```

### 2. Uniform Ring Buffer

The single biggest cost in naive renderers is uploading per-object uniforms.
Three.js calls `gl.uniformMatrix4fv` for each object individually. Mantis
writes all per-object data into a single ring buffer and uses dynamic offsets.

**Per-Object Uniform Block (208 bytes, padded to 256 for alignment)**

```
Offset 0:    mat4 modelMatrix       (64 bytes)
Offset 64:   mat3x4 normalMatrix    (48 bytes, mat3 padded to std140)
Offset 112:  vec4 customData        (16 bytes — reserved)
Offset 128:  padding                (128 bytes — alignment to 256)
```

**Why 256-byte alignment**: WebGPU requires `minUniformBufferOffsetAlignment`
of 256 bytes. WebGL2's `UNIFORM_BUFFER_OFFSET_ALIGNMENT` is typically 256 bytes
on mobile.

**Ring Buffer Upload**

```typescript
const uploadRingBuffer = (device: GALDevice, objects: DrawCommand[], count: number) => {
  const STRIDE = 256
  const totalBytes = count * STRIDE
  const writeOffset = ringBufferHead

  // Write all objects sequentially into a staging Float32Array
  for (let i = 0; i < count; i++) {
    const obj = objects[i]
    const byteOffset = writeOffset + i * STRIDE
    const floatOffset = byteOffset / 4

    // Copy model matrix (16 floats) directly from SoA array
    stagingBuffer.set(
      worldMatrices.subarray(obj.nodeIndex * 16, obj.nodeIndex * 16 + 16),
      floatOffset
    )
    // Copy normal matrix (12 floats)
    computeNormalMatrix(obj.nodeIndex, stagingBuffer, floatOffset + 16)
  }

  // Single GPU upload for all 2000 objects
  device.writeBuffer(ringBuffer, writeOffset, stagingBuffer, 0, totalBytes)
  ringBufferHead = (writeOffset + totalBytes) % ringBufferCapacity
}
```

**Cost**: One `writeBuffer` call for 2000 objects (512 KB) instead of 2000
individual uniform calls. On mobile, this reduces driver overhead by 10–50×.

### 3. SoA Transform Storage

The scene graph stores transforms in Structure-of-Arrays layout for cache
efficiency:

```
Traditional (AoS):  [Node0{pos,rot,scl,mat}, Node1{pos,rot,scl,mat}, ...]
Mantis (SoA):       positions[x0,y0,z0, x1,y1,z1, ...]
                    rotations[x0,y0,z0,w0, x1,y1,z1,w1, ...]
                    scales[x0,y0,z0, x1,y1,z1, ...]
                    worldMatrices[m0[16], m1[16], ...]
                    dirtyFlags[d0, d1, ...]
                    parentIndices[p0, p1, ...]
```

**Why SoA is faster:**
- When updating world matrices, we iterate over `positions`, `rotations`,
  `scales`, and `worldMatrices` — all contiguous arrays. CPU prefetchers
  handle sequential access patterns optimally.
- When uploading to the ring buffer, we copy directly from `worldMatrices` —
  a single `subarray` view, no object traversal.
- Dirty flag checking is a linear scan of a `Uint8Array` — fits in L1 cache
  for 2000 objects (2 KB).

### 4. Dirty Flag Propagation

Only ~2–5% of objects typically move each frame (50 of 2000). Mantis skips
recomputing world matrices for clean objects:

```typescript
const updateWorldMatrices = () => {
  // Breadth-first order guarantees parents update before children
  for (let i = 0; i < nodeCount; i++) {
    if (!dirtyFlags[i]) continue

    const parentIdx = parentIndices[i]
    if (parentIdx === -1) {
      // Root node — world matrix = local matrix
      composeMatrix(i, worldMatrices, positions, rotations, scales)
    } else {
      // Child — world = parent.world × local
      composeMatrix(i, localMatrixTemp, positions, rotations, scales)
      multiplyMatrices(worldMatrices, parentIdx * 16, localMatrixTemp, 0, worldMatrices, i * 16)
    }

    dirtyFlags[i] = 0
  }
}
```

When a node is dirtied, all descendants are also marked dirty via a
pre-computed children list. This cascading happens immediately on `setPosition`
etc., not deferred.

### 5. Frustum Culling via BVH

For 2000 objects, brute-force culling (6 plane tests × 2000 = 12,000 tests)
takes ~0.3 ms. The scene-level BVH reduces this to ~400 tests by skipping
entire subtrees whose bounding boxes are outside the frustum.

```
Brute force:  12,000 plane tests  →  ~0.3 ms
BVH-based:       400 plane tests  →  ~0.05 ms (6× faster)
```

The BVH is rebuilt incrementally: only nodes whose AABBs changed are refitted.
A full rebuild is triggered only when cumulative AABB growth exceeds 2× the
original.

### 6. Shader Variant Cache

Shader compilation is expensive (10–100 ms per variant). Mantis pre-compiles
common variants during engine initialization and lazy-compiles rare ones:

**Pre-compiled on init (4 variants):**
1. Lambert + vertex colors + material index + shadows
2. Lambert + color texture + shadows
3. Basic (unlit) + vertex colors
4. Basic (unlit) + color texture

**Lazy-compiled on first use:**
- Any combination with skinning, AO texture, etc.

To avoid first-frame hitches, the asset loader can trigger shader compilation
for all variants a loaded model needs before the model is added to the scene.

### 7. Minimal State Changes (Sorted Execution)

The draw key sorting ensures that in the best case, consecutive draw calls
differ only in their ring buffer offset. Measured state change frequency for a
typical scene with 2000 objects and 8 materials:

```
Pipeline switches:    ~4   (once per pipeline)
Material switches:    ~8   (once per material)
Texture switches:     ~12  (once per texture combo)
Vertex buffer swaps:  ~50  (once per unique geometry)
Dynamic offset only:  ~1926 (just bind group offset change)
```

Each "dynamic offset only" draw call costs approximately:
- WebGPU: `setBindGroup` + `drawIndexed` → ~0.8 µs JS + GPU overhead
- WebGL2: `bindBufferRange` + `drawElements` → ~1.2 µs JS + GPU overhead

### 8. Memory Bandwidth Optimization

Mobile GPUs have limited memory bandwidth. Mantis optimizes for this:

- **16-bit indices** (`Uint16Array`) used by default for meshes < 65,536
  vertices. Halves index bandwidth vs 32-bit.
- **Interleaved vertex attributes**: position (12B) + normal (12B) + UV (8B)
  packed into 32-byte stride for cache-line alignment.
- **Half-resolution bloom**: The bloom downsample chain starts at half canvas
  resolution, reducing fragment shader bandwidth by 4× for the first level.
- **Compressed textures**: KTX2/Basis transcodes to ASTC 4×4 on mobile —
  4 bits/pixel vs 32 bits/pixel for uncompressed RGBA8.

### 9. Avoiding GC Pressure

JavaScript garbage collection pauses are the enemy of consistent 60 fps.
Mantis eliminates GC pressure during rendering:

| Source of allocation | Mantis approach |
|---|---|
| Temporary vectors/matrices | Scratch pool with frame-scoped reset |
| Draw command objects | Pre-allocated flat array, reused |
| Sort key arrays | Pre-allocated, grown via doubling (rare) |
| Uniform upload staging | Pre-allocated staging buffer |
| Event callbacks | No closures created per frame |
| String concatenation | No string ops in render loop |

The scratch pool provides temporary math objects:

```typescript
// Acquire a scratch Vec3 — valid until end of frame
const temp = vec3Pool.acquire()  // returns view into pre-allocated Float32Array
// ... use temp ...
// At frame end: vec3Pool.reset() — zero-cost "deallocation"
```

### 10. WebGPU-Specific Optimizations

When running on WebGPU, Mantis leverages features unavailable in WebGL2:

- **Render bundles** for static geometry — pre-recorded GPU command sequences
  that replay at near-zero CPU cost
- **Compute shaders** for BVH traversal during frustum culling (optional,
  falls back to CPU on WebGL2)
- **Timestamp queries** for GPU profiling (development mode only)
- **Indirect drawing** potential — not used in v1 but the command buffer
  architecture is designed to support `drawIndexedIndirect` in the future

### Profiling Hooks

In development mode, the renderer emits timing data:

```typescript
engine.onProfile((stats) => {
  stats.cpuTime          // Total CPU frame time
  stats.gpuTime          // GPU frame time (WebGPU timestamp queries)
  stats.drawCalls        // Number of draw calls issued
  stats.pipelineSwitches // Number of pipeline state changes
  stats.triangles        // Total triangles rendered
  stats.culled           // Objects rejected by frustum culling
})
```
