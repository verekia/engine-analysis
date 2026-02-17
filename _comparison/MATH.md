# Math Library Comparison

This document compares math library design and coordinate systems across all 9 engine implementations: bonobo, caracal, fennec, hyena, lynx, mantis, rabbit, shark, and wren.

## Universal Agreement

All implementations agree on the following core math principles:

### Coordinate System
**Z-up, right-handed coordinate system** is universally adopted:
```
       Z (up)
       │
       │
       └──── X (right)
      /
     /
    Y (forward)
```

- **X-axis**: Right
- **Y-axis**: Forward
- **Z-axis**: Up
- **Handedness**: Right-handed (X × Y = Z)

This matches Blender, Unreal Engine, and many CAD tools. Differs from Three.js which uses Y-up.

### Float32Array Backing
All implementations use `Float32Array` for all math types:
- Compatible with WebGL/WebGPU uniform uploads
- No conversion overhead
- Better cache locality than object properties
- SIMD-friendly (though not currently used)

### Column-Major Matrices
All implementations use column-major 4×4 matrix layout, matching WebGL/WebGPU conventions:

```
Memory layout:
[m0  m1  m2  m3    m4  m5  m6  m7    m8  m9  m10 m11   m12 m13 m14 m15]
 col0              col1              col2              col3 (translation)

Logical view:
┌────────────────────────┐
│ m0  m4  m8   m12 │  Row 0
│ m1  m5  m9   m13 │  Row 1
│ m2  m6  m10  m14 │  Row 2
│ m3  m7  m11  m15 │  Row 3
└────────────────────────┘

Translation: [m12, m13, m14]
```

### Zero-Allocation API
All implementations provide output parameter style for hot paths:
```typescript
// Allocation-free (hot path)
vec3Add(out, a, b)
mat4Multiply(out, a, b)

// Some also provide convenience forms (setup code)
const result = Vec3.add(a, b)  // allocates
```

### Quaternions for Rotations
All implementations prefer quaternions over Euler angles:
- Avoids gimbal lock
- Better for interpolation (slerp)
- More compact than matrices
- Order: `[x, y, z, w]` (glTF convention)

## Key Variations

### 1. API Style

**Pure Functional (5 implementations)**
- **Bonobo, Hyena, Lynx, Mantis, Wren** use standalone functions only
- Example: `vec3Add(out, a, b)`, `mat4Multiply(out, a, b)`
- Maximum tree-shaking
- No classes, no `this` binding

**Functional + OOP Wrappers (3 implementations)**
- **Fennec, Caracal, Rabbit** provide both APIs
- Core is functional, but offer class wrappers for convenience
- Example:
  ```typescript
  // Functional core
  vec3.add(out, a, b)

  // OOP wrapper
  class Vec3 {
    add(other: Vec3): Vec3 { ... }
  }
  ```

**Not Specified (1 implementation)**
- **Shark** doesn't detail math API style

### 2. Type System

**Type Aliases (5 implementations)**
- **Hyena, Lynx, Mantis, Rabbit, Wren** use type aliases:
  ```typescript
  type Vec3 = Float32Array
  type Mat4 = Float32Array
  type Quat = Float32Array
  ```
- Zero overhead
- Direct typed array manipulation

**Wrapper Classes (3 implementations)**
- **Fennec, Caracal, Bonobo** use classes:
  ```typescript
  class Vec3 {
    readonly data: Float32Array
    get x(): number { return this.data[0] }
    set x(v: number) { this.data[0] = v }
  }
  ```
- Better IDE autocomplete
- Can attach methods
- Can wire change callbacks (for dirty flags)

**Not Specified (1 implementation)**
- **Shark** doesn't detail type system

### 3. Scratch/Pool System

**Module-Level Scratch Variables (6 implementations)**
- **Bonobo, Fennec, Caracal, Hyena, Rabbit, Wren** use pre-allocated scratch:
  ```typescript
  const _tempVec3A = new Float32Array(3)
  const _tempVec3B = new Float32Array(3)
  const _tempMat4 = new Float32Array(16)
  ```
- Used for intermediate calculations
- Never allocated in hot paths

**Frame-Scoped Pool with Reset (3 implementations)**
- **Hyena, Mantis, Lynx** use pools that reset each frame:
  ```typescript
  const pool = {
    vec3: Array.from({ length: 32 }, () => new Float32Array(3)),
    index: 0,
    acquire: () => pool.vec3[pool.index++],
    reset: () => { pool.index = 0 }
  }
  ```
- Acquire temporaries during frame
- Reset pointer at frame start
- Zero cost "deallocation"

**Both Approaches (3 implementations)**
- **Hyena, Lynx, Mantis** mention both scratch and pools

