# Mineradio: Adding a New Music Provider

When adding a new music source (e.g., Kugou, Qishui) to Mineradio, you must update these locations.

## Backend (server.js)

### 1. Handler functions — insert before the HTTP server section (~L3243)
Follow the pattern of existing handlers (QQ music handlers are the best template):
```
async function handleNewSearch(kw, limit) { ... }
async function handleNewSongUrl(id, quality) { ... }
async function handleNewLyric(id, duration) { ... }
```

For Kugou specifically, use `requestKugouJson()` helper (wraps `requestText` with proper headers).

### 2. Routes — insert in the if/else chain (~L3400+)
Follow existing pattern:
```
if (pn === '/api/newprovider/search') { ... sendJSON(res, { provider: 'new', songs }); return; }
if (pn === '/api/newprovider/song/url') { ... }
if (pn === '/api/newprovider/lyric') { ... }
```

### 3. Audio proxy headers (~L2342)
Update `audioProxyHeadersFor()` to include the new provider's domain for Referer headers.

## Frontend (index.html)

CRITICAL: Make changes ONE AT A TIME with brace-balance verification after each.

### Phase 1: Data flow (safest, no UI)
1. **Add login status var** (~L2710):
   `var newLoginStatus = { provider: 'new', loggedIn: true/false, ... };`
2. **Update `platformMeta()`** (~L23248): add `if (provider === 'new') return { key, short, label, app, dot };`
3. **Update `platformStatus()`** (~L23252): add `if (provider === 'new') return newLoginStatus;`
4. **Update `hasAnyPlatformLogin()`** (~L23289): add `|| hasPlatformLogin('new')`
5. **Update `firstLoggedProvider()`** (~L23292): add fallback for new provider
6. **Update `songProviderKey()`** (~L17532): detect new provider from song object
7. **Update `songSourceTagHtml()`** (~L17536): render label for new provider
8. **Inject into playlist fetch** (~L19445): add API call to `Promise.all()`, merge results into `userPlaylists`
9. **Update `playlistPanelKey()`** (~L19462): handle new provider in key generation

### Phase 2: Track loading (4 locations)
All 4 use the pattern `provider === 'qq' ? apiJson('/api/qq/playlist/tracks?...') : apiJson('/api/playlist/tracks?...')`. Add new provider ternary BEFORE the default fallback:

```
: (provider === 'new' ? apiJson('/api/new/songs?...') : apiJson('/api/playlist/tracks?...'))
```

Locations:
- ~L14428: `ShelfContentLoadApi` function
- ~L19563: `loadPlaylistPanelDetail` function
- ~L19861: `loadPlaylistShelfContent` or similar
- ~L17142: standalone playlist fetch

### Phase 3: Login UI (most fragile)
1. **HTML buttons** (~L2469): add `<button>` to `login-platform-tabs`
2. **`setLoginProvider()`** (~L23444): add `else if (provider === 'new') loginProvider = 'new';`
3. **`updateLoginProviderUi()`** (~L23449):
   - Add `document.getElementById('login-provider-new')` reference
   - Toggle active class
   - Add title/desc for new provider
   - Keep QR logic untouched (let it hide naturally for non-auth providers)

### Phase 4: CSS (~L197-805 region)
1. Add CSS color variable: `--source-new:#HEXCOLOR` in `:root` block (~L24)
2. Add active state: `.login-platform-tabs button.new.active{background:rgba(R,G,B,.16);color:#COLOR}`
3. Add tag color: `.tag-source.new{color:#...;border-color:rgba(...);background:rgba(...)}`
4. Add search hover: `.search-result.new-source:hover{background:rgba(...)}`

### Phase 5: Search integration
Update `fetchMusicSearchResults()` (~L17672): add `if (mode === 'new') { ... }` block.

## Provider Identity Convention

| Provider | Color | Short | API prefix |
|----------|-------|-------|------------|
| netease | `#d95b67` | NE | `/api/` |
| qq | `#00F5D4` | QQ | `/api/qq/` |
| kugou | `#FF6B35` | KG | `/api/kugou/` |
| qishui | `#FF2A6D` | QS | `/api/qishui/` |

## Verification Sequence

After ALL changes:
1. `node --check server.js`
2. Brace balance on index.html (compare to original)
3. `grep -c $'\\\\n' index.html` → should be 0
4. External API smoke tests (curl the new endpoints)
5. Kill + restart Mineradio, check process count
6. ASK USER to visually confirm UI
