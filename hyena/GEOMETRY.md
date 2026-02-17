# Geometry

## Overview

Hyena provides parametric geometry generators for common 3D primitives and a `BufferGeometry` base class for custom or loaded geometry. All generators produce indexed vertex buffers with positions, normals, and UVs — ready for GPU upload.

## BufferGeometry

The base class for all geometry in Hyena:

```typescript
interface BufferGeometry {
  readonly positions: Float32Array     // vec3 per vertex
  readonly normals: Float32Array       // vec3 per vertex
  readonly uvs: Float32Array           // vec2 per vertex
  readonly indices: Uint16Array | Uint32Array

  // Optional attributes
  readonly vertexColors?: Float32Array       // vec4 per vertex (RGBA)
  readonly materialIndices?: Uint8Array      // scalar per vertex (_materialindex)
  readonly jointIndices?: Uint8Array         // vec4 per vertex (skinning)
  readonly jointWeights?: Float32Array       // vec4 per vertex (skinning)

  // Computed on first access, cached
  readonly boundingBox: AABB
  readonly vertexCount: number
  readonly indexCount: number

  // GPU resources (created on first render)
  _vertexBuffer: GpuBuffer | null
  _indexBuffer: GpuBuffer | null

  dispose(): void
}
```

### Index Format Selection

`Uint16Array` is used when vertex count ≤ 65535 (saves memory and bandwidth). `Uint32Array` for larger meshes. The generator picks the smallest format that fits.

### Bounding Box

Computed lazily from position data on first access:

```
computeBoundingBox(positions):
  min = [Infinity, Infinity, Infinity]
  max = [-Infinity, -Infinity, -Infinity]
  for i in 0..positions.length step 3:
    min = min(min, positions[i..i+3])
    max = max(max, positions[i..i+3])
  return { min, max }
```

Cached — recomputed only if vertex data changes (which is rare after creation).

## Parametric Primitives

All generators follow a consistent API pattern:

```typescript
const geometry = new XxxGeometry(options)
```

Options have sensible defaults. All primitives are generated in local space centered at the origin.

### Z-Up Convention

All primitives respect Hyena's Z-up, right-handed coordinate system:
- **Plane** lies in the XY plane (normal = +Z)
- **Box** is axis-aligned with width=X, depth=Y, height=Z
- **Cylinder/Cone/Capsule** stand upright along the Z axis
- **Sphere** has poles at ±Z

---

### PlaneGeometry

A flat quad in the XY plane, facing +Z.

```typescript
interface PlaneGeometryOptions {
  width?: number            // extent along X — default 1
  height?: number           // extent along Y — default 1
  widthSegments?: number    // subdivisions along X — default 1
  heightSegments?: number   // subdivisions along Y — default 1
}
```

Generation:

```
for iy in 0..heightSegments+1:
  for ix in 0..widthSegments+1:
    x = (ix / widthSegments - 0.5) * width
    y = (iy / heightSegments - 0.5) * height
    z = 0

    position = [x, y, z]
    normal   = [0, 0, 1]
    uv       = [ix / widthSegments, iy / heightSegments]
```

Vertex count: `(widthSegments + 1) × (heightSegments + 1)`
Triangle count: `widthSegments × heightSegments × 2`

---

### BoxGeometry

An axis-aligned box centered at the origin.

```typescript
interface BoxGeometryOptions {
  width?: number             // extent along X — default 1
  depth?: number             // extent along Y — default 1
  height?: number            // extent along Z — default 1
  widthSegments?: number     // default 1
  depthSegments?: number     // default 1
  heightSegments?: number    // default 1
}
```

Generated as 6 independent face planes (separate vertices per face for sharp normals):

```
Faces:
  +X face: normal [1, 0, 0]
  -X face: normal [-1, 0, 0]
  +Y face: normal [0, 1, 0]
  -Y face: normal [0, -1, 0]
  +Z face: normal [0, 0, 1]   (top)
  -Z face: normal [0, 0, -1]  (bottom)
```

Each face is a subdivided quad, generated like PlaneGeometry but oriented along the face normal.

