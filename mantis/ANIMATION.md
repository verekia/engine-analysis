# Skeletal Animation

## Overview

Mantis supports skeletal animation with GPU skinning, weighted crossfade
blending between clips, and the ability to attach static meshes to bones
(e.g., a sword to a hand).

## Architecture

```
AnimationClip          — Keyframe data for one animation (idle, run, attack)
  └── Track[]          — One track per animated joint per channel (T/R/S)
       └── Keyframe[]  — Time + value pairs

Skeleton               — Bone hierarchy + inverse bind matrices
  └── Joint[]          — Index, name, parent index, inverse bind matrix

AnimationMixer         — Plays clips on a skeleton, handles blending
  └── ActiveAction[]   — Currently playing clips with weights and times

SkinnedMesh            — Mesh with joint indices + weights per vertex
```

## Skeleton

A skeleton is a flat array of joints with parent indices:

```typescript
interface Skeleton {
  jointCount: number
  jointNames: string[]                    // for bone attachment lookup
  parentIndices: Int32Array               // -1 for root joints
  inverseBindMatrices: Float32Array       // jointCount × 16 floats (mat4 each)
  jointMatrices: Float32Array             // jointCount × 16 floats (computed each frame)
  jointWorldMatrices: Float32Array        // jointCount × 16 floats (for bone attachment)
}
```

### Joint Matrix Computation

Each frame, the animation mixer produces local transforms for each joint. These
are combined with the skeleton hierarchy and inverse bind matrices to produce
the final joint matrices uploaded to the GPU:

```
For each joint i:
  1. localMatrix[i]    = compose(animPosition[i], animRotation[i], animScale[i])
  2. worldMatrix[i]    = worldMatrix[parent[i]] × localMatrix[i]
  3. jointMatrix[i]    = worldMatrix[i] × inverseBindMatrix[i]
```

The `jointMatrix` array is what the vertex shader uses to skin vertices.

```typescript
const computeJointMatrices = (skeleton: Skeleton, localTransforms: Float32Array) => {
  const { jointCount, parentIndices, inverseBindMatrices, jointMatrices, jointWorldMatrices } = skeleton

  for (let i = 0; i < jointCount; i++) {
    // Compose local matrix from animated T/R/S
    composeMatrix(localTransforms, i, tempLocal)

    // World = parent.world × local
    const parent = parentIndices[i]
    if (parent === -1) {
      jointWorldMatrices.set(tempLocal, i * 16)
    } else {
      mat4Multiply(
        jointWorldMatrices, parent * 16,
        tempLocal, 0,
        jointWorldMatrices, i * 16
      )
    }

    // Joint = world × inverseBind
    mat4Multiply(
      jointWorldMatrices, i * 16,
      inverseBindMatrices, i * 16,
      jointMatrices, i * 16
    )
  }
}
```

## Animation Clips

An animation clip contains keyframe tracks for one or more joints:

```typescript
interface AnimationClip {
  name: string
  duration: number           // seconds
  tracks: AnimationTrack[]
}

interface AnimationTrack {
  jointIndex: number
  channel: 'translation' | 'rotation' | 'scale'
  times: Float32Array        // keyframe timestamps
  values: Float32Array       // keyframe values (vec3 for T/S, quat for R)
  interpolation: 'linear' | 'step'
}
```

### Keyframe Sampling

For a given time `t`, find the two surrounding keyframes and interpolate:

```typescript
const sampleTrack = (track: AnimationTrack, time: number, out: Float32Array, outOffset: number) => {
  const { times, values, channel, interpolation } = track
  const keyCount = times.length

  // Clamp to clip range
  if (time <= times[0]) {
    copyValues(values, 0, out, outOffset, channel)
    return
  }
  if (time >= times[keyCount - 1]) {
    copyValues(values, keyCount - 1, out, outOffset, channel)
    return
  }

  // Binary search for surrounding keyframes
  let lo = 0, hi = keyCount - 1
  while (lo < hi - 1) {
    const mid = (lo + hi) >> 1
    if (times[mid] <= time) lo = mid
    else hi = mid
  }

  const t0 = times[lo]
  const t1 = times[hi]
  const alpha = (time - t0) / (t1 - t0)

  if (interpolation === 'step') {
    copyValues(values, lo, out, outOffset, channel)
    return
  }

  // Linear interpolation
  if (channel === 'rotation') {
    // Slerp for quaternions
    slerpValues(values, lo, hi, alpha, out, outOffset)
  } else {
    // Lerp for translation/scale
    lerpValues(values, lo, hi, alpha, out, outOffset, channel === 'translation' ? 3 : 3)
  }
}
```

