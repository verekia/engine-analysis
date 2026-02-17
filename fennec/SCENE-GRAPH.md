# Scene Graph — Transforms, Groups, Bone Attachment

## Overview

Fennec's scene graph is a lightweight parent/child hierarchy with dirty-flag transform propagation. It mirrors the three.js `Object3D` → `Group` → `Mesh` pattern but is designed to minimize per-frame overhead for large numbers of objects.

## Node Types

```
SceneNode (base)
├── Group            — transform-only container (like THREE.Group)
├── Mesh             — geometry + material
├── SkinnedMesh      — mesh + skeleton reference
├── DirectionalLight — light source
├── AmbientLight     — ambient light
├── Camera           — perspective camera
└── BoneAttachment   — attaches a subtree to a skeleton bone
```

## SceneNode — Base Class

Every scene node stores:

```typescript
interface SceneNode {
  readonly id: number                // Unique monotonic ID

  // Local transform (relative to parent)
  position: Vec3                     // Translation
  rotation: Quat                     // Rotation as quaternion
  scale: Vec3                        // Scale (default [1,1,1])

  // Computed world transform
  worldMatrix: Mat4                  // Recomputed when dirty
  localMatrix: Mat4                  // Recomputed when dirty

  // Hierarchy
  parent: SceneNode | null
  children: SceneNode[]

  // Flags
  visible: boolean                   // If false, skip this subtree
  castShadow: boolean
  receiveShadow: boolean
  _dirtyLocal: boolean               // Local matrix needs recompute
  _dirtyWorld: boolean               // World matrix needs recompute

  // Culling
  boundingBox: AABB                  // Local-space AABB
  worldBoundingBox: AABB             // World-space AABB (recomputed with world matrix)

  // User data
  name: string
  userData: Record<string, unknown>
}
```

## Transform System

### Local Matrix Computation

The local matrix is computed from position, rotation, scale only when `_dirtyLocal` is true:

```typescript
const updateLocalMatrix = (node: SceneNode) => {
  if (!node._dirtyLocal) return
  mat4.fromRotationTranslationScale(node.localMatrix, node.rotation, node.position, node.scale)
  node._dirtyLocal = false
  node._dirtyWorld = true
}
```

### World Matrix Propagation

World matrices propagate top-down. A node's world matrix = parent.worldMatrix * node.localMatrix.

```typescript
const updateWorldMatrix = (node: SceneNode, parentDirty: boolean) => {
  updateLocalMatrix(node)

  const needsUpdate = node._dirtyWorld || parentDirty
  if (needsUpdate) {
    if (node.parent) {
      mat4.multiply(node.worldMatrix, node.parent.worldMatrix, node.localMatrix)
    } else {
      mat4.copy(node.worldMatrix, node.localMatrix)
    }
    node._dirtyWorld = false
    // Update world AABB
    aabb.transform(node.worldBoundingBox, node.boundingBox, node.worldMatrix)
  }

  for (const child of node.children) {
    updateWorldMatrix(child, needsUpdate)
  }
}
```

### Dirty Flag Triggers

Setting `position`, `rotation`, or `scale` automatically marks the node dirty. This is done via setters:

```typescript
// Vec3 with dirty callback
const createTrackedVec3 = (onChange: () => void, x = 0, y = 0, z = 0): Vec3 => {
  const data = new Float32Array([x, y, z])
  return new Proxy(data, {
    set(target, prop, value) {
      target[prop as any] = value
      onChange()
      return true
    }
  })
}
```

In production, we avoid Proxy overhead by providing explicit `setPosition()`, `setRotation()`, `setScale()` methods alongside direct property access. The Proxy version is only used in debug mode for convenience.

**Production fast path:**

```typescript
// Users call this to set position and mark dirty in one step
const setPosition = (node: SceneNode, x: number, y: number, z: number) => {
  node.position[0] = x
  node.position[1] = y
  node.position[2] = z
  markDirty(node)
}
```

## Group

A Group is simply a SceneNode with no geometry or material. It exists purely for hierarchical organization:

```typescript
const createGroup = (): Group => ({
  ...createSceneNode(),
  type: 'Group',
})
```

Usage mirrors three.js:

```typescript
const group = new Group()
group.add(mesh1)
group.add(mesh2)
group.position.set(5, 0, 10)
scene.add(group)
```

