# Renderer — Render Pipeline, Draw Call Optimization, Command Sorting

## Render Pipeline Overview

The Lynx renderer consumes a `Scene` and a `Camera` and produces a final image. It uses a multi-pass architecture where each pass writes to one or more render targets. The passes execute in a fixed order every frame:

```
Pass 0  Shadow Map Pass
        Render scene depth from the light's perspective into 3 CSM cascade maps.

Pass 1  Opaque Pass
        Render all opaque geometry front-to-back into a 4x MSAA color+depth target.
        Optional depth pre-pass can be enabled for scenes with high overdraw.

Pass 2  Transparent Pass (OIT Accumulation)
        Render all transparent geometry into two float targets using
        Weighted Blended Order-Independent Transparency (McGuire & Bavoil 2013).
        Target A: RGBA16F accumulation (premultiplied-alpha weighted color).
        Target B: R8 revealage.

Pass 3  OIT Composite
        Fullscreen quad that blends the OIT accumulation/revealage result
        onto the opaque color target.

Pass 4  Post-Processing
        Bloom: extract bright pixels -> progressive downscale chain ->
               progressive upscale chain with additive blend.
        Tone mapping: applied as the final post-process step.

Pass 5  MSAA Resolve + Final Output
        Resolve the 4x MSAA target to the canvas backbuffer (or to an
        offscreen target for further compositing).
```

### Pass Execution Diagram

```
                          ┌──────────────┐
                          │  Scene Graph  │
                          │  + Camera     │
                          └──────┬───────┘
                                 │
                          ┌──────▼───────┐
                          │Frustum Cull  │
                          │AABB vs 6-plane│
                          └──────┬───────┘
                                 │
                          ┌──────▼───────┐
                          │Command Gen   │
                          │+ Sort (radix)│
                          └──────┬───────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
   ┌──────▼───────┐      ┌──────▼───────┐       ┌──────▼──────┐
   │Pass 0        │      │Pass 1        │       │Pass 2       │
   │Shadow Maps   │      │Opaque (MSAA) │       │OIT Accum    │
   │2048² × 3 CSM │      │4× MSAA color │       │RGBA16F +    │
   │              │      │+ depth       │       │R8 revealage │
   └──────┬───────┘      └──────┬───────┘       └──────┬──────┘
          │                      │                      │
          │               ┌──────▼──────────────────────▼──────┐
          │               │Pass 3: OIT Composite               │
          │               │Blend transparency onto opaque      │
          └───────────────►                                    │
                          └──────────────┬─────────────────────┘
                                         │
                          ┌──────────────▼─────────────────────┐
                          │Pass 4: Post-Processing              │
                          │Bloom extract → down chain →         │
                          │up chain → tone map                  │
                          └──────────────┬─────────────────────┘
                                         │
                          ┌──────────────▼─────────────────────┐
                          │Pass 5: MSAA Resolve + Final Output  │
                          │Resolve to canvas / output target    │
                          └────────────────────────────────────┘
```

---

## Command Sorting Strategy

### RenderCommand Structure

Every visible mesh in the frustum generates a single `RenderCommand`. These commands form a flat array that is sorted and then iterated to produce GPU draw calls.

```typescript
type RenderCommand = {
  sortKey: bigint        // u64 — encodes pipeline, material, depth
  mesh: MeshNode         // reference to the scene graph mesh node
  material: MaterialID   // numeric material identifier
  geometry: GeometryID   // numeric geometry identifier (maps to VAO / buffer set)
  worldMatrix: Float32Array  // 4×4 column-major, Z-up right-handed
  boneMatrices: Float32Array | null  // null for non-skinned meshes
  materialIndex: number  // index into material UBO palette
}
```

### Sort Key Encoding (64-bit)

The sort key packs three fields into a single 64-bit integer so that a single radix sort produces the correct draw order while minimizing GPU state changes.

```
Bit layout (MSB → LSB):

  63       60 59            48 47                                  0
  ┌─────────┬────────────────┬─────────────────────────────────────┐
  │Pipeline │  Material ID   │              Depth                  │
  │ 4 bits  │   12 bits      │             48 bits                 │
  └─────────┴────────────────┴─────────────────────────────────────┘
```

| Field | Bits | Range | Purpose |
|---|---|---|---|
| Pipeline ID | 63-60 | 0-15 (16 pipelines) | Groups draws that share the same GPU pipeline state (shader + blend + depth + rasterizer). Changing pipeline is the most expensive state switch. |
| Material ID | 59-48 | 0-4095 (4096 materials) | Groups draws that share the same material bind group (textures, uniforms). Second most expensive switch. |
| Depth | 47-0 | 48-bit quantized | For opaque: front-to-back (smaller depth = drawn first) to maximize early-Z rejection. For transparent: back-to-front (larger depth = drawn first). |

### Key Construction

```typescript
const buildSortKeyOpaque = (
  pipelineId: number,
  materialId: number,
  viewSpaceDepth: number,
  nearPlane: number,
  farPlane: number
): bigint => {
  const pBits = BigInt(pipelineId & 0xF) << 60n
  const mBits = BigInt(materialId & 0xFFF) << 48n

  // Normalize depth to [0, 1] and quantize to 48 bits
  // Front-to-back: smaller depth gets smaller integer
  const t = (viewSpaceDepth - nearPlane) / (farPlane - nearPlane)
  const clamped = Math.max(0, Math.min(1, t))
  const dBits = BigInt(Math.floor(clamped * 0xFFFFFFFFFFFF)) & 0xFFFFFFFFFFFFn

  return pBits | mBits | dBits
}

const buildSortKeyTransparent = (
  pipelineId: number,
  materialId: number,
  viewSpaceDepth: number,
  nearPlane: number,
  farPlane: number
): bigint => {
  const pBits = BigInt(pipelineId & 0xF) << 60n
  const mBits = BigInt(materialId & 0xFFF) << 48n

  // Back-to-front: larger depth gets smaller integer (invert)
  const t = (viewSpaceDepth - nearPlane) / (farPlane - nearPlane)
  const clamped = Math.max(0, Math.min(1, t))
  const inverted = 1.0 - clamped
  const dBits = BigInt(Math.floor(inverted * 0xFFFFFFFFFFFF)) & 0xFFFFFFFFFFFFn

  return pBits | mBits | dBits
}
```

### Radix Sort

Sorting is performed with a least-significant-digit radix sort operating on the 64-bit keys. Radix sort is O(n) in command count and avoids the comparison overhead of `Array.prototype.sort`.

```typescript
// 8 passes of 8-bit radix sort over the 64-bit key
// Uses a pre-allocated auxiliary buffer to avoid per-frame allocation
const radixSort64 = (commands: RenderCommand[], aux: RenderCommand[], count: number): void => {
  const RADIX_BITS = 8
  const RADIX_SIZE = 1 << RADIX_BITS   // 256
  const PASSES = 8                      // 64 / 8

  const counts = new Uint32Array(RADIX_SIZE)

  let src = commands
  let dst = aux

  for (let pass = 0; pass < PASSES; pass++) {
    const shift = BigInt(pass * RADIX_BITS)

    // Count occurrences
    counts.fill(0)
    for (let i = 0; i < count; i++) {
      const bucket = Number((src[i].sortKey >> shift) & 0xFFn)
      counts[bucket]++
    }

    // Prefix sum
    let total = 0
    for (let i = 0; i < RADIX_SIZE; i++) {
      const c = counts[i]
      counts[i] = total
      total += c
    }

    // Scatter
    for (let i = 0; i < count; i++) {
      const bucket = Number((src[i].sortKey >> shift) & 0xFFn)
      dst[counts[bucket]++] = src[i]
    }

    // Swap buffers
    const tmp = src
    src = dst
    dst = tmp
  }

  // If final result ended up in aux, copy back
  if (src !== commands) {
    for (let i = 0; i < count; i++) {
      commands[i] = src[i]
    }
  }
}
```

### Command Caching

The command list is **not** rebuilt every frame. It is cached and only invalidated when the scene graph changes:

