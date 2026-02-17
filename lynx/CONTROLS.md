# Orbit Controls

## Overview

Orbit controls provide intuitive camera manipulation for 3D scenes: rotate around a target point, zoom in/out, and pan. This is the primary camera interaction mode for games with a third-person or inspection-style camera.

## Interaction Model

| Input | Action | Description |
|---|---|---|
| Left mouse drag / single touch drag | Orbit | Rotate camera around target |
| Right mouse drag / two-finger drag | Pan | Move target + camera laterally |
| Scroll wheel / pinch | Zoom | Move camera closer/farther from target |
| Middle mouse drag | Pan | Alternative pan input |

## Core State

```typescript
interface OrbitControlsConfig {
    target: Float32Array          // [x, y, z] — point to orbit around
    minDistance: number            // minimum zoom distance (default 0.1)
    maxDistance: number            // maximum zoom distance (default Infinity)
    minPolarAngle: number         // minimum vertical angle in radians (default 0 — straight up Z)
    maxPolarAngle: number         // maximum vertical angle in radians (default π — straight down)
    minAzimuthAngle: number       // minimum horizontal angle (default -Infinity — no limit)
    maxAzimuthAngle: number       // maximum horizontal angle (default Infinity — no limit)
    enableDamping: boolean        // smooth deceleration (default true)
    dampingFactor: number         // damping strength (default 0.08)
    rotateSpeed: number           // orbit sensitivity (default 1.0)
    panSpeed: number              // pan sensitivity (default 1.0)
    zoomSpeed: number             // zoom sensitivity (default 1.0)
    enableRotate: boolean         // allow orbiting (default true)
    enablePan: boolean            // allow panning (default true)
    enableZoom: boolean           // allow zooming (default true)
}
```

## Coordinate System

Lynx uses **Z-up, right-handed**. The orbit controls use spherical coordinates where:

- **Azimuth angle (θ)**: rotation around the Z axis (horizontal orbit)
- **Polar angle (φ)**: angle from the Z+ axis (vertical orbit). φ = 0 means looking straight down, φ = π/2 means at horizon level.
- **Radius (r)**: distance from target

```
       Z (up)
       │
       │   · camera at (r, θ, φ)
       │  /
       │ /  φ (polar — angle from Z+)
       │/
       ├───────── Y
      /  θ (azimuth — rotation around Z)
     /
    X
```

### Spherical to Cartesian (Z-up)

```typescript
const sphericalToCartesian = (
    radius: number,
    azimuth: number,   // θ — around Z
    polar: number,     // φ — from Z+
    out: Float32Array,
): void => {
    const sinPolar = Math.sin(polar)
    out[0] = radius * sinPolar * Math.cos(azimuth)  // X
    out[1] = radius * sinPolar * Math.sin(azimuth)  // Y
    out[2] = radius * Math.cos(polar)                // Z
}
```

## Implementation

