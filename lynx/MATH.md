# Math Library

## Design Principles

- **Z-up, right-handed** coordinate system throughout
- **Column-major** 4×4 matrices (same layout as WebGL/WebGPU, column-by-column in memory)
- **Float32Array** backed — every type is a view into typed arrays for zero-copy GPU upload
- **No classes** — plain Float32Array with standalone functions for maximum performance and tree-shaking
- **No allocations in hot paths** — all functions take an `out` parameter for the result
- **Minimal** — only what the engine actually needs, no unused operations

## Coordinate System

```
       Z (up)
       │
       │
       │
       │
       ├──────── Y (forward in many setups)
      /
     /
    X (right)

Right-handed: X × Y = Z
```

- **Up**: +Z
- **Forward** (default camera): -Y or configurable
- **Right**: +X
- Matches Blender and Unreal Engine conventions

## Vec3 — 3-Component Vector

Stored as `Float32Array(3)`.

```typescript
// Creation
const vec3Create = (x = 0, y = 0, z = 0): Float32Array => {
    const out = new Float32Array(3)
    out[0] = x; out[1] = y; out[2] = z
    return out
}

const vec3Set = (out: Float32Array, x: number, y: number, z: number): Float32Array => {
    out[0] = x; out[1] = y; out[2] = z
    return out
}

const vec3Copy = (out: Float32Array, a: Float32Array): Float32Array => {
    out[0] = a[0]; out[1] = a[1]; out[2] = a[2]
    return out
}

// Arithmetic
const vec3Add = (out: Float32Array, a: Float32Array, b: Float32Array): Float32Array => {
    out[0] = a[0] + b[0]; out[1] = a[1] + b[1]; out[2] = a[2] + b[2]
    return out
}

const vec3Sub = (out: Float32Array, a: Float32Array, b: Float32Array): Float32Array => {
    out[0] = a[0] - b[0]; out[1] = a[1] - b[1]; out[2] = a[2] - b[2]
    return out
}

const vec3Scale = (out: Float32Array, a: Float32Array, s: number): Float32Array => {
    out[0] = a[0] * s; out[1] = a[1] * s; out[2] = a[2] * s
    return out
}

const vec3ScaleAndAdd = (out: Float32Array, a: Float32Array, b: Float32Array, s: number): Float32Array => {
    out[0] = a[0] + b[0] * s; out[1] = a[1] + b[1] * s; out[2] = a[2] + b[2] * s
    return out
}

const vec3Multiply = (out: Float32Array, a: Float32Array, b: Float32Array): Float32Array => {
    out[0] = a[0] * b[0]; out[1] = a[1] * b[1]; out[2] = a[2] * b[2]
    return out
}

// Products
const vec3Dot = (a: Float32Array, b: Float32Array): number =>
    a[0] * b[0] + a[1] * b[1] + a[2] * b[2]

const vec3Cross = (out: Float32Array, a: Float32Array, b: Float32Array): Float32Array => {
    // Right-handed cross product
    out[0] = a[1] * b[2] - a[2] * b[1]
    out[1] = a[2] * b[0] - a[0] * b[2]
    out[2] = a[0] * b[1] - a[1] * b[0]
    return out
}

// Length
const vec3Length = (a: Float32Array): number =>
    Math.sqrt(a[0] * a[0] + a[1] * a[1] + a[2] * a[2])

const vec3LengthSq = (a: Float32Array): number =>
    a[0] * a[0] + a[1] * a[1] + a[2] * a[2]

const vec3Distance = (a: Float32Array, b: Float32Array): number => {
    const dx = a[0] - b[0], dy = a[1] - b[1], dz = a[2] - b[2]
    return Math.sqrt(dx * dx + dy * dy + dz * dz)
}

const vec3Normalize = (out: Float32Array, a: Float32Array): Float32Array => {
    const len = vec3Length(a)
    if (len > 1e-8) {
        const inv = 1 / len
        out[0] = a[0] * inv; out[1] = a[1] * inv; out[2] = a[2] * inv
    }
    return out
}

const vec3Lerp = (out: Float32Array, a: Float32Array, b: Float32Array, t: number): Float32Array => {
    out[0] = a[0] + (b[0] - a[0]) * t
    out[1] = a[1] + (b[1] - a[1]) * t
    out[2] = a[2] + (b[2] - a[2]) * t
    return out
}

// Transform by 4×4 matrix (position — applies translation)
const vec3TransformMat4 = (out: Float32Array, a: Float32Array, m: Float32Array): Float32Array => {
    const x = a[0], y = a[1], z = a[2]
    out[0] = m[0] * x + m[4] * y + m[8]  * z + m[12]
    out[1] = m[1] * x + m[5] * y + m[9]  * z + m[13]
    out[2] = m[2] * x + m[6] * y + m[10] * z + m[14]
    return out
}

// Transform by 4×4 matrix (direction — ignores translation)
const vec3TransformMat4Dir = (out: Float32Array, a: Float32Array, m: Float32Array): Float32Array => {
    const x = a[0], y = a[1], z = a[2]
    out[0] = m[0] * x + m[4] * y + m[8]  * z
    out[1] = m[1] * x + m[5] * y + m[9]  * z
    out[2] = m[2] * x + m[6] * y + m[10] * z
    return out
}

// Transform by quaternion
const vec3TransformQuat = (out: Float32Array, a: Float32Array, q: Float32Array): Float32Array => {
    const x = a[0], y = a[1], z = a[2]
    const qx = q[0], qy = q[1], qz = q[2], qw = q[3]
    const ix = qw * x + qy * z - qz * y
    const iy = qw * y + qz * x - qx * z
    const iz = qw * z + qx * y - qy * x
    const iw = -qx * x - qy * y - qz * z
    out[0] = ix * qw + iw * -qx + iy * -qz - iz * -qy
    out[1] = iy * qw + iw * -qy + iz * -qx - ix * -qz
    out[2] = iz * qw + iw * -qz + ix * -qy - iy * -qx
    return out
}
```

