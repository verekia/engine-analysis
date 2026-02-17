# SCENE-GRAPH — Final Decision

## Storage Strategy

**Decision: Traditional object graph with contiguous matrix storage** (6/9: Caracal, Fennec, Hyena, Lynx, Rabbit, Wren)

Users interact with an object graph (nodes with `.parent`, `.children`, named properties). Internally, world matrices are packed into a contiguous `Float32Array` for cache-friendly GPU upload.

## Node Types

- **Group** — transform-only container, no renderable data
- **Mesh** — geometry + material, rendered
- **SkinnedMesh** — extends Mesh with skeleton reference
- **Camera** — perspective projection, transform = view position
- **DirectionalLight** — direction derived from world rotation
- **AmbientLight** — no spatial properties (stored on Scene directly)

All node types share a common base with: position, rotation, scale, visible, frustumCulled, parent, children, name.

## Transform Representation

**Decision: Position (Vec3) + Rotation (Quat) + Scale (Vec3)** (universal agreement)

```typescript
mesh.position.set(1, 0, 3)             // Vec3, Float32Array view
mesh.rotation = quatFromEuler(0, 0, Math.PI / 4)  // Quaternion
mesh.scale.set(2, 2, 2)                // Vec3, Float32Array view
```

Setting any transform property marks the node dirty for local matrix recomputation. Euler angles are a convenience conversion, not the storage format. Quaternions avoid gimbal lock and enable slerp blending for animations.

## Dirty Flag System

**Decision: Two-level dirty flags with early-exit propagation** (Caracal, Fennec, Shark approach)

- `_dirtyLocal` — local matrix needs recomputation (position/rotation/scale changed)
- `_dirtyWorld` — world matrix needs recomputation (self or ancestor changed)

```typescript
// Phase 1: Propagate dirty flags down the tree
const propagateDirty = (node: Node): void => {
  if (node._dirtyLocal) node._dirtyWorld = true
  if (node._dirtyWorld) {
    for (const child of node.children) {
      if (!child._dirtyWorld) {  // Early exit: already marked
        child._dirtyWorld = true
        propagateDirty(child)
      }
    }
  }
}

// Phase 2: Recompute only dirty matrices (depth-first)
const updateWorldMatrices = (node: Node): void => {
  if (node._dirtyLocal) {
    composeLocalMatrix(node)
    node._dirtyLocal = false
  }
  if (node._dirtyWorld) {
    if (node.parent) {
      mat4Multiply(node._worldMatrix, node.parent._worldMatrix, node._localMatrix)
    } else {
      mat4Copy(node._worldMatrix, node._localMatrix)
    }
    node._dirtyWorld = false
  }
  for (const child of node.children) updateWorldMatrices(child)
}
```

Typical frame: 2-5% of objects move → 95-98% of matrix work skipped. ~0.3ms for 1000 dirty nodes.

## Frustum Culling

**Decision: AABB-based frustum culling with hierarchical support** (universal agreement)

Each mesh stores a world-space AABB computed from geometry bounds + world matrix. Frustum planes extracted from the camera's view-projection matrix via Gribb-Hartmann method.

P-vertex/N-vertex AABB-plane test: 6 plane tests per object. For 2000 objects, brute-force culling completes in ~0.1-0.2ms.

BVH-accelerated culling is available for scenes exceeding 5000 objects, using the scene-level BVH that already exists for raycasting.

Hierarchical culling: if a parent's AABB is fully outside the frustum, skip all children.

## Bones as Scene Nodes

**Decision: Bones are regular scene nodes** (Caracal, Fennec, Lynx, Mantis, Rabbit, Shark — 6/9)

Bone attachment is achieved by adding any mesh as a child of a bone node:

```typescript
const handBone = skeleton.getBone('hand_R')
handBone.add(swordMesh)  // Sword inherits bone's world transform
```

No special attachment API needed — the standard scene graph hierarchy handles it.

## Per-Node Properties

```typescript
node.visible = true        // Skip rendering and children traversal
node.frustumCulled = true  // Default: test AABB against frustum
node.castShadow = true     // Include in shadow pass
node.receiveShadow = true  // Sample shadow map in fragment shader
node.name = 'player'       // For lookup
```

## Scene Graph Operations

```typescript
// Parent/child
scene.add(mesh)
group.add(child1, child2)
group.remove(child1)

// Traversal
scene.traverse((node) => { /* depth-first, visits every node */ })

// Lookup
scene.getByName('player')  // Recursive find, optional Map for performance

// Transform helpers
node.lookAt([10, 0, 0])    // Z-up aware lookAt
```

## Contiguous Matrix Storage

World matrices are stored in a contiguous `Float32Array(activeNodeCount * 16)` indexed by node ID. The renderer reads this array directly for GPU upload — no per-node object access during rendering.

When a node is added to the scene, it receives an index into the shared matrix pool. When removed, the slot is returned to a free list.
