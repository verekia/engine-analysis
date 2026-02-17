# Transparency — Weighted Blended Order-Independent Transparency

## The Problem with Traditional Transparency

Standard alpha blending in WebGL/WebGPU uses the Porter-Duff over operator:

```
dst = src.rgb * src.a + dst.rgb * (1 - src.a)
```

This only produces correct results when fragments are composited **back-to-front**. Three.js and most engines sort transparent objects by distance from camera, but this is fundamentally broken:

- **Sorting is per-object, not per-pixel.** Two intersecting transparent meshes will always have one entirely in front of the other.
- **Concave meshes self-overlap.** A single transparent sphere's front and back faces cannot both sort correctly.
- **Sorting is expensive.** O(n log n) per frame for n transparent objects, with distance ties causing frame-to-frame popping.
- **It just doesn't work reliably.** Users spend hours fighting transparency in three.js with no good solution.

Lynx eliminates all of this with **Weighted Blended OIT**.

## Weighted Blended OIT (McGuire & Bavoil 2013)

The key insight: instead of compositing fragments in order, accumulate all transparent fragments simultaneously using a weighted average, then composite the result in a single final pass.

### How It Works

Two render targets capture transparent fragment contributions:

**Accumulation buffer** (RGBA16F, cleared to 0,0,0,0):
- RGB: sum of `premultiplied_color × weight` across all fragments
- A: sum of `alpha × weight` across all fragments

**Revealage buffer** (R8, cleared to 1.0):
- R: product of `(1 - alpha)` across all fragments — how much background shows through

### Weight Function

The weight function gives closer fragments more visual influence:

```
w(z, alpha) = alpha × max(1e-2, min(3e3, 10.0 / (1e-5 + pow(z / 200.0, 4.0))))
```

- `z` is the linear view-space depth
- `z / 200` normalizes for a 200m world (tunable via `oitDepthRange`)
- The quartic falloff means nearby fragments dominate, far ones contribute less
- Clamped to `[0.01, 3000]` to prevent numerical issues

### Blend Modes

Both targets are written in a single pass with different blend equations:

| Target | Src Factor | Dst Factor | Equation |
|---|---|---|---|
| Accumulation (RGBA) | ONE | ONE | Add — accumulate weighted colors |
| Revealage (R) | ZERO | ONE_MINUS_SRC_ALPHA | Multiply — chain of (1 - alpha) |

### Composite Formula

A full-screen pass reads both targets and composites onto the opaque scene:

```glsl
vec4 accum = texelFetch(uAccumulation, ivec2(gl_FragCoord.xy), 0);
float revealage = texelFetch(uRevealage, ivec2(gl_FragCoord.xy), 0).r;

// Average transparent color
vec3 transparentColor = accum.rgb / max(accum.a, 1e-5);

// Blend with opaque background
// revealage = 1.0 means fully transparent (no transparent fragments)
// revealage = 0.0 means fully opaque transparent coverage
fragColor = vec4(transparentColor * (1.0 - revealage) + opaqueColor.rgb * revealage, 1.0);
```

## Render Pipeline Integration

```
┌─────────────────────────────────────────────────────┐
│ Pass 1: Opaque                                      │
│  ─ Render all opaque meshes                         │
│  ─ Writes color (MSAA) + depth (MSAA)               │
│  ─ Standard depth test + depth write                │
└─────────────┬───────────────────────────────────────┘
              │ depth buffer (read-only)
              ▼
┌─────────────────────────────────────────────────────┐
│ Pass 2: OIT Accumulation                            │
│  ─ Render ALL transparent meshes (any order)        │
│  ─ Depth test ON, depth write OFF                   │
│  ─ No backface culling                              │
│  ─ Writes accumulation (RGBA16F) + revealage (R8)   │
│  ─ Additive / multiplicative blending               │
└─────────────┬───────────────────────────────────────┘
              │ accumulation + revealage textures
              ▼
┌─────────────────────────────────────────────────────┐
│ Pass 3: OIT Composite                               │
│  ─ Full-screen triangle                             │
│  ─ Reads accumulation, revealage, opaque color      │
│  ─ Outputs final composited color                   │
└─────────────────────────────────────────────────────┘
```

The opaque depth buffer is critical — it prevents transparent fragments behind opaque surfaces from contributing to the accumulation.

## OIT Accumulation Shader

### GLSL 300 es

```glsl
#version 300 es
precision highp float;

layout(location = 0) out vec4 outAccumulation;
layout(location = 1) out float outRevealage;

in vec3 vNormal;
in vec2 vUV;
in vec4 vColor;
flat in int vMaterialIndex;

uniform sampler2D uColorTexture;
uniform float uOitDepthRange; // default 200.0

// Material palette UBO
layout(std140) uniform MaterialPalette {
    vec4 palette[64]; // .rgb = color, .a = opacity override (-1 = use mesh opacity)
};

// Lighting uniforms
uniform vec3 uLightDir;
uniform vec3 uLightColor;
uniform vec3 uAmbientColor;

void main() {
    // Base color from texture, vertex color, and material palette
    vec4 texColor = texture(uColorTexture, vUV);
    vec4 matEntry = palette[vMaterialIndex];
    vec3 baseColor = texColor.rgb * vColor.rgb * matEntry.rgb;
    float alpha = texColor.a * vColor.a;
    if (matEntry.a >= 0.0) alpha *= matEntry.a;

    // Lambert shading
    vec3 N = normalize(vNormal);
    float NdotL = max(dot(N, uLightDir), 0.0);
    vec3 lighting = uAmbientColor + uLightColor * NdotL;
    vec3 color = baseColor * lighting;

    // Premultiply alpha
    vec3 premul = color * alpha;

    // Depth-based weight
    float viewZ = gl_FragCoord.z; // [0, 1] — nonlinear depth
    float linearZ = gl_FragCoord.w; // 1/w ≈ linear depth in many setups
    // Use screen-space z for weight (simpler, works well in practice)
    float w = alpha * max(1e-2, min(3e3,
        10.0 / (1e-5 + pow(gl_FragCoord.z / 0.9, 4.0))
    ));

    outAccumulation = vec4(premul * w, alpha * w);
    outRevealage = alpha;
}
```

### WGSL

```wgsl
struct OitOutput {
    @location(0) accumulation: vec4f,
    @location(1) revealage: f32,
}

@fragment
fn fragmentMain(input: VertexOutput) -> OitOutput {
    let texColor = textureSample(colorTexture, colorSampler, input.uv);
    let matEntry = materialPalette.entries[input.materialIndex];
    let baseColor = texColor.rgb * input.color.rgb * matEntry.rgb;
    var alpha = texColor.a * input.color.a;
    if (matEntry.a >= 0.0) { alpha *= matEntry.a; }

    let N = normalize(input.normal);
    let NdotL = max(dot(N, lightData.direction), 0.0);
    let lighting = lightData.ambientColor + lightData.color * NdotL;
    let color = baseColor * lighting;
    let premul = color * alpha;

    let w = alpha * max(1e-2, min(3e3,
        10.0 / (1e-5 + pow(input.position.z / 0.9, 4.0))
    ));

    return OitOutput(
        vec4f(premul * w, alpha * w),
        alpha,
    );
}
```

## OIT Composite Shader

### GLSL 300 es

```glsl
#version 300 es
precision highp float;

uniform sampler2D uAccumulation;
uniform sampler2D uRevealage;
uniform sampler2D uOpaqueColor;

out vec4 fragColor;

void main() {
    ivec2 coord = ivec2(gl_FragCoord.xy);
    vec4 accum = texelFetch(uAccumulation, coord, 0);
    float revealage = texelFetch(uRevealage, coord, 0).r;
    vec3 opaqueColor = texelFetch(uOpaqueColor, coord, 0).rgb;

    // No transparent fragments at this pixel
    if (revealage >= 0.999) {
        fragColor = vec4(opaqueColor, 1.0);
        return;
    }

    // Reconstruct transparent color
    vec3 transparentColor = accum.rgb / max(accum.a, 1e-5);

    // Composite: transparent over opaque
    vec3 result = transparentColor * (1.0 - revealage) + opaqueColor * revealage;
    fragColor = vec4(result, 1.0);
}
```

### WGSL

