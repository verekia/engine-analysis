# Bloom & Post-Processing Pipeline

## Pipeline Overview

After opaque rendering + OIT composite, the scene exists in an HDR buffer (values can exceed 1.0 due to emissive materials). The post-processing chain converts this to a displayable image:

```
HDR Scene Buffer
      │
      ▼
┌──────────────┐    ┌────────────────────────────────────────┐
│ Bloom Extract │───▶│ Downscale Chain (5 levels)              │
│ (threshold)   │    │ full → 1/2 → 1/4 → 1/8 → 1/16 → 1/32 │
└──────────────┘    └────────────────┬───────────────────────┘
                                     │
                    ┌────────────────▼───────────────────────┐
                    │ Upscale Chain (additive blend back up)  │
                    │ 1/32 → 1/16 → 1/8 → 1/4 → 1/2 → full  │
                    └────────────────┬───────────────────────┘
                                     │
      ┌──────────────────────────────┘
      ▼
┌──────────────┐
│ Bloom Add    │  finalHDR = scene + bloom × strength
│ + Tone Map   │  LDR = ACES(finalHDR)
│ + Gamma      │  sRGB = pow(LDR, 1/2.2)
└──────┬───────┘
       ▼
   Final Output
```

All passes use a **full-screen triangle** (3 vertices, no index buffer) rather than a quad — this eliminates the diagonal seam and reduces fragment shader invocations at screen edges.

## Per-Vertex Bloom via Emissive

Bloom in Lynx is driven by the emissive channel of materials. The `_materialindex` vertex attribute system allows per-region control:

```typescript
// Material palette: material index → visual properties
const palette: MaterialPaletteEntry[] = [
    { color: [1, 1, 1], emissive: [0, 0, 0], emissiveIntensity: 0 },        // 0: white, no glow
    { color: [0, 0, 0], emissive: [0, 0, 0], emissiveIntensity: 0 },        // 1: black, no glow
    { color: [0, 0.5, 0.5], emissive: [0, 0.8, 0.8], emissiveIntensity: 0.7 }, // 2: teal, glows
]
```

In the fragment shader, emissive adds to the output:

```glsl
vec3 finalColor = baseColor * lighting * shadow;
finalColor += emissive * emissiveIntensity;  // values > 1.0 trigger bloom
```

Vertices with material index 2 output `tealColor + [0, 0.56, 0.56]` — the emissive component pushes values above 1.0 in the HDR buffer, which the bloom extraction pass picks up.

## Pass 1: Bloom Extraction

Extract bright areas from the HDR scene buffer using a soft threshold:

```glsl
#version 300 es
precision highp float;

uniform sampler2D uSceneHDR;
uniform float uThreshold;    // default 1.0
uniform float uKnee;         // default 0.1 — softness of threshold

out vec4 fragColor;

void main() {
    ivec2 coord = ivec2(gl_FragCoord.xy);
    vec3 color = texelFetch(uSceneHDR, coord, 0).rgb;

    // Luminance
    float lum = dot(color, vec3(0.2126, 0.7152, 0.0722));

    // Soft threshold — avoids hard cutoff artifacts
    // Quadratic ease from (threshold - knee) to (threshold + knee)
    float soft = lum - uThreshold + uKnee;
    soft = clamp(soft / (2.0 * uKnee + 1e-5), 0.0, 1.0);
    soft = soft * soft;

    float contribution = max(soft, step(uThreshold, lum));
    fragColor = vec4(color * contribution, 1.0);
}
```

The soft knee prevents a harsh boundary where pixels just below the threshold pop in and out of the bloom.

## Pass 2: Downscale Chain (13-Tap Filter)

Each downscale level halves the resolution while filtering to prevent aliasing:

```glsl
#version 300 es
precision highp float;

uniform sampler2D uSource;
uniform vec2 uTexelSize; // 1.0 / source resolution

out vec4 fragColor;

void main() {
    vec2 uv = gl_FragCoord.xy * uTexelSize * 2.0; // scale to source UV

    // 13-tap filter: 4 bilinear samples in a + pattern, 4 corner samples, 1 center
    // This matches the Unreal Engine bloom downscale filter
    vec3 a = texture(uSource, uv + uTexelSize * vec2(-1.0, -1.0)).rgb;
    vec3 b = texture(uSource, uv + uTexelSize * vec2( 0.0, -1.0)).rgb;
    vec3 c = texture(uSource, uv + uTexelSize * vec2( 1.0, -1.0)).rgb;
    vec3 d = texture(uSource, uv + uTexelSize * vec2(-1.0,  0.0)).rgb;
    vec3 e = texture(uSource, uv).rgb;
    vec3 f = texture(uSource, uv + uTexelSize * vec2( 1.0,  0.0)).rgb;
    vec3 g = texture(uSource, uv + uTexelSize * vec2(-1.0,  1.0)).rgb;
    vec3 h = texture(uSource, uv + uTexelSize * vec2( 0.0,  1.0)).rgb;
    vec3 i = texture(uSource, uv + uTexelSize * vec2( 1.0,  1.0)).rgb;

    // Weighted combination — center gets most weight, corners least
    // Karis average for first downscale to prevent fireflies
    vec3 result = e * 0.25
               + (b + d + f + h) * 0.125
               + (a + c + g + i) * 0.0625;

    fragColor = vec4(result, 1.0);
}
```

For the first downscale from full resolution, apply **Karis average** (weighted by 1/(1+luminance)) to prevent bright pixels from bleeding excessively (the "firefly" artifact):

```glsl
// First downscale only: Karis average to suppress fireflies
vec3 karisAvg(vec3 a, vec3 b, vec3 c, vec3 d) {
    float wa = 1.0 / (1.0 + dot(a, vec3(0.2126, 0.7152, 0.0722)));
    float wb = 1.0 / (1.0 + dot(b, vec3(0.2126, 0.7152, 0.0722)));
    float wc = 1.0 / (1.0 + dot(c, vec3(0.2126, 0.7152, 0.0722)));
    float wd = 1.0 / (1.0 + dot(d, vec3(0.2126, 0.7152, 0.0722)));
    return (a * wa + b * wb + c * wc + d * wd) / (wa + wb + wc + wd);
}
```

### Downscale Levels

| Level | Resolution (1920×1080 base) | Purpose |
|---|---|---|
| 0 | 960 × 540 | Tight, fine bloom |
| 1 | 480 × 270 | Medium glow |
| 2 | 240 × 135 | Wide glow |
| 3 | 120 × 68 | Very wide glow |
| 4 | 60 × 34 | Ultra-wide ambient glow |

## Pass 3: Upscale Chain (Tent Filter + Additive Blend)

Upscale from smallest back to full resolution, additively blending each level with the one above:

```glsl
#version 300 es
precision highp float;

uniform sampler2D uSource;      // lower-res bloom level (being upscaled)
uniform sampler2D uHigherLevel;  // current-res bloom level (from downscale)
uniform vec2 uTexelSize;         // 1.0 / source (lower) resolution
uniform float uBloomRadius;      // controls spread, default 0.85

out vec4 fragColor;

void main() {
    vec2 uv = gl_FragCoord.xy * uTexelSize;

    // 9-tap tent filter for smooth upscale
    vec3 a = texture(uSource, uv + vec2(-1.0, -1.0) * uTexelSize * uBloomRadius).rgb;
    vec3 b = texture(uSource, uv + vec2( 0.0, -1.0) * uTexelSize * uBloomRadius).rgb;
    vec3 c = texture(uSource, uv + vec2( 1.0, -1.0) * uTexelSize * uBloomRadius).rgb;
    vec3 d = texture(uSource, uv + vec2(-1.0,  0.0) * uTexelSize * uBloomRadius).rgb;
    vec3 e = texture(uSource, uv).rgb;
    vec3 f = texture(uSource, uv + vec2( 1.0,  0.0) * uTexelSize * uBloomRadius).rgb;
    vec3 g = texture(uSource, uv + vec2(-1.0,  1.0) * uTexelSize * uBloomRadius).rgb;
    vec3 h = texture(uSource, uv + vec2( 0.0,  1.0) * uTexelSize * uBloomRadius).rgb;
    vec3 i = texture(uSource, uv + vec2( 1.0,  1.0) * uTexelSize * uBloomRadius).rgb;

    // Tent kernel weights
    vec3 upscaled = (a + c + g + i) * (1.0 / 16.0)
                  + (b + d + f + h) * (2.0 / 16.0)
                  + e * (4.0 / 16.0);

    // Additive blend with higher-resolution level
    vec3 higher = texelFetch(uHigherLevel, ivec2(gl_FragCoord.xy), 0).rgb;
    fragColor = vec4(upscaled + higher, 1.0);
}
```

