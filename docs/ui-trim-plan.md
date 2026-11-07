# DSH Go 去作者服务器 · 页面与请求裁剪清单

> 目标：fork（`D:\dsh-ai\dshgo-fork`，HEAD `a33c3da`）只走「内嵌 tailnet + SSH 直连」，不再碰作者 `dsh.r2049.cn` / `*.dshtui.com`。
> 方法：本地 clone `/workspace/tmp/dsh-audit/dsh-Go`（`48f8665`）先摸结构，**全部结论用 PC 上 fork 复核**（fork 相对 `48f8665` 只有 `a33c3da`（tailnet，新增 15 文件 + 改 `connection_state.dart`）、两个 README docs 提交，工作区干净 → 下列行号与 fork 一致，已逐条核对）。
> 只读审计，未改 PC 任何文件。

---

## 0. 结论速览

**主线仍会碰到作者服务器：是。三个口子与 SSH 无关，是 App 全局行为。**

| # | 请求 | 触发时机 | 证据 |
|---|---|---|---|
| 1 | `POST https://go.dshtui.com/api/usage/v1`（匿名使用统计） | **每次冷启动 + 每次回到前台**，默认开 | `app/lib/usage/usage_net.dart:21`、`app/lib/usage/usage_stats.dart:98,214,241`、`app/lib/app.dart:60` |
| 2 | `GET https://dl.dshtui.com/update.json`（回落到 `go.dshtui.com`、GitHub） | 启动后 6 秒，每天一次，默认开 | `app/lib/update/update_config.dart:43,48`、`app/lib/ui/update/update_launch_check.dart:46`、`app/lib/ui/shell/home_shell.dart:342` |
| 3 | `GET https://dsh.r2049.cn/api/v1/auth/oauth/providers`（relay 登录方式探测） | **每次冷启动**（未登录时），即使只用 SSH | `app/lib/ui/shell/home_shell.dart:306` → `pets/pet_overlay_bridge.dart:24-37` → `pets/pet_overlay_actions.dart:47` → `state/hosts_state.dart:490-503` → `packages/dsh_remote/lib/src/account/remote_account.dart:376` → `server/server_api.dart:299` |

第 3 条最阴：它不是「你打开远程页才连」，是宠物状态在首页 shell 里 `ref.listen` 就被建起来的。**结论：SSH 直连模式 ≠ 不发外网请求**，要自己接管必须把这三处掐掉。

**能直接隐藏的 Top5 入口**（不影响 SSH + tailnet 主线）：

1. 远程页「远程 Web 控制台」整段 —— `ui/remote/remote_screen.dart:120-130` + `app_info.dart:41`（`https://dsh.r2049.cn/app`，点开就是一整个作者网页）
2. 远程页「DSH Remote 账号」整段（登录卡 + 主机列表 + 署名）—— `remote_screen.dart:365-470`（调用点 `:99`）
3. 远程页 relay 失败时的「重新登录」与自建 server 输入 —— `remote_screen.dart:209-216`、`relay_login_sheet.dart:30,73,140`
4. 宠物菜单 / 切主机弹窗里的 relay 主机 —— `pets/pet_overlay_actions.dart:47`、`ui/remote/host_switcher.dart:86`
5. 设置里的「改进 DSH Go（使用统计）」+ 首次引导里的勾选框 —— `ui/settings/privacy_page.dart:39`、`ui/settings/usage_stats_section.dart:25`、`ui/onboarding/usage_notice_row.dart:35`

---

## 1. 清点表

「是否阻断主线」= 会不会挡住「SSH 直连 + 内嵌 tailnet」这条线。UI 隐藏类一律「无关」；会发外网请求或抢 activeHost 的算「阻断」。

### 1.1 远程连接页 / 通道选择

