# SCENE-GRAPH - Design Decisions

## Storage Strategy

**Decision: Traditional object graph with contiguous matrix storage for GPU upload**

- Sources: Caracal, Fennec, Lynx, Rabbit, Shark, Wren (6/9)
- Rejected: Pure flat SoA (Bonobo) — too rigid for complex scenes with hierarchy
- Rejected: SoA with hierarchy (Mantis, Hyena) — good for perf but harder to add/remove nodes dynamically

```typescript
// Public: object-oriented nodes
const mesh = createMesh(geometry, material)
mesh.position.set(1, 0, 3)
scene.add(mesh)

// Internal: contiguous matrix pool for GPU upload
worldMatrixPool: Float32Array(activeNodeCount * 16)
```

Rationale: The requirements specify Three.js-like DX with parent/child and Groups. An object graph provides the most natural API. Internally, world matrices are packed into a contiguous buffer for efficient GPU upload (Caracal approach).

## Node Types

**Decision: Standard node types (universal agreement)**

- **Node / Group**: Base container with transform, parent/child
- **Mesh**: Geometry + Material
- **SkinnedMesh**: Mesh + Skeleton for animation
- **Camera**: Perspective projection + view matrix
- **DirectionalLight**: Shadow-casting directional light
- **AmbientLight**: Global uniform illumination (no spatial properties, stored on Scene)

## Transform Representation

**Decision: Position (Vec3) + Rotation (Quat) + Scale (Vec3)**

- Sources: All 9 implementations use quaternions internally
- Provide `setRotationFromEuler()` helper for convenience (Shark approach)
- Store rotation as quaternion only — no separate Euler storage
- Dirty flag set automatically when position/rotation/scale change

## Dirty Flag System

**Decision: Two-level dirty flags with early-exit propagation**

- Sources: Caracal, Fennec, Shark (two-level); Mantis, Hyena, Fennec (early exit)

```typescript
interface Node {
  _dirtyLocal: boolean   // local matrix needs recompute from pos/rot/scale
  _dirtyWorld: boolean   // world matrix needs recompute from parent
}
```

- Setting position/rotation/scale sets `_dirtyLocal = true`
- Propagation marks children `_dirtyWorld = true`
- Early exit: if child is already `_dirtyWorld`, skip recursion (avoids O(n²) cascading)

## Update Strategy

**Decision: Deferred propagation + depth-first recursive update**

- Sources: Depth-first from Caracal, Fennec, Lynx, Rabbit, Shark, Wren (6/9); Deferred from Bonobo, Fennec

```typescript
// Phase 1: Propagate dirty flags (once per frame)
const propagateDirty = (node: Node) => {
  if (node._dirtyLocal || node._dirtyWorld) {
    for (const child of node.children) {
      if (!child._dirtyWorld) {
        child._dirtyWorld = true
        propagateDirty(child)
      }
    }
  }
}

// Phase 2: Update world matrices (dirty only)
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

## Frustum Culling

**Decision: AABB per mesh + hierarchical culling**

- Sources: 8/9 use AABB (all except Bonobo which uses spheres)
- Hierarchical culling from Caracal, Lynx, Shark

Each mesh has a local-space AABB computed at geometry creation. During culling, the AABB is tested against the camera frustum in world space.

Hierarchical optimization: if a Group's world AABB is fully outside the frustum, skip the entire subtree without testing children individually.

Standard 6-plane frustum test using Gribb-Hartmann extraction and p-vertex optimization.

## Bone Attachment

**Decision: Bones are regular scene nodes — use standard addChild**

- Sources: Lynx, Mantis, Shark, Wren (4/9)
- Rejected: Dedicated BoneAttachment class (Caracal, Fennec) — unnecessary complexity

```typescript
// Attach a sword to a hand bone
const handBone = skeleton.getBone('hand_R')
handBone.add(swordMesh)
```

World matrix propagates automatically through the scene graph. No special API needed.

## Visibility

**Decision: Per-node visible flag + per-node frustumCulled toggle**

- Sources: All implementations have `node.visible`; Caracal, Shark, Fennec, Lynx have per-node culling toggle

```typescript
node.visible = false     // hide node and all children from rendering
node.frustumCulled = true // default: true. Set false to skip frustum test
```

## lookAt Helper

**Decision: Z-up aware lookAt**

- Sources: Caracal, Fennec, Hyena, Lynx, Shark, Wren (6/9)

```typescript
node.lookAt(targetPosition)  // Orients node to face target, respecting Z-up
```

Handles the degenerate case where the forward direction is nearly parallel to Z.

## Scene API

```typescript
const scene = createScene()
scene.add(mesh)
scene.add(group)
scene.remove(mesh)
scene.ambientLight = { color: [1, 1, 1], intensity: 0.3 }

// Traversal
scene.traverse((node) => { /* visit every node */ })

// Lookup
scene.getByName('player')  // recursive name search
```

## Contiguous Matrix Storage

**Decision: Pack world matrices into contiguous Float32Array for GPU upload**

- Sources: Bonobo, Caracal, Mantis (contiguous approach)

```typescript
// Renderer maintains a contiguous matrix buffer
// Each visible node's world matrix is copied here before upload
const worldMatrixBuffer = new Float32Array(maxVisibleObjects * 16)
```

This enables a single `writeBuffer` call to upload all object matrices, rather than individual uploads per object. The ring buffer uniform strategy subsumes this — per-object uniforms (including world matrix) are written sequentially into the ring buffer region for the current frame.
