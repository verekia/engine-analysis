# Skeletal Animation System

## Overview

Caracal supports GPU-accelerated skeletal animation with:
- Up to 4 bone influences per vertex
- Animation clips with keyframed position/rotation/scale tracks
- Linear crossfade blending between animations
- Attaching static meshes to bones (weapons, accessories)

## Architecture

```
AnimationMixer
├── manages multiple AnimationActions
├── blends actions during crossfade
└── writes final bone matrices to Skeleton

Skeleton
├── holds Bone hierarchy (array of Bone nodes)
├── stores inverse bind matrices
├── stores final bone matrices (Float32Array for GPU upload)
└── computes bone world matrices from animation pose

AnimationClip
├── named set of tracks
├── duration, loop mode
└── contains AnimationTrack[]

AnimationTrack
├── targets a specific bone by index
├── keyframes: times[] + values[]
├── interpolation: position (lerp), rotation (slerp), scale (lerp)
```

## Skeleton

A skeleton is a hierarchy of bones with pre-computed inverse bind matrices:

```typescript
interface Skeleton {
  readonly bones: Bone[]                    // Flat array of all bones
  readonly inverseBindMatrices: Mat4[]      // One per bone, from glTF
  readonly boneMatrices: Float32Array       // Final matrices for GPU (bones.length × 16)
  readonly boneTexture: GPUTextureHandle | null  // For GPU skinning via texture

  update(): void  // Recompute boneMatrices from current bone transforms
}
```

### Bone Matrix Computation

Each frame, the skeleton computes the final bone matrices used by the vertex shader:

```
boneMatrix[i] = skeleton.rootWorldInverse × bone[i].worldMatrix × inverseBindMatrix[i]
```

Where:
- `skeleton.rootWorldInverse` — inverse of the skeleton root's world transform (anchors bones to mesh space)
- `bone[i].worldMatrix` — the bone's current world transform (computed from animation pose + hierarchy)
- `inverseBindMatrix[i]` — transforms from mesh space to bone space at bind pose (from glTF)

### GPU Skinning

Bone matrices are uploaded to the GPU as either:
- **Uniform array** (WebGL 2 / WebGPU): `uniform mat4 u_boneMatrices[MAX_BONES]` — fast for <64 bones
- **Bone texture** (for >64 bones): Bone matrices packed into a `RGBA32F` texture, sampled in the vertex shader

The vertex shader applies skinning:

```glsl
// GLSL 300 ES
in uvec4 a_skinIndex;   // 4 bone indices
in vec4 a_skinWeight;   // 4 bone weights

uniform mat4 u_boneMatrices[MAX_BONES]; // MAX_BONES = 64 or 128

void main() {
  mat4 skinMatrix =
    u_boneMatrices[a_skinIndex.x] * a_skinWeight.x +
    u_boneMatrices[a_skinIndex.y] * a_skinWeight.y +
    u_boneMatrices[a_skinIndex.z] * a_skinWeight.z +
    u_boneMatrices[a_skinIndex.w] * a_skinWeight.w;

  vec4 skinnedPosition = skinMatrix * vec4(a_position, 1.0);
  vec3 skinnedNormal = mat3(skinMatrix) * a_normal;

  gl_Position = u_viewProjection * u_worldMatrix * skinnedPosition;
  // ...
}
```

## Animation Clips

An animation clip contains keyframe tracks for one or more bones:

```typescript
interface AnimationClip {
  name: string
  duration: number        // In seconds
  tracks: AnimationTrack[]
}

interface AnimationTrack {
  boneIndex: number       // Which bone this track animates
  property: 'position' | 'rotation' | 'scale'
  times: Float32Array     // Keyframe timestamps
  values: Float32Array    // Keyframe values (vec3 for pos/scale, quat for rotation)
  interpolation: 'linear' | 'step'
}
```

### Keyframe Sampling

To evaluate a track at time `t`:

1. Binary search `times[]` to find the surrounding keyframes `[i, i+1]`
2. Compute interpolation factor: `alpha = (t - times[i]) / (times[i+1] - times[i])`
3. Interpolate:
   - Position/Scale: `lerp(values[i], values[i+1], alpha)`
   - Rotation: `slerp(values[i], values[i+1], alpha)` (quaternion spherical interpolation)

