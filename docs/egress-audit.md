# dsh-go fork 全量网络出口审计（egress audit）

- **审计对象**：主人 Windows PC 上的 fork `D:\dsh-ai\dshgo-fork`（用户 `TosakaRin`）
- **基线**：`HEAD = a33c3da198e813849c63baf9ad2f06c64d200343`（2026-11-05 23:49:45 +0900，`feat(tailnet): embedded userspace Tailnet for SSH remote access`）
- **工作树改动（一并审计）**：`git status --porcelain` 3 条 —— `M app/lib/remote/ssh/ssh_transport.dart`、`?? app/android/app/libs/`、`?? app/lib/remote/ssh/ssh_transport.dart.bak`。
  已逐行看 diff：只加了 `connectTimeoutFor()` 的短超时逻辑（私网 3s / 公网 12s），**不新增任何网络出口**。
- **上游**：`ccch1mneyyy/dsh-go`（AGPL-3.0）。fork 的 origin 是 `https://github.com/TohsakaRin233/dsh-Go.git`。
- **审计方式**：PC 端 `findstr` 全量扫 + sftp 只读取关键文件逐行读 + **对预编译 AAR 里的 Go 二进制抽字符串取证**。除 sftp 读以外没有改动 PC 任何文件。
- **审计日期**：2026-11-06（JST）
- **回答的原话**：「我希望整体网络交互完全不经过作者那边，麻烦深度检查下有没有类似的后门或者问题」

> 结论纪律：本文每条都带 `文件:行号`。找不到证据的地方写「未发现」，不写「可能存在」。

---

## §0 结论速览

### ① 除了已知的三处，还有没有别的外部出口？

**有，而且不止一处。** 作者域名的出口在代码里一共 **7 类**（原有 3 类 + 新增 4 类），另有 5 类「Tailscale/第三方」出口。没有发现后门级代码（没有隐蔽回传、没有把对话内容或密钥发出去的路径），但「完全不经过作者」按现状**做不到**，必须改。

**新增（此前三处之外）的作者域名出口：**

| # | 出口 | 触发 | 位置 |
|---|---|---|---|
| A | 插件市场索引 `go.dshtui.com/market/index.json`、`dl.dshtui.com/market/` | 打开插件市场 / 插件页首次读取（会话内缓存，6h 新鲜期） | `app/lib/state/plugins_state.dart:226`、`:230`、`:322` |
| B | 托管分享上传 `POST https://go.dshtui.com/api/v1/c` | **用户主动**点「托管分享链接」才发（内容本地密封后上传） | `app/lib/share_hosted/hosted_client.dart:71` |
| C | 论坛 `forum.dshtui.com` | 打开论坛 / 登录（可在「论坛设置」改地址） | `app/lib/forum/forum_config.dart:6` |
| D | 社区 hub feed `raw.githubusercontent.com/ccch1mneyyy/dsh-go/main/community.json`（+ jsDelivr 两个 CDN 镜像） | 打开社区页，6h 刷新一次 | `app/lib/state/community_state.dart:41-43` |

**已被强调的三处（这里只引用、不再重复论证，但补两条新事实）：**

| # | 出口 | 补充事实 | 位置 |
|---|---|---|---|
| E | 匿名遥测 `go.dshtui.com/api/usage/v1` | 默认开；**必须在首启看过那个开关（`noticed`）之后才真正开始发**；body 固定 11 个键，不含机型/对话内容 | `app/lib/usage/usage_net.dart:21`、`app/lib/usage/usage_stats.dart:241`、`app/lib/usage/usage_ledger.dart:29` |
| F | 自动更新 `dl.dshtui.com` / `go.dshtui.com` | **默认每天自检一次、启动时触发**；而且每次检查的第一个请求就是 `dl.dshtui.com/update.json`（作者 R2），不是 GitHub | `app/lib/update/update_config.dart:43/48`、`app/lib/state/update_state.dart:329`（`autoCheck` 默认 true）、`:381`、`app/lib/ui/update/update_launch_check.dart:46` |
| G | relay 账号探测 `dsh.r2049.cn/api/v1/auth/oauth/providers` | 默认 relay 服务器 = `https://dsh.r2049.cn`；**TURN 凭据也来自它**（`GET /api/v1/turn/credentials`），所以走 relay 时中继流量落在作者的 TURN 上 | `packages/dsh_remote/lib/src/server/server_api.dart:13`、`:299`、`:521`、`app/lib/state/hosts_state.dart:382` |

### ② 哪些是「必须」的（不算后门）

1. **Tailscale 控制面**：`https://controlplane.tailscale.com`（默认），外加 `login.tailscale.com`（登录页）与 `derp*.tailscale.com`（中继）——不连它就没有 tailnet。
2. **Tailscale 的 DNS 兜底**：系统 DNS 解析不了控制面/DERP 时才走的 DoH 列表（quad9 / cloudflare / mullvad / controld / nextdns）。
3. **`log.tailscale.com`**：**这个是新发现，且不该被忽略**——tsnet 默认**一直**把节点日志上传到 Tailscale 的日志服务（详见 §2-⑧）。
4. 用户自己配的模型/语音/媒体服务端点（DeepSeek、火山、DashScope、SiliconFlow、Gemini、ModelScope/HF、npm registry）——用户自己的 key，与本议题无关。