## Parent/Child API

```typescript
interface SceneNode {
  add(child: SceneNode): void
  remove(child: SceneNode): void
  traverse(callback: (node: SceneNode) => void): void
  find(predicate: (node: SceneNode) => boolean): SceneNode | null
  getWorldPosition(out?: Vec3): Vec3
  getWorldQuaternion(out?: Quat): Quat
  lookAt(target: Vec3): void
}
```

### `add(child)`

```typescript
const add = (parent: SceneNode, child: SceneNode) => {
  if (child.parent) {
    remove(child.parent, child)
  }
  child.parent = parent
  parent.children.push(child)
  child._dirtyWorld = true
}
```

### `remove(child)`

```typescript
const remove = (parent: SceneNode, child: SceneNode) => {
  const idx = parent.children.indexOf(child)
  if (idx !== -1) {
    parent.children.splice(idx, 1)
    child.parent = null
  }
}
```

## Bone Attachment — Attaching Static Meshes to Skeleton Bones

A core requirement is attaching objects (like a sword) to a bone (like a hand). `BoneAttachment` is a special node that inherits its world transform from a bone on a `SkinnedMesh`:

```typescript
interface BoneAttachment extends SceneNode {
  type: 'BoneAttachment'
  targetSkeleton: Skeleton
  targetBoneIndex: number
  offset: Mat4  // Optional local offset relative to the bone
}
```

### How It Works

During the world matrix update phase, a `BoneAttachment` node does NOT use its parent's world matrix. Instead, it reads the bone's world transform from the skeleton:

```typescript
const updateBoneAttachment = (node: BoneAttachment) => {
  const skeleton = node.targetSkeleton
  const boneWorldMatrix = skeleton.boneWorldMatrices[node.targetBoneIndex]

  // BoneAttachment world = bone world matrix * local offset * local TRS
  mat4.multiply(_tempMat4, boneWorldMatrix, node.offset)
  mat4.multiply(node.worldMatrix, _tempMat4, node.localMatrix)

  // Continue propagation to children
  for (const child of node.children) {
    updateWorldMatrix(child, true)
  }
}
```

### Usage

```typescript
// Load character and sword
const character = await loadGLTF('/character.glb')
const sword = await loadGLTF('/sword.glb')

// Find the hand bone index
const handBoneIndex = character.skeleton.getBoneIndex('hand_R')

// Attach sword to hand
const attachment = new BoneAttachment(character.skeleton, handBoneIndex)
attachment.add(sword)
scene.add(attachment)

// The sword now follows the hand bone automatically,
// including during animations
```

### Offset Transforms

The `offset` matrix allows positioning the attached object relative to the bone. For example, a sword grip might need to be rotated and translated relative to the hand bone:

```typescript
const attachment = new BoneAttachment(skeleton, handBoneIndex)
// Rotate sword 90° and offset grip point
mat4.fromRotationTranslation(attachment.offset,
  quat.fromEuler(_tempQuat, 0, 0, Math.PI / 2),
  vec3.fromValues(0, 0, 0.1)
)
attachment.add(sword)
```

## Scene

The `Scene` is the root node of the graph:

```typescript
interface Scene extends SceneNode {
  type: 'Scene'
  background: number           // Clear color (0xRRGGBB)
  lights: Light[]              // Flat list of all lights (rebuilt on add/remove)
  shadowLight: DirectionalLight | null  // The one light that casts shadows

  // Performance: flat lists rebuilt when scene graph changes
  _allMeshes: Mesh[]           // All meshes in the scene (for culling)
  _allSkinnedMeshes: SkinnedMesh[]  // All skinned meshes (for bone updates)
  _dirty: boolean              // Scene structure changed (object added/removed)
}
```

When objects are added/removed, the flat lists are rebuilt by traversal. This is infrequent (scene setup, load events) and keeps per-frame iteration fast (iterating a flat array vs. recursive traversal).

## Traversal Order for Updates

Each frame, the scene graph is processed in this order:

1. **Animation mixer** updates bone local transforms on all skeletons
2. **Top-down world matrix propagation** from scene root
3. **Bone attachment resolution** reads final bone world matrices
4. **Bounding box update** for all moved nodes
5. **Frustum culling** against the flat mesh list

This ordering ensures bone attachments see fully-updated skeleton poses from the current frame.
