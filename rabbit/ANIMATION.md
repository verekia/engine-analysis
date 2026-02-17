# Animation

## Overview

Rabbit supports skeletal animation with GPU skinning, keyframe interpolation, and smooth crossfading between animations. The system is split into three layers:

1. **Data layer** — `AnimationClip` stores keyframe data
2. **Playback layer** — `AnimationAction` controls playback state
3. **Mixing layer** — `AnimationMixer` blends actions and drives the skeleton

## Skeleton

A `Skeleton` is a hierarchy of `Bone` nodes with pre-computed inverse bind matrices:

```typescript
class Skeleton {
  readonly bones: Bone[]                    // flat array, indexed by boneIndex
  readonly inverseBindMatrices: Mat4[]      // one per bone
  readonly boneMatrices: Float32Array       // flat mat4x3 array for GPU upload

  // Lookup
  getBoneByName(name: string): Bone | null
  getBoneByIndex(index: number): Bone

  // Update bone matrices for GPU skinning
  updateBoneMatrices(): void
}
```

### Bone Matrix Computation

Each frame, after animation updates bone transforms, the skeleton computes the final bone matrices for the vertex shader:

```typescript
const updateBoneMatrices = (skeleton: Skeleton) => {
  const { bones, inverseBindMatrices, boneMatrices } = skeleton

  for (let i = 0; i < bones.length; i++) {
    // boneMatrix = bone.worldMatrix * inverseBindMatrix
    // This transforms vertices from bind pose to current pose
    _tempMat4.multiplyMatrices(bones[i].worldMatrix, inverseBindMatrices[i])

    // Pack as mat4x3 (drop the last row [0,0,0,1]) to save uniform space
    // 12 floats per bone instead of 16
    const offset = i * 12
    const m = _tempMat4.elements
    boneMatrices[offset + 0] = m[0]
    boneMatrices[offset + 1] = m[1]
    boneMatrices[offset + 2] = m[2]
    boneMatrices[offset + 3] = m[4]
    boneMatrices[offset + 4] = m[5]
    boneMatrices[offset + 5] = m[6]
    boneMatrices[offset + 6] = m[8]
    boneMatrices[offset + 7] = m[9]
    boneMatrices[offset + 8] = m[10]
    boneMatrices[offset + 9] = m[12]
    boneMatrices[offset + 10] = m[13]
    boneMatrices[offset + 11] = m[14]
  }
}
```

Using `mat4x3` instead of `mat4` saves 25% uniform buffer space, allowing ~170 bones within a single 16KB uniform buffer (WebGL2 minimum guarantee) instead of ~128.

## AnimationClip

Immutable keyframe data parsed from glTF:

```typescript
class AnimationClip {
  readonly name: string
  readonly duration: number                 // total duration in seconds
  readonly tracks: AnimationTrack[]
}

interface AnimationTrack {
  readonly boneIndex: number                // which bone this track drives
  readonly property: 'position' | 'rotation' | 'scale'
  readonly interpolation: 'linear' | 'step' | 'cubicspline'
  readonly times: Float32Array              // sorted keyframe timestamps
  readonly values: Float32Array             // keyframe values
  // position/scale: 3 floats per keyframe
  // rotation: 4 floats per keyframe (quaternion)
  // cubicspline: 3x values per keyframe (in-tangent, value, out-tangent)
}
```

## AnimationAction

Mutable playback state for a single clip:

```typescript
class AnimationAction {
  clip: AnimationClip
  time: number                              // current playback time
  speed: number                             // playback speed (1 = normal)
  weight: number                            // blend weight (0..1)
  loop: LoopMode                            // once, repeat, pingpong
  playing: boolean
  paused: boolean

  // Internal state for crossfade
  _fadeIn: number                           // remaining fade-in duration
  _fadeOut: number                          // remaining fade-out duration
  _fadeInDuration: number
  _fadeOutDuration: number

  play(): AnimationAction
  stop(): AnimationAction
  pause(): AnimationAction
  reset(): AnimationAction
  setSpeed(speed: number): AnimationAction
  setWeight(weight: number): AnimationAction
  setLoop(mode: LoopMode): AnimationAction
}

type LoopMode = 'once' | 'repeat' | 'pingpong'
```

