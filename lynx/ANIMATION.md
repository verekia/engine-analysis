# Lynx Engine — Animation System

The animation system drives skeletal animation for characters and articulated objects. It handles keyframe interpolation, multi-animation blending, crossfade transitions, GPU skinning, and bone attachment. The system is built on the scene graph -- bones are regular nodes, and skinning is a vertex shader transformation that maps rest-pose vertices onto an animated skeleton.

Lynx uses a **Z-up, right-handed coordinate system**. All animation data imported from GLTF (Y-up) is converted to Z-up at load time.

---

## 1. Core Types

### Type Hierarchy

```
AnimationMixer
├── AnimationAction (one per playing clip)
│   └── AnimationClip
│       └── KeyframeTrack[] (one per bone property)
│
Skeleton
├── Bone[] (ordered array, each Bone extends Node)
├── inverseBindMatrices: Mat4[]
└── boneMatrices: Float32Array (flat, uploaded to GPU)
```

### Interface Definitions

```typescript
interface Bone extends NodeBase {
  readonly type: "Bone"
  readonly index: number  // index into parent skeleton's bones array
}

interface Skeleton {
  readonly bones: readonly Bone[]
  readonly inverseBindMatrices: readonly Mat4[]  // one per bone, rest-pose inverse
  readonly boneMatrices: Float32Array             // flat array: bones.length * 16 floats
  readonly boneCount: number
  getBone: (name: string) => Bone | null
}

interface KeyframeTrack {
  readonly boneIndex: number
  readonly property: "position" | "rotation" | "scale"
  readonly times: Float32Array     // sorted ascending, in seconds
  readonly values: Float32Array    // interleaved values: 3 floats per key (pos/scale) or 4 (rotation quat)
}

interface AnimationClip {
  readonly name: string
  readonly duration: number        // seconds
  readonly tracks: readonly KeyframeTrack[]
}

type LoopMode = "once" | "repeat" | "pingpong"

interface AnimationAction {
  readonly clip: AnimationClip
  time: number                     // current playback time in seconds
  speed: number                    // playback rate multiplier (1.0 = normal, negative = reverse)
  weight: number                   // blend weight 0-1
  loop: LoopMode
  playing: boolean
  paused: boolean

  play: () => AnimationAction
  stop: () => AnimationAction
  pause: () => AnimationAction
  reset: () => AnimationAction
  setWeight: (weight: number) => AnimationAction
  setSpeed: (speed: number) => AnimationAction
  setLoop: (mode: LoopMode) => AnimationAction
}

interface CrossFadeState {
  fromAction: AnimationAction
  toAction: AnimationAction
  duration: number
  elapsed: number
}

interface AnimationMixer {
  readonly skeleton: Skeleton
  readonly actions: readonly AnimationAction[]

  clipAction: (clip: AnimationClip) => AnimationAction
  play: (clip: AnimationClip) => AnimationAction
  stopAll: () => void
  crossFade: (fromAction: AnimationAction, toAction: AnimationAction, duration: number) => void
  update: (deltaTime: number) => void
}
```

### Key Design Points

- **`Bone`** extends `NodeBase`. It is a regular scene graph node with an additional `index` field. Because bones are nodes, any object can be parented to a bone with `addChild` -- no special attachment API needed.
- **`Skeleton`** is a flat, ordered array of `Bone` nodes. The order matters: each bone's `index` corresponds to the joint indices stored in skinned mesh vertex data. The skeleton also owns the `inverseBindMatrices` (the inverse of each bone's world matrix in the rest pose) and a flat `Float32Array` of computed bone matrices ready for GPU upload.
- **`KeyframeTrack`** stores keyframes for a single property of a single bone. Times and values are flat typed arrays for cache-friendly access. Position and scale tracks store 3 floats per keyframe. Rotation tracks store 4 floats per keyframe (quaternion XYZW).
- **`AnimationClip`** is a named collection of tracks. A "walk" clip might have 60+ tracks (position, rotation, scale for 20+ bones). Clips are immutable data -- they are shared across all instances that play the same animation.
- **`AnimationAction`** is a playing instance of a clip. It carries the mutable playback state: current time, speed, weight, loop mode. Multiple actions can reference the same clip (e.g., two characters both playing "walk").
- **`AnimationMixer`** manages all active actions for one skeleton. It advances time, evaluates keyframes, blends multiple active actions by weight, and writes the final bone transforms into the skeleton.

---

## 2. Keyframe Interpolation

### Finding the Surrounding Keyframes

Each `KeyframeTrack` has a sorted `times` array. To evaluate at a given time `t`, find the two surrounding keyframes using binary search. The result is the indices `i` and `i+1` and an interpolation factor `alpha` in [0, 1].

```typescript
const findKeyframeIndex = (times: Float32Array, t: number): [number, number, number] => {
  const n = times.length

  // Clamp to range
  if (t <= times[0]) return [0, 0, 0.0]
  if (t >= times[n - 1]) return [n - 1, n - 1, 0.0]

  // Binary search for the interval containing t
  let lo = 0
  let hi = n - 1

  while (hi - lo > 1) {
    const mid = (lo + hi) >>> 1
    if (times[mid] <= t) {
      lo = mid
    } else {
      hi = mid
    }
  }

  const t0 = times[lo]
  const t1 = times[hi]
  const alpha = (t - t0) / (t1 - t0)

  return [lo, hi, alpha]
}
```

### Position and Scale: Linear Interpolation

Position and scale keyframes are interpolated linearly. Each keyframe stores 3 floats (x, y, z).

