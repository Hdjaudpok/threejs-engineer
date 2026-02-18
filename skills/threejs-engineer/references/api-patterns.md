# Three.js Base API Patterns & Architecture

## Minimal Scene Setup (Vanilla TS)

```ts
// src/main.ts
import * as THREE from 'three'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'

// Canvas
const canvas = document.querySelector<HTMLCanvasElement>('#webgl')!

// Scene
const scene = new THREE.Scene()

// Camera
const camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 100)
camera.position.set(0, 2, 5)
scene.add(camera)

// Renderer
const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true })
renderer.setSize(window.innerWidth, window.innerHeight)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
renderer.outputColorSpace = THREE.SRGBColorSpace
renderer.toneMapping = THREE.ACESFilmicToneMapping
renderer.toneMappingExposure = 1.0

// Controls
const controls = new OrbitControls(camera, canvas)
controls.enableDamping = true

// Resize
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})

// Animation loop
const clock = new THREE.Clock()
function tick() {
  const delta = clock.getDelta()
  const elapsed = clock.getElapsedTime()

  controls.update()
  // Update animations, mixers, shader uniforms here

  renderer.render(scene, camera)
  requestAnimationFrame(tick)
}
tick()
```

## Experience Class Pattern

For complex projects, centralize everything in an Experience class:

```ts
// src/Experience.ts
import * as THREE from 'three'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'

export class Experience {
  canvas: HTMLCanvasElement
  scene: THREE.Scene
  camera: THREE.PerspectiveCamera
  renderer: THREE.WebGLRenderer
  controls: OrbitControls
  clock: THREE.Clock
  sizes: { width: number; height: number; pixelRatio: number }

  constructor(canvas: HTMLCanvasElement) {
    this.canvas = canvas
    this.sizes = {
      width: window.innerWidth,
      height: window.innerHeight,
      pixelRatio: Math.min(window.devicePixelRatio, 2),
    }

    this.scene = new THREE.Scene()
    this.camera = this.setupCamera()
    this.renderer = this.setupRenderer()
    this.controls = new OrbitControls(this.camera, this.canvas)
    this.controls.enableDamping = true
    this.clock = new THREE.Clock()

    this.setupLights()
    this.setupEnvironment()
    this.setupModels()

    window.addEventListener('resize', this.onResize.bind(this))
    this.tick()
  }

  private setupCamera(): THREE.PerspectiveCamera {
    const camera = new THREE.PerspectiveCamera(45, this.sizes.width / this.sizes.height, 0.1, 100)
    camera.position.set(0, 2, 5)
    this.scene.add(camera)
    return camera
  }

  private setupRenderer(): THREE.WebGLRenderer {
    const renderer = new THREE.WebGLRenderer({ canvas: this.canvas, antialias: true, alpha: true })
    renderer.setSize(this.sizes.width, this.sizes.height)
    renderer.setPixelRatio(this.sizes.pixelRatio)
    renderer.outputColorSpace = THREE.SRGBColorSpace
    renderer.toneMapping = THREE.ACESFilmicToneMapping
    renderer.shadowMap.enabled = true
    renderer.shadowMap.type = THREE.PCFSoftShadowMap
    return renderer
  }

  private setupLights(): void {
    const ambient = new THREE.AmbientLight(0xffffff, 0.2)
    const directional = new THREE.DirectionalLight(0xffffff, 1.0)
    directional.position.set(5, 10, 5)
    directional.castShadow = true
    directional.shadow.mapSize.set(2048, 2048)
    directional.shadow.camera.near = 0.1
    directional.shadow.camera.far = 50
    this.scene.add(ambient, directional)
  }

  private setupEnvironment(): void {
    // Background, environment map, fog
    this.scene.fog = new THREE.Fog(0x000000, 10, 50)
  }

  private setupModels(): void {
    // Load glTF/GLB models, setup instancing, etc.
  }

  private onResize(): void {
    this.sizes.width = window.innerWidth
    this.sizes.height = window.innerHeight
    this.sizes.pixelRatio = Math.min(window.devicePixelRatio, 2)

    this.camera.aspect = this.sizes.width / this.sizes.height
    this.camera.updateProjectionMatrix()
    this.renderer.setSize(this.sizes.width, this.sizes.height)
    this.renderer.setPixelRatio(this.sizes.pixelRatio)
  }

  private tick(): void {
    const delta = this.clock.getDelta()

    this.controls.update()
    this.renderer.render(this.scene, this.camera)
    requestAnimationFrame(this.tick.bind(this))
  }

  dispose(): void {
    window.removeEventListener('resize', this.onResize)
    this.renderer.dispose()
    this.controls.dispose()
  }
}
```

