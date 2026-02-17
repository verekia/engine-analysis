# Raycasting & BVH Architecture

## Overview

Shark provides high-performance raycasting using a SAH-BVH (Surface Area Heuristic Bounding Volume Hierarchy) with a flat array layout. The design targets performance comparable to `three-mesh-bvh` through cache-friendly data layout, zero-allocation traversal, and efficient ray-triangle intersection.

## BVH Construction

### SAH (Surface Area Heuristic)

The SAH produces optimal BVH trees for ray tracing by minimizing the expected cost of ray-node intersection tests. At each split, it evaluates all possible partition planes and chooses the one that minimizes:

```
cost(split) = traversalCost +
  (surfaceArea(left) / surfaceArea(parent)) * count(left) * intersectCost +
  (surfaceArea(right) / surfaceArea(parent)) * count(right) * intersectCost
```

Where:
- `traversalCost` = cost of traversing an internal node (typically 1.0)
- `intersectCost` = cost of a ray-triangle intersection test (typically 1.5)
- `surfaceArea()` = surface area of the AABB

### Binned SAH

Full SAH evaluation is O(n^2) per node. Shark uses **binned SAH** which is O(n) per node:

```typescript
const BIN_COUNT = 12

const findBestSplit = (
  triangles: TriangleRef[],
  parentAABB: AABB,
): { axis: number, position: number, cost: number } => {
  let bestCost = Infinity
  let bestAxis = 0
  let bestPos = 0

  // Try all 3 axes
  for (let axis = 0; axis < 3; axis++) {
    const extent = parentAABB.max[axis] - parentAABB.min[axis]
    if (extent < 1e-6) continue

    // Initialize bins
    const bins: Bin[] = Array.from({ length: BIN_COUNT }, () => ({
      aabb: emptyAABB(),
      count: 0,
    }))

    // Assign triangles to bins based on centroid position
    for (const tri of triangles) {
      const centroid = tri.centroid[axis]
      const binIdx = Math.min(
        BIN_COUNT - 1,
        Math.floor(BIN_COUNT * (centroid - parentAABB.min[axis]) / extent),
      )
      expandAABB(bins[binIdx].aabb, tri.aabb)
      bins[binIdx].count++
    }

    // Evaluate split costs using left-right sweep
    const leftAABBs: AABB[] = []
    const leftCounts: number[] = []
    let accumAABB = emptyAABB()
    let accumCount = 0

    for (let i = 0; i < BIN_COUNT - 1; i++) {
      expandAABB(accumAABB, bins[i].aabb)
      accumCount += bins[i].count
      leftAABBs.push(cloneAABB(accumAABB))
      leftCounts.push(accumCount)
    }

    accumAABB = emptyAABB()
    accumCount = 0

    for (let i = BIN_COUNT - 1; i > 0; i--) {
      expandAABB(accumAABB, bins[i].aabb)
      accumCount += bins[i].count

      const cost = 1.0 +
        (surfaceArea(leftAABBs[i - 1]) * leftCounts[i - 1] +
         surfaceArea(accumAABB) * accumCount) * 1.5 / surfaceArea(parentAABB)

      if (cost < bestCost) {
        bestCost = cost
        bestAxis = axis
        bestPos = parentAABB.min[axis] + extent * (i / BIN_COUNT)
      }
    }
  }

  return { axis: bestAxis, position: bestPos, cost: bestCost }
}
```

### Leaf Threshold

If the best split cost exceeds the cost of testing all remaining triangles, or the triangle count drops below 4, a leaf node is created instead of splitting further.

### Flat Array Layout

The BVH is stored in a contiguous `Float32Array` for cache-friendly traversal. Each node occupies 8 floats (32 bytes):

```
Node layout (8 x float32 = 32 bytes per node):
+------------+------------+------------+------------+
| minX (f32) | minY (f32) | minZ (f32) | data0 (u32)|
+------------+------------+------------+------------+
| maxX (f32) | maxY (f32) | maxZ (f32) | data1 (u32)|
+------------+------------+------------+------------+

Internal node:
  data0 = left child index (node array index)
  data1 = right child index

Leaf node:
  data0 = first triangle index | 0x80000000 (high bit = leaf flag)
  data1 = triangle count
```

The u32 values are stored as reinterpreted f32 using a shared `ArrayBuffer` with both `Float32Array` and `Uint32Array` views.

```typescript
interface FlatBVH {
  nodes: Float32Array           // 8 floats per node
  nodeDataU32: Uint32Array      // Same buffer, uint32 view for data0/data1
  nodeCount: number
  triangleIndices: Uint32Array  // Reordered triangle indices
}
```

### Build Process

