# Husky Sled - Game Design Document

## 1. Game Overview

**Title:** Husky Sled
**Genre:** 3D forward-scrolling endless runner
**Platform:** Web browser (Three.js WebGL)
**Target:** Single HTML file with embedded JavaScript and CSS -- Three.js loaded via CDN
**Visual Style:** Low-poly, dusk atmosphere

Steer a husky sled down a snowy trail, dodge trees, go as far as you can. Camera behind the sled, Tux Racer style. Dusk lighting inspired by Outer Wilds -- warm golden light against cold blue snow. Simple, polished, fun.

---

## 2. Design References

### 2.1 Tux Racer -- Gameplay
- Third-person chase camera looking down the slope
- Left/right steering, dodge obstacles, speed increases
- Pick-up-and-play: instant start, instant restart, no menus

### 2.2 Outer Wilds -- Atmosphere
- Warm-vs-cold lighting: golden directional light against cool blue ambient
- Dusk sky: deep navy above fading to warm amber at horizon
- The world feels beautiful and vast, even though gameplay is simple

---

## 3. Core Mechanics

### 3.1 Forward Motion
- The world moves toward the camera (sled stays at fixed Z = 0).
- Base speed: **0.3 units/frame**, increasing gradually, capping at **0.8 units/frame**.
- Speed formula: `speed = min(0.8, 0.3 + elapsedSeconds * 0.002)`

### 3.2 Lateral Movement
- Player steers left/right along the X axis.
- Movement speed: **0.15 units/frame** (constant).
- Clamped to trail boundaries: **X = -3.0 to X = 3.0**.
- Responsive and direct -- no drift or momentum.

### 3.3 Collision Detection
- Distance-based collision on the XZ plane.
- Sled collision radius: **0.6 units**. Tree collision radius: **0.4 units**.
- Collision = Game Over.

---

## 4. Controls

| Action         | Primary Key | Alternate Key |
|----------------|-------------|---------------|
| Steer Left     | ArrowLeft   | A             |
| Steer Right    | ArrowRight  | D             |
| Start / Restart| Space       | Space         |

Space starts the game from the title screen and restarts from game over. Instant, no friction.

---

## 5. Visual Design

### 5.1 Renderer Setup
- **Three.js** via CDN.
- **Renderer:** `THREE.WebGLRenderer` with antialiasing.
- **Canvas:** Fills the browser window, resizes on window resize.
- **Clear color:** `#2A1B3D` (deep dusk purple).
- **Fog:** `THREE.Fog`, color `#D4A574` (warm amber haze), near = 10, far = 80.

### 5.2 Camera
- `THREE.PerspectiveCamera`, FOV = 60.
- Position: offset `(0, 3, 6)` relative to sled.
- LookAt: `(0, 0.5, -2)` relative to sled.
- Follows sled X with lerp smoothing (factor 0.1).

### 5.3 Lighting

Two lights create the dusk warm/cold contrast:

- **Ambient:** `THREE.AmbientLight`, color `#6B7B99` (cool blue), intensity 0.4.
- **Directional:** `THREE.DirectionalLight`, color `#FFD4A0` (golden amber), intensity 1.0, position `(10, 3, -5)` -- low-angle setting sun.
- No shadows.

### 5.4 Ground
- `THREE.PlaneGeometry` (width = 10, length = 200), flat on XZ plane.
- `THREE.MeshLambertMaterial`, color `#F0F0F5` (snow white).
- Repositioned as the sled advances to create infinite ground.

### 5.5 Mountains (Background)
- 5-8 large `THREE.ConeGeometry` (radius 8-15, height 10-20, 5-6 radial segments) placed far from trail (X = +/-15 to +/-30).
- Color `#7B8FA8` (muted blue-gray).
- Static or very slow scroll. Recycle when behind camera.

### 5.6 Scenery Trees (Decorative)
- Pine trees along both sides of the trail (X = -5 to -4 and X = 4 to 5).
- Not collidable. Same mesh as obstacle trees.
- Spawn/recycle as they scroll past.

### 5.7 Husky Sled (Player)

`THREE.Group` of primitive geometries:

- **Sled:** `BoxGeometry` (1.2 x 0.15 x 0.5), color `#8B5E3C` (brown). Two thin runner bars underneath, color `#5C3A1E`.
- **Husky:** In front of sled. Body: `BoxGeometry` (0.3 x 0.3 x 0.5), color `#A0A0A8`. Head: `BoxGeometry` (0.22 x 0.22 x 0.22), color `#C0C0C8`. Two cone ears, color `#707078`. Cone tail. Four box legs with simple run-cycle oscillation (speed tied to game speed). Harness line connecting dog to sled.
- Position: Z = 0, Y = 0.2, X = player-controlled.

