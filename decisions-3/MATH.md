# MATH.md - Math Library

## Coordinate System

**Z-up, right-handed** (universal agreement):

```
       Z (up)
       |
       |
       +------ X (right)
      /
     /
    Y (forward)
```

- X x Y = Z (right-hand rule)
- Matches Blender, Unreal Engine, and many CAD tools
- Differs from Three.js/glTF (Y-up)

## Float32Array Backing

**Decision: All math types backed by Float32Array** (universal agreement)

```typescript
type Vec3 = Float32Array  // length 3
type Vec4 = Float32Array  // length 4
type Quat = Float32Array  // length 4, order [x, y, z, w]
type Mat4 = Float32Array  // length 16, column-major
```

Benefits:
- Direct upload to WebGL/WebGPU uniforms (no conversion)
- Better cache locality than object properties
- SIMD-friendly (future potential)
- Compatible with `bufferSubData` / `writeBuffer`

## API Style

**Decision: Pure functional with type aliases** (Hyena, Lynx, Mantis, Wren - 5/9 prefer functional, 5/9 prefer type aliases)

```typescript
// All functions take output parameter first (allocation-free)
vec3Add(out, a, b)
vec3Scale(out, v, scalar)
vec3Normalize(out, v)
mat4Multiply(out, a, b)
mat4FromTRS(out, translation, rotation, scale)
quatSlerp(out, a, b, t)
```

### Why Pure Functional Over OOP Wrappers

- **Maximum tree-shaking**: Unused functions are eliminated. OOP wrappers bundle all methods.
- **Zero overhead**: No `this` binding, no class instantiation, no prototype chain.
- **Explicit data flow**: Output parameter makes it clear where results go.
- **No hidden state**: Functions are pure (output depends only on inputs).

OOP wrappers (Fennec, Caracal, Rabbit) offer better IDE autocomplete (`vec.add(other)`) but at the cost of bundle size and allocation patterns. For a performance-focused engine, the functional approach is the right default. Users can create their own wrappers if desired.

### Why Type Aliases Over Classes

- `Vec3` is just a `Float32Array` - no wrapper overhead
- Can be used directly in uniform buffer uploads
- No GC pressure from wrapper objects
- Debugging shows raw Float32Array values in devtools (immediately useful)

## Column-Major Matrices

**Decision: Column-major 4x4 matrices** (universal agreement, matches WebGL/WebGPU)

```
Memory layout: [m0, m1, m2, m3, m4, m5, m6, m7, m8, m9, m10, m11, m12, m13, m14, m15]
                col0          col1          col2          col3 (translation)

Logical view:
| m0  m4  m8   m12 |
| m1  m5  m9   m13 |
| m2  m6  m10  m14 |
| m3  m7  m11  m15 |

Translation = [m12, m13, m14]
```

## Quaternions

**Decision: [x, y, z, w] order** (universal agreement, matches glTF convention)

```typescript
type Quat = Float32Array  // [x, y, z, w]

// Identity quaternion
const identity: Quat = new Float32Array([0, 0, 0, 1])
```

Preferred over Euler angles for all rotations:
- No gimbal lock
- Smooth interpolation via slerp
- More compact than matrices (4 floats vs 16)
- Stable concatenation

## Scratch Variables

**Decision: Module-level scratch variables + frame-scoped pool** (Hyena, Lynx, Mantis approach - both strategies)

### Module-Level Scratch (For Internal Use)

```typescript
// Pre-allocated per-module, never exposed to users
const _tempVec3A = vec3Create()
const _tempVec3B = vec3Create()
const _tempMat4A = mat4Create()
const _tempQuat = quatCreate()
```

Used for intermediate calculations within engine functions. Never returned to callers (would alias).

### Frame-Scoped Pool (For User Callbacks)

```typescript
const scratchPool = {
  vec3s: Array.from({ length: 32 }, vec3Create),
  mat4s: Array.from({ length: 8 }, mat4Create),
  index: 0,

  vec3(): Vec3 { return this.vec3s[this.index++] },
  mat4(): Mat4 { return this.mat4s[this.index++] },
  reset() { this.index = 0 },
}

// Reset at frame start
scratchPool.reset()
```

Users can acquire temporary vectors/matrices from the pool during `useFrame` callbacks. The pool resets at frame start, providing zero-allocation temporary math for user code.

## Essential Operations

### Vec3