### ③ 哪些能关、怎么关（不写代码就能关的）

| 能关 | 怎么关 |
|---|---|
| 匿名遥测 | App 内：设置 › 隐私与分享 → 关闭使用统计（`usage_stats_enabled`）；关掉后「不计数、不缓存、不发请求」 |
| 自动更新检查 | App 内：设置 › 关于 › 自动检查更新 → 关（`update_state.dart:528`） |
| 论坛 / 社区 / 插件市场 / 托管分享 | 不用就不出网（都是页面打开才拉）；托管分享只在点「托管」时上传 |
| relay（含 TURN） | 远程接入改用自己的 SSH 主机（默认路径本来就不碰 relay）；relay 页可把服务器改成自建（`hosts_state.dart:334-365`） |
| Tailscale 控制面/DERP | 只能自建 Headscale + 自建 DERP（`tsnet.Server.ControlURL` 未被设置，见 §3） |

**写代码/重编才能关的**：更新源域名（`--dart-define=DSH_UPDATE_MIRROR=`）、beta 通道、社区 feed（`--dart-define=DSH_COMMUNITY_FEED`）、遥测端点（`--dart-define=DSH_USAGE_ENDPOINT`）、插件市场索引（硬编码，只能改代码）。全是编译期常量，改不了运行时。

### ④ 风险分级

- **🟥 真问题（建议必改，但不是「后门」）**
  1. **`log.tailscale.com` 默认上传节点日志**（tsnet 默认行为，主人多半不知道）。
  2. **relay 密码登录把邮箱+密码明文 HTTPS 发到 `dsh.r2049.cn`**（`server_api.dart:396`）——用了托管 relay 账号就等于把凭据交给作者的服务器。
  3. **自动更新每天自检、首个请求打作者 R2**（即使你只用 GitHub 也会先问作者那边）。
- **🟧 建议关（隐私口径）**：遥测、社区 feed、插件市场索引（`go.dshtui.com` / `dl.dshtui.com` / `api.dshfind.com`）。
- **🟩 可留**：
  - Tailscale 官方控制面/DERP（不连就没网，属于「必须」）；
  - 作者个人链接（`b23.tv` / `x.com`，只在你手指点了才开）；
  - `ccch1mneyyy.github.io`（旧分享落地页，只出现在「识别」列表里，不主动访问）；
  - 第三方 CDN（mermaid、语音模型、npm registry）。

**一句话回答主人**：**没有后门，但有「默认悄悄出网」的地方，且明确不止三处——共 7 类打作者域名，其中自动更新每天一次、遥测默认开、市场索引和社区 feed 一开页面就拉。彻底「不经过作者」= 改 4 处编译期常量 + 关 2 个开关 + 自建 Headscale/relay，代价见 §3。**

---

## §1 出口总表

> 「能否关闭」列里：`开关` = App 内可关；`构建` = 只能改 `--dart-define` 重编；`代码` = 要改源码；`无` = 关不掉（协议必需）。

### 1.1 作者域名（ccch1mneyyy / dshtui / r2049）