- A node is added or removed
- A material assignment changes on a mesh
- A mesh's geometry reference changes
- The camera frustum changes substantially (new objects enter or leave)

When invalidated, the command list is regenerated from the visible set and re-sorted. During steady-state rendering of a static scene with a static camera, the command list is reused as-is, and the only per-frame work is updating dynamic UBO data (transforms for animated objects).

---

## Draw Call Execution

### State Tracking

Iterating sorted commands in order naturally groups draws by pipeline and material. The renderer tracks the currently bound pipeline, material, and geometry to skip redundant GPU calls.

```typescript
const executeCommands = (
  commands: RenderCommand[],
  count: number,
  device: GALDevice,
  passEncoder: GALRenderPassEncoder,
  frameUBO: GALBuffer,
  dynamicUBO: DynamicUBO
): void => {
  let currentPipeline: PipelineID = -1
  let currentMaterial: MaterialID = -1
  let currentGeometry: GeometryID = -1

  for (let i = 0; i < count; i++) {
    const cmd = commands[i]

    // Bind pipeline only when it changes
    const cmdPipeline = Number((cmd.sortKey >> 60n) & 0xFn)
    if (cmdPipeline !== currentPipeline) {
      passEncoder.setPipeline(device.getPipeline(cmdPipeline))
      currentPipeline = cmdPipeline
    }

    // Bind material only when it changes
    if (cmd.material !== currentMaterial) {
      passEncoder.setBindGroup(1, device.getMaterialBindGroup(cmd.material))
      currentMaterial = cmd.material
    }

    // Bind geometry only when it changes
    if (cmd.geometry !== currentGeometry) {
      passEncoder.setVertexBuffer(0, device.getVertexBuffer(cmd.geometry))
      passEncoder.setIndexBuffer(device.getIndexBuffer(cmd.geometry))
      currentGeometry = cmd.geometry
    }

    // Write per-object data to dynamic UBO
    const offset = dynamicUBO.write(cmd.worldMatrix, cmd.materialIndex, cmd.boneMatrices)
    passEncoder.setBindGroup(2, dynamicUBO.bindGroup, [offset])

    // Issue draw call
    const geo = device.getGeometryInfo(cmd.geometry)
    passEncoder.drawIndexed(geo.indexCount, 1, geo.indexOffset, 0, 0)
  }
}
```

### Dynamic Uniform Buffer

Per-object data is written to a large pre-allocated UBO each frame. Each write advances an internal offset. Bind groups reference the same buffer but with different dynamic offsets, so no new bind groups are allocated per frame.

```typescript
type DynamicUBO = {
  buffer: GALBuffer           // GPU buffer, large enough for max objects per frame
  cpuBuffer: ArrayBuffer      // CPU-side staging area
  floatView: Float32Array     // Float view into cpuBuffer
  offset: number              // Current write offset (bytes), 256-byte aligned
  bindGroup: GALBindGroup     // Reused bind group with dynamic offset
}

const DYNAMIC_UBO_ALIGNMENT = 256  // WebGPU requires 256-byte alignment

const dynamicUBOWrite = (
  ubo: DynamicUBO,
  worldMatrix: Float32Array,
  materialIndex: number,
  boneMatrices: Float32Array | null
): number => {
  const baseFloat = ubo.offset / 4

  // World matrix: 16 floats (64 bytes)
  ubo.floatView.set(worldMatrix, baseFloat)

  // Material palette index: 1 float (4 bytes), padded to 16 bytes
  ubo.floatView[baseFloat + 16] = materialIndex

  // Bone matrices: up to 128 × 16 floats = 8192 bytes
  if (boneMatrices) {
    ubo.floatView.set(boneMatrices, baseFloat + 20)  // offset past world + padding
  }

  const currentOffset = ubo.offset

  // Advance offset, aligned to 256 bytes
  const dataSize = boneMatrices ? 64 + 16 + boneMatrices.byteLength : 64 + 16
  ubo.offset += Math.ceil(dataSize / DYNAMIC_UBO_ALIGNMENT) * DYNAMIC_UBO_ALIGNMENT

  return currentOffset
}
```

### WebGPU: Render Bundles

For static objects (non-animated meshes whose transforms have not changed), WebGPU render bundles are pre-recorded and replayed each frame. This avoids re-encoding draw commands for the majority of a typical scene.

```typescript
const buildStaticRenderBundle = (
  device: GALDevice,
  staticCommands: RenderCommand[],
  count: number,
  renderBundleDescriptor: GPURenderBundleEncoderDescriptor,
  dynamicUBO: DynamicUBO
): GPURenderBundle => {
  const encoder = device.nativeDevice.createRenderBundleEncoder(renderBundleDescriptor)

  // Record commands into the bundle — same logic as executeCommands
  let currentPipeline: PipelineID = -1
  let currentMaterial: MaterialID = -1
  let currentGeometry: GeometryID = -1

  for (let i = 0; i < count; i++) {
    const cmd = staticCommands[i]

    const cmdPipeline = Number((cmd.sortKey >> 60n) & 0xFn)
    if (cmdPipeline !== currentPipeline) {
      encoder.setPipeline(device.getPipeline(cmdPipeline))
      currentPipeline = cmdPipeline
    }

    if (cmd.material !== currentMaterial) {
      encoder.setBindGroup(1, device.getMaterialBindGroup(cmd.material))
      currentMaterial = cmd.material
    }

    if (cmd.geometry !== currentGeometry) {
      encoder.setVertexBuffer(0, device.getVertexBuffer(cmd.geometry))
      encoder.setIndexBuffer(device.getIndexBuffer(cmd.geometry), 'uint16')
      currentGeometry = cmd.geometry
    }

    const offset = dynamicUBO.write(cmd.worldMatrix, cmd.materialIndex, cmd.boneMatrices)
    encoder.setBindGroup(2, dynamicUBO.bindGroup, [offset])

    const geo = device.getGeometryInfo(cmd.geometry)
    encoder.drawIndexed(geo.indexCount, 1, geo.indexOffset, 0, 0)
  }

  return encoder.finish()
}
```

When the scene graph marks a static bundle as dirty (node added/removed, material changed), the bundle is discarded and rebuilt on the next frame. Animated objects always use the dynamic encoder path.

### WebGL2: VAO and State Minimization

On WebGL2, the same sorted-command-list approach applies. Each geometry is backed by a Vertex Array Object (VAO). The WebGL2 backend tracks bound state and skips redundant `gl.bindVertexArray`, `gl.useProgram`, and `gl.bindTexture` calls.

```typescript
const executeCommandsWebGL2 = (
  commands: RenderCommand[],
  count: number,
  gl: WebGL2RenderingContext,
  state: WebGL2StateTracker,
  dynamicUBO: WebGL2DynamicUBO
): void => {
  for (let i = 0; i < count; i++) {
    const cmd = commands[i]

    const cmdPipeline = Number((cmd.sortKey >> 60n) & 0xFn)
    if (cmdPipeline !== state.currentPipeline) {
      const pipeline = state.pipelines[cmdPipeline]
      gl.useProgram(pipeline.program)
      state.applyDepthStencil(pipeline.depthStencil)
      state.applyBlend(pipeline.blend)
      state.applyRasterizer(pipeline.rasterizer)
      state.currentPipeline = cmdPipeline
    }

    if (cmd.material !== state.currentMaterial) {
      state.bindMaterialTextures(gl, cmd.material)
      state.currentMaterial = cmd.material
    }

    if (cmd.geometry !== state.currentGeometry) {
      gl.bindVertexArray(state.vaos[cmd.geometry])
      state.currentGeometry = cmd.geometry
    }

    // Write per-object data via gl.bufferSubData to dynamic UBO
    const offset = dynamicUBO.write(gl, cmd.worldMatrix, cmd.materialIndex, cmd.boneMatrices)
    gl.bindBufferRange(
      gl.UNIFORM_BUFFER,
      2,  // binding point 2 = per-object
      dynamicUBO.buffer,
      offset,
      dynamicUBO.stride
    )

    const geo = state.geometryInfo[cmd.geometry]
    gl.drawElements(gl.TRIANGLES, geo.indexCount, gl.UNSIGNED_SHORT, geo.indexOffset * 2)
  }
}
```

