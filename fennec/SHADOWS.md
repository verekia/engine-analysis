# Shadows — Cascaded Shadow Maps

## Overview

Fennec implements **Cascaded Shadow Maps (CSM)** for the single directional light that casts shadows. CSM splits the camera frustum into multiple depth ranges (cascades), each rendered to its own region of a shadow atlas. This provides smooth, high-resolution shadows across the entire 200m game world without requiring enormous shadow map resolutions.

## Why CSM

For a 200x200m low-poly world, a single shadow map would need to be extremely high-resolution to avoid blocky shadows near the camera. CSM solves this by allocating more shadow map resolution to nearby geometry and less to distant geometry:

| Approach | Resolution Needed | Quality |
|----------|------------------|---------|
| Single shadow map (200m range) | 8192x8192 | Still blocky near camera |
| CSM 3 cascades | 3x 1024x1024 | Sharp near, acceptable far |
| CSM 3 cascades | 3x 2048x2048 | Sharp everywhere |

## Architecture

### Cascade Configuration

```typescript
interface ShadowConfig {
  enabled: boolean              // Default: true
  cascades: number              // Number of cascades (default: 3, max: 4)
  mapSize: number               // Shadow map size per cascade (default: 1024)
  bias: number                  // Depth bias (default: 0.005)
  normalBias: number            // Normal-based bias (default: 0.02)
  near: number                  // Override camera near for shadow (default: camera.near)
  far: number                   // Max shadow distance (default: 200)
  lambda: number                // Log/linear split ratio (default: 0.5)
  pcfRadius: number             // PCF kernel radius (default: 1)
}
```

### Shadow Atlas Layout

All cascades are packed into a single 2D texture atlas to minimize texture binds:

```
For 3 cascades at 1024x1024:
Atlas = 3072 x 1024

┌──────────┬──────────┬──────────┐
│ Cascade 0│ Cascade 1│ Cascade 2│
│ 0-10m    │ 10-50m   │ 50-200m  │
│ 1024x1024│ 1024x1024│ 1024x1024│
└──────────┴──────────┴──────────┘
```

### Cascade Split Scheme

Splits are computed using a logarithmic/linear blend (controlled by `lambda`):

```typescript
const computeCascadeSplits = (near: number, far: number, cascadeCount: number, lambda: number): number[] => {
  const splits: number[] = []
  const range = far - near
  const ratio = far / near

  for (let i = 0; i < cascadeCount; i++) {
    const p = (i + 1) / cascadeCount

    // Logarithmic split
    const log = near * Math.pow(ratio, p)
    // Linear split
    const linear = near + range * p
    // Blend
    const d = lambda * log + (1 - lambda) * linear

    splits.push(d)
  }

  return splits // e.g., [10, 50, 200] for 3 cascades
}
```

With `lambda = 0.5`, `near = 0.1`, `far = 200`:
- Cascade 0: 0.1m – ~8m (near detail)
- Cascade 1: ~8m – ~45m (mid range)
- Cascade 2: ~45m – 200m (far range)

## Shadow Map Rendering

### Per-Cascade Light Matrix

For each cascade, compute a tight orthographic projection that fits the cascade's frustum slice:

```typescript
const computeCascadeLightMatrix = (
  camera: Camera,
  light: DirectionalLight,
  nearSplit: number,
  farSplit: number,
): Mat4 => {
  // 1. Compute the frustum corners for this cascade slice
  const corners = getFrustumCornersWorldSpace(camera, nearSplit, farSplit)
  // Returns 8 Vec3 corners in world space

  // 2. Compute the center of the frustum slice
  const center = vec3.create()
  for (const corner of corners) {
    vec3.add(center, center, corner)
  }
  vec3.scale(center, center, 1 / 8)

  // 3. Build light view matrix (looking along light direction)
  const lightDir = vec3.normalize(vec3.create(), light.direction)
  const lightUp = Math.abs(lightDir[2]) > 0.99
    ? vec3.fromValues(0, 1, 0)
    : vec3.fromValues(0, 0, 1)
  const lightView = mat4.lookAt(mat4.create(),
    vec3.sub(vec3.create(), center, vec3.scale(vec3.create(), lightDir, 100)),
    center,
    lightUp,
  )

  // 4. Transform frustum corners to light space
  let minX = Infinity, maxX = -Infinity
  let minY = Infinity, maxY = -Infinity
  let minZ = Infinity, maxZ = -Infinity
  for (const corner of corners) {
    const lc = vec3.transformMat4(vec3.create(), corner, lightView)
    minX = Math.min(minX, lc[0]); maxX = Math.max(maxX, lc[0])
    minY = Math.min(minY, lc[1]); maxY = Math.max(maxY, lc[1])
    minZ = Math.min(minZ, lc[2]); maxZ = Math.max(maxZ, lc[2])
  }

  // 5. Extend Z range to catch shadow casters behind the camera
  minZ -= 100 // Generous extension for tall objects behind camera

  // 6. Build orthographic projection
  const lightProjection = mat4.ortho(mat4.create(), minX, maxX, minY, maxY, minZ, maxZ)

  // 7. Stabilize to reduce shadow shimmer on camera movement
  const lightVP = mat4.multiply(mat4.create(), lightProjection, lightView)
  stabilizeShadowMatrix(lightVP, mapSize)

  return lightVP
}
```

### Shadow Stabilization

To prevent shadow edges from shimmering as the camera moves, snap the shadow matrix to texel-sized increments:

```typescript
const stabilizeShadowMatrix = (lightVP: Mat4, mapSize: number) => {
  // Transform the origin to shadow map space
  const shadowOrigin = vec3.transformMat4(vec3.create(), [0, 0, 0], lightVP)

  // Scale to texel units
  const texelX = shadowOrigin[0] * mapSize * 0.5
  const texelY = shadowOrigin[1] * mapSize * 0.5

  // Round to nearest texel
  const roundedX = Math.round(texelX)
  const roundedY = Math.round(texelY)

  // Apply offset to snap to texel grid
  const offsetX = (roundedX - texelX) / (mapSize * 0.5)
  const offsetY = (roundedY - texelY) / (mapSize * 0.5)

  lightVP[12] += offsetX
  lightVP[13] += offsetY
}
```

### Depth-Only Render Pass

Shadow map rendering uses a minimal depth-only shader with no fragment output:

```glsl
// Vertex shader (shadow pass)
layout(std140) uniform ShadowUniforms {
  mat4 lightViewProjection;
};

void main() {
  vec4 worldPos = objectMatrix * vec4(a_position, 1.0);
  gl_Position = lightViewProjection * worldPos;
}

// Fragment shader (shadow pass) — WebGL2
// Empty or just writes depth (which happens automatically)
void main() {
  // No-op: depth is written by the rasterizer
}
```

For skinned meshes, the shadow vertex shader also applies GPU skinning.

### Per-Cascade Culling

Objects are culled per-cascade. An object might be visible in cascade 1 but not cascade 0. This avoids rendering distant objects into the near cascade:

```typescript
const renderShadowMaps = (light: DirectionalLight, meshes: Mesh[], camera: Camera, config: ShadowConfig) => {
  const splits = computeCascadeSplits(config.near, config.far, config.cascades, config.lambda)

  for (let i = 0; i < config.cascades; i++) {
    const nearSplit = i === 0 ? config.near : splits[i - 1]
    const farSplit = splits[i]

    // Compute light VP for this cascade
    const lightVP = computeCascadeLightMatrix(camera, light, nearSplit, farSplit)
    cascadeLightVPs[i] = lightVP
    cascadeSplitDepths[i] = farSplit

    // Extract frustum from lightVP for cascade-specific culling
    const cascadeFrustum = extractFrustumPlanes(lightVP)

    // Cull meshes for this cascade
    const visibleCount = frustumCull(meshes, cascadeFrustum, _shadowVisible)

    // Render to atlas region
    const viewport = { x: i * config.mapSize, y: 0, w: config.mapSize, h: config.mapSize }
    backend.beginRenderPass(shadowAtlas, { viewport, clearDepth: 1.0 })

    for (let j = 0; j < visibleCount; j++) {
      const mesh = _shadowVisible[j]
      if (!mesh.castShadow) continue
      bindShadowMaterial(mesh, lightVP)
      backend.drawIndexed(mesh.geometry.indexCount, 1, 0)
    }

    backend.endRenderPass()
  }
}
```

