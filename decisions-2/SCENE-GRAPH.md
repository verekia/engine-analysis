# SCENE-GRAPH.md - Decisions

## Decision: Traditional Object Graph with SoA Matrix Storage

### Storage Strategy: Object Graph with Contiguous Matrix Pool

**Chosen**: Traditional object graph for the public API with a contiguous `Float32Array` pool for world matrices (combines Caracal's contiguous matrix storage with the standard object graph used by 6/9 implementations)
**Rejected**: Flat SoA with no hierarchy (Bonobo) - doesn't match the prompt's requirement for parent/children and Groups
**Rejected**: Pure SoA hierarchy (Mantis/Hyena) - more complex for marginal benefit at target scale

```typescript
// Public API: familiar object model
const group = createGroup()
const mesh = createMesh(geometry, material)
group.add(mesh)
scene.add(group)
mesh.position.set(1, 0, 3)

// Internal: world matrices live in a contiguous Float32Array
// Each node gets an index into the pool
// worldMatrixPool: Float32Array(nodeCount * 16)
// Enables single-copy GPU upload
```

### Node Types

Universal agreement across all 9:
- **Group** / **Node**: Base container with transform, no renderable
- **Mesh**: Geometry + Material reference
- **SkinnedMesh**: Mesh + Skeleton for animation
- **Camera**: Projection + view computation
- **DirectionalLight**: Direction, color, intensity, shadow config
- **AmbientLight**: Color, intensity (no spatial component)

### Coordinate System

Universal agreement: **Z-up, right-handed**

```
       Z (up)
       |
       |
       +---- X (right)
      /
     /
    Y (forward)
```

- Cross product: X x Y = Z
- Matches Blender, Unreal Engine
- glTF Y-up assets converted at import time

### Transform Representation

- **Position**: `Vec3` (Float32Array-backed)
- **Rotation**: `Quat` internally (avoids gimbal lock, better interpolation)
- **Scale**: `Vec3` (Float32Array-backed)
- **Local matrix**: Computed from TRS when dirty
- **World matrix**: `parent.worldMatrix * localMatrix`

Euler angle helpers provided for convenience (`setRotationFromEuler`) but quaternions are the internal representation (all 9 agree).

### Dirty Flag System: Two-Level with Early Exit

**Chosen**: Two-level dirty flags (Caracal/Fennec/Shark approach, 3/9 explicit)

```typescript
// Setting position marks local as dirty
node.position.set(x, y, z)
// Internally: node._dirtyLocal = true; propagateDirtyWorld(node)

// Propagation stops if already dirty (early exit optimization)
const propagateDirtyWorld = (node: Node) => {
  if (node._dirtyWorld) return  // Already marked, subtree already dirty
  node._dirtyWorld = true
  for (const child of node.children) {
    propagateDirtyWorld(child)
  }
}
```

This prevents O(n^2) cascading when multiple ancestors are modified in the same frame.

### Update Traversal: Depth-First Recursive

**Chosen**: Depth-first recursive (6/9: Caracal, Fennec, Lynx, Rabbit, Shark, Wren)
**Rejected**: Breadth-first iterative (Mantis/Hyena) - requires maintaining traversal order array, more complex for no measurable benefit at target scale

```typescript
const updateWorldMatrices = (node: Node) => {
  if (node._dirtyLocal) {
    mat4FromTRS(node._localMatrix, node.position, node.rotation, node.scale)
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
  for (const child of node.children) {
    updateWorldMatrices(child)
  }
}
```

### Frustum Culling: AABB with Hierarchical Subtree Culling

**Chosen**: AABB per mesh node + hierarchical subtree culling (Caracal/Lynx/Shark approach)

- Each mesh stores a local-space AABB computed at geometry creation
- World-space AABB derived from transformed local AABB during update
- If a parent group's AABB is fully outside the frustum, skip the entire subtree
- If fully inside, add all children without further testing

P-vertex/n-vertex optimization for the AABB-frustum test (universal agreement):

```typescript
for (const plane of frustum.planes) {
  const px = plane.nx >= 0 ? aabb.maxX : aabb.minX
  const py = plane.ny >= 0 ? aabb.maxY : aabb.minY
  const pz = plane.nz >= 0 ? aabb.maxZ : aabb.minZ
  if (plane.nx * px + plane.ny * py + plane.nz * pz + plane.d < 0) {
    return OUTSIDE
  }
}
```

### Bone Attachment: Scene Graph Integration

**Chosen**: Bones are regular scene nodes; attaching a mesh to a bone is just `bone.add(mesh)` (4/9: Lynx, Mantis, Shark, Wren)
**Rejected**: Dedicated `BoneAttachment` class (Caracal/Fennec/Hyena) - unnecessary abstraction when bones are already nodes in the graph

The skinning system updates bone world matrices each frame. Any child of a bone node inherits its world transform through normal scene graph propagation. This means attaching a sword to a hand bone requires zero special API:

```typescript
const hand = skeleton.getBone('hand_R')
hand.add(swordMesh)
```

### Visibility

- `node.visible: boolean` - when false, node and all descendants skip rendering
- `node.frustumCulled: boolean` - per-node toggle to disable automatic frustum culling (Caracal/Shark approach, useful for always-visible objects like skyboxes)

### Transform Setter API: Direct Property Mutation with Dirty Marking

**Chosen**: Property-style setters that automatically mark dirty (Caracal/Shark/Wren approach)

```typescript
node.position.set(x, y, z)  // Marks _dirtyLocal = true
node.position.x = 5         // Also marks dirty via setter
```

No Proxy overhead - use Object.defineProperty or a lightweight wrapper class on Vec3/Quat that calls `_markDirty()` on write operations.

### Node Lookup

- `scene.getByName(name)`: Recursive find by name string
- Optional: `Map<string, Node>` for O(1) lookup (Mantis approach) if names are registered

### lookAt Helper

Z-up aware `lookAt` implementation (6/9 provide this):

```typescript
node.lookAt(targetX, targetY, targetZ)
// Computes rotation quaternion so node's forward (-Y) points at target
// Handles degenerate case when target is directly above/below
```
