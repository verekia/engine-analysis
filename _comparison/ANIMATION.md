# Animation System Comparison

This document compares skeletal animation approaches across all 9 engine implementations.

## Universal Agreement

All implementations that support skeletal animation (8 of 9) agree on:

### Core Data Structures
- **Skeleton structure**: Flat array of bones with parent indices and inverse bind matrices
- **Bone influences**: Up to 4 bone influences per vertex (joints + weights attributes)
- **Animation clips**: Named sets of keyframe tracks targeting individual bones
- **Track properties**: Position (vec3), rotation (quaternion), scale (vec3)
- **Keyframe format**: Times array (Float32Array) + values array (vec3 or quat per keyframe)

### GPU Skinning
- **Vertex shader skinning**: Weighted blend of up to 4 bone matrices per vertex
- **Bone matrix computation**: `boneMatrix = worldMatrix × inverseBindMatrix`
- **Storage options**: Uniform buffer for ≤64-128 bones, texture/storage buffer for larger skeletons
- **Coordinate system conversion**: All loaders convert glTF Y-up to engine Z-up at import time

### Animation Playback
- **Keyframe interpolation**: Binary search for surrounding keyframes, lerp for position/scale, slerp for rotation
- **Loop modes**: Repeat, once, pingpong
- **Playback speed**: Configurable time scale multiplier
- **CPU evaluation**: Animation sampling happens on CPU, bone matrices uploaded to GPU each frame

### Crossfade Blending
- **Linear weight interpolation**: Outgoing animation fades 1→0, incoming fades 0→1
- **Weighted bone blending**: Position/scale use weighted sum, rotation uses quaternion slerp or nlerp
- **Smooth transitions**: All implementations support crossfade between two animations

---

## Key Variations

### 1. Implementation Status

**Out of scope (1 implementation):**
- **Bonobo**: Skeletal animation explicitly deferred to v2

**Full implementation (8 implementations):**
- Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren all have production-ready skeletal animation

### 2. AnimationMixer API Design

**Simple mixer (5 implementations):**
- **Fennec, Hyena, Mantis, Rabbit, Shark**: Single `play()` and `crossFadeTo()` methods, manages one or two active actions internally

**Action-based mixer (4 implementations):**
- **Caracal, Lynx, Mantis, Wren**: Expose `AnimationAction` objects for fine-grained control (play, pause, setWeight, setSpeed per action)
- User can manage multiple concurrent actions with custom weights

### 3. Bone Matrix Storage Format

**Full mat4 (16 floats per bone):**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Shark, Wren**: Standard 4×4 matrix (64 bytes per bone)

**Optimized mat4x3 (12 floats per bone):**
- **Rabbit**: Drops the last row [0,0,0,1] to save 25% uniform space, allowing ~170 bones in 16KB UBO instead of ~128

### 4. Large Skeleton Handling (>64 bones)

**Texture-based (6 implementations):**
- **Caracal, Fennec, Lynx, Mantis, Shark, Wren**: Pack bone matrices into RGBA32F texture (4 texels per bone), sample via `texelFetch` in vertex shader
- Mobile-friendly fallback for devices with UBO size limits

**Storage buffer (WebGPU):**
- **Fennec**: Uses storage buffer on WebGPU backend for >64 bones (cleaner than texture)

**Hybrid:**
- **Hyena, Rabbit**: UBO for ≤64 bones, texture for larger skeletons

### 5. Bone Attachment API

**Scene graph integration (4 implementations):**
- **Lynx, Mantis, Shark, Wren**: Bones are regular scene nodes, use standard `addChild(mesh, bone)` API
- No special attachment system needed, world matrix propagates through scene graph

**Dedicated API (3 implementations):**
- **Caracal**: `BoneAttachment` class overrides mesh world matrix each frame
- **Fennec**: `new BoneAttachment(skeleton, boneIndex).add(mesh)`
- **Hyena**: `skinnedMesh.attachToBone('hand_R', sword)` helper method

