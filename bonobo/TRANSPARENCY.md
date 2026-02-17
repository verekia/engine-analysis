# Transparency

## Status: Not Yet Specified

Transparency rendering is not yet fully specified for v1. This is a known hard problem that requires careful design.

## Requirements

- Must work without headaches (no visible sorting artifacts in common cases)
- Support for alpha blending (transparent objects like glass, water, particles)
- Support for alpha testing (cutout materials like foliage)

## Challenges

1. **Sorting** — Transparent objects must be sorted back-to-front per frame
2. **Overdraw** — Transparent objects can't early-Z cull, expensive for GPU
3. **Instancing** — Harder to instance transparent objects due to sorting requirements

## Potential Approaches

### 1. Alpha Testing (Cutout)

For foliage, fences, etc. — discard fragments below a threshold:

```wgsl
if (alpha < 0.5) {
  discard;
}
```

No sorting needed, writes to depth buffer, fast.

### 2. Back-to-Front Sorting (Classic Alpha Blending)

- Separate render pass for transparent objects
- Sort by distance from camera (back-to-front)
- Disable depth writes, enable alpha blending
- Render after opaque pass

**Problem:** Sorting is per-object, not per-triangle. Intersecting transparent objects still have artifacts.

### 3. Depth Peeling

- Multiple passes to peel layers of transparency
- First pass renders front-most layer
- Second pass renders second layer, etc.
- Expensive (N passes for N layers)

### 4. Weighted Blended Order-Independent Transparency (WBOIT)

- Render all transparent objects in single pass
- Accumulate color and weight in two buffers
- Resolve in final composite
- No sorting needed, single pass, good quality

### 5. Hybrid Approach (Most Likely for v1)

- **Alpha testing** for cutout materials (trees, grass)
- **Sorted alpha blending** for true transparency (glass, water, UI)
- Render in two separate passes:
  1. Opaque + alpha-tested (depth write enabled)
  2. Transparent sorted (depth write disabled, back-to-front)

## Implementation Plan (TBD)

For v1, start with:
- Alpha testing support (add `alphaMode: 'opaque' | 'mask'` to material)
- Defer true transparency to v2
- Use alpha testing + emissive for most use cases (UI, particles with additive blend)

For v2, add:
- Back-to-front sorting for transparent objects
- Separate transparent render pass
- Investigate WBOIT if sorting quality is insufficient

## Flags

Per-entity transparency flags:

```ts
OPAQUE_FLAG       // Default, writes depth
ALPHA_TEST_FLAG   // Cutout, writes depth, discards fragments
TRANSPARENT_FLAG  // True transparency, no depth write, sorted
```

## Related Topics

See POST-PROCESSING.md for alpha compositing in post-effects.
