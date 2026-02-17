# MATH.md - Decisions

## Decision: Pure Functional API, Type Aliases, Module-Level Scratch, Custom Library

### Coordinate System: Z-Up Right-Handed

Universal agreement across all 9 implementations:

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
- Matches Blender and Unreal Engine
- Differs from Three.js (Y-up)
- glTF Y-up converted at import time

### Float32Array Backing

Universal agreement. All math types backed by `Float32Array`:
- Compatible with WebGL/WebGPU uniform uploads (zero conversion overhead)
- Better cache locality than object properties
- SIMD-friendly layout (future-proofing)

### Column-Major Matrices

Universal agreement: column-major 4x4 matrix layout matching WebGL/WebGPU conventions.

```
Memory: [m0 m1 m2 m3 | m4 m5 m6 m7 | m8 m9 m10 m11 | m12 m13 m14 m15]
         col0           col1           col2              col3 (translation)
```

Translation at indices [12, 13, 14].

### API Style: Pure Functional

**Chosen**: Standalone functions with output parameter (5/9: Bonobo, Hyena, Lynx, Mantis, Wren)
**Rejected**: OOP wrappers (3/9: Fennec, Caracal, Rabbit) - convenient but add object overhead and prevent tree-shaking

```typescript
// Hot path: allocation-free, output parameter
vec3Add(out, a, b)
mat4Multiply(out, a, b)
quatSlerp(out, a, b, t)

// No class methods, no `this` binding
// Maximum tree-shaking potential
```

The public scene graph API wraps these in property-style setters (see SCENE-GRAPH.md) for ergonomics, but the math library itself is purely functional.

### Type System: Type Aliases

**Chosen**: Type aliases over `Float32Array` (5/9: Hyena, Lynx, Mantis, Rabbit, Wren)
**Rejected**: Wrapper classes (3/9: Fennec, Caracal, Bonobo) - add per-object overhead and GC pressure

```typescript
type Vec2 = Float32Array  // length 2
type Vec3 = Float32Array  // length 3
type Vec4 = Float32Array  // length 4
type Quat = Float32Array  // length 4, [x, y, z, w]
type Mat4 = Float32Array  // length 16, column-major
```

Zero overhead at runtime. TypeScript still provides type safety at compile time.

### Quaternion Layout

`[x, y, z, w]` - glTF convention, used by all 9 implementations.

### Scratch System: Module-Level Scratch Variables

**Chosen**: Module-level pre-allocated scratch variables (6/9: Bonobo, Fennec, Caracal, Hyena, Rabbit, Wren)

```typescript
// Internal scratch - never exposed, never allocated in hot paths
const _tempVec3A = vec3Create()
const _tempVec3B = vec3Create()
const _tempMat4A = mat4Create()
const _tempMat4B = mat4Create()
const _tempQuat = quatCreate()
```

Simpler than frame-scoped pools (3/9: Hyena, Mantis, Lynx) and sufficient for the engine's needs. Frame-scoped pools add a reset step and require tracking pool capacity.

### Library Scope: Minimal, Custom

**Chosen**: Custom math library with only needed operations (8/9 use custom, 5/9 are minimal)
**Rejected**: External dependency (gl-matrix etc.) - impedance mismatch with Z-up coordinate system, brings unnecessary operations

### Core Operations

**Vec3** (essential set from all implementations):

```typescript
vec3Create(): Vec3
vec3Set(out, x, y, z): Vec3
vec3Copy(out, a): Vec3
vec3Add(out, a, b): Vec3
vec3Sub(out, a, b): Vec3
vec3Scale(out, a, s): Vec3
vec3Dot(a, b): number
vec3Cross(out, a, b): Vec3
vec3Length(a): number
vec3LengthSq(a): number
vec3Distance(a, b): number
vec3Normalize(out, a): Vec3
vec3Lerp(out, a, b, t): Vec3
vec3TransformMat4(out, a, m): Vec3       // Position (includes translation)
vec3TransformMat4Dir(out, a, m): Vec3    // Direction (ignores translation)
```

**Mat4** (essential set):

```typescript
mat4Create(): Mat4
mat4Identity(out): Mat4
mat4Copy(out, a): Mat4
mat4Multiply(out, a, b): Mat4
mat4Invert(out, a): Mat4 | null
mat4Transpose(out, a): Mat4
mat4FromTRS(out, t, r, s): Mat4          // Translation, Rotation (quat), Scale
mat4Perspective(out, fov, aspect, near, far, zeroToOne?: boolean): Mat4
mat4Ortho(out, left, right, bottom, top, near, far): Mat4
mat4LookAt(out, eye, target, up): Mat4   // Z-up aware
mat4GetTranslation(out, m): Vec3
mat4GetScaling(out, m): Vec3
mat4GetRotation(out, m): Quat
```

Depth range parameter on `mat4Perspective`: `zeroToOne` for WebGPU [0,1] vs WebGL2 [-1,1] (Mantis/Lynx approach).

**Quat** (essential set):

```typescript
quatCreate(): Quat
quatIdentity(out): Quat
quatMultiply(out, a, b): Quat
quatNormalize(out, a): Quat
quatInvert(out, a): Quat
quatSlerp(out, a, b, t): Quat
quatFromEuler(out, x, y, z): Quat       // ZYX order for Z-up
quatFromAxisAngle(out, axis, angle): Quat
```

**AABB** (stored as Float32Array(6) = [minX, minY, minZ, maxX, maxY, maxZ]):

```typescript
aabbCreate(): AABB
aabbFromPoints(out, positions): AABB
aabbExpandByPoint(out, point): AABB
aabbUnion(out, a, b): AABB
aabbIntersects(a, b): boolean
aabbSurfaceArea(a): number              // For BVH SAH
aabbTransformMat4(out, a, m): AABB
```

**Frustum** (stored as Float32Array(24) = 6 planes x 4 floats):

```typescript
frustumExtractFromViewProjection(out, vp): Frustum
frustumIntersectsAABB(frustum, aabb): boolean
```

**Ray**:

```typescript
rayCreate(origin, direction): Ray
rayAt(out, ray, t): Vec3
rayIntersectAABB(ray, aabb): number | null
rayIntersectTriangle(ray, v0, v1, v2): { t, u, v } | null
```

### Constants

```typescript
const VEC3_ZERO: Vec3    = [0, 0, 0]
const VEC3_ONE: Vec3     = [1, 1, 1]
const VEC3_UP: Vec3      = [0, 0, 1]     // Z-up!
const VEC3_RIGHT: Vec3   = [1, 0, 0]
const VEC3_FORWARD: Vec3 = [0, 1, 0]
const QUAT_IDENTITY: Quat = [0, 0, 0, 1]
```

These are frozen/readonly to prevent accidental mutation.

### Euler Angle Order: ZYX

**Chosen**: ZYX rotation order (Mantis approach) - yaw-pitch-roll for Z-up coordinate systems.

Euler angles are a convenience API. Internal storage is always quaternion.

### GLTF Y-up to Z-up Conversion

Handled in the asset loader (see ASSETS.md), not in the math library. The math library is purely Z-up; it has no concept of coordinate system conversion.

### Performance

All implementations agree:
- Float32Array is fast enough (SIMD not needed for current scope)
- Scalar JS math: ~1ms for 5000 matrix computations
- No allocation in hot paths is the critical optimization
- Cache-friendly SoA layout where applicable
