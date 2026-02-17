# Lynx Engine — Scene Graph

The scene graph is the spatial backbone of the engine. Every object in a 3D scene — meshes, cameras, lights, bones — lives in a tree of nodes. Each node carries a transform (position, rotation, scale) and inherits its parent's world transform. This hierarchy enables grouped movement, frustum culling, skeletal animation, and bone attachment with zero special-case code.

Lynx uses a **Z-up, right-handed coordinate system**. The default "forward" direction is **-Y**, and the default "up" is **+Z**.

---

## 1. Node Types

### Type Hierarchy

```
Scene (root)
├── Node (base)
│   ├── Group (container, no geometry)
│   ├── Mesh (geometry + material)
│   │   └── SkinnedMesh (skeleton + bind matrices)
│   ├── Camera (projection + view)
│   ├── DirectionalLight (shadow-casting light)
│   └── Bone (skeleton joint)
└── AmbientLight (not a node — scene-level config)
```

### Interface Definitions

```typescript
interface AABB {
  readonly min: Vec3
  readonly max: Vec3
}

interface NodeBase {
  readonly id: number
  name: string
  position: Vec3
  rotation: Quat
  scale: Vec3
  localMatrix: Mat4
  worldMatrix: Mat4
  parent: NodeBase | null
  children: NodeBase[]
  visible: boolean
  dirty: boolean
  aabb: AABB | null
}

interface Group extends NodeBase {
  readonly type: 'Group'
}

interface MaterialPalette {
  readonly materials: Material[]
}

interface Mesh extends NodeBase {
  readonly type: 'Mesh'
  geometry: Geometry
  material: Material
  materialPalette: MaterialPalette | null  // for _materialindex per-face system
}

interface Skeleton {
  readonly bones: Bone[]
  readonly boneMatrices: Float32Array      // flat array of 4x4 matrices
  readonly boneInverses: Mat4[]            // inverse bind matrices
}

interface SkinnedMesh extends Mesh {
  readonly type: 'SkinnedMesh'
  skeleton: Skeleton
  bindMatrix: Mat4
  bindMatrixInverse: Mat4
}

interface FrustumPlanes {
  readonly planes: [Plane, Plane, Plane, Plane, Plane, Plane] // left, right, bottom, top, near, far
}

interface Camera extends NodeBase {
  readonly type: 'Camera'
  projectionMatrix: Mat4
  viewMatrix: Mat4
  frustum: FrustumPlanes
  near: number
  far: number
  fov: number           // vertical field of view in radians (perspective)
  aspect: number
  zoom: number           // for orthographic
}

interface DirectionalLight extends NodeBase {
  readonly type: 'DirectionalLight'
  color: Vec3
  intensity: number
  shadow: ShadowConfig | null
}

interface ShadowConfig {
  enabled: boolean
  mapSize: number        // shadow map resolution (e.g., 1024, 2048)
  bias: number
  near: number
  far: number
  bounds: number         // orthographic projection half-extent
}

interface AmbientLight {
  readonly color: Vec3
  readonly intensity: number
}

interface FogConfig {
  color: Vec3
  near: number
  far: number
}

interface Bone extends NodeBase {
  readonly type: 'Bone'
}

interface Scene extends NodeBase {
  readonly type: 'Scene'
  ambientLight: AmbientLight
  background: Vec3 | null       // background clear color
  fog: FogConfig | null
}
```

### Key Design Points

- **`Node`** is the base. Everything that exists in 3D space extends it. It carries position, rotation (quaternion), scale, local/world matrices, a dirty flag, and an optional AABB.
- **`Group`** extends Node with no additions. It is a pure container — identical to Three.js `Group`. Use it to logically group children so they move together.
- **`Mesh`** extends Node and adds geometry, material, and an optional `materialPalette` for the `_materialindex` system (where per-face or per-vertex indices select from a palette of materials).
- **`SkinnedMesh`** extends Mesh. It references a `Skeleton` (a flat list of `Bone` nodes and their computed matrices) plus bind matrices that define the rest pose.
- **`Camera`** extends Node. Its position/rotation in the scene graph define the view matrix. Projection is set separately (perspective or orthographic). Frustum planes are extracted from the combined viewProjection matrix.
- **`DirectionalLight`** extends Node. Its rotation defines light direction. Position is ignored for illumination but used for shadow map placement.
- **`AmbientLight`** is deliberately **not** a node. It is a flat config on the Scene — there is no spatial concept for ambient light, so putting it in the graph would be misleading.
- **`Bone`** extends Node. Bones form a hierarchy within a Skeleton. Because they are regular nodes, any other node (meshes, particles, etc.) can be parented to a bone with zero special-case logic.
- **`Scene`** is the root node. It holds global config: ambient light, background color, and optional fog.

---

## 2. Transform System

### Local Transform Composition

Each node stores position, rotation, and scale separately. The local matrix is composed from these three components when marked dirty:

```typescript
const composeLocalMatrix = (node: NodeBase): Mat4 => {
  // Compose: Translation × Rotation × Scale
  // Result is a 4x4 matrix in column-major order
  const t = node.position
  const r = node.rotation
  const s = node.scale

  // Convert quaternion to rotation matrix components
  const x2 = r.x + r.x, y2 = r.y + r.y, z2 = r.z + r.z
  const xx = r.x * x2, xy = r.x * y2, xz = r.x * z2
  const yy = r.y * y2, yz = r.y * z2, zz = r.z * z2
  const wx = r.w * x2, wy = r.w * y2, wz = r.w * z2

  return mat4Create(
    (1 - (yy + zz)) * s.x,  (xy + wz) * s.x,        (xz - wy) * s.x,        0,
    (xy - wz) * s.y,        (1 - (xx + zz)) * s.y,   (yz + wx) * s.y,        0,
    (xz + wy) * s.z,        (yz - wx) * s.z,          (1 - (xx + yy)) * s.z, 0,
    t.x,                     t.y,                      t.z,                    1
  )
}
```

### World Matrix Computation

The world matrix places a node in global space. It is the product of the parent's world matrix and the node's local matrix:

```
worldMatrix = parent.worldMatrix × localMatrix
```

For the root (Scene), the world matrix equals the local matrix.

```typescript
const updateWorldMatrix = (node: NodeBase): void => {
  if (!node.dirty) return

  node.localMatrix = composeLocalMatrix(node)

  if (node.parent) {
    node.worldMatrix = mat4Multiply(node.parent.worldMatrix, node.localMatrix)
  } else {
    node.worldMatrix = mat4Copy(node.localMatrix)
  }

  node.dirty = false
}
```

### Dirty Flag Propagation

When any transform property changes, the node and **all descendants** are marked dirty. This ensures that the next world matrix update recomputes the entire subtree:

```typescript
const markDirty = (node: NodeBase): void => {
  if (node.dirty) return  // already dirty, subtree must be dirty too
  node.dirty = true
  for (let i = 0; i < node.children.length; i++) {
    markDirty(node.children[i])
  }
}

const setPosition = (node: NodeBase, x: number, y: number, z: number): void => {
  node.position.x = x
  node.position.y = y
  node.position.z = z
  markDirty(node)
}

const setRotation = (node: NodeBase, q: Quat): void => {
  node.rotation = q
  markDirty(node)
}

const setScale = (node: NodeBase, x: number, y: number, z: number): void => {
  node.scale.x = x
  node.scale.y = y
  node.scale.z = z
  markDirty(node)
}
```

### Top-Down Update Traversal

Before rendering, the scene graph is traversed top-down. Only dirty nodes recompute their matrices. Because dirty propagation marks the entire subtree, a top-down pass guarantees that a parent is always updated before its children:

```typescript
const updateSceneGraph = (node: NodeBase): void => {
  updateWorldMatrix(node)
  for (let i = 0; i < node.children.length; i++) {
    updateSceneGraph(node.children[i])
  }
}

// Called once per frame before rendering
const prepareFrame = (scene: Scene, camera: Camera): void => {
  updateSceneGraph(scene)

  // Camera view matrix is the inverse of its world matrix
  camera.viewMatrix = mat4Invert(camera.worldMatrix)

  // Extract frustum planes from viewProjection
  const viewProjection = mat4Multiply(camera.projectionMatrix, camera.viewMatrix)
  camera.frustum = extractFrustumPlanes(viewProjection)
}
```

### Z-Up Right-Handed Conventions

Lynx uses a Z-up, right-handed coordinate system:

| Axis | Direction |
|------|-----------|
| +X   | Right     |
| +Y   | Forward (into the scene) |
| +Z   | Up        |

