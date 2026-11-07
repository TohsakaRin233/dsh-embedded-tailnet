# 4G ↔ 家庭局域网 自动切换 —— 改造方案

- **项目**：Embedded Tailnet for dsh-go
- **代码基线**：`D:\dsh-ai\dshgo-fork`（Windows 10 / TosakaRin），HEAD = `a33c3da feat(tailnet): embedded userspace Tailnet for SSH remote access`，工作区仅 `app/android/app/libs/` 未跟踪
- **调查方式**：只读。所有结论均在源码里核实，带 `文件:行号` 与代码原文
- **日期**：2026-11-05（JST）
- **一句话**：现有的「先直连、失败才走内嵌 tailnet」**只解决"连不上"，没解决"连得不好"**——建立连接后没有任何重选路径的钩子；反向（家→4G）的 fallback 事实上已经能用，只是慢。

---

## 0. 结论速览

| # | 问题 | 答案 | 依据 |
|---|---|---|---|
| 1 | 在外用 4G（走 tailnet）→ 回家连 WiFi，会**自动切回直连**吗？ | **不会。完全没有这个钩子。** 连接建立后，唯一会被检查的是"这条连接还活着吗"，不是"有没有更好的路"。而 tailnet 那条路是 `127.0.0.1` 本地回环，**永远活着**，所以永远不会触发重连。 | `tailnet_transport.dart:24-25` + `ssh_link.dart:286/380-390/516-538` |
| 2 | 家里 WiFi 直连 → 出门切 4G，会 fallback 到 tailnet 吗？ | **会，而且不需要改代码**（已能工作）。断线检测有两条：切网事件触发的一次 5s ping（快），或 keepalive 15s/10s 超时连丢 2 次（慢）。重连时 `tailnetAwareTransport` 会重新跑"直连→tailnet"。**主要缺陷是每次都要先陪 12s 的直连超时。** | `network_watch.dart:20-27` → `connection_state.dart:194` → `ssh_link.dart:77/519-529/440-454` → `tailnet_transport.dart:29-36` |
| 3 | 「切网后自动重选最优路径并重连」最小改动面？ | **新增 1 个文件 + 改 3 个文件，约 250 行**。核心技巧：**换路 = 换底层 `SshTransport`，不换 `LocalForwarder`、不换 `DshClient`**，所以 UI 不闪断、`dsh web` 不重启、host-key 校验原样重跑。 | 见第 4 节 |

**建议改动量（人日）**

| 阶段 | 内容 | 人日 |
|---|---|---|
| P0 止血 | 路由偏好记忆（sticky route）+ 直连探测超时 12s→2s，出门不再每次白等 12s | 0.5–1 |
| P1 主菜 | 切网事件 → 探测 → 换 transport → 验收到位（含 4 层防抖、双向 fallback） | 2–2.5 |
| P2 打磨 | 诊断面板显示当前路径、tailnet 空闲自动停、共享链路文案 | 1 |

合计 **3–4 人日**（含单测与真机验收）。不碰 Go 侧、不重编 native、不改 Kotlin。

---

## 1. 现状链路地图（谁在什么时候做决定）

```
connectivity_plus ──► networkKicks()            network_watch.dart:11-42
                       │  (切网 / App 回前台)
                       ├──► connection_state.dart:194  kicks: networkKicksProvider
                       │      └──► ssh_link.dart:77  _kickSub = kicks.listen((_) => kick())
                       │             └──► kick()  ssh_link.dart:516-538
                       │                    connected ? 一次 5s ping : 无动作
                       ├──► connection_state.dart:72   本地引擎连接失败时 invalidateSelf
                       └──► ui_config_state.dart:33    UI 配置刷新

SshHostLink._connectOnce()                      ssh_link.dart:202-307
   └── _transportFactory(params, verifyHostKey) ssh_link.dart:212
         └── sshTransportFactoryProvider         connection_state.dart:174-177
               └── tailnetAwareTransport(...)    tailnet_transport.dart:19-76
                     1) 直连 params.host:params.port          :25
                     2) 仅当 HostLinkError.retryable            :26-29
                          → bridge.ensureEndpoint(...)         :36
                          → 连 127.0.0.1:<port>                 :56-65
```

关键点：**路径选择只发生在 `_connectOnce()` 里，也就是"建立连接"这一刻。连接建立后（`_installTransport`，`ssh_link.dart:380-390`）这份 transport 就被一直用到底，没有任何地方重新问过"现在该走哪条路"。**

---

## 2. 问题 1：回家连上 WiFi，会不会自动切回直连？

### 2.1 结论

**不会。** 不是"没实现好"，是"根本没有这个概念"。

### 2.2 证据

**① 选路函数是一次性的，`return` 之后生命周期就结束了**

`app/lib/remote/tailnet/tailnet_transport.dart:24-25`
```dart
  try {
    return await DartSshTransport.connect(params, verifyHostKey);
```