---

## MSAA Strategy

### Render Target Layout for MSAA

Opaque geometry renders into a 4x MSAA render target. The transparency pass renders into separate non-MSAA float targets because Weighted Blended OIT requires additive blending into float textures, which does not benefit from multisampling.

```
Opaque Pass (4x MSAA):
  Color attachment: RGBA8 multisample (count=4)
  Depth attachment: Depth24Plus multisample (count=4)

Transparency Pass (no MSAA):
  Color attachment 0: RGBA16F (OIT accumulation)
  Color attachment 1: R8 (OIT revealage)
  Depth attachment: read-only copy of opaque depth (for depth testing without writing)

OIT Composite:
  Writes onto the MSAA color target (blends transparency result)

Post-Processing:
  Operates on the MSAA target

MSAA Resolve:
  Resolves 4x MSAA → 1x final output
```

### Why Resolve Happens Last

Resolving MSAA before compositing transparency would lose subpixel edge information on opaque geometry behind transparent surfaces. By keeping the MSAA target intact through OIT compositing and post-processing, the final resolve produces clean antialiased edges everywhere.

### WebGPU MSAA

```typescript
// Pipeline creation with MSAA
const createOpaquePipeline = (device: GPUDevice, format: GPUTextureFormat): GPURenderPipeline => {
  return device.createRenderPipeline({
    // ...shader, layout, vertex state...
    multisample: {
      count: 4,
    },
    fragment: {
      // ...
      targets: [{ format }],
    },
    depthStencil: {
      format: 'depth24plus',
      depthWriteEnabled: true,
      depthCompare: 'less',
    },
  })
}

// Render pass with MSAA color and resolve target
const beginOpaquePass = (
  encoder: GPUCommandEncoder,
  msaaView: GPUTextureView,
  resolveView: GPUTextureView,
  depthView: GPUTextureView
): GPURenderPassEncoder => {
  return encoder.beginRenderPass({
    colorAttachments: [{
      view: msaaView,           // 4x MSAA texture view
      resolveTarget: undefined, // resolve happens in Pass 5, not here
      loadOp: 'clear',
      storeOp: 'store',
      clearValue: { r: 0.0, g: 0.0, b: 0.0, a: 1.0 },
    }],
    depthStencilAttachment: {
      view: depthView,          // 4x MSAA depth view
      depthLoadOp: 'clear',
      depthStoreOp: 'store',
      depthClearValue: 1.0,
    },
  })
}
```

### WebGL2 MSAA

```typescript
const setupMSAAFramebuffer = (gl: WebGL2RenderingContext, width: number, height: number) => {
  const msaaFBO = gl.createFramebuffer()
  gl.bindFramebuffer(gl.FRAMEBUFFER, msaaFBO)

  // Color renderbuffer — 4x MSAA
  const colorRB = gl.createRenderbuffer()
  gl.bindRenderbuffer(gl.RENDERBUFFER, colorRB)
  gl.renderbufferStorageMultisample(gl.RENDERBUFFER, 4, gl.RGBA8, width, height)
  gl.framebufferRenderbuffer(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.RENDERBUFFER, colorRB)

  // Depth renderbuffer — 4x MSAA
  const depthRB = gl.createRenderbuffer()
  gl.bindRenderbuffer(gl.RENDERBUFFER, depthRB)
  gl.renderbufferStorageMultisample(gl.RENDERBUFFER, 4, gl.DEPTH_COMPONENT24, width, height)
  gl.framebufferRenderbuffer(gl.FRAMEBUFFER, gl.DEPTH_ATTACHMENT, gl.RENDERBUFFER, depthRB)

  return { msaaFBO, colorRB, depthRB }
}

// Resolve by blitting to a non-MSAA framebuffer
const resolveMSAA = (
  gl: WebGL2RenderingContext,
  msaaFBO: WebGLFramebuffer,
  resolveFBO: WebGLFramebuffer,
  width: number,
  height: number
): void => {
  gl.bindFramebuffer(gl.READ_FRAMEBUFFER, msaaFBO)
  gl.bindFramebuffer(gl.DRAW_FRAMEBUFFER, resolveFBO)
  gl.blitFramebuffer(
    0, 0, width, height,
    0, 0, width, height,
    gl.COLOR_BUFFER_BIT,
    gl.NEAREST
  )
}
```

---

## Frustum Culling Integration

### Pipeline Position

Frustum culling runs before command generation. Only meshes whose world-space AABBs intersect the camera frustum produce `RenderCommand` entries. This means culled objects incur zero cost in sorting and draw call execution.

```
Scene Graph Traversal
        │
        ▼
Update World Transforms (dirty nodes only)
        │
        ▼
Frustum Cull (AABB vs 6 planes)  ◄── rejects ~60-80% of objects in typical scenes
        │
        ▼
Generate RenderCommands (visible set only)
        │
        ▼
Radix Sort
        │
        ▼
Execute Draw Calls
```

### 6-Plane Frustum Test

The frustum is represented as six planes in world space (left, right, top, bottom, near, far), each stored as a `Vec4` (nx, ny, nz, d) with the normal pointing inward. A world-space AABB is tested against all six planes. If the AABB is entirely outside any single plane, the object is culled.

```typescript
type Frustum = {
  planes: Float32Array  // 6 planes × 4 components = 24 floats
}

// Returns true if the AABB is at least partially inside the frustum
const testAABBFrustum = (
  frustum: Frustum,
  minX: number, minY: number, minZ: number,
  maxX: number, maxY: number, maxZ: number
): boolean => {
  const p = frustum.planes

  for (let i = 0; i < 6; i++) {
    const offset = i * 4
    const nx = p[offset]
    const ny = p[offset + 1]
    const nz = p[offset + 2]
    const d  = p[offset + 3]

    // Find the AABB vertex most in the direction of the plane normal (p-vertex)
    const px = nx >= 0 ? maxX : minX
    const py = ny >= 0 ? maxY : minY
    const pz = nz >= 0 ? maxZ : minZ

    // If the p-vertex is outside, the entire AABB is outside this plane
    if (nx * px + ny * py + nz * pz + d < 0) {
      return false  // early-out: fully outside
    }
  }

  return true  // inside or intersecting all planes
}
```

The early-out on the first failing plane means that on average only 2-3 planes need to be tested per object (objects behind the camera fail the near or far plane immediately).

### Frustum Extraction from View-Projection Matrix

```typescript
// Extracts 6 frustum planes from a column-major 4x4 view-projection matrix
// Z-up right-handed coordinate system
const extractFrustum = (vp: Float32Array, out: Frustum): void => {
  const p = out.planes

  // Left:   row3 + row0
  p[0]  = vp[3]  + vp[0]
  p[1]  = vp[7]  + vp[4]
  p[2]  = vp[11] + vp[8]
  p[3]  = vp[15] + vp[12]
  normalizePlane(p, 0)

  // Right:  row3 - row0
  p[4]  = vp[3]  - vp[0]
  p[5]  = vp[7]  - vp[4]
  p[6]  = vp[11] - vp[8]
  p[7]  = vp[15] - vp[12]
  normalizePlane(p, 4)

  // Bottom: row3 + row1
  p[8]  = vp[3]  + vp[1]
  p[9]  = vp[7]  + vp[5]
  p[10] = vp[11] + vp[9]
  p[11] = vp[15] + vp[13]
  normalizePlane(p, 8)

  // Top:    row3 - row1
  p[12] = vp[3]  - vp[1]
  p[13] = vp[7]  - vp[5]
  p[14] = vp[11] - vp[9]
  p[15] = vp[15] - vp[13]
  normalizePlane(p, 12)

  // Near:   row3 + row2
  p[16] = vp[3]  + vp[2]
  p[17] = vp[7]  + vp[6]
  p[18] = vp[11] + vp[10]
  p[19] = vp[15] + vp[14]
  normalizePlane(p, 16)

  // Far:    row3 - row2
  p[20] = vp[3]  - vp[2]
  p[21] = vp[7]  - vp[6]
  p[22] = vp[11] - vp[10]
  p[23] = vp[15] - vp[14]
  normalizePlane(p, 20)
}

const normalizePlane = (planes: Float32Array, offset: number): void => {
  const nx = planes[offset]
  const ny = planes[offset + 1]
  const nz = planes[offset + 2]
  const len = Math.sqrt(nx * nx + ny * ny + nz * nz)
  if (len > 0) {
    const invLen = 1.0 / len
    planes[offset]     *= invLen
    planes[offset + 1] *= invLen
    planes[offset + 2] *= invLen
    planes[offset + 3] *= invLen
  }
}
```