```typescript
const interpolateVec3 = (
  values: Float32Array,
  i0: number,
  i1: number,
  alpha: number,
  out: Vec3
): Vec3 => {
  const base0 = i0 * 3
  const base1 = i1 * 3

  out[0] = values[base0]     + (values[base1]     - values[base0])     * alpha
  out[1] = values[base0 + 1] + (values[base1 + 1] - values[base0 + 1]) * alpha
  out[2] = values[base0 + 2] + (values[base1 + 2] - values[base0 + 2]) * alpha

  return out
}
```

### Rotation: Spherical Linear Interpolation (Slerp)

Rotation keyframes store unit quaternions (4 floats: x, y, z, w). Quaternions must be interpolated with slerp to preserve unit length and produce smooth rotational motion. Linear interpolation of quaternions would produce varying angular velocity and non-unit results.

```typescript
const interpolateQuat = (
  values: Float32Array,
  i0: number,
  i1: number,
  alpha: number,
  out: Quat
): Quat => {
  const base0 = i0 * 4
  const base1 = i1 * 4

  let ax = values[base0]
  let ay = values[base0 + 1]
  let az = values[base0 + 2]
  let aw = values[base0 + 3]

  let bx = values[base1]
  let by = values[base1 + 1]
  let bz = values[base1 + 2]
  let bw = values[base1 + 3]

  // Ensure shortest path: if dot product is negative, negate one quaternion
  let dot = ax * bx + ay * by + az * bz + aw * bw
  if (dot < 0.0) {
    bx = -bx
    by = -by
    bz = -bz
    bw = -bw
    dot = -dot
  }

  // If quaternions are very close, fall back to normalized lerp (nlerp)
  if (dot > 0.9995) {
    out[0] = ax + (bx - ax) * alpha
    out[1] = ay + (by - ay) * alpha
    out[2] = az + (bz - az) * alpha
    out[3] = aw + (bw - aw) * alpha

    // Normalize
    const len = Math.sqrt(out[0] * out[0] + out[1] * out[1] + out[2] * out[2] + out[3] * out[3])
    const invLen = 1.0 / len
    out[0] *= invLen
    out[1] *= invLen
    out[2] *= invLen
    out[3] *= invLen

    return out
  }

  // Standard slerp
  const theta = Math.acos(dot)
  const sinTheta = Math.sin(theta)
  const w0 = Math.sin((1.0 - alpha) * theta) / sinTheta
  const w1 = Math.sin(alpha * theta) / sinTheta

  out[0] = ax * w0 + bx * w1
  out[1] = ay * w0 + by * w1
  out[2] = az * w0 + bz * w1
  out[3] = aw * w0 + bw * w1

  return out
}
```

### Evaluating a Track

```typescript
const evaluateTrack = (
  track: KeyframeTrack,
  time: number,
  outPosition: Vec3,
  outRotation: Quat,
  outScale: Vec3
): void => {
  const [i0, i1, alpha] = findKeyframeIndex(track.times, time)

  switch (track.property) {
    case "position":
      interpolateVec3(track.values, i0, i1, alpha, outPosition)
      break
    case "rotation":
      interpolateQuat(track.values, i0, i1, alpha, outRotation)
      break
    case "scale":
      interpolateVec3(track.values, i0, i1, alpha, outScale)
      break
  }
}
```

---

## 3. Animation Blending

When multiple animations play simultaneously on the same skeleton, their bone transforms must be blended together. Each `AnimationAction` has a weight between 0 and 1. The final transform for each bone is the weighted average of all active actions' transforms for that bone.

### Blend Formula

For position and scale (vector quantities):

```
finalPosition = sum(action_i.position * action_i.weight) / sum(action_i.weight)
finalScale    = sum(action_i.scale    * action_i.weight) / sum(action_i.weight)
```

For rotation (quaternion quantities), a weighted blend with normalization:

```
finalRotation = normalize(sum(action_i.rotation * action_i.weight))
```

The quaternion sum-and-normalize approach (sometimes called "nlerp blend") is fast and produces acceptable results for typical blend scenarios. It is not as accurate as iterative slerp-based blending, but the difference is negligible when weights change gradually (as in crossfades).

### Implementation

