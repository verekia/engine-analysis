# Decisions Summary

Summary of all decisions made across 9 implementation proposals (Bonobo, Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren). Consensus level noted where relevant.

---

## 1. PLAN — Overall Scope & Philosophy

- **Ship all features in v1** (8/9). No phased rollout — shadows, animation, glTF, React bindings, HTML overlay are all v1.
- **Performance is architecture**: every decision traces back to **2000 draw calls @ 60fps on mobile**.
- **No automatic instancing** (8/9). At <2-3us per draw, 2000 unique draws are feasible. Instancing adds unjustified complexity at this scale.

### Out of scope for v1
Point/spot lights, PBR, morph targets, physics, text, particles, PCSS/VSM/ESM shadows, multiple shadow casters.

---

## 2. ARCHITECTURE

- **Strict layered architecture** (universal). One-way deps: Math -> Device -> Scene/Materials -> Renderer -> Animation/Loaders -> Controls/Overlay -> React.
- **Monolithic single package** (5/9) with internal module folders. React bindings in separate `engine-react` package.
- **OOP public API with SoA internals** (6/9). Users interact with objects (`mesh.position.set()`), renderer iterates flat typed arrays (`Float32Array`).
- **Deferred dirty propagation** — setting a property is O(1), world matrices recomputed once per frame for dirty nodes only.
- **Render list rebuilt every frame** (5/9). Cull + radix sort < 0.3ms for 2000 objects.
- **3 bind groups by update frequency**: per-frame (camera/lights), per-material (textures/palette), per-object (dynamic offsets).
- **GPU abstraction named "Device"**, modeled after WebGPU. WebGL2 is a translation layer, not the design target.
- **Zero allocations in render loop** — pre-allocated pools, no `new`, no closures on hot paths.
- **Workers** for Draco/Basis decoding. 2-4 workers scaled to `hardwareConcurrency`. Render loop stays on main thread.

### Frame lifecycle (universal)

```
1. Time update -> 2. User callbacks -> 3. Animation update -> 4. Dirty propagation
-> 5. World matrices -> 6. Frustum cull -> 7. Build draw list -> 8. Radix sort
-> 9. Upload uniforms -> 10. Shadow pass (3 CSM) -> 11. Opaque pass (MSAA, MRT)
-> 12. Transparent pass (OIT) -> 13. MSAA resolve -> 14. OIT composite
-> 15. Bloom -> 16. Tone map + blit -> 17. HTML overlay update -> 18. Reset pools
```

---

## 3. RENDERER

- **WebGPU-first, WebGL2 fallback** (universal). Auto-detect via `navigator.gpu.requestAdapter()`.
- **64-bit radix sort** for draw calls (5/9). Key layout:

```
Bits 63-62: Layer | 61-60: Pass | 59: Transparent | 58-48: Pipeline ID | 47-32: Material ID | 31-0: Depth
```

Pipeline switches are most expensive -> highest bits. 90-99% state change reduction.

- **Dual-source WGSL + GLSL** (universal). No runtime transpilation. Shader variants via `#ifdef` feature flags, compiled lazily and cached by bitmask. Pre-warm 5 common variants during loading.
- **4x MSAA by default** (universal). Resolved before post-processing. Nearly free on tile-based mobile GPUs.
- **WebGL2 backend**: full state cache (8/9), pipeline = GL state bundle, VAO caching per geometry+pipeline.
- **WebGPU backend**: render bundles for static geometry (6/9, not primary optimization), pipeline caching by descriptor hash.
- **Per-draw overhead target**: <2-3us JS. 2000 draws = ~4-6ms JS, leaving ~10ms for GPU.

---

## 4. SCENE GRAPH

