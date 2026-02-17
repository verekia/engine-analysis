# SCENE-GRAPH.md - Scene Graph and Transform System

## Hierarchical Object Graph

**Decision: Traditional object graph with parent/child references** (6/9 implementations: Caracal, Fennec, Lynx, Rabbit, Shark, Wren)

The scene graph uses JavaScript objects with parent/child references. This is the most intuitive approach for developers coming from Three.js or any other 3D engine.

### Why Not Flat SoA (Bonobo)?

A flat entity list with no hierarchy cannot represent the scene structures required by the prompt: parent/children groups, bone attachment (sword to hand), and scene graph traversal. The prompt explicitly requires "parent/children and Group like Three.js". Hierarchy is not optional.

### Why Not SoA Hierarchy (Mantis/Hyena)?

SoA with breadth-first traversal order cached is clever, but hierarchy changes become expensive (rebuild traversal order). For a game engine where objects are dynamically added/removed, the overhead of rebuilding traversal indices on every scene change outweighs the cache benefit of contiguous arrays. The object graph approach handles dynamic scenes naturally.

## Node Types

```typescript
// Base container with transform
interface Node {
  position: Vec3
  rotation: Quat
  scale: Vec3
  visible: boolean
  parent: Node | null
  children: Node[]

  add(child: Node): void
  remove(child: Node): void
}

// Concrete types
Group extends Node           // Pure transform container
Mesh extends Node            // Geometry + Material
SkinnedMesh extends Mesh     // + Skeleton + AnimationMixer
Camera extends Node          // Projection + view
DirectionalLight extends Node // Direction from rotation, shadow config
AmbientLight                 // No spatial properties (scene-level)
```

## Coordinate System

**Z-up, right-handed** (universal agreement):
- +X = right
- +Y = forward
- +Z = up
- Cross product: X x Y = Z

## Transform System

### Local Transform

Each node stores position (Vec3), rotation (Quat), and scale (Vec3). Setting any of these marks the node dirty.

### Dirty Flag Propagation

**Decision: Two-level dirty flags with early exit** (Caracal, Fennec, Shark approach)

```typescript
_dirtyLocal: boolean   // Local matrix needs recomputation from TRS
_dirtyWorld: boolean   // World matrix needs recomputation from parent
```

When a transform property changes:
1. Set `_dirtyLocal = true`
2. Recursively set `_dirtyWorld = true` on all descendants
3. **Early exit**: If a descendant is already `_dirtyWorld`, stop recursion (prevents O(n^2) cascading)

### World Matrix Update

**Decision: Depth-first recursive traversal** (6/9 implementations use this)

```typescript
const updateWorldMatrix = (node: Node) => {
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
    updateWorldMatrix(child)
  }
}
```

This runs once per frame, before frustum culling. Only dirty nodes (typically 2-5% of scene) do actual matrix math.

### Contiguous Matrix Storage for GPU Upload

While the scene graph is object-oriented, world matrices are also written into a contiguous Float32Array for efficient GPU upload. Each node maintains an index into this array. When `_dirtyWorld` is cleared, the matrix is copied to the flat buffer at the node's index.

## Frustum Culling

**Decision: AABB per mesh node** (8/9 implementations)

Each mesh computes a local AABB from its geometry at creation time. The world AABB is derived by transforming the local AABB corners by the world matrix (or using the conservative AABB-of-transformed-AABB).

### Hierarchical Culling

**Decision: Cull subtrees when parent AABB is outside frustum** (Caracal, Lynx, Shark approach)

If a group node's aggregate AABB is entirely outside the frustum, skip the entire subtree. This provides logarithmic culling cost for structured scenes.

For flat scenes (no meaningful hierarchy), brute-force per-mesh testing is fine - 2000 AABB-frustum tests cost ~0.1ms.

## Bone Attachment

**Decision: Bones are regular scene nodes** (Lynx, Mantis, Shark, Wren approach)

Attaching a static mesh to a bone is simply `bone.add(sword)`. The world matrix propagation handles the rest automatically. No special `BoneAttachment` class is needed.

```typescript
const skeleton = gltf.skeletons[0]
const handBone = skeleton.getBone('hand_R')
handBone.add(swordMesh)
```

This approach is the simplest and most consistent with the rest of the scene graph.

## Visibility

```typescript
node.visible = false  // Hides node and all descendants from rendering
```

When `visible` is false, the node and all its children are skipped during render list building. The transform is still updated (in case other systems query it), but no draw calls are generated.

## Scene Container

```typescript
const scene = createScene()
scene.add(mesh)
scene.add(group)
scene.ambientLight = createAmbientLight({ color: [1, 1, 1], intensity: 0.3 })
```

The scene is the root node. Ambient light is stored as a property of the scene (not a spatial node) since it has no position or direction - it is a global illumination term.

## Node Lookup

**Decision: Recursive find by name + optional Map for O(1) lookup** (Mantis approach for performance, standard recursive for simplicity)

```typescript
scene.findByName('player')  // Recursive depth-first search
```

For performance-critical lookups (e.g., finding bones repeatedly), users can build their own Map at load time. The engine does not force a global name registry to keep the common case simple.

## Transform Setter API

**Decision: Direct property mutation with dirty marking** (Caracal, Shark, Wren approach)

```typescript
mesh.position.set(1, 2, 3)  // Sets values + marks dirty
mesh.rotation.setFromEuler(0, 0, Math.PI / 4)  // Sets quat from Euler + marks dirty
```

Vec3 and Quat objects are lightweight wrappers over Float32Array slices. The `set()` method triggers dirty flag marking on the owning node. This avoids the overhead of Proxy objects while still providing automatic dirty tracking.

## Performance Characteristics

| Operation | Cost |
|-----------|------|
| World matrix update (1000 dirty nodes) | ~0.3ms |
| Frustum culling (2000 AABBs) | ~0.1ms |
| Add/remove child | O(1) amortized |
| Find by name | O(n) tree traversal |
| Transform change + dirty propagation | O(subtree depth) |