## Quat — Quaternion

Stored as `Float32Array(4)` in `[x, y, z, w]` order.

```typescript
const quatCreate = (): Float32Array => {
    const out = new Float32Array(4)
    out[3] = 1 // identity: [0, 0, 0, 1]
    return out
}

const quatIdentity = (out: Float32Array): Float32Array => {
    out[0] = 0; out[1] = 0; out[2] = 0; out[3] = 1
    return out
}

const quatMultiply = (out: Float32Array, a: Float32Array, b: Float32Array): Float32Array => {
    const ax = a[0], ay = a[1], az = a[2], aw = a[3]
    const bx = b[0], by = b[1], bz = b[2], bw = b[3]
    out[0] = ax * bw + aw * bx + ay * bz - az * by
    out[1] = ay * bw + aw * by + az * bx - ax * bz
    out[2] = az * bw + aw * bz + ax * by - ay * bx
    out[3] = aw * bw - ax * bx - ay * by - az * bz
    return out
}

const quatNormalize = (out: Float32Array, a: Float32Array): Float32Array => {
    const len = Math.sqrt(a[0] * a[0] + a[1] * a[1] + a[2] * a[2] + a[3] * a[3])
    if (len > 1e-8) {
        const inv = 1 / len
        out[0] = a[0] * inv; out[1] = a[1] * inv
        out[2] = a[2] * inv; out[3] = a[3] * inv
    }
    return out
}

const quatSlerp = (out: Float32Array, a: Float32Array, b: Float32Array, t: number): Float32Array => {
    let cosHalf = a[0] * b[0] + a[1] * b[1] + a[2] * b[2] + a[3] * b[3]
    const bx = cosHalf < 0 ? -b[0] : b[0]
    const by = cosHalf < 0 ? -b[1] : b[1]
    const bz = cosHalf < 0 ? -b[2] : b[2]
    const bw = cosHalf < 0 ? -b[3] : b[3]
    cosHalf = Math.abs(cosHalf)

    if (cosHalf >= 1.0 - 1e-6) {
        // Nearly identical — lerp
        out[0] = a[0] + (bx - a[0]) * t
        out[1] = a[1] + (by - a[1]) * t
        out[2] = a[2] + (bz - a[2]) * t
        out[3] = a[3] + (bw - a[3]) * t
        return quatNormalize(out, out)
    }

    const halfAngle = Math.acos(cosHalf)
    const sinHalf = Math.sin(halfAngle)
    const wa = Math.sin((1 - t) * halfAngle) / sinHalf
    const wb = Math.sin(t * halfAngle) / sinHalf

    out[0] = a[0] * wa + bx * wb
    out[1] = a[1] * wa + by * wb
    out[2] = a[2] * wa + bz * wb
    out[3] = a[3] * wa + bw * wb
    return out
}

// Euler angles (ZYX intrinsic = XYZ extrinsic) to quaternion — Z-up convention
const quatFromEuler = (out: Float32Array, x: number, y: number, z: number): Float32Array => {
    const hx = x * 0.5, hy = y * 0.5, hz = z * 0.5
    const cx = Math.cos(hx), sx = Math.sin(hx)
    const cy = Math.cos(hy), sy = Math.sin(hy)
    const cz = Math.cos(hz), sz = Math.sin(hz)
    out[0] = sx * cy * cz - cx * sy * sz
    out[1] = cx * sy * cz + sx * cy * sz
    out[2] = cx * cy * sz - sx * sy * cz
    out[3] = cx * cy * cz + sx * sy * sz
    return out
}

// Axis-angle to quaternion
const quatFromAxisAngle = (out: Float32Array, axis: Float32Array, angle: number): Float32Array => {
    const half = angle * 0.5
    const s = Math.sin(half)
    out[0] = axis[0] * s
    out[1] = axis[1] * s
    out[2] = axis[2] * s
    out[3] = Math.cos(half)
    return out
}

const quatInvert = (out: Float32Array, a: Float32Array): Float32Array => {
    const dot = a[0] * a[0] + a[1] * a[1] + a[2] * a[2] + a[3] * a[3]
    const invDot = dot > 0 ? 1 / dot : 0
    out[0] = -a[0] * invDot
    out[1] = -a[1] * invDot
    out[2] = -a[2] * invDot
    out[3] = a[3] * invDot
    return out
}
```

