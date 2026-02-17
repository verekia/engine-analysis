# Scene Graph Architecture

## Coordinate System

Shark uses a **Z-up, right-handed** coordinate system:

```
        Z (up)
        │
        │
        ●──────── X (right)
       ╱
      ╱
    Y (forward)
```

Cross product: X × Y = Z. This matches Blender's default and many game engines. GLTF files (Y-up) are automatically rotated on import by applying a -90° rotation around X.

## Node Hierarchy

### Base: Node3D

Every object in the scene graph extends `Node3D`:

```typescript
interface Node3D {
  // Identity
  readonly id: number           // Unique auto-incrementing ID
  name: string                  // Human-readable name

  // Local transform
  position: Vec3                // Translation relative to parent
  rotation: Quat                // Orientation relative to parent
  scale: Vec3                   // Scale relative to parent

  // Computed world transform (lazy, cached)
  readonly worldMatrix: Mat4    // Local-to-world matrix
  readonly worldPosition: Vec3  // Extracted world position

  // Hierarchy
  parent: Node3D | null
  readonly children: readonly Node3D[]

  // Visibility
  visible: boolean              // If false, skip this node and all descendants
  frustumCulled: boolean        // If true, test AABB against frustum (default: true)

  // Bounding volume
  readonly boundingBox: AABB    // Local-space AABB (set by geometry or manually)
  readonly worldBoundingBox: AABB // World-space AABB (computed from worldMatrix + boundingBox)

  // Lifecycle
  add(child: Node3D): void
  remove(child: Node3D): void
  traverse(callback: (node: Node3D) => void): void
  traverseVisible(callback: (node: Node3D) => void): void
  lookAt(target: Vec3): void
  getWorldDirection(out: Vec3): Vec3
}
```

### Concrete Node Types

| Type | Extends | Purpose |
|---|---|---|
| `Group` | `Node3D` | Pure grouping, no renderable |
| `Mesh` | `Node3D` | Geometry + Material, renderable |
| `SkinnedMesh` | `Mesh` | Adds skeleton + bone matrices |
| `Camera` | `Node3D` | Perspective or orthographic projection |
| `DirectionalLight` | `Node3D` | Infinite light in -Z local direction |
| `AmbientLight` | `Node3D` | Global uniform illumination |
| `BoneAttachment` | `Node3D` | Follows a specific bone's world transform |
| `HtmlAnchor` | `Node3D` | Tracks a 3D position for HTML overlay |

### Scene Root

```typescript
interface Scene extends Group {
  background: Color | null      // Clear color
  ambientLight: AmbientLight | null
  fog: { color: Color, near: number, far: number } | null
}
```

The scene is the root `Group`. All nodes must be descendants of a scene to be rendered.

## Transform System

### Local Transform

Each node stores position (`Vec3`), rotation (`Quat`), and scale (`Vec3`) separately. The local matrix is computed as:

```
localMatrix = T(position) * R(rotation) * S(scale)
```

This decomposed representation is preferred over a raw matrix because:
- Interpolation (animation blending) works correctly on decomposed transforms
- More intuitive API (`node.position.z = 5` vs. `node.matrix[14] = 5`)
- Euler angles are **not stored** — only quaternions. Helper methods accept Euler inputs.

### World Matrix Computation

World matrices are computed lazily using dirty flags:

```typescript
// When any local transform component changes:
const markDirty = (node: Node3D) => {
  if (node._worldDirty) return  // Already dirty, subtree already marked
  node._worldDirty = true
  for (const child of node.children) {
    markDirty(child)
  }
}

// When worldMatrix is accessed:
const getWorldMatrix = (node: Node3D): Mat4 => {
  if (node._worldDirty) {
    computeLocalMatrix(node, node._localMatrix)
    if (node.parent) {
      mat4Multiply(node._worldMatrix, getWorldMatrix(node.parent), node._localMatrix)
    } else {
      mat4Copy(node._worldMatrix, node._localMatrix)
    }
    node._worldDirty = false
  }
  return node._worldMatrix
}
```

**Key optimization**: Dirty flag propagation stops at nodes already marked dirty. This means moving a parent only traverses its children once, even if the parent moves multiple times per frame.

### Flat Matrix Storage

All world matrices are stored in a single contiguous `Float32Array`:

```typescript
// Pre-allocated for MAX_OBJECTS (e.g., 4096)
const worldMatrixPool = new Float32Array(MAX_OBJECTS * 16)

// Each node's worldMatrix is a view into this pool
node._worldMatrix = new Float32Array(worldMatrixPool.buffer, node.id * 16 * 4, 16)
```

This allows bulk uploading all matrices to the GPU in a single buffer write.

## Frustum Culling

### AABB-Based Culling

Each renderable node has an axis-aligned bounding box (AABB) in world space. Before rendering, all visible nodes are tested against the camera's view frustum.

### Frustum Representation

The frustum is defined by 6 planes (left, right, top, bottom, near, far), each stored as a `Vec4` (normal.xyz, distance):

```typescript
interface Frustum {
  planes: [Vec4, Vec4, Vec4, Vec4, Vec4, Vec4]  // 6 planes
}

// Extract frustum planes from viewProjection matrix
const extractFrustum = (vp: Mat4, out: Frustum): void => {
  // Left:   row3 + row0
  // Right:  row3 - row0
  // Bottom: row3 + row1
  // Top:    row3 - row1
  // Near:   row3 + row2
  // Far:    row3 - row2
  // Normalize each plane
}
```

### AABB vs Frustum Test

```typescript
// Returns true if the AABB is at least partially inside the frustum
const isAABBInFrustum = (aabb: AABB, frustum: Frustum): boolean => {
  for (let i = 0; i < 6; i++) {
    const plane = frustum.planes[i]
    // Find the AABB vertex most in the direction of the plane normal
    const px = plane[0] > 0 ? aabb.max[0] : aabb.min[0]
    const py = plane[1] > 0 ? aabb.max[1] : aabb.min[1]
    const pz = plane[2] > 0 ? aabb.max[2] : aabb.min[2]
    // If the most-positive vertex is behind the plane, the AABB is fully outside
    if (plane[0] * px + plane[1] * py + plane[2] * pz + plane[3] < 0) {
      return false
    }
  }
  return true
}
```

This is a conservative test — some AABBs near frustum corners may pass when they shouldn't, but no visible objects will be incorrectly culled.

### Culling Pipeline

```
1. Extract frustum planes from camera.viewProjectionMatrix
2. Traverse scene graph (skip invisible nodes and their subtrees)
3. For each node with frustumCulled=true:
   a. Compute world AABB (transform local AABB by world matrix)
   b. Test against frustum
   c. If outside → skip this node and all descendants
   d. If inside → add to visible list
4. Return visible list partitioned into:
   - Shadow casters
   - Opaque objects (sorted by render key)
   - Transparent objects
```

**Hierarchical culling optimization**: If a parent's world AABB is fully inside the frustum, skip frustum tests for all descendants (they're guaranteed visible). If fully outside, skip the entire subtree.

## Parent-Child Transform Propagation

### Standard Hierarchy

```typescript
const group = new Group()
group.position.set(10, 0, 0)

const mesh = new Mesh(geometry, material)
mesh.position.set(0, 0, 5)  // 5 units above the group

group.add(mesh)
scene.add(group)

// mesh world position = (10, 0, 5)
```

### Bone Attachment

For attaching static meshes (e.g., a sword) to animated skeleton bones:

```typescript
const sword = new Mesh(swordGeometry, swordMaterial)
const attachment = new BoneAttachment(skinnedMesh, 'hand_right')
attachment.add(sword)
scene.add(attachment)

// Each frame, after animation update:
// attachment.worldMatrix = skinnedMesh.skeleton.getBoneWorldMatrix('hand_right')
// sword inherits this transform through normal parent-child propagation
```

`BoneAttachment` overrides the normal transform computation to follow the bone's world-space transform instead of using its own local position/rotation/scale.

## Scene Traversal Utilities

```typescript
// Find a node by name (breadth-first)
scene.findByName('player')

// Get all meshes in the scene
scene.findAll((node) => node instanceof Mesh)

// Traverse only visible nodes
scene.traverseVisible((node) => {
  // Process node
})
```

## Memory Layout

Nodes are stored in a flat array indexed by ID. This enables:
- O(1) lookup by ID
- Cache-friendly iteration over all nodes
- Bulk operations on node properties (e.g., update all world matrices)

```typescript
// Internal storage
const nodes: (Node3D | null)[] = []  // Sparse array indexed by node ID
const freeIds: number[] = []          // Recycled IDs from removed nodes

const createNodeId = (): number => {
  return freeIds.length > 0 ? freeIds.pop()! : nodes.length
}
```
