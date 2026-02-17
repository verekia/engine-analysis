# ARCHITECTURE.md - Decisions

## Decision: OOP Public API with SoA Internals, Monolithic Package

### Module Organization: Monolithic Single Package

**Chosen**: Monolithic single package (Hyena/Shark/Wren style, 5/9 implementations)
**Rejected**: Multi-package monorepo (Fennec/Caracal/Lynx/Mantis)

Rationale: the engine is deliberately minimal - two materials, one light type, seven primitives. There isn't enough surface area to justify the build complexity of a multi-package monorepo. A single package with clear internal module boundaries achieves the same separation of concerns without the overhead of multiple package.json files and cross-package versioning.

React bindings are the one exception - they export from a subpath (`engine/react`) because they carry a React peer dependency.

### Data Storage: Object-Oriented with SoA Internals

**Chosen**: OOP wrappers for the public API, flat typed arrays for the renderer (6/9 implementations: Caracal, Fennec, Hyena, Lynx, Rabbit, Wren)
**Rejected**: Pure SoA everywhere (Bonobo), pure OOP throughout

```typescript
// Public API: familiar objects
const mesh = createMesh(geometry, material)
mesh.position.set(1, 2, 3)
scene.add(mesh)

// Internal: renderer reads flat arrays directly
// worldMatrices: Float32Array - contiguous, GPU-uploadable
```

This gives users a familiar Three.js-like API while the renderer operates on cache-friendly contiguous arrays. The two representations are synchronized via dirty flags - objects mark themselves dirty when properties change, and the renderer reads the flat arrays after the update pass.

### Layered Architecture

One-way dependency flow (universal agreement across all 9):

```
React Bindings / HTML Overlay (leaf modules)
    |
High-Level API: Controls, Animation, Asset Loading
    |
Rendering Pipeline: Renderer, Materials, Shadows, Post-Processing
    |
GPU Abstraction Layer (GAL)
    |
Math Library (zero dependencies)
```

Lower layers never import from upper layers.

### GPU Abstraction: WebGPU-Modeled

**Chosen**: Abstraction modeled after WebGPU concepts, named "GAL" (GPU Abstraction Layer, 4/9 use this name - most common)

All 9 implementations agree the abstraction should follow WebGPU semantics:
- Immutable pipelines (full render state baked at creation)
- Explicit bind groups (resources grouped by update frequency)
- Command recording with explicit begin/end passes

WebGL2 is the translation layer that emulates these concepts via its state machine.

### Frame Lifecycle

All 9 implementations converge on this sequence:

```
1. Time/clock update
2. User callbacks (onFrame / useFrame)
3. Animation system update
4. Dirty flag propagation + world matrix recomputation
5. Frustum culling
6. Build/sort render list
7. Upload per-frame uniforms (camera, lights, shadows)
8. Render passes:
   a. Shadow maps (3 CSM cascades, depth-only)
   b. Opaque geometry (sorted, MSAA)
   c. Transparent geometry (OIT accumulation)
   d. MSAA resolve
   e. OIT composite
   f. Bloom (threshold -> downsample -> upsample)
   g. Final blit (tone mapping -> screen)
9. HTML overlay update
10. Reset scratch pools
```

### Scene Graph Update: Depth-First with Dirty Flags

**Chosen**: Depth-first recursive traversal with two-level dirty flags (Caracal/Fennec/Shark approach)
**Rejected**: Breadth-first (Mantis/Hyena) - more complex for marginal benefit at engine's target scale

Two-level flags:
- `_dirtyLocal`: local matrix needs recompute (position/rotation/scale changed)
- `_dirtyWorld`: world matrix needs recompute (self or ancestor changed)

Early exit: if a node is already marked `_dirtyWorld`, skip propagation to its subtree (prevents O(n^2) cascading).

### Render List: Rebuild Every Frame

**Chosen**: Rebuild and sort each frame (5/9: Bonobo, Fennec, Hyena, Rabbit, Wren)
**Rejected**: Persistent sorted list (4/9: Caracal, Lynx, Mantis, Shark)

For 2000 objects, a full rebuild + radix sort completes in <0.2ms. The persistent list approach adds state management complexity (insert/remove/visibility tracking) for negligible performance gain at this scale. The simpler approach is easier to reason about and debug.

### Uniform Buffer Strategy: Dynamic Offsets with Per-Frame Upload

**Chosen**: Single large UBO per bind group frequency, dynamic offsets per object (universal agreement for per-object data), per-frame upload (6/9 implementations)
**Considered**: Ring buffer (Mantis/Shark/Wren) - adds complexity for a benefit that only matters with >3 frames in flight, which is uncommon in browser contexts

Three bind groups by update frequency:

| Slot | Name | Update Frequency | Contents |
|------|------|-----------------|----------|
| 0 | Per-frame | Once per frame | Camera VP, lights, shadow matrices |
| 1 | Per-material | Per material change | Textures, palette, material params |
| 2 | Per-object | Per draw call (offset only) | World matrix, bone matrices |

### Memory Management

- Pre-allocated scratch variables at module scope for math temporaries
- Frame-scoped pool with reset for intermediate allocations (Mantis/Lynx approach)
- Zero `new` calls in the render loop
- GPU resources live until explicitly destroyed via `dispose()`
- Geometric buffer growth (double capacity) for dynamic arrays (Mantis/Shark)

### Threading Model

Universal agreement across all 9:
- **Main thread**: Scene graph, render loop, GPU command submission
- **Web Workers**: Draco mesh decoding, Basis texture transcoding, optionally BVH construction
- **Communication**: `postMessage` with `Transferable` typed arrays (zero-copy)
