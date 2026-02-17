# Lighting & Shadows

## Lighting System (v1)

Keep it simple. Lambert diffuse (directional + ambient) is enough for most stylized games.

### Light State

```ts
interface LightState {
  direction: [number, number, number]
  color: [number, number, number]
  ambientColor: [number, number, number]
}
```

Uploaded as a uniform buffer (bind group 1), shared across all draws.

### Shader Implementation

Basic Lambert lighting in the fragment shader:

```wgsl
struct Light {
  direction: vec3<f32>,
  color: vec3<f32>,
  ambientColor: vec3<f32>,
}

@group(1) @binding(0) var<uniform> light: Light;

fn computeLighting(normal: vec3<f32>, albedo: vec3<f32>) -> vec3<f32> {
  let diffuse = max(dot(normal, -light.direction), 0.0);
  let ambient = light.ambientColor;
  let lit = albedo * (light.color * diffuse + ambient);
  return lit;
}
```

### Unlit Flag

Per-entity unlit flag for UI elements, skyboxes, emissive objects:

```ts
// Set via entity flags
entity.setFlag(UNLIT_FLAG)
```

In the shader, skip lighting calculations if the unlit flag is set:

```wgsl
if (flags & UNLIT_FLAG) {
  return albedo;  // No lighting
} else {
  return computeLighting(normal, albedo);
}
```

## Shadows (Out of Scope for v1)

Shadows are explicitly out of scope for v1 due to significant complexity. This will be added in v2.

### Planned Features (v2)

#### Cascaded Shadow Maps (CSM)

- 3-4 cascades for directional light
- Shadow map resolution: 2048x2048 per cascade
- PCF (Percentage Closer Filtering) for soft shadows
- Bias and slope-scale bias to prevent shadow acne

#### Shadow Rendering Pipeline

1. **Shadow pass**: Render scene from light's perspective into depth texture (per cascade)
2. **Main pass**: Sample shadow map in fragment shader to determine if pixel is in shadow

#### Shadow Map Storage

- Texture array with one layer per cascade
- Depth format: `depth32float` or `depth24plus`

#### Shadow Uniforms

```ts
interface ShadowUniforms {
  lightViewProjection: mat4[4]  // One VP matrix per cascade
  cascadeSplits: vec4            // Split distances for cascade selection
  shadowBias: number
}
```

#### Shader Implementation (v2)

```wgsl
fn computeShadow(worldPos: vec3<f32>) -> f32 {
  // Select cascade based on view-space depth
  let cascade = selectCascade(viewSpaceDepth);

  // Transform to light space
  let lightSpacePos = shadowUniforms.lightViewProjection[cascade] * vec4(worldPos, 1.0);
  let shadowCoord = lightSpacePos.xyz / lightSpacePos.w;

  // Sample shadow map with PCF
  return pcfShadow(shadowMap, shadowCoord, cascade);
}
```

## Additional Lighting Features (Future)

### Point Lights (v2)

- Per-light position, color, radius, intensity
- Forward+ or clustered shading for many lights
- Shadow cubemaps for point light shadows (optional)

### Spot Lights (v2)

- Cone angle, penumbra
- Shadow maps (2D depth texture per light)

### Image-Based Lighting (IBL) (v3)

- Environment maps (cubemap or equirectangular)
- Diffuse and specular convolution
- PBR material integration

### Lightmaps (v3)

- Pre-baked indirect lighting
- Second UV channel for lightmap coordinates
- Lightmap atlas texture

## Performance Considerations

- v1 lighting is extremely cheap: 1 uniform buffer, no shadow maps, simple Lambert shading
- This allows hitting the 2000 draw calls @ 60fps target without lighting bottlenecks
- v2 shadows will require careful optimization (cascade selection, PCF sampling, shadow map resolution)