## Mat4 — 4×4 Matrix

Stored as `Float32Array(16)` in **column-major** order:

```
Memory layout:
[m0  m1  m2  m3    m4  m5  m6  m7    m8  m9  m10 m11   m12 m13 m14 m15]
 col0              col1              col2              col3 (translation)

Logical layout:
| m0  m4  m8   m12 |
| m1  m5  m9   m13 |
| m2  m6  m10  m14 |
| m3  m7  m11  m15 |
```

```typescript
const mat4Create = (): Float32Array => {
    const out = new Float32Array(16)
    out[0] = out[5] = out[10] = out[15] = 1 // identity
    return out
}

const mat4Identity = (out: Float32Array): Float32Array => {
    out.fill(0)
    out[0] = out[5] = out[10] = out[15] = 1
    return out
}

const mat4Multiply = (out: Float32Array, a: Float32Array, b: Float32Array): Float32Array => {
    const a00 = a[0], a01 = a[1], a02 = a[2], a03 = a[3]
    const a10 = a[4], a11 = a[5], a12 = a[6], a13 = a[7]
    const a20 = a[8], a21 = a[9], a22 = a[10], a23 = a[11]
    const a30 = a[12], a31 = a[13], a32 = a[14], a33 = a[15]

    for (let i = 0; i < 4; i++) {
        const bi0 = b[i], bi1 = b[i + 4], bi2 = b[i + 8], bi3 = b[i + 12]
        out[i]      = a00 * bi0 + a10 * bi1 + a20 * bi2 + a30 * bi3
        out[i + 4]  = a01 * bi0 + a11 * bi1 + a21 * bi2 + a31 * bi3
        out[i + 8]  = a02 * bi0 + a12 * bi1 + a22 * bi2 + a32 * bi3
        out[i + 12] = a03 * bi0 + a13 * bi1 + a23 * bi2 + a33 * bi3
    }
    return out
}

const mat4Invert = (out: Float32Array, a: Float32Array): Float32Array | null => {
    // Standard 4×4 inversion via cofactors
    const a00 = a[0], a01 = a[1], a02 = a[2], a03 = a[3]
    const a10 = a[4], a11 = a[5], a12 = a[6], a13 = a[7]
    const a20 = a[8], a21 = a[9], a22 = a[10], a23 = a[11]
    const a30 = a[12], a31 = a[13], a32 = a[14], a33 = a[15]

    const b00 = a00 * a11 - a01 * a10, b01 = a00 * a12 - a02 * a10
    const b02 = a00 * a13 - a03 * a10, b03 = a01 * a12 - a02 * a11
    const b04 = a01 * a13 - a03 * a11, b05 = a02 * a13 - a03 * a12
    const b06 = a20 * a31 - a21 * a30, b07 = a20 * a32 - a22 * a30
    const b08 = a20 * a33 - a23 * a30, b09 = a21 * a32 - a22 * a31
    const b10 = a21 * a33 - a23 * a31, b11 = a22 * a33 - a23 * a32

    let det = b00 * b11 - b01 * b10 + b02 * b09 + b03 * b08 - b04 * b07 + b05 * b06
    if (Math.abs(det) < 1e-8) return null
    det = 1.0 / det

    out[0]  = (a11 * b11 - a12 * b10 + a13 * b09) * det
    out[1]  = (a02 * b10 - a01 * b11 - a03 * b09) * det
    out[2]  = (a31 * b05 - a32 * b04 + a33 * b03) * det
    out[3]  = (a22 * b04 - a21 * b05 - a23 * b03) * det
    out[4]  = (a12 * b08 - a10 * b11 - a13 * b07) * det
    out[5]  = (a00 * b11 - a02 * b08 + a03 * b07) * det
    out[6]  = (a32 * b02 - a30 * b05 - a33 * b01) * det
    out[7]  = (a20 * b05 - a22 * b02 + a23 * b01) * det
    out[8]  = (a10 * b10 - a11 * b08 + a13 * b06) * det
    out[9]  = (a01 * b08 - a00 * b10 - a03 * b06) * det
    out[10] = (a30 * b04 - a31 * b02 + a33 * b00) * det
    out[11] = (a21 * b02 - a20 * b04 - a23 * b00) * det
    out[12] = (a11 * b07 - a10 * b09 - a12 * b06) * det
    out[13] = (a00 * b09 - a01 * b07 + a02 * b06) * det
    out[14] = (a31 * b01 - a30 * b03 - a32 * b00) * det
    out[15] = (a20 * b03 - a21 * b01 + a22 * b00) * det
    return out
}

// Compose TRS (Translation, Rotation as quat, Scale) into a 4×4 matrix
const mat4FromTRS = (
    out: Float32Array,
    t: Float32Array, // vec3
    r: Float32Array, // quat [x, y, z, w]
    s: Float32Array, // vec3
): Float32Array => {
    const x = r[0], y = r[1], z = r[2], w = r[3]
    const x2 = x + x, y2 = y + y, z2 = z + z
    const xx = x * x2, xy = x * y2, xz = x * z2
    const yy = y * y2, yz = y * z2, zz = z * z2
    const wx = w * x2, wy = w * y2, wz = w * z2

    out[0]  = (1 - (yy + zz)) * s[0]
    out[1]  = (xy + wz) * s[0]
    out[2]  = (xz - wy) * s[0]
    out[3]  = 0
    out[4]  = (xy - wz) * s[1]
    out[5]  = (1 - (xx + zz)) * s[1]
    out[6]  = (yz + wx) * s[1]
    out[7]  = 0
    out[8]  = (xz + wy) * s[2]
    out[9]  = (yz - wx) * s[2]
    out[10] = (1 - (xx + yy)) * s[2]
    out[11] = 0
    out[12] = t[0]
    out[13] = t[1]
    out[14] = t[2]
    out[15] = 1
    return out
}

// Perspective projection (Z-up, right-handed, depth [0, 1] for WebGPU, [-1, 1] for WebGL2)
const mat4Perspective = (
    out: Float32Array,
    fovY: number,       // vertical FOV in radians
    aspect: number,
    near: number,
    far: number,
    clipDepthZeroToOne: boolean, // true for WebGPU, false for WebGL2
): Float32Array => {
    const f = 1.0 / Math.tan(fovY / 2)
    out.fill(0)
    out[0] = f / aspect
    out[5] = f
    out[11] = -1 // right-handed

    if (clipDepthZeroToOne) {
        // WebGPU: depth range [0, 1]
        out[10] = far / (near - far)
        out[14] = (near * far) / (near - far)
    } else {
        // WebGL2: depth range [-1, 1]
        out[10] = (far + near) / (near - far)
        out[14] = (2 * far * near) / (near - far)
    }
    return out
}

// Orthographic projection (for shadow maps)
const mat4Orthographic = (
    out: Float32Array,
    left: number, right: number,
    bottom: number, top: number,
    near: number, far: number,
    clipDepthZeroToOne: boolean,
): Float32Array => {
    const lr = 1 / (left - right)
    const bt = 1 / (bottom - top)
    const nf = 1 / (near - far)
    out.fill(0)
    out[0] = -2 * lr
    out[5] = -2 * bt
    out[12] = (left + right) * lr
    out[13] = (top + bottom) * bt
    out[15] = 1

    if (clipDepthZeroToOne) {
        out[10] = nf
        out[14] = near * nf
    } else {
        out[10] = 2 * nf
        out[14] = (far + near) * nf
    }
    return out
}

// LookAt (Z-up, right-handed)
const mat4LookAt = (
    out: Float32Array,
    eye: Float32Array,
    target: Float32Array,
    up: Float32Array, // default [0, 0, 1] for Z-up
): Float32Array => {
    const fx = target[0] - eye[0], fy = target[1] - eye[1], fz = target[2] - eye[2]
    const fLen = Math.sqrt(fx * fx + fy * fy + fz * fz)
    const f0 = fx / fLen, f1 = fy / fLen, f2 = fz / fLen // forward

    // right = forward × up
    const r0 = f1 * up[2] - f2 * up[1]
    const r1 = f2 * up[0] - f0 * up[2]
    const r2 = f0 * up[1] - f1 * up[0]
    const rLen = Math.sqrt(r0 * r0 + r1 * r1 + r2 * r2)
    const s0 = r0 / rLen, s1 = r1 / rLen, s2 = r2 / rLen // right (normalized)

    // recalculated up = right × forward
    const u0 = s1 * f2 - s2 * f1
    const u1 = s2 * f0 - s0 * f2
    const u2 = s0 * f1 - s1 * f0

    out[0] = s0; out[1] = u0; out[2] = -f0; out[3] = 0
    out[4] = s1; out[5] = u1; out[6] = -f1; out[7] = 0
    out[8] = s2; out[9] = u2; out[10] = -f2; out[11] = 0
    out[12] = -(s0 * eye[0] + s1 * eye[1] + s2 * eye[2])
    out[13] = -(u0 * eye[0] + u1 * eye[1] + u2 * eye[2])
    out[14] = (f0 * eye[0] + f1 * eye[1] + f2 * eye[2])
    out[15] = 1
    return out
}
```

