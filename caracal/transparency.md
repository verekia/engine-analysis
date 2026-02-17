# Transparency — Weighted Blended OIT

## The Problem

Traditional transparency in real-time rendering uses the painter's algorithm: sort transparent objects back-to-front and draw with alpha blending. This breaks in many common cases:

- **Intersecting geometry**: Two transparent objects clipping through each other
- **Concave meshes**: A single transparent object that overlaps itself
- **Inconsistent sort order**: Objects sorted by centroid depth can swap when viewed from different angles
- **Per-object vs per-triangle sorting**: Three.js sorts per-object; per-triangle is too expensive

These issues cause visible popping, flickering, and incorrect rendering — the "transparency headache" of Three.js.

## The Solution: Weighted Blended OIT

Caracal uses **Weighted Blended Order-Independent Transparency** (McGuire & Bavoil, 2013). This technique:

- Renders all transparent objects in a **single pass** (no sorting needed)
- Produces **correct results** for overlapping, intersecting, and self-overlapping geometry
- Has a **fixed, low cost** regardless of the number of transparent objects
- Is **simple to implement** with standard GPU features

### Trade-offs

WBOIT is an approximation. It works well for:
- Semi-transparent surfaces (glass, water, fog)
- Particle effects
- UI elements in 3D space
- Any scene where transparent objects have similar opacity

It works less well for:
- Extreme opacity differences in overlapping layers (e.g., α=0.01 behind α=0.99)
- Cases requiring exact per-pixel ordering

For stylized/low-poly games, WBOIT is an excellent fit since most transparency use cases involve similar opacity values.

## How WBOIT Works

### Render Targets

Two additional render targets are needed:

| Target | Format | Blend Mode | Description |
|--------|--------|------------|-------------|
| `accumulation` | RGBA16F | Additive (ONE, ONE) | Weighted color accumulation |
| `revealage` | R8 | Multiplicative (ZERO, ONE_MINUS_SRC_ALPHA) | Alpha coverage product |

### Pass 1: Accumulation (Transparent Objects)

All transparent objects are rendered in any order. The fragment shader outputs to both targets:

```glsl
// oit_accumulate.frag
layout(location = 0) out vec4 accumulation;
layout(location = 1) out float revealage;

void main() {
  // Compute lit color and alpha normally
  vec4 color = computeLitColor(); // Lambert/Basic shading
  float alpha = color.a * u_opacity;

  // Skip fully transparent fragments
  if (alpha < 0.001) discard;

  // Compute depth-based weight
  float viewDepth = gl_FragCoord.z; // [0, 1] depth buffer value
  float weight = computeWeight(alpha, viewDepth);

  // Output to accumulation buffer (additive blend)
  accumulation = vec4(color.rgb * alpha * weight, alpha * weight);

  // Output to revealage buffer (multiplicative blend)
  revealage = alpha;
}
```

#### Weight Function

The weight function determines how much each layer contributes based on its depth and opacity. From the McGuire & Bavoil paper:

```glsl
float computeWeight(float alpha, float depth) {
  // Weight function emphasizes nearby transparent surfaces
  // depth is in [0, 1] range (gl_FragCoord.z)

  // Equation 10 from McGuire & Bavoil 2013
  float a = min(1.0, alpha) * 8.0 + 0.01;
  float b = -(1.0 - depth) * 0.95 + 1.0;

  // This weight function gives higher weight to:
  // - More opaque fragments (higher alpha)
  // - Nearer fragments (lower depth)
  return clamp(a * a * a * 1e8 * b * b * b, 1e-2, 3e3);
}
```

There are several weight function variants. The above provides good results for most scenes. The key requirements:
- Weight must be positive
- Weight must increase with alpha (more opaque = more weight)
- Weight must increase with nearness (closer = more weight)
- Weight should not vary too wildly (clamped to [1e-2, 3e3])

### Blend State Setup

```typescript
// Accumulation target: additive blending
{
  srcColorBlendFactor: 'one',
  dstColorBlendFactor: 'one',
  colorBlendOperation: 'add',
  srcAlphaBlendFactor: 'one',
  dstAlphaBlendFactor: 'one',
  alphaBlendOperation: 'add',
}

// Revealage target: multiplicative blending
// revealage starts at 1.0 (clear value) and each fragment multiplies by (1 - alpha)
{
  srcColorBlendFactor: 'zero',
  dstColorBlendFactor: 'one-minus-src-alpha',
  colorBlendOperation: 'add',
}
```

