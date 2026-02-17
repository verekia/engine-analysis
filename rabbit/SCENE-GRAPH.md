# Scene Graph

## Node Hierarchy

The scene graph is a tree of `Node` objects. Each node has a local transform (position, rotation, scale), a computed world matrix, and a parent/children relationship.

```
Scene (root Node)
├── AmbientLight
├── DirectionalLight
├── Group "environment"
│   ├── Mesh "ground"
│   └── Mesh "tree_01"
├── SkinnedMesh "character"
│   └── Skeleton
│       ├── Bone "root"
│       │   ├── Bone "spine"
│       │   │   ├── Bone "chest"
│       │   │   │   ├── Bone "head"
│       │   │   │   ├── Bone "arm_l"
│       │   │   │   │   └── Bone "hand_l"
│       │   │   │   │       └── Mesh "sword" (attached to bone)
│       │   │   │   └── Bone "arm_r"
│       │   │   │       └── Bone "hand_r"
│       │   │   └── ...
│       │   └── ...
│       └── ...
└── Group "ui_anchors"
    └── Node "healthbar_pos" (used by HtmlOverlay)
```

## Node Base Class

```typescript
class Node {
  // Identity
  readonly id: number           // auto-incrementing unique ID
  name: string

  // Hierarchy
  parent: Node | null
  readonly children: Node[]

  // Local transform
  readonly position: Vec3       // local position
  readonly rotation: Quat       // local rotation (quaternion)
  readonly scale: Vec3          // local scale, default [1,1,1]

  // Computed
  readonly localMatrix: Mat4    // T * R * S
  readonly worldMatrix: Mat4    // parent.worldMatrix * localMatrix

  // Flags
  visible: boolean              // if false, skip this node and all children
  private _matrixDirty: boolean // local matrix needs recompute
  private _worldDirty: boolean  // world matrix needs recompute

  // Bounding (set by Mesh subclass)
  localAABB: AABB | null
  worldAABB: AABB | null
}
```

### Transform API

```typescript
// Position
node.position.set(10, 0, 5)

// Rotation — helper methods on Node
node.setRotationFromEuler(0, 0, Math.PI / 4)          // Z-up: rotate around Z
node.setRotationFromAxisAngle(Vec3.Z_AXIS, Math.PI / 4) // same thing
node.rotation.setFromEulerZXY(roll, pitch, yaw)        // direct quat access

// Scale
node.scale.set(2, 2, 2)

// Convenience: look at a point (Z-up)
node.lookAt(target: Vec3)
```

Setting any transform property automatically marks `_matrixDirty = true`. The dirty flag propagates to children: when a parent's world matrix changes, all descendants are marked `_worldDirty`.

### Matrix Update Strategy

World matrices are recomputed lazily during the scene traversal phase of rendering:

```typescript
const updateWorldMatrix = (node: Node, parentWorldMatrix: Mat4 | null) => {
  if (node._matrixDirty) {
    node.localMatrix.compose(node.position, node.rotation, node.scale)
    node._matrixDirty = false
    node._worldDirty = true
  }

  if (node._worldDirty || parentWorldMatrix) {
    if (parentWorldMatrix) {
      node.worldMatrix.multiplyMatrices(parentWorldMatrix, node.localMatrix)
    } else {
      node.worldMatrix.copyFrom(node.localMatrix)
    }
    node._worldDirty = false

    // Update world AABB if this is a Mesh
    if (node.localAABB) {
      node.worldAABB!.transformFrom(node.localAABB, node.worldMatrix)
    }

    // Propagate to children
    for (const child of node.children) {
      updateWorldMatrix(child, node.worldMatrix)
    }
  } else {
    // Parent unchanged, but children might have local changes
    for (const child of node.children) {
      if (child._matrixDirty || child._worldDirty) {
        updateWorldMatrix(child, node.worldMatrix)
      }
    }
  }
}
```

This avoids recomputing matrices for static objects every frame. Only dirty nodes and their descendants are updated.

## Node Types

### Group

A container node with no renderable component. Used to organize objects and apply group transforms.

```typescript
class Group extends Node {
  // No additional properties — purely organizational
}
```

### Mesh

A renderable node with geometry and material.

```typescript
class Mesh extends Node {
  geometry: Geometry
  material: Material

  // Computed from geometry at construction
  localAABB: AABB
  worldAABB: AABB

  // For raycasting
  bvh: BVH | null  // built lazily on first raycast
}
```

