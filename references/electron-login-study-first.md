# Electron Login Implementation: Study-First Protocol

> Learned 2026-07-22 from failing to implement Kugou login before understanding how Netease/QQ login works.

## The Anti-Pattern (What Went Wrong)

1. **Assumed "wait for window close → grab all cookies" would work** — it doesn't because tracking cookies fire before auth cookies
2. **Navigated to homepage instead of dedicated login page** — `www.kugou.com` is a full site, not a login page; Netease uses `music.163.com/#/login`
3. **Tried to auto-close the window on cookie detection** — the detection was too eager (matched analytics cookies)
4. **Didn't check if the provider's API even supports cookie-based auth** — Kugou's user playlist API returns "Access Deny" regardless of cookies
5. **Modified main.js + preload.js without full Electron restart** — `taskkill //F //IM App.exe && sleep 3 && relaunch` is mandatory, not optional
6. **showLoginModal() silently forced new providers to netease** — the hardcoded ternary `opts.provider === 'qq' ? 'qq' : 'netease'` is a gatekeeper

## The Correct Protocol

### Phase 0: Search Before Build ⭐ DO THIS FIRST
Before writing ANY login code (or any integration code):

1. **Search GitHub for forks/extensions**: `{appname} extended`, `{appname} fork`, `{appname} plugin`
2. **Search Douyin for 二创 videos**: Chinese developers often share Mineradio/Electron forks on 抖音
3. **Check results deeper (page 3-5)**: The best fork may not be the top result
4. **If a working fork exists**: `git clone --depth 1 <url>` and compare:
   - Check `server.js` for new API routes (`grep -n "provider" server.js`)
   - Check `package.json` for additional npm deps (`npm install sql.js` etc.)
   - Check `desktop/main.js` for native changes
   - Copy the entire directory and test before customizing
5. **If no fork exists**: proceed to Phase 1

### Phase 1: API Feasibility Assessment
Before writing ANY login code:

1. **Can the provider's API be accessed without auth?** Test search, lyrics, public playlists
2. **Does the provider have a cookie-based auth flow for its API?** Not all providers expose mobile/web APIs that accept browser cookies
3. **What specific cookies indicate login?** For Netease: `MUSIC_U`. For QQ: `qm_keyst`. Don't guess — inspect cookies after manual browser login.

### Phase 2: Study an Existing Working Login
Pick a provider that already works (Netease, QQ) and trace the entire flow:

| Checkpoint | Netease Pattern | QQ Pattern |
|------------|----------------|------------|
| Login URL | `https://music.163.com/#/login` (dedicated) | `https://y.qq.com/n/ryqq/profile` (dedicated) |
| Cookie checking | Poll every 1.2s for `MUSIC_U` | Poll every 1.2s for `uin` + `qm_keyst` |
| Detection function | `neteaseCookieHasLogin()` | `qqCookieHasPlaybackLogin()` |
| Pre-check | `if (neteaseCookieHasLogin(initialCookie)) return { reused: true }` | Same pattern |
| Window close | `resolve(neteaseCookieHasLogin(cookie) ? ... : { cancelled: true })` | Similar fallback |

### Phase 3: Implement Following the Pattern EXACTLY
1. Same structure (pre-check → dedicated URL → polling → finish pattern)
2. Same polling interval (1.2-2 seconds)
3. Same cookie detection approach (specific names, not patterns)
4. Same `finish()` semantics (settled flag, clearInterval, closeWindow, resolve)
5. **After modifying main.js/preload.js**: `taskkill //F //IM App.exe && sleep 3 && relaunch`

### Phase 4: Fallback If API Is Inaccessible
If the provider's API denies access regardless of login method:
- **Stop trying to implement login.** Switch to manual cookie import panel.
- This is the same pattern QQ Music uses (`qq-cookie-panel`)
- Frontend: show a `<textarea>` for cookie input + "保存" button
- Backend: accept any cookie string, save to file, validate minimally

## When to Give Up on QR Login

- API returns "Access Deny" to authenticated requests
- Login page uses iframes/WebSocket/token-based auth (not cookies)
- Provider blocks non-browser User-Agent headers
- Streaming URLs require app-level credentials (not user cookies)
- **The provider's mobile API requires signature/crypto (Kugou case)** → use existing fork that already reverse-engineered it

## Electron-Specific Traps

| Trap | Symptom | Fix |
|------|---------|-----|
| main.js/preload.js not reloaded | Buttons render but IPC undefined | taskkill + 3s wait + relaunch |
| showLoginModal defaults new provider | New tab shows netease login | Add to ternary whitelist |
| refreshQr overwrites non-QR UI | Provider UI reverts to netease | Add early return at refreshQr top |
