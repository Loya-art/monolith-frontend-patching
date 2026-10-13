# MediaPipe Gesture Effect Architecture

> Deployed 2026-07-22 from `Beverly621/spatial-ar-gesture-demo` — browser-based recreation of Instagram @wxll.hx TouchDesigner effect.

## Architecture (single HTML file, no build)

```
User Camera → MediaPipe Hands + SelfieSegmentation → Canvas 2D Filters
     ↓                   ↓                               ↓
  getUserMedia()     CDN-loaded WASM models           drawRed/Blue/Green()
```

## Three-Layer AI Pipeline

All models load from jsDelivr CDN at runtime — no server, no build:

### 1. Hands (手部追踪)
- `@mediapipe/hands` — 21 landmarks per hand, 2 hands max
- Options: `modelComplexity: 1`, `minDetectionConfidence: 0.6`
- Callback fires per frame with `multiHandLandmarks`

### 2. SelfieSegmentation (人像分割)  
- `@mediapipe/selfie_segmentation` — person/background mask
- Renders to offscreen canvas at full resolution for pixel-accurate sampling
- Edge blur: `filter: blur(${Math.max(3, width * 0.005)}px)` for anti-aliasing

### 3. Canvas 2D Filter Rendering

Three quadrilateral zones formed by 8 landmarks (both hands × 4 fingertips):

| Zone | Landmarks | Effect | Technique |
|------|-----------|--------|-----------|
| 🔴 Red | thumb→index (both hands) | Halftone dots | `ctx.clip()` + pattern fill with per-pixel alpha from mask |
| 🔵 Blue | index→middle (both hands) | Thermal noise | Blue fill + screen blend + random dot noise |
| 🟢 Green | middle→pinky (both hands) | Ghost overlay | `getBaseFrameSample()` → color sampling → blended draw |

## Key Canvas Rendering Patterns

### Quadrilateral clipping
```javascript
ctx.save();
ctx.beginPath();
ctx.moveTo(pts[0].x, pts[0].y);
pts.slice(1).forEach(p => ctx.lineTo(p.x, p.y));
ctx.closePath();
ctx.clip();
ctx.restore();
```

### Per-pixel mask sampling
```javascript
const index = (y * maskCanvas.width + x) * 4;
const alpha = maskSample.data[index] / 255;
```

## Start-Up Flow
```
btnStart.click → initModels() → getUserMedia() → video.onloadedmetadata → processVideoLoop() + renderLoop()
```

## CDN Loading
Models from `https://cdn.jsdelivr.net/npm/@mediapipe/hands/` — requires internet, cached by browser.

## Mineradio Extended 内置手势集成

自 2026-07-22 起，Mineradio Extended 手势系统已用于 Two-hand particle control。

### 加载
```javascript
maxNumHands: 2
modelComplexity: 1
minDetectionConfidence: 0.7
```

### GLSL Uniforms
`uHandActive`(活跃度), `uHandXY`(位置), `uGestureGrip`(收缩), `uBurstAmt`(爆发)

### 手势映射表
| 手势 | 条件 | Uniform | 效果 |
|------|------|---------|------|
| 移动 | openness 0.28-0.72 | `applyParticleSpinDrag` | 粒子跟随旋转 |
| 捏合 | pinchDist < 0.075 | `uGestureGrip: 0.34` | 粒子聚集旋转 |
| 张开 | openness > 0.72 | `uBurstAmt: 0.68` | 粒子爆炸散开 |
| 握拳 | gripTarget > 0.68 | `uGestureGrip: 0.55+` | 粒子收缩 |

### 双手处理
```javascript
function processHandFrame(allHands) {
  if (!allHands.length) { onHandLost(); return; }
  if (allHands.length >= 2) sortByX(allHands);
  // Main hand drives orbit + gesture, second hand reserved
}
```

## Deployment (any static server)
```bash
npm start  # PORT=5173
```
⚠️ Must serve via `http://localhost:PORT`, not `file://`.
