# Math Utilities and Coordinate System

## Coordinate System

**Z is up, right-handed coordinate system**

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

## Math Library Design

Wren's math utilities are custom-built with the following principles:

1. **Z-up right-handed**: The coordinate system is baked into all operations, not converted at runtime
2. **Float32Array-backed**: All vectors and matrices use typed arrays for performance and GPU compatibility
3. **No allocations**: Operations mutate destination arrays; no new objects are created
4. **Minimal API**: Only the operations needed by the engine are implemented

## Core Types

```ts
// Vectors are Float32Array
type Vec3 = Float32Array  // [x, y, z]
type Vec4 = Float32Array  // [x, y, z, w]

// Quaternions for rotations
type Quat = Float32Array  // [x, y, z, w]

// Matrices (column-major, matching WebGL/WebGPU)
type Mat4 = Float32Array  // 16 floats
type Mat3 = Float32Array  // 9 floats

// Axis-aligned bounding box
interface AABB {
  min: Float32Array  // [x, y, z]
  max: Float32Array  // [x, y, z]
}
```

## Vector Operations

```ts
// Create
const vec3Create = (): Float32Array => new Float32Array(3)
const vec3FromValues = (x: number, y: number, z: number): Float32Array =>
  new Float32Array([x, y, z])

// Copy
const vec3Copy = (out: Float32Array, a: Float32Array): Float32Array => {
  out[0] = a[0]
  out[1] = a[1]
  out[2] = a[2]
  return out
}

// Set
const vec3Set = (out: Float32Array, x: number, y: number, z: number): Float32Array => {
  out[0] = x
  out[1] = y
  out[2] = z
  return out
}

// Add
const vec3Add = (out: Float32Array, a: Float32Array, b: Float32Array): Float32Array => {
  out[0] = a[0] + b[0]
  out[1] = a[1] + b[1]
  out[2] = a[2] + b[2]
  return out
}

// Scale
const vec3Scale = (out: Float32Array, a: Float32Array, scale: number): Float32Array => {
  out[0] = a[0] * scale
  out[1] = a[1] * scale
  out[2] = a[2] * scale
  return out
}

// Length
const vec3Length = (a: Float32Array): number => {
  return Math.sqrt(a[0] * a[0] + a[1] * a[1] + a[2] * a[2])
}

// Normalize
const vec3Normalize = (out: Float32Array, a: Float32Array): Float32Array => {
  const len = vec3Length(a)
  if (len > 0) {
    out[0] = a[0] / len
    out[1] = a[1] / len
    out[2] = a[2] / len
  }
  return out
}

// Dot product
const vec3Dot = (a: Float32Array, b: Float32Array): number => {
  return a[0] * b[0] + a[1] * b[1] + a[2] * b[2]
}

// Cross product
const vec3Cross = (out: Float32Array, a: Float32Array, b: Float32Array): Float32Array => {
  const ax = a[0], ay = a[1], az = a[2]
  const bx = b[0], by = b[1], bz = b[2]
  out[0] = ay * bz - az * by
  out[1] = az * bx - ax * bz
  out[2] = ax * by - ay * bx
  return out
}

// Transform by matrix
const vec3TransformMat4 = (out: Float32Array, a: Float32Array, m: Float32Array): Float32Array => {
  const x = a[0], y = a[1], z = a[2]
  const w = m[3] * x + m[7] * y + m[11] * z + m[15]
  out[0] = (m[0] * x + m[4] * y + m[8] * z + m[12]) / w
  out[1] = (m[1] * x + m[5] * y + m[9] * z + m[13]) / w
  out[2] = (m[2] * x + m[6] * y + m[10] * z + m[14]) / w
  return out
}
```

## Quaternion Operations

