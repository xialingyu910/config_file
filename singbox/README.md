# Momo 仅内核 / reF1nd sing-box 配置

[singbox-simple-ref1nd.json](./singbox-simple-ref1nd.json) 适用于 **reF1nd Fork 1.14.4-reF1nd**，通过 Momo 的“仅内核”模式启动。由 sing-box 的 TUN 自动路由、自动重定向接管流量。

## 使用前填写

公开模板已移除真实订阅地址和面板密钥：

- 将 `providers` 中的 `https://example.com/subscription/tag`、`https://example.com/subscription/wd` 替换为自己的订阅链接。Fork 可读取 Clash YAML 节点订阅。
- 将 `experimental.clash_api.secret` 中的 `CHANGE_ME` 替换为自己的随机密钥。
- 修改后的私有订阅、节点引导文件及密钥不要提交到公开仓库。

## 首次启动引导

订阅及规则集配置了 `initial_path`，用于在无缓存时读取首次启动副本，避免远程下载失败导致核心无法初始化：

- 节点引导文件：`/etc/momo/providers/tag.yaml`、`/etc/momo/providers/wd.yaml`。可从自己现有客户端的同一订阅缓存复制，文件需包含有效节点。
- 规则集引导文件：`/etc/momo/rules/`，按每个规则集的 `initial_path` 和 `format` 放置相应 JSON/SRS。多标签中的 `{tag}` 替换为实际标签。
- 节点引导文件包含私有凭据，本仓库不提供。迁移设备时需自行准备。

引导数据加载后，订阅和规则集继续按远程 URL、更新周期经自身代理更新。节点订阅使用 `clash.meta` User-Agent。缓存数据库启用，后续启动优先读取可用缓存；引导副本不替代远程自动更新。

## 接管与面板

| 项目 | 设置 |
| --- | --- |
| Momo 运行方式 | 仅内核 |
| TUN | `momo-tun`，system 栈，自动路由和自动重定向开启 |
| TUN 地址 | `172.19.0.1/30`、`fdfe:dcba:9876::1/126` |
| DNS 入站 | `dns-in`，TCP/UDP 1053 |
| HTTP/SOCKS 混合代理 | 7890 |
| Clash API / Zashboard | 9095，面板路径 `/ui/` |
| 面板文件 | `/etc/sing-box/ui` |
| 面板下载 | Zashboard 官方 Releases 的 `dist-cdn-fonts.zip` |

Momo 仅内核模式跳过它的混入及透明接管脚本，DNS、TUN、端口和密钥以 JSON 为准。启用前停用其他透明代理，避免端口和接管冲突。

路由器使用 dnsmasq 时，核对上游 DNS 并按需转发到 `127.0.0.1#1053`。面板地址为 `http://路由器IP:9095/ui/`，核心完成初始化后才提供服务。只有一个 Clash API 面板入口，没有独立的 9090 API 服务。

## 分组与分流

- 33 个业务、区域、手动和故障转移策略组，保留候选项顺序。Apple、Spotify、Microsoft 默认直连。
- 应用净化、广告拦截优先于一般业务分类和国内域名/IP 直连；私网及明确的直连例外优先。
- Direct/Global 模式覆盖一般业务规则；未匹配流量交给 `Proxy`。
- 不统一拒绝 STUN、QUIC；保留 853 端口拒绝规则。ICMP 按配置直连，面板中的 `目标IP:0` 可表示 ICMP，不能据此判断为 DNS 连接。
- 38 个有引用的规则集标签，重复分类已统一；不包含 `proxylite`。
- 故障转移使用 Fork 的 urltest/fallback，间隔 180 秒、最大可接受延迟 2 秒。

## DNS

- 国内域名和路由共用 `oc-cn-domain`；国内 IP 响应判断共用 `cn_ip`。
- Rule 模式下，APNs、国服游戏、固定直连的 AppStore/Apple Music/Apple TV 使用阿里 DNS 返回真实 IP。
- Google DoH 经代理，节点域名使用阿里 DNS 解析。
- 保留 IPv4 策略、8192 条缓存、乐观缓存、evaluate 和响应判断。
- Fake-IP 地址段为 `198.19.0.0/16`。

AppStore 使用 2026-10-29 的内嵌两域名快照（`apps.apple.com`、`itunes.apple.com`），不自动更新；其他远程规则集按配置更新。

## 验证

使用 `1.14.4-reF1nd` 检查配置语法；保留的远程规则集已检查下载及编译/反编译。填写真实订阅并准备引导数据后，曾通过不接管网络的完整核心初始化测试，确认 Zashboard 页面、脚本资源及带密钥的 API 可访问。

可自行执行：

```sh
sing-box check -c singbox-simple-ref1nd.json
```

该命令校验配置结构；实际节点、DNS、网络接管和代理连通性仍取决于部署环境。

参考：[Momo](https://github.com/nikkinikki-org/OpenWrt-momo)、[reF1nd Fork](https://github.com/reF1nd/sing-box)、[Zashboard](https://github.com/Zephyruso/zashboard)。