成功路径直接 `return`，函数返回一个已连好的 `SshTransport`。里没有任何"监听网络变化 / 定时复检"的代码 —— 整个文件 76 行，只有一个 `Future` 函数，没有类、没有订阅、没有定时器。

**② 连接建立后，从 tailnet 走的那条其实是本地回环，所以"它永远不会断"**

`tailnet_transport.dart:56-58`
```dart
    final local = SshConnectParams(
      host: '127.0.0.1',
      port: port,
```
`port` 来自 `bridge.ensureEndpoint()`（`:36`），即 Kotlin 侧在本进程内起的 loopback listener（`TailnetBridgeManager.kt:131` `return b.ensureEndpoint().toInt()`）。**这条 TCP 连接的物理介质是本机回环**：WiFi 断、蜂窝起、换 AP，它都不受影响（真正换底层路径的是 Go 那边的 netmon，Dart 侧完全看不见）。于是"断线检测"这条路对它是瞎的。

**③ `kick()` 在 connected 状态下只做一件事：ping 一下，活着就什么都不做**

`app/lib/remote/ssh/ssh_link.dart:516-538`
```dart
  void kick() {
    if (_closed) return;
    switch (_status.phase) {
      case HostLinkPhase.connected:
        unawaited(() async {
          final alive = await _ping(const Duration(seconds: 5));
          if (!alive) {
            _backoff.reset();
            _lost(HostLinkError(HostLinkErrorKind.unreachable, appL10n.remoteErrSshNetworkChanged, retryable: true));
          }
        }());
```
注意 `if (!alive)` 之后没有 `else`。ping 成功 = 无操作。而且这个 ping 走的是回环 SSH 连接（`ping()` → `_client.ping()`，`ssh_transport.dart:293`），回环 + tsnet 通路正常时必然成功。

**④ keepalive 同理，只判死活**

`ssh_link.dart:434-438`
```dart
  Future<void> _keepAliveTick() async {
    if (!await _ping(keepAliveTimeout)) return;
    final mux = _client?.mux.state;
    if (mux == DshConnectionState.connecting || mux == DshConnectionState.reconnecting) await _probeDsh();
```
`_probeDsh()`（`:365-378`）只检查"远端 DSH 端口还开着吗"，与"我现在走的哪条路"无关。

**⑤ 上层没有任何"路径"概念可观察**

`HostLinkStatus` 里唯一和路径有关的字段是 `path`（`link_status.dart:174` `final RelayPath? path;`），而 `RelayPath` 是给 WebRTC/中继链路用的枚举（`link_status.dart:67-81`：lan/p2p/direct/turn/relay）。`SshHostLink` 发出的每一处 status 都没传这个字段（`ssh_link.dart:167/193/291/480`），所以 SSH 链路在诊断里 `transport` 永远缺失（`connection_state.dart:395` `if (status.path case final path?) 'transport': path.name`）。

### 2.3 附带后果（要说清楚，别夸大）

回家后**不是不能用**，是"用着一条绕远的路"：

- 数据仍在走 tsnet（Go 侧通常会自己把 DERP 换成打洞直连），所以延迟不会离谱；
- 但多一层 `127.0.0.1 socket → Kotlin listener → tsnet 加密 → LAN` 的拷贝与加解密，功耗、吞吐、首字节延迟都比纯 LAN SSH 差；
- 最要命的是**不可观测**：用户看到"已连接"，不知道自己在绕远路，出问题时也无从判断。

---

## 3. 问题 2：出门切 4G，现有重连会不会 fallback 到 tailnet？

### 3.1 结论

**会。这条方向现有代码已经能用，不需要改动就能工作。** 需要优化的是"多久能连上"。

### 3.2 触发链（逐跳带行号）

**第一跳：切网事件**

`app/lib/remote/network_watch.dart:20-27`
```dart
        connectivity = Connectivity().onConnectivityChanged.listen((results) {
          final previous = last;
          last = results;
          if (previous == null) return;
          if (!results.contains(ConnectivityResult.none)) controller.add(null);
```
注释（`:24-25`）写得很直白：「sockets that were bound to the old interface are dead」。WiFi→4G 时 `[wifi]`→`[mobile]`，两次都会各发一次事件；中间的 `[none]` 被过滤掉。

**第二跳：事件到达 link**

`connection_state.dart:194`（`sshLinkFactoryProvider` 里）
```dart
      kicks: ref.read(networkKicksProvider),
```
`ssh_link.dart:77`
```dart
    _kickSub = kicks?.listen((_) => kick());
```

**第三跳：5 秒 ping 快速判定**

`ssh_link.dart:523-529` —— 直连的那条 socket 绑在 WiFi 网卡上，WiFi 一断，`_client.ping()` 不会得到回应，5s 超时抛异常 → `_missedPings++`（`:448`）→ 但注意 `kick` 这条路径**不看 `_missedPings`**，直接 `_lost(...)`（`:527`），比 keepalive 快得多。

