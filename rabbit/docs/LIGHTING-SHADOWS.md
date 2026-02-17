# Lighting & Shadows

## Light Types

### Ambient Light

A constant light applied uniformly to all surfaces. No direction, no shadows.

```typescript
class AmbientLight extends Node {
  color: [number, number, number]  // default [1, 1, 1]
  intensity: number                // default 0.1
}
```

Shader contribution:

```glsl
vec3 ambient = u_ambientColor * u_ambientIntensity;
```

### Directional Light

An infinitely distant light source with a fixed direction (like the sun). Supports shadow casting via cascaded shadow maps.

```typescript
class DirectionalLight extends Node {
  color: [number, number, number]  // default [1, 1, 1]
  intensity: number                // default 1.0

  // Shadow config
  castShadow: boolean              // default false
  shadowMapSize: number            // default 2048 (per cascade)
  shadowCascades: number           // default 3
  shadowBias: number               // default 0.001
  shadowNormalBias: number         // default 0.02
  shadowRadius: number             // PCF kernel radius, default 1.0
}
```

The light direction is the node's forward vector (derived from rotation). By convention, the forward direction in Z-up is `-Y` (north), so a light pointing "down and south" would be:

```typescript
const sun = new DirectionalLight()
sun.setRotationFromEuler(
  -Math.PI / 4,  // pitch: 45° downward
  0,              // roll
  Math.PI / 6,   // yaw: 30° from north
)
```

Shader contribution (Lambert):

```glsl
vec3 lightDir = normalize(-u_lightDirection); // toward light
float NdotL = max(dot(normal, lightDir), 0.0);
vec3 diffuse = baseColor * u_lightColor * u_lightIntensity * NdotL;
```

## Cascaded Shadow Maps (CSM)

### Why Cascades?

A single shadow map covering a 200×200m world at reasonable quality requires enormous resolution (8K+ textures). Shadow map resolution is wasted on distant areas where the player can't see fine detail.

CSM splits the camera frustum into distance-based slices (cascades), each with its own shadow map of equal resolution. Near cascades cover less world space but at higher effective resolution.

### Cascade Configuration

For a 200×200m low-poly world with `near = 0.1`, `far = 300`:

| Cascade | Range | Shadow Map | Effective Texels/m |
|---------|-------|------------|--------------------|
| 0 | 0.1 – 15m | 2048×2048 | ~137 |
| 1 | 15 – 60m | 2048×2048 | ~34 |
| 2 | 60 – 300m | 2048×2048 | ~8.5 |

The cascade split distances follow an exponential distribution blended with a linear one (practical split scheme, same as most engines):

```typescript
const computeCascadeSplits = (
  cascadeCount: number,
  near: number,
  far: number,
  lambda: number,  // 0 = linear, 1 = logarithmic, typical 0.5-0.8
): number[] => {
  const splits: number[] = []
  for (let i = 1; i <= cascadeCount; i++) {
    const t = i / cascadeCount
    const log = near * Math.pow(far / near, t)
    const lin = near + (far - near) * t
    splits.push(lambda * log + (1 - lambda) * lin)
  }
  return splits
}
```

### Shadow Map Atlas

Instead of separate textures per cascade, all cascades are packed into a single texture atlas. This avoids texture switching during the shadow sampling phase.

```
Shadow Atlas (e.g. 4096×2048 for 3 cascades of 2048×2048):
┌──────────┬──────────┬──────────┐
│ Cascade 0│ Cascade 1│ Cascade 2│
│ 2048×2048│          │          │
└──────────┴──────────┴──────────┘

Actually, using a square atlas is more compatible:
Shadow Atlas 4096×4096:
┌──────────┬──────────┐
│ Cascade 0│ Cascade 1│
│ 2048×2048│ 2048×2048│
├──────────┼──────────┤
│ Cascade 2│ (unused) │
│ 2048×2048│          │
└──────────┴──────────┘
```

### Light View-Projection Matrix per Cascade

For each cascade:

1. Compute the frustum corners for this cascade's near/far range
2. Transform frustum corners to world space
3. Compute the AABB of the frustum corners in light space
4. Build an orthographic projection that tightly encloses this AABB
5. **Snap to texel grid** to prevent shadow shimmer when the camera moves

```typescript
const computeCascadeMatrix = (
  cascade: number,
  splitNear: number,
  splitFar: number,
  camera: Camera,
  light: DirectionalLight,
  shadowMapSize: number,
): Mat4 => {
  // 1. Compute frustum corners for this cascade slice
  const corners = computeFrustumCorners(camera, splitNear, splitFar)

  // 2. Compute center of the frustum slice
  const center = new Vec3()
  for (const corner of corners) center.add(corner)
  center.scale(1 / 8)

  // 3. Build light view matrix looking at the center from the light direction
  const lightDir = light.getWorldDirection()
  const lightView = new Mat4().lookAt(
    center.clone().addScaled(lightDir, -100),  // eye: back along light dir
    center,                                      // target: frustum center
    Vec3.Z_AXIS,                                 // up: Z (our world up)
  )

  // 4. Transform corners to light view space, compute AABB
  let minX = Infinity, maxX = -Infinity
  let minY = Infinity, maxY = -Infinity
  let minZ = Infinity, maxZ = -Infinity

  for (const corner of corners) {
    const lc = corner.clone().applyMatrix4(lightView)
    minX = Math.min(minX, lc.x); maxX = Math.max(maxX, lc.x)
    minY = Math.min(minY, lc.y); maxY = Math.max(maxY, lc.y)
    minZ = Math.min(minZ, lc.z); maxZ = Math.max(maxZ, lc.z)
  }

  // 5. Expand Z range to catch shadow casters behind the camera
  minZ -= 100  // extend backward for tall objects

  // 6. Build orthographic projection
  const lightProj = new Mat4().orthographic(minX, maxX, minY, maxY, minZ, maxZ)

  // 7. Snap to texel grid (prevents shimmer)
  const shadowMatrix = new Mat4().multiplyMatrices(lightProj, lightView)
  const texelSize = (maxX - minX) / shadowMapSize
  const origin = new Vec3(0, 0, 0).applyMatrix4(shadowMatrix)
  origin.x = Math.round(origin.x / texelSize) * texelSize
  origin.y = Math.round(origin.y / texelSize) * texelSize
  // Adjust matrix to snap
  shadowMatrix.elements[12] = origin.x
  shadowMatrix.elements[13] = origin.y

  return shadowMatrix
}
```