### SkinnedMesh

A mesh driven by a skeleton.

```typescript
class SkinnedMesh extends Mesh {
  skeleton: Skeleton

  // Uniform buffer of bone matrices (mat4x3 packed)
  boneMatrixBuffer: HalBuffer

  // Bind pose inverse matrices
  readonly inverseBindMatrices: Mat4[]
}
```

The skinning pipeline:
1. `AnimationMixer` updates bone local transforms each frame
2. Bone world matrices are computed via skeleton traversal
3. `boneMatrix[i] = bone[i].worldMatrix * inverseBindMatrix[i]` — computed once per frame
4. The array of bone matrices is uploaded to a uniform buffer
5. The vertex shader applies skinning: `position = sum(weight[j] * boneMatrix[joint[j]] * localPosition)`

### Bone

A node within a skeleton hierarchy. Bones are regular nodes with an additional index for lookup.

```typescript
class Bone extends Node {
  readonly boneIndex: number  // index into the bone matrix array
}
```

### Camera

```typescript
class Camera extends Node {
  // Projection
  fov: number               // vertical field of view in radians
  aspect: number            // width / height
  near: number
  far: number
  projectionType: 'perspective' | 'orthographic'

  // Orthographic-specific
  orthoSize?: number        // half-height of the ortho frustum

  // Computed
  readonly viewMatrix: Mat4
  readonly projectionMatrix: Mat4
  readonly viewProjectionMatrix: Mat4

  updateViewProjectionMatrix(): void
}
```

The camera's view matrix is derived from its world matrix: `viewMatrix = inverse(worldMatrix)`. Since the camera is a Node, it participates in the scene graph — you can parent a camera to a node to make it follow that node.

### DirectionalLight

```typescript
class DirectionalLight extends Node {
  color: [number, number, number]
  intensity: number

  // Shadow configuration
  castShadow: boolean
  shadowMapSize: number       // texture size (e.g. 2048)
  shadowCascades: number      // 1-4, default 3
  shadowBias: number
  shadowNormalBias: number
}
```

The light direction is derived from the node's rotation (forward vector). The light does not have a position for shading purposes (directional lights are infinitely far away), but the node position is used to compute shadow map view matrices.

### AmbientLight

```typescript
class AmbientLight extends Node {
  color: [number, number, number]
  intensity: number
}
```

## Parent/Child Operations

```typescript
// Add child
group.add(mesh)         // mesh.parent = group, group.children.push(mesh)

// Remove child
group.remove(mesh)      // mesh.parent = null, splice from group.children

// Reparent (preserving world transform)
node.reparentTo(newParent)
// Computes: node.localMatrix = inverse(newParent.worldMatrix) * node.worldMatrix
// Then decomposes back to position/rotation/scale

// Traverse
scene.traverse((node) => { /* visit every descendant */ })
scene.traverseVisible((node) => { /* skip invisible subtrees */ })
```

## Bone Attachment

Attaching a static mesh to a bone (e.g., sword in hand):

```typescript
// The sword mesh becomes a child of the hand bone
const handBone = skeleton.getBoneByName('hand_r')
handBone.add(swordMesh)
```

Because bones are regular `Node` objects in the scene graph, the sword's world matrix is automatically computed as `handBone.worldMatrix * sword.localMatrix` during the matrix update phase. No special-case code needed.

To offset the sword from the bone:

```typescript
swordMesh.position.set(0, 0, 0.1)  // offset along bone's local Z
swordMesh.setRotationFromEuler(0, 0, Math.PI / 2)  // rotate to align
```

## Scene

The `Scene` is the root node of the graph, with convenience methods for finding lights:

```typescript
class Scene extends Node {
  // Cached light references (updated when lights are added/removed)
  directionalLights: DirectionalLight[]
  ambientLights: AmbientLight[]

  // Background color
  backgroundColor: [number, number, number]
}
```

## Traversal Optimization

Scene traversal is a hot path (runs every frame). Optimizations:

1. **No allocation**: The `traverse` method uses no closures or temporary arrays. It's a simple recursive walk.
2. **Early exit on invisible**: `traverseVisible` skips entire subtrees when `visible = false`.
3. **Flat render list reuse**: The renderer maintains pre-allocated `RenderItem[]` arrays that are cleared and refilled each frame (no new array allocations).
4. **Node pool**: Nodes are created from a pool to avoid GC pressure during rapid add/remove cycles (e.g., particle-like effects with meshes).
