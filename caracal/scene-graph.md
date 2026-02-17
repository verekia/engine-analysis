# Scene Graph

## Coordinate System

Caracal uses a **Z-up, right-handed** coordinate system:

```
    Z (up)
    │
    │
    │
    └──────── X (right)
   ╱
  Y (forward)
```

- **X**: Right
- **Y**: Forward (into the screen in default camera orientation)
- **Z**: Up

This matches Blender and Unreal Engine conventions. When importing glTF (which uses Y-up, right-handed), the loader applies a 90° rotation around X to convert.

### Euler Angle Convention

Euler angles use **intrinsic ZYX** order (yaw-pitch-roll):
1. Rotate around Z (yaw — turning left/right)
2. Rotate around Y (pitch — looking up/down)
3. Rotate around X (roll — tilting sideways)

This is the natural decomposition for Z-up worlds where characters rotate around the vertical Z axis.

## Node Hierarchy

### Base Node

Every object in the scene is a `Node`. The node stores transform data and manages parent/child relationships:

```typescript
interface Node {
  // Identity
  readonly id: number          // Auto-incrementing unique ID
  name: string                 // User-assigned name (for lookups)

  // Hierarchy
  parent: Node | null
  readonly children: Node[]

  // Local transform
  position: Vec3               // Local position
  rotation: Quat               // Local rotation (quaternion is primary)
  scale: Vec3                  // Local scale
  euler: Euler                 // Euler view of rotation (synced lazily)

  // Derived transform
  readonly worldMatrix: Mat4   // Computed world matrix (read-only view)
  readonly worldPosition: Vec3 // Extracted from worldMatrix

  // Culling
  localAABB: AABB | null       // Local-space bounding box (set by geometry or user)
  readonly worldAABB: AABB     // Transformed AABB (for frustum culling)

  // Flags
  visible: boolean             // If false, node and descendants skip rendering
  frustumCulled: boolean       // If false, skip frustum test (always render)
  castShadow: boolean          // Include in shadow pass
  receiveShadow: boolean       // Apply shadow mapping in material

  // Methods
  add(child: Node): void
  remove(child: Node): void
  traverse(callback: (node: Node) => void): void
  find(name: string): Node | null
  lookAt(target: Vec3): void   // Orients node to face target (Z-up aware)
}
```

### Specialized Node Types

```
Node (base)
├── Mesh          — has .geometry and .material
├── SkinnedMesh   — has .geometry, .material, .skeleton
├── Group         — lightweight container (no geometry/material)
├── Camera        — has .projection, .fov, .near, .far
├── DirectionalLight — has .color, .intensity, .shadow config
├── AmbientLight  — has .color, .intensity
└── Bone          — skeleton bone (has .bindMatrix, .inverseBindMatrix)
```

Each type is a factory function that creates a `Node` with the appropriate component data attached:

```typescript
// Creating a mesh
const mesh = createMesh(boxGeometry, lambertMaterial)

// It's still a Node — just with geometry + material populated
mesh.position.set(0, 0, 1)
scene.add(mesh)

// Creating a group
const group = createGroup()
group.add(mesh)
group.position.set(5, 0, 0)
scene.add(group)
```

## Transform System

### Local → World Matrix Computation

World matrices are computed by traversing the scene graph from root to leaves, concatenating parent transforms:

```
worldMatrix = parent.worldMatrix × localMatrix
localMatrix = T × R × S
```

Where `T` is translation, `R` is rotation (from quaternion), and `S` is scale.

### Dirty Flag Propagation

Modifying any local transform property (position, rotation, scale) sets a `_dirtyLocal` flag on the node. This flag propagates to all descendants as `_dirtyWorld`:

```typescript
// Setting position marks node + all descendants as needing world matrix update
mesh.position.set(1, 2, 3)
// Internally: mesh._dirtyLocal = true → propagate _dirtyWorld to children
```

World matrices are only recomputed during `scene.updateWorldMatrices()`, which happens once per frame before rendering:

```typescript
const updateWorldMatrices = (node: Node, parentDirty: boolean) => {
  const dirty = parentDirty || node._dirtyLocal

  if (dirty) {
    // Recompute local matrix from position/rotation/scale
    composeMatrix(node._localMatrix, node.position, node.rotation, node.scale)

    // Multiply by parent's world matrix
    if (node.parent) {
      mat4Multiply(node._worldMatrix, node.parent._worldMatrix, node._localMatrix)
    } else {
      mat4Copy(node._worldMatrix, node._localMatrix)
    }

    // Update world AABB
    if (node.localAABB) {
      transformAABB(node._worldAABB, node.localAABB, node._worldMatrix)
    }

    node._dirtyLocal = false
  }

  // Recurse into children
  for (let i = 0; i < node.children.length; i++) {
    updateWorldMatrices(node.children[i], dirty)
  }
}
```