| 域名/IP:端口 | 协议 | 触发条件 | 代码位置 (file:line) | 发送了什么 | 能否关闭（怎么关） | 风险 |
|---|---|---|---|---|---|---|
| `go.dshtui.com:443` `/api/usage/v1` | HTTPS POST | 默认开；首启看过开关后，每天关账次日发一次 | `app/lib/usage/usage_net.dart:21`、`app/lib/usage/usage_stats.dart:241`、`app/lib/usage/usage_ledger.dart:29` | `v,iid,day,app,build,sdk,abi,maker,lang,engine,counts`（装机随机 id、版本、SDK、ABI、厂商小写、每日计数） | 开关：设置›隐私与分享 | 建议关 |
| `dl.dshtui.com:443` `/update.json`、`/download/<ver>/<file>` | HTTPS GET | **启动时自检（默认每天一次）**，下载更新包时 | `app/lib/update/update_config.dart:48`、`:148`（`_mirrorDefine` 默认值）、`app/lib/state/update_state.dart:381`、`app/lib/ui/update/update_launch_check.dart:46` | 无凭据的 GET；UA=`dsh-go/<ver>`；带版本号 | 开关（自动检查）+ 构建期 `DSH_UPDATE_MIRROR=` | 建议关（首个请求打作者 R2） |
| `go.dshtui.com:443` `/beta/update.json` | HTTPS GET + `Authorization: Bearer <token>` | 仅 beta 包（编译进了 channel token） | `app/lib/update/update_config.dart:51-83`、`:174` | 版本号 + 编译进包的 beta token | 构建期（`DSH_BETA_CHANNEL_TOKEN` 空则不启用） | 可留（正式包不带） |
| `go.dshtui.com:443` `/market/index.json`、`dl.dshtui.com:443` `/market/` | HTTPS GET | 打开插件市场/插件页（会话内缓存，6h 新鲜期） | `app/lib/state/plugins_state.dart:226`、`:230`、`:322`、`:353` | 无凭据的 GET | 代码（硬编码常量） | 建议关 |
| `go.dshtui.com:443` `POST /api/v1/c?ttl=…` | HTTPS POST | **用户主动**点「托管分享链接」 | `app/lib/share_hosted/hosted_client.dart:71`、`app/lib/share_hosted/hosted_config.dart:7` | 本地 AEAD 密封后的密文 + TTL；**解密 key 只在 URL `#k=` 片段里，不发服务器**（`hosted_link.dart:68`） | 不用该功能即无 | 可留（内容端到端，服务端只见密文+大小+时间+IP） |
| `forum.dshtui.com:443` | HTTPS | 打开论坛 / 论坛登录 | `app/lib/forum/forum_config.dart:6` | 论坛请求 +（登录后）Discourse User API Key | App 内「论坛设置」可改地址 | 可留/建议关 |
| `dsh.r2049.cn:443` | HTTPS + WSS | 远程 relay 模式（默认 relay 服务器） | `packages/dsh_remote/lib/src/server/server_api.dart:13`、`:299`、`:442-527`、`app/lib/state/hosts_state.dart:382`、`app/lib/remote/relay/relay_backend.dart:182` | 设备注册（`deviceId/name/role/platform/identityKey/clientVersion`）、在线状态、设备列表、**账号密码登录**、TURN 凭据 | 改自建服务器（`hosts_state.dart:334-365`）或只用 SSH 直连 | 🟥 见 §2-⑥ |
| `dsh.r2049.cn:443` TURN（`turn:`/`stuns:`） | UDP/TCP/TLS | relay 连接被判定需要中继时 | `packages/dsh_remote/lib/src/transport/peer_connection.dart:15-28`、`server_api.dart:521-527` | WebRTC 中继的加密 UDP 流（Noise 之上） | 同上 | 可留（内容 E2E） |
| `dsh.r2049.cn:443` `/app` | HTTPS（浏览器） | 点「远程 Web 控制台」 | `app/lib/state/app_info.dart:41` | 只是打开外部页面 | 不点即无 | 可留 |
| `api.dshfind.com:443` `/market/dsh-index.json` | HTTPS GET | 打开市场（第三方索引，非作者域名但默认拉） | `app/lib/state/plugins_state.dart:35`、`:216` | 无凭据的 GET | 代码 | 建议关 |
| `raw.githubusercontent.com` / `cdn.jsdelivr.net` / `fastly.jsdelivr.net` `ccch1mneyyy/dsh-go/main/community.json` | HTTPS GET | 打开社区 hub，6h 刷新 | `app/lib/state/community_state.dart:41-43` | 无凭据的 GET | 构建期 `DSH_COMMUNITY_FEED`（只能替换首个源） | 建议关 |
| `ccch1mneyyy.github.io` `/dsh-go/share/` | 仅字符串匹配 | **不主动访问**，只用于识别旧分享链接 | `app/lib/share_code/share_link.dart:22` | 无 | — | 可留 |
| `b23.tv` / `x.com` | 浏览器打开 | 用户点作者主页链接 | `app/lib/state/app_info.dart:35`、`:37` | 无 | 不点即无 | 可留 |
| `ghfast.top` / `gh-proxy.com` | HTTPS GET | 更新包下载失败后的公共加速器 | `app/lib/update/update_config.dart:143-144` | 更新包请求（**下完核 SHA-256 + 签名**） | 构建期 `DSH_UPDATE_MIRRORS=` | 可留 |

### 1.2 Tailscale（新增内嵌 tailnet，Go + AAR）

| 域名/IP:端口 | 协议 | 触发条件 | 代码位置 (file:line) | 发送了什么 | 能否关闭 | 风险 |
|---|---|---|---|---|---|---|
| `controlplane.tailscale.com:443` | HTTPS | tailnet 起来时（连接/心跳/map 更新） | `native/tailnet-go/bridge.go:110-114`（`tsnet.Server` 未设 `ControlURL`）、上游 `tsnet.go:779-783`、AAR 二进制字符串 | 节点公钥、hostname（`dshgo-phone`）、netmap 请求；不含 SSH 内容 | 只能自建 Headscale（§3） | 🟧 必须 |
| `login.tailscale.com:443` | HTTPS（浏览器） | 首次交互登录，用户在浏览器批准 | `bridge.go:134-141`（`srv.LocalClient().Status().AuthURL`）、AAR 字符串 | 登录会话 | 必须（或用 auth key / Headscale） | 可留 |
| `derpN.tailscale.com:443` / `:80` | HTTPS/HTTP2 或 UDP:3478 | 打洞失败时中继 WireGuard 流量 | AAR 二进制里 25 个 `derp*.tailscale.com` 域名 | 加密的 WireGuard 包（节点私钥在手机） | 只能自建 DERP | 可留（E2E） |
| **`log.tailscale.com:443`** | HTTPS | **tailnet 一启动就持续上传** | 上游 `tsnet.go:1100-1136`（`startLogger` 无条件建 logtail）+ `logtail/config.go:16-18`（`DefaultHost = "log.tailscale.com"`）、AAR 字符串 `tailnode.log.tailscale.io` | 节点日志：hostname、tailnet IP、DERP 状态、错误、计数器 | 目前**无开关**（见 §3-③） | 🟥 建议处理 |
| DoH 兜底：`dns.quad9.net`/`dns10`/`dns11`、`cloudflare-dns.com`、`dns.mullvad.net`、`freedns.controld.com`、`dns.nextdns.io` | HTTPS POST (DoH) | 系统 DNS 解析不了控制面/DERP 时 | `net/dnsfallback` 逻辑 + AAR 二进制字符串 | 要解析的域名（`controlplane.tailscale.com` / `derpN`） | 关不掉（兜底逻辑） | 可留 |