```typescript
// Per-bone blend accumulators, pre-allocated to avoid per-frame allocation
interface BoneBlendState {
  position: Vec3       // accumulated weighted position
  rotation: Quat       // accumulated weighted rotation
  scale: Vec3          // accumulated weighted scale
  totalWeight: number  // sum of weights that contributed to this bone
  hasRotation: boolean // whether any action contributed a rotation
}

const createBoneBlendStates = (boneCount: number): BoneBlendState[] => {
  const states: BoneBlendState[] = new Array(boneCount)
  for (let i = 0; i < boneCount; i++) {
    states[i] = {
      position: new Float32Array(3) as Vec3,
      rotation: new Float32Array(4) as Quat,
      scale: new Float32Array(3) as Vec3,
      totalWeight: 0,
      hasRotation: false,
    }
  }
  return states
}

const resetBlendStates = (states: BoneBlendState[]): void => {
  for (let i = 0; i < states.length; i++) {
    const s = states[i]
    s.position[0] = 0; s.position[1] = 0; s.position[2] = 0
    s.rotation[0] = 0; s.rotation[1] = 0; s.rotation[2] = 0; s.rotation[3] = 0
    s.scale[0] = 0; s.scale[1] = 0; s.scale[2] = 0
    s.totalWeight = 0
    s.hasRotation = false
  }
}

const accumulateAction = (
  action: AnimationAction,
  blendStates: BoneBlendState[],
  tempPos: Vec3,
  tempRot: Quat,
  tempScale: Vec3
): void => {
  const w = action.weight
  if (w <= 0) return

  for (const track of action.clip.tracks) {
    const bone = blendStates[track.boneIndex]

    // Reset temp to identity
    tempPos[0] = 0; tempPos[1] = 0; tempPos[2] = 0
    tempRot[0] = 0; tempRot[1] = 0; tempRot[2] = 0; tempRot[3] = 1
    tempScale[0] = 1; tempScale[1] = 1; tempScale[2] = 1

    evaluateTrack(track, action.time, tempPos, tempRot, tempScale)

    switch (track.property) {
      case "position":
        bone.position[0] += tempPos[0] * w
        bone.position[1] += tempPos[1] * w
        bone.position[2] += tempPos[2] * w
        bone.totalWeight += w
        break

      case "rotation":
        // Ensure consistent hemisphere before accumulating
        if (bone.hasRotation) {
          const dot = bone.rotation[0] * tempRot[0]
                    + bone.rotation[1] * tempRot[1]
                    + bone.rotation[2] * tempRot[2]
                    + bone.rotation[3] * tempRot[3]
          if (dot < 0) {
            tempRot[0] = -tempRot[0]
            tempRot[1] = -tempRot[1]
            tempRot[2] = -tempRot[2]
            tempRot[3] = -tempRot[3]
          }
        }
        bone.rotation[0] += tempRot[0] * w
        bone.rotation[1] += tempRot[1] * w
        bone.rotation[2] += tempRot[2] * w
        bone.rotation[3] += tempRot[3] * w
        bone.hasRotation = true
        break

      case "scale":
        bone.scale[0] += tempScale[0] * w
        bone.scale[1] += tempScale[1] * w
        bone.scale[2] += tempScale[2] * w
        break
    }
  }
}

const finalizeBlendStates = (blendStates: BoneBlendState[], bones: readonly Bone[]): void => {
  for (let i = 0; i < blendStates.length; i++) {
    const s = blendStates[i]
    const bone = bones[i]

    if (s.totalWeight > 0) {
      const invW = 1.0 / s.totalWeight
      bone.position[0] = s.position[0] * invW
      bone.position[1] = s.position[1] * invW
      bone.position[2] = s.position[2] * invW
    }

    if (s.hasRotation) {
      // Normalize the accumulated quaternion
      const rx = s.rotation[0], ry = s.rotation[1]
      const rz = s.rotation[2], rw = s.rotation[3]
      const len = Math.sqrt(rx * rx + ry * ry + rz * rz + rw * rw)
      if (len > 0) {
        const invLen = 1.0 / len
        bone.rotation[0] = rx * invLen
        bone.rotation[1] = ry * invLen
        bone.rotation[2] = rz * invLen
        bone.rotation[3] = rw * invLen
      }
    }

    if (s.totalWeight > 0) {
      const invW = 1.0 / s.totalWeight
      bone.scale[0] = s.scale[0] * invW
      bone.scale[1] = s.scale[1] * invW
      bone.scale[2] = s.scale[2] * invW
    }

    markDirty(bone)
  }
}
```

---

## 4. Crossfade

Crossfading smoothly transitions from one animation to another over a specified duration. During the crossfade, both animations play simultaneously. The outgoing animation's weight ramps from 1 to 0, and the incoming animation's weight ramps from 0 to 1.

### API

```typescript
// Transition from idle to walk over 0.3 seconds
mixer.crossFade(idleAction, walkAction, 0.3)
```

### Implementation

```typescript
const createAnimationMixer = (skeleton: Skeleton): AnimationMixer => {
  const actions: AnimationAction[] = []
  const blendStates = createBoneBlendStates(skeleton.boneCount)
  let activeCrossFade: CrossFadeState | null = null

  // Pre-allocate temporaries for interpolation
  const tempPos = new Float32Array(3) as Vec3
  const tempRot = new Float32Array(4) as Quat
  const tempScale = new Float32Array(3) as Vec3

  const clipAction = (clip: AnimationClip): AnimationAction => {
    // Reuse existing action for the same clip if present
    const existing = actions.find(a => a.clip === clip)
    if (existing) return existing

    const action: AnimationAction = {
      clip,
      time: 0,
      speed: 1.0,
      weight: 1.0,
      loop: "repeat",
      playing: false,
      paused: false,

      play: () => { action.playing = true; action.paused = false; return action },
      stop: () => { action.playing = false; action.time = 0; return action },
      pause: () => { action.paused = true; return action },
      reset: () => { action.time = 0; return action },
      setWeight: (w) => { action.weight = w; return action },
      setSpeed: (s) => { action.speed = s; return action },
      setLoop: (m) => { action.loop = m; return action },
    }

    actions.push(action)
    return action
  }

  const crossFade = (
    fromAction: AnimationAction,
    toAction: AnimationAction,
    duration: number
  ): void => {
    // Ensure both actions are playing
    fromAction.play()
    toAction.play().reset()

    // Set initial weights
    fromAction.weight = 1.0
    toAction.weight = 0.0

    activeCrossFade = {
      fromAction,
      toAction,
      duration,
      elapsed: 0,
    }
  }

  const updateCrossFade = (dt: number): void => {
    if (!activeCrossFade) return

    const cf = activeCrossFade
    cf.elapsed += dt

    const t = Math.min(cf.elapsed / cf.duration, 1.0)

    cf.fromAction.weight = 1.0 - t
    cf.toAction.weight = t

    // Crossfade complete
    if (t >= 1.0) {
      cf.fromAction.stop()
      cf.fromAction.weight = 0.0
      cf.toAction.weight = 1.0
      activeCrossFade = null
    }
  }

  const update = (deltaTime: number): void => {
    // 1. Update crossfade weights
    updateCrossFade(deltaTime)

    // 2. Advance time for all playing actions
    for (let i = 0; i < actions.length; i++) {
      const action = actions[i]
      if (!action.playing || action.paused) continue

      action.time += deltaTime * action.speed

      // Handle looping
      const duration = action.clip.duration
      switch (action.loop) {
        case "repeat":
          if (action.time >= duration) {
            action.time = action.time % duration
          } else if (action.time < 0) {
            action.time = duration + (action.time % duration)
          }
          break

        case "pingpong":
          if (action.time >= duration || action.time < 0) {
            action.speed = -action.speed
            action.time = Math.max(0, Math.min(duration, action.time))
          }
          break

        case "once":
          if (action.time >= duration) {
            action.time = duration
            action.playing = false
          } else if (action.time < 0) {
            action.time = 0
            action.playing = false
          }
          break
      }
    }

    // 3. Blend all active actions into bone transforms
    resetBlendStates(blendStates)

    for (let i = 0; i < actions.length; i++) {
      const action = actions[i]
      if (!action.playing || action.weight <= 0) continue
      accumulateAction(action, blendStates, tempPos, tempRot, tempScale)
    }

    finalizeBlendStates(blendStates, skeleton.bones)

    // 4. Update skeleton world matrices (top-down traversal)
    updateSkeletonWorldMatrices(skeleton)

    // 5. Compute final bone matrices for GPU upload
    computeBoneMatrices(skeleton)
  }

  const stopAll = (): void => {
    for (let i = 0; i < actions.length; i++) {
      actions[i].stop()
    }
    activeCrossFade = null
  }

  return {
    skeleton,
    get actions() { return actions },
    clipAction,
    play: (clip) => clipAction(clip).play(),
    stopAll,
    crossFade,
    update,
  }
}
```

