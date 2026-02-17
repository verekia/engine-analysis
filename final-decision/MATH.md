# MATH — Final Decision

## Coordinate System

**Decision: Z-up, right-handed** (universal agreement)

- X = right
- Y = forward
- Z = up

Matches Blender convention. glTF Y-up assets are converted at import time.

## Data Backing

**Decision: All math types backed by Float32Array** (universal agreement)

```typescript
type Vec3 = Float32Array  // length 3
type Vec4 = Float32Array  // length 4
type Quat = Float32Array  // length 4
type Mat4 = Float32Array  // length 16
type AABB = Float32Array  // length 6 (minX, minY, minZ, maxX, maxY, maxZ)
```

Type aliases over `Float32Array` rather than classes. This enables:
- Direct GPU upload (no copying)
- Pooling and reuse without allocation
- Sub-array views into contiguous buffers

## Matrix Convention

**Decision: Column-major 4×4 matrices** (universal agreement, matches WebGL/WebGPU)

Memory layout: `[m00, m10, m20, m30, m01, m11, m21, m31, m02, m12, m22, m32, m03, m13, m23, m33]`

Column-major is the native format for both WebGL (`uniformMatrix4fv`) and WebGPU (WGSL `mat4x4<f32>`). No transpose needed.

## Quaternion Convention

**Decision: [x, y, z, w] order** (universal agreement, matches glTF)

```typescript
const QUAT_IDENTITY: Quat = new Float32Array([0, 0, 0, 1])
```

## API Style

**Decision: Pure functional API with output parameter first** (universal agreement)

```typescript
// Output parameter first, inputs follow
const vec3Add = (out: Vec3, a: Vec3, b: Vec3): Vec3 => {
  out[0] = a[0] + b[0]
  out[1] = a[1] + b[1]
  out[2] = a[2] + b[2]
  return out
}

const mat4Multiply = (out: Mat4, a: Mat4, b: Mat4): Mat4 => { /* ... */ }
const quatSlerp = (out: Quat, a: Quat, b: Quat, t: number): Quat => { /* ... */ }
```

All functions return the output parameter for chaining. No allocations inside math functions — the caller provides the output buffer.

## Scratch Variables

**Decision: Module-level scratch variables + frame-scoped pool** (universal agreement)

```typescript
// Module-level: reused by functions in this module only
const _tempVec3 = vec3Create()
const _tempMat4 = mat4Create()
const _tempQuat = quatCreate()

// Frame-scoped: reset at the end of each frame
const scratchPool = {
  vec3s: new Float32Array(64 * 3),  // 64 Vec3 slots
  index: 0,
  nextVec3(): Vec3 {
    const offset = this.index * 3
    this.index++
    return this.vec3s.subarray(offset, offset + 3)
  },
  reset() { this.index = 0 },
}
```

Module-level scratches for internal use. Frame-scoped pool for temporary values in user callbacks.

## Core Operations

### Vec3

```
vec3Create, vec3Set, vec3Copy
vec3Add, vec3Sub, vec3Scale, vec3Negate
vec3Dot, vec3Cross, vec3Length, vec3Normalize
vec3Lerp, vec3Min, vec3Max
vec3TransformMat4, vec3TransformQuat
vec3Distance, vec3DistanceSq
```

### Mat4

```
mat4Create, mat4Identity, mat4Copy
mat4Multiply, mat4Invert, mat4Transpose
mat4FromTranslation, mat4FromRotation, mat4FromScale
mat4Compose (TRS), mat4Decompose
mat4Perspective, mat4Ortho
mat4LookAt (Z-up aware)
mat4GetTranslation, mat4GetRotation, mat4GetScale
```

### Quat

```
quatCreate, quatIdentity, quatCopy
quatMultiply, quatInvert, quatConjugate, quatNormalize
quatFromEuler (ZYX order), quatToEuler
quatFromAxisAngle, quatSlerp
quatLookRotation (Z-up aware)
```

### AABB

```
aabbCreate, aabbFromPoints, aabbExpandByPoint
aabbUnion, aabbIntersects
aabbTransform (by Mat4)
aabbCenter, aabbSize
```

### Frustum

```
frustumFromViewProjection (Gribb-Hartmann extraction)
frustumContainsAABB (p-vertex/n-vertex test)
frustumContainsPoint
```

### Ray

```
rayCreate, raySet
rayAt (point at distance t)
rayIntersectsAABB (slab method)
rayIntersectsTriangle (Möller-Trumbore)
rayFromCamera (unproject screen point)
```

## Depth Range

**Decision: [0, 1] depth range for WebGPU, [-1, 1] for WebGL2** (universal agreement)

The perspective projection matrix adjusts based on the active backend:

```typescript
const mat4Perspective = (out: Mat4, fov: number, aspect: number, near: number, far: number, depthRange: 'zero-to-one' | 'negative-one-to-one'): Mat4 => {
  // ... adjust Z mapping based on depth range
}
```

WebGPU uses `[0, 1]` natively. WebGL2 uses `[-1, 1]` natively.

## Euler Angles

**Decision: ZYX rotation order** (universal agreement)

```typescript
const quatFromEuler = (out: Quat, x: number, y: number, z: number): Quat => {
  // ZYX intrinsic rotation: rotate around Z, then Y, then X
}
```

ZYX is the most common game engine convention and avoids gimbal lock in typical camera orientations.

## Constants

```typescript
const VEC3_ZERO:    Vec3 = Object.freeze(new Float32Array([0, 0, 0]))
const VEC3_ONE:     Vec3 = Object.freeze(new Float32Array([1, 1, 1]))
const VEC3_UP:      Vec3 = Object.freeze(new Float32Array([0, 0, 1]))    // Z-up
const VEC3_FORWARD: Vec3 = Object.freeze(new Float32Array([0, 1, 0]))    // Y-forward
const VEC3_RIGHT:   Vec3 = Object.freeze(new Float32Array([1, 0, 0]))    // X-right
const QUAT_IDENTITY: Quat = Object.freeze(new Float32Array([0, 0, 0, 1]))
const MAT4_IDENTITY: Mat4 = Object.freeze(new Float32Array([1,0,0,0, 0,1,0,0, 0,0,1,0, 0,0,0,1]))
```

Constants are frozen to prevent accidental mutation.

## Z-Up Aware Functions

`mat4LookAt` and spherical coordinate conversions are implemented with Z-up in mind. No coordinate system conversion happens in the math library — the Z-up convention is baked into all directional assumptions.

## No External Dependencies

**Decision: Custom math library with no dependencies** (universal agreement)

No gl-matrix or similar. The math library is small (~5-8KB minified), purpose-built for Z-up, and avoids the overhead of a general-purpose library.

## Testing

Unit tests for all math functions with known-good values. Special attention to:
- Quaternion-matrix round-trip consistency
- Z-up lookAt correctness
- Perspective projection depth mapping
- AABB transform correctness
- Frustum culling edge cases
