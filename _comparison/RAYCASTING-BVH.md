# Raycasting & BVH Comparison

This document compares raycasting and BVH (Bounding Volume Hierarchy) approaches across all 9 engine implementations.

## Universal Agreement

All implementations that support raycasting/BVH (8 out of 9) agree on these fundamental principles:

1. **SAH Construction** - All use Surface Area Heuristic for optimal BVH tree quality
2. **Flat Array Layout** - BVH nodes stored in contiguous typed arrays for cache efficiency
3. **Möller-Trumbore** - Standard algorithm for ray-triangle intersection
4. **Slab Method** - Fast AABB-ray intersection using axis-aligned slab tests
5. **Stack-Based Traversal** - Iterative traversal with explicit stack (no recursion)
6. **Lazy Construction** - BVH built on first raycast, cached per geometry
7. **32-Byte Nodes** - Each node occupies 32 bytes (8 floats or mixed float/uint)

## Key Variations

### 1. BVH Implementation Status

**Full BVH Implementation (7)**:
- caracal, fennec, hyena, lynx, mantis, rabbit, shark

**Basic/Deferred (2)**:
- **bonobo** - Marked as "out of scope for v1", plans sphere-based raycasting
- **wren** - Full implementation documented

### 2. SAH Construction Method

**Binned SAH (All 8)**:
All implementations use binned SAH to reduce construction from O(n²) to O(n log n)

**Bin Count Variations**:
- **12 bins** (4): caracal, hyena, mantis, rabbit
- **16 bins** (2): fennec, rabbit (also mentions)
- **32 bins** (2): fennec (configurable), wren
- **Not specified** (1): shark (mentions binned SAH)

**Karis Average for First Downsample** (6):
- caracal, fennec, hyena, lynx, mantis, rabbit
- Prevents "firefly" artifacts from bright pixels dominating
- Weights samples by `1/(1 + luminance)`

### 3. Node Layout (32 Bytes)

All implementations store nodes as 8 floats (32 bytes), but with variations in data packing:

**Standard Layout** (Most implementations):
```
Offset 0-2:  AABB min (x, y, z)
Offset 3:    left child index OR triangle offset
Offset 4-6:  AABB max (x, y, z)
Offset 7:    right child index OR triangle count
```

**Leaf Detection Methods**:

**Negative Triangle Count** (3):
- **fennec**, **lynx**, **mantis** - `rightOrCount < 0` indicates leaf
- Absolute value gives triangle count

**Negative Offset** (2):
- **hyena**, **rabbit** - `leftOrStart < 0` indicates leaf
- Must add 1 to distinguish from 0

**High Bit Flag** (2):
- **shark**, **wren** - `data0 | 0x80000000` for leaf flag
- Mask with `0x7FFFFFFF` to get offset

**Not specified** (1):
- **caracal** - Uses skip pointers for stackless traversal

### 4. Traversal Strategy

**Stack-Based (Standard) (7)**:
- fennec, hyena, lynx, mantis, rabbit, shark, wren
- Pre-allocated Int32Array(64) for traversal stack
- Push children, pop to visit

**Stackless with Skip Pointers (1)**:
- **caracal** - Each node knows where to jump if ray misses
- Right child offset serves as skip target
- More complex construction, simpler traversal

**Stack Size**:
- **64 entries** (6): fennec, hyena, lynx, mantis, rabbit, wren
- Sufficient for millions of triangles
- No overflow checks needed in practice

### 5. Child Ordering During Traversal

**Near-First Ordering (6)**:
- caracal, fennec, hyena, lynx, mantis, rabbit
- Push far child first so near child is popped first
- Better early termination (closestDist shrinks faster)

**Simple Left-Right (1)**:
- **shark** - Push right then left (left visited first)
- Less optimal but simpler

**Not specified** (1):
- **wren** - Uses far-first heuristic mentioned

### 6. Leaf Threshold

**4 Triangles** (6):
- caracal, fennec, hyena, lynx, mantis, rabbit (default)

**Configurable** (Most):
All implementations allow tuning leaf size via `maxLeafSize` or `maxLeafTriangles` parameter

**SAH Cost Check**:
All implementations also create leaf if split cost exceeds leaf cost:
```
if (splitCost >= leafCost) → create leaf
```

## Implementation Breakdown

### BVH Construction Performance

Reported build times (approximate):

