# Husky Sled v2 -- Changes Specification

All changes below are relative to the current `index.html` implementation.

---

## 1. Musher (New Feature)

Add a low-poly person standing on the sled runners behind the basket, gripping the handlebar.

**Parent:** Add to `sledGroup` after the handlebar is built.

**Reference points from existing code:**
- Handlebar top bar: `y=0.70, z=0.65`
- Handlebar posts: `x=+/-0.30, z=0.65`
- Runners: `y=0.015, z` range roughly `-0.7 to +0.7`

**Musher geometry (all BoxGeometry/SphereGeometry):**

| Part | Geometry | Size (w,h,d) | Position (x,y,z) | Color |
|------|----------|--------------|-------------------|-------|
| Head | SphereGeometry(0.09, 8, 6) | r=0.09 | (0, 1.22, 0.80) | `#D4A080` (skin) |
| Hat | CylinderGeometry(0.10, 0.10, 0.08, 8) | -- | (0, 1.30, 0.80) | `#1A1A2A` |
| Torso | BoxGeometry | 0.30 x 0.35 x 0.18 | (0, 0.92, 0.80) | `#2A3040` (coat) |
| Left upper arm | BoxGeometry | 0.07 x 0.22 x 0.07 | (-0.19, 0.88, 0.75) | `#2A3040` |
| Right upper arm | BoxGeometry | 0.07 x 0.22 x 0.07 | (0.19, 0.88, 0.75) | `#2A3040` |
| Left forearm | BoxGeometry | 0.06 x 0.18 x 0.06 | (-0.25, 0.78, 0.68) | `#2A3040` |
| Right forearm | BoxGeometry | 0.06 x 0.18 x 0.06 | (0.25, 0.78, 0.68) | `#2A3040` |
| Left hand | BoxGeometry | 0.05 x 0.05 x 0.05 | (-0.28, 0.70, 0.65) | `#D4A080` |
| Right hand | BoxGeometry | 0.05 x 0.05 x 0.05 | (0.28, 0.70, 0.65) | `#D4A080` |
| Left leg | BoxGeometry | 0.10 x 0.40 x 0.10 | (-0.08, 0.45, 0.85) | `#2A3040` |
| Right leg | BoxGeometry | 0.10 x 0.40 x 0.10 | (0.08, 0.45, 0.85) | `#2A3040` |
| Left boot | BoxGeometry | 0.11 x 0.08 x 0.16 | (-0.08, 0.08, 0.83) | `#3A2A1A` |
| Right boot | BoxGeometry | 0.11 x 0.08 x 0.16 | (0.08, 0.08, 0.83) | `#3A2A1A` |

**Materials (create once, reuse):**
```
musherCoatMat  = MeshLambertMaterial({ color: 0x2A3040 })
musherSkinMat  = MeshLambertMaterial({ color: 0xD4A080 })
musherBootMat  = MeshLambertMaterial({ color: 0x3A2A1A })
musherHatMat   = MeshLambertMaterial({ color: 0x1A1A2A })
```

**Animation:** None required. Static pose -- standing upright, arms reaching forward to the handlebar.

---

## 2. Obstacles (Verification)

The current implementation already has all required obstacle types:
- **Trees** (`createTreeMesh`) -- 80% spawn chance, `collisionRadius: 0.4`
- **Rocks** (`createRockMesh`, DodecahedronGeometry) -- 20% spawn chance, `collisionRadius: 0.35`
- **Double-tree spawns** -- chance increases with elapsed time up to 40%, with minimum 1.5-unit X gap

**Status: No changes needed.**

---

## 3. Trail Variation (New Feature)

Add sinusoidal terrain undulation to create gentle hills (Y offset) and turns (X offset). The sled stays at a fixed position but the world offsets create the feeling of terrain variation.

### 3.1 Terrain Offset Formulas

Track a cumulative `trailDistance` variable, incremented by `speed` each frame.

```javascript
// Y offset: gentle hills
const hillY = Math.sin(trailDistance * 0.04) * 0.6
            + Math.sin(trailDistance * 0.017) * 0.3;

// X offset: gentle turns
const turnX = Math.sin(trailDistance * 0.025) * 1.5
            + Math.sin(trailDistance * 0.011) * 0.8;
```

- Two sine waves per axis produce organic, non-repetitive movement
- Hill amplitude: +/-0.9 units max vertical
- Turn amplitude: +/-2.3 units max lateral

### 3.2 Applying Offsets

- **Sled group Y:** Lerp `sledGroup.position.y` toward `0.05 + hillY` with factor `0.08`
- **Sled group tilt:** Set `sledGroup.rotation.x` toward the hill slope: `hillSlope = cos(trailDistance * 0.04) * 0.04 * 0.6 + cos(trailDistance * 0.017) * 0.017 * 0.3` (derivative of hillY scaled down)
- **Obstacle spawn X:** Add current `turnX` to the random X when spawning obstacles, then clamp to trail bounds
- **Scenery trees:** Add current `turnX` offset when repositioning
- **Ground mesh:** Lerp ground X toward `turnX`, Y toward `hillY`

### 3.3 Camera Follow

Update `updateCamera()`:

```javascript
function updateCamera() {
  const targetX = sledGroup.position.x + turnX;
  const targetY = 3 + hillY;
  camera.position.x += (targetX - camera.position.x) * 0.08;
  camera.position.y += (targetY - camera.position.y) * 0.06;
  camera.position.z = 6;

  // Subtle camera roll on turns
  const turnRate = Math.cos(trailDistance * 0.025) * 0.025 * 1.5
                 + Math.cos(trailDistance * 0.011) * 0.011 * 0.8;
  camera.rotation.z += (-turnRate * 0.8 - camera.rotation.z) * 0.05;

  camera.lookAt(
    sledGroup.position.x + turnX * 0.5,
    0.5 + hillY,
    -2
  );
}
```

