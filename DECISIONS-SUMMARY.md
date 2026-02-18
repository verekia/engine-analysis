# Engine Decisions Summary

A concise overview of all architectural and implementation decisions. Unanimous decisions are marked ✅. Split decisions note the winning approach.

---

## Philosophy & Structure

**Goal:** 2000 draw calls at 60fps on recent mobile phones. Every decision traces back to this target.

**Package structure:** Single monolithic package (`engine`) with internal modules. React bindings as a separate `engine-react` package (adds `react`/`react-reconciler` peer deps). *(5/9 implementations agreed)*

**API style:** OOP public API with SoA internals. ✅
- Users get familiar object-oriented objects (mesh, material, node)
- Internally, contiguous `Float32Array` pools for cache-friendly GPU upload

**No instancing.** ✅ Per-draw JS overhead <2–3μs makes 2000 unique draws feasible without it.

**Code style:** TypeScript strict mode, no semicolons, const arrow functions. ✅

---

## Coordinate System & Math

**Z-up right-handed** (X=right, Y=forward, Z=up). Matches Blender. ✅

```typescript
// glTF (Y-up) converted at load time — no per-frame overhead
const engineVertex = [x, z, -y]  // swap Y↔Z, negate new Y
```

**Custom math library, no dependencies.** All types backed by `Float32Array`. Column-major 4×4 matrices. `[x,y,z,w]` quaternion order. Pure functional API with output parameter first. Pre-allocated module-level scratch variables — zero allocations in hot paths. ✅

---

## Initialization & API

```typescript
const engine = await createEngine(canvas, {
  backend: 'auto',      // tries WebGPU, falls back to WebGL2
  antialias: true,      // 4x MSAA
  shadows: true,        // or { mapSize: 1024, cascades: 3, maxDistance: 200 }
  bloom: { intensity: 0.5 },
  toneMapping: 'aces',
})

const scene = createScene()
scene.ambientLight = { color: [0.4, 0.45, 0.5], intensity: 0.3 }

const mesh = createMesh(
  createBoxGeometry({ width: 1, height: 1, depth: 1 }),
  createLambertMaterial({ color: [0.8, 0.6, 0.4], receiveShadow: true })
)
scene.add(mesh)

engine.onFrame((dt) => { controls.update(dt); mixer.update(dt) })
engine.start(scene, camera)
```

All factory functions take a single options object. Boolean `true` on shadows/bloom uses sensible defaults.

---

## Scene Graph

**Traditional object graph with contiguous matrix storage.** *(6/9)* Users work with nodes; world matrices are packed into a shared `Float32Array` for GPU upload.

**Node types:** Group, Mesh, SkinnedMesh, Camera, DirectionalLight, AmbientLight.

**Transform:** position (Vec3) + rotation (Quat) + scale (Vec3). ✅ Quaternions prevent gimbal lock; Euler angles are convenience only.

**Two-level dirty flags with early-exit propagation:** *(Caracal/Fennec/Shark approach)*

```typescript
// Only ~2-5% of nodes move per frame → 95%+ of matrix work skipped
node._dirtyLocal  // local matrix needs recompute
node._dirtyWorld  // world matrix needs recompute (self or ancestor moved)
```

**Frustum culling:** AABB P-vertex test against 6 frustum planes. ✅ ~0.1–0.2ms for 2000 objects. BVH-accelerated available for >5000 objects.

**Bones are regular scene nodes.** ✅ Attachment = standard parenting:
```typescript
skeleton.getBone('hand_R').add(swordMesh)
```

---

## Renderer

**WebGPU-first abstraction, WebGL2 translation layer.** ✅

**7-pass pipeline per frame:** ✅
1. Shadow (3 CSM cascades, depth-only)
2. Opaque (MSAA, MRT: color + emissive)
3. Transparent (OIT accumulation)
4. MSAA Resolve
5. OIT Composite
6. Bloom
7. Final Blit (tone mapping + gamma)

**3 bind groups by update frequency:** ✅