```wgsl
@fragment
fn compositeMain(@builtin(position) position: vec4f) -> @location(0) vec4f {
    let coord = vec2i(position.xy);
    let accum = textureLoad(accumTexture, coord, 0);
    let revealage = textureLoad(revealageTexture, coord, 0).r;
    let opaque = textureLoad(opaqueTexture, coord, 0).rgb;

    if (revealage >= 0.999) {
        return vec4f(opaque, 1.0);
    }

    let transparent = accum.rgb / max(accum.a, 1e-5);
    let result = transparent * (1.0 - revealage) + opaque * revealage;
    return vec4f(result, 1.0);
}
```

## Render Target Setup

```typescript
const createOitTargets = (device: GalDevice, width: number, height: number): OitTargets => {
    const accumulation = device.createTexture({
        width,
        height,
        format: 'rgba16float',
        usage: 'render-target' | 'sampled',
    })

    const revealage = device.createTexture({
        width,
        height,
        format: 'r8unorm',
        usage: 'render-target' | 'sampled',
    })

    return { accumulation, revealage }
}
```

## Pipeline State

```typescript
const createOitAccumulationPipeline = (device: GalDevice, shader: GalShader): GalPipeline =>
    device.createPipeline({
        shader,
        depthStencil: {
            depthTest: true,
            depthWrite: false,     // Read-only depth — do not write
            depthCompare: 'less',
        },
        rasterizer: {
            cullMode: 'none',      // Render both faces of transparent geometry
        },
        blend: [
            // Attachment 0: Accumulation (RGBA16F) — additive
            {
                color: { src: 'one', dst: 'one', op: 'add' },
                alpha: { src: 'one', dst: 'one', op: 'add' },
            },
            // Attachment 1: Revealage (R8) — multiplicative
            {
                color: { src: 'zero', dst: 'one-minus-src-alpha', op: 'add' },
                alpha: { src: 'zero', dst: 'one-minus-src-alpha', op: 'add' },
            },
        ],
        targets: [
            { format: 'rgba16float' },
            { format: 'r8unorm' },
        ],
    })
```

## WebGPU vs WebGL2

| Feature | WebGPU | WebGL2 |
|---|---|---|
| Float render targets | Built-in | `EXT_color_buffer_float` (97%+ support) |
| MRT | Native fragment outputs | `gl.drawBuffers([...])` |
| Blend per-attachment | Native | `gl.blendFuncSeparatei` (WebGL2 extension) or draw separately |
| Depth read-only | `depthReadOnly: true` | Depth test on, `gl.depthMask(false)` |

WebGL2 fallback for per-attachment blend (if `blendFuncSeparatei` unavailable): render accumulation and revealage in two separate passes reading the same geometry. Costs an extra draw pass but is fully compatible.

## Usage API

From the user's perspective, transparency just works:

```typescript
// Transparent material — no sorting, no special handling needed
const glassMaterial = createLambertMaterial({
    color: [0.2, 0.5, 1.0],
    opacity: 0.4,
    transparent: true,
})

const windowMesh = createMesh(boxGeometry, glassMaterial)
scene.add(windowMesh)

// Overlapping transparent objects? No problem.
const overlapping = createMesh(sphereGeometry, createLambertMaterial({
    color: [1.0, 0.3, 0.3],
    opacity: 0.5,
    transparent: true,
}))
overlapping.position.set(0.5, 0, 0) // intersects windowMesh
scene.add(overlapping)

// Both render correctly without any sorting artifacts
```

## Limitations

| Limitation | Impact | Mitigation |
|---|---|---|
| Weighted average, not exact compositing | Subtle color bleeding with very different alpha values stacked | Tune weight function; rarely noticeable in games |
| Not suitable for thick volumetrics | Dense smoke/fog needs raymarching | Use dedicated volumetric pass if needed |
| Float render target required | Older WebGL2 without `EXT_color_buffer_float` | 97%+ browser support; could fall back to RGBA8 with reduced quality |
| No refraction | Cannot distort background through transparent objects | Add screen-space refraction pass separately if needed |

For stylized low-poly games — the primary use case for Lynx — these limitations are nearly invisible. The massive benefit of "transparency that just works" far outweighs the edge cases.
