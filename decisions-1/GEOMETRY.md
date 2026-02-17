# GEOMETRY - Design Decisions

## Vertex Buffer Layout

**Decision: Separate buffers per attribute**

- Sources: Lynx, Mantis, Rabbit, Wren (4/9)
- Rejected: Interleaved (Caracal, Fennec, Hyena) — harder to add/remove attributes
- Rejected: Pre-allocated SoA (Bonobo) — too rigid

```
Buffer 0: positions    Float32Array (3 components)
Buffer 1: normals      Float32Array (3 components)
Buffer 2: uvs          Float32Array (2 components)
Buffer 3: colors       Float32Array (4 components, optional)
Buffer 4: materialIdx  Uint8Array   (1 component, optional)
Buffer 5: joints       Uint8Array   (4 components, optional)
Buffer 6: weights      Float32Array (4 components, optional)
```

Rationale: Easy to add/remove attributes. Simple partial updates (e.g., morph targets updating only positions). Negligible performance difference vs interleaved on modern hardware. Better alignment when mixing data types (avoids the Shark hybrid issue). Preserves glTF buffer layouts on import without forced conversion.

## Standard Vertex Attributes

**Decision: Standard attribute set (universal agreement)**

| Attribute | Type | Components | Required |
|-----------|------|-----------|----------|
| position | Float32 | 3 (x, y, z) | Yes |
| normal | Float32 | 3 | Yes |
| uv | Float32 | 2 | Yes |
| color | Float32 | 3 or 4 | Optional |
| _materialindex | Uint8 | 1 | Optional |
| joints | Uint8 | 4 | Optional (skinning) |
| weights | Float32 | 4 | Optional (skinning) |

## Index Buffer

**Decision: Automatic Uint16/Uint32 selection based on vertex count**

- Sources: All 9 implementations agree

```typescript
const indexFormat = vertexCount <= 65535 ? 'uint16' : 'uint32'
```

Uint16 saves half the memory and improves vertex cache hit rates on mobile GPUs. Sufficient for 99% of game assets.

## Bounding Volume

**Decision: AABB (Axis-Aligned Bounding Box) computed at creation**

- Sources: 8/9 use AABB (all except Bonobo which uses spheres)

```typescript
interface AABB {
  min: Float32Array  // [x, y, z]
  max: Float32Array  // [x, y, z]
}
```

Computed once from vertex positions at geometry creation. Used for frustum culling and coarse raycasting.

Also compute bounding sphere (center + radius) for cases where sphere tests are preferred (distance checks, LOD).

## Parametric Primitives

**Decision: Options object API, all in Z-up coordinate system (universal agreement)**

All primitives generated in Z-up, right-handed coordinates.

### Plane
```typescript
createPlaneGeometry({ width?: 1, height?: 1, widthSegments?: 1, heightSegments?: 1 })
```
Lies in XY plane, normal +Z. UVs: (0,0) at bottom-left, (1,1) at top-right.

### Box
```typescript
createBoxGeometry({ width?: 1, height?: 1, depth?: 1, widthSegments?: 1, heightSegments?: 1, depthSegments?: 1 })
```
6 faces with separate vertices per face (sharp edges). Normals point outward.

### Sphere (UV Sphere)
```typescript
createSphereGeometry({ radius?: 1, widthSegments?: 32, heightSegments?: 16 })
```
Poles at ±Z. Standard spherical-to-Cartesian for Z-up.

### Cone
```typescript
createConeGeometry({ radius?: 0.5, height?: 1, radialSegments?: 32, heightSegments?: 1, openEnded?: false })
```
Base at Z=0, tip at Z=height.

### Cylinder
```typescript
createCylinderGeometry({ radiusTop?: 0.5, radiusBottom?: 0.5, height?: 1, radialSegments?: 32, heightSegments?: 1, openEnded?: false })
```
Extends along Z axis. Cone is a cylinder with `radiusTop = 0`.

### Capsule
```typescript
createCapsuleGeometry({ radius?: 0.5, height?: 1, capSegments?: 8, radialSegments?: 32 })
```
Cylinder body + hemispherical caps. Vertices shared at hemisphere-cylinder boundary.

### Circle
```typescript
createCircleGeometry({ radius?: 0.5, segments?: 32 })
```
Flat disc in XY plane, normal +Z. Triangle fan from center.

## Normal Generation

**Decision: Smooth normals for curved surfaces, flat normals for box (universal agreement)**

- Sphere, cylinder, capsule, cone: averaged/interpolated smooth normals
- Box: separate vertices per face with flat normals (sharp edges)
- All normals normalized to unit length

## UV Mapping

- Box: each face independently mapped to (0,0)→(1,1)
- Sphere: standard UV sphere mapping (U = longitude/2π, V = latitude/π)
- Capsule: arc-length parameterization for uniform texture density (Lynx approach)
- Cylinder/Cone: cylindrical projection
- Plane/Circle: direct XY mapping

## GPU Upload

**Decision: Lazy GPU upload on first render (universal agreement)**

CPU-side typed arrays created immediately. GPU buffers created on first draw call. Avoids wasting GPU memory for geometry that may never be rendered.

## Custom Attributes

**Decision: Named properties on geometry object**

```typescript
const geometry = createGeometry({
  positions: new Float32Array([...]),
  normals: new Float32Array([...]),
  uvs: new Float32Array([...]),
  indices: new Uint16Array([...]),
  _materialindex: new Uint8Array([...]),  // custom attribute
})
```

The `_materialindex` attribute is recognized as standard by the material system. Additional custom attributes can be added as named properties.

## Geometry Lifecycle

**Decision: Reference counting for shared geometry**

- Sources: Lynx (reference counting); all support sharing

```typescript
// Multiple meshes share one geometry
const geo = createBoxGeometry({ width: 1, height: 1, depth: 1 })
const mesh1 = createMesh(geo, materialA)
const mesh2 = createMesh(geo, materialB)

// Explicit disposal
geo.dispose()  // destroys GPU buffers, optionally clears CPU arrays
```

GPU buffers are uploaded once and shared across all mesh instances. Reference counting ensures cleanup when the last mesh using a geometry is removed. The `dispose()` call is explicit for non-React usage; React bindings handle disposal automatically on unmount.

## BVH Integration

**Decision: Lazy BVH construction on first raycast**

- Sources: Fennec (BVH builder integrated with geometry)

BVH for triangle-level raycasting is built on the first `raycast()` call against a mesh, then cached on the geometry. CPU-side position/index arrays are retained for raycasting (not freed after GPU upload) when the geometry participates in raycasting.

Optional `retainCPUData` flag (Mantis approach) to explicitly keep or release CPU arrays.
