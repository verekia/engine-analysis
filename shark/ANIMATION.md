# Animation Architecture

## Overview

Shark supports skeletal animation with GPU skinning, simple cross-fade blending between animations, and bone attachment for parenting static meshes to animated bones.

## Skeleton

### Joint Hierarchy

A skeleton is a tree of joints (bones). Each joint has:

```typescript
interface Joint {
  index: number               // Index in the skeleton's joint array
  name: string                // e.g., 'spine', 'hand_left', 'head'
  parentIndex: number         // -1 for root joint
  localPosition: Vec3         // Rest pose local position
  localRotation: Quat         // Rest pose local rotation
  localScale: Vec3            // Rest pose local scale
}

interface Skeleton {
  joints: Joint[]                     // Flat array of all joints
  inverseBindMatrices: Float32Array   // 16 floats per joint (mat4)
  boneMatrices: Float32Array          // 16 floats per joint, updated each frame
  boneTexture: GpuTexture             // RGBA32F texture for GPU skinning
  jointCount: number
  jointNameMap: Map<string, number>   // Name → index lookup
}
```

### Bone Matrix Computation

Each frame, after animation sampling, the final bone matrices are computed:

```
For each joint (in hierarchy order, parents before children):
  1. Get animated local transform (position, rotation, scale)
  2. Compute local matrix: T * R * S
  3. Compute world matrix: parent.worldMatrix * localMatrix
  4. Compute bone matrix: worldMatrix * inverseBindMatrix
  5. Write bone matrix to boneMatrices array
```

```typescript
const updateBoneMatrices = (skeleton: Skeleton, animatedPose: AnimatedPose) => {
  const { joints, inverseBindMatrices, boneMatrices } = skeleton
  const tempLocal = mat4Create()
  const tempWorld = mat4Create()

  for (let i = 0; i < joints.length; i++) {
    const joint = joints[i]
    const pose = animatedPose.joints[i]

    // Build local matrix from animated transform
    mat4FromTRS(tempLocal, pose.position, pose.rotation, pose.scale)

    // Compute world matrix
    if (joint.parentIndex >= 0) {
      // Parent's world matrix is already computed (processing in hierarchy order)
      const parentWorld = new Float32Array(boneMatrices.buffer, joint.parentIndex * 64 + /* worldOffset */, 16)
      mat4Multiply(tempWorld, parentWorld, tempLocal)
    } else {
      mat4Copy(tempWorld, tempLocal)
    }

    // Store world matrix (needed for bone attachments)
    // worldMatrices[i] = tempWorld

    // bone matrix = world matrix * inverse bind matrix
    const ibm = new Float32Array(inverseBindMatrices.buffer, i * 64, 16)
    const bone = new Float32Array(boneMatrices.buffer, i * 64, 16)
    mat4Multiply(bone, tempWorld, ibm)
  }
}
```

### Bone Texture Upload

Bone matrices are uploaded to the GPU as a floating-point texture:

```typescript
// Texture size: ceil(sqrt(jointCount * 4)) × same (each matrix = 4 texels of RGBA32F)
// For 64 joints: 64 * 4 = 256 texels → 16×16 RGBA32F texture

const updateBoneTexture = (skeleton: Skeleton, device: GpuDevice) => {
  device.writeTexture(skeleton.boneTexture, skeleton.boneMatrices)
}
```

**Why texture instead of UBO?**
- UBO size limits vary (often 16KB on mobile → ~256 mat4s → 64 joints max with other uniforms)
- Texture has no such limit
- Texture fetch is efficient on modern GPUs
- WebGPU alternative: storage buffer (even better, but textures work on both backends)

### Vertex Shader Skinning

