# Mineradio: Kugou Music API Reference

> All endpoints tested 2026-07-22. No authentication required for search/lyrics/ranks.

## Working Endpoints

### Search
```
GET http://mobilecdn.kugou.com/api/v3/search/song
  ?format=json
  &keyword=KEYWORD
  &page=1
  &pagesize=20
  &showtype=1
```
Returns: `{status:1, data:{total:N, info:[{hash, songname, singername, album_name, album_id, album_audio_id, duration, 320hash, sqhash, ...}]}}`

### Lyrics (raw LRC, not JSON!)
```
GET http://m.kugou.com/app/i/krc.php
  ?cmd=100
  &hash=HASH
  &timelength=DURATION_MS
```
Returns: Raw LRC text starting with `[ti:SongName]\n[ar:ArtistName]\n...`

### Song Metadata
```
GET http://m.kugou.com/app/i/getSongInfo.php
  ?cmd=playInfo
  &hash=HASH
  &key=HASH
```
Returns: `{authors:[{author_name,...}], hash, author_name, imgUrl, audio_id, status:0}` — **status is always 0, no play URL**

### Rank/Playlist List
```
GET http://mobilecdn.kugou.com/api/v3/rank/list
  ?format=json
```
Returns: `{data:{total:55, info:[{rankid, rankname, banner7url, imgurl, ...}]}}` — 55 ranks including TOP500 (8888), 国潮音乐榜 (85897), 民谣榜 (51341), 纯音乐榜 (59900), etc.

### Rank Songs
```
GET http://mobilecdn.kugou.com/api/v3/rank/song
  ?format=json
  &rankid=8888
  &page=1
  &pagesize=20
```
Returns: `{data:{info:[{hash, filename:"Artist - SongName", album_id, album_audio_id, duration, ...}]}}`

### Album Cover URL Template
```
http://imge.kugou.com/stdmusic/{album_id_last_2_chars}/{album_id}.jpg
```

## Streaming URL — NOT AVAILABLE

- `m.kugou.com/app/i/getSongInfo.php?cmd=playInfo` → status:0, no URL (DEPRECATED)
- `wwwapi.kugou.com/yy/index.php?r=play/getdata&hash=...&album_id=...` → err_code:30020
- `trackercdn.kugou.com/i/v2/?cmd=23&...` → "Bad key" or "Empty appid"
- `mobilecdn.kugou.com/api/v3/track/songurl` → "Access Deny !!!"

**Conclusion**: Kugou streaming requires authentication or app credentials. Search and lyrics work without auth.

## Headers
```
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36
Referer: http://m.kugou.com/
```

## Qishui Music (汽水音乐) — INACCESSIBLE

ByteDance's Janus API gateway blocks all requests:
- `www.douyin.com/aweme/v1/web/music/search/` → "Unsupported path(Janus)"
- `aweme.snssdk.com/aweme/v1/music/search/` → returns empty results
- All endpoints require X-Gorgon/X-Khronos/X-Argus signing headers

**Do not attempt direct HTTP access.** Future support would require a maintained API bridge service.

## Electron Login Window Pattern (polling, like Netease)

For services that require web login via Electron's BrowserWindow, follow the Netease pattern:

1. **Dedicated login URL** — navigate to a SPECIFIC login page, not the main site
2. **Cookie polling** — poll every 1500-2000ms using `setInterval` in the Promise constructor
3. **Session partition** — use `session.fromPartition('persist:provider-login')` for isolated cookie storage
4. **Specific cookie detection** — filter for known auth cookie names (e.g. `MUSIC_U` for Netease, `userid|uid|token` for Kugou)
5. **Initial session reuse** — before opening the window, check if auth cookies already exist
6. **Auto-finish on detection** — when login cookies found, call `finish()` to close window and return
7. **On-close fallback** — when user closes window manually, do one final cookie check

```javascript
const LOGIN_PARTITION = 'persist:mineradio-provider-login';
async function openProviderLoginWindow(owner) {
  const cookieSession = session.fromPartition(LOGIN_PARTITION);
  const initial = await cookieSession.cookies.get({});
  if (hasLoginCookie(initial)) return { ok: true, cookie: toCookieString(initial), reused: true };
  return new Promise((resolve) => {
    let settled = false, pollTimer = null;
    const win = new BrowserWindow({ webPreferences: { partition: LOGIN_PARTITION } });
    const finish = (r) => { if (!settled) { settled = true; clearInterval(pollTimer); win.close(); resolve(r); } };
    const check = async () => {
      const cookies = await cookieSession.cookies.get({});
      if (cookies.some(c => isLoginCookie(c))) finish({ ok: true, cookie: toCookieString(cookies) });
    };
    win.on('ready-to-show', () => win.show());
    win.on('closed', async () => { /* final check on close */ });
    pollTimer = setInterval(check, 2000);
    win.loadURL(LOGIN_URL);
  });
}
```