### Crossfade Timeline

```
Time ──────────────────────────────────────────────►
           crossFade(idle, walk, 0.3)
                    │
                    ▼
Idle weight:  1.0 ─────╲                    0.0
                        ╲
Walk weight:  0.0        ╱──────────────── 1.0
                        ╱
                    ◄──0.3s──►

During the 0.3s window, both idle and walk play simultaneously.
Their transforms are blended by weight at each bone.
After 0.3s, idle is stopped and walk runs alone at full weight.
```

---

## 5. Skinning

Skinning transforms each vertex of a mesh from its rest pose to match the current skeleton pose. Each vertex is influenced by up to 4 bones, specified by joint indices and weights stored as vertex attributes.

### Vertex Attributes

```typescript
// Skinned mesh geometry has two additional vertex attributes:

// joints0: uvec4 — indices into the skeleton's bone array
// Each component is a bone index (0-255 for uint8, 0-65535 for uint16)

// weights0: vec4 — blend weights for each joint influence
// Components sum to 1.0. Up to 4 bone influences per vertex.
```

### Skinning Matrix Computation

For each vertex, the skinning matrix is:

```
skinMatrix = weight[0] * boneMatrices[joint[0]]
           + weight[1] * boneMatrices[joint[1]]
           + weight[2] * boneMatrices[joint[2]]
           + weight[3] * boneMatrices[joint[3]]
```

Where `boneMatrices[i]` is defined in section 6 (Bone Matrix Upload).

### Skinning Vertex Shader (GLSL 300 es)

```glsl
#version 300 es
precision highp float;

// --- Per-frame uniforms ---
layout(std140) uniform FrameUniforms {
    mat4 u_viewMatrix;
    mat4 u_projectionMatrix;
    mat4 u_viewProjectionMatrix;
    vec3 u_cameraPosition;
    float u_time;
};

// --- Per-object uniforms ---
layout(std140) uniform ObjectUniforms {
    mat4 u_worldMatrix;
};

// --- Bone matrices (skinning UBO, binding point 3) ---
// For <= 128 bones: UBO. Each mat4 = 64 bytes. 128 * 64 = 8192 bytes.
// This fits within the minimum guaranteed UBO size (16384 bytes).
#ifdef USE_SKINNING_UBO
layout(std140) uniform SkinningUniforms {
    mat4 u_boneMatrices[MAX_BONES];  // MAX_BONES <= 128
};
#endif

// --- Bone matrices (data texture fallback for > 128 bones) ---
#ifdef USE_SKINNING_TEXTURE
uniform highp sampler2D u_boneTexture;
uniform int u_boneTextureWidth;

mat4 getBoneMatrix(int boneIndex) {
    // Each bone matrix = 4 texels (one row per texel, RGBA32F)
    int row = boneIndex * 4;
    float y = (float(row) + 0.5) / float(u_boneTextureWidth);

    vec4 col0 = texture(u_boneTexture, vec2(0.125, y));
    vec4 col1 = texture(u_boneTexture, vec2(0.375, y));
    vec4 col2 = texture(u_boneTexture, vec2(0.625, y));
    vec4 col3 = texture(u_boneTexture, vec2(0.875, y));

    return mat4(col0, col1, col2, col3);
}
#endif

// --- Vertex attributes ---
layout(location = 0) in vec3 a_position;
layout(location = 1) in vec3 a_normal;
layout(location = 2) in vec2 a_uv;
layout(location = 3) in uvec4 a_joints;   // bone indices
layout(location = 4) in vec4 a_weights;    // bone weights

// --- Varyings ---
out vec3 v_worldPosition;
out vec3 v_worldNormal;
out vec2 v_uv;
out float v_viewDepth;

void main() {
    // Compute skinning matrix from up to 4 bone influences
#ifdef USE_SKINNING_UBO
    mat4 skinMatrix =
        u_boneMatrices[a_joints.x] * a_weights.x +
        u_boneMatrices[a_joints.y] * a_weights.y +
        u_boneMatrices[a_joints.z] * a_weights.z +
        u_boneMatrices[a_joints.w] * a_weights.w;
#endif

#ifdef USE_SKINNING_TEXTURE
    mat4 skinMatrix =
        getBoneMatrix(int(a_joints.x)) * a_weights.x +
        getBoneMatrix(int(a_joints.y)) * a_weights.y +
        getBoneMatrix(int(a_joints.z)) * a_weights.z +
        getBoneMatrix(int(a_joints.w)) * a_weights.w;
#endif

    // Apply skinning in local space, then world transform
    vec4 skinnedPosition = skinMatrix * vec4(a_position, 1.0);
    vec3 skinnedNormal = mat3(skinMatrix) * a_normal;

    vec4 worldPos = u_worldMatrix * skinnedPosition;
    v_worldPosition = worldPos.xyz;
    v_worldNormal = normalize(mat3(u_worldMatrix) * skinnedNormal);
    v_uv = a_uv;

    vec4 viewPos = u_viewMatrix * worldPos;
    v_viewDepth = -viewPos.z;

    gl_Position = u_projectionMatrix * viewPos;
}
```