The progressive additive blending creates the characteristic **multi-scale glow**: fine detail from early levels, wide ambient glow from deep levels.

## Pass 4: Tone Mapping (ACES Filmic)

The final pass adds bloom to the scene and maps from HDR to LDR:

```glsl
#version 300 es
precision highp float;

uniform sampler2D uSceneHDR;
uniform sampler2D uBloom;
uniform float uBloomStrength;  // default 0.5
uniform float uExposure;       // default 1.0

out vec4 fragColor;

// ACES filmic tone mapping (Stephen Hill's fit)
vec3 acesToneMap(vec3 x) {
    float a = 2.51;
    float b = 0.03;
    float c = 2.43;
    float d = 0.59;
    float e = 0.14;
    return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0);
}

// Linear to sRGB
vec3 linearToSrgb(vec3 linear) {
    return pow(linear, vec3(1.0 / 2.2));
}

void main() {
    ivec2 coord = ivec2(gl_FragCoord.xy);
    vec3 scene = texelFetch(uSceneHDR, coord, 0).rgb;
    vec3 bloom = texelFetch(uBloom, coord, 0).rgb;

    // Add bloom
    vec3 hdr = scene + bloom * uBloomStrength;

    // Exposure
    hdr *= uExposure;

    // Tone map
    vec3 ldr = acesToneMap(hdr);

    // Gamma (skip if output format handles sRGB)
    vec3 srgb = linearToSrgb(ldr);

    fragColor = vec4(srgb, 1.0);
}
```

### WGSL Version

```wgsl
@group(0) @binding(0) var sceneTexture: texture_2d<f32>;
@group(0) @binding(1) var bloomTexture: texture_2d<f32>;
@group(0) @binding(2) var<uniform> params: PostProcessParams;

fn acesToneMap(x: vec3f) -> vec3f {
    let a = 2.51; let b = 0.03;
    let c = 2.43; let d = 0.59; let e = 0.14;
    return clamp((x * (a * x + b)) / (x * (c * x + d) + e), vec3f(0.0), vec3f(1.0));
}

@fragment
fn toneMapFragment(@builtin(position) pos: vec4f) -> @location(0) vec4f {
    let coord = vec2i(pos.xy);
    let scene = textureLoad(sceneTexture, coord, 0).rgb;
    let bloom = textureLoad(bloomTexture, coord, 0).rgb;

    var hdr = scene + bloom * params.bloomStrength;
    hdr *= params.exposure;

    let ldr = acesToneMap(hdr);
    let srgb = pow(ldr, vec3f(1.0 / 2.2));
    return vec4f(srgb, 1.0);
}
```

## Full-Screen Triangle

All post-processing passes use a single triangle that covers the entire screen:

```glsl
#version 300 es

// No vertex buffer needed — vertex ID determines position
void main() {
    // Generates a triangle covering [-1, -1] to [3, 1] (clips to screen)
    vec2 pos = vec2(
        float((gl_VertexID & 1) * 4 - 1),
        float((gl_VertexID >> 1) * 4 - 1)
    );
    gl_Position = vec4(pos, 0.0, 1.0);
}
```

Advantages over a quad: no index buffer, no diagonal edge artifacts, GPU processes one large triangle instead of two with a shared edge.

## Bloom Texture Management

```typescript
interface BloomChain {
    readonly levels: GalTexture[]  // downscale chain textures
    readonly width: number
    readonly height: number
}

const createBloomChain = (device: GalDevice, width: number, height: number, levels = 5): BloomChain => {
    const textures: GalTexture[] = []
    let w = Math.floor(width / 2)
    let h = Math.floor(height / 2)

    for (let i = 0; i < levels; i++) {
        textures.push(device.createTexture({
            width: Math.max(1, w),
            height: Math.max(1, h),
            format: 'rgba16float', // could use r11g11b10float for memory savings
            usage: 'render-target' | 'sampled',
        }))
        w = Math.floor(w / 2)
        h = Math.floor(h / 2)
    }

    return { levels: textures, width, height }
}
```

