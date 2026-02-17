# 03 — Scene Graph

The scene graph manages spatial hierarchy (parent/child transforms), grouping, and bone attachment. It is designed for minimal per-frame overhead: world matrices are only recomputed when local transforms change, and the flat matrix cache enables O(1) lookups.

## Coordinate System

**Z is up, right-handed.**

```
       Z (up)
       |
       |
       +------ X (right)
      /
     /
    Y (forward)
```

All math, cameras, and loaders enforce this convention. GLTF models (Y-up) are rotated on import.

## Node Types

```ts
// Base node — every object in the scene graph
interface SceneNode {
  readonly id: number              // Unique, auto-incrementing
  name: string
  parent: SceneNode | null
  children: SceneNode[]

  // Local transform (relative to parent)
  position: Float32Array           // [x, y, z] — vec3
  rotation: Float32Array           // [x, y, z, w] — quaternion
  scale: Float32Array              // [x, y, z] — vec3

  // Cached world transform
  worldMatrix: Float32Array        // 4x4 column-major
  normalMatrix: Float32Array       // 3x3 for lighting (inverse-transpose of upper-left 3x3)

  // Flags
  visible: boolean
  dirty: boolean                   // True when local transform changed
  worldDirty: boolean              // True when worldMatrix needs recomputation

  // Culling
  localAABB: AABB | null          // Bounding box in local space
  worldAABB: AABB | null          // Bounding box in world space (updated with worldMatrix)

  // For frustum culling
  culled: boolean                  // Set by frustum culler each frame
}
```

### Specialized Node Types

```ts
interface MeshNode extends SceneNode {
  type: 'mesh'
  geometry: GeometryHandle
  material: MaterialHandle
  castShadow: boolean
  receiveShadow: boolean
  isTransparent: boolean
  isStatic: boolean               // Hint for render bundle caching (WebGPU)
}

interface SkinnedMeshNode extends SceneNode {
  type: 'skinned_mesh'
  geometry: GeometryHandle         // Must include bone indices/weights
  material: MaterialHandle
  skeleton: Skeleton
  castShadow: boolean
  receiveShadow: boolean
}

interface GroupNode extends SceneNode {
  type: 'group'
  // No geometry or material — purely organizational
}

interface BoneNode extends SceneNode {
  type: 'bone'
  boneIndex: number                // Index into skeleton's bone array
}

interface CameraNode extends SceneNode {
  type: 'camera'
  projection: 'perspective' | 'orthographic'
  fov: number
  near: number
  far: number
  aspect: number
  viewMatrix: Float32Array         // Inverse of worldMatrix
  projectionMatrix: Float32Array
  viewProjectionMatrix: Float32Array
}

interface LightNode extends SceneNode {
  type: 'directional_light' | 'ambient_light'
  color: Float32Array              // [r, g, b]
  intensity: number
  // Directional lights have implicit direction from rotation
}
```

## Scene

The `Scene` is the root node and the top-level container:

```ts
interface Scene {
  root: GroupNode
  background: [number, number, number] | null  // Clear color
  ambientLight: { color: Float32Array, intensity: number }

  add(node: SceneNode): void
  remove(node: SceneNode): void
  traverse(callback: (node: SceneNode) => void): void
  getNodeById(id: number): SceneNode | null
  getNodeByName(name: string): SceneNode | null
}
```

## Transform Propagation

### Dirty Flag System

When a node's local transform (position, rotation, or scale) is modified, its `dirty` flag is set. This is done via setter proxies or explicit `markDirty()` calls:

```ts
const setPosition = (node: SceneNode, x: number, y: number, z: number): void => {
  node.position[0] = x
  node.position[1] = y
  node.position[2] = z
  markDirty(node)
}

const markDirty = (node: SceneNode): void => {
  if (node.worldDirty) return  // Already dirty, children already marked
  node.worldDirty = true
  for (let i = 0; i < node.children.length; i++) {
    markDirty(node.children[i])
  }
}
```

### World Matrix Update

Before rendering, a single traversal updates all dirty world matrices:

