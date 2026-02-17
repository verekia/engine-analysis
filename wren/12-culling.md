# 12 — Frustum Culling, MSAA & Performance Strategies

## Frustum Culling

### Overview

Before each frame, every renderable's world-space AABB is tested against the camera's view frustum. Objects fully outside the frustum are skipped entirely — no draw command is generated for them.

### Frustum Extraction

The six frustum planes are extracted directly from the view-projection matrix (Gribb/Hartmann method):

```ts
interface Frustum {
  planes: Float32Array  // 6 planes × 4 components (nx, ny, nz, d) = 24 floats
}

const extractFrustum = (viewProjection: Float32Array, frustum: Frustum): void => {
  const m = viewProjection
  const p = frustum.planes

  // Left:   row3 + row0
  p[0]  = m[3]  + m[0];  p[1]  = m[7]  + m[4];  p[2]  = m[11] + m[8];   p[3]  = m[15] + m[12]
  // Right:  row3 - row0
  p[4]  = m[3]  - m[0];  p[5]  = m[7]  - m[4];  p[6]  = m[11] - m[8];   p[7]  = m[15] - m[12]
  // Bottom: row3 + row1
  p[8]  = m[3]  + m[1];  p[9]  = m[7]  + m[5];  p[10] = m[11] + m[9];   p[11] = m[15] + m[13]
  // Top:    row3 - row1
  p[12] = m[3]  - m[1];  p[13] = m[7]  - m[5];  p[14] = m[11] - m[9];   p[15] = m[15] - m[13]
  // Near:   row3 + row2
  p[16] = m[3]  + m[2];  p[17] = m[7]  + m[6];  p[18] = m[11] + m[10];  p[19] = m[15] + m[14]
  // Far:    row3 - row2
  p[20] = m[3]  - m[2];  p[21] = m[7]  - m[6];  p[22] = m[11] - m[10];  p[23] = m[15] - m[14]

  // Normalize each plane
  for (let i = 0; i < 6; i++) {
    const o = i * 4
    const len = Math.sqrt(p[o] * p[o] + p[o+1] * p[o+1] + p[o+2] * p[o+2])
    p[o] /= len; p[o+1] /= len; p[o+2] /= len; p[o+3] /= len
  }
}
```

### AABB-Frustum Test

Each AABB is tested against all 6 planes. If the AABB is fully outside any plane, it's culled. We use the "p-vertex/n-vertex" optimization for fast rejection:

```ts
const isAABBInFrustum = (frustum: Frustum, aabb: AABB): boolean => {
  const p = frustum.planes
  const min = aabb.min  // [x, y, z]
  const max = aabb.max  // [x, y, z]

  for (let i = 0; i < 6; i++) {
    const o = i * 4
    const nx = p[o], ny = p[o+1], nz = p[o+2], d = p[o+3]

    // P-vertex: the corner of the AABB most in the direction of the plane normal
    const px = nx >= 0 ? max[0] : min[0]
    const py = ny >= 0 ? max[1] : min[1]
    const pz = nz >= 0 ? max[2] : min[2]

    // If the p-vertex is behind the plane, the entire AABB is outside
    if (nx * px + ny * py + nz * pz + d < 0) {
      return false
    }
  }

  return true
}
```

### Culling Loop

```ts
const cullScene = (renderables: RenderableList, frustum: Frustum): number => {
  let visibleCount = 0

  for (let i = 0; i < renderables.count; i++) {
    const node = renderables.meshes[i]

    if (!node.visible) {
      node.culled = true
      continue
    }

    if (node.worldAABB && !isAABBInFrustum(frustum, node.worldAABB)) {
      node.culled = true
      continue
    }

    node.culled = false
    visibleCount++
  }

  return visibleCount
}
```

For 2000 objects, this takes < 0.1ms. The test is 6 dot products per AABB with early exit.

### WebGPU Compute Culling (Optional Enhancement)

When WebGPU is available, frustum culling can run as a compute shader:

```wgsl
@group(0) @binding(0) var<storage, read> objectAABBs: array<AABB>;
@group(0) @binding(1) var<storage, read_write> visibilityFlags: array<u32>;
@group(0) @binding(2) var<uniform> frustumPlanes: array<vec4f, 6>;

@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) id: vec3u) {
  let idx = id.x;
  if (idx >= arrayLength(&objectAABBs)) { return; }

  let aabb = objectAABBs[idx];
  var visible = 1u;

  for (var p = 0u; p < 6u; p++) {
    let plane = frustumPlanes[p];
    let pVertex = vec3f(
      select(aabb.min.x, aabb.max.x, plane.x >= 0.0),
      select(aabb.min.y, aabb.max.y, plane.y >= 0.0),
      select(aabb.min.z, aabb.max.z, plane.z >= 0.0)
    );
    if (dot(plane.xyz, pVertex) + plane.w < 0.0) {
      visible = 0u;
      break;
    }
  }

  visibilityFlags[idx] = visible;
}
```

This moves culling entirely off the main thread. The visibility buffer is read back or used directly for indirect draw calls.

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

## Performance Strategies

### Draw Call Overhead Minimization

The primary performance target is 2000 draw calls at 60fps on mobile. Here's how each system contributes:

| Strategy | Savings |
|----------|---------|
| **Sort-based renderer** | Eliminates 80-95% of state changes |
| **State cache** | Skips redundant GL/WebGPU calls |
| **UBO for uniforms** | One buffer bind per material, not N uniform calls |
| **Render bundles (WebGPU)** | Reduces 2000 JS→GPU calls to 1 |
| **Frustum culling** | Skips invisible objects entirely |
| **Single-mesh models** | Material index system avoids per-material draw calls |

### Memory & GC Avoidance

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

### Batch-Friendly Design

The material index system is a key performance feature. Instead of splitting a model into N meshes (one per material), the entire model is a single mesh with per-vertex material indices. This means:
- **1 draw call** instead of N (for a model with N material regions)
- **1 geometry buffer** instead of N
- **1 BVH** instead of N

For a scene with 100 buildings, each with 5 material regions, this reduces draw calls from 500 to 100.

### Shader Compilation Strategy

- Compile shader variants lazily on first use
- Cache compiled programs by variant key
- On WebGPU, use `createRenderPipelineAsync` to avoid janks
- Pre-warm common variants during loading screen

### Frame Budget Breakdown (Target: 16.6ms on mobile)

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

### Profiling Hooks

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
