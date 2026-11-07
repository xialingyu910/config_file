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
| TUN 地址 | `172.19.0.1/30`，仅 IPv4 |
| DNS 入站 | `dns-in`，`0.0.0.0:1053`，TCP/UDP |
| HTTP/SOCKS 混合代理 | 7890 |
| Clash API / Zashboard | 9095，面板路径 `/ui/` |
| 面板文件 | `/etc/sing-box/ui` |
| 面板下载 | Zashboard 官方 Releases 的 `dist-cdn-fonts.zip` |

Momo 仅内核模式跳过它的混入及透明接管脚本，DNS、TUN、端口和密钥以 JSON 为准。启用前停用其他透明代理，避免端口和接管冲突。

路由器使用 dnsmasq 时，将其上游转发到 `127.0.0.1#1053`，关闭 dnsmasq 的额外 DNS 缓存、强制最短 TTL 及自身 DNS 劫持，由 sing-box 负责缓存和透明 DNS 接管。不要将核心返回的 1 秒 Fake-IP TTL 强制延长至 1 小时。面板地址为 `http://路由器IP:9095/ui/`，核心完成初始化后才提供服务。只有一个 Clash API 面板入口，没有独立的 9090 API 服务。

## 分组与分流

- 33 个业务、区域、手动和故障转移策略组，保留候选项顺序。Apple、Spotify、Microsoft 默认直连。
- 应用净化、广告拦截优先于一般业务分类和国内域名/IP 直连；私网及明确的直连例外优先。
- Direct/Global 模式覆盖一般业务规则；未匹配流量交给 `Proxy`。
- UDP STUN 先由独立嗅探规则识别，再优先交给 `默认代理`，顺序高于私网、国内直连及 Direct/Global 模式规则，用于减少 WebRTC 经直连出口收集公网地址。所选节点需要支持 UDP。QUIC 继续按业务规则分流，保留 853 端口拒绝规则。ICMP 按配置直连，面板中的 `目标IP:0` 可表示 ICMP。
- 38 个有引用的规则集标签，重复分类已统一；不包含 `proxylite`。
- 故障转移使用 Fork 的 urltest/fallback，间隔 180 秒、最大可接受延迟 2 秒。

## DNS

- 国内域名和路由共用 `oc-cn-domain`；国内 IP 响应判断共用 `cn_ip`。
- Rule 模式下，APNs、国服游戏、固定直连的 AppStore/Apple Music/Apple TV 使用阿里 DNS 返回真实 IP。
- Google DoH 经代理，节点域名使用阿里 DNS 解析。
- 保留 IPv4 策略、8192 条缓存、乐观缓存、evaluate 和响应判断。
- AAAA 查询优先返回 `NOERROR` 空结果，域名解析器显式使用 `ipv4_only`，路由优先拒绝显式 IPv6 目标。
- Fake-IP 地址段为 `198.19.0.0/16`，启用 `store_fakeip` 持久化映射，避免核心重启后客户端仍持有旧地址而核心丢失对应域名。
- 单次 DNS 查询默认超时 5 秒；单条规则可覆盖此值。
- `223.5.5.5/32`、`223.6.6.6/32` 从 TUN 接管中排除，供直连 DNS 和独立上游连通性检测使用。

AppStore 使用 2026-10-29 的内嵌两域名快照（`apps.apple.com`、`itunes.apple.com`），不自动更新；其他远程规则集按配置更新。

2026-11-07，设备按 IPv4-only 使用。WAN6 已停用并关闭自动启动，LAN 的 RA、DHCPv6、NDP 及 IPv6 前缀分配已关闭，dnsmasq 过滤 AAAA。设备接口关闭 IPv6，系统保留本机回环 `::1`。这些系统设置独立于 JSON，迁移设备时需另行配置。客户端应重连网络并清理 DNS 缓存，以更新此前取得的 IPv6 地址和解析结果。

## 长期运行与故障恢复

以下是 Momo / dnsmasq 的 UCI 设置，不会因为导入 JSON 而自动生效。先备份 `/etc/config/momo`、`/etc/config/dhcp` 和私有配置，再在适用此模板的仅内核环境中设置：

