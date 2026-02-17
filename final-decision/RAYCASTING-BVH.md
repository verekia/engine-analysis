# RAYCASTING-BVH — Final Decision

## BVH Construction

**Decision: Binned SAH with 12 bins** (universal agreement)

Surface Area Heuristic determines optimal split planes. 12-bin approximation gives O(n log n) build time while producing near-optimal trees.

```typescript
const SAH_COST = {
  traversal: 1.0,
  intersection: 1.5,
}
const BIN_COUNT = 12
const MAX_LEAF_TRIANGLES = 4

// SAH cost for a split:
// cost = traversalCost + (leftArea/parentArea) * leftCount * intersectCost
//                      + (rightArea/parentArea) * rightCount * intersectCost
```

For each axis (X, Y, Z), distribute triangles into 12 bins by centroid, evaluate the SAH cost at each bin boundary, and choose the split with the lowest cost. If no split improves over the leaf cost, create a leaf node.

## Node Layout

**Decision: 32-byte flat array layout** (universal agreement)

```
struct BVHNode {
  float minX, minY, minZ   // AABB min (12 bytes)
  int   rightChildOrCount  // >0: right child offset, <0: -(triangle count) (4 bytes)
  float maxX, maxY, maxZ   // AABB max (12 bytes)
  int   leftChildOrStart   // Inner: left child offset, Leaf: first triangle index (4 bytes)
}
// Total: 32 bytes per node
```

Leaf detection: `rightChildOrCount < 0` means leaf, with `|rightChildOrCount|` triangles starting at `leftChildOrStart`.

Flat array layout enables linear memory access patterns and avoids pointer chasing. The BVH is stored in a contiguous `Float32Array` / `Int32Array` (aliased views on the same buffer).

## Traversal

**Decision: Stack-based iterative traversal with near-first ordering** (universal agreement)

```typescript
const intersectBVH = (ray: Ray, nodes: Float32Array, triangles: Float32Array): Hit | null => {
  const stack: number[] = [0]  // Pre-allocated, reused across frames
  let closest: Hit | null = null

  while (stack.length > 0) {
    const nodeIndex = stack.pop()!
    const node = getNode(nodes, nodeIndex)

    if (!rayIntersectsAABB(ray, node.min, node.max, closest?.distance ?? Infinity)) continue

    if (node.isLeaf) {
      for (let i = node.start; i < node.start + node.count; i++) {
        const hit = rayIntersectsTriangle(ray, triangles, i)
        if (hit && (!closest || hit.distance < closest.distance)) {
          closest = hit
        }
      }
    } else {
      // Push far child first, near child second (near child processed first)
      const leftFirst = /* compare ray direction with split axis */
      if (leftFirst) {
        stack.push(node.rightChild)
        stack.push(node.leftChild)
      } else {
        stack.push(node.leftChild)
        stack.push(node.rightChild)
      }
    }
  }

  return closest
}
```

Near-first ordering reduces average traversal by testing the child closer to the ray origin first, allowing early termination.

## Ray-AABB Intersection

**Decision: Slab method** (universal agreement)

```typescript
const rayIntersectsAABB = (ray: Ray, min: Vec3, max: Vec3, maxDist: number): boolean => {
  const invDirX = 1 / ray.direction[0]
  const invDirY = 1 / ray.direction[1]
  const invDirZ = 1 / ray.direction[2]

  const t1 = (min[0] - ray.origin[0]) * invDirX
  const t2 = (max[0] - ray.origin[0]) * invDirX
  const t3 = (min[1] - ray.origin[1]) * invDirY
  const t4 = (max[1] - ray.origin[1]) * invDirY
  const t5 = (min[2] - ray.origin[2]) * invDirZ
  const t6 = (max[2] - ray.origin[2]) * invDirZ

  const tmin = Math.max(Math.min(t1, t2), Math.min(t3, t4), Math.min(t5, t6))
  const tmax = Math.min(Math.max(t1, t2), Math.max(t3, t4), Math.max(t5, t6))

  return tmax >= Math.max(0, tmin) && tmin < maxDist
}
```

Inverse direction pre-computed once per ray.

## Ray-Triangle Intersection

**Decision: Möller-Trumbore algorithm** (universal agreement)

Fast, branch-minimal ray-triangle intersection that computes barycentric coordinates for UV interpolation and normal interpolation at the hit point.

## Two-Level BVH

**Decision: Scene-level BVH + per-mesh BVH** (universal agreement)

1. **Scene BVH**: Contains mesh AABBs. Quickly rejects meshes the ray cannot hit.
2. **Mesh BVH**: Contains triangle data. Precise intersection within a mesh.

A ray first traverses the scene BVH to find candidate meshes, then traverses each candidate's mesh BVH for triangle-level intersection.

## Lazy Construction

**Decision: Build BVH on first raycast, then cache** (universal agreement)

- Mesh BVH is built on the first raycast against that mesh, then cached on the geometry
- Scene BVH is rebuilt when the scene structure changes (add/remove objects)
- Avoids wasting time building BVHs for geometry that is never raycasted
- BVH construction can optionally be offloaded to a Web Worker

## Frustum Culling via BVH

The scene-level BVH can also accelerate frustum culling for large scenes (>5000 objects). For ≤2000 objects, brute-force AABB testing is sufficient (~0.1ms).

Frustum planes extracted from the view-projection matrix via Gribb-Hartmann method. P-vertex/N-vertex optimization for AABB-plane classification.

## Raycaster API

```typescript
const raycaster = createRaycaster()

// From camera + mouse position (NDC coordinates)
raycaster.setFromCamera([ndcX, ndcY], camera)

// From arbitrary ray
raycaster.set(origin, direction)

// Query
const hits = raycaster.intersectObjects(scene.children, true)  // true = recursive

// Hit result
interface RaycastHit {
  distance: number       // World-space distance from ray origin
  point: Vec3            // World-space intersection point
  normal: Vec3           // Interpolated surface normal at hit point
  uv: Vec2 | null       // Interpolated UV coordinates (if geometry has UVs)
  triangleIndex: number  // Index of the triangle that was hit
  object: Mesh           // The mesh that was hit
}
```

Hits are returned sorted by distance (nearest first).

## Performance

- Scene BVH traversal: ~0.01ms per ray (2000 objects)
- Mesh BVH traversal: ~0.01-0.05ms per ray (depending on triangle count)
- Total per ray: ~0.02-0.06ms
- BVH construction: ~2-5ms for 100K triangles (one-time cost)
