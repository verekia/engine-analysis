# Scene Graph

## Design Goals

1. **Intuitive API**: Parent/children hierarchy like three.js — `parent.add(child)`, `group.add(mesh)`.
2. **High-performance traversal**: Flat iteration array alongside the tree structure.
3. **Minimal per-frame cost**: Dirty flag system so only changed transforms are recomputed.
4. **Z-up right-handed**: All defaults assume `+Z` is up.

## Object3D

Every entity in the scene extends `Object3D`:

```typescript
interface Object3D {
  // Identity
  readonly id: number          // unique auto-incrementing ID
  name: string                 // optional human-readable name

  // Local transform
  position: Vec3               // default [0, 0, 0]
  rotation: Quat               // default identity
  scale: Vec3                  // default [1, 1, 1]

  // Computed matrices (backed by SoA Float32Array)
  readonly localMatrix: Mat4   // position × rotation × scale
  readonly worldMatrix: Mat4   // parent.worldMatrix × localMatrix

  // Hierarchy
  parent: Object3D | null
  readonly children: readonly Object3D[]

  // Methods
  add(child: Object3D): void
  remove(child: Object3D): void
  traverse(callback: (obj: Object3D) => void): void
  getWorldPosition(out: Vec3): Vec3

  // Visibility and culling
  visible: boolean             // default true — skips rendering + children
  frustumCulled: boolean       // default true — participate in frustum culling
  readonly boundingBox: AABB   // world-space AABB (auto-computed for Mesh)

  // Render state
  renderOrder: number          // manual override for draw order (default 0)
}
```

## Transform Storage: Structure-of-Arrays

Individual `Object3D` instances expose `position`, `rotation`, `scale`, `localMatrix`, `worldMatrix` as friendly objects, but under the hood these are **views into contiguous typed arrays**.

```typescript
// Global SoA storage — one entry per Object3D in the scene
const MAX_OBJECTS = 8192

const positions = new Float32Array(MAX_OBJECTS * 3)    // x, y, z
const rotations = new Float32Array(MAX_OBJECTS * 4)    // qx, qy, qz, qw
const scales    = new Float32Array(MAX_OBJECTS * 3)    // sx, sy, sz
const localMats = new Float32Array(MAX_OBJECTS * 16)   // 4×4 local matrix
const worldMats = new Float32Array(MAX_OBJECTS * 16)   // 4×4 world matrix
const dirtyFlags = new Uint8Array(MAX_OBJECTS)         // 1 = needs recompute
```

When an `Object3D` is created, it receives an index `i` into these arrays. Its `position` is a `Vec3` backed by `positions.subarray(i*3, i*3+3)`. Writes to `position.x` directly mutate the SoA array and set `dirtyFlags[i] = 1`.

Benefits:
- **Cache-friendly**: Matrix updates iterate linearly over contiguous memory.
- **Zero allocation**: No temporary `Matrix4` objects created during traversal.
- **GPU upload**: The entire `worldMats` array can be uploaded as a single buffer.

## Dirty Flag Propagation

When a transform property is mutated, the object and all its descendants are marked dirty:

```
setPosition(x, y, z):
  positions[i*3+0] = x
  positions[i*3+1] = y
  positions[i*3+2] = z
  markDirty(i)

markDirty(i):
  if dirtyFlags[i] === 1: return  // already dirty
  dirtyFlags[i] = 1
  for each child c of object[i]:
    markDirty(c.index)
```

Matrix recomputation happens once per frame, before rendering:

```
updateWorldMatrices(root):
  // Breadth-first traversal of scene graph
  for each object in traversal order:
    if dirtyFlags[i] === 0: continue
    composeLocalMatrix(i)  // TRS → localMat
    if parent:
      multiplyMatrices(worldMats, parent.index, localMats, i, worldMats, i)
    else:
      copyMatrix(localMats, i, worldMats, i)
    dirtyFlags[i] = 0
```

**Critical optimization**: Children are always traversed after their parent (breadth-first or sorted by depth). This guarantees the parent's world matrix is up-to-date before the child reads it.

## Hierarchy Operations

### add(child)

```
parent.add(child):
  if child.parent: child.parent.remove(child)
  child.parent = parent
  parent.children.push(child)
  scene.registerObject(child)    // add to flat render list
  markDirty(child.index)         // world matrix needs recompute
```

### remove(child)

```
parent.remove(child):
  splice child from parent.children
  child.parent = null
  scene.unregisterObject(child)  // remove from flat render list
```

### traverse(callback)

```
object.traverse(fn):
  fn(object)
  for child of object.children:
    child.traverse(fn)
```

## Group

`Group` is an `Object3D` with no geometry or material. It serves purely as a transform container for organizing hierarchy:

```typescript
const buildings = new Group()
buildings.position.set(10, 20, 0)
scene.add(buildings)

buildings.add(tower)
buildings.add(house)
// tower and house inherit buildings' transform
```

## Scene

`Scene` is the root `Object3D`. It also maintains:

```typescript
interface Scene extends Object3D {
  // Flat lists for efficient iteration
  readonly meshes: Mesh[]           // all meshes in scene
  readonly skinnedMeshes: SkinnedMesh[]  // all skinned meshes
  readonly lights: Light[]          // all lights
  readonly renderList: RenderItem[] // sorted draw list

  // Spatial index
  readonly bvh: BVH                 // scene-level BVH of mesh AABBs

  // Background
  backgroundColor: Vec3
}
```

When objects are added/removed, these flat lists are updated. The `renderList` is the sorted array the renderer iterates.

## Frustum Culling

Before rendering, visible objects are determined:

```
frustumCull(scene, camera):
  frustum = extractFrustumPlanes(camera.viewProjection)

  // Option A: Brute force (fine for < 500 objects)
  for each mesh in scene.meshes:
    mesh._visible = frustum.intersectsAABB(mesh.worldBoundingBox)

  // Option B: BVH query (better for > 500 objects)
  visibleSet = scene.bvh.frustumQuery(frustum)
  for each mesh in scene.meshes:
    mesh._visible = visibleSet.has(mesh)
```

The `_visible` flag is read by the render loop to skip invisible draw calls. The render list itself is not modified — only the per-mesh visibility flag changes.

## World-Space Bounding Boxes

Each `Mesh` computes a world-space AABB from its geometry's local AABB and its world matrix:

```
mesh.updateBoundingBox():
  localAABB = mesh.geometry.boundingBox  // computed once from vertex positions
  mesh.worldBoundingBox = transformAABB(localAABB, mesh.worldMatrix)
```

This is done lazily (only when the world matrix changes) using the same dirty flag system.

## Object Lifecycle

```
1. const mesh = new Mesh(geometry, material)
   → Allocates SoA index
   → Computes local bounding box from geometry

2. scene.add(mesh)
   → Sets parent, adds to children
   → Adds to scene.meshes, scene.renderList
   → Inserts into scene BVH
   → Marks dirty

3. scene.remove(mesh)
   → Removes from parent.children
   → Removes from scene.meshes, scene.renderList
   → Removes from scene BVH

4. mesh.dispose()
   → Releases SoA index back to pool
   → Does NOT dispose geometry/material (may be shared)
```

## Helper: lookAt

Since Z is up, `lookAt` orients an object to face a target:

```typescript
// lookAt assumes:
// - forward = +Y (into screen in default view)
// - up = +Z
// - right = +X
object.lookAt(target: Vec3): void
```

## Coordinate System Reference

```
        +Z (up)
         |
         |
         |
         +------→ +X (right)
        /
       /
      +Y (forward)
```

Right-handed: cross(+X, +Y) = +Z.