| 文件:行号 | 入口 | 阻断主线 | 建议 | 成本 |
|---|---|---|---|---|
| `ui/remote/remote_screen.dart:46-48` | 「远程 Web 控制台」WebView，`url = AppInfo.remoteConsole`，副标题写死 `'dsh.r2049.cn'` | 是（点开=整站作者页，且会占住 App 前台） | 隐藏入口 | 删 12 行 + 2 条 l10n |
| `state/app_info.dart:41` | `static const remoteConsole = 'https://dsh.r2049.cn/app';` | 常量源 | 删除常量 | 1 行 |
| `ui/remote/remote_screen.dart:120-130` | 上面那个入口所在的 `SettingsSection(remoteOtherMethods)` | 否 | 整段删除 | 8 行 |
| `ui/remote/remote_screen.dart:378-391` | `_RelaySection` 未登录态：`DSH Remote` 卡 + 登录卡 + `RelayCredit` 页脚 | 是（建 relay 账号态会打 `r2049.cn`） | 隐藏 | 25 行 |
| `ui/remote/remote_screen.dart:392-470` | `_RelaySection` 已登录态：账号行、刷新主机、主机列表 | 是 | 隐藏 | 60 行 |
| `ui/remote/remote_screen.dart:471-664` | `_orderedProviders` + `_LoginCard` + `_Waiting`（GitHub/知乎/Google/邮箱） | 是 | 随上一行一起删 | 200 行左右，整块可切 |
| `ui/remote/relay_login_sheet.dart:30,73,140` | 自建 server 输入框 + 「改回默认服务器 dsh.r2049.cn」 | 是（`loginWithPassword` 直连任意 origin） | 默认关闭 / 隐藏 | 15 行 |
| `ui/remote/relay_credit.dart:30-...` | 「服务由 liguobao/ds-harness-remote 提供」署名链 | 否（外链，点了才开浏览器） | 隐藏（随 relay 段一起） | 跟段落一起 |
| `state/hosts_state.dart:382` | `const defaultRelayServer = 'https://dsh.r2049.cn';` | 是（所有 relay 请求的兜底 origin） | 改常量或删 provider | 1 行 |
| `packages/dsh_remote/lib/src/server/server_api.dart:13` | `final Uri defaultRemoteServer = Uri.parse('https://dsh.r2049.cn');` | 是（account 兜底） | 改常量 | 1 行 |
| `state/hosts_state.dart:336-379` | `RelayServerController`（存/读自建 server） | 否，但连带 `relayAccountProvider` | 保留或整块删 | 45 行 |
| `state/hosts_state.dart:388-402` | `relayBackendProvider`（构造 `DshRemoteRelayBackend`） | 是（`currentUser` / `listHosts` 都从这走） | **保留 provider 但断掉启动期构建**（见 §2.3） | 3-5 行 |
| `ui/remote/host_switcher.dart:86` | 「切换主机」弹窗里的 relay 主机 | 否（点开弹窗才 build） | 隐藏 relay 行 | 6 行 |
| `pets/pet_overlay_actions.dart:47` | 宠物菜单的 relay 主机 | 否，但它是**启动即建 account 的元凶** | 隐藏 relay 行 | 1 行（去掉那个 for） |
| `dev/remote_p2p_probe_main.dart:47` | 开发探针，传 `serverUrl` | 否（不在 release 入口） | 保留 | 0 |

### 1.2 登录 / 注册

| 文件:行号 | 入口 | 阻断主线 | 建议 | 成本 |
|---|---|---|---|---|
| `ui/remote/remote_screen.dart:209-216` | 失败卡上的「重新登录」→ `notifier.login()` | 是 | 隐藏（该分支只在 relay 主机才出现，隐藏 relay 即消失） | 0 |
| `ui/remote/relay_login_sheet.dart` 整文件（155 行） | 浏览器/QR 登录 + 邮箱密码登录 | 是 | 隐藏 | 155 行 |
| `state/hosts_state.dart:377` | `loginWithPassword(email, password)` | 是 | 随 UI 一起 | 2 行 |

**没有注册流程**（账号由 relay server 侧发放），也**没有订阅/付费/捐赠**：全库 grep `sponsor|donate|打赏|捐赠|订阅|充值` 无命中；唯一「求 star」是 `state/star_prompt.dart:29`：

```dart
const starPromptEnabled = bool.fromEnvironment('DSH_STAR_PROMPT');
```

默认 false，且 `home_shell.dart:346` 的 `StarPromptHost` 在关闭时不渲染 → **无需处理**。

### 1.3 遥测 / 诊断上报