This is an iterative depth-first traversal (using an explicit stack to avoid recursion overhead for deep trees). Only dirty subtrees do work — static objects cost zero after the first frame.

### Contiguous Matrix Storage

For GPU upload efficiency, world matrices for all renderable nodes are packed into a contiguous `Float32Array`:

```typescript
// All world matrices in one buffer
const worldMatrixBuffer = new Float32Array(maxNodes * 16)

// Each node has an index into this buffer
// node._matrixOffset = nodeIndex * 16
// worldMatrixBuffer.set(node._worldMatrix.elements, node._matrixOffset)
```

This allows uploading all transforms to the GPU in a single buffer write, and enables future optimization with compute-based skinning or transform feedback.

## Scene

The `Scene` is the root container:

```typescript
interface Scene {
  readonly root: Group          // Root node of the scene graph
  backgroundColor: Vec3         // Clear color
  ambientLight: AmbientLight | null

  // Derived lists (rebuilt each frame from traversal)
  readonly opaqueList: Mesh[]           // Opaque meshes passing frustum cull
  readonly transparentList: Mesh[]      // Transparent meshes passing frustum cull
  readonly shadowCasterList: Mesh[]     // Shadow-casting meshes
  readonly lightList: DirectionalLight[] // Active lights

  add(node: Node): void         // Shorthand for root.add(node)
  remove(node: Node): void      // Shorthand for root.remove(node)
  updateWorldMatrices(): void    // Propagate transforms
  buildRenderLists(camera: Camera): void  // Cull + classify
}
```

### Render List Building

Each frame, the scene traverses the graph and partitions visible meshes:

```typescript
const buildRenderLists = (scene: Scene, camera: Camera) => {
  const frustum = computeFrustum(camera)

  scene.opaqueList.length = 0
  scene.transparentList.length = 0
  scene.shadowCasterList.length = 0
  scene.lightList.length = 0

  scene.root.traverse((node) => {
    if (!node.visible) return // skip subtree

    if (isMesh(node)) {
      // Frustum culling
      if (node.frustumCulled && !frustumIntersectsAABB(frustum, node._worldAABB)) {
        return
      }

      if (node.material.transparent) {
        scene.transparentList.push(node)
      } else {
        scene.opaqueList.push(node)
      }

      if (node.castShadow) {
        scene.shadowCasterList.push(node)
      }
    } else if (isDirectionalLight(node)) {
      scene.lightList.push(node)
    }
  })
}
```

## Groups and Hierarchical Transforms

Groups are empty nodes used purely for hierarchical transforms:

```typescript
// Create a car with wheels as a group
const car = createGroup()
const body = createMesh(bodyGeometry, bodyMaterial)
const wheelFL = createMesh(wheelGeometry, wheelMaterial)
const wheelFR = createMesh(wheelGeometry, wheelMaterial)

wheelFL.position.set(-1, 1.5, -0.5)
wheelFR.position.set(1, 1.5, -0.5)

car.add(body)
car.add(wheelFL)
car.add(wheelFR)

// Moving the car moves all children
car.position.set(10, 20, 0)

// Rotating a wheel only affects that wheel
wheelFL.rotation.setFromAxisAngle(VEC3_RIGHT, angle)
```

## lookAt (Z-Up Aware)

The `lookAt` function orients a node to face a target position. In Z-up convention, "forward" is +Y and "up" is +Z:

```typescript
const lookAt = (node: Node, target: Vec3) => {
  const forward = vec3Normalize(_v0, vec3Sub(_v0, target, node.worldPosition))
  const right = vec3Normalize(_v1, vec3Cross(_v1, forward, VEC3_UP)) // VEC3_UP = (0,0,1)
  const up = vec3Cross(_v2, right, forward)

  // Build rotation matrix from basis vectors and extract quaternion
  mat4FromBasis(_m0, right, forward, up)
  quatFromMat4(node.rotation, _m0)
  node._dirtyLocal = true
}
```

Note: Handles the degenerate case where forward ≈ (0,0,±1) by choosing an arbitrary right vector.

## Node Finding and Traversal

```typescript
// Find by name (depth-first search)
const sword = scene.root.find('Sword')

// Traverse all descendants
scene.root.traverse((node) => {
  if (isMesh(node) && node.material.transparent) {
    node.material.opacity = 0.5
  }
})

// Get all meshes in subtree
const meshes: Mesh[] = []
group.traverse((node) => {
  if (isMesh(node)) meshes.push(node)
})
```
