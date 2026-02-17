# 07 — Lighting & Cascaded Shadow Maps

## Light Types

### Ambient Light

A flat, non-directional light that uniformly illuminates all surfaces. Set on the scene, not as a node.

```ts
interface AmbientLight {
  color: Float32Array    // [r, g, b] linear
  intensity: number      // multiplier
}

// Shader contribution:
// ambient = u_ambientColor * u_ambientIntensity
```

### Directional Light

An infinitely distant light with parallel rays. Direction is derived from the light node's rotation (points along the node's -Y axis in local space, which becomes an arbitrary world direction after rotation).

```ts
interface DirectionalLightNode extends SceneNode {
  type: 'directional_light'
  color: Float32Array    // [r, g, b] linear
  intensity: number
  castShadow: boolean
  shadow: DirectionalShadowConfig | null
}

interface DirectionalShadowConfig {
  cascades: number         // 2 or 3. Default: 3
  mapSize: number          // Pixels per cascade. Default: 1024
  near: number             // Shadow camera near. Default: 0.5
  far: number              // Shadow distance. Default: 200
  bias: number             // Depth bias. Default: 0.001
  normalBias: number       // Normal-direction bias. Default: 0.02
  splits: number[] | null  // Manual cascade split distances, or null for auto
  blendRegion: number      // Cascade blend width as fraction. Default: 0.1
}
```

## Lambert Lighting Model

The fragment shader computes diffuse lighting as:

```glsl
// Inputs
uniform vec3 u_lightDir;         // World-space light direction (normalized)
uniform vec3 u_lightColor;
uniform float u_lightIntensity;
uniform vec3 u_ambientColor;
uniform float u_ambientIntensity;

// In fragment shader
vec3 N = normalize(v_normal);
float NdotL = max(dot(N, -u_lightDir), 0.0);

vec3 ambient = u_ambientColor * u_ambientIntensity;
vec3 diffuse = u_lightColor * u_lightIntensity * NdotL;

vec3 litColor = baseColor * (ambient + diffuse * shadow);
```

## Cascaded Shadow Maps (CSM)

### Overview

CSM solves the fundamental problem of shadow maps: a single shadow map covering a large area has insufficient resolution for nearby objects. By splitting the view frustum into multiple depth ranges and assigning each a dedicated shadow map, we get high-resolution shadows near the camera and acceptable shadows far away.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    View Frustum                         │
│  ┌────────┬──────────────┬──────────────────────────┐   │
│  │Cascade0│  Cascade 1   │      Cascade 2           │   │
│  │0.5-15m │  15-60m      │      60-200m             │   │
│  │1024px  │  1024px      │      1024px              │   │
│  └────────┴──────────────┴──────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Cascade Split Computation

Using the practical split scheme (log-linear interpolation between logarithmic and uniform splits):

```ts
const computeCascadeSplits = (
  cascades: number,
  near: number,
  far: number,
  lambda: number  // 0 = uniform, 1 = logarithmic, 0.5-0.7 recommended
): number[] => {
  const splits: number[] = []
  for (let i = 1; i <= cascades; i++) {
    const fraction = i / cascades
    const log = near * Math.pow(far / near, fraction)
    const uniform = near + (far - near) * fraction
    splits.push(lambda * log + (1 - lambda) * uniform)
  }
  return splits
}
```

For a 200m world with 3 cascades and lambda=0.6:
- Split 0: 0.5m → ~12m
- Split 1: ~12m → ~55m
- Split 2: ~55m → 200m

### Light-Space Projection per Cascade

For each cascade:

1. **Compute frustum corners** of the sub-frustum in world space
2. **Find bounding sphere** of the frustum corners (sphere is rotation-invariant, preventing shimmer)
3. **Build orthographic projection** from the light's point of view, encompassing the sphere
4. **Snap to texel grid** to eliminate shadow edge shimmer when the camera moves

```ts
const computeCascadeMatrix = (
  frustumCorners: Float32Array[],  // 8 corners in world space
  lightDir: Float32Array,
  mapSize: number
): { view: Float32Array, projection: Float32Array, viewProjection: Float32Array } => {
  // 1. Compute bounding sphere
  const center = computeFrustumCenter(frustumCorners)
  let radius = 0
  for (const corner of frustumCorners) {
    radius = Math.max(radius, vec3Distance(center, corner))
  }
  radius = Math.ceil(radius)  // Round up for stability

  // 2. Build light view matrix (looking along lightDir, centered on sphere)
  const lightPos = vec3ScaleAndAdd([], center, lightDir, -radius)
  const lightView = mat4LookAt([], lightPos, center, [0, 0, 1])  // Z-up

  // 3. Build orthographic projection encompassing the sphere
  const lightProj = mat4Ortho([], -radius, radius, -radius, radius, 0, radius * 2)

  // 4. Snap to texel grid (prevents shimmer)
  const shadowMatrix = mat4Multiply([], lightProj, lightView)
  const texelSize = (radius * 2) / mapSize
  // Transform origin through shadow matrix, snap, compute offset
  const originInShadow = vec4TransformMat4([], [0, 0, 0, 1], shadowMatrix)
  originInShadow[0] = Math.floor(originInShadow[0] / texelSize) * texelSize
  originInShadow[1] = Math.floor(originInShadow[1] / texelSize) * texelSize
  // Apply offset back to projection matrix...

  return { view: lightView, projection: lightProj, viewProjection: shadowMatrix }
}
```

