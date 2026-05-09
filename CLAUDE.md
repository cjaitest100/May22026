# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**Cube Tac Toe** — a single-file browser game (`index.html`) combining a 3×3 Rubik's Cube with Tic Tac Toe. No build system, no dependencies to install, no tests. Open `index.html` directly in a browser or serve it over HTTP.

## Running locally

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

Or just double-click `index.html` (works because Three.js is loaded from a CDN via classic `<script>` tags, not ES modules).

## Stack

- **Three.js r128** via jsDelivr CDN (global script tags — `window.THREE`).
- `THREE.OrbitControls` from the same CDN (`examples/js/controls/OrbitControls.js`).
- No framework, no bundler, no package.json.

## Code architecture (all in `index.html`)

Everything is in one `<script>` block. Key sections in order:

### 1. Coordinate helpers
- `stickerToCubie(face, row, col)` — converts a logical sticker position to its cubie's integer grid position `(x,y,z)` and outward normal `(nx,ny,nz)`.
- `cubieToSticker(x,y,z, nx,ny,nz)` — reverse: physical cubie position + normal → logical `{face, row, col}`.
- Face convention: `0=U(+Y), 1=D(-Y), 2=F(+Z), 3=B(-Z), 4=L(-X), 5=R(+X)`. Sticker index = `face*9 + row*3 + col`.

### 2. Move permutations
- `FACE_DEFS` — per-face `{axis, layer, cwSign}` where `cwSign` is the sign of the rotation about the positive axis that produces a clockwise visual turn from outside the cube.
- `buildPerm(faceLetter, prime)` — algorithmically derives the full 54-element permutation for any move. No hand-coded index tables.
- `PERMS[moveName]` — pre-built permutation for each of the 12 moves (`U U' D D' F F' B B' L L' R R'`).
- A startup self-test applies each move 4× to a verified state and asserts identity; it logs to the console.

### 3. Three.js scene
- 26 black cubies (`BoxGeometry(0.95,0.95,0.95)`), each with its own `MeshLambertMaterial` so emissive can be set per-cubie for the glow effect.
- 54 white sticker planes (`PlaneGeometry(0.85,0.85)`, `MeshBasicMaterial` + `CanvasTexture`) are children of their parent cubie; they rotate with it automatically during animation.
- `allStickerMeshes[]` — flat array of all 54 sticker meshes, used for raycasting.

### 4. Dynamic sticker→index mapping
- `meshToLogicalIndex(mesh)` — computes the current logical sticker index for any mesh by deriving its world-space position and normal at runtime. Required because cubies physically move during rotations; the mapping is not static after the first move.

### 5. Visual feedback
- `setLayerGlow(faceLetter, on)` — sets `emissive` on the 9 cubies in the layer.
- `createMoveArrow(moveName)` / `removeMoveArrow()` — builds a `TubeGeometry` arc with a `ConeGeometry` arrowhead in the rotation plane. Arrow direction is computed as `arcDir = def.axis === 'y' ? -sign : sign` (Y-axis has opposite handedness relative to X/Z).

### 6. Game state machine
States: `MARK_PHASE → MOVE_PHASE → ANIMATING → (GAME_OVER | MARK_PHASE)`.
- `MARK_PHASE`: OrbitControls active, raycaster fires on tap/click.
- `MOVE_PHASE`: move buttons enabled; hover shows glow + arrow preview (desktop).
- `ANIMATING`: OrbitControls disabled, buttons disabled; pivot-attach animation runs.

### 7. Move animation
Pivot technique: create an empty `Object3D`, `pivot.attach(cubie)` for each of the 9 layer cubies (preserves world transform), tween `pivot.rotation[axis]`, then `scene.attach(cubie)` and snap positions to integers with `Math.round`.

## Critical invariants

- **Do not share cubie materials** — each cubie needs its own `MeshLambertMaterial` so `emissive` can be controlled per-layer.
- **Permutation direction and animation sign must match** — `sign = (prime ? -1 : 1) * def.cwSign` is used for both. Verify with the 4× self-test in the console.
- **Integer snapping after animation** — `Math.round` on cubie positions after every `scene.attach` call prevents float drift accumulating over many moves. `meshToLogicalIndex` depends on this.
- **Arc direction formula** — `arcDir = def.axis === 'y' ? -sign : sign`. Changing this breaks the arrow direction for Y-axis moves (U/D).
