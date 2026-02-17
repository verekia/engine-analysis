# MATH - Design Decisions

## Coordinate System

**Decision: Z-up, right-handed (universal agreement)**

```
       Z (up)
       │
       │
       └──── X (right)
      /
     /
    Y (forward)
```

- X = right, Y = forward, Z = up
- Cross product: X × Y = Z (right-handed)
- Matches Blender, Unreal Engine, many CAD tools
- Differs from Three.js/Unity (Y-up)

## Float32Array Backing

**Decision: All math types backed by Float32Array (universal agreement)**

```typescript
type Vec3 = Float32Array  // 3 elements
type Vec4 = Float32Array  // 4 elements
type Mat4 = Float32Array  // 16 elements
type Quat = Float32Array  // 4 elements [x, y, z, w]
```

- Compatible with WebGL/WebGPU uniform uploads (zero conversion)
- Better cache locality than object properties
- SIMD-friendly memory layout (future optimization path)

## API Style

**Decision: Pure functional API with type aliases + OOP convenience wrappers for setup code**

- Sources: Pure functional from Bonobo, Hyena, Lynx, Mantis, Wren (5/9); OOP wrappers from Fennec, Caracal, Rabbit (3/9)

Primary API (hot path):
```typescript
// Type aliases — zero overhead
type Vec3 = Float32Array

// Functional, output-parameter style (zero allocation)
const vec3Add = (out: Vec3, a: Vec3, b: Vec3): Vec3 => { ... }
const mat4Multiply = (out: Mat4, a: Mat4, b: Mat4): Mat4 => { ... }
const quatSlerp = (out: Quat, a: Quat, b: Quat, t: number): Quat => { ... }
```

Convenience API (setup/initialization code):
```typescript
// Allocating variants for non-hot-path usage
const vec3Create = (x = 0, y = 0, z = 0): Vec3 => {
  const out = new Float32Array(3)
  out[0] = x; out[1] = y; out[2] = z
  return out
}
```

Rationale: Type aliases provide zero overhead and maximum tree-shaking. Standalone functions are better than methods for dead code elimination. Hot paths use output-parameter style (zero allocation). Setup code can use allocating variants.

## Matrix Layout

**Decision: Column-major 4×4 matrices (universal agreement, matches WebGL/WebGPU)**

```
Memory layout:
[m0  m1  m2  m3    m4  m5  m6  m7    m8  m9  m10 m11   m12 m13 m14 m15]
 col0              col1              col2              col3 (translation)

Logical view:
┌──────────────────────┐
│ m0  m4  m8   m12 │  Row 0
│ m1  m5  m9   m13 │  Row 1
│ m2  m6  m10  m14 │  Row 2
│ m3  m7  m11  m15 │  Row 3
└──────────────────────┘

Translation: [m12, m13, m14]
```

## Quaternion Convention

**Decision: [x, y, z, w] order (glTF convention, universal agreement)**

All implementations use quaternions for rotations. `[x, y, z, w]` matches glTF binary format — no swizzle needed at import.

## Scratch / Pool System

**Decision: Module-level scratch variables + frame-scoped pool with reset**

- Sources: Module scratch from 6/9; frame pool from Hyena, Mantis, Lynx (3/9)

Module-level scratch (for internal math library use):
```typescript
const _tempVec3A = new Float32Array(3)
const _tempVec3B = new Float32Array(3)
const _tempMat4 = new Float32Array(16)
const _tempQuat = new Float32Array(4)
```

Frame-scoped pool (for user-facing hot paths):
```typescript
const tempPool = {
  vec3: Array.from({ length: 32 }, () => new Float32Array(3)),
  mat4: Array.from({ length: 8 }, () => new Float32Array(16)),
  index: 0,
  acquire: (type: 'vec3' | 'mat4') => { /* return next from pool */ },
  reset: () => { tempPool.index = 0 },  // called at frame start
}
```

Zero allocation in hot paths. Pool resets each frame — no deallocation needed.

## Custom Library (No Dependencies)

**Decision: Custom math library, no gl-matrix dependency**

- Sources: All 8 specified implementations use custom math

