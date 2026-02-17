# 11 — BVH Raycasting

## Overview

Wren implements a SAH-based (Surface Area Heuristic) Bounding Volume Hierarchy stored as a flat typed array for cache-friendly traversal. The design is inspired by three-mesh-bvh's approach: packed flat-array nodes, index buffer reordering, and optimized first-hit traversal with early termination.

## BVH Data Structure

### Flat Array Layout

The BVH is stored in a single `ArrayBuffer` with overlaid `Float32Array` and `Uint32Array` views. Each node occupies a fixed 32 bytes:

```
Node layout (32 bytes):
┌─────────────────────────────────────────────────┐
│ Float32: minX                                   │  byte 0-3
│ Float32: minY                                   │  byte 4-7
│ Float32: minZ                                   │  byte 8-11
│ Float32: maxX                                   │  byte 12-15
│ Float32: maxY                                   │  byte 16-19
│ Float32: maxZ                                   │  byte 20-23
│ Uint32:  meta                                   │  byte 24-27
│ Uint32:  secondChildOrCount                     │  byte 28-31
└─────────────────────────────────────────────────┘
```

For **internal nodes**:
- `meta`: right child index (left child is always `nodeIndex + 1` — stored adjacently for cache locality)
- `secondChildOrCount`: split axis encoded in bits 30-31, unused otherwise

For **leaf nodes** (flagged by `meta & IS_LEAF_FLAG`):
- `meta`: `IS_LEAF_FLAG | triangleOffset` — starting index into the (reordered) index buffer
- `secondChildOrCount`: triangle count in this leaf

```ts
const BYTES_PER_NODE = 32
const FLOATS_PER_NODE = 8
const IS_LEAF_FLAG = 0x80000000
const OFFSET_MASK = 0x7FFFFFFF

interface FlatBVH {
  nodeBuffer: ArrayBuffer
  nodeFloats: Float32Array    // View over nodeBuffer
  nodeUints: Uint32Array      // View over nodeBuffer
  nodeCount: number

  // The geometry's index buffer is reordered during construction
  // so leaf nodes reference contiguous triangle ranges
  reorderedIndex: Uint16Array | Uint32Array

  // Original geometry reference for intersection testing
  positions: Float32Array
}
```

### Why Flat Arrays?

1. **Cache locality**: Sequential memory access during traversal maximizes L1/L2 cache hits
2. **Zero GC pressure**: No object allocations, no pointer chasing
3. **Transferable**: Can be sent to Web Workers via `postMessage` with zero-copy transfer
4. **GPU uploadable**: Can be stored in a texture for GPU-side raycasting (WebGPU compute)

## BVH Construction

### SAH (Surface Area Heuristic)

SAH evaluates split candidates by estimating the expected ray traversal cost. For each potential split plane:

```
cost(split) = traversalCost + (SA_left / SA_parent) * N_left * intersectCost
                             + (SA_right / SA_parent) * N_right * intersectCost
```

Where:
- `SA` = surface area of the bounding box
- `N` = number of triangles
- `traversalCost` = cost of traversing one node (~1.0)
- `intersectCost` = cost of one triangle intersection test (~1.5)

If the cost of splitting exceeds the cost of testing all triangles in the current node, the node becomes a leaf.

### Binned SAH

Full SAH evaluates every triangle as a potential split. Binned SAH discretizes into 32 bins per axis for O(n) construction instead of O(n²):

```ts
const BIN_COUNT = 32

const buildBVH = (positions: Float32Array, indices: Uint32Array): FlatBVH => {
  // 1. Compute centroid and AABB for each triangle
  const triCount = indices.length / 3
  const centroids = new Float32Array(triCount * 3)
  const triBounds = new Float32Array(triCount * 6)  // minX,minY,minZ,maxX,maxY,maxZ
  computeTriangleData(positions, indices, centroids, triBounds)

  // 2. Allocate node buffer (max 2*triCount - 1 nodes)
  const maxNodes = triCount * 2 - 1
  const nodeBuffer = new ArrayBuffer(maxNodes * BYTES_PER_NODE)
  const nodeFloats = new Float32Array(nodeBuffer)
  const nodeUints = new Uint32Array(nodeBuffer)

  // 3. Index indirection array (will be reordered)
  const triIndices = new Uint32Array(triCount)
  for (let i = 0; i < triCount; i++) triIndices[i] = i

  // 4. Recursive build
  let nodeCount = 0

  const buildNode = (start: number, end: number): number => {
    const nodeIdx = nodeCount++
    const nodeOffset = nodeIdx * FLOATS_PER_NODE

    // Compute bounds of all triangles in [start, end)
    const bounds = computeBounds(triBounds, triIndices, start, end)
    nodeFloats[nodeOffset + 0] = bounds.minX
    nodeFloats[nodeOffset + 1] = bounds.minY
    nodeFloats[nodeOffset + 2] = bounds.minZ
    nodeFloats[nodeOffset + 3] = bounds.maxX
    nodeFloats[nodeOffset + 4] = bounds.maxY
    nodeFloats[nodeOffset + 5] = bounds.maxZ

    const count = end - start
    if (count <= MAX_LEAF_SIZE) {
      // Leaf node
      nodeUints[nodeOffset + 6] = IS_LEAF_FLAG | start
      nodeUints[nodeOffset + 7] = count
      return nodeIdx
    }

    // Find best split using binned SAH
    const { axis, splitPos, cost } = findBestSplit(
      centroids, triBounds, triIndices, start, end, bounds
    )

    // If splitting is more expensive than leaf, make leaf
    const leafCost = count * INTERSECT_COST
    if (cost >= leafCost) {
      nodeUints[nodeOffset + 6] = IS_LEAF_FLAG | start
      nodeUints[nodeOffset + 7] = count
      return nodeIdx
    }

    // Partition triangles
    const mid = partition(triIndices, centroids, start, end, axis, splitPos)

    // Build children (left child is adjacent, right child index stored)
    buildNode(start, mid)  // Left child = nodeIdx + 1
    const rightChild = buildNode(mid, end)
    nodeUints[nodeOffset + 6] = rightChild
    nodeUints[nodeOffset + 7] = axis << 30

    return nodeIdx
  }

  buildNode(0, triCount)

  // 5. Reorder the index buffer to match triIndices ordering
  const reorderedIndex = reorderIndexBuffer(indices, triIndices)

  return { nodeBuffer, nodeFloats, nodeUints, nodeCount, reorderedIndex, positions }
}
```

### Binned Split Finding

```ts
const findBestSplit = (
  centroids: Float32Array,
  triBounds: Float32Array,
  triIndices: Uint32Array,
  start: number,
  end: number,
  parentBounds: AABB
): { axis: number, splitPos: number, cost: number } => {
  let bestCost = Infinity
  let bestAxis = 0
  let bestPos = 0

  for (let axis = 0; axis < 3; axis++) {
    const min = parentBounds.min[axis]
    const max = parentBounds.max[axis]
    if (max - min < 1e-6) continue  // Degenerate axis

    // Initialize bins
    const bins = new Array(BIN_COUNT)
    for (let b = 0; b < BIN_COUNT; b++) {
      bins[b] = { count: 0, bounds: newAABB() }
    }

    // Assign triangles to bins based on centroid
    const scale = BIN_COUNT / (max - min)
    for (let i = start; i < end; i++) {
      const tri = triIndices[i]
      const c = centroids[tri * 3 + axis]
      const bin = Math.min(BIN_COUNT - 1, Math.floor((c - min) * scale))
      bins[bin].count++
      expandAABB(bins[bin].bounds, triBounds, tri)
    }

    // Sweep from left, accumulating bounds and counts
    // Then sweep from right, computing SAH cost at each split
    // ... (standard binned SAH evaluation)

    // Track best split
    if (bestBinCost < bestCost) {
      bestCost = bestBinCost
      bestAxis = axis
      bestPos = min + (bestBin + 1) * (max - min) / BIN_COUNT
    }
  }

  return { axis: bestAxis, splitPos: bestPos, cost: bestCost }
}
```

## Ray Traversal

### First-Hit Traversal (raycastFirst)

The most common operation: find the closest intersection along a ray. Uses a stack-based traversal with ordered child visits (near child first) and early termination when `tMax` shrinks past remaining nodes.

```ts
const raycastFirst = (
  bvh: FlatBVH,
  rayOrigin: Float32Array,
  rayDir: Float32Array,
  tMin: number,
  tMax: number
): RayHit | null => {
  const invDir = [1 / rayDir[0], 1 / rayDir[1], 1 / rayDir[2]]
  const dirIsNeg = [invDir[0] < 0, invDir[1] < 0, invDir[2] < 0]

  const stack = new Int32Array(64)  // Pre-allocated traversal stack
  let stackPtr = 0
  let nodeIdx = 0
  let closestHit: RayHit | null = null
  let closestT = tMax

  while (true) {
    const offset = nodeIdx * FLOATS_PER_NODE
    const meta = bvh.nodeUints[offset + 6]

    if (meta & IS_LEAF_FLAG) {
      // Leaf: test triangles
      const triStart = meta & OFFSET_MASK
      const triCount = bvh.nodeUints[offset + 7]

      for (let i = 0; i < triCount; i++) {
        const triIdx = triStart + i
        const hit = intersectTriangle(
          bvh.positions, bvh.reorderedIndex, triIdx,
          rayOrigin, rayDir, tMin, closestT
        )
        if (hit && hit.t < closestT) {
          closestT = hit.t
          closestHit = hit
        }
      }

      // Pop stack
      if (stackPtr === 0) break
      nodeIdx = stack[--stackPtr]
      continue
    }

    // Internal node: test both children's AABBs
    const leftIdx = nodeIdx + 1
    const rightIdx = meta
    const splitAxis = bvh.nodeUints[offset + 7] >>> 30

    const leftOffset = leftIdx * FLOATS_PER_NODE
    const rightOffset = rightIdx * FLOATS_PER_NODE

    const tLeftHit = intersectAABB(bvh.nodeFloats, leftOffset, rayOrigin, invDir, tMin, closestT)
    const tRightHit = intersectAABB(bvh.nodeFloats, rightOffset, rayOrigin, invDir, tMin, closestT)

    if (tLeftHit >= 0 && tRightHit >= 0) {
      // Both hit: visit closer one first, push farther
      if (tLeftHit < tRightHit) {
        stack[stackPtr++] = rightIdx
        nodeIdx = leftIdx
      } else {
        stack[stackPtr++] = leftIdx
        nodeIdx = rightIdx
      }
    } else if (tLeftHit >= 0) {
      nodeIdx = leftIdx
    } else if (tRightHit >= 0) {
      nodeIdx = rightIdx
    } else {
      // Neither hit: pop
      if (stackPtr === 0) break
      nodeIdx = stack[--stackPtr]
    }
  }

  return closestHit
}
```

### AABB-Ray Intersection (Slab Method)

```ts
const intersectAABB = (
  nodeFloats: Float32Array,
  offset: number,
  origin: Float32Array,
  invDir: number[],
  tMin: number,
  tMax: number
): number => {
  const minX = nodeFloats[offset + 0]
  const minY = nodeFloats[offset + 1]
  const minZ = nodeFloats[offset + 2]
  const maxX = nodeFloats[offset + 3]
  const maxY = nodeFloats[offset + 4]
  const maxZ = nodeFloats[offset + 5]

  let t0x = (minX - origin[0]) * invDir[0]
  let t1x = (maxX - origin[0]) * invDir[0]
  if (invDir[0] < 0) { const tmp = t0x; t0x = t1x; t1x = tmp }

  let t0y = (minY - origin[1]) * invDir[1]
  let t1y = (maxY - origin[1]) * invDir[1]
  if (invDir[1] < 0) { const tmp = t0y; t0y = t1y; t1y = tmp }

  let t0z = (minZ - origin[2]) * invDir[2]
  let t1z = (maxZ - origin[2]) * invDir[2]
  if (invDir[2] < 0) { const tmp = t0z; t0z = t1z; t1z = tmp }

  const tEnter = Math.max(t0x, t0y, t0z, tMin)
  const tExit = Math.min(t1x, t1y, t1z, tMax)

  return tExit >= tEnter ? tEnter : -1
}
```