Vertex count: `6 × (segsA + 1) × (segsB + 1)` (each face has separate vertices)
Triangle count: `6 × segsA × segsB × 2`

---

### SphereGeometry

UV sphere with poles along the Z axis.

```typescript
interface SphereGeometryOptions {
  radius?: number             // default 0.5
  widthSegments?: number      // horizontal slices — default 32
  heightSegments?: number     // vertical slices — default 16
  phiStart?: number           // horizontal start angle — default 0
  phiLength?: number          // horizontal sweep — default 2π
  thetaStart?: number         // vertical start angle — default 0
  thetaLength?: number        // vertical sweep — default π
}
```

Generation (Z-up — theta measured from +Z pole):

```
for iy in 0..heightSegments:
  for ix in 0..widthSegments:
    u = ix / widthSegments
    v = iy / heightSegments

    phi   = phiStart + u * phiLength          // azimuth (around Z)
    theta = thetaStart + v * thetaLength      // elevation (from +Z)

    x = radius * sin(theta) * cos(phi)
    y = radius * sin(theta) * sin(phi)
    z = radius * cos(theta)

    position = [x, y, z]
    normal   = normalize([x, y, z])
    uv       = [u, v]
```

Pole vertices are shared (single vertex at each pole with averaged UV).

Vertex count: `(widthSegments + 1) × (heightSegments + 1)`
Triangle count: `widthSegments × heightSegments × 2` (minus degenerate pole triangles)

---

### ConeGeometry

A cone standing along the Z axis with its base at Z=0 and tip at Z=height.

```typescript
interface ConeGeometryOptions {
  radius?: number            // base radius — default 0.5
  height?: number            // height along Z — default 1
  radialSegments?: number    // segments around circumference — default 32
  heightSegments?: number    // segments along height — default 1
  openEnded?: boolean        // skip base cap — default false
}
```

Generation:

```
// Side surface
for iy in 0..heightSegments:
  for ix in 0..radialSegments:
    u = ix / radialSegments
    v = iy / heightSegments

    phi = u * 2π
    currentRadius = radius * (1 - v)     // linearly taper to 0
    x = currentRadius * cos(phi)
    y = currentRadius * sin(phi)
    z = v * height

    // Normal: perpendicular to the slant surface
    slant = atan2(radius, height)
    normal = normalize([cos(phi) * cos(slant), sin(phi) * cos(slant), sin(slant)])

// Tip vertex (single shared vertex at [0, 0, height])

// Base cap (if !openEnded)
// Circle of vertices at z=0, normal [0, 0, -1]
```

---

### CylinderGeometry

A cylinder standing along the Z axis.

```typescript
interface CylinderGeometryOptions {
  radiusTop?: number         // top radius — default 0.5
  radiusBottom?: number      // bottom radius — default 0.5
  height?: number            // height along Z — default 1
  radialSegments?: number    // default 32
  heightSegments?: number    // default 1
  openEnded?: boolean        // skip caps — default false
}
```

When `radiusTop !== radiusBottom`, this generates a truncated cone (frustum).

Generation:

```
// Side surface
for iy in 0..heightSegments:
  for ix in 0..radialSegments:
    v = iy / heightSegments
    currentRadius = lerp(radiusBottom, radiusTop, v)
    phi = (ix / radialSegments) * 2π

    x = currentRadius * cos(phi)
    y = currentRadius * sin(phi)
    z = v * height - height / 2     // centered at origin

    // Normal: perpendicular to the slant
    dr = radiusTop - radiusBottom
    slant = atan2(dr, height)
    normal = normalize([cos(phi) * cos(slant), sin(phi) * cos(slant), sin(slant)])

// Top cap (if !openEnded && radiusTop > 0)
// Circle at z = height/2, normal [0, 0, 1]

// Bottom cap (if !openEnded && radiusBottom > 0)
// Circle at z = -height/2, normal [0, 0, -1]
```

---

### CapsuleGeometry

A cylinder with hemispherical caps on both ends, standing along Z.

