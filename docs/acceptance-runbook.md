# dsh-go fork 验收清单（真机 / 模拟器）

> 2026-11-06 版本。改动：关作者出口、关 tailscale 日志上传、私网超时 3s、切网后自动重选直连。
> 每一条都写清「怎么点、看什么、通过标准」。

## 0. 装包

```powershell
adb devices                                  # 真机 671d43b / 模拟器 emulator-5554
adb install -r D:\dsh-ai\dshgo-fork\app\build\app\outputs\flutter-apk\app-arm64-debug.apk
# 模拟器（x86_64）用 app-x64-debug.apk
```

保留数据重装（`-r`）不会丢 tsnet 节点身份，所以正常情况下**不需要重新批准 Tailscale**。

## 1. 作者服务器出口：应该 0 条

在 App 冷启动、切前台、开各个页面之后，PC 上跑：

```powershell
# 看 App 进程的所有 TCP 连接（把 <pid> 换成 App 的 pid）
Get-Process -Name "*dsh*" | Select-Object Id,ProcessName
Get-NetTCPConnection -OwningProcess <pid> -ErrorAction SilentlyContinue |
  Select-Object RemoteAddress,RemotePort,State
```

通过标准：
- **不应该出现** `go.dshtui.com` / `dl.dshtui.com` / `dsh.r2049.cn` / `forum.dshtui.com` 解析出的 IP
- **不应该出现** `log.tailscale.com`（若出现说明 `TS_NO_LOGS_NO_SUPPORT` 没生效，要回头查）
- 允许出现：Tailscale 控制面（`controlplane.tailscale.com`、`login.tailscale.com`）、DERP（`derp*.tailscale.com`）、你填的 NAS 地址

更省事的抓法（按域名看）：

```powershell
# 抓 DNS 查询（需要管理员）
Get-DnsClientCache | Where-Object { $_.Entry -match 'dshtui|r2049|log\.tailscale' } |
  Select-Object Entry,RecordName,Data
```

## 2. 场景 A：在家（WiFi）——应该是直连

1. 手机连家里 WiFi
2. App 里 host 填 NAS 的局域网地址（`192.168.151.243:22`）
3. 点连接

通过标准：
- 连上后，`Get-NetTCPConnection` 里 NAS 的 SSH 端口是**直连的 TCP 连接**（不是 127.0.0.1）
- 连接耗时 < 2 秒（3 秒超时后立刻走回落，不会等 12 秒）

## 3. 场景 B：出门（4G）——应该自动走内嵌 tailnet

1. 关 WiFi，只留移动数据
2. 断开重连，或等它自己断

通过标准：
- 第一次连接约 **3 秒内**失败 → 自动起 tsnet → 连上（第一次要等 tsnet Up，可能要 5–15 秒）
- 之后重连不再问授权（节点身份已存）
- 失败时的错误文案指向「你填的地址」，不是 127.0.0.1

## 4. 场景 C：回家（WiFi 回来了）——应该自动切回直连 ★ 本次新增

1. 保持「4G 走 tailnet」的连接不断
2. 打开 WiFi 连回家里

通过标准：
- 网络变化后 **60 秒内**（冷却期一到就试）自动重连，并且这次是**直连**
- 验证方式：PC 上 `Get-NetTCPConnection` 看是否有来自手机的直连（LAN 地址）；App 里连接不中断（重连期间 UI 短暂 reconnecting 属正常）
- 如果一直没切：说明网络变化信号没触发（`networkKicks` 没收到 connectivity 变化），需查 App 权限（部分机型要「始终允许」位置/网络能力）

## 5. 场景 D：抖动不应该反复断

1. 在同一网络下反复开关飞行模式 2–3 次

通过标准：
- 不会每次抖动都重连一遍（冷却 60s 生效）
- 最终稳定下来，不卡在 reconnecting

## 6. 回归项（别把旧功能弄坏）

| 项 | 怎么验 |
|---|---|
| host-key 校验 | 换一台机器填错指纹，应报「指纹变化」而不是连上 |
| `dsh web` 引导 | 连上后能正常打开会话（web 面板能加载） |
| 文件传输 | 传一个小文件，成功 |
| 断线重连 | 关 WiFi 再开，能自动恢复 |
| 后台切回 | App 切后台再回前台，连接还在 |

## 7. 已知未验证（写在这里免得以为通过了）

- 真机 **arm64** 上本次改动还没跑过（只有 analyze 通过 + 之前 P2 真机通过）
- 反复启停 / fd 泄漏压测未做
- 关机重启后 tsnet 身份恢复：之前验过（`install -r` 后直接 running），本次未复验
- MagicDNS 未用（只用 IP 直连，绕开了 Android 没有 `/etc/resolv.conf` 的问题）

## 8. 回滚

```powershell
# 代码回滚（只回滚本次未提交改动）
cd D:\dsh-ai\dshgo-fork
git checkout -- app/lib/remote/ssh/ssh_transport.dart app/lib/remote/ssh/ssh_link.dart `
  app/lib/remote/tailnet/tailnet_transport.dart app/lib/usage/usage_stats.dart `
  app/lib/state/update_state.dart native/tailnet-go/bridge.go
# 或直接用 .bak（每个文件旁边都有）
```