### Shadow Rendering Pass

```typescript
const renderShadowMaps = (
  scene: Scene,
  opaqueList: RenderItem[],
  light: DirectionalLight,
  camera: Camera,
) => {
  const cascadeMatrices: Mat4[] = []

  // Bind shadow atlas framebuffer
  beginRenderPass(shadowAtlasPass)

  for (let c = 0; c < light.shadowCascades; c++) {
    const splitNear = c === 0 ? camera.near : cascadeSplits[c - 1]
    const splitFar = cascadeSplits[c]

    // Compute light VP matrix for this cascade
    const lightVP = computeCascadeMatrix(c, splitNear, splitFar, camera, light, light.shadowMapSize)
    cascadeMatrices.push(lightVP)

    // Set viewport to this cascade's region in the atlas
    setViewport(cascadeViewports[c])

    // Render depth-only for all opaque objects
    for (const item of opaqueList) {
      // Use a simple depth-only shader (no material, no lighting)
      setPipeline(shadowPipeline)
      setBindGroup(0, { lightVP })
      setBindGroup(1, { modelMatrix: item.node.worldMatrix })
      setVertexBuffer(0, item.node.geometry.vertexBuffers.position)
      setIndexBuffer(item.node.geometry.indexBuffer)
      drawIndexed(item.node.geometry.indexCount)
    }
  }

  endRenderPass()
}
```

### Shadow Sampling (Fragment Shader)

The fragment shader selects the appropriate cascade and samples the shadow map:

```glsl
uniform mat4 u_shadowMatrices[3]; // light VP per cascade
uniform float u_cascadeSplits[3]; // split distances in view space
uniform sampler2DShadow u_shadowMap; // shadow atlas with comparison sampler

float calcShadow(vec3 worldPos, float viewDepth) {
  // Select cascade based on fragment view-space depth
  int cascade = 0;
  if (viewDepth > u_cascadeSplits[0]) cascade = 1;
  if (viewDepth > u_cascadeSplits[1]) cascade = 2;

  // Transform to shadow map space
  vec4 shadowCoord = u_shadowMatrices[cascade] * vec4(worldPos, 1.0);
  shadowCoord.xyz = shadowCoord.xyz * 0.5 + 0.5; // [-1,1] → [0,1]

  // Offset UV to the correct cascade region in the atlas
  shadowCoord.xy = shadowCoord.xy * u_cascadeScale[cascade] + u_cascadeOffset[cascade];

  // PCF filtering (3×3 kernel)
  float shadow = 0.0;
  vec2 texelSize = 1.0 / vec2(textureSize(u_shadowMap, 0));

  for (int x = -1; x <= 1; x++) {
    for (int y = -1; y <= 1; y++) {
      vec2 offset = vec2(float(x), float(y)) * texelSize * u_shadowRadius;
      shadow += texture(u_shadowMap, vec3(shadowCoord.xy + offset, shadowCoord.z - u_shadowBias));
    }
  }
  shadow /= 9.0;

  return shadow;
}
```

**WebGL2**: Uses `sampler2DShadow` with `GL_COMPARE_REF_TO_TEXTURE` for hardware PCF. The `texture()` call with a shadow sampler returns a 0/1 comparison result per sample.

**WebGPU**: Uses `textureSampleCompare()` with a comparison sampler, which provides the same hardware-accelerated depth comparison.

### Cascade Blending

To hide the transition between cascades, we blend the shadow values at cascade boundaries:

```glsl
// Optional: smooth cascade transition
float cascadeBlendRange = 2.0; // meters
if (viewDepth > u_cascadeSplits[cascade] - cascadeBlendRange && cascade < 2) {
  float nextShadow = sampleCascade(cascade + 1, worldPos);
  float blendFactor = (viewDepth - (u_cascadeSplits[cascade] - cascadeBlendRange)) / cascadeBlendRange;
  shadow = mix(shadow, nextShadow, blendFactor);
}
```

## Light Uniform Buffer

All light data is packed into a single uniform buffer (bind group 0):

```glsl
layout(std140) uniform LightData {
  vec3 u_ambientColor;
  float u_ambientIntensity;

  vec3 u_lightDirection;
  float u_lightIntensity;

  vec3 u_lightColor;
  float u_shadowBias;

  float u_shadowNormalBias;
  float u_shadowRadius;
  float u_cascadeSplits[3];
  float _pad;

  mat4 u_shadowMatrices[3];

  vec4 u_cascadeScaleOffset[3]; // xy = scale, zw = offset in atlas
};
```

This is updated once per frame, keeping light data coherent in a single buffer read.
