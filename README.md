# dsh-embedded-tailnet

把 **Tailscale 的 tsnet（用户态 tailnet 网络栈）嵌进 Android App 进程**，让 App 自己能连到 tailnet 里的私有地址，**不需要开系统级 VPN**。

这是给 [DSH Go](https://github.com/ccch1mneyyy/dsh-Go)（DeepSeek Harness 的第三方 Android 客户端，AGPL-3.0）做的一块独立组件：只搬运 App 自己的那一份连接，不接管系统流量。

> 代码是独立实现的，不含 DSH Go 的任何源码，按 MIT 发布（`go.mod` 里引用的 `tailscale.com` 仍按它自己的 BSD-3-Clause）。

## 它解决什么问题

DSH Go 连远程主机有三条路：局域网直连、作者的中继（P2P/TURN/WebSocket）、SSH。想在「家里 NAS 只在内网 + 人在外面」时连上，常见做法有两个：

1. **依赖作者的中继**——流量过别人的服务器（虽然端到端加密），而且有人不想依赖第三方（见 dsh-Go 的 #553、#74、#328 一类诉求）；
2. **手机常开官方 Tailscale App（系统 VPN）**——能连，但 Android 上切 VPN（代理 → Tailscale，或反过来）会让已有 TCP 连接全部失效，SSH 卡在 `authFailed` 要手动重试（这正是 [dsh-Go#567](https://github.com/ccch1mneyyy/dsh-Go/issues/567) 报的 bug）。

**内嵌之后没有 VPN 可切**：tailnet 只活在 App 自己的进程里，跟着 App 生灭，系统 VPN 状态怎么变都不影响这条连接。

## 原理

```
┌─────────────────────── App 进程 ───────────────────────┐
│  Dart: TailnetBridge (MethodChannel)                    │
│      │ start/stop/status/forward                        │
│  Go: tailnet-go (gomobile AAR)                          │
│      tsnet.Server ─ userspace wireguard + gVisor netstack│
│      listener 127.0.0.1:<port> ── forward ──▶ 100.x.x.x:22│
│  Dart: SSH over 127.0.0.1:<port>（就是普通本地端口）      │
└────────────────────────────────────────────────────────┘
```

关键点：`tsnet` 是**用户态**网络栈（gVisor netstack + userspace wireguard），

- **不创建 TUN、不建 VPN 接口、不改系统路由表** —— 别的 App 的流量走不到它；
- 只在本机回环上监听，Dart 侧当成普通 `127.0.0.1:port` 去连，SSH 握手/主机指纹/端口转发全部复用原有链路；
- 进程退出即消失，不会残留 VPN 状态。

## 已验证

- Android 16 / arm64 真机：`state=running`，拿到 tailnet IP，成功 `SSH` 到 NAS 私有地址（读到 `SSH-2.0-OpenSSH_...` banner）；
- 切换网络（Wi-Fi ↔ 移动数据）后由 Dart 侧重选直连，无需重开 VPN（因为没有 VPN）；
- 出站审计：见 [`docs/egress-audit.md`](docs/egress-audit.md)，默认不打作者域名、不上报节点日志（`TS_NO_LOGS_NO_SUPPORT`，AAR 已重编）。

## 编译

需要 Go ≥ 1.24、`gomobile`、Android SDK/NDK：

```sh
cd go
go mod download
gomobile bind -target=android/arm64 -androidapi 26 -o ../android/tailnet.aar .
```

`go/` 是一个 `package mobile` 包，导出 `Start/Stop/Status/Forward` 等函数，`gomobile bind` 直接生成 Java/Kotlin 可调的 AAR。

## Dart 侧

`dart/tailnet_bridge.dart` 是**独立可用**的（只依赖 `flutter/services`）：

```dart
final bridge = TailnetBridge();
await bridge.start(hostname: 'phone', authKey: 'tskey-...', stateDir: '/data/.../tailnet');
final status = await bridge.status();     // {state: running, ips: [100.x.x.x]}
await bridge.forward(localPort: 2200, host: '100.x.x.x', port: 22);
// 然后当成普通本地端口：SSH 连 127.0.0.1:2200
```

`dart/tailnet_transport.dart` 是**接入示例**，它引用了 DSH Go 自己的 `ssh_transport.dart` / `link_status.dart`，展示「先试直连，retryable 失败再起内嵌 tailnet 走回环」的写法——那段不能直接编译，请按你的项目接口改写。

## Android 上的四个坑（都在代码里处理了）

1. **`HOME`/`TMPDIR` 不可写**：tsnet 会往 `$HOME`、`$TMPDIR` 落状态和日志 → 显式设 `TS_LOGS_DIR` 并指向 App 私有目录；
2. **netmon 找不到网卡**：Android 上 Go 的 `netlink` 路由监听拿不到事件，切网后不更新 → 用 `android_netmon.go` 的自注册 netmon，配合 Android `ConnectivityManager` 回调手动触发重选；
3. **NDK/SDK 路径含空格**：Windows 上 `C:\Users\...\AppData\Local\Android\Sdk` + junction 会让 cgo 找不到头文件 → 用不带空格的短路径或 junction 到 `D:\`；
4. **`PUB_CACHE` 跨盘**：Flutter 的 pub 缓存在 C 盘、工程在 D 盘时，Kotlin 增量资源任务会随机失败 → 把 `PUB_CACHE` 挪到与工程同盘。

细节和实测日志见 [`docs/`](docs/)：`source-audit.md`（源码审计）、`p2-probe-log.md`（真机探针）、`net-switch-plan.md`（切网方案）、`acceptance-runbook.md`（验收步骤）。

## 限制（先说清楚，别抱错期待）

- **不是系统 VPN**：只有 App 自己的连接走 tailnet，别的 App 不受影响——这也正是它不会掉线的原因；
- 需要 tailnet 里有一台常开设备（NAS/树莓派）当出口，以及一个 **auth key**（或一次交互式登录）；
- 依赖 DERP 中继做打洞/兜底，首次连接可能走中继，之后自动切直连（实测冷启动 0.85–1.5s 走 DERP，稳定后直连 68ms）；
- 只做了 Android（gomobile）；iOS/桌面没验证；
- 状态目录在 App 私有空间，卸载即清。

## 相关

- 上游 App：[ccch1mneyyy/dsh-Go](https://github.com/ccch1mneyyy/dsh-Go)（AGPL-3.0）
- 相关 issue：[#567 SSH host fails to reconnect after Android VPN handover](https://github.com/ccch1mneyyy/dsh-Go/issues/567)
- 本仓库只包含内嵌 tailnet 这一块的自研代码，**不含** DSH Go 的源码与资源。

## License

MIT，见 [LICENSE](LICENSE)。`tailscale.com` 及其依赖按各自许可证（BSD-3-Clause 等）分发。