- **Object graph with contiguous matrix storage** (6/9). Nodes have `.parent`, `.children`. World matrices packed in `Float32Array(nodeCount * 16)`.
- **Node types**: Group, Mesh, SkinnedMesh, Camera, DirectionalLight, AmbientLight.
- **Transforms**: Position (Vec3) + Rotation (Quat) + Scale (Vec3) (universal). Quaternions avoid gimbal lock.
- **Two-level dirty flags**: `_dirtyLocal` + `_dirtyWorld` with early-exit propagation. Typical frame: 95-98% of matrix work skipped.
- **Frustum culling**: AABB P-vertex/N-vertex test, 6 planes, ~0.1-0.2ms for 2000 objects. Hierarchical culling (skip children if parent outside).
- **Bones are regular scene nodes** (6/9). Bone attachment = standard `handBone.add(swordMesh)`, no special API.

---

## 5. MATERIALS

- **Two materials**: BasicMaterial (unlit) and LambertMaterial (diffuse) (universal).
- **32-entry material palette via UBO** (7/9). Per-vertex `_materialindex` attribute selects palette entry. Each entry has color, opacity, emissive, emissiveIntensity. Enables glowing eyes, colored armor parts, etc. in a single draw call.

```ts
const mat = createLambertMaterial({
  palette: [
    { color: [0.9, 0.7, 0.6] },                                       // 0: skin
    { color: [1, 0.8, 0.2], emissive: [1, 0.8, 0.2], emissiveIntensity: 2.0 }, // 1: glow
  ],
})
```

- **Vertex colors**: multiplicative blending with base color (universal).
- **Emissive via MRT**: separate emissive render target feeds bloom directly. No global threshold pass needed (universal).
- **1x1 white default textures** (universal). Avoids shader branching for textured vs untextured paths.
- **Shader variants**: feature flag bitmask (`HAS_COLOR_TEXTURE`, `HAS_SKINNING`, etc.), lazy compilation, cache by bitmask. ~10-30 variants per scene.

---

## 6. GEOMETRY

- **Separate buffers per attribute** (universal). No interleaving — GPU only fetches used attributes.
- **Auto Uint16/Uint32 index selection** based on vertex count (universal).
- **AABB as primary bounding volume** (universal).
- **7 parametric primitives** (universal): plane, box, sphere, cone, cylinder, capsule, circle. All Z-up oriented, centered at origin.
- **Lazy GPU upload** on first render (universal). `geometry.needsUpdate = true` for re-upload.
- **Lazy BVH construction** on first raycast (universal).
- **Geometry sharing** via reference counting.

---

## 7. ANIMATION

- **Flat bone array with parent indices + inverse bind matrices** (universal). Bones are scene nodes.
- **GPU vertex shader skinning**, 4 bones per vertex (universal). `mat4` in UBO for <=128 bones, RGBA32F texture fallback for more.
- **Named clips with per-bone keyframe tracks**. Interpolation: linear (lerp/slerp), step, cubic spline.
- **Binary search with cached last keyframe** — O(1) for sequential playback, O(log n) for seeks.
- **Action-based mixer** (6/9):

```ts
const mixer = createAnimationMixer(skeleton)
const idle = mixer.clipAction(idleClip)
idle.play()
idle.crossFadeTo(walk, 0.3) // 300ms crossfade
walk.loop = 'repeat'
walk.timeScale = 1.5
```

- **Crossfade**: source weight -> 0, target weight -> 1 over duration. Normalized blending with slerp for rotations.
- **Y-up to Z-up conversion baked at import time** (universal). No per-frame overhead.
- **~0.04ms per animated character** (60-bone skeleton).

---

## 8. ASSETS