| Slot | Contents | Updated |
|------|----------|---------|
| 0 | Camera VP, lights, shadow matrices | Once/frame |
| 1 | Material UBO, palette, textures | Per material switch |
| 2 | World matrix, bone matrices | Per draw (dynamic offset) |

Dynamic offsets on bind group 2 avoid per-object bind group creation.

**64-bit radix sort for draw calls.** *(5/9)* Sort key layout:
```
Bits 63-60: Layer + Pass
Bit  59:    Transparent flag
Bits 58-48: Pipeline ID   ← most expensive state change, highest bits
Bits 47-32: Material ID
Bits 31-0:  Depth
```
Result: 90–99% reduction in state changes (from ~2000 to ~10–200).

**WebGL2:** Full GL state cache (track all state, skip redundant calls). Pipeline = frozen program + blend + depth + cull state bundle. VAO cached per geometry+pipeline.

**WebGPU:** Render bundles available for static geometry but not the primary strategy. Pipeline cache by descriptor hash.

**Shaders:** Dual WGSL + GLSL (no runtime transpilation). ✅ Variants compiled lazily by feature bitmask (`HAS_SKINNING`, `HAS_VERTEX_COLORS`, `HAS_MATERIAL_INDEX`, etc.). Pre-warm 5 common variants during asset loading to prevent hitching.

**4x MSAA by default.** ✅ Nearly free on tile-based mobile GPUs. Resolved before post-processing.

---

## Performance

**Target: 2000 draw calls @ 60fps on mobile.** ✅ 16.6ms budget, <2–3μs JS per draw.

**Zero per-frame allocations.** ✅ No `new`, no array creation, no closures in hot paths. Pre-allocate everything.

**Render list rebuilt every frame.** *(5/9)* Collect + radix sort completes in <0.3ms — always correct, simple.

**Uniform strategy: dynamic offsets on shared buffer.** ✅ 2000 × 256B = ~512KB per frame via single write.

**Frame budget (CPU ~6ms, GPU ~10ms, they overlap):**

| Phase | Cost |
|-------|------|
| Matrix updates (dirty only) | 0.2–0.5ms CPU |
| Frustum culling | 0.1–0.2ms CPU |
| Radix sort | 0.05–0.1ms CPU |
| Draw submission (2000 draws) | 4–6ms CPU |
| Shadow maps (3 CSM) | ~1.5ms GPU |
| Opaque pass | 3–5ms GPU |
| Post-processing | 0.6–1.1ms GPU |

---

## Materials

**Two materials only:** BasicMaterial (unlit) and LambertMaterial (diffuse). ✅

**Lambert shading — no specular.** ✅ ~15 ALU ops vs ~80+ for PBR. Sufficient for stylized/low-poly.

```glsl
vec3 litColor = baseColor * (ambient + diffuse * NdotL * shadow) + emissive;
```

**32-entry material palette via UBO.** *(7/9)* Per-vertex `_materialindex` selects palette entry — complex visuals in a single draw call.

**MRT emissive output** for bloom input (only emissive surfaces contribute). Multiplicative vertex color blending. Feature flag bitmask for lazy shader compilation. Transparent materials auto-routed to OIT pass.

---

## Geometry

**Separate buffers per attribute (not interleaved).** ✅ GPU fetches only what the shader variant uses.

**Auto Uint16/Uint32 index selection** based on vertex count. ✅

**7 parametric primitives:** plane, box, sphere, cone, cylinder, capsule, circle. ✅ All Z-up, centered at origin, with normals + UVs.

**Lazy GPU buffer upload** on first render. `geometry.needsUpdate = true` to re-upload. ✅

**AABB as primary bounding volume.** ✅ Computed at creation, used for culling + BVH + raycasting.

**Lazy BVH construction** on first raycast, then cached. ✅

---

## Lighting & Shadows

**Lambert diffuse: directional + ambient lights only.** ✅ Point/spot lights out of scope.

**3-cascade CSM for 200×200m worlds.** ✅ 1024×1024 per cascade (configurable to 2048 for desktop).