### Skinning Vertex Shader (WGSL)

```wgsl
struct FrameUniforms {
    viewMatrix: mat4x4<f32>,
    projectionMatrix: mat4x4<f32>,
    viewProjectionMatrix: mat4x4<f32>,
    cameraPosition: vec3<f32>,
    time: f32,
}

struct ObjectUniforms {
    worldMatrix: mat4x4<f32>,
}

struct SkinningUniforms {
    boneMatrices: array<mat4x4<f32>, 128>,  // MAX_BONES
}

@group(0) @binding(0) var<uniform> frame: FrameUniforms;
@group(2) @binding(0) var<uniform> object: ObjectUniforms;
@group(3) @binding(0) var<uniform> skin: SkinningUniforms;

struct VertexInput {
    @location(0) position: vec3<f32>,
    @location(1) normal: vec3<f32>,
    @location(2) uv: vec2<f32>,
    @location(3) joints: vec4<u32>,
    @location(4) weights: vec4<f32>,
}

struct VertexOutput {
    @builtin(position) clipPosition: vec4<f32>,
    @location(0) worldPosition: vec3<f32>,
    @location(1) worldNormal: vec3<f32>,
    @location(2) uv: vec2<f32>,
    @location(3) viewDepth: f32,
}

@vertex
fn main(input: VertexInput) -> VertexOutput {
    let skinMatrix =
        skin.boneMatrices[input.joints.x] * input.weights.x +
        skin.boneMatrices[input.joints.y] * input.weights.y +
        skin.boneMatrices[input.joints.z] * input.weights.z +
        skin.boneMatrices[input.joints.w] * input.weights.w;

    let skinnedPos = skinMatrix * vec4<f32>(input.position, 1.0);
    let skinnedNormal = (mat3x3<f32>(
        skinMatrix[0].xyz,
        skinMatrix[1].xyz,
        skinMatrix[2].xyz
    )) * input.normal;

    let worldPos = object.worldMatrix * skinnedPos;
    let viewPos = frame.viewMatrix * worldPos;

    var out: VertexOutput;
    out.clipPosition = frame.projectionMatrix * viewPos;
    out.worldPosition = worldPos.xyz;
    out.worldNormal = normalize((mat3x3<f32>(
        object.worldMatrix[0].xyz,
        object.worldMatrix[1].xyz,
        object.worldMatrix[2].xyz
    )) * skinnedNormal);
    out.uv = input.uv;
    out.viewDepth = -viewPos.z;
    return out;
}
```

### UBO vs Data Texture Decision

| Strategy | Bone Limit | Memory | Access Pattern |
|---|---|---|---|
| UBO (`std140`) | <= 128 bones | 128 x 64 = 8,192 bytes | Direct indexed array access, fast |
| Data Texture (RGBA32F) | Unlimited | 4 texels per bone, width = boneCount | Texture fetch, slightly slower |

The decision is made at skeleton creation time:

```typescript
const createSkinningResource = (
  device: GalDevice,
  skeleton: Skeleton
): SkinningResource => {
  if (skeleton.boneCount <= 128) {
    // UBO path: fits within the minimum guaranteed UBO size (16KB)
    const buffer = device.createBuffer({
      label: "skinning-ubo",
      size: skeleton.boneCount * 64,  // 64 bytes per mat4
      usage: ["uniform"],
      dynamic: true,
    })
    return { type: "ubo", buffer }
  }

  // Data texture fallback for large skeletons
  // Each bone = 4 rows x 4 columns = 4 RGBA32F texels
  const texHeight = skeleton.boneCount * 4
  const texture = device.createTexture({
    label: "skinning-texture",
    dimension: "2d",
    format: "rgba32float",
    width: 4,
    height: texHeight,
    usage: ["texture-binding", "copy-dst"],
  })
  return { type: "texture", texture, width: 4, height: texHeight }
}
```

---

## 6. Bone Matrix Upload

After the animation mixer updates all bone local transforms, the skeleton must compute final bone matrices for the GPU. This involves two steps: computing world matrices via a top-down traversal, then multiplying by inverse bind matrices.

### Step 1: Compute Bone World Matrices

Bones form a hierarchy. The root bone's world matrix equals its local matrix (or parent node's world matrix if the skeleton is a child of another node). Each child bone's world matrix is `parent.worldMatrix * child.localMatrix`.

The scene graph's normal `updateWorldMatrix` traversal handles this. Since `markDirty` was called on each bone whose transform changed (during blend finalization), only the dirty subtrees are recomputed.

```typescript
const updateSkeletonWorldMatrices = (skeleton: Skeleton): void => {
  // Find the root bone(s) and traverse from there.
  // Typically a skeleton has one root bone.
  // The standard scene graph top-down traversal handles this.
  for (const bone of skeleton.bones) {
    if (bone.dirty) {
      updateWorldMatrix(bone)
    }
  }
}
```