- Camera lerp factor 0.08 (X) and 0.06 (Y) for smooth follow
- Camera roll proportional to turn rate (derivative of turnX), capped by the math itself (~0.03 rad max)
- LookAt target includes partial turnX to look slightly into turns

### 3.4 State

```javascript
let trailDistance = 0;     // cumulative, incremented by speed each frame
let currentHillY = 0;     // smoothed current hill offset
let currentTurnX = 0;     // smoothed current turn offset
```

Reset `trailDistance = 0` on `startGame()`.

---

## 4. White Snow

### 4.1 Ground Color

Change ground material color from `0xF0F0F5` to `0xFFFFFF`:

```javascript
const groundMat = new THREE.MeshLambertMaterial({ color: 0xFFFFFF });
```

### 4.2 Mountain Snow Caps

No change -- mountains remain `#7B8FA8` to contrast against white ground.

---

## 5. Aurora Night Sky

Transform the atmosphere from warm dusk to aurora night. This touches sky dome, clear color, fog, and lighting.

### 5.1 Sky Dome Gradient

Replace the existing canvas gradient color stops with:

| Stop | Color | Description |
|------|-------|-------------|
| 0.00 | `#050510` | Deep space at zenith |
| 0.15 | `#0A0A2E` | Dark blue upper sky |
| 0.30 | `#0D1A2F` | Transition zone |
| 0.40 | `#1A4A3A` | Aurora green lower edge |
| 0.48 | `#2AFF6A` | Aurora green bright band |
| 0.52 | `#1ADA5A` | Aurora green core |
| 0.58 | `#1A4A3A` | Aurora green upper edge |
| 0.70 | `#0D1A2F` | Below aurora |
| 0.85 | `#0A1020` | Near horizon dark |
| 1.00 | `#0A1525` | Horizon |

### 5.2 Aurora Movement

Add slow Y-axis rotation to the sky dome for a drifting aurora effect:

```javascript
skyDome.rotation.y += delta * 0.015;
```

Add this line in the `animate()` function, executing in all game states.

### 5.3 Renderer Clear Color

Change from `0x2A1B3D` to `0x050510` (very deep dark blue):

```javascript
renderer.setClearColor(0x050510);
```

Also update CSS body background to match:
```css
body { background: #050510; }
```

### 5.4 Fog

Change fog color and range for nighttime visibility:

```javascript
scene.fog = new THREE.Fog(0x0A1525, 15, 90);
```

- Color: `#0A1525` (dark blue-tinted, matches horizon)
- Near: `15` (slightly further to keep foreground clear at night)
- Far: `90` (slightly further for better visibility)

### 5.5 Lighting

Replace existing lights entirely.

**Remove:**
- `AmbientLight(0x6B7B99, 0.4)`
- `DirectionalLight(0xFFD4A0, 1.0)`

**Add:**

| Light | Type | Color | Intensity | Position | Purpose |
|-------|------|-------|-----------|----------|---------|
| Moonlight ambient | AmbientLight | `#2A3A5A` | 0.5 | -- | Cool overall illumination |
| Moon directional | DirectionalLight | `#8899CC` | 0.7 | (5, 8, -3) | Cool moonlight from upper left |
| Aurora glow | PointLight | `#2AFF6A` | 0.3 | (0, 15, -20) | Subtle green tint from above |

```javascript
const ambientLight = new THREE.AmbientLight(0x2A3A5A, 0.5);
scene.add(ambientLight);

const moonLight = new THREE.DirectionalLight(0x8899CC, 0.7);
moonLight.position.set(5, 8, -3);
scene.add(moonLight);

const auroraLight = new THREE.PointLight(0x2AFF6A, 0.3, 60);
auroraLight.position.set(0, 15, -20);
scene.add(auroraLight);
```

### 5.6 UI Text Shadow Adjustment

Update overlay title text shadow from warm amber to cool aurora glow:

```css
#overlay-title {
  text-shadow: 0 4px 20px rgba(42, 255, 106, 0.4);
}
```

---

## 6. Sound (Verification)

The current implementation has full Web Audio API procedural sound:
- Sled-on-snow filtered noise
- Wind ambience with LFO oscillation
- Periodic husky howl with pitch sweep and vibrato
- Collision crash noise burst
- Master gain, pause/resume/stop controls

**Status: No changes needed.**

---

## 7. Color Palette Update (Summary)

| Element | v1 | v2 |
|---------|----|----|
| Clear color | `#2A1B3D` | `#050510` |
| CSS body background | `#2A1B3D` | `#050510` |
| Fog color | `#D4A574` | `#0A1525` |
| Fog near/far | 10/80 | 15/90 |
| Ground | `#F0F0F5` | `#FFFFFF` |
| Ambient light | `#6B7B99` @ 0.4 | `#2A3A5A` @ 0.5 |
| Directional light | `#FFD4A0` @ 1.0 | `#8899CC` @ 0.7 |
| Aurora point light | -- | `#2AFF6A` @ 0.3 |
| Title text shadow | warm amber | green aurora |
| Sky gradient | dusk (navy to amber) | night (deep blue + green aurora bands) |
| Musher coat | -- | `#2A3040` |
| Musher skin | -- | `#D4A080` |
| Musher boots | -- | `#3A2A1A` |
| Musher hat | -- | `#1A1A2A` |