**Texture array** (`TEXTURE_2D_ARRAY`, 3 layers) for shadow maps. Cleaner than atlas, no UV offset math.

**Lambda=0.7 cascade splits** (logarithmic-weighted). Near camera gets most resolution:
- Cascade 0: ~0.1–12m
- Cascade 1: ~12–50m
- Cascade 2: ~50–200m

**3×3 Poisson PCF** (9 samples). Poisson disk breaks aliasing better than regular grid. Hardware comparison samplers on both backends.

**Cascade blending** with `smoothstep` over ~10% blend zone. ✅ Eliminates visible seams.

**Texel snapping** to prevent shadow shimmer during camera movement. ✅

**Bias:** constant + slope-scaled, combined with back-face culling during shadow render. ✅

---

## Transparency

**Weighted Blended OIT (WBOIT) — no sorting required.** ✅

```glsl
float weight = alpha * clamp(0.03 / (1e-5 + pow(depth / 200.0, 4.0)), 1e-2, 3e3);
// Accumulation: additive blend (ONE, ONE)
// Revealage:    multiplicative (ZERO, ONE_MINUS_SRC_ALPHA)
```

Render targets: RGBA16F accumulation + R8 revealage. Depth test ON, depth write OFF. Composite pass blends result over resolved opaque.

Limitations (acceptable for stylized aesthetics): approximate ordering at similar depths, saturation with >8 overlapping layers.

WebGL2 requires `EXT_color_buffer_float` + `EXT_float_blend`. Falls back to sorted alpha blend if unavailable.

---

## Post-Processing

**Bloom: Unreal-style progressive downsample/upsample.** ✅ Driven by MRT emissive — no global threshold needed.

- Downsample: 13-tap Jimenez filter + Karis average on first pass (suppresses fireflies)
- Upsample: 3×3 tent filter, additive blend between levels
- 5 levels starting at half-res (960×540 for 1080p)
- Format: RGBA16F throughout

```glsl
// Karis average (first downsample step only)
float weight = 1.0 / (1.0 + luminance(color));
```

**Tone mapping: ACES filmic.** ✅

```glsl
vec3 acesFilmic(vec3 x) {
  return clamp((x * (2.51*x + 0.03)) / (x * (2.43*x + 0.59) + 0.14), 0.0, 1.0);
}
```

**Full-screen triangle** (not quad) for all post-processing passes. ✅ More efficient, no vertex buffer needed.

---

## Assets

**glTF 2.0 only.** ✅ Both `.gltf` and `.glb`.

**Draco mesh compression via Web Worker.** ✅ WASM loaded lazily, user provides URL, decoded data transferred zero-copy.

**KTX2/Basis texture compression via Web Worker.** ✅ Format priority: ASTC > BC7 > ETC2 > RGBA8 (detected at init).

**Worker pool:** 2–4 workers scaled to `hardwareConcurrency`. ✅ ~2× speedup for complex models.

**PBR → Lambert mapping:** `baseColorFactor` → color, `baseColorTexture` → texture, `emissiveFactor` → emissive. Metallic/roughness/normal maps ignored. `KHR_materials_unlit` → BasicMaterial. ✅

**Y-up → Z-up baked at load time.** ✅ Applied to vertices, normals, animation keyframes, bone poses. No per-frame conversion.

**URL-based Promise caching.** ✅ Same URL returns cached Promise. Results can be cloned for multiple instances sharing GPU resources.

---

## Animation

**Flat bone array with parent indices.** ✅ Bones are regular scene nodes.

**GPU skinning, 4 bones per vertex.** ✅

```glsl
mat4 boneMatrix = a_weights.x * u_boneMatrices[a_joints.x]
               + a_weights.y * u_boneMatrices[a_joints.y]
               + a_weights.z * u_boneMatrices[a_joints.z]
               + a_weights.w * u_boneMatrices[a_joints.w];
```

