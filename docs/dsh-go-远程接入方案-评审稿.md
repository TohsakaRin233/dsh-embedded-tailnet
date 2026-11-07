# dsh-go 手机端远程接入方案评审稿

> 版本：2026-11-05 21:2x JST · 作者：鲸鱼娘（本会话）
> 用途：给外部技术顾问评审。请重点看第 6 节「风险与未验证项」和第 8 节「评审问题清单」。
> 结论先行：**推荐在 dsh-go 里内嵌 Tailscale 的 userspace 网络栈（tsnet / libtailscale），
> 用「进程内 tailnet + 127.0.0.1 本地桥」接进 App 已有的「SSH 直连」通道；不改协议、不改 dsh_remote 包，
> 改动面约 6 个文件。最大未知是 tsnet 在 Android 上能否稳定跑，需要先做 PoC。**

---

## 1. 需求（主人原话拆解）

| 编号 | 需求 | 约束来源 |
|---|---|---|
| R1 | 手机端用 dsh-go（Flutter App），不是浏览器 | 主人选型 |
| R2 | 不走作者/第三方服务器，自己控制通路 | 「不走作者服务器，哪天服务器没了就不太方便了」 |
| R3 | 手机不额外装 Tailscale 客户端，能内置最好 | 「我不想额外下 tailscale 客户端……还是说 dshgo 里内置轻量的」 |
| R4 | 不能代理全局流量，只让 dsh 走这条通道 | 「不能代理全局流量，只能代理 dsh go 的」 |
| R5 | 网络自适应：在家走局域网，在外走穿透 | 「在家就局域网，在外面就穿透」 |
| R6 | 手机上能给鲸鱼娘开权限/审批，QQ 桥断了也不影响 | 「我可以给你开权限或者审批了，qq 桥掉了也不碍事」 |

**已经排除的路线（别再提）**

- 公网端口转发：家里是日本公寓自带网络，没有主人可管的路由器/光猫，入站口开不了。
- 依赖作者默认 relay（dsh.r2049.cn）：违反 R2。
- 让主人装官方 Tailscale App：违反 R3（虽然它本身不代理全局，见 §3 方案 A）。

---

## 2. 现状事实（带源码/环境证据）

### 2.1 App 架构

- Flutter 3.47 + Riverpod 3.4，`app/lib` 770+ Dart 文件（全仓 5572 个文件）；原生层 Kotlin 负责 proot 跑 Ubuntu + Node + dsh（本地模式）。
- 依赖里有关键两项：`dartssh2 ^4.1.0`（SSH 客户端）、`dsh_remote`（本地 path 包，远程协议）。
- Android：`minSdk 26`、`targetSdk 28`（为 rootfs 内 exec 放开限制），Manifest 里**没有 VpnService 权限**。

### 2.2 两条远程通道

**通道 1：SSH 直连（主人要的那条）**

```dart
// app/lib/remote/ssh/ssh_transport.dart
socket = await SSHSocket.connect(params.host, params.port, timeout: params.connectTimeout);
// host/port/user/auth 来自 SshHostConfig；host 是任意字符串，填 100.x.y.z 完全合法
```

- 进了 SSH 之后，用 `openTunnel(host, port)` 开 direct-tcpip 通道，把 DSH 的 HTTP/WS 端口（宿主 3080）转发到手机本地。
- 传输层有工厂抽象 `SshTransportFactory`，socket 可替换 —— 这是后续做内嵌通道的注入点。

**通道 2：DSH Remote（P2P）**

```dart
// packages/dsh_remote/lib/src/transport/peer_connection.dart
// LAN → P2P(WebRTC DataChannel 'dsh') → TURN → 加密 WebSocket Relay
// Noise_IK 端到端；TURN 凭证由 Server 短时下发；只走 data channel，不碰音频
```

- 断网自适应逻辑已存在：`network_route.dart` 按 WiFi/以太网 = local（尝试全部传输），蜂窝 = remote（只试 p2p/turn/relay）。
- 也就是说 **R5 的「自适应」App 里已经有骨架**，缺的是「怎么进 tailnet」。

