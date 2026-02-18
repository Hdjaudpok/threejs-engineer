---
name: Three.js Animation & Controls
description: This skill should be used when the user asks to "animate objects in Three.js", "Three.js AnimationMixer", "play glTF animation", "Three.js animation clips", "Three.js controls", "OrbitControls Three.js", "Three.js GSAP animation", "Three.js tween", "Tween.js Three.js", "Three.js physics", "Rapier with Three.js", "Cannon.js Three.js", "Three.js camera animation", "animate camera Three.js", "Three.js spring animation", "Three.js FPS controls", "PointerLockControls", or "Three.js render loop delta time". Provides expert guidance on AnimationMixer, GSAP, OrbitControls, PointerLockControls, physics integration, and frame-rate-independent animation for Three.js r160+ ESM.
version: 0.1.0
---

# Three.js Animation & Controls

## Overview

This skill covers frame-rate-independent animation in Three.js r160+ ESM: `Clock`-based delta time, `AnimationMixer` for glTF and code-defined clips, GSAP integration, `OrbitControls`, `PointerLockControls`, and physics with Rapier/Cannon-es.

## Clock & Delta Time

Always use `THREE.Clock` to drive time-based animation. Never use fixed offsets like `object.rotation.y += 0.01` — this ties speed to monitor refresh rate.

```js
import * as THREE from 'three';

const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();       // seconds since last frame (≈0.016 at 60fps)
  const elapsed = clock.getElapsedTime(); // total seconds since clock start

  // Frame-rate-independent rotation (1 full rotation per 4 seconds)
  mesh.rotation.y += (Math.PI * 2 / 4) * delta;

  // Oscillation (use elapsed, not delta)
  mesh.position.y = Math.sin(elapsed * 2) * 0.5;

  renderer.render(scene, camera);
}
animate();
```

`clock.getDelta()` returns the interval since the **last** call — always call it once per frame, at the top of `animate()`, and pass `delta` to all systems that need it.

## AnimationMixer — glTF Animations

`AnimationMixer` is the primary system for skeletal, morph-target, and property-track animations loaded from glTF/GLB files.

```js
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('/draco/'); // path to Draco WASM decoder

const loader = new GLTFLoader();
loader.setDRACOLoader(dracoLoader);

let mixer;
loader.load('/model.glb', (gltf) => {
  scene.add(gltf.scene);
  gltf.scene.traverse((node) => {
    if (node.isMesh) {
      node.castShadow = true;
      node.receiveShadow = true;
    }
  });

  mixer = new THREE.AnimationMixer(gltf.scene);

  // List available clips
  console.log(gltf.animations.map(a => a.name));

  // Play a specific clip
  const action = mixer.clipAction(gltf.animations[0]);
  action.play();
});

// In animate():
if (mixer) mixer.update(delta);
```

### AnimationAction Controls

```js
const action = mixer.clipAction(clip);

// Playback
action.play();
action.stop();
action.paused = true;
action.paused = false;

// Speed and direction
action.timeScale = 2.0;   // 2× speed
action.timeScale = -1.0;  // play backwards

// Looping modes
action.setLoop(THREE.LoopRepeat, Infinity); // default
action.setLoop(THREE.LoopOnce, 1);          // play once, stop
action.setLoop(THREE.LoopPingPong, 10);     // back and forth

// When using LoopOnce, reset to start for replay
action.clampWhenFinished = true; // hold last frame after LoopOnce
action.reset().play();           // reset time pointer and play

// Weight for blending
action.weight = 0.5; // blend with another action (0–1)

// Fading between actions
currentAction.fadeOut(0.3);       // fade out over 0.3 seconds
nextAction.reset().fadeIn(0.3).play();

// Cross-fade (synchronised)
currentAction.crossFadeTo(nextAction, 0.5, true);
```

### Programmatic AnimationClip

Create clips entirely in code (no glTF needed):

```js
// Animate position.y from 0 to 3 and back over 2 seconds
const times = [0, 1, 2];
const values = [0, 3, 0]; // y values at each keyframe

const track = new THREE.NumberKeyframeTrack(
  'mesh.position[y]', // path: objectName.property[component]
  times,
  values,
  THREE.InterpolateSmooth // Smooth, Linear, or Discrete
);

// Color animation
const colorTrack = new THREE.ColorKeyframeTrack(
  'mesh.material.color',
  [0, 1, 2],
  [1,0,0, 0,1,0, 0,0,1] // RGB interleaved
);

const clip = new THREE.AnimationClip('bounce', 2, [track, colorTrack]);
const action = mixer.clipAction(clip, mesh);
action.play();
```

For full `KeyframeTrack` type list and path syntax, see `references/animation-clips.md`.

## GSAP Integration

GSAP is the preferred library for UI-driven, timeline-based, and spring animations in Three.js.