```typescript
interface CapsuleGeometryOptions {
  radius?: number            // cap and cylinder radius — default 0.25
  height?: number            // total height (including caps) — default 1
  radialSegments?: number    // default 32
  heightSegments?: number    // cylinder body segments — default 1
  capSegments?: number       // hemisphere subdivisions — default 8
}
```

The capsule is composed of three parts:
1. **Top hemisphere** (half sphere at Z = cylinderHeight/2)
2. **Cylinder body** (from -cylinderHeight/2 to +cylinderHeight/2)
3. **Bottom hemisphere** (half sphere at Z = -cylinderHeight/2)

Where `cylinderHeight = height - 2 * radius`.

Generation:

```
// Top hemisphere (theta: 0 → π/2)
for iy in 0..capSegments:
  theta = (iy / capSegments) * π/2
  for ix in 0..radialSegments:
    phi = (ix / radialSegments) * 2π
    x = radius * sin(theta) * cos(phi)
    y = radius * sin(theta) * sin(phi)
    z = radius * cos(theta) + cylinderHeight/2

    normal = normalize([sin(theta)*cos(phi), sin(theta)*sin(phi), cos(theta)])

// Cylinder body (same as CylinderGeometry with equal radii)

// Bottom hemisphere (theta: π/2 → π)
for iy in 0..capSegments:
  theta = π/2 + (iy / capSegments) * π/2
  // ... same pattern, z offset = -cylinderHeight/2
```

Vertices are shared at the seams between hemisphere and cylinder for a smooth surface.

---

### CircleGeometry

A flat disc in the XY plane, facing +Z. Useful for ground markers, decals, etc.

```typescript
interface CircleGeometryOptions {
  radius?: number            // default 0.5
  segments?: number          // default 32
  thetaStart?: number        // start angle — default 0
  thetaLength?: number       // sweep angle — default 2π
}
```

Generation:

```
// Center vertex
position = [0, 0, 0]
normal = [0, 0, 1]
uv = [0.5, 0.5]

// Ring vertices
for i in 0..segments:
  theta = thetaStart + (i / segments) * thetaLength
  x = radius * cos(theta)
  y = radius * sin(theta)

  position = [x, y, 0]
  normal = [0, 0, 1]
  uv = [(cos(theta) + 1) / 2, (sin(theta) + 1) / 2]

// Triangles fan from center to each adjacent pair of ring vertices
```

Vertex count: `segments + 2` (center + ring + closing vertex)
Triangle count: `segments`

---

## Custom Geometry

Users can create geometry from raw arrays:

```typescript
const geometry = new BufferGeometry({
  positions: new Float32Array([0,0,0, 1,0,0, 0,1,0]),
  normals: new Float32Array([0,0,1, 0,0,1, 0,0,1]),
  uvs: new Float32Array([0,0, 1,0, 0,1]),
  indices: new Uint16Array([0, 1, 2]),
})
```

## GPU Buffer Upload

Geometry is uploaded to the GPU lazily on first render:

```
prepareGeometry(geometry, device):
  if geometry._vertexBuffer: return   // already uploaded

  // Interleave vertex data for better cache performance
  interleaved = interleaveAttributes(geometry)
  geometry._vertexBuffer = device.createBuffer({
    data: interleaved,
    usage: VERTEX
  })
  geometry._indexBuffer = device.createBuffer({
    data: geometry.indices,
    usage: INDEX
  })
```

### Vertex Layout

Vertex data is interleaved for cache-friendly GPU access:

```
Per vertex (32–44 bytes depending on features):
  [position: 3×f32] [normal: 3×f32] [uv: 2×f32]
  [vertexColor: 4×f32]?  [materialIndex: 1×u8]?
  [jointIndices: 4×u8]?  [jointWeights: 4×f32]?
```

Interleaved layout means a single vertex buffer binding per draw call, and sequential vertex reads hit the same cache line.

## Disposal

```typescript
geometry.dispose():
  device.destroyBuffer(geometry._vertexBuffer)
  device.destroyBuffer(geometry._indexBuffer)
  geometry._vertexBuffer = null
  geometry._indexBuffer = null
```

Geometry can be shared across multiple meshes. Dispose only when no mesh references it (tracked via reference counting in the React bindings, manual in imperative usage).
