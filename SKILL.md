---
name: monolith-frontend-patching
description: Safely modify large monolithic single-file HTML/JS apps (like Mineradio's 26K-line index.html). Use when patching inline JS/CSS in a buildless single-file SPA served by Electron or a local server.
version: 1.0.0
metadata:
  hermes:
    tags: [frontend, monolith, patching, minified, electron, safety]
---

# Monolith Frontend Patching

Modify large, buildless single-file HTML/JS apps without breaking them.

## When to Use

- The app bundles ALL frontend code (CSS, HTML templates, JS logic) in one file — typically `index.html`, often 15K+ lines and 1MB+
- No build step, no module system, no component separation
- Served by Electron (loaded from disk) or a local Node HTTP server
- Minified or semi-minified — CSS/JS on single lines with minimal whitespace

**MANDATORY before coding any feature:** Read the [study-first protocol](references/electron-login-study-first.md). The user explicitly requires: learn first, practice only after understanding. Never write code without first understanding how the existing working code does it.

**MANDATORY search-first (prevents wasted hours):** Before writing ANY code for a new provider or feature:
1. Search GitHub for `ProjectName-Extended`, `ProjectName-fork`, or `ProjectName-plugin` — someone may have already done the work
2. Search Douyin/Bilibili for the project name + "二创" (fork/remix) or "教程" (tutorial)
3. Check the project's GitHub Issues/Discussions for relevant forks
This single step saved hours of dead-end Kugou API work in a real session — the public API was locked, but an Extended fork had reverse-engineered the mobile API.

**MANDATORY search-first:** Before writing ANY code for a new provider or feature, search GitHub for AppName-Extended/fork/plugin and search Douyin for AppName erchuang tutorials. Someone may have already done the work. This single step saved hours of dead-end Kugou API work.

**MANDATORY code quality:** "不要堆砌代码，代码不是写得越多越好，而是要学得写得精。" Prefer 4-5 targeted data-flow patches over bulk replacements.

## Safety Rules

### 1. Optional API calls MUST have .catch() in Promise.all

When adding a non-critical API call to a `Promise.all()` array, it MUST be wrapped with `.catch()` to prevent a single API failure from collapsing the entire page section:

```javascript
// ✅ SAFE — one API failing won't block the whole page
var result = await Promise.all([
  apiJson('/api/playlists').catch(function(){ return { playlists: [] }; }),
  apiJson('/api/kugou/rank/list').catch(function(){ return { ranks: [] }; })
]);

// ❌ DANGER — if /api/kugou/rank/list fails, ALL playlists disappear
var result = await Promise.all([
  apiJson('/api/playlists'),
  apiJson('/api/kugou/rank/list')
]);
```

### 2. Button active-state toggles must be comprehensive

When adding new provider buttons AND modifying `updateLoginProviderUi` with an early-return block, BOTH code paths must toggle ALL buttons. Missing one path causes double-highlight:

```javascript
function updateLoginProviderUi() {
  // Early-return for new provider — toggle ALL 4
  if (loginProvider === 'kugou') {
    neteaseBtn.classList.toggle('active', false);
    qqBtn.classList.toggle('active', false);
    kugouBtn.classList.toggle('active', true);
    qishuiBtn.classList.toggle('active', false);
    return;
  }
  // Main code path — ALSO toggle ALL 4
  if (neteaseBtn) neteaseBtn.classList.toggle('active', loginProvider === 'netease');
  if (qqBtn) qqBtn.classList.toggle('active', isQQ);
  if (kugouBtn) kugouBtn.classList.toggle('active', loginProvider === 'kugou');
  if (qishuiBtn) qishuiBtn.classList.toggle('active', loginProvider === 'qishui');
}
```

### 3. One change, one verify

After EVERY patch to the frontend file, run a brace-balance check:

```
node -e "
const html = require('fs').readFileSync('index.html','utf8');
const js = (html.match(/<script>([\\s\\S]*?)<\\/script>/g)||[]).map(s=>s.replace(/<\\/?script>/g,'')).join('\\n');
let b = 0;
for (const c of js) { if (c === '{') b++; if (c === '}') b--; }
console.log('Braces:', b === 0 ? 'OK' : 'FAIL ' + b);
"
```

**Pre-existing paren imbalance is normal.** The original file may show -7 or similar from template literals, regexes, and JSX-like strings. Only NEW imbalance from your changes is a problem. Compare against the original to confirm.

### 4. Limit to 4-5 data-flow only changes

**REAL-WORLD LESSON (2026-07-22)**: A Python script with 22 bulk replacements crashed the app. 4 targeted `patch` calls succeeded. The only safe approach:

- **Only modify data-flow paths**: where data is fetched (`Promise.all` additions), how it's mapped (key/label functions), and where it's loaded (API endpoint ternaries)
- **NEVER touch**: UI rendering logic, login flows, CSS, complex conditional chains, or functions with 10+ local variables
- **Each change = one `patch` call** with exact old_string/new_string matching
- **Verify brace balance after each** — don't batch them
- **If a change looks "complex" — it IS complex.** Find a simpler way.

### 5. The early-return pattern for complex functions

When you MUST modify a complex multi-branch function (e.g. `updateLoginProviderUi` with 20+ lines of QR/status/shell logic), **NEVER restructure the existing branches**. Instead, add simple if-blocks at the top that handle new cases and `return` early:

```
function complexFunction(param) {
  // --- ADD: early return for new case ---
  if (param === 'newThing') {
    document.getElementById('some-element').textContent = 'New thing';
    document.getElementById('other-element').innerHTML = 'Description';
    return;  // ← KEY: leaves existing code untouched
  }

  // --- existing complex logic below, completely untouched ---
  var isQQ = param === 'qq';
  // ... 40 lines of QR code, auth, shell toggles ...
}
```

This pattern was proven safe in production — it adds functionality without touching the risky existing code.

### 6. Maintain a rollback path

Extract the original from the release archive:
```
unzip -o AppName-X.X.X.zip "AppName-X.X.X/public/index.html" -d /tmp/restore/
cp /tmp/restore/AppName-X.X.X/public/index.html /path/to/running/app/public/index.html
```

### 7. Check for patch serialization artifacts

Literal `\n` (backslash + n) can appear at code boundaries. Most legitimate `\n` are inside string literals — check context:
- **Inside quotes**: legitimate string escape
- **At code boundary (after `{`/`}`/`;`)**: artifact — fix with `sed -i 's/\\\\n/\n/g' index.html`

**PREVENTION**: When the `patch` tool's `new_string` contains multi-line JS, the tool can serialize literal `\n` into the file instead of actual newlines. To avoid this:
- Keep `old_string`/`new_string` to <20 lines each
- After every patch, run: `grep -c $'\\\\n' <file>` — if non-zero and matches are at code boundaries (not inside strings), run the sed fix above
- Verify with `node --check`

### 8. Keep a session-changes backup

When modifying a minified SPA across many patches, the file WILL get corrupted eventually and you'll need to restore from backup. Maintain a working backup:
```bash
cp index.html index.html.safe-base    # before first change
cp index.html index.html.safe-v2      # after every few successful patches
```
This prevents the "lost all changes because I had to restore the original" scenario.

### 9. Windows + Chinese path workaround

On Windows with Chinese usernames, `node --check "C:\\Users\\name\\file.js"` may fail with drive-letter corruption. Workaround:
```bash
cd /path/to/directory
node --check file.js
```

### 10. Server-side changes are safe

Backend code (Node server, API routes) is structured normally. Add handlers and routes following the existing pattern. No special safety rules needed beyond `node --check`.

### 11. When to pivot vs push through (API blocked)

When a public API returns "Access Deny", "No Actions", or similar persistent auth errors:
1. **First attempt**: Try the native Electron login window (BrowserWindow + cookie polling)
2. **Second attempt**: If login works but API still blocks user data, try manual cookie import with the same pattern
3. **Third attempt**: If the provider's API fundamentally blocks all user-data access, **pivot** — look for:
   - An existing fork/extended version that already solved it (search GitHub for `ProjectName-Extended`)
   - A mobile app API via reverse engineering (search GitHub for the provider's API wrappers)
   - A community-maintained API bridge
4. **DO NOT** keep trying the same approach with different parameters — this is what frustrates users

**Key indicator**: If the original project's existing providers (Netease, QQ) use straightforward web auth, but the new provider's web endpoints all return access-denied, the auth model is fundamentally incompatible. Don't force it — find the fork.

### 12. Delivery discipline: come back only when done

The user explicitly requires: **"你先完成，然后自行检查好，或者是还不够好，自我优化。你最好给我优化几轮，拿到一个没有问题的成果，再给我。"**

- Do NOT ping the user after every small change
- Do NOT ask "is this working?" after every patch
- Do NOT send "I fixed X, try again" — batch fixes, verify yourself, then present a complete result
- Only come to the user when: (a) the work is self-contained and tested, OR (b) you've tried 3 different approaches and all failed

| **`patch` tool fails on `${}` template literal syntax** | `Escape-drift detected` — patch tool refuses the edit | Use Python file read/write (write_file + python) instead of patch when old_string/new_string contain `${...}` template literal expressions. The patch tool's escape-detection heuristic incorrectly treats `${}` as an escaping artifact. |

| Level | Content | Issue |
|-------|---------|-------|
| 1 | Outer template literal `` `...` `` | JS parses `${}` substitutions |
| 2 | HTML attribute `onclick="..."` | Quotes interpreted by HTML parser |
| 3 | JS function call `someFunc('...')` | Single quotes in onclick value conflict |
| 4 | Nested substitution `'${...}'` | Template substitution inside attribute |

The result is fragile code that fails with "Unexpected string" syntax errors.

**FIX: Use event delegation instead.** Put `data-*` attributes on elements and handle clicks via `addEventListener` with `.closest()`:

```javascript
// ✅ SAFE — data attributes + event delegation
html += `<button class="scene-btn" 
  data-title="${e.title}" 
  data-dyn="${dynasty}" 
  data-year="${e.year}">🖼️</button>`;

// In a separate init block:
panel.addEventListener('click', function(ev) {
  var btn = ev.target.closest('.scene-btn');
  if (btn) openPanorama(btn.dataset.title, btn.dataset.dyn, parseInt(btn.dataset.year));
});
```

**Even safer: Use a separate `<script>` block.** When the main script is 3000+ lines, add a small third `<script>` block at the end of `<body>`:
- Zero risk of breaking the main script
- No complex template literal nesting to debug
- The main script's syntax error won't prevent the new script from running
- Simple string concatenation works instead of nested template literals

### 14. Validate JS with `node --check` extract pattern

After patching a large HTML file, the browser silently skips the entire script block on syntax errors. Use this pattern to catch them early:

```bash
python -c "
import re
with open('index.html', 'r', encoding='utf-8') as f:
    c = f.read()
scripts = list(re.finditer(r'<script>(.*?)</script>', c, re.DOTALL))
with open('main_script.js', 'w', encoding='utf-8') as f:
    f.write(scripts[1].group(1))
" && node --check main_script.js
```

This catches: unterminated strings, unescaped quotes, broken regex literals, and other syntax errors long before they reach the browser.

### 15. Home page section pattern add (home-rail ranking)

When adding a new content section below the existing card grid:
1. Add HTML with the same CSS class as existing sections (`home-rail`, `home-tile-row`) between existing sections
2. Write a `renderXxx()` function that fetches data and populates the tile row
3. Call it at the end of `renderHomeDiscover()` (the main page re-render function) — NOT inside it
4. Use `.catch()` to hide the section on failure: `if (!ranks.length) { rail.style.display = 'none'; return; }`
5. Each tile follows the `home-tile` class pattern: art image + label + title
6. Assign a global click handler per tile that loads content into the main player

### 16. Pen/tablet drag fix: PointerEvent → MouseEvent bridge (CEP/Electron/Chromium)

**When to use**: App works with a mouse but a pen/tablet (Windows Ink, Wacom, 高漫) can only click — pressing and dragging does nothing. Root cause: legacy code binds only `mousedown/mousemove/mouseup`, but Windows Ink pen input in Chromium (Adobe CEP panels, Electron) produces **PointerEvent**; the compatibility mousemove stream is missing/broken, so drag logic never fires.

**The fix — DO NOT rewrite drag logic. Bridge pointer events into synthesized mouse events** (proven on Coolorus 2.5.17 CEP plugin, PS2026 + 高漫 tablet):

```javascript
(function () {
    'use strict';
    if (typeof window.PointerEvent === 'undefined') return; // legacy env, skip
    var dragging = false, target = null;

    function makeMouseEvent(type, pe) {   // coordinates passed through untouched
        var props = { bubbles: true, cancelable: true, view: window, detail: 1,
            screenX: pe.screenX || 0, screenY: pe.screenY || 0,
            clientX: pe.clientX || 0, clientY: pe.clientY || 0,
            ctrlKey: !!pe.ctrlKey, shiftKey: !!pe.shiftKey,
            altKey: !!pe.altKey, metaKey: !!pe.metaKey,
            button: type === 'mouseup' ? 0 : (pe.button || 0),
            buttons: type === 'mouseup' ? 0 : (pe.buttons || 1) };
        try {
            var ev = new MouseEvent(type, props);
            try { // offsetX/offsetY not settable in constructor — patch manually
                Object.defineProperty(ev, 'offsetX', { value: pe.offsetX || pe.clientX, enumerable: true });
                Object.defineProperty(ev, 'offsetY', { value: pe.offsetY || pe.clientY, enumerable: true });
            } catch (e) {}
            return ev;
        } catch (e) { // ancient engine fallback
            var ev2 = document.createEvent('MouseEvent');
            ev2.initMouseEvent(type, true, true, window, 1, props.screenX, props.screenY,
                props.clientX, props.clientY, props.ctrlKey, props.altKey, props.shiftKey,
                props.metaKey, props.button, null);
            return ev2;
        }
    }
    function onDown(pe) {
        if (pe.pointerType === 'mouse') return;      // mouse keeps native events — never bridge it
        var t = pe.target || document.body;
        pe.preventDefault();                          // suppress system compat mouse events (prevents double-fire)
        dragging = true; target = t;
        try { t.setPointerCapture(pe.pointerId); } catch (e) {}
        t.dispatchEvent(makeMouseEvent('mousedown', pe));
    }
    function onMove(pe) {
        if (pe.pointerType === 'mouse' || !dragging || !target) return;
        pe.preventDefault();
        target.dispatchEvent(makeMouseEvent('mousemove', pe)); // capture keeps it flowing outside the element
    }
    function onUp(pe) {
        if (pe.pointerType === 'mouse' || !dragging || !target) return;
        var t = target;
        pe.preventDefault();
        try { t.dispatchEvent(makeMouseEvent('mouseup', pe)); } catch (e) {}
        try { t.releasePointerCapture(pe.pointerId); } catch (e) {}
        dragging = false; target = null;
    }
    // capture phase + document-level: covers every interactive element regardless of layout
    document.addEventListener('pointerdown', onDown, true);
    document.addEventListener('pointermove', onMove, true);
    document.addEventListener('pointerup', onUp, true);
    document.addEventListener('pointercancel', onUp, true);
})();
```

Key points:
- **`pointerType === 'mouse'` → return**: mouse keeps its native events; only pen/touch are bridged. Bridging mouse would double-fire every drag.
- **`setPointerCapture`/`releasePointerCapture`**: pen drag keeps delivering `pointermove` to the element even when the nib leaves it — required for drag-outside-element color picking.
- **`preventDefault()` on pointer events** suppresses the browser's own (broken/partial) compatibility mouse events, preventing double triggers.
- **`touch-action: none` CSS** on interactive elements is also needed — inject a `<style>`: `#picker_canvas, #color_dragger { touch-action: none !important; }`.
- Synthesized events carry real `clientX/clientY`, so frameworks like EaselJS/CreateJS (`stageX/stageY`) and jQuery compute positions exactly as with a mouse — **zero changes to existing color/drag math**.
- **Testing without the hardware**: build a test harness page that loads the patch, binds dummy mousedown/move/up listeners, dispatches synthetic `new PointerEvent('pointerdown', {pointerType:'pen', pointerId:7, clientX.., buttons:1})` sequences, and asserts the mouse listeners fire. Verify via browser_navigate to the file:// harness + browser_console log check.

### 17. Patching obfuscated single-file apps (obfuscator.io etc.)

When the monolith is minified AND obfuscated (e.g. `_0x1a9e("0x5e")` hex-string lookups, 6 lines / 1MB):

- **NEVER edit inside the obfuscated blob.** String lookups are indirect (`_0x1a9e("0x..")` table lookups); one wrong byte breaks the whole IIFE and you can't reliably find/replace anything.
- **Inject an external file instead** — append `<script src="js/your-fix.js"></script>` at the very end of the HTML (after the last `</script>`). A separate script block:
  - runs after the app's own scripts, so DOM/globals already exist;
  - cannot corrupt the obfuscated code — zero risk to existing logic;
  - is readable, versionable, and rollback = delete one line + one file.
- **Use a readable same-family source as reference**: if a newer version of the same app exists unobfuscated (e.g. Coolorus v2.6.0's `main.js` on disk), read IT to learn the real event names/flow (`handlePaletteMouseDown`, `stagemousemove`, etc.) — far cheaper than deobfuscating the shipped blob.
- **Deobfuscation steps if you truly must edit the blob** (last resort): (1) format with `npx js-beautify`; (2) the `_0x1a9e("0x..")` lookup function + string array are self-contained in the file — run the decoder in Node or use an online tool like deobfuscate.io to inline plaintext strings; (3) map back to real method names using the readable reference version; (4) re-minify with `npx terser`. For 1MB blobs this is high-risk/low-reward — prefer the injection approach.
- **Where CEP panels actually load from**: `C:\Program Files (x86)\Common Files\Adobe\CEP\extensions\<bundle-id>\` (NOT `Adobe\CEP\extensions\` — that often holds only installers). Manifest: `CSXS/manifest.xml` → `MainPath` = index.html. CEP = Chromium, so PointerEvent/setPointerCapture are fully supported in modern PS versions.
- **PS must be fully restarted** after patching a CEP panel (index.html is cached).
- Full case study (event-name map, CEP install locations, UAC write-permission trap, test harness): [references/coolorus-cep-pointer-fix.md](references/coolorus-cep-pointer-fix.md)

## Testing Without Screenshots

When you can't capture the app visually (computer-use session expired, remote setup):
1. **Structural checks**: grep for new HTML elements, CSS classes, JS function references
2. **Brace balance**: confirm no new syntax errors
3. **External API smoke tests**: curl the new endpoints independently
4. **Process check**: confirm the app starts and runs (no crash loops)
5. **ASK THE USER**: the final verification is the user seeing the UI change

**Script-based fallbacks (for when `patch` fails):**  
- [references/node-script-replacement.md](references/node-script-replacement.md) — write_file + node -e as alternative to the patch tool for large blocks, multi-line replacements, and Chinese-path workaround.
- [references/python-script-replacement.md](references/python-script-replacement.md) — write_file + python as alternative when patch tool fails on template literals, nested quotes, or Chinese chars (Windows: use `python`, not `python3`).

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Broke JS syntax with sed | App stuck on loading screen, blank page | Restore from original, redo with patch tool |
| Added too many changes at once | Hard to identify which change broke it | Revert, apply one at a time with interleaved verification |
| Literal `\n` in patched code | Syntax error, brace imbalance | `sed -i 's/\\\\n/\n/g' index.html` |
| Modified complex multi-branch function | Hard-to-debug runtime errors | Keep changes minimal — add if/else branches, don't restructure |
| Touched login/QR generation code | QR polling breaks, modal hangs | Avoid touching `refreshQr()`, `openProviderWebLogin()`, QR status logic |
| Optional API call without .catch() in Promise.all | One failing API takes down the entire page section | Wrap optional calls: `apiJson(...).catch(function(){ return []; })` |
| Button toggles only in one code path | Double-highlight when switching providers | Toggle ALL buttons in BOTH the early-return block AND the main function body |
| **Windows: `python3` exit code 49** | Script doesn't run on Windows | Use `python script.py` instead — Windows aliases `python3` inconsistently via Microsoft Store redirect |
| **Electron restart missed** | Login buttons render but native IPC is undefined | `taskkill //F //IM App.exe && sleep 3 && relaunch` |
| **showLoginModal defaults new provider to netease** | Clicking new tab shows netease login | Add provider to `showLoginModal()` whitelist ternary |
| **refreshQr destroys non-QR provider UI** | Kugou UI briefly correct then reverts to netease QR | Add early return at top of `refreshQr()` for non-QR providers |
| **Chunk storage: result index shifts** | Wrong data after adding to Promise.all | Verify ALL `result[N]` indices after reordering |
| **Electron reload: preload.js + main.js unchanged at runtime** | New login buttons click but don't open windows; new IPC channels silently fail | Electron loads preload/main ONLY at startup. After modifying these files, you MUST `taskkill //F //IM App.exe` and restart — a simple page refresh won't pick up the new native code. |
| **showLoginModal defaults new providers to netease** | Clicking "酷狗" tab shows Netease login, not Kugou | The function `showLoginModal(opts)` has `loginProvider = opts.provider === 'qq' ? 'qq' : 'netease'` — this forces ANY new provider back to netease. Fix: add the new provider to the ternary. |
| **Chunk storage — result index shifts when adding to Promise.all** | After login, playlists disappear or wrong provider's playlists show | When you add a new entry to `Promise.all([A, B, C])` → `Promise.all([A, B, C, D])`, the index of the NEW entry is 3. But if you also REORDER entries, ALL result indices shift. Update ALL `result[N]` references. |
| **App restore loses all patches — keep incremental backups** | After a crash, you restore the original and ALL changes are gone | Save `index.html.safe-vN` after every 3-4 successful patches. The 1.3MB overhead is negligible compared to redoing 2+ hours of work. |

### Pitfall: showLoginModal default-provider trap

When `showLoginModal()` is called without opts, or with a provider it doesn't recognize, it falls through to netease. This silently overrides the user's tab selection. The fix:

```javascript
// BEFORE — kugou/qishui forced to netease
function showLoginModal(opts) {
  if (opts.provider) loginProvider = opts.provider === 'qq' ? 'qq' : 'netease';
  // ...
}

// AFTER — new providers preserved
function showLoginModal(opts) {
  if (opts.provider === 'qq') loginProvider = 'qq';
  else if (opts.provider === 'kugou') loginProvider = 'kugou';
  else if (opts.provider) loginProvider = 'netease';
  // ...
}
```

## Safe Entry Points (from least to most risky)

| Entry Point | Risk | What you can do |
|-------------|------|-----------------|
| **HTML elements** (buttons, divs, spans) | 🟢 None | Add pure markup — elements that reference non-existent JS functions just silently fail on click |
| **CSS variables & selectors** | 🟢 None | Colors, backgrounds, borders — can't break execution |
| **server.js** (structured Node code) | 🟢 Low | Add functions + routes using patch tool, verify with `node --check` |
| **Data-flow JS** (Promise.all, key mappers, labels) | 🟡 Medium | Inject into existing arrays/ternaries — 4-5 patches max, verify each with brace balance |
| **Login UI functions** (updateLoginProviderUi, setLoginProvider) | 🔴 High | ONLY use early-return pattern — never modify the existing function body |
| **QR generation / polling** (refreshQr, openProviderWebLogin) | 🔴 Extreme | DO NOT TOUCH — one misplaced call can freeze the entire login modal |

**External Reference — use instead of building from scratch:**
**External Reference — use instead of building from scratch:**  \n- [references/mineradio-extended.md](references/mineradio-extended.md) — daaimengermengzhu/Mineradio-Extended: full Kugou/Qishui via reverse-engineered mobile API (QR login, playlists, streaming)  \n- [references/mineradio-builtin-gesture.md](references/mineradio-builtin-gesture.md) — Built-in MediaPipe hand tracking (gestureRotation particle group orbit, pinch drag, two-hand zoom). Check what exists BEFORE writing custom gesture code.

**Shader modification pattern — atomic orbital effect in GLSL:**  
- [references/mineradio-shader-gesture-control.md](references/mineradio-shader-gesture-control.md)

**Gesture integration pattern — adding hand tracking to a particle visualizer:**
- [references/mineradio-gesture-integration.md](references/mineradio-gesture-integration.md) — MediaPipe hand tracking → existing Three.js particle system: open palm resets view, fist spins particles (atomic orbit), thumb-index pinch zooms, two hands pan, single hand orbits like mouse drag

## Provider Pattern (for apps with multiple music/data sources)

When adding a new source/provider to an existing app:
1. **Backend (server.js)**: Add handler functions + API routes, following the existing if/else chain
2. **Frontend (index.html)**: 
   - Add HTML buttons/elements for the new provider
   - Update `platformMeta()`, `platformStatus()`, `songProviderKey()`, `songSourceTagHtml()`
   - Add CSS color variables and active-state classes
   - Update playlist/track loading functions to route to new API endpoints
   - Update labels in rendering functions (`currentItems()`, etc.)
3. **Never restructure**: add branches to existing ternaries/if-else, don't rewrite the control flow

## Verification Checklist

- [ ] `node --check` passes on extracted script blocks (use Python regex + `node --check main_script.js`)
- [ ] JS brace balance matches original (or improved)
- [ ] No literal `\n` in code (outside JS string escapes)
- [ ] New HTML elements present (grep for IDs)
- [ ] New CSS classes present (grep for selectors)
- [ ] External API endpoints respond correctly (curl test)
- [ ] App process starts without crash loops
- [ ] **Electron full restart**: for `main.js`/`preload.js` changes, kill + restart app (not just refresh)
- [ ] User confirms UI looks correct

---

## ⚠️ CRITICAL: Electron Native Code Never Picks Up Without Restart

`main.js` and `preload.js` are Electron native modules loaded ONCE at process start. A browser refresh (`Ctrl+R`) or page reload does NOT reload them. After modifying either file:

```bash
# 1. Force kill ALL Electron processes
taskkill //F //IM Mineradio.exe

# 2. Wait 3 seconds (Electron cleanup + port release)
sleep 3

# 3. Start fresh
"/path/to/Mineradio.exe"
```

Symptom when this is missed: login buttons exist but `api.openKugouMusicLogin()` is `undefined` → silently falls through to netease or manual panel.

---

## ⚠️ CRITICAL: `showLoginModal()` Has a Hardcoded Provider Whitelist

```javascript
// LINE ~23456 — THIS KILLS NEW PROVIDERS
if (opts.provider) loginProvider = opts.provider === 'qq' ? 'qq' : 'netease';
```

ANY new provider added to the login modal will silently default to netease if this line is not updated. This is a single-source-of-truth that gatekeeps ALL provider access to the login flow.

**Fix**: Add new providers to the ternary or convert to if/else.

---

## ⚠️ CRITICAL: `refreshQr()` Destroys Non-QR Provider UI

`refreshQr()` only handles 'qq' and netease. When called while `loginProvider === 'kugou'` or 'qishui', it falls through to netease QR generation, overwriting the kugou-specific title/desc/button set by the early-return block.

**Fix**: Add early return at top of `refreshQr()`:
```javascript
async function refreshQr() {
  if (loginProvider === 'kugou' || loginProvider === 'qishui') {
    updateLoginProviderUi();  // re-apply kugou UI
    return;
  }
  // ... existing QQ/Netease QR logic below
}
```

---

## ⚠️ CRITICAL: Learn the Working Pattern Before Coding Login

Netease and QQ login work because they follow a specific pattern. Study it BEFORE implementing a new login:

1. **Dedicated login URL**: Navigate to a page designed for login (NOT the main homepage)
2. **Cookie polling every 1-2 seconds**: `setInterval(checkCookies, 1200)` NOT waiting for `on('closed')`
3. **Specific cookie name detection**: `MUSIC_U` for Netease, `qm_keyst` for QQ — NOT generic patterns
4. **Isolated session**: `session.fromPartition('persist:appname-provider-login')`
5. **`finish()` pattern**: `settled = true; clearInterval(pollTimer); closeWindow(); resolve(result)`

The "wait for close → grab all cookies" approach FAILS because:
- Tracking/analytics cookies match before login completes
- Auth cookies may not persist in the session partition on close
- User doesn't know when to close the window

**Correct implementation (matching Netease/QQ):**
```javascript
// 1. Check for existing session FIRST (reuse)
const cookieSession = session.fromPartition(PARTITION);
const initialCookies = await cookieSession.cookies.get({});
if (hasLoginCookie(initialCookies)) return { ok: true, cookie, reused: true };

// 2. Open dedicated login window
const loginWindow = new BrowserWindow({ ... });

// 3. POLL every 2 seconds for auth cookies
const checkCookies = async () => {
  const cookies = await cookieSession.cookies.get({});
  const loginCookies = cookies.filter(c => 
    /specific|cookie|names/i.test(c.name) && c.value.length > 3);
  if (loginCookies.length > 0) {
    finish({ ok: true, cookie: cookies.map(c => `${c.name}=${c.value}`).join('; ') });
  }
};
const pollTimer = setInterval(checkCookies, 2000);

// 4. On window close, one final check
loginWindow.on('closed', async () => {
  clearInterval(pollTimer);
  // Final check + return remaining cookies or cancelled
});
```

**Known limitation**: If the provider's website doesn't set specific auth cookies on the root domain (uses subdomain cookies, HTTP-only, or localStorage tokens), the cookie polling won't detect the login. Fall back to manual cookie import panel for these providers.

> **Reference**: [electron-login-study-first.md](references/electron-login-study-first.md) — full protocol for assessing API feasibility and studying existing login patterns before implementing new ones.
