# ANIMATION.md - Skeletal Animation System

## Core Architecture

**Decision: GPU-skinned skeletal animation with action-based mixer** (8/9 implementations support full animation)

### Data Structures (Universal Agreement)

- **Skeleton**: Flat array of bones with parent indices and inverse bind matrices
- **Bone influences**: Up to 4 bones per vertex (joints + weights attributes)
- **Animation clips**: Named sets of keyframe tracks targeting individual bones
- **Track properties**: Position (Vec3), rotation (Quat), scale (Vec3)
- **Keyframe format**: `times: Float32Array` + `values: Float32Array` per track

### GPU Skinning (Universal Agreement)

Weighted blend of up to 4 bone matrices per vertex in the vertex shader:

```glsl
mat4 skinMatrix =
  a_weights.x * boneMatrices[a_joints.x] +
  a_weights.y * boneMatrices[a_joints.y] +
  a_weights.z * boneMatrices[a_joints.z] +
  a_weights.w * boneMatrices[a_joints.w];

vec4 skinnedPosition = skinMatrix * vec4(a_position, 1.0);
vec3 skinnedNormal = mat3(skinMatrix) * a_normal;
```

Bone matrix: `boneMatrix = worldMatrix * inverseBindMatrix`

## AnimationMixer API

**Decision: Action-based mixer** (Caracal, Fennec, Lynx, Mantis, Rabbit, Wren approach over Hyena/Shark's simple mixer)

The action-based API provides individual `AnimationAction` objects with per-action play/pause/weight/speed control. This is necessary for complex scenarios like layered animation (upper body idle + lower body walking).

```typescript
const mixer = createAnimationMixer(skeleton)
const idleAction = mixer.clipAction(idleClip)
const walkAction = mixer.clipAction(walkClip)

idleAction.play()
// Later, crossfade to walk
mixer.crossFadeTo(walkAction, 0.3)

// Per-frame update
mixer.update(deltaTime)
```

### Why Not Simple Mixer (Hyena/Shark)?

A simple mixer with just `play()` and `crossFadeTo()` covers basic state machines (idle/walk/run) but cannot handle:
- Multiple concurrent animations with custom weights
- Layered blending (e.g., different animations for upper and lower body)
- Fine-grained per-action speed/weight control

The action-based API is a superset that handles both simple and complex cases.

## Keyframe Interpolation

**Decision: Linear + Step + Cubic Spline** (Fennec, Rabbit, Wren approach for full glTF compatibility)

- **Linear**: Lerp for position/scale, slerp for rotation (default)
- **Step**: Snap to nearest keyframe (for discrete state changes)
- **Cubic spline**: glTF `cubicSpline` interpolation (tangent-based Hermite)

Most exporters use linear, but supporting cubic spline ensures correct playback of all valid glTF animations.

## Keyframe Cache

**Decision: Cached last keyframe index per track** (Caracal, Fennec, Hyena, Mantis, Shark approach)

Store the last-used keyframe index on each track. For sequential playback (the common case), the next frame's keyframe is either the same or one step forward - O(1) lookup instead of O(log n) binary search.

```typescript
interface Track {
  times: Float32Array
  values: Float32Array
  _lastKeyIndex: number  // Cached for O(1) sequential access
}
```

This is ~30% faster than always binary searching (Lynx, Rabbit, Wren approach). The implementation cost is one extra integer per track.

## Quaternion Blending

**Decision: Slerp for rotation blending** (Caracal, Fennec, Hyena, Lynx, Mantis, Shark - 6/8 use this)

Slerp provides constant angular velocity, which produces smoother transitions than nlerp. The performance difference is negligible for typical bone counts (50-100 bones, ~0.01ms difference per character).

Nlerp (Rabbit, Wren) is faster but produces non-constant velocity that can be visible during long crossfades. For short crossfades (<0.5s) the difference is imperceptible, but slerp is the correct choice for an engine that may face longer transitions.

## Blend Weight Normalization

**Decision: Normalized weights** (Fennec, Lynx, Rabbit, Wren approach)

```
finalTransform = sum(action.value * action.weight) / sum(action.weight)
```

This ensures correct results even when total weight != 1.0 (e.g., during crossfade transitions or when the user sets custom weights). Unnormalized blending (Caracal, Hyena, Mantis, Shark) requires the user to manually ensure weights sum to 1.0, which is error-prone.

## Bone Matrix Storage

**Decision: Full mat4 via UBO for <=128 bones, texture fallback for larger** (Fennec, Lynx, Shark approach)

### UBO Path (Default)

```
128 bones * 16 floats * 4 bytes = 8192 bytes (8KB)
```

8KB is well within the minimum UBO size for both WebGL2 (16KB minimum) and WebGPU. 128 bones covers the vast majority of game characters.

### Texture Fallback (>128 bones)

For large skeletons, pack bone matrices into an RGBA32F texture (4 texels per bone):

```glsl
vec4 row0 = texelFetch(u_boneTexture, ivec2(boneIndex * 4 + 0, 0), 0);
vec4 row1 = texelFetch(u_boneTexture, ivec2(boneIndex * 4 + 1, 0), 0);
vec4 row2 = texelFetch(u_boneTexture, ivec2(boneIndex * 4 + 2, 0), 0);
vec4 row3 = texelFetch(u_boneTexture, ivec2(boneIndex * 4 + 3, 0), 0);
```

This is the proven cross-platform approach (6/8 implementations use texture fallback).

### Why Not mat4x3 (Rabbit)?

Rabbit's mat4x3 optimization saves 25% uniform space (12 floats vs 16 per bone, ~170 bones in 16KB). Clever, but:
- Adds complexity to the shader (reconstruct 4th row)
- 128 bones in 8KB UBO is sufficient for the target games
- The texture fallback already handles large skeletons

The simplicity of full mat4 outweighs the marginal space saving.

## Bone Attachment

**Decision: Bones are regular scene nodes** (Lynx, Mantis, Shark, Wren approach)

See SCENE-GRAPH.md. Attaching a mesh to a bone is simply `bone.add(mesh)`. The scene graph handles world matrix propagation automatically. No dedicated `BoneAttachment` class needed.

## Coordinate Conversion

All glTF animation keyframe data is converted from Y-up to Z-up at import time. This is a one-time cost during loading. Bones store their transforms in engine-native Z-up coordinates.

## Loop Modes

- **Repeat**: Loop from start (default)
- **Once**: Play once and stop at last frame
- **PingPong**: Alternate forward and reverse playback

## Performance

| Metric | Budget |
|--------|--------|
| Keyframe sampling (per character) | ~0.02ms |
| Blend computation | ~0.01ms |
| Bone matrix computation | ~0.01ms |
| GPU skinning (10K vertices) | ~0.1ms |
| Total per character | ~0.14ms |
| Max animated characters @ 60fps | ~40 (at 6ms CPU budget) |

All temporary vectors/quats are pre-allocated. Zero GC pressure during animation playback.