### Step 2: Compute Final Bone Matrices

The bone matrix uploaded to the GPU for bone `i` is:

```
boneMatrix[i] = bone[i].worldMatrix * inverseBindMatrix[i]
```

The `inverseBindMatrix[i]` is the inverse of bone `i`'s world matrix in the rest pose. This ensures that a vertex defined in rest-pose space is first "un-transformed" from rest-pose bone space, then re-transformed into the current animated bone space.

```typescript
const computeBoneMatrices = (skeleton: Skeleton): void => {
  const { bones, inverseBindMatrices, boneMatrices } = skeleton

  for (let i = 0; i < bones.length; i++) {
    const boneWorldMatrix = bones[i].worldMatrix
    const inverseBindMatrix = inverseBindMatrices[i]

    // boneMatrices[i] = bone.worldMatrix * inverseBindMatrix
    // Write directly into the flat Float32Array at offset i * 16
    const offset = i * 16
    mat4Multiply(boneWorldMatrix, inverseBindMatrix, boneMatrices, offset)
  }
}

// mat4Multiply variant that writes into a flat array at an offset
const mat4Multiply = (
  a: Mat4,
  b: Mat4,
  out: Float32Array,
  outOffset: number
): void => {
  for (let col = 0; col < 4; col++) {
    for (let row = 0; row < 4; row++) {
      let sum = 0
      for (let k = 0; k < 4; k++) {
        sum += a[k * 4 + row] * b[col * 4 + k]
      }
      out[outOffset + col * 4 + row] = sum
    }
  }
}
```

### Step 3: Upload to GPU

After `computeBoneMatrices` fills `skeleton.boneMatrices`, the flat Float32Array is uploaded to the GPU resource (UBO or data texture). One upload per skeleton per frame.

```typescript
const uploadBoneMatrices = (
  device: GalDevice,
  skeleton: Skeleton,
  resource: SkinningResource
): void => {
  if (resource.type === "ubo") {
    device.writeBuffer(resource.buffer, skeleton.boneMatrices)
  } else {
    // Write bone matrices as texel data into the data texture
    device.writeTexture(resource.texture, skeleton.boneMatrices, {
      offset: 0,
      bytesPerRow: 4 * 16,  // 4 RGBA32F texels per row = 64 bytes
      rowsPerImage: skeleton.boneCount * 4,
    })
  }
}
```

### One UBO Per Skeleton Instance

Each skeleton instance has its own UBO (or data texture). If two skinned meshes share the same skeleton (e.g., two parts of a character with separate materials but the same bone hierarchy), they reference the same skinning UBO. The UBO is bound once per skeleton, not once per mesh.

```
Skeleton Instance (1 UBO)
├── SkinnedMesh: character_body    (references skeleton UBO at bind group 3)
├── SkinnedMesh: character_armor   (references same skeleton UBO at bind group 3)
└── SkinnedMesh: character_hair    (references same skeleton UBO at bind group 3)
```

---

## 7. Bone Attachment

Attaching a static mesh (weapon, armor piece, accessory) to a bone requires no special API. Because bones are regular scene graph nodes, you parent the mesh to the bone with `addChild`. The mesh's world matrix is automatically computed from the bone's world matrix during the standard scene graph traversal.

### Attach

```typescript
// Find the bone by name
const handBone = skeleton.getBone("hand_R")

// Create the weapon mesh
const swordMesh = createMesh(swordGeometry, swordMaterial)

// Position and orient relative to the bone
setPosition(swordMesh, 0, 0, 0.15)   // slight offset in Z (up)
setRotation(swordMesh, quatFromAxisAngle(vec3(1, 0, 0), -Math.PI / 2))

// Attach — one line
handBone.add(swordMesh)
```

After attachment, every frame:

```
swordMesh.worldMatrix = handBone.worldMatrix * swordMesh.localMatrix
```

The sword follows the hand bone perfectly. When the animation system updates `handBone`'s transform, `markDirty` propagates to the sword. The next scene graph update recomputes the sword's world matrix.

### Detach

```typescript
skeleton.getBone("hand_R").remove(swordMesh)
```

After removal, the sword is no longer part of the scene graph (unless re-parented elsewhere).

### How It Works Internally

Nothing special happens. The scene graph does not distinguish between "regular" children and "attached" objects:

```
// Mesh parented to a Group:
mesh.worldMatrix = group.worldMatrix * mesh.localMatrix

// Sword parented to a Bone:
sword.worldMatrix = handBone.worldMatrix * sword.localMatrix

// Same computation. Same code path. Same traversal.
```

The attachment participates in the normal scene graph transform update, frustum culling, and rendering pipeline. The sword appears in the visible mesh list if the bone's world-space AABB (which encloses the sword) passes the frustum test.

---

## 8. GLTF Animation Import

GLTF is the primary asset format for Lynx. GLTF stores animations as samplers and channels that must be converted to Lynx's `KeyframeTrack` and `AnimationClip` format. Additionally, GLTF uses a Y-up right-handed coordinate system, so all animation data must be converted to Lynx's Z-up system.

### GLTF Animation Structure

A GLTF animation contains:
- **Samplers**: each has an `input` accessor (keyframe times) and an `output` accessor (keyframe values)
- **Channels**: each targets a specific node and property (translation, rotation, scale, weights), referencing a sampler

### Conversion to Lynx Format

