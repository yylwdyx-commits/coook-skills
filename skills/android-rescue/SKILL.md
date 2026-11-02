---
name: android-rescue
description: "用电脑远程救安卓手机：开机循环、系统界面反复重启、整机卡死、重启后 Wi-Fi 起不来、重启后 adb 或无线调试连不上、精简卸了系统 App 之后出事。先读日志定真因再动手，趁系统活着的几秒修，修完读回真值、恢复依赖、同批手机一起排雷、把会再踩的自动化堵上。触发：手机一直重启、卡在开机画面、开不了机、手机变砖了、系统界面已停止运行、重启后连不上、adb 显示 offline、Wi-Fi 打不开、手机突然卡死。"
---

# 远程救手机

主线：分层 → 读日志定真因（不猜）→ 趁窗口修 → 读回真值确认稳住 → 恢复依赖 → 同批手机排雷 → 堵住会再踩的路 → 一句话报结果。

## 1. 先分层：现在是哪种死法
| 看 | 命令 | 判 |
|---|---|---|
| 连得上吗 | `adb devices -l` | offline / 找不到：USB 线松就换无线 adb，两条都试 |
| 整机重启过吗 | `cut -d. -f1 /proc/uptime` 隔 10 秒读两次 | 秒数变小 = 整机重启 |
| 系统进程崩过吗 | `pidof system_server` 隔 10 秒读几次 | 进程号在变、偶尔为空 = 系统进程循环崩，整机没重启 |
| 开机完成没 | `getprop sys.boot_completed` | 1 才算；`cmd: Can't find service` = 系统服务还没起来 |
把每次读到的时间、uptime、进程号记成一张小表，循环的周期一眼就出来。

## 2. 读日志定真因，每一轮都重读
- `logcat -d -b crash | tail -40`，`logcat -d -b all | grep -E -A25 "WATCHDOG KILLING|FATAL EXCEPTION IN SYSTEM"`，只看异常第一行和主线程栈顶几行。
- 同一台手机连着出事，每一轮的原因可能不同：先是看门狗、再是某个服务卡死、整机重启后才露出真因。上一轮的结论不能直接套到这一轮。
- 常见栈顶和修法：
  - `cannot find system app : <包名>`（厂商安全服务校验签名）：以前用 `pm uninstall --user 0` 卸过厂商要校验的系统 App（小米系常见是应用商店），平时不发作，一重启就开机循环 → `cmd package install-existing <包名>`。
  - `Blocked in ... takeScreenshot`（无障碍截图和窗口锁互等）：系统卡死被看门狗杀；同一时刻别让多个工具一起截屏。
  - 某个系统服务的 native 方法卡住（音频、显示 HAL）：系统进程救不回来，整机重启。
  - `Wifi is disabled` 但 `settings get global wifi_on` 是 1：系统进程重启后 Wi-Fi 没起来 → `svc wifi enable`，再 `ping -c1 网关` 确认。

## 3. 趁窗口修
- 系统进程每次能活几秒：把修复命令写成循环，一秒试一次，看到成功字样就停：
  `for i in $(seq 1 60); do R=$(adb shell cmd package install-existing <包名> 2>&1); echo "$R" | grep -q "installed for user" && break; sleep 1; done`
- 稳住的标准：`pidof system_server` 连续 1～2 分钟不变，`sys.boot_completed` 为 1。
- 有锁屏密码的手机，开机后第一次解锁前 App 数据是加密的，聊天类和常驻后台的 App 都起不来：稳住后先唤醒解锁一次。

## 4. 动手的边界
- 只做可逆的：装回、启用、打开 Wi-Fi、重启。恢复出厂、清数据、刷机、删文件一律先问用户、说清后果。
- 一台很久没重启的手机，重启可能引爆潜伏的问题。重启前先比对「被卸掉的系统 App」（`pm list packages -u` 对比 `pm list packages`），能装回的先装回。
- 不关 Wi-Fi、不开飞行模式、不改系统代理：远程时这等于把自己锁在门外。

## 5. 修完恢复依赖，逐项读回
- 无线调试：重启后会关。靠它拿 shell 身份的 App 要重连：`settings put global adb_wifi_enabled 1`，系统弹「允许在此网络上进行无线调试」时先勾「始终允许」再点「允许」。
- 无障碍：`dumpsys accessibility | grep -A1 "Bound services"`，没绑上就让对应 App 自己重开。
- 原来在跑的业务：聊天 App 进程在不在（`pidof 包名`）、网络通不通、后台服务有没有回来，逐项读回再说「好了」。

## 6. 同批手机一起排雷
同一个精简脚本、同一批设置处理过的手机都埋着同一颗雷。查到真因后，把同样的修复在其它手机上先做掉（只做可逆的装回），不用等它们也砖。

## 7. 把会再踩的路堵上
- 清理、复原类自动化只卸第三方包（`pm list packages -3`），系统包只装回不卸。
- 拿「正常状态」当基线的脚本，修好后重记基线，旧的留档标作废。
- 长时间无人值守的测试，每轮记 uptime 和 system_server 进程号，崩过就把看门狗日志存进那一轮的记录。

## 8. 给用户的话
一句结论 + 一句现状：「手机在开机循环，原因是之前精简时卸了系统要校验的应用商店，已经装回；现在 2 分钟没再重启，Wi-Fi 和微信都在线。另一台同样精简过的也已提前装回。」日志原文放附件，不贴在正文里。
