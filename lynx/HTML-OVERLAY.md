# HTML Overlay System

## Overview

The HTML overlay system renders DOM elements at specific 3D positions on top of the WebGL/WebGPU canvas. This is useful for labels, health bars, tooltips, chat bubbles, and UI elements that are easier to build in HTML/CSS than in 3D.

Similar to three.js CSS3DRenderer or Drei's `<Html>` component, but simpler and more integrated.

## Architecture

```
┌─────────────────────────────────────────┐
│ Browser Viewport                        │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │ Canvas (WebGL/WebGPU)            │   │
│  │                                  │   │
│  │     ┌─3D Scene────────────┐     │   │
│  │     │                     │     │   │
│  │     │    · anchor point   │     │   │
│  │     │                     │     │   │
│  │     └─────────────────────┘     │   │
│  │                                  │   │
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │ Overlay Container (absolute)     │   │
│  │ pointer-events: none             │   │
│  │                                  │   │
│  │         ┌──────────┐            │   │
│  │         │ HTML elem │ ← projected│   │
│  │         │ at screen │   to 2D   │   │
│  │         │ position  │            │   │
│  │         └──────────┘            │   │
│  │                                  │   │
│  └──────────────────────────────────┘   │
│                                         │
└─────────────────────────────────────────┘
```

## 3D to Screen Projection

Given a 3D world position, compute the 2D screen pixel coordinates:

```typescript
const projectToScreen = (
    worldPos: Float32Array,
    viewProjectionMatrix: Float32Array,
    viewportWidth: number,
    viewportHeight: number,
): { x: number, y: number, z: number, visible: boolean } => {
    // Transform to clip space
    const x = worldPos[0] * viewProjectionMatrix[0]
              + worldPos[1] * viewProjectionMatrix[4]
              + worldPos[2] * viewProjectionMatrix[8]
              + viewProjectionMatrix[12]
    const y = worldPos[0] * viewProjectionMatrix[1]
              + worldPos[1] * viewProjectionMatrix[5]
              + worldPos[2] * viewProjectionMatrix[9]
              + viewProjectionMatrix[13]
    const z = worldPos[0] * viewProjectionMatrix[2]
              + worldPos[1] * viewProjectionMatrix[6]
              + worldPos[2] * viewProjectionMatrix[10]
              + viewProjectionMatrix[14]
    const w = worldPos[0] * viewProjectionMatrix[3]
              + worldPos[1] * viewProjectionMatrix[7]
              + worldPos[2] * viewProjectionMatrix[11]
              + viewProjectionMatrix[15]

    // Behind camera
    if (w <= 0) return { x: 0, y: 0, z: 0, visible: false }

    // NDC
    const ndcX = x / w
    const ndcY = y / w
    const ndcZ = z / w

    // Outside frustum
    if (ndcX < -1.2 || ndcX > 1.2 || ndcY < -1.2 || ndcY > 1.2 || ndcZ < -1 || ndcZ > 1) {
        return { x: 0, y: 0, z: ndcZ, visible: false }
    }

    // Screen coordinates (CSS pixels)
    const screenX = (ndcX * 0.5 + 0.5) * viewportWidth
    const screenY = (1 - (ndcY * 0.5 + 0.5)) * viewportHeight // flip Y for CSS

    return { x: screenX, y: screenY, z: ndcZ, visible: true }
}
```

## Overlay Manager

```typescript
interface OverlayItem {
    element: HTMLElement
    worldPosition: Float32Array    // [x, y, z] in world space
    offset: [number, number]       // CSS pixel offset from projected point
    anchor: [number, number]       // [0.5, 0.5] = centered, [0, 0] = top-left
    occlude: boolean               // hide when behind other objects (default false)
    distanceScale: boolean         // scale element based on distance (default false)
    referenceDistance: number       // distance at which scale = 1 (default 10)
    minScale: number               // minimum scale factor (default 0.3)
    maxScale: number               // maximum scale factor (default 2.0)
    zIndexBase: number             // base z-index (closer elements get higher z-index)
    visible: boolean               // manual visibility toggle
}

interface HtmlOverlay {
    container: HTMLDivElement
    add: (config: Partial<OverlayItem> & { element: HTMLElement, worldPosition: Float32Array }) => OverlayItem
    remove: (item: OverlayItem) => void
    update: (camera: Camera, viewportWidth: number, viewportHeight: number) => void
    dispose: () => void
}

const createHtmlOverlay = (parentElement: HTMLElement): HtmlOverlay => {
    // Create overlay container positioned over the canvas
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
    parentElement.style.position = 'relative'
    parentElement.appendChild(container)

    const items: Set<OverlayItem> = new Set()

    const add = (config: Partial<OverlayItem> & { element: HTMLElement, worldPosition: Float32Array }): OverlayItem => {
        const item: OverlayItem = {
            offset: [0, 0],
            anchor: [0.5, 0.5],
            occlude: false,
            distanceScale: false,
            referenceDistance: 10,
            minScale: 0.3,
            maxScale: 2.0,
            zIndexBase: 0,
            visible: true,
            ...config,
        }

        // Setup element styles
        item.element.style.cssText += `
            position: absolute;
            top: 0;
            left: 0;
            will-change: transform;
            pointer-events: auto;
        `

        container.appendChild(item.element)
        items.add(item)
        return item
    }

    const remove = (item: OverlayItem) => {
        container.removeChild(item.element)
        items.delete(item)
    }

    // Pre-allocate for update loop
    const vpMatrix = new Float32Array(16)

    const update = (camera: Camera, viewportWidth: number, viewportHeight: number) => {
        mat4Multiply(vpMatrix, camera.projectionMatrix, camera.viewMatrix)

        for (const item of items) {
            if (!item.visible) {
                item.element.style.display = 'none'
                continue
            }

            const proj = projectToScreen(item.worldPosition, vpMatrix, viewportWidth, viewportHeight)

            if (!proj.visible) {
                item.element.style.display = 'none'
                continue
            }

            item.element.style.display = ''

            // Distance-based scaling
            let scale = 1
            if (item.distanceScale) {
                const dist = vec3Distance(camera.position, item.worldPosition)
                scale = item.referenceDistance / Math.max(dist, 0.01)
                scale = Math.max(item.minScale, Math.min(item.maxScale, scale))
            }

            // Compute final screen position with anchor and offset
            const anchorX = item.element.offsetWidth * item.anchor[0]
            const anchorY = item.element.offsetHeight * item.anchor[1]
            const finalX = proj.x + item.offset[0] - anchorX
            const finalY = proj.y + item.offset[1] - anchorY

            // Use transform for GPU-accelerated positioning (no layout thrashing)
            item.element.style.transform = `translate(${finalX}px, ${finalY}px) scale(${scale})`

            // Depth-based z-index (closer = higher z-index)
            item.element.style.zIndex = String(item.zIndexBase + Math.round((1 - proj.z) * 10000))
        }
    }

    const dispose = () => {
        for (const item of items) {
            container.removeChild(item.element)
        }
        items.clear()
        parentElement.removeChild(container)
    }

    return { container, add, remove, update, dispose }
}
```

## Occlusion (Optional)

When `occlude: true`, the overlay item is hidden when the 3D point is behind another mesh. Two approaches:

### GPU Depth Read (Accurate, Slight Latency)

```typescript
const checkOcclusion = (
    device: GalDevice,
    depthTexture: GalTexture,
    screenX: number,
    screenY: number,
    expectedDepth: number,
): boolean => {
    // Read a single pixel from the depth buffer
    // WebGPU: readback via staging buffer (1 frame latency)
    // WebGL2: readPixels on depth framebuffer
    const actualDepth = device.readDepthPixel(depthTexture, Math.floor(screenX), Math.floor(screenY))
    return expectedDepth > actualDepth + 0.001 // occluded if scene depth is closer
}
```