| Triangles | caracal | fennec | hyena | lynx | mantis | rabbit | shark | wren |
|---|---|---|---|---|---|---|---|---|
| 1,000 | <1ms | Not spec. | <1ms | Not spec. | <1ms | Not spec. | Not spec. | ~2ms |
| 10,000 | ~3ms | Not spec. | ~5ms | ~3ms | ~5ms | Not spec. | Not spec. | ~15ms |
| 100,000 | ~20ms | ~15ms | ~40ms | ~15ms | ~40ms | Not spec. | ~50ms | ~120ms |
| 1,000,000 | ~150ms | Not spec. | Not spec. | Not spec. | Not spec. | Not spec. | Not spec. | Not spec. |

Build is one-time cost at load time. Can be offloaded to Web Worker (wren, fennec mention).

### Raycast Performance

Reported raycast times (approximate, per ray):

| Implementation | Scene (objects) | Mesh (triangles) | Notes |
|---|---|---|---|
| caracal | ~0.4ms (2000 obj) | ~0.01ms (50K tri) | Stackless traversal |
| fennec | ~0.5ms (2000 obj) | ~0.01ms (50K tri) | With AABB pre-filter |
| hyena | Not specified | ~0.1ms (100K tri) | Single ray estimate |
| lynx | Not specified | ~0.02ms (100K tri) | Closest-hit query |
| mantis | Not specified | ~0.1ms (100K tri) | Single raycast |
| rabbit | Not specified | ~0.01ms (50K tri) | Typical scene |
| shark | Not specified | ~0.01ms (100K tri) | SAH + flat array |
| wren | Not specified | < 5ms (500 rays, 80K tri) | ~0.01ms per ray |

All implementations report logarithmic traversal: ~10-30 nodes visited out of thousands.

### Memory Overhead

All implementations report similar memory usage:

**Per Node**: 32 bytes (8 floats)

**Total BVH Memory**:
- Worst case: `(2 * triangleCount - 1) * 32 bytes`
- For 100K triangles: ~6-7 MB
- Actual usually less due to early leaf termination

**Additional Memory**:
- Reordered index buffer (Uint16Array or Uint32Array)
- Triangle centroids during build (temporary)
- Per-triangle AABBs during build (temporary)

### SAH Cost Function

All implementations use the same fundamental formula:

```
cost = traversalCost +
       (leftArea / parentArea) * leftCount * intersectCost +
       (rightArea / parentArea) * rightCount * intersectCost
```

**Constants Used**:

| Implementation | Traversal Cost | Intersection Cost |
|---|---|---|
| caracal | 1.0 | 1.0 |
| fennec | 1.0 | 1.0 |
| hyena | Not specified | Not specified |
| lynx | 1.0 | 1.5 |
| mantis | 1.0 | 1.5 |
| rabbit | 1.0 | 1.5 |
| shark | 1.0 | 1.5 |
| wren | 1.0 | 1.5 |

Most use 1.5 for intersection cost (triangle test more expensive than AABB test).

### Binned SAH Algorithm

All implementations follow this approach:

```
1. Compute centroid bounds for current node
2. For each axis (X, Y, Z):
   a. Divide centroid range into N equal bins
   b. Assign each triangle to a bin based on centroid position
   c. Compute AABB and count for each bin
   d. Sweep left-to-right, accumulating AABBs and counts
   e. Sweep right-to-left, evaluating SAH cost at each bin boundary
3. Choose axis and split with lowest SAH cost
4. If no split is cheaper than leaf, create leaf
```

### Ray-AABB Intersection (Slab Method)

All implementations use similar slab method:

```glsl
// Pre-compute inverse direction (outside hot loop)
invDir = 1.0 / ray.direction

// Slab test
tMin = (aabb.min - ray.origin) * invDir
tMax = (aabb.max - ray.origin) * invDir

// Swap if needed (handles negative direction)
if (tMin > tMax) swap(tMin, tMax)

// Test all 3 axes
tNear = max(tMinX, tMinY, tMinZ)
tFar = min(tMaxX, tMaxY, tMaxZ)

// Hit if ray enters before exiting and exit is positive
return tFar >= max(tNear, 0) && tNear < closestDist
```

### Ray-Triangle Intersection (Möller-Trumbore)

All implementations use the same algorithm with minor variations:

```glsl
// Compute edges
edge1 = v1 - v0
edge2 = v2 - v0

// Cross product: ray.dir × edge2
h = cross(ray.direction, edge2)
det = dot(edge1, h)

// Parallel test
if (abs(det) < epsilon) return miss

// Barycentric u
s = ray.origin - v0
u = dot(s, h) / det
if (u < 0 || u > 1) return miss

// Barycentric v
q = cross(s, edge1)
v = dot(ray.direction, q) / det
if (v < 0 || u + v > 1) return miss

// Distance
t = dot(edge2, q) / det
if (t < epsilon) return miss

return hit(t, u, v)
```

**Epsilon Values**:
- Most use `1e-8` or `1e-10` for parallel test
- Most use `1e-6` or `1e-8` for minimum t

## Two-Level BVH Strategy

All implementations use a two-level approach:

### Level 1: Scene BVH (Object-Level)

**Purpose**: Frustum culling and coarse raycasting

**Primitives**: World-space AABBs of all scene objects

**Usage**:
1. Frustum culling - cull entire branches outside frustum
2. Coarse raycast - find which objects ray potentially hits

**Refit for Dynamic Scenes** (6 implementations mention):
- caracal, fennec, hyena, lynx, mantis, wren
- Update AABBs bottom-up without rebuilding tree
- Trigger full rebuild if AABB growth exceeds threshold (typically 2×)

### Level 2: Mesh BVH (Triangle-Level)

**Purpose**: Precise raycasting

**Primitives**: Triangles in mesh local space

**Usage**:
1. Transform world ray to mesh local space
2. Traverse mesh BVH for precise triangle intersection
3. Transform hit back to world space

## Raycaster API

All implementations expose similar high-level APIs:

```typescript
interface Raycaster {
  // Set ray from camera + screen coordinates (NDC: -1 to 1)
  setFromCamera(coords: Vec2, camera: Camera): void

  // Set ray directly
  set(origin: Vec3, direction: Vec3): void

  // Raycast against objects
  intersectObject(object: Mesh, recursive?: boolean): Hit[]
  intersectObjects(objects: Mesh[], recursive?: boolean): Hit[]

  // Query options
  near: number   // Min distance (default: 0)
  far: number    // Max distance (default: Infinity)
}

interface Hit {
  distance: number          // World-space distance from ray origin
  point: Vec3               // World-space hit point
  normal: Vec3              // World-space surface normal (interpolated)
  uv?: Vec2                 // Texture coordinates (if available)
  triangleIndex: number     // Index of hit triangle
  faceIndex?: number        // Face index (alternative naming)
  object: Mesh              // The mesh that was hit

  // Barycentric coordinates (some implementations)
  u?: number
  v?: number
  barycentricU?: number
  barycentricV?: number
}
```

## Frustum Culling

All implementations use frustum culling for performance:

### Frustum Extraction

All use Gribb/Hartmann method (extract from view-projection matrix):

```typescript
// Extract 6 planes from view-projection matrix
// Each plane: [normalX, normalY, normalZ, d]

leftPlane   = row3 + row0
rightPlane  = row3 - row0
bottomPlane = row3 + row1
topPlane    = row3 - row1
nearPlane   = row3 + row2
farPlane    = row3 - row2

// Normalize each plane
for each plane:
  length = sqrt(nx² + ny² + nz²)
  plane /= length
```

### AABB-Frustum Test

All use p-vertex/n-vertex optimization:

```typescript
for each plane:
  // P-vertex: AABB corner most aligned with plane normal
  px = plane.normal.x >= 0 ? aabb.max.x : aabb.min.x
  py = plane.normal.y >= 0 ? aabb.max.y : aabb.min.y
  pz = plane.normal.z >= 0 ? aabb.max.z : aabb.min.z

  // If p-vertex is behind plane, entire AABB is outside
  dist = dot(plane.normal, p-vertex) + plane.d
  if (dist < 0) return OUTSIDE

return INSIDE or INTERSECTING
```

### BVH-Accelerated Culling (Mentioned by 4)

- **fennec**, **hyena**, **mantis**, **rabbit** explicitly mention BVH for frustum culling
- Test frustum against BVH nodes
- If node is fully outside: skip entire subtree
- If node is fully inside: add all leaves without further testing
- If intersecting: recurse into children

**Performance**:
- Without BVH: O(n) plane tests (n = object count)
- With BVH: O(log n) typical, O(n) worst case
- Reported speedup: 2-5× for large scenes (10K+ objects)

## Special Features & Optimizations

### Caracal
- **Stackless traversal** with skip pointers
- Right child offset encodes where to jump on miss
- Simpler traversal logic, more complex construction
- Detailed Karis average explanation for firefly prevention