| 文件:行号 | 入口 | 阻断主线 | 建议 | 成本 |
|---|---|---|---|---|
| `usage/usage_net.dart:21` | `defaultUsageEndpoint = Uri.parse('https://go.dshtui.com/api/usage/v1')` | **是（隐私级）** | 删除 / 指向自建 | 1 行 |
| `usage/usage_net.dart:38` | `UsageBuild.active = kReleaseMode \|\| …` | 是（release 恒开） | 改成 `false` | 1 行 |
| `usage/usage_stats.dart:98` | `if (UsageBuild.active) _watch();` → 冷启动 `foreground()` | 是 | 随上一行即失效 | 0 |
| `usage/usage_stats.dart:214-224` | `foreground()`：计数 + `send()` | 是 | 同上 | 0 |
| `usage/usage_stats.dart:241` | `status = await http.post(endpoint, …)` —— 真正的上报口 | 是 | 同上 | 0 |
| `usage/usage_stats.dart:106` | `final enabled = prefs.getBool(enabledKey) ?? true;`（默认开） | 是 | 改 `?? false` | 1 行 |
| `ui/onboarding/usage_notice_row.dart:35` | 首次引导里 `markNoticed()` → 打开后立刻开始计数发送 | 是 | 隐藏该行 | 3 行 |
| `ui/settings/usage_stats_section.dart:25` | 设置 › 隐私与分享 的开关 | 否 | 保留（可看可关）或隐藏 | 3 行 |
| `ui/shell/usage_upgrade_notice.dart:131` | 用满阈值后的「升级」提示 | 否 | 保留 | 0 |
| `ui/settings/diagnostics_page.dart` | 诊断页 | 否 | **保留**（纯本地导出，无 http/上报，已核） | 0 |

### 1.4 更新检查

| 文件:行号 | 入口 | 阻断主线 | 建议 | 成本 |
|---|---|---|---|---|
| `update/update_config.dart:48` | `updateDownloadOrigin = 'https://dl.dshtui.com'`（更新文件副本，**元数据第一个问它**） | 是（每次自动检查的第一跳） | 删 → 只留 GitHub | 1 行 |
| `update/update_config.dart:43` | `updateMirrorOrigin = 'https://go.dshtui.com'`（beta 频道） | 是（beta 构建恒先问） | 删 | 1 行 |
| `update/update_config.dart:24,28` | `updateRepo = 'ccch1mneyyy/dsh-go'` + `api.github.com/.../releases/latest` | 第三方（GitHub），非作者服务器 | 保留，或改自己的 repo | 1 行 |
| `update/update_config.dart:236-267` | `UpdateSources`：channel → dl.dshtui.com → GitHub 加速镜像 → GitHub API | 是 | 只留 GitHub | 约 10 行 |
| `state/update_state.dart:85` | `this.autoCheck = true`（默认自动检查） | 是 | 默认 false，或删自动段 | 1 行 |
| `state/update_state.dart:381` | `autoCheckIfDue()` | 是 | 随上一行 | 0 |
| `ui/update/update_launch_check.dart:46` | `unawaited(... autoCheckIfDue())`，**启动 6 秒后触发** | 是 | 不挂载即可 | 0-2 行 |
| `ui/shell/home_shell.dart:342` | `const UpdateLaunchCheck(),` | 是 | 删这一行 | 1 行 |
| `ui/update/update_tiles.dart:31-44` | 设置 › 关于 里的「检查更新 / 自动检查」 | 否 | 保留或隐藏 | 0-5 行 |

### 1.5 插件市场 / 论坛 / 云分享 / Deep Link

