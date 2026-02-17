# Math — Utilities and Coordinate System

## Coordinate System

**Z-up, right-handed.** X = right, Y = forward, Z = up.

```
        Z (up)
        │
        │
        │
        └───── Y (forward)
       /
      /
     X (right)
```

All internal math, scene graph transforms, and physics use this convention. GLTF files (Y-up) are converted during import by swapping Y↔Z and adjusting rotations.

## Math Library

Fennec ships its own minimal math library rather than depending on gl-matrix or similar. This avoids:
- Extra dependency weight
- Impedance mismatch with our coordinate system
- Allocation patterns we can't control

### Types

- `Vec2`, `Vec3`, `Vec4` — backed by `Float32Array(2|3|4)`
- `Mat3`, `Mat4` — backed by `Float32Array(9|16)`, column-major
- `Quat` — backed by `Float32Array(4)`, `(x, y, z, w)` layout
- `AABB` — `{ min: Vec3, max: Vec3 }`
- `Ray` — `{ origin: Vec3, direction: Vec3 }`
- `Frustum` — 6 planes as `Vec4` (nx, ny, nz, d)

### Allocation-Free API

All math operations support both allocation-free (output parameter) and convenience (returns new) forms:

```typescript
// Allocation-free (hot path)
vec3.add(out, a, b)
vec3.normalize(out, v)
mat4.multiply(out, a, b)

// Convenience (setup code)
const result = Vec3.add(a, b)
const normalized = Vec3.normalize(v)
const product = Mat4.multiply(a, b)
```

## Vec3 Operations

```typescript
namespace vec3 {
  // Creation
  create(): Vec3
  fromValues(x: number, y: number, z: number): Vec3
  clone(v: Vec3): Vec3
  copy(out: Vec3, v: Vec3): Vec3

  // Basic operations
  add(out: Vec3, a: Vec3, b: Vec3): Vec3
  subtract(out: Vec3, a: Vec3, b: Vec3): Vec3
  multiply(out: Vec3, a: Vec3, b: Vec3): Vec3  // Component-wise
  scale(out: Vec3, v: Vec3, s: number): Vec3
  scaleAndAdd(out: Vec3, a: Vec3, b: Vec3, s: number): Vec3  // out = a + b * s

  // Geometric operations
  dot(a: Vec3, b: Vec3): number
  cross(out: Vec3, a: Vec3, b: Vec3): Vec3
  length(v: Vec3): number
  lengthSquared(v: Vec3): number
  distance(a: Vec3, b: Vec3): number
  distanceSquared(a: Vec3, b: Vec3): number
  normalize(out: Vec3, v: Vec3): Vec3

  // Interpolation
  lerp(out: Vec3, a: Vec3, b: Vec3, t: number): Vec3
  slerp(out: Vec3, a: Vec3, b: Vec3, t: number): Vec3  // Spherical lerp

  // Transformation
  transformMat3(out: Vec3, v: Vec3, m: Mat3): Vec3
  transformMat4(out: Vec3, v: Vec3, m: Mat4): Vec3
  transformQuat(out: Vec3, v: Vec3, q: Quat): Vec3

  // Utility
  min(out: Vec3, a: Vec3, b: Vec3): Vec3
  max(out: Vec3, a: Vec3, b: Vec3): Vec3
  negate(out: Vec3, v: Vec3): Vec3
  inverse(out: Vec3, v: Vec3): Vec3  // Component-wise 1/x
}
```

## Mat4 Operations

```typescript
namespace mat4 {
  // Creation
  create(): Mat4
  identity(out: Mat4): Mat4
  clone(m: Mat4): Mat4
  copy(out: Mat4, m: Mat4): Mat4

  // Basic operations
  multiply(out: Mat4, a: Mat4, b: Mat4): Mat4
  transpose(out: Mat4, m: Mat4): Mat4
  invert(out: Mat4, m: Mat4): Mat4
  determinant(m: Mat4): number

  // Transformation construction
  fromTranslation(out: Mat4, v: Vec3): Mat4
  fromRotation(out: Mat4, axis: Vec3, angle: number): Mat4
  fromRotationZ(out: Mat4, angle: number): Mat4  // Z-up convention
  fromScale(out: Mat4, v: Vec3): Mat4
  fromQuat(out: Mat4, q: Quat): Mat4

  // TRS composition
  fromRotationTranslationScale(out: Mat4, q: Quat, v: Vec3, s: Vec3): Mat4

  // Projection matrices
  perspective(out: Mat4, fovy: number, aspect: number, near: number, far: number): Mat4
  ortho(out: Mat4, left: number, right: number, bottom: number, top: number, near: number, far: number): Mat4

  // View matrices
  lookAt(out: Mat4, eye: Vec3, target: Vec3, up: Vec3): Mat4

  // Decomposition
  getTranslation(out: Vec3, m: Mat4): Vec3
  getRotation(out: Quat, m: Mat4): Quat
  getScaling(out: Vec3, m: Mat4): Vec3
}
```