### 4. Library Size

**Minimal (5 implementations)**
- **Bonobo, Hyena, Lynx, Mantis, Wren** implement only what's needed
- No unused operations
- Maximum tree-shaking potential
- Typical: 20-40 vector ops, 15-25 matrix ops

**Comprehensive (2 implementations)**
- **Fennec, Caracal** provide more complete APIs
- Include less-common operations
- More convenient but larger bundle

**Not Specified (2 implementations)**
- **Rabbit, Shark** don't detail scope

### 5. Dependency Strategy

**Custom Math Library (8 implementations)**
- **Bonobo, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren** implement their own
- Reasons cited:
  - Avoid dependency weight
  - Z-up coordinate system baked in
  - Control over allocation patterns
  - Impedance mismatch with gl-matrix

**Not Specified (1 implementation)**
- **Caracal** doesn't specify if custom or dependency

### 6. GLTF Y-up Conversion

**Explicit Conversion Mentioned (3 implementations)**
- **Fennec, Caracal, Wren** explicitly convert GLTF Y-up to Z-up on import
- Swap Y and Z axes
- Adjust rotations accordingly

**Implied by Coordinate System (6 implementations)**
- **Bonobo, Hyena, Lynx, Mantis, Rabbit, Shark** mention Z-up but don't detail conversion

## Implementation Breakdown

### Core Vector Operations (Present in All)

All implementations provide these Vec3 operations:
- `create`, `set`, `copy`
- `add`, `subtract`, `scale`, `multiply` (component-wise)
- `dot`, `cross`
- `length`, `lengthSquared`, `distance`
- `normalize`, `lerp`
- `transformMat4` (position - includes translation)
- `transformMat4Dir` or equivalent (direction - ignores translation)

### Core Matrix Operations (Present in All)

All implementations provide these Mat4 operations:
- `create`, `identity`, `copy`
- `multiply`, `invert`, `transpose`
- `fromTRS` or `compose` (create from translation, rotation, scale)
- `perspective`, `ortho` (projection matrices)
- `lookAt` (view matrix, Z-up adapted)
- `getTranslation`, `getScaling`, `getRotation` (decomposition)

### Core Quaternion Operations (Present in All)

All implementations provide these Quat operations:
- `create`, `identity`
- `multiply`, `normalize`, `invert`/`conjugate`
- `fromEuler`, `fromAxisAngle`
- `slerp` (spherical linear interpolation)

### AABB Operations

**Explicit AABB Type (8 implementations)**
- **Bonobo, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren** have AABB utilities
- Stored as: `Float32Array(6)` = `[minX, minY, minZ, maxX, maxY, maxZ]`
- Or: `{ min: Vec3, max: Vec3 }`

Common operations:
- `create`, `fromPoints`
- `expandByPoint`, `union`
- `intersects`, `contains`
- `surfaceArea` (for BVH SAH)
- `transform` (by mat4)

**Not Specified (1 implementation)**
- **Caracal** minimal mention

### Frustum Operations

**Dedicated Frustum Type (7 implementations)**
- **Bonobo, Fennec, Hyena, Lynx, Mantis, Rabbit, Wren** provide frustum utilities
- Stored as: `Float32Array(24)` = 6 planes × 4 floats `(nx, ny, nz, d)`
- Plane equation: `nx*x + ny*y + nz*z + d = 0`

Operations:
- `extractFromViewProjection` (Gribb-Hartmann method)
- `intersectsAABB` or `containsAABB`
- `containsPoint`

**Not Specified (2 implementations)**
- **Caracal, Shark** don't detail frustum utilities

### Ray Operations

**Dedicated Ray Type (6 implementations)**
- **Fennec, Hyena, Lynx, Mantis, Rabbit, Wren** provide ray utilities
- Stored as: `{ origin: Vec3, direction: Vec3 }`

Operations:
- `create`, `at` (get point at distance t)
- `intersectAABB`, `intersectTriangle`, `intersectSphere`
- `transformMat4`

**Not Specified (3 implementations)**
- **Bonobo, Caracal, Shark** don't detail ray utilities

## Specific Differences

### Perspective Matrix

**Standard Approach (6 implementations)**
- **Bonobo, Fennec, Hyena, Lynx, Rabbit, Wren** use standard perspective formula
- Adapted for Z-up (camera looks along -Y or configurable)

**Depth Range Variant (2 implementations)**
- **Mantis, Lynx** support both [0,1] (WebGPU) and [-1,1] (WebGL2):
  ```typescript
  mat4Perspective(out, fov, aspect, near, far, clipDepthZeroToOne: boolean)
  ```

