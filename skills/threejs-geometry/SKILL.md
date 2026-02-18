---
name: Three.js Geometry & Meshes
description: This skill should be used when the user asks to "create custom geometry in Three.js", "use BufferGeometry in Three.js", "Three.js InstancedMesh", "instanced rendering in Three.js", "add custom vertex attributes Three.js", "Three.js LOD", "level of detail Three.js", "merge geometries in Three.js", "dispose geometry in Three.js", "Three.js indexed geometry", "Three.js morph targets", "Three.js points cloud", "Three.js line geometry", or "Three.js custom mesh". Provides expert guidance on BufferGeometry construction, InstancedMesh, custom vertex attributes, LOD, geometry merging, and disposal for Three.js r160+ ESM.
version: 0.1.0
---

# Three.js Geometry & Meshes

## Overview

This skill covers Three.js geometry from built-in helpers to low-level `BufferGeometry` construction, `InstancedMesh` for mass rendering, LOD, morph targets, and proper disposal. All code targets Three.js r160+ ESM.

## Built-in Geometries

```js
import * as THREE from 'three';

// Most common primitives
new THREE.BoxGeometry(w, h, d, wSegs, hSegs, dSegs);
new THREE.SphereGeometry(radius, widthSegs, heightSegs);
new THREE.PlaneGeometry(w, h, wSegs, hSegs);
new THREE.CylinderGeometry(radiusTop, radiusBottom, height, radialSegs);
new THREE.TorusGeometry(radius, tube, radialSegs, tubularSegs);
new THREE.TorusKnotGeometry(radius, tube, tubularSegs, radialSegs, p, q);
new THREE.ConeGeometry(radius, height, radialSegs);
new THREE.IcosahedronGeometry(radius, detail); // good for sphere alternatives
new THREE.TetrahedronGeometry(radius, detail);
new THREE.OctahedronGeometry(radius, detail);
new THREE.LatheGeometry(points, segments);    // revolution surface
new THREE.ExtrudeGeometry(shape, options);    // extrude 2D Shape
new THREE.TubeGeometry(curve, tubularSegs, radius, radialSegs, closed);
```

Segment counts trade tessellation quality vs draw-call triangle count. For `SphereGeometry`, 32×16 is typical; 64×32 for close-up hero objects.

## BufferGeometry

`BufferGeometry` is the low-level geometry primitive for all custom and procedural meshes.

```js
const geo = new THREE.BufferGeometry();

// Required: positions (Float32Array, 3 values per vertex)
const positions = new Float32Array([
  0, 0, 0,   // vertex 0
  1, 0, 0,   // vertex 1
  0.5, 1, 0, // vertex 2
]);
geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));

// Optional: normals (3 per vertex, must be normalized unit vectors)
const normals = new Float32Array([0,0,1, 0,0,1, 0,0,1]);
geo.setAttribute('normal', new THREE.BufferAttribute(normals, 3));

// Optional: UVs (2 per vertex)
const uvs = new Float32Array([0,0, 1,0, 0.5,1]);
geo.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));

// Optional: vertex colors (3 per vertex, RGB 0–1)
const colors = new Float32Array([1,0,0, 0,1,0, 0,0,1]);
geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));
// Material must have vertexColors: true

// Optional: indexed (reduces vertex count for shared vertices)
geo.setIndex([0, 1, 2]); // Uint16Array for <65k vertices, Uint32Array for more

// Auto-compute normals from positions (flat shading)
geo.computeVertexNormals(); // call after setting positions
```

### Dynamic Geometry (GPU-side update)

Mark attributes `needsUpdate = true` after CPU-side mutation:

```js
const posAttr = geo.getAttribute('position');
// Modify Float32Array values...
posAttr.array[0] = newX;
posAttr.needsUpdate = true; // triggers GPU buffer re-upload

// Optional: hint GPU this buffer updates frequently
posAttr.usage = THREE.DynamicDrawUsage; // set before first upload
```

For frequent full-buffer updates (particle systems, cloth), use `DynamicDrawUsage` and mutate the typed array directly — avoid creating new `BufferAttribute` objects each frame.

For detailed BufferGeometry API and computed attributes, see `references/buffer-geometry-api.md`.

## InstancedMesh

`InstancedMesh` renders thousands of identical meshes in a **single draw call** using instanced rendering. Essential for particles, foliage, crowds, or any repeated object.

```js
const count = 10_000;
const geo = new THREE.SphereGeometry(0.1, 8, 6); // low-poly for instances
const mat = new THREE.MeshStandardMaterial({ color: 0x4488ff });

const mesh = new THREE.InstancedMesh(geo, mat, count);
mesh.castShadow = true;
scene.add(mesh);

// Set per-instance transform via Matrix4
const dummy = new THREE.Object3D();
for (let i = 0; i < count; i++) {
  dummy.position.set(
    (Math.random() - 0.5) * 50,
    Math.random() * 10,
    (Math.random() - 0.5) * 50
  );
  dummy.rotation.set(Math.random() * Math.PI, 0, 0);
  dummy.scale.setScalar(0.5 + Math.random());
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
}
mesh.instanceMatrix.needsUpdate = true; // required after bulk set

// Per-instance color (optional)
mesh.instanceColor = new THREE.InstancedBufferAttribute(
  new Float32Array(count * 3), 3
);
for (let i = 0; i < count; i++) {
  mesh.setColorAt(i, new THREE.Color().setHSL(Math.random(), 0.8, 0.5));
}
mesh.instanceColor.needsUpdate = true;
```