```ts
// Identity
const quatIdentity = (): Float32Array => new Float32Array([0, 0, 0, 1])

// From euler angles (Z-up)
const quatFromEuler = (out: Float32Array, x: number, y: number, z: number): Float32Array => {
  // Implementation using Z-up convention
  // ...
  return out
}

// Spherical linear interpolation
const quatSlerp = (out: Float32Array, a: Float32Array, b: Float32Array, t: number): Float32Array => {
  // Ensure shortest path
  let dot = a[0] * b[0] + a[1] * b[1] + a[2] * b[2] + a[3] * b[3]
  const sign = dot < 0 ? -1 : 1
  dot = Math.abs(dot)

  // Implementation...
  return out
}

// Multiply
const quatMultiply = (out: Float32Array, a: Float32Array, b: Float32Array): Float32Array => {
  const ax = a[0], ay = a[1], az = a[2], aw = a[3]
  const bx = b[0], by = b[1], bz = b[2], bw = b[3]
  out[0] = ax * bw + aw * bx + ay * bz - az * by
  out[1] = ay * bw + aw * by + az * bx - ax * bz
  out[2] = az * bw + aw * bz + ax * by - ay * bx
  out[3] = aw * bw - ax * bx - ay * by - az * bz
  return out
}
```

## Matrix Operations

```ts
// Identity
const mat4Identity = (): Float32Array => new Float32Array([
  1, 0, 0, 0,
  0, 1, 0, 0,
  0, 0, 1, 0,
  0, 0, 0, 1
])

// Perspective projection (Z-up)
const mat4Perspective = (
  out: Float32Array,
  fov: number,
  aspect: number,
  near: number,
  far: number
): Float32Array => {
  const f = 1.0 / Math.tan(fov / 2)
  out[0] = f / aspect
  out[1] = 0
  out[2] = 0
  out[3] = 0
  out[4] = 0
  out[5] = f
  out[6] = 0
  out[7] = 0
  out[8] = 0
  out[9] = 0
  out[10] = (far + near) / (near - far)
  out[11] = -1
  out[12] = 0
  out[13] = 0
  out[14] = (2 * far * near) / (near - far)
  out[15] = 0
  return out
}

// Look-at (Z-up)
const mat4LookAt = (
  out: Float32Array,
  eye: Float32Array,
  target: Float32Array,
  up: Float32Array
): Float32Array => {
  // Z-up specific implementation
  // Forward = normalize(target - eye)
  // Right = normalize(cross(forward, up))
  // Up = cross(right, forward)
  // ...
  return out
}

// Multiply
const mat4Multiply = (out: Float32Array, a: Float32Array, b: Float32Array): Float32Array => {
  // Standard 4x4 matrix multiplication
  // ...
  return out
}

// Compose from translation, rotation (quat), scale
const mat4Compose = (
  out: Float32Array,
  position: Float32Array,
  rotation: Float32Array,
  scale: Float32Array
): Float32Array => {
  // Build TRS matrix
  // ...
  return out
}

// Invert
const mat4Invert = (out: Float32Array, a: Float32Array): Float32Array => {
  // Matrix inversion
  // ...
  return out
}

// Normal matrix (inverse transpose of upper-left 3x3)
const mat3NormalFromMat4 = (out: Float32Array, a: Float32Array): Float32Array => {
  // Extract 3x3, invert, transpose
  // ...
  return out
}
```

## AABB Operations

```ts
const createAABB = (): AABB => ({
  min: new Float32Array([Infinity, Infinity, Infinity]),
  max: new Float32Array([-Infinity, -Infinity, -Infinity])
})

const expandAABB = (aabb: AABB, point: Float32Array): void => {
  aabb.min[0] = Math.min(aabb.min[0], point[0])
  aabb.min[1] = Math.min(aabb.min[1], point[1])
  aabb.min[2] = Math.min(aabb.min[2], point[2])
  aabb.max[0] = Math.max(aabb.max[0], point[0])
  aabb.max[1] = Math.max(aabb.max[1], point[1])
  aabb.max[2] = Math.max(aabb.max[2], point[2])
}

const transformAABB = (out: AABB, aabb: AABB, matrix: Float32Array): void => {
  // Transform all 8 corners and compute new bounds
  // ...
}
```

## Temporary Vector Pool

To avoid allocations in hot paths, Wren uses module-level temporary vectors:

```ts
// math/temp.ts
const _tempVec3A = new Float32Array(3)
const _tempVec3B = new Float32Array(3)
const _tempVec3C = new Float32Array(3)
const _tempMat4A = new Float32Array(16)
const _tempMat4B = new Float32Array(16)
const _tempQuatA = new Float32Array(4)

// Usage in functions:
const someFunction = () => {
  vec3Set(_tempVec3A, 1, 2, 3)
  vec3Normalize(_tempVec3B, _tempVec3A)
  // Use _tempVec3B...
}
```

These are safe to use because they're never held across asynchronous boundaries.