```js
import { gsap } from 'gsap';

// Tween a Three.js object's position
gsap.to(mesh.position, {
  x: 5,
  y: 2,
  duration: 1.5,
  ease: 'power3.out',
});

// Tween camera position (with lookAt update)
gsap.to(camera.position, {
  x: 0, y: 5, z: 10,
  duration: 2,
  ease: 'expo.inOut',
  onUpdate: () => camera.lookAt(0, 0, 0),
});

// Tween a uniform (for shader animation)
gsap.to(mat.uniforms.uProgress, {
  value: 1,
  duration: 1.5,
  ease: 'sine.inOut',
});

// Timeline for sequenced animations
const tl = gsap.timeline({ repeat: -1, yoyo: true });
tl.to(mesh.rotation, { y: Math.PI * 2, duration: 3 })
  .to(mesh.scale, { x: 2, y: 2, z: 2, duration: 0.5 }, '-=0.5');

// Spring physics (GSAP Physics2D or Elastic ease)
gsap.to(mesh.position, { x: 3, duration: 1.5, ease: 'elastic.out(1, 0.3)' });
```

GSAP interpolates Three.js object properties directly — no adapter needed. Avoid mixing GSAP and `AnimationMixer` on the **same property** of the same object.

## OrbitControls

```js
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;     // smooth momentum
controls.dampingFactor = 0.05;
controls.enableZoom = true;
controls.minDistance = 2;
controls.maxDistance = 50;
controls.maxPolarAngle = Math.PI / 2; // prevent going below ground
controls.target.set(0, 1, 0);     // orbit around this point
controls.update();                 // call once after setting target

// In animate():
controls.update(); // required when enableDamping = true
```

**Damping** requires `controls.update()` every frame — omitting it freezes the inertia effect.

For `PointerLockControls` (FPS), `FlyControls`, `TransformControls`, and `DragControls`, see `references/controls-guide.md`.

## Physics — Rapier (Recommended)

Rapier is the recommended physics engine for Three.js: deterministic, WASM-accelerated, and actively maintained.

```js
import RAPIER from '@dimforge/rapier3d-compat';
await RAPIER.init();

const gravity = { x: 0.0, y: -9.81, z: 0.0 };
const world = new RAPIER.World(gravity);

// Static ground
const groundBody = world.createRigidBody(
  RAPIER.RigidBodyDesc.fixed()
);
world.createCollider(
  RAPIER.ColliderDesc.cuboid(50, 0.1, 50),
  groundBody
);

// Dynamic sphere
const sphereBody = world.createRigidBody(
  RAPIER.RigidBodyDesc.dynamic().setTranslation(0, 5, 0)
);
world.createCollider(
  RAPIER.ColliderDesc.ball(0.5), // radius matches Three.js sphere
  sphereBody
);

// In animate():
world.step();
const t = sphereBody.translation();
const r = sphereBody.rotation();
sphereMesh.position.set(t.x, t.y, t.z);
sphereMesh.quaternion.set(r.x, r.y, r.z, r.w);
```

Step the world once per animation frame. For fixed-step physics (more stable), accumulate elapsed time and step in fixed increments:

```js
const FIXED_STEP = 1 / 60;
let physicsAccumulator = 0;

function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();
  physicsAccumulator += delta;
  while (physicsAccumulator >= FIXED_STEP) {
    world.step();
    physicsAccumulator -= FIXED_STEP;
  }
  renderer.render(scene, camera);
}
```

For Cannon-es setup and character controller patterns, see `references/controls-guide.md`.

## Tween.js (Lightweight Alternative to GSAP)

```js
import * as TWEEN from '@tweenjs/tween.js';

const tween = new TWEEN.Tween(mesh.position)
  .to({ x: 5, y: 2 }, 1000) // target, duration ms
  .easing(TWEEN.Easing.Quadratic.Out)
  .onUpdate(() => {})
  .start();

// In animate():
TWEEN.update(); // advances all active tweens
```

## Scroll-Based Animation

```js
// Drive animation progress from scroll position
window.addEventListener('scroll', () => {
  const progress = window.scrollY / (document.body.scrollHeight - window.innerHeight);
  mixer.setTime(progress * clip.duration); // scrub animation clip
});
```

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `clock.getDelta()` called multiple times per frame | Call once at top of `animate()`, pass result |
| `controls.update()` missing with `enableDamping` | Add to every animation frame |
| `mixer.update(delta)` not called per frame | Add to animation loop |
| GSAP and AnimationMixer on same property | Pick one; they conflict |
| Physics step not called every frame | `world.step()` must be in animation loop |
| glTF animations reference wrong mixer root | Pass `gltf.scene` (root), not a child node |

## Additional Resources

- **`references/animation-clips.md`** — Full KeyframeTrack type list, path syntax, additive layers, animation events
- **`references/controls-guide.md`** — PointerLockControls (FPS), FlyControls, TransformControls, Cannon-es setup, character controllers