```glsl
// 4 bone influences per vertex
in uvec4 a_joints;   // Bone indices
in vec4 a_weights;   // Bone weights (sum to 1.0)

// Fetch bone matrix from texture
mat4 getBoneMatrix(uint index) {
  // Each matrix = 4 consecutive texels in the bone texture
  uint x = (index * 4u) % boneTextureWidth;
  uint y = (index * 4u) / boneTextureWidth;
  vec4 row0 = texelFetch(boneTexture, ivec2(x, y), 0);
  vec4 row1 = texelFetch(boneTexture, ivec2(x + 1u, y), 0);
  vec4 row2 = texelFetch(boneTexture, ivec2(x + 2u, y), 0);
  vec4 row3 = texelFetch(boneTexture, ivec2(x + 3u, y), 0);
  return mat4(row0, row1, row2, row3);
}

void main() {
  mat4 skinMatrix =
    a_weights.x * getBoneMatrix(a_joints.x) +
    a_weights.y * getBoneMatrix(a_joints.y) +
    a_weights.z * getBoneMatrix(a_joints.z) +
    a_weights.w * getBoneMatrix(a_joints.w);

  vec4 skinnedPosition = skinMatrix * vec4(a_position, 1.0);
  vec3 skinnedNormal = mat3(skinMatrix) * a_normal;

  gl_Position = viewProjection * model * skinnedPosition;
  // ...
}
```

## Animation Clips

### Data Structure

```typescript
interface AnimationClip {
  name: string                    // e.g., 'idle', 'walk', 'attack'
  duration: number                // Seconds
  channels: AnimationChannel[]    // One per animated joint per property
}

interface AnimationChannel {
  jointIndex: number              // Which joint this channel animates
  property: 'position' | 'rotation' | 'scale'
  timestamps: Float32Array        // Keyframe times in seconds
  values: Float32Array            // Keyframe values (vec3 or quat)
  interpolation: 'linear' | 'step'
}
```

### Keyframe Sampling

```typescript
const sampleChannel = (channel: AnimationChannel, time: number, out: Float32Array) => {
  const { timestamps, values, interpolation, property } = channel
  const count = timestamps.length

  // Clamp or wrap time
  if (time <= timestamps[0]) {
    copyValues(values, 0, out, property)
    return
  }
  if (time >= timestamps[count - 1]) {
    copyValues(values, count - 1, out, property)
    return
  }

  // Binary search for surrounding keyframes
  let lo = 0, hi = count - 1
  while (lo < hi - 1) {
    const mid = (lo + hi) >> 1
    if (timestamps[mid] <= time) lo = mid
    else hi = mid
  }

  if (interpolation === 'step') {
    copyValues(values, lo, out, property)
    return
  }

  // Linear interpolation
  const t = (time - timestamps[lo]) / (timestamps[hi] - timestamps[lo])

  if (property === 'rotation') {
    // Slerp for quaternions
    quatSlerp(out, getQuat(values, lo), getQuat(values, hi), t)
  } else {
    // Lerp for position and scale
    vec3Lerp(out, getVec3(values, lo), getVec3(values, hi), t)
  }
}
```

## Animation Mixer

The `AnimationMixer` manages animation playback for a single `SkinnedMesh`.

### State

```typescript
interface AnimationMixer {
  mesh: SkinnedMesh
  currentAction: AnimationAction | null
  fadingAction: AnimationAction | null   // Previous action being faded out
  fadeProgress: number                    // 0 = fully fadingAction, 1 = fully currentAction
  fadeDuration: number                   // Cross-fade duration in seconds
}

interface AnimationAction {
  clip: AnimationClip
  time: number                    // Current playback time
  speed: number                   // Playback speed multiplier (default: 1)
  loop: boolean                   // Whether to loop (default: true)
  playing: boolean
}
```

### Playback

```typescript
const play = (mixer: AnimationMixer, clipName: string, fadeDuration = 0.3) => {
  const clip = findClip(mixer.mesh, clipName)

  if (mixer.currentAction && fadeDuration > 0) {
    // Start cross-fade: current → fading, new → current
    mixer.fadingAction = mixer.currentAction
    mixer.fadeProgress = 0
    mixer.fadeDuration = fadeDuration
  }

  mixer.currentAction = {
    clip,
    time: 0,
    speed: 1,
    loop: true,
    playing: true,
  }
}
```

### Per-Frame Update