- **glTF 2.0 only** (universal). `.gltf` and `.glb`.
- **Draco via Web Worker** (universal). User provides WASM URL, lazily loaded. Zero-copy transfer back via `Transferable`.
- **KTX2/Basis via Web Worker** (universal). Format priority: ASTC > BC7 > ETC2 > RGBA8.
- **Worker pool**: 2-4 workers scaled to `hardwareConcurrency`.
- **PBR -> Lambert mapping**: baseColor -> color, baseColorTexture -> colorTexture, emissiveFactor -> emissive. Metallic/roughness/normal maps ignored. `KHR_materials_unlit` -> BasicMaterial.
- **Y-up -> Z-up baked at load time**: vertex positions `[x, y, z]` -> `[x, z, -y]`. Applied to animations, bones, transforms.
- **`_MATERIALINDEX` custom attribute** recognized automatically.
- **URL-based Promise caching**: same URL returns cached Promise. GLTFResult can be cloned for multiple instances.

---

## 9. LIGHTING & SHADOWS

- **Lambert diffuse (N dot L), no specular** (universal). ~15 ALU ops vs ~80+ for PBR. Emissive is additive, unaffected by lighting.
- **DirectionalLight** = scene node (direction from world rotation). **AmbientLight** = scene property (no spatial data).
- **3-cascade CSM** (universal). 1024x1024 per cascade default, configurable to 2048.
- **Texture array** (`TEXTURE_2D_ARRAY` with 3 layers) for shadow maps. No atlas bleed issues.
- **Lambda 0.7 logarithmic splits**: ~0.1-12m / 12-50m / 50-200m for a 200m world.
- **3x3 Poisson PCF** (9 samples) with hardware comparison samplers.
- **Cascade blending**: smooth transition at cascade boundaries, ~10% blend zone.
- **Texel snapping**: prevents shadow shimmer during camera movement.
- **Depth bias + slope-scaled bias** combined with front-face culling during shadow rendering.
- **Per-cascade frustum culling** + extend light projection backwards ~50-100m for casters behind camera.
- **Total shadow overhead**: ~1.5ms GPU.

---

## 10. POST-PROCESSING

- **Unreal-style bloom** (universal). Driven by MRT emissive output (no threshold pass).
  - **Downsample**: 13-tap Jimenez filter, Karis average on first pass (prevents fireflies).
  - **Upsample**: 3x3 tent filter (9-tap), additive blending.
  - **5 levels** starting at half resolution.
  - **RGBA16F** render targets for HDR precision.
- **ACES filmic tone mapping** (universal). Applied after bloom composite.
- **Full-screen triangle** for all full-screen passes (more efficient than a quad).
- **MSAA resolved before post-processing** (universal).
- **Total post-processing**: ~0.6-1.1ms GPU.

---

## 11. TRANSPARENCY

- **Weighted Blended OIT (WBOIT)** (universal). Single pass, no sorting required.
- **McGuire Equation 10** weight function: depth-based, `/200.0` tuned for 200m world range.
- **Two render targets**: RGBA16F accumulation + R8 revealage.
- **Depth test ON, depth write OFF** in transparent pass.
- **Composite pass**: blend OIT result over resolved opaque image.
- **Known limitations**: subtle ordering errors at similar depths, saturation above ~8 overlapping layers. Acceptable for stylized/low-poly.
- **WebGL2 requires** `EXT_color_buffer_float` + `EXT_float_blend`. Fallback to sorted alpha blending if unavailable.
- **Total OIT overhead**: ~0.4-0.65ms GPU.

---

## 12. RAYCASTING & BVH

- **Binned SAH with 12 bins** (universal). O(n log n) build, near-optimal trees. Max 4 triangles per leaf.
- **32-byte flat array node layout** (universal). Stored in contiguous `Float32Array`/`Int32Array`. No pointer chasing.
- **Stack-based iterative traversal with near-first ordering** (universal). Reduces average traversal by testing closer child first.
- **Slab method** for ray-AABB, **Moller-Trumbore** for ray-triangle (universal).
- **Two-level BVH**: scene BVH (mesh AABBs) + per-mesh BVH (triangles).
- **Lazy construction**: built on first raycast, then cached. Scene BVH rebuilt on structure changes.