## AABB — Axis-Aligned Bounding Box

```typescript
// Stored as Float32Array(6): [minX, minY, minZ, maxX, maxY, maxZ]
const aabbCreate = (): Float32Array =>
    new Float32Array([Infinity, Infinity, Infinity, -Infinity, -Infinity, -Infinity])

const aabbExpandByPoint = (aabb: Float32Array, x: number, y: number, z: number): void => {
    if (x < aabb[0]) aabb[0] = x; if (x > aabb[3]) aabb[3] = x
    if (y < aabb[1]) aabb[1] = y; if (y > aabb[4]) aabb[4] = y
    if (z < aabb[2]) aabb[2] = z; if (z > aabb[5]) aabb[5] = z
}

const aabbUnion = (out: Float32Array, a: Float32Array, b: Float32Array): Float32Array => {
    out[0] = Math.min(a[0], b[0]); out[1] = Math.min(a[1], b[1]); out[2] = Math.min(a[2], b[2])
    out[3] = Math.max(a[3], b[3]); out[4] = Math.max(a[4], b[4]); out[5] = Math.max(a[5], b[5])
    return out
}

const aabbSurfaceArea = (aabb: Float32Array): number => {
    const dx = aabb[3] - aabb[0], dy = aabb[4] - aabb[1], dz = aabb[5] - aabb[2]
    return 2 * (dx * dy + dy * dz + dz * dx)
}

// Transform AABB by a 4×4 matrix (produces a larger AABB that encloses the transformed box)
const aabbTransform = (out: Float32Array, aabb: Float32Array, m: Float32Array): Float32Array => {
    // Use the fast AABB transform method (8 corners → new AABB, optimized to avoid 8 matrix multiplies)
    const center = [
        (aabb[0] + aabb[3]) * 0.5,
        (aabb[1] + aabb[4]) * 0.5,
        (aabb[2] + aabb[5]) * 0.5,
    ]
    const extent = [
        (aabb[3] - aabb[0]) * 0.5,
        (aabb[4] - aabb[1]) * 0.5,
        (aabb[5] - aabb[2]) * 0.5,
    ]

    // Transform center
    const tc = [
        m[0] * center[0] + m[4] * center[1] + m[8] * center[2] + m[12],
        m[1] * center[0] + m[5] * center[1] + m[9] * center[2] + m[13],
        m[2] * center[0] + m[6] * center[1] + m[10] * center[2] + m[14],
    ]

    // Transform extent (absolute values of matrix columns × extent)
    const te = [
        Math.abs(m[0]) * extent[0] + Math.abs(m[4]) * extent[1] + Math.abs(m[8]) * extent[2],
        Math.abs(m[1]) * extent[0] + Math.abs(m[5]) * extent[1] + Math.abs(m[9]) * extent[2],
        Math.abs(m[2]) * extent[0] + Math.abs(m[6]) * extent[1] + Math.abs(m[10]) * extent[2],
    ]

    out[0] = tc[0] - te[0]; out[1] = tc[1] - te[1]; out[2] = tc[2] - te[2]
    out[3] = tc[0] + te[0]; out[4] = tc[1] + te[1]; out[5] = tc[2] + te[2]
    return out
}
```

