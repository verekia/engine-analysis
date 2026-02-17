# Renderer

## GPU Renderer Interface

```ts
interface Renderer {
  init(canvas: HTMLCanvasElement): Promise<void>
  draw(draws: DrawCommand[], world: World): void
  destroy(): void
}

interface DrawCommand {
  pipelineId: number
  materialId: number
  geometryId: number
  instanceCount: number
  // Offset into the visible list where this run starts
  startIndex: number
}
```

Two implementations: `WebGPURenderer` and `WebGLRenderer`, selected at init time based on `navigator.gpu` availability.

## WebGPU Renderer

- **Bind group 0**: Camera (view + projection matrices)
- **Bind group 1**: Light (direction, color, ambient)
- **Bind group 2**: Material-specific (textures, samplers)
- **Instance buffer**: Written each frame with per-instance data for all visible entities
- **Draw call**: `renderPass.drawIndexed(indexCount, instanceCount, 0, 0, firstInstance)`

### Instance Buffer Layout

Per instance, written each frame:

```
worldMatrix:  mat4x4<f32>   (64 bytes)
color:        vec4<f32>     (16 bytes)
flags:        u32           (4 bytes)
padding:      12 bytes      (align to 96)
                            ─────────────
                            96 bytes/instance
```

### Pipeline Architecture

Three main pipelines:

1. **Static** — unlit/lit static meshes
2. **Skinned** — skeletal animation (v2)
3. **Textured** — texture-mapped meshes

### Bind Groups

- **Bind group 0**: Camera uniforms (shared across all draws)
- **Bind group 1**: Light uniforms (shared across all draws)
- **Bind group 2**: Material-specific data (textures, samplers) — varies per material

## WebGL2 Renderer

- **UBO 0**: Camera
- **UBO 1**: Light
- Instancing via `ANGLE_instanced_arrays` / WebGL2 native
- `gl.drawElementsInstanced()` for each draw command

### Fallback Strategy

**Fallback:** WebGL2 for Safari and older browsers. Same API shape, different backend implementation (UBOs instead of bind groups, separate draw calls instead of indirect).

## Render Loop (World.render())

```ts
render() {
  // 1. Recompute dirty matrices
  for (let i = 0; i < this.entityCount; i++) {
    if (this.flags[i] & DIRTY) {
      mat4_from_trs(this.worldMatrices, i, this.positions, this.rotations, this.scales)
      this.flags[i] &= ~DIRTY
    }
  }

  // 2. Update camera matrices
  this.camera.update()  // writes view, projection, VP into scratch arrays

  // 3. Frustum cull
  extractFrustumPlanes(this.camera.vp, _frustumPlanes)
  let visibleCount = 0
  for (let i = 0; i < this.entityCount; i++) {
    if (!(this.flags[i] & VISIBLE)) continue
    if (sphereInFrustum(this.boundingSpheres, i, _frustumPlanes)) {
      this.visibleList[visibleCount++] = i
    }
  }

  // 4. Build sort keys for visible entities
  for (let j = 0; j < visibleCount; j++) {
    const i = this.visibleList[j]
    this.sortKeys[j] = packSortKey(this.pipelineIds[i], this.materialIds[i], this.geometryIds[i])
  }

  // 5. Sort visible list by sort key
  sortByKey(this.visibleList, this.sortKeys, visibleCount)

  // 6. Build instanced draw commands (merge consecutive same-key runs)
  const draws = buildDrawCommands(this.visibleList, this.sortKeys, visibleCount)

  // 7. Upload instance data + submit GPU draws
  this.renderer.draw(draws, this)
}
```

## WebGPU Performance Benefits

WebGPU has dramatically lower CPU overhead per draw call compared to WebGL:

- No synchronous driver validation per call
- Command buffers are recorded and submitted in batch
- Bind groups reduce state setting
- `drawIndexedIndirect` enables true GPU-driven rendering later

**Expected impact:** 20-40% frame time improvement over WebGL2 for draw-call-heavy scenes.