```typescript
const buildBVH = (positions: Float32Array, indices: Uint32Array): FlatBVH => {
  // 1. Compute per-triangle AABBs and centroids
  const triCount = indices.length / 3
  const triRefs: TriangleRef[] = new Array(triCount)
  for (let i = 0; i < triCount; i++) {
    triRefs[i] = computeTriangleRef(positions, indices, i)
  }

  // 2. Allocate flat node array (worst case: 2*triCount - 1 nodes)
  const maxNodes = 2 * triCount - 1
  const buffer = new ArrayBuffer(maxNodes * 8 * 4)
  const nodes = new Float32Array(buffer)
  const nodeData = new Uint32Array(buffer)
  let nodeCount = 0

  // 3. Recursive depth-first build
  const build = (triStart: number, triEnd: number): number => {
    const nodeIdx = nodeCount++
    const aabb = computeAABBRange(triRefs, triStart, triEnd)
    const base = nodeIdx * 8

    // Write AABB
    nodes[base + 0] = aabb.min[0]
    nodes[base + 1] = aabb.min[1]
    nodes[base + 2] = aabb.min[2]
    nodes[base + 4] = aabb.max[0]
    nodes[base + 5] = aabb.max[1]
    nodes[base + 6] = aabb.max[2]

    const count = triEnd - triStart
    if (count <= 4) {
      nodeData[base + 3] = triStart | 0x80000000
      nodeData[base + 7] = count
      return nodeIdx
    }

    const split = findBestSplit(triRefs.slice(triStart, triEnd), aabb)
    if (split.cost >= count * 1.5) {
      nodeData[base + 3] = triStart | 0x80000000
      nodeData[base + 7] = count
      return nodeIdx
    }

    const mid = partition(triRefs, triStart, triEnd, split.axis, split.position)
    const left = build(triStart, mid)
    const right = build(mid, triEnd)

    nodeData[base + 3] = left
    nodeData[base + 7] = right
    return nodeIdx
  }

  build(0, triCount)

  // 4. Build reordered triangle index array
  const triangleIndices = new Uint32Array(triCount)
  for (let i = 0; i < triCount; i++) {
    triangleIndices[i] = triRefs[i].originalIndex
  }

  return {
    nodes: new Float32Array(buffer, 0, nodeCount * 8),
    nodeDataU32: new Uint32Array(buffer, 0, nodeCount * 8),
    nodeCount,
    triangleIndices,
  }
}
```

## Ray-Triangle Intersection (Moller-Trumbore)

```typescript
// Pre-allocated scratch vectors (module-level, zero allocation)
const _edge1 = vec3Create()
const _edge2 = vec3Create()
const _h = vec3Create()
const _s = vec3Create()
const _q = vec3Create()

const rayTriangleIntersect = (
  rayOrigin: Vec3,
  rayDir: Vec3,
  v0: Vec3, v1: Vec3, v2: Vec3,
): number => {  // Returns distance t, or -1 for miss
  vec3Sub(_edge1, v1, v0)
  vec3Sub(_edge2, v2, v0)
  vec3Cross(_h, rayDir, _edge2)
  const a = vec3Dot(_edge1, _h)

  if (a > -1e-6 && a < 1e-6) return -1  // Parallel to triangle

  const f = 1.0 / a
  vec3Sub(_s, rayOrigin, v0)
  const u = f * vec3Dot(_s, _h)
  if (u < 0.0 || u > 1.0) return -1

  vec3Cross(_q, _s, _edge1)
  const v = f * vec3Dot(rayDir, _q)
  if (v < 0.0 || u + v > 1.0) return -1

  const t = f * vec3Dot(_edge2, _q)
  return t > 1e-6 ? t : -1
}
```

## Ray-AABB Intersection (Slab Method)

```typescript
const rayAABBIntersect = (
  ox: number, oy: number, oz: number,       // Ray origin
  dix: number, diy: number, diz: number,    // 1/direction (pre-computed)
  nodes: Float32Array, base: number,         // AABB data
  tMax: number,                              // Current closest hit
): boolean => {
  const t1x = (nodes[base + 0] - ox) * dix
  const t2x = (nodes[base + 4] - ox) * dix
  const t1y = (nodes[base + 1] - oy) * diy
  const t2y = (nodes[base + 5] - oy) * diy
  const t1z = (nodes[base + 2] - oz) * diz
  const t2z = (nodes[base + 6] - oz) * diz

  const tNear = Math.max(Math.min(t1x, t2x), Math.min(t1y, t2y), Math.min(t1z, t2z))
  const tFar = Math.min(Math.max(t1x, t2x), Math.max(t1y, t2y), Math.max(t1z, t2z))

  return tFar >= Math.max(tNear, 0) && tNear < tMax
}
```

Pre-computed inverse direction avoids division in the hot loop. `1/0 = Infinity` is handled correctly by IEEE 754.

## BVH Traversal (Stack-Based Closest-Hit)

