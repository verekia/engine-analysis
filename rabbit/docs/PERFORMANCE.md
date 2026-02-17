# Performance

## Target

**2000 draw calls at 60fps on recent phones** (iPhone 14+, Pixel 7+, Galaxy S23+).

This is the defining performance requirement. Every design decision flows from it.

## Why Three.js Can't Do This

Three.js struggles with high draw call counts because of:

1. **Per-draw-call overhead**: Each `WebGLRenderer.renderObject()` call does ~50-100 GL state changes, many redundant
2. **No state sorting**: Three.js sorts by depth but not by material/program, causing excessive shader switches
3. **Object creation in the hot path**: `new Vector3()`, `new Matrix4()` allocations during rendering trigger GC pauses
4. **Material compilation on demand**: Shader variants are compiled on first use, causing frame drops
5. **Uniform uploads via individual calls**: Each uniform is set individually via `gl.uniform*()` instead of UBOs
6. **No frustum culling by default**: Every object is submitted to the GPU regardless of visibility

Rabbit addresses each of these.

## Draw Call Cost Breakdown

On mobile GPUs, the cost of a single draw call has three components:

| Component | Cost | Mitigation |
|-----------|------|------------|
| JavaScript overhead | ~50μs (three.js) → ~5μs (goal) | Pre-sorted render lists, zero alloc, cached state |
| Driver validation | ~10-30μs | Minimize state changes via sorting |
| GPU command processing | ~2-5μs | Unavoidable per draw call |

At 60fps, we have 16.6ms per frame. Budget:
- 2000 × 5μs JS overhead = 10ms
- 2000 × 15μs driver overhead = 30ms (but most eliminated by sorting)
- With sorting: ~200 state changes × 20μs = 4ms
- Total: ~14ms — fits in 16.6ms budget

## Strategy 1: Material/Pipeline Sort

The single most impactful optimization. Instead of drawing objects in arbitrary order, sort by pipeline ID:

```
Without sorting (2000 objects, 10 materials):
  Shader A → Shader B → Shader A → Shader C → Shader A → ...
  ~2000 pipeline switches (worst case)

With sorting:
  [200 objects with Shader A] → [200 with Shader B] → ... → [200 with Shader J]
  ~10 pipeline switches total
```

Pipeline switches include:
- Binding a different shader program (`gl.useProgram`)
- Setting depth/blend state
- Binding different vertex attribute layouts

Reducing from ~2000 to ~10 switches eliminates ~99.5% of pipeline switch overhead.

### Sort Key Design

```typescript
// Encode sort key as a single number for fast comparison
// Upper 16 bits: pipeline ID (material sort)
// Lower 16 bits: depth (front-to-back within same material)
const computeSortKey = (pipelineId: number, depth: number): number => {
  const depthBits = (Math.min(depth, 65535) | 0) & 0xFFFF
  return (pipelineId << 16) | depthBits
}
```

Sorting 2000 items by a single numeric key takes <0.1ms with `Array.prototype.sort`.

## Strategy 2: WebGL2 State Cache

The WebGL2 backend maintains a shadow copy of all GL state. Before issuing any GL call, it checks if the state has actually changed:

```typescript
// Without cache: 2000 × gl.useProgram() calls = 2000 driver roundtrips
// With cache: ~10 actual gl.useProgram() calls (only when program changes)

const setProgram = (program: WebGLProgram) => {
  if (this._cache.program === program) return
  this._gl.useProgram(program)
  this._cache.program = program
}
```

States cached:
- Current program
- Current VAO
- Current framebuffer
- Viewport
- Depth write / compare function
- Cull face mode
- Blend enabled / factors
- Bound textures per unit
- Bound UBOs per binding point

This eliminates 40-60% of all GL calls in a typical frame.

## Strategy 3: Uniform Buffer Objects (UBOs)

Three.js sets uniforms individually:
```javascript
// Three.js: 5-15 gl.uniform*() calls PER object
gl.uniformMatrix4fv(modelLoc, false, modelMatrix)
gl.uniform3fv(colorLoc, color)
gl.uniform1f(opacityLoc, opacity)
// ... per material, per object
```

Rabbit uses UBOs (WebGL2) / bind groups (WebGPU):
```typescript
// Write all per-object uniforms in one call
uniformBuffer.write(objectUniformData, dynamicOffset)
gl.bindBufferRange(gl.UNIFORM_BUFFER, 3, uniformBuffer, dynamicOffset, size)
// One call instead of 5-15
```

### Ring Buffer for Dynamic Uniforms

A large uniform buffer (e.g., 4MB) is used as a ring buffer. Each frame, per-object uniforms are written sequentially into the buffer with dynamic offsets:

```
Frame N:
┌──────────────────────────────────────────────────────────┐
│ obj0 uniforms │ obj1 uniforms │ ... │ obj1999 uniforms │  │
└──────────────────────────────────────────────────────────┘
                                                     ▲ write offset

Frame N+1 (wraps around):
┌──────────────────────────────────────────────────────────┐
│ obj0 uniforms │ obj1 uniforms │ ... │ obj1999 uniforms │  │
└──────────────────────────────────────────────────────────┘
```

With double/triple buffering to avoid GPU/CPU contention.

## Strategy 4: Zero-Allocation Render Loop