---

## Render Targets and Framebuffer Management

### Full Set of Render Targets

| Target | Format | Size | MSAA | Purpose |
|---|---|---|---|---|
| Opaque Color | RGBA8 | viewport | 4x | Main opaque rendering target |
| Opaque Depth | Depth24Plus | viewport | 4x | Depth buffer for opaque pass |
| Shadow Cascade 0 | Depth24 | 2048 x 2048 | 1x | CSM near cascade |
| Shadow Cascade 1 | Depth24 | 2048 x 2048 | 1x | CSM mid cascade |
| Shadow Cascade 2 | Depth24 | 2048 x 2048 | 1x | CSM far cascade |
| OIT Accumulation | RGBA16F | viewport | 1x | Weighted blended color accumulation |
| OIT Revealage | R8 | viewport | 1x | Weighted blended revealage |
| Bloom 0 | RGBA16F | viewport / 2 | 1x | Bloom bright-pass extract + downscale level 0 |
| Bloom 1 | RGBA16F | viewport / 4 | 1x | Bloom downscale level 1 |
| Bloom 2 | RGBA16F | viewport / 8 | 1x | Bloom downscale level 2 |
| Bloom 3 | RGBA16F | viewport / 16 | 1x | Bloom downscale level 3 |
| Bloom 4 | RGBA16F | viewport / 32 | 1x | Bloom downscale level 4 |
| Resolve Output | RGBA8 | viewport | 1x | MSAA resolve destination / final output |

### Lazy Allocation

Render targets are allocated on demand, not at initialization time. The renderer tracks which features the scene actually uses:

- **Shadow maps**: only allocated if the scene contains a `DirectionalLight` with `castShadow: true`
- **OIT targets**: only allocated if the scene contains at least one mesh with a transparent material
- **Bloom chain**: only allocated if post-processing bloom is enabled (`renderer.bloom = true`)
- **MSAA targets**: always allocated when MSAA is enabled (default)

When the viewport resizes, all viewport-relative targets are released and re-allocated at the new resolution on the next frame.

```typescript
type RenderTargetCache = {
  opaqueColor: GALTexture | null
  opaqueDepth: GALTexture | null
  shadowCascades: GALTexture[] | null    // array of 3 depth textures
  oitAccumulation: GALTexture | null
  oitRevealage: GALTexture | null
  bloomChain: GALTexture[] | null        // array of 5 half-res textures
  resolveOutput: GALTexture | null
  viewportWidth: number
  viewportHeight: number
}

const ensureRenderTargets = (
  cache: RenderTargetCache,
  device: GALDevice,
  width: number,
  height: number,
  scene: Scene,
  config: RendererConfig
): void => {
  // Release all if viewport changed
  if (width !== cache.viewportWidth || height !== cache.viewportHeight) {
    releaseViewportTargets(cache, device)
    cache.viewportWidth = width
    cache.viewportHeight = height
  }

  // Always need opaque targets when MSAA enabled
  if (!cache.opaqueColor && config.msaa) {
    cache.opaqueColor = device.createTexture({
      width, height,
      format: 'rgba8unorm',
      sampleCount: 4,
      usage: 'render-target',
    })
    cache.opaqueDepth = device.createTexture({
      width, height,
      format: 'depth24plus',
      sampleCount: 4,
      usage: 'render-target',
    })
  }

  // Shadow maps — only if scene has shadow-casting light
  if (!cache.shadowCascades && scene.hasShadowCastingLight()) {
    cache.shadowCascades = [0, 1, 2].map(() =>
      device.createTexture({
        width: 2048, height: 2048,
        format: 'depth24',
        sampleCount: 1,
        usage: 'render-target|sampled',
      })
    )
  }

  // OIT targets — only if scene has transparent meshes
  if (!cache.oitAccumulation && scene.hasTransparentMeshes()) {
    cache.oitAccumulation = device.createTexture({
      width, height,
      format: 'rgba16float',
      sampleCount: 1,
      usage: 'render-target|sampled',
    })
    cache.oitRevealage = device.createTexture({
      width, height,
      format: 'r8unorm',
      sampleCount: 1,
      usage: 'render-target|sampled',
    })
  }

  // Bloom chain — only if bloom enabled
  if (!cache.bloomChain && config.bloom) {
    cache.bloomChain = []
    let w = Math.floor(width / 2)
    let h = Math.floor(height / 2)
    for (let i = 0; i < 5; i++) {
      cache.bloomChain.push(device.createTexture({
        width: w, height: h,
        format: 'rgba16float',
        sampleCount: 1,
        usage: 'render-target|sampled',
      }))
      w = Math.max(1, Math.floor(w / 2))
      h = Math.max(1, Math.floor(h / 2))
    }
  }

  // Resolve output — always needed
  if (!cache.resolveOutput) {
    cache.resolveOutput = device.createTexture({
      width, height,
      format: 'rgba8unorm',
      sampleCount: 1,
      usage: 'render-target|sampled',
    })
  }
}
```

---

## Shader Programs

### Shader Inventory

| Shader | Vertex | Fragment | Purpose |
|---|---|---|---|
| Opaque Lambert | `opaque.vert` | `lambert.frag` | Diffuse-only lit shading (N dot L + ambient) |
| Opaque Basic | `opaque.vert` | `basic.frag` | Unlit solid color / texture |
| Shadow Depth | `shadow.vert` | `shadow.frag` | Depth-only output for shadow maps |
| OIT Accumulation | `opaque.vert` | `oit_accum.frag` | Weighted blended OIT accumulation pass |
| OIT Composite | `fullscreen.vert` | `oit_composite.frag` | Composites OIT result onto opaque |
| Bloom Extract | `fullscreen.vert` | `bloom_extract.frag` | Extracts bright pixels above threshold |
| Bloom Blur | `fullscreen.vert` | `bloom_blur.frag` | Dual Kawase blur (down and up) |
| Tone Map | `fullscreen.vert` | `tonemap.frag` | ACES filmic tone mapping + gamma correction |
| Fullscreen Blit | `fullscreen.vert` | `blit.frag` | Simple texture copy to screen |

### Preprocessor Define System

All shaders share a common preprocessor system. Defines are injected at the top of the shader source before compilation. This allows a single shader source file to produce multiple pipeline variants.

```typescript
type ShaderDefines = {
  HAS_UV?: boolean
  HAS_VERTEX_COLOR?: boolean
  HAS_NORMAL_MAP?: boolean
  HAS_SKINNING?: boolean
  NUM_BONES?: number
  NUM_CSM_CASCADES?: number
  RECEIVE_SHADOWS?: boolean
  USE_FOG?: boolean
  TONE_MAPPING?: 'ACES' | 'REINHARD' | 'LINEAR'
}

const buildShaderSource = (source: string, defines: ShaderDefines): string => {
  let header = ''
  for (const [key, value] of Object.entries(defines)) {
    if (value === true) {
      header += `#define ${key}\n`
    } else if (value !== false && value !== undefined) {
      header += `#define ${key} ${value}\n`
    }
  }
  return header + source
}
```

### Vertex Shader Architecture

**opaque.vert** — Used by Lambert, Basic, and OIT accumulation.

```glsl
// --- Injected defines ---
// #define HAS_UV
// #define HAS_VERTEX_COLOR
// #define HAS_SKINNING
// #define NUM_BONES 128
// #define RECEIVE_SHADOWS
// #define NUM_CSM_CASCADES 3

