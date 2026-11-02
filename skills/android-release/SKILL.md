---
name: android-release
description: "安卓 App 打包发布通识（Mac 上从零搭工具链、签名与 keystore、版本号、应用内更新、WebView 壳、真机验收）。触发：打个安卓包/出 APK/发安卓新版/装不上/签名不对/覆盖安装失败/gradle 报错/应用内更新/安卓壳白屏/存相册翻不到，或动任何 android/ 工程与 release.sh。"
---

# 安卓打包发布

判断先行：安卓发版的事故几乎都不在代码，在**签名、版本号、发错包**三件事上；而「装不上」对普通用户等于永久失联。所以出包脚本要会拒绝坏包，验收要上真机。

## 何时用
- 新建或接手一个安卓工程（原生或 WebView 壳），要出可分发的 release 包。
- 发新版、做应用内更新、排查「装不上 / 不更新 / 白屏 / 功能哑掉」。

## 判据（怎样算成功）
- `apksigner verify --print-certs` 通过，证书指纹 = 线上老版本的指纹（换签名 = 所有人得卸载重装）。
- versionCode 严格大于线上那一版（先读线上 latest.json 再出包，不往前走的包脚本直接拒）。
- 发出去的是 release 包，不是 debug 包（debug 常指向本机 dev 地址，装上就是死 App）。
- **真机**装一次：覆盖安装成功、主路径点一遍、改动的那段代码真执行到。模拟器双档（新/老 Android 版本）只是及格线。
- 应用内更新：下载 → sha256 校验 → 安装，校验错必须拒装；真机走通一次「旧版收到更新提示 → 装上新版」。

## 坑与真因
- **gradle 没配签名时静默产出 `-unsigned.apk`**，不报错。脚本里找不到 `app-release.apk` 就退出，别去捡 unsigned 那个。
- **apksigner 输出格式换过**（老版 `Signer #1 certificate SHA-256 digest:`，新版 `V3.0 Signer: certificate SHA-256 digest:`），只认一种写法会把签好的包判死。
- **keystore 丢了 = 全员卸载重装**，安卓不允许换签名覆盖。keystore 和口令放私密目录 + 离线备份，绝不进仓库。
- **系统 gradle 太新，AGP 不兼容**：项目一律用 gradle wrapper 钉版本；gradle 新版拒绝在空目录跑 `wrapper`，要先建只含 `rootProject.name` 的 settings.gradle。
- **JDK 用 brew formula（`openjdk@17`）不用 cask**：cask 装 .pkg 要 sudo，无 TTY 必失败；`java_home` 找不到 formula 版，直接写全路径。
- **代理把 gradle/google()/mavenCentral() 打成 503**：JVM 走系统代理不吃 env。wrapper 和仓库换国内镜像直连。
- **release 别贸然开 R8/混淆**：keep 规则漏一条，JS 桥方法变 undefined，不崩不报错、功能直接哑掉。省几百 KB 不值。
- **安装走 `PackageInstaller` 不走 `ACTION_VIEW`**：只有前者回传 ROM 的失败原因（如国产 ROM 纯净模式拦截）；后者失败完全静默。
- **WebView 壳的 `shouldOverrideUrlLoading` 要先判 `isForMainFrame`**：否则验证码/支付 iframe 被当外链踢去系统浏览器，壳内登录注册全断。
- **老 WebView 不认新语法**（Android 9 自带 Chrome 66 不认 `??` / `?.`），整份 JS SyntaxError = 白屏，浏览器里同样中招。要么降语法目标，要么加能力探测兜底给人话。
- **网页存图在安卓上进不了相册**：`<a download>` 落进 Downloads 集合，图库和其他 App 的选图器翻不到。要原生桥写 MediaStore；拍摄时间只能写进文件 EXIF（ContentValues 的 DATE_TAKEN 写不进），写 EXIF 前先把 fd seek 回 0。
- **部分机型只跑 64 位**：去第三方市场找的 32 位包装不上；判端用「JS 桥在不在」而不是 UA。
- **HarmonyOS NEXT 不装 APK**，这批用户只能走网页版。
- **微信内置浏览器拦 apk 下载**：下载页第一句先引导「右上角在浏览器打开」。

## 检查清单（出包前逐条打勾）
1. JDK / SDK / gradle wrapper 版本钉死，镜像直连可用。
2. 签名配置读私密目录的 properties；产物 `apksigner verify` 通过且指纹与线上一致。
3. versionCode > 线上；versionName 人能读懂。
4. 产物是 release，不是 debug；包里没有 dev 地址、调试开关。
5. 算 sha256，写更新清单（版本号、下载地址、sha256、最低支持版本），清单最后写——包没传完不发清单。
6. 真机覆盖安装 + 主路径 + 改动点；老系统版本至少模拟器过一遍。
7. 上传后从线上地址重新下载一次，sha256 与本地一致才算发出。
8. 发版说明给用户看的那句只讲「新了什么 + 怎么更新」。