## Recommended File Structure

### Vanilla TS (Vite)

```
src/
├── main.ts                  # Entry point, instantiate Experience
├── Experience.ts            # Central orchestrator
├── world/
│   ├── Environment.ts       # Lights, fog, environment maps
│   ├── Model.ts             # glTF loader, model setup
│   └── Particles.ts         # Particle systems
├── shaders/
│   ├── water/
│   │   ├── vertex.glsl
│   │   └── fragment.glsl
│   └── portal/
│       ├── vertex.glsl
│       └── fragment.glsl
├── utils/
│   ├── ResourceLoader.ts    # Centralized loading manager
│   ├── Debug.ts             # lil-gui or tweakpane integration
│   └── Sizes.ts             # Responsive sizing utility
├── styles/
│   └── main.css
└── index.html
```

### React Three Fiber

```
src/
├── App.tsx
├── components/
│   ├── Canvas3D.tsx         # R3F Canvas wrapper
│   ├── Scene.tsx            # Scene composition
│   ├── Environment.tsx      # drei Environment, lights
│   ├── Model.tsx            # useGLTF model component
│   └── Effects.tsx          # Post-processing (EffectComposer)
├── hooks/
│   ├── useLoadingProgress.ts
│   └── useScrollAnimation.ts
├── shaders/
│   └── ...
├── stores/
│   └── useAppStore.ts       # Zustand store
└── styles/
    └── main.css
```

## Renderer Configuration Reference

| Setting | Default | Notes |
|---------|---------|-------|
| `antialias` | `true` | Disable on mobile for perf |
| `alpha` | `true` | Transparent background support |
| `outputColorSpace` | `SRGBColorSpace` | Correct color display |
| `toneMapping` | `ACESFilmicToneMapping` | Cinematic look |
| `toneMappingExposure` | `1.0` | Adjust brightness |
| `shadowMap.enabled` | `true` | Enable if shadows needed |
| `shadowMap.type` | `PCFSoftShadowMap` | Good quality/perf ratio |
| `pixelRatio` | `min(dpr, 2)` | Always cap for performance |

## Camera Types

- **PerspectiveCamera**: default for 3D scenes, use FOV 35-75 depending on style
- **OrthographicCamera**: isometric views, 2D-like 3D, UI elements

## Controls Reference

| Control | Use Case |
|---------|----------|
| `OrbitControls` | Product viewers, general exploration |
| `TrackballControls` | Full 360 rotation without gimbal lock |
| `DragControls` | Drag objects in scene |
| `TransformControls` | Editor-like transform gizmos |
| `PointerLockControls` | FPS-style navigation |
| `MapControls` | Map/top-down views |

## Loading Models

```ts
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js'

const dracoLoader = new DRACOLoader()
dracoLoader.setDecoderPath('/draco/')

const gltfLoader = new GLTFLoader()
gltfLoader.setDRACOLoader(dracoLoader)

gltfLoader.load('/models/scene.glb', (gltf) => {
  const model = gltf.scene
  model.traverse((child) => {
    if (child instanceof THREE.Mesh) {
      child.castShadow = true
      child.receiveShadow = true
    }
  })
  scene.add(model)

  // If the model has animations
  const mixer = new THREE.AnimationMixer(model)
  gltf.animations.forEach((clip) => {
    mixer.clipAction(clip).play()
  })
  // Update mixer in tick: mixer.update(delta)
})
```

## Post-Processing Setup

```ts
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js'
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js'
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js'
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js'

const composer = new EffectComposer(renderer)
composer.addPass(new RenderPass(scene, camera))

const bloomPass = new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  0.5,  // strength
  0.4,  // radius
  0.85  // threshold
)
composer.addPass(bloomPass)
composer.addPass(new OutputPass())

// In tick: replace renderer.render() with composer.render()
```
