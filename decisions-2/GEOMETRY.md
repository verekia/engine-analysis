# GEOMETRY.md - Decisions

## Decision: Separate Buffers, Options Object API, AABB Bounding Volumes

### Vertex Buffer Layout: Separate Buffers

**Chosen**: One GPU buffer per attribute (4/9: Lynx, Mantis, Rabbit, Wren)
**Rejected**: Interleaved single buffer (3/9: Caracal, Fennec, Hyena) - slightly better cache behavior for static geometry but harder to modify individual attributes
**Rejected**: Hybrid (Shark) - adds complexity for marginal benefit

Separate buffers are the pragmatic choice because:
- Easier to add/remove attributes (e.g., `_materialindex` is optional)
- Simpler partial updates (morph targets, procedural deformation)
- Negligible performance difference on modern GPUs with vertex pulling
- One extra `vertexAttribPointer` call per attribute per VAO creation (amortized away by VAO caching)

```typescript
// Buffer 0: positions (Float32Array, 3 components)
// Buffer 1: normals (Float32Array, 3 components)
// Buffer 2: uvs (Float32Array, 2 components)
// Buffer 3: colors (Float32Array, 4 components) [optional]
// Buffer 4: materialIndex (Uint8Array, 1 component) [optional]
// Buffer 5: joints (Uint8Array, 4 components) [optional, skinning]
// Buffer 6: weights (Float32Array, 4 components) [optional, skinning]
// Index buffer: Uint16Array or Uint32Array
```

### Standard Vertex Attributes

Universal agreement across all 9:

| Attribute | Type | Components | Required |
|-----------|------|-----------|----------|
| position | Float32Array | 3 | Yes |
| normal | Float32Array | 3 | Yes |
| uv | Float32Array | 2 | Yes |
| color | Float32Array | 4 | Optional |
| _materialindex | Uint8Array | 1 | Optional |
| joints | Uint8Array | 4 | Skinning only |
| weights | Float32Array | 4 | Skinning only |

### Index Format Selection

Universal agreement:

```typescript
const indexFormat = vertexCount <= 65535 ? 'uint16' : 'uint32'
```

Uint16 uses half the memory and gives better vertex cache hit rates on mobile GPUs. Sufficient for 99% of game assets.

### Bounding Volume: AABB

**Chosen**: AABB per geometry (8/9 use AABB)
**Rejected**: Bounding sphere (Bonobo) - AABB gives tighter fits for non-spherical objects, more effective frustum culling

Computed once at geometry creation from vertex positions:

```typescript
const aabb = { min: [Infinity, Infinity, Infinity], max: [-Infinity, -Infinity, -Infinity] }
for (let i = 0; i < positions.length; i += 3) {
  aabb.min[0] = Math.min(aabb.min[0], positions[i])
  // ... etc
}
```

### Parametric Primitives: Options Object API

**Chosen**: Options object pattern (universal agreement across all 9)

All 7 required primitives with defaults:

**Plane**:
```typescript
createPlaneGeometry({ width?: 1, height?: 1, widthSegments?: 1, heightSegments?: 1 })
```
- Lies in XY plane, normal points +Z

**Box**:
```typescript
createBoxGeometry({ width?: 1, height?: 1, depth?: 1, widthSegments?: 1, heightSegments?: 1, depthSegments?: 1 })
```
- 6 faces, separate vertices per face (flat normals on edges)

**Sphere**:
```typescript
createSphereGeometry({ radius?: 1, widthSegments?: 32, heightSegments?: 16 })
```
- UV sphere, poles at +/-Z, smooth normals

**Cone**:
```typescript
createConeGeometry({ radius?: 1, height?: 1, radialSegments?: 32, heightSegments?: 1, openEnded?: false })
```
- Base at Z=0, tip at Z=height

**Cylinder**:
```typescript
createCylinderGeometry({ radiusTop?: 1, radiusBottom?: 1, height?: 1, radialSegments?: 32, heightSegments?: 1, openEnded?: false })
```
- Extends along Z axis

**Capsule**:
```typescript
createCapsuleGeometry({ radius?: 0.5, height?: 1, capSegments?: 8, radialSegments?: 32 })
```
- Cylinder + two hemispherical caps, total height = height, cylinder portion = height - 2*radius

**Circle**:
```typescript
createCircleGeometry({ radius?: 1, segments?: 32 })
```
- Flat disc in XY plane, normal +Z, triangle fan from center

### Z-Up Consistency

All primitives generated in Z-up, right-handed coordinates (universal agreement):
- Plane in XY, normal +Z
- Cylinder/Cone/Capsule along Z axis
- Sphere poles at +/-Z
- Box aligned to XYZ axes

### GPU Upload: Lazy

Universal agreement: defer GPU buffer creation until first render.

- CPU-side typed arrays created immediately at geometry creation
- GPU buffers created on first draw call
- Avoids wasting GPU memory for unused geometry
- Geometry flagged as needing upload; upload happens in the render prepare phase

### Custom Geometry

Factory function for user-provided data:

```typescript
const geometry = createGeometry({
  positions: new Float32Array([...]),
  normals: new Float32Array([...]),
  uvs: new Float32Array([...]),
  indices: new Uint16Array([...]),
  // Optional:
  colors: new Float32Array([...]),
  materialIndex: new Uint8Array([...]),
})
```

### Geometry Sharing

Multiple meshes can reference the same geometry instance. GPU buffers are uploaded once and shared. Geometry stays alive as long as at least one mesh references it.

### Disposal

```typescript
geometry.dispose() // Destroys GPU buffers, optionally clears CPU arrays
```

CPU arrays can optionally be retained for raycasting (Mantis approach: `retainCPU` flag). By default, CPU arrays are kept since BVH construction needs them.

### Normal Generation

- **Smooth normals** for curved surfaces (sphere, cylinder, capsule, cone sides)
- **Flat normals** for box faces (separate vertices per face)
- Cone slant normal: `normal = [cos(phi) * cos(slant), sin(phi) * cos(slant), sin(slant)]`
