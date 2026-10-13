# Shader Gesture Control: Atomic Orbital Effect

> Pattern from Mineradio Extended gesture v5 — GLSL vertex shader modification for hand-tracked particle orbit.

## Problem

Default particle shaders handle "grip" by contracting particles toward center (`pos.xy *= 0.66`). Users want atomic orbital motion — particles smoothly orbiting around the center like electrons around a nucleus.

## Solution: Replace Contraction Block with Orbital Math

### Before (contraction)
```
if (uGestureGrip > 0.001) { float grip = clamp(uGestureGrip, 0.0, 1.0); pos.xy *= mix(1.0, 0.66, grip); }
```

### After (orbital)
```
if (uGestureGrip > 0.001) {
    float grip = clamp(uGestureGrip, 0.0, 1.0);
    float orbitAngle = uTime * 0.6 * grip;
    float r = length(pos.xy);
    float pa = atan(pos.y, pos.x);
    float na = pa + orbitAngle * (0.5 + 0.5 * hash11(aRand));
    pos.x = r * cos(na);  pos.y = r * sin(na);
    pos.xy *= 1.0 + grip * 0.12 * (0.5 + 0.5 * hash11(aRand * 73.1));
    pos.z += grip * 0.25 * sin(uTime * 0.6 * (1.0 + 0.3 * hash11(aRand * 13.1)) + aRand * 6.28315);
}
```

## Key parameters
| Param | Value | Effect |
|-------|-------|--------|
| orbitAngle speed | `uTime * 0.6 * grip` | Base rotation rate |
| Per-particle speed | `0.5 + 0.5 * hash11(aRand)` | 50-100% of base — not synchronous |
| Orbit spread | `grip * 0.12` | Drift outward by up to 12% of radius |
| Z-float | `grip * 0.25 * sin(...)` | Gentle vertical bobbing |

## Finding shader blocks in a monolith
Search for the uniform name: `grep -n "uGestureGrip\|uniform float" index.html`. Uniforms declared at top of `<script>`, used in vertex shader `main()`.

## Safety rules for shader modification
1. Only replace the inner `if` block — never restructure the function
2. GLSL compiles at runtime; a syntax error silently kills particle animation
3. Test each gesture state separately (the block only activates when uniform > 0)
4. Use `hash11()` for per-particle randomness — deterministic, no flicker
5. Prefer `sin(uTime * freq)` over `snoise()` — cheaper and more predictable

## JS-side handshake
```javascript
if (uniforms.uGestureGrip)
    uniforms.uGestureGrip.value += (gestureGrip.target - uniforms.uGestureGrip.value) * 0.12;
```
Targets: fist=0.75-0.95, open=0, pinch=0.34.

## Transition guard (one-shot actions)
```javascript
if (isOpen && gestureGrip.lastState !== 'open') {
    setPreset(0, { silent: true, skipTransition: false, noSave: true });
    gestureGrip.target = 0;
}
gestureGrip.lastState = 'open';
```
Without the guard, `setPreset(0)` fires every frame, locking the preset and preventing subsequent gestures from changing it.
