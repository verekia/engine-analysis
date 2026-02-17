# LIGHTING-SHADOWS - Design Decisions

## Lighting Model

**Decision: Lambert diffuse shading (N·L dot product) — no specular**

- Sources: All 9 implementations agree
- Directional light as primary light source (sun-like, parallel rays)
- Ambient light for uniform base illumination
- No specular highlights — purely diffuse for stylized/low-poly aesthetic

```glsl
vec3 diffuse = materialColor * lightColor * max(dot(normal, lightDir), 0.0);
vec3 finalColor = ambientColor * ambientIntensity + diffuse * shadow;
finalColor += emissiveColor * emissiveIntensity;  // additive, unaffected by lighting
```

## Light Data Upload

**Decision: Light data in per-frame UBO (bind group 0)**

One upload per frame, zero per-draw-call overhead. Contains:
- Directional light direction, color, intensity
- Ambient light color, intensity
- Shadow matrices (3× mat4)
- Cascade split distances
- Shadow bias parameters

Total: ~256-320 bytes per frame.

## Light Structure

**Decision: Directional light as scene node; ambient light as scene property**

- Sources: Directional as node from Caracal, Fennec, Lynx, Mantis, Rabbit, Shark (6/8); ambient as scene property from Fennec, Lynx, Mantis, Wren (4/8)

```typescript
const dirLight = createDirectionalLight({
  color: [1, 1, 0.95],
  intensity: 1.0,
  castShadow: true,
})
dirLight.position.set(50, 50, 100)
scene.add(dirLight)

scene.ambientLight = { color: [0.4, 0.45, 0.5], intensity: 0.3 }
```

Directional light as a scene node allows it to be parented, transformed, and animated. Ambient has no spatial properties so it's simpler as a scene property.

## Shadow Technique

**Decision: 3-cascade CSM (Cascaded Shadow Maps)**

- Sources: All 8 active implementations agree on 3 cascades for 200×200m worlds

Practical split scheme blending logarithmic and linear distribution. Orthographic projection per cascade with tight fit around frustum slice.

## Shadow Map Storage

**Decision: Texture array (TEXTURE_2D_ARRAY with 3 layers)**

- Sources: Fennec, Lynx (2/8 — texture array); majority use atlas
- Rejected: Texture atlas (Caracal, Hyena, Mantis, Rabbit, Shark, Wren)

```typescript
// Texture array: 3 layers, one per cascade
shadowMap = createTexture({
  format: 'depth24plus',
  size: [cascadeSize, cascadeSize, 3],  // width, height, layers
  dimension: '2d-array',
})
```

Rationale: Cleaner shader code (array index instead of UV offset calculations). Better GPU cache behavior. Supported on all WebGL2 devices and WebGPU. No UV offset math errors. The atlas approach is equally valid but texture arrays are the cleaner abstraction.

## Shadow Map Resolution

**Decision: 2048×2048 per cascade by default, configurable to 1024 for mobile**

- Sources: Fennec, Lynx, Mantis, Wren (4/8) use 2048 default

```typescript
shadows: {
  mapSize: 2048,  // default: high quality
  // Mobile preset: mapSize: 1024
}
```

2048 provides smooth shadows for the 200×200m world target. Configurable down to 1024 for mobile performance budgets.

## Cascade Split Scheme

**Decision: Lambda = 0.7 (logarithmic-weighted) with practical split distances**

- Sources: Fennec, Lynx, Mantis, Wren (4/8) use lambda 0.7

```typescript
// Practical split scheme (Engel, 2006)
for (let i = 0; i < cascadeCount; i++) {
  const t = (i + 1) / cascadeCount
  const log = near * Math.pow(far / near, t)
  const linear = near + (far - near) * t
  splits[i] = lambda * log + (1 - lambda) * linear
}
```

For 200m world with lambda 0.7:
- Cascade 0: 0.1m - ~12m (near, high detail)
- Cascade 1: ~12m - ~50m (mid range)
- Cascade 2: ~50m - 200m (far range)

More resolution near camera where shadow quality matters most.

## PCF Filtering

**Decision: 3×3 PCF by default, configurable to 5×5**

- Sources: 3×3 from Caracal, Hyena, Rabbit, Shark (4/8); configurable from Fennec, Mantis (2/8)

```glsl
// 3×3 PCF (9 samples)
float shadow = 0.0;
for (int x = -1; x <= 1; x++) {
  for (int y = -1; y <= 1; y++) {
    shadow += textureSampleCompare(shadowMap, shadowSampler,
      shadowUV + vec2(x, y) * texelSize, cascade, depth);
  }
}
shadow /= 9.0;
```

3×3 provides good shadow softness at acceptable GPU cost (~0.15ms desktop, ~0.4ms mobile). 5×5 option (25 samples) available for high-quality desktop rendering.

## Cascade Blending

**Decision: Smooth transition between cascades**

- Sources: Fennec, Hyena, Lynx, Mantis, Rabbit, Wren (6/8)
- Rejected: Hard boundaries (Caracal, Shark) — visible seams

```glsl
// Blend zone: last 10% of each cascade range (or 2-5m)
float blendFactor = smoothstep(splitEnd - blendRange, splitEnd, viewDepth);
float shadow = mix(cascadeShadow[i], cascadeShadow[i+1], blendFactor);
```

## Texel Snapping

**Decision: Projection matrix texel snapping to prevent shadow shimmer**

- Sources: Caracal, Fennec, Hyena, Mantis, Wren (5/8)

```typescript
// Snap light matrix translation to texel grid
const texelSize = cascadeWorldSize / cascadeResolution
lightMatrix[12] = Math.floor(lightMatrix[12] / texelSize) * texelSize
lightMatrix[13] = Math.floor(lightMatrix[13] / texelSize) * texelSize
```

Prevents sub-texel shifts during camera movement that cause shadow edge flickering.

## Shadow Bias

**Decision: Depth bias + slope-scaled bias**

- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit (6/8)

```glsl
float bias = constantBias + slopeBias * tan(acos(clamp(NdotL, 0.0, 1.0)));
bias = clamp(bias, 0.0, maxBias);
```

Combined with front-face culling during shadow rendering to further reduce shadow acne.

## Shadow Rendering

**Decision: Depth-only shader, front-face culling, per-cascade frustum culling**

- Position-only vertex shader (minimal, no attributes beyond position + skinning)
- Front-face culling: render back faces during shadow pass to reduce acne
- Per-cascade frustum culling: only render objects that intersect each cascade's light frustum
- Skinned mesh shadow support (same bone matrices as main pass)

## Shadow Distance

**Decision: 200m default, configurable**

- Sources: Caracal, Fennec, Hyena, Mantis (4/8) default to 200m

Matches the 200×200m low-poly world requirement. Configurable for smaller arenas (50m) or larger worlds (500m).

## Z-Range Extension

**Decision: Extend light orthographic projection Z-range backwards by 50-100m**

- Sources: All implementations mention this technique
- Catches shadow casters behind the camera frustum that may cast shadows into the visible area

## Configuration API

```typescript
const engine = await createEngine(canvas, {
  shadows: {
    enabled: true,
    mapSize: 2048,         // per cascade
    cascadeCount: 3,
    maxDistance: 200,
    lambda: 0.7,
    pcfSamples: 9,         // 9 = 3×3, 25 = 5×5
    cascadeBlend: true,
    blendRange: 2.0,       // meters
    bias: 0.001,
    slopeBias: 0.005,
  },
})
```
