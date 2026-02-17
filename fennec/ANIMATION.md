# Animation — Skeletal Animation, GPU Skinning, Crossfade

## Overview

Fennec supports skeletal animation with GPU-based vertex skinning and a CPU-side animation mixer that handles playback, blending, and simple crossfades between clips. The design prioritizes minimal overhead for common game patterns: idle/walk/run blending, attack animations, and death sequences.

## Data Model

### Skeleton

A skeleton is a hierarchy of bones with inverse bind matrices:

```typescript
interface Skeleton {
  readonly bones: Bone[]                    // Flat array of all bones
  readonly boneCount: number
  readonly inverseBindMatrices: Float32Array // boneCount * 16 floats (Mat4[])
  readonly boneWorldMatrices: Float32Array   // boneCount * 16 floats, updated each frame
  readonly boneTexture: DataTexture | null   // For GPU upload (WebGL2 fallback)
  readonly boneUBO: GPUUniformBufferHandle   // For WebGPU / WebGL2 UBO

  getBoneIndex(name: string): number
  getBone(name: string): Bone | undefined
}
```

### Bone

```typescript
interface Bone {
  readonly index: number
  readonly name: string
  readonly parentIndex: number       // -1 for root bones
  readonly childIndices: number[]

  // Local transform (animated)
  position: Vec3
  rotation: Quat
  scale: Vec3

  // Computed
  localMatrix: Mat4
  worldMatrix: Mat4
}
```

### AnimationClip

A clip contains keyframe tracks for one or more bones:

```typescript
interface AnimationClip {
  readonly name: string
  readonly duration: number          // Seconds
  readonly tracks: AnimationTrack[]
}

interface AnimationTrack {
  readonly boneIndex: number
  readonly property: 'position' | 'rotation' | 'scale'
  readonly times: Float32Array       // Keyframe timestamps
  readonly values: Float32Array      // Keyframe values (vec3 or quat)
  readonly interpolation: 'linear' | 'step' | 'cubicSpline'
}
```

### Keyframe Interpolation

```typescript
const sampleTrack = (track: AnimationTrack, time: number, out: Float32Array) => {
  const { times, values, interpolation, property } = track
  const stride = property === 'rotation' ? 4 : 3

  // Binary search for the two surrounding keyframes
  let lo = 0, hi = times.length - 1
  while (lo < hi - 1) {
    const mid = (lo + hi) >> 1
    if (times[mid] <= time) lo = mid
    else hi = mid
  }

  const t0 = times[lo]
  const t1 = times[hi]
  const alpha = t1 > t0 ? (time - t0) / (t1 - t0) : 0

  const offset0 = lo * stride
  const offset1 = hi * stride

  if (interpolation === 'step') {
    for (let i = 0; i < stride; i++) out[i] = values[offset0 + i]
  } else if (property === 'rotation') {
    // Spherical linear interpolation for quaternions
    quat.slerp(out, values.subarray(offset0, offset0 + 4), values.subarray(offset1, offset1 + 4), alpha)
  } else {
    // Linear interpolation for position/scale
    for (let i = 0; i < stride; i++) {
      out[i] = values[offset0 + i] + (values[offset1 + i] - values[offset0 + i]) * alpha
    }
  }
}
```

## Animation Mixer

The mixer manages playback of animation clips on a skeleton. It supports playing one or more clips simultaneously with weighted blending.

### AnimationAction

An action represents an active clip being played:

```typescript
interface AnimationAction {
  clip: AnimationClip
  time: number                // Current playback time
  timeScale: number           // Playback speed (1.0 = normal, -1.0 = reverse)
  weight: number              // Blend weight (0..1)
  loop: 'once' | 'repeat' | 'pingpong'
  paused: boolean
  _fadeTarget: number         // Target weight for crossfade
  _fadeDuration: number       // Remaining fade time
  _fadeElapsed: number        // Elapsed fade time
}
```

### AnimationMixer API

