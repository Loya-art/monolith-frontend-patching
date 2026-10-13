# Mineradio Extended — Built-in Gesture Control Architecture

> Discovered July 2026. The Extended fork (daaimengermengzhu/Mineradio-Extended) already ships a complete gesture control system. I spent hours re-implementing what was already there before finding it.

## Entry Point

**DIY panel → 摄像头交互 → 手势触碰**

```
DIY 工坊（右下 fx-advanced 面板）
  → fx-section-label: 摄像头交互
    → fx-seg#cam-seg
      → button[data-cam="off"] 关闭
      → button[data-cam="gesture"] 手势触碰  ← THIS
```

## How It Works (already built in)

### Initialization (`startGestureControl()`)
- Loads `@mediapipe/camera_utils` and `@mediapipe/hands` from CDN
- Creates hidden `<video>` element for camera feed
- Creates `Hands` instance with `maxNumHands: 1`
- Creates `Camera` instance at 480×360 resolution
- Shows a HUD at bottom-center, a transparent skeleton overlay (`#hand-canvas`)

### Per-frame Pipeline (`processHandFrame()`)
```
MediaPipe 21 landmarks
  → smoothLandmarks() [exponential moving average, alpha=0.48]
  → palmCenter() [avg of wrist + 4 MCP knuckles]
  → handOpenness() [avg fingertip distance / palm span]
  → gesture state machine (pinch / fist / open / hover)
```

### 3 Shader Interfaces

| Uniform | Type | Effect |
|---------|------|--------|
| `uHandActive` | float | Blends in hand-interaction color boost (0→1) |
| `uHandXY` | vec2 | Particle push direction in local space |
| `uGestureGrip` | float | Contraction: `pos.xy *= mix(1.0, 0.66, grip)` + Z boost |

### Gesture State Machine

| State | Detection | Effect |
|-------|-----------|--------|
| **hover** | `openness < 0.62` | Mild hand position tracking |
| **open** | `openness > 0.62` → `gestureGrip.lastState === 'open'` | Gentle particle push via `uHandXY` |
| **pinch** | `thumb-index dist < 0.075 && openness > 0.28` → `pinchState.active` | Drag rotation via `gestureRotation.x/y` + `particleSpin` |
| **fist** | `gripTarget > 0.68` | `uBurstAmt` pulse + `uGestureGrip` contraction |

### Rotation Architecture (key insight)

`gestureRotation` IS the particle system rotation. It feeds into:
- `particles.rotation` (main particle group)
- `bloomParticles.rotation`
- `floatGroup.rotation`
- `backCoverGroup.rotation`
- `customShapeGroup.rotation`
- Skull model rotation (when that preset is active)

## What I Couldn't Achieve (and why)

| Feature | Attempted | Why it failed |
|---------|-----------|---------------|
| Shader-level orbital motion | Modifed GLSL `uGestureGrip` block | `pos.z +=` accumulates every frame — particles fly to infinity. Fixing to non-accumulative works but looks like random drift, not orbital rings. |
| Palm open → galaxy reset | `setPreset(0)` | `gestureGrip.lastState` started as `'open'` so the detection guard `lastState !== 'open'` always failed. Fix: init to `'init'` not `'open'`. |
| Fist → electron-like orbit | gestureRotation.y += speed | Actually works. The gestureRotation system rotates the ENTIRE particle field, creating a smooth disc/cloud rotation that looks like atomic orbital motion when combined with mild grip contraction. |

## Lesson: The Right Approach (found after 3+ hours of wrong turns)

1. **Search FIRST**: Before writing any gesture/feature code, search GitHub for `<app-name>-Extended` or fork names. The Extended fork had built everything.
2. **Read the shader carefully BEFORE editing**: `pos.z +=` in a vertex shader with grip that oscillates every frame WILL accumulate. Use `pos.z = mix(baseZ, baseZ + offset, grip)` instead.
3. **Use gestureRotation for particle GROUP rotation** (not shader per-particle orbit): `gestureRotation.y += speed` on fist creates a clean, stable orbital effect for all particles together. This is the closest visual match to "electrons orbiting a nucleus" without complex shader math.
4. **Gated drag during fist**: When fist is active, skip `applyParticleSpinDrag()` so the auto-rotation doesn't fight with hand-movement rotation.
5. **Palm open → reset view**: `recenterCamera()` + `resetParticleRotationTarget(true)` + moderate burst. Switch preset only if the user is OK with losing their current visual mode.