# Math

## Coordinate System

- **Z-up, right-handed** (Blender convention, same as VoidCore)
- **Column-major mat4**
- **Euler ZXY rotation order**

### Axes

```
+X: Right
+Y: Forward
+Z: Up
```

### Handedness

Right-handed coordinate system:
- Cross product: X × Y = Z
- Rotation: Counter-clockwise is positive

## Matrix Layout

Column-major matrix layout (same as OpenGL, WebGPU):

```
mat4 = [
  m0, m1, m2,  m3,   // column 0
  m4, m5, m6,  m7,   // column 1
  m8, m9, m10, m11,  // column 2
  m12,m13,m14, m15   // column 3
]

Translation is in [m12, m13, m14]
```

## Typed Array Math

All math operates directly on `Float32Array` sections — no wrapper classes.

### Core Math Functions

All functions work on typed arrays by index:

```ts
mat4_from_trs(out: Float32Array, index: number, pos, rot, scale)
mat4_multiply(out: Float32Array, a: Float32Array, b: Float32Array)
mat4_perspective(out: Float32Array, fov, aspect, near, far)
mat4_lookAt(out: Float32Array, eye, target, up)
vec3_transformMat4(out: Float32Array, v: Float32Array, m: Float32Array)
extractFrustumPlanes(vp: Float32Array, out: Float32Array)
sphereInFrustum(spheres: Float32Array, index: number, planes: Float32Array): boolean
```

### mat4_from_trs

Compose TRS (Translation, Rotation, Scale) into a 4×4 matrix:

```ts
function mat4_from_trs(
  out: Float32Array,
  index: number,
  positions: Float32Array,
  rotations: Float32Array,
  scales: Float32Array
) {
  const pi = index * 3
  const ri = index * 3
  const si = index * 3
  const mi = index * 16

  // Read TRS
  const tx = positions[pi]
  const ty = positions[pi + 1]
  const tz = positions[pi + 2]

  const rx = rotations[ri]
  const ry = rotations[ri + 1]
  const rz = rotations[ri + 2]

  const sx = scales[si]
  const sy = scales[si + 1]
  const sz = scales[si + 2]

  // Compute rotation matrix from Euler ZXY
  const cx = Math.cos(rx), sx = Math.sin(rx)
  const cy = Math.cos(ry), sy = Math.sin(ry)
  const cz = Math.cos(rz), sz = Math.sin(rz)

  // Compose into out[mi..mi+15] (column-major)
  // ... (implementation details)
}
```

### mat4_multiply

Multiply two 4×4 matrices:

```ts
function mat4_multiply(
  out: Float32Array,
  a: Float32Array,
  b: Float32Array
) {
  // Standard matrix multiplication (column-major)
  // ... (implementation)
}
```

### mat4_perspective

Create perspective projection matrix:

```ts
function mat4_perspective(
  out: Float32Array,
  fov: number,      // Field of view (radians)
  aspect: number,   // Aspect ratio (width/height)
  near: number,
  far: number
) {
  const f = 1.0 / Math.tan(fov / 2)
  const nf = 1.0 / (near - far)

  out[0] = f / aspect
  out[5] = f
  out[10] = (far + near) * nf
  out[11] = -1
  out[14] = 2 * far * near * nf
  // ... (zero other entries)
}
```

### mat4_lookAt

Create view matrix from eye, target, up:

```ts
function mat4_lookAt(
  out: Float32Array,
  eye: [number, number, number],
  target: [number, number, number],
  up: [number, number, number]
) {
  // Compute forward, right, up vectors
  // Construct view matrix
  // ... (implementation)
}
```

## Frustum Culling

### extractFrustumPlanes

Extract 6 frustum planes from view-projection matrix:

```ts
function extractFrustumPlanes(
  vp: Float32Array,    // 4×4 view-projection matrix
  out: Float32Array    // 6 planes × 4 floats (nx, ny, nz, d)
) {
  // Standard Gribb/Hartmann method
  // Each plane: ax + by + cz + d = 0
  // Left:   vp[3] + vp[0]
  // Right:  vp[3] - vp[0]
  // Bottom: vp[3] + vp[1]
  // Top:    vp[3] - vp[1]
  // Near:   vp[3] + vp[2]
  // Far:    vp[3] - vp[2]
  // ... (implementation)
}
```

### sphereInFrustum

Test if a bounding sphere is inside the frustum:

```ts
function sphereInFrustum(
  spheres: Float32Array,   // [cx, cy, cz, radius, ...]
  index: number,
  planes: Float32Array     // 6 planes × 4 floats
): boolean {
  const si = index * 4
  const cx = spheres[si]
  const cy = spheres[si + 1]
  const cz = spheres[si + 2]
  const radius = spheres[si + 3]

  // Test against all 6 planes
  for (let i = 0; i < 6; i++) {
    const pi = i * 4
    const nx = planes[pi]
    const ny = planes[pi + 1]
    const nz = planes[pi + 2]
    const d = planes[pi + 3]

    const dist = nx * cx + ny * cy + nz * cz + d
    if (dist < -radius) return false  // Outside this plane
  }

  return true  // Inside or intersecting frustum
}
```

## Scratch Variables

Module-level scratch arrays for zero-allocation math:

```ts
// Module-level scratch — never allocate in the render loop
const _mat4A = new Float32Array(16)
const _mat4B = new Float32Array(16)
const _vec3A = new Float32Array(3)
const _vec3B = new Float32Array(3)
const _vec4A = new Float32Array(4)
const _frustumPlanes = new Float32Array(24) // 6 planes × 4 floats
```

Use these for intermediate calculations, never allocate new arrays in hot paths.

## Vector Math

Additional vector utilities:

```ts
vec3_add(out: Float32Array, a: Float32Array, b: Float32Array)
vec3_sub(out: Float32Array, a: Float32Array, b: Float32Array)
vec3_scale(out: Float32Array, v: Float32Array, s: number)
vec3_dot(a: Float32Array, b: Float32Array): number
vec3_cross(out: Float32Array, a: Float32Array, b: Float32Array)
vec3_normalize(out: Float32Array, v: Float32Array)
vec3_length(v: Float32Array): number
vec3_distance(a: Float32Array, b: Float32Array): number
```

## Quaternion Math (Future)

For v2, add quaternion support for rotation:

```ts
quat_from_euler(out: Float32Array, x: number, y: number, z: number)
quat_multiply(out: Float32Array, a: Float32Array, b: Float32Array)
quat_slerp(out: Float32Array, a: Float32Array, b: Float32Array, t: number)
mat4_from_quat(out: Float32Array, q: Float32Array)
```

Quaternions avoid gimbal lock and are better for interpolation.

## Performance Notes

- All math functions operate on typed arrays by index (no allocation)
- SIMD could be added later for vec/mat operations (via WASM or native SIMD)
- For now, scalar JS is fast enough (matrix computation is <1ms for 5000 entities)

## Testing

Unit tests for all math functions:

- Validate against known matrices (identity, translation, rotation, scale)
- Compare with reference implementations (gl-matrix, glMatrix)
- Ensure no NaN/Inf edge cases