**Important**: The depth buffer from the opaque pass is bound as **read-only** during the transparent pass. Transparent fragments that are behind opaque geometry are discarded by the depth test, but transparent fragments don't write to the depth buffer (since they can overlap each other).

### Pass 2: Composite

A full-screen pass reads both OIT buffers and composites the result over the opaque scene:

```glsl
// oit_composite.frag
uniform sampler2D u_accumulation;
uniform sampler2D u_revealage;

void main() {
  vec4 accum = texture(u_accumulation, v_uv);
  float reveal = texture(u_revealage, v_uv).r;

  // If no transparent pixels were rendered here, skip
  if (accum.a < 0.00001) discard;

  // Reconstruct weighted average color
  vec3 averageColor = accum.rgb / max(accum.a, 0.00001);

  // revealage = product of (1 - alpha) for all layers
  // So (1 - revealage) is the total opacity
  float totalOpacity = 1.0 - reveal;

  // Blend over the opaque scene
  fragColor = vec4(averageColor, totalOpacity);
}
```

This composite pass uses standard **alpha blending** (SRC_ALPHA, ONE_MINUS_SRC_ALPHA) to draw the transparent result over the opaque scene.

## Implementation Details

### WebGL 2

Uses `drawBuffers` extension (built into WebGL 2) to write to both accumulation and revealage targets simultaneously:

```typescript
// Set up MRT (Multiple Render Targets)
gl.bindFramebuffer(gl.FRAMEBUFFER, oitFramebuffer)
gl.drawBuffers([gl.COLOR_ATTACHMENT0, gl.COLOR_ATTACHMENT1])

// Clear: accumulation to (0,0,0,0), revealage to (1,0,0,0)
gl.clearBufferfv(gl.COLOR, 0, [0, 0, 0, 0]) // accumulation
gl.clearBufferfv(gl.COLOR, 1, [1, 0, 0, 0]) // revealage starts at 1.0

// Set depth to read-only
gl.depthMask(false)

// Render all transparent objects...
```

### WebGPU

WebGPU natively supports multiple color attachments:

```typescript
const renderPassDesc: GPURenderPassDescriptor = {
  colorAttachments: [
    {
      view: accumulationView,
      clearValue: { r: 0, g: 0, b: 0, a: 0 },
      loadOp: 'clear',
      storeOp: 'store',
    },
    {
      view: revealageView,
      clearValue: { r: 1, g: 0, b: 0, a: 0 },
      loadOp: 'clear',
      storeOp: 'store',
    },
  ],
  depthStencilAttachment: {
    view: depthView,
    depthLoadOp: 'load',       // Keep opaque depth
    depthStoreOp: 'store',
    depthReadOnly: true,        // Don't write depth for transparent
  },
}
```

## Integration with Scene Graph

The material's `transparent` flag determines which pass a mesh enters:

```typescript
if (mesh.material.transparent) {
  // → OIT accumulation pass
  // → No sorting needed
  scene.transparentList.push(mesh)
} else {
  // → Opaque pass
  // → State-sorted for efficiency
  scene.opaqueList.push(mesh)
}
```

### Setting Transparency

```typescript
const glassMaterial = createBasicMaterial({
  color: [0.8, 0.9, 1.0],
  opacity: 0.3,
  transparent: true,
  side: 'both',  // Glass is visible from both sides
})

// Material index system also supports transparency
const materialIndexColors = createMaterialIndexMap({
  0: { color: [1, 1, 1], opacity: 0.5 }, // Semi-transparent white
  1: { color: [0, 0, 1], opacity: 0.8 }, // Mostly opaque blue
})
```

## Render Order

The complete render order with OIT:

```
1. Shadow pass (depth-only, shadow casters only)
2. Opaque pass (state-sorted, depth write ON)
3. OIT accumulation pass (all transparent, depth write OFF, depth test ON)
4. OIT composite pass (full-screen quad over opaque, alpha blend)
5. Post-processing (bloom, tone map)
```

This order ensures:
- Transparent objects are correctly occluded by opaque geometry (depth test in step 3)
- Transparent objects composite correctly with each other (OIT)
- Post-processing (bloom) applies to both opaque and transparent pixels