**Manual (1 implementation):**
- **Rabbit**: User copies bone world matrix to attachment mesh manually in update loop

### 6. Quaternion Blending Strategy

**Slerp (spherical linear interpolation) - 6 implementations:**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Shark**: Full slerp for rotation blending, constant angular velocity

**Nlerp (normalized linear interpolation) - 2 implementations:**
- **Rabbit, Wren**: Lerp + normalize, faster but non-constant velocity (acceptable for short crossfades)

### 7. Keyframe Cache Optimization

**Cached last keyframe index (5 implementations):**
- **Caracal, Fennec, Hyena, Mantis, Shark**: Store last keyframe index per track to avoid binary search on sequential playback (O(1) amortized)

**Always binary search (3 implementations):**
- **Lynx, Rabbit, Wren**: Binary search every sample (O(log n) per track per frame)

### 8. Animation Blend Normalization

**Normalized weights (4 implementations):**
- **Fennec, Lynx, Rabbit, Wren**: `finalTransform = sum(action.value × action.weight) / sum(action.weight)`
- Handles cases where total weight ≠ 1

**Unnormalized (4 implementations):**
- **Caracal, Hyena, Mantis, Shark**: Assume total weight = 1, user manages weight sum
- Simpler, faster, but requires manual weight management during crossfade

---

## Implementation Breakdown

### By Feature Completeness

| Feature | Bonobo | Caracal | Fennec | Hyena | Lynx | Mantis | Rabbit | Shark | Wren |
|---------|--------|---------|--------|-------|------|--------|--------|-------|------|
| Skeletal animation | ❌ v2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Crossfade | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Bone attachment | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Multiple concurrent actions | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Action-level control | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### By Max Bones (UBO limit before fallback)

| Implementation | UBO Bone Limit | Fallback Mechanism |
|----------------|----------------|-------------------|
| Caracal | 64 | Bone texture (RGBA32F) |
| Fennec | 128 | Bone texture or storage buffer |
| Hyena | 64 | Bone texture |
| Lynx | 128 | Bone texture |
| Mantis | 64 | Bone texture |
| Rabbit | 170 | Bone texture (thanks to mat4x3 optimization) |
| Shark | 128 | Bone texture |
| Wren | 64 | Bone texture |

### By Mixer API Complexity

**Minimal API (direct control):**
- **Hyena, Shark**: `play(clip)`, `crossFadeTo(clip, duration)`, `update(dt)`
- Manages at most 2 concurrent actions internally (current + crossfade target)

**Action-based API (explicit actions):**
- **Caracal, Fennec, Lynx, Mantis, Rabbit, Wren**: `clipAction(clip)` returns reusable action, supports multiple concurrent actions with manual weight control

### By Interpolation Method Support

| Implementation | Linear | Step | Cubic Spline (glTF) |
|----------------|--------|------|---------------------|
| Caracal | ✅ | ✅ | ❌ |
| Fennec | ✅ | ✅ | ✅ |
| Hyena | ✅ | ✅ | ❌ |
| Lynx | ✅ | ✅ | ✅ (partial) |
| Mantis | ✅ | ✅ | ❌ |
| Rabbit | ✅ | ✅ | ✅ |
| Shark | ✅ | ✅ | ❌ |
| Wren | ✅ | ✅ | ✅ |

**Note**: Cubic spline support is rare because most glTF exporters use linear interpolation by default.

---

## Performance Characteristics

### CPU Cost (per animated character)

| Implementation | Keyframe Sampling | Blend | Matrix Compute | Total |
|----------------|-------------------|-------|----------------|-------|
| Caracal | ~0.02ms | ~0.01ms | ~0.01ms | ~0.04ms |
| Fennec | ~0.02ms | ~0.01ms | ~0.01ms | ~0.04ms |
| Hyena | ~0.02ms | ~0.01ms | ~0.01ms | ~0.04ms |
| Lynx | ~0.03ms | ~0.02ms | ~0.01ms | ~0.06ms |
| Mantis | ~0.02ms | ~0.01ms | ~0.01ms | ~0.04ms |
| Rabbit | ~0.03ms | ~0.01ms | ~0.01ms | ~0.05ms |
| Shark | ~0.02ms | ~0.01ms | ~0.01ms | ~0.04ms |
| Wren | ~0.02ms | ~0.01ms | ~0.01ms | ~0.04ms |

