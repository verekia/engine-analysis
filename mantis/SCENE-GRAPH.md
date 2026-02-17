# Scene Graph

## Design Principles

1. **SoA (Structure-of-Arrays)** — All transform data lives in contiguous typed
   arrays for cache-friendly traversal and direct GPU upload.
2. **Dirty flag propagation** — Only changed nodes recompute world matrices.
3. **Breadth-first update** — Parents always process before children, guaranteeing
   correct matrix multiplication order.
4. **Flat render list** — After traversal, the renderer sees a flat array of
   drawable objects, not a tree.

## Node Types

All scene objects inherit from `Node`:

```
Node (base)
├── Group           — Empty transform node for hierarchy (like THREE.Group)
├── Mesh            — Geometry + Material, renderable
├── SkinnedMesh     — Mesh + Skeleton, with GPU skinning
├── Camera          — Perspective projection, defines the viewpoint
├── DirectionalLight — Direction + color + intensity + shadow config
└── AmbientLight    — Color + intensity
```

Each node has:
- A **slot index** into the SoA arrays (assigned on addition to scene)
- Local transform (position, rotation, scale)
- Parent index (-1 for root nodes)
- Children indices (sparse array of child slot indices)
- A name (optional, for lookup)
- Visibility flag (`visible: boolean`)

## SoA Transform Storage

All transforms are stored in parallel typed arrays indexed by node slot:

```typescript
// Capacity grows via doubling (starts at 256, doubles when full)
let capacity = 256

// Local transform components
const positions    = new Float32Array(capacity * 3)   // [x,y,z, x,y,z, ...]
const rotations    = new Float32Array(capacity * 4)   // [x,y,z,w, x,y,z,w, ...]
const scales       = new Float32Array(capacity * 3)   // [x,y,z, x,y,z, ...]

// Computed world matrices (column-major 4×4)
const worldMatrices = new Float32Array(capacity * 16)

// Hierarchy
const parentIndices = new Int32Array(capacity)         // -1 = root
const dirtyFlags    = new Uint8Array(capacity)         // 1 = needs update
const visibleFlags  = new Uint8Array(capacity)         // 1 = visible

// Traversal order (breadth-first, rebuilt on hierarchy change)
const traversalOrder = new Uint32Array(capacity)
let traversalCount = 0
```

### Why SoA?

Consider updating world matrices for 2000 nodes where 50 are dirty:

**AoS (traditional):** For each dirty node, access `node.position` (object
field), `node.rotation` (different object), `node.parent.worldMatrix` (pointer
chase). Each access is a cache miss.

**SoA (Mantis):** Scan `dirtyFlags` (2 KB, fits in L1). For each dirty node,
read `positions[i*3..i*3+3]`, `rotations[i*4..i*4+4]`, `scales[i*3..i*3+3]`
— all sequential memory. Multiply with `worldMatrices[parent*16..parent*16+16]`
— also sequential. Write to `worldMatrices[i*16..i*16+16]`. Prefetchers handle
this pattern efficiently.

**Measured difference:** SoA is 2–4× faster than AoS for transform updates on
mobile devices due to cache efficiency.

## Hierarchy Operations

### Adding a Child

```typescript
const addChild = (parentSlot: number, childSlot: number) => {
  parentIndices[childSlot] = parentSlot
  childrenLists[parentSlot].push(childSlot)
  markDirty(childSlot)
  rebuildTraversalOrder()
}
```

### Removing a Child

```typescript
const removeChild = (parentSlot: number, childSlot: number) => {
  parentIndices[childSlot] = -1
  const list = childrenLists[parentSlot]
  list.splice(list.indexOf(childSlot), 1)
  rebuildTraversalOrder()
}
```

### Traversal Order Rebuild

When the hierarchy changes (add/remove child), the breadth-first traversal
order is rebuilt. This is an infrequent operation (hierarchy changes are rare
in games vs. per-frame movement).

```typescript
const rebuildTraversalOrder = () => {
  traversalCount = 0
  const queue: number[] = []

  // Enqueue all root nodes
  for (let i = 0; i < nodeCount; i++) {
    if (parentIndices[i] === -1 && activeFlags[i]) {
      queue.push(i)
    }
  }

  // BFS
  let head = 0
  while (head < queue.length) {
    const slot = queue[head++]
    traversalOrder[traversalCount++] = slot
    const children = childrenLists[slot]
    for (let c = 0; c < children.length; c++) {
      if (activeFlags[children[c]]) {
        queue.push(children[c])
      }
    }
  }
}
```

## Dirty Flag Propagation

Setting any local transform component marks the node and all descendants dirty:

```typescript
const setPosition = (slot: number, x: number, y: number, z: number) => {
  const i = slot * 3
  positions[i] = x
  positions[i + 1] = y
  positions[i + 2] = z
  markDirtyRecursive(slot)
}

const markDirtyRecursive = (slot: number) => {
  if (dirtyFlags[slot]) return  // already dirty, subtree must be too
  dirtyFlags[slot] = 1
  const children = childrenLists[slot]
  for (let c = 0; c < children.length; c++) {
    markDirtyRecursive(children[c])
  }
}
```

The early return when already dirty prevents O(n²) cascading in deep trees.

## World Matrix Update

Executed once per frame, before culling and rendering:

```typescript
const updateWorldMatrices = () => {
  for (let t = 0; t < traversalCount; t++) {
    const i = traversalOrder[t]
    if (!dirtyFlags[i]) continue

    // Compose local matrix from position, rotation (quat), scale
    composeFromSoA(i, tempLocalMatrix)

    const parent = parentIndices[i]
    if (parent === -1) {
      // Root: world = local
      worldMatrices.set(tempLocalMatrix, i * 16)
    } else {
      // Child: world = parent.world × local
      mat4Multiply(
        worldMatrices, parent * 16,  // A (parent world)
        tempLocalMatrix, 0,           // B (local)
        worldMatrices, i * 16         // out (this world)
      )
    }

    dirtyFlags[i] = 0
  }
}
```

Because traversal is breadth-first, `worldMatrices[parent * 16]` is guaranteed
to be up-to-date when we process a child.

### composeFromSoA

Composes a 4×4 matrix from separate position (vec3), rotation (quat), and
scale (vec3) stored in the SoA arrays:

```typescript
const composeFromSoA = (slot: number, out: Float32Array) => {
  const px = positions[slot * 3]
  const py = positions[slot * 3 + 1]
  const pz = positions[slot * 3 + 2]
  const qx = rotations[slot * 4]
  const qy = rotations[slot * 4 + 1]
  const qz = rotations[slot * 4 + 2]
  const qw = rotations[slot * 4 + 3]
  const sx = scales[slot * 3]
  const sy = scales[slot * 3 + 1]
  const sz = scales[slot * 3 + 2]

  // Quat → rotation matrix, pre-multiplied by scale
  const x2 = qx + qx, y2 = qy + qy, z2 = qz + qz
  const xx = qx * x2, xy = qx * y2, xz = qx * z2
  const yy = qy * y2, yz = qy * z2, zz = qz * z2
  const wx = qw * x2, wy = qw * y2, wz = qw * z2

  out[0]  = (1 - yy - zz) * sx
  out[1]  = (xy + wz) * sx
  out[2]  = (xz - wy) * sx
  out[3]  = 0
  out[4]  = (xy - wz) * sy
  out[5]  = (1 - xx - zz) * sy
  out[6]  = (yz + wx) * sy
  out[7]  = 0
  out[8]  = (xz + wy) * sz
  out[9]  = (yz - wx) * sz
  out[10] = (1 - xx - yy) * sz
  out[11] = 0
  out[12] = px
  out[13] = py
  out[14] = pz
  out[15] = 1
}
```

## Frustum Culling

Each renderable node has a local-space AABB (from its geometry). The world-space
AABB is computed lazily when the world matrix changes:

```typescript
const getWorldAABB = (slot: number): AABB => {
  if (worldAABBDirty[slot]) {
    transformAABB(localAABBs[slot], worldMatrices, slot * 16, worldAABBs[slot])
    worldAABBDirty[slot] = false
  }
  return worldAABBs[slot]
}
```

Frustum culling tests world-space AABBs against the 6 planes of the view
frustum. See [RAYCASTING.md](./RAYCASTING.md) for BVH-accelerated culling.

```typescript
const cullFrustum = (frustum: Frustum) => {
  for (let i = 0; i < renderableCount; i++) {
    const slot = renderables[i]
    if (!visibleFlags[slot]) {
      culledFlags[slot] = 1
      continue
    }
    const aabb = getWorldAABB(slot)
    culledFlags[slot] = frustum.intersectsAABB(aabb) ? 0 : 1
  }
}
```

For scenes with > 500 objects, the scene-level BVH replaces this linear scan
with a tree traversal that skips entire spatial regions.

## Groups

A `Group` is a `Node` with no geometry or material — a pure transform container:

```typescript
const group = scene.createGroup()
group.position = [10, 0, 5]

const mesh1 = scene.createMesh(geometry, material)
const mesh2 = scene.createMesh(geometry, material)

group.add(mesh1)
group.add(mesh2)

// Moving the group moves both meshes
group.position = [20, 0, 5]
```

Groups participate in dirty flag propagation and world matrix computation
identically to meshes. They are simply skipped during draw command generation
(they have no geometry to render).

## Node Lookup

Nodes can be found by name for convenience:

```typescript
const sword = scene.findByName('Sword_Mesh')
const hand = scene.findByName('Hand_Bone')
```

This uses a `Map<string, number>` built at scene construction time. It is O(1)
lookup, not a tree traversal.

## Capacity Growth

When node count exceeds the current SoA capacity, all arrays are reallocated
at 2× the current capacity:

```typescript
const grow = () => {
  const newCap = capacity * 2
  const newPositions = new Float32Array(newCap * 3)
  newPositions.set(positions)
  positions = newPositions
  // ... same for rotations, scales, worldMatrices, etc.
  capacity = newCap
}
```

This happens rarely (doubling strategy: 256 → 512 → 1024 → 2048 covers most
games). The cost is a one-time memcpy.

## Scene API

```typescript
const scene = engine.scene

// Create nodes
const mesh = scene.createMesh(geometry, material)
const group = scene.createGroup()
const camera = scene.createCamera({ fov: 60, near: 0.1, far: 500 })
const dirLight = scene.createDirectionalLight({ color: [1, 1, 1], intensity: 1.0 })
const ambientLight = scene.createAmbientLight({ color: [0.3, 0.3, 0.4], intensity: 1.0 })

// Hierarchy
group.add(mesh)
scene.root.add(group)

// Transform
mesh.position = [10, 0, 5]       // sets x=10, y=0, z=5 (Z-up)
mesh.rotation = [0, 0, 0.5, 0.87] // quaternion [x, y, z, w]
mesh.scale = [2, 2, 2]

// Visibility
mesh.visible = false               // hides mesh and all children

// Removal
group.remove(mesh)
mesh.dispose()                      // frees GPU resources
```
