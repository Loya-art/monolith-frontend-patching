# Coolorus 2.5.17 CEP 数位笔修复 — 完整案例

2026-08 实战：Coolorus 2.5.17（Adobe CEP 插件，PS2026 + 高漫数位板）鼠标可拖色轮、数位笔只能点击不能拖。

## 故障链路（诊断结论）

1. 插件所有交互监听**原生 mouse 事件**（`mousedown`/`mousemove`/`mouseup`）——通过 EaselJS Stage 分发（`handlePaletteMouseDown`、`stagemousemove`、`stagemouseup`）。
2. Windows Ink 数位笔在 CEP（Chromium）环境产生 **PointerEvent**；笔按下的兼容 mousedown 能到（所以点击取色正常），但拖动时兼容 mousemove 流缺失 → 拖拽逻辑永远收不到移动。
3. 结论：**不是坐标问题、不是算法问题，是事件类型问题**。修复 = pointer→mouse 桥接，颜色算法一行不用改。

## 事件名映射（从 v2.6.0 可读源码 O3a 常量表确认）

| 常量 | 值 | 用途 |
|---|---|---|
| N4 | mousedown | 色轮 hitArea / 取色拖拽触发 |
| p20 | stagemousemove | EaselJS stage 拖拽中 |
| c7 | stagemouseup | EaselJS stage 松开 |
| V4 | mousemove | ColorDragger 绑在 #body |
| l00 | mouseup | ColorDragger 松开 |

EaselJS 计算坐标用 `t.stageX/stageY`，由 Stage 从原生事件的 clientX/Y 换算。合成 MouseEvent 带真实 clientX/Y → 自动正确。

## 文件位置（重要）

- **真正加载目录**：`C:\Program Files (x86)\Common Files\Adobe\CEP\extensions\com.moongorilla.coolorus2\`（manifest: `CSXS/manifest.xml`, MainPath=index.html）
- `C:\Program Files (x86)\Adobe\CEP\extensions\` 下往往只有**安装包**（install_coolorus_2_win.exe），不是运行代码
- 混淆版 v2.5.17：单文件 `index.html`（945KB / 6 行，obfuscator.io 风格 `_0x1a9e("0x..")` 字符串查表，EaselJS 库方法名是明文）
- **可读参照**：v2.6.0 的 `js/main.js`（1MB，未混淆）——用 grep 找 `handlePaletteMouseDown`、`O3a = {` 直接读懂拖拽流程

## 安装步骤（Program Files 写权限陷阱）

1. 普通用户对 Program Files (x86) 只有 RX——直接 cp 会 Permission denied。
2. **UAC 提权链路在 git-bash 里很脆**：`Start-Process powershell -Verb RunAs -ArgumentList '...' -Wait` 遇到中文路径（Loya-art）+ 引号嵌套时经常静默失败（退出码 0 但没执行）。
3. **可靠做法**：把补丁文件先 cp 到纯 ASCII 路径（`C:\Temp\`），用 write_file 写一个 .bat（内容只有 copy 命令 + 结果重定向到日志文件），然后：
   ```bash
   powershell -NoProfile -Command "Start-Process cmd -Verb RunAs -ArgumentList '/c','C:\Temp\copy-fix.bat' -Wait"
   cat /c/Temp/copy-result.txt   # 用重定向日志验证真实执行结果
   ```
4. **验证必须看真实证据**：bat 里 `echo EXIT=%ERRORLEVEL% > C:\Temp\copy-result.txt` + `dir ... >> result`，事后 cat 日志确认文件真的复制到位（退出码 0 不代表提权脚本执行成功）。
5. **git-bash 的 printf 写 bat 会炸**：`%` 被 printf 当格式符解释（`printf: \`>': invalid format character`）。写 .bat 一律用 write_file。

## 注入方式

在 index.html **文件末尾**（最后一个 `</script>` 之后）追加一行：

```html
<script src="js/pointer-fix.js"></script>
```

- 不碰混淆主体，零风险
- 备份：`cp index.html index.html.bak-$(date +%Y%m%d-%H%M%S)`（提权前）
- 回滚：删 js/pointer-fix.js + 删注入行，或恢复 .bak
- 改完必须**完全退出 Photoshop 重开**（CEP 缓存 index.html）

## 验证方法（无硬件也能测）

测试台 `test-pointer-fix.html`：
1. 引用补丁脚本 + 绑定假 mousedown/mousemove/mouseup 监听器记录日志
2. 用 `new PointerEvent('pointerdown', {pointerType:'pen', pointerId:7, clientX, clientY, button:0, buttons:1})` 派发完整序列（down→move×3→up）
3. 用 browser_navigate 打开 file:// 测试台，browser_console 读日志：
   ```
   mousedown @ 278,325 (isTrusted=false)
   mousemove @ 338,295 → 398,345 → 458,315
   mouseup @ 458,315
   ```
4. 鼠标路径对照测试：原生 MouseEvent 直通不受桥接干扰（pointerType==='mouse' return）
5. CSS 注入验证：`getComputedStyle(canvas).touchAction === 'none'`

## 关键坑

- 鼠标事件**不能**也桥接（pointerType==='mouse' 必须 return）——否则鼠标拖拽双触发
- `preventDefault()` 必须调用——否则浏览器自身残缺的兼容 mouse 事件 + 桥接合成事件 = 双触发
- `Object.defineProperty(ev, 'offsetX', ...)` 是合成事件补 offsetX/Y 的唯一办法（构造器不支持）
- 极老 CEF 无 MouseEvent 构造器 → 降级 `document.createEvent('MouseEvent')` + initMouseEvent