**慢路径（没有切网事件时的兜底）**：`ssh_link.dart:70-71` `keepAliveInterval = 15s` / `keepAliveTimeout = 10s`，`ssh_link.dart:449` `if (_missedPings >= 2 ...) _lost(...)` —— **连丢 2 次才判死**，约 25–40s。

**第四跳：重连 = 重新选路**

`ssh_link.dart:456-461` `_lost()` → `_recover(reason)` → 循环里 `_connectOnce()`（`:491`）→ 又走到 `_transportFactory` → `tailnetAwareTransport`：

`tailnet_transport.dart:26-36`
```dart
  } on HostLinkError catch (direct) {
    if (!direct.retryable) rethrow;
    final where = '${params.host}:${params.port}';
    final int port;
    try {
      port = await bridge.ensureEndpoint(targetHost: params.host, targetPort: params.port);
```

而 4G 上直连内网地址失败，会被 `describeSocketError` 全部标成 `retryable: true`（`ssh_transport.dart:194-215`，refused / host lookup / 兜底三条分支都写着 `retryable: true`），超时也是（`:115-121`）。**所以 fallback 一定成立。**

### 3.3 时间预算（按代码常量推算，不是实测）

变量：`connectTimeout` 默认 **12s**（`ssh_transport.dart:59`，`SshConnectParams.of` 不覆盖，`:62-68`）；kick ping **5s**（`ssh_link.dart:524`）；backoff 首延迟 **0.5–1s**（`link_status.dart:247-252, 265-272`）；Kotlin 等节点 running 上限 **45s**（`TailnetBridgeManager.kt:28`），轮询 400ms（`:126`）；SSH 握手 20s / 认证 30s（`ssh_transport.dart:145-146`）；tunnel 探测上限 10s（`ssh_link.dart:72`）；`_openChannel` 等 transport 上限 20s（`ssh_link.dart:405`）。

| 场景 | 拆解 | 合计 |
|---|---|---|
| **有切网事件**（正常情况） | 5s ping + 0.5–1s backoff + **12s 直连超时** + tailnet 节点已在跑 ≈2s + bootstrap/登录 ≈1–2s | **约 20–22s** |
| **有切网事件 + 冷启动 tailnet** | 同上，但节点起机 2–45s（含 `tsnet.Up()`） | **约 25–60s** |
| **无切网事件**（后台被杀、醒来才发现断） | 25–40s 判死 + 12s 直连 + 节点 | **约 40–55s** |
| **如果省掉那 12s 直连超时**（P0 改动后） | 5s + 1s + 2s + 2s | **约 8–10s** |

**一句话：能连上，但慢，而且慢的时间几乎全花在"明知连不上还等 12 秒"。**

### 3.4 两个"体感垃圾点"（这是 P0 的由来）

1. **`connectTimeout` 12s 全部白等**：`describeSocketError` 只在拿到 `SocketException` 时才秒失败；4G 上往内网地址发包通常是**静默丢包**，只能等满 12s。
2. **没有路由记忆**：每次重连都从"直连"试起，出门在外每次重连都白付 12s，回家后每次重连又都从直连开始（这个方向歪打正着是对的，但纯属巧合）。

---

## 4. 问题 3：最小改动面

### 4.1 设计原则（先定这个，代码自然就小了）

> **换路 = 只换最底下那一层 `SshTransport`；`LocalForwarder`、`DshClient`、`_endpoint` 全部不动。**

原因在 `local_forward.dart:10-15` 的注释里已经写好了：

```dart
/// The listener outlives individual SSH connections: when SSH drops, open
/// sockets die (so DshClient's mux notices and retries), and new sockets wait
/// in [TunnelOpener] until the link has reconnected. The local port, and so
/// the DshClient's base URL, stays the same across reconnects.
```

也就是：`DshClient` 连的是 `127.0.0.1:<forwarderPort>`（`ssh_link.dart:320-324`），forwarder 的端口在 `_connectOnce` 里只 bind 一次（`:277-284`），换 transport 后新 socket 会经 `_openChannel()`（`:403-426`）自动走新 transport。

**结果：上层聊天流不断、mux 自己重连、`_ensureClient` 因为 port+token 没变而直接复用旧 client**：

`ssh_link.dart:311-317`
```dart
    final sameServer = previous != null && previous.port == endpoint.port && previous.token == endpoint.token;
    if (existing != null && sameServer && !_needsRelogin) {
      existing.mux.reconnect();
      return existing;
```

这就是本次改造成本低、风险低的根本原因 —— **不需要新造一套连接管理，只需要给现有的重连循环加一个"换个理由重连"的入口。**

### 4.2 改动清单