### Raycaster (CPU, No Latency)

```typescript
const checkOcclusionRaycast = (
    raycaster: Raycaster,
    camera: Camera,
    worldPos: Float32Array,
    meshes: Mesh[],
): boolean => {
    const dir = new Float32Array(3)
    vec3Sub(dir, worldPos, camera.position)
    const dist = vec3Length(dir)
    vec3Normalize(dir, dir)

    const ray = createRay(camera.position, dir)
    const hit = raycaster.intersectFirst(ray, meshes)
    return hit !== null && hit.distance < dist - 0.1
}
```

## Usage Examples

### Basic Label

```typescript
const overlay = createHtmlOverlay(canvasContainer)

// Create a label at a world position
const label = document.createElement('div')
label.textContent = 'Player 1'
label.className = 'player-label'

const item = overlay.add({
    element: label,
    worldPosition: new Float32Array([10, 5, 2]),
    anchor: [0.5, 1.0], // bottom-center aligned
    offset: [0, -10],    // 10px above the point
})

// Update in render loop
const animate = () => {
    controls.update(dt)
    renderer.render(scene, camera)
    overlay.update(camera, canvas.width, canvas.height)
    requestAnimationFrame(animate)
}
```

### Health Bar

```typescript
const healthBar = document.createElement('div')
healthBar.innerHTML = `
    <div style="width: 60px; height: 6px; background: #333; border-radius: 3px;">
        <div class="fill" style="width: 75%; height: 100%; background: #4f4; border-radius: 3px;"></div>
    </div>
`

overlay.add({
    element: healthBar,
    worldPosition: enemy.position, // reference to live position
    anchor: [0.5, 1.0],
    offset: [0, -20],
    distanceScale: true,
    referenceDistance: 15,
})
```

### Dynamic Position (Follow a Node)

```typescript
// Update overlay world position to follow a scene node
const followNode = (item: OverlayItem, node: Node) => {
    // Copy world position from node's world matrix
    item.worldPosition[0] = node.worldMatrix[12]
    item.worldPosition[1] = node.worldMatrix[13]
    item.worldPosition[2] = node.worldMatrix[14]
}

// In render loop:
followNode(labelItem, characterHead)
overlay.update(camera, width, height)
```

## React Integration

```tsx
// In lynx-react
const Html = ({ children, position, anchor, offset, distanceScale, ...props }) => {
    const overlayRef = useOverlay()
    const itemRef = useRef<OverlayItem>()

    useEffect(() => {
        const el = document.createElement('div')
        itemRef.current = overlayRef.add({
            element: el,
            worldPosition: new Float32Array(position),
            anchor,
            offset,
            distanceScale,
        })
        return () => overlayRef.remove(itemRef.current!)
    }, [])

    // Render children into the overlay element via portal
    return itemRef.current
        ? createPortal(children, itemRef.current.element)
        : null
}

// Usage
const App = () => (
    <Canvas>
        <mesh position={[0, 0, 2]}>
            <boxGeometry />
            <lambertMaterial color="orange" />
            <Html position={[0, 0, 1.5]} anchor={[0.5, 1]}>
                <div className="label">Hello World</div>
            </Html>
        </mesh>
    </Canvas>
)
```

## Performance Considerations

- **CSS transforms only**: All positioning uses `transform: translate()` which triggers GPU compositing, not layout/paint.
- **`will-change: transform`**: Hints the browser to promote the element to its own layer.
- **`pointer-events: none`** on container: Prevents the overlay from blocking canvas mouse events. Individual items opt-in with `pointer-events: auto`.
- **Batch DOM reads**: `offsetWidth/offsetHeight` are read once and cached. The update loop only writes styles.
- **Occlusion opt-in**: Depth reading or raycasting is expensive — only enable for items that need it.
- **100+ overlays**: For large numbers of overlays, consider using a single container with `display: none` on off-screen items rather than removing/re-adding DOM nodes.
