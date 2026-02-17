# Math Library

## Design Principles

1. **Zero allocation**: All operations write to output parameters, never create new objects.
2. **Float32Array backing**: All math types are views into `Float32Array`. Compatible with direct GPU upload.
3. **Functional API with optional OOP wrappers**: Core math is pure functions on typed arrays. Scene-graph-facing API wraps these in convenient classes.
4. **Z-up right-handed**: All defaults, constructors, and utilities assume Z-up.

## Core Types

### Vec3

```typescript
// A Vec3 is a Float32Array of length 3
type Vec3Data = Float32Array  // [x, y, z]

// Functional API — all ops take output parameter
const vec3 = {
  create: (): Vec3Data => new Float32Array(3),
  set: (out: Vec3Data, x: number, y: number, z: number): Vec3Data => {
    out[0] = x; out[1] = y; out[2] = z; return out
  },
  copy: (out: Vec3Data, a: Vec3Data): Vec3Data => {
    out[0] = a[0]; out[1] = a[1]; out[2] = a[2]; return out
  },
  add: (out: Vec3Data, a: Vec3Data, b: Vec3Data): Vec3Data => {
    out[0] = a[0] + b[0]; out[1] = a[1] + b[1]; out[2] = a[2] + b[2]; return out
  },
  subtract: (out: Vec3Data, a: Vec3Data, b: Vec3Data): Vec3Data => { /* ... */ },
  scale: (out: Vec3Data, a: Vec3Data, s: number): Vec3Data => { /* ... */ },
  dot: (a: Vec3Data, b: Vec3Data): number => a[0]*b[0] + a[1]*b[1] + a[2]*b[2],
  cross: (out: Vec3Data, a: Vec3Data, b: Vec3Data): Vec3Data => { /* ... */ },
  length: (a: Vec3Data): number => Math.sqrt(a[0]*a[0] + a[1]*a[1] + a[2]*a[2]),
  normalize: (out: Vec3Data, a: Vec3Data): Vec3Data => { /* ... */ },
  lerp: (out: Vec3Data, a: Vec3Data, b: Vec3Data, t: number): Vec3Data => { /* ... */ },
  transformMat4: (out: Vec3Data, a: Vec3Data, m: Mat4Data): Vec3Data => { /* ... */ },
  distance: (a: Vec3Data, b: Vec3Data): number => { /* ... */ },
  negate: (out: Vec3Data, a: Vec3Data): Vec3Data => { /* ... */ },
}

// Constants
const VEC3_ZERO: Vec3Data    = new Float32Array([0, 0, 0])
const VEC3_ONE: Vec3Data     = new Float32Array([1, 1, 1])
const VEC3_UP: Vec3Data      = new Float32Array([0, 0, 1])  // Z-up
const VEC3_FORWARD: Vec3Data = new Float32Array([0, 1, 0])  // +Y forward
const VEC3_RIGHT: Vec3Data   = new Float32Array([1, 0, 0])  // +X right
```

### Vec4

```typescript
type Vec4Data = Float32Array  // [x, y, z, w]

const vec4 = {
  create: (): Vec4Data => new Float32Array(4),
  set: (out: Vec4Data, x: number, y: number, z: number, w: number): Vec4Data => { /* ... */ },
  transformMat4: (out: Vec4Data, a: Vec4Data, m: Mat4Data): Vec4Data => { /* ... */ },
  // ... standard vector ops
}
```

### Mat4

