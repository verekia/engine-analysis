I want to create a minimal 3D engine for games that has a developer experience somewhat similar to three.js but with very high performance (unlike three.js). The engine should support:
- both webgl and webgpu. If WebGPU has useful advanced features unavailable in WebGL, feel free to use them, but there should always be a webgl fallback 
- basic and Lambert materials
- gltf with draco decoding
- color textures, ao textures, KTX2 with basis decoding
- per vertex bloom, vertex colors, being able to apply vertex color and emissive per material index (models will be provided as single meshes with '_materialindex' attributes, so we should be able to say material 0 is white, 1 is black, 2 is teal with emissive of intensity 0.7 and have them colored with vertex colors + emit bloom (Unreal Bloom)
- efficient simple MSAA
- Raycasting and BVH as efficient as three-mesh-bvh
- skinned meshes, with skeletal animations, simple cross fades between animations
- frustum culling
- transparency that actually works without headaches (unlike three.js)
- very high performance: it should be able to render 2000 draw calls at 60fps on recent phones
- orbit controls
- parametrized plane, box, sphere, cone, cylinder, capsule, circle geometries 
- directional light, ambient light
- cascading shadow maps (worlds are low poly 200x200m, we need smooth shadows)
- z is up, right handed
- no instancing, but by default it should be so performant and allow very high number of draw calls (unlike three.js) that instancing wouldn't be needed in many cases.
- it should be possible to have parent/children and Group like Three.js.
- it should be possible to easily attach a static mesh to a bone, such as a sword to a hand
- an HTML overlay feature similar to CSS3DRenderer or Drei Html to render DOM elements at given scene positions on top of the canvas.
- React bindings similar to React Three Fiber
- use TS, no semicolons, prefer const arrow functions over function blocks.
- code should be minimal, highly performant, well documented, understandable


Figure out the best technical decisions to succeed with this project, then pick a random animal name, create a folder and commit PLAN.md in it for this library, as if it was the name of the library. Describe in several markdown files each piece of the architecture in detail.