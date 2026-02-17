# Math Library

## Design Goals

1. **Zero allocation** — All operations write to caller-provided output
   parameters. No temporary objects created.
2. **Float32Array backed** — All math types are views into `Float32Array`,
   enabling direct GPU upload without conversion.
3. **Functional core** — Pure functions that operate on typed arrays. No
   classes or `this` binding in the hot path.
4. **Scratch pool** — Pre-allocated temporary registers for intermediate
   calculations, reset each frame.
5. **Column-major matrices** — Matches WebGL/WebGPU convention for direct
   upload to uniform buffers.

## Types

All math types are type aliases over `Float32Array`:

```typescript
// Type aliases — zero overhead, just documentation
type Vec2 = Float32Array  // length 2: [x, y]
type Vec3 = Float32Array  // length 3: [x, y, z]
type Vec4 = Float32Array  // length 4: [x, y, z, w]
type Quat = Float32Array  // length 4: [x, y, z, w]
type Mat4 = Float32Array  // length 16: column-major 4×4

// AABB is a pair of Vec3s stored contiguously
type AABB = Float32Array  // length 6: [minX, minY, minZ, maxX, maxY, maxZ]
```

### Creation Functions

```typescript
const vec3Create = (x = 0, y = 0, z = 0): Vec3 => {
  const out = new Float32Array(3)
  out[0] = x; out[1] = y; out[2] = z
  return out
}

const quatCreate = (x = 0, y = 0, z = 0, w = 1): Quat => {
  const out = new Float32Array(4)
  out[0] = x; out[1] = y; out[2] = z; out[3] = w
  return out
}

const mat4Create = (): Mat4 => {
  const out = new Float32Array(16)
  out[0] = 1; out[5] = 1; out[10] = 1; out[15] = 1  // identity
  return out
}
```

## Vec3 Operations

```typescript
// All operations: first arg is output, last args are inputs
const vec3Set = (out: Vec3, x: number, y: number, z: number): Vec3
const vec3Copy = (out: Vec3, a: Vec3): Vec3
const vec3Add = (out: Vec3, a: Vec3, b: Vec3): Vec3
const vec3Sub = (out: Vec3, a: Vec3, b: Vec3): Vec3
const vec3Scale = (out: Vec3, a: Vec3, s: number): Vec3
const vec3Dot = (a: Vec3, b: Vec3): number
const vec3Cross = (out: Vec3, a: Vec3, b: Vec3): Vec3
const vec3Length = (a: Vec3): number
const vec3LengthSq = (a: Vec3): number
const vec3Normalize = (out: Vec3, a: Vec3): Vec3
const vec3Lerp = (out: Vec3, a: Vec3, b: Vec3, t: number): Vec3
const vec3Distance = (a: Vec3, b: Vec3): number
const vec3Negate = (out: Vec3, a: Vec3): Vec3
const vec3Min = (out: Vec3, a: Vec3, b: Vec3): Vec3
const vec3Max = (out: Vec3, a: Vec3, b: Vec3): Vec3
const vec3TransformMat4 = (out: Vec3, a: Vec3, m: Mat4): Vec3
const vec3TransformQuat = (out: Vec3, a: Vec3, q: Quat): Vec3
```

**Z-up helpers:**
```typescript
const VEC3_UP: Vec3    = new Float32Array([0, 0, 1])   // Z is up
const VEC3_RIGHT: Vec3 = new Float32Array([1, 0, 0])   // X is right
const VEC3_FORWARD: Vec3 = new Float32Array([0, 1, 0]) // Y is forward
```

## Quaternion Operations

```typescript
const quatSet = (out: Quat, x: number, y: number, z: number, w: number): Quat
const quatIdentity = (out: Quat): Quat
const quatMultiply = (out: Quat, a: Quat, b: Quat): Quat
const quatSlerp = (out: Quat, a: Quat, b: Quat, t: number): Quat
const quatNormalize = (out: Quat, a: Quat): Quat
const quatConjugate = (out: Quat, a: Quat): Quat
const quatInvert = (out: Quat, a: Quat): Quat
const quatFromAxisAngle = (out: Quat, axis: Vec3, radians: number): Quat
const quatFromEulerZUp = (out: Quat, x: number, y: number, z: number): Quat
const quatLookRotation = (out: Quat, forward: Vec3, up?: Vec3): Quat
```

### Euler Convenience (Z-up)

Since Mantis is Z-up, Euler angles follow the ZYX convention
(yaw around Z, pitch around Y, roll around X):

```typescript
const quatFromEulerZUp = (out: Quat, roll: number, pitch: number, yaw: number): Quat => {
  // ZYX order: first rotate around Z (yaw), then Y (pitch), then X (roll)
  const cx = Math.cos(roll * 0.5),  sx = Math.sin(roll * 0.5)
  const cy = Math.cos(pitch * 0.5), sy = Math.sin(pitch * 0.5)
  const cz = Math.cos(yaw * 0.5),   sz = Math.sin(yaw * 0.5)

  out[0] = sx * cy * cz - cx * sy * sz
  out[1] = cx * sy * cz + sx * cy * sz
  out[2] = cx * cy * sz - sx * sy * cz
  out[3] = cx * cy * cz + sx * sy * sz
  return out
}
```

## Mat4 Operations

```typescript
const mat4Identity = (out: Mat4): Mat4
const mat4Copy = (out: Mat4, a: Mat4): Mat4
const mat4Multiply = (out: Mat4, a: Mat4, b: Mat4): Mat4
const mat4Invert = (out: Mat4, a: Mat4): Mat4
const mat4Transpose = (out: Mat4, a: Mat4): Mat4
const mat4Determinant = (a: Mat4): number

// Transform composition
const mat4FromTranslation = (out: Mat4, v: Vec3): Mat4
const mat4FromRotation = (out: Mat4, q: Quat): Mat4
const mat4FromScaling = (out: Mat4, v: Vec3): Mat4
const mat4Compose = (out: Mat4, position: Vec3, rotation: Quat, scale: Vec3): Mat4
const mat4Decompose = (m: Mat4, position: Vec3, rotation: Quat, scale: Vec3): void

// Projection
const mat4Perspective = (out: Mat4, fov: number, aspect: number, near: number, far: number): Mat4
const mat4Ortho = (out: Mat4, left: number, right: number, bottom: number, top: number, near: number, far: number): Mat4

// View
const mat4LookAt = (out: Mat4, eye: Vec3, target: Vec3, up: Vec3): Mat4

// Extract
const mat4GetTranslation = (out: Vec3, m: Mat4): Vec3
const mat4GetScaling = (out: Vec3, m: Mat4): Vec3
const mat4GetRotation = (out: Quat, m: Mat4): Quat
```

### Column-Major Layout

Matrices are stored in column-major order, matching WebGL/WebGPU's expected
layout for `uniformMatrix4fv` and WGSL/GLSL `mat4`:

```
Column-major indices:
┌──────────────────────────────┐
│  m[0]   m[4]   m[8]   m[12] │   Row 0
│  m[1]   m[5]   m[9]   m[13] │   Row 1
│  m[2]   m[6]   m[10]  m[14] │   Row 2
│  m[3]   m[7]   m[11]  m[15] │   Row 3
└──────────────────────────────┘
  Col 0   Col 1  Col 2   Col 3

Translation = [m[12], m[13], m[14]]  (column 3, rows 0–2)
```

### Z-up Perspective Matrix

The standard perspective matrix assumes Y-up. Mantis's perspective matrix
accounts for Z-up by swapping the Y and Z basis:

```typescript
const mat4PerspectiveZUp = (out: Mat4, fovY: number, aspect: number, near: number, far: number): Mat4 => {
  const f = 1.0 / Math.tan(fovY * 0.5)
  const rangeInv = 1.0 / (near - far)

  // Standard perspective with Z-up adjustment
  // Camera looks along -Y in Z-up world, up is +Z
  out[0]  = f / aspect
  out[1]  = 0
  out[2]  = 0
  out[3]  = 0

  out[4]  = 0
  out[5]  = 0
  out[6]  = (far + near) * rangeInv
  out[7]  = -1

  out[8]  = 0
  out[9]  = f
  out[10] = 0
  out[11] = 0

  out[12] = 0
  out[13] = 0
  out[14] = 2 * far * near * rangeInv
  out[15] = 0

  return out
}
```

## AABB Operations

```typescript
const aabbCreate = (): AABB => new Float32Array(6)  // [minX,minY,minZ, maxX,maxY,maxZ]
const aabbFromPoints = (out: AABB, positions: Float32Array, count: number): AABB
const aabbExpand = (out: AABB, a: AABB, b: AABB): AABB
const aabbContainsPoint = (aabb: AABB, point: Vec3): boolean
const aabbIntersectsAABB = (a: AABB, b: AABB): boolean
const aabbSurfaceArea = (aabb: AABB): number
const aabbTransform = (out: AABB, a: AABB, m: Mat4): AABB
const aabbCenter = (out: Vec3, aabb: AABB): Vec3
```

## Frustum

A frustum is defined by 6 planes (near, far, left, right, top, bottom):

