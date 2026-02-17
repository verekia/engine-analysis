# LIGHTING-SHADOWS.md - Decisions

## Decision: Lambert + 3 CSM Cascades, Texture Array, 3x3 PCF with Cascade Blending

### Lighting Model

Universal agreement across all 9:

- **Lambert diffuse**: `N dot L` for directional light (simple, fast, good for stylized aesthetic)
- **Directional light**: One shadow-casting directional light (sun)
- **Ambient light**: Uniform base illumination, no direction, no shadows
- **No specular**: Purely diffuse for low-poly/stylized art
- **Light data in per-frame UBO**: Zero per-draw overhead

```glsl
vec3 lighting = ambientColor * ambientIntensity
             + lightColor * lightIntensity * max(dot(normal, lightDir), 0.0) * shadow;
vec3 finalColor = lighting * surfaceColor + emissive;
```

### Shadow Technique: 3-Cascade CSM

Universal agreement across all 8 active implementations:

- **Cascaded Shadow Maps** for directional light
- **3 cascades** for 200x200m worlds (the prompt specifies this world size)
- **Practical split scheme**: blend of logarithmic and linear distribution

### Cascade Split Lambda

**Chosen**: Lambda = 0.7 (4/9: Fennec, Lynx, Mantis, Wren - more resolution near camera)
**Rejected**: Lambda = 0.5 (4/9: Caracal, Hyena, Rabbit, Shark) - balanced but gives less near-camera detail

For a 200m world with 3 cascades and lambda=0.7:
- Cascade 0: ~0.1m - 12m (high detail near camera)
- Cascade 1: ~12m - 50m (mid range)
- Cascade 2: ~50m - 200m (far range)

Lambda = 0.7 concentrates more resolution near the camera where players look most, which matches the "smooth shadows" requirement from the prompt.

### Shadow Map Storage: Texture Array

**Chosen**: `TEXTURE_2D_ARRAY` with 3 layers (2/9: Fennec, Lynx)
**Rejected**: Texture atlas (6/9) - works but requires UV offset calculations in the shader and has worse cache behavior

Texture array advantages:
- Cleaner shader code (array index instead of UV offset)
- Better GPU cache behavior (each cascade is a separate layer)
- Supported on all WebGL2 devices
- Simpler on WebGPU (`texture_depth_2d_array`)

### Shadow Map Resolution: Configurable, Default 1024

**Chosen**: 1024x1024 per cascade as default (4/9: Caracal, Hyena, Rabbit, Shark), configurable up to 2048
**Rationale**: The prompt targets mobile performance. 1024 per cascade costs ~2.5ms on mobile vs ~3.5ms for 2048. Users can increase to 2048 for desktop.

```typescript
shadows: {
  mapSize: 1024,  // Per cascade, configurable
}
```

### PCF: 3x3 with Poisson Disk

**Chosen**: 3x3 PCF using Poisson disk sampling (4/9 use 3x3, Caracal/Mantis mention Poisson)
**Rejected**: 5x5 PCF (3/9: Lynx, Mantis, Wren) - smoother but 2.5x more expensive in fragment shader, problematic on mobile

Poisson disk breaks up aliasing patterns better than a regular grid at the same sample count:

```glsl
const vec2 poissonDisk[9] = vec2[](
  vec2(-0.94, -0.34), vec2(0.94, 0.34),
  vec2(-0.34, 0.94), vec2(0.34, -0.94),
  vec2(-0.54, -0.54), vec2(0.54, 0.54),
  vec2(-0.07, -0.83), vec2(0.07, 0.83),
  vec2(0.0, 0.0)
);
```

Hardware comparison samplers used on both backends:
- WebGL2: `sampler2DShadow` with `TEXTURE_COMPARE_MODE`
- WebGPU: `sampler_comparison` with `textureSampleCompareLevel()`

### Cascade Blending

**Chosen**: Smooth blending between cascades (6/9: Fennec, Hyena, Lynx, Mantis, Rabbit, Wren)
**Rejected**: Hard boundaries (2/9: Caracal, Shark) - visible seams between cascades

Blend zone: ~10% of cascade range or 2-5 meters:

```glsl
float blend = smoothstep(splitEnd - blendRange, splitEnd, viewDepth);
float shadow = mix(shadowFromCascadeN, shadowFromCascadeN1, blend);
```

### Cascade Stabilization: Texel Snapping

**Chosen**: Projection matrix snapping (5/9: Caracal, Fennec, Hyena, Mantis, Wren)

```typescript
// Snap light matrix translation to texel grid
const texelSize = cascadeSize / mapSize
lightMatrix[12] = Math.floor(lightMatrix[12] / texelSize) * texelSize
lightMatrix[13] = Math.floor(lightMatrix[13] / texelSize) * texelSize
```

Prevents shadow shimmer/swimming during camera movement.

### Shadow Bias: Depth + Slope-Scaled

**Chosen**: Constant depth bias + slope-scaled bias (6/9: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit)

```glsl
float bias = constantBias + slopeBias * (1.0 - NdotL);
```

Combined with front-face culling during shadow rendering (render back faces to depth map) to further reduce acne.

### Shadow Caster Culling

Per-cascade frustum culling (universal across all 8 active implementations):
- Cull objects against each cascade's light-space frustum
- Avoid rendering distant objects into near cascade
- Extend light projection Z range backwards ~50-100m to catch casters behind camera

### Shadow Pass Shader

Position-only vertex shader (6/9 implementations), plus skinning support:

```glsl
// Minimal shadow vertex shader
void main() {
  vec4 pos = u_lightVP * u_worldMatrix * vec4(a_position, 1.0);
  #ifdef HAS_SKINNING
    pos = u_lightVP * u_worldMatrix * boneMatrix * vec4(a_position, 1.0);
  #endif
  gl_Position = pos;
}
// Fragment shader: empty (depth-only)
```

### Light as Scene Node

**Chosen**: Lights extend Node (6/9: Caracal, Fennec, Lynx, Mantis, Rabbit, Shark)

```typescript
const sun = createDirectionalLight({
  color: [1, 1, 0.9],
  intensity: 1.0,
  castShadow: true,
})
scene.add(sun)
sun.rotation.setFromEuler(Math.PI / 4, 0, Math.PI / 6)
```

Direction derived from node's world rotation. Ambient light stored on Scene directly (no spatial properties).

### Shadow Distance: 200m Default

Matching the prompt's "200x200m" world specification. Configurable at runtime.

### Shadow Caching: Not in v1

**Rejected for v1**: Static shadow caching (2/9: Lynx, Wren)

Shadow caching adds complexity (dirty tracking for all shadow casters). The depth-only shadow pass is fast enough (<2.5ms on mobile with 1024 resolution) that caching isn't needed for the target performance budget. Can be added as an optimization later.

### Configuration API

```typescript
{
  shadows: {
    enabled: true,
    mapSize: 1024,         // Per cascade
    cascadeCount: 3,
    maxDistance: 200,
    lambda: 0.7,
    bias: 0.001,
    slopeBias: 0.005,
    cascadeBlend: true,
    blendRange: 2.0,       // Meters
  }
}
```