## Quaternion Operations

Quaternions are used for all rotations to avoid gimbal lock:

```typescript
namespace quat {
  // Creation
  create(): Quat
  identity(out: Quat): Quat
  fromAxisAngle(out: Quat, axis: Vec3, angle: number): Quat
  fromEuler(out: Quat, x: number, y: number, z: number): Quat  // Z-up Euler angles
  fromMat3(out: Quat, m: Mat3): Quat

  // Operations
  multiply(out: Quat, a: Quat, b: Quat): Quat
  conjugate(out: Quat, q: Quat): Quat
  invert(out: Quat, q: Quat): Quat
  normalize(out: Quat, q: Quat): Quat

  // Interpolation
  slerp(out: Quat, a: Quat, b: Quat, t: number): Quat  // Spherical lerp

  // Utility
  dot(a: Quat, b: Quat): number
  length(q: Quat): number
  angle(a: Quat, b: Quat): number  // Angle between rotations
}
```

## AABB Operations

Axis-Aligned Bounding Boxes for culling and raycasting:

```typescript
interface AABB {
  min: Vec3
  max: Vec3
}

namespace aabb {
  create(): AABB
  fromPoints(out: AABB, points: Vec3[]): AABB
  fromPositionSize(out: AABB, center: Vec3, size: Vec3): AABB

  // Queries
  contains(aabb: AABB, point: Vec3): boolean
  intersects(a: AABB, b: AABB): boolean
  intersectRay(aabb: AABB, ray: Ray): number | null  // Returns t or null

  // Transformations
  transform(out: AABB, aabb: AABB, mat: Mat4): AABB
  union(out: AABB, a: AABB, b: AABB): AABB
  expand(out: AABB, aabb: AABB, delta: Vec3): AABB

  // Utility
  center(out: Vec3, aabb: AABB): Vec3
  size(out: Vec3, aabb: AABB): Vec3
  surfaceArea(aabb: AABB): number  // For SAH
}
```

## Ray Operations

Rays for raycasting and picking:

```typescript
interface Ray {
  origin: Vec3
  direction: Vec3  // Should be normalized
}

namespace ray {
  create(origin: Vec3, direction: Vec3): Ray

  // Raycasting
  intersectTriangle(ray: Ray, a: Vec3, b: Vec3, c: Vec3): number | null  // Returns t
  intersectAABB(ray: Ray, aabb: AABB): number | null
  intersectSphere(ray: Ray, center: Vec3, radius: number): number | null

  // Utility
  at(out: Vec3, ray: Ray, t: number): Vec3  // out = origin + direction * t
  closestPoint(out: Vec3, ray: Ray, point: Vec3): Vec3  // Closest point on ray to point
}
```

## Frustum Operations

Frustum planes for frustum culling:

```typescript
interface Frustum {
  planes: Vec4[]  // 6 planes: [left, right, bottom, top, near, far]
                  // Each plane = (nx, ny, nz, d) where dot(normal, point) + d = 0
}

namespace frustum {
  create(): Frustum
  fromMatrix(out: Frustum, vp: Mat4): Frustum  // Extract from view-projection matrix

  // Culling
  containsPoint(frustum: Frustum, point: Vec3): boolean
  containsAABB(frustum: Frustum, aabb: AABB): boolean  // True if any part intersects
  intersectsSphere(frustum: Frustum, center: Vec3, radius: number): boolean
}
```

## Coordinate Conversions

### GLTF Y-up to Fennec Z-up

When loading GLTF files (Y-up, right-handed), convert to Z-up:

```typescript
const convertGLTFToZUp = (mat: Mat4) => {
  // Swap Y and Z axes
  const temp = mat4.create()
  mat4.fromValues(
    1, 0, 0, 0,
    0, 0, 1, 0,
    0, 1, 0, 0,
    0, 0, 0, 1,
  )
  mat4.multiply(mat, temp, mat)
}
```

### Spherical to Cartesian (Z-up)

Used by orbit controls:

```typescript
const sphericalToCartesian = (distance: number, azimuth: number, elevation: number): Vec3 => {
  const cosElev = Math.cos(elevation)
  return vec3.fromValues(
    distance * cosElev * Math.sin(azimuth),   // X (right)
    distance * cosElev * Math.cos(azimuth),   // Y (forward)
    distance * Math.sin(elevation),            // Z (up)
  )
}
```

## Performance Notes

- All math types use `Float32Array` for WebGL compatibility
- Column-major matrices match OpenGL/WebGL convention
- Allocation-free operations avoid GC pressure in hot paths
- SIMD not used (browser support still patchy), but could be added in the future

## Testing

Fennec's math library includes comprehensive unit tests:

```typescript
// Example tests
assert(vec3.equals(vec3.add(vec3.create(), [1,2,3], [4,5,6]), [5,7,9]))
assert(Math.abs(quat.angle(quat.identity(), quat.fromAxisAngle([0,0,1], Math.PI))) < 1e-6)
```

All operations are tested for correctness against known values and edge cases.