**通道 3：自建 Server**（ds-harness-remote，MIT，李国宝）

- UI 里可填 `serverUrl` + 邮箱登录自己部署的那份。但自建 server 本身要公网可达 → 公寓网做不到，除非再套 CF Tunnel/VPS，等于把 R2 又还回去了。**不作为主路线。**

### 2.3 Host 侧（NAS）现状 —— 关键有利条件

| 项 | 值 | 证据 |
|---|---|---|
| Tailscale 版本 | 1.104.5 | 宿主 `tailscale version` |
| tailnet IP | 100.126.219.7（节点名 nas） | `tailscale status` |
| tailnet 账号 | tohsakarin990203@ | 同上 |
| 同 tailnet 另一节点 | 100.92.182.109（z4proplus-7tym） | 同上 |
| sshd | 0.0.0.0:22 监听 | 宿主 `ss -ltnp` |
| dsh Web | 容器 3080-3081 → 宿主 3080-3081 | `docker ps` |

**含义：NAS 这头一行都不用改。** 手机只要能进 tailnet 并解析 100.126.219.7，SSH 直连通道立刻可用。

---

## 3. 候选方案与对比

| 方案 | 做法 | R2 自控 | R3 免装 | R4 不代理全局 | 改动成本 | 结论 |
|---|---|---|---|---|---|---|
| **A 官方 Tailscale App** | 手机装官方 App，只开 tailnet 路由 | 中（控制面在 Tailscale） | ✗ 要装 | ✓ 默认只路由 100.64/10 与 fd7a::/48 | 0（只改 App 里填的地址） | 兜底/对照基线 |
| **B 内嵌 userspace**（推荐） | Go 侧 tsnet 起节点，进程内网栈，127.0.0.1 桥进 App | 中高 | ✓ | ✓ 完全不建系统 VPN | 3～5 人日 PoC | **主选** |
| C 自建 relay + CF Tunnel | 自部署 ds-harness-remote，cloudflared 出站 | 高 | ✓ | ✓ | 中（要部署+证书） | 备选：B 失败时 |
| D 公网端口转发 | 路由器开洞 | 高 | ✓ | ✓ | 低 | ✗ 已排除（公寓网） |
| E Headscale 自建控制面 | 自建 Tailscale 控制服务器 + DERP，客户端仍用 tsnet | 最高 | ✓ | ✓ | 高（要公网 VPS） | B 的彻底版，后续可升级 |

**为什么不是 A：** 它技术上已经满足 R4（Android 上 Tailscale 只把 CGNAT 段写进 VPN，浏览器/微信照走原网），但违反 R3，而且多一个常驻 VPN 图标对主人是噪音。

**为什么 B 是主选：** tsnet 的设计目标就是「编进你的程序、从 userspace 拿到一个 tailnet IP」，不创建 TUN、不接管系统路由 —— 正好同时命中 R3 + R4。libtailscale 的 README 原话："compile Tailscale into your program and get an IP address on a tailnet, entirely from userspace."

---

## 4. 推荐架构（B 方案落地形态）

```
┌─────────────────────── 手机（dsh-go，Flutter） ───────────────────────┐
│  Dart 层                                                             │
│    SSH 直连  host = 127.0.0.1  port = <localBridgePort>              │
│        │                                                             │
│        ▼                                                             │
│  Native 层（Kotlin / Go）                                            │
│    Go: tsnet.Server (net=tsnet, hostname=dshgo-phone)                │
│        ├─ Dial("100.126.219.7:22")                                   │
│        └─ 127.0.0.1:<random> 监听，双向 io.Copy 桥接                  │
│    Kotlin: 启动/停止/状态查询的 platform channel                      │
└──────────────────────────────────────────────────────────────────────┘
          │  tailnet（WireGuard，NAT 打洞失败走 DERP 中继）
          ▼
    NAS（已装 Tailscale 1.104.5，100.126.219.7）
      sshd :22  →  direct-tcpip →  dsh Web 3080
```