| # | 文件 | 动作 | 行数级 |
|---|---|---|---|
| 1 | `app/lib/remote/tailnet/route_select.dart` | **新增**：`HostRoute` 枚举、`RoutePreference`（含 `lastGood`/冷却）、`probeTcp()`、纯函数 `preferredRoute()` | ~120 行 |
| 2 | `app/lib/remote/tailnet/tailnet_transport.dart` | **改**：抽出 `TailnetRouteFactory` 类（保留旧函数签名做兼容），新增 `RoutedTransport` 包装器暴露 `route` | ~+90 行 |
| 3 | `app/lib/remote/ssh/ssh_link.dart` | **改**：新增 `_route`、`_switchPath()`、`_maybeReroute()`；`kick()` connected 分支加一行调用；构造参数加 2 个可选入参 | ~+60 行 |
| 4 | `app/lib/state/connection_state.dart` | **改**：新增 `networkRouteChangesProvider` / `sshRouteSelectorProvider`（单例，偏好要跨 link 存活）；两个工厂接上 | ~+30 行 |
| 5 | `app/lib/remote/relay/network_route.dart` | **改（小）**：加 `Stream<RemoteNetworkRoute> networkRouteChanges()`，复用已有的 `shouldRenegotiateRemoteRoute` | ~+25 行 |
| 6 | `app/lib/remote/link_status.dart` | **改（小）**：`RelayPath` 旁边加 `HostRoute? hostRoute` 字段，或直接在诊断里加一个键 | ~+10 行 |
| 7 | `app/test/remote/*`（3 个新文件） | 单测，见 4.7 | ~200 行 |

**不动**：`ssh_transport.dart`（一行都不用碰）、`local_forward.dart`、`remote_bootstrap.dart`、任何 Kotlin、任何 Go。

### 4.3 逐文件设计

#### (1) 新文件 `route_select.dart`

```dart
enum HostRoute { direct, tailnet }

/// Wi-Fi / 以太网偏好直连；蜂窝偏好 tailnet；unknown 沿用上次结论。
HostRoute preferredRoute({required RemoteNetworkRoute net, required HostRoute? lastGood}) => switch (net) {
  RemoteNetworkRoute.local => HostRoute.direct,
  RemoteNetworkRoute.remote => HostRoute.tailnet,
  RemoteNetworkRoute.unknown => lastGood ?? HostRoute.direct,
};

/// 只做 TCP 连通探测：不握手、不认证、不接触 host-key。
/// 这是"直连到底通不通"的唯一事实来源，不需要读 SSID、不需要额外权限。
Future<bool> probeTcp(String host, int port, {required Duration timeout});

class RoutePreference {
  HostRoute? lastGood;
  DateTime? _lastSwitch;
  bool _probing = false;

  Duration probeTimeout = const Duration(seconds: 2);
  Duration minSwitchGap = const Duration(seconds: 60);

  bool get inCooldown => ...;   // now - _lastSwitch < minSwitchGap
  void remember(HostRoute route) { lastGood = route; _lastSwitch = DateTime.now(); }
  void rollback() { ... }        // 切过去没成功时回退
}
```

**为什么用探测而不是 SSID**：`network_route.dart:6-14` 只区分 wifi/ethernet/mobile，拿不到"是不是**我家的** WiFi"。装 `wifi_info` 之类要位置权限，不值得。而"能不能直连上 NAS 的 SSH 端口"本身就是最终判据，2s 一个 TCP SYN 的成本可以忽略。

#### (2) `tailnet_transport.dart`

把现有函数体搬进类，行为对 `lastGood` 敏感：

```dart
class TailnetRouteFactory {
  TailnetRouteFactory(this._bridge, {RoutePreference? preference}) : _pref = preference ?? RoutePreference();
  final TailnetBridge _bridge;
  final RoutePreference _pref;

  /// 保留旧签名，测试与其它调用点不用动
  Future<SshTransport> connect(SshConnectParams params, HostKeyCheck verify) async {
    final net = await resolveRemoteNetworkRoute();          // network_route.dart:28-33
    final first = preferredRoute(net: net, lastGood: _pref.lastGood);
    // 双向 fallback：首选谁，另一个永远跟在后面
    final order = first == HostRoute.tailnet
        ? const [HostRoute.tailnet, HostRoute.direct]
        : const [HostRoute.direct, HostRoute.tailnet];
    ...
  }
}
```

要点三条：

- **顺序按偏好走**，不再是写死的"先直连"。出门在外省掉 12s；在家秒进直连。
- **双向 fallback**：首选 tailnet 失败时**仍要试一次直连**。否则"内嵌 tailnet 没登录 / 被墙 / 节点崩了"就会变成"原本公网能连的 host 也连不上"——这是硬回归，不能省。
- **仍然只在 `retryable` 时换路**：`tailnet_transport.dart:29` `if (!direct.retryable) rethrow;` 原样保留。auth 失败、key 不对、host-key 不匹配照旧不重试。

包装器（约 40 行，6 个方法纯转发）：

