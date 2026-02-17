# HTML Overlay Architecture

## Overview

Shark provides an HTML overlay system that positions DOM elements at 3D scene positions on top of the WebGL/WebGPU canvas. This is similar to Three.js's `CSS3DRenderer` or Drei's `<Html>` component — but simpler, positioning flat DOM elements at projected screen coordinates rather than full CSS3D transforms.

## Use Cases

- Health bars above characters
- Name tags and labels
- Tooltips on hover
- 2D UI panels anchored to 3D objects
- Speech bubbles
- Interaction prompts ("Press E")

## Architecture

### Container Setup

An overlay `<div>` is positioned directly on top of the canvas:

```html
<div style="position: relative; width: 800px; height: 600px;">
  <canvas id="shark-canvas" style="width: 100%; height: 100%;"></canvas>
  <div id="shark-overlay" style="
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    overflow: hidden;
  "></div>
</div>
```

The overlay div has `pointer-events: none` so clicks pass through to the canvas. Individual overlay items can set `pointer-events: auto` to receive mouse events.

### HtmlAnchor Node

An `HtmlAnchor` is a scene graph node that tracks a 3D position and manages a DOM element:

```typescript
interface HtmlAnchor extends Node3D {
  // DOM element to position
  element: HTMLElement

  // Projection behavior
  center: Vec2                // Anchor point [0,0]=top-left, [0.5,0.5]=center (default)
  occlude: boolean            // Hide when behind other geometry (default: false)
  distanceFactor: number | null // Scale by 1/distance (null = no scaling, default: null)
  zIndex: 'auto' | number    // CSS z-index (default: 'auto')

  // Visibility state (readonly)
  behindCamera: boolean       // Is the anchor behind the camera?
  screenPosition: Vec2        // Current screen coordinates in pixels

  // Lifecycle
  dispose(): void             // Remove element from overlay container
}
```

### Projection Pipeline

Each frame, after the scene renders, all `HtmlAnchor` nodes are projected to screen space:

```typescript
const updateOverlays = (
  anchors: HtmlAnchor[],
  camera: Camera,
  canvasWidth: number,
  canvasHeight: number,
) => {
  const vp = camera.viewProjectionMatrix

  for (const anchor of anchors) {
    if (!anchor.visible) {
      anchor.element.style.display = 'none'
      continue
    }

    // 1. Get world position
    const worldPos = anchor.worldPosition

    // 2. Project to clip space
    const clipX = vp[0]*worldPos[0] + vp[4]*worldPos[1] + vp[8]*worldPos[2] + vp[12]
    const clipY = vp[1]*worldPos[0] + vp[5]*worldPos[1] + vp[9]*worldPos[2] + vp[13]
    const clipW = vp[3]*worldPos[0] + vp[7]*worldPos[1] + vp[11]*worldPos[2] + vp[15]

    // 3. Check if behind camera
    anchor.behindCamera = clipW <= 0
    if (anchor.behindCamera) {
      anchor.element.style.display = 'none'
      continue
    }

    // 4. Perspective divide to NDC
    const ndcX = clipX / clipW
    const ndcY = clipY / clipW

    // 5. NDC to screen pixels
    const screenX = (ndcX * 0.5 + 0.5) * canvasWidth
    const screenY = (1 - (ndcY * 0.5 + 0.5)) * canvasHeight  // Flip Y

    // 6. Off-screen culling
    const margin = 100
    if (screenX < -margin || screenX > canvasWidth + margin ||
        screenY < -margin || screenY > canvasHeight + margin) {
      anchor.element.style.display = 'none'
      continue
    }

    anchor.screenPosition[0] = screenX
    anchor.screenPosition[1] = screenY

    // 7. Center offset (percentage-based via CSS translate)
    const offsetX = -anchor.center[0] * 100
    const offsetY = -anchor.center[1] * 100

    // 8. Distance scaling
    let scale = 1
    if (anchor.distanceFactor !== null) {
      const dist = vec3Distance(worldPos, camera.worldPosition)
      scale = anchor.distanceFactor / dist
    }

    // 9. Update DOM via CSS transform (GPU-composited, no layout/paint)
    anchor.element.style.display = ''
    anchor.element.style.transform =
      `translate(${screenX}px, ${screenY}px) translate(${offsetX}%, ${offsetY}%) scale(${scale})`

    // 10. Depth-based z-index
    if (anchor.zIndex === 'auto') {
      const ndcZ = (vp[2]*worldPos[0] + vp[6]*worldPos[1] + vp[10]*worldPos[2] + vp[14]) / clipW
      anchor.element.style.zIndex = String(Math.round((1 - ndcZ) * 10000))
    } else {
      anchor.element.style.zIndex = String(anchor.zIndex)
    }
  }
}
```