## Shadow Sampling (Main Pass)

### Cascade Selection

In the main fragment shader, determine which cascade to sample by comparing the fragment's view-space depth to the cascade split distances:

```glsl
uniform float cascadeSplits[3];    // View-space Z of each split
uniform mat4 cascadeVPs[3];        // Light VP for each cascade
uniform sampler2D shadowAtlas;
uniform float shadowMapSize;
uniform float shadowBias;
uniform float shadowNormalBias;

int getCascadeIndex(float viewZ) {
  for (int i = 0; i < 3; i++) {
    if (viewZ < cascadeSplits[i]) return i;
  }
  return 2; // Furthest cascade
}
```

### PCF Soft Shadows

Percentage-Closer Filtering samples multiple shadow map texels and averages the results for soft edges:

```glsl
float sampleShadow(vec3 worldPos, vec3 worldNormal, float viewZ) {
  int cascade = getCascadeIndex(viewZ);

  // Apply normal bias (push along normal to reduce shadow acne)
  vec3 biasedPos = worldPos + worldNormal * shadowNormalBias;

  // Project into shadow map space
  vec4 shadowCoord = cascadeVPs[cascade] * vec4(biasedPos, 1.0);
  shadowCoord.xyz /= shadowCoord.w;
  shadowCoord.xy = shadowCoord.xy * 0.5 + 0.5;

  // Offset into atlas (cascade 0 at x=0, cascade 1 at x=1/3, etc.)
  shadowCoord.x = (shadowCoord.x + float(cascade)) / float(NUM_CASCADES);

  // Apply depth bias
  float currentDepth = shadowCoord.z - shadowBias;

  // 3x3 PCF kernel
  float shadow = 0.0;
  float texelSize = 1.0 / (shadowMapSize * float(NUM_CASCADES));

  for (int x = -1; x <= 1; x++) {
    for (int y = -1; y <= 1; y++) {
      float pcfDepth = texture(shadowAtlas, shadowCoord.xy + vec2(x, y) * texelSize).r;
      shadow += currentDepth > pcfDepth ? 0.0 : 1.0;
    }
  }
  shadow /= 9.0;

  return shadow;
}
```

### WebGPU Shadow Sampling

WebGPU has native `textureSampleCompare` which does hardware PCF:

```wgsl
@group(1) @binding(2) var shadowMap: texture_depth_2d;
@group(1) @binding(3) var shadowSampler: sampler_comparison;

fn sampleShadowWebGPU(shadowCoord: vec3f) -> f32 {
  // Hardware PCF with comparison sampler
  return textureSampleCompare(shadowMap, shadowSampler, shadowCoord.xy, shadowCoord.z);
}
```

## Cascade Debug Visualization

In debug mode, cascades can be color-coded to visualize which cascade covers which area:

```glsl
#ifdef FENNEC_DEBUG
vec3 cascadeDebugColor(int cascade) {
  if (cascade == 0) return vec3(1.0, 0.0, 0.0); // Red - near
  if (cascade == 1) return vec3(0.0, 1.0, 0.0); // Green - mid
  return vec3(0.0, 0.0, 1.0);                     // Blue - far
}
finalColor = mix(finalColor, cascadeDebugColor(cascade), 0.3);
#endif
```

## Performance Budget

| Operation | Cost | Notes |
|-----------|------|-------|
| Cascade split computation | ~0.01ms | CPU, once per frame |
| Light matrix computation | ~0.02ms | CPU, per cascade |
| Shadow caster culling | ~0.05ms | CPU, per cascade, 2000 objects |
| Shadow depth rendering | ~0.5-1ms | GPU, per cascade (depends on scene) |
| Shadow sampling (PCF 3x3) | ~0.2ms | GPU, integrated into main pass |
| **Total shadow cost** | **~1-2ms** | Within 16ms budget |

## Configuration Recommendations

| World Size | Cascades | Map Size | Far | Notes |
|------------|----------|----------|-----|-------|
| 50x50m | 2 | 1024 | 50 | Small indoor/arena |
| 200x200m | 3 | 1024 | 200 | Standard game world |
| 200x200m | 3 | 2048 | 200 | Higher quality, costs more GPU |
| 500x500m | 4 | 2048 | 500 | Large open world |
