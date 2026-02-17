# Spatial — BVH, Raycasting, Frustum Culling

## Overview

Fennec's spatial module provides three capabilities: a **flat-array BVH** for fast raycasting (matching three-mesh-bvh performance), **frustum culling** for per-frame visibility determination, and efficient **AABB** operations. The BVH supports both static meshes and refit for skinned meshes.

## Bounding Volume Hierarchy (BVH)

### Design Goals

- Match or exceed three-mesh-bvh in raycast performance
- Flat array layout for cache-efficient traversal
- SAH (Surface Area Heuristic) construction for optimal tree quality
- Support per-triangle raycasting for mesh hit detection
- Support refit for animated meshes (update bounds without rebuild)

### BVH Node Layout

The BVH uses a flat array of nodes instead of a pointer-based tree. Each node is 32 bytes, cache-line aligned:

```typescript
// Flat BVH node layout in a Float32Array
// Each node = 8 floats (32 bytes)
//
// Offset  | Field                    | Notes
// --------|--------------------------|------
//  0-2    | AABB min (x, y, z)       |
//  3      | left child / tri offset  | If leaf: triangle start index
//  4-6    | AABB max (x, y, z)       |
//  7      | right child / tri count  | If leaf: negative triangle count
//
// Leaf detection: node[7] < 0 means this is a leaf with abs(node[7]) triangles

const BVH_STRIDE = 8
const BVH_MIN_X = 0
const BVH_MIN_Y = 1
const BVH_MIN_Z = 2
const BVH_LEFT = 3     // or triangle offset for leaves
const BVH_MAX_X = 4
const BVH_MAX_Y = 5
const BVH_MAX_Z = 6
const BVH_RIGHT = 7    // or negative triangle count for leaves
```

### SAH Construction

The Surface Area Heuristic builds a high-quality tree by choosing split planes that minimize expected traversal cost:

```typescript
interface BVHBuildOptions {
  maxLeafTriangles?: number     // Max triangles per leaf (default: 4)
  sahBins?: number              // Number of SAH bins (default: 32)
}

const buildBVH = (positions: Float32Array, indices: Uint32Array, options?: BVHBuildOptions): BVHData => {
  const maxLeaf = options?.maxLeafTriangles ?? 4
  const bins = options?.sahBins ?? 32

  // Pre-compute triangle centroids and AABBs
  const triCount = indices.length / 3
  const triCentroids = new Float32Array(triCount * 3)
  const triAABBs = new Float32Array(triCount * 6) // min xyz, max xyz

  for (let i = 0; i < triCount; i++) {
    const i0 = indices[i * 3] * 3
    const i1 = indices[i * 3 + 1] * 3
    const i2 = indices[i * 3 + 2] * 3
    // Compute triangle AABB and centroid...
  }

  // Recursive SAH build
  const nodes: number[] = []
  const triIndices: number[] = [] // Reordered triangle indices

  const buildNode = (start: number, end: number): number => {
    const nodeIndex = nodes.length / BVH_STRIDE
    nodes.length += BVH_STRIDE

    // Compute AABB for this range
    const aabb = computeAABB(triAABBs, start, end)

    if (end - start <= maxLeaf) {
      // Leaf node
      const triOffset = triIndices.length
      for (let i = start; i < end; i++) {
        triIndices.push(i)
      }
      setLeafNode(nodes, nodeIndex, aabb, triOffset, end - start)
      return nodeIndex
    }

    // Find best SAH split
    const { axis, splitPos, cost } = findBestSplit(triCentroids, triAABBs, start, end, aabb, bins)

    // Partition triangles
    const mid = partition(triCentroids, triAABBs, start, end, axis, splitPos)

    if (mid === start || mid === end) {
      // SAH couldn't split, make leaf
      // ...
    }

    // Recurse
    const left = buildNode(start, mid)
    const right = buildNode(mid, end)
    setInternalNode(nodes, nodeIndex, aabb, left, right)
    return nodeIndex
  }

  buildNode(0, triCount)

  return {
    nodes: new Float32Array(nodes),
    triIndices: new Uint32Array(triIndices),
  }
}
```

