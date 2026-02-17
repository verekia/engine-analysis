# Lighting & Shadows Architecture

## Light Types

### Ambient Light

Global, directionless illumination applied uniformly to all surfaces.

```typescript
interface AmbientLight extends Node3D {
  color: Color        // default: white
  intensity: number   // default: 0.3
}
```

Shader contribution:
```glsl
vec3 ambient = u_ambientColor * u_ambientIntensity;
```

### Directional Light

Infinite light source (like the sun). Direction is derived from the node's orientation (local -Z direction in world space, i.e. the light shines in the direction the node "faces").

```typescript
interface DirectionalLight extends Node3D {
  color: Color            // default: white
  intensity: number       // default: 1.0
  castShadow: boolean     // default: false

  // Shadow configuration (when castShadow = true)
  shadow: {
    mapSize: number       // Shadow map resolution per cascade (default: 1024)
    cascades: number      // Number of CSM cascades (default: 3)
    near: number          // Shadow camera near plane (default: 0.1)
    far: number           // Shadow camera far plane (default: 200)
    bias: number          // Depth bias (default: 0.002)
    normalBias: number    // Normal offset bias (default: 0.5)
  }
}
```

Shader contribution:
```glsl
float NdotL = max(dot(normal, -u_lightDirection), 0.0);
vec3 diffuse = u_lightColor * u_lightIntensity * NdotL;
float shadow = sampleShadowMap(worldPosition, depth);  // 0 = shadowed, 1 = lit
vec3 lighting = ambient + diffuse * shadow;
```

## Cascading Shadow Maps (CSM)

### Why CSM?

A single shadow map covering a 200×200m world would need extremely high resolution to avoid aliasing near the camera. CSM solves this by splitting the view frustum into ranges, each with its own shadow map that covers a smaller area at higher detail.

### Cascade Configuration

For a 200×200m low-poly world:

```
Cascade 0 (near):   0m  — 15m    High detail for close objects
Cascade 1 (mid):    15m — 50m    Medium detail
Cascade 2 (far):    50m — 200m   Low detail for distant objects
```

### Split Scheme

Practical split: blend between logarithmic and uniform distributions.

```typescript
const computeCascadeSplits = (
  near: number,
  far: number,
  cascadeCount: number,
  lambda: number = 0.7,  // 0 = uniform, 1 = logarithmic
): number[] => {
  const splits: number[] = []
  for (let i = 1; i <= cascadeCount; i++) {
    const uniform = near + (far - near) * (i / cascadeCount)
    const log = near * Math.pow(far / near, i / cascadeCount)
    splits.push(lambda * log + (1 - lambda) * uniform)
  }
  return splits
}
```

### Shadow Map Atlas

All cascades are rendered into a single 2D texture atlas to minimize texture binds:

```
┌──────────┬──────────┬──────────┐
│          │          │          │
│ Cascade  │ Cascade  │ Cascade  │
│    0     │    1     │    2     │
│ 1024×1024│ 1024×1024│ 1024×1024│
│          │          │          │
└──────────┴──────────┴──────────┘
         3072 × 1024 atlas
```

Alternative for WebGPU: Use a texture array (3 layers of 1024×1024). Texture arrays have better cache behavior and cleaner shader code. WebGL2 also supports `TEXTURE_2D_ARRAY`.

**Decision**: Use `TEXTURE_2D_ARRAY` on both backends. This simplifies the shader (array index instead of UV offset calculations) and is well-supported.

### Per-Cascade Rendering

For each cascade:

```typescript
const renderCascade = (
  cascadeIndex: number,
  light: DirectionalLight,
  frustumCorners: Vec3[],    // 8 corners of the frustum slice
  shadowCasters: Mesh[],
) => {
  // 1. Compute light-space bounding box of frustum slice
  const lightView = mat4LookAt(lightPosition, lightTarget, lightUp)
  const transformedCorners = frustumCorners.map(c => vec3TransformMat4(c, lightView))

  // 2. Compute tight orthographic projection
  const bounds = computeAABB(transformedCorners)

  // 3. Snap to texel grid (prevents shadow edge shimmer on camera movement)
  const texelSize = (bounds.max[0] - bounds.min[0]) / shadowMapSize
  bounds.min[0] = Math.floor(bounds.min[0] / texelSize) * texelSize
  bounds.max[0] = Math.floor(bounds.max[0] / texelSize) * texelSize
  bounds.min[1] = Math.floor(bounds.min[1] / texelSize) * texelSize
  bounds.max[1] = Math.floor(bounds.max[1] / texelSize) * texelSize

  const lightProjection = mat4Ortho(
    bounds.min[0], bounds.max[0],
    bounds.min[1], bounds.max[1],
    bounds.min[2] - 50,  // Extend behind to catch shadow casters behind frustum
    bounds.max[2],
  )

  // 4. Store VP matrix for this cascade
  cascadeVP[cascadeIndex] = mat4Multiply(lightProjection, lightView)

  // 5. Render shadow casters with depth-only shader
  // Use front-face culling (render back faces) to reduce shadow acne
  for (const caster of shadowCasters) {
    drawShadowDepth(caster, cascadeVP[cascadeIndex], cascadeIndex)
  }
}
```