### 4.1 为什么用「127.0.0.1 本地桥」而不是 JNI 直传 fd

- Dart 没有稳定公开的 `Socket.fromRawFd`，把 native socket 塞回 Dart 要么走 platform channel 逐包转发（性能与复杂度都差），要么改 `dsh_remote` 包。
- 本地环回一次拷贝的成本在手机上可以忽略（SSH 流量本来就是 KB/s 级，只有文件传输会高一些）。
- 好处：**Dart 侧一行不用改**，SSH 通道把它当普通 TCP 服务器，主机名填 `127.0.0.1`。App 的 host-key 校验、隧道、重连全沿用现有逻辑。

### 4.2 认证与生命周期

- 首次：用 tailnet 的 **auth key**（一次性、可设 preauthorized + ephemeral），存在 `flutter_secure_storage`，不走明文配置。
- 之后：tsnet 的 state 落盘到 App 私有目录（`node-state` 目录），重启免登录。
- 交互登录兜底：tsnet 提供 LoginURL，弹 WebView 让主人点一下（这样连 auth key 都不用发）。
- 生命周期：App 前台时 ensureUp；后台/被杀即断（远程本来也要 App 在前台）。

### 4.3 R5 自适应怎么落

现有 `network_route.dart` 只区分「WiFi/以太网 vs 蜂窝」。加一层：先探测 `192.168.151.243:22`（LAN 直连宿主），3 秒不通再启用内嵌 tailnet 连 `100.126.219.7:22`。两套 host 配置都填，App 里做一个「内置通道」开关即可。

---

## 5. 改动面清单（上游兼容性）

| 文件 | 动作 | 说明 |
|---|---|---|
| `app/packages/tailscale_bridge/` | **新增** | 独立 Flutter plugin：Dart API + Android 实现（Go 编译产物 + Kotlin channel） |
| `app/android/settings.gradle(.kts)` | 改 1 行 | 引入本地 plugin |
| `app/lib/remote/ssh/ssh_host_config.dart` | 加字段 | `embeddedTailnet: bool`（或新 host 类型） |
| `app/lib/ui/remote/ssh_host_form.dart` | 加 1 个开关 + 1 段说明 | UI |
| `app/lib/remote/ssh/ssh_transport.dart` | **不改** | host 就是 127.0.0.1 |
| `packages/dsh_remote/**` | **不改** | 协议、传输、Noise 全部不动 |

**上游更新会不会冲掉？** 不会冲掉新增目录；冲突只可能出现在 `ssh_host_form.dart`（UI 改动频繁）和 `settings.gradle`。
建议做法：fork 出 `feat/embedded-tailnet` 分支，上游更新时 `git rebase`；把 UI 改动压到最小（一个 ListTile + 一个 bool 字段），冲突基本是「加一行」级别。

**另一个风险**：上游若自己内置了类 tailscale 能力，我们的分支就该弃用 —— 这属于「好事」。

---

## 6. 风险与未验证项（评审重点）

| # | 风险 | 严重度 | 现状 | 应对 |
|---|---|---|---|---|
| K1 | **tsnet 在 Android 上能否跑通**（gvisor netstack、DNS、MTU、arm64） | 高 | **未实测** | P1 PoC：先在 Linux 编 tsnet demo 连通 NAS，再 `GOOS=android GOARCH=arm64` 交叉编译跑真机 |
| K2 | gomobile/NDK 构建链与 Gradle 集成（Go 产物随 APK 打包） | 中 | 未做 | 用 `-buildmode=c-shared` 出 `.so` + JNI 薄壳，或 gomobile bind 出 AAR；两路都试，留一个 |
| K3 | Android 后台/省电策略杀进程 → tailnet 断 | 中 | App 已有前台服务骨架（`DshService`） | 把 bridge 挂到已有前台服务里，不新起服务 |
| K4 | auth key 泄露/管理 | 中 | 无 | 用 ephemeral + 定时换；或改用交互登录 URL |
| K5 | NAT 打洞成功率（双方都在运营商 NAT 后） | 中 | NAS 与手机均非公网 | tailnet 自动降级 DERP 中继（中继带宽有限，但 SSH 够用） |
| K6 | 上游更新冲突 | 低 | 计划 rebase | 见 §5 |
| K7 | 许可证：tailscale 客户端 BSD-3，dsh-go MIT | 低 | — | 保留 NOTICE；libtailscale 是 BSD-3 |