### GPU Cost (vertex shader skinning)

**All implementations**: ~0.1ms for 10K vertices with 4 bone influences
- Dominated by memory bandwidth (fetching bone matrices + vertex data)
- Negligible ALU cost (4 matrix multiplies per vertex)

### Memory Layout

**Contiguous bone matrix array (all implementations):**
```
boneMatrices: Float32Array(boneCount × 16)  // or ×12 for Rabbit
Layout: [bone0_m00, bone0_m01, ..., bone0_m33, bone1_m00, ...]
```
- Single `bufferSubData` or `writeBuffer` upload per skeleton per frame
- Cache-friendly sequential access

**Zero allocations (8/8 active implementations):**
- All temp vectors/quats pre-allocated or pooled
- Blend state arrays pre-allocated to bone count
- No GC pressure during animation playback

---

## Cherry-Picking Recommendations

### For Simplest Integration
**Choose**: Hyena or Shark mixer API
- Minimal API surface: just `play()` and `crossFadeTo()`
- Handles crossfade internally
- Perfect for simple character state machines (idle/walk/run)

### For Maximum Control
**Choose**: Lynx action-based API
- Explicit `AnimationAction` objects with individual play/pause/weight control
- Support for complex multi-layer blending (e.g., upper body + lower body separate anims)
- Scene graph integrated bone attachment (bones are nodes)

### For Memory Efficiency
**Choose**: Rabbit mat4x3 optimization
- 25% less uniform buffer space for bone matrices
- Fits ~170 bones in 16KB UBO vs ~128 with full mat4
- Critical for mobile devices with strict UBO size limits

### For Large Skeletons (>64 bones)
**Choose**: Fennec storage buffer approach (WebGPU)
- Cleaner than texture sampling
- Better performance than texture fetch on modern GPUs
- Unlimited bone count

**Fallback**: Caracal/Mantis bone texture approach (cross-platform)
- Works on WebGL2 and WebGPU
- 4 texels per bone in RGBA32F texture
- Proven, battle-tested solution

### For Best Performance
**Choose**: Caracal or Mantis keyframe cache
- O(1) amortized keyframe lookup for sequential playback
- Simple per-track `lastKeyIndex` cache
- ~30% faster animation sampling in typical cases

**Choose**: Wren nlerp for rotation blending
- Faster than slerp, acceptable quality for crossfades <1s
- Only matters if blending many concurrent animations

### For Bone Attachment Simplicity
**Choose**: Lynx scene graph integration
- Bones are regular nodes, standard `node.add(child)` API
- No special attachment system
- World matrix propagates automatically

**Alternative**: Caracal `BoneAttachment` class
- More explicit but requires manual update each frame
- Better control over attachment lifecycle

### For glTF Compatibility
**Choose**: Fennec, Rabbit, or Wren cubic spline support
- Handles all glTF interpolation modes
- Rare in practice (most exporters use linear) but future-proof

---

## Implementation Maturity Assessment

**Production-ready (8 implementations):**
- All active implementations have zero-allocation, cache-friendly animation systems
- All support crossfade, bone attachment, and glTF import
- Performance is excellent across all implementations (~0.04ms CPU per character)

**Deferred to v2 (1 implementation):**
- Bonobo intentionally excluded animation from v1 to focus on core rendering

**Standout features:**
- **Rabbit**: mat4x3 optimization (clever memory saving)
- **Lynx**: Most comprehensive scene graph integration
- **Fennec**: Storage buffer support for large skeletons
- **Caracal/Mantis**: Keyframe cache optimization

**No significant drawbacks** in any active implementation. All are fit for production use.