```typescript
interface AnimationMixer {
  // Skeleton this mixer controls
  readonly skeleton: Skeleton

  // Play a clip (returns the action for further control)
  play(clip: AnimationClip, options?: PlayOptions): AnimationAction

  // Stop a specific action
  stop(action: AnimationAction): void

  // Stop all actions
  stopAll(): void

  // Crossfade from current action to a new clip
  crossFadeTo(clip: AnimationClip, duration: number, options?: PlayOptions): AnimationAction

  // Update (called once per frame by the engine)
  update(deltaTime: number): void
}

interface PlayOptions {
  loop?: 'once' | 'repeat' | 'pingpong'  // Default: 'repeat'
  timeScale?: number                       // Default: 1.0
  weight?: number                          // Default: 1.0
  startTime?: number                       // Default: 0.0
}
```

### Crossfade Implementation

Crossfade linearly interpolates the weight of the old action from 1→0 and the new action from 0→1 over the specified duration:

```typescript
const crossFadeTo = (mixer: AnimationMixer, clip: AnimationClip, duration: number, options?: PlayOptions): AnimationAction => {
  // Fade out all current actions
  for (const action of mixer._actions) {
    if (!action.paused) {
      action._fadeTarget = 0
      action._fadeDuration = duration
      action._fadeElapsed = 0
    }
  }

  // Start new action, fade in
  const newAction = play(mixer, clip, { ...options, weight: 0 })
  newAction._fadeTarget = 1
  newAction._fadeDuration = duration
  newAction._fadeElapsed = 0

  return newAction
}
```

### Weight Update During Fade

```typescript
const updateFade = (action: AnimationAction, dt: number): boolean => {
  if (action._fadeDuration <= 0) return action.weight > 0

  action._fadeElapsed += dt
  const t = Math.min(action._fadeElapsed / action._fadeDuration, 1)
  action.weight = action.weight + (action._fadeTarget - action.weight) * t

  if (t >= 1) {
    action.weight = action._fadeTarget
    action._fadeDuration = 0
    // Remove actions that faded to 0
    return action.weight > 0
  }
  return true
}
```

### Mixer Update Loop

```typescript
const updateMixer = (mixer: AnimationMixer, dt: number) => {
  const skeleton = mixer.skeleton

  // 1. Reset all bone transforms to bind pose
  for (let i = 0; i < skeleton.boneCount; i++) {
    vec3.set(skeleton.bones[i].position, 0, 0, 0)
    quat.identity(skeleton.bones[i].rotation)
    vec3.set(skeleton.bones[i].scale, 1, 1, 1)
  }

  // 2. Advance time and sample each active action
  let totalWeight = 0
  for (const action of mixer._actions) {
    if (action.paused) continue

    // Update fade weight
    const alive = updateFade(action, dt)
    if (!alive) { mixer._removeAction(action); continue }
    if (action.weight < 0.001) continue

    // Advance time
    action.time += dt * action.timeScale
    if (action.loop === 'repeat') {
      action.time = action.time % action.clip.duration
    } else if (action.loop === 'pingpong') {
      const cycle = action.time / action.clip.duration
      const inCycle = cycle % 2
      action.time = inCycle > 1
        ? (2 - inCycle) * action.clip.duration
        : inCycle * action.clip.duration
    } else if (action.time > action.clip.duration) {
      action.time = action.clip.duration
      action.paused = true
    }

    // Sample tracks and blend
    const w = action.weight
    totalWeight += w

    for (const track of action.clip.tracks) {
      const bone = skeleton.bones[track.boneIndex]
      sampleTrack(track, action.time, _tempSample)

      if (track.property === 'position') {
        // Additive blend: weighted sum
        bone.position[0] += _tempSample[0] * w
        bone.position[1] += _tempSample[1] * w
        bone.position[2] += _tempSample[2] * w
      } else if (track.property === 'rotation') {
        // Spherical blend
        quat.slerp(bone.rotation, bone.rotation, _tempSample as Quat, w / totalWeight)
      } else if (track.property === 'scale') {
        // Multiplicative blend
        bone.scale[0] *= Math.pow(_tempSample[0], w)
        bone.scale[1] *= Math.pow(_tempSample[1], w)
        bone.scale[2] *= Math.pow(_tempSample[2], w)
      }
    }
  }

  // 3. Compute bone world matrices
  computeBoneWorldMatrices(skeleton)

  // 4. Compute final skinning matrices and upload to GPU
  computeSkinningMatrices(skeleton)
  uploadBoneData(skeleton)
}
```

