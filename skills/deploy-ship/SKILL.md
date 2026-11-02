---
name: deploy-ship
description: "服务器部署与代码收尾的通识判据（rsync/scp 推代码、nginx 与 sshd 改配置、证书、pm2/systemd 重启、多副本收敛、git commit/push 与推 GitHub 前扫密钥）。触发：部署/上线到服务器/推上去/rsync/改 nginx/证书过期/重启服务/pm2/新进程没生效/提交推送/推 GitHub/建仓，或写、改任何 deploy.sh。具体项目的部署命令看项目自己的说明文档。"
---

# 部署与收尾

判断先行：部署事故几乎都是**例行动作顺手抹掉了带外的东西**（生产 .env、运行时数据、certbot 加的 SSL 段、手工补丁），而且当场不报错，延迟成别的症状。所以部署脚本要会拒绝、成功要看真结果。

## 何时用
- 任何把代码推到服务器、改服务器配置、重启服务的动作。
- 收尾提交代码、推远端。

## 判据（怎样算部署成功）
- 新进程真起来了：`pm2 list` uptime < 60 秒 / `systemctl status` 启动时间是刚才；不是看到脚本最后一句「部署完成」。
- 所有副本收敛：本机真源、服务器、其他运行机 `rsync -acn` 按内容比 0 差异。
- 线上真路径打开一次（真页面用真浏览器截图 / 真接口看响应体 / 真任务跑一条），有证据才说「好了」。
- 提交收尾：`git status --porcelain` 为空，`git rev-list --left-right --count origin/main...HEAD` = `0 0`。

## 坑与真因
**rsync**
- 默认同步隐藏文件：本地 `.env` 盖掉生产密钥，dotenv 不报缺，服务静默挂。永远 `--exclude .env --exclude '.env.*'`；生产 env 只在服务器上改。
- `--delete` 会删远端独有的运行时数据（sqlite、用户上传）。**豁免别按名字枚举**，用目录通配一条管整类（如 `--exclude 'data/*/'`、`*.db`、`*.db-journal`）；跑前读 exclude、先 `-n` 干跑、跑后数 `deleting` 行。
- 代码和数据住在同一目录时，「按目录整推」这条规矩本身有毒。部署前先 `ls` 目标目录。
- 数据只有一份才是真因：带生产数据的项目要有每日快照（硬链接快照验 inode 相同、总量没翻倍）。

**配置**
- 改 nginx / sshd：`cp` 备份 → 改 → `nginx -t` / `sshd -t` → `reload`（不是 restart）；失败立即还原再 reload。改 sshd 前先开第二个会话保命。
- deploy 脚本 cp 仓库版 vhost 会抹掉 certbot 原地加的 SSL 段：nginx 语法照样合法，443 落到默认 server 返别的域名证书。certbot 跑完把服务器版拉回仓库，deploy 里断言 `ssl_certificate` 存在。
- 证书自动续期在 cron 里静默失败（PATH 没有 `/usr/sbin`）：看板白屏先查 `certbot certificates | grep EXPIRED`。
- 带外补丁封装成唯一来源 + 启动自检，不然下一次无关编辑就把它抹了。

**构建与重启**
- `deploy.sh 2>&1 | tail -5` 会把中间的 build error 截掉，旧产物继续跑。看全量日志或看进程 uptime。
- 构建期变量（如 `NEXT_PUBLIC_*`）焊进产物，改 env 重启无效，要重建。
- 带内存任务队列的服务，长任务跑着时别重启；先部署后触发任务。
- `|| true`、`2>/dev/null`、空 catch 会把「连不上 / 路径错 / 没权限」吞成「已是最新」。每写一个先问会吞掉哪种真失败。
- 长构建别挂在一条 ssh 上：`nohup … > log 2>&1 &` 后读日志；隧道 ssh 长连必掉。密集重试会被服务器防爆破封出口 IP，退避重试。

**流量**
- 大文件（上传、下载、视频、安装包）走对象存储 / CDN 直传直下，别穿带宽小的中转机，会拖垮同机所有站。
- 静态资源 nginx 直出带 `expires`，node 只接动态请求。

**代码**
- 用脚本按「锚点 A 到锚点 B」整段替换代码会连带删掉中间的函数，`node --check` 查不出；替换前列区间内函数名，替换后逐个 grep；部署闸上 lint `no-undef`。
- 主路径改完必须真跑一条（空队列跑一次不算）。
- 线上跑的脚本目录必须进 git；「已部署」≠「已提交」。

**git / GitHub**
- 推前扫密钥：`git check-ignore -q .env` 确认挡住；diff 里 grep `secret|api_key|BEGIN .*PRIVATE`。扫描器报错 ≠ 干净，重写简单模式重扫到有结论。
- 推前确认远端是私有仓：`gh repo view --json visibility`。新建 GitHub 仓是对外动作，先问。
- 按功能分组写人话 commit message；别人留下的已部署未提交改动，按功能一起如实提交，别晾着。
- 别用无条件 `echo "干净"` 收尾，跑真命令看输出。

## 检查清单（每次部署）
1. 读项目说明文档里的部署路径、端口、进程名，不猜。
2. `rsync -acn` 比对所有副本，有漂移先归并回本机真源。
3. exclude 含 `.env*`、运行时数据目录通配、数据库文件；有 `--delete` 先干跑。
4. 改配置走 备份 → 语法检查 → reload → 失败回滚。
5. 部署后看进程 uptime + 真路径验一遍 + 所有副本 0 差异。
6. 收尾 commit + push，核到 `0 0`。
