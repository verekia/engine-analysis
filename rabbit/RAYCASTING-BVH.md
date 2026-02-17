# Culling & Raycasting

## Frustum Culling

### Overview

Every frame, the renderer tests each node's world-space AABB against the camera frustum. Nodes outside the frustum are skipped entirely — no draw calls, no uniform uploads, nothing.

This is critical for performance: in a scene with 2000 objects, culling might reduce the visible set to 500-800, saving 1200+ draw calls.

### Frustum Extraction

The frustum is extracted from the camera's view-projection matrix as six planes:

```typescript
class Frustum {
  readonly planes: [MathPlane, MathPlane, MathPlane, MathPlane, MathPlane, MathPlane]
  // [left, right, bottom, top, near, far]

  extractFromMatrix(viewProjection: Mat4): void {
    const m = viewProjection.elements

    // Left plane: row4 + row1
    this.planes[0].set(m[3] + m[0], m[7] + m[4], m[11] + m[8], m[15] + m[12])
    // Right plane: row4 - row1
    this.planes[1].set(m[3] - m[0], m[7] - m[4], m[11] - m[8], m[15] - m[12])
    // Bottom plane: row4 + row2
    this.planes[2].set(m[3] + m[1], m[7] + m[5], m[11] + m[9], m[15] + m[13])
    // Top plane: row4 - row2
    this.planes[3].set(m[3] - m[1], m[7] - m[5], m[11] - m[9], m[15] - m[13])
    // Near plane: row4 + row3
    this.planes[4].set(m[3] + m[2], m[7] + m[6], m[11] + m[10], m[15] + m[14])
    // Far plane: row4 - row3
    this.planes[5].set(m[3] - m[2], m[7] - m[6], m[11] - m[10], m[15] - m[14])

    // Normalize all planes
    for (const plane of this.planes) {
      plane.normalize()
    }
  }
}
```

### AABB-Frustum Test

The test uses the "p-vertex / n-vertex" method (fastest for AABB vs. plane):

```typescript
const intersectsAABB = (frustum: Frustum, aabb: AABB): boolean => {
  for (const plane of frustum.planes) {
    // Find the "positive vertex" — the AABB corner most aligned with the plane normal
    const px = plane.normal.x >= 0 ? aabb.max.x : aabb.min.x
    const py = plane.normal.y >= 0 ? aabb.max.y : aabb.min.y
    const pz = plane.normal.z >= 0 ? aabb.max.z : aabb.min.z

    // If the p-vertex is behind the plane, the AABB is outside
    const dist = plane.normal.x * px + plane.normal.y * py + plane.normal.z * pz + plane.d
    if (dist < 0) return false
  }
  return true
}
```

This is a conservative test — it may report some objects as visible when they're not (false positives at frustum corners), but never misses a visible object. The false positive rate is low and the test is extremely fast (6 plane dot products, early exit).

### Hierarchical Culling

The scene graph enables hierarchical culling: if a parent's AABB is fully outside the frustum, all children are also outside. This prunes entire subtrees early.

```typescript
const collectVisible = (
  node: Node,
  frustum: Frustum,
  opaqueList: RenderItem[],
  transparentList: RenderItem[],
) => {
  if (!node.visible) return

  // If this node has a bounding box, test it
  if (node.worldAABB && !frustum.intersectsAABB(node.worldAABB)) {
    return // entire subtree culled
  }

  // If this is a renderable mesh, add to the appropriate list
  if (node instanceof Mesh) {
    const list = node.material.transparent ? transparentList : opaqueList
    list.push(createRenderItem(node))
  }

  // Recurse into children
  for (const child of node.children) {
    collectVisible(child, frustum, opaqueList, transparentList)
  }
}
```

For Group nodes without explicit bounding boxes, we could compute a merged AABB of children. However, for the typical case (flat or shallow hierarchies), per-mesh AABB testing is sufficient and avoids the overhead of maintaining group AABBs.

## BVH (Bounding Volume Hierarchy)

### Purpose

The BVH provides fast raycasting against triangle meshes. Without a BVH, raycasting against a 10K-triangle mesh requires 10K ray-triangle tests. With a SAH-built BVH, this drops to ~10-20 tests on average.

This is comparable to `three-mesh-bvh` performance.

### Architecture

```
Raycaster
  │
  ├─ Scene-level: test ray against all Mesh worldAABBs (linear scan)
  │                                                      │
  │   For each hit AABB:                                 │
  ├─ Mesh-level: test ray against mesh BVH              ▼
  │              (SAH binary tree of triangles)     [coarse candidates]
  │                                                      │
  └─ Return: sorted list of intersections               ▼
             (distance, point, normal, mesh, triangleIndex)
```

### BVH Construction

The BVH is built lazily on first raycast against a mesh. Construction uses the Surface Area Heuristic (SAH) for high-quality splits.

