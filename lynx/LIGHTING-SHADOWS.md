# Lighting and Cascading Shadow Maps

Lynx v1 ships two light types — directional and ambient — and a Lambert diffuse shading model. This combination is cheap, predictable, and a good fit for stylized and low-poly games. Shadow coverage over large worlds is handled by Cascaded Shadow Maps (CSM) with 3 cascades, PCF filtering, and stable texel snapping to prevent shimmer.

---

## Table of Contents

1. [Light Types](#1-light-types)
2. [Lambert Shading Model](#2-lambert-shading-model)
3. [Cascading Shadow Maps (CSM)](#3-cascading-shadow-maps-csm)
4. [Shadow Rendering](#4-shadow-rendering)
5. [Shadow Sampling](#5-shadow-sampling)
6. [WebGPU vs WebGL2 Differences](#6-webgpu-vs-webgl2-differences)
7. [Performance Considerations](#7-performance-considerations)

---

## 1. Light Types

### DirectionalLight

A directional light represents an infinitely distant light source where all rays are parallel (e.g., the sun). It has a direction, color, and intensity. The direction is derived from the node's rotation in the scene graph — the light points along the node's local -Y axis (forward in Z-up right-handed), transformed by the world rotation. Alternatively, a convenience setter accepts an explicit direction vector and computes the corresponding quaternion.

```typescript
interface DirectionalLight extends NodeBase {
  readonly type: "DirectionalLight"
  color: Vec3                    // linear RGB, default [1, 1, 1]
  intensity: number              // scalar multiplier, default 1.0
  shadow: ShadowConfig | null    // null = no shadow casting
}

interface ShadowConfig {
  enabled: boolean
  mapSize: number                // per-cascade resolution, e.g. 2048
  cascadeCount: number           // 3 for CSM (fixed in v1)
  bias: number                   // depth bias to prevent shadow acne, e.g. 0.0005
  slopeBias: number              // slope-scaled bias, e.g. 0.003
  near: number                   // shadow camera near plane
  far: number                    // shadow camera far plane (max shadow distance)
  cascadeSplits: Float32Array    // distances where cascades split (computed automatically or user-overridden)
  cascadeBlendRange: number      // world units over which adjacent cascades blend, e.g. 2.0
}

// Direction derived from node rotation
const getDirectionalLightDirection = (light: DirectionalLight, out: Vec3): Vec3 => {
  // Light's forward axis is -Y in local space (Z-up right-handed)
  // Rotate [0, -1, 0] by the node's world rotation
  const forward = vec3(0, -1, 0)
  return vec3TransformQuat(forward, quatFromMat4(light.worldMatrix), out)
}

// Convenience: set direction explicitly, compute rotation
const setDirectionalLightDirection = (light: DirectionalLight, direction: Vec3): void => {
  const normalized = vec3Normalize(direction)
  light.rotation = quatFromLookRotation(normalized, vec3(0, 0, 1))
  markDirty(light)
}
```

### AmbientLight

Ambient light provides a uniform base illumination from all directions. It is not a scene graph node because it has no spatial properties — no position, no direction, no rotation. It is stored as a property on the `Scene` object.

```typescript
interface AmbientLight {
  color: Vec3        // linear RGB, default [1, 1, 1]
  intensity: number  // scalar multiplier, default 0.3
}

interface Scene extends NodeBase {
  readonly type: "Scene"
  ambientLight: AmbientLight
  background: Vec3 | null
  fog: FogConfig | null
}

// Usage:
// scene.ambientLight = { color: vec3(0.6, 0.7, 1.0), intensity: 0.25 }
```

The ambient contribution is added to every fragment unconditionally: `ambientColor * ambientIntensity`. It fills in shadows and prevents completely black areas on surfaces facing away from the directional light.

### No Point or Spot Lights in v1

Point lights and spot lights are intentionally omitted from the first version. They introduce complexity in shadow mapping (cube maps for point, perspective projection for spot), per-light culling, and clustered/tiled forward shading. These can be added later without architectural changes — the uniform buffer and bind group layout already reserve space for additional light types.

---

## 2. Lambert Shading Model

### Why Lambert

Lambert is the simplest physically plausible diffuse model. It assumes light is scattered equally in all directions from the surface, producing a soft, matte appearance. There is no specular highlight. This is ideal for:

- **Stylized and low-poly games** where detailed specular response is not needed
- **Performance** — one dot product per light, no view-dependent terms
- **Predictability** — the look does not change as the camera moves around an object

### Shading Equation

```
diffuse = max(dot(N, L), 0.0) * lightColor * lightIntensity
ambient = ambientColor * ambientIntensity
lighting = ambient + diffuse * shadow
finalColor = lighting * materialColor
```

If the material has an emissive component:

```
finalColor += emissive * emissiveIntensity
```

The emissive term is additive and unaffected by lighting or shadows. It produces values greater than 1.0 in the HDR render target, which the bloom extraction pass picks up in post-processing.

### Complete Fragment Shader (GLSL 300 es)

This is the full `lambert.frag` shader used by the opaque pass. It handles texture sampling, vertex colors, CSM shadow lookup, and emissive output.

```glsl
#version 300 es
// --- Injected defines ---
// #define HAS_UV
// #define HAS_VERTEX_COLOR
// #define RECEIVE_SHADOWS
// #define NUM_CSM_CASCADES 3

precision highp float;

// --- Per-frame uniforms (bind group 0) ---
layout(std140) uniform FrameUniforms {
    mat4 u_viewMatrix;
    mat4 u_projectionMatrix;
    mat4 u_viewProjectionMatrix;
    vec3 u_cameraPosition;
    float u_time;
    vec3 u_sunDirection;           // normalized, world space, points toward the sun
    float u_padding0;
    vec3 u_sunColor;
    float u_sunIntensity;
    vec3 u_ambientColor;
    float u_ambientIntensity;
#ifdef RECEIVE_SHADOWS
    mat4 u_shadowMatrices[NUM_CSM_CASCADES];
    vec4 u_cascadeSplits;          // x=split0, y=split1, z=split2 (view-space distances)
    float u_shadowBias;
    float u_shadowSlopeBias;
    float u_shadowMapSize;         // e.g. 2048.0
    float u_cascadeBlendRange;
#endif
};

// --- Per-material uniforms (bind group 1) ---
layout(std140) uniform MaterialUniforms {
    vec4 u_baseColor;
    vec3 u_emissive;
    float u_emissiveIntensity;
};

#ifdef HAS_UV
uniform sampler2D u_baseColorMap;
#endif

#ifdef RECEIVE_SHADOWS
uniform sampler2DShadow u_shadowMap;   // 2D array texture or atlas
#endif

// --- Varyings ---
in vec3 v_worldPosition;
in vec3 v_worldNormal;
#ifdef HAS_UV
in vec2 v_uv;
#endif
#ifdef HAS_VERTEX_COLOR
in vec4 v_color;
#endif
#ifdef RECEIVE_SHADOWS
in vec4 v_shadowCoords[NUM_CSM_CASCADES];
#endif
in float v_viewDepth;

layout(location = 0) out vec4 fragColor;

// -------------------------------------------------
// Shadow sampling with 5x5 PCF
// -------------------------------------------------
#ifdef RECEIVE_SHADOWS
float sampleShadowPCF(int cascade, vec4 shadowCoord) {
    vec3 projected = shadowCoord.xyz / shadowCoord.w;
    projected = projected * 0.5 + 0.5;  // [-1,1] -> [0,1]

    // Out-of-bounds check
    if (projected.x < 0.0 || projected.x > 1.0 ||
        projected.y < 0.0 || projected.y > 1.0 ||
        projected.z > 1.0) {
        return 1.0;  // outside shadow map = fully lit
    }

    // Slope-scaled bias
    vec3 N = normalize(v_worldNormal);
    vec3 L = normalize(u_sunDirection);
    float cosTheta = max(dot(N, L), 0.0);
    float bias = u_shadowBias + u_shadowSlopeBias * (1.0 - cosTheta);
    projected.z -= bias;

    float texelSize = 1.0 / u_shadowMapSize;
    float shadow = 0.0;

    // 5x5 PCF kernel
    for (int x = -2; x <= 2; x++) {
        for (int y = -2; y <= 2; y++) {
            vec3 sampleCoord = vec3(
                projected.xy + vec2(float(x), float(y)) * texelSize,
                projected.z
            );
            shadow += texture(u_shadowMap, sampleCoord);
        }
    }
    return shadow / 25.0;
}

float computeShadow() {
    // Determine cascade based on view-space depth
    int cascade = NUM_CSM_CASCADES - 1;
    float cascadeSplits[3] = float[3](
        u_cascadeSplits.x,
        u_cascadeSplits.y,
        u_cascadeSplits.z
    );

    for (int i = 0; i < NUM_CSM_CASCADES; i++) {
        if (v_viewDepth < cascadeSplits[i]) {
            cascade = i;
            break;
        }
    }

    float shadow = sampleShadowPCF(cascade, v_shadowCoords[cascade]);

    // Cascade blending: blend with the next cascade near the boundary
    if (cascade < NUM_CSM_CASCADES - 1) {
        float distToEdge = cascadeSplits[cascade] - v_viewDepth;
        if (distToEdge < u_cascadeBlendRange) {
            float nextShadow = sampleShadowPCF(cascade + 1, v_shadowCoords[cascade + 1]);
            float blendFactor = distToEdge / u_cascadeBlendRange;
            shadow = mix(nextShadow, shadow, blendFactor);
        }
    }

    return shadow;
}
#endif

// -------------------------------------------------
// Main
// -------------------------------------------------
void main() {
    // --- Albedo ---
    vec4 albedo = u_baseColor;

#ifdef HAS_UV
    albedo *= texture(u_baseColorMap, v_uv);
#endif
#ifdef HAS_VERTEX_COLOR
    albedo *= v_color;
#endif

    // Alpha test (optional, for cutout materials)
    if (albedo.a < 0.01) discard;

    // --- Lambert diffuse ---
    vec3 N = normalize(v_worldNormal);
    vec3 L = normalize(u_sunDirection);

    float NdotL = max(dot(N, L), 0.0);
    vec3 diffuse = u_sunColor * u_sunIntensity * NdotL;

    // --- Ambient ---
    vec3 ambient = u_ambientColor * u_ambientIntensity;

    // --- Shadow ---
    float shadow = 1.0;
#ifdef RECEIVE_SHADOWS
    shadow = computeShadow();
#endif

    // --- Final color ---
    vec3 lighting = ambient + diffuse * shadow;
    vec3 color = albedo.rgb * lighting;

    // --- Emissive (feeds bloom) ---
    color += u_emissive * u_emissiveIntensity;

    fragColor = vec4(color, albedo.a);
}
```

### Lighting Data in the Frame Uniform Buffer

All light data is packed into the per-frame UBO (bind group 0). This keeps the lighting cost at zero additional bind group switches.

```typescript
interface FrameLightData {
  sunDirection: Vec3     // offset 0, normalized, world space
  sunColor: Vec3         // offset 16
  sunIntensity: number   // offset 28
  ambientColor: Vec3     // offset 32
  ambientIntensity: number // offset 44
}

const packLightUniforms = (
  light: DirectionalLight,
  ambient: AmbientLight,
  buffer: Float32Array,
  offset: number
): void => {
  const dir = getDirectionalLightDirection(light, tempVec3)

  // Sun direction (points toward the sun, used as L in N dot L)
  buffer[offset]     = dir[0]
  buffer[offset + 1] = dir[1]
  buffer[offset + 2] = dir[2]
  buffer[offset + 3] = 0  // padding

  // Sun color
  buffer[offset + 4] = light.color[0]
  buffer[offset + 5] = light.color[1]
  buffer[offset + 6] = light.color[2]
  buffer[offset + 7] = light.intensity

  // Ambient
  buffer[offset + 8]  = ambient.color[0]
  buffer[offset + 9]  = ambient.color[1]
  buffer[offset + 10] = ambient.color[2]
  buffer[offset + 11] = ambient.intensity
}
```

---

## 3. Cascading Shadow Maps (CSM)

### Why CSM

A directional light casts shadows across the entire visible world. A single shadow map covering 200x200 meters at 2048x2048 resolution gives approximately 10 cm per texel — far too blocky for objects near the camera where the player expects crisp, detailed shadows.

Cascaded Shadow Maps solve this by splitting the camera's view frustum into multiple depth ranges (cascades), each with its own shadow map. Near cascades cover a small area at high resolution; far cascades cover a large area at lower effective resolution. The eye perceives this as uniformly high-quality shadows because nearby detail gets more texels.

```
Side view of the camera frustum split into 3 cascades:

         Camera
           *
          /|\
         / | \
        /  |  \
       /   |   \
      /    |    \
     / C0  | C1  \   C2
    /  near| mid   \  far
   / 0-15m |15-50m  \ 50-200m
  /________|_________\________
  |        |         |        |
  |  2048  |  2048   |  2048  |   each cascade: 2048x2048 shadow map
  |________|_________|________|
```

### Cascade Split Distances

Lynx uses 3 cascades with a practical logarithmic/linear blend for split distances. The logarithmic scheme gives more resolution to near distances (where detail matters most), while the linear component prevents the near cascade from being too small.

```
Default cascade splits:
  Cascade 0:  near  ->  15 m   (high detail near camera)
  Cascade 1:  15 m  ->  50 m   (medium detail)
  Cascade 2:  50 m  -> 200 m   (far distance, lower detail but still covered)
```

```typescript
interface CascadeSplitConfig {
  cascadeCount: number       // 3
  near: number               // camera near plane
  far: number                // max shadow distance (e.g. 200)
  lambda: number             // blend factor: 0 = linear, 1 = logarithmic (default 0.7)
}

const computeCascadeSplits = (
  config: CascadeSplitConfig,
  out: Float32Array
): Float32Array => {
  const { cascadeCount, near, far, lambda } = config

  for (let i = 1; i <= cascadeCount; i++) {
    const t = i / cascadeCount

    // Logarithmic split: near * (far / near)^t
    const log = near * Math.pow(far / near, t)

    // Linear split: near + (far - near) * t
    const lin = near + (far - near) * t

    // Blend between logarithmic and linear
    out[i - 1] = lambda * log + (1.0 - lambda) * lin
  }

  return out
}

// Example with near=0.5, far=200, lambda=0.7:
//   out[0] = 14.8  (~15m)
//   out[1] = 52.3  (~50m)
//   out[2] = 200.0 (far)
```

The split distances are tunable. For smaller scenes the far distance can be reduced (e.g. 50m with splits at 5m, 15m, 50m). For open-world scenes it can be extended.

### Shadow Map Resolution and Storage

Each cascade renders into a 2048x2048 depth texture. The three cascades are stored as a 2D texture array with 3 layers (one layer per cascade). On WebGL2 backends that lack texture array support, a 6144x2048 atlas is used with manual UV offset computation.

```typescript
interface ShadowMapAtlas {
  texture: GalTexture
  cascadeCount: number
  resolution: number        // per-cascade, e.g. 2048
  useTextureArray: boolean  // true on WebGPU and most WebGL2 devices
}

const createShadowMapAtlas = (
  device: GalDevice,
  config: ShadowConfig
): ShadowMapAtlas => {
  const resolution = config.mapSize  // default 2048
  const cascadeCount = config.cascadeCount // 3

  if (device.limits.maxTextureLayers >= cascadeCount) {
    // Texture array: one layer per cascade
    return {
      texture: device.createTexture({
        label: "csm-shadow-map",
        dimension: "2d-array",
        format: "depth32float",
        width: resolution,
        height: resolution,
        depthOrArrayLayers: cascadeCount,
        usage: ["render-attachment", "texture-binding"],
      }),
      cascadeCount,
      resolution,
      useTextureArray: true,
    }
  }

  // Fallback: atlas (cascades side by side)
  return {
    texture: device.createTexture({
      label: "csm-shadow-atlas",
      dimension: "2d",
      format: "depth32float",
      width: resolution * cascadeCount,
      height: resolution,
      usage: ["render-attachment", "texture-binding"],
    }),
    cascadeCount,
    resolution,
    useTextureArray: false,
  }
}
```

### Light-Space Matrices

For each cascade, a tight orthographic projection is computed that fits the cascade's portion of the camera frustum. The projection is snapped to texel boundaries to prevent shadow shimmering when the camera moves.

```
Top-down view: fitting the light-space ortho box around a cascade frustum slice

      Light direction
          ↓  ↓  ↓  ↓
    ┌─────────────────────┐
    │  Orthographic       │
    │  projection box     │
    │   ┌───────────┐     │
    │   │  Cascade  │     │
    │   │  frustum  │     │
    │   │  slice    │     │
    │   └───────────┘     │
    │                     │
    └─────────────────────┘

    The ortho box is aligned to the light direction
    and tightly encloses the cascade's sub-frustum.
```

```typescript
interface CascadeData {
  viewProjection: Mat4      // light-space VP matrix for this cascade
  splitNear: number         // view-space near distance
  splitFar: number          // view-space far distance
  texelSize: number         // world units per shadow texel (for snapping)
}

const computeCascadeLightMatrix = (
  camera: Camera,
  light: DirectionalLight,
  splitNear: number,
  splitFar: number,
  resolution: number
): CascadeData => {
  // 1. Compute the 8 corners of the cascade sub-frustum in world space
  const frustumCorners = computeFrustumCornersWorldSpace(
    camera.viewMatrix,
    camera.projectionMatrix,
    splitNear,
    splitFar
  )

  // 2. Compute the center of the frustum slice
  const center = vec3(0, 0, 0)
  for (let i = 0; i < 8; i++) {
    vec3Add(center, frustumCorners[i], center)
  }
  vec3Scale(center, 1.0 / 8.0, center)

  // 3. Build light view matrix: look from above along the light direction
  const lightDir = getDirectionalLightDirection(light, tempVec3A)
  const lightUp = vec3(0, 0, 1)  // Z-up

  // Handle degenerate case: light pointing straight up/down
  if (Math.abs(vec3Dot(lightDir, lightUp)) > 0.999) {
    vec3Set(lightUp, 0, 1, 0)
  }

  const lightView = mat4LookAt(
    vec3Sub(center, vec3Scale(lightDir, 50, tempVec3B), tempVec3C), // eye (offset from center)
    center,                                                         // target
    lightUp
  )

  // 4. Transform frustum corners to light space and compute bounds
  let minX = Infinity, maxX = -Infinity
  let minY = Infinity, maxY = -Infinity
  let minZ = Infinity, maxZ = -Infinity

  for (let i = 0; i < 8; i++) {
    const p = vec3TransformMat4(frustumCorners[i], lightView, tempVec3D)
    minX = Math.min(minX, p[0])
    maxX = Math.max(maxX, p[0])
    minY = Math.min(minY, p[1])
    maxY = Math.max(maxY, p[1])
    minZ = Math.min(minZ, p[2])
    maxZ = Math.max(maxZ, p[2])
  }

  // 5. Extend Z range to capture shadow casters behind the frustum
  const zExtend = 100.0  // extend backwards to catch tall buildings etc.
  minZ -= zExtend

  // 6. Snap to texel grid to prevent shadow shimmering
  const worldUnitsPerTexel = Math.max(maxX - minX, maxY - minY) / resolution

  minX = Math.floor(minX / worldUnitsPerTexel) * worldUnitsPerTexel
  maxX = Math.floor(maxX / worldUnitsPerTexel) * worldUnitsPerTexel
  minY = Math.floor(minY / worldUnitsPerTexel) * worldUnitsPerTexel
  maxY = Math.floor(maxY / worldUnitsPerTexel) * worldUnitsPerTexel

  // 7. Build orthographic projection
  const lightProjection = mat4Orthographic(minX, maxX, minY, maxY, minZ, maxZ)
  const lightViewProjection = mat4Multiply(lightProjection, lightView)

  return {
    viewProjection: lightViewProjection,
    splitNear,
    splitFar,
    texelSize: worldUnitsPerTexel,
  }
}
```

### Stable Cascades (Texel Snapping)

Without snapping, the light-space projection shifts by sub-texel amounts as the camera moves. This causes shadow edges to "swim" — a distracting flickering artifact. The fix is to quantize the light-space bounds to shadow map texel boundaries:

```typescript
// Quantize to shadow texel grid
const stabilizeCascadeProjection = (
  minX: number,
  maxX: number,
  minY: number,
  maxY: number,
  resolution: number
): { minX: number, maxX: number, minY: number, maxY: number } => {
  // Compute the size of the ortho frustum in light-space units
  const sizeX = maxX - minX
  const sizeY = maxY - minY

  // Size of one texel in light-space units
  const texelSizeX = sizeX / resolution
  const texelSizeY = sizeY / resolution

  // Snap min to texel grid (floor) and recompute max to preserve size
  const snappedMinX = Math.floor(minX / texelSizeX) * texelSizeX
  const snappedMinY = Math.floor(minY / texelSizeY) * texelSizeY

  return {
    minX: snappedMinX,
    maxX: snappedMinX + sizeX,
    minY: snappedMinY,
    maxY: snappedMinY + sizeY,
  }
}
```

This ensures that the shadow map projection always aligns to integer texel boundaries. As the camera translates, the projection jumps by whole texels rather than sliding continuously, eliminating shimmer.

### Frustum Corner Computation

The 8 corners of the cascade sub-frustum are computed by unprojecting NDC corners at the cascade's near and far depths:

```typescript
const computeFrustumCornersWorldSpace = (
  viewMatrix: Mat4,
  projectionMatrix: Mat4,
  splitNear: number,
  splitFar: number
): Vec3[] => {
  const viewProj = mat4Multiply(projectionMatrix, viewMatrix)
  const invViewProj = mat4Invert(viewProj)

  // NDC corners: 4 at near plane (z=0 in [0,1] depth), 4 at far plane (z=1)
  const ndcCorners = [
    [-1, -1, 0], [ 1, -1, 0], [ 1,  1, 0], [-1,  1, 0],  // near
    [-1, -1, 1], [ 1, -1, 1], [ 1,  1, 1], [-1,  1, 1],  // far
  ]

  // Unproject all 8 corners to world space at the full frustum extents
  const worldCorners: Vec3[] = []
  for (const ndc of ndcCorners) {
    const clip = vec4(ndc[0], ndc[1], ndc[2] * 2 - 1, 1) // adjust for [-1,1] depth range
    const world = vec4TransformMat4(clip, invViewProj)
    worldCorners.push(vec3(world[0] / world[3], world[1] / world[3], world[2] / world[3]))
  }

  // Interpolate between near and far corners to get the cascade sub-frustum
  const result: Vec3[] = []
  const cameraNear = 0.1  // from camera config
  const cameraFar = 500   // from camera config
  const tNear = (splitNear - cameraNear) / (cameraFar - cameraNear)
  const tFar = (splitFar - cameraNear) / (cameraFar - cameraNear)

  for (let i = 0; i < 4; i++) {
    const nearCorner = worldCorners[i]
    const farCorner = worldCorners[i + 4]

    // Near plane of this cascade
    result.push(vec3Lerp(nearCorner, farCorner, tNear))
    // Far plane of this cascade
    result.push(vec3Lerp(nearCorner, farCorner, tFar))
  }

  return result
}
```

### Cascade Frustum Diagram

```
Plan view (looking down the Z axis) of the camera frustum split into 3 cascades:

                          Camera position
                               *
                              / \
                             /   \
                            / C0  \           Cascade 0: 0-15m
                           /  15m  \          High resolution
                          /─────────\         Covers objects near the player
                         /           \
                        /    C1       \       Cascade 1: 15-50m
                       /    50m        \      Medium resolution
                      /─────────────────\     Most gameplay-relevant objects
                     /                   \
                    /        C2           \   Cascade 2: 50-200m
                   /        200m           \  Lower resolution
                  /─────────────────────────\ Distant terrain, large structures
                 /                           \

Each cascade has its own 2048x2048 shadow map:
  C0: covers ~15m  -> ~0.7cm per texel  (sharp shadows on the player's feet)
  C1: covers ~35m  -> ~1.7cm per texel  (crisp shadows on nearby buildings)
  C2: covers ~150m -> ~7.3cm per texel  (visible shadows on distant terrain)
```

---

## 4. Shadow Rendering

### Shadow Pass Overview

The shadow pass runs before the opaque and transparency passes. For each cascade, it renders all shadow-casting meshes from the light's perspective into the cascade's region of the shadow map using a depth-only shader.

```
Frame flow:

  ┌─────────────────────────────────────────────────────┐
  │ Pass 0: Shadow Maps                                  │
  │                                                      │
  │   for each cascade (0, 1, 2):                        │
  │     1. Compute light-space VP matrix                 │
  │     2. Set viewport / array layer                    │
  │     3. Cull shadow casters against cascade frustum   │
  │     4. Render depth-only with front-face culling     │
  │                                                      │
  ├─────────────────────────────────────────────────────┤
  │ Pass 1: Opaque (reads shadow maps as textures)       │
  ├─────────────────────────────────────────────────────┤
  │ Pass 2: Transparent (OIT)                            │
  ├─────────────────────────────────────────────────────┤
  │ Pass 3-5: Post-processing...                         │
  └─────────────────────────────────────────────────────┘
```

### Shadow Pass Execution

```typescript
const executeShadowPass = (
  device: GalDevice,
  encoder: GalCommandEncoder,
  atlas: ShadowMapAtlas,
  cascades: CascadeData[],
  shadowCasters: RenderCommand[],
  shadowPipeline: GalRenderPipeline,
  shadowUBO: GalBuffer
): void => {
  for (let cascade = 0; cascade < atlas.cascadeCount; cascade++) {
    const cascadeData = cascades[cascade]

    // Write the light-space VP matrix to the shadow UBO
    device.writeBuffer(shadowUBO, cascadeData.viewProjection, 0)

    // Create render pass targeting this cascade's layer/region
    const depthView = atlas.useTextureArray
      ? atlas.texture.createView({ baseArrayLayer: cascade, arrayLayerCount: 1 })
      : atlas.texture.createView()  // atlas: use viewport to select region

    const pass = encoder.beginRenderPass({
      label: `shadow-cascade-${cascade}`,
      colorAttachments: [],  // depth-only pass, no color output
      depthStencilAttachment: {
        view: depthView,
        depthLoadOp: "clear",
        depthStoreOp: "store",
        depthClearValue: 1.0,
      },
    })

    // If using atlas layout, set viewport to cascade region
    if (!atlas.useTextureArray) {
      pass.setViewport(
        cascade * atlas.resolution, 0,
        atlas.resolution, atlas.resolution,
        0.0, 1.0
      )
    }

    pass.setPipeline(shadowPipeline)
    pass.setBindGroup(0, device.createBindGroup({
      layout: shadowPipeline.layout,
      entries: [
        { binding: 0, resource: { type: "buffer", buffer: shadowUBO } },
      ],
    }))

    // Render each shadow caster
    for (let i = 0; i < shadowCasters.length; i++) {
      const cmd = shadowCasters[i]

      // Per-cascade frustum culling (separate from camera frustum cull)
      if (!isAABBInFrustum(cmd.worldAABB, cascadeData.frustum)) continue

      pass.setBindGroup(2, cmd.objectBindGroup, [cmd.uniformOffset])
      pass.setVertexBuffer(0, cmd.vertexBuffer)
      pass.setIndexBuffer(cmd.indexBuffer, "uint16")
      pass.drawIndexed(cmd.indexCount)
    }

    pass.end()
  }
}
```

### Shadow Caster Culling

Each cascade has its own frustum (the light-space orthographic box). Shadow casters are culled against each cascade's frustum independently. An object that is visible in cascade 0 might not be visible in cascade 2, and vice versa. This culling is separate from the camera's view frustum cull.

```typescript
const cullShadowCasters = (
  allCasters: RenderCommand[],
  cascadeFrustum: FrustumPlanes,
  result: RenderCommand[]
): void => {
  result.length = 0
  for (let i = 0; i < allCasters.length; i++) {
    const cmd = allCasters[i]
    if (!cmd.mesh.castShadow) continue
    if (isAABBInFrustum(cmd.worldAABB, cascadeFrustum)) {
      result.push(cmd)
    }
  }
}
```

### Front-Face Culling (Peter Panning Fix)

During the shadow pass, normal back-face culling is reversed: front faces are culled instead. This means the shadow map records the depth of back faces rather than front faces. The benefit is that shadow acne (self-shadowing artifacts caused by depth buffer precision) is largely eliminated because the depth comparison happens against the back surface, which is behind the lit front surface. The trade-off is a small outward shift of shadows (Peter Panning), but with reasonable bias values this is imperceptible.

```typescript
// Shadow pipeline uses front-face culling
const shadowPipelineDescriptor: GalRenderPipelineDescriptor = {
  label: "shadow-depth-pipeline",
  vertex: {
    module: shadowVertexShader,
    entryPoint: "main",
    buffers: [positionOnlyLayout],
  },
  fragment: {
    module: shadowFragmentShader,
    entryPoint: "main",
    targets: [],  // no color output
  },
  depthStencil: {
    format: "depth32float",
    depthWriteEnabled: true,
    depthCompare: "less",
  },
  primitive: {
    topology: "triangle-list",
    cullMode: "front",  // <-- front-face culling: renders back faces into shadow map
    frontFace: "ccw",
  },
}
```

### Shadow Depth Shader

The shadow vertex shader is minimal — it transforms positions to light-space clip coordinates. The fragment shader is empty because only depth output is needed.

```glsl
// shadow.vert (GLSL 300 es)
#version 300 es

layout(std140) uniform ShadowUniforms {
    mat4 u_lightViewProjection;
};

layout(std140) uniform ObjectUniforms {
    mat4 u_worldMatrix;
};

layout(location = 0) in vec3 a_position;

#ifdef HAS_SKINNING
layout(location = 4) in vec4 a_joints;
layout(location = 5) in vec4 a_weights;

layout(std140) uniform BoneUniforms {
    mat4 u_boneMatrices[NUM_BONES];
};
#endif

void main() {
    vec4 localPos = vec4(a_position, 1.0);

#ifdef HAS_SKINNING
    mat4 skinMatrix =
        u_boneMatrices[int(a_joints.x)] * a_weights.x +
        u_boneMatrices[int(a_joints.y)] * a_weights.y +
        u_boneMatrices[int(a_joints.z)] * a_weights.z +
        u_boneMatrices[int(a_joints.w)] * a_weights.w;
    localPos = skinMatrix * localPos;
#endif

    gl_Position = u_lightViewProjection * u_worldMatrix * localPos;
}

// shadow.frag (GLSL 300 es)
#version 300 es
precision highp float;
void main() {
    // Depth written automatically by the rasterizer.
    // No color output — this is a depth-only pass.
}
```

---

## 5. Shadow Sampling

### PCF (Percentage Closer Filtering)

Raw shadow map lookups produce hard, aliased shadow edges. PCF smooths them by sampling multiple points around the shadow coordinate and averaging the comparison results. Lynx uses a 5x5 kernel (25 samples) for soft shadow edges.

```
5x5 PCF kernel layout around the sample point (center):

    . . . . .
    . . . . .
    . . X . .      X = fragment's shadow coordinate
    . . . . .      . = offset sample positions
    . . . . .

Each sample tests: is this texel in shadow?
Average of 25 results = smooth shadow factor [0, 1]
```

### Cascade Selection

The fragment shader determines which cascade a fragment belongs to by comparing the fragment's view-space depth against the cascade split distances. View-space depth is computed in the vertex shader as `-viewPosition.z` (positive into the screen in a right-handed system) and interpolated to the fragment.

```
View-space depth:

    0m          15m          50m             200m
    |-----C0-----|-----C1-----|------C2-------|

    Fragment at depth 12m  -> cascade 0
    Fragment at depth 30m  -> cascade 1
    Fragment at depth 120m -> cascade 2
```

### Cascade Blending

Without blending, there is a hard visible line where one cascade meets the next — shadow resolution changes abruptly. Lynx blends between adjacent cascades over a configurable range (default 2 meters) near each cascade boundary.

```
Cascade blending zone:

    |----- Cascade 0 -----|----- Cascade 1 -----|
                     13m  15m  17m
                      |blend|
                      zone

    At 13m: 100% cascade 0
    At 14m: 50% cascade 0, 50% cascade 1
    At 15m: 0% cascade 0, 100% cascade 1
```

### Complete Shadow Sampling Shader Code

#### GLSL 300 es (WebGL2)

```glsl
// Shadow sampling functions used in lambert.frag
// Assumes shadow map is bound as sampler2DShadow (hardware comparison)

#ifdef RECEIVE_SHADOWS

// Sample a single cascade with 5x5 PCF
float sampleCascadePCF(int cascade, vec4 shadowCoord) {
    // Perspective divide (orthographic: w is always 1.0, but keep for correctness)
    vec3 projected = shadowCoord.xyz / shadowCoord.w;

    // Map from clip space [-1, 1] to texture space [0, 1]
    projected = projected * 0.5 + 0.5;

    // Discard samples outside the shadow map
    if (projected.x < 0.0 || projected.x > 1.0 ||
        projected.y < 0.0 || projected.y > 1.0 ||
        projected.z > 1.0) {
        return 1.0;
    }

    // Slope-scaled bias: surfaces nearly parallel to the light need more bias
    vec3 N = normalize(v_worldNormal);
    vec3 L = normalize(u_sunDirection);
    float cosTheta = max(dot(N, L), 0.001);
    float bias = u_shadowBias + u_shadowSlopeBias * tan(acos(cosTheta));
    bias = min(bias, 0.01);  // clamp to avoid excessive bias

    float currentDepth = projected.z - bias;
    float texelSize = 1.0 / u_shadowMapSize;
    float shadow = 0.0;

    // 5x5 PCF kernel
    for (int x = -2; x <= 2; x++) {
        for (int y = -2; y <= 2; y++) {
            vec2 offset = vec2(float(x), float(y)) * texelSize;
            // sampler2DShadow: texture() returns 0.0 or 1.0 based on depth comparison
            shadow += texture(u_shadowMap,
                vec3(projected.xy + offset, currentDepth));
        }
    }

    return shadow / 25.0;
}

// Determine cascade and sample with blending
float computeShadowFactor() {
    // Find the cascade for this fragment
    int cascade = NUM_CSM_CASCADES - 1;
    for (int i = 0; i < NUM_CSM_CASCADES; i++) {
        if (v_viewDepth < u_cascadeSplits[i]) {
            cascade = i;
            break;
        }
    }

    float shadow = sampleCascadePCF(cascade, v_shadowCoords[cascade]);

    // Blend with the next cascade near the boundary
    if (cascade < NUM_CSM_CASCADES - 1) {
        float edgeDist = u_cascadeSplits[cascade] - v_viewDepth;
        float blendRange = u_cascadeBlendRange;

        if (edgeDist < blendRange) {
            float nextShadow = sampleCascadePCF(cascade + 1, v_shadowCoords[cascade + 1]);
            float t = edgeDist / blendRange;  // 1.0 at cascade center, 0.0 at boundary
            shadow = mix(nextShadow, shadow, t);
        }
    }

    return shadow;
}
#endif
```

#### WGSL (WebGPU)

```wgsl
// Shadow sampling for WebGPU using comparison sampler and texture array

// Bind group 3: shadow resources
@group(3) @binding(0) var shadowMap: texture_depth_2d_array;
@group(3) @binding(1) var shadowSampler: sampler_comparison;

struct ShadowUniforms {
    cascadeSplits: vec4<f32>,        // x=split0, y=split1, z=split2
    shadowBias: f32,
    shadowSlopeBias: f32,
    shadowMapSize: f32,
    cascadeBlendRange: f32,
    shadowMatrices: array<mat4x4<f32>, 3>,
}
@group(3) @binding(2) var<uniform> shadow: ShadowUniforms;

fn sampleCascadePCF(cascade: i32, shadowCoord: vec4<f32>, worldNormal: vec3<f32>, sunDir: vec3<f32>) -> f32 {
    // Perspective divide
    let projected = shadowCoord.xyz / shadowCoord.w;
    let uv = projected.xy * 0.5 + 0.5;
    let uvFlipped = vec2<f32>(uv.x, 1.0 - uv.y);  // flip Y for texture coordinates

    // Out-of-bounds check
    if (uvFlipped.x < 0.0 || uvFlipped.x > 1.0 ||
        uvFlipped.y < 0.0 || uvFlipped.y > 1.0 ||
        projected.z > 1.0) {
        return 1.0;
    }

    // Slope-scaled bias
    let cosTheta = max(dot(normalize(worldNormal), normalize(sunDir)), 0.001);
    let bias = shadow.shadowBias + shadow.shadowSlopeBias * (1.0 - cosTheta);
    let currentDepth = projected.z - min(bias, 0.01);

    let texelSize = 1.0 / shadow.shadowMapSize;
    var result: f32 = 0.0;

    // 5x5 PCF kernel using textureSampleCompare
    for (var x: i32 = -2; x <= 2; x = x + 1) {
        for (var y: i32 = -2; y <= 2; y = y + 1) {
            let offset = vec2<f32>(f32(x), f32(y)) * texelSize;
            result += textureSampleCompare(
                shadowMap,
                shadowSampler,
                uvFlipped + offset,
                cascade,             // array layer index
                currentDepth
            );
        }
    }

    return result / 25.0;
}

fn computeShadowFactor(viewDepth: f32, worldPos: vec3<f32>, worldNormal: vec3<f32>, sunDir: vec3<f32>) -> f32 {
    // Select cascade
    var cascade: i32 = 2;
    let splits = shadow.cascadeSplits;

    if (viewDepth < splits.x) {
        cascade = 0;
    } else if (viewDepth < splits.y) {
        cascade = 1;
    }

    // Transform world position to shadow space for this cascade
    let shadowCoord = shadow.shadowMatrices[cascade] * vec4<f32>(worldPos, 1.0);
    var result = sampleCascadePCF(cascade, shadowCoord, worldNormal, sunDir);

    // Cascade blending
    let splitDist = select(
        select(splits.z, splits.y, cascade == 1),
        splits.x,
        cascade == 0
    );
    let edgeDist = splitDist - viewDepth;

    if (cascade < 2 && edgeDist < shadow.cascadeBlendRange) {
        let nextCoord = shadow.shadowMatrices[cascade + 1] * vec4<f32>(worldPos, 1.0);
        let nextShadow = sampleCascadePCF(cascade + 1, nextCoord, worldNormal, sunDir);
        let t = edgeDist / shadow.cascadeBlendRange;
        result = mix(nextShadow, result, t);
    }

    return result;
}
```

### Shadow Bias

Shadow acne appears when a surface shadows itself due to limited depth buffer precision. The depth of a shadow map texel does not perfectly match the depth of the surface it was rendered from, causing alternating lit and shadowed fragments across flat surfaces.

Two bias strategies are combined:

1. **Constant bias** (`shadowBias`): A fixed depth offset subtracted from the fragment's depth before comparison. Typical value: 0.0005.

2. **Slope-scaled bias** (`shadowSlopeBias`): An additional offset proportional to the surface's slope relative to the light. Surfaces nearly edge-on to the light need more bias. Typical value: 0.003.

```
Without bias:                    With bias:
   ┌─────────────────┐           ┌─────────────────┐
   │ ████░░░░████░░░░│           │                 │
   │ ░░░░████░░░░████│  acne     │   (clean)       │
   │ ████░░░░████░░░░│           │                 │
   └─────────────────┘           └─────────────────┘
```

---

## 6. WebGPU vs WebGL2 Differences

The shadow system works on both backends. The core algorithm (cascade splitting, frustum fitting, PCF sampling) is identical. The differences lie in how the shadow map texture is stored and sampled.

### Texture Storage

| Feature | WebGPU | WebGL2 |
|---|---|---|
| Shadow texture | `texture_depth_2d_array` (3 layers) | `sampler2DArray` or atlas `sampler2D` |
| Cascade selection | Array layer index in `textureSampleCompare` | `gl.framebufferTextureLayer()` for render, layer index for sample |
| Comparison sampler | `sampler_comparison` with `compare: "less"` | `sampler2DShadow` with `TEXTURE_COMPARE_MODE = COMPARE_REF_TO_TEXTURE` |

### Shadow Map Creation

```typescript
// WebGPU: depth texture array with comparison sampler
const createShadowResourcesWebGPU = (device: GPUDevice, resolution: number, cascadeCount: number) => {
  const texture = device.createTexture({
    size: [resolution, resolution, cascadeCount],
    format: "depth32float",
    usage: GPUTextureUsage.RENDER_ATTACHMENT | GPUTextureUsage.TEXTURE_BINDING,
  })

  const sampler = device.createSampler({
    compare: "less",
    magFilter: "linear",
    minFilter: "linear",
  })

  return { texture, sampler }
}

// WebGL2: depth texture array with shadow comparison
const createShadowResourcesWebGL2 = (gl: WebGL2RenderingContext, resolution: number, cascadeCount: number) => {
  const texture = gl.createTexture()
  gl.bindTexture(gl.TEXTURE_2D_ARRAY, texture)
  gl.texStorage3D(gl.TEXTURE_2D_ARRAY, 1, gl.DEPTH_COMPONENT32F, resolution, resolution, cascadeCount)

  // Enable hardware shadow comparison
  gl.texParameteri(gl.TEXTURE_2D_ARRAY, gl.TEXTURE_COMPARE_MODE, gl.COMPARE_REF_TO_TEXTURE)
  gl.texParameteri(gl.TEXTURE_2D_ARRAY, gl.TEXTURE_COMPARE_FUNC, gl.LEQUAL)
  gl.texParameteri(gl.TEXTURE_2D_ARRAY, gl.TEXTURE_MIN_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D_ARRAY, gl.TEXTURE_MAG_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D_ARRAY, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE)
  gl.texParameteri(gl.TEXTURE_2D_ARRAY, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE)

  // Framebuffer for rendering each cascade layer
  const framebuffers: WebGLFramebuffer[] = []
  for (let i = 0; i < cascadeCount; i++) {
    const fbo = gl.createFramebuffer()!
    gl.bindFramebuffer(gl.FRAMEBUFFER, fbo)
    gl.framebufferTextureLayer(gl.FRAMEBUFFER, gl.DEPTH_ATTACHMENT, texture, 0, i)
    framebuffers.push(fbo)
  }

  return { texture, framebuffers }
}
```

### Sampling Differences

```typescript
// WebGPU: uses textureSampleCompare on a depth texture array
// The comparison sampler performs the depth test in hardware.
//   textureSampleCompare(shadowMap, shadowSampler, uv, arrayLayer, depthRef)
// Returns 0.0 (in shadow) or 1.0 (lit), with hardware PCF on some GPUs.

// WebGL2: uses texture() on a sampler2DShadow bound to a 2D array texture
// The comparison mode is set on the texture, not the sampler.
//   texture(u_shadowMap, vec4(uv, layer, depthRef))
// For sampler2DArrayShadow, the lookup is:
//   texture(u_shadowMap, vec4(uv.x, uv.y, float(layer), depthRef))
```

### Backend Abstraction

The GAL hides these differences. The renderer creates a shadow texture and sampler through the GAL:

```typescript
const createShadowResources = (device: GalDevice, resolution: number, cascadeCount: number) => {
  const texture = device.createTexture({
    label: "csm-shadow-map",
    dimension: "2d-array",
    format: "depth32float",
    width: resolution,
    height: resolution,
    depthOrArrayLayers: cascadeCount,
    usage: ["render-attachment", "texture-binding"],
  })

  const sampler = device.createSampler({
    label: "shadow-comparison-sampler",
    compare: "less-equal",
    magFilter: "linear",
    minFilter: "linear",
  })

  return { texture, sampler }
}
```

The WebGPU backend creates `GPUTexture` + `GPUSampler`. The WebGL2 backend creates `TEXTURE_2D_ARRAY` + configures `TEXTURE_COMPARE_MODE`. The shader code uses the appropriate syntax (WGSL `textureSampleCompare` or GLSL `texture` with `sampler2DArrayShadow`), but the logic — cascade selection, PCF kernel, bias, blending — is identical.

---

## 7. Performance Considerations

### Shadow Map Rendering Is GPU-Limited

Shadow pass rendering is almost entirely GPU-bound. The depth-only shader is trivial (just a matrix multiply), so the bottleneck is rasterization throughput and memory bandwidth to the depth buffer. Draw call overhead is low because:

- The shadow pipeline is the same for every object (one `setPipeline` call per cascade)
- No material bind group changes (depth-only shader ignores materials)
- Only the per-object bind group (world matrix) changes per draw

### Aggressive Per-Cascade Culling

Each cascade's light-space frustum is typically much smaller than the camera frustum. Culling shadow casters against each cascade independently avoids rendering distant objects into the near cascade and nearby objects into the far cascade:

```
Without per-cascade cull:           With per-cascade cull:
  Cascade 0: renders 800 objects     Cascade 0: renders 120 objects
  Cascade 1: renders 800 objects     Cascade 1: renders 350 objects
  Cascade 2: renders 800 objects     Cascade 2: renders 600 objects
  Total: 2400 draw calls             Total: 1070 draw calls
```

### Static Shadow Map Caching

For scenes with no moving shadow casters, the shadow maps can be cached and reused across frames. The renderer tracks a "shadow dirty" flag that is set when:

- A shadow caster's world matrix changes
- A shadow caster is added or removed
- The directional light's direction changes
- The camera moves enough to require cascade recomputation

When the flag is clear, the shadow pass is skipped entirely — the opaque pass reads the cached shadow maps from the previous frame.

```typescript
interface ShadowCache {
  dirty: boolean
  lastCameraPosition: Vec3
  lastCameraRotation: Quat
  lastLightDirection: Vec3
  cameraMovementThreshold: number  // meters — re-render if camera moves more than this
}

const shouldUpdateShadows = (cache: ShadowCache, camera: Camera, light: DirectionalLight): boolean => {
  if (cache.dirty) return true

  // Check if camera moved significantly
  const cameraDelta = vec3Distance(camera.position, cache.lastCameraPosition)
  if (cameraDelta > cache.cameraMovementThreshold) return true

  // Check if light direction changed
  const lightDir = getDirectionalLightDirection(light, tempVec3)
  if (vec3Distance(lightDir, cache.lastLightDirection) > 0.001) return true

  return false
}
```

### Resolution Scaling for Lower-End Devices

Shadow map resolution can be reduced for devices with lower GPU capabilities. The engine queries `device.limits.maxTextureSize` and scales accordingly:

```typescript
const chooseShadowResolution = (device: GalDevice, preferredSize: number): number => {
  const maxSize = device.limits.maxTextureSize

  // Ensure resolution fits within device limits
  let resolution = Math.min(preferredSize, maxSize)

  // On low-end devices, reduce to save bandwidth
  if (maxSize <= 4096) {
    resolution = Math.min(resolution, 1024)
  }

  return resolution
}
```

### Budget Breakdown

Typical shadow pass timing for a scene with 500 shadow-casting meshes at 2048x2048 per cascade:

| Stage | Desktop GPU | Mobile GPU |
|---|---|---|
| Cascade 0 (near, ~120 casters) | 0.15 ms | 0.5 ms |
| Cascade 1 (mid, ~250 casters) | 0.25 ms | 0.8 ms |
| Cascade 2 (far, ~400 casters) | 0.35 ms | 1.2 ms |
| PCF sampling (opaque pass) | 0.20 ms | 0.6 ms |
| **Total shadow cost** | **~0.95 ms** | **~3.1 ms** |

On mobile, shadows consume roughly 3 ms of the 16.6 ms frame budget (60 fps). If this is too expensive, the options are:

1. **Reduce cascade resolution** from 2048 to 1024 (saves ~50% rasterization time)
2. **Reduce PCF kernel** from 5x5 to 3x3 (saves ~65% fragment shader cost, harder edges)
3. **Reduce cascade count** from 3 to 2 (saves one full cascade render + simplifies sampling)
4. **Cache shadow maps** for static scenes (cost drops to near zero when the scene is not changing)

### Shadow Pass Pipeline Pseudocode

```typescript
const renderShadows = (context: FrameContext): void => {
  const { device, camera, scene, shadowAtlas, encoder } = context
  const light = scene.getDirectionalLight()
  if (!light || !light.shadow?.enabled) return

  // Check cache
  if (!shouldUpdateShadows(context.shadowCache, camera, light)) return

  // Compute cascade splits and matrices
  const splits = computeCascadeSplits({
    cascadeCount: 3,
    near: camera.near,
    far: light.shadow.far,
    lambda: 0.7,
  }, context.cascadeSplitBuffer)

  const cascades: CascadeData[] = []
  for (let i = 0; i < 3; i++) {
    const near = i === 0 ? camera.near : splits[i - 1]
    const far = splits[i]
    cascades.push(computeCascadeLightMatrix(camera, light, near, far, shadowAtlas.resolution))
  }

  // Execute shadow pass for each cascade
  executeShadowPass(device, encoder, shadowAtlas, cascades, context.shadowCasters, context.shadowPipeline, context.shadowUBO)

  // Upload cascade matrices and splits to the frame UBO for the opaque pass to read
  packCascadeUniforms(cascades, splits, context.frameUBO)

  // Update cache
  context.shadowCache.dirty = false
  vec3Copy(camera.position, context.shadowCache.lastCameraPosition)
  quatCopy(camera.rotation, context.shadowCache.lastCameraRotation)
  vec3Copy(getDirectionalLightDirection(light, tempVec3), context.shadowCache.lastLightDirection)
}
```