```dart
abstract interface class RouteAware { HostRoute get route; }

class RoutedTransport implements SshTransport, RouteAware {
  RoutedTransport(this._inner, this.route);
  final SshTransport _inner;
  @override final HostRoute route;
  @override Future<TunnelChannel> openTunnel(String h, int p) => _inner.openTunnel(h, p);
  @override Future<void> ping({Duration timeout = const Duration(seconds: 10)}) => _inner.ping(timeout: timeout);
  @override Future<void> get done => _inner.done;
  @override bool get isClosed => _inner.isClosed;
  @override void close() => _inner.close();
  @override Future<RemoteExecResult> run(String script, {Duration timeout = const Duration(seconds: 30)}) => _inner.run(script, timeout: timeout);
}
```

`SshTransport` 是 `implements RemoteShell` 的 abstract class（`ssh_transport.dart:79-92`），所以 `run()` 也要转发（`RemoteShell` 要求）。转发是纯机械代码，`identical(_transport, transport)` 之类的判断（`ssh_link.dart:302/374/385/414/449`）全部照旧成立。

#### (3) `ssh_link.dart` —— 只需要一个"重新选路"的入口

```dart
  HostRoute? _route;          // 由 _installTransport 时读出来
  bool _rerouting = false;
  final SshPathSelector? pathSelector;         // 新可选参数，null = 行为完全不变
  final Stream<RemoteNetworkRoute>? routeChanges;

  void _installTransport(SshTransport transport) {
    _route = (transport is RouteAware) ? transport.route : null;   // ← 加这一行
    ...原样...
  }

  /// 换路：复用 _recover 的整套重连流程，只换 transport。
  void _switchPath(HostRoute target) {
    if (_closed || _rerouting || _status.phase != HostLinkPhase.connected) return;
    _rerouting = true;
    _teardownTransport();                       // ssh_link.dart:392-400
    _backoff.reset();
    unawaited(_recover(HostLinkError(HostLinkErrorKind.unreachable,
        appL10n.remoteErrSshNetworkChanged, detail: 'path→${target.name}', retryable: true))
      .then((_) {}, onError: (_) {}).whenComplete(() => _rerouting = false));
  }

  Future<void> _maybeReroute() async {
    final sel = pathSelector;
    if (sel == null || _route == null || _rerouting) return;
    if (_status.phase != HostLinkPhase.connected) return;
    if (!await sel.shouldSwitch(current: _route!)) return;   // 内部含探测 + 冷却
    _switchPath(...);
  }
```

`kick()` 里只加一行（`ssh_link.dart:523-529` 的 `if (!alive)` 之后补 `else`）：

```dart
          final alive = await _ping(const Duration(seconds: 5));
          if (!alive) { ...原样... } else { unawaited(_maybeReroute()); }
```

**为什么走 `_recover` 而不是自己重连**：`_recover` 已经处理了 8 次退避（`link_status.dart:250`）、`_delay.skip()` 被新事件打断（`ssh_link.dart:530-532`）、非 retryable 立即终态（`:496-499`）。自己写一份必然是 bug 温床。

**为什么不直接调 `retry()`**：`retry()` 在 connected 时只做 `_probeDsh()`（`ssh_link.dart:548`），不会换路。

#### (4) `connection_state.dart`

```dart
/// 只认"确认的 local↔remote 切换"，unknown 一律忽略（network_route.dart:16-17 的语义）
final networkRouteChangesProvider = Provider<Stream<RemoteNetworkRoute>>((ref) => networkRouteChanges());

/// 单例：lastGood 必须跨 HostLink 存活，否则每个 host 各学一遍
final sshRouteSelectorProvider = Provider<TailnetRouteFactory>((ref) =>
    TailnetRouteFactory(ref.read(tailnetBridgeProvider)));

final sshTransportFactoryProvider = Provider<SshTransportFactory>((ref) {
  final factory = ref.read(sshRouteSelectorProvider);
  return (params, verify) => factory.connect(params, verify);
});
```

`sshLinkFactoryProvider`（`connection_state.dart:183-197`）里加两个入参即可：

```dart
      kicks: ref.read(networkKicksProvider),
      routeChanges: ref.read(networkRouteChangesProvider),   // 新增
      pathSelector: ref.read(sshRouteSelectorProvider),      // 新增
```

### 4.4 防抖 / 防风暴（四层，缺一层就会横跳）

| 层 | 位置 | 规则 |
|---|---|---|
| 1 事件层 | `network_route.dart:16-17` 现成 | `shouldRenegotiateRemoteRoute` 只在"确认的 local↔remote 变化"时放行，`unknown` 一律忽略（连 AP、开 VPN 的过渡态不会误触发） |
| 2 时间层 | 新 `networkRouteChanges()` | **1500ms 合并突发**：`wifi→none→mobile` 会连发两三次，不合并就是两三次重连 |
| 3 动作层 | `ssh_link.dart` | `minSwitchGap = 60s` 冷却 + `_rerouting` 单飞（同一时刻只换一次）+ 只在 `HostLinkPhase.connected` 动手 + 探测失败不动 |
| 4 结果层 | `ssh_link.dart` | 切过去后 **30s 内没到 connected 就回滚** `lastGood` 并把冷却拉到 5 分钟（防止"A 试 12s 失败→B→A"来回横跳） |