- The default camera looks along **-Y** (forward into the scene) with **+Z** as up.
- Gravity is along **-Z**.
- A character "facing forward" faces **-Y** by default.

```typescript
// Default camera orientation: looking along -Y, Z is up
const createDefaultCamera = (): Camera => {
  const camera = createCamera()
  // Position the camera 10 units back (along +Y) and 5 units up (along +Z)
  setPosition(camera, 0, 10, 5)
  // Look toward the origin — rotate to face -Y direction, tilted down
  camera.rotation = quatFromLookRotation(
    vec3Normalize(vec3Sub(vec3(0, 0, 0), camera.position)),  // direction toward origin
    vec3(0, 0, 1)                                             // Z-up
  )
  return camera
}
```

---

## 3. Frustum Culling

### AABB Per Node

Each Mesh node maintains a local-space axis-aligned bounding box computed from its geometry vertices. Non-mesh nodes (Groups, Bones) may optionally compute an AABB that encloses all children, but this is not required.

```typescript
const computeGeometryAABB = (geometry: Geometry): AABB => {
  const positions = geometry.attributes.position.data // Float32Array, xyz interleaved
  const min = vec3(Infinity, Infinity, Infinity)
  const max = vec3(-Infinity, -Infinity, -Infinity)

  for (let i = 0; i < positions.length; i += 3) {
    min.x = Math.min(min.x, positions[i])
    min.y = Math.min(min.y, positions[i + 1])
    min.z = Math.min(min.z, positions[i + 2])
    max.x = Math.max(max.x, positions[i])
    max.y = Math.max(max.y, positions[i + 1])
    max.z = Math.max(max.z, positions[i + 2])
  }

  return { min, max }
}
```

### World AABB

The world AABB is the local AABB transformed by the node's world matrix. This produces a conservative (enlarged) box. It is fast but not tight — rotated objects get a larger bounding box. This is acceptable because the cost of a tighter OBB test outweighs the savings from fewer false positives in typical scenes.

```typescript
const transformAABB = (aabb: AABB, matrix: Mat4): AABB => {
  // Transform all 8 corners and recompute min/max
  const corners = [
    vec3(aabb.min.x, aabb.min.y, aabb.min.z),
    vec3(aabb.max.x, aabb.min.y, aabb.min.z),
    vec3(aabb.min.x, aabb.max.y, aabb.min.z),
    vec3(aabb.max.x, aabb.max.y, aabb.min.z),
    vec3(aabb.min.x, aabb.min.y, aabb.max.z),
    vec3(aabb.max.x, aabb.min.y, aabb.max.z),
    vec3(aabb.min.x, aabb.max.y, aabb.max.z),
    vec3(aabb.max.x, aabb.max.y, aabb.max.z),
  ]

  const min = vec3(Infinity, Infinity, Infinity)
  const max = vec3(-Infinity, -Infinity, -Infinity)

  for (let i = 0; i < 8; i++) {
    const p = vec3TransformMat4(corners[i], matrix)
    min.x = Math.min(min.x, p.x)
    min.y = Math.min(min.y, p.y)
    min.z = Math.min(min.z, p.z)
    max.x = Math.max(max.x, p.x)
    max.y = Math.max(max.y, p.y)
    max.z = Math.max(max.z, p.z)
  }

  return { min, max }
}
```

### Frustum Plane Extraction

Six planes are extracted from the combined view-projection matrix. Each plane is represented as `(nx, ny, nz, d)` where the plane equation is `nx*x + ny*y + nz*z + d = 0`. Points on the positive side of the normal are inside the frustum for that plane.

```typescript
interface Plane {
  nx: number
  ny: number
  nz: number
  d: number
}

const extractFrustumPlanes = (vp: Mat4): FrustumPlanes => {
  // vp is column-major: vp[col * 4 + row]
  const m = vp

  const left:   Plane = normalizePlane(m[3]+m[0], m[7]+m[4], m[11]+m[8],  m[15]+m[12])
  const right:  Plane = normalizePlane(m[3]-m[0], m[7]-m[4], m[11]-m[8],  m[15]-m[12])
  const bottom: Plane = normalizePlane(m[3]+m[1], m[7]+m[5], m[11]+m[9],  m[15]+m[13])
  const top:    Plane = normalizePlane(m[3]-m[1], m[7]-m[5], m[11]-m[9],  m[15]-m[13])
  const near:   Plane = normalizePlane(m[3]+m[2], m[7]+m[6], m[11]+m[10], m[15]+m[14])
  const far:    Plane = normalizePlane(m[3]-m[2], m[7]-m[6], m[11]-m[10], m[15]-m[14])

  return { planes: [left, right, bottom, top, near, far] }
}

const normalizePlane = (nx: number, ny: number, nz: number, d: number): Plane => {
  const len = Math.sqrt(nx * nx + ny * ny + nz * nz)
  return { nx: nx / len, ny: ny / len, nz: nz / len, d: d / len }
}
```