```sh
uci set momo.procd.rlimit_nofile_soft='65535'
uci set momo.procd.rlimit_nofile_hard='65535'
uci set momo.log.clear_at_stop='0'
uci set momo.log.scheduled_clear_size_limit='8'
uci set momo.log.scheduled_clear_size_limit_unit='MB'
uci set dhcp.@dnsmasq[0].noresolv='1'
uci -q delete dhcp.@dnsmasq[0].server
uci add_list dhcp.@dnsmasq[0].server='127.0.0.1#1053'
uci set dhcp.@dnsmasq[0].cachesize='0'
uci set dhcp.@dnsmasq[0].min_cache_ttl='0'
uci -q delete dhcp.@dnsmasq[0].use_stale_cache
uci set dhcp.@dnsmasq[0].dns_redirect='0'
uci commit momo
uci commit dhcp
/etc/init.d/momo restart
/etc/init.d/dnsmasq restart
```

模板日志级别为 `warn`。提高文件描述符上限可避免全网代理共用默认的约 4096 个描述符限制；正常连接、UDP 会话和数据转发管道均会占用描述符。`store_dns` 与 `store_fakeip` 是不同选项，前者不能替代后者。

[momo-health.sh](./momo-health.sh) 可作为仅内核模式的恢复兜底。将脚本复制到 `/usr/libexec/momo-health` 并设置权限 `700`，在 `/etc/crontabs/root` 中添加一次：

```cron
* * * * * /usr/libexec/momo-health #singbox-core-health
```

随后运行 `/etc/init.d/cron restart`。注释不能包含 `#momo`，否则会被 Momo 重启时的定时任务清理逻辑删除。

- 每分钟检测国内 HTTPS 和本机 7890 代理，各有两个候选站点。
- 启动后等待至少 2 分钟；两条路径连续 3 次均失败才考虑恢复。
- 恢复前通过排除 TUN 的阿里 DNS 地址验证上游 TLS 可达；上游同样失败时不重启。
- 先保存核心日志、系统日志、路由、nftables 和资源用量，再重启 Momo；重启间隔至少 15 分钟。仅某一条路径失败时不重启。
- 仅在指定配置 `file:singbox-simple-ref1nd.json` 启用且运行方式为仅内核时执行恢复；手动停用服务后不拉起服务。
- 日志在 `/tmp/momo-health/health.log`，最近 3 次故障现场在同目录的 `incident-*` 中。均在内存盘，设备重启后清空；现场可能包含访问域名/IP，请勿直接公开提交。

手动只检测、不触发重启：`/usr/libexec/momo-health --check`。该检测覆盖路由器本机路径，仍需结合局域网客户端的实际访问判断透明接管是否正常。自动重启是恢复措施，不能替代故障原因定位。

2026-11-01 排查时发现 dnsmasq 强制最短 TTL 为 3600 秒、Fake-IP 映射未持久化、核心描述符上限约 4096；已修正这些配置。此前故障日志在手动重启时被清空，尚不能将约 8 小时后断网归因于某一项，需结合后续现场及跨夜运行结果确认。

## 验证

使用 `1.14.4-reF1nd` 检查配置语法；保留的远程规则集已检查下载及编译/反编译。填写真实订阅并准备引导数据后，曾通过不接管网络的完整核心初始化测试，确认 Zashboard 页面、脚本资源及带密钥的 API 可访问。

2026-11-01 修正后，私有部署配置与运行配置均通过核心检查；局域网客户端实测国内 HTTPS 200、经代理的检测请求 204、面板 200，Fake-IP DNS TTL 为 1 秒。隔离实例验证了 Fake-IP 映射跨重启保留；恢复脚本用模拟故障验证了连续失败阈值、先记录再恢复、冷却时间、上游断网及仅代理失败时不重启。跨夜稳定性尚需后续运行验证。

可自行执行：

```sh
sing-box check -c singbox-simple-ref1nd.json
```

该命令校验配置结构；实际节点、DNS、网络接管和代理连通性仍取决于部署环境。

参考：[Momo](https://github.com/nikkinikki-org/OpenWrt-momo)、[reF1nd Fork](https://github.com/reF1nd/sing-box)、[Zashboard](https://github.com/Zephyruso/zashboard)。
