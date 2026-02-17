# Raycasting & BVH

## Goals

Provide raycasting performance matching `three-mesh-bvh`:
- Build BVH for any mesh geometry in milliseconds
- Raycast against thousands of triangles in microseconds
- Support both mesh-level and triangle-level intersection
- Zero allocation during ray traversal
- Efficient for both static and dynamic scenes

## BVH Architecture

### SAH-Based Construction

Caracal uses the **Surface Area Heuristic (SAH)** to build optimally balanced BVH trees. SAH minimizes the expected cost of ray traversal by choosing split planes that minimize the probability of a ray intersecting child nodes:

```
SAH cost = traversalCost + (leftArea / parentArea) × leftCount × intersectCost
                         + (rightArea / parentArea) × rightCount × intersectCost
```

At each node, the builder:
1. Evaluates potential split planes along all 3 axes
2. For each axis, tests multiple candidate split positions (typically 12-16 bins)
3. Chooses the split that minimizes the SAH cost
4. Partitions triangles into left/right children
5. Recurses until leaf nodes have ≤ 4 triangles

### Binned SAH

Full SAH evaluation (testing every triangle as a split candidate) is O(n²). Caracal uses **binned SAH**, which discretizes the axis into fixed bins:

```typescript
const buildBVHNode = (
  triangles: Uint32Array,   // Triangle indices
  bounds: Float32Array,     // Triangle AABBs (6 floats each)
  centroids: Float32Array,  // Triangle centroids (3 floats each)
  start: number,
  count: number
): BVHNode => {
  if (count <= MAX_LEAF_SIZE) {
    return createLeafNode(triangles, start, count)
  }

  // Compute centroid bounds for this node
  const centroidBounds = computeCentroidBounds(centroids, start, count)

  let bestCost = Infinity
  let bestAxis = 0
  let bestSplit = 0

  // Try each axis
  for (let axis = 0; axis < 3; axis++) {
    const axisMin = centroidBounds[axis]
    const axisMax = centroidBounds[axis + 3]

    if (axisMax - axisMin < 1e-8) continue // Degenerate axis

    // Bin triangles along this axis
    const bins = new Array(NUM_BINS) // NUM_BINS = 16
    for (let i = start; i < start + count; i++) {
      const centroid = centroids[triangles[i] * 3 + axis]
      const binIdx = Math.min(
        NUM_BINS - 1,
        Math.floor(NUM_BINS * (centroid - axisMin) / (axisMax - axisMin))
      )
      bins[binIdx].count++
      bins[binIdx].bounds.expand(bounds, triangles[i])
    }

    // Evaluate SAH cost for each split between bins
    for (let split = 1; split < NUM_BINS; split++) {
      const leftBounds = mergeBins(bins, 0, split)
      const rightBounds = mergeBins(bins, split, NUM_BINS)
      const leftCount = countBins(bins, 0, split)
      const rightCount = countBins(bins, split, NUM_BINS)

      const cost = TRAVERSAL_COST +
        leftBounds.surfaceArea() / parentBounds.surfaceArea() * leftCount * INTERSECT_COST +
        rightBounds.surfaceArea() / parentBounds.surfaceArea() * rightCount * INTERSECT_COST

      if (cost < bestCost) {
        bestCost = cost
        bestAxis = axis
        bestSplit = split
      }
    }
  }

  // Partition triangles around best split
  partition(triangles, centroids, start, count, bestAxis, bestSplit, centroidBounds)

  // Recurse
  const leftCount = /* triangles in left partition */
  const left = buildBVHNode(triangles, bounds, centroids, start, leftCount)
  const right = buildBVHNode(triangles, bounds, centroids, start + leftCount, count - leftCount)

  return createInternalNode(left, right)
}
```

### Flat BVH Layout

After construction, the tree is flattened into a contiguous array for cache-friendly traversal. Each node is 32 bytes:

```
Flat BVH Node (32 bytes):
┌──────────────────────────────────────────┐
│ boundsMinX (f32) │ boundsMinY (f32)      │
│ boundsMinZ (f32) │ boundsMaxX (f32)      │
│ boundsMaxY (f32) │ boundsMaxZ (f32)      │
│ offset (u32)     │ count (u32)           │
└──────────────────────────────────────────┘
```

- For **internal nodes**: `count = 0`, `offset = index of right child` (left child is always the next node)
- For **leaf nodes**: `count > 0`, `offset = start index into triangle array`

This layout means:
- Left child traversal = increment index by 1 (sequential memory access)
- Right child traversal = jump to `offset` (skip pointer)
- No pointers, no objects — pure flat data

```typescript
interface FlatBVH {
  nodes: Float32Array     // Packed node data (8 floats per node)
  triangles: Uint32Array  // Triangle indices (reordered by BVH)
  nodeCount: number
}
```

## Ray Traversal

### Stackless Traversal

Traditional BVH traversal uses a stack to track which nodes to visit. Caracal uses a **stackless** approach with skip pointers — each node knows where to jump if the ray misses:

```typescript
const raycast = (
  bvh: FlatBVH,
  positions: Float32Array,    // Vertex positions
  indices: Uint32Array,       // Triangle indices
  origin: Vec3,
  direction: Vec3,
  maxDistance: number = Infinity
): RayHit | null => {
  // Pre-compute inverse direction for slab test
  const invDirX = 1 / direction.x
  const invDirY = 1 / direction.y
  const invDirZ = 1 / direction.z

  let nodeIndex = 0
  let closestHit: RayHit | null = null
  let closestDistance = maxDistance

  while (nodeIndex < bvh.nodeCount) {
    const nodeOffset = nodeIndex * 8
    const nodes = bvh.nodes

    // AABB-ray intersection (slab test)
    const minX = nodes[nodeOffset + 0]
    const minY = nodes[nodeOffset + 1]
    const minZ = nodes[nodeOffset + 2]
    const maxX = nodes[nodeOffset + 3]
    const maxY = nodes[nodeOffset + 4]
    const maxZ = nodes[nodeOffset + 5]

    const hitAABB = rayIntersectsAABB(
      origin, invDirX, invDirY, invDirZ,
      minX, minY, minZ, maxX, maxY, maxZ,
      closestDistance
    )

    const rightChildOffset = nodes[nodeOffset + 6] | 0  // as integer
    const triCount = nodes[nodeOffset + 7] | 0           // as integer

    if (!hitAABB) {
      // Miss: skip this subtree
      if (triCount > 0) {
        // Leaf: move to next sibling
        nodeIndex++
      } else {
        // Internal: jump to right child's skip target
        nodeIndex = rightChildOffset
      }
      continue
    }

    if (triCount > 0) {
      // Leaf node: test triangles
      const triStart = rightChildOffset
      for (let i = 0; i < triCount; i++) {
        const triIdx = bvh.triangles[triStart + i]
        const hit = rayTriangleIntersect(
          origin, direction,
          positions, indices,
          triIdx,
          closestDistance
        )
        if (hit && hit.distance < closestDistance) {
          closestDistance = hit.distance
          closestHit = hit
        }
      }
      nodeIndex++
    } else {
      // Internal node: descend to left child (next in array)
      nodeIndex++
    }
  }

  return closestHit
}
```

### Ray-AABB Intersection (Slab Test)

The optimized slab test for AABB-ray intersection:

```typescript
const rayIntersectsAABB = (
  originX: number, originY: number, originZ: number,
  invDirX: number, invDirY: number, invDirZ: number,
  minX: number, minY: number, minZ: number,
  maxX: number, maxY: number, maxZ: number,
  maxDist: number
): boolean => {
  let tmin = (minX - originX) * invDirX
  let tmax = (maxX - originX) * invDirX
  if (tmin > tmax) { const t = tmin; tmin = tmax; tmax = t }

  let tymin = (minY - originY) * invDirY
  let tymax = (maxY - originY) * invDirY
  if (tymin > tymax) { const t = tymin; tymin = tymax; tymax = t }

  if (tmin > tymax || tymin > tmax) return false
  if (tymin > tmin) tmin = tymin
  if (tymax < tmax) tmax = tymax

  let tzmin = (minZ - originZ) * invDirZ
  let tzmax = (maxZ - originZ) * invDirZ
  if (tzmin > tzmax) { const t = tzmin; tzmin = tzmax; tzmax = t }

  if (tmin > tzmax || tzmin > tmax) return false
  if (tzmin > tmin) tmin = tzmin
  if (tzmax < tmax) tmax = tzmax

  return tmax >= 0 && tmin < maxDist
}
```

### Ray-Triangle Intersection (Möller-Trumbore)

The standard fast ray-triangle intersection algorithm:

```typescript
const rayTriangleIntersect = (
  origin: Vec3,
  direction: Vec3,
  positions: Float32Array,
  indices: Uint32Array,
  triIndex: number,
  maxDist: number
): RayHit | null => {
  const i0 = indices[triIndex * 3] * 3
  const i1 = indices[triIndex * 3 + 1] * 3
  const i2 = indices[triIndex * 3 + 2] * 3

  // Triangle vertices
  const v0x = positions[i0], v0y = positions[i0 + 1], v0z = positions[i0 + 2]
  const v1x = positions[i1], v1y = positions[i1 + 1], v1z = positions[i1 + 2]
  const v2x = positions[i2], v2y = positions[i2 + 1], v2z = positions[i2 + 2]

  // Edges
  const e1x = v1x - v0x, e1y = v1y - v0y, e1z = v1z - v0z
  const e2x = v2x - v0x, e2y = v2y - v0y, e2z = v2z - v0z

  // Cross product: direction × e2
  const px = direction.y * e2z - direction.z * e2y
  const py = direction.z * e2x - direction.x * e2z
  const pz = direction.x * e2y - direction.y * e2x

  const det = e1x * px + e1y * py + e1z * pz
  if (Math.abs(det) < 1e-10) return null

  const invDet = 1 / det

  // Barycentric u
  const tx = origin.x - v0x, ty = origin.y - v0y, tz = origin.z - v0z
  const u = (tx * px + ty * py + tz * pz) * invDet
  if (u < 0 || u > 1) return null

  // Barycentric v
  const qx = ty * e1z - tz * e1y
  const qy = tz * e1x - tx * e1z
  const qz = tx * e1y - ty * e1x
  const v = (direction.x * qx + direction.y * qy + direction.z * qz) * invDet
  if (v < 0 || u + v > 1) return null

  // Distance
  const t = (e2x * qx + e2y * qy + e2z * qz) * invDet
  if (t < 0 || t > maxDist) return null

  return {
    distance: t,
    point: vec3AddScaled(_hitPoint, origin, direction, t),
    triangleIndex: triIndex,
    barycentricU: u,
    barycentricV: v,
    faceNormal: computeFaceNormal(e1x, e1y, e1z, e2x, e2y, e2z),
  }
}
```

