# Scene Spec Format

Encourage structured scene descriptions via JSON spec. When a user provides a spec, treat it as
source of truth while flagging inconsistencies and proposing sensible defaults for missing fields.

## Full Schema

```json
{
  "projectName": "string — Name of the project",
  "stack": "vanilla | react-three-fiber",
  "canvasMode": "fullscreen | embedded",

  "camera": {
    "type": "perspective | orthographic",
    "fov": "number — Field of view in degrees (perspective only, default: 45)",
    "near": "number — Near clipping plane (default: 0.1)",
    "far": "number — Far clipping plane (default: 100)",
    "position": "[x, y, z] — Camera world position",
    "lookAt": "[x, y, z] — Camera target point"
  },

  "environment": {
    "background": "color | texture | envMap — Background type",
    "backgroundValue": "string — Hex color, texture path, or env map path",
    "fog": {
      "type": "linear | exp | exp2",
      "color": "string — Hex color",
      "near": "number — Start distance (linear fog)",
      "far": "number — End distance (linear fog)",
      "density": "number — Density (exp/exp2 fog)"
    }
  },

  "lighting": [
    {
      "type": "ambient | directional | point | spot | hemisphere | rectarea",
      "color": "string — Hex color",
      "intensity": "number",
      "position": "[x, y, z] — Light position (not for ambient/hemisphere)",
      "target": "[x, y, z] — Light target (directional/spot only)",
      "castShadow": "boolean",
      "shadow": {
        "mapSize": "number — Shadow map resolution (default: 2048)",
        "camera": {
          "near": "number",
          "far": "number",
          "left": "number",
          "right": "number",
          "top": "number",
          "bottom": "number"
        }
      }
    }
  ],

  "assets": [
    {
      "id": "string — Unique identifier for reference",
      "type": "gltf | texture | envMap | hdr",
      "src": "string — File path",
      "position": "[x, y, z]",
      "rotation": "[x, y, z] — Euler rotation in radians",
      "scale": "[x, y, z] | number — Uniform or per-axis scale",
      "receiveShadow": "boolean",
      "castShadow": "boolean",
      "animations": "boolean — Auto-play embedded animations"
    }
  ],

  "effects": {
    "postprocessing": ["bloom", "fxaa", "taa", "ssao", "dof", "motionBlur", "outline", "lut"],
    "shaders": ["string — Custom shader identifiers (e.g. 'waterSurface', 'portal', 'hologram')"]
  },

  "interactions": [
    {
      "type": "hover | click | drag",
      "target": "string — Asset ID",
      "effect": "string — Effect name (e.g. 'highlight', 'scale', 'glow')",
      "action": "string — Action name (e.g. 'openPanel', 'navigate', 'playAnimation')"
    }
  ],

  "animation": {
    "type": "timeline | loop | scroll",
    "library": "gsap | tween | framer | native",
    "scrollDriven": "boolean",
    "sections": [
      {
        "name": "string — Section identifier",
        "trigger": "string — CSS selector for scroll trigger",
        "camera": { "position": "[x, y, z]", "lookAt": "[x, y, z]" },
        "objects": [
          { "target": "string — Asset ID", "property": "string", "value": "any" }
        ]
      }
    ]
  },

  "constraints": {
    "maxBundleSizeMB": "number — Max total asset size in MB",
    "targetFPS": "number — Target frame rate (default: 60)",
    "mobileSupport": "boolean — Must work on mobile",
    "maxDrawCalls": "number — Draw call budget",
    "fallback": "boolean — Non-WebGL fallback needed"
  }
}
```

## Field Defaults

When fields are omitted, apply these defaults:

| Field | Default |
|-------|---------|
| `stack` | `"vanilla"` |
| `canvasMode` | `"fullscreen"` |
| `camera.type` | `"perspective"` |
| `camera.fov` | `45` |
| `camera.near` | `0.1` |
| `camera.far` | `100` |
| `camera.position` | `[0, 2, 5]` |
| `camera.lookAt` | `[0, 0, 0]` |
| `constraints.targetFPS` | `60` |
| `constraints.mobileSupport` | `true` |

## Usage Guidelines

1. Parse the spec before writing any code
2. Flag missing critical fields (e.g., no lighting defined)
3. Warn about potential performance issues (e.g., too many lights with shadows + mobile)
4. Propose alternatives if constraints conflict with requested effects
5. Use asset IDs consistently when referencing objects in interactions and animations