| 文件:行号 | 入口 | 阻断主线 | 建议 | 成本 |
|---|---|---|---|---|
| `state/plugins_state.dart:226` | `officialIndexUrl = 'https://go.dshtui.com/market/index.json'` | 否（打开市场才请求，6 小时间隔） | 指向自建 / 隐藏市场页 | 1 行 + 入口 |
| `state/plugins_state.dart:230` | `officialMarketBases = ['https://dl.dshtui.com/market/', 'https://go.dshtui.com/market/']` | 否 | 同上 | 1 行 |
| `forum/forum_config.dart:6` | `forumDefaultBaseUrl = 'https://forum.dshtui.com'` | 否（论坛页才请求） | 隐藏论坛入口 | 1 行 |
| `share_hosted/hosted_config.dart:7` | `shareHostOrigin = 'https://go.dshtui.com'`（云分享链接） | 否（**只在用户点「生成链接」时上传**） | 隐藏云分享，保留分享码/本地导出 | 入口 5 行 |
| `ui/shell/home_shell.dart:354` | `const ForumLinkHost(),` 论坛 deep link 注册 | 否 | 保留（无请求）或删 | 1 行 |
| `links/plugin_links.dart:23,80` | `dshgo://go.dshtui.com/p/<name>` 市场插件链 | 否 | 保留（无请求） | 0 |
| `android/.../AndroidManifest.xml:168` | `<data android:scheme="dshgo" />` | 否 | **保留**：只有自定义 scheme，**没有 https host 的 autoVerify**，不会因别人点链接就替你去连作者站 | 0 |
| `state/app_info.dart:36-48` | 作者 B 站 / X / GitHub、`remoteProjectUrl`、`remoteConsole` | 否 | 关于页外链可保留；`remoteConsole` 删 | 3 行 |

---

## 2. SSH 直连模式下 App 还会向作者服务器发什么（硬证据）

### 2.1 匿名使用统计：会，每次冷启动 + 每次回前台

`app/lib/app.dart:60`（`initState`，与主机无关）：

```dart
// Anonymous usage statistics live as long as the app, in either 界面
// (release builds: today is noted, and earlier days go out).
ref.read(usageStatsProvider);
```

`app/lib/usage/usage_stats.dart:98`：

```dart
_loaded = _load();
if (UsageBuild.active) _watch();   // _watch() 里 unawaited(foreground()) —— 冷启动就在前台
```

`app/lib/usage/usage_stats.dart:241`（真正的出口）：

```dart
status = await http.post(endpoint, jsonEncode(payload), timeout: timeout);
```

`endpoint` = `app/lib/usage/usage_net.dart:21`：

```dart
final defaultUsageEndpoint = Uri.parse('https://go.dshtui.com/api/usage/v1');
```

开关默认是开：`usage_stats.dart:106` `prefs.getBool(enabledKey) ?? true`，而首次引导 `ui/onboarding/usage_notice_row.dart:35` 一挂载就 `markNoticed()`（= 开始计数并发送）。报文含 app 版本、build 号、SDK、ABI、厂商、语言、每天的使用次数统计、安装 ID。

**→ 与 SSH/tailnet 完全无关，走任何通道都发。**

### 2.2 更新检查：会，启动 6 秒后每天一次

`app/lib/ui/shell/home_shell.dart:342`（首页 shell 常挂）→ `ui/update/update_launch_check.dart:46`：

```dart
final offer = await ref.read(appUpdateProvider.notifier).autoCheckIfDue();
```

`state/update_state.dart:381-385` 受 `state.autoCheck`（`:85` `this.autoCheck = true`）与「每天一次」控制。元数据第一跳是 `update/update_config.dart:48` 的 `https://dl.dshtui.com/update.json`，失败才轮 GitHub 镜像/API。

**→ 默认开、与通道无关。作者 R2 桶 + Worker 都会被问。**

### 2.3 relay 登录方式探测：会，每次冷启动

链条（全部在 fork 上核对过行号）：

```
ui/shell/home_shell.dart:306            ref.listen<void>(petOverlayBridgeProvider, (_, _) {});
  → pets/pet_overlay_bridge.dart:24-37  petOverlayBridgeProvider 里 ref.listen(petStateProvider, …, fireImmediately: true)
  → pets/pet_signals.dart:565           petScopedStateProvider: hosts: ref.watch(petHostChoicesProvider)
  → pets/pet_overlay_actions.dart:47    for (final h in ref.watch(relayAccountProvider).value?.hosts ?? …)
  → state/hosts_state.dart:490-503      RelayAccountController.build()
```

`state/hosts_state.dart:498-503`：

```dart
user = await backend.currentUser();          // 无本地凭证 → null，不发请求
if (user == null) return base.copyWith(providers: await _loginProviders(backend));   // ← 这里发请求
```

`packages/dsh_remote/lib/src/account/remote_account.dart:376-379`：

```dart
Future<List<OAuthProvider>> loginProviders() => _guard(() async {
  final providers = await api.oauthProviders();
```

