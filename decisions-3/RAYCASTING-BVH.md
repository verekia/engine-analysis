# RAYCASTING-BVH.md - Raycasting and BVH

## BVH Construction

**Decision: Binned SAH with 12 bins, flat array layout** (Caracal, Hyena, Mantis, Rabbit approach)

### Surface Area Heuristic (SAH)

All 8 active implementations agree on SAH. The algorithm selects split planes that minimize the expected cost of ray traversal:

```
cost = traversalCost +
       (leftArea / parentArea) * leftCount * intersectCost +
       (rightArea / parentArea) * rightCount * intersectCost
```

**SAH cost constants**: `traversalCost = 1.0`, `intersectCost = 1.5` (5/8 implementations use these - triangle intersection is ~1.5x the cost of an AABB test)

### Binned Construction

12-bin SAH reduces construction from O(n^2) to O(n log n):

1. Compute centroid bounds for the current set of primitives
2. For each axis (X, Y, Z):
   - Divide centroid range into 12 equal bins
   - Assign each triangle to a bin based on centroid position
   - Compute AABB and count for each bin
   - Sweep left-to-right, accumulating AABBs and counts
   - Sweep right-to-left, evaluating SAH cost at each boundary
3. Choose the axis and split position with the lowest SAH cost
4. If no split is cheaper than a leaf, create a leaf node

**Why 12 bins** (over 16 or 32): Good balance of build quality vs build speed. More bins give marginally better trees but cost more time. 12 bins hit the diminishing-returns point.

### Leaf Threshold

**Decision: 4 triangles** (6/8 implementations default to this)

Nodes with <= 4 triangles become leaf nodes. This balances tree depth against leaf intersection cost.

## Node Layout

**Decision: 32-byte flat array nodes with negative count for leaf detection** (Fennec, Lynx, Mantis approach)

```
Each node: 32 bytes (8 floats / 8 uint32s)

Offset 0-2:  AABB min (x, y, z)       [Float32 x3]
Offset 3:    left child index          [Uint32]
Offset 4-6:  AABB max (x, y, z)       [Float32 x3]
Offset 7:    right child index OR      [Int32]
             negative triangle count (leaf)
```

**Leaf detection**: If `data[7] < 0`, node is a leaf. `|data[7]|` is the triangle count. `data[3]` is the offset into the reordered index buffer.

32 bytes = half a cache line. Two sibling nodes fit in one 64-byte cache line, which is optimal for traversal since we always test both children.

### Storage

Single `Float32Array` + `Int32Array` view on the same `ArrayBuffer`. Float32Array for AABB access, Int32Array for child/count access. Zero-copy dual views.

## Traversal