### Triangle Intersection (Möller-Trumbore)

```ts
const intersectTriangle = (
  positions: Float32Array,
  indices: Uint32Array,
  triIdx: number,
  origin: Float32Array,
  dir: Float32Array,
  tMin: number,
  tMax: number
): RayHit | null => {
  const i0 = indices[triIdx * 3 + 0] * 3
  const i1 = indices[triIdx * 3 + 1] * 3
  const i2 = indices[triIdx * 3 + 2] * 3

  // Edge vectors
  const e1x = positions[i1] - positions[i0]
  const e1y = positions[i1 + 1] - positions[i0 + 1]
  const e1z = positions[i1 + 2] - positions[i0 + 2]
  const e2x = positions[i2] - positions[i0]
  const e2y = positions[i2 + 1] - positions[i0 + 1]
  const e2z = positions[i2 + 2] - positions[i0 + 2]

  // Cross product: dir × e2
  const px = dir[1] * e2z - dir[2] * e2y
  const py = dir[2] * e2x - dir[0] * e2z
  const pz = dir[0] * e2y - dir[1] * e2x

  const det = e1x * px + e1y * py + e1z * pz
  if (Math.abs(det) < 1e-10) return null  // Parallel

  const invDet = 1 / det

  // Barycentric u
  const tx = origin[0] - positions[i0]
  const ty = origin[1] - positions[i0 + 1]
  const tz = origin[2] - positions[i0 + 2]
  const u = (tx * px + ty * py + tz * pz) * invDet
  if (u < 0 || u > 1) return null

  // Barycentric v
  const qx = ty * e1z - tz * e1y
  const qy = tz * e1x - tx * e1z
  const qz = tx * e1y - ty * e1x
  const v = (dir[0] * qx + dir[1] * qy + dir[2] * qz) * invDet
  if (v < 0 || u + v > 1) return null

  // Distance t
  const t = (e2x * qx + e2y * qy + e2z * qz) * invDet
  if (t < tMin || t > tMax) return null

  return { t, u, v, triangleIndex: triIdx }
}
```

## Raycaster API

```ts
interface RayHit {
  t: number                    // Distance along ray
  u: number                    // Barycentric u
  v: number                    // Barycentric v
  triangleIndex: number        // Index of hit triangle
  point?: Float32Array         // World-space hit point (computed on demand)
  normal?: Float32Array        // Interpolated normal (computed on demand)
  object?: MeshNode            // The mesh that was hit
}

interface Raycaster {
  ray: { origin: Float32Array, direction: Float32Array }
  near: number
  far: number

  setFromCamera(coords: { x: number, y: number }, camera: CameraNode): void
  intersectObject(mesh: MeshNode, recursive?: boolean): RayHit[]
  intersectObjects(meshes: MeshNode[]): RayHit[]
}
```

### Usage

```ts
const raycaster = createRaycaster()

// From screen coordinates (normalized -1 to +1)
raycaster.setFromCamera({ x: mouseNDC.x, y: mouseNDC.y }, camera)

// Test against scene
const hits = raycaster.intersectObjects(scene.meshes)
if (hits.length > 0) {
  const closest = hits[0]  // Already sorted by distance
  console.log('Hit:', closest.object.name, 'at distance:', closest.t)
}
```

## BVH Construction Timing

| Triangle Count | SAH Build Time | Memory |
|---------------|---------------|--------|
| 1,000 | ~2ms | ~60KB |
| 10,000 | ~15ms | ~600KB |
| 100,000 | ~120ms | ~6MB |

For large models, BVH construction can be offloaded to a Web Worker. The resulting flat buffer is transferable back to the main thread with zero copy.

## Performance Target

**500 rays against 80,000 triangles in < 5ms** (comparable to three-mesh-bvh).

The flat array layout and ordered traversal with early termination make this achievable. Most rays terminate after visiting 10-30 nodes out of thousands.