### 5.8 Obstacles -- Pine Trees

The only obstacle type. Keep it simple.

- **Trunk:** `CylinderGeometry` (radiusTop 0.06, radiusBottom 0.1, height 0.5, 6 segments), color `#6B4226`.
- **Foliage:** 2-3 stacked `ConeGeometry` (6 segments), colors `#1B4A1E`, `#2E5D32`, `#3A7040` (dark to medium green).
- Total height ~2.0 units.
- Spawn at Z = -80, random X within **-2.8 to 2.8**.
- Minimum Z spacing: **8 units** at base speed, decreasing to **4 units** at max speed.
- Gap formula: `gap = max(4, 8 - elapsedSeconds * 0.035)`
- Remove when Z > 10.

### 5.9 Snow Particles

- `THREE.Points` with `BufferGeometry`.
- **150 particles.** Random positions, size 0.05-0.15.
- `PointsMaterial`, color `#FFFFFF`, opacity 0.7.
- Drift downward with slight horizontal sway. Reset to top when below ground.
- Particle system follows sled Z position.

### 5.10 Sky

- **Simple approach (recommended):** Just use the clear color (`#2A1B3D`) and let fog blend the horizon to warm amber. This looks good and costs nothing.
- **Optional upgrade:** Sky dome (`SphereGeometry` radius 90, `BackSide`) with canvas-generated gradient: zenith `#0D0D2B` (deep navy) -> mid `#2A3A6B` (twilight blue) -> horizon `#D4885A` (warm amber) -> below `#D4A574` (matches fog). Only add if the simple approach doesn't look good enough.

---

## 6. Scoring

- **1 point per frame** while alive (~60 points/second).
- Score displayed in top-right corner as HTML overlay div.
- White text, 24px, subtle text shadow. Minimal and unobtrusive.

---

## 7. Game States

Three states, simple state machine.

### 7.1 Title Screen
- 3D world visible, slowly scrolling, snow falling. No obstacles.
- HTML overlay: "HUSKY SLED" (56px white bold) and "Press SPACE to start" (22px, pulsing opacity).
- Sled visible but stationary.

### 7.2 Playing
- Everything active: scrolling, steering, obstacles, collisions, scoring, snow.
- Speed increases over time.

### 7.3 Game Over
- Everything freezes.
- Dark overlay (`rgba(0,0,0,0.4)`).
- "GAME OVER" (48px, `#FF8B6B`), score, "Press SPACE to restart" (pulsing).
- Snow keeps falling.
- Space instantly restarts.

---

## 8. Difficulty Curve

| Time (seconds) | Speed (units/frame) | Min Obstacle Gap (units) |
|-----------------|--------------------|--------------------------|
| 0 - 10          | 0.30               | 8.0                      |
| 10 - 30         | 0.34               | 7.0                      |
| 30 - 60         | 0.40               | 6.0                      |
| 60 - 120        | 0.52               | 5.0                      |
| 120+            | 0.64 - 0.80 (cap) | 4.0 (cap)                |

---

## 9. Color Palette

| Element           | Hex       |
|-------------------|-----------|
| Clear color       | `#2A1B3D` |
| Fog               | `#D4A574` |
| Ground            | `#F0F0F5` |
| Mountains         | `#7B8FA8` |
| Sled              | `#8B5E3C` |
| Sled runners      | `#5C3A1E` |
| Husky body        | `#A0A0A8` |
| Husky head        | `#C0C0C8` |
| Tree trunk        | `#6B4226` |
| Tree foliage      | `#1B4A1E` / `#2E5D32` / `#3A7040` |
| Snow              | `#FFFFFF` |
| Ambient light     | `#6B7B99` |
| Directional light | `#FFD4A0` |
| Game Over text    | `#FF8B6B` |

---

## 10. Technical Notes

- Single `index.html`, one CDN dependency (Three.js r128).
- `requestAnimationFrame` loop with `THREE.Clock` delta time.
- `keydown`/`keyup` tracking for smooth input.
- Obstacle array: spawn ahead, move toward camera, remove when behind.
- Scenery tree pool: ~20 per side, recycle.
- HUD: HTML div overlay, not 3D text.
- Window resize: update camera aspect + renderer size.
- No audio, no textures, no images. All geometry is primitives.

---

## 11. What NOT to Build

To keep scope tight:
- No collectibles or power-ups
- No multiple obstacle types (just trees)
- No complex UI or menus
- No score persistence or leaderboards
- No mobile/touch controls
- No sound
- No texture maps or external assets
