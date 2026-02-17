# 08 — Skeletal Animation & Crossfade

## Overview

Wren's animation system supports skeletal animation with linear blend skinning (LBS) on the GPU. Animation poses are computed on the CPU (interpolating keyframes), bone matrices are uploaded as uniforms, and the vertex shader applies skinning. Crossfading between animations is achieved by blending two poses on the CPU before upload.

## Skeleton

```ts
interface Skeleton {
  bones: BoneNode[]                       // Ordered by bone index
  boneCount: number
  inverseBindMatrices: Float32Array       // Flat: boneCount * 16 floats
  boneMatrices: Float32Array              // Flat: boneCount * 16 floats (skinning result)
  boneMatricesTexture: GPUTextureHandle | null  // For large skeletons (>64 bones)

  getBoneByName(name: string): BoneNode | null
  getBoneByIndex(index: number): BoneNode
}
```

### Bone Matrix Computation

Each frame, after animation poses are applied:

```ts
const updateBoneMatrices = (skeleton: Skeleton): void => {
  for (let i = 0; i < skeleton.boneCount; i++) {
    const bone = skeleton.bones[i]
    // bone.worldMatrix was updated by scene graph traversal

    // skinningMatrix = bone.worldMatrix * inverseBindMatrix[i]
    mat4Multiply(
      skeleton.boneMatrices,         // Write destination (offset i*16)
      bone.worldMatrix,
      skeleton.inverseBindMatrices,  // Read source (offset i*16)
      i * 16
    )
  }
}
```

### Upload Strategy

