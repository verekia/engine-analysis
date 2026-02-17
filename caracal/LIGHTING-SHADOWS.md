# Lighting & Shadow System

## Light Types

Caracal supports two light types, sufficient for most stylized/low-poly games:

### Ambient Light

A constant, directionless light that illuminates all surfaces equally. Simulates indirect/bounced light:

```typescript
const ambient = createAmbientLight({
  color: [0.3, 0.35, 0.4],   // Slightly blue-tinted ambient
  intensity: 1.0,
})
scene.add(ambient)
```

In the shader:
```glsl
vec3 ambient = u_ambientColor * u_ambientIntensity;
// Added directly to fragment color before tone mapping
```

### Directional Light

An infinitely distant light with parallel rays (like the sun). Supports shadow mapping:

```typescript
const sun = createDirectionalLight({
  color: [1.0, 0.95, 0.8],     // Warm sunlight
  intensity: 1.0,
  castShadow: true,
  shadowMapSize: 2048,          // Shadow atlas resolution
  shadowBias: 0.001,            // Depth bias to reduce acne
  shadowNormalBias: 0.02,       // Normal-direction bias
  shadowCascades: 3,            // Number of CSM cascades
  shadowDistance: 200,           // Maximum shadow distance (meters)
})

// Direction is derived from the light node's rotation
// Default: pointing down (-Z in Z-up convention)
sun.rotation.setFromEuler(-Math.PI / 4, 0, Math.PI / 6) // Angled sunlight
scene.add(sun)
```

In the shader (Lambert):
```glsl
// Directional light diffuse
float NdotL = max(dot(v_normal, u_lightDirection), 0.0);
vec3 diffuse = u_lightColor * u_lightIntensity * NdotL;

// Shadow
float shadow = receiveShadow ? sampleCSMShadow(v_worldPosition) : 1.0;

vec3 finalColor = (ambient + diffuse * shadow) * baseColor + emissive;
```

## Cascading Shadow Maps (CSM)

### Why CSM

A single shadow map for a 200×200 m world would need impossibly high resolution to avoid visible shadow texels near the camera. CSM solves this by splitting the view frustum into cascades, each with its own shadow map at appropriate resolution.

### Cascade Configuration

For a 200×200 m low-poly world with 3 cascades:

| Cascade | Near | Far | Coverage | Quality |
|---------|------|-----|----------|---------|
| 0 | 0.1 m | 15 m | Near range | Highest detail |
| 1 | 15 m | 60 m | Mid range | Medium detail |
| 2 | 60 m | 200 m | Far range | Lower detail |

### Split Scheme

Cascade splits use a practical split scheme that blends logarithmic and uniform distribution:

```typescript
const computeCascadeSplits = (
  near: number,
  far: number,
  cascadeCount: number,
  lambda: number = 0.5  // 0 = uniform, 1 = logarithmic
): number[] => {
  const splits: number[] = []

  for (let i = 1; i <= cascadeCount; i++) {
    const t = i / cascadeCount

    // Logarithmic split (better distribution for perspective)
    const log = near * Math.pow(far / near, t)

    // Uniform split
    const uni = near + (far - near) * t

    // Blend
    splits.push(lambda * log + (1 - lambda) * uni)
  }

  return splits
}
```

With `lambda = 0.5` (default), this produces splits that give more resolution near the camera while still covering the full range smoothly.

### Shadow Atlas Layout

All cascades share a single depth texture (shadow atlas) to minimize texture switches:

```
Shadow Atlas (2048 × 2048):
┌────────────────┬────────────────┐
│                │                │
│   Cascade 0    │   Cascade 1    │
│  (1024×1024)   │  (1024×1024)   │
│                │                │
├────────────────┼────────────────┤
│                │                │
│   Cascade 2    │   (unused)     │
│  (1024×1024)   │                │
│                │                │
└────────────────┴────────────────┘
```

Each cascade renders into its viewport region. The fragment shader selects the appropriate cascade based on the fragment's distance from the camera.

### Light-Space Matrix Computation

For each cascade, a light-space projection matrix is computed to tightly fit the cascade's frustum slice:

```typescript
const computeCascadeMatrix = (
  camera: Camera,
  light: DirectionalLight,
  nearSplit: number,
  farSplit: number
): Mat4 => {
  // 1. Compute the 8 corners of the frustum slice (near/far split planes)
  const frustumCorners = computeFrustumSliceCorners(camera, nearSplit, farSplit)

  // 2. Compute centroid of frustum slice
  const centroid = computeCentroid(frustumCorners)

  // 3. Build light view matrix (looking from light direction at centroid)
  const lightView = mat4LookAt(
    vec3Add(_v0, centroid, vec3Scale(_v1, light.direction, -100)),
    centroid,
    VEC3_UP // Z-up: (0, 0, 1)
  )

  // 4. Transform frustum corners to light space
  const lightSpaceCorners = frustumCorners.map(c => vec3TransformMat4(_v0, c, lightView))

  // 5. Compute tight orthographic bounds
  let minX = Infinity, maxX = -Infinity
  let minY = Infinity, maxY = -Infinity
  let minZ = Infinity, maxZ = -Infinity
  for (const c of lightSpaceCorners) {
    minX = Math.min(minX, c.x); maxX = Math.max(maxX, c.x)
    minY = Math.min(minY, c.y); maxY = Math.max(maxY, c.y)
    minZ = Math.min(minZ, c.z); maxZ = Math.max(maxZ, c.z)
  }

  // 6. Extend Z range to include shadow casters behind the frustum
  minZ -= 100 // Pull near plane back to catch casters outside view

  // 7. Build orthographic projection
  const lightProjection = mat4Ortho(minX, maxX, minY, maxY, minZ, maxZ)

  // 8. Texel snapping (see below)
  return snapToTexels(mat4Multiply(_m0, lightProjection, lightView), cascadeSize)
}
```

### Texel Snapping (Stable Cascades)

Without snapping, shadow edges shimmer as the camera moves because the light matrix shifts by sub-texel amounts. Texel snapping rounds the light matrix offset to the nearest shadow texel:

```typescript
const snapToTexels = (lightMatrix: Mat4, shadowMapSize: number): Mat4 => {
  // Transform origin to shadow map space
  const origin = vec3TransformMat4(_v0, VEC3_ZERO, lightMatrix)

  // Compute texel size
  const texelSize = 2.0 / shadowMapSize

  // Round to nearest texel
  origin.x = Math.round(origin.x / texelSize) * texelSize
  origin.y = Math.round(origin.y / texelSize) * texelSize

  // Compute offset and apply
  const dx = origin.x - vec3TransformMat4(_v1, VEC3_ZERO, lightMatrix).x
  const dy = origin.y - vec3TransformMat4(_v1, VEC3_ZERO, lightMatrix).y

  lightMatrix.elements[12] += dx
  lightMatrix.elements[13] += dy

  return lightMatrix
}
```

### Shadow Pass Rendering

Each cascade renders all shadow casters:

```typescript
const renderShadowPass = (renderer: Renderer, scene: Scene, light: DirectionalLight) => {
  const cascades = light._shadowCascades

  for (let i = 0; i < cascades.length; i++) {
    // Set viewport to cascade's atlas region
    renderer.setViewport(cascades[i].viewport)

    // Upload cascade's light matrix
    renderer.setLightMatrix(cascades[i].lightMatrix)

    // Render all shadow casters with depth-only shader
    for (const mesh of scene.shadowCasterList) {
      // Optional: coarse frustum cull against cascade frustum
      if (frustumIntersectsAABB(cascades[i].frustum, mesh._worldAABB)) {
        renderer.drawShadow(mesh)
      }
    }
  }
}
```

The shadow shader is ultra-simple — just transforms positions and writes depth:

```glsl
// Vertex shader (shadow pass)
void main() {
  gl_Position = u_lightMatrix * u_worldMatrix * vec4(a_position, 1.0);
}
// Fragment shader: empty (depth-only)
```

For skinned meshes, the shadow vertex shader includes the skinning calculation.

### Shadow Sampling (Fragment Shader)

The main fragment shader selects the cascade and samples the shadow map:

```glsl
float sampleCSMShadow(vec3 worldPos) {
  // Select cascade based on view-space depth
  float viewDepth = (u_viewMatrix * vec4(worldPos, 1.0)).z; // z in view space
  int cascade = 0;
  if (viewDepth > u_cascadeSplits[0]) cascade = 1;
  if (viewDepth > u_cascadeSplits[1]) cascade = 2;

  // Transform world position to shadow map UV
  vec4 shadowCoord = u_shadowMatrices[cascade] * vec4(worldPos, 1.0);
  shadowCoord.xyz = shadowCoord.xyz * 0.5 + 0.5; // [-1,1] → [0,1]

  // Apply atlas offset for cascade
  shadowCoord.xy = shadowCoord.xy * u_cascadeScale[cascade] + u_cascadeOffset[cascade];

  // PCF filtering
  return pcfSample(shadowCoord.xyz, cascade);
}
```

### PCF Filtering

Percentage-Closer Filtering with a rotated Poisson disk produces smooth shadow edges:

```glsl
// 5×5 PCF with Poisson disk (16 samples)
const vec2 poissonDisk[16] = vec2[](
  vec2(-0.94201624, -0.39906216),
  vec2( 0.94558609, -0.76890725),
  vec2(-0.09418410, -0.92938870),
  vec2( 0.34495938,  0.29387760),
  // ... 12 more samples
);

float pcfSample(vec3 shadowCoord, int cascade) {
  float shadow = 0.0;
  float texelSize = 1.0 / float(u_shadowMapSize);
  float filterRadius = 2.0 * texelSize; // 2 texel radius

  for (int i = 0; i < 16; i++) {
    vec2 offset = poissonDisk[i] * filterRadius;
    float depth = texture(u_shadowMap, shadowCoord.xy + offset).r;
    shadow += step(shadowCoord.z - u_shadowBias, depth);
  }

  return shadow / 16.0;
}
```

### Shadow Bias

Two bias techniques prevent shadow acne:

1. **Depth bias**: Offsets the comparison depth by a small constant
2. **Normal bias**: Offsets the world position along the surface normal before shadow lookup

```glsl
vec3 biasedWorldPos = worldPos + v_normal * u_shadowNormalBias;
// Then use biasedWorldPos for shadow coordinate computation
```

The normal bias is more effective than depth bias alone because it scales with surface orientation relative to the light.

## Lighting Uniforms (UBO)

All lighting data is packed into a single UBO updated once per frame:

```
LightsUBO (std140 layout):
  offset 0:   vec4 directionalDirection   (xyz = direction, w = padding)
  offset 16:  vec4 directionalColor       (xyz = color × intensity, w = intensity)
  offset 32:  vec4 ambientColor           (xyz = color × intensity, w = intensity)
  offset 48:  mat4 shadowMatrix0          (cascade 0 light-view-projection)
  offset 112: mat4 shadowMatrix1          (cascade 1)
  offset 176: mat4 shadowMatrix2          (cascade 2)
  offset 240: vec4 cascadeSplits          (xyz = split distances, w = shadowDistance)
  offset 256: vec4 cascadeScaleOffset0    (xy = scale, zw = offset — atlas coords)
  offset 272: vec4 cascadeScaleOffset1
  offset 288: vec4 cascadeScaleOffset2
  offset 304: vec4 shadowParams           (x = bias, y = normalBias, z = mapSize, w = enabled)
  Total: 320 bytes
```

This is uploaded once per frame — zero per-draw-call cost for lighting.