// --- Uniform blocks ---
layout(std140) uniform FrameUniforms {       // binding 0
    mat4 u_viewMatrix;
    mat4 u_projectionMatrix;
    mat4 u_viewProjectionMatrix;
    vec3 u_cameraPosition;                   // Z-up world space
    float u_time;
    vec3 u_sunDirection;                     // normalized, Z-up
    float u_padding0;
    vec3 u_sunColor;
    float u_sunIntensity;
    vec3 u_ambientColor;
    float u_ambientIntensity;
#ifdef RECEIVE_SHADOWS
    mat4 u_shadowMatrices[NUM_CSM_CASCADES]; // world → shadow clip per cascade
    float u_cascadeSplits[NUM_CSM_CASCADES];
#endif
};

layout(std140) uniform ObjectUniforms {      // binding 2, dynamic offset
    mat4 u_worldMatrix;
    float u_materialIndex;
    float u_pad1;
    float u_pad2;
    float u_pad3;
#ifdef HAS_SKINNING
    mat4 u_boneMatrices[NUM_BONES];
#endif
};

// --- Attributes ---
layout(location = 0) in vec3 a_position;     // Z-up right-handed
layout(location = 1) in vec3 a_normal;
#ifdef HAS_UV
layout(location = 2) in vec2 a_uv;
#endif
#ifdef HAS_VERTEX_COLOR
layout(location = 3) in vec4 a_color;
#endif
#ifdef HAS_SKINNING
layout(location = 4) in vec4 a_joints;
layout(location = 5) in vec4 a_weights;
#endif

// --- Varyings ---
out vec3 v_worldPosition;
out vec3 v_worldNormal;
#ifdef HAS_UV
out vec2 v_uv;
#endif
#ifdef HAS_VERTEX_COLOR
out vec4 v_color;
#endif
#ifdef RECEIVE_SHADOWS
out vec4 v_shadowCoords[NUM_CSM_CASCADES];
#endif
out float v_viewDepth;

void main() {
    vec4 localPos = vec4(a_position, 1.0);
    vec3 localNormal = a_normal;

#ifdef HAS_SKINNING
    mat4 skinMatrix =
        u_boneMatrices[int(a_joints.x)] * a_weights.x +
        u_boneMatrices[int(a_joints.y)] * a_weights.y +
        u_boneMatrices[int(a_joints.z)] * a_weights.z +
        u_boneMatrices[int(a_joints.w)] * a_weights.w;
    localPos = skinMatrix * localPos;
    localNormal = mat3(skinMatrix) * localNormal;
#endif

    vec4 worldPos = u_worldMatrix * localPos;
    v_worldPosition = worldPos.xyz;
    v_worldNormal = normalize(mat3(u_worldMatrix) * localNormal);

#ifdef HAS_UV
    v_uv = a_uv;
#endif
#ifdef HAS_VERTEX_COLOR
    v_color = a_color;
#endif

#ifdef RECEIVE_SHADOWS
    for (int i = 0; i < NUM_CSM_CASCADES; i++) {
        v_shadowCoords[i] = u_shadowMatrices[i] * worldPos;
    }
#endif

    vec4 viewPos = u_viewMatrix * worldPos;
    v_viewDepth = -viewPos.z;   // positive depth into screen (Z-up RH)

    gl_Position = u_projectionMatrix * viewPos;
}
```

### Fragment Shader: Lambert

**lambert.frag** — Diffuse-only lit shading.

```glsl
// --- Injected defines ---
// #define HAS_UV
// #define HAS_VERTEX_COLOR
// #define RECEIVE_SHADOWS
// #define NUM_CSM_CASCADES 3

precision highp float;

layout(std140) uniform FrameUniforms {       // same as vertex
    mat4 u_viewMatrix;
    mat4 u_projectionMatrix;
    mat4 u_viewProjectionMatrix;
    vec3 u_cameraPosition;
    float u_time;
    vec3 u_sunDirection;
    float u_padding0;
    vec3 u_sunColor;
    float u_sunIntensity;
    vec3 u_ambientColor;
    float u_ambientIntensity;
#ifdef RECEIVE_SHADOWS
    mat4 u_shadowMatrices[NUM_CSM_CASCADES];
    float u_cascadeSplits[NUM_CSM_CASCADES];
#endif
};

layout(std140) uniform MaterialUniforms {    // binding 1
    vec4 u_baseColor;
    float u_emissiveIntensity;
    float u_pad_m1;
    float u_pad_m2;
    float u_pad_m3;
};

#ifdef HAS_UV
uniform sampler2D u_baseColorMap;
#endif
#ifdef RECEIVE_SHADOWS
uniform sampler2DShadow u_shadowMaps[NUM_CSM_CASCADES];
#endif

in vec3 v_worldPosition;
in vec3 v_worldNormal;
#ifdef HAS_UV
in vec2 v_uv;
#endif
#ifdef HAS_VERTEX_COLOR
in vec4 v_color;
#endif
#ifdef RECEIVE_SHADOWS
in vec4 v_shadowCoords[NUM_CSM_CASCADES];
#endif
in float v_viewDepth;

layout(location = 0) out vec4 fragColor;

float sampleShadowPCF(sampler2DShadow map, vec4 coords) {
    vec3 projected = coords.xyz / coords.w;
    projected = projected * 0.5 + 0.5;

    float shadow = 0.0;
    vec2 texelSize = vec2(1.0 / 2048.0);

    // 5x5 PCF kernel
    for (int x = -2; x <= 2; x++) {
        for (int y = -2; y <= 2; y++) {
            vec3 sampleCoord = vec3(projected.xy + vec2(x, y) * texelSize, projected.z);
            shadow += texture(map, sampleCoord);
        }
    }
    return shadow / 25.0;
}

void main() {
    vec4 albedo = u_baseColor;

#ifdef HAS_UV
    albedo *= texture(u_baseColorMap, v_uv);
#endif
#ifdef HAS_VERTEX_COLOR
    albedo *= v_color;
#endif

    vec3 N = normalize(v_worldNormal);
    vec3 L = normalize(u_sunDirection);

    // Lambert diffuse
    float NdotL = max(dot(N, L), 0.0);
    vec3 diffuse = u_sunColor * u_sunIntensity * NdotL;
    vec3 ambient = u_ambientColor * u_ambientIntensity;

    // Shadow
    float shadow = 1.0;
#ifdef RECEIVE_SHADOWS
    // Select cascade based on view depth
    int cascade = NUM_CSM_CASCADES - 1;
    for (int i = 0; i < NUM_CSM_CASCADES; i++) {
        if (v_viewDepth < u_cascadeSplits[i]) {
            cascade = i;
            break;
        }
    }
    shadow = sampleShadowPCF(u_shadowMaps[cascade], v_shadowCoords[cascade]);
#endif

    vec3 lighting = ambient + diffuse * shadow;
    vec3 color = albedo.rgb * lighting;

    // Emissive contribution (for bloom extraction)
    color += albedo.rgb * u_emissiveIntensity;

    fragColor = vec4(color, albedo.a);
}
```

### Fragment Shader: Basic (Unlit)

**basic.frag** — No lighting, just base color and optional texture/vertex color.

```glsl
precision highp float;

layout(std140) uniform MaterialUniforms {
    vec4 u_baseColor;
    float u_emissiveIntensity;
    float u_pad_m1;
    float u_pad_m2;
    float u_pad_m3;
};

#ifdef HAS_UV
uniform sampler2D u_baseColorMap;
#endif

in vec3 v_worldPosition;
in vec3 v_worldNormal;
#ifdef HAS_UV
in vec2 v_uv;
#endif
#ifdef HAS_VERTEX_COLOR
in vec4 v_color;
#endif

layout(location = 0) out vec4 fragColor;

