# Cascaded Shadow Maps

## Overview

Mantis uses Cascaded Shadow Maps (CSM) for directional light shadows. CSM
splits the view frustum into multiple depth ranges (cascades), each rendered
into its own shadow map. Near objects get high-resolution shadows; far objects
get lower resolution. This is ideal for the target world size (200×200 m).

## Cascade Configuration

| Parameter | Value | Rationale |
|---|---|---|
| Cascade count | 3 | Good quality/performance balance for 200 m range |
| Shadow atlas size | 3072 × 1024 | 3 × 1024×1024 cascades side by side |
| Depth format | Depth24Plus | Standard precision, no stencil needed |
| PCF kernel | 3×3 Poisson (9 samples) | Smooth edges without excessive cost |
| Cascade blend zone | 10% of each cascade range | Eliminates visible seams |

### Cascade Splits

Splits use a practical logarithmic-linear blend (Nvidia's PSSM approach):

```
λ = 0.7  (blend factor: 0 = fully linear, 1 = fully logarithmic)

For camera near=0.1, far=200:
  Cascade 0:   0.1 –  8 m    (near range, highest detail)
  Cascade 1:   8   – 40 m    (mid range)
  Cascade 2:  40   – 200 m   (far range, lowest detail)
```

Logarithmic splits allocate more resolution to near geometry where shadow
detail is most noticeable. The linear blend prevents the near cascade from
being too small.

```typescript
const computeCascadeSplits = (near: number, far: number, cascadeCount: number, lambda: number): number[] => {
  const splits: number[] = [near]

  for (let i = 1; i < cascadeCount; i++) {
    const t = i / cascadeCount
    const log = near * Math.pow(far / near, t)
    const lin = near + (far - near) * t
    splits.push(lambda * log + (1 - lambda) * lin)
  }

  splits.push(far)
  return splits
}
```

## Shadow Atlas Layout

All three cascades are packed into a single depth texture to avoid texture
switching during the shading pass:

```
┌────────────────┬────────────────┬────────────────┐
│  Cascade 0     │  Cascade 1     │  Cascade 2     │
│  1024 × 1024   │  1024 × 1024   │  1024 × 1024   │
│  0–8 m         │  8–40 m        │  40–200 m      │
│  viewport:     │  viewport:     │  viewport:     │
│  (0,0,1024,    │  (1024,0,1024, │  (2048,0,1024, │
│   1024)        │   1024)        │   1024)        │
└────────────────┴────────────────┴────────────────┘
            3072 × 1024 depth texture
```

One bind group binding, one texture, zero switches.

## Shadow Map Rendering

### Per-Cascade Light Projection

For each cascade, compute a tight orthographic projection that encloses the
cascade's portion of the view frustum:

```typescript
const computeCascadeVP = (
  lightDir: Vec3,
  cameraVP: Mat4,
  nearSplit: number,
  farSplit: number,
): Mat4 => {
  // 1. Compute the 8 corners of the cascade frustum slice in world space
  const corners = getFrustumCorners(cameraVP, nearSplit, farSplit)

  // 2. Compute frustum center
  const center = vec3Pool.acquire()
  vec3Zero(center)
  for (const corner of corners) {
    vec3Add(center, corner, center)
  }
  vec3Scale(center, 1 / 8, center)

  // 3. Build light view matrix looking along lightDir from above the center
  const lightView = mat4Pool.acquire()
  const up = Math.abs(lightDir[2]) < 0.99 ? [0, 0, 1] : [0, 1, 0]
  mat4LookAt(center, vec3Add(center, lightDir, tempVec), up, lightView)

  // 4. Transform corners into light space, find tight AABB
  let minX = Infinity, maxX = -Infinity
  let minY = Infinity, maxY = -Infinity
  let minZ = Infinity, maxZ = -Infinity

  for (const corner of corners) {
    const ls = vec3TransformMat4(corner, lightView, tempVec)
    minX = Math.min(minX, ls[0]); maxX = Math.max(maxX, ls[0])
    minY = Math.min(minY, ls[1]); maxY = Math.max(maxY, ls[1])
    minZ = Math.min(minZ, ls[2]); maxZ = Math.max(maxZ, ls[2])
  }

  // 5. Extend Z range to catch shadow casters behind the camera
  minZ -= 100 // extend behind to catch tall objects casting forward

  // 6. Build orthographic projection
  const lightProj = mat4Pool.acquire()
  mat4Ortho(minX, maxX, minY, maxY, minZ, maxZ, lightProj)

  // 7. Combine
  const lightVP = mat4Pool.acquire()
  mat4Multiply(lightProj, 0, lightView, 0, lightVP, 0)

  return lightVP
}
```

### Texel Snapping

As the camera moves, the light's orthographic projection shifts. Without
snapping, shadow texels jitter, causing a distracting "swimming" effect.

Fix: snap the projection's origin to shadow map texel boundaries:

```typescript
const snapToTexelGrid = (lightVP: Mat4, shadowMapSize: number) => {
  // Compute world units per texel
  const worldUnitsPerTexel = 2.0 / shadowMapSize // ortho range is [-1, 1] mapped to shadowMapSize

  // Snap translation components
  lightVP[12] = Math.floor(lightVP[12] / worldUnitsPerTexel) * worldUnitsPerTexel
  lightVP[13] = Math.floor(lightVP[13] / worldUnitsPerTexel) * worldUnitsPerTexel
}
```

### Shadow Pass Execution

The shadow pass renders opaque objects into each cascade's viewport:

```typescript
const renderShadowPass = (renderer: Renderer, cascades: CascadeInfo[]) => {
  const pass = device.beginRenderPass({
    depthStencilAttachment: {
      view: shadowAtlas.createView(),
      depthLoadOp: 'clear',
      depthStoreOp: 'store',
      depthClearValue: 1.0,
    },
  })

  for (let c = 0; c < 3; c++) {
    // Set viewport for this cascade
    pass.setViewport(c * 1024, 0, 1024, 1024, 0, 1)

    // Upload cascade VP matrix
    uploadCascadeUniforms(c, cascades[c].viewProjMatrix)

    // Draw opaque objects visible in this cascade (front-to-back for early-Z)
    for (const cmd of cascades[c].drawCommands) {
      pass.setPipeline(shadowPipeline)  // depth-only shader, no fragment output
      pass.setVertexBuffer(0, cmd.vertexBuffer)
      pass.setIndexBuffer(cmd.indexBuffer, 'uint16')
      pass.setBindGroup(0, shadowFrameBindGroup)
      pass.setBindGroup(2, objectBindGroup, [cmd.ringBufferOffset])
      pass.drawIndexed(cmd.indexCount)
    }
  }

  pass.end()
}
```

**Depth-only shader:** The shadow pass uses a minimal vertex shader (just
transform position) and no fragment shader output. This is ~3× faster than
full shading.

**Front-to-back sorting:** Objects are sorted by distance to the light for
each cascade. Early-Z rejection discards occluded fragments without running
the vertex shader.

## Shadow Sampling (Fragment Shader)

### Cascade Selection

In the fragment shader, determine which cascade contains the current fragment:

```glsl
// viewSpaceDepth is the fragment's depth in camera view space
int cascadeIndex = 0;
if (viewSpaceDepth > cascadeSplits.x) cascadeIndex = 1;
if (viewSpaceDepth > cascadeSplits.y) cascadeIndex = 2;
```

### Shadow Coordinate Computation

```glsl
// Transform world position into light's clip space for the selected cascade
vec4 shadowCoord = cascadeVP[cascadeIndex] * vec4(worldPosition, 1.0);
shadowCoord.xyz = shadowCoord.xyz * 0.5 + 0.5;  // [-1,1] → [0,1]

// Offset UV to the correct cascade tile in the atlas
shadowCoord.x = (shadowCoord.x + float(cascadeIndex)) / 3.0;
```

### PCF Filtering (3×3 Poisson)

Percentage Closer Filtering samples multiple points around the shadow
coordinate and averages the results for smooth shadow edges:

```glsl
const vec2 poissonDisk[9] = vec2[](
  vec2( 0.0,  0.0),
  vec2(-0.94, -0.34), vec2( 0.94,  0.34),
  vec2(-0.34,  0.94), vec2( 0.34, -0.94),
  vec2(-0.56,  0.56), vec2( 0.56, -0.56),
  vec2(-0.56, -0.56), vec2( 0.56,  0.56)
);

float computeShadow(vec3 shadowCoord, int cascadeIndex) {
  float texelSize = 1.0 / 1024.0;  // shadow map resolution
  float shadow = 0.0;
  float bias = 0.002;  // depth bias to prevent shadow acne

  for (int i = 0; i < 9; i++) {
    vec2 offset = poissonDisk[i] * texelSize * 1.5;
    vec2 sampleUV = shadowCoord.xy + offset;

    // Atlas offset
    sampleUV.x = (sampleUV.x + float(cascadeIndex)) / 3.0;

    float closestDepth = texture(shadowMap, sampleUV).r;
    shadow += (shadowCoord.z - bias > closestDepth) ? 0.0 : 1.0;
  }

  return shadow / 9.0;
}
```

### Cascade Blending

At cascade boundaries, blend between adjacent cascades to prevent visible seams:

```glsl
float cascadeBlend(float viewDepth, float splitNear, float splitFar) {
  float blendZone = (splitFar - splitNear) * 0.1;  // 10% blend zone
  float distFromEdge = splitFar - viewDepth;
  return clamp(distFromEdge / blendZone, 0.0, 1.0);
}

// In shadow calculation:
float shadow = computeShadow(shadowCoord, cascadeIndex);

// Blend with next cascade if near boundary
if (cascadeIndex < 2) {
  float blend = cascadeBlend(viewDepth, splits[cascadeIndex], splits[cascadeIndex + 1]);
  if (blend < 1.0) {
    float nextShadow = computeShadow(nextShadowCoord, cascadeIndex + 1);
    shadow = mix(nextShadow, shadow, blend);
  }
}
```

## Shadow Bias

Shadow acne (self-shadowing artifacts) is prevented with:

1. **Constant depth bias:** Small offset added to shadow comparison depth
2. **Slope-scale bias:** Bias proportional to the surface's angle to the light
3. **Normal offset:** Offset the shadow lookup position along the surface normal

```glsl
float bias = max(0.005 * (1.0 - dot(normal, lightDir)), 0.001);
```

## Performance

| Operation | Cost |
|---|---|
| Cascade frustum computation | < 0.02 ms (CPU) |
| Shadow pass (3 cascades, 1500 objects) | ~2.5 ms (GPU, depth-only) |
| Shadow sampling (9-tap PCF) | ~0.3 ms (GPU, per full-screen pixel) |
| Total shadow overhead | ~3 ms GPU |

## Shadow API

```typescript
const dirLight = scene.createDirectionalLight({
  direction: [-0.5, -0.3, -0.8],  // pointing down and to the side
  color: [1, 0.95, 0.9],
  intensity: 1.0,
  castShadow: true,
  shadowConfig: {
    cascades: 3,              // default
    resolution: 1024,         // per cascade, default
    bias: 0.002,              // default
    normalBias: 0.02,         // default
    far: 200,                 // shadow render distance
    lambda: 0.7,              // cascade split blend factor
  },
})

// Meshes cast and receive shadows by default if material has shadows: true
const mesh = scene.createMesh(geometry, lambertMaterial)
mesh.castShadow = true       // default true
mesh.receiveShadow = true    // controlled by material.shadows
```
