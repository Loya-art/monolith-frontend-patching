# Gesture → Particle System Integration Pattern

Adding hand-tracking (MediaPipe) gesture control to an existing Three.js particle visualizer in a monolith SPA.

## Architecture

```
Camera stream → MediaPipe Hands (21 landmarks) → processHandFrame() → existing particle input API
                                                         ↓
                                                  gesture HUD (auto-hide, bottom-left)
```

**Key principle**: Do NOT modify the shader or particle rendering core. Map hand data to EXISTING input functions the particle system already uses.

## Step 1: Learn the Existing Input Pipeline

Before writing any gesture code, trace how mouse/touch input reaches the particles:

1. Find the `mousedown`, `mousemove`, `mouseup` listeners
2. Identify the functions they call: typically `applyParticleSpinDrag(dx, dy, dt)`, `queueParticlePointerFrame(x, y)`, `unlockCenteredView()`
3. Find the camera control functions: `recenterCamera()`, `gestureRotation.{x,y}`, `freeCamera.fov`
4. Find the shader uniforms used for gesture effects: typically `uHandXY`, `uHandActive`, `uGestureGrip`, `uBurstAmt`, `uScatter`

## Step 2: Add MediaPipe Hand Tracking

```javascript
// Load from CDN in the async start function
await loadScriptOnce('https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js');
await loadScriptOnce('https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js');

// Create hidden video element
var video = document.createElement('video');
video.playsInline = true; video.muted = true; video.style.display = 'none';
document.body.appendChild(video);

// Init Hands model
var hands = new Hands({ locateFile: f => 'https://cdn.jsdelivr.net/npm/@mediapipe/hands/' + f });
hands.setOptions({ maxNumHands: 2, modelComplexity: 1, minDetectionConfidence: 0.7, minTrackingConfidence: 0.7 });

// Camera utility — sends frames to the model
var camera = new Camera(video, {
  onFrame: async () => { if (hands) await hands.send({ image: video }); },
  width: 480, height: 360
});
await camera.start();
```

## Step 3: Gesture Detection from 21 Landmarks

### Palm center
Average of wrist (0) + four finger bases (5, 9, 13, 17):
```javascript
function palmCenter(lm) {
  var px = (lm[0].x + lm[5].x + lm[9].x + lm[13].x + lm[17].x) / 5;
  var py = (lm[0].y + lm[5].y + lm[9].y + lm[13].y + lm[17].y) / 5;
  return { x: px, y: py };
}
```

### Hand openness
Ratio of fingertip-to-palm distance vs hand span:
```javascript
function handOpenness(lm, palm) {
  var span = Math.hypot(lm[5].x - lm[17].x, lm[5].y - lm[17].y);
  span = Math.max(0.055, span);
  var tips = [8, 12, 16, 20]; // finger tips
  var avg = tips.reduce((s, t) => s + Math.hypot(lm[t].x - palm.x, lm[t].y - palm.y), 0) / tips.length;
  return clampRange((avg / span - 0.62) / 0.78, 0, 1);
}
// openness > 0.72 = open palm, > 0.28 = not-fist
```

### Thumb-index pinch distance
```javascript
var idxTip = Math.hypot(lm[8].x - lm[4].x, lm[8].y - lm[4].y);
var isPinch = idxTip < 0.075 && openness > 0.28;
```

### Grip strength (fist detection)
```javascript
var gripTarget = clampRange(1 - openness, 0, 1);  // 1 = fully closed
var isFist = !isPinch && gripTarget > 0.68;
```

## Step 4: Map Gestures to Particle System

### Default: Single hand movement → orbit drag
Map the palm position through the existing pointer system so every gesture also rotates particles:
```javascript
if (particleLocalPointFromNdc(ndcX, ndcY, particlePointerLocalHit)) {
  var dx = (ndcX - lastNdcX);
  var dy = (ndcY - lastNdcY);
  unlockCenteredView();
  particlePointerSpin.active = true;
  applyParticleSpinDrag(dx * 120, dy * 120, 1/60);
}
```

### Open palm → reset to initial state
```javascript
if (isOpen) {
  recenterCamera();          // reset camera position/FOV
  resetParticleRotationTarget(true);  // reset rotation to zero
  // Clear grip/shader effects
  gestureGrip.target = 0;
  uniforms.uGestureGrip.value = 0;
}
```