## Frustum — 6-Plane Frustum

```typescript
// 6 planes, each stored as [nx, ny, nz, d] in a Float32Array(24)
// Plane equation: dot(normal, point) + d >= 0 means inside
const frustumFromViewProjection = (out: Float32Array, vp: Float32Array): Float32Array => {
    // Extract planes from view-projection matrix (Gribb-Hartmann method)
    // Left:   row3 + row0
    out[0]  = vp[3] + vp[0]; out[1]  = vp[7] + vp[4]; out[2]  = vp[11] + vp[8];  out[3]  = vp[15] + vp[12]
    // Right:  row3 - row0
    out[4]  = vp[3] - vp[0]; out[5]  = vp[7] - vp[4]; out[6]  = vp[11] - vp[8];  out[7]  = vp[15] - vp[12]
    // Bottom: row3 + row1
    out[8]  = vp[3] + vp[1]; out[9]  = vp[7] + vp[5]; out[10] = vp[11] + vp[9];  out[11] = vp[15] + vp[13]
    // Top:    row3 - row1
    out[12] = vp[3] - vp[1]; out[13] = vp[7] - vp[5]; out[14] = vp[11] - vp[9];  out[15] = vp[15] - vp[13]
    // Near:   row3 + row2
    out[16] = vp[3] + vp[2]; out[17] = vp[7] + vp[6]; out[18] = vp[11] + vp[10]; out[19] = vp[15] + vp[14]
    // Far:    row3 - row2
    out[20] = vp[3] - vp[2]; out[21] = vp[7] - vp[6]; out[22] = vp[11] - vp[10]; out[23] = vp[15] - vp[14]

    // Normalize each plane
    for (let i = 0; i < 6; i++) {
        const o = i * 4
        const len = Math.sqrt(out[o] * out[o] + out[o + 1] * out[o + 1] + out[o + 2] * out[o + 2])
        if (len > 0) {
            const inv = 1 / len
            out[o] *= inv; out[o + 1] *= inv; out[o + 2] *= inv; out[o + 3] *= inv
        }
    }
    return out
}

// Test AABB against frustum — returns true if AABB is at least partially inside
const frustumContainsAABB = (frustum: Float32Array, aabb: Float32Array): boolean => {
    for (let i = 0; i < 6; i++) {
        const o = i * 4
        const nx = frustum[o], ny = frustum[o + 1], nz = frustum[o + 2], d = frustum[o + 3]

        // Test the p-vertex (the AABB corner most in the direction of the plane normal)
        const px = nx >= 0 ? aabb[3] : aabb[0]
        const py = ny >= 0 ? aabb[4] : aabb[1]
        const pz = nz >= 0 ? aabb[5] : aabb[2]

        if (nx * px + ny * py + nz * pz + d < 0) return false // entirely outside this plane
    }
    return true
}
```