```typescript
interface BVHNode {
  aabb: AABB
  left: BVHNode | null      // null for leaf nodes
  right: BVHNode | null
  firstTriangle: number     // leaf only: index of first triangle
  triangleCount: number     // leaf only: number of triangles (0 for internal nodes)
}
```

#### SAH Split

For each internal node, we evaluate potential split planes along all three axes and choose the one that minimizes the SAH cost:

```
SAH cost = traversalCost + (SA_left / SA_parent) * count_left * intersectionCost
                         + (SA_right / SA_parent) * count_right * intersectionCost
```

Where `SA` is surface area of the AABB. This produces trees that closely match `three-mesh-bvh` quality.

```typescript
const buildBVH = (triangles: Triangle[], maxLeafSize: number = 4): BVHNode => {
  const build = (start: number, end: number): BVHNode => {
    const count = end - start
    const aabb = computeAABB(triangles, start, end)

    // Leaf node
    if (count <= maxLeafSize) {
      return { aabb, left: null, right: null, firstTriangle: start, triangleCount: count }
    }

    // Try splits along each axis, pick best SAH cost
    let bestCost = Infinity
    let bestAxis = 0
    let bestSplit = 0

    const parentSA = aabb.surfaceArea()

    for (let axis = 0; axis < 3; axis++) {
      // Sort triangles by centroid along this axis
      const sorted = triangles.slice(start, end)
        .sort((a, b) => a.centroid[axis] - b.centroid[axis])

      // Sweep from left, accumulating AABB surface areas
      // Use binned SAH for O(n) instead of O(n²):
      const binCount = 16
      const bins = createBins(sorted, axis, binCount, aabb)

      for (let i = 0; i < binCount - 1; i++) {
        const leftSA = bins.leftSA[i]
        const rightSA = bins.rightSA[i]
        const leftCount = bins.leftCount[i]
        const rightCount = bins.rightCount[i]

        const cost = 1 + (leftSA / parentSA) * leftCount + (rightSA / parentSA) * rightCount
        if (cost < bestCost) {
          bestCost = cost
          bestAxis = axis
          bestSplit = bins.splitPosition[i]
        }
      }
    }

    // Partition triangles at the best split
    const mid = partition(triangles, start, end, bestAxis, bestSplit)

    // Guard against degenerate splits
    if (mid === start || mid === end) {
      return { aabb, left: null, right: null, firstTriangle: start, triangleCount: count }
    }

    return {
      aabb,
      left: build(start, mid),
      right: build(mid, end),
      firstTriangle: 0,
      triangleCount: 0,
    }
  }

  return build(0, triangles.length)
}
```

### Flat BVH Layout (Cache-Friendly)

After construction, the tree is flattened into a contiguous array for cache-friendly traversal:

```typescript
interface FlatBVHNode {
  minX: number; minY: number; minZ: number  // AABB min
  maxX: number; maxY: number; maxZ: number  // AABB max
  leftChild: number    // index of left child, or -1 for leaf
  firstTriangle: number
  triangleCount: number
}

// Stored as a Float32Array for cache efficiency
// Layout: [minX, minY, minZ, maxX, maxY, maxZ, leftChild_as_float, firstTri_as_float, triCount_as_float] × N
```

The flat layout ensures BVH nodes are contiguous in memory, minimizing cache misses during traversal.

### Ray-BVH Traversal

Iterative traversal with an explicit stack (avoids function call overhead):

```typescript
const intersectBVH = (
  ray: Ray,
  bvh: FlatBVHNode[],
  triangles: Triangle[],
): Intersection | null => {
  const stack: number[] = new Array(64) // pre-allocated stack
  let stackPtr = 0
  stack[stackPtr++] = 0 // start at root

  let closest: Intersection | null = null
  let closestDist = Infinity

  while (stackPtr > 0) {
    const nodeIdx = stack[--stackPtr]
    const node = bvh[nodeIdx]

    // Test ray against node AABB
    const tHit = rayIntersectsAABB(ray, node)
    if (tHit < 0 || tHit > closestDist) continue // miss or too far

    if (node.triangleCount > 0) {
      // Leaf: test triangles
      for (let i = 0; i < node.triangleCount; i++) {
        const tri = triangles[node.firstTriangle + i]
        const t = rayIntersectsTriangle(ray, tri)
        if (t > 0 && t < closestDist) {
          closestDist = t
          closest = {
            distance: t,
            point: ray.at(t),
            normal: tri.normal,
            triangleIndex: node.firstTriangle + i,
          }
        }
      }
    } else {
      // Internal: push children (push far child first so near child is popped first)
      const leftChild = node.leftChild
      const rightChild = leftChild + 1

      const tLeft = rayIntersectsAABB(ray, bvh[leftChild])
      const tRight = rayIntersectsAABB(ray, bvh[rightChild])

      // Traverse nearer child first
      if (tLeft >= 0 && tRight >= 0) {
        if (tLeft < tRight) {
          stack[stackPtr++] = rightChild
          stack[stackPtr++] = leftChild
        } else {
          stack[stackPtr++] = leftChild
          stack[stackPtr++] = rightChild
        }
      } else if (tLeft >= 0) {
        stack[stackPtr++] = leftChild
      } else if (tRight >= 0) {
        stack[stackPtr++] = rightChild
      }
    }
  }

  return closest
}
```