```ts
const raycaster = createRaycaster()
raycaster.setFromCamera([ndcX, ndcY], camera)
const hits = raycaster.intersectObjects(scene.children, true) // -> RaycastHit[]
```

- **~0.02-0.06ms per ray** (2000 objects). BVH build: ~2-5ms one-time for 100K triangles.

---

## 13. CONTROLS

- **Orbit controls only** (universal). Orbit (rotate), pan (move target), zoom (dolly).
- **Elevation from XY plane** (universal). 0 = horizontal, +pi/2 = looking down. Z-up aware spherical coords.
- **Velocity-based damping with exponential decay** (universal). `dampingFactor: 0.1` default, frame-rate normalized.
- **Mouse**: left drag = orbit, right/middle drag = pan, scroll = zoom. **Touch**: 1-finger = orbit, 2-finger drag = pan, pinch = zoom.
- **Pan scaled by distance** — consistent feel at any zoom level.
- **Animated transitions**: `setTarget([x,y,z], { animate: true, duration: 0.5 })`.

---

## 14. REACT BINDINGS

- **Custom React reconciler via `react-reconciler`** (universal). Separate `engine-react` package.
- **`<Canvas>` root component** creates canvas, engine, render loop, provides context.
- **Lowercase JSX elements** map to engine types: `<mesh>`, `<group>`, `<boxGeometry>`, `<lambertMaterial>`, etc.
- **Geometry/material as children of mesh**: reconciler attaches them to parent mesh, not scene graph.

```tsx
<Canvas shadows bloom camera={{ position: [5, -10, 5], fov: 60 }}>
  <ambientLight intensity={0.3} />
  <directionalLight position={[10, -10, 10]} castShadow />
  <mesh position={[0, 0, 0.5]} castShadow>
    <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
    <lambertMaterial color={[0.8, 0.2, 0.2]} />
  </mesh>
</Canvas>
```

- **Hooks**: `useFrame(cb)`, `useEngine()`, `useLoader()`, `useGLTF()`, `useAnimations()`.
- **Pointer events** via raycasting: `onClick`, `onPointerOver`, `onPointerOut` on meshes.
- **`<primitive object={gltf.scene} />`** for pre-built scene graphs.
- **`<Html position={...} center occlude>`** for DOM overlays.
- **Automatic disposal on unmount**. Render loop runs outside React via rAF (never blocked by reconciliation).
- **~25-35KB** minified.

---

## 15. HTML OVERLAY

- **Absolute-positioned overlay div** above canvas. Elements positioned via CSS `transform: translate()` (GPU-accelerated) (universal).
- **World-to-screen projection**: view-projection matrix -> NDC -> screen pixels. Hidden if behind camera or outside frustum.
- **Node tracking**: overlay follows scene node + optional offset.
- **Occlusion**: raycast-based, opt-in. **Staggered every 3 frames** to spread cost.
- **Distance scaling** (optional): CSS `scale` adjusted by camera distance.
- **Auto z-index** from projected depth.
- **Pointer events opt-in** per element (container is `pointer-events: none`).
- **Dirty checking**: skip DOM update if position change < 0.5px.
- **Total overhead**: ~0.3ms for 20 overlays.

---

## 16. PERFORMANCE

### Top optimizations ranked by impact

1. **Draw call sorting** (64-bit radix sort) — 90-99% state change reduction, ~9ms saved on mobile.
2. **Frustum culling** — 30-70% of objects invisible per frame.
3. **WebGPU-first** — 30-50% cheaper draw submission than WebGL2.
4. **Dirty flags** — skip 95%+ of matrix work per frame.
5. **WebGL2 state cache** — eliminates 40-60% of redundant GL calls.
6. **Zero per-frame allocations** — no GC collections from render loop.
7. **Shader warm-up** — pre-compile 5 common variants during loading.

### Frame budget

