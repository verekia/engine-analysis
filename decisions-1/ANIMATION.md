# ANIMATION - Design Decisions

## Core Data Structures

**Decision: Flat bone array with parent indices and inverse bind matrices (universal agreement)**

```typescript
interface Skeleton {
  bones: Node[]                          // Bone nodes in the scene graph
  inverseBindMatrices: Float32Array[]     // One mat4 per bone
  boneMatrices: Float32Array             // Contiguous: boneCount × 16 floats
}
```

- Up to 4 bone influences per vertex (joints + weights attributes)
- Bone matrix computation: `boneMatrix = jointWorldMatrix × inverseBindMatrix`

## GPU Skinning

**Decision: Vertex shader skinning with UBO storage, texture fallback for large skeletons**

- Sources: All 8 active implementations agree on GPU skinning

```glsl
// Vertex shader
mat4 skinMatrix =
  a_weights.x * u_boneMatrices[a_joints.x] +
  a_weights.y * u_boneMatrices[a_joints.y] +
  a_weights.z * u_boneMatrices[a_joints.z] +
  a_weights.w * u_boneMatrices[a_joints.w];

vec4 skinnedPosition = skinMatrix * vec4(a_position, 1.0);
```

## Bone Matrix Storage

**Decision: Full mat4 (16 floats per bone) in UBO, texture fallback for >128 bones**

- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Shark, Wren (7/8 active)
- Considered: mat4x3 optimization from Rabbit (25% space savings, fits ~170 bones in 16KB UBO)
- Verdict: Use full mat4 for simplicity. The mat4x3 optimization can be added later if UBO limits become a bottleneck in practice. Most game characters have <100 bones.

**UBO limit**: 128 bones × 64 bytes = 8192 bytes (well within 16KB minimum UBO size)

**Texture fallback for >128 bones**: Pack bone matrices into RGBA32F texture (4 texels per bone), sample via `texelFetch` in vertex shader. Supported on both WebGL2 and WebGPU.

## Animation Clips and Keyframes

**Decision: Named clips with per-bone keyframe tracks (universal agreement)**

```typescript
interface AnimationClip {
  name: string
  duration: number
  tracks: KeyframeTrack[]
}

interface KeyframeTrack {
  boneIndex: number
  property: 'position' | 'rotation' | 'scale'
  times: Float32Array
  values: Float32Array    // vec3 for position/scale, quat for rotation
  interpolation: 'linear' | 'step' | 'cubicspline'
}
```

## Interpolation Support

**Decision: Linear + Step + Cubic spline (future-proof for glTF)**

- Sources: Fennec, Rabbit, Wren (3/8 support cubic spline)
- Linear and step are essential; cubic spline ensures full glTF compatibility
- Most exporters use linear by default, so cubic spline is rarely exercised but should work

## Keyframe Sampling

**Decision: Binary search with cached last keyframe index**

- Sources: Caracal, Fennec, Hyena, Mantis, Shark (5/8) cache last index

```typescript
// Per track: cache last keyframe index for O(1) amortized sequential playback
let lastKeyIndex = 0

const findKeyframe = (times: Float32Array, t: number): number => {
  // Start search from lastKeyIndex (likely nearby for sequential playback)
  if (times[lastKeyIndex] <= t && times[lastKeyIndex + 1] > t) {
    return lastKeyIndex  // O(1) hit
  }
  // Fall back to binary search
  lastKeyIndex = binarySearch(times, t)
  return lastKeyIndex
}
```

~30% faster animation sampling in typical sequential playback compared to always binary searching.

## Animation Mixer API

**Decision: Action-based mixer (explicit AnimationAction objects)**

- Sources: Caracal, Lynx, Mantis, Wren (4/8)
- Rejected: Simple play/crossFade API (Hyena, Shark) — insufficient for complex blending scenarios

```typescript
const mixer = createAnimationMixer(skeleton)

// Create reusable actions
const idleAction = mixer.clipAction(idleClip)
const walkAction = mixer.clipAction(walkClip)
const runAction = mixer.clipAction(runClip)

// Play
idleAction.play()

// Crossfade
idleAction.crossFadeTo(walkAction, 0.3)

// Per-action control
walkAction.timeScale = 1.5
walkAction.weight = 0.7

// Update each frame
mixer.update(deltaTime)
```

Rationale: Action-based API supports complex multi-layer blending (e.g., upper body attack animation + lower body walk animation), which a simple play/crossFade API cannot handle.

## Crossfade Blending

**Decision: Linear weight interpolation with slerp for rotations, normalized weights**

- Sources: Slerp from Caracal, Fennec, Hyena, Lynx, Mantis, Shark (6/8); normalized weights from Fennec, Lynx, Rabbit, Wren (4/8)

```typescript
// During crossfade:
// outgoing: weight 1→0 over duration
// incoming: weight 0→1 over duration

// Per bone:
// position/scale: weighted sum, then normalize by total weight
// rotation: weighted slerp, then normalize

finalPosition = sum(action.position * action.weight) / sum(weights)
finalRotation = normalizedSlerp(actions, weights)
```

Normalized weights handle edge cases where total weight ≠ 1 (e.g., during crossfade ramp-up/down).

Slerp rather than nlerp (Rabbit, Wren) because slerp provides constant angular velocity, which matters for smooth visual quality during transitions.

## Bone Attachment

**Decision: Bones are regular scene nodes — standard parent/child attachment**

- Sources: Lynx, Mantis, Shark, Wren (4/8)
- Rejected: Dedicated BoneAttachment class (Caracal, Fennec)

```typescript
const handBone = skeleton.getBone('hand_R')
handBone.add(swordMesh)
// swordMesh world matrix automatically follows hand bone
```

No special API needed. The scene graph's dirty flag propagation handles the update chain: animation system writes bone local transforms → dirty flags propagate → world matrices recompute → attached mesh follows.

## Coordinate Conversion

**Decision: Y-up to Z-up conversion baked into animation keyframes at import time**

Animation keyframes from glTF (Y-up) are converted to Z-up at load time. No runtime conversion cost.

## Playback Modes

**Decision: Repeat, Once, PingPong (standard set)**

```typescript
action.loop = 'repeat'    // default: loop forever
action.loop = 'once'      // play once, hold last frame
action.loop = 'pingpong'  // play forward then backward

action.timeScale = 1.0    // playback speed multiplier
```

## Zero Allocations

**Decision: Pre-allocated bone transform arrays, pooled temporaries**

- Sources: All 8 active implementations agree
- Blend state arrays pre-allocated to bone count
- All temporary Vec3/Quat from scratch pool
- No `new` calls during animation update
- Single `writeBuffer` upload per skeleton per frame (contiguous bone matrix array)

## Performance Characteristics

- CPU cost per animated character: ~0.04ms (keyframe sampling + blend + matrix compute)
- GPU cost: ~0.1ms per 10K skinned vertices (memory bandwidth dominated)
- Fits well within 16.6ms frame budget even with 20+ animated characters