量化目标：**WiFi 开关 3 次 / 5 分钟内，实际换路 ≤ 1 次。**

### 4.5 为什么不破坏现有保证（逐条对账）

**host-key 校验不变**：校验发生在 `_connectOnce` 的同一个回调闭包里（`ssh_link.dart:212-239`），比较的是 `_config.hostKeyFingerprint`（`:40-44`），**与 socket 从哪来无关**。换路只是让同一台机器以另一条路再握手一次，看到的是同一把 key。若真不一致（比如 DNS 劫持），会得到 `hostKeyMismatch`，而 `hostKeyMismatchError` 的 `retryable` 用默认值 `false`（`ssh_link.dart:46-50` → `link_status.dart:48`），`_recover` 会立刻终态（`:496-499`）——**这正是期望：路径切换绝不能变成"换个路绕过 key 校验"的旁路。**

**`dsh web` 引导不变**：`_forceRestart` 只在两个地方置 true —— tunnel 被拒（`ssh_link.dart:271`）和 401 复用失败（`:326-329`），**换路不碰它**，且 `_connectOnce` 成功时会清掉（`:259`）。所以远端 `dsh web` 进程不会重启，`ensureRunning` 走"复用"分支（`:254-257`）。

**UI 不闪断**：`_ensureClient` 因 port+token 未变而复用同一个 `DshClient`（`:311-317`），`_offers`/`clients` 流不会发新 client（`:346-349` 只在 `existing != null` 时发），`RemoteConnectionController` 收到的是"状态从 connected 短暂经过 reconnecting 又回 connected"（`connection_state.dart:314-316` 只在 `state.hasError` 时才 invalidate，正常不会）。**建议 UI 上把这次切换显示成一句"正在切换线路…"而不是掉线重连**（文案可复用 `appL10n.remoteErrSshNetworkChanged`，`l10n/gen/app_localizations_zh.dart:12263` 已有）。

**首次连接流程不变**：`_openFirst`→`_recover` 的链路一行没动，`retryFirstOpen`（`ssh_link.dart:73`）语义不变。

### 4.6 明确不做的事（划界，防止方案膨胀）

- ❌ **不动 Go / Kotlin**。虽然 tsnet 侧能给出最权威的"当前是直连还是 DERP"（`ipnstate.PeerStatus.Relay`），但要走重编 AAR + 真机流程，收益不抵成本。列为 P2 可选。
- ❌ **不引入 SSID / WiFi 权限**。探测已足够，少一个权限就少一堆兼容性坑（小米的后台定位限制）。
- ❌ **不换掉 forwarder、不重建 DshClient**（见 4.1）。
- ❌ **不在换路时 stop tailnet node**。`TailnetBridgeManager.kt:111-119` 里 target 变了就 `b?.stop(); Mobile_newBridge(target)`，且 `stop()` 会清掉 `currentTarget`（`:79`），下次要重新 `tsnet.Up()`（冷启 2–45s，`TailnetBridgeManager.kt:28`）。省电改成"5 分钟无 tailnet 连接才停"，P2 再说。

### 4.7 单测清单（现状：tailnet 相关测试 **0 个**）

`app/test/` 下 grep `tailnet|Tailnet` 命中数为 0 —— 这个模块现在完全没测试覆盖，改之前先把这几条补上：

| 文件 | 覆盖 |
|---|---|
| `test/remote/tailnet_route_test.dart` | `preferredRoute` 全枚举、冷却生效、`shouldRenegotiate` 去重、1500ms 合并 |
| `test/remote/tailnet_fallback_test.dart` | 直连 fail-fast → tailnet 成功；tailnet 先失败 → 直连成功（双向）；非 retryable 不换路 |
| `test/remote/tailnet_reroute_test.dart` | 用 `test/remote/support/fake_ssh.dart` 驱动：connected → kick → selector 返回 true → 断言 ①`_transport` 换了 ②`identical(client, before)` 为真（client 未换）③host-key 回调被再次调用 ④`_forceRestart` 仍为 false ⑤换路中途 `routeChanges` 再发事件不触发第二次 |

---

## 5. 边界与已知坑（写代码前先看这段）

