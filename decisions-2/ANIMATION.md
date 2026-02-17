# ANIMATION.md - Decisions

## Decision: Action-Based Mixer, mat4 Bones, Keyframe Cache, Slerp Blending

### Core Data Structures

Universal agreement across all 8 active implementations:

- **Skeleton**: Flat array of bones with parent indices and inverse bind matrices
- **Bone influences**: Up to 4 per vertex (joints uint8 + weights float32)
- **Animation clips**: Named sets of keyframe tracks per bone
- **Track properties**: Position (vec3), rotation (quat), scale (vec3)
- **Keyframe format**: Sorted times array + values array

### GPU Skinning

Universal agreement: vertex shader skinning.

```glsl
mat4 boneMatrix =
  a_weights.x * u_bones[a_joints.x] +
  a_weights.y * u_bones[a_joints.y] +
  a_weights.z * u_bones[a_joints.z] +
  a_weights.w * u_bones[a_joints.w];

vec4 skinnedPos = boneMatrix * vec4(a_position, 1.0);
```

Where `u_bones[i] = boneWorldMatrix * inverseBindMatrix`.

### Bone Matrix Storage: Full mat4

**Chosen**: Full 4x4 matrices (7/9 implementations)
**Considered**: mat4x3 (Rabbit) - saves 25% UBO space (12 floats vs 16 per bone) but adds complexity to the shader and breaks the standard mat4 multiplication path

Full mat4 is simpler and universally supported. For the target use case (characters in a stylized game), skeletons rarely exceed 64 bones, so the extra 25% UBO cost is negligible.

**UBO limit**: 128 bones (128 * 64 bytes = 8192 bytes, well within WebGL2's 16KB minimum UBO size)

**Fallback for >128 bones**: Bone texture (RGBA32F, 4 texels per bone). Used by 6/9 implementations. Pack matrices into a float texture and `texelFetch` in the vertex shader. Rarely needed in practice.

### Animation Mixer: Action-Based

**Chosen**: Action-based mixer with explicit `AnimationAction` objects (4/9: Caracal, Lynx, Mantis, Wren)
**Rejected**: Simple play/crossFadeTo API (Hyena/Shark) - insufficient for layered blending scenarios

```typescript
const mixer = createAnimationMixer(skeleton)

// Each clip gets a reusable action
const idle = mixer.clipAction(idleClip)
const walk = mixer.clipAction(walkClip)

idle.play()

// Later: crossfade from idle to walk
idle.crossFadeTo(walk, 0.3)

// Per frame
mixer.update(deltaTime)
```

Actions expose: `play()`, `stop()`, `pause()`, `setWeight(w)`, `setSpeed(s)`, `crossFadeTo(other, duration)`.

The action-based design supports future scenarios like upper-body/lower-body blending with independent weights.

### Keyframe Interpolation: Binary Search with Cache

**Chosen**: Binary search with cached last keyframe index (5/9: Caracal, Fennec, Hyena, Mantis, Shark)
**Rejected**: Always binary search (Lynx/Rabbit/Wren) - works but ~30% slower for sequential playback

```typescript
interface TrackState {
  lastKeyIndex: number  // Cache for O(1) amortized lookup
}

const findKeyframe = (times: Float32Array, t: number, state: TrackState): number => {
  // Check if time is between cached index and next
  const i = state.lastKeyIndex
  if (t >= times[i] && t < times[i + 1]) return i

  // Check next index (sequential playback optimization)
  if (t >= times[i + 1] && i + 2 < times.length && t < times[i + 2]) {
    state.lastKeyIndex = i + 1
    return i + 1
  }

  // Fall back to binary search
  state.lastKeyIndex = binarySearch(times, t)
  return state.lastKeyIndex
}
```

For sequential playback (the common case), this gives O(1) amortized lookup per track per frame.

### Quaternion Blending: Slerp

**Chosen**: Slerp for rotation blending (6/9: Caracal, Fennec, Hyena, Lynx, Mantis, Shark)
**Rejected**: Nlerp (Rabbit/Wren) - faster but non-constant angular velocity, noticeable on long crossfades

Slerp provides constant angular velocity interpolation, which matters for crossfades > 0.5s where nlerp produces visible acceleration/deceleration.

### Animation Blend Normalization

**Chosen**: Normalized weights (4/9: Fennec, Lynx, Rabbit, Wren)

```typescript
// During crossfade, weights sum to 1.0 by construction
// For arbitrary multi-action blending, normalize:
finalTransform = sum(action.value * action.weight) / sum(action.weight)
```

This is more robust than assuming weights sum to 1.0, especially when multiple actions overlap during transitions.

### Bone Attachment

**Chosen**: Scene graph integration - bones are nodes, use `bone.add(mesh)` (see SCENE-GRAPH.md)

No special attachment API needed. The animation system updates bone world matrices each frame, and any child of a bone inherits its transform through normal scene graph propagation.

### Interpolation Modes

- **Linear**: Position/scale lerp, rotation slerp (all implementations)
- **Step**: Snap to nearest keyframe (all implementations)
- **Cubic spline**: Support for glTF cubic spline interpolation (Fennec/Rabbit/Wren, 3/9)

Include cubic spline support for full glTF compatibility, even though most exporters use linear interpolation by default.

### Loop Modes

- **Repeat**: Loop from start when reaching end (default)
- **Once**: Play once, then stop
- **PingPong**: Alternate forward/backward playback

### Memory Layout

Universal agreement: contiguous bone matrix array, single GPU upload per skeleton per frame.

```typescript
boneMatrices: Float32Array(boneCount * 16)
// Layout: [bone0_m00, bone0_m01, ..., bone0_m33, bone1_m00, ...]
```

Single `writeBuffer` call per skeleton per frame. All temp vectors/quats pre-allocated, zero GC pressure during playback.

### Performance Budget

~0.04ms CPU per animated character (universal across implementations). For a scene with 50 animated characters: ~2ms total animation cost, well within budget.
