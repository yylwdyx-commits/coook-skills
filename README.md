# coook skills

给 AI 代理用的实战技能包：把上架、部署、H5、办公文档这些活里反复踩过的坑，写成代理能直接照做的判据和检查清单。

每个技能一个文件夹，`SKILL.md` 开头是 `name` / `description`，正文统一四段：

- **判断先行**：这类活最容易死在哪里
- **判据**：怎样才算真的做成（以真实状态为准，不以「脚本跑完」为准）
- **坑与真因**：现象 → 根因 → 做法
- **检查清单**：交付前逐条打勾

[English](#english)

## 技能

| 领域 | 技能 | 管什么 |
|---|---|---|
| 发布上架 | [ios-release](skills/ios-release/SKILL.md) | App Store 全链路：建记录、上传、TestFlight、隐私问卷、中国区备案、被拒处理 |
| | [android-release](skills/android-release/SKILL.md) | 安卓签名、版本号、应用内更新、WebView 壳、真机验收 |
| | [wx-miniprogram](skills/wx-miniprogram/SKILL.md) | 微信小程序 / 小游戏：隐私指引、合法域名、备案、提审材料 |
| 网页 | [h5-webview](skills/h5-webview/SKILL.md) | H5 在微信、App 壳、手机浏览器里的兼容坑 |
| | [web-and-games](skills/web-and-games/SKILL.md) | 单文件网页、落地页、浏览器小游戏 |
| | [preflight](skills/preflight/SKILL.md) | 对外页面上线前的八条检查 |
| | [browser-verify](skills/browser-verify/SKILL.md) | 用真浏览器像真人一样验收 |
| 服务器 | [deploy-ship](skills/deploy-ship/SKILL.md) | rsync、nginx、证书、重启、多副本收敛、推 GitHub 前扫密钥 |
| | [servers](skills/servers/SKILL.md) | 多台机器上机与排障：先分故障域再动手 |
| | [one-click-button](skills/one-click-button/SKILL.md) | 把定时脚本改成网页上的「立即跑一次」 |
| | [api-sdk](skills/api-sdk/SKILL.md) | 读接口文档，封装 SDK 和对接文档 |
| 设备 | [android-rescue](skills/android-rescue/SKILL.md) | 远程救安卓：开机循环、系统进程崩溃、重启后连不上 |
| 办公 | [office-docs](skills/office-docs/SKILL.md) | Word / PPT / Excel |
| | [data-charts](skills/data-charts/SKILL.md) | 数据分析、图表、单文件看板 |
| | [invoices](skills/invoices/SKILL.md) | 发票识别、去重、报销汇总 |
| | [files-tidy](skills/files-tidy/SKILL.md) | 整理文件夹、找重复、批量改名转图 |
| | [feishu-docs](skills/feishu-docs/SKILL.md) | 读飞书 / Lark 文档、表格、多维表格 |
| | [visa](skills/visa/SKILL.md) | 签证与电子旅行许可：查规则、列材料、排时间线 |
| 创作 | [video-review](skills/video-review/SKILL.md) | 抽帧拼图 + 转写，审片和拆片 |
| | [mockup-shot](skills/mockup-shot/SKILL.md) | 截图做成样机图，像素不经 AI 重画 |
| | [model-3d](skills/model-3d/SKILL.md) | 代码建模，导出可直接 3D 打印的 STL |

## 用法

把需要的技能文件夹复制到你所用代理的技能目录即可，格式是通用的 `SKILL.md`：

```sh
git clone https://github.com/yylwdyx-commits/coook-skills
cp -R coook-skills/skills/deploy-ship <你的代理的 skills 目录>/
```

不需要的技能不用装；每个技能互不依赖。

## 写技能的几条规矩

- 只写通识，不写某个项目的地址、账号、密钥。
- 判据必须能验证：给出命令、状态值或截图标准，不写「注意一下」。
- 一条坑写清真因，不只写现象。
- 动钱、对外提交、删数据这类不可逆动作，技能里写明交给人确认。

欢迎提 Issue 和 PR：新坑、过期的判据、平台规则变化都算。

## English

Battle-tested skills for AI agents. Each skill is a folder with a `SKILL.md` (front matter `name` / `description`) written as: the judgment up front, verifiable success criteria, pitfalls with root causes, and a pre-ship checklist. Topics cover App Store and Android releases, WeChat mini programs, H5 in WebViews, server deploys, remote Android recovery, office documents, data charts and more. Content is in Chinese; copy any skill folder into your agent's skills directory to use it.

License: MIT