| | Budget |
|---|---|
| JS total | ~6ms (matrix updates, cull, sort, draw submission, animation) |
| GPU total | ~10ms (shadows 1.5ms, opaque 3-5ms, MSAA 0.3ms, OIT 0.4ms, bloom 0.7ms) |
| **Effective frame** | **max(JS, GPU) = 6-10ms** (JS and GPU overlap) |

### Memory budget: <400MB GPU total

---

## 17. MATH

- **Z-up, right-handed** (universal): X=right, Y=forward, Z=up.
- **All types are `Float32Array` aliases**: `Vec3`, `Vec4`, `Quat`, `Mat4`, `AABB`. No classes. Direct GPU upload, pooling, sub-array views.
- **Column-major matrices** (universal, matches WebGL/WebGPU).
- **Quaternion [x,y,z,w]** order (matches glTF).
- **Pure functional API, output parameter first**: `vec3Add(out, a, b)`. Zero allocations.
- **Scratch variables**: module-level `_tempVec3`, `_tempMat4`, plus frame-scoped pool with pointer reset.
- **ZYX Euler rotation order** (universal).
- **Depth range**: [0,1] for WebGPU, [-1,1] for WebGL2. Projection matrix adjusts per backend.
- **Custom library, no deps** (~5-8KB minified). Z-up baked into `lookAt`, spherical coords, etc.
- **Frozen constants**: `VEC3_UP = [0,0,1]`, `VEC3_FORWARD = [0,1,0]`.

---

## 18. API

- **TypeScript strict, no semicolons, const arrow functions** (universal).
- **`createEngine(canvas, options)` async factory** (4/9). Single entry point, auto WebGPU/WebGL2.

```ts
const engine = await createEngine(canvas, {
  shadows: true,        // boolean or config object
  bloom: { intensity: 0.5 },
  toneMapping: 'aces',
})
```

- **Factory functions for everything**: `createScene()`, `createMesh()`, `createBoxGeometry()`, `createLambertMaterial()`, `createPerspectiveCamera()`, `createDirectionalLight()`, `createAnimationMixer()`, `createRaycaster()`, `createOrbitControls()`, `createOverlayManager()`.
- **Scene is a separate object** (7/9). Supports multiple scenes.
- **Options objects with defaults** for all configuration. Boolean shorthand: `shadows: true` = sensible defaults.
- **Render loop**: built-in (`engine.onFrame(cb)` + `engine.start()`) or manual (`requestAnimationFrame` + `engine.render()`).
- **Frame statistics**: `engine.getStats()` -> fps, frameTime, drawCalls, triangles, stateChanges, etc.
- **Debug mode**: `{ debug: true }` for verbose validation.
- **Explicit disposal**: `geometry.dispose()`, `material.dispose()`, `gltf.dispose()`, `engine.dispose()`. React handles it automatically.

### Complete example

```ts
const engine = await createEngine(canvas, { shadows: true, bloom: { intensity: 0.5 } })
const scene = createScene()
scene.ambientLight = { color: [0.4, 0.4, 0.5], intensity: 0.3 }

const sun = createDirectionalLight({ color: [1, 1, 0.95], intensity: 1.0, castShadow: true })
sun.position.set(50, 50, 100)
scene.add(sun)

const ground = createMesh(
  createPlaneGeometry({ width: 200, height: 200 }),
  createLambertMaterial({ color: [0.3, 0.5, 0.2], receiveShadow: true })
)
scene.add(ground)

const gltf = await loadGLTF('/character.glb', engine)
scene.add(gltf.scene)

const mixer = createAnimationMixer(gltf.skeletons[0])
mixer.clipAction(gltf.animations[0]).play()

const camera = createPerspectiveCamera({ fov: 60 })
camera.position.set(5, -10, 5)
camera.lookAt([0, 0, 1])

const controls = createOrbitControls(camera, canvas, { target: [0, 0, 1] })

engine.onFrame((dt) => {
  mixer.update(dt)
  controls.update(dt)
})
engine.start(scene, camera)
```