**Not Specified (1 implementation)**
- **Caracal** doesn't detail

### Euler Angle Order

**ZXY Order (2 implementations)**
- **Bonobo** explicitly uses ZXY rotation order
- Matches Blender default

**ZYX Order (2 implementations)**
- **Mantis** uses ZYX (yaw-pitch-roll for Z-up)

**Configurable (2 implementations)**
- **Fennec, Caracal** allow order parameter (default varies)

**Not Specified (3 implementations)**
- **Hyena, Lynx, Rabbit, Shark, Wren** don't specify order

### Spherical Coordinates

**Explicit Z-up Spherical (3 implementations)**
- **Fennec, Rabbit, Caracal** provide spherical↔cartesian for orbit controls
- Adapted for Z-up:
  ```typescript
  x = distance * cosElev * sin(azimuth)
  y = distance * cosElev * cos(azimuth)
  z = distance * sin(elevation)
  ```

**Not Specified (6 implementations)**
- Others likely implement in orbit controls but don't document in math

## Constants

Most implementations provide standard vector constants:

```typescript
VEC3_ZERO    = [0, 0, 0]
VEC3_ONE     = [1, 1, 1]
VEC3_UP      = [0, 0, 1]  // Z-up!
VEC3_RIGHT   = [1, 0, 0]
VEC3_FORWARD = [0, 1, 0]
```

## Testing

**Explicit Testing Mentioned (2 implementations)**
- **Bonobo, Fennec** mention unit tests for all math functions
- Validate against known values
- Compare with reference implementations (gl-matrix)
- Edge case testing (NaN, Inf, zero vectors)

**Not Mentioned (7 implementations)**
- Others likely test but don't document

## Performance Notes

All implementations that discuss performance agree:
- Float32Array is fast enough (SIMD not needed currently)
- Scalar JS math is ~1ms for 5000 matrix computations
- No allocation in hot paths is critical
- Cache-friendly layout (SoA where applicable)

**SIMD Future Work Mentioned (3 implementations)**
- **Bonobo, Fennec, Rabbit** mention potential WASM or native SIMD future work
- Not currently implemented (browser support patchy)

## Summary Table

| Feature | Bonobo | Caracal | Fennec | Hyena | Lynx | Mantis | Rabbit | Shark | Wren |
|---------|--------|---------|--------|-------|------|--------|--------|-------|------|
| **Coord System** | Z-up RH | Z-up RH | Z-up RH | Z-up RH | Z-up RH | Z-up RH | Z-up RH | Z-up RH | Z-up RH |
| **API Style** | Functional | Hybrid | Hybrid | Functional | Functional | Functional | Hybrid | - | Functional |
| **Types** | Classes | - | Classes | Aliases | Aliases | Aliases | Aliases | - | Aliases |
| **Scratch Pool** | Module | Module | Module | Pool+Module | Pool+Module | Pool | Module | - | Module |
| **Library** | Custom | Custom | Custom | Custom | Custom | Custom | Custom | Custom | Custom |
| **AABB** | Yes | Minimal | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| **Frustum** | Yes | - | Yes | Yes | Yes | Yes | Yes | - | Yes |
| **Ray** | - | - | Yes | Yes | Yes | Yes | Yes | - | Yes |
| **Column-Major** | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes |

## Recommendations for Cherry-Picking

**For Maximum Performance:**
- Use **type aliases** over classes (5/9 prefer this)
- Implement **frame-scoped pool** with reset (Hyena, Mantis, Lynx approach)
- Pure **functional API** for tree-shaking (5/9 use this)
- **Minimal library** with only needed operations

**For Developer Experience:**
- Provide **OOP wrappers** for convenience (Fennec, Caracal, Rabbit approach)
- Use classes with **change callbacks** for dirty flag integration (Fennec, Hyena)
- Offer both allocation-free and allocating variants

**For Correctness:**
- Use **quaternions** for all rotations (universal agreement)
- Implement **unit tests** for all operations (Bonobo, Fennec approach)
- Validate against known reference implementations

**Z-up Specific:**
- Bake Z-up into all operations (universal agreement)
- Document **GLTF Y-up conversion** strategy (Fennec, Caracal, Wren)
- Provide Z-up spherical coordinate helpers for orbit controls

**Essential Operations:**
All implementations agree these are core:
- Vec3: create, add, sub, scale, dot, cross, normalize, lerp, transformMat4
- Mat4: identity, multiply, invert, fromTRS, perspective, lookAt
- Quat: identity, multiply, slerp, fromEuler, fromAxisAngle
- AABB: create, union, intersects, transform
- Frustum: extractFromVP, intersectsAABB