## GPU Skinning

### Bone Matrix Storage

Bone matrices are uploaded to the GPU as either:

**WebGPU**: A storage buffer (SSBO) in bind group slot 3:
```wgsl
@group(3) @binding(1) var<storage, read> boneMatrices: array<mat4x4f>;
```

**WebGL2**: A float texture (data texture). Each bone matrix = 4 pixels in an RGBA32F texture:
```glsl
uniform highp sampler2D boneTexture;
uniform int boneTextureSize;

mat4 getBoneMatrix(int boneIndex) {
  int pixel = boneIndex * 4;
  int row = pixel / boneTextureSize;
  int col = pixel - row * boneTextureSize;
  float dx = 1.0 / float(boneTextureSize);
  float y = (float(row) + 0.5) * dx;
  vec4 c0 = texture(boneTexture, vec2((float(col) + 0.5) * dx, y));
  vec4 c1 = texture(boneTexture, vec2((float(col) + 1.5) * dx, y));
  vec4 c2 = texture(boneTexture, vec2((float(col) + 2.5) * dx, y));
  vec4 c3 = texture(boneTexture, vec2((float(col) + 3.5) * dx, y));
  return mat4(c0, c1, c2, c3);
}
```

### Skinning Matrices

The skinning matrix for each bone is: `boneWorldMatrix * inverseBindMatrix`

```typescript
const computeSkinningMatrices = (skeleton: Skeleton) => {
  const { boneWorldMatrices, inverseBindMatrices, boneCount } = skeleton
  const out = skeleton._skinningMatrices  // Pre-allocated Float32Array(boneCount * 16)

  for (let i = 0; i < boneCount; i++) {
    const offset = i * 16
    mat4.multiply(
      out.subarray(offset, offset + 16),
      boneWorldMatrices.subarray(offset, offset + 16),
      inverseBindMatrices.subarray(offset, offset + 16),
    )
  }
}
```

### Vertex Shader Skinning

```glsl
// Vertex attributes
layout(location = 4) in vec4 a_joints;   // Bone indices (up to 4)
layout(location = 5) in vec4 a_weights;  // Bone weights (sum to 1)

vec4 applySkinning(vec4 position, vec3 normal) {
  mat4 skinMatrix =
    a_weights.x * getBoneMatrix(int(a_joints.x)) +
    a_weights.y * getBoneMatrix(int(a_joints.y)) +
    a_weights.z * getBoneMatrix(int(a_joints.z)) +
    a_weights.w * getBoneMatrix(int(a_joints.w));

  v_normal = normalize((skinMatrix * vec4(normal, 0.0)).xyz);
  return skinMatrix * position;
}
```

### Performance Characteristics

| Metric | Value |
|--------|-------|
| Max bones per skeleton | 128 (configurable, 256 with storage buffer) |
| Skinning cost (100-bone character) | ~0.1ms GPU |
| Bone texture size | 512x1 (supports up to 128 bones) |
| Animation sampling (CPU) | ~0.02ms per clip per skeleton |
| Crossfade overhead | Negligible (just weights two samples) |

## Usage Examples

### Basic Playback

```typescript
const { scene, animations, skeletons } = await loadGLTF('/character.glb', { draco: true })
const mixer = new AnimationMixer(skeletons[0])

// Play idle animation, looping
const idleAction = mixer.play(animations.find(a => a.name === 'idle')!)

// Later: crossfade to run
const runAction = mixer.crossFadeTo(
  animations.find(a => a.name === 'run')!,
  0.3, // 300ms crossfade
)

// Per frame (handled automatically by engine if using React bindings)
mixer.update(deltaTime)
```

### Bone Attachment with Animation

```typescript
const { skeletons } = await loadGLTF('/character.glb')
const sword = await loadGLTF('/sword.glb')

const handBone = skeletons[0].getBoneIndex('hand_R')
const attachment = new BoneAttachment(skeletons[0], handBone)
attachment.add(sword.scene)
scene.add(attachment)

// Sword automatically follows animated hand bone every frame
```