1. **前提：内网地址必须能从 tailnet 到达。** `ensureEndpoint(targetHost: params.host, targetPort: params.port)`（`tailnet_transport.dart:36`）把**配置里的同一个 host:port** 当作 tailnet 内的目标。所以要么 NAS 本身在 tailnet 里（MagicDNS / 100.x），要么有 subnet router 宣告了家里网段。这点现有实机已验证过能 SSH 到 NAS，说明前提成立；但换到"没跑 tailscale 的机器"上就只剩直连一条路，方案要能优雅退化（探测失败 → 不切、不报错）。
2. **`ensureEndpoint` 一次只能盯一个 target。** `TailnetBridgeManager.kt:114-119`：target 一变就把旧 bridge `stop()` 重建。**多台远程主机交替连接会互相踢掉对方的 tailnet 端点**。这不是新问题，但自动换路会把它变得更频繁 —— 建议 P1 顺带在 `TailnetBridgeManager` 里改成按 target 缓存多个 bridge（Kotlin 小改，约 30 行），或至少加一条日志便于诊断。
3. **tailnet 节点没人停。** `app/lib` 内 `TailnetBridge.stop()` 的**生产调用点为 0**（只有 `dev/runtime_probe_main.dart:91`）。一旦起过就常驻，含 Go runtime 内存与心跳流量。这是 P2 的省电项。
4. **loopback ping 是"检测盲区"**：它只能告诉你"经 tsnet 到目标这条路还通吗"，不能告诉你"是不是绕远了"。所以**别试图用 ping 时延做选路**，一定用"直连探测 + 网络类型"。
5. **`_missedPings` 的计数怪癖**：`kick` 路径不看计数（`ssh_link.dart:527` 直接 `_lost`），keepalive 路径要 2 次（`:449`）。换路逻辑复用 `kick` 的话，注意别把"误判一次"当"网络没了"。
6. **`probeTcp` 必须限时且可取消**：`minSwitchGap` 期间不探测；探测飞行中再来事件直接丢弃（`_probing` 标志），否则 WiFi 抖动时会有并发 SYN 堆积。
7. **不要用探测结果直接判定"路径质量"**：家里 WiFi 的 SSH 端口通不代表延迟低（可能是隔壁 5G 同名 SSID 走了运营商链路）。P1 只做"通/不通"二值决策；要做"择优"需要更细的信号（二期）。

---

## 6. 工作量与排期建议

| 阶段 | 交付 | 人日 | 前置 |
|---|---|---|---|
| P0 | ①`RoutePreference` + `preferredRoute`（纯函数，可先单测）②`connectTimeout` 12s→3s（探测式）③双向 fallback 顺序化 | 0.5–1 | 无 |
| P1 | ④`RoutedTransport` + `_route` ⑤`_switchPath`/`_maybeReroute` ⑥`networkRouteChanges` + 四层防抖 ⑦`connection_state` 接线 ⑧3 个单测文件 | 2–2.5 | P0 |
| P2 | ⑨诊断面板显示 `route`（`HostRoute` 进 `HostLinkStatus`）⑩tailnet 空闲自动停 ⑪多 target bridge 缓存 | 1 | P1 |

**风险最高的两处**（建议 code review 重点看）：`_switchPath` 的并发（与 `_recover`/`kick`/`close` 交叉）、`probeTcp` 的取消语义。

---

## 7. 验收清单（真机）

### 7.1 环境与构建

```powershell
# 在 PC 上（PowerShell，工作目录 D:\dsh-ai\dshgo-fork\app）
flutter --version
flutter build apk --release          # 或既有的 release 流程
adb devices                          # 应看到 Xiaomi arm64 / Android 16
adb install -r build\app\outputs\flutter-apk\app-release.apk
```

日志窗口（**两个都开**，一个 Dart、一个 Kotlin）：

```powershell
adb logcat -c
adb logcat -v time -s flutter Tailnet
# 建议在 _switchPath / _maybeReroute 里补一行
#   debugPrint('[route] current=$_route probe=${probeResult} switch→$target')
# 便于用日志判定，而不是靠猜
```

网络开关（按可靠度排序）：

```powershell
adb shell svc wifi disable           # 关 WiFi（进 4G）
adb shell svc wifi enable            # 开 WiFi（回家）
adb shell svc data disable           # 关蜂窝
adb shell settings get global wifi_on
# 飞行模式（部分 Android 16 上 cmd 方式不可用，用 settings + broadcast）
adb shell "settings put global airplane_mode_on 1; am broadcast -a android.intent.action.AIRPLANE_MODE --ez state true"
```

### 7.2 场景与通过标准

**场景 A：4G（tailnet）→ 回家 WiFi（应切回直连）**

1. 关 WiFi，确认手机走蜂窝：`adb shell dumpsys connectivity | Select-String -Pattern "NetworkAgentInfo.*MOBILE"`；
2. App 连 NAS，确认可用（发一条消息，收到回复）；
3. `adb shell svc wifi enable`，等到 WiFi 关联（`settings get global wifi_on` = 1 且 `dumpsys connectivity` 出现 WIFI）；
4. 观察日志出现 `[route] ... switch→direct`。

- ✅ WiFi 关联成功 → `route=direct` **≤ 30s**；
- ✅ 期间 `DshClient` 未换对象（日志 `client reused` / `identical` 断言的等效输出）；
- ✅ **无 host-key 弹窗**、**远端 `dsh web` pid 不变**（`adb shell` 进 NAS：`pgrep -f "dsh web"` 前后对比）；
- ✅ 切换期间聊天不报错，最多显示一次"正在切换线路"。

**场景 B：家里 WiFi（直连）→ 出门 4G（应 fallback 到 tailnet）**