```typescript
const updateMixer = (mixer: AnimationMixer, deltaTime: number) => {
  const { currentAction, fadingAction } = mixer
  if (!currentAction?.playing) return

  // Advance time
  currentAction.time += deltaTime * currentAction.speed
  if (currentAction.loop) {
    currentAction.time %= currentAction.clip.duration
  } else {
    currentAction.time = Math.min(currentAction.time, currentAction.clip.duration)
  }

  // Sample current animation
  const currentPose = sampleClip(currentAction.clip, currentAction.time)

  if (fadingAction && mixer.fadeProgress < 1) {
    // Advance fading animation
    fadingAction.time += deltaTime * fadingAction.speed
    if (fadingAction.loop) fadingAction.time %= fadingAction.clip.duration

    // Sample fading animation
    const fadingPose = sampleClip(fadingAction.clip, fadingAction.time)

    // Cross-fade
    mixer.fadeProgress = Math.min(1, mixer.fadeProgress + deltaTime / mixer.fadeDuration)
    blendPoses(fadingPose, currentPose, mixer.fadeProgress, mixer.mesh.skeleton)

    if (mixer.fadeProgress >= 1) {
      mixer.fadingAction = null  // Fade complete
    }
  } else {
    applyPose(currentPose, mixer.mesh.skeleton)
  }

  // Update bone matrices and upload to GPU
  updateBoneMatrices(mixer.mesh.skeleton, /* result pose */)
  updateBoneTexture(mixer.mesh.skeleton, mixer.mesh._device)
}
```

### Cross-Fade Blending

```typescript
const blendPoses = (
  poseA: AnimatedPose,   // Fading out
  poseB: AnimatedPose,   // Fading in
  t: number,             // 0 = A, 1 = B
  skeleton: Skeleton,
) => {
  for (let i = 0; i < skeleton.jointCount; i++) {
    vec3Lerp(result.joints[i].position, poseA.joints[i].position, poseB.joints[i].position, t)
    quatSlerp(result.joints[i].rotation, poseA.joints[i].rotation, poseB.joints[i].rotation, t)
    vec3Lerp(result.joints[i].scale, poseA.joints[i].scale, poseB.joints[i].scale, t)
  }
}
```

## Bone Attachment

### Use Case

Attaching a static mesh (sword, shield, hat) to an animated bone so it follows the animation.

### Implementation

```typescript
interface BoneAttachment extends Node3D {
  sourceMesh: SkinnedMesh         // The animated character
  boneName: string                // e.g., 'hand_right'
  boneIndex: number               // Resolved from boneName
}
```

`BoneAttachment` overrides the world matrix computation:

```typescript
const updateBoneAttachment = (attachment: BoneAttachment) => {
  const skeleton = attachment.sourceMesh.skeleton
  const boneWorldMatrix = skeleton.getJointWorldMatrix(attachment.boneIndex)

  // The attachment's world matrix = bone's world matrix * local offset
  mat4Multiply(
    attachment._worldMatrix,
    boneWorldMatrix,
    attachment._localMatrix,  // Optional offset (e.g., grip position)
  )

  // Mark children dirty so they update too
  for (const child of attachment.children) {
    markDirty(child)
  }
}
```

### Usage

```typescript
// Load character and sword
const character = await gltfLoader.load('character.glb')
const sword = await gltfLoader.load('sword.glb')

// Create attachment
const swordAttachment = new BoneAttachment(character.skinnedMeshes[0], 'hand_right')
swordAttachment.add(sword.scene)

// Optional offset (adjust grip)
swordAttachment.position.set(0, 0, 0.1)
swordAttachment.rotation.setFromEuler(0, 0, Math.PI / 4)

scene.add(swordAttachment)
```

## Performance Considerations

1. **Bone matrix computation**: O(joints) per frame per animated mesh. For 64 joints, this is ~64 matrix multiplications ≈ 0.02ms.

2. **Bone texture upload**: One `writeTexture` call per animated mesh per frame. For 64 joints: 64 × 64 bytes = 4KB upload. Negligible.

3. **Keyframe binary search**: O(log n) per channel per frame. Cached last-frame index for O(1) amortized (keyframes are usually sampled sequentially).

4. **Cross-fade**: 2× sampling cost during the fade period (two clips sampled simultaneously). Typically 0.1-0.5s duration, so the overhead is brief.

5. **Skinning in vertex shader**: 4 bone influences per vertex. 4 matrix fetches from bone texture + 4 matrix-vector multiplications. Modern mobile GPUs handle this with ease.
