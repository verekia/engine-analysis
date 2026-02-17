# Performance Strategy Comparison

This document compares performance strategies across all 9 engine implementations: bonobo, caracal, fennec, hyena, lynx, mantis, rabbit, shark, and wren.

## Universal Agreement

All implementations agree on the following core performance principles:

### Shared Performance Target
**2000 draw calls at 60fps on mobile** is the universal target (16.6ms frame budget), mentioned consistently across 8 of 9 implementations.

### Draw Call Sorting
All implementations use sort-based rendering to minimize GPU state changes:
- Sort by pipeline/shader first (most expensive)
- Then by material/texture (moderate cost)
- Then by geometry (cheapest)
- Depth sorting (front-to-back for opaque, enables early-Z rejection)

### Frustum Culling
All implementations perform frustum culling to skip invisible objects:
- Extract 6 planes from view-projection matrix
- Test AABBs or bounding spheres against planes
- Typically culls 30-70% of objects

### Zero Per-Frame Allocation
All implementations avoid GC pressure by:
- Pre-allocating typed arrays at startup
- Using module-level scratch/pool variables
- Never allocating in the render loop
- Operating on Float32Array/Uint32Array directly

### Dirty Flag System
All implementations track which objects changed:
- Only recompute world matrices for dirty objects
- Typical: 2-5% of objects move per frame
- Dramatically reduces matrix computation cost

### WebGPU First
All implementations prioritize WebGPU with WebGL2 fallback:
- Lower CPU overhead per draw call
- Command buffers for batch submission
- Bind groups reduce state setting

## Key Variations

### 1. Instance Buffer Strategy

**Approach A: Automatic Instancing (1 implementation)**
- **Bonobo** automatically groups consecutive entities with same geometry+material into instanced draws
- Per-instance data written to dynamic buffer (96 bytes/instance)
- Forest of 1000 trees = 1 draw call instead of 1000

**Approach B: No Instancing (8 implementations)**
- **Fennec, Hyena, Lynx, Mantis, Rabbit, Wren, Caracal, Shark** intentionally skip instancing
- Rationale: Per-draw overhead optimized to <2-3μs makes 2000 unique draws feasible
- Instancing adds complexity (instance buffers, material limits) without meaningful benefit

### 2. Sort Algorithm

**Radix Sort (6 implementations)**
- **Caracal, Fennec, Hyena, Mantis, Rabbit, Wren** use radix sort
- O(n) time complexity vs O(n log n)
- For 2000 keys: ~0.05-0.1ms
- Multiple passes on 8-bit buckets (typically 2-4 passes for 32-64 bit keys)

**Standard Sort (2 implementations)**
- **Bonobo, Lynx** use `Array.prototype.sort` on numeric keys
- Still fast enough for 2000 items (<0.1ms)
- Simpler implementation

**Unspecified (1 implementation)**
- **Shark** mentions sorting but doesn't specify algorithm

### 3. Uniform Upload Strategy

**Approach A: UBO with Dynamic Offsets (6 implementations)**
- **Caracal, Hyena, Lynx, Mantis, Rabbit, Wren** use large uniform buffer with dynamic offsets
- Single buffer upload per frame, offset changes per draw
- 128-256 bytes per object (256 for alignment)
- Ring buffer on WebGL2 to avoid stalls

**Approach B: Per-Instance Buffer (1 implementation)**
- **Bonobo** writes per-instance data (world matrix, color, flags) to instance buffer
- 96 bytes per instance
- Used in conjunction with automatic instancing

**Approach C: Unspecified (2 implementations)**
- **Fennec, Shark** don't specify uniform upload details

### 4. WebGPU Render Bundles

**Explicitly Used (5 implementations)**
- **Caracal, Hyena, Lynx, Mantis, Rabbit** pre-record static geometry into render bundles
- Near-zero JS cost for replaying bundles
- Invalidated only when static objects change
- Typical split: 1800 static + 200 dynamic objects

**Not Mentioned (4 implementations)**
- **Bonobo, Fennec, Shark, Wren** don't explicitly mention render bundles

### 5. Data Layout