**Decision: Stack-based iterative traversal with near-first ordering** (7/8 implementations, better than Caracal's stackless)

### Pre-allocated Stack

```typescript
const stack = new Int32Array(64)  // Pre-allocated, reused across raycasts
```

64 entries support trees with up to 2^64 nodes (far more than needed).

### Near-First Ordering

Push the far child first so the near child is popped first. This ensures we test closer nodes before farther ones, causing `closestDist` to shrink faster and enabling earlier termination of distant subtrees.

```typescript
// Determine which child is closer along the ray
if (ray.direction[splitAxis] >= 0) {
  stack[stackPtr++] = rightChild
  stack[stackPtr++] = leftChild  // Popped first = tested first
} else {
  stack[stackPtr++] = leftChild
  stack[stackPtr++] = rightChild
}
```

### Why Stack-Based Over Stackless (Caracal)?

Stackless traversal (skip pointers) is elegant but:
- More complex construction (must compute skip targets)
- Harder to modify traversal order (near-first ordering is awkward with skip pointers)
- Debugging is harder (no stack to inspect)
- Performance is comparable (stack fits in L1 cache)

7/8 implementations chose stack-based. The consensus is clear.

## Ray-AABB Intersection (Slab Method)

```typescript
// Pre-compute inverse direction ONCE per ray (outside BVH traversal)
const invDirX = 1.0 / ray.direction[0]
const invDirY = 1.0 / ray.direction[1]
const invDirZ = 1.0 / ray.direction[2]

// Slab test per node
const t1x = (aabb.minX - ray.origin[0]) * invDirX
const t2x = (aabb.maxX - ray.origin[0]) * invDirX
const tMinX = Math.min(t1x, t2x)
const tMaxX = Math.max(t1x, t2x)
// ... same for Y, Z

const tNear = Math.max(tMinX, tMinY, tMinZ)
const tFar = Math.min(tMaxX, tMaxY, tMaxZ)

return tFar >= Math.max(tNear, 0) && tNear < closestDist
```

## Ray-Triangle Intersection (Moller-Trumbore)

Standard algorithm used by all 8 implementations. Returns distance `t` and barycentric coordinates `(u, v)` for interpolating normals and UVs at the hit point.

Epsilon values:
- Parallel test: `abs(det) < 1e-8`
- Minimum t: `t > 1e-6`

## Two-Level BVH

### Level 1: Scene BVH

- Primitives: World-space AABBs of all mesh nodes
- Purpose: Frustum culling and coarse raycasting
- Built from mesh world AABBs, rebuilt/refit when objects move

**Refit vs rebuild**: Refit (update AABBs bottom-up without restructuring the tree) for moderate movement. Full rebuild when cumulative AABB growth exceeds 2x the original surface area. Refit is O(n), rebuild is O(n log n).

### Level 2: Mesh BVH

- Primitives: Triangles in mesh local space
- Purpose: Precise raycasting
- Built lazily on first raycast against a mesh, cached per geometry
- Shared across mesh instances (same geometry = same BVH)

Transform ray to mesh local space before traversal, transform hit point and normal back to world space after.

## Raycaster API

```typescript
const raycaster = createRaycaster()

// From camera and screen coordinates (NDC: -1 to 1)
raycaster.setFromCamera(camera, ndcX, ndcY)

// Direct ray
raycaster.set(origin, direction)

// Query
raycaster.near = 0
raycaster.far = Infinity
const hits = raycaster.intersectObjects(scene.children, { recursive: true })
```

### Hit Result

```typescript
interface Hit {
  distance: number          // World-space distance from ray origin
  point: Vec3               // World-space hit position
  normal: Vec3              // World-space interpolated surface normal
  uv: Vec2 | null           // Texture coordinates (if available)
  triangleIndex: number     // Index of hit triangle
  object: Mesh              // The mesh that was hit
}
```

Results are sorted by distance (closest first).

## Frustum Culling

### Frustum Extraction (Gribb-Hartmann Method)

Extract 6 planes from the view-projection matrix:

```typescript
// Each plane: [nx, ny, nz, d] where nx*x + ny*y + nz*z + d = 0
leftPlane   = row3 + row0
rightPlane  = row3 - row0
bottomPlane = row3 + row1
topPlane    = row3 - row1
nearPlane   = row3 + row2
farPlane    = row3 - row2
// Normalize each plane
```

### AABB-Frustum Test (P-vertex Optimization)

```typescript
for (const plane of frustumPlanes) {
  // P-vertex: AABB corner most aligned with plane normal
  const px = plane.nx >= 0 ? aabb.maxX : aabb.minX
  const py = plane.ny >= 0 ? aabb.maxY : aabb.minY
  const pz = plane.nz >= 0 ? aabb.maxZ : aabb.minZ
  if (plane.nx * px + plane.ny * py + plane.nz * pz + plane.d < 0) {
    return OUTSIDE  // Early exit: entirely behind this plane
  }
}
return VISIBLE
```

6 dot products per AABB, with early exit. For 2000 meshes: ~0.1ms.

### BVH-Accelerated Culling

When the scene BVH is available, frustum culling traverses the BVH instead of testing every mesh:
- Node fully outside: skip entire subtree
- Node fully inside: add all leaves without further testing
- Node intersecting: recurse into children

This provides O(log n) typical culling cost. For scenes with >2000 objects, this is significantly faster than brute-force.

## Performance

| Operation | Cost |
|-----------|------|
| BVH build (10K triangles) | ~3-5ms (one-time) |
| BVH build (100K triangles) | ~15-40ms (one-time) |
| Single raycast (50K triangle mesh) | ~0.01ms |
| Scene raycast (2000 objects) | ~0.4-0.5ms |
| Frustum culling (2000 AABBs, brute force) | ~0.1ms |
| Frustum culling (2000 AABBs, BVH) | ~0.03ms |