```typescript
vec3Create(): Vec3
vec3Set(out, x, y, z): Vec3
vec3Copy(out, src): Vec3
vec3Add(out, a, b): Vec3
vec3Sub(out, a, b): Vec3
vec3Scale(out, v, scalar): Vec3
vec3Multiply(out, a, b): Vec3           // Component-wise
vec3Dot(a, b): number
vec3Cross(out, a, b): Vec3
vec3Length(v): number
vec3LengthSquared(v): number
vec3Distance(a, b): number
vec3Normalize(out, v): Vec3
vec3Lerp(out, a, b, t): Vec3
vec3TransformMat4(out, v, m): Vec3      // Position (w=1, includes translation)
vec3TransformMat4Dir(out, v, m): Vec3   // Direction (w=0, ignores translation)
vec3Min(out, a, b): Vec3
vec3Max(out, a, b): Vec3
```

### Mat4

```typescript
mat4Create(): Mat4
mat4Identity(out): Mat4
mat4Copy(out, src): Mat4
mat4Multiply(out, a, b): Mat4
mat4Invert(out, m): Mat4
mat4Transpose(out, m): Mat4
mat4FromTRS(out, t, r, s): Mat4         // Compose from translation, rotation, scale
mat4Perspective(out, fov, aspect, near, far): Mat4
mat4Ortho(out, left, right, bottom, top, near, far): Mat4
mat4LookAt(out, eye, target, up): Mat4  // Z-up adapted
mat4GetTranslation(out, m): Vec3
mat4GetScaling(out, m): Vec3
mat4GetRotation(out, m): Quat
```

### Perspective Matrix Depth Range

**Decision: Support both [0,1] and [-1,1] depth ranges** (Mantis, Lynx approach)

```typescript
mat4Perspective(out, fov, aspect, near, far, depthZeroToOne = true)
```

WebGPU uses [0,1] clip depth. WebGL2 uses [-1,1]. The perspective matrix adjusts accordingly. Default is `true` (WebGPU convention) since WebGPU is the primary target.

### Quat

```typescript
quatCreate(): Quat
quatIdentity(out): Quat
quatMultiply(out, a, b): Quat
quatNormalize(out, q): Quat
quatInvert(out, q): Quat
quatConjugate(out, q): Quat
quatSlerp(out, a, b, t): Quat
quatFromEuler(out, x, y, z): Quat       // Radians, Z-up convention
quatFromAxisAngle(out, axis, angle): Quat
quatToEuler(out, q): Vec3               // Returns [x, y, z] radians
```

### AABB

```typescript
aabbCreate(): Float32Array              // [minX, minY, minZ, maxX, maxY, maxZ]
aabbFromPoints(out, positions, count): AABB
aabbExpandByPoint(out, aabb, point): AABB
aabbUnion(out, a, b): AABB
aabbIntersects(a, b): boolean
aabbContainsPoint(aabb, point): boolean
aabbSurfaceArea(aabb): number           // For BVH SAH
aabbTransform(out, aabb, mat4): AABB    // Conservative transformed AABB
```

### Frustum

```typescript
frustumCreate(): Float32Array           // 6 planes * 4 floats = 24 floats
frustumExtractFromVP(out, viewProjection): Frustum  // Gribb-Hartmann
frustumContainsAABB(frustum, aabb): boolean          // P-vertex test
frustumContainsPoint(frustum, point): boolean
```

### Ray

```typescript
rayCreate(origin, direction): { origin: Vec3, direction: Vec3 }
rayAt(out, ray, t): Vec3                // Point at distance t
rayIntersectAABB(ray, aabb): number     // Returns t or -1 (slab method)
rayIntersectTriangle(ray, v0, v1, v2): { t, u, v } | null  // Moller-Trumbore
rayTransformMat4(out, ray, mat4): Ray   // Transform ray into object space
```

## Constants

```typescript
const VEC3_ZERO:    Vec3 = [0, 0, 0]
const VEC3_ONE:     Vec3 = [1, 1, 1]
const VEC3_UP:      Vec3 = [0, 0, 1]  // Z-up
const VEC3_RIGHT:   Vec3 = [1, 0, 0]
const VEC3_FORWARD: Vec3 = [0, 1, 0]
const QUAT_IDENTITY: Quat = [0, 0, 0, 1]
```

Constants are frozen `Float32Array` instances. Never mutate them.

## Custom Library

**Decision: Custom math library, no external dependencies** (8/9 implementations - universal)

Reasons:
- Z-up coordinate system baked into all operations
- Zero-allocation API (gl-matrix allocates in some convenience functions)
- Minimal surface area (only the operations the engine needs)
- No impedance mismatch with external library conventions
- Tiny bundle size (~5-8KB minified)

## Testing

Unit tests for all math operations validated against:
- Known analytical results (identity, orthogonality, inverse)
- Edge cases (zero vectors, degenerate quaternions, near-singular matrices)
- Cross-reference with gl-matrix for numerical accuracy