### 1.3 本地与第三方（和作者无关，但属于出口，一并列清）

| 目标 | 协议 | 触发 | 位置 | 说明 | 风险 |
|---|---|---|---|---|---|
| `127.0.0.1:<随机>` / `localhost` | HTTP/SSE/WS | 本地 proot DSH 引擎、SSH 本地转发 | `app/android/.../runtime/DshProcess.kt:343`、`app/lib/remote/ssh/local_forward.dart` | 全程 loopback；清文只对 127.0.0.1 放行（`app/android/app/src/main/res/xml/network_security_config.xml`） | 可留 |
| 用户配置的 SSH 主机 | TCP 22（可改） | 连远程主机 | `app/lib/remote/ssh/ssh_transport.dart` | 宿主是主人自己填的，不含作者域名（已复核） | 可留 |
| `api.deepseek.com` / `platform.deepseek.com` | HTTPS | 查余额 / 充值页 | `app/lib/state/balance_state.dart:110-111` | 主人自己的 key | 可留 |
| `ark.cn-beijing.volces.com`、`dashscope.aliyuncs.com`、`api.siliconflow.cn`、`generativelanguage.googleapis.com` | HTTPS | 生成图片/视频（媒体服务默认端点） | `app/lib/state/media_catalog.dart:419-496` | 主人自己的 key | 可留 |
| `www.modelscope.cn`、`huggingface.co`、`hf-mirror.com` | HTTPS | 首次下载语音模型 | `app/lib/voice/voice_model_assets.dart:48-56` | 模型文件，用户可拒绝 | 可留 |
| `registry.npmjs.org`、`registry.npmmirror.com` | HTTPS | 分享码/插件 npm 解析 | `app/lib/share_code/share_net.dart:57`、`app/android/.../runtime/GuestSetup.kt:32` | 只解析元数据 | 可留 |
| `registry.npmmirror.com/mermaid/…`、`cdn.jsdelivr.net/npm/mermaid@…`、`unpkg.com/mermaid@…` | HTTPS | 聊天里渲染 mermaid 图 | `app/lib/ui/chat/render/mermaid_preview.dart:24-26` | 拉一个 11.4.1 的 js | 可留 |
| `gist.githubusercontent.com` | HTTPS | 导入 gist 形式的分享码 | `app/lib/share_code/share_net.dart:133` | 用户给的 URL | 可留 |
| `shizuku.rikka.app` | 浏览器 | 点「下载 Shizuku」 | `app/lib/ui/phone/dnd_mode_page.dart:24` | 不点即无 | 可留 |
| `www.youtube.com`、`vimeo.com` | iframe | 论坛帖里嵌的视频 | `app/lib/forum/render/cooked.dart:430-431` | 用户看帖时 | 可留 |

**硬编码 IP 复核**：`app/lib` + `packages/*/lib` 全量扫 `x.x.x.x` 形式，除 `127.0.0.1`/`localhost`/注释里的 `100.64.0.2`（CGNAT 示例）/`10.0.2.2`（模拟器）外，**未发现任何硬编码公网 IP**。

---

## §2 可疑点逐条分析

### ① 遥测默认开、还带装机 id —— 不是后门，是设计
- **现象**：`usage_stats.dart` 首启即 `enabled=true`，每天关账次日 POST 一次。
- **证据**：`usage_stats.dart:25`（`enabled = true` 默认）、`:36`（`collecting = loaded && enabled && noticed`）；`usage_ledger.dart:29`（body 键全集）；`usage_net.dart:52-58`（一请求一 client，不存 cookie，10s 超时，不跟随重定向）。
- **最坏情况**：作者能统计「有多少台机、什么厂商、哪天用了多少轮对话」——**没有内容，没有主机名，没有模型名**。
- **判定**：**是设计，不是后门**；但违背「完全不经过作者」，建议关。

### ② 更新自检每天一次、第一个请求打作者 R2 —— 是设计，但比宣传的更「主动」
- **现象**：即使你从 GitHub 装包，检查更新的**第一个**请求也是 `dl.dshtui.com/update.json`。
- **证据**：`update_config.dart:5-10`（库注释自己写明「Every build asks our own copy of the release files first」）、`:148`（`_mirrorDefine` 默认 `updateDownloadOrigin`）、`:183`（feed = `<origin>/updateManifestName`，`:315` 定义文件名 `update.json`）、`update_state.dart:329`（`autoCheck` 默认 `true`）、`update_state.dart:381-409`、`update_launch_check.dart:46`。
- **最坏情况**：作者能记录「哪台设备的 IP + UA(`dsh-go/<ver>`) 每天来问一次」，形成活跃度面板。
- **判定**：**是设计但建议关**（设置里一个开关即可，代价是失去自动更新）。

### ③ 插件市场索引硬编码 —— 无关闭入口
- **现象**：进入市场页就拉 `go.dshtui.com/market/index.json`，失败回退 `dl.dshtui.com/market/`。
- **证据**：`plugins_state.dart:226`、`:230`、`:322-341`、`:353-361`。
- **最坏情况**：作者知道谁在看市场、装了什么（安装动作本身也会再拉一次插件包，`dl.dshtui.com`）。
- **判定**：**是设计但建议关**；只能改代码或让它拿不到（见 §3-①）。

### ④ 社区 hub 走作者 GitHub 仓 —— 标注清楚
- **现象**：`community.json` 从 `raw.githubusercontent.com/ccch1mneyyy/dsh-go/main/` 拉，另有 jsDelivr 两个镜像。
- **证据**：`community_state.dart:39-44`、`:28`（6h 刷新）。
- **最坏情况**：GitHub 侧能关联「某 IP 定期取这个仓的 community.json」。
- **判定**：**是设计**（内容本身是作者维护的公开脚本/推荐），但与「不经过作者」冲突；建议关。

### ⑤ 托管分享：内容会进作者服务器，但是端到端加密
- **现象**：`POST /api/v1/c?ttl=` 上传二进制。
- **证据**：`hosted_client.dart:65-83`（`:67` `sealHosted(file, key)` 本地密封后再传）、`hosted_link.dart:68`（链接 = `$origin/c/$id#k=$key`，key 在 `#` 后，**不会出现在 HTTP 请求里**）、`hosted_config.dart:6`（注释：the key never reaches the server）。
- **最坏情况**：服务器拿到密文 + 大小 + 时间 + IP；如果 `hosted_crypto` 的算法被换掉，加密就形同虚设（本次未逐行审 `hosted_crypto.dart` 的 AEAD 实现，见 §5 未决）。
- **判定**：**是设计，可留**（前提：只在你主动点「托管」时才发生）。

### ⑥ 🟥 relay 账号：邮箱+密码直接发作者服务器
- **现象**：`RemoteServerApi.loginWithPassword` 把 `{email, password}` POST 到 `dsh.r2049.cn/api/v1/auth/login`。
- **证据**：`server_api.dart:396-400`；默认服务器 `server_api.dart:13`；`hosts_state.dart:382`。
- **最坏情况**：如果你用托管 relay 账号登录，**凭据由作者服务器持有**；再叠加 relay 服务器能看到：设备名、设备公钥、在线状态、连接元数据、TURN 凭据（`server_api.dart:423-441`、`:521-527`）。
- **缓解事实**：relay 的**内容**是 Noise IK 端到端加密，且**pinned 到主机的静态身份公钥**（`secure_channel.dart:41`「the Host's static key must equal the pinned hostIdentityKey」）；TLS 走系统默认校验，**没有** `badCertificateCallback`/pin 之类放宽（全仓库 grep 无命中）——也就是说 TLS 层无自定义信任锚，反而没有「偷偷接受坏证书」的口子。
- **判定**：**是设计（第三方 relay 的必然），但建议关**——只走 SSH 直连 / 自建 relay 服务器；不要把常用密码填进去。

### ⑦ relay/传输层有没有「偷偷回传」？—— 未发现
- **现象**：通读 `packages/dsh_remote/lib`：所有 HTTP 出口都收敛到 `RemoteServerApi._request`（`server_api.dart:553-604`），路径只有 `/health`、`/api/v1/auth/*`、`/api/v1/devices*`、`/api/v1/turn/credentials`、`/api/v1/c`（托管分享在 app 层）。
- **证据**：`server_api.dart:290-527` 全量 endpoint 清单；`dsh_remote` 里没有「事件上报 / 埋点 / metrics 上传」函数；WebRTC/TURN 只服务连接本身（`peer_connection.dart`、`relay_transport.dart`）。
- **判定**：**不是后门**。

### ⑧ 🟥 Tailscale 默认把节点日志传到 `log.tailscale.com`
- **现象**：内嵌 tailnet 一启动，tsnet 就建了一个 logtail 上传器；`bridge.go` 里把 `Logf` 设成空函数**只是让本地不打印，并没有关掉上传**。
- **证据**：
  - `bridge.go:110-114`：`&tsnet.Server{Hostname, Dir, Logf: func(...){}}` —— **没有** `Logtail` 相关字段，也没有 `ControlURL`。
  - 上游 `tsnet.go:1100-1136`：`startLogger()` 在 `start()` 里被无条件调用，`logpolicy.NewConfig(logtail.CollectionNode)` 生成公共 id，`logtail.NewLogger(...)` 开始上传（只有 `testenv.InTest()` 才 `return nil`）。
  - 上游 `logtail/config.go:16-18`：`const DefaultHost = "log.tailscale.com"`；`:25` `CollectionNode = "tailnode.log.tailscale.io"`。
  - **二进制铁证**：`app/android/app/libs/tailnet-arm64.aar` → `jni/arm64-v8a/libgojni.so` 字符串表里有 `tailnode.log.tailscale.io`、`logtail: upload: %v`、`logpolicy: using U…`。
- **最坏情况**：Tailscale 侧能看到「这台手机在哪个 tailnet、节点名、DERP 地区、错误」，**看不到 SSH 内容**（内容在 SSH 之上还有 WireGuard 加密，且日志不含密钥）。
- **判定**：**不是作者的后门，是 tailscale 的默认行为**；但按主人「整体网络交互可控」的口径，属于**应该知情、建议处理**的一条。处理办法见 §3-③。

### ⑨ Android 权限与指纹采集 —— 比预想的干净
- **现象**：权限表里确实有一批「危险权」，但都跟着功能走且多可关。
- **证据**：
  - `READ_PHONE_STATE` 被**显式移除**：`app/android/app/src/main/AndroidManifest.xml:39-41`（`tools:node="remove"`，注释说明是 libadb 的暗示权限，代码不读）。
  - `allowBackup="false"`：`:50`（rootfs 里有明文凭据，禁止备份）。
  - 明文 HTTP 只放行 loopback：`app/android/app/src/main/res/xml/network_security_config.xml`。
  - 无 `OAID`/`IMEI`/`ANDROID_ID` 读取：Kotlin 全量 grep 只命中 `Build.MODEL`（`diagnostics/DiagnosticsPlugin.kt:88`），而且只进本地诊断 bundle。
  - 设备标识走自己生成的 uuid/ulid（`server_api.dart:133-172` 校验规则里就是 uuid/ulid）。
  - deep link 只有自定义 scheme `dshgo`（`AndroidManifest.xml:168`），**没有声明 https App Links 的 autoVerify**。
  - `ShizukuProvider`（`:260`）与无障碍服务（`:248`）都是「手机操控/不打扰」功能，默认关，导出但带权限限制。
- **判定**：**不是后门**。

### ⑩ 日志 / 崩溃 / 诊断导出 —— 不发网
- **现象**：诊断能力齐全，但**只落本地**。
- **证据**：
  - 诊断 bundle 只写本地文件：`runtime/RuntimeController.kt:235-240`（写到 `$DSH_HOME/dsh-app/diagnostics.md`，给引擎里的 agent 读）。
  - 日志脱敏：`app/lib/diagnostics/redact.dart:15-74`（`token=•••`、`Bearer …`、`sk-…`/`AIza…`/`ghp_…`/JWT/私钥块/URL 里密码全打码）。
  - 引擎启动 token 在写入日志前就被正则替换：`runtime/DshProcess.kt:444`（`TOKEN_URL`）+ `:338-343`。
  - `DiagnosticsPlugin.kt:30-33` 注释：`Nothing here reads user content, keys or tokens.`（核对 `:66-100` 的实际字段，确实只有版本/机型/内存/显示/清单）。
- **判定**：**不是后门**；没有「上传日志」的代码路径。

### ⑪ 本地 proot 引擎（Node + dsh 本体）—— 未发现作者出口
- **现象**：`runtime/` 里是打补丁的 DSH 引擎 + rootfs 打包，22 个 patch 全是功能/安全修复，没有网络行为改动。
- **证据**：`runtime/patches/zzz*.diff` 全部为 engine 修复；`runtime/direct/dns.mjs:1-25` 只是把 `dns.lookup` 改成读 guest 的 `resolv.conf`（**用的是系统 DNS，没有硬编码外网解析器**）；`runtime/locks/{dsh,pnpm}/package-lock.json` 里 `resolved` 全指向 `registry.npmmirror.com`（**构建期**装包，运行时不 npm install）。
- **说明**：引擎本体（`@deepseek-ai/dsh` 系列）自己是否发遥测不在本 fork 的代码范围内，需另外按 DSH 本体口径查（见 §5 未决）。
- **判定**：**未发现**作者相关出口。

---

## §3 「完全不经过作者」的可行改造清单（按改动量排序）

### ① 零代码（改配置 / 关开关）—— 半小时内
1. 关遥测：设置 › 隐私与分享 › 使用统计 → 关。
2. 关自动更新检查：设置 › 关于 › 自动检查更新 → 关（代价：手动点检查）。
3. 别填 relay 账号密码；远程主机一律用 SSH 直连（当前默认路径）。
4. 不打开：插件市场、社区 hub、论坛、托管分享。
5. **可选兜底（改 hosts，一行一个）**：把 `go.dshtui.com`、`dl.dshtui.com`、`forum.dshtui.com`、`api.dshfind.com`、`dsh.r2049.cn` 指向 `127.0.0.1`。
   - 需要 root（MuMu 里可行：`adb root` 后改 `/system/etc/hosts` 或 `mount -o rw,remount /`），代价是这几个功能直接不可用，且**更新和遥测会静默失败**（代码对失败是静默的，不会崩）。

