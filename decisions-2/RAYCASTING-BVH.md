# RAYCASTING-BVH.md - Decisions

## Decision: 12-Bin SAH, Stack-Based Traversal, Near-First Ordering, Two-Level BVH

### BVH Construction: Binned SAH with 12 Bins

**Chosen**: Binned Surface Area Heuristic with 12 bins (4/9: Caracal, Hyena, Mantis, Rabbit)
**Rejected**: 32 bins (Wren) - higher quality tree but slower construction, diminishing returns past ~12 bins
**Rejected**: 16 bins (Fennec) - marginal quality improvement over 12

12 bins provide O(n log n) construction with near-optimal tree quality. For 100K triangles, construction takes ~20ms (one-time cost at load time).

### SAH Cost Function

```
cost = traversalCost + (leftArea / parentArea) * leftCount * intersectCost
                     + (rightArea / parentArea) * rightCount * intersectCost
```

**Constants**: `traversalCost = 1.0`, `intersectCost = 1.5` (5/9: Lynx, Mantis, Rabbit, Shark, Wren)

The 1.5x intersection cost reflects that a triangle test is ~1.5x more expensive than an AABB slab test.

### Node Layout: 32 Bytes, Flat Array

Universal agreement: 32-byte nodes in a contiguous typed array.

```
Offset 0-2:  AABB min (x, y, z)       [Float32]
Offset 3:    left child index          [Uint32, reinterpreted]
Offset 4-6:  AABB max (x, y, z)       [Float32]
Offset 7:    right child / leaf data   [Uint32, reinterpreted]
```

Dual view (Float32Array + Uint32Array over the same buffer) for reading AABBs as floats and indices as integers.

### Leaf Detection: Negative Count

**Chosen**: `rightOrCount < 0` indicates leaf node (3/9: Fennec, Lynx, Mantis)

```typescript
const isLeaf = nodes_u32[nodeOffset + 7] & 0x80000000  // Check high bit
const triangleOffset = nodes_u32[nodeOffset + 3]
const triangleCount = nodes_u32[nodeOffset + 7] & 0x7FFFFFFF
```

### Leaf Threshold: 4 Triangles

**Chosen**: Max 4 triangles per leaf (6/9: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit)

Also create leaf early if split cost exceeds leaf cost (SAH termination).

### Traversal: Stack-Based with Near-First Ordering

**Chosen**: Iterative stack-based traversal with near-first child ordering (7/9 use stack-based, 6/9 use near-first)
**Rejected**: Stackless with skip pointers (Caracal only) - more complex construction, marginal benefit

```typescript
const stack = new Int32Array(64)  // Pre-allocated, sufficient for millions of triangles
let stackPtr = 0
stack[stackPtr++] = 0  // Push root

while (stackPtr > 0) {
  const nodeIndex = stack[--stackPtr]

  if (!intersectsAABB(ray, nodeAABB)) continue

  if (isLeaf(node)) {
    // Test triangles
    for (let i = 0; i < triangleCount; i++) {
      testTriangle(ray, triangles[triangleOffset + i])
    }
  } else {
    // Push far child first so near child is popped first
    const nearChild = ray.direction[splitAxis] > 0 ? leftChild : rightChild
    const farChild = ray.direction[splitAxis] > 0 ? rightChild : leftChild
    stack[stackPtr++] = farChild
    stack[stackPtr++] = nearChild
  }
}
```

Near-first ordering enables better early termination - `closestDist` shrinks faster, pruning more nodes.

### Ray-AABB Intersection: Slab Method

Universal agreement:

```typescript
// Pre-computed outside hot loop
const invDirX = 1.0 / ray.direction[0]
const invDirY = 1.0 / ray.direction[1]
const invDirZ = 1.0 / ray.direction[2]

const intersectAABB = (min, max): number => {
  let tmin = (min[0] - ray.origin[0]) * invDirX
  let tmax = (max[0] - ray.origin[0]) * invDirX
  if (tmin > tmax) { const t = tmin; tmin = tmax; tmax = t }

  let tymin = (min[1] - ray.origin[1]) * invDirY
  let tymax = (max[1] - ray.origin[1]) * invDirY
  if (tymin > tymax) { const t = tymin; tymin = tymax; tymax = t }

  tmin = Math.max(tmin, tymin)
  tmax = Math.min(tmax, tymax)

  let tzmin = (min[2] - ray.origin[2]) * invDirZ
  let tzmax = (max[2] - ray.origin[2]) * invDirZ
  if (tzmin > tzmax) { const t = tzmin; tzmin = tzmax; tzmax = t }

  tmin = Math.max(tmin, tzmin)
  tmax = Math.min(tmax, tzmax)

  return tmax >= Math.max(tmin, 0) && tmin < closestDist ? tmin : -1
}
```

### Ray-Triangle Intersection: Moller-Trumbore

Universal agreement. Standard algorithm with epsilon = 1e-8 for parallel test.

### Two-Level BVH

**Level 1 - Scene BVH** (object-level):
- World-space AABBs of all scene objects
- Used for coarse raycasting and frustum culling
- Refit when objects move (update AABBs bottom-up without restructuring)
- Full rebuild when AABB growth exceeds 2x threshold

**Level 2 - Mesh BVH** (triangle-level):
- Triangles in mesh local space
- Built lazily on first raycast, cached per geometry
- Transform world ray to local space, traverse, transform hit back to world

### BVH Construction: Lazy + Cacheable

- Build on first raycast (not at load time unless explicitly requested)
- Cache per geometry instance (shared across meshes)
- Optionally build in a Web Worker with `Transferable` ArrayBuffer for large meshes (Wren/Fennec approach)

### Scene BVH for Frustum Culling

Use the scene-level BVH for frustum culling (4/9: Fennec, Hyena, Mantis, Rabbit):
- If node fully outside frustum: skip entire subtree
- If node fully inside: add all leaves without further testing
- If intersecting: recurse into children

Provides O(log n) culling vs O(n) brute force. Measurable benefit at 5000+ objects.

### Raycaster API

```typescript
interface Raycaster {
  setFromCamera(coords: Vec2, camera: Camera): void
  set(origin: Vec3, direction: Vec3): void
  intersectObject(object: Mesh, recursive?: boolean): Hit[]
  intersectObjects(objects: Mesh[], recursive?: boolean): Hit[]
  near: number   // Default: 0
  far: number    // Default: Infinity
}

interface Hit {
  distance: number
  point: Vec3
  normal: Vec3          // Interpolated from barycentric coordinates
  uv: Vec2 | null
  triangleIndex: number
  object: Mesh
}
```

Results sorted by distance (closest first).

### Performance

- Raycast against 50K triangles: ~0.01ms per ray (typical)
- Raycast against 2000 scene objects: ~0.5ms (scene BVH + mesh BVH)
- Traversal visits ~10-30 nodes out of thousands (logarithmic)

### Memory Overhead

Per node: 32 bytes. Worst case: `(2 * triangleCount - 1) * 32 bytes`.
For 100K triangles: ~6-7 MB. Additional: reordered index buffer + temporary build arrays.