```typescript
const sampleTrack = (track: AnimationTrack, time: number, out: Float32Array) => {
  const times = track.times
  const values = track.values
  const n = times.length

  // Clamp to range
  if (time <= times[0]) {
    copyValues(out, values, 0, track.property)
    return
  }
  if (time >= times[n - 1]) {
    copyValues(out, values, n - 1, track.property)
    return
  }

  // Binary search for surrounding keyframes
  let lo = 0, hi = n - 1
  while (hi - lo > 1) {
    const mid = (lo + hi) >> 1
    if (times[mid] <= time) lo = mid
    else hi = mid
  }

  const alpha = (time - times[lo]) / (times[hi] - times[lo])

  if (track.property === 'rotation') {
    slerpQuat(out, values, lo, hi, alpha)
  } else {
    lerpVec3(out, values, lo, hi, alpha)
  }
}
```

The binary search result for each track is cached between frames (since time usually advances monotonically), making subsequent lookups O(1) in the common case.

## Animation Mixer

The mixer manages playback of animation clips on a skeleton:

```typescript
interface AnimationMixer {
  readonly skeleton: Skeleton

  // Playback control
  play(clip: AnimationClip, options?: PlayOptions): AnimationAction
  crossFadeTo(clip: AnimationClip, duration: number): AnimationAction
  stop(): void

  // Frame update
  update(deltaTime: number): void
}

interface PlayOptions {
  loop?: boolean          // Default: true
  speed?: number          // Playback speed multiplier (default: 1)
  startTime?: number      // Start from specific time (default: 0)
}

interface AnimationAction {
  clip: AnimationClip
  time: number            // Current playback time
  weight: number          // Blend weight (0-1), used during crossfade
  speed: number
  loop: boolean
  paused: boolean
  isRunning: boolean
}
```

### Crossfade

Crossfading smoothly transitions between two animations over a specified duration:

```typescript
// Example: transition from idle to walk
const idleAction = mixer.play(idleClip)

// Later, when the character starts moving:
const walkAction = mixer.crossFadeTo(walkClip, 0.3) // 0.3 second crossfade
```

During the crossfade:
1. The **outgoing** action's weight decreases linearly from 1 → 0
2. The **incoming** action's weight increases linearly from 0 → 1
3. Both actions are sampled and blended per-bone:

```typescript
const updateMixer = (mixer: AnimationMixer, dt: number) => {
  // Update crossfade weights
  if (mixer._crossfadeActive) {
    mixer._crossfadeElapsed += dt
    const t = Math.min(mixer._crossfadeElapsed / mixer._crossfadeDuration, 1)

    mixer._outgoingAction.weight = 1 - t
    mixer._incomingAction.weight = t

    if (t >= 1) {
      // Crossfade complete — remove outgoing action
      mixer._outgoingAction.isRunning = false
      mixer._crossfadeActive = false
    }
  }

  // Sample all active actions and blend
  for (const action of mixer._activeActions) {
    if (!action.isRunning || action.paused) continue

    // Advance time
    action.time += dt * action.speed
    if (action.loop) {
      action.time = action.time % action.clip.duration
    } else {
      action.time = Math.min(action.time, action.clip.duration)
    }

    // Sample each track
    for (const track of action.clip.tracks) {
      const bone = mixer.skeleton.bones[track.boneIndex]
      sampleTrack(track, action.time, _tempValues)

      // Blend with weight
      if (action.weight < 1) {
        blendBoneProperty(bone, track.property, _tempValues, action.weight)
      } else {
        setBoneProperty(bone, track.property, _tempValues)
      }
    }
  }

  // Update skeleton bone matrices
  mixer.skeleton.update()
}
```

### Blend Function

Per-bone blending during crossfade:

```typescript
const blendBoneProperty = (
  bone: Bone,
  property: 'position' | 'rotation' | 'scale',
  values: Float32Array,
  weight: number
) => {
  if (property === 'rotation') {
    // Slerp between current rotation and sampled rotation
    quatSlerp(bone.rotation, bone.rotation, valuesAsQuat, weight)
  } else {
    // Lerp position or scale
    const target = property === 'position' ? bone.position : bone.scale
    vec3Lerp(target, target, valuesAsVec3, weight)
  }
}
```

## Bone Attachment

Static meshes can be attached to bones so they follow the bone's transform:

```typescript
interface BoneAttachment {
  mesh: Mesh           // The static mesh to attach
  bone: Bone           // The bone to attach to
  offset: Mat4         // Local offset relative to bone
}

// Usage: attach a sword to the right hand bone
const sword = createMesh(swordGeometry, swordMaterial)
const handBone = skeleton.bones.find(b => b.name === 'hand_R')

const attachment = createBoneAttachment(sword, handBone, {
  position: [0, 0, 0.1],    // Slight offset
  rotation: [0, 0, 0, 1],   // No rotation
})

scene.add(sword) // Sword is in the scene graph but its world matrix is overridden
```

### How It Works

During the skeleton update, after bone world matrices are computed, each attachment's mesh world matrix is overridden:

```typescript
const updateBoneAttachments = (attachments: BoneAttachment[]) => {
  for (const att of attachments) {
    // Sword's world matrix = bone's world matrix × offset
    mat4Multiply(att.mesh._worldMatrix, att.bone._worldMatrix, att.offset)
    att.mesh._dirtyLocal = false // Prevent normal transform update from overwriting
  }
}
```

The attached mesh remains in the scene graph (so it's visible, cullable, and raycastable) but its transform is driven by the bone instead of its parent node.

### Practical Example

```typescript
// Load character with skeleton
const { meshes, skeleton, clips } = await loadGLTF('character.glb')
const character = meshes[0] as SkinnedMesh

// Create animation mixer
const mixer = createAnimationMixer(skeleton)
mixer.play(clips.find(c => c.name === 'Idle')!)

// Load and attach weapon
const { meshes: weaponMeshes } = await loadGLTF('sword.glb')
const sword = weaponMeshes[0]
const handBone = skeleton.bones.find(b => b.name === 'mixamorig:RightHand')!

createBoneAttachment(sword, handBone, {
  position: [0, 0.05, 0],
  rotation: quatFromEuler(0, 0, Math.PI / 4), // Rotate to grip angle
})

scene.add(character)
scene.add(sword)

// In game loop:
mixer.update(dt)
// Sword automatically follows the hand bone
```

## Memory Layout for Bone Matrices

For GPU upload efficiency, bone matrices are stored in a contiguous `Float32Array`:

```
boneMatrices: Float32Array(boneCount × 16)

Layout: [bone0_m00, bone0_m01, ..., bone0_m33, bone1_m00, ..., bone1_m33, ...]
         ├────── 16 floats (64 bytes) ──────┤├────── 16 floats ──────────────┤
```

This is uploaded as a single `bufferSubData` (WebGL) or `writeBuffer` (WebGPU) call per skeleton per frame. For a character with 60 bones, that's 60 × 64 = 3,840 bytes — trivial bandwidth.

## Bone Texture (Large Skeletons)

For skeletons exceeding the uniform array limit (~64 bones on mobile), bone matrices are packed into a floating-point texture:

```
Texture size: (boneCount × 4) × 1 pixels, RGBA32F format
Each bone = 4 texels (4 columns of the 4×4 matrix)

Vertex shader samples:
  vec4 col0 = texelFetch(u_boneTexture, ivec2(boneIndex * 4 + 0, 0), 0);
  vec4 col1 = texelFetch(u_boneTexture, ivec2(boneIndex * 4 + 1, 0), 0);
  vec4 col2 = texelFetch(u_boneTexture, ivec2(boneIndex * 4 + 2, 0), 0);
  vec4 col3 = texelFetch(u_boneTexture, ivec2(boneIndex * 4 + 3, 0), 0);
  mat4 boneMatrix = mat4(col0, col1, col2, col3);
```

This removes the bone count limit entirely and is the fallback for complex characters.