### ② 只改构建参数（重编译一次，代码不动）
```bash
flutter build apk --dart-define=DSH_USAGE_ENDPOINT= \
                 --dart-define=DSH_UPDATE_MIRROR= \
                 --dart-define=DSH_UPDATE_FEED= \
                 --dart-define=DSH_UPDATE_MIRRORS= \
                 --dart-define=DSH_COMMUNITY_FEED=
```
- 依据：`usage_net.dart:15-24`（**注意**：`--dart-define=DSH_USAGE_ENDPOINT=` 传空值时 `usageEndpointFrom('')` 返回 null → 仍然回落 `defaultUsageEndpoint`（`:24`），所以遥测**不能靠构建参数关**，只能靠开关或改代码）；`update_config.dart:17-22`（`DSH_UPDATE_MIRROR=` 空字符串 = 不自建镜像）、`:183`（mirror 空 → feed 为 null）。
- **代价**：更新改从 GitHub 官方 + 加速器拿；社区 feed 仍会走 GitHub raw（除非自己也把 `DSH_COMMUNITY_FEED` 指向自建，但那只是**加**一个源，不是替换，`community_state.dart:39-44`）。
- **结论**：这一档能砍掉「更新首站打作者 R2」，但**砍不干净**（社区源、遥测、市场索引都还在）。

### ③ 改代码（fork 里的 patch，改动小但要重编）
| 目标 | 改哪里 | 改动量 | 代价 |
|---|---|---|---|
| 市场索引改自建/删掉 | `plugins_state.dart:226/230/35` 三个常量 | 3 行 | 失去在线市场，只能本地装插件 |
| 社区 feed 去掉 GitHub 源 | `community_state.dart:41-43` | 3 行 | 社区页只显示内置快照 |
| 遥测彻底移除 | `usage_stats.dart` 里 `collecting` 直接 false，或删 `usageProvider` 调用 | 1-5 行 | 无（本来就能关） |
| 论坛默认地址清空 | `forum_config.dart:6` | 1 行 | 论坛需手动填地址 |
| **关掉 tailscale 日志上传** | `bridge.go:110` 的 `tsnet.Server` 加 `Logtail: false`（若 v1.104.5 无此字段，则改为在 `startLogger` 路径上禁：设 `TS_LOGTAIL=0` 无效，需改用 `tsnet.Server.UserLogf` + patch 上游，**实测前不要写死**） | 1-3 行 + 重新 `gomobile bind` | 失去 tsnet 诊断日志（Tailscale 侧排障变难） |
| relay 默认服务器清空 | `server_api.dart:13`、`hosts_state.dart:382` | 2 行 | relay 功能必须手填服务器 |

> ⚠️ 关于 `Logtail`：v1.104.5 的 `tsnet.Server` 结构体里**没有** 直接的 `Logtail bool` 字段（`tsnet.go:311-355` 只有 `LogtailWriter()` 方法），要真正关掉得在 `startLogger` 上打补丁或用 `logtail_omit` 构建标签。**这一条需要在有 Go 工具链的机器上确认后再动手**（PC 上没有 go/NDK，本次无法验证）。

### ④ 只能自建的（协议层面绕不过）
| 项 | 为什么绕不过 | 代价 |
|---|---|---|
| Tailscale 控制面 | `bridge.go:110` 没设 `ControlURL`，默认 `controlplane.tailscale.com`（`tsnet.go:779-783`：字段为空 → 读 `TS_CONTROL_URL` → 再空就用上游默认）。**好消息**：`tsnet.go:311-315` 有 `ControlURL` 字段、`getControlURL` 也认环境变量，所以**支持自建 Headscale** 只需两处之一：Go 加一行 `ControlURL: "https://headscale.自建"`，或 Kotlin 起进程前设 `TS_CONTROL_URL`。 | 自建 Headscale + 证书；登录流程改走自己的控制面 |
| DERP 中继 | 官方 DERP 是控制面下发的地图，自建 Headscale 要配自定义 DERP 地图 | 一台有公网 IP 的机器 + `derper` |
| **`log.tailscale.com`** | 同上，走 tailscale 自己的日志服务；自建 Headscale 不等于关 logtail | 见 §3-③ |
| relay（`dsh.r2049.cn`） | 已支持换服务器（`hosts_state.dart:334-365` + UI 输入框），**这是最省事的一条** | 自建 ds-harness-remote 服务端 |

**总体代价排序**：① 关开关（0 成本）→ ④ relay 换自建（中）→ ② 构建参数（一次重编）→ ③ Patch 常量（一次重编，功能有损）→ ④ Headscale + DERP（最重，但只有自建才算「彻底不经过 Tailscale」）。

---

## §4 怎么现场验证（主人 PC 上可直接抄）

> 下面的命令**都可以只读**。会改系统设置的那几条（防火墙日志）本审计**没有执行**，标注了 `需主人点头`。

### 4.1 谁在连谁：抓 App 的 TCP 连接（只读，最有用）
先找 App 进程号（MuMu 里是模拟器进程；真机是 adb）：
```powershell
# MuMu 模拟器 / 真机 adb 两种都列出来
Get-Process | Where-Object { $_.ProcessName -match 'MuMu|Nemu|dsh|flutter' } |
  Select-Object Id, ProcessName, Path | Format-Table -AutoSize

# 某进程的全部出站连接（把 12345 换成上面的 Id）
Get-NetTCPConnection -OwningProcess 12345 -State Established |
  Select-Object RemoteAddress, RemotePort, CreationTime |
  Sort-Object RemoteAddress | Format-Table -AutoSize
```
轮询 5 分钟，把结果攒起来看「哪些域名/IP 反复出现」：
```powershell
1..60 | ForEach-Object {
  Get-NetTCPConnection -State Established |
    Select-Object @{n='t';e={Get-Date -Format HH:mm:ss}}, RemoteAddress, RemotePort, OwningProcess
  Start-Sleep -Seconds 5
} | Export-Csv -NoTypeInformation "$env:USERPROFILE\Desktop\egress.csv"
```
把 `RemoteAddress` 的 IP 反查域名（只读）：
```powershell
'main' | Out-Null
Resolve-DnsName <上一步抓到的IP> -ErrorAction SilentlyContinue
# 或者查 PTR：nslookup <IP>
```
**要看什么**：`go.dshtui.com` / `dl.dshtui.com` / `dsh.r2049.cn` / `controlplane.tailscale.com` / `log.tailscale.com` / `derp*.tailscale.com` 的 IP 是否出现，以及**出现频率**（更新自检 = 每天一次、启动时；遥测 = 次日一次）。