```typescript
type Frustum = Float32Array  // length 24: 6 planes × 4 floats (nx, ny, nz, d)

const frustumFromVPMatrix = (out: Frustum, vp: Mat4): Frustum
const frustumTestAABB = (frustum: Frustum, aabb: AABB): FrustumResult

const enum FrustumResult {
  OUTSIDE = 0,
  INTERSECT = 1,
  INSIDE = 2,
}
```

### Plane Extraction

Frustum planes are extracted directly from the view-projection matrix using the
Gribb-Hartmann method:

```typescript
const frustumFromVPMatrix = (out: Frustum, m: Mat4): Frustum => {
  // Left plane:   row3 + row0
  out[0] = m[3] + m[0];  out[1] = m[7] + m[4];  out[2] = m[11] + m[8];   out[3] = m[15] + m[12]
  // Right plane:  row3 - row0
  out[4] = m[3] - m[0];  out[5] = m[7] - m[4];  out[6] = m[11] - m[8];   out[7] = m[15] - m[12]
  // Bottom plane: row3 + row1
  out[8] = m[3] + m[1];  out[9] = m[7] + m[5];  out[10] = m[11] + m[9];  out[11] = m[15] + m[13]
  // Top plane:    row3 - row1
  out[12] = m[3] - m[1]; out[13] = m[7] - m[5]; out[14] = m[11] - m[9];  out[15] = m[15] - m[13]
  // Near plane:   row3 + row2
  out[16] = m[3] + m[2]; out[17] = m[7] + m[6]; out[18] = m[11] + m[10]; out[19] = m[15] + m[14]
  // Far plane:    row3 - row2
  out[20] = m[3] - m[2]; out[21] = m[7] - m[6]; out[22] = m[11] - m[10]; out[23] = m[15] - m[14]

  // Normalize planes
  for (let i = 0; i < 6; i++) {
    const o = i * 4
    const len = Math.sqrt(out[o] * out[o] + out[o+1] * out[o+1] + out[o+2] * out[o+2])
    const invLen = 1 / len
    out[o] *= invLen; out[o+1] *= invLen; out[o+2] *= invLen; out[o+3] *= invLen
  }

  return out
}
```

## Scratch Pool

The scratch pool provides temporary math objects that are valid for the current
frame. At the end of each frame, the pool's write pointer resets to 0 —
"freeing" all allocations instantly with zero cost.

```typescript
// Pool sizes (pre-allocated once at engine init)
const VEC3_POOL_SIZE = 64   // 64 scratch Vec3s
const VEC4_POOL_SIZE = 32
const MAT4_POOL_SIZE = 32
const QUAT_POOL_SIZE = 32

// Backing storage
const vec3PoolData = new Float32Array(VEC3_POOL_SIZE * 3)
const mat4PoolData = new Float32Array(MAT4_POOL_SIZE * 16)
const quatPoolData = new Float32Array(QUAT_POOL_SIZE * 4)

let vec3PoolPtr = 0
let mat4PoolPtr = 0
let quatPoolPtr = 0

// Acquire a temporary Vec3 (valid until frame end)
const vec3Pool = {
  acquire: (): Vec3 => {
    const offset = vec3PoolPtr * 3
    vec3PoolPtr++
    if (vec3PoolPtr > VEC3_POOL_SIZE) throw new Error('Vec3 pool exhausted')
    return vec3PoolData.subarray(offset, offset + 3)
  },
  reset: () => { vec3PoolPtr = 0 },
}

// Similarly for mat4Pool, quatPool, etc.
```

### Usage Pattern

```typescript
// In user code or internal engine code:
const temp = vec3Pool.acquire()
vec3Sub(temp, targetPos, currentPos)
vec3Normalize(temp, temp)
// ... use temp ...
// No need to "free" — pool resets at frame end
```

### Why This Works

In a game render loop, temporary math objects follow a strict stack-like
pattern: they're created, used, and abandoned within the same frame. The
scratch pool exploits this by allocating sequentially and resetting the pointer
each frame. This is:

- **Zero GC** — No heap objects are created or collected
- **Zero cost reset** — Just set an integer to 0
- **Cache friendly** — All temporaries are adjacent in memory
- **Safe** — Pool exhaustion throws an error (debug only, stripped in prod)

## Numerical Conventions

| Convention | Value | Rationale |
|---|---|---|
| Angles | Radians | GPU shaders use radians |
| Matrix order | Column-major | WebGL/WebGPU convention |
| Quaternion order | [x, y, z, w] | glTF convention, matches WGSL |
| Coordinate system | Z-up, right-handed | X right, Y forward, Z up |
| Rotation order | ZYX (intrinsic) | Yaw-pitch-roll in Z-up |
| Winding order | Counter-clockwise | WebGL/WebGPU front face default |