Reasons:
- Z-up coordinate system baked into all operations (lookAt, spherical, frustum)
- Control over allocation patterns (output parameter style)
- No unused operations bloating bundle
- Impedance mismatch with gl-matrix API conventions
- Typical size: 20-40 vector ops, 15-25 matrix ops — small enough to maintain

## Core Operations

### Vec3 (required)
```
create, set, copy, clone
add, subtract, scale, multiplyComponents
dot, cross
length, lengthSquared, distance, distanceSquared
normalize, negate
lerp
transformMat4, transformMat4Direction (ignore translation)
min, max (component-wise)
equals, approximatelyEquals
```

### Mat4 (required)
```
create, identity, copy
multiply, invert, transpose
fromTRS (compose from translation, rotation, scale)
decompose (extract translation, rotation, scale)
perspective, orthographic
lookAt (Z-up adapted)
getTranslation, getScaling, getRotation
determinant
```

### Quat (required)
```
create, identity, copy
multiply, normalize, invert, conjugate
fromEuler, toEuler
fromAxisAngle
fromMat4 (extract rotation from matrix)
slerp
rotateVec3
```

### AABB (required)
```
create, fromPoints
expandByPoint, union
intersects, contains, containsPoint
surfaceArea (for BVH SAH)
transform (by mat4)
center, size
```

### Frustum (required)
```
extractFromViewProjection (Gribb-Hartmann method)
intersectsAABB (p-vertex test)
containsPoint
```

### Ray (required)
```
create, set
at (get point at distance t)
intersectAABB (slab method)
intersectTriangle (Möller-Trumbore)
intersectSphere
transformMat4 (transform ray by matrix)
```

## Perspective Matrix (Z-up)

**Decision: Support both [0,1] (WebGPU) and [-1,1] (WebGL2) depth ranges**

- Sources: Mantis, Lynx (2/9)

```typescript
const mat4Perspective = (
  out: Mat4,
  fov: number,
  aspect: number,
  near: number,
  far: number,
  zeroToOne: boolean  // true for WebGPU, false for WebGL2
): Mat4 => { ... }
```

## lookAt (Z-up)

**Decision: Z-up aware lookAt function**

```typescript
const mat4LookAt = (out: Mat4, eye: Vec3, target: Vec3, up: Vec3 = VEC3_UP): Mat4 => {
  // up defaults to [0, 0, 1] (Z-up)
  // Handles degenerate case where forward ≈ up
}
```

## Spherical Coordinates (Z-up)

**Decision: Z-up adapted spherical-to-Cartesian for orbit controls**

```typescript
const sphericalToCartesian = (out: Vec3, distance: number, azimuth: number, elevation: number): Vec3 => {
  out[0] = distance * Math.cos(elevation) * Math.cos(azimuth)
  out[1] = distance * Math.cos(elevation) * Math.sin(azimuth)
  out[2] = distance * Math.sin(elevation)
  return out
}
```

## Constants

```typescript
const VEC3_ZERO: Vec3    = Float32Array.of(0, 0, 0)
const VEC3_ONE: Vec3     = Float32Array.of(1, 1, 1)
const VEC3_UP: Vec3      = Float32Array.of(0, 0, 1)   // Z-up!
const VEC3_RIGHT: Vec3   = Float32Array.of(1, 0, 0)
const VEC3_FORWARD: Vec3 = Float32Array.of(0, 1, 0)
const QUAT_IDENTITY: Quat = Float32Array.of(0, 0, 0, 1)
const MAT4_IDENTITY: Mat4 = Float32Array.of(1,0,0,0, 0,1,0,0, 0,0,1,0, 0,0,0,1)
const DEG2RAD = Math.PI / 180
const RAD2DEG = 180 / Math.PI
```

## GLTF Y-up Conversion

**Decision: Vertex data conversion baked at import time**

- Sources: Fennec, Caracal, Wren (3/9)

Convert positions, normals, and animation keyframes from Y-up to Z-up at load time. No runtime conversion overhead. No extra root rotation node.

## Testing

**Decision: Unit tests for all math functions**

- Sources: Bonobo, Fennec (2/9 mention explicitly; all should have them)
- Validate against known values and reference implementations (gl-matrix)
- Edge case testing: NaN, Infinity, zero vectors, degenerate matrices
- Property-based tests for inverse operations (e.g., `invert(invert(M)) ≈ M`)
