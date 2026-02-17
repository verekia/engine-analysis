# Raycasting & BVH Spatial Acceleration

This document covers Wren's raycasting system with SAH-based BVH spatial acceleration, plus frustum culling for performance.

## BVH Raycasting

### Overview

Wren implements a SAH-based (Surface Area Heuristic) Bounding Volume Hierarchy stored as a flat typed array for cache-friendly traversal. The design is inspired by three-mesh-bvh's approach: packed flat-array nodes, index buffer reordering, and optimized first-hit traversal with early termination.

### BVH Data Structure

#### Flat Array Layout

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

#### Why Flat Arrays?

1. **Cache locality**: Sequential memory access during traversal maximizes L1/L2 cache hits
2. **Zero GC pressure**: No object allocations, no pointer chasing
3. **Transferable**: Can be sent to Web Workers via `postMessage` with zero-copy transfer
4. **GPU uploadable**: Can be stored in a texture for GPU-side raycasting (WebGPU compute)

### BVH Construction

#### SAH (Surface Area Heuristic)

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

#### Binned SAH

Full SAH evaluates every triangle as a potential split. Binned SAH discretizes into 32 bins per axis for O(n) construction instead of O(n²). See file 11-raycasting.md for full implementation details.

### Ray Traversal

#### First-Hit Traversal (raycastFirst)

The most common operation: find the closest intersection along a ray. Uses a stack-based traversal with ordered child visits (near child first) and early termination when `tMax` shrinks past remaining nodes. See file 11-raycasting.md for full implementation.

#### AABB-Ray Intersection (Slab Method)

Fast rejection test using the slab method. See file 11-raycasting.md for implementation.

#### Triangle Intersection (Möller-Trumbore)

Efficient triangle-ray intersection. See file 11-raycasting.md for implementation.

### Raycaster API

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

### BVH Construction Timing

| Triangle Count | SAH Build Time | Memory |
|---------------|---------------|--------|
| 1,000 | ~2ms | ~60KB |
| 10,000 | ~15ms | ~600KB |
| 100,000 | ~120ms | ~6MB |

For large models, BVH construction can be offloaded to a Web Worker. The resulting flat buffer is transferable back to the main thread with zero copy.

### Performance Target

**500 rays against 80,000 triangles in < 5ms** (comparable to three-mesh-bvh).

The flat array layout and ordered traversal with early termination make this achievable. Most rays terminate after visiting 10-30 nodes out of thousands.

## Frustum Culling

### Overview

Before each frame, every renderable's world-space AABB is tested against the camera's view frustum. Objects fully outside the frustum are skipped entirely — no draw command is generated for them.

### Frustum Extraction

The six frustum planes are extracted directly from the view-projection matrix (Gribb/Hartmann method):

```ts
interface Frustum {
  planes: Float32Array  // 6 planes × 4 components (nx, ny, nz, d) = 24 floats
}

const extractFrustum = (viewProjection: Float32Array, frustum: Frustum): void => {
  const m = viewProjection
  const p = frustum.planes

  // Left:   row3 + row0
  p[0]  = m[3]  + m[0];  p[1]  = m[7]  + m[4];  p[2]  = m[11] + m[8];   p[3]  = m[15] + m[12]
  // Right:  row3 - row0
  p[4]  = m[3]  - m[0];  p[5]  = m[7]  - m[4];  p[6]  = m[11] - m[8];   p[7]  = m[15] - m[12]
  // Bottom: row3 + row1
  p[8]  = m[3]  + m[1];  p[9]  = m[7]  + m[5];  p[10] = m[11] + m[9];   p[11] = m[15] + m[13]
  // Top:    row3 - row1
  p[12] = m[3]  - m[1];  p[13] = m[7]  - m[5];  p[14] = m[11] - m[9];   p[15] = m[15] - m[13]
  // Near:   row3 + row2
  p[16] = m[3]  + m[2];  p[17] = m[7]  + m[6];  p[18] = m[11] + m[10];  p[19] = m[15] + m[14]
  // Far:    row3 - row2
  p[20] = m[3]  - m[2];  p[21] = m[7]  - m[6];  p[22] = m[11] - m[10];  p[23] = m[15] - m[14]

  // Normalize each plane
  for (let i = 0; i < 6; i++) {
    const o = i * 4
    const len = Math.sqrt(p[o] * p[o] + p[o+1] * p[o+1] + p[o+2] * p[o+2])
    p[o] /= len; p[o+1] /= len; p[o+2] /= len; p[o+3] /= len
  }
}
```

### AABB-Frustum Test

Each AABB is tested against all 6 planes. If the AABB is fully outside any plane, it's culled. We use the "p-vertex/n-vertex" optimization for fast rejection:

```ts
const isAABBInFrustum = (frustum: Frustum, aabb: AABB): boolean => {
  const p = frustum.planes
  const min = aabb.min  // [x, y, z]
  const max = aabb.max  // [x, y, z]

  for (let i = 0; i < 6; i++) {
    const o = i * 4
    const nx = p[o], ny = p[o+1], nz = p[o+2], d = p[o+3]

    // P-vertex: the corner of the AABB most in the direction of the plane normal
    const px = nx >= 0 ? max[0] : min[0]
    const py = ny >= 0 ? max[1] : min[1]
    const pz = nz >= 0 ? max[2] : min[2]

    // If the p-vertex is behind the plane, the entire AABB is outside
    if (nx * px + ny * py + nz * pz + d < 0) {
      return false
    }
  }

  return true
}
```

### Culling Loop

```ts
const cullScene = (renderables: RenderableList, frustum: Frustum): number => {
  let visibleCount = 0

  for (let i = 0; i < renderables.count; i++) {
    const node = renderables.meshes[i]

    if (!node.visible) {
      node.culled = true
      continue
    }

    if (node.worldAABB && !isAABBInFrustum(frustum, node.worldAABB)) {
      node.culled = true
      continue
    }

    node.culled = false
    visibleCount++
  }

  return visibleCount
}
```

For 2000 objects, this takes < 0.1ms. The test is 6 dot products per AABB with early exit.

### WebGPU Compute Culling (Optional Enhancement)

When WebGPU is available, frustum culling can run as a compute shader:

```wgsl
@group(0) @binding(0) var<storage, read> objectAABBs: array<AABB>;
@group(0) @binding(1) var<storage, read_write> visibilityFlags: array<u32>;
@group(0) @binding(2) var<uniform> frustumPlanes: array<vec4f, 6>;

@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) id: vec3u) {
  let idx = id.x;
  if (idx >= arrayLength(&objectAABBs)) { return; }

  let aabb = objectAABBs[idx];
  var visible = 1u;

  for (var p = 0u; p < 6u; p++) {
    let plane = frustumPlanes[p];
    let pVertex = vec3f(
      select(aabb.min.x, aabb.max.x, plane.x >= 0.0),
      select(aabb.min.y, aabb.max.y, plane.y >= 0.0),
      select(aabb.min.z, aabb.max.z, plane.z >= 0.0)
    );
    if (dot(plane.xyz, pVertex) + plane.w < 0.0) {
      visible = 0u;
      break;
    }
  }

  visibilityFlags[idx] = visible;
}
```

This moves culling entirely off the main thread. The visibility buffer is read back or used directly for indirect draw calls.
