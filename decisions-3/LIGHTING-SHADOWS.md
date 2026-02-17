# LIGHTING-SHADOWS.md - Lighting and Shadows

## Lighting Model

**Decision: Lambert diffuse only** (universal agreement)

Two light types:

### Ambient Light

Global constant illumination term. No direction, no position.

```typescript
scene.ambientLight = createAmbientLight({
  color: [1, 1, 1],
  intensity: 0.3,
})
```

### Directional Light

Single infinite-distance light with direction derived from rotation. Used for sun/moon.

```typescript
const light = createDirectionalLight({
  color: [1, 1, 0.9],
  intensity: 0.8,
  castShadow: true,
})
light.rotation.setFromEuler(-0.8, 0, 0.3)
scene.add(light)
```

### Shader

```glsl
vec3 ambient = u_ambientColor * u_ambientIntensity;
float NdotL = max(dot(normal, u_lightDirection), 0.0);
vec3 diffuse = u_lightColor * u_lightIntensity * NdotL;
vec3 litColor = baseColor * (ambient + diffuse);
```

### Why Lambert Only

- ~15 ALU ops per fragment vs ~80+ for Cook-Torrance PBR
- Sufficient for stylized/low-poly games (the target aesthetic)
- No need for metallic/roughness textures, environment probes, or IBL
- Point lights and spot lights are not needed for the initial target (outdoor/dungeon scenes with a single sun)
- Can be extended later without architectural changes

## Cascaded Shadow Maps (CSM)

**Decision: 3 cascades** (universal agreement)

### Why 3 Cascades

- 2 cascades: Visible quality drop at cascade boundaries
- 3 cascades: Good coverage for ~200m world range at reasonable cost
- 4 cascades: Diminishing returns, adds ~25% more shadow pass cost

### Shadow Map Resolution

**Decision: Configurable, default 1024x1024 per cascade** (Caracal, Mantis, Shark approach)

1024x1024 per cascade provides good quality at reasonable memory cost:
- 3 cascades * 1024 * 1024 * 4 bytes (depth32) = ~12MB GPU memory
- 2048x2048 (some implementations default to this) uses ~48MB - too expensive for mobile

Users can configure resolution based on quality/performance tradeoff:

```typescript
{
  shadows: {
    cascades: 3,
    mapSize: 1024,      // or 2048 for desktop
  }
}
```

### Shadow Storage

**Decision: Texture array** (Caracal, Fennec, Hyena, Lynx, Mantis, Shark use this or atlas)

Store each cascade as a layer in a depth texture array. This is cleaner than a texture atlas (no UV offset math) and avoids bleed between cascades.

WebGL2: Use `TEXTURE_2D_ARRAY` with `gl.framebufferTextureLayer` for per-cascade rendering.
WebGPU: Use `texture_2d_array<f32>` with array index in the shader.

### Cascade Split Scheme

**Decision: Logarithmic-linear blend** (universal agreement)

```typescript
const lambda = 0.5  // 0=linear, 1=logarithmic, 0.5=balanced
for (let i = 0; i < cascadeCount; i++) {
  const t = (i + 1) / cascadeCount
  const log = near * Math.pow(far / near, t)
  const lin = near + (far - near) * t
  splits[i] = lambda * log + (1 - lambda) * lin
}
```

Lambda of 0.5 provides good distribution: near cascades get more resolution (close objects need detail), far cascades cover more area (distant objects tolerate lower resolution).

### Shadow Bias

**Decision: Both depth bias and normal bias** (universal agreement)

```glsl
float bias = 0.001;        // Constant depth bias
float normalBias = 0.02;   // Push receiver along normal
vec3 biasedPosition = worldPosition + normal * normalBias;
```

Depth bias prevents shadow acne. Normal bias pushes the sampling position along the surface normal, which fixes acne on slopes without causing excessive Peter Pan effect.

### PCF Filtering

**Decision: 3x3 Poisson PCF** (Mantis, Shark approach)

Sample the shadow map at 9 Poisson-distributed offsets within a 3x3 texel kernel and average the results:

```glsl
float shadow = 0.0;
for (int i = 0; i < 9; i++) {
  vec2 offset = poissonDisk[i] * texelSize;
  shadow += texture(u_shadowMap, vec3(uv + offset, cascadeIndex));
}
shadow /= 9.0;
```

**Why 3x3 Over Larger Kernels**

- 3x3 (9 samples): ~0.3ms per cascade, soft edges sufficient for stylized look
- 5x5 (25 samples): ~0.8ms per cascade, overkill for low-poly aesthetic
- 1-tap: Hard edges, noticeable aliasing

The stylized/low-poly art style does not demand soft shadow penumbras. 3x3 PCF provides adequate softness at low cost.

### Texel Snapping

**Decision: Snap light projection to texel grid** (Mantis, Shark, Wren approach)

Prevents shadow map swimming when the camera moves:

```typescript
const texelSize = cascadeSize / shadowMapSize
lightMatrix[12] = Math.floor(lightMatrix[12] / texelSize) * texelSize
lightMatrix[13] = Math.floor(lightMatrix[13] / texelSize) * texelSize
```

This rounds the light's translation to the nearest shadow texel, eliminating sub-texel jitter.

### Cascade Selection in Fragment Shader

**Decision: View-space depth comparison** (universal agreement)

```glsl
int cascade = 2;  // Default to farthest
if (viewSpaceDepth < u_cascadeSplits[0]) cascade = 0;
else if (viewSpaceDepth < u_cascadeSplits[1]) cascade = 1;

// Sample shadow from the selected cascade
float shadow = sampleShadowMap(worldPos, cascade);
```

### Cascade Blending

**Decision: Smooth blend between cascades** (Hyena, Mantis, Shark approach)

Blend between cascades near split boundaries to hide the transition:

```glsl
float blendFactor = smoothstep(splitStart, splitEnd, viewSpaceDepth);
float shadow = mix(shadowNear, shadowFar, blendFactor);
```

Blend zone width: ~5% of cascade range. This eliminates visible seams between cascades.

### Per-Mesh Shadow Control

```typescript
mesh.castShadow = true     // Include in shadow pass rendering
mesh.receiveShadow = true  // Sample shadow map in fragment shader (via material flag)
```

## Uniform Layout

Per-frame lighting UBO:

```
struct LightData {
  vec4 ambientColorAndIntensity;     // rgb + intensity
  vec4 directionalColorAndIntensity; // rgb + intensity
  vec4 lightDirection;               // xyz + padding
  mat4 shadowMatrices[3];            // One per cascade
  vec4 cascadeSplits;                // xyz = splits, w = unused
  vec4 shadowConfig;                 // x = bias, y = normalBias, z = mapSize, w = enabled
}
```

Total: ~224 bytes. Fits easily in bind group 0 alongside camera data.

## Performance Budget

| Pass | Cost (1024 resolution) |
|------|----------------------|
| Shadow cascade 0 | ~0.5ms GPU |
| Shadow cascade 1 | ~0.4ms GPU |
| Shadow cascade 2 | ~0.3ms GPU |
| PCF sampling (opaque pass) | ~0.3ms GPU |
| **Total shadow overhead** | **~1.5ms GPU** |

At 1024x1024 resolution with 3x3 PCF, shadows consume ~1.5ms of the 10ms GPU budget. This leaves ample headroom for the opaque pass, bloom, and OIT.