## AnimationMixer

Manages all actions for a single skeleton and blends them together:

```typescript
class AnimationMixer {
  readonly skeleton: Skeleton
  private actions: AnimationAction[]
  private _clipCache: Map<string, AnimationAction>

  // Create or retrieve an action for a clip
  clipAction(clip: AnimationClip): AnimationAction

  // Simple crossfade between two actions
  crossFade(
    fromAction: AnimationAction,
    toAction: AnimationAction,
    duration: number,
  ): void

  // Update all actions, blend, apply to skeleton
  update(deltaTime: number): void
}
```

### Crossfade Implementation

Crossfading smoothly transitions from one animation to another by ramping weights:

```typescript
const crossFade = (
  mixer: AnimationMixer,
  fromAction: AnimationAction,
  toAction: AnimationAction,
  duration: number,
) => {
  // Start the new animation from the beginning
  toAction.reset().play()
  toAction.weight = 0
  toAction._fadeIn = duration
  toAction._fadeInDuration = duration

  // Fade out the old animation
  fromAction._fadeOut = duration
  fromAction._fadeOutDuration = duration
}
```

During `mixer.update()`, fade weights are interpolated:

```typescript
// Inside mixer.update(dt):
for (const action of this.actions) {
  if (!action.playing) continue

  // Update fade-in
  if (action._fadeIn > 0) {
    action._fadeIn -= dt
    const t = 1 - Math.max(0, action._fadeIn) / action._fadeInDuration
    action.weight = t  // 0 → 1 over duration
  }

  // Update fade-out
  if (action._fadeOut > 0) {
    action._fadeOut -= dt
    const t = Math.max(0, action._fadeOut) / action._fadeOutDuration
    action.weight = t  // 1 → 0 over duration
    if (action._fadeOut <= 0) {
      action.stop()  // fully faded out
    }
  }

  // Advance time
  action.time += dt * action.speed
  if (action.loop === 'repeat') {
    action.time = action.time % action.clip.duration
  } else if (action.loop === 'pingpong') {
    // bounce back and forth
    const t = action.time % (action.clip.duration * 2)
    action.time = t > action.clip.duration
      ? action.clip.duration * 2 - t
      : t
  } else if (action.time >= action.clip.duration) {
    action.time = action.clip.duration
    action.playing = false
  }
}
```

### Blending

After updating all actions, the mixer blends their contributions to each bone:

```typescript
const blend = (mixer: AnimationMixer) => {
  const { skeleton, actions } = mixer

  // Reset all bones to identity/bind pose
  for (const bone of skeleton.bones) {
    bone.position.set(0, 0, 0)
    bone.rotation.identity()
    bone.scale.set(1, 1, 1)
  }

  // Normalize weights
  let totalWeight = 0
  for (const action of actions) {
    if (action.playing) totalWeight += action.weight
  }
  const weightScale = totalWeight > 0 ? 1 / totalWeight : 0

  // Accumulate weighted contributions
  for (const action of actions) {
    if (!action.playing || action.weight === 0) continue

    const normalizedWeight = action.weight * weightScale

    for (const track of action.clip.tracks) {
      const value = sampleTrack(track, action.time)
      const bone = skeleton.bones[track.boneIndex]

      switch (track.property) {
        case 'position':
          // Additive blend: bone.position += weight * value
          bone.position.addScaled(value as Vec3, normalizedWeight)
          break
        case 'rotation':
          // Spherical blend: slerp toward this rotation
          bone.rotation.slerp(value as Quat, normalizedWeight)
          break
        case 'scale':
          // Multiplicative blend via log-space interpolation
          // Simplified: lerp scale factors
          bone.scale.lerp(value as Vec3, normalizedWeight)
          break
      }
    }
  }
}
```

### Track Sampling

Keyframe interpolation with binary search for the active keyframe:

```typescript
const sampleTrack = (track: AnimationTrack, time: number): Vec3 | Quat => {
  const { times, values, interpolation, property } = track
  const stride = property === 'rotation' ? 4 : 3

  // Binary search for the keyframe pair bracketing `time`
  let lo = 0
  let hi = times.length - 1

  if (time <= times[0]) {
    return readValue(values, 0, stride)
  }
  if (time >= times[hi]) {
    return readValue(values, hi * stride, stride)
  }

  while (hi - lo > 1) {
    const mid = (lo + hi) >> 1
    if (times[mid] <= time) lo = mid
    else hi = mid
  }

  const t = (time - times[lo]) / (times[hi] - times[lo])

  if (interpolation === 'step') {
    return readValue(values, lo * stride, stride)
  }

  if (interpolation === 'linear') {
    if (property === 'rotation') {
      return _tempQuat.slerpQuaternions(
        readQuat(values, lo * 4),
        readQuat(values, hi * 4),
        t,
      )
    } else {
      return _tempVec3.lerpVectors(
        readVec3(values, lo * 3),
        readVec3(values, hi * 3),
        t,
      )
    }
  }

  // cubicspline interpolation
  // glTF cubicspline: each keyframe has [in-tangent, value, out-tangent]
  return cubicSplineInterpolate(values, lo, hi, t, stride)
}
```

## GPU Skinning Shader

Vertex shader skinning (GLSL 300 es):

```glsl
#ifdef HAS_SKINNING

in uvec4 a_joints;     // bone indices (4 per vertex)
in vec4 a_weights;      // bone weights (4 per vertex, sum to 1)

// Bone matrices packed as mat4x3 (3 vec4s per bone)
uniform BoneMatrices {
  vec4 u_bones[MAX_BONES * 3]; // MAX_BONES typically 128-170
};

mat4 getBoneMatrix(uint index) {
  uint base = index * 3u;
  return mat4(
    vec4(u_bones[base + 0u].xyz, 0.0),
    vec4(u_bones[base + 1u].xyz, 0.0),
    vec4(u_bones[base + 2u].xyz, 0.0),
    vec4(u_bones[base + 0u].w, u_bones[base + 1u].w, u_bones[base + 2u].w, 1.0)
  );
}

vec4 applySkinning(vec4 position) {
  mat4 skinMatrix =
    a_weights.x * getBoneMatrix(a_joints.x) +
    a_weights.y * getBoneMatrix(a_joints.y) +
    a_weights.z * getBoneMatrix(a_joints.z) +
    a_weights.w * getBoneMatrix(a_joints.w);

  return skinMatrix * position;
}

vec3 applySkinningNormal(vec3 normal) {
  mat4 skinMatrix =
    a_weights.x * getBoneMatrix(a_joints.x) +
    a_weights.y * getBoneMatrix(a_joints.y) +
    a_weights.z * getBoneMatrix(a_joints.z) +
    a_weights.w * getBoneMatrix(a_joints.w);

  return mat3(skinMatrix) * normal;
}

#endif

void main() {
  vec4 localPos = vec4(a_position, 1.0);
  vec3 localNormal = a_normal;

  #ifdef HAS_SKINNING
    localPos = applySkinning(localPos);
    localNormal = applySkinningNormal(localNormal);
  #endif

  gl_Position = u_viewProjection * u_model * localPos;
  v_normal = normalize(mat3(u_normalMatrix) * localNormal);
  // ...
}
```

## Typical Usage

```typescript
// Load a character with animations
const gltf = await gltfLoader.load('character.glb')
const character = gltf.scene
const clips = gltf.animations

scene.add(character)

// Set up animation mixer
const skinned = character.children.find(c => c instanceof SkinnedMesh) as SkinnedMesh
const mixer = new AnimationMixer(skinned.skeleton)

// Play idle animation
const idleAction = mixer.clipAction(clips.find(c => c.name === 'Idle')!)
idleAction.setLoop('repeat').play()

// Later: crossfade to run
const runAction = mixer.clipAction(clips.find(c => c.name === 'Run')!)
mixer.crossFade(idleAction, runAction, 0.3) // 300ms crossfade

// Attach sword to hand
const handBone = skinned.skeleton.getBoneByName('hand_r')!
handBone.add(swordMesh)

// Each frame
engine.onTick((dt) => {
  mixer.update(dt)
})
```