**Structure of Arrays (SoA) - 5 implementations**
- **Bonobo, Hyena, Lynx, Mantis, Wren** use flat typed arrays for transforms
- Example: `positions: Float32Array(MAX * 3)`, `rotations: Float32Array(MAX * 3)`
- Better cache locality for batch operations
- Zero GC overhead

**Object-Oriented with Typed Array Backing (3 implementations)**
- **Fennec, Caracal, Rabbit** use classes/objects but backed by typed arrays
- Wrapper provides friendly API while maintaining performance
- Typed array data can still be uploaded directly to GPU

**Unspecified (1 implementation)**
- **Shark** doesn't detail data layout

### 6. Render List Persistence

**Persistent, Updated Incrementally (1 implementation)**
- **Hyena** keeps render list between frames
- Only re-sorts when objects/materials change
- O(1) to O(log n) typical cost vs O(n log n) rebuild

**Rebuilt Every Frame (7 implementations)**
- **Bonobo, Caracal, Fennec, Lynx, Mantis, Rabbit, Wren** rebuild and sort each frame
- Still fast enough (0.05-0.2ms for 2000 objects)
- Simpler implementation

**Unspecified (1 implementation)**
- **Shark** doesn't specify

### 7. Shadow Map Optimization

**Common Techniques (6 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Rabbit, Wren** all mention:
  - Depth-only shader (minimal fragment work)
  - Per-cascade culling
  - Front-face culling to reduce acne
  - Shadow atlas (single texture for all cascades)

**Basic Mention (2 implementations)**
- **Bonobo, Mantis** mention shadows but less detail

**Not Specified (1 implementation)**
- **Shark** defers to other docs

## Implementation Breakdown

### Frame Budget Distribution (where specified)

**Typical CPU Budget for 2000 draws:**
- JS overhead: 3-6ms total
  - Matrix updates: 0.2-0.5ms (dirty only)
  - Frustum culling: 0.1-0.2ms
  - Sorting: 0.05-0.2ms
  - Draw submission: 3-4ms (1.5-3μs per draw)
  - Misc: 0.5-1ms

**Typical GPU Budget:**
- Shadow maps: 1.5-3ms
- Opaque geometry: 3-5ms
- Transparency (OIT): 0.5-1ms
- Post-processing: 1-2ms
- MSAA resolve: 0.2-0.5ms

### Per-Draw JS Overhead Targets

| Implementation | Target | Strategy |
|---------------|--------|----------|
| Bonobo | ~5μs | Instancing + sorting |
| Caracal | <3μs | State cache + sorting |
| Fennec | <2μs | State sorting + no instancing |
| Hyena | ~2-3μs | Zero allocation + sorting |
| Lynx | ~2-3μs | Dynamic UBO offsets |
| Mantis | ~1.5μs | Ring buffer + radix sort |
| Rabbit | ~5μs | State cache + sorting |
| Wren | ~1.5μs | Command buffer + radix sort |
| Shark | Not specified | - |

### State Change Reduction

All implementations report dramatic reductions:
- **Without sorting**: ~2000 state changes (worst case)
- **With sorting**: ~10-200 state changes
- **Reduction**: 90-99%

Typical breakdown with sorting:
- Pipeline switches: 4-10
- Material switches: 8-200
- Texture switches: 12-500
- Geometry switches: 50-500
- Dynamic offset only: 1500-1900

### Mobile-Specific Optimizations

**Tile-Based GPU Awareness (4 implementations)**
- **Caracal, Hyena, Mantis, Wren** explicitly mention tile-based GPU optimizations:
  - MSAA nearly free on mobile (on-chip resolve)
  - Minimize render target switches (forces tile flush)
  - Half-resolution bloom (reduces bandwidth)

**General Mobile Mentions (5 implementations)**
- **Bonobo, Fennec, Lynx, Rabbit, Shark** mention mobile but less GPU-specific detail

### Memory Budget Estimates

Where specified, typical budgets for 2000 objects:
- **Vertex buffers**: 100-200MB
- **Index buffers**: 20-40MB
- **Textures**: 50-200MB
- **Shadow maps**: 30-50MB
- **Uniform/Instance buffers**: 4-10MB
- **Total GPU memory**: 300-500MB

## Profiling & Instrumentation

**Detailed Stats APIs (7 implementations)**
- **Bonobo, Caracal, Fennec, Hyena, Lynx, Mantis, Wren** provide frame stats:
  - Draw calls, triangles, state changes
  - CPU time, GPU time (when available)
  - Culled vs visible objects
  - Per-phase timing

**Basic Stats (1 implementation)**
- **Rabbit** mentions stats but less detail

**Not Specified (1 implementation)**
- **Shark** defers to other docs

## Advanced Techniques

### BVH for Frustum Culling

**BVH-Accelerated (4 implementations)**
- **Fennec, Hyena, Lynx, Mantis** use BVH for O(log n) culling
- Reduces 12,000 plane tests to ~200-400 typical
- 6× faster than brute force

**Brute Force (5 implementations)**
- **Bonobo, Caracal, Rabbit, Shark, Wren** test all objects
- Still fast enough: ~0.1-0.2ms for 2000 objects
- Simpler implementation

### Shader Variant Cache

**Explicit Pre-compilation (5 implementations)**
- **Caracal, Fennec, Hyena, Mantis, Rabbit** pre-compile shader variants
- Avoids frame drops from on-demand compilation
- Pre-warm common variants during loading

**Lazy Compilation (2 implementations)**
- **Lynx, Wren** compile on first use but cache

**Not Specified (2 implementations)**
- **Bonobo, Shark** don't detail shader compilation strategy

## Comparison to Three.js

All implementations that mention Three.js report similar issues:
- **Per-draw overhead**: Three.js ~15-50μs vs these engines ~1.5-5μs (3-30× improvement)
- **GC pressure**: High in Three.js vs zero in these engines
- **Matrix updates**: All objects every frame vs dirty-only (40× improvement typical)
- **No state sorting**: Causes excessive state changes
- **Individual uniform calls**: vs UBOs/batched uploads

## Summary Table

| Feature | Bonobo | Caracal | Fennec | Hyena | Lynx | Mantis | Rabbit | Shark | Wren |
|---------|--------|---------|--------|-------|------|--------|--------|-------|------|
| **Target** | 2000@60 | 2000@60 | 2000@60 | 2000@60 | 2000@60 | 2000@60 | 2000@60 | 2000@60 | 2000@60 |
| **Instancing** | Auto | No | No | No | No | No | No | - | No |
| **Sort Algo** | Array.sort | Radix | Radix | Radix | Array.sort | Radix | Radix | - | Radix |
| **SoA Layout** | Yes | Partial | Partial | Yes | Yes | Yes | Partial | - | Yes |
| **Render Bundles** | - | Yes | - | Yes | Yes | Yes | Yes | - | - |
| **UBO Strategy** | Instance | Dynamic | - | Dynamic | Dynamic | Ring | Dynamic | - | Dynamic |
| **BVH Culling** | No | No | Yes | Yes | Yes | Yes | No | - | No |
| **Per-Draw Target** | ~5μs | <3μs | <2μs | 2-3μs | 2-3μs | ~1.5μs | ~5μs | - | ~1.5μs |

## Recommendations for Cherry-Picking

**For Maximum Performance:**
- Use **radix sort** (Caracal, Fennec, Hyena, Mantis, Rabbit, Wren approach)
- Implement **render bundles** for static geometry (Caracal, Hyena, Lynx, Mantis, Rabbit)
- Use **ring buffer UBO** with dynamic offsets (Mantis, Wren approach)
- Target **<2μs per draw** in JS (Fennec, Hyena, Lynx, Mantis, Wren)

**For Simplicity:**
- Skip instancing (8/9 implementations agree it's not needed)
- Use `Array.sort` if <3000 draws (Bonobo, Lynx approach)
- Brute-force frustum culling is fine for <5000 objects (5/9 implementations)

**For Mobile:**
- Explicitly optimize for tile-based GPUs (Caracal, Hyena, Mantis, Wren approach)
- Half-resolution bloom
- MSAA with proper load/store ops
- Minimize render target switches

**For Debugging:**
- Implement detailed stats API (7/9 implementations provide this)
- Track state changes, culling results, per-phase timing
- Enable GPU timestamp queries (WebGPU) or timer extension (WebGL2)