1. WiFi 下连上 NAS，确认 `route=direct`；
2. `adb shell svc wifi disable`；
3. 观察 `reconnecting` → `connected`。

- ✅ 蜂窝生效 → `connected` **≤ 45s**（tailnet 节点已启动）/ **≤ 90s**（冷启动）；
- ✅ 重连尝试次数 ≤ 2（`status.attempt` 字段）；
- ✅ 最终不落 `failed`；
- ✅ 消息可继续收发（mux 复用）。

**场景 C：抖动（防风暴）**

5 分钟内 `svc wifi disable/enable` 交替 3 次：

- ✅ 实际 `switch→` 日志出现 **≤ 1 次**（冷却生效）；
- ✅ 无 `failed`、无 ANR、无反复 `connecting`；
- ✅ tailnet 节点未反复 `tsnet.Up()`（`adb logcat -s Tailnet` 里 `ensureEndpoint` 的 rebuild 次数 ≤ 1）。

**场景 D：失败态（不能更糟）**

1. 在 tailnet 侧退出登录（`phase=needsLogin`，`tailnet_bridge.dart:17`）；
2. 在家 WiFi：必须仍能**直连成功**（回归项，不能被 tailnet 拖死）；
3. 切到 4G：应进 `failed` 且 UI 可点"重试"（`ssh_link.dart:534` `case HostLinkPhase.failed: if (_status.error?.retryable ?? false) retry();`）。

- ✅ 3 条全过。**第 2 条是本次改造的硬约束**：偏好记忆绝不能把"直连可用"的 host 带进死路。

### 7.3 回归项（改完必须重跑，一条都不能退）

| # | 项 | 判据 |
|---|---|---|
| 1 | 首次连一台新主机 | 仍会弹 host-key 确认，pin 后能连（`ssh_link.dart:220-237`） |
| 2 | host-key 不匹配 | 仍**不重试**、直接报 fingerprint 变化（`link_status.dart:48` 默认 `retryable:false`） |
| 3 | 密码/私钥错误 | 仍立即终态，不换路重试（`tailnet_transport.dart:29`） |
| 4 | `dsh web` 未运行 | 仍会拉起来（`ssh_link.dart:254-257`），换路时不重启 |
| 5 | 远端 DSH 换端口 | 仍会 `_forceRestart` 重启（`ssh_link.dart:271/326-329`） |
| 6 | 只连本地引擎（不用 SSH） | 完全不受影响（`connection_state.dart:430-442` 分支未改） |
| 7 | `flutter test test/remote` 全绿 | 含新增 3 个文件；`fake_ssh.dart` 不传 `pathSelector` 时行为与今天逐字节相同 |

---

## 8. 未决 / 待负责人拍板

1. **`HostRoute` 要不要进 `HostLinkStatus`（影响 UI 文案与诊断面板）？** 本方案里做了最小版（诊断里加键），完整版要动 `link_status.dart` + 各 UI 行组件，多约 0.5 人日。
2. **多 target 的 bridge 缓存（§5 第 2 条）是 P1 还是 P2？** 如果有"手机同时连 NAS + 家里台式"的用法，建议提到 P1，否则自动换路会放大这个已有缺陷。
3. **冷却时长取值**（60s 切换 / 5 分钟回滚）是按"人走路进门的时延"估的，需要在真机上用场景 C 校准。
4. **`connectTimeout` 从 12s 降到 3s** 会影响"慢速但可达"的公网链路（比如 4G 弱信号下真能连上的情况）。P0 里建议只对**内网私网地址**（RFC1918）缩短，公网地址保持 12s —— 这一条需要确认 NAS 的地址形态。

---

## 附录：本次调查读过的文件（全部只读）

```
app/lib/remote/tailnet/tailnet_transport.dart          (76 行, 全读)
app/lib/remote/tailnet/tailnet_bridge.dart            (155 行, 全读)
app/lib/remote/ssh/ssh_link.dart                      (602 行, 全读)
app/lib/remote/ssh/ssh_transport.dart                 (318 行, 全读)
app/lib/remote/ssh/local_forward.dart                 (前 90 行)
app/lib/remote/network_watch.dart                     (42 行, 全读)
app/lib/remote/link_status.dart                       (302 行, 全读)
app/lib/remote/relay/network_route.dart               (34 行, 全读)
app/lib/state/connection_state.dart                   (443 行, 全读)
app/android/app/src/main/kotlin/.../tailnet/TailnetPlugin.kt        (69 行, 全读)
app/android/app/src/main/kotlin/.../tailnet/TailnetBridgeManager.kt (141 行, 全读)
app/test/remote/remote_network_route_test.dart        (40 行, 全读)
app/test/remote/network_resume_path_test.dart         (52 行, 全读)
```

未在本环境执行：`flutter analyze` / `flutter test` / `flutter build`（沙箱无 Flutter 工具链），本方案所有改动均未落盘到 `D:\dsh-ai\dshgo-fork`。