### Shadow Sampling in Main Pass

```glsl
uniform mat4 u_cascadeVP[3];       // Light VP matrices
uniform float u_cascadeSplits[3];   // View-space Z split distances
uniform sampler2DArray u_shadowMap;

float sampleShadow(vec3 worldPos, float viewDepth) {
  // Determine which cascade to use
  int cascadeIndex = 2;  // Default to farthest
  for (int i = 0; i < 3; i++) {
    if (viewDepth < u_cascadeSplits[i]) {
      cascadeIndex = i;
      break;
    }
  }

  // Project world position into light space for this cascade
  vec4 lightSpacePos = u_cascadeVP[cascadeIndex] * vec4(worldPos, 1.0);
  vec3 projCoords = lightSpacePos.xyz / lightSpacePos.w;
  projCoords = projCoords * 0.5 + 0.5;  // [-1,1] → [0,1]

  // PCF 3×3 for smooth edges
  float shadow = 0.0;
  vec2 texelSize = 1.0 / vec2(textureSize(u_shadowMap, 0).xy);
  for (int x = -1; x <= 1; x++) {
    for (int y = -1; y <= 1; y++) {
      vec2 offset = vec2(x, y) * texelSize;
      float depth = texture(u_shadowMap, vec3(projCoords.xy + offset, cascadeIndex)).r;
      shadow += (projCoords.z - u_shadowBias) > depth ? 0.0 : 1.0;
    }
  }
  shadow /= 9.0;

  // Cascade blend (smooth transition between cascades)
  float blendDistance = u_cascadeSplits[cascadeIndex] * 0.1;  // 10% blend band
  float blendStart = u_cascadeSplits[cascadeIndex] - blendDistance;
  if (viewDepth > blendStart && cascadeIndex < 2) {
    float nextShadow = sampleCascade(worldPos, cascadeIndex + 1);
    float blendFactor = (viewDepth - blendStart) / blendDistance;
    shadow = mix(shadow, nextShadow, blendFactor);
  }

  return shadow;
}
```

### Shadow Bias

Two bias techniques to combat shadow acne:

1. **Depth bias**: Offset the comparison depth by a small constant
   ```glsl
   shadow += (projCoords.z - bias) > depth ? 0.0 : 1.0;
   ```

2. **Normal bias**: Offset the world position along the surface normal before projecting
   ```glsl
   vec3 biasedPos = worldPos + normal * normalBias * texelSize;
   ```

The combination of both biases eliminates acne without visible peter-panning (shadow detachment).

### Performance

Shadow pass rendering cost for 3 cascades:
- Each cascade renders shadow casters visible to that cascade's frustum
- Depth-only shader (no lighting calculations)
- Front-face culling (reversed from main pass)
- Only meshes with `castShadow: true` are rendered

For a 200×200m low-poly world:
- Near cascade: ~100 objects (small frustum, high detail)
- Mid cascade: ~500 objects
- Far cascade: ~1000 objects
- Total shadow draw calls: ~1600 (depth-only, very cheap)

### Uniform Upload

All shadow-related uniforms are part of the per-frame bind group (BindGroup 0):

```
u_cascadeVP[3]       — 3 × mat4 = 192 bytes
u_cascadeSplits[3]   — 3 × float = 12 bytes (padded to 16)
u_shadowBias         — float = 4 bytes
u_shadowNormalBias   — float = 4 bytes
u_lightDirection     — vec3 = 12 bytes (padded to 16)
u_lightColor         — vec3 = 12 bytes (padded to 16)
u_lightIntensity     — float = 4 bytes
```

Total: ~260 bytes, uploaded once per frame.
