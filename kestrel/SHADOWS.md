# Cascading Shadow Maps

## Requirements

- World size: ~200×200 meters (low poly)
- Smooth shadows with no visible banding or aliasing
- Works on mobile GPUs
- Minimal performance cost

## Cascade Configuration

3 cascades with a practical split scheme:

```
Cascade 0: near → 10m    (high detail, close to camera)
Cascade 1: 10m → 40m     (medium detail)
Cascade 2: 40m → 150m    (low detail, far from camera)
```

Split distances use a **logarithmic-linear blend** (Practical Split Scheme from GPU Gems 3):

```
splitDistance[i] = lerp(
  near + (far - near) * (i / numCascades),          // linear
  near * (far / near) ^ (i / numCascades),           // logarithmic
  lambda                                              // blend factor, typically 0.5
)
```

This gives more resolution to nearby shadows (where the player notices quality) and less to distant shadows (where the low resolution is hidden by distance).

## Shadow Atlas

All 3 cascades are packed into a **single depth texture** (shadow atlas) to avoid texture switching:

```
Shadow atlas: 2048 × 2048 (configurable)

┌──────────────┬──────────────┐
│              │              │
│  Cascade 0   │  Cascade 1   │
│  1024×1024   │  1024×1024   │
│              │              │
├──────────────┴──────────────┤
│                             │
│         Cascade 2           │
│         2048×1024           │
│                             │
└─────────────────────────────┘
```

Alternatively, three equal 1024×1024 regions in a 2048×1536 atlas. The exact layout is configurable.

## Light Space Matrix Computation

For each cascade:

```
computeCascadeMatrix(cascade, camera, light):
  // 1. Compute the camera frustum slice for this cascade
  frustumCorners = computeFrustumCorners(camera, cascade.near, cascade.far)

  // 2. Transform frustum corners to world space
  worldCorners = frustumCorners.map(c => camera.inverseViewMatrix × c)

  // 3. Compute the center of the frustum slice
  center = average(worldCorners)

  // 4. Build light view matrix looking at the center
  lightView = lookAt(center - lightDir * farOffset, center, upHint)

  // 5. Compute tight AABB in light space
  lightSpaceCorners = worldCorners.map(c => lightView × c)
  aabb = computeAABB(lightSpaceCorners)

  // 6. Build orthographic projection from AABB
  lightProj = orthographic(aabb.min.x, aabb.max.x, aabb.min.y, aabb.max.y, aabb.min.z, aabb.max.z)

  // 7. Snap to texel grid to prevent shadow swimming
  lightViewProj = lightProj × lightView
  shadowMapSize = cascade.resolution
  texelSize = (aabb.max.x - aabb.min.x) / shadowMapSize
  offset.x = floor(lightViewProj[12] / texelSize) * texelSize - lightViewProj[12]
  offset.y = floor(lightViewProj[13] / texelSize) * texelSize - lightViewProj[13]
  lightViewProj[12] += offset.x
  lightViewProj[13] += offset.y

  return lightViewProj
```

### Texel Snapping

Step 7 is critical. Without it, the shadow map "swims" (shimmers) as the camera moves because the light projection shifts by sub-texel amounts. Snapping to the texel grid eliminates this.

## Shadow Pass

Render shadow casters into the shadow atlas using a depth-only shader:

```
shadowPass(scene, cascades):
  beginRenderPass(shadowAtlas, { loadOp: 'clear', clearDepth: 1.0 })

  for each cascade in cascades:
    setViewport(cascade.viewport)
    setPipeline(depthOnlyPipeline)  // simple vertex-only shader
    setBindGroup(0, { viewProjection: cascade.lightViewProj })

    // Draw shadow casters (sorted front-to-back for early-z)
    for each mesh in scene.shadowCasters:
      if not frustumIntersects(cascade.lightFrustum, mesh.worldBoundingBox): continue
      setBindGroup(1, { worldMatrix: mesh.worldMatrix })
      drawIndexed(mesh)

  endRenderPass()
```

The depth-only pipeline has no fragment shader (or a trivial one for alpha-tested geometry), making it very fast.

## Shadow Sampling

In the geometry pass fragment shader, shadows are sampled using PCF (Percentage Closer Filtering):

```wgsl
fn sampleCascadedShadowMap(worldPos: vec3<f32>, NdotL: f32) -> f32 {
  // Determine which cascade this fragment belongs to
  let viewZ = (frameUniforms.viewMatrix * vec4(worldPos, 1.0)).z;
  var cascadeIndex = 2u;
  if viewZ > cascadeSplits.x { cascadeIndex = 0u; }
  else if viewZ > cascadeSplits.y { cascadeIndex = 1u; }

  // Project into shadow map space
  let shadowCoord = cascadeViewProjections[cascadeIndex] * vec4(worldPos, 1.0);
  let projCoord = shadowCoord.xyz / shadowCoord.w;
  let uv = projCoord.xy * 0.5 + 0.5;
  let depth = projCoord.z;

  // Offset UV to cascade region in atlas
  let atlasUV = cascadeOffsets[cascadeIndex] + uv * cascadeScales[cascadeIndex];

  // Bias to prevent shadow acne
  let bias = max(0.005 * (1.0 - NdotL), 0.001);

  // 4-sample PCF for smooth edges
  let texelSize = 1.0 / f32(shadowMapSize);
  var shadow = 0.0;
  for (var x = -1; x <= 1; x += 2) {
    for (var y = -1; y <= 1; y += 2) {
      let offset = vec2<f32>(f32(x), f32(y)) * texelSize * 0.5;
      let sampleDepth = textureSample(shadowAtlas, shadowSampler, atlasUV + offset);
      shadow += select(0.0, 1.0, depth - bias <= sampleDepth);
    }
  }
  return shadow * 0.25;
}
```

### PCF Configuration

- **4-sample PCF** (2×2 Poisson-like): Good quality, cheap. 4 texture samples per fragment.
- Optional: **9-sample PCF** (3×3) for higher quality on desktop.
- On WebGPU, `comparison` samplers do hardware PCF, reducing this to `textureSampleCompare()` calls.
- On WebGL2, `sampler2DShadow` with `texture()` does hardware comparison.

## Cascade Blending

At cascade boundaries, there can be a visible seam where shadow quality changes abruptly. A simple blend in the transition zone fixes this:

```wgsl
// Blend zone: last 10% of each cascade
let blendFactor = smoothstep(0.9, 1.0, cascadeFade);
let shadowA = sampleShadow(cascadeIndex, ...);
let shadowB = sampleShadow(cascadeIndex + 1, ...);
let shadow = mix(shadowA, shadowB, blendFactor);
```

This costs an extra shadow sample in the transition zone but eliminates visible cascade boundaries.

## Configuration API

```typescript
const engine = await Engine.create({
  canvas,
  shadows: {
    enabled: true,
    mapSize: 2048,           // shadow atlas resolution
    cascadeCount: 3,         // 1–4 cascades
    maxDistance: 150,         // max shadow distance in world units
    lambda: 0.5,             // logarithmic-linear split blend
    bias: 0.005,             // depth bias
    pcfSamples: 4,           // 4 or 9
  }
})
```

## Performance Notes

- **Depth-only rendering** is ~3× faster than full shading (no fragment shader, no textures).
- **Single atlas texture** means zero texture switches during the geometry pass.
- **3 cascades** at 1024×1024 each is a good mobile balance. Desktop can use 4 cascades at 2048.
- **Front-to-back sorting** in the shadow pass maximizes early-z rejection — fragments behind already-written depth are skipped.
- Shadow pass can be cached in a **WebGPU render bundle** if the shadow caster list doesn't change.