### Occlusion

When `occlude: true`, the anchor is hidden when another object is between it and the camera.

**Option A: Raycasting (precise, expensive)**
```typescript
const checkOcclusion = (anchor: HtmlAnchor, camera: Camera, scene: Scene) => {
  const dir = vec3Normalize(vec3Create(), vec3Sub(vec3Create(), anchor.worldPosition, camera.worldPosition))
  const dist = vec3Distance(anchor.worldPosition, camera.worldPosition)

  raycaster.set(camera.worldPosition, dir)
  const hits = raycaster.intersectObjects(scene.children, true)

  if (hits.length > 0 && hits[0].distance < dist - 0.1) {
    anchor.element.style.display = 'none'
  }
}
```

**Option B: Depth buffer readback (cheaper for many anchors)**
```typescript
// Read depth at the projected pixel and compare with anchor's NDC depth
const sceneDepth = readDepthAtPixel(screenX, screenY)
if (sceneDepth < anchorNdcZ) {
  anchor.element.style.display = 'none'  // Occluded
}
```

Depth buffer readback has 1-frame latency but is O(1) per anchor vs. full BVH traversal.

## Performance Optimizations

1. **Transform-only updates**: `transform: translate(...)` is GPU-composited by the browser. No layout or paint — only composite. This is the cheapest way to move DOM elements.

2. **Batch DOM updates**: All DOM writes happen in a single render loop pass — no interleaved reads/writes that trigger reflow.

3. **Visibility culling**: Off-screen anchors are set to `display: none` to avoid compositing cost.

4. **Element pooling**: For dynamic content (damage numbers, hit markers), pre-create a pool of DOM elements and reuse them rather than creating/destroying per event.

## API

### Imperative

```typescript
const label = new HtmlAnchor({ center: [0.5, 1] }) // Center-bottom anchor

label.element.innerHTML = `
  <div class="health-bar">
    <div class="fill" style="width: 80%"></div>
  </div>
`
label.element.style.pointerEvents = 'auto'

label.position.set(0, 0, 2.5)  // 2.5m above ground
characterGroup.add(label)

// Update health
const setHealth = (hp: number) => {
  label.element.querySelector('.fill')!.style.width = `${hp}%`
}
```

### React

```tsx
import { Html } from '@shark3d/react'

const HealthBar = ({ hp, position }: { hp: number, position: [number, number, number] }) => (
  <Html position={position} center={[0.5, 1]}>
    <div className="health-bar">
      <div className="fill" style={{ width: `${hp}%` }} />
    </div>
  </Html>
)

const Character = ({ hp }: { hp: number }) => (
  <group>
    <mesh>
      <characterGeometry />
      <lambertMaterial />
    </mesh>
    <HealthBar hp={hp} position={[0, 0, 2.5]} />
  </group>
)
```

The React `<Html>` component:
1. Creates an `HtmlAnchor` in the scene graph
2. Uses a React portal to render children into the anchor's DOM element
3. Cleans up both the scene node and DOM element on unmount
4. Reactively updates position, center, occlude when props change