```ts
const updateWorldMatrices = (node: SceneNode): void => {
  if (node.worldDirty) {
    // Compose local matrix from position, rotation, scale
    composeMatrix(node.worldMatrix, node.position, node.rotation, node.scale)

    // Multiply by parent's world matrix
    if (node.parent) {
      mat4Multiply(node.worldMatrix, node.parent.worldMatrix, node.worldMatrix)
    }

    // Update normal matrix (inverse transpose of upper-left 3x3)
    mat3NormalFromMat4(node.normalMatrix, node.worldMatrix)

    // Update world AABB
    if (node.localAABB) {
      transformAABB(node.worldAABB, node.localAABB, node.worldMatrix)
    }

    node.worldDirty = false
  }

  for (let i = 0; i < node.children.length; i++) {
    updateWorldMatrices(node.children[i])
  }
}
```

This is O(dirty nodes), not O(all nodes). For a scene where 90% of objects are static, this is very cheap.

## Parent/Child Relationships

Adding and removing children works like three.js:

```ts
const addChild = (parent: SceneNode, child: SceneNode): void => {
  if (child.parent) {
    removeChild(child.parent, child)
  }
  child.parent = parent
  parent.children.push(child)
  markDirty(child)
}

const removeChild = (parent: SceneNode, child: SceneNode): void => {
  const idx = parent.children.indexOf(child)
  if (idx !== -1) {
    parent.children.splice(idx, 1)
    child.parent = null
  }
}
```

## Groups

`GroupNode` is a transform-only container, equivalent to three.js's `Group`:

```ts
const createGroup = (name?: string): GroupNode => ({
  id: nextId(),
  name: name ?? '',
  type: 'group',
  parent: null,
  children: [],
  position: new Float32Array(3),
  rotation: new Float32Array([0, 0, 0, 1]), // Identity quaternion
  scale: new Float32Array([1, 1, 1]),
  worldMatrix: mat4Identity(),
  normalMatrix: mat3Identity(),
  visible: true,
  dirty: true,
  worldDirty: true,
  localAABB: null,
  worldAABB: null,
  culled: false,
})
```

## Bone Attachment (Static Mesh to Bone)

Attaching a static mesh (e.g., a sword) to a bone (e.g., a hand) is done by making the mesh a child of the bone node in the scene graph:

```ts
// Attach sword to right hand bone
const skeleton = character.skeleton
const handBone = skeleton.getBoneByName('hand_R')
addChild(handBone, swordMesh)
```

Because `BoneNode` participates in the scene graph, its `worldMatrix` is updated each frame from the skeleton's animation pose. Any children inherit this transform automatically through the standard dirty-flag propagation.

### How Bone World Matrices Are Computed

During animation update (see [08-animation.md](08-animation.md)), each bone's local transform is set from the current animation pose. The scene graph traversal then computes bone world matrices normally (local × parent world). The skeleton stores an additional `boneMatrices` array that combines the world matrix with the inverse bind matrix for skinning — but the bone's `worldMatrix` is always available for child attachment.

```ts
// During animation update:
const updateSkeleton = (skeleton: Skeleton): void => {
  for (let i = 0; i < skeleton.bones.length; i++) {
    const bone = skeleton.bones[i]
    // bone.position/rotation/scale are set from animation clip
    // worldMatrix is computed by scene graph traversal
    // skinning matrix = worldMatrix * inverseBindMatrix
    mat4Multiply(
      skeleton.boneMatrices, // Flat Float32Array, 16 floats per bone
      bone.worldMatrix,
      skeleton.inverseBindMatrices, // Offset: i * 16
      i * 16 // Write offset
    )
  }
}
```

## Traversal Patterns

### Depth-First Traversal (for rendering)

```ts
const traverse = (node: SceneNode, callback: (node: SceneNode) => void): void => {
  if (!node.visible) return
  callback(node)
  for (let i = 0; i < node.children.length; i++) {
    traverse(node.children[i], callback)
  }
}
```

### Flat Iteration (for culling, sorting)

For frustum culling and draw call collection, a flat list of renderable nodes is maintained and updated when nodes are added/removed. This avoids tree traversal overhead:

```ts
interface RenderableList {
  meshes: MeshNode[]
  skinnedMeshes: SkinnedMeshNode[]
  lights: LightNode[]
  count: number
}
```

The list is rebuilt when the scene graph structure changes (node added/removed) but not every frame.