No `new` in the hot path. All temporary math objects are pre-allocated:

```typescript
// Module-level temporaries (reused every frame)
const _tempVec3 = new Vec3()
const _tempMat4 = new Mat4()
const _tempQuat = new Quat()
const _tempAABB = new AABB()

// Render list: pre-allocated array, cleared and refilled each frame
const opaqueList: RenderItem[] = new Array(4096)
let opaqueCount = 0

const collectRenderables = (node: Node) => {
  // Reuse existing array slots instead of push()
  opaqueList[opaqueCount++] = renderItemPool.acquire()
  // ...
}

// After frame: release all items back to pool
const endFrame = () => {
  for (let i = 0; i < opaqueCount; i++) {
    renderItemPool.release(opaqueList[i])
  }
  opaqueCount = 0
}
```

### Object Pooling

For frequently created/destroyed objects (RenderItem, intersection results), we use simple pools:

```typescript
class Pool<T> {
  private _items: T[] = []
  private _create: () => T

  constructor(create: () => T, prealloc: number = 0) {
    this._create = create
    for (let i = 0; i < prealloc; i++) {
      this._items.push(create())
    }
  }

  acquire(): T {
    return this._items.pop() ?? this._create()
  }

  release(item: T): void {
    this._items.push(item)
  }
}
```

## Strategy 5: Frustum Culling

In a scene with 2000 objects where only 800 are visible, frustum culling eliminates 1200 draw calls — a 60% reduction.

The AABB-frustum test costs ~0.1μs per object (6 dot products with early exit). Testing 2000 objects: ~0.2ms. This is negligible compared to the draw call savings.

## Strategy 6: WebGPU Render Bundles

On WebGPU, static scenes (objects that don't move or change material) can use render bundles:

```typescript
// Build once:
const bundle = device.createRenderBundle(...)
// Record draw calls into bundle
// ...

// Each frame: single call replaces 2000 individual draw calls
passEncoder.executeBundles([bundle])
```

Render bundles pre-validate all draw calls at creation time, so the per-frame cost approaches zero for static objects.

For mixed static/dynamic scenes:
```
Static objects → render bundle (very fast)
Dynamic objects → recorded fresh each frame
```

## Strategy 7: Dirty Flags for Matrix Updates

Only recompute world matrices for nodes whose transforms changed:

```
Static scene (nothing moved): 0 matrix recomputations
One character moved: ~30 recomputations (character + bones)
Camera moved: 0 recomputations (camera matrix is separate)
```

This avoids the three.js pattern of recomputing every world matrix every frame.

## Strategy 8: Efficient Shadow Maps

Shadow passes are expensive. Optimizations:

1. **Single atlas texture**: No texture switches between cascades
2. **Depth-only shader**: Minimal vertex shader (just model × lightVP × position), no fragment work
3. **Front-face culling**: Render back faces into shadow map to reduce shadow acne (eliminates need for large bias)
4. **Stable cascades**: Snap to texel grid so shadow map content doesn't change when camera rotates (avoids re-rendering static shadow casters)

## Strategy 9: Shader Precompilation

All shader variants are compiled during loading, not on first use. The `ShaderCache` accepts a list of material configurations and pre-compiles all variants:

```typescript
const precompileShaders = (device: HalDevice, materials: Material[]) => {
  for (const material of materials) {
    const features = material.getShaderFeatures()
    const key = computeVariantKey(features)
    if (!shaderCache.has(key)) {
      const source = generateShader(material, features)
      const shader = device.createShader({ source })
      shaderCache.set(key, shader)
    }
  }
}
```

This prevents frame drops from shader compilation during gameplay.

## Memory Budget

For a 200×200m low-poly game world on mobile:

| Resource | Estimate |
|----------|----------|
| Geometry buffers (2000 meshes, ~5K tris avg) | ~200MB vertex + 40MB index |
| Textures (compressed ASTC/ETC2) | ~50-100MB |
| Shadow atlas (4096×4096 depth) | ~32MB |
| MSAA render targets (1080p × 4x) | ~32MB |
| Bloom mip chain | ~16MB |
| OIT buffers | ~16MB |
| Uniform buffers | ~4MB |
| BVH structures | ~20MB (built lazily) |
| **Total GPU memory** | **~370-460MB** |

This fits comfortably within the ~2-4GB GPU memory of recent phones.

Note: 2000 meshes at ~5K triangles average would be 10M triangles total. For low-poly games, this is more likely 500-2K triangles per mesh, which would be ~200MB less.

## Profiling Hooks

Rabbit provides optional frame timing instrumentation:

```typescript
const stats = engine.getFrameStats()
// {
//   fps: 60,
//   frameTime: 16.2,        // total frame time in ms
//   jsTime: 3.1,            // JS-side work
//   gpuTime: 8.4,           // GPU-side work (if timer queries available)
//   drawCalls: 847,         // actual draw calls (after culling)
//   triangles: 423000,      // total triangles rendered
//   stateChanges: 12,       // pipeline switches
//   culled: 1153,           // objects frustum-culled
// }
```

GPU timing uses `EXT_disjoint_timer_query_webgl2` (WebGL2) or `GPUCommandEncoder.writeTimestamp` (WebGPU) when available.