### SAH Cost Function

```typescript
const sahCost = (leftArea: number, leftCount: number, rightArea: number, rightCount: number, parentArea: number): number => {
  const TRAVERSAL_COST = 1.0
  const INTERSECT_COST = 1.0
  return TRAVERSAL_COST + INTERSECT_COST * (
    (leftArea / parentArea) * leftCount +
    (rightArea / parentArea) * rightCount
  )
}
```

### Binned SAH

Instead of testing every triangle as a potential split, we bin triangles into `N` buckets per axis and evaluate SAH cost at each bin boundary. This reduces construction from O(n^2) to O(n * bins):

```typescript
const findBestSplit = (centroids: Float32Array, aabbs: Float32Array, start: number, end: number, nodeAABB: AABB, bins: number) => {
  let bestCost = Infinity
  let bestAxis = 0
  let bestSplitPos = 0

  for (let axis = 0; axis < 3; axis++) {
    const min = nodeAABB.min[axis]
    const max = nodeAABB.max[axis]
    if (max - min < 1e-6) continue

    const binSize = (max - min) / bins
    const binCounts = new Uint32Array(bins)
    const binAABBs = new Float32Array(bins * 6) // Initialize to empty

    // Assign triangles to bins
    for (let i = start; i < end; i++) {
      const c = centroids[i * 3 + axis]
      const bin = Math.min(Math.floor((c - min) / binSize), bins - 1)
      binCounts[bin]++
      expandAABB(binAABBs, bin, aabbs, i)
    }

    // Sweep from left, accumulating counts and AABBs
    // Evaluate SAH cost at each boundary
    // Track minimum cost split
    // ...
  }

  return { axis: bestAxis, splitPos: bestSplitPos, cost: bestCost }
}
```

## Raycasting

### Ray-BVH Traversal

The traversal uses a stack-based iterative approach (no recursion) for performance:

```typescript
interface RaycastHit {
  distance: number
  point: Vec3
  normal: Vec3
  triangleIndex: number
  faceIndex: number
  object: Mesh
}

const raycast = (ray: Ray, bvh: BVHData, positions: Float32Array, indices: Uint32Array, worldMatrix: Mat4): RaycastHit | null => {
  // Transform ray to local space
  const localRay = transformRay(ray, mat4.invert(_tempMat4, worldMatrix))

  const { nodes, triIndices } = bvh
  const stack = _raycastStack // Pre-allocated Int32Array(64)
  let stackPtr = 0
  stack[stackPtr++] = 0 // Start at root

  let closestHit: RaycastHit | null = null
  let closestDist = Infinity

  while (stackPtr > 0) {
    const nodeIdx = stack[--stackPtr] * BVH_STRIDE

    // Test ray vs node AABB
    const tMin = rayAABBIntersect(localRay, nodes, nodeIdx)
    if (tMin > closestDist || tMin < 0) continue

    const rightOrCount = nodes[nodeIdx + BVH_RIGHT]

    if (rightOrCount < 0) {
      // Leaf node: test triangles
      const triOffset = nodes[nodeIdx + BVH_LEFT]
      const triCount = -rightOrCount

      for (let i = 0; i < triCount; i++) {
        const triIdx = triIndices[triOffset + i]
        const hit = rayTriangleIntersect(localRay, positions, indices, triIdx)
        if (hit && hit.distance < closestDist) {
          closestDist = hit.distance
          closestHit = hit
        }
      }
    } else {
      // Internal node: push children (far child first for front-to-back)
      const left = nodes[nodeIdx + BVH_LEFT]
      const right = rightOrCount

      // Push far child first so near child is popped first
      const leftDist = rayAABBIntersect(localRay, nodes, left * BVH_STRIDE)
      const rightDist = rayAABBIntersect(localRay, nodes, right * BVH_STRIDE)

      if (leftDist < rightDist) {
        if (rightDist < closestDist) stack[stackPtr++] = right
        if (leftDist < closestDist) stack[stackPtr++] = left
      } else {
        if (leftDist < closestDist) stack[stackPtr++] = left
        if (rightDist < closestDist) stack[stackPtr++] = right
      }
    }
  }

  if (closestHit) {
    // Transform hit point and normal back to world space
    vec3.transformMat4(closestHit.point, closestHit.point, worldMatrix)
    vec3.transformMat3(closestHit.normal, closestHit.normal, mat3.normalFromMat4(_tempMat3, worldMatrix))
    vec3.normalize(closestHit.normal, closestHit.normal)
    closestHit.distance = vec3.distance(ray.origin, closestHit.point)
  }

  return closestHit
}
```

### Moller-Trumbore Ray-Triangle Intersection

```typescript
const rayTriangleIntersect = (ray: Ray, positions: Float32Array, indices: Uint32Array, triIdx: number): { distance: number, u: number, v: number } | null => {
  const i0 = indices[triIdx * 3] * 3
  const i1 = indices[triIdx * 3 + 1] * 3
  const i2 = indices[triIdx * 3 + 2] * 3

  // Edge vectors
  const e1x = positions[i1] - positions[i0]
  const e1y = positions[i1 + 1] - positions[i0 + 1]
  const e1z = positions[i1 + 2] - positions[i0 + 2]
  const e2x = positions[i2] - positions[i0]
  const e2y = positions[i2 + 1] - positions[i0 + 1]
  const e2z = positions[i2 + 2] - positions[i0 + 2]

  // Cross product of ray direction and e2
  const px = ray.direction[1] * e2z - ray.direction[2] * e2y
  const py = ray.direction[2] * e2x - ray.direction[0] * e2z
  const pz = ray.direction[0] * e2y - ray.direction[1] * e2x

  const det = e1x * px + e1y * py + e1z * pz
  if (Math.abs(det) < 1e-10) return null

  const invDet = 1 / det

  const tx = ray.origin[0] - positions[i0]
  const ty = ray.origin[1] - positions[i0 + 1]
  const tz = ray.origin[2] - positions[i0 + 2]

  const u = (tx * px + ty * py + tz * pz) * invDet
  if (u < 0 || u > 1) return null

  const qx = ty * e1z - tz * e1y
  const qy = tz * e1x - tx * e1z
  const qz = tx * e1y - ty * e1x

  const v = (ray.direction[0] * qx + ray.direction[1] * qy + ray.direction[2] * qz) * invDet
  if (v < 0 || u + v > 1) return null

  const t = (e2x * qx + e2y * qy + e2z * qz) * invDet
  if (t < 0) return null

  return { distance: t, u, v }
}
```

### Scene-Level Raycasting

Raycasting the full scene first tests object-level AABBs, then BVH traversal only for objects that pass the AABB test:

```typescript
const raycastScene = (ray: Ray, scene: Scene): RaycastHit[] => {
  const hits: RaycastHit[] = []

  for (const mesh of scene._allMeshes) {
    if (!mesh.visible) continue

    // Quick AABB rejection against world-space bounding box
    if (!rayAABBTest(ray, mesh.worldBoundingBox)) continue

    // Detailed BVH raycast
    const hit = raycast(ray, mesh.geometry.bvh, mesh.geometry.positions, mesh.geometry.indices, mesh.worldMatrix)
    if (hit) {
      hit.object = mesh
      hits.push(hit)
    }
  }

  // Sort by distance
  hits.sort((a, b) => a.distance - b.distance)
  return hits
}
```

### BVH Refit for Skinned Meshes

Skinned meshes change shape every frame. Full BVH rebuild is too expensive. Instead, we **refit**: keep the same tree structure but update leaf AABBs bottom-up:

```typescript
const refitBVH = (bvh: BVHData, skinnedPositions: Float32Array, indices: Uint32Array) => {
  const { nodes, triIndices } = bvh
  const nodeCount = nodes.length / BVH_STRIDE

  // Process nodes bottom-up (leaves first, then internal nodes)
  for (let i = nodeCount - 1; i >= 0; i--) {
    const offset = i * BVH_STRIDE
    const rightOrCount = nodes[offset + BVH_RIGHT]

    if (rightOrCount < 0) {
      // Leaf: recompute AABB from triangles
      const triOffset = nodes[offset + BVH_LEFT]
      const triCount = -rightOrCount
      recomputeLeafAABB(nodes, offset, skinnedPositions, indices, triIndices, triOffset, triCount)
    } else {
      // Internal: union of child AABBs
      const left = nodes[offset + BVH_LEFT] * BVH_STRIDE
      const right = rightOrCount * BVH_STRIDE
      nodes[offset + BVH_MIN_X] = Math.min(nodes[left + BVH_MIN_X], nodes[right + BVH_MIN_X])
      nodes[offset + BVH_MIN_Y] = Math.min(nodes[left + BVH_MIN_Y], nodes[right + BVH_MIN_Y])
      nodes[offset + BVH_MIN_Z] = Math.min(nodes[left + BVH_MIN_Z], nodes[right + BVH_MIN_Z])
      nodes[offset + BVH_MAX_X] = Math.max(nodes[left + BVH_MAX_X], nodes[right + BVH_MAX_X])
      nodes[offset + BVH_MAX_Y] = Math.max(nodes[left + BVH_MAX_Y], nodes[right + BVH_MAX_Y])
      nodes[offset + BVH_MAX_Z] = Math.max(nodes[left + BVH_MAX_Z], nodes[right + BVH_MAX_Z])
    }
  }
}
```

## Frustum Culling

### Frustum Representation

The camera frustum is 6 planes in world space, extracted from the view-projection matrix:

```typescript
const extractFrustumPlanes = (vp: Mat4, frustum: Frustum) => {
  // Left plane
  frustum.planes[0][0] = vp[3] + vp[0]
  frustum.planes[0][1] = vp[7] + vp[4]
  frustum.planes[0][2] = vp[11] + vp[8]
  frustum.planes[0][3] = vp[15] + vp[12]
  normalizePlane(frustum.planes[0])

  // Right plane
  frustum.planes[1][0] = vp[3] - vp[0]
  // ... (similar for all 6 planes: left, right, bottom, top, near, far)
}
```

### AABB vs Frustum Test

Test whether a world-space AABB intersects the frustum. Returns `true` if any part of the AABB is inside:

```typescript
const aabbInFrustum = (aabb: AABB, frustum: Frustum): boolean => {
  for (let i = 0; i < 6; i++) {
    const plane = frustum.planes[i]

    // Find the AABB vertex most in the direction of the plane normal
    const px = plane[0] > 0 ? aabb.max[0] : aabb.min[0]
    const py = plane[1] > 0 ? aabb.max[1] : aabb.min[1]
    const pz = plane[2] > 0 ? aabb.max[2] : aabb.min[2]

    // If the "most positive" vertex is outside this plane, the whole AABB is outside
    if (plane[0] * px + plane[1] * py + plane[2] * pz + plane[3] < 0) {
      return false
    }
  }
  return true
}
```

### Culling Pass

Frustum culling runs once per frame over all meshes in the scene. The result is a flat list of visible meshes:

```typescript
const frustumCull = (meshes: Mesh[], frustum: Frustum, output: Mesh[]): number => {
  let count = 0
  for (let i = 0; i < meshes.length; i++) {
    const mesh = meshes[i]
    if (!mesh.visible) continue
    if (aabbInFrustum(mesh.worldBoundingBox, frustum)) {
      output[count++] = mesh
    }
  }
  return count
}
```

The `output` array is pre-allocated to `scene._allMeshes.length` to avoid per-frame allocation. The returned count indicates how many entries are valid.

### Performance

| Operation | Time (2000 objects) |
|-----------|---------------------|
| Frustum culling | ~0.05ms |
| BVH construction (50K tri mesh) | ~15ms (one-time) |
| BVH raycast (50K tri mesh) | ~0.01ms per ray |
| BVH refit (50K tri mesh) | ~1ms |
| Scene raycast (2000 objects) | ~0.5ms (with AABB pre-filter) |