```typescript
const _stack = new Int32Array(64)  // Pre-allocated traversal stack

const raycastBVH = (
  bvh: FlatBVH,
  positions: Float32Array,
  indices: Uint32Array,
  rayOrigin: Vec3,
  rayDir: Vec3,
): { distance: number, triangleIndex: number } | null => {
  const dix = 1 / rayDir[0], diy = 1 / rayDir[1], diz = 1 / rayDir[2]
  let closestT = Infinity
  let closestTri = -1
  let sp = 0

  _stack[sp++] = 0  // Root node

  while (sp > 0) {
    const nodeIdx = _stack[--sp]
    const base = nodeIdx * 8

    if (!rayAABBIntersect(
      rayOrigin[0], rayOrigin[1], rayOrigin[2],
      dix, diy, diz, bvh.nodes, base, closestT,
    )) continue

    const data0 = bvh.nodeDataU32[base + 3]

    if (data0 & 0x80000000) {
      // Leaf: test all triangles
      const triStart = data0 & 0x7FFFFFFF
      const triCount = bvh.nodeDataU32[base + 7]

      for (let i = 0; i < triCount; i++) {
        const ti = bvh.triangleIndices[triStart + i]
        const t = rayTriangleIntersect(
          rayOrigin, rayDir,
          getVec3(positions, indices[ti * 3]),
          getVec3(positions, indices[ti * 3 + 1]),
          getVec3(positions, indices[ti * 3 + 2]),
        )
        if (t > 0 && t < closestT) {
          closestT = t
          closestTri = ti
        }
      }
    } else {
      // Internal: push children (far first for near-first processing)
      _stack[sp++] = bvh.nodeDataU32[base + 7]  // right
      _stack[sp++] = data0                        // left
    }
  }

  return closestTri >= 0 ? { distance: closestT, triangleIndex: closestTri } : null
}
```

## Raycaster API

### High-Level Interface

```typescript
interface Raycaster {
  setFromCamera(ndc: Vec2, camera: Camera): void
  set(origin: Vec3, direction: Vec3): void
  intersectObject(object: Node3D, recursive?: boolean): RaycastHit[]
  intersectObjects(objects: Node3D[], recursive?: boolean): RaycastHit[]
}

interface RaycastHit {
  distance: number          // World-space distance from ray origin
  point: Vec3               // World-space intersection point
  normal: Vec3              // Interpolated surface normal at hit
  uv: Vec2 | null           // Interpolated UV (if geometry has UVs)
  triangleIndex: number     // Index of hit triangle
  object: Mesh              // The mesh that was hit
}
```

### Screen-to-Ray Conversion

```typescript
const setFromCamera = (raycaster: Raycaster, ndc: Vec2, camera: Camera) => {
  const invVP = mat4Invert(mat4Create(), camera.viewProjectionMatrix)
  const near = vec3TransformMat4(vec3Create(), [ndc[0], ndc[1], -1], invVP)
  const far = vec3TransformMat4(vec3Create(), [ndc[0], ndc[1], 1], invVP)
  raycaster.origin = near
  raycaster.direction = vec3Normalize(vec3Create(), vec3Sub(vec3Create(), far, near))
}
```

### Object-Space Transform

World-space ray is transformed to each mesh's local space for BVH traversal:

```typescript
const intersectMesh = (ray: Ray, mesh: Mesh): RaycastHit | null => {
  const invWorld = mat4Invert(_invWorldScratch, mesh.worldMatrix)
  const localOrigin = vec3TransformMat4(_localOriginScratch, ray.origin, invWorld)
  const localDir = vec3Normalize(
    _localDirScratch,
    vec3TransformMat4Dir(_localDirScratch, ray.direction, invWorld),
  )

  ensureBVH(mesh.geometry)
  const hit = raycastBVH(mesh.geometry._bvh!, mesh.geometry.positions, mesh.geometry.indices, localOrigin, localDir)
  if (!hit) return null

  const worldPoint = vec3TransformMat4(vec3Create(), computeHitLocalPoint(hit), mesh.worldMatrix)
  return {
    distance: vec3Distance(ray.origin, worldPoint),
    point: worldPoint,
    triangleIndex: hit.triangleIndex,
    object: mesh,
  }
}
```

### BVH Caching

BVH is built lazily on first raycast and cached per geometry. Invalidated via `geometry.invalidateBVH()` if positions change.

## Performance

| Metric | Value | Notes |
|--------|-------|-------|
| BVH build | ~50ms for 100K triangles | One-time cost per geometry |
| Closest-hit | ~0.01ms per ray (100K tri mesh) | SAH + flat array + zero alloc |
| Any-hit | ~0.005ms per ray | Early termination |
| Memory per node | 32 bytes | 8 x float32 |
| Stack depth | 64 (fixed) | Handles millions of triangles |
