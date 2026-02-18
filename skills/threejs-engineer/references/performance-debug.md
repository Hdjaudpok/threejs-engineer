# Performance & Debugging

## Performance Rules of Thumb

### Pixel Ratio
Always cap: `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))`

On high-DPI screens (3x, 4x), rendering at full resolution is extremely costly. Capping at 2x is visually indistinguishable on most devices.

### Draw Calls
- **Target**: < 100 draw calls for mobile, < 300 for desktop
- Each unique material + geometry combination = 1 draw call
- Monitor with `renderer.info.render.calls`

### Reducing Draw Calls

#### InstancedMesh
For many identical objects (trees, particles, tiles):

```ts
const count = 1000
const instancedMesh = new THREE.InstancedMesh(geometry, material, count)

const matrix = new THREE.Matrix4()
const position = new THREE.Vector3()
const rotation = new THREE.Euler()
const quaternion = new THREE.Quaternion()
const scale = new THREE.Vector3(1, 1, 1)

for (let i = 0; i < count; i++) {
  position.set(Math.random() * 100 - 50, 0, Math.random() * 100 - 50)
  quaternion.setFromEuler(rotation.set(0, Math.random() * Math.PI * 2, 0))
  matrix.compose(position, quaternion, scale)
  instancedMesh.setMatrixAt(i, matrix)
}
instancedMesh.instanceMatrix.needsUpdate = true
scene.add(instancedMesh)
```

#### Geometry Merging
For static objects with the same material:

```ts
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js'

const geometries: THREE.BufferGeometry[] = []
// ... collect geometries with applied transforms
const mergedGeometry = mergeGeometries(geometries)
const mergedMesh = new THREE.Mesh(mergedGeometry, sharedMaterial)
```

### Textures
- Use power-of-two sizes (512, 1024, 2048)
- Limit max resolution: 2048x2048 for most textures, 4096 only when critical
- Use compressed formats (KTX2/Basis) for production
- Reuse textures across materials when possible
- Set `texture.generateMipmaps = false` if not needed (e.g., UI textures)

### Lights
- Limit dynamic lights: 1-3 max for mobile
- Bake lighting into lightmaps when possible
- Use `AmbientLight` + 1 `DirectionalLight` as baseline
- Avoid `PointLight` / `SpotLight` with shadows (expensive)
- `RectAreaLight` does not support shadows

### Shadows
- Use only when visually necessary
- Limit shadow-casting lights to 1-2
- Tune `shadow.camera` frustum tightly around the scene
- Shadow map sizes: 1024 for mobile, 2048 for desktop
- Consider `PCFShadowMap` over `PCFSoftShadowMap` for mobile
- Use baked shadows (texture on ground plane) as alternative

### Post-Processing
- Each pass = 1 full-screen render
- Limit to 2-3 passes total
- Combine effects when possible
- Consider cheaper alternatives:
  - Bloom → emissive materials + selective bloom
  - SSAO → baked AO maps
  - DOF → simple blur shader on distance
- Use half-resolution for expensive passes

## Quality Toggle Pattern

```ts
type QualityLevel = 'low' | 'high'

function applyQuality(level: QualityLevel) {
  if (level === 'low') {
    renderer.setPixelRatio(1)
    renderer.shadowMap.enabled = false
    bloomPass.enabled = false
    // Reduce particle count, texture resolution, etc.
  } else {
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
    renderer.shadowMap.enabled = true
    bloomPass.enabled = true
  }
}

// Auto-detect: drop quality if FPS < threshold
let frameCount = 0
let lastTime = performance.now()

function checkPerformance() {
  frameCount++
  const now = performance.now()
  if (now - lastTime >= 2000) { // check every 2 seconds
    const fps = (frameCount * 1000) / (now - lastTime)
    if (fps < 30) applyQuality('low')
    frameCount = 0
    lastTime = now
  }
}
```

## Debug Tools

### renderer.info

```ts
// Log in tick (throttled)
if (frameCount % 60 === 0) {
  console.log('Draw calls:', renderer.info.render.calls)
  console.log('Triangles:', renderer.info.render.triangles)
  console.log('Geometries:', renderer.info.memory.geometries)
  console.log('Textures:', renderer.info.memory.textures)
}
```

### stats-gl (FPS/GPU panel)

```ts
import Stats from 'stats-gl'

const stats = new Stats()
document.body.appendChild(stats.dom)

// In tick:
stats.begin()
renderer.render(scene, camera)
stats.end()
```

### lil-gui (Debug UI)

```ts
import GUI from 'lil-gui'

const gui = new GUI()

// Tweak values live
gui.add(material, 'metalness', 0, 1, 0.01)
gui.add(material, 'roughness', 0, 1, 0.01)
gui.addColor(params, 'color').onChange((v) => material.color.set(v))
gui.add(light, 'intensity', 0, 5, 0.1)
gui.add(light.position, 'x', -10, 10, 0.1)

// Shader uniforms
gui.add(shaderMaterial.uniforms.u_intensity, 'value', 0, 2, 0.01).name('intensity')

// Only show in dev
if (import.meta.env.PROD) gui.hide()
```

### Spector.js
For deep GPU profiling:
- Install browser extension
- Capture a frame to inspect all draw calls, shader programs, textures, state changes
- Identify redundant state changes and expensive operations

### React Three Fiber: r3f-perf

```tsx
import { Perf } from 'r3f-perf'

function Scene() {
  return (
    <>
      <Perf position="top-left" />
      {/* scene content */}
    </>
  )
}
```

## Common Performance Bottlenecks

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Low FPS, high GPU usage | Too many pixels (high DPR) | Cap pixelRatio to 2 |
| Low FPS, many draw calls | Too many unique meshes | Use InstancedMesh or merge |
| Stuttering on load | Large textures loading | Compress (KTX2), reduce size |
| Shadows flickering | Shadow frustum too large | Tighten shadow camera bounds |
| Post-processing sluggish | Too many passes | Reduce passes, use half-res |
| Mobile black screen | WebGL context lost | Reduce memory usage, handle `webglcontextlost` event |
| Janky scroll animations | Layout thrashing | Use `will-change: transform`, separate 3D from DOM updates |

## Disposal Pattern

Prevent memory leaks when removing objects:

```ts
function disposeObject(object: THREE.Object3D) {
  object.traverse((child) => {
    if (child instanceof THREE.Mesh) {
      child.geometry.dispose()
      if (Array.isArray(child.material)) {
        child.material.forEach((m) => m.dispose())
      } else {
        child.material.dispose()
      }
    }
  })
  object.parent?.remove(object)
}

// Dispose textures
texture.dispose()

// Dispose render targets
renderTarget.dispose()
```