For small skeletons (<=64 bones, which covers most game characters):
- Upload as a uniform buffer array: `mat4 u_boneMatrices[64]`
- UBO size: 64 * 64 bytes = 4KB (well within WebGL2's 16KB UBO limit)

For large skeletons (>64 bones):
- Upload as a float texture (RGBA32F), 4 texels per bone (4x4 matrix)
- Sample in vertex shader via `texelFetch`

## Animation Clip

```ts
interface AnimationClip {
  name: string
  duration: number                 // Seconds
  tracks: AnimationTrack[]
}

interface AnimationTrack {
  boneIndex: number
  property: 'position' | 'rotation' | 'scale'
  times: Float32Array              // Keyframe timestamps
  values: Float32Array             // Keyframe values (3 floats for pos/scale, 4 for quat)
  interpolation: 'linear' | 'step' | 'cubicspline'
}
```

### Keyframe Interpolation

```ts
const sampleTrack = (track: AnimationTrack, time: number, output: Float32Array): void => {
  // Binary search for the two keyframes surrounding `time`
  const [i0, i1, t] = findKeyframePair(track.times, time)

  const stride = track.property === 'rotation' ? 4 : 3
  const offset0 = i0 * stride
  const offset1 = i1 * stride

  switch (track.interpolation) {
    case 'step':
      // Use the left keyframe's value
      copyValues(output, track.values, offset0, stride)
      break

    case 'linear':
      if (track.property === 'rotation') {
        // Quaternion spherical interpolation
        quatSlerp(output, track.values, offset0, track.values, offset1, t)
      } else {
        // Linear interpolation for position/scale
        lerpValues(output, track.values, offset0, track.values, offset1, t, stride)
      }
      break

    case 'cubicspline':
      // Cubic Hermite spline (GLTF spec)
      cubicSplineInterpolate(output, track, i0, i1, t)
      break
  }
}
```

### Binary Search for Keyframes

```ts
const findKeyframePair = (times: Float32Array, time: number): [number, number, number] => {
  // Clamp time to clip range
  if (time <= times[0]) return [0, 0, 0]
  if (time >= times[times.length - 1]) {
    const last = times.length - 1
    return [last, last, 0]
  }

  // Binary search
  let lo = 0
  let hi = times.length - 1
  while (hi - lo > 1) {
    const mid = (lo + hi) >> 1
    if (times[mid] <= time) lo = mid
    else hi = mid
  }

  const t = (time - times[lo]) / (times[hi] - times[lo])
  return [lo, hi, t]
}
```

## Animation Mixer

The mixer manages playback of animation clips on a skeleton, supporting crossfading between animations.

```ts
interface AnimationAction {
  clip: AnimationClip
  weight: number             // 0-1, blend weight
  speed: number              // Playback speed multiplier
  time: number               // Current playback time
  loop: boolean              // Loop when reaching end
  playing: boolean
  fadeIn: number             // Remaining fade-in duration (seconds)
  fadeOut: number             // Remaining fade-out duration (seconds)
  fadeDuration: number       // Total fade duration for interpolation
}

interface AnimationMixer {
  skeleton: Skeleton
  actions: AnimationAction[]

  play(clip: AnimationClip, options?: PlayOptions): AnimationAction
  crossFadeTo(newClip: AnimationClip, duration: number, options?: PlayOptions): AnimationAction
  stop(action: AnimationAction): void
  stopAll(): void
  update(deltaTime: number): void
}

interface PlayOptions {
  loop?: boolean              // Default: true
  speed?: number              // Default: 1
  weight?: number             // Default: 1
  startTime?: number          // Default: 0
}
```

### Crossfade Implementation

Crossfading blends two animation poses over a specified duration:

```ts
const crossFadeTo = (mixer: AnimationMixer, newClip: AnimationClip, duration: number, options?: PlayOptions): AnimationAction => {
  // Fade out all currently playing actions
  for (const action of mixer.actions) {
    if (action.playing) {
      action.fadeOut = duration
      action.fadeDuration = duration
    }
  }

  // Create and fade in the new action
  const newAction: AnimationAction = {
    clip: newClip,
    weight: 0,                // Starts at 0, fades to target weight
    speed: options?.speed ?? 1,
    time: options?.startTime ?? 0,
    loop: options?.loop ?? true,
    playing: true,
    fadeIn: duration,
    fadeDuration: duration,
    fadeOut: 0,
  }

  mixer.actions.push(newAction)
  return newAction
}
```

### Mixer Update

Each frame, the mixer:
1. Advances playback time for all active actions
2. Updates fade weights
3. Samples each action's clip at its current time
4. Blends the sampled poses by weight
5. Applies the final blended pose to the skeleton's bone nodes

```ts
const updateMixer = (mixer: AnimationMixer, dt: number): void => {
  // Temporary storage for blended pose
  const blendedPositions = tempPositions   // Pre-allocated
  const blendedRotations = tempRotations
  const blendedScales = tempScales

  let totalWeight = 0

  // Phase 1: Update time and weights for all actions
  for (const action of mixer.actions) {
    if (!action.playing) continue

    // Advance time
    action.time += dt * action.speed
    if (action.loop) {
      action.time = action.time % action.clip.duration
    } else {
      action.time = Math.min(action.time, action.clip.duration)
    }

    // Update fade
    if (action.fadeIn > 0) {
      action.fadeIn -= dt
      action.weight = 1.0 - Math.max(0, action.fadeIn) / action.fadeDuration
    }
    if (action.fadeOut > 0) {
      action.fadeOut -= dt
      action.weight = Math.max(0, action.fadeOut) / action.fadeDuration
      if (action.fadeOut <= 0) {
        action.playing = false  // Fully faded out
      }
    }

    totalWeight += action.weight
  }

  // Normalize weights if they exceed 1
  const weightScale = totalWeight > 1 ? 1 / totalWeight : 1

  // Phase 2: Sample and blend
  let firstAction = true
  for (const action of mixer.actions) {
    if (!action.playing || action.weight <= 0) continue

    const w = action.weight * weightScale

    for (const track of action.clip.tracks) {
      const boneIdx = track.boneIndex
      sampleTrack(track, action.time, tempSample)

      if (firstAction) {
        // First action: write directly (scaled by weight)
        applyTrackValue(blendedPositions, blendedRotations, blendedScales, boneIdx, track.property, tempSample, w)
      } else {
        // Subsequent actions: blend additively
        blendTrackValue(blendedPositions, blendedRotations, blendedScales, boneIdx, track.property, tempSample, w)
      }
    }
    firstAction = false
  }

  // Phase 3: Apply blended pose to skeleton bones
  for (let i = 0; i < mixer.skeleton.boneCount; i++) {
    const bone = mixer.skeleton.bones[i]
    vec3Copy(bone.position, blendedPositions, i * 3)
    quatCopy(bone.rotation, blendedRotations, i * 4)
    vec3Copy(bone.scale, blendedScales, i * 3)
    markDirty(bone)  // Triggers world matrix recomputation
  }

  // Clean up finished actions
  mixer.actions = mixer.actions.filter(a => a.playing)
}
```

### Quaternion Blending

During crossfade, rotation blending uses **nlerp** (normalized linear interpolation) instead of slerp for performance. Nlerp is not constant-speed but is much cheaper and visually indistinguishable for small blend weights during short crossfades:

```ts
const blendQuaternions = (out: Float32Array, a: Float32Array, aOffset: number, b: Float32Array, bOffset: number, t: number): void => {
  // Ensure shortest path
  let dot = a[aOffset] * b[bOffset] + a[aOffset+1] * b[bOffset+1] + a[aOffset+2] * b[bOffset+2] + a[aOffset+3] * b[bOffset+3]
  const sign = dot < 0 ? -1 : 1

  out[0] = a[aOffset]   * (1 - t) + b[bOffset]   * t * sign
  out[1] = a[aOffset+1] * (1 - t) + b[bOffset+1] * t * sign
  out[2] = a[aOffset+2] * (1 - t) + b[bOffset+2] * t * sign
  out[3] = a[aOffset+3] * (1 - t) + b[bOffset+3] * t * sign

  // Normalize
  const len = Math.sqrt(out[0]*out[0] + out[1]*out[1] + out[2]*out[2] + out[3]*out[3])
  out[0] /= len
  out[1] /= len
  out[2] /= len
  out[3] /= len
}
```

## Vertex Shader Skinning

```glsl
#ifdef HAS_SKINNING
  in uvec4 a_joints;    // 4 bone indices
  in vec4 a_weights;    // 4 bone weights

  uniform mat4 u_boneMatrices[MAX_BONES];  // 64

  mat4 getSkinMatrix() {
    mat4 skin = u_boneMatrices[a_joints.x] * a_weights.x
              + u_boneMatrices[a_joints.y] * a_weights.y
              + u_boneMatrices[a_joints.z] * a_weights.z
              + u_boneMatrices[a_joints.w] * a_weights.w;
    return skin;
  }
#endif

void main() {
  vec4 localPos = vec4(a_position, 1.0);
  vec3 localNormal = a_normal;

  #ifdef HAS_SKINNING
    mat4 skinMat = getSkinMatrix();
    localPos = skinMat * localPos;
    localNormal = mat3(skinMat) * localNormal;
  #endif

  gl_Position = u_viewProjection * u_worldMatrix * localPos;
  v_normal = normalize(mat3(u_normalMatrix) * localNormal);
}
```

## Bone Attachment (Detailed Flow)

Attaching a static mesh to a bone (e.g., sword to hand):

```ts
// 1. Load character with skeleton
const character = await loadGLTF('character.glb', device)
const skeleton = character.skeletons[0]
scene.add(character.scenes[0])

// 2. Load weapon
const sword = await loadGLTF('sword.glb', device)
const swordMesh = sword.meshes[0]

// 3. Attach sword to hand bone
const handBone = skeleton.getBoneByName('hand_R')
addChild(handBone, swordMesh)

// Optional: offset the sword relative to the hand
setPosition(swordMesh, 0, 0, 0.1)          // Shift slightly forward
setRotation(swordMesh, quatFromEuler(0, 0, Math.PI / 4))  // Rotate 45 degrees

// 4. Each frame: animation updates bone transforms → scene graph propagates
//    to sword → sword renders at hand position. No manual sync needed.
```
