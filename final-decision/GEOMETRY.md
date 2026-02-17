# GEOMETRY — Final Decision

## Vertex Layout

**Decision: Separate buffers per attribute (not interleaved)** (universal agreement)

Each vertex attribute stored in its own buffer:

| Attribute | Type | Size | Required |
|-----------|------|------|----------|
| position | Float32×3 | 12 bytes | Yes |
| normal | Float32×3 | 12 bytes | Yes |
| uv | Float32×2 | 8 bytes | Optional |
| color | Float32×4 | 16 bytes | Optional |
| _materialindex | Uint8×1 | 1 byte | Optional |
| joints | Uint8×4 or Uint16×4 | 4-8 bytes | Skinned only |
| weights | Float32×4 | 16 bytes | Skinned only |

Separate buffers avoid wasted bandwidth when optional attributes are absent. The GPU only fetches the attributes that are actually used by the shader variant.

## Index Buffer

**Decision: Automatic Uint16/Uint32 selection based on vertex count** (universal agreement)

```typescript
const indexFormat = vertexCount > 65535 ? 'uint32' : 'uint16'
```

Uint16 saves memory and bandwidth for small meshes (<65K vertices). Uint32 for larger meshes.

## Bounding Volumes

**Decision: AABB as primary bounding volume** (universal agreement)

AABB computed at geometry creation from vertex positions. Used for:
- Frustum culling (6 plane tests)
- BVH construction
- Coarse raycast rejection

Optional bounding sphere available for distance-based culling.

## Parametric Primitives

**Decision: 7 parametric primitives** (universal agreement)

All primitives generated in Z-up orientation with options objects:

```typescript
const plane = createPlaneGeometry({ width: 10, height: 10, widthSegments: 1, heightSegments: 1 })
const box = createBoxGeometry({ width: 1, height: 1, depth: 1 })
const sphere = createSphereGeometry({ radius: 1, widthSegments: 32, heightSegments: 16 })
const cone = createConeGeometry({ radius: 1, height: 2, radialSegments: 32 })
const cylinder = createCylinderGeometry({ radiusTop: 1, radiusBottom: 1, height: 2, radialSegments: 32 })
const capsule = createCapsuleGeometry({ radius: 0.5, height: 2, radialSegments: 32, heightSegments: 1 })
const circle = createCircleGeometry({ radius: 1, segments: 32 })
```

All primitives:
- Center at origin
- Use Z-up convention (plane lies on XY with normal pointing +Z, cylinder/cone/capsule axis along Z)
- Generate smooth normals by default
- Include UV coordinates
- Include index buffers

## Custom Geometry

```typescript
const custom = createGeometry({
  positions: new Float32Array([...]),
  normals: new Float32Array([...]),
  uvs: new Float32Array([...]),
  indices: new Uint16Array([...]),
  colors: new Float32Array([...]),          // optional
  materialIndices: new Uint8Array([...]),   // optional
})
```

Accepts any named attribute via a generic map for extensibility.

## GPU Buffer Management

**Decision: Lazy upload on first render** (universal agreement)

Geometry vertex/index data is kept in CPU-side typed arrays until the first frame that renders the geometry. At that point, GPU buffers are created and uploaded. This avoids uploading geometry that may never be rendered.

Subsequent modifications to CPU data require an explicit `geometry.needsUpdate = true` flag to trigger re-upload.

## Geometry Sharing

Multiple meshes can reference the same geometry. Geometry tracks a reference count and is only disposed when all referencing meshes are gone (or explicitly disposed by the user).

## BVH Integration

**Decision: Lazy BVH construction on first raycast** (universal agreement)

The BVH for a geometry is not built at creation time. It is built on the first raycast that hits the geometry, then cached. This avoids wasting time building BVHs for geometry that is never raycasted.

## Disposal

```typescript
geometry.dispose()  // Release GPU buffers and CPU data
```

React bindings handle disposal automatically on unmount.