## Post-Processing Execution

```typescript
const executePostProcessing = (
    encoder: GalCommandEncoder,
    scene: GalTexture,
    bloom: BloomConfig,
    chain: BloomChain,
    output: GalTexture,
) => {
    if (!bloom.enabled) {
        // Skip bloom, just tone map directly
        const pass = encoder.beginRenderPass({ colorAttachments: [{ target: output }] })
        pass.setPipeline(toneMapPipeline)
        pass.setBindGroup(0, createToneMapBindGroup(scene, null, bloom))
        pass.draw(3) // full-screen triangle
        pass.end()
        return
    }

    // 1. Extract bright pixels
    const extractPass = encoder.beginRenderPass({
        colorAttachments: [{ target: chain.levels[0], clearColor: [0, 0, 0, 0] }],
    })
    extractPass.setPipeline(bloomExtractPipeline)
    extractPass.setBindGroup(0, createExtractBindGroup(scene, bloom.threshold, bloom.knee))
    extractPass.draw(3)
    extractPass.end()

    // 2. Downscale chain
    for (let i = 1; i < chain.levels.length; i++) {
        const pass = encoder.beginRenderPass({
            colorAttachments: [{ target: chain.levels[i], clearColor: [0, 0, 0, 0] }],
        })
        pass.setPipeline(i === 1 ? bloomDownscaleKarisPipeline : bloomDownscalePipeline)
        pass.setBindGroup(0, createDownscaleBindGroup(chain.levels[i - 1]))
        pass.draw(3)
        pass.end()
    }

    // 3. Upscale chain (in-place: each level accumulates from below)
    for (let i = chain.levels.length - 2; i >= 0; i--) {
        const pass = encoder.beginRenderPass({
            colorAttachments: [{ target: chain.levels[i] }], // no clear — additive
        })
        pass.setPipeline(bloomUpscalePipeline)
        pass.setBindGroup(0, createUpscaleBindGroup(chain.levels[i + 1], chain.levels[i], bloom.radius))
        pass.draw(3)
        pass.end()
    }

    // 4. Tone map + bloom composite
    const finalPass = encoder.beginRenderPass({
        colorAttachments: [{ target: output }],
    })
    finalPass.setPipeline(toneMapPipeline)
    finalPass.setBindGroup(0, createToneMapBindGroup(scene, chain.levels[0], bloom))
    finalPass.draw(3)
    finalPass.end()
}
```

## Configuration API

```typescript
interface BloomConfig {
    enabled: boolean        // default true
    threshold: number       // default 1.0 — HDR values above this bloom
    knee: number            // default 0.1 — soft threshold transition
    strength: number        // default 0.5 — bloom intensity
    radius: number          // default 0.85 — bloom spread
    levels: number          // default 5 — downscale chain depth
}

interface PostProcessConfig {
    bloom: BloomConfig
    exposure: number        // default 1.0
    toneMapping: 'aces' | 'reinhard' | 'none'  // default 'aces'
}

// Usage
const renderer = createRenderer(canvas, {
    postProcess: {
        bloom: { enabled: true, threshold: 0.8, strength: 0.7 },
        exposure: 1.2,
        toneMapping: 'aces',
    },
})

// Runtime adjustment
renderer.postProcess.bloom.strength = 0.3
renderer.postProcess.exposure = 1.5
```

## Performance Budget

| Pass | Draw Calls | Estimated Mobile (ms) | Estimated Desktop (ms) |
|---|---|---|---|
| Bloom extract | 1 | 0.10 | 0.03 |
| Downscale × 5 | 5 | 0.15 | 0.05 |
| Upscale × 5 | 5 | 0.15 | 0.05 |
| Tone map + composite | 1 | 0.15 | 0.05 |
| **Total** | **12** | **~0.55** | **~0.18** |

The bloom chain operates on progressively smaller textures, so the GPU cost drops rapidly with each level. Total post-processing overhead is well under 1ms even on mobile.

### Disabling for Low-End Devices

```typescript
// Detect low-end and disable bloom
const renderer = createRenderer(canvas, {
    postProcess: {
        bloom: { enabled: gpu.tier >= 2 }, // disable on lowest-tier devices
        toneMapping: 'aces', // tone mapping is always fast
    },
})
```