```typescript
// Column-major 4×4 matrix (matches WebGL/WebGPU convention)
type Mat4Data = Float32Array  // 16 floats, column-major

const mat4 = {
  create: (): Mat4Data => {
    const out = new Float32Array(16)
    out[0] = out[5] = out[10] = out[15] = 1  // identity
    return out
  },
  identity: (out: Mat4Data): Mat4Data => { /* ... */ },
  multiply: (out: Mat4Data, a: Mat4Data, b: Mat4Data): Mat4Data => { /* ... */ },
  invert: (out: Mat4Data, a: Mat4Data): Mat4Data | null => { /* ... */ },
  transpose: (out: Mat4Data, a: Mat4Data): Mat4Data => { /* ... */ },
  determinant: (a: Mat4Data): number => { /* ... */ },

  // Transform composition
  compose: (out: Mat4Data, position: Vec3Data, rotation: QuatData, scale: Vec3Data): Mat4Data => { /* ... */ },
  decompose: (m: Mat4Data, position: Vec3Data, rotation: QuatData, scale: Vec3Data): void => { /* ... */ },

  // Projection (Z-up right-handed)
  perspective: (out: Mat4Data, fovY: number, aspect: number, near: number, far: number): Mat4Data => { /* ... */ },
  orthographic: (out: Mat4Data, left: number, right: number, bottom: number, top: number, near: number, far: number): Mat4Data => { /* ... */ },

  // View (Z-up)
  lookAt: (out: Mat4Data, eye: Vec3Data, target: Vec3Data, up?: Vec3Data): Mat4Data => {
    // Default up = [0, 0, 1] (Z-up)
    /* ... */
  },

  // Transform operations
  translate: (out: Mat4Data, a: Mat4Data, v: Vec3Data): Mat4Data => { /* ... */ },
  scale: (out: Mat4Data, a: Mat4Data, v: Vec3Data): Mat4Data => { /* ... */ },
  rotateX: (out: Mat4Data, a: Mat4Data, rad: number): Mat4Data => { /* ... */ },
  rotateY: (out: Mat4Data, a: Mat4Data, rad: number): Mat4Data => { /* ... */ },
  rotateZ: (out: Mat4Data, a: Mat4Data, rad: number): Mat4Data => { /* ... */ },

  // Extract
  getTranslation: (out: Vec3Data, m: Mat4Data): Vec3Data => { /* ... */ },
  getScaling: (out: Vec3Data, m: Mat4Data): Vec3Data => { /* ... */ },
  getRotation: (out: QuatData, m: Mat4Data): QuatData => { /* ... */ },
}
```

### Quat

```typescript
type QuatData = Float32Array  // [x, y, z, w]

const quat = {
  create: (): QuatData => {
    const out = new Float32Array(4)
    out[3] = 1  // identity quaternion
    return out
  },
  identity: (out: QuatData): QuatData => { out[0] = out[1] = out[2] = 0; out[3] = 1; return out },
  multiply: (out: QuatData, a: QuatData, b: QuatData): QuatData => { /* ... */ },
  normalize: (out: QuatData, a: QuatData): QuatData => { /* ... */ },
  slerp: (out: QuatData, a: QuatData, b: QuatData, t: number): QuatData => { /* ... */ },
  invert: (out: QuatData, a: QuatData): QuatData => { /* ... */ },
  conjugate: (out: QuatData, a: QuatData): QuatData => { /* ... */ },

  // Conversions
  fromEuler: (out: QuatData, x: number, y: number, z: number): QuatData => { /* ... */ },
  fromAxisAngle: (out: QuatData, axis: Vec3Data, angle: number): QuatData => { /* ... */ },
  fromMat4: (out: QuatData, m: Mat4Data): QuatData => { /* ... */ },
  toEuler: (out: Vec3Data, q: QuatData): Vec3Data => { /* ... */ },

  // Rotate a vector by this quaternion
  rotateVec3: (out: Vec3Data, q: QuatData, v: Vec3Data): Vec3Data => { /* ... */ },
}
```

## OOP Wrappers

For the scene graph API, thin OOP wrappers provide a friendlier interface. These wrap a `Float32Array` view and expose named accessors:

```typescript
class Vec3 {
  readonly data: Vec3Data

  constructor(data?: Vec3Data) {
    this.data = data ?? new Float32Array(3)
  }

  get x(): number { return this.data[0] }
  set x(v: number) { this.data[0] = v; this._onChanged?.() }
  get y(): number { return this.data[1] }
  set y(v: number) { this.data[1] = v; this._onChanged?.() }
  get z(): number { return this.data[2] }
  set z(v: number) { this.data[2] = v; this._onChanged?.() }

  set(x: number, y: number, z: number): this {
    this.data[0] = x; this.data[1] = y; this.data[2] = z
    this._onChanged?.()
    return this
  }

  // Optional callback for dirty flag propagation (used by Object3D)
  _onChanged?: () => void
}
```

When used in `Object3D`, the `Vec3` wrapper's `_onChanged` is wired to the dirty flag system:

```typescript
class Object3D {
  readonly position = new Vec3(positions.subarray(idx * 3, idx * 3 + 3))

  constructor() {
    this.position._onChanged = () => { this._markDirty() }
  }
}
```

This way, `mesh.position.x = 5` directly writes into the SoA array AND triggers the dirty flag.

## Scratch Pool

A global pool of temporary math objects for zero-allocation operations in hot paths:

```typescript
const pool = {
  _v3: Array.from({ length: 32 }, () => new Float32Array(3)),
  _v4: Array.from({ length: 16 }, () => new Float32Array(4)),
  _m4: Array.from({ length: 16 }, () => new Float32Array(16)),
  _qt: Array.from({ length: 8 },  () => new Float32Array(4)),
  _i: { v3: 0, v4: 0, m4: 0, qt: 0 },

  v3: (): Vec3Data => pool._v3[pool._i.v3++],
  v4: (): Vec4Data => pool._v4[pool._i.v4++],
  m4: (): Mat4Data => pool._m4[pool._i.m4++],
  qt: (): QuatData => pool._qt[pool._i.qt++],

  reset: () => { pool._i.v3 = pool._i.v4 = pool._i.m4 = pool._i.qt = 0 },
}
```

Called at the start of each frame:
```typescript
// In Engine.update():
pool.reset()
```

If a pool runs out (more than 32 Vec3 temporaries in one frame), it wraps around. This is a bug indicator during development and can be caught with a debug assertion.

## Geometry Types

### AABB

```typescript
interface AABB {
  min: Vec3Data  // [minX, minY, minZ]
  max: Vec3Data  // [maxX, maxY, maxZ]
}

const aabb = {
  create: (): AABB => ({
    min: new Float32Array([Infinity, Infinity, Infinity]),
    max: new Float32Array([-Infinity, -Infinity, -Infinity]),
  }),
  expandByPoint: (box: AABB, point: Vec3Data): void => { /* ... */ },
  expandByAABB: (box: AABB, other: AABB): void => { /* ... */ },
  intersectsAABB: (a: AABB, b: AABB): boolean => { /* ... */ },
  containsPoint: (box: AABB, point: Vec3Data): boolean => { /* ... */ },
  surfaceArea: (box: AABB): number => { /* ... */ },
  center: (out: Vec3Data, box: AABB): Vec3Data => { /* ... */ },
  size: (out: Vec3Data, box: AABB): Vec3Data => { /* ... */ },
  transformMat4: (out: AABB, box: AABB, m: Mat4Data): AABB => { /* ... */ },
}
```

### Frustum

```typescript
// 6 planes, each [nx, ny, nz, d] where nx*x + ny*y + nz*z + d = 0
type FrustumData = Float32Array  // 24 floats (6 planes × 4)

const frustum = {
  extractFromViewProjection: (out: FrustumData, vp: Mat4Data): FrustumData => { /* ... */ },
  intersectsAABB: (f: FrustumData, box: AABB): 'inside' | 'outside' | 'intersecting' => { /* ... */ },
  containsPoint: (f: FrustumData, point: Vec3Data): boolean => { /* ... */ },
}
```

### Ray

```typescript
interface Ray {
  origin: Vec3Data
  direction: Vec3Data  // normalized
}

const ray = {
  create: (origin: Vec3Data, direction: Vec3Data): Ray => ({
    origin: new Float32Array(origin),
    direction: new Float32Array(direction),
  }),
  at: (out: Vec3Data, r: Ray, t: number): Vec3Data => {
    out[0] = r.origin[0] + r.direction[0] * t
    out[1] = r.origin[1] + r.direction[1] * t
    out[2] = r.origin[2] + r.direction[2] * t
    return out
  },
  intersectsAABB: (r: Ray, box: AABB): number | null => { /* slab test, returns t or null */ },
  intersectsTriangle: (r: Ray, v0: Vec3Data, v1: Vec3Data, v2: Vec3Data): RayHit | null => { /* Moller-Trumbore */ },
  transformMat4: (out: Ray, r: Ray, m: Mat4Data): Ray => { /* transform origin and direction */ },
}
```

## Column-Major Convention

Matrices are stored in column-major order (matching WebGL and WebGPU expectations):

```
Mat4 indices:
[m0  m4  m8   m12]     [Xx  Yx  Zx  Tx]
[m1  m5  m9   m13]  =  [Xy  Yy  Zy  Ty]
[m2  m6  m10  m14]     [Xz  Yz  Zz  Tz]
[m3  m7  m11  m15]     [0   0   0   1 ]

where X = right, Y = forward, Z = up (Z-up convention)
m12, m13, m14 = translation (Tx, Ty, Tz)
```

This means `mat4.data[12]` is the X translation, `mat4.data[13]` is Y, `mat4.data[14]` is Z. Column-major storage allows the `Float32Array` to be uploaded directly to the GPU without transposition.