**Bone matrices:** Full mat4 in UBO (128 bones × 64B = 8KB). RGBA32F texture fallback for >128 bones. ✅

**Animation mixer:** Action-based API (Three.js style). *(6/9)*

```typescript
const mixer = createAnimationMixer(skeleton)
const idle = mixer.clipAction(idleClip)
idle.crossFadeTo(walkClip, 0.3)  // 300ms blend
mixer.update(deltaTime)          // call each frame
```

**Keyframe interpolation:** Binary search with cached last index (O(1) for sequential, O(log n) for seek). Linear lerp / slerp for rotation / cubic spline (glTF). ✅

Zero allocations during `mixer.update()`. ~0.04ms per 60-bone character.

---

## Raycasting / BVH

**Binned SAH BVH, 12 bins, max 4 triangles/leaf.** ✅ O(n log n) build, near-optimal trees.

**32-byte flat node layout** (contiguous `Float32Array`/`Int32Array` aliased views). ✅ Cache-friendly, no pointer chasing.

**Stack-based iterative traversal with near-first ordering.** ✅ Push far child first, process near child first for early termination.

**Two-level BVH:** scene-level (mesh AABBs) + per-mesh (triangle data). ✅

**Lazy construction** on first raycast, then cached. Scene BVH rebuilt on scene structure changes. ✅ Optional Web Worker offload.

Ray-AABB: slab method. Ray-triangle: Möller-Trumbore. ✅

Performance: ~0.02–0.06ms per ray. Build time: ~2–5ms for 100K triangles (one-time).

---

## Controls

**Orbit controls only.** ✅ Orbit / pan / zoom.

**Elevation convention:** 0 = horizontal, +π/2 = looking down. More intuitive than polar-from-Z. ✅

**Exponential decay damping** (frame-rate normalized via `deltaTime * 60`). ✅

**Screen-space pan scaled by distance** — consistent feel regardless of zoom level. ✅

```typescript
controls.setTarget([10, 0, 5], { animate: true, duration: 0.5 })
controls.setPosition([20, -15, 10], { animate: true, duration: 0.5 })
```

---

## HTML Overlay

**Absolute-positioned container div + CSS transforms.** ✅ `will-change: transform` for GPU layer promotion.

**Standard world-to-screen projection** per element each frame. ✅ Hide when behind camera or outside frustum.

**Modes:** static world position or node-tracking with offset. ✅

**Occlusion:** opt-in raycast, staggered every 3 frames (N/3 rays per frame). *(decisions-3 approach)*

**Dirty check:** skip DOM write if position change <0.5px. *(decisions-3 approach)*

Depth-based z-index for proper stacking. Pointer events opt-in per element (container has `pointer-events: none`). ✅

---

## React Bindings (`engine-react`)

**Custom `react-reconciler` host.** ✅ Maps JSX tree to scene graph. Render loop runs via `requestAnimationFrame` outside React — 60fps never blocked by React reconciliation.

```tsx
<Canvas shadows bloom camera={{ position: [5, -10, 5], fov: 60 }}>
  <directionalLight position={[10, -10, 10]} castShadow />
  <mesh castShadow onClick={(e) => console.log(e.point)}>
    <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
    <lambertMaterial color={[0.8, 0.2, 0.2]} />
  </mesh>
</Canvas>
```

Child geometry/material in JSX attaches to parent mesh rather than entering scene graph. ✅

**Hooks:** `useFrame(dt => ...)`, `useEngine()`, `useLoader()`, `useGLTF()`, `useAnimations()`. ✅

**Pointer events** on meshes via raycasting: `onClick`, `onPointerOver`, `onPointerOut`. ✅

**Automatic disposal** on unmount. `<primitive object={gltf.scene} />` for pre-built graphs. `<Html>` via `ReactDOM.createPortal`. ✅

Bundle: ~25–35KB minified.

---

## Out of Scope for v1

Point/spot lights, PBR materials, morph targets, physics, text, particles, post-processing beyond bloom+tone mapping, PCSS/VSM shadows, multiple shadow-casting lights.