### Shadow Map Rendering

Each cascade renders the scene from the light's perspective into a depth-only render target:

```ts
// For each cascade
for (let c = 0; c < cascadeCount; c++) {
  const pass = encoder.beginRenderPass({
    depthStencilAttachment: {
      view: shadowMapArray[c],    // Depth texture (or texture array layer)
      depthLoadOp: 'clear',
      depthStoreOp: 'store',
      depthClearValue: 1.0,
    },
  })

  // Submit depth-only draw calls for shadow casters
  submitShadowDrawCalls(pass, visibleCasters, cascadeViewProjection[c])
  pass.end()
}
```

### Shadow Map Storage

Three approaches, in order of preference:

1. **Texture array** (WebGPU, WebGL2 with `TEXTURE_2D_ARRAY`): One texture with N layers, one per cascade. Simplest to sample in shader.
2. **Atlas**: Single large texture divided into regions (e.g., 2048x2048 for 4x 1024x1024 cascades). Works everywhere.
3. **Separate textures**: One texture per cascade. Uses more sampler slots.

Wren uses a **texture array** (supported in both WebGL2 and WebGPU).

### Shadow Sampling (Fragment Shader)

```glsl
uniform sampler2DArray u_shadowMap;
uniform mat4 u_cascadeViewProj[3];
uniform float u_cascadeSplits[3];
uniform float u_shadowBias;
uniform float u_shadowNormalBias;

float computeShadow(vec3 worldPos, vec3 worldNormal, float viewDepth) {
  // Determine which cascade this fragment falls into
  int cascade = 0;
  for (int i = 0; i < NUM_CASCADES - 1; i++) {
    if (viewDepth > u_cascadeSplits[i]) {
      cascade = i + 1;
    }
  }

  // Apply normal bias (push along normal to reduce acne)
  vec3 biasedPos = worldPos + worldNormal * u_shadowNormalBias;

  // Project into shadow map space
  vec4 shadowCoord = u_cascadeViewProj[cascade] * vec4(biasedPos, 1.0);
  shadowCoord.xyz /= shadowCoord.w;
  shadowCoord.xy = shadowCoord.xy * 0.5 + 0.5;

  // PCF 5x5 sampling
  float shadow = 0.0;
  vec2 texelSize = 1.0 / vec2(textureSize(u_shadowMap, 0).xy);

  for (int x = -2; x <= 2; x++) {
    for (int y = -2; y <= 2; y++) {
      vec2 offset = vec2(float(x), float(y)) * texelSize;
      float depth = texture(u_shadowMap, vec3(shadowCoord.xy + offset, float(cascade))).r;
      shadow += (shadowCoord.z - u_shadowBias > depth) ? 0.0 : 1.0;
    }
  }
  shadow /= 25.0;

  // Cascade blending (smooth transition between cascades)
  #ifdef CASCADE_BLEND
    float blendStart = u_cascadeSplits[cascade] * (1.0 - BLEND_REGION);
    if (cascade < NUM_CASCADES - 1 && viewDepth > blendStart) {
      float blendFactor = (viewDepth - blendStart) / (u_cascadeSplits[cascade] - blendStart);
      // Sample next cascade
      float nextShadow = sampleCascade(cascade + 1, biasedPos);
      shadow = mix(shadow, nextShadow, blendFactor);
    }
  #endif

  return shadow;
}
```

### PCF (Percentage Closer Filtering)

The 5x5 PCF kernel produces smooth shadow edges at reasonable cost. For each of the 25 samples, we compare the fragment's depth against the shadow map depth. The average of the comparison results produces soft penumbra.

On WebGPU, hardware PCF is available via `comparison` samplers:

```wgsl
@group(0) @binding(3) var u_shadowSampler: sampler_comparison;

// Single sample with hardware PCF:
let shadow = textureSampleCompare(u_shadowMap, u_shadowSampler, uv, layer, depth);
```

This is faster than manual comparison because the GPU can do the depth test during texture fetch.

### Performance Budget

| Component | Cost |
|-----------|------|
| 3 cascade renders (1024px, depth-only) | ~1.5ms GPU |
| PCF 5x5 sampling in lighting pass | ~0.3ms GPU |
| Total shadow cost | ~1.8ms GPU |

For a 200m low-poly world, this is well within budget.

## Uniform Buffer Layout

All lighting data goes into the per-frame UBO (bind group 0):

```
struct FrameUniforms {
  viewMatrix: mat4x4<f32>
  projectionMatrix: mat4x4<f32>
  viewProjectionMatrix: mat4x4<f32>
  cameraPosition: vec4<f32>        // xyz + padding
  lightDirection: vec4<f32>        // xyz + padding
  lightColor: vec4<f32>            // rgb + intensity
  ambientColor: vec4<f32>          // rgb + intensity
  cascadeViewProj: array<mat4x4<f32>, 3>
  cascadeSplits: vec4<f32>         // xyz = split depths, w = shadow distance
  shadowParams: vec4<f32>          // x = bias, y = normalBias, z = mapSize, w = blend
  time: f32
  padding: vec3<f32>
}
// Total: ~400 bytes
```