void main() {
    vec4 color = u_baseColor;

#ifdef HAS_UV
    color *= texture(u_baseColorMap, v_uv);
#endif
#ifdef HAS_VERTEX_COLOR
    color *= v_color;
#endif

    fragColor = color;
}
```

### Fragment Shader: Shadow Depth

**shadow.vert / shadow.frag** — Minimal shaders that output only depth.

```glsl
// shadow.vert
layout(std140) uniform ShadowUniforms {
    mat4 u_lightViewProjection;
};
layout(std140) uniform ObjectUniforms {
    mat4 u_worldMatrix;
    float u_materialIndex;
    float u_pad1;
    float u_pad2;
    float u_pad3;
#ifdef HAS_SKINNING
    mat4 u_boneMatrices[NUM_BONES];
#endif
};

layout(location = 0) in vec3 a_position;
#ifdef HAS_SKINNING
layout(location = 4) in vec4 a_joints;
layout(location = 5) in vec4 a_weights;
#endif

void main() {
    vec4 localPos = vec4(a_position, 1.0);
#ifdef HAS_SKINNING
    mat4 skinMatrix =
        u_boneMatrices[int(a_joints.x)] * a_weights.x +
        u_boneMatrices[int(a_joints.y)] * a_weights.y +
        u_boneMatrices[int(a_joints.z)] * a_weights.z +
        u_boneMatrices[int(a_joints.w)] * a_weights.w;
    localPos = skinMatrix * localPos;
#endif
    gl_Position = u_lightViewProjection * u_worldMatrix * localPos;
}

// shadow.frag
precision highp float;
void main() {
    // Depth is written automatically by the hardware
    // No color output needed — depth-only pass
}
```

### Fragment Shader: OIT Accumulation

**oit_accum.frag** — Weighted Blended OIT (McGuire & Bavoil 2013).

```glsl
precision highp float;

layout(std140) uniform MaterialUniforms {
    vec4 u_baseColor;
    float u_emissiveIntensity;
    float u_pad_m1;
    float u_pad_m2;
    float u_pad_m3;
};

#ifdef HAS_UV
uniform sampler2D u_baseColorMap;
#endif

in vec3 v_worldPosition;
in vec3 v_worldNormal;
#ifdef HAS_UV
in vec2 v_uv;
#endif
#ifdef HAS_VERTEX_COLOR
in vec4 v_color;
#endif
in float v_viewDepth;

// MRT output: accumulation (RGBA16F) and revealage (R8)
layout(location = 0) out vec4 accumulation;
layout(location = 1) out float revealage;

void main() {
    vec4 color = u_baseColor;

#ifdef HAS_UV
    color *= texture(u_baseColorMap, v_uv);
#endif
#ifdef HAS_VERTEX_COLOR
    color *= v_color;
#endif

    // TODO: lighting could be applied here too for lit transparency

    float alpha = color.a;

    // Depth weight function (McGuire & Bavoil 2013, Equation 10)
    float viewDepth = v_viewDepth;
    float weight = max(min(1.0, max(max(color.r, color.g), color.b) * alpha), alpha) *
                   clamp(0.03 / (1e-5 + pow(viewDepth / 200.0, 4.0)), 1e-2, 3e3);

    // Accumulation: premultiplied-alpha color × weight
    accumulation = vec4(color.rgb * alpha * weight, alpha * weight);

    // Revealage: how much background shows through
    revealage = alpha;
}
```

**Blend state for OIT accumulation pass:**
- Accumulation attachment: `src=ONE, dst=ONE` (additive)
- Revealage attachment: `src=ZERO, dst=ONE_MINUS_SRC_COLOR` (product)

### Fragment Shader: OIT Composite

**oit_composite.frag** — Blends OIT result onto the opaque framebuffer.

```glsl
precision highp float;

uniform sampler2D u_accumulation;
uniform sampler2D u_revealage;

in vec2 v_uv;

layout(location = 0) out vec4 fragColor;

void main() {
    vec4 accum = texture(u_accumulation, v_uv);
    float reveal = texture(u_revealage, v_uv).r;

    // If revealage is ~1.0, nothing transparent was drawn here
    if (reveal >= 1.0) {
        discard;
    }

    // Reconstruct average color
    vec3 averageColor = accum.rgb / max(accum.a, 1e-5);

    // Composite: transparent color * (1 - revealage) over destination
    fragColor = vec4(averageColor, 1.0 - reveal);
}
```

**Blend state for OIT composite pass:** `src=SRC_ALPHA, dst=ONE_MINUS_SRC_ALPHA` (standard alpha blending).

### Fragment Shader: Bloom Extract

**bloom_extract.frag** — Extracts pixels brighter than a threshold.

```glsl
precision highp float;

uniform sampler2D u_sceneColor;
uniform float u_threshold;

in vec2 v_uv;

layout(location = 0) out vec4 fragColor;

void main() {
    vec3 color = texture(u_sceneColor, v_uv).rgb;

    float brightness = dot(color, vec3(0.2126, 0.7152, 0.0722));
    vec3 bloom = color * max(brightness - u_threshold, 0.0) / max(brightness, 1e-5);

    fragColor = vec4(bloom, 1.0);
}
```

### Fragment Shader: Bloom Blur (Dual Kawase)

**bloom_blur.frag** — Used for both downscale and upscale passes with different UV offsets.

```glsl
precision highp float;

uniform sampler2D u_sourceTexture;
uniform vec2 u_texelSize;     // 1.0 / source resolution
uniform float u_direction;    // 0.0 = downsample, 1.0 = upsample

in vec2 v_uv;

layout(location = 0) out vec4 fragColor;

void main() {
    vec2 ts = u_texelSize;

    if (u_direction < 0.5) {
        // Downsample: 5-tap pattern
        vec3 color = texture(u_sourceTexture, v_uv).rgb * 4.0;
        color += texture(u_sourceTexture, v_uv + vec2(-ts.x, -ts.y)).rgb;
        color += texture(u_sourceTexture, v_uv + vec2( ts.x, -ts.y)).rgb;
        color += texture(u_sourceTexture, v_uv + vec2(-ts.x,  ts.y)).rgb;
        color += texture(u_sourceTexture, v_uv + vec2( ts.x,  ts.y)).rgb;
        fragColor = vec4(color / 8.0, 1.0);
    } else {
        // Upsample: 8-tap tent filter
        vec3 color = vec3(0.0);
        color += texture(u_sourceTexture, v_uv + vec2(-ts.x,  0.0)).rgb * 2.0;
        color += texture(u_sourceTexture, v_uv + vec2( ts.x,  0.0)).rgb * 2.0;
        color += texture(u_sourceTexture, v_uv + vec2( 0.0, -ts.y)).rgb * 2.0;
        color += texture(u_sourceTexture, v_uv + vec2( 0.0,  ts.y)).rgb * 2.0;
        color += texture(u_sourceTexture, v_uv + vec2(-ts.x, -ts.y)).rgb;
        color += texture(u_sourceTexture, v_uv + vec2( ts.x, -ts.y)).rgb;
        color += texture(u_sourceTexture, v_uv + vec2(-ts.x,  ts.y)).rgb;
        color += texture(u_sourceTexture, v_uv + vec2( ts.x,  ts.y)).rgb;
        fragColor = vec4(color / 12.0, 1.0);
    }
}
```

### Fragment Shader: Tone Map

**tonemap.frag** — ACES filmic tone mapping with gamma correction.

```glsl
precision highp float;

uniform sampler2D u_hdrColor;
uniform sampler2D u_bloomTexture;
uniform float u_bloomIntensity;
uniform float u_exposure;

in vec2 v_uv;

layout(location = 0) out vec4 fragColor;

// ACES filmic tone mapping (Narkowicz 2015 fit)
vec3 acesToneMap(vec3 x) {
    float a = 2.51;
    float b = 0.03;
    float c = 2.43;
    float d = 0.59;
    float e = 0.14;
    return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0);
}

