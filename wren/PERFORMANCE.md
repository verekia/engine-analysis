# Performance Optimization Strategies

## Performance Target

**2000 draw calls at 60fps on mobile** with a total frame budget of 16.6ms.

## Draw Call Overhead Minimization

The primary performance target is 2000 draw calls at 60fps on mobile. Here's how each system contributes:

| Strategy | Savings |
|----------|---------|
| **Sort-based renderer** | Eliminates 80-95% of state changes |
| **State cache** | Skips redundant GL/WebGPU calls |
| **UBO for uniforms** | One buffer bind per material, not N uniform calls |
| **Render bundles (WebGPU)** | Reduces 2000 JS→GPU calls to 1 |
| **Frustum culling** | Skips invisible objects entirely |
| **Single-mesh models** | Material index system avoids per-material draw calls |

## Memory & GC Avoidance

```ts
// BAD: allocates every frame
const update = () => {
  const position = new Float32Array([x, y, z])  // GC pressure!
  setPosition(node, position)
}

// GOOD: mutate in place
const update = () => {
  node.position[0] = x
  node.position[1] = y
  node.position[2] = z
  markDirty(node)
}
```

Rules:
- Pre-allocate all typed arrays at initialization
- Reuse temporary math vectors (module-level `const _tempVec3 = new Float32Array(3)`)
- Draw command array is pre-allocated and reused each frame
- No object spread (`{ ...obj }`) in hot paths
- No `Array.map/filter/reduce` in hot paths — use `for` loops

## Batch-Friendly Design

The material index system is a key performance feature. Instead of splitting a model into N meshes (one per material), the entire model is a single mesh with per-vertex material indices. This means:
- **1 draw call** instead of N (for a model with N material regions)
- **1 geometry buffer** instead of N
- **1 BVH** instead of N

For a scene with 100 buildings, each with 5 material regions, this reduces draw calls from 500 to 100.

## Shader Compilation Strategy

- Compile shader variants lazily on first use
- Cache compiled programs by variant key
- On WebGPU, use `createRenderPipelineAsync` to avoid janks
- Pre-warm common variants during loading screen

## Frame Budget Breakdown (Target: 16.6ms on mobile)

```
Phase                    Budget
─────────────────────────────────
Scene graph update       0.3ms
Frustum culling          0.1ms
Sort draw calls          0.1ms
Shadow rendering (GPU)   1.5ms
Opaque pass (GPU)        8.0ms   ← Most budget here
Transparency (GPU)       1.0ms
Bloom (GPU)              0.5ms
MSAA resolve (GPU)       0.3ms
Draw submission (CPU)    2.0ms
React reconciler         0.5ms
Headroom                 2.3ms
─────────────────────────────────
Total                   16.6ms
```

## MSAA (Multi-Sample Anti-Aliasing)

### Implementation

MSAA is the simplest and most efficient anti-aliasing for a forward renderer. It's handled at the framebuffer level — no shader changes required.

**WebGL2:**
```ts
// Create MSAA render target
const msaaRB = gl.createRenderbuffer()
gl.bindRenderbuffer(GL.RENDERBUFFER, msaaRB)
gl.renderbufferStorageMultisample(GL.RENDERBUFFER, 4, GL.RGBA8, width, height)

// Create MSAA depth buffer
const msaaDepth = gl.createRenderbuffer()
gl.bindRenderbuffer(GL.RENDERBUFFER, msaaDepth)
gl.renderbufferStorageMultisample(GL.RENDERBUFFER, 4, GL.DEPTH24_STENCIL8, width, height)

// Attach to framebuffer
gl.framebufferRenderbuffer(GL.FRAMEBUFFER, GL.COLOR_ATTACHMENT0, GL.RENDERBUFFER, msaaRB)
gl.framebufferRenderbuffer(GL.FRAMEBUFFER, GL.DEPTH_STENCIL_ATTACHMENT, GL.RENDERBUFFER, msaaDepth)

// Resolve: blit MSAA FBO to single-sample FBO
gl.blitFramebuffer(0, 0, w, h, 0, 0, w, h, GL.COLOR_BUFFER_BIT, GL.LINEAR)
```

**WebGPU:**
```ts
const msaaTexture = device.createTexture({
  size: [width, height],
  format: 'bgra8unorm',
  sampleCount: 4,
  usage: GPUTextureUsage.RENDER_ATTACHMENT,
})

// In render pass descriptor:
const renderPass = encoder.beginRenderPass({
  colorAttachments: [{
    view: msaaTexture.createView(),
    resolveTarget: swapChainTexture.createView(),  // Auto-resolve
    loadOp: 'clear',
    storeOp: 'discard',  // MSAA texture is discarded after resolve
  }],
})
```

### Sample Count

4x MSAA is the default. It provides good quality at modest cost:

| Sample Count | Quality | Cost (relative) |
|-------------|---------|-----------------|
| 1x (none) | Aliased | 1.0x |
| 4x | Good | ~1.3x |
| 8x | Great | ~1.6x |

On mobile, 4x MSAA is essentially free on tile-based GPUs (Adreno, Mali, Apple) because multisampling happens on-chip during tile resolve.

### MSAA + WBOIT

When both MSAA and WBOIT are active, the accumulation and revealage textures must also be MSAA. The MSAA resolve happens after the WBOIT composite pass, before post-processing.

### MSAA + Bloom

The emissive render target can be at half resolution without MSAA (bloom is blurry by nature). The scene color target uses MSAA normally.

## Profiling Hooks

The renderer exposes timing data for profiling:

```ts
interface FrameStats {
  drawCalls: number
  triangles: number
  stateChanges: number
  cullRejected: number
  cullPassed: number
  sortTimeMs: number
  submitTimeMs: number
  gpuTimeMs: number        // Via EXT_disjoint_timer_query (WebGL2) or timestamp queries (WebGPU)
}

renderer.onFrameStats = (stats: FrameStats) => {
  console.log(`Draws: ${stats.drawCalls}, State changes: ${stats.stateChanges}`)
}
```