### Fist → atomic orbital rotation
Instead of contracting particles (the default `uGestureGrip` behavior), apply continuous rotation:
```javascript
if (isFist) {
  var orbitSpeed = 0.008 + gripTarget * 0.025;
  gestureRotation.y += orbitSpeed;        // spin around Y
  gestureRotation.x += orbitSpeed * 0.3;  // slight X tilt
  // Also tell shader to pull particles in (grip)
  gestureGrip.target = 0.55 + gripTarget * 0.3;
}
```

### Thumb-index distance → zoom
Map pinch distance to `idleGuideInteraction.zoomTarget` (or the app's zoom uniform):
```javascript
var zoomPinch = 1 - Math.min(1, Math.max(0, (idxTip - 0.02) / 0.18));
if (zoomPinch > 0.15 && openness > 0.28) {
  var targetZoom = 0.58 + zoomPinch * 1.24;
  idleGuideInteraction.zoomTarget += (targetZoom - idleGuideInteraction.zoomTarget) * 0.15;
}
```

### Two hands open → pan (translate view)
Sort hands by X position (left vs right), track center of both palms:
```javascript
if (allHands.length >= 2 && isOpen) {
  var cx = (palm.x + secPalm.x) / 2;
  var cy = (palm.y + secPalm.y) / 2;
  if (panLast) {
    var ddx = (cx - panLast.x) * 180;
    var ddy = (cy - panLast.y) * 180;
    applyParticleSpinDrag(ddy * 0.3, ddx * 0.3, 1/60);
  }
  panLast = { x: cx, y: cy };
}
```

## Step 5: Smoothing

MediaPipe returns jittery per-frame landmarks. Apply exponential smoothing:
```javascript
var SMOOTH_ALPHA = 0.48;
for (var i = 0; i < 21; i++) {
  smooth[i].x += ((1 - raw[i].x) - smooth[i].x) * SMOOTH_ALPHA;  // mirror X
  smooth[i].y += (raw[i].y - smooth[i].y) * SMOOTH_ALPHA;
  smooth[i].z += ((raw[i].z || 0) - smooth[i].z) * SMOOTH_ALPHA;
}
```
`ALPHA < 0.5` = smoother but laggier; `ALPHA > 0.5` = responsive but jittery.

## Step 6: Auto-hide HUD

The gesture HUD (instruction bar) should auto-hide after showing a new command:
```javascript
function showGestureHUD(label, progress, detail) {
  // ... update DOM ...
  hud.classList.add('show');
  if (showGestureHUD._timer) clearTimeout(showGestureHUD._timer);
  showGestureHUD._timer = setTimeout(() => hud.classList.remove('show'), 2500);
}
```

CSS for bottom-left mini HUD:
```css
.gesture-hud {
  position: fixed; left: 16px; bottom: 60px; z-index: 21;
  padding: 6px 12px; border-radius: 10px;
  background: rgba(10,10,16,.32); border: 1px solid rgba(255,255,255,.06);
  backdrop-filter: blur(16px); font-size: 11px; color: rgba(255,255,255,.60);
  opacity: 0; transform: translateY(6px); transition: all .25s;
  pointer-events: none; max-width: 200px;
}
.gesture-hud.show { opacity: 1; transform: translateY(0); }
```

## Step 7: Two-Hand Identification

Landmarks are unlabeled (the model returns them in arbitrary order). Identify left vs right by X position:
```javascript
if (allHands.length >= 2) {
  var l = allHands[0][0], r = allHands[1][0];
  if (l.x > r.x) { allHands.reverse(); }  // sort: leftmost = left hand
  var leftHand = allHands[0];
  var rightHand = allHands[1];
}
```

## Pitfalls

1. **Overlapping gesture states** — A fist also has a small pinch distance. Check `isPinch` BEFORE `isFist`.
2. **Landmark index confusion** — MediaPipe landmark indices are: 0=wrist, 4=thumb tip, 8=index tip, 12=middle tip, 16=ring tip, 20=pinky tip. The first values in each finger group are the bases (5, 9, 13, 17).
3. **Camera mirroring** — Webcam image is mirrored. Mirror hand X: `smoothX = 1 - rawX`.
4. **CDN loading** — MediaPipe CDN files can be slow in China. Consider bundling the WASM model files locally.
5. **maxNumHands performance** — Setting `maxNumHands: 2` roughly doubles per-frame processing. On low-power machines, use `modelComplexity: 0` or reduce the video resolution to 320x240.
6. **Confidence thresholds** — Start with `minDetectionConfidence: 0.6, minTrackingConfidence: 0.6`. If too many false detections, raise to 0.7-0.8. If hands are often lost, lower to 0.5.
