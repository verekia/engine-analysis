# LIGHTING-SHADOWS — Final Decision

## Lighting Model

**Decision: Lambert diffuse shading (N·L) — no specular** (universal agreement)

```glsl
vec3 ambient = u_ambientColor * u_ambientIntensity;
float NdotL = max(dot(normal, u_lightDirection), 0.0);
vec3 diffuse = u_lightColor * u_lightIntensity * NdotL;
vec3 litColor = baseColor * (ambient + diffuse * shadow);
vec3 finalColor = litColor + emissive;  // Emissive is additive, unaffected by lighting
```

Why Lambert only:
- ~15 ALU ops per fragment vs ~80+ for Cook-Torrance PBR
- Sufficient for stylized/low-poly games
- No need for metallic/roughness textures, environment probes, or IBL
- Can be extended later without architectural changes

## Light Types

### Directional Light (Scene Node)

Direction derived from node's world rotation. Can be parented, transformed, animated.

```typescript
const sun = createDirectionalLight({
  color: [1, 1, 0.95],
  intensity: 1.0,
  castShadow: true,
})
sun.position.set(50, 50, 100)
scene.add(sun)
```

### Ambient Light (Scene Property)

No spatial properties — simpler as a scene property than a node.

```typescript
scene.ambientLight = { color: [0.4, 0.45, 0.5], intensity: 0.3 }
```

## Light Data Upload

Light data packed into per-frame UBO (bind group 0). One upload per frame, zero per-draw overhead.

Contains: directional light direction/color/intensity, ambient color/intensity, 3× shadow matrices, cascade split distances, bias parameters. Total: ~256-320 bytes.

## Cascaded Shadow Maps

**Decision: 3-cascade CSM** (universal agreement for 200×200m worlds)

### Shadow Map Resolution

**Decision: 1024×1024 per cascade by default, configurable to 2048** (mobile-first)

1024 per cascade provides good quality at reasonable cost:
- 3 cascades × 1024 × 1024 × 4 bytes = ~12MB GPU memory
- ~1.5ms total shadow pass on mobile
- 2048 is available for desktop via configuration

### Shadow Map Storage

**Decision: Texture array (`TEXTURE_2D_ARRAY` with 3 layers)** (Fennec, Lynx approach)

Cleaner than a texture atlas:
- Array index in shader instead of UV offset calculations
- Better GPU cache behavior (each cascade is a separate layer)
- No bleed between cascades
- Supported on all WebGL2 and WebGPU devices

### Cascade Split Scheme

**Decision: Lambda = 0.7 (logarithmic-weighted)** (decisions-1 and -2 agree)

```typescript
const lambda = 0.7  // More resolution near camera
for (let i = 0; i < cascadeCount; i++) {
  const t = (i + 1) / cascadeCount
  const log = near * Math.pow(far / near, t)
  const linear = near + (far - near) * t
  splits[i] = lambda * log + (1 - lambda) * linear
}
```

For 200m world with lambda 0.7:
- Cascade 0: ~0.1m – 12m (high detail near camera)
- Cascade 1: ~12m – 50m (mid range)
- Cascade 2: ~50m – 200m (far range)

Lambda 0.7 concentrates more resolution near the camera where players look most, which matches the "smooth shadows" requirement.

### PCF Filtering

**Decision: 3×3 Poisson PCF (9 samples)** (decisions-2 and -3 agree on Poisson)

Poisson disk breaks up aliasing patterns better than a regular grid at the same sample count:

```glsl
const vec2 poissonDisk[9] = vec2[](
  vec2(-0.94, -0.34), vec2(0.94, 0.34),
  vec2(-0.34, 0.94), vec2(0.34, -0.94),
  vec2(-0.54, -0.54), vec2(0.54, 0.54),
  vec2(-0.07, -0.83), vec2(0.07, 0.83),
  vec2(0.0, 0.0)
);

float shadow = 0.0;
for (int i = 0; i < 9; i++) {
  vec2 offset = poissonDisk[i] * texelSize;
  shadow += textureSampleCompare(shadowMap, shadowSampler,
    shadowUV + offset, cascadeIndex, depth);
}
shadow /= 9.0;
```

Hardware comparison samplers on both backends:
- WebGL2: `sampler2DShadow` with `TEXTURE_COMPARE_MODE`
- WebGPU: `sampler_comparison` with `textureSampleCompareLevel()`

### Cascade Blending

**Decision: Smooth transition between cascades** (6/9 agree)

```glsl
float blendFactor = smoothstep(splitEnd - blendRange, splitEnd, viewDepth);
float shadow = mix(cascadeShadow[i], cascadeShadow[i + 1], blendFactor);
```

Blend zone: ~10% of cascade range or 2-5 meters. Eliminates visible seams between cascades.

### Texel Snapping

**Decision: Projection matrix texel snapping** (5/9 agree)

```typescript
const texelSize = cascadeWorldSize / cascadeResolution
lightMatrix[12] = Math.floor(lightMatrix[12] / texelSize) * texelSize
lightMatrix[13] = Math.floor(lightMatrix[13] / texelSize) * texelSize
```

Prevents shadow shimmer/swimming during camera movement.

### Shadow Bias

**Decision: Depth bias + slope-scaled bias** (6/9 agree)

```glsl
float bias = constantBias + slopeBias * (1.0 - NdotL);
bias = clamp(bias, 0.0, maxBias);
```

Combined with front-face culling during shadow rendering (render back faces to the depth map) to further reduce acne.

### Shadow Pass Rendering

- Position-only vertex shader (minimal attributes beyond position + skinning)
- Front-face culling: render back faces to reduce acne
- Per-cascade frustum culling: only render objects intersecting each cascade's light frustum
- Extend light projection Z range backwards ~50-100m to catch casters behind camera

### Per-Mesh Shadow Control

```typescript
mesh.castShadow = true     // Include in shadow pass
mesh.receiveShadow = true  // Sample shadow map in fragment shader
```

## Configuration API

```typescript
const engine = await createEngine(canvas, {
  shadows: {
    enabled: true,
    mapSize: 1024,         // Per cascade, configurable (1024 default, 2048 for desktop)
    cascadeCount: 3,
    maxDistance: 200,
    lambda: 0.7,
    bias: 0.001,
    slopeBias: 0.005,
    cascadeBlend: true,
    blendRange: 2.0,       // Meters
  },
})
```

## Performance Budget

| Pass | Cost (1024 resolution) |
|------|----------------------|
| Shadow cascade 0 | ~0.5ms GPU |
| Shadow cascade 1 | ~0.4ms GPU |
| Shadow cascade 2 | ~0.3ms GPU |
| PCF sampling (opaque pass) | ~0.3ms GPU |
| **Total shadow overhead** | **~1.5ms GPU** |