```typescript
interface GltfAnimation {
  name: string
  channels: GltfChannel[]
  samplers: GltfSampler[]
}

interface GltfChannel {
  sampler: number
  target: {
    node: number
    path: "translation" | "rotation" | "scale" | "weights"
  }
}

interface GltfSampler {
  input: number   // accessor index for times
  output: number  // accessor index for values
  interpolation: "LINEAR" | "STEP" | "CUBICSPLINE"
}

const convertGltfAnimation = (
  gltfAnim: GltfAnimation,
  accessors: GltfAccessor[],
  nodeToBonesIndex: Map<number, number>
): AnimationClip => {
  const tracks: KeyframeTrack[] = []
  let duration = 0

  for (const channel of gltfAnim.channels) {
    const boneIndex = nodeToBonesIndex.get(channel.target.node)
    if (boneIndex === undefined) continue  // not a bone node

    const sampler = gltfAnim.samplers[channel.sampler]
    const times = getAccessorData(accessors[sampler.input]) as Float32Array
    const values = getAccessorData(accessors[sampler.output]) as Float32Array

    // Track the longest channel to determine clip duration
    if (times.length > 0 && times[times.length - 1] > duration) {
      duration = times[times.length - 1]
    }

    // Convert Y-up to Z-up
    const property = convertPathToProperty(channel.target.path)
    const convertedValues = convertAnimationValues(
      values,
      channel.target.path,
      property
    )

    tracks.push({
      boneIndex,
      property,
      times,
      values: convertedValues,
    })
  }

  return {
    name: gltfAnim.name ?? "unnamed",
    duration,
    tracks,
  }
}

const convertPathToProperty = (
  gltfPath: "translation" | "rotation" | "scale" | "weights"
): "position" | "rotation" | "scale" => {
  switch (gltfPath) {
    case "translation": return "position"
    case "rotation": return "rotation"
    case "scale": return "scale"
    case "weights": return "scale"  // morph weights mapped to scale tracks (simplified)
  }
}
```

### Y-Up to Z-Up Conversion

GLTF uses Y-up right-handed. Lynx uses Z-up right-handed. The conversion swaps Y and Z axes and negates the new Y to preserve handedness:

```
GLTF (Y-up):  X_gltf  Y_gltf  Z_gltf
Lynx (Z-up):  X_lynx  Y_lynx  Z_lynx

X_lynx =  X_gltf
Y_lynx = -Z_gltf
Z_lynx =  Y_gltf
```

```typescript
const convertAnimationValues = (
  values: Float32Array,
  gltfPath: string,
  property: "position" | "rotation" | "scale"
): Float32Array => {
  if (property === "position") {
    // Translation: swap Y/Z, negate new Y
    const out = new Float32Array(values.length)
    for (let i = 0; i < values.length; i += 3) {
      out[i]     =  values[i]       // X_lynx =  X_gltf
      out[i + 1] = -values[i + 2]   // Y_lynx = -Z_gltf
      out[i + 2] =  values[i + 1]   // Z_lynx =  Y_gltf
    }
    return out
  }

  if (property === "rotation") {
    // Quaternion: apply the same axis swap to the imaginary components
    // For quaternion (qx, qy, qz, qw) in Y-up:
    //   qx_lynx =  qx_gltf
    //   qy_lynx = -qz_gltf
    //   qz_lynx =  qy_gltf
    //   qw_lynx =  qw_gltf
    const out = new Float32Array(values.length)
    for (let i = 0; i < values.length; i += 4) {
      out[i]     =  values[i]       // qx_lynx =  qx_gltf
      out[i + 1] = -values[i + 2]   // qy_lynx = -qz_gltf
      out[i + 2] =  values[i + 1]   // qz_lynx =  qy_gltf
      out[i + 3] =  values[i + 3]   // qw_lynx =  qw_gltf
    }
    return out
  }

  if (property === "scale") {
    // Scale: swap Y/Z (no negation -- scale is always positive)
    const out = new Float32Array(values.length)
    for (let i = 0; i < values.length; i += 3) {
      out[i]     = values[i]        // X_lynx = X_gltf
      out[i + 1] = values[i + 2]    // Y_lynx = Z_gltf
      out[i + 2] = values[i + 1]    // Z_lynx = Y_gltf
    }
    return out
  }

  return values
}
```

### Skeleton Import

The skeleton is also imported from GLTF with the same coordinate conversion applied to the inverse bind matrices and rest-pose bone transforms:

```typescript
const importSkeleton = (
  gltfSkin: GltfSkin,
  gltfNodes: GltfNode[],
  accessors: GltfAccessor[]
): Skeleton => {
  const boneCount = gltfSkin.joints.length
  const bones: Bone[] = new Array(boneCount)
  const inverseBindMatrices: Mat4[] = new Array(boneCount)

  // Read inverse bind matrices from accessor
  const ibmData = getAccessorData(accessors[gltfSkin.inverseBindMatrices]) as Float32Array

  // Coordinate conversion matrix: Y-up to Z-up
  // This is applied to each inverse bind matrix
  const yUpToZUp = mat4FromAxes(
    vec3(1, 0, 0),    // X stays X
    vec3(0, 0, -1),   // Y becomes -Z
    vec3(0, 1, 0)     // Z becomes Y
  )
  const zUpToYUp = mat4Transpose(yUpToZUp)  // inverse of orthogonal matrix = transpose

  for (let i = 0; i < boneCount; i++) {
    const gltfNodeIndex = gltfSkin.joints[i]
    const gltfNode = gltfNodes[gltfNodeIndex]

    // Create bone node with converted transform
    bones[i] = createBone(gltfNode.name ?? `bone_${i}`, i)
    applyConvertedTransform(bones[i], gltfNode)

    // Convert inverse bind matrix: IBMz = yUpToZUp * IBMy * zUpToYUp
    const ibm = ibmData.subarray(i * 16, i * 16 + 16)
    inverseBindMatrices[i] = convertMatrix(ibm, yUpToZUp, zUpToYUp)
  }

  // Build bone hierarchy from GLTF node parent-child relationships
  for (let i = 0; i < boneCount; i++) {
    const gltfNodeIndex = gltfSkin.joints[i]
    const gltfNode = gltfNodes[gltfNodeIndex]

    if (gltfNode.children) {
      for (const childNodeIndex of gltfNode.children) {
        const childBoneIndex = gltfSkin.joints.indexOf(childNodeIndex)
        if (childBoneIndex >= 0) {
          addChild(bones[i], bones[childBoneIndex])
        }
      }
    }
  }

  return createSkeleton(bones, inverseBindMatrices)
}
```

