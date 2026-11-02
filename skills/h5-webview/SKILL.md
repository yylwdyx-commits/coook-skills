---
name: h5-webview
description: "H5 在微信内置浏览器、App 的 WebView 壳（iOS WKWebView / 安卓 WebView）、手机浏览器里跑的兼容坑与验收判据：分享卡、域名被微信拦、下载存图、缓存不更新、白屏、安全区、跳转变 localhost。触发：H5/手机网页/微信里打不开/被微信拦了/分享卡不对/壳里白屏/改了不生效/存图翻不到/iPhone 底部被挡/套个壳。"
---

# H5 在微信与 App 壳里

判断先行：同一份网页在桌面 Chrome 里完美，在微信、WebView 壳、老安卓里各有各的死法，而且**多数是静默的**（不报错、只是不工作）。判据只认真机真环境截图；puppeteer 换个 UA 只能验分支逻辑。

## 何时用
- 做给手机用户的 H5、落地页、套壳 App 的网页部分。
- 排查「微信里打不开 / 壳里白屏 / 改了不生效 / 按钮点了没反应 / 存的图找不到」。

## 判据
- 先用真浏览器（Playwright / Puppeteer 等）按手机宽度（390）和电脑宽度各开一次、看截图，控制台零报错；再上真机。
- 微信里**真转发一次**看分享卡（标题、图、描述三样都对）。
- 微信内、iOS Safari、安卓 Chrome、App 壳（如有）各真机打开主路径并截图；390px 宽不横滚。
- 改完发布后，在**已经打开过旧版**的 WebView 里重进一次，确认拿到新版（缓存问题只在这里暴露）。
- 壳里首屏请求就带上端标记（服务端按端下发的功能首屏即正确，不是刷新一次才对）。

## 坑与真因
**微信**
- 无 JS-SDK 时分享卡：标题 = `<title>`；图 = 正文里第一张 ≥300×300 的图（移到屏外，不能 `display:none`）；描述 = `meta description`。`og:image` 给其他聊天软件用，绝对地址。
- **域名被拦，「部署验证文件」验的是被拦那条链接的域名**（常是对象存储桶域名而不是自家站）。微信抓取走 http 且不跟 301：80 口要 `location = /<文件名>.txt` 直出，certbot 加的 server 级 `return 301` 会压过 location，得整块改成 location 式。先放文件再点验证，它只在点按钮那一刻抓一次；申诉页约 10 分钟过期。
- 对象存储默认域名在微信里信任分低，下载常被拦，治本是绑自有下载域名。
- 微信拦 apk 下载；下载页先引导「在浏览器打开」。

**缓存**
- WebView / 电视 / kiosk 会把带长缓存头的 HTML 和 JS 缓存死，改了不生效。HTML `no-store`，静态资源带内容指纹长缓存。
- 静态资源（favicon、logo、字体）在 nginx 直出带 `expires`，别穿进 node：WebView 无缓存头时会高频重取，能把 node 进程 CPU 打满。
- `NEXT_PUBLIC_*` 这类构建期变量焊进产物，改 .env 重启无效，必须重新构建。

**布局**
- `env(safe-area-inset-*)` 只有 `viewport-fit=cover` 时才非 0；「同样的 CSS 两个站高度不同」去看 meta，别量组件。
- Grid 的 `1fr` 最小值是 min-content，会被定宽子元素撑爆，手机断点用 `minmax(0,1fr)`。
- 手机 `100vh` 被地址栏挡，用 `100dvh`。

**跳转与环境**
- 反代后 route handler 用请求 URL 拼绝对跳转会变成 `https://localhost:端口/`，一律发相对 Location；验收断言要盯 Location 本身。
- ESM 里被 import 的模块顶层读 `process.env` 恒为空（env 文件在 import 之后才加载），一律惰性读；用真环境变量启动测不出来。
- https 页面渲染库里存的 `http://IP` 图会被混合内容拦掉，显示层转相对路径，库里不动。
- 老安卓 WebView 不认 `??` / `?.`，整份 JS 白屏；降语法或加能力探测兜底。

**存图与下载**
- iOS WKWebView 对 `<a download>` 无反应；安卓落 Downloads，相册翻不到。要进相册只能走原生桥，原生按 URL 取图，别把 data URL 塞进桥。

**App 壳**
- iOS：UA 标记在 `applicationNameForUserAgent` 里同步设（异步改 `customUserAgent` 首屏请求不带）；外链只在 `linkActivated` 时外开（否则验证码页被踢去 Safari）；实现 `createWebViewWith` 和 alert/confirm，否则弹窗「点了没反应」；会话 cookie 进后台时抄进 Keychain、启动 `load()` 前塞回（cookie 刷盘异步，上划杀掉会丢）；锁缩放要原生 + 注入 viewport 两道。
- 安卓：`shouldOverrideUrlLoading` 先判 `isForMainFrame`；网页判端用「JS 桥在不在」，不是 UA。

## 检查清单
1. 分享三件套 + favicon；微信真转发看卡。
2. 微信 / iOS Safari / 安卓 Chrome / 壳 四个环境真机截图主路径。
3. HTML 不缓存、静态资源指纹 + nginx 直出。
4. 错误不透传原始报错；对外文案不露内部系统名。
5. 跳转一律相对；env 惰性读；构建期变量改了就重建。
6. 下载与存图走对路（壳内原生桥 / 微信先引导出浏览器）。
7. 域名被拦按验证文件流程走，先认清被拦的是哪个域名。