void main() {
    vec3 hdr = texture(u_hdrColor, v_uv).rgb;
    vec3 bloom = texture(u_bloomTexture, v_uv).rgb;

    // Add bloom
    hdr += bloom * u_bloomIntensity;

    // Exposure
    hdr *= u_exposure;

    // Tone map
    vec3 ldr = acesToneMap(hdr);

    // Gamma correction (linear → sRGB)
    ldr = pow(ldr, vec3(1.0 / 2.2));

    fragColor = vec4(ldr, 1.0);
}
```

### Vertex Shader: Fullscreen Quad

**fullscreen.vert** — Used by all post-processing and composite shaders. Generates a fullscreen triangle from vertex ID alone (no vertex buffer needed).

```glsl
out vec2 v_uv;

void main() {
    // Fullscreen triangle trick — 3 vertices, no VBO
    v_uv = vec2((gl_VertexID << 1) & 2, gl_VertexID & 2);
    gl_Position = vec4(v_uv * 2.0 - 1.0, 0.0, 1.0);
}
```

### Fragment Shader: Blit

**blit.frag** — Simple texture-to-screen copy.

```glsl
precision highp float;

uniform sampler2D u_source;

in vec2 v_uv;

layout(location = 0) out vec4 fragColor;

void main() {
    fragColor = texture(u_source, v_uv);
}
```

---

## Main Render Loop

The following TypeScript pseudocode shows the complete per-frame render function. It ties together all of the systems described above: frustum culling, command generation, sorting, multi-pass rendering, and post-processing.

```typescript
type RendererConfig = {
  msaa: boolean
  msaaSamples: number
  bloom: boolean
  bloomThreshold: number
  bloomIntensity: number
  bloomLevels: number
  exposure: number
  toneMapping: 'ACES' | 'REINHARD' | 'LINEAR'
  depthPrepass: boolean
}

type RendererState = {
  device: GALDevice
  config: RendererConfig
  renderTargets: RenderTargetCache
  commandPool: RenderCommand[]
  commandPoolAux: RenderCommand[]
  commandCount: number
  commandsDirty: boolean
  opaqueCount: number
  transparentCount: number
  dynamicUBO: DynamicUBO
  frameUBO: GALBuffer
  frustum: Frustum
  shaders: ShaderCache
  staticBundle: GPURenderBundle | null
  staticBundleDirty: boolean
}

const render = (state: RendererState, scene: Scene, camera: Camera): void => {
  const { device, config, renderTargets } = state
  const width = device.canvasWidth
  const height = device.canvasHeight

  // ---------------------------------------------------------------
  // Phase 1: Update transforms and cull
  // ---------------------------------------------------------------

  // Walk the scene graph and update world matrices for dirty nodes
  scene.updateWorldTransforms()

  // Build view-projection matrix and extract frustum planes
  const viewMatrix = camera.getViewMatrix()        // Z-up right-handed
  const projMatrix = camera.getProjectionMatrix()
  const vpMatrix = mat4Multiply(projMatrix, viewMatrix)
  extractFrustum(vpMatrix, state.frustum)

  // ---------------------------------------------------------------
  // Phase 2: Generate and sort commands (only if dirty)
  // ---------------------------------------------------------------

  if (state.commandsDirty || camera.frustumChanged) {
    state.commandCount = 0
    state.opaqueCount = 0
    state.transparentCount = 0

    const near = camera.near
    const far = camera.far

    // Iterate all mesh nodes in the scene
    const meshNodes = scene.getMeshNodes()
    for (let i = 0; i < meshNodes.length; i++) {
      const node = meshNodes[i]
      const aabb = node.worldAABB

      // Frustum cull — 6-plane test with early-out
      if (!testAABBFrustum(
        state.frustum,
        aabb.minX, aabb.minY, aabb.minZ,
        aabb.maxX, aabb.maxY, aabb.maxZ
      )) {
        continue  // culled — no command generated
      }

      // Compute view-space depth for sort key
      const centerX = (aabb.minX + aabb.maxX) * 0.5
      const centerY = (aabb.minY + aabb.maxY) * 0.5
      const centerZ = (aabb.minZ + aabb.maxZ) * 0.5
      const viewDepth = -(
        viewMatrix[2] * centerX +
        viewMatrix[6] * centerY +
        viewMatrix[10] * centerZ +
        viewMatrix[14]
      )

      const cmd = state.commandPool[state.commandCount]
      cmd.mesh = node
      cmd.material = node.materialId
      cmd.geometry = node.geometryId
      cmd.worldMatrix = node.worldMatrix
      cmd.boneMatrices = node.boneMatrices
      cmd.materialIndex = node.materialIndex

      if (node.isTransparent) {
        cmd.sortKey = buildSortKeyTransparent(
          node.pipelineId, node.materialId, viewDepth, near, far
        )
        state.transparentCount++
      } else {
        cmd.sortKey = buildSortKeyOpaque(
          node.pipelineId, node.materialId, viewDepth, near, far
        )
        state.opaqueCount++
      }

      state.commandCount++
    }

    // Partition: opaque commands first, then transparent
    // (Simple approach: sort all, opaque pipelines have lower IDs)
    radixSort64(state.commandPool, state.commandPoolAux, state.commandCount)

    state.commandsDirty = false
    state.staticBundleDirty = true
  }

  // ---------------------------------------------------------------
  // Phase 3: Ensure render targets are allocated
  // ---------------------------------------------------------------

  ensureRenderTargets(renderTargets, device, width, height, scene, config)

  // ---------------------------------------------------------------
  // Phase 4: Upload per-frame uniforms
  // ---------------------------------------------------------------

  state.dynamicUBO.offset = 0  // reset dynamic UBO for this frame

  const frameData = buildFrameUniformData(camera, scene, config)
  device.writeBuffer(state.frameUBO, 0, frameData)

  // ---------------------------------------------------------------
  // Phase 5: Begin GPU command encoding
  // ---------------------------------------------------------------

  const encoder = device.beginCommandEncoder()

  // ---------------------------------------------------------------
  // Pass 0: Shadow Maps
  // ---------------------------------------------------------------

  if (renderTargets.shadowCascades && scene.hasShadowCastingLight()) {
    const light = scene.getDirectionalLight()
    const cascadeMatrices = computeCSMCascades(camera, light, 3)

    for (let cascade = 0; cascade < 3; cascade++) {
      const shadowPass = encoder.beginRenderPass({
        colorAttachments: [],
        depthStencilAttachment: {
          view: renderTargets.shadowCascades[cascade].view,
          depthLoadOp: 'clear',
          depthStoreOp: 'store',
          depthClearValue: 1.0,
        },
      })

      // Render all shadow-casting opaque objects
      const shadowUniforms = { lightViewProjection: cascadeMatrices[cascade] }
      device.writeBuffer(state.frameUBO, 0, shadowUniforms)

      for (let i = 0; i < state.opaqueCount; i++) {
        const cmd = state.commandPool[i]
        if (!cmd.mesh.castShadow) continue

        shadowPass.setPipeline(device.getPipeline(state.shaders.shadowPipeline))

        const offset = dynamicUBOWrite(
          state.dynamicUBO,
          cmd.worldMatrix,
          cmd.materialIndex,
          cmd.boneMatrices
        )
        shadowPass.setBindGroup(2, state.dynamicUBO.bindGroup, [offset])
        shadowPass.setVertexBuffer(0, device.getVertexBuffer(cmd.geometry))
        shadowPass.setIndexBuffer(device.getIndexBuffer(cmd.geometry))

        const geo = device.getGeometryInfo(cmd.geometry)
        shadowPass.drawIndexed(geo.indexCount, 1, geo.indexOffset, 0, 0)
      }

      shadowPass.end()
    }

    // Re-upload frame uniforms with camera matrices (shadow pass overwrote them)
    device.writeBuffer(state.frameUBO, 0, frameData)
  }

  // ---------------------------------------------------------------
  // Pass 1: Opaque Pass (4x MSAA)
  // ---------------------------------------------------------------

  const opaquePass = encoder.beginRenderPass({
    colorAttachments: [{
      view: renderTargets.opaqueColor!.view,
      loadOp: 'clear',
      storeOp: 'store',
      clearValue: { r: 0.0, g: 0.0, b: 0.0, a: 1.0 },
    }],
    depthStencilAttachment: {
      view: renderTargets.opaqueDepth!.view,
      depthLoadOp: 'clear',
      depthStoreOp: 'store',
      depthClearValue: 1.0,
    },
  })

  opaquePass.setBindGroup(0, device.getFrameBindGroup(state.frameUBO))

  executeCommands(
    state.commandPool,
    state.opaqueCount,
    device,
    opaquePass,
    state.frameUBO,
    state.dynamicUBO
  )

  opaquePass.end()

  // ---------------------------------------------------------------
  // Pass 2: Transparent Pass (OIT Accumulation)
  // ---------------------------------------------------------------

  if (state.transparentCount > 0 && renderTargets.oitAccumulation) {
    const oitPass = encoder.beginRenderPass({
      colorAttachments: [
        {
          view: renderTargets.oitAccumulation.view,
          loadOp: 'clear',
          storeOp: 'store',
          clearValue: { r: 0.0, g: 0.0, b: 0.0, a: 0.0 },
        },
        {
          view: renderTargets.oitRevealage!.view,
          loadOp: 'clear',
          storeOp: 'store',
          clearValue: { r: 1.0, g: 0.0, b: 0.0, a: 0.0 },
        },
      ],
      depthStencilAttachment: {
        view: renderTargets.opaqueDepth!.view,
        depthLoadOp: 'load',        // reuse opaque depth
        depthStoreOp: 'store',
        depthReadOnly: true,         // depth test but no write
      },
    })

    oitPass.setBindGroup(0, device.getFrameBindGroup(state.frameUBO))

    // Transparent commands start after opaque commands in the sorted array
    const transparentOffset = state.opaqueCount

    executeCommands(
      state.commandPool.slice(transparentOffset),
      state.transparentCount,
      device,
      oitPass,
      state.frameUBO,
      state.dynamicUBO
    )

    oitPass.end()

    // ---------------------------------------------------------------
    // Pass 3: OIT Composite
    // ---------------------------------------------------------------

    const compositePass = encoder.beginRenderPass({
      colorAttachments: [{
        view: renderTargets.opaqueColor!.view,
        loadOp: 'load',    // preserve opaque result
        storeOp: 'store',
      }],
    })

    compositePass.setPipeline(device.getPipeline(state.shaders.oitCompositePipeline))
    compositePass.setBindGroup(0, device.createBindGroup({
      texture0: renderTargets.oitAccumulation,
      texture1: renderTargets.oitRevealage!,
    }))
    compositePass.draw(3)  // fullscreen triangle
    compositePass.end()
  }

  // ---------------------------------------------------------------
  // Pass 4: Post-Processing
  // ---------------------------------------------------------------

  if (config.bloom && renderTargets.bloomChain) {
    const bloomChain = renderTargets.bloomChain

    // Bloom extract: scene color → bloom level 0
    const extractPass = encoder.beginRenderPass({
      colorAttachments: [{
        view: bloomChain[0].view,
        loadOp: 'clear',
        storeOp: 'store',
      }],
    })

    extractPass.setPipeline(device.getPipeline(state.shaders.bloomExtractPipeline))
    extractPass.setBindGroup(0, device.createBindGroup({
      sceneColor: renderTargets.opaqueColor!,
      threshold: config.bloomThreshold,
    }))
    extractPass.draw(3)
    extractPass.end()

    // Progressive downscale
    for (let i = 1; i < config.bloomLevels; i++) {
      const downPass = encoder.beginRenderPass({
        colorAttachments: [{
          view: bloomChain[i].view,
          loadOp: 'clear',
          storeOp: 'store',
        }],
      })

      downPass.setPipeline(device.getPipeline(state.shaders.bloomBlurPipeline))
      downPass.setBindGroup(0, device.createBindGroup({
        sourceTexture: bloomChain[i - 1],
        texelSize: [1.0 / bloomChain[i - 1].width, 1.0 / bloomChain[i - 1].height],
        direction: 0.0,  // downsample
      }))
      downPass.draw(3)
      downPass.end()
    }

    // Progressive upscale with additive blend
    for (let i = config.bloomLevels - 2; i >= 0; i--) {
      const upPass = encoder.beginRenderPass({
        colorAttachments: [{
          view: bloomChain[i].view,
          loadOp: 'load',     // additive onto existing content
          storeOp: 'store',
        }],
      })

      upPass.setPipeline(device.getPipeline(state.shaders.bloomBlurPipeline))
      upPass.setBindGroup(0, device.createBindGroup({
        sourceTexture: bloomChain[i + 1],
        texelSize: [1.0 / bloomChain[i + 1].width, 1.0 / bloomChain[i + 1].height],
        direction: 1.0,  // upsample
      }))
      upPass.draw(3)
      upPass.end()
    }
  }

  // Tone mapping
  const toneMapPass = encoder.beginRenderPass({
    colorAttachments: [{
      view: renderTargets.opaqueColor!.view,
      loadOp: 'load',
      storeOp: 'store',
    }],
  })

  toneMapPass.setPipeline(device.getPipeline(state.shaders.toneMapPipeline))
  toneMapPass.setBindGroup(0, device.createBindGroup({
    hdrColor: renderTargets.opaqueColor!,
    bloomTexture: config.bloom ? renderTargets.bloomChain![0] : device.whiteTexture,
    bloomIntensity: config.bloom ? config.bloomIntensity : 0.0,
    exposure: config.exposure,
  }))
  toneMapPass.draw(3)
  toneMapPass.end()

  // ---------------------------------------------------------------
  // Pass 5: MSAA Resolve + Final Output
  // ---------------------------------------------------------------

  if (config.msaa) {
    encoder.resolveTexture(
      renderTargets.opaqueColor!,   // MSAA source
      renderTargets.resolveOutput!   // 1x destination
    )

    // Blit resolved result to canvas
    const finalPass = encoder.beginRenderPass({
      colorAttachments: [{
        view: device.getCurrentCanvasView(),
        loadOp: 'clear',
        storeOp: 'store',
      }],
    })

    finalPass.setPipeline(device.getPipeline(state.shaders.blitPipeline))
    finalPass.setBindGroup(0, device.createBindGroup({
      source: renderTargets.resolveOutput!,
    }))
    finalPass.draw(3)
    finalPass.end()
  } else {
    // No MSAA — blit directly from opaque color to canvas
    const finalPass = encoder.beginRenderPass({
      colorAttachments: [{
        view: device.getCurrentCanvasView(),
        loadOp: 'clear',
        storeOp: 'store',
      }],
    })

    finalPass.setPipeline(device.getPipeline(state.shaders.blitPipeline))
    finalPass.setBindGroup(0, device.createBindGroup({
      source: renderTargets.opaqueColor!,
    }))
    finalPass.draw(3)
    finalPass.end()
  }

  // ---------------------------------------------------------------
  // Submit
  // ---------------------------------------------------------------

  device.submitCommands(encoder.finish())
}
```

### Frame Lifecycle Summary

```
1. scene.updateWorldTransforms()     — walk dirty flags, update matrices
2. extractFrustum()                  — build 6 planes from VP matrix
3. frustum cull + command gen        — O(n) scan, ~20-40% survive
4. radixSort64()                     — O(n) sort on 64-bit key
5. ensureRenderTargets()             — lazy alloc if needed
6. upload frame UBO                  — camera, lights, shadows
7. Pass 0: shadow maps              — 3 cascades, depth only
8. Pass 1: opaque                   — sorted front-to-back, 4x MSAA
9. Pass 2: OIT accumulation         — transparent, additive blend to float MRT
10. Pass 3: OIT composite           — fullscreen quad, alpha blend onto opaque
11. Pass 4: post-processing         — bloom chain + tone mapping
12. Pass 5: MSAA resolve + blit     — resolve 4x → 1x, copy to canvas
13. device.submitCommands()          — single command buffer submission
```