`packages/dsh_remote/lib/src/server/server_api.dart:299-301`：

```dart
Future<List<OAuthProvider>> oauthProviders() async {
  Object? body;
  try {
    body = await _request('GET', '/api/v1/auth/oauth/providers');
```

origin = `state/hosts_state.dart:382` `const defaultRelayServer = 'https://dsh.r2049.cn';`（除非用户改过自建 server）。

**→ 结论：只要没删宠物→relay 这条 watch 链、也没登录过，App 每次冷启动都会 `GET https://dsh.r2049.cn/api/v1/auth/oauth/providers`。这条与「用 SSH 还是 relay」毫无关系。**

### 2.4 反证：SSH 直连本身不发外网

- `remote/ssh/ssh_transport.dart` / `ssh_link.dart` / `local_forward.dart` 里没有作者域名（全库 `r2049` 命中只落在 relay 后端、`hosts_state`/`app_info` 常量、`remote_screen` 与 l10n 文案里，**没有一处出现在 `remote/ssh/**`**）；SSH 只连用户填的 host。
- `state/read_list_identity.dart:144-182` 是 `switch (selected.kind)`，**SSH/本机分支不 watch `relayAccountProvider`**，所以会话列表、模型、插件列表这些读操作也不会因 SSH 而建 relay 账号态。relay 那条只由宠物链（§2.3）和远程页/切主机弹窗带起来。
- `state/connection_state.dart:200-250` 的 `hostLinkProvider` 也按 kind 分支，SSH 分支不碰 `relayBackendProvider`。
- 诊断页 `ui/settings/diagnostics_page.dart` 纯本地，无 http。
- Deep link 只有自定义 scheme `dshgo`（`AndroidManifest.xml:168`），无 https autoVerify → 别人发链接不会自动触发作者站请求。

**净结论：SSH 直连模式自己的流量是干净的；脏的是「遥测 + 更新检查 + relay 账号探测」这三块全局件。**

---

## 3. host 表单现状（问题 4 的答案）

`app/lib/ui/remote/ssh_host_form.dart`（788 行，**a33c3da 没有改它**，`git show --stat a33c3da` 的文件清单里只有 `app/lib/remote/tailnet/*`、`app/lib/state/connection_state.dart` 和原生侧）：

- 全靠手填，8 个控制器，`ssh_host_form.dart:43-51`：

```dart
late final _name    = TextEditingController(text: widget.existing?.name ?? '');
late final _host    = TextEditingController(text: widget.existing?.host ?? '');
late final _port    = TextEditingController(text: '${widget.existing?.port ?? 22}');
late final _user    = TextEditingController(text: widget.existing?.user ?? '');
late final _command = TextEditingController(text: widget.existing?.launchCommand ?? defaultDshLaunchCommand);
late final _cwd     = TextEditingController(text: widget.existing?.workingDirectory ?? '');
final _password     = TextEditingController();
final _privateKey   = TextEditingController();
final _passphrase   = TextEditingController();
```

- 分四段：基本信息（`:401`）、认证（`:478`，密码 / 私钥二选一）、启动命令 + 工作目录（`:583`、`:588`）、host key 指纹（`:602`）。
- 地址提示只写了 `192.168.1.20 或 example.com`（`l10n/gen/app_localizations_zh.dart:13035`），**没提 tailnet 的 100.x**；但 `remote/ssh/ssh_host_config.dart` 对 host 不做网段校验（`:100-102` 只检查非空），所以填 `100.x.y.z` 能存。

**「内嵌 tailnet」开关：没有，一行 UI 都没有。** 全库 grep `tailnet|Tailnet` 命中只有 3 个文件（`state/connection_state.dart:12,13,160-176` + `remote/tailnet/tailnet_bridge.dart` + `remote/tailnet/tailnet_transport.dart`），全是纯逻辑：

`app/lib/state/connection_state.dart:173-177`（fork）：

```dart
final sshTransportFactoryProvider = Provider<SshTransportFactory>((ref) {
  final tailnet = ref.read(tailnetBridgeProvider);
  return (params, verifyHostKey) => tailnetAwareTransport(tailnet, params, verifyHostKey);
});
```

