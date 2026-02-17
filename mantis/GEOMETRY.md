# Parametric Geometry Generators

## Overview

Mantis provides built-in generators for common geometric primitives. Each
generator returns raw typed arrays (positions, normals, UVs, indices) that are
uploaded to the GPU as a `Geometry` object.

All generators:
- Produce Z-up geometry (consistent with Mantis's coordinate system)
- Generate smooth normals by default
- Output indexed triangles with `Uint16Array` indices
- Return interleaved vertex data ready for GPU upload

## Geometry Object

```typescript
interface GeometryData {
  positions: Float32Array   // vec3 per vertex
  normals: Float32Array     // vec3 per vertex
  uvs: Float32Array         // vec2 per vertex
  indices: Uint16Array      // triangle indices (or Uint32Array if > 65535 vertices)
  vertexCount: number
  indexCount: number
}

// Create GPU geometry from raw data
const geometry = new Geometry(device, data)
```

## Plane

A flat quad in the XY plane (Z-up means the plane faces +Z).

```typescript
const createPlane = (opts?: {
  width?: number       // default 1, extent along X
  height?: number      // default 1, extent along Y
  widthSegments?: number   // default 1
  heightSegments?: number  // default 1
}): GeometryData
```

```
  Y
  ↑
  ├───┬───┐
  │   │   │  (2×2 segments shown)
  ├───┼───┤
  │   │   │
  └───┴───┘ → X
  Normal: +Z (pointing up)
```

Vertices: `(widthSegments + 1) × (heightSegments + 1)`
Triangles: `widthSegments × heightSegments × 2`

UVs: (0,0) at bottom-left, (1,1) at top-right.

## Box

An axis-aligned box centered at the origin.

```typescript
const createBox = (opts?: {
  width?: number       // default 1, extent along X
  height?: number      // default 1, extent along Y
  depth?: number       // default 1, extent along Z
  widthSegments?: number   // default 1
  heightSegments?: number  // default 1
  depthSegments?: number   // default 1
}): GeometryData
```

6 faces, each subdivided independently. Normals point outward per face. UVs
mapped per face (0,0)→(1,1).

Vertices: depends on segments, minimum 24 (4 per face, shared normals require
separate vertices per face).

## Sphere

A UV sphere centered at the origin.

```typescript
const createSphere = (opts?: {
  radius?: number          // default 0.5
  widthSegments?: number   // default 32 (longitude divisions)
  heightSegments?: number  // default 16 (latitude divisions)
}): GeometryData
```

Poles are single vertices connected to triangle fans. Normals are normalized
position vectors (smooth sphere normals). UVs wrap horizontally (0→1 around
equator) and vertically (0 at south pole, 1 at north pole).

```
      Z (up)
      │  ╱ North pole
      │╱
  ────●──── → X
     ╱│
    ╱  │  South pole
  Y
```

Vertices: `(widthSegments + 1) × (heightSegments + 1)`
Triangles: `widthSegments × heightSegments × 2` (minus degenerate triangles at poles)

## Cone

A cone with its tip pointing up (+Z) and base centered at origin.

```typescript
const createCone = (opts?: {
  radius?: number          // default 0.5, base radius
  height?: number          // default 1, tip at Z=height
  radialSegments?: number  // default 32
  heightSegments?: number  // default 1
  openEnded?: boolean      // default false (include base cap)
}): GeometryData
```

The side surface has smoothed normals around the circumference but sharp edges
at the tip. If `openEnded` is false, a circular cap is generated at Z=0.

## Cylinder

A cylinder along the Z axis, centered at origin (base at Z=0, top at Z=height).

```typescript
const createCylinder = (opts?: {
  radiusTop?: number       // default 0.5
  radiusBottom?: number    // default 0.5
  height?: number          // default 1
  radialSegments?: number  // default 32
  heightSegments?: number  // default 1
  openEnded?: boolean      // default false (include top and bottom caps)
}): GeometryData
```

Setting `radiusTop !== radiusBottom` creates a truncated cone.
Setting `radiusTop = 0` is equivalent to a cone.

Side normals are smooth around the circumference. Cap normals point along ±Z.

## Capsule

A cylinder with hemispherical caps. Commonly used for character collision
shapes but also useful as visible geometry.

```typescript
const createCapsule = (opts?: {
  radius?: number          // default 0.25, hemisphere and cylinder radius
  height?: number          // default 1, total height including caps
  radialSegments?: number  // default 32
  heightSegments?: number  // default 1 (cylinder body only)
  capSegments?: number     // default 8 (hemisphere subdivisions)
}): GeometryData
```

```
      Z (up)
      ╭──╮    ← top hemisphere (Z = height/2 + radius)
      │  │
      │  │    ← cylinder body
      │  │
      ╰──╯    ← bottom hemisphere (Z = -height/2 - radius)
```

The cylinder body spans from `Z = -(height/2 - radius)` to `Z = (height/2 - radius)`.
Hemispheres are attached at both ends. Normals are continuous across the
cylinder-hemisphere junction.

## Circle

A flat circular disk in the XY plane facing +Z.

```typescript
const createCircle = (opts?: {
  radius?: number          // default 0.5
  segments?: number        // default 32
  thetaStart?: number      // default 0, start angle in radians
  thetaLength?: number     // default 2π, arc length in radians
}): GeometryData
```

For partial circles (`thetaLength < 2π`), generates a pie-slice shape.
Center vertex is at the origin. Normals all point +Z.

UVs: center = (0.5, 0.5), edge mapped circularly.

## Implementation Pattern

All generators follow the same pattern:

```typescript
const createSphere = (opts: SphereOptions = {}): GeometryData => {
  const {
    radius = 0.5,
    widthSegments = 32,
    heightSegments = 16,
  } = opts

  const vertexCount = (widthSegments + 1) * (heightSegments + 1)
  const indexCount = widthSegments * heightSegments * 6

  const positions = new Float32Array(vertexCount * 3)
  const normals = new Float32Array(vertexCount * 3)
  const uvs = new Float32Array(vertexCount * 2)
  const indices = vertexCount > 65535
    ? new Uint32Array(indexCount)
    : new Uint16Array(indexCount)

  let vIdx = 0
  let iIdx = 0

  for (let y = 0; y <= heightSegments; y++) {
    const v = y / heightSegments
    const phi = v * Math.PI  // 0 at north pole, π at south pole

    for (let x = 0; x <= widthSegments; x++) {
      const u = x / widthSegments
      const theta = u * Math.PI * 2

      // Z-up sphere: X = sin(phi)cos(theta), Y = sin(phi)sin(theta), Z = cos(phi)
      const sinPhi = Math.sin(phi)
      const nx = sinPhi * Math.cos(theta)
      const ny = sinPhi * Math.sin(theta)
      const nz = Math.cos(phi)

      positions[vIdx * 3]     = nx * radius
      positions[vIdx * 3 + 1] = ny * radius
      positions[vIdx * 3 + 2] = nz * radius

      normals[vIdx * 3]     = nx
      normals[vIdx * 3 + 1] = ny
      normals[vIdx * 3 + 2] = nz

      uvs[vIdx * 2]     = u
      uvs[vIdx * 2 + 1] = 1 - v  // flip V so 0 is bottom

      vIdx++
    }
  }

  // Generate triangle indices
  for (let y = 0; y < heightSegments; y++) {
    for (let x = 0; x < widthSegments; x++) {
      const a = y * (widthSegments + 1) + x
      const b = a + widthSegments + 1

      indices[iIdx++] = a
      indices[iIdx++] = b
      indices[iIdx++] = a + 1

      indices[iIdx++] = a + 1
      indices[iIdx++] = b
      indices[iIdx++] = b + 1
    }
  }

  return { positions, normals, uvs, indices, vertexCount, indexCount: iIdx }
}
```

## Usage

```typescript
// Create geometry
const sphereData = createSphere({ radius: 1, widthSegments: 64 })
const sphereGeometry = new Geometry(engine.device, sphereData)

// Create mesh
const sphere = scene.createMesh(sphereGeometry, material)
sphere.position = [0, 0, 5]

// Reuse geometry across meshes (GPU buffers shared)
const sphere2 = scene.createMesh(sphereGeometry, differentMaterial)
sphere2.position = [10, 0, 5]
```

## Memory

Geometries are lightweight. A 32-segment sphere:
- Vertices: 33 × 17 = 561 → ~18 KB (positions + normals + UVs)
- Indices: 32 × 16 × 6 = 3072 → ~6 KB
- Total GPU memory: ~24 KB

CPU-side typed arrays are released after GPU upload by default. To retain them
(for raycasting without mesh BVH), pass `retainCPUData: true` to the Geometry
constructor.