## Ray

```typescript
interface Ray {
    origin: Float32Array   // vec3
    direction: Float32Array // vec3, normalized
}

const createRay = (origin: Float32Array, direction: Float32Array): Ray => ({
    origin: new Float32Array(origin),
    direction: new Float32Array(direction),
})

const rayAt = (out: Float32Array, ray: Ray, t: number): Float32Array =>
    vec3ScaleAndAdd(out, ray.origin, ray.direction, t)
```

## Exports

```typescript
// math/index.ts — all public exports
export {
    // Vec3
    vec3Create, vec3Set, vec3Copy, vec3Add, vec3Sub, vec3Scale, vec3ScaleAndAdd,
    vec3Multiply, vec3Dot, vec3Cross, vec3Length, vec3LengthSq, vec3Distance,
    vec3Normalize, vec3Lerp, vec3TransformMat4, vec3TransformMat4Dir, vec3TransformQuat,

    // Quat
    quatCreate, quatIdentity, quatMultiply, quatNormalize, quatSlerp,
    quatFromEuler, quatFromAxisAngle, quatInvert,

    // Mat4
    mat4Create, mat4Identity, mat4Multiply, mat4Invert, mat4FromTRS,
    mat4Perspective, mat4Orthographic, mat4LookAt,

    // AABB
    aabbCreate, aabbExpandByPoint, aabbUnion, aabbSurfaceArea, aabbTransform,

    // Frustum
    frustumFromViewProjection, frustumContainsAABB,

    // Ray
    createRay, rayAt,
}
```