```typescript
interface OrbitControls {
    update: (dt: number) => void
    dispose: () => void
    target: Float32Array
    config: OrbitControlsConfig
}

const createOrbitControls = (
    camera: Camera,
    domElement: HTMLElement,
    config?: Partial<OrbitControlsConfig>,
): OrbitControls => {
    const state: OrbitControlsConfig = {
        target: new Float32Array([0, 0, 0]),
        minDistance: 0.1,
        maxDistance: Infinity,
        minPolarAngle: 0.01,          // avoid gimbal lock at poles
        maxPolarAngle: Math.PI - 0.01,
        minAzimuthAngle: -Infinity,
        maxAzimuthAngle: Infinity,
        enableDamping: true,
        dampingFactor: 0.08,
        rotateSpeed: 1.0,
        panSpeed: 1.0,
        zoomSpeed: 1.0,
        enableRotate: true,
        enablePan: true,
        enableZoom: true,
        ...config,
    }

    // Current spherical coordinates
    const offset = new Float32Array(3)
    vec3Sub(offset, camera.position, state.target)

    let radius = vec3Length(offset)
    let azimuth = Math.atan2(offset[1], offset[0])
    let polar = Math.acos(Math.max(-1, Math.min(1, offset[2] / radius)))

    // Delta accumulators (for damping)
    let azimuthDelta = 0
    let polarDelta = 0
    let radiusDelta = 1 // multiplicative
    const panDelta = new Float32Array(3)

    // Input state
    let pointerDown = false
    let pointerButton = 0
    let lastX = 0
    let lastY = 0

    // Touch state
    let touchCount = 0
    let pinchStartDist = 0
    let lastTouchX = 0
    let lastTouchY = 0

    const onPointerDown = (e: PointerEvent) => {
        pointerDown = true
        pointerButton = e.button
        lastX = e.clientX
        lastY = e.clientY
        domElement.setPointerCapture(e.pointerId)
    }

    const onPointerMove = (e: PointerEvent) => {
        if (!pointerDown) return

        const dx = e.clientX - lastX
        const dy = e.clientY - lastY
        lastX = e.clientX
        lastY = e.clientY

        const el = domElement
        if (pointerButton === 0 && state.enableRotate) {
            // Left button: orbit
            azimuthDelta -= (dx / el.clientWidth) * Math.PI * 2 * state.rotateSpeed
            polarDelta -= (dy / el.clientHeight) * Math.PI * state.rotateSpeed
        } else if ((pointerButton === 2 || pointerButton === 1) && state.enablePan) {
            // Right/middle button: pan
            const panScale = 2 * radius * Math.tan(camera.fov * 0.5 * Math.PI / 180) / el.clientHeight
            // Pan in camera-local X and Z-up plane
            const right = new Float32Array(3)
            const up = new Float32Array([0, 0, 1]) // world up
            vec3Cross(right, camera.forward, up)
            vec3Normalize(right, right)
            const camUp = new Float32Array(3)
            vec3Cross(camUp, right, camera.forward)
            vec3Normalize(camUp, camUp)

            panDelta[0] += -dx * panScale * right[0] * state.panSpeed + dy * panScale * camUp[0] * state.panSpeed
            panDelta[1] += -dx * panScale * right[1] * state.panSpeed + dy * panScale * camUp[1] * state.panSpeed
            panDelta[2] += -dx * panScale * right[2] * state.panSpeed + dy * panScale * camUp[2] * state.panSpeed
        }
    }

    const onPointerUp = (e: PointerEvent) => {
        pointerDown = false
        domElement.releasePointerCapture(e.pointerId)
    }

    const onWheel = (e: WheelEvent) => {
        if (!state.enableZoom) return
        e.preventDefault()
        if (e.deltaY > 0) {
            radiusDelta *= 1.0 + 0.05 * state.zoomSpeed
        } else {
            radiusDelta *= 1.0 - 0.05 * state.zoomSpeed
        }
    }

    // Touch handlers for mobile
    const onTouchStart = (e: TouchEvent) => {
        touchCount = e.touches.length
        if (touchCount === 1) {
            lastTouchX = e.touches[0].clientX
            lastTouchY = e.touches[0].clientY
        } else if (touchCount === 2) {
            const dx = e.touches[1].clientX - e.touches[0].clientX
            const dy = e.touches[1].clientY - e.touches[0].clientY
            pinchStartDist = Math.sqrt(dx * dx + dy * dy)
            lastTouchX = (e.touches[0].clientX + e.touches[1].clientX) / 2
            lastTouchY = (e.touches[0].clientY + e.touches[1].clientY) / 2
        }
    }

    const onTouchMove = (e: TouchEvent) => {
        e.preventDefault()
        if (touchCount === 1 && state.enableRotate) {
            const dx = e.touches[0].clientX - lastTouchX
            const dy = e.touches[0].clientY - lastTouchY
            lastTouchX = e.touches[0].clientX
            lastTouchY = e.touches[0].clientY
            azimuthDelta -= (dx / domElement.clientWidth) * Math.PI * 2 * state.rotateSpeed
            polarDelta -= (dy / domElement.clientHeight) * Math.PI * state.rotateSpeed
        } else if (touchCount === 2) {
            // Pinch zoom
            if (state.enableZoom) {
                const dx = e.touches[1].clientX - e.touches[0].clientX
                const dy = e.touches[1].clientY - e.touches[0].clientY
                const dist = Math.sqrt(dx * dx + dy * dy)
                radiusDelta *= pinchStartDist / dist
                pinchStartDist = dist
            }
            // Two-finger pan
            if (state.enablePan) {
                const cx = (e.touches[0].clientX + e.touches[1].clientX) / 2
                const cy = (e.touches[0].clientY + e.touches[1].clientY) / 2
                const pdx = cx - lastTouchX
                const pdy = cy - lastTouchY
                lastTouchX = cx
                lastTouchY = cy
                const panScale = 2 * radius * Math.tan(camera.fov * 0.5 * Math.PI / 180) / domElement.clientHeight
                // Simplified pan for touch
                panDelta[0] += -pdx * panScale * state.panSpeed
                panDelta[1] += -pdy * panScale * state.panSpeed
            }
        }
    }

    // Register events
    domElement.addEventListener('pointerdown', onPointerDown)
    domElement.addEventListener('pointermove', onPointerMove)
    domElement.addEventListener('pointerup', onPointerUp)
    domElement.addEventListener('wheel', onWheel, { passive: false })
    domElement.addEventListener('touchstart', onTouchStart, { passive: true })
    domElement.addEventListener('touchmove', onTouchMove, { passive: false })
    domElement.addEventListener('contextmenu', (e: Event) => e.preventDefault())

    const update = (_dt: number) => {
        if (state.enableDamping) {
            azimuth += azimuthDelta * state.dampingFactor
            polar += polarDelta * state.dampingFactor
            radius *= 1 + (radiusDelta - 1) * state.dampingFactor
            state.target[0] += panDelta[0] * state.dampingFactor
            state.target[1] += panDelta[1] * state.dampingFactor
            state.target[2] += panDelta[2] * state.dampingFactor

            azimuthDelta *= (1 - state.dampingFactor)
            polarDelta *= (1 - state.dampingFactor)
            radiusDelta = 1 + (radiusDelta - 1) * (1 - state.dampingFactor)
            panDelta[0] *= (1 - state.dampingFactor)
            panDelta[1] *= (1 - state.dampingFactor)
            panDelta[2] *= (1 - state.dampingFactor)
        } else {
            azimuth += azimuthDelta
            polar += polarDelta
            radius *= radiusDelta
            state.target[0] += panDelta[0]
            state.target[1] += panDelta[1]
            state.target[2] += panDelta[2]

            azimuthDelta = 0
            polarDelta = 0
            radiusDelta = 1
            panDelta[0] = panDelta[1] = panDelta[2] = 0
        }

        // Clamp
        polar = Math.max(state.minPolarAngle, Math.min(state.maxPolarAngle, polar))
        azimuth = Math.max(state.minAzimuthAngle, Math.min(state.maxAzimuthAngle, azimuth))
        radius = Math.max(state.minDistance, Math.min(state.maxDistance, radius))

        // Update camera position
        sphericalToCartesian(radius, azimuth, polar, offset)
        camera.position[0] = state.target[0] + offset[0]
        camera.position[1] = state.target[1] + offset[1]
        camera.position[2] = state.target[2] + offset[2]

        // Look at target
        camera.lookAt(state.target)
    }

    const dispose = () => {
        domElement.removeEventListener('pointerdown', onPointerDown)
        domElement.removeEventListener('pointermove', onPointerMove)
        domElement.removeEventListener('pointerup', onPointerUp)
        domElement.removeEventListener('wheel', onWheel)
        domElement.removeEventListener('touchstart', onTouchStart)
        domElement.removeEventListener('touchmove', onTouchMove)
    }

    return { update, dispose, target: state.target, config: state }
}
```

## Usage

```typescript
const camera = createPerspectiveCamera({ fov: 60, near: 0.1, far: 500 })
camera.position.set(5, 5, 3)

const controls = createOrbitControls(camera, renderer.canvas, {
    target: new Float32Array([0, 0, 1]),
    enableDamping: true,
    dampingFactor: 0.08,
    minDistance: 2,
    maxDistance: 50,
    maxPolarAngle: Math.PI * 0.45, // prevent going below ground
})

// In render loop
const animate = (dt: number) => {
    controls.update(dt)
    renderer.render(scene, camera)
    requestAnimationFrame(animate)
}
```

## Integration with React

```tsx
// In lynx-react, controls are a component
const App = () => (
    <Canvas>
        <OrbitControls
            target={[0, 0, 1]}
            minDistance={2}
            maxDistance={50}
            enableDamping
        />
        <mesh>
            <boxGeometry />
            <lambertMaterial color="tomato" />
        </mesh>
    </Canvas>
)
```
