# RAYCASTING-BVH - Design Decisions

## BVH Construction

**Decision: SAH-based (Surface Area Heuristic) with 12-bin construction**

- Sources: SAH from all 8 active implementations; 12 bins from Caracal, Hyena, Mantis, Rabbit (4/8)
- Rejected: 32 bins (Wren) — slower build, marginal quality improvement
- Rejected: 16 bins (Fennec) — slightly better quality but 12 is sufficient

```
Algorithm:
1. Compute centroid bounds for current node
2. For each axis (X, Y, Z):
   a. Divide centroid range into 12 equal bins
   b. Assign each triangle to a bin based on centroid
   c. Sweep left-to-right accumulating AABBs and counts
   d. Sweep right-to-left evaluating SAH cost at each boundary
3. Choose axis and split with lowest SAH cost
4. If no split is cheaper than leaf cost, create leaf
```

## SAH Cost Function

**Decision: traversalCost = 1.0, intersectCost = 1.5**

- Sources: Lynx, Mantis, Rabbit, Shark, Wren (5/8)

```
cost = traversalCost +
       (leftArea / parentArea) * leftCount * intersectCost +
       (rightArea / parentArea) * rightCount * intersectCost
```

Triangle intersection (~1.5× cost) is more expensive than AABB traversal test.

## Node Layout

**Decision: Flat array, 32 bytes per node (8 floats)**

- Sources: All 8 active implementations agree on flat array + 32 bytes

```
Offset 0-2:  AABB min (x, y, z)        [3 × float32]
Offset 3:    left child index / tri offset [uint32 via reinterpret]
Offset 4-6:  AABB max (x, y, z)        [3 × float32]
Offset 7:    right child / tri count    [uint32 via reinterpret]
```

Dual view: Float32Array for AABB data, Uint32Array overlay for integer data.

## Leaf Detection

**Decision: Negative triangle count flag**

- Sources: Fennec, Lynx, Mantis (3/8)

```typescript
// Internal node: triCount >= 0
// Leaf node: triCount < 0, absolute value = actual count
const isLeaf = nodeData[offset + 7] < 0
const triCount = Math.abs(nodeData[offset + 7])
```

Simple, no bit masking needed, easy to debug.

## Leaf Threshold

**Decision: 4 triangles per leaf**

- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit (6/8)

Sweet spot between tree depth and per-leaf triangle testing. Also create leaf early if SAH split cost exceeds leaf cost. Configurable via `maxLeafTriangles` parameter.

## Traversal Strategy

**Decision: Stack-based iterative traversal with near-first child ordering**

- Sources: Stack-based from 7/8; near-first from Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit (6/8)
- Rejected: Stackless with skip pointers (Caracal) — more complex construction, only 1 implementation uses it

```typescript
const stack = new Int32Array(64)  // pre-allocated
let stackPtr = 0

stack[stackPtr++] = 0  // root node

while (stackPtr > 0) {
  const nodeIndex = stack[--stackPtr]

  if (!rayIntersectsAABB(ray, nodeAABB)) continue

  if (isLeaf(node)) {
    testTriangles(node)
  } else {
    // Push far child first so near child is popped first
    const nearFirst = ray.direction[splitAxis] > 0
    stack[stackPtr++] = nearFirst ? rightChild : leftChild
    stack[stackPtr++] = nearFirst ? leftChild : rightChild
  }
}
```

Near-first ordering improves early termination: closestDist shrinks faster, pruning more subtrees.

## Ray-AABB Intersection

**Decision: Slab method with pre-computed inverse direction (universal agreement)**

```typescript
// Pre-compute outside hot loop
const invDirX = 1.0 / ray.direction[0]
const invDirY = 1.0 / ray.direction[1]
const invDirZ = 1.0 / ray.direction[2]

// Slab test per node
const t1x = (aabbMin[0] - ray.origin[0]) * invDirX
const t2x = (aabbMax[0] - ray.origin[0]) * invDirX
// ... swap if needed, compute tNear/tFar across all 3 axes
// hit = tFar >= max(tNear, 0) && tNear < closestDist
```

