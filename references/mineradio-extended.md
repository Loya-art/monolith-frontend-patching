# Mineradio-Extended: 反向工程版 Kugou API 架构

> 来源：`daaimengermengzhu/Mineradio-Extended` (GitHub, 2026-07)
> 抖音作者：Memory- (视频 ID: 7661835139147648256)

## 核心突破

扩展版通过逆向 Kugou **手机 APP 的私有 API**（而非公开 Web API）实现了完整功能。

## Kugou 内部 API 网关

不是用 `mobilecdn.kugou.com`（公开接口），而是用 Kugou APP 的内部网关。

关键架构：
```
kugouApiRequest(pathname, params, opts)
  ├── kugouDefaultParams()      — 补充 deviceId, mid, dfid 等设备参数
  ├── kugouSignKey()            — hash 加密密钥
  ├── kugouAndroidSignature()   — 安卓签名（请求完整性）
  ├── kugouApiCookieHeader()    — 认证 Cookie 头
  └── headers: { kg-rc, kg-thash, kg-rec, kg-rf } — APP 特定头
```

## 两套 Cookie（双账号）

| Cookie 文件 | 用途 | 平台 |
|---|---|---|
| `.kugou-cookie` | Kugou 酷狗概念版 | Kugou Lite |
| `.kugou-music-cookie` | Kugou 普通版 | Kugou Music |

## QR 登录（API 方式，非 Electron IPC）

扩展版的 QR 登录不依赖 Electron 的 BrowserWindow/IPC，而是直接调用 Kugou 的 API：

```
POST /api/kugou/login/qr/key      — 获取 QR key
POST /api/kugou/login/qr/create   — 生成二维码
POST /api/kugou/login/qr/check    — 轮询扫码状态
POST /api/kugou/login/cookie      — 手动 Cookie 导入
```

## 已实现的接口（含认证）

| 接口 | API 路径 | 说明 |
|---|---|---|
| 搜索 | `/v3/search/song` | 带 `x-router: complexsearch.kugou.com` |
| 播放 URL | `/v5/url` | 带 `x-router: trackercdn.kugou.com` |
| 用户歌单 | `/v2/get_list` + `/v7/get_all_list` | 带 `x-router: cloudlist.service.kugou.com` |
| 歌单增删 | `/cloudlist.service/v5/add_list` + `/v4/delete_songs` | 完整读写 |
| 评论 | `/mcomment/v1/cmtlist` | 歌曲评论 |

## Cookie 解析

```javascript
kugouCookieObject()      → { userid, token, nickname, avatar, mid, dfid, guid, dev, mac }
kugouCookieUserId()      → userid / userId / user_id / KG_UID / kugou_userid
kugouCookieToken()       → token / Token / kugou_token / KG_TOKEN / musicToken
kugouCookieNickname()    → nickname / nickName / username / userName / m_name
kugouCookieAvatar()      → avatar URL
```

## 内置手势控制（MediaPipe）

扩展版 `index.html` 内置完整手势控制系统（DIY 面板 → 摄像头交互 → 手势触碰）：

```
MediaPipe Hands → uHandActive/uHandXY → Three.js 粒子着色器
       ↓
  手势映射: 移动=粒子旋转, 捏合=聚集, 张开=爆炸, 握拳=收缩
```

详见 `references/mediapipe-gesture-architecture.md`

## DIY 视觉工坊增强

扩展版新增文件：
- `public/diy-sandbox-concept.html` — DIY 独立页面
- `public/wallpaper.html` — 壁纸模块
- `public/desktop-lyrics.html` — 桌面歌词

## 关键教训

1. **不要用公开 Web API** — Kugou 的手机 APP API 才是真正能用的（有认证+签名）
2. **API 扫码 > Electron IPC 扫码** — 纯 HTTP API 的 QR 登录比打开 BrowserWindow 更可靠
3. **签名是必须的** — `kugouAndroidSignature()` 是请求能通过的核心
4. **设备指纹** — mid, dfid, guid, dev, mac 等设备参数构成完整的 APP 伪装
5. **搜索现成版本优先** — 编码前先搜 GitHub `AppName-Extended` 和抖音教程

## 参考来源

- GitHub: https://github.com/daaimengermengzhu/Mineradio-Extended
- 抖音: Memory- 的二创视频 (2026-07-13)
- 原版 Mineradio: https://github.com/XxHuberrr/Mineradio