### AABB vs Frustum Test

An AABB is tested against all six planes. For each plane, the "positive vertex" (the corner of the AABB most in the direction of the plane normal) is checked. If the positive vertex is behind any plane, the AABB is fully outside the frustum and the mesh is culled. Early-out on the first failing plane makes this fast in practice.

```typescript
const isAABBInFrustum = (aabb: AABB, frustum: FrustumPlanes): boolean => {
  for (let i = 0; i < 6; i++) {
    const plane = frustum.planes[i]

    // Find the positive vertex (p-vertex) — the corner most aligned with the plane normal
    const px = plane.nx >= 0 ? aabb.max.x : aabb.min.x
    const py = plane.ny >= 0 ? aabb.max.y : aabb.min.y
    const pz = plane.nz >= 0 ? aabb.max.z : aabb.min.z

    // If the p-vertex is behind this plane, the entire AABB is outside
    if (plane.nx * px + plane.ny * py + plane.nz * pz + plane.d < 0) {
      return false
    }
  }

  return true
}
```

### Hierarchical Culling

If a parent node is culled, all its children are skipped entirely. This avoids testing potentially thousands of descendant nodes when an entire subtree is off-screen:

```typescript
const collectVisibleMeshes = (
  node: NodeBase,
  frustum: FrustumPlanes,
  result: Mesh[]
): void => {
  if (!node.visible) return

  // If this node has an AABB, test it against the frustum
  if (node.aabb) {
    const worldAABB = transformAABB(node.aabb, node.worldMatrix)
    if (!isAABBInFrustum(worldAABB, frustum)) {
      return  // culled — skip this node and all children
    }
  }

  // Collect this node if it is a mesh
  if (node.type === 'Mesh' || node.type === 'SkinnedMesh') {
    result.push(node as Mesh)
  }

  // Recurse into children
  for (let i = 0; i < node.children.length; i++) {
    collectVisibleMeshes(node.children[i], frustum, result)
  }
}
```

---

## 4. Scene Graph Operations

### add / remove

```typescript
const addChild = (parent: NodeBase, child: NodeBase): void => {
  // Remove from previous parent if any
  if (child.parent) {
    removeChild(child.parent, child)
  }

  child.parent = parent
  parent.children.push(child)
  markDirty(child) // world matrix must be recomputed under new parent
}

const removeChild = (parent: NodeBase, child: NodeBase): void => {
  const index = parent.children.indexOf(child)
  if (index === -1) return

  parent.children.splice(index, 1)
  child.parent = null

  // The renderer's command list is now stale and must be rebuilt
  markRendererDirty()
}
```

### traverse

Depth-first traversal visits every node in the subtree. The callback receives the current node. Return `false` from the callback to skip that node's children.

```typescript
const traverse = (node: NodeBase, callback: (node: NodeBase) => boolean | void): void => {
  const shouldContinue = callback(node)
  if (shouldContinue === false) return

  for (let i = 0; i < node.children.length; i++) {
    traverse(node.children[i], callback)
  }
}

// Example: find all meshes in the scene
const findAllMeshes = (scene: Scene): Mesh[] => {
  const meshes: Mesh[] = []
  traverse(scene, (node) => {
    if (node.type === 'Mesh' || node.type === 'SkinnedMesh') {
      meshes.push(node as Mesh)
    }
  })
  return meshes
}
```

### getVisibleMeshes

The primary query for rendering. Updates the scene graph, then frustum-culls to produce a flat list of visible meshes:

```typescript
const getVisibleMeshes = (scene: Scene, camera: Camera): Mesh[] => {
  // 1. Update all dirty world matrices
  updateSceneGraph(scene)

  // 2. Compute camera view + frustum
  camera.viewMatrix = mat4Invert(camera.worldMatrix)
  const viewProjection = mat4Multiply(camera.projectionMatrix, camera.viewMatrix)
  camera.frustum = extractFrustumPlanes(viewProjection)

  // 3. Collect visible meshes via hierarchical frustum culling
  const result: Mesh[] = []
  collectVisibleMeshes(scene, camera.frustum, result)
  return result
}
```

---

## 5. Parent/Children and Groups

### How It Works

Any node can have children. This is identical to Three.js. When a child is added to a parent, the child's world matrix becomes `parent.worldMatrix * child.localMatrix`. Moving the parent moves everything underneath it.

`Group` is a convenience class. It is functionally identical to `Node` — it adds no new fields or behavior. Its purpose is semantic: it signals to the reader that this node exists solely to group children together.

```typescript
const createGroup = (name?: string): Group => ({
  ...createNode(),
  type: 'Group',
  name: name ?? 'Group',
})
```

### Practical Usage

Groups are useful for:

- **Organizing objects**: Group all parts of a vehicle so they move together.
- **Applying a shared transform**: Offset or rotate an entire group of objects.
- **Selective visibility**: Set `group.visible = false` to hide everything inside.

```typescript
// A tank made of several parts, grouped so they move as one unit
const tank = createGroup('tank')
const hull = createMesh(hullGeometry, armorMaterial)
const turret = createGroup('turret')
const barrel = createMesh(barrelGeometry, metalMaterial)

addChild(tank, hull)
addChild(tank, turret)
addChild(turret, barrel)

// Position the turret on top of the hull
setPosition(turret, 0, 0, 1.5)  // Z-up: turret sits 1.5 units above hull origin

// Position the barrel extending forward from the turret
setPosition(barrel, 0, -2, 0)   // -Y is forward: barrel extends 2 units forward

// Move the entire tank — hull, turret, and barrel all follow
setPosition(tank, 10, 20, 0)

// Rotate just the turret — barrel follows, hull stays
const turretRotation = quatFromAxisAngle(vec3(0, 0, 1), Math.PI / 4)  // 45 degrees around Z-up
setRotation(turret, turretRotation)
```

---

## 6. Bone Attachment

### The Principle

Bones are regular nodes in the scene graph. They participate in the same parent-child hierarchy as everything else. To attach any object to a bone, you parent it to that bone with `addChild`. The attached object's world matrix is computed as:

```
worldMatrix = bone.worldMatrix × attachedObject.localMatrix
```

No special API. No attachment points. No socket system. The scene graph handles it.

### How Skeletons Fit In

A `Skeleton` is a flat list of `Bone` nodes. These bones form a tree (root bone has child bones, those have child bones, etc.). During skeletal animation, each bone's local transform is updated by the animation system, and the scene graph's dirty-flag mechanism ensures world matrices are recomputed.

When you parent a static mesh to a bone, that mesh becomes a child of the bone node. It moves with the bone exactly as if it were parented to any other node.

```typescript
// --- Skeleton hierarchy (simplified) ---
// spine
//   └── upperArm
//         └── forearm
//               └── hand    <-- we attach the sword here

// The skeleton is already set up and animating.
// To attach a sword to the hand bone:

const swordMesh = createMesh(swordGeometry, swordMaterial)

// Position and rotate the sword relative to the hand
setPosition(swordMesh, 0, 0, 0.1)    // slight offset in Z (up)
setRotation(swordMesh, quatFromAxisAngle(vec3(1, 0, 0), -Math.PI / 2))  // grip alignment

// Attach — this is it. One line.
addChild(handBone, swordMesh)

// Now when the animation moves the hand bone, the sword follows.
// swordMesh.worldMatrix = handBone.worldMatrix × swordMesh.localMatrix

// To detach:
removeChild(handBone, swordMesh)
```

### Why This Works

The scene graph does not distinguish between "regular" children and "attached" objects. A bone is just a node. A mesh is just a node. Parenting one to the other is the same operation as parenting a mesh to a group. The world matrix math is identical:

```
// Mesh parented to a Group:
mesh.worldMatrix = group.worldMatrix × mesh.localMatrix

// Sword parented to a Bone:
sword.worldMatrix = handBone.worldMatrix × sword.localMatrix

// They are the same computation.
```

This eliminates an entire category of bugs and special cases. Bone attachment is not a feature — it is an emergent property of the scene graph.