## Ray-Triangle Intersection

**Decision: Möller-Trumbore algorithm (universal agreement)**

Standard algorithm with early rejection on barycentric coordinates. Returns distance, barycentric u/v for interpolation. Epsilon: 1e-8 for parallel test, 1e-6 for minimum distance.

## Two-Level BVH

**Decision: Scene BVH (object-level) + Mesh BVH (triangle-level)**

- Sources: All active implementations describe this pattern

### Level 1: Scene BVH
- Primitives: world-space AABBs of all scene objects
- Used for: frustum culling and coarse raycasting (which objects does the ray potentially hit?)
- Refit when objects move (update AABBs bottom-up); full rebuild if AABB growth > 2× threshold

### Level 2: Mesh BVH
- Primitives: triangles in mesh local space
- Built lazily on first raycast, cached per geometry
- Transform world ray to mesh local space before traversal

```typescript
// Raycast flow:
// 1. Test scene BVH → list of candidate meshes
// 2. For each candidate:
//    a. Transform ray to mesh local space
//    b. Traverse mesh BVH for triangle intersection
//    c. Transform hit point/normal back to world space
```

## BVH Construction Timing

**Decision: Lazy construction on first raycast, cache per geometry**

- Sources: Universal agreement on lazy construction
- Not built at load time (wasteful if geometry is never raycasted)
- Cached on geometry — shared across all mesh instances using the same geometry
- Can be offloaded to Web Worker for large meshes (Wren, Fennec approach)

## BVH Refit for Dynamic Scenes

**Decision: Bottom-up AABB refit for moderate movement, full rebuild on threshold**

- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Wren (6/8)

```typescript
// When objects move:
refitSceneBVH()  // update leaf AABBs, propagate up

// When AABB growth exceeds 2× original:
rebuildSceneBVH()  // full reconstruction
```

For skinned meshes: refit mesh BVH on demand when raycasting is requested, not every frame.

## Frustum Culling

**Decision: 6-plane extraction (Gribb-Hartmann) with p-vertex/n-vertex AABB test**

- Sources: All implementations use this approach

```typescript
// Extract 6 planes from view-projection matrix
// Each plane: [nx, ny, nz, d]
const planes = extractFrustumPlanes(viewProjectionMatrix)

// P-vertex test per AABB
for (const plane of planes) {
  const px = plane[0] >= 0 ? aabb.max[0] : aabb.min[0]
  const py = plane[1] >= 0 ? aabb.max[1] : aabb.min[1]
  const pz = plane[2] >= 0 ? aabb.max[2] : aabb.min[2]
  if (dot(plane, [px, py, pz]) + plane[3] < 0) return OUTSIDE
}
return INSIDE_OR_INTERSECTING
```

For scenes under 5000 objects, brute-force culling (test all objects) is fast enough (~0.1-0.2ms). BVH-accelerated culling is available for larger scenes.

## Raycaster API

```typescript
interface Raycaster {
  setFromCamera(coords: Vec2, camera: Camera): void
  set(origin: Vec3, direction: Vec3): void
  intersectObject(object: Mesh, recursive?: boolean): Hit[]
  intersectObjects(objects: Mesh[], recursive?: boolean): Hit[]
  near: number   // min distance (default: 0)
  far: number    // max distance (default: Infinity)
}

interface Hit {
  distance: number
  point: Vec3
  normal: Vec3
  uv?: Vec2
  triangleIndex: number
  object: Mesh
}
```

Results sorted by distance. Closest hit returned first.

## Performance Characteristics

- BVH build: ~3ms for 10K triangles, ~15-20ms for 100K triangles (one-time)
- Single raycast: ~0.01ms per ray for 50K triangle mesh
- Scene raycast (2000 objects): ~0.4-0.5ms per ray
- Frustum culling (2000 objects): ~0.1-0.2ms
- BVH memory: ~6-7MB for 100K triangles

Logarithmic traversal: typically 10-30 nodes visited out of thousands.
