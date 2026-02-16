# Skeletal Animation

## Overview

Kestrel supports skeletal animation with GPU skinning, per-bone TRS channels, and simple crossfade blending between animation states. The system is designed for game characters: walk, run, idle, attack — with smooth transitions.

## Skeleton

A `Skeleton` is a hierarchy of bones (joints). Each bone has a local transform (relative to its parent bone) and a bind-pose inverse matrix.

```typescript
interface Skeleton {
  readonly bones: Bone[]
  readonly boneCount: number

  // Flat array of joint matrices, ready for GPU upload
  // jointMatrices[i] = bone[i].worldMatrix × bone[i].inverseBindMatrix
  readonly jointMatrices: Float32Array  // boneCount × 16 floats

  // Update joint matrices from current bone transforms
  update(): void
}

interface Bone {
  readonly index: number
  readonly name: string
  parent: Bone | null
  children: Bone[]

  // Local transform (relative to parent bone)
  position: Vec3
  rotation: Quat
  scale: Vec3

  // Computed
  localMatrix: Mat4
  worldMatrix: Mat4

  // From GLTF — the inverse of the bind-pose world matrix
  readonly inverseBindMatrix: Mat4
}
```

### Joint Matrix Computation

Each frame, after animation has updated bone local transforms:

```
skeleton.update():
  for each bone in topological order (parents before children):
    bone.localMatrix = compose(bone.position, bone.rotation, bone.scale)
    if bone.parent:
      bone.worldMatrix = bone.parent.worldMatrix × bone.localMatrix
    else:
      bone.worldMatrix = bone.localMatrix
    jointMatrices[bone.index] = bone.worldMatrix × bone.inverseBindMatrix
```

The `jointMatrices` array is uploaded as a uniform buffer (for small skeletons ≤64 bones) or as a data texture (for larger skeletons).

## GPU Skinning

The vertex shader applies skinning using 4 bone influences per vertex:

```wgsl
// Vertex attributes
@location(6) jointIndices: vec4<u32>    // 4 bone indices
@location(7) jointWeights: vec4<f32>    // 4 blend weights

// Uniform
@group(2) @binding(1) var<uniform> joints: array<mat4x4<f32>, 64>;

fn applySkinning(pos: vec3<f32>, norm: vec3<f32>) -> SkinResult {
  let w = jointWeights;
  let skinMatrix =
    joints[jointIndices.x] * w.x +
    joints[jointIndices.y] * w.y +
    joints[jointIndices.z] * w.z +
    joints[jointIndices.w] * w.w;

  return SkinResult(
    (skinMatrix * vec4(pos, 1.0)).xyz,
    normalize((skinMatrix * vec4(norm, 0.0)).xyz)
  );
}
```

On WebGPU, for skeletons with >64 bones, a storage buffer or data texture is used instead of a uniform array.

## Animation Clips

An `AnimationClip` contains channels that target individual bone transforms:

```typescript
interface AnimationClip {
  readonly name: string
  readonly duration: number         // seconds
  readonly channels: AnimationChannel[]
}

interface AnimationChannel {
  readonly targetBoneIndex: number
  readonly property: 'position' | 'rotation' | 'scale'
  readonly times: Float32Array      // keyframe timestamps
  readonly values: Float32Array     // keyframe values (3 floats for pos/scale, 4 for rotation)
  readonly interpolation: 'linear' | 'step'
}
```

### Keyframe Sampling

For a given time `t`, each channel finds the surrounding keyframes and interpolates:

```
sample(channel, t):
  // Binary search for the keyframe pair
  let [i, j] = findKeyframePair(channel.times, t)
  let alpha = (t - channel.times[i]) / (channel.times[j] - channel.times[i])

  if channel.interpolation === 'step':
    return channel.values[i]

  if channel.property === 'rotation':
    return slerp(channel.values[i], channel.values[j], alpha)
  else:
    return lerp(channel.values[i], channel.values[j], alpha)
```

Binary search result is cached per channel (since time typically advances forward, the cache hit rate is near 100%).

## Animation Mixer

The `AnimationMixer` manages animation playback on a skeleton:

```typescript
interface AnimationMixer {
  readonly skeleton: Skeleton

  // Play a clip, optionally looping
  play(clip: AnimationClip, options?: { loop?: boolean, speed?: number }): AnimationAction

  // Crossfade from current animation to a new one
  crossFadeTo(clip: AnimationClip, duration: number, options?: { loop?: boolean }): AnimationAction

  // Stop all animations
  stopAll(): void

  // Called each frame
  update(deltaTime: number): void
}

interface AnimationAction {
  readonly clip: AnimationClip
  time: number
  speed: number
  loop: boolean
  weight: number          // 0–1, used for blending
  playing: boolean

  play(): void
  stop(): void
  fadeIn(duration: number): void
  fadeOut(duration: number): void
}
```

### Crossfade Implementation

A crossfade runs two animations simultaneously with complementary weights:

```
crossFadeTo(newClip, duration):
  // Create new action
  newAction = createAction(newClip)
  newAction.weight = 0
  newAction.play()

  // Fade out current, fade in new
  currentAction.fadeOut(duration)
  newAction.fadeIn(duration)

update(dt):
  // Update all active actions
  for each action in activeActions:
    action.time += dt * action.speed
    if action.loop: action.time %= action.clip.duration

    // Update fade weight
    if action.fading:
      action.weight += (action.fadeTarget - action.weight) * (dt / action.fadeDuration)
      if close enough to target: action.weight = action.fadeTarget; action.fading = false
      if action.weight <= 0: deactivate(action)

  // Apply weighted blend to skeleton
  resetBones(skeleton)  // identity transforms
  for each action in activeActions:
    for each channel in action.clip.channels:
      value = sample(channel, action.time)
      applyWeighted(skeleton.bones[channel.targetBoneIndex], channel.property, value, action.weight)
```

### Weighted Blending

For position and scale, weighted blending is simple lerp:

```
bone.position += value * weight    // accumulate
```

For rotation, blending uses `slerp` toward each animation's rotation, weighted:

```
bone.rotation = slerp(bone.rotation, value, weight)
```

When only one animation is active (weight = 1.0), this collapses to a direct assignment — no blending overhead.

## Bone Attachment

Attaching a static mesh (like a sword) to a bone:

```typescript
// Create attachment
const sword = new Mesh(swordGeometry, swordMaterial)

// Attach to the "hand_R" bone of a skinned mesh
skinnedMesh.attachToBone('hand_R', sword)
```

Implementation:

```
attachToBone(boneName, mesh):
  let bone = skeleton.getBoneByName(boneName)
  // The mesh's world matrix is overridden each frame:
  // mesh.worldMatrix = bone.worldMatrix × mesh.localMatrix
  bone.attachments.push(mesh)
  scene.add(mesh)  // mesh is in the scene graph, but its transform is driven by the bone

// In the animation update:
skeleton.update():
  ... compute joint matrices ...
  // Update attachments
  for each bone:
    for each attachment in bone.attachments:
      attachment.worldMatrix = bone.worldMatrix × attachment.localMatrix
      attachment._dirtyFlag = false  // prevent scene graph from overwriting
```

The attachment's `localMatrix` acts as an offset from the bone (allowing adjustment of the sword's position/rotation relative to the hand).

## SkinnedMesh

`SkinnedMesh` extends `Mesh` with a skeleton reference:

```typescript
interface SkinnedMesh extends Mesh {
  readonly skeleton: Skeleton
  readonly mixer: AnimationMixer

  // Bind the skeleton to this mesh
  bind(skeleton: Skeleton): void
}
```

When the renderer encounters a `SkinnedMesh`, it:
1. Calls `mixer.update(dt)` (if not already updated this frame)
2. Calls `skeleton.update()` to recompute joint matrices
3. Uploads `skeleton.jointMatrices` to the GPU
4. Draws with the skinning shader variant

## Performance Considerations

- **Keyframe cache**: Each channel caches the last-used keyframe index. Sequential playback almost never binary-searches.
- **Bone limit**: Up to 64 bones fit in a uniform buffer (64 × 64 bytes = 4KB). Beyond that, a texture is used.
- **Shared skeletons**: Multiple `SkinnedMesh` instances can share the same `Skeleton` definition but have independent `AnimationMixer` state (each gets its own `jointMatrices` copy).
- **No allocation per frame**: All scratch vectors/quats come from the math pool.
