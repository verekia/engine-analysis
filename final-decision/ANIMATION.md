# ANIMATION — Final Decision

## Skeleton Structure

**Decision: Flat bone array with parent indices and inverse bind matrices** (universal agreement)

```typescript
interface Skeleton {
  bones: Node[]                  // Bone nodes in the scene graph
  boneInverseBindMatrices: Mat4[] // Inverse bind pose per bone
  boneMatrices: Float32Array     // Computed skinning matrices (contiguous)
}
```

Bones are regular scene nodes. The flat array with parent indices enables efficient iteration. Inverse bind matrices are loaded from glTF and stored once.

## GPU Skinning

**Decision: Vertex shader skinning with 4 bones per vertex** (universal agreement)

```glsl
mat4 boneMatrix = a_weights.x * u_boneMatrices[a_joints.x]
               + a_weights.y * u_boneMatrices[a_joints.y]
               + a_weights.z * u_boneMatrices[a_joints.z]
               + a_weights.w * u_boneMatrices[a_joints.w];

vec4 skinnedPosition = boneMatrix * vec4(a_position, 1.0);
vec3 skinnedNormal = mat3(boneMatrix) * a_normal;
```

### Bone Matrix Storage

**Decision: Full mat4 (16 floats per bone) in UBO, texture fallback for >128 bones** (universal agreement)

```
UBO capacity: 128 bones × 64 bytes = 8KB (within UBO size limits)
Texture fallback: RGBA32F texture, 4 texels per bone matrix
```

For characters with ≤128 bones (vast majority), bone matrices are uploaded as a UBO array. For larger skeletons, a floating-point texture is used instead.

The contiguous `Float32Array` of bone matrices is uploaded directly — no per-bone copies needed.

## Animation Clips

**Decision: Named clips with per-bone keyframe tracks** (universal agreement)

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
  values: Float32Array
  interpolation: 'linear' | 'step' | 'cubicspline'
}
```

## Keyframe Interpolation

**Decision: Binary search with cached last keyframe index** (universal agreement)

```typescript
// O(1) for sequential playback (common case)
// O(log n) for random seek
const findKeyframe = (times: Float32Array, time: number, lastIndex: number): number => {
  // Check if we're still in the same segment (sequential playback)
  if (lastIndex < times.length - 1 && time >= times[lastIndex] && time < times[lastIndex + 1]) {
    return lastIndex
  }
  // Binary search fallback
  return binarySearch(times, time)
}
```

Interpolation modes:
- **Linear**: lerp for position/scale, slerp for rotation
- **Step**: snap to nearest keyframe
- **Cubic spline**: glTF cubic spline with in/out tangents

## Animation Mixer

**Decision: Action-based mixer API** (6/9: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit)

```typescript
const mixer = createAnimationMixer(skeleton)

const idle = mixer.clipAction(idleClip)
const walk = mixer.clipAction(walkClip)

idle.play()
idle.crossFadeTo(walk, 0.3)  // 300ms crossfade

walk.loop = 'repeat'     // 'repeat' | 'once' | 'pingpong'
walk.timeScale = 1.5     // Playback speed
walk.weight = 1.0        // Blend weight
```

### Blending

Multiple actions can play simultaneously with normalized weights:

```typescript
// If idle.weight = 0.3 and walk.weight = 0.7, the final pose is:
// normalizedIdle = 0.3 / (0.3 + 0.7) = 0.3
// normalizedWalk = 0.7 / (0.3 + 0.7) = 0.7
```

Rotation blending uses slerp. Position and scale use linear interpolation.

### Crossfade

```typescript
action.crossFadeTo(targetAction, duration)
```

Smoothly transitions weight from current action to target over the specified duration. Source weight fades to 0, target weight fades to 1.

## Mixer Update

```typescript
// Each frame:
mixer.update(deltaTime)
```

Update sequence:
1. Advance action times
2. Update crossfade weights
3. Sample keyframes for all active actions
4. Blend bone transforms (weighted)
5. Compute final bone matrices: `boneMatrix[i] = worldMatrix[bone[i]] × inverseBindMatrix[i]`
6. Upload contiguous bone matrix array to GPU

## Bone Attachment

Bones are regular scene nodes in the hierarchy. Attaching objects to bones is standard parenting:

```typescript
const handBone = skeleton.getBone('hand_R')
handBone.add(swordMesh)
```

The attached mesh inherits the bone's animated world transform automatically.

## Coordinate Conversion

**Decision: Y-up to Z-up conversion baked at import time** (universal agreement)

glTF uses Y-up. Bone rest poses and keyframe data are converted to Z-up during loading, not at runtime. This means the animation system operates entirely in Z-up coordinates with no per-frame conversion overhead.

## Memory

- Pre-allocated bone transform arrays sized at skeleton creation
- Pre-allocated scratch quaternions and vectors for blending
- Zero allocations during `mixer.update()`
- Performance budget: ~0.04ms per animated character (60-bone skeleton)