**Performance rules:**
- Keep geometry complexity low for instances (8–16 radial segments for spheres).
- Update `instanceMatrix.needsUpdate = true` only when transforms change.
- For animated instances, update only the changed subset using `setMatrixAt(i, m)` per changed index.
- `InstancedMesh` supports frustum culling per-instance only in Three.js r153+; for older versions, set `mesh.frustumCulled = false` if instances vanish unexpectedly.

For advanced instancing patterns (picking, custom attributes, animated instances), see `references/instancing-patterns.md`.

## Level of Detail (LOD)

```js
import * as THREE from 'three';

const lod = new THREE.LOD();

// High-detail mesh: shown within 5 units
lod.addLevel(
  new THREE.Mesh(new THREE.SphereGeometry(1, 64, 32), mat),
  0   // minimum distance for this level
);
// Medium-detail: 5–20 units
lod.addLevel(
  new THREE.Mesh(new THREE.SphereGeometry(1, 16, 8), mat),
  5
);
// Low-detail: 20–50 units
lod.addLevel(
  new THREE.Mesh(new THREE.SphereGeometry(1, 8, 4), mat),
  20
);
// Billboard or hidden: beyond 50 units
lod.addLevel(new THREE.Object3D(), 50); // empty object = invisible

lod.position.set(0, 1, 0);
scene.add(lod);

// In animation loop — LOD auto-updates based on camera distance
lod.update(camera); // call once per frame
```

LOD levels must be added in ascending distance order. The `update(camera)` call should be in the animation loop.

## Geometry Merging

Merge multiple separate geometries into one draw call with `mergeGeometries`:

```js
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';

const geos = [];
const dummy = new THREE.Object3D();
for (let i = 0; i < 100; i++) {
  const g = new THREE.BoxGeometry(1, 1, 1);
  dummy.position.set(Math.random() * 20, 0, Math.random() * 20);
  dummy.updateMatrix();
  g.applyMatrix4(dummy.matrix); // bake transform into geometry
  geos.push(g);
}

const merged = mergeGeometries(geos, false); // false = no groups
const mesh = new THREE.Mesh(merged, new THREE.MeshStandardMaterial());
scene.add(mesh);

// Dispose individual geometries after merge
geos.forEach(g => g.dispose());
```

**When to merge vs InstancedMesh:**
- Use `InstancedMesh` when all instances share the **same transform-independent material** and you need per-instance transforms at runtime.
- Use `mergeGeometries` when instances are **static** or have **varied materials per mesh**, and you want a zero-overhead single draw call.

## Morph Targets

```js
const baseGeo = new THREE.SphereGeometry(1, 32, 16);
const morphGeo = new THREE.SphereGeometry(2, 32, 16); // inflated version

// Morph target must have same vertex count as base
baseGeo.morphAttributes.position = [
  morphGeo.getAttribute('position') // the target positions
];

const mesh = new THREE.Mesh(baseGeo, new THREE.MeshStandardMaterial({
  morphTargets: true
}));
mesh.morphTargetInfluences[0] = 0; // 0 = base, 1 = fully morphed
scene.add(mesh);

// Animate morph influence in loop
mesh.morphTargetInfluences[0] = Math.sin(elapsed) * 0.5 + 0.5;
```

## Bounding Boxes & Raycasting

```js
// Compute bounding sphere/box (required for raycasting and frustum culling)
geo.computeBoundingSphere();
geo.computeBoundingBox();

// Raycasting
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();

window.addEventListener('pointermove', (e) => {
  pointer.x =  (e.clientX / window.innerWidth)  * 2 - 1;
  pointer.y = -(e.clientY / window.innerHeight) * 2 + 1;
});

// In animate():
raycaster.setFromCamera(pointer, camera);
const hits = raycaster.intersectObjects(scene.children, true); // recursive=true
if (hits.length > 0) {
  console.log(hits[0].object, hits[0].point, hits[0].distance);
}
```

## Disposal

```js
// Dispose geometry and all associated GPU buffers
geo.dispose();

// For InstancedMesh, also clear the instance matrix buffer
instancedMesh.geometry.dispose();
instancedMesh.material.dispose();
```

Never call `dispose()` on geometries that are still in use by any mesh in the scene.

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| New geometry created inside animation loop | Create once, reuse |
| `instanceMatrix.needsUpdate` not set | Set after every `setMatrixAt()` batch |
| LOD `update(camera)` not called per frame | Add to animation loop |
| `mergeGeometries` source arrays not disposed | Dispose input geometries after merge |
| Raycasting fails on custom geometry | Call `geo.computeBoundingSphere()` |
| Morph targets material missing `morphTargets: true` | Required flag on material |

## Additional Resources

- **`references/buffer-geometry-api.md`** — Full `BufferAttribute` API, interleaved buffers, draw ranges, custom vertex attribute patterns
- **`references/instancing-patterns.md`** — Animated instancing, picking instances by index, custom per-instance attributes, frustum culling workarounds
