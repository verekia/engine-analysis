# Raycasting & BVH

## Status: Out of Scope for v1

Raycasting and BVH spatial acceleration are out of scope for v1. Basic interaction can use bounding spheres; precise raycasting will be added in v2.

## Basic Raycasting (v1)

For simple interaction, use sphere-ray intersection with entity bounding spheres:

```ts
function raycastSphere(
  rayOrigin: vec3,
  rayDir: vec3,
  sphereCenter: vec3,
  sphereRadius: number
): number | null {
  // Returns distance along ray, or null if no hit
}
```

Iterate all entities (or visible entities only) and find closest hit:

```ts
function raycast(ray: Ray): Entity | null {
  let closestDist = Infinity
  let closestEntity = null

  for (let i = 0; i < entityCount; i++) {
    const dist = raycastSphere(
      ray.origin,
      ray.direction,
      boundingSpheres[i].center,
      boundingSpheres[i].radius
    )
    if (dist !== null && dist < closestDist) {
      closestDist = dist
      closestEntity = i
    }
  }

  return closestEntity
}
```

This is O(N) but fast enough for <10k entities (typically only raycast on click, not every frame).

## BVH Spatial Acceleration (v2)

For precise raycasting and larger scenes, use a BVH (Bounding Volume Hierarchy):

### Goals

- three-mesh-bvh level quality and performance
- Support for triangle-level intersection
- Dynamic BVH rebuild for moving objects
- GPU acceleration (optional, for very large scenes)

### BVH Structure

```ts
interface BVHNode {
  bounds: AABB           // Axis-aligned bounding box
  left: BVHNode | null
  right: BVHNode | null
  primitives: number[]   // Leaf: triangle indices; Internal: empty
}
```

### Build Strategy

- SAH (Surface Area Heuristic) for optimal splits
- Rebuild incrementally (only dirty subtrees)
- Refit for small changes (update bounds without rebuild)

### Raycasting with BVH

```ts
function raycastBVH(ray: Ray, node: BVHNode): Hit | null {
  if (!intersectAABB(ray, node.bounds)) return null

  if (node.isLeaf()) {
    // Test ray against triangles
    return raycastTriangles(ray, node.primitives)
  } else {
    // Recurse into children
    const hitLeft = raycastBVH(ray, node.left)
    const hitRight = raycastBVH(ray, node.right)
    return closestHit(hitLeft, hitRight)
  }
}
```

### Integration with three-mesh-bvh

Alternatively, use [three-mesh-bvh](https://github.com/gkjohnson/three-mesh-bvh) as a library:

- Build BVH from mesh geometry
- Raycast against BVH
- Extract hit information (distance, normal, UV, face index)

```ts
import { MeshBVH } from 'three-mesh-bvh'

const bvh = new MeshBVH(geometry)
const hit = bvh.raycastFirst(ray)
```

This avoids reimplementing a complex BVH from scratch.

## Frustum Culling with BVH (v2)

BVH can also accelerate frustum culling:

- Test frustum against BVH nodes
- Cull entire subtrees if node is outside frustum
- Only test leaf entities against frustum planes

Expected speedup: 2-5x for large scenes (10k+ entities)

## Other Use Cases

BVH can be used for:
- **Physics broadphase** (collision detection)
- **AI queries** (find nearest entity, all entities in radius)
- **LOD selection** (select LOD based on distance, using BVH traversal)
- **Occlusion culling** (test visibility against occluders)

## Performance Target (v2)

- BVH build: <5ms for 100k triangles (can be done async)
- Raycasting: <0.1ms for typical scene (camera ray into 10k entities)
- Frustum culling with BVH: <1ms for 50k entities

## Implementation Priority

v2 feature. For v1, sphere-based raycasting is sufficient for basic interaction.
