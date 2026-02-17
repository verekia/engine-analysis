# Transparency

## Weighted Blended Order-Independent Transparency (OIT)

### The Problem with Sorted Transparency

Three.js (and most engines) sort transparent objects back-to-front each frame and draw them in order. This breaks in many common cases:

1. **Intersecting geometry** — two transparent planes crossing each other can never be sorted correctly
2. **Cyclic overlap** — three transparent objects where A is in front of B, B in front of C, C in front of A
3. **Large objects** — a transparent object surrounding the camera can't be sorted relative to objects inside it
4. **Per-pixel correctness** — sorting is per-object, not per-fragment. A single transparent mesh with concavities will self-occlude incorrectly.
5. **CPU cost** — sorting thousands of transparent objects every frame adds measurable overhead

### Weighted Blended OIT (McGuire & Bavoil 2013)

This technique achieves order-independent transparency in a single rendering pass with two extra render targets. No sorting needed.

**Principle**: Instead of blending fragments in order (which requires sorting), we accumulate all transparent fragments and compute an approximate blend using depth-based weights. Fragments closer to the camera get higher weights.

### Render Targets

```
Opaque pass target:   RGBA8 (standard color, MSAA)
OIT accumulation:     RGBA16F (premultiplied color × weight, sum of weights)
OIT revealage:        R8 (product of 1-alpha, i.e. total visibility through all transparent layers)
```

### OIT Pass Setup

**WebGL2:**
```typescript
// Framebuffer with two color attachments
gl.bindFramebuffer(gl.FRAMEBUFFER, oitFBO)
gl.drawBuffers([gl.COLOR_ATTACHMENT0, gl.COLOR_ATTACHMENT1])

// Attachment 0: accumulation (RGBA16F)
// Clear to (0, 0, 0, 0)
// Blend: src*ONE + dst*ONE (additive)

// Attachment 1: revealage (R8)
// Clear to (1, 1, 1, 1) — fully transparent = nothing occluded
// Blend: ZERO, ONE_MINUS_SRC_COLOR (multiplicative)
```

**WebGPU:**
```typescript
const oitPassDescriptor: GPURenderPassDescriptor = {
  colorAttachments: [
    {
      view: accumTexture.createView(),
      clearValue: [0, 0, 0, 0],
      loadOp: 'clear',
      storeOp: 'store',
    },
    {
      view: revealageTexture.createView(),
      clearValue: [1, 0, 0, 0],
      loadOp: 'clear',
      storeOp: 'store',
    },
  ],
  depthStencilAttachment: {
    view: depthTexture.createView(),
    depthLoadOp: 'load',        // reuse depth from opaque pass (read-only)
    depthStoreOp: 'store',
    depthReadOnly: true,         // DON'T write depth for transparent objects
  },
}
```

### OIT Fragment Shader

```glsl
// OIT output function — called instead of normal fragment output for transparent objects

layout(location = 0) out vec4 outAccum;     // accumulation
layout(location = 1) out float outRevealage; // revealage

void outputOIT(vec4 color) {
  float alpha = color.a;

  // Depth-based weight function
  // Objects closer to camera get higher weight → appear more opaque
  float viewDepth = gl_FragCoord.z; // 0 (near) to 1 (far)

  // Weight function from McGuire & Bavoil, tuned for typical game scenes
  float weight = alpha * max(1e-2, min(
    3e3,
    10.0 / (1e-5 + pow(viewDepth / 200.0, 4.0))
  ));

  // Accumulation: premultiplied color × weight
  outAccum = vec4(color.rgb * alpha * weight, alpha * weight);

  // Revealage: product of (1 - alpha)
  outRevealage = alpha;
}
```

### OIT Composite Pass

After drawing all transparent objects, composite the OIT result over the opaque scene:

```glsl
// Full-screen quad fragment shader
uniform sampler2D u_accum;
uniform sampler2D u_revealage;
uniform sampler2D u_opaqueColor;

out vec4 fragColor;

void main() {
  vec2 uv = gl_FragCoord.xy / u_resolution;

  vec4 accum = texture(u_accum, uv);
  float revealage = texture(u_revealage, uv).r;
  vec4 opaque = texture(u_opaqueColor, uv);

  // If no transparent fragments were written, revealage = 1.0
  if (revealage >= 1.0) {
    fragColor = opaque;
    return;
  }

  // Reconstruct transparent color
  vec3 transparentColor = accum.rgb / max(accum.a, 1e-5);

  // Blend: transparent over opaque
  // revealage = product of (1-alpha) of all transparent layers
  // So (1 - revealage) is the total opacity of all transparent layers combined
  fragColor = vec4(
    transparentColor * (1.0 - revealage) + opaque.rgb * revealage,
    1.0
  );
}
```

### Blend States

The accumulation and revealage targets require specific blend states:

**Accumulation (RGBA16F):**
- Color: `src × ONE + dst × ONE` (additive)
- Alpha: `src × ONE + dst × ONE` (additive)

**Revealage (R8):**
- Color: `ZERO × dst + SRC_COLOR × dst` → effectively `dst × (1 - srcAlpha)` when outputting alpha as color
- Actually: `src = ZERO, dst = ONE_MINUS_SRC_COLOR`

In WebGL2:
```typescript
// For accumulation attachment
gl.blendFunci(0, gl.ONE, gl.ONE)
// For revealage attachment
gl.blendFunci(1, gl.ZERO, gl.ONE_MINUS_SRC_COLOR)
```

In WebGPU, blend state is configured per color target in the pipeline descriptor.

### Depth Handling

Transparent objects read the depth buffer (from the opaque pass) but do not write to it. This ensures:
1. Transparent objects behind opaque objects are correctly occluded
2. Transparent objects do not occlude each other (OIT handles this through weights)

### Limitations

- Not physically accurate for deeply stacked colored glass (e.g., red glass behind blue glass won't look correct)
- Very similar depths with very different colors may not separate cleanly
- For game use (particles, windows, water, UI elements), it looks correct and natural