### 4.2 想看的更细：Windows 防火墙出站日志（`需主人点头`，会改防火墙设置）
```powershell
# 只记录被丢弃的包不够用，这里开「记录成功的连接」
netsh advfirewall set allprofiles logging droppedconnections enable
netsh advfirewall set allprofiles logging allowedconnections enable
# 日志位置（默认）：
notepad "$env:systemroot\system32\LogFiles\Firewall\pfirewall.log"
```
看完记得关掉（日志长得很快）：
```powershell
netsh advfirewall set allprofiles logging droppedconnections disable
netsh advfirewall set allprofiles logging allowedconnections disable
```

### 4.3 真要抓包（MuMu 模拟器）
方案 A：模拟器走主机网卡，直接在 PC 抓
```powershell
# 装 Npcap + Wireshark 后（只读抓包，不改系统）
& "C:\Program Files\Wireshark\dumpcap.exe" -i 1 -f "tcp port 443 or udp port 3478" -w "$env:USERPROFILE\Desktop\egress.pcap"
# 只看 DNS 想解析谁：
& "C:\Program Files\Wireshark\tshark.exe" -r "$env:USERPROFILE\Desktop\egress.pcap" -Y "dns.flags.response==0" -T fields -e dns.qry.name | Sort-Object -Unique
```
方案 B：只抓 DNS（最省事，能立刻暴露「谁在问作者域名」）
```powershell
& "C:\Program Files\Wireshark\tshark.exe" -i 1 -f "udp port 53" -T fields -e dns.qry.name -l |
  Select-String -Pattern 'dshtui|r2049|dshfind|tailscale'
```
方案 C：MuMu 里 root 后看它自己的连接
```bash
adb shell su -c "ss -tnp | grep -E '443|3478'"   # 需要模拟器已 root
```

### 4.4 交叉验证「关掉之后真的不出网」
1. App 内关遥测 + 关自动检查更新。
2. 杀掉 App，重开，按 4.1 轮询 10 分钟。
3. 期望：**不应再看到** `go.dshtui.com`/`dl.dshtui.com`；`controlplane.tailscale.com`/`log.tailscale.com` **仍然会出现**（这两条是 tailnet 必需 + 默认日志上传，见 §2-⑧）。

---

## §5 未决 / 本次没做完的

1. **`hosted_crypto.dart` 的 AEAD 实现没有逐行审**（只确认「key 不进 HTTP 请求」这一层）；如果哪天真要用托管分享，建议单独看一眼算法与 nonce 管理。
2. **AAR 里的 tsnet 是否真的在跑、跑了多久、传了多大的日志**：本次只有静态证据（源码 + 二进制字符串），**没有实抓 `log.tailscale.com` 的包**——需要主人在 §4.1 里跑一次才能确认频率。
3. **`tsnet.Server` 关日志的具体 API**：v1.104.5 里没找到 `Logtail bool` 字段，需在有 Go 工具链的机器上确认（PC 上 `go env` 返回 not found，无 NDK，无法复编 AAR）。
4. **DSH 引擎本体（`runtime/locks/dsh-*`）的遥测/更新检查**未审：那是 DSH 自己的口径，不在本 fork 的代码范围；如果主人要「连 DSH 本体都不过任何第三方」，要单独出一份。
5. **AAR 只抽了 `tailnet-arm64.aar`** 的字符串；`tailnet-x64.aar` 未抽（同源构建，预计一致，但严格说未验）。
6. 仓库里 `app/android/app/libs/` 是**未提交的预编译产物**——以后升级 tsnet 时，这个二进制要跟着重建，否则会出现「源码改了、包里还是老版本」的错觉。

---

## 附：证据链（本次实际执行的命令类别）

- PC 端全量扫：`findstr /s /i /n /c:"https://" D:\dsh-ai\dshgo-fork\app\lib\*.dart`（及 packages / android / runtime / native）
- 只读取文件：sftp 单文件读（`fetch-one.mjs`，未写 PC）
- 二进制取证：`python3 zipfile` 读 `app/android/app/libs/tailnet-arm64.aar` → `jni/arm64-v8a/libgojni.so` → 正则抽可打印串
- 上游比对：`raw.githubusercontent.com/tailscale/tailscale/v1.104.5/{tsnet/tsnet.go,logtail/config.go,net/dnsfallback/dnsfallback.go}`
- fork 改动：`git diff -- app/lib/remote/ssh/ssh_transport.dart`（只加了连接超时，无新出口）