**Optimization: Sequential access hint.** Animations typically play forward.
A per-track `lastKeyIndex` hint avoids binary search in the common case:

```typescript
// Start search from last known position
if (track.lastKeyIndex < keyCount - 1 && times[track.lastKeyIndex] <= time && time < times[track.lastKeyIndex + 1]) {
  lo = track.lastKeyIndex
  hi = track.lastKeyIndex + 1
} else {
  // Fall back to binary search
}
track.lastKeyIndex = lo
```

## Animation Mixer

The mixer manages playback of one or more clips on a skeleton, handling
crossfade blending:

```typescript
interface AnimationMixer {
  skeleton: Skeleton
  actions: ActiveAction[]

  play(clip: AnimationClip, options?: PlayOptions): ActiveAction
  crossFadeTo(clip: AnimationClip, duration: number): ActiveAction
  stop(action: ActiveAction): void
  update(deltaTime: number): void
}

interface ActiveAction {
  clip: AnimationClip
  time: number             // current playback time
  speed: number            // playback speed multiplier
  weight: number           // blend weight (0–1)
  loop: boolean            // loop playback
  fadeIn: number           // remaining fade-in duration
  fadeOut: number          // remaining fade-out duration
}

interface PlayOptions {
  speed?: number           // default 1.0
  loop?: boolean           // default true
  weight?: number          // default 1.0
  fadeIn?: number          // seconds to fade in from weight 0
}
```

### Crossfade Blending

A crossfade smoothly transitions from clip A to clip B over a given duration:

```typescript
const crossFadeTo = (mixer: AnimationMixer, newClip: AnimationClip, duration: number) => {
  // Fade out all current actions
  for (const action of mixer.actions) {
    action.fadeOut = duration
  }

  // Start new action with fade in
  const newAction = mixer.play(newClip, { fadeIn: duration, weight: 0 })
  return newAction
}
```

During `update`, weights are adjusted:

```typescript
const updateMixer = (mixer: AnimationMixer, dt: number) => {
  // Update action times and weights
  for (let i = mixer.actions.length - 1; i >= 0; i--) {
    const action = mixer.actions[i]

    // Advance time
    action.time += dt * action.speed
    if (action.loop) {
      action.time = action.time % action.clip.duration
    } else {
      action.time = Math.min(action.time, action.clip.duration)
    }

    // Fade in
    if (action.fadeIn > 0) {
      action.weight = Math.min(action.weight + dt / action.fadeIn, 1.0)
      action.fadeIn = Math.max(action.fadeIn - dt, 0)
    }

    // Fade out
    if (action.fadeOut > 0) {
      action.weight = Math.max(action.weight - dt / action.fadeOut, 0)
      action.fadeOut = Math.max(action.fadeOut - dt, 0)
      if (action.weight <= 0) {
        mixer.actions.splice(i, 1) // remove fully faded out action
        continue
      }
    }
  }

  // Sample all actions and blend
  blendActions(mixer)
}
```

### Blending Strategy

When multiple actions are active (during crossfade), their sampled poses are
blended by weight:

```typescript
const blendActions = (mixer: AnimationMixer) => {
  const { skeleton, actions } = mixer
  const jointCount = skeleton.jointCount

  // Clear output transforms
  outputTranslations.fill(0)
  outputScales.fill(0)
  // Rotations need special handling (slerp, not lerp)
  let firstRotation = true
  let totalWeight = 0

  for (const action of actions) {
    if (action.weight <= 0) continue

    // Sample this clip at current time
    sampleClip(action.clip, action.time, tempTranslations, tempRotations, tempScales)

    const w = action.weight
    totalWeight += w

    // Additive blend for translation and scale
    for (let j = 0; j < jointCount * 3; j++) {
      outputTranslations[j] += tempTranslations[j] * w
      outputScales[j] += tempScales[j] * w
    }

    // Slerp blend for rotations
    if (firstRotation) {
      outputRotations.set(tempRotations)
      firstRotation = false
    } else {
      for (let j = 0; j < jointCount; j++) {
        const alpha = w / totalWeight
        slerpInPlace(outputRotations, j * 4, tempRotations, j * 4, alpha)
      }
    }
  }

  // Normalize translation/scale by total weight
  if (totalWeight > 0 && totalWeight !== 1) {
    const invW = 1 / totalWeight
    for (let j = 0; j < jointCount * 3; j++) {
      outputTranslations[j] *= invW
      outputScales[j] *= invW
    }
  }

  // Compute joint matrices from blended transforms
  computeJointMatrices(skeleton, outputTranslations, outputRotations, outputScales)
}
```

## GPU Skinning

### Vertex Attributes

Skinned meshes have two additional vertex attributes:

```
JOINTS_0:  uvec4  — 4 bone indices per vertex (uint8 or uint16)
WEIGHTS_0: vec4   — 4 bone weights per vertex (float32, sum to 1.0)
```

### Joint Matrix Upload

Joint matrices are uploaded as a **texture** (for > 64 joints) or a **UBO**
(for ≤ 64 joints):

**UBO path (≤ 64 joints, 4 KB):**
```
layout(std140) uniform JointMatrices {
  mat4 joints[64];  // 64 × 64 bytes = 4096 bytes
};
```

**Texture path (> 64 joints):**
A `RGBA32F` texture of size `(4 × jointCount) × 1`. Each joint occupies 4
texels (one mat4 row per texel). The vertex shader reads with `texelFetch`:

```glsl
mat4 getJointMatrix(uint jointIndex) {
  int x = int(jointIndex) * 4;
  return mat4(
    texelFetch(jointTexture, ivec2(x,     0), 0),
    texelFetch(jointTexture, ivec2(x + 1, 0), 0),
    texelFetch(jointTexture, ivec2(x + 2, 0), 0),
    texelFetch(jointTexture, ivec2(x + 3, 0), 0)
  );
}
```

### Vertex Shader Skinning

```glsl
// Apply bone transforms to position and normal
mat4 skinMatrix =
  getJointMatrix(a_joints.x) * a_weights.x +
  getJointMatrix(a_joints.y) * a_weights.y +
  getJointMatrix(a_joints.z) * a_weights.z +
  getJointMatrix(a_joints.w) * a_weights.w;

vec4 skinnedPosition = skinMatrix * vec4(a_position, 1.0);
vec3 skinnedNormal = mat3(skinMatrix) * a_normal;

gl_Position = camera.vpMatrix * object.modelMatrix * skinnedPosition;
```

## Bone Attachment

To attach a static mesh to a bone (e.g., a sword to a hand bone), use the
bone's world matrix as the mesh's parent transform:

```typescript
// Find the bone by name
const handBoneIndex = skeleton.jointNames.indexOf('Hand_R')

// Each frame, after animation update, copy bone world matrix to mesh
engine.onFrame(() => {
  const boneWorldMatrix = skeleton.jointWorldMatrices.subarray(
    handBoneIndex * 16,
    handBoneIndex * 16 + 16
  )

  // The sword mesh's world matrix = bone world matrix × attachment offset
  mat4Multiply(
    boneWorldMatrix, 0,
    swordOffsetMatrix, 0,        // local offset (grip position/rotation)
    swordMesh.worldMatrix, 0
  )
})
```

**Convenience API:**

```typescript
// Attach sword mesh to the right hand bone with an offset
skeleton.attach(swordMesh, 'Hand_R', {
  position: [0, 0, 0.1],         // slight offset along grip
  rotation: [0, 0, 0.38, 0.92],  // rotated to align with hand
})

// Detach
skeleton.detach(swordMesh)
```

When attached, the mesh's world matrix is updated automatically each frame
after animation update, before rendering. The mesh does not need to be a child
in the scene graph hierarchy — it is positioned via direct matrix assignment.

## Animation API

```typescript
// Load model with animations
const model = await loadGLTF(engine, '/models/character.glb')
const mixer = new AnimationMixer(model.skeleton)

// Play idle animation
const idleAction = mixer.play(model.animations.find(a => a.name === 'Idle'), {
  loop: true,
  speed: 1.0,
})

// Crossfade to run animation over 0.3 seconds
const runClip = model.animations.find(a => a.name === 'Run')
mixer.crossFadeTo(runClip, 0.3)

// Crossfade to attack (no loop), then back to idle
const attackClip = model.animations.find(a => a.name === 'Attack')
const attackAction = mixer.crossFadeTo(attackClip, 0.15)
attackAction.loop = false
attackAction.onFinished = () => {
  mixer.crossFadeTo(model.animations.find(a => a.name === 'Idle'), 0.2)
}

// Update mixer each frame (called automatically if registered with engine)
engine.onFrame((dt) => {
  mixer.update(dt)
})

// Attach weapon to bone
skeleton.attach(swordMesh, 'Hand_R')
```

## Performance

| Operation | Cost (per frame) |
|---|---|
| Sample 1 clip (60 joints) | ~0.08 ms |
| Blend 2 clips (crossfade) | ~0.12 ms |
| Compute joint matrices | ~0.05 ms |
| Upload joint texture | ~0.02 ms (1 writeTexture call) |
| Vertex shader skinning | GPU-bound, ~0.1 ms for 10K vertices |

Total animation cost for one character: ~0.25 ms CPU + ~0.1 ms GPU.
Ten animated characters: ~2.5 ms CPU — fits within budget.