`app/lib/remote/tailnet/tailnet_transport.dart:24-40`：先直连，**只有直连抛「可重试」错误**才 `bridge.ensureEndpoint(targetHost: params.host, …)`，再连 `127.0.0.1:<port>`。也就是：

1. 用户仍要自己知道并手填 tailnet 地址；写错网段 → 直连必然先失败一次（等超时）→ 才回落 tailnet，白等一轮。
2. **tailnet 状态、节点 IP、登录 URL 全都没有 UI**：`remote/tailnet/tailnet_bridge.dart:29-56` 已经把 `authUrl`、`tailnetIps`、`listenPort`、`connCount` 解析出来了，但除 `connection_state.dart` 外无人引用。
3. Kotlin 侧 `TailnetBridgeManager.kt` 不传 auth key，Go 侧 `native/tailnet-go/bridge.go:110-114` 只给 `Hostname/Dir/Logf`，走 **tsnet 默认控制面（Tailscale 官方 login）**；`bridge.go:139` 注释写「No auth key is ever stored in the APK」并靠 `watchAuth()` 拿交互登录 URL。**目前没人把这个 authUrl 显示出来 → 首次用 tailnet 是死路**（除非那台机器之前在该 state 目录登录过）。
4. 没有 `ControlURL` 可配 → 想换自建 headscale 也没有缝。

---

## 4. 下一步动手清单（按优先级）

1. **掐遥测**：`usage/usage_net.dart:38` 把 `UsageBuild.active` 写死 false（或删 `:21` 的 endpoint 常量），`usage/usage_stats.dart:106` 默认改 false；顺手摘掉 `ui/onboarding/usage_notice_row.dart:35` 的 `markNoticed()`。→ 涉及 `usage/usage_net.dart`、`usage/usage_stats.dart`、`ui/onboarding/usage_notice_row.dart`。
2. **断开启动期 relay 探测**：把 `pets/pet_overlay_actions.dart:47` 那个 relay for 循环去掉（宠物菜单只列本机 + SSH），并让 `RelayAccountController.build()` 在「未登录」时不要调 `_loginProviders`（`state/hosts_state.dart:503` 改成立即返回空 providers，等用户真的打开远程页再拉）。→ 涉及 `pets/pet_overlay_actions.dart`、`state/hosts_state.dart`。
3. **隐藏远程页的 relay 与作者控制台两段**：删 `ui/remote/remote_screen.dart:120-130`（+`_openConsole` 与 `:46-48`）和 `:365-470`（`_RelaySection` 及其调用点 `:99` 的 `_RelaySection(onUse: …)`），同时删 `state/app_info.dart:41` 的 `remoteConsole`。→ 涉及 `ui/remote/remote_screen.dart`、`state/app_info.dart`（连带 `relay_login_sheet.dart`、`relay_credit.dart` 变孤儿，可直接删文件）。
4. **更新只留 GitHub 或直接不自动检查**：`ui/shell/home_shell.dart:342` 摘掉 `UpdateLaunchCheck()`；`update/update_config.dart:43,48` 删掉两个 dshtui 常量（`UpdateSources` 里对应分支一起去掉）。→ 涉及 `ui/shell/home_shell.dart`、`update/update_config.dart`。
5. **给 tailnet 补最小 UI**（不然它永远是「静默回落」的黑盒）：host 表单加一个「经内嵌 tailnet 连接」开关（勾上就跳过直连，直接 `ensureEndpoint`）+ 顶部小字显示节点状态/IP；首次 `TailnetPhase.needsLogin` 时把 `authUrl` 用 `urlOpenerProvider` 弹出来。要自建就再给 Go 侧 `tsnet.Server` 加 `ControlURL` 字段并在 Kotlin/Dart 透传。→ 涉及 `ui/remote/ssh_host_form.dart`、`remote/tailnet/tailnet_transport.dart`、`remote/tailnet/tailnet_bridge.dart`、`android/.../tailnet/TailnetBridgeManager.kt`、`native/tailnet-go/bridge.go`。

> 市场（`go.dshtui.com/market`）、论坛（`forum.dshtui.com`）、云分享（`go.dshtui.com/c/…`）都不是启动期请求，属于「点开才连」，优先级低于上面五条；要彻底干净再按 §1.5 改常量或藏入口。
