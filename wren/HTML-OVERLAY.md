# HTML Overlay System

Renders DOM elements at 3D scene positions, projected onto a 2D overlay div that sits on top of the canvas. Similar to three.js's CSS3DRenderer or drei's `<Html>` component.

## Architecture

```
┌─────────────────────────────────┐
│  Container div (position:relative) │
│  ┌────────────────────────────┐ │
│  │  Canvas (3D scene)         │ │
│  └────────────────────────────┘ │
│  ┌────────────────────────────┐ │
│  │  Overlay div               │ │  ← position:absolute, pointer-events:none
│  │  (same size as canvas)     │ │
│  │  ┌──────┐   ┌──────┐      │ │
│  │  │Label │   │HP bar│      │ │  ← Individual overlays have pointer-events:auto
│  │  └──────┘   └──────┘      │ │
│  └────────────────────────────┘ │
└─────────────────────────────────┘
```

## API

```ts
interface HtmlOverlay {
  container: HTMLDivElement         // The overlay container (created automatically)

  add(element: HTMLElement, options: OverlayOptions): OverlayHandle
  remove(handle: OverlayHandle): void
  update(camera: CameraNode): void  // Call each frame to reproject
  dispose(): void
}

interface OverlayOptions {
  // Position source (one of these):
  position?: Float32Array           // Static world position [x, y, z]
  node?: SceneNode                  // Track a scene node's world position
  bone?: { skeleton: Skeleton, boneName: string }  // Track a bone

  // Projection options
  center?: boolean                  // Center the element on the projected point. Default: true
  occlude?: boolean                 // Hide when behind other objects (via depth). Default: false
  zIndexRange?: [number, number]    // Z-index range for depth ordering. Default: [0, 1000]
  distanceFactor?: number           // If set, scale element by 1/distance * factor
  sprite?: boolean                  // If true, always face camera (no 3D rotation). Default: true
}

interface OverlayHandle {
  element: HTMLElement
  options: OverlayOptions
  visible: boolean
}
```

## Projection

Each frame, every overlay element is projected from 3D world space to 2D screen coordinates:

```ts
const updateOverlays = (overlay: HtmlOverlay, camera: CameraNode): void => {
  const vp = camera.viewProjectionMatrix
  const canvasWidth = overlay.container.clientWidth
  const canvasHeight = overlay.container.clientHeight

  for (const handle of overlay.handles) {
    // Get world position
    let worldPos: Float32Array
    if (handle.options.node) {
      worldPos = getWorldPosition(handle.options.node)
    } else if (handle.options.bone) {
      const bone = handle.options.bone.skeleton.getBoneByName(handle.options.bone.boneName)
      worldPos = getWorldPosition(bone)
    } else {
      worldPos = handle.options.position!
    }

    // Project to clip space
    const clipX = vp[0]*worldPos[0] + vp[4]*worldPos[1] + vp[8]*worldPos[2] + vp[12]
    const clipY = vp[1]*worldPos[0] + vp[5]*worldPos[1] + vp[9]*worldPos[2] + vp[13]
    const clipZ = vp[2]*worldPos[0] + vp[6]*worldPos[1] + vp[10]*worldPos[2] + vp[14]
    const clipW = vp[3]*worldPos[0] + vp[7]*worldPos[1] + vp[11]*worldPos[2] + vp[15]

    // Behind camera
    if (clipW <= 0) {
      handle.element.style.display = 'none'
      handle.visible = false
      continue
    }

    // NDC
    const ndcX = clipX / clipW
    const ndcY = clipY / clipW
    const ndcZ = clipZ / clipW  // 0 to 1 (depth)

    // Screen coordinates
    const screenX = (ndcX * 0.5 + 0.5) * canvasWidth
    const screenY = (1 - (ndcY * 0.5 + 0.5)) * canvasHeight  // Flip Y

    // Apply position
    handle.element.style.display = ''
    handle.element.style.transform = `translate(${screenX}px, ${screenY}px)${
      handle.options.center ? ' translate(-50%, -50%)' : ''
    }`

    // Depth-based z-index
    if (handle.options.zIndexRange) {
      const [min, max] = handle.options.zIndexRange
      handle.element.style.zIndex = String(Math.round(max - ndcZ * (max - min)))
    }

    // Distance-based scaling
    if (handle.options.distanceFactor) {
      const dist = clipW  // W is approximately the distance in view space
      const scale = handle.options.distanceFactor / dist
      handle.element.style.transform += ` scale(${scale})`
    }

    handle.visible = true
  }
}
```

## Occlusion (Optional)

When `occlude: true`, the overlay checks whether the 3D point is behind opaque geometry by reading the depth buffer at the projected screen coordinate:

```ts
// Only works with WebGL2 (readPixels from depth) or WebGPU (buffer readback)
const isOccluded = (ndcZ: number, screenX: number, screenY: number, depthBuffer: Float32Array): boolean => {
  const depthAtPixel = sampleDepthBuffer(depthBuffer, screenX, screenY)
  return ndcZ > depthAtPixel + 0.001  // Small bias
}
```

Depth readback is expensive, so occlusion is opt-in and best limited to a small number of overlays.

## CSS Setup

```ts
const createOverlay = (canvas: HTMLCanvasElement): HtmlOverlay => {
  const container = document.createElement('div')
  container.style.cssText = `
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    overflow: hidden;
  `
  canvas.parentElement!.style.position = 'relative'
  canvas.parentElement!.appendChild(container)

  return {
    container,
    handles: [],
    add: (element, options) => {
      element.style.position = 'absolute'
      element.style.left = '0'
      element.style.top = '0'
      element.style.willChange = 'transform'
      element.style.pointerEvents = 'auto'
      container.appendChild(element)
      const handle = { element, options, visible: true }
      overlay.handles.push(handle)
      return handle
    },
    remove: (handle) => {
      container.removeChild(handle.element)
      overlay.handles.splice(overlay.handles.indexOf(handle), 1)
    },
    update: (camera) => updateOverlays(overlay, camera),
    dispose: () => container.remove(),
  }
}
```

## Usage Example

```ts
// Create a floating label above a character
const label = document.createElement('div')
label.className = 'player-name'
label.textContent = 'Player 1'
label.style.cssText = 'color: white; font-size: 14px; text-shadow: 0 0 4px black;'

overlay.add(label, {
  node: characterMesh,
  center: true,
  distanceFactor: 100,
  zIndexRange: [0, 1000],
})

// Health bar above an enemy, attached to head bone
const healthBar = document.createElement('div')
healthBar.innerHTML = '<div style="width:80%;height:100%;background:red"></div>'
healthBar.style.cssText = 'width:60px;height:6px;background:#333;border-radius:3px;overflow:hidden;'

overlay.add(healthBar, {
  bone: { skeleton: enemySkeleton, boneName: 'head' },
  center: true,
  distanceFactor: 80,
})
```