## High-Level Raycaster API

```typescript
interface Raycaster {
  origin: Vec3
  direction: Vec3
  near: number        // default: 0
  far: number         // default: Infinity

  // Set ray from camera + screen coordinates
  setFromCamera(camera: Camera, screenX: number, screenY: number): void

  // Cast against a single mesh (uses its BVH)
  intersectMesh(mesh: Mesh): RayHit | null
  intersectMeshAll(mesh: Mesh): RayHit[]

  // Cast against all meshes in the scene
  intersectScene(scene: Scene): SceneRayHit | null
  intersectSceneAll(scene: Scene): SceneRayHit[]
}

interface RayHit {
  distance: number
  point: Vec3
  triangleIndex: number
  barycentricU: number
  barycentricV: number
  faceNormal: Vec3
}

interface SceneRayHit extends RayHit {
  mesh: Mesh
}
```

### Usage

```typescript
const raycaster = createRaycaster()

// From mouse position
const onMouseClick = (event: MouseEvent) => {
  const x = (event.clientX / canvas.width) * 2 - 1
  const y = -((event.clientY / canvas.height) * 2 - 1)

  raycaster.setFromCamera(camera, x, y)

  const hit = raycaster.intersectScene(scene)
  if (hit) {
    console.log(`Hit ${hit.mesh.name} at distance ${hit.distance}`)
    console.log(`Triangle: ${hit.triangleIndex}`)
    console.log(`World point: ${hit.point}`)
  }
}
```

### Scene-Level Raycasting

For scene raycasting, a two-level strategy is used:

1. **Coarse phase**: Test ray against each mesh's world AABB (fast rejection)
2. **Fine phase**: For meshes that pass the AABB test, transform ray to mesh local space and traverse the mesh's BVH

```typescript
const intersectScene = (raycaster: Raycaster, scene: Scene): SceneRayHit | null => {
  let closestHit: SceneRayHit | null = null
  let closestDist = raycaster.far

  for (const mesh of scene.allMeshes) {
    // Coarse: test world AABB
    if (!rayIntersectsAABB(raycaster.origin, raycaster.direction, mesh._worldAABB)) {
      continue
    }

    // Transform ray to mesh local space
    const invWorld = mat4Invert(_m0, mesh._worldMatrix)
    const localOrigin = vec3TransformMat4(_v0, raycaster.origin, invWorld)
    const localDir = vec3TransformDirection(_v1, raycaster.direction, invWorld)

    // Fine: BVH traversal in local space
    const hit = raycastBVH(mesh._bvh, mesh.geometry, localOrigin, localDir, closestDist)

    if (hit && hit.distance < closestDist) {
      closestDist = hit.distance
      closestHit = {
        ...hit,
        mesh,
        point: vec3TransformMat4(hit.point, hit.point, mesh._worldMatrix), // back to world
      }
    }
  }

  return closestHit
}
```

## BVH Construction Timing

BVH construction is designed to be fast enough for runtime use:

| Triangle Count | Build Time (approx) |
|---------------|---------------------|
| 1,000 | < 1 ms |
| 10,000 | ~3 ms |
| 100,000 | ~20 ms |
| 1,000,000 | ~150 ms |

For static meshes, the BVH is built once at load time. For dynamic meshes (deformable), the BVH can be rebuilt per frame for small meshes or use a two-level approach (top-level per-object, bottom-level static per-mesh).

## BVH Auto-Building

When a geometry is used for raycasting, its BVH is built automatically on first access:

```typescript
// Lazy BVH construction
const getMeshBVH = (mesh: Mesh): FlatBVH => {
  if (!mesh._bvh) {
    mesh._bvh = buildBVH(
      mesh.geometry.attributes.get('position')!.array as Float32Array,
      mesh.geometry.index?.array as Uint32Array
    )
  }
  return mesh._bvh
}
```

Users can also pre-build BVHs for important meshes to avoid hitches:

```typescript
// Pre-build BVH at load time
mesh.geometry.buildBVH()
```