**K1 的诚实说明**：libtailscale 的 README 只承诺 C 库 + userspace，没有官方 Android/Flutter 支持声明。
tailscale 自家 Android 客户端走的是 `VpnService` + TUN，不是进程内模式。
所以「tsnet 在 Android 上能跑」目前是**合理推断**（Go 官方支持 android/arm64，netstack 纯用户态不依赖 TUN），
**不是已验证事实**。这也是为什么必须先做 P1 PoC 再谈开发排期。

---

## 7. 分阶段计划

**P0（0.5 天，零代码）** — 用官方 Tailscale App 临时装一台手机，验证：
1. 手机能 ping 通 100.126.219.7
2. dsh-go 的 SSH 直连填 `100.126.219.7` + 端口 22 能连上并打开 DSH
→ 证明「Host 侧零改动」且网络路径成立，再去掉 App 换成内嵌。

**P1（1～2 天）** — tsnet PoC：
1. Linux 上写 30 行 Go：tsnet 起节点 → `net.Dial("tcp","100.126.219.7:22")` → 本地 127.0.0.1 桥
2. 交叉编译 `GOOS=android GOARCH=arm64`
3. 塞进真机（或用 Termux 先跑 arm64 二进制）验证连通
→ 这一步决定 B 方案生死。

**P2（2～3 天）** — 接进 App：
gomobile/AAR → Flutter plugin → host 填 127.0.0.1 → SSH 握手成功 → 打开会话。

**P3（1～2 天）** — 产品化：UI 开关、auth key 存储、LAN/tailnet 自适应、断线重连、诊断日志。

**总计**：4～7 人日（含真机调试）。若 P1 失败，退到方案 C（自建 relay + CF Tunnel），排期 +2 天。

---

## 8. 给评审帮手的提问清单

1. tsnet 在 Android（arm64, API 26+）上的已知坑有哪些？gvisor netstack 的 DNS/MTU 是否有社区案例？
2. `c-shared + JNI 薄壳` 与 `gomobile bind AAR`，在 Flutter 项目里哪个更好维护？（二进制体积、符号冲突、Gradle 增量）
3. 本地 127.0.0.1 桥 vs 直接把 fd 交给 Dart，评审倾向哪种？有没有第三种更干净的（比如 Flutter 侧自带 socket 抽象）？
4. auth key 的存放与轮换，有没有比 flutter_secure_storage 更合适的？ephemeral node 每次上线都新建会不会产生大量僵尸节点？
5. 上游 rebase 成本我估得对不对（§5 的 6 文件）？有没有漏掉的构建/签名相关文件？
6. 如果主人在意「连控制平面也不依赖 Tailscale 官方」，Headscale + 自建 DERP 的额外成本值不值？（现在 NAS 已经在用官方 tailnet，换 Headscale 要迁移）
7. 安全面：内嵌 tailnet 后，手机端 ACL 应该怎么收紧（只允许 22 和 3080？tag 化？）

---

## 9. 一句话总结

NAS 侧的 Tailscale 已经在跑，缺的只是手机端怎么「零安装」地进同一个 tailnet。
官方 App 能满足功能但违反「不装客户端」；内嵌 tsnet 是唯一同时满足「免安装 + 不代理全局 + 自控」的路子，
改动面小、与上游解耦，但**必须先花 1～2 天证明 tsnet 在 Android 上跑得动**再排期。

---

*附：本文所有源码引用来自 `github.com/ccch1mneyyy/dsh-go`（main 分支，MIT），Host 侧数据来自本机实测（2026-11-05 21:1x JST）。*
