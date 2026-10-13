# Electron Login Polling Pattern (from Netease)

> Source: Mineradio desktop/main.js lines 400-499 (working reference)

## Recipe: Copy this pattern for any new provider login

```javascript
const PROVIDER_LOGIN_PARTITION = 'persist:appname-provider-login';
const PROVIDER_LOGIN_URL = 'https://provider.com/login-page'; // ← MUST be login page, not homepage

async function openProviderMusicLoginWindow(owner) {
  const cookieSession = session.fromPartition(PROVIDER_LOGIN_PARTITION);
  
  // === STEP 1: Check existing session ===
  const initialCookies = await cookieSession.cookies.get({});
  if (hasProviderLoginCookie(initialCookies)) {
    return { ok: true, cookie: allCookies, reused: true };
  }

  // === STEP 2: Open login window ===
  return new Promise((resolve) => {
    let settled = false;
    let pollTimer = null;

    const loginWindow = new BrowserWindow({
      width: 900, height: 700, minWidth: 760, minHeight: 540,
      parent: owner, modal: false, show: false, autoHideMenuBar: true,
      title: 'Provider 登录', backgroundColor: '#111111',
      webPreferences: { partition: PROVIDER_LOGIN_PARTITION, contextIsolation: true, nodeIntegration: false, sandbox: true },
    });

    const finish = async (result) => {
      if (settled) return;
      settled = true;
      if (pollTimer) clearInterval(pollTimer);
      if (loginWindow && !loginWindow.isDestroyed()) loginWindow.close();
      resolve(result);
    };

    // === STEP 3: Poll every 2 seconds for auth cookies ===
    const checkCookies = async () => {
      try {
        const cookies = await cookieSession.cookies.get({});
        const loginCookies = cookies.filter(c =>
          c.name && c.value.length > 3 && /SPECIFIC_COOKIE_NAME/i.test(c.name));
        if (loginCookies.length > 0) {
          const allCookies = cookies.map(c => `${c.name}=${c.value}`).join('; ');
          finish({ ok: true, cookie: allCookies });
        }
      } catch (e) { console.warn('[Login] check failed:', e.message); }
    };

    loginWindow.on('ready-to-show', () => loginWindow.show());
    loginWindow.on('closed', async () => {
      if (settled) return;
      clearInterval(pollTimer);
      // Final check on close
      try {
        const cookies = await cookieSession.cookies.get({});
        const loginCookies = cookies.filter(c => /* same pattern */);
        resolve(loginCookies.length > 0
          ? { ok: true, cookie: cookies.map(c => `${c.name}=${c.value}`).join('; ') }
          : { ok: false, cancelled: true, message: '登录窗口已关闭' });
      } catch (e) {
        resolve({ ok: false, cancelled: true, message: '登录窗口已关闭' });
      }
    });

    pollTimer = setInterval(checkCookies, 2000); // ← 2 second interval
    loginWindow.loadURL(PROVIDER_LOGIN_URL).catch((e) => finish({ ok: false, error: e.message }));
  });
}
```

## Key rules
1. **Login URL must be a DEDICATED login page** — not the homepage. If the provider doesn't have one, this pattern won't work and manual cookie import is the fallback.
2. **Cookie names must be SPECIFIC** — `/userid|uid|token|session/` is too broad. Use exact known auth cookie names.
3. **Poll, don't wait for close** — checking on close alone misses auth cookies.
4. **finish() locks with `settled` flag** — prevents double-resolve.
5. **Preload.js must expose IPC** — `contextBridge.exposeInMainWorld('desktopWindow', { openProviderMusicLogin: () => ipcRenderer.invoke('provider-music-open-login') })`.

## Anti-pattern: the "wait for close → grab all" approach
```javascript
// ❌ FAILS — closes immediately because tracking cookies match
loginWindow.on('closed', async () => {
  const cookies = await cookieSession.cookies.get({});
  const allCookies = cookies.map(c => `${c.name}=${c.value}`).join('; ');
  resolve({ ok: true, cookie: allCookies }); // ← tracking cookies, not auth
});
```

## Netease example (working reference)
- URL: `https://music.163.com/#/login`
- Cookie: `MUSIC_U`
- Interval: 1200ms
- Partition: `persist:mineradio-netease-login`
