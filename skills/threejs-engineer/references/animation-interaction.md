# Animation & Interaction Patterns

## Animation Loop Best Practices

```ts
const clock = new THREE.Clock()

function tick() {
  const delta = clock.getDelta()      // Time since last frame (seconds)
  const elapsed = clock.getElapsedTime() // Total elapsed time

  // Update animation mixers (glTF animations)
  mixer?.update(delta)

  // Update shader uniforms
  material.uniforms.u_time.value = elapsed

  // Update GSAP (if using manual ticker)
  // gsap.ticker is auto by default

  // Update controls
  controls.update()

  // Render
  renderer.render(scene, camera)
  // or composer.render() for post-processing

  requestAnimationFrame(tick)
}
```

Always use `delta` for frame-rate-independent animations. Multiply movement speeds and rotations by `delta`.

## GSAP + ScrollTrigger Integration

### Setup

```ts
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

gsap.registerPlugin(ScrollTrigger)
```

### Scroll-Driven Camera Movement

```ts
// Camera follows scroll through sections
const sections = document.querySelectorAll('.section')

// Section 1 → move camera forward
gsap.to(camera.position, {
  z: -5,
  scrollTrigger: {
    trigger: sections[0],
    start: 'top top',
    end: 'bottom top',
    scrub: 1, // smooth scrub
  },
})

// Section 2 → rotate camera and change material
gsap.to(camera.rotation, {
  y: Math.PI * 0.5,
  scrollTrigger: {
    trigger: sections[1],
    start: 'top top',
    end: 'bottom top',
    scrub: 1,
  },
})

// Animate material properties
gsap.to(material, {
  opacity: 0,
  scrollTrigger: {
    trigger: sections[2],
    start: 'top center',
    end: 'bottom center',
    scrub: true,
  },
})
```

### Timeline-Based Intro Animation

```ts
const introTimeline = gsap.timeline({ defaults: { ease: 'power2.out' } })

introTimeline
  .from(camera.position, { z: 20, duration: 2 })
  .from(model.scale, { x: 0, y: 0, z: 0, duration: 1.5 }, '-=1')
  .from(model.rotation, { y: Math.PI * 2, duration: 2 }, '-=1.5')
  .to('.overlay', { opacity: 0, duration: 0.8 }, '-=0.5')
```

### Structured Timeline Approach

For narrative/multi-section experiences, structure timelines:

```ts
// Timeline phases
function createIntroTimeline() {
  return gsap.timeline()
    .from(camera.position, { z: 30, duration: 2.5, ease: 'power3.out' })
    .from('.title', { opacity: 0, y: 50, duration: 1 }, '-=1')
}

function createIdleTimeline() {
  return gsap.timeline({ repeat: -1 })
    .to(model.rotation, { y: Math.PI * 0.1, duration: 3, ease: 'sine.inOut' })
    .to(model.rotation, { y: -Math.PI * 0.1, duration: 3, ease: 'sine.inOut' })
}

function createTransitionTimeline(fromSection: number, toSection: number) {
  const tl = gsap.timeline()
  // Transition logic between sections
  return tl
}

// Master timeline
const master = gsap.timeline()
master.add(createIntroTimeline())
master.add(createIdleTimeline(), '+=0.5')
```

## Raycaster Interactions

### Hover & Click Detection

```ts
const raycaster = new THREE.Raycaster()
const pointer = new THREE.Vector2()
const interactiveObjects: THREE.Object3D[] = [] // populate with clickable meshes

let hoveredObject: THREE.Object3D | null = null

function onPointerMove(event: PointerEvent) {
  pointer.x = (event.clientX / window.innerWidth) * 2 - 1
  pointer.y = -(event.clientY / window.innerHeight) * 2 + 1
}

function onPointerDown(event: PointerEvent) {
  raycaster.setFromCamera(pointer, camera)
  const intersects = raycaster.intersectObjects(interactiveObjects, true)

  if (intersects.length > 0) {
    const target = intersects[0].object
    handleClick(target)
  }
}

// Check hover in animation loop
function checkHover() {
  raycaster.setFromCamera(pointer, camera)
  const intersects = raycaster.intersectObjects(interactiveObjects, true)

  if (intersects.length > 0) {
    const target = intersects[0].object
    if (hoveredObject !== target) {
      if (hoveredObject) handleHoverOut(hoveredObject)
      hoveredObject = target
      handleHoverIn(target)
      document.body.style.cursor = 'pointer'
    }
  } else {
    if (hoveredObject) {
      handleHoverOut(hoveredObject)
      hoveredObject = null
      document.body.style.cursor = 'default'
    }
  }
}

window.addEventListener('pointermove', onPointerMove)
window.addEventListener('pointerdown', onPointerDown)
```

### Hover/Click Handlers

```ts
function handleHoverIn(object: THREE.Object3D) {
  if (object instanceof THREE.Mesh) {
    gsap.to(object.scale, { x: 1.1, y: 1.1, z: 1.1, duration: 0.3 })
    // or change material emissive
    ;(object.material as THREE.MeshStandardMaterial).emissive.setHex(0x333333)
  }
}

function handleHoverOut(object: THREE.Object3D) {
  if (object instanceof THREE.Mesh) {
    gsap.to(object.scale, { x: 1, y: 1, z: 1, duration: 0.3 })
    ;(object.material as THREE.MeshStandardMaterial).emissive.setHex(0x000000)
  }
}

function handleClick(object: THREE.Object3D) {
  // Dispatch custom event or call callback
  const event = new CustomEvent('object-click', { detail: { object } })
  document.dispatchEvent(event)
}
```

## Canvas Integration Layouts

### Fullscreen Canvas (Behind Content)

```css
.webgl-canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: -1;
}

.content {
  position: relative;
  z-index: 1;
  pointer-events: auto;
}
```

### Embedded Canvas (In Layout)

```css
.canvas-container {
  width: 100%;
  height: 60vh;
  position: relative;
}

.canvas-container canvas {
  width: 100%;
  height: 100%;
  display: block;
}
```

Resize observer for embedded canvases:

```ts
const resizeObserver = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const { width, height } = entry.contentRect
    camera.aspect = width / height
    camera.updateProjectionMatrix()
    renderer.setSize(width, height)
  }
})
resizeObserver.observe(container)
```

### Synchronized Sections (Scroll-Driven)

```html
<canvas id="webgl" class="webgl-canvas"></canvas>

<section class="section" data-scene="intro">
  <h1>Welcome</h1>
</section>

<section class="section" data-scene="product">
  <h2>Product</h2>
  <p>Details...</p>
</section>

<section class="section" data-scene="contact">
  <h2>Contact</h2>
</section>
```

Each section drives camera position, object visibility, and material transitions via ScrollTrigger.

## Simple Event System

For separating 3D logic from business logic:

```ts
class EventEmitter {
  private listeners: Map<string, Set<Function>> = new Map()

  on(event: string, callback: Function) {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set())
    this.listeners.get(event)!.add(callback)
  }

  off(event: string, callback: Function) {
    this.listeners.get(event)?.delete(callback)
  }

  emit(event: string, ...args: any[]) {
    this.listeners.get(event)?.forEach((cb) => cb(...args))
  }
}

// Usage
const events = new EventEmitter()
events.on('model-loaded', (model) => { /* update UI */ })
events.on('section-enter', (index) => { /* trigger animation */ })
```