---

## Complete Example: Building a Scene

```typescript
// --- Create the scene ---
const scene = createScene()
scene.ambientLight = { color: vec3(1, 1, 1), intensity: 0.3 }
scene.background = vec3(0.1, 0.1, 0.15)
scene.fog = { color: vec3(0.1, 0.1, 0.15), near: 50, far: 200 }

// --- Create a camera ---
const camera = createCamera()
camera.fov = Math.PI / 3                      // 60 degrees
camera.aspect = window.innerWidth / window.innerHeight
camera.near = 0.1
camera.far = 500
camera.projectionMatrix = mat4Perspective(camera.fov, camera.aspect, camera.near, camera.far)

// Position: 10 units back (+Y), 5 units up (+Z)
setPosition(camera, 0, 10, 5)
camera.rotation = quatFromLookRotation(
  vec3Normalize(vec3(0, -10, -5)),  // look toward origin
  vec3(0, 0, 1)                     // Z-up
)
addChild(scene, camera)

// --- Create a directional light (sun) ---
const sun = createDirectionalLight()
sun.color = vec3(1, 0.95, 0.8)
sun.intensity = 1.2
sun.shadow = {
  enabled: true,
  mapSize: 2048,
  bias: 0.0005,
  near: 0.5,
  far: 100,
  bounds: 30,
}
// Light points downward and slightly from the side
sun.rotation = quatFromLookRotation(
  vec3Normalize(vec3(-0.3, -0.5, -1)),  // direction toward ground
  vec3(0, 1, 0)
)
addChild(scene, sun)

// --- Create ground plane ---
const ground = createMesh(createPlaneGeometry(100, 100), grassMaterial)
// Plane is already in XY plane by default; in Z-up that means it lies flat. Perfect.
addChild(scene, ground)

// --- Create a group of crates ---
const crateGroup = createGroup('crates')
setPosition(crateGroup, 5, -3, 0)
addChild(scene, crateGroup)

for (let i = 0; i < 4; i++) {
  const crate = createMesh(boxGeometry, crateMaterial)
  setPosition(crate, i * 1.5, 0, 0)
  addChild(crateGroup, crate)
}

// Stack one crate on top
const topCrate = createMesh(boxGeometry, crateMaterial)
setPosition(topCrate, 0.75, 0, 1.0)  // Z-up: 1.0 unit above
addChild(crateGroup, topCrate)

// --- Create a character with a weapon ---
const character = createGroup('character')
setPosition(character, 0, 0, 0)
addChild(scene, character)

const bodyMesh = createSkinnedMesh(characterGeometry, characterMaterial, characterSkeleton)
addChild(character, bodyMesh)

// Attach a sword to the right hand bone
const rightHandBone = characterSkeleton.bones.find(b => b.name === 'hand_R')!
const sword = createMesh(swordGeometry, swordMaterial)
setPosition(sword, 0, 0, 0.15)
setRotation(sword, quatFromAxisAngle(vec3(1, 0, 0), -Math.PI / 2))
addChild(rightHandBone, sword)

// --- Render loop ---
const frame = (): void => {
  // Animate (update skeleton, move objects, etc.)
  animationSystem.update(deltaTime)

  // Get visible meshes — handles scene graph update + frustum culling
  const visibleMeshes = getVisibleMeshes(scene, camera)

  // Submit to renderer
  renderer.render(visibleMeshes, camera, scene)

  requestAnimationFrame(frame)
}

requestAnimationFrame(frame)
```

### What Happens Each Frame

1. **Animation update** — the animation system modifies bone transforms (position, rotation), which calls `markDirty` on each changed bone.
2. **`getVisibleMeshes`** — calls `updateSceneGraph(scene)`, which walks the tree top-down and recomputes world matrices only for dirty nodes. Then extracts frustum planes from the camera and performs hierarchical frustum culling.
3. **Render** — the flat list of visible meshes is passed to the renderer, which sorts by material, issues draw calls, and presents the frame.

The sword attached to the hand bone updates automatically. When the animation system sets a new rotation on the hand bone, `markDirty` propagates to the sword. When `updateSceneGraph` runs, the hand bone's world matrix is recomputed first (it is higher in the tree), then the sword's world matrix is computed as `handBone.worldMatrix * sword.localMatrix`. The sword follows the hand perfectly, every frame, with no additional code.
