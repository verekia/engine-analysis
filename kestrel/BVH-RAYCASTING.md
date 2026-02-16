# BVH and Raycasting

## Design Goals

- **Match three-mesh-bvh performance**: SAH-based construction, flat array traversal.
- **Dual purpose**: Used for both raycasting and frustum culling.
- **Two levels**: Scene-level BVH (mesh AABBs) and per-mesh BVH (triangle-level) for raycasting.
- **Dynamic support**: Incremental refit for moving objects.

## BVH Structure

### Flat Array Layout

The BVH uses a **flat array layout** instead of pointer-based nodes. This is critical for cache efficiency — traversal reads contiguous memory instead of chasing pointers.

```typescript
interface BVH {
  // Flat node array — 32 bytes per node
  readonly nodes: Float32Array
  // nodes layout per node (8 floats):
  //   [0-2]: AABB min (x, y, z)
  //   [3]:   left child index (or -1 for leaf → primitiveIndex)
  //   [4-6]: AABB max (x, y, z)
  //   [7]:   right child index (or primitiveCount for leaf)

  readonly primitiveIndices: Uint32Array  // maps leaf primitives to original indices
}
```

Node layout (32 bytes, cache-line friendly):
```
Offset  0: minX (f32)
Offset  4: minY (f32)
Offset  8: minZ (f32)
Offset 12: leftChild / primitiveOffset (i32)
Offset 16: maxX (f32)
Offset 20: maxY (f32)
Offset 24: maxZ (f32)
Offset 28: rightChild / primitiveCount (i32)
```

A node is a leaf if `primitiveCount > 0` (stored in offset 28). Otherwise, it's an internal node with `leftChild` and `rightChild` indices.

## Construction: Surface Area Heuristic (SAH)

SAH minimizes the expected cost of ray traversal by considering the surface area of each potential split:

```
buildBVH(primitives, start, end):
  if end - start <= LEAF_SIZE:
    return createLeaf(primitives, start, end)

  // Try splits along each axis
  bestCost = Infinity
  bestAxis = -1
  bestSplit = -1

  for axis in [0, 1, 2]:  // X, Y, Z
    sort primitives[start..end] by centroid[axis]

    // Sweep from left, accumulating AABB and count
    for i in start..end-1:
      leftAABB = union of primitives[start..i+1]
      rightAABB = union of primitives[i+1..end]
      leftCount = i - start + 1
      rightCount = end - i - 1

      cost = leftAABB.surfaceArea * leftCount + rightAABB.surfaceArea * rightCount

      if cost < bestCost:
        bestCost = cost
        bestAxis = axis
        bestSplit = i + 1

  // If splitting is more expensive than a leaf, make a leaf
  if bestCost >= (end - start) * parentAABB.surfaceArea:
    return createLeaf(primitives, start, end)

  // Partition and recurse
  sort primitives[start..end] by centroid[bestAxis]
  node.left = buildBVH(primitives, start, bestSplit)
  node.right = buildBVH(primitives, bestSplit, end)
  return node
```

### Binned SAH (Optimization)

For large primitive counts, full SAH is O(n²). Binned SAH reduces this to O(n):

```
BINS = 12

for axis in [0, 1, 2]:
  // Distribute primitives into bins along this axis
  for each primitive:
    binIndex = floor((centroid[axis] - min[axis]) / (max[axis] - min[axis]) * BINS)
    bins[binIndex].count++
    bins[binIndex].aabb.expand(primitive.aabb)

  // Sweep bins to find best split
  for split in 1..BINS:
    leftAABB = union(bins[0..split])
    rightAABB = union(bins[split..BINS])
    cost = leftAABB.surfaceArea * leftCount + rightAABB.surfaceArea * rightCount
```

12 bins gives results within 1–2% of full SAH at a fraction of the cost.

## Scene-Level BVH

The scene-level BVH indexes **mesh world-space AABBs**. Used for:
- Frustum culling (which meshes are visible?)
- Coarse raycasting (which meshes does the ray potentially hit?)

```typescript
const sceneBVH = new BVH()
sceneBVH.build(scene.meshes.map(m => m.worldBoundingBox))

// Frustum query
const visibleMeshes = sceneBVH.frustumQuery(camera.frustum)

// Coarse raycast
const candidates = sceneBVH.raycast(ray)
```

### Incremental Refit

When objects move, their AABBs change. Instead of rebuilding the entire BVH:

```
refit(node):
  if node.isLeaf:
    node.aabb = primitive.currentAABB  // updated world AABB
  else:
    refit(node.left)
    refit(node.right)
    node.aabb = union(node.left.aabb, node.right.aabb)
```

Refit is O(n) but preserves the tree structure. For moderate movement (game characters walking around), this is sufficient. If objects teleport dramatically, a full rebuild is triggered.

### Rebuild Heuristic

```
if averageAABBGrowth > 2.0:  // AABBs have expanded significantly
  fullRebuild()
else:
  refit()
```

## Per-Mesh BVH (Triangle-Level)

For precise raycasting (click detection, physics), each mesh can have a triangle-level BVH:

```typescript
const meshBVH = new TriangleBVH()
meshBVH.build(geometry.positions, geometry.indices)

// Precise raycast in local space
const hit = meshBVH.raycast(localRay)
```

### Triangle Intersection

Uses Moller-Trumbore algorithm:

```typescript
const rayTriangleIntersect = (
  rayOrigin: Vec3,
  rayDirection: Vec3,
  v0: Vec3, v1: Vec3, v2: Vec3
): { t: number, u: number, v: number } | null => {
  const edge1 = vec3.subtract(pool.vec3(), v1, v0)
  const edge2 = vec3.subtract(pool.vec3(), v2, v0)
  const h = vec3.cross(pool.vec3(), rayDirection, edge2)
  const a = vec3.dot(edge1, h)

  if (Math.abs(a) < 1e-8) return null  // parallel

  const f = 1.0 / a
  const s = vec3.subtract(pool.vec3(), rayOrigin, v0)
  const u = f * vec3.dot(s, h)
  if (u < 0.0 || u > 1.0) return null

  const q = vec3.cross(pool.vec3(), s, edge1)
  const v = f * vec3.dot(rayDirection, q)
  if (v < 0.0 || u + v > 1.0) return null

  const t = f * vec3.dot(edge2, q)
  if (t < 1e-8) return null

  return { t, u, v }
}
```

All temporary vectors come from the math pool — zero allocation during raycasting.

## Raycasting API

```typescript
// Cast a ray from screen coordinates
const ray = camera.screenToRay(mouseX, mouseY)

// Scene-level raycast (returns all hit meshes with intersection info)
const hits = scene.raycast(ray, {
  recursive: true,           // traverse children
  firstHitOnly: false,       // return all hits (true = early exit)
  maxDistance: 1000,          // ignore hits beyond this distance
  layers: 0xFFFFFFFF,        // bitmask filter
})

// hits is sorted by distance:
// [{ object: Mesh, distance: number, point: Vec3, normal: Vec3, faceIndex: number }]
```

### Raycast Flow

```
scene.raycast(worldRay):
  // 1. Scene BVH: coarse test against mesh AABBs
  candidates = sceneBVH.raycast(worldRay)

  // 2. For each candidate mesh, transform ray to local space
  hits = []
  for mesh in candidates:
    localRay = transformRay(worldRay, mesh.worldMatrix.inverse)

    // 3. If mesh has a triangle BVH, use it
    if mesh.geometry.bvh:
      localHit = mesh.geometry.bvh.raycast(localRay)
      if localHit:
        // Transform hit back to world space
        worldHit = transformHit(localHit, mesh.worldMatrix)
        hits.push(worldHit)

    // 4. Otherwise, brute-force triangle test
    else:
      localHit = bruteForceTriangleTest(localRay, mesh.geometry)
      ...

  // 5. Sort by distance
  hits.sort((a, b) => a.distance - b.distance)
  return hits
```

## Frustum Culling via BVH

```
frustumQuery(node, frustum, results):
  // Test node AABB against frustum
  result = frustum.intersectsAABB(node.aabb)

  if result === OUTSIDE:
    return  // entire subtree is invisible

  if result === INSIDE:
    // entire subtree is visible — add all leaves without further testing
    addAllLeaves(node, results)
    return

  // INTERSECTING — need to test children
  if node.isLeaf:
    results.push(node.primitiveIndex)
  else:
    frustumQuery(node.left, frustum, results)
    frustumQuery(node.right, frustum, results)
```

This prunes entire branches of the tree when they're fully inside or outside the frustum, reducing typical test count from O(n) to O(log n).

## Performance Characteristics

| Operation | Without BVH | With BVH |
|---|---|---|
| Frustum cull (2000 objects) | 12,000 plane tests | ~400 plane tests |
| Raycast (scene, 2000 objects) | 2000 AABB tests | ~20 AABB tests |
| Raycast (50K triangle mesh) | 50,000 triangle tests | ~50 triangle tests |
| Build (2000 objects) | N/A | ~2ms |
| Refit (2000 objects) | N/A | ~0.5ms |

## Build Options

```typescript
interface BVHOptions {
  maxLeafSize?: number       // default 4 — primitives per leaf
  sahBins?: number           // default 12 — bins for binned SAH
  strategy?: 'sah' | 'median'  // default 'sah' — split strategy
}
```

`median` split is faster to build but produces worse trees. Use it for dynamic BVHs that are rebuilt frequently. `sah` is used for static or semi-static content.