---

## Complete Usage Example

### Loading and Playing a Single Animation

```typescript
// Load a GLTF character model
const gltf = await loadGltf("models/character.glb")

// The loader returns a SkinnedMesh with a Skeleton and AnimationClips
const characterMesh = gltf.skinnedMeshes[0]
const skeleton = characterMesh.skeleton
const clips = gltf.animations

// Add to scene
scene.add(characterMesh)

// Create an animation mixer for this skeleton
const mixer = createAnimationMixer(skeleton)

// Play the "idle" clip
const idleClip = clips.find(c => c.name === "idle")!
const idleAction = mixer.play(idleClip)
idleAction.setLoop("repeat")

// In the render loop:
const frame = (dt: number): void => {
  mixer.update(dt)
  renderer.render(scene, camera)
  requestAnimationFrame(() => frame(clock.getDelta()))
}
```

### Crossfading Between Animations

```typescript
const idleClip = clips.find(c => c.name === "idle")!
const walkClip = clips.find(c => c.name === "walk")!
const runClip = clips.find(c => c.name === "run")!

// Start with idle
const idleAction = mixer.play(idleClip).setLoop("repeat")

// Player starts moving — crossfade to walk
const onStartWalking = (): void => {
  const walkAction = mixer.clipAction(walkClip).setLoop("repeat")
  mixer.crossFade(idleAction, walkAction, 0.3)
}

// Player speeds up — crossfade walk to run
const onStartRunning = (): void => {
  const walkAction = mixer.clipAction(walkClip)
  const runAction = mixer.clipAction(runClip).setLoop("repeat")
  mixer.crossFade(walkAction, runAction, 0.2)
}

// Player stops — crossfade back to idle
const onStop = (): void => {
  const currentAction = mixer.actions.find(a => a.playing && a.weight > 0)!
  mixer.crossFade(currentAction, mixer.clipAction(idleClip).setLoop("repeat"), 0.4)
}
```

### Attaching a Weapon

```typescript
// Load weapon model
const swordGltf = await loadGltf("models/sword.glb")
const swordMesh = swordGltf.meshes[0]

// Position the sword relative to the hand bone
setPosition(swordMesh, 0, 0, 0.15)
setRotation(swordMesh, quatFromAxisAngle(vec3(1, 0, 0), -Math.PI / 2))

// Attach to right hand
skeleton.getBone("hand_R")!.add(swordMesh)

// Later, swap to a different weapon
const axeGltf = await loadGltf("models/axe.glb")
const axeMesh = axeGltf.meshes[0]

skeleton.getBone("hand_R")!.remove(swordMesh)
setPosition(axeMesh, 0, 0, 0.1)
skeleton.getBone("hand_R")!.add(axeMesh)
```

### Multiple Skinned Meshes on One Skeleton

```typescript
// A character might have body, armor, and hair as separate skinned meshes
// sharing the same skeleton
const bodyMesh = createSkinnedMesh(bodyGeometry, bodyMaterial, skeleton)
const armorMesh = createSkinnedMesh(armorGeometry, armorMaterial, skeleton)
const hairMesh = createSkinnedMesh(hairGeometry, hairMaterial, skeleton)

scene.add(bodyMesh)
scene.add(armorMesh)
scene.add(hairMesh)

// One mixer controls the skeleton, all three meshes follow
const mixer = createAnimationMixer(skeleton)
mixer.play(idleClip).setLoop("repeat")

// One bone matrix upload per frame, shared by all three meshes
// Each mesh's draw call binds the same skinning UBO (bind group 3)
```

---

## Performance Characteristics

### Per-Frame Work

| Operation | Cost | Allocation |
|---|---|---|
| Action time advancement | O(actions) | Zero |
| Crossfade weight update | O(1) | Zero |
| Keyframe evaluation (binary search) | O(tracks * log(keyframes)) | Zero |
| Animation blending | O(actions * tracks) | Zero |
| Bone world matrix update | O(dirty bones) | Zero |
| Bone matrix computation | O(bones) - mat4 multiply | Zero |
| GPU upload | 1 writeBuffer per skeleton | Zero |

### Memory Layout

```
Skeleton:
  bones[]              — Array of Bone node references (scene graph nodes)
  inverseBindMatrices[] — Array of Mat4 (Float32Array, 16 floats each)
  boneMatrices         — Single contiguous Float32Array (boneCount * 16 floats)
                         Written linearly, uploaded to GPU in one call

KeyframeTrack:
  times   — Float32Array, sorted ascending
  values  — Float32Array, tightly packed (3 or 4 floats per key)
            Contiguous layout enables CPU cache-friendly binary search

BoneBlendState[] — Pre-allocated array, reset each frame (zero GC pressure)
```

### Steady-State Allocation

During steady-state animation playback (no new clips loaded, no actions created/destroyed), the animation system performs zero JavaScript heap allocations per frame. All temporaries are pre-allocated in the mixer. The only GPU write is `device.writeBuffer` for the bone matrices, which does not allocate on the JavaScript side.