### Fennec
- **BVH refit** for animated meshes (skinned geometry)
- Recompute leaf AABBs from current positions
- Propagate bounds bottom-up without restructuring tree
- Flat array layout emphasized for cache efficiency

### Hyena
- **Binned SAH** with 12 bins for O(n) construction
- Detailed split evaluation with left-right sweep
- Emphasis on preventing degenerate splits

### Lynx
- **13-tap filter** for downsample (bloom-related, not BVH)
- BVH used for both raycasting and frustum culling
- Detailed refit algorithm for dynamic objects
- Rebuild heuristic: if AABB growth > 2×, full rebuild

### Mantis
- **Two-level approach** clearly documented
- Scene BVH for coarse queries, mesh BVH for precision
- Explicit stack-based traversal with ordered children
- Performance comparison vs brute force

### Rabbit
- **All-hits** vs **first-hit** query modes documented
- Sorted results by distance
- Multiple weight function options for different use cases

### Shark
- **Flat array** with dual view (Float32Array + Uint32Array)
- High-bit flag for leaf detection
- Pre-computed inverse direction for slab test efficiency
- Transferable to Web Workers via ArrayBuffer

### Wren
- **32-byte aligned nodes** (half a cache line)
- Two nodes fit per cache line (optimal for sibling tests)
- Detailed flat array layout documentation
- WebGPU compute culling option (offload to GPU)
- Web Worker construction support with zero-copy transfer

## Recommendations for Cherry-Picking

**For Best Construction Performance**:
- Use **12-bin SAH** (caracal, hyena, mantis, rabbit) for good balance
- Or **32-bin SAH** (wren) for highest quality (slower build)
- Implement **Karis average** on first downsample to prevent fireflies

**For Best Traversal Performance**:
- Use **near-first ordering** (push far child first)
- Pre-allocate traversal stack (Int32Array(64))
- Pre-compute inverse ray direction (1.0 / direction) outside loop
- Use **flat array layout** with cache-aligned nodes (32 bytes)

**For Memory Efficiency**:
- Store BVH as shared ArrayBuffer with Float32Array and Uint32Array views
- Use Uint16Array for index buffer if < 65K vertices
- Reuse traversal stack and scratch buffers (fennec, mantis, shark approach)

**For Dynamic Scenes**:
- Implement **refit** for moderate movement (fennec, hyena, lynx approach)
- Bottom-up AABB update: recompute leaves from current positions, propagate up
- Trigger full rebuild if cumulative AABB growth > 2× original area
- For skinned meshes: refit on demand when raycasting, not every frame

**For Web Worker Support**:
- Use **transferable ArrayBuffer** for BVH (shark, wren approach)
- Construct BVH in worker, transfer back to main thread (zero copy)
- Store geometry as transferable buffers too

**Stackless vs Stack-Based**:

**Stack-Based** (recommended for most cases):
- Simpler to implement and understand
- Better debugging (can inspect stack state)
- Easier to modify traversal order
- Used by 7 out of 8 implementations

**Stackless** (advanced optimization):
- Requires skip pointers in node structure
- Slightly more complex construction
- Potentially faster on some architectures
- **caracal** is the only implementation using this

**Leaf Threshold Selection**:
- **4 triangles** is the sweet spot (used by 6 implementations)
- Smaller leaves = deeper tree = more AABB tests
- Larger leaves = shallower tree = more triangle tests
- Allow configuration for different geometry types

**SAH Constants**:
- Use `traversalCost = 1.0` and `intersectCost = 1.5`
- These reflect relative costs: triangle intersection is ~1.5× AABB test
- Most implementations use these values (lynx, mantis, rabbit, shark, wren)

**Scene-Level BVH**:
- Build scene BVH from mesh world AABBs (not triangles)
- Use for frustum culling and coarse raycasting
- Refit when objects move (don't rebuild every frame)
- Full rebuild when AABB growth threshold exceeded

**Mesh-Level BVH**:
- Build lazily on first raycast (not at load time unless needed)
- Cache per geometry (shared across mesh instances)
- Store in mesh local space (transform ray in/out for each query)
- Invalidate on geometry modification

**Integration with Frustum Culling**:
- Extract frustum from view-projection matrix (Gribb/Hartmann)
- Use p-vertex test (6 dot products, early exit)
- Optionally: traverse scene BVH instead of linear scan
- Cull entire branches when fully outside frustum
