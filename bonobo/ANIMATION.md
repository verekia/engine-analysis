# Animation

## Status: Out of Scope for v1

Skeletal animation is explicitly out of scope for v1. This will be added in v2 once the core engine is solid.

## Planned Features (v2)

### Skeletal Animation

- GPU-skinned meshes (vertex shader bone transformations)
- Support for up to 256 bones per skeleton
- Joint matrices uploaded as uniform buffer or storage buffer

### Animation System

- Animation clips (keyframe data for bone transforms)
- Animation blending and cross-fading
- Animation state machines
- Root motion support

### Bone Attachment

- Attach static meshes to bones (weapons, accessories)
- Per-bone world matrix computation for attachment points

### Material Support

Skinned material type:

```ts
interface SkinnedMaterial {
  type: 'skinned'
  id: number
  // Joint matrices provided per-entity at draw time
}
```

### Instance Buffer Extension

For skinned meshes, the instance buffer would need additional data:

```
worldMatrix:  mat4x4<f32>   (64 bytes)
color:        vec4<f32>     (16 bytes)
flags:        u32           (4 bytes)
boneOffset:   u32           (4 bytes) - offset into bone matrix buffer
padding:      8 bytes       (align to 96)
```

## Implementation Notes

- Bone matrices stored in a separate buffer (shared across all skinned instances)
- Each skinned entity references a range in the bone matrix buffer
- Animation evaluation happens on CPU (for now), bone matrices uploaded to GPU each frame
- Future: GPU animation evaluation using compute shaders

## Workarounds for v1

For v1, animated meshes can be pre-baked or use vertex morphing (if needed). Most games can launch with static meshes only.