### Ray-AABB Intersection (Slab Method)

```typescript
const rayIntersectsAABB = (ray: Ray, aabb: { minX: number, ... }): number => {
  const invDirX = 1 / ray.direction.x
  const invDirY = 1 / ray.direction.y
  const invDirZ = 1 / ray.direction.z

  const t1 = (aabb.minX - ray.origin.x) * invDirX
  const t2 = (aabb.maxX - ray.origin.x) * invDirX
  const t3 = (aabb.minY - ray.origin.y) * invDirY
  const t4 = (aabb.maxY - ray.origin.y) * invDirY
  const t5 = (aabb.minZ - ray.origin.z) * invDirZ
  const t6 = (aabb.maxZ - ray.origin.z) * invDirZ

  const tmin = Math.max(Math.min(t1, t2), Math.min(t3, t4), Math.min(t5, t6))
  const tmax = Math.min(Math.max(t1, t2), Math.max(t3, t4), Math.max(t5, t6))

  if (tmax < 0 || tmin > tmax) return -1
  return tmin >= 0 ? tmin : tmax
}
```

### Ray-Triangle Intersection (Möller–Trumbore)

```typescript
const rayIntersectsTriangle = (ray: Ray, tri: Triangle): number => {
  const edge1 = _v0.subVectors(tri.b, tri.a)
  const edge2 = _v1.subVectors(tri.c, tri.a)
  const h = _v2.crossVectors(ray.direction, edge2)
  const a = edge1.dot(h)

  if (a > -1e-8 && a < 1e-8) return -1 // parallel

  const f = 1 / a
  const s = _v3.subVectors(ray.origin, tri.a)
  const u = f * s.dot(h)
  if (u < 0 || u > 1) return -1

  const q = _v4.crossVectors(s, edge1)
  const v = f * ray.direction.dot(q)
  if (v < 0 || u + v > 1) return -1

  const t = f * edge2.dot(q)
  return t > 1e-8 ? t : -1
}
```

## Raycaster API

```typescript
class Raycaster {
  ray: Ray
  near: number   // min distance, default 0
  far: number    // max distance, default Infinity

  // Set ray from camera + mouse position (normalized device coordinates)
  setFromCamera(coords: { x: number, y: number }, camera: Camera): void {
    // coords: x,y in [-1, 1] (NDC)
    const invVP = _tempMat4.inverse(camera.viewProjectionMatrix)

    // Near plane point
    this.ray.origin.set(coords.x, coords.y, -1).applyMatrix4(invVP)

    // Far plane point
    const farPoint = _tempVec3.set(coords.x, coords.y, 1).applyMatrix4(invVP)

    this.ray.direction.subVectors(farPoint, this.ray.origin).normalize()
  }

  // Test against scene
  intersectObjects(
    objects: Node[],
    recursive: boolean = true,
  ): Intersection[] {
    const results: Intersection[] = []

    for (const obj of objects) {
      this.intersectObject(obj, recursive, results)
    }

    // Sort by distance (nearest first)
    results.sort((a, b) => a.distance - b.distance)
    return results
  }

  private intersectObject(
    obj: Node,
    recursive: boolean,
    results: Intersection[],
  ): void {
    if (obj instanceof Mesh) {
      // Quick AABB test first
      if (!this.ray.intersectsAABB(obj.worldAABB)) return

      // Transform ray to mesh local space
      const invWorld = _tempMat4.inverse(obj.worldMatrix)
      const localRay = _tempRay.copy(this.ray).applyMatrix4(invWorld)

      // Build BVH lazily
      if (!obj.bvh) {
        obj.bvh = buildBVH(obj.geometry)
      }

      // Test against BVH
      const hit = intersectBVH(localRay, obj.bvh.nodes, obj.bvh.triangles)
      if (hit && hit.distance >= this.near && hit.distance <= this.far) {
        // Transform hit back to world space
        hit.point.applyMatrix4(obj.worldMatrix)
        hit.normal.applyMatrix3(obj.worldMatrix).normalize()
        hit.distance = hit.point.distanceTo(this.ray.origin)
        hit.object = obj
        results.push(hit)
      }
    }

    if (recursive) {
      for (const child of obj.children) {
        this.intersectObject(child, recursive, results)
      }
    }
  }
}
```

### Intersection Result

```typescript
interface Intersection {
  distance: number          // distance from ray origin
  point: Vec3               // world-space hit point
  normal: Vec3              // world-space surface normal
  triangleIndex: number     // index of the hit triangle
  object: Mesh              // the mesh that was hit
}
```
