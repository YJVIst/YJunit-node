# Shadowsocks Rust 节点

在节点管理中选择 **Shadowsocks Rust**，由独立的 Rust 内核提供服务。已有 Shadowsocks 和 AnyTLS 节点继续使用 sing-box；Xray 扩展节点继续使用 Xray。

本功能需要配套 agent v0.11.1 和 `shadowsocks-rust 1.25.0-yjunit.1`。主控 v0.20.1 起提供配置入口，请先更新主控，再在服务器管理中更新被控。

## 节点配置

1. 选择已安装配套内核的服务器，填写节点名称、连接地址和端口，绑定套餐。
2. 选择加密算法。默认使用 `2022-blake3-aes-128-gcm`；SS2022 AES 的服务端密钥由面板生成、加密保存。用户密码沿用该用户的节点凭据。
3. 选择 TCP 或 WebSocket。TCP 可以选择无伪装或 HTTP 伪装；HTTP 伪装与 WebSocket 可填写 Host，两者均可填写各自的路径；HTTP 默认路径为 `/`。
4. 如需 TLS，启用后填写 SNI，并选择自签、ACME 或上传证书。允许不安全连接只影响客户端的证书校验，不会取消 TLS。
5. 原生无伪装 TCP 可启用 TCP、UDP 或两者。HTTP 伪装、WebSocket、TLS 组合仅承载 TCP，界面会相应限制网络选项。

TLS 是实际的外层 TLS，不是 simple-obfs 的 TLS 外观伪装。HTTP 伪装使用 simple-obfs 风格的 HTTP 前导头；WebSocket 使用标准二进制帧，二者不能互相替代。

### 算法

| 类型 | 已集成算法 |
| --- | --- |
| SS2022 | `2022-blake3-aes-128-gcm`、`2022-blake3-aes-256-gcm`、`2022-blake3-chacha20-poly1305`、`2022-blake3-chacha8-poly1305` |
| AEAD | `aes-128-gcm`、`aes-256-gcm`、`chacha20-ietf-poly1305`、`xchacha20-ietf-poly1305` |
| AEAD 扩展 | `aes-128-ccm`、`aes-256-ccm`、`aes-128-gcm-siv`、`aes-256-gcm-siv` |

这不是上游所有编译特性的全集。未集成无认证的旧式流加密、明文模式、QUIC 插件和第三方插件任意命令执行。旧式算法无法按本项目的多用户身份认证模型可靠区分账户，因此不在可选列表中。

### 客户端兼容

订阅按客户端能力过滤，不会把无法表达的配置转换成另一种传输。Xray JSON 支持常用算法的 TCP、HTTP 伪装、WS 及相应 TLS 组合；扩展算法需要实际支持它的 Shadowsocks Rust 客户端。

sing-box、Clash/Mihomo 的 HTTP/WS 通过各自的 SIP003 兼容功能导出。SIP003 的 TLS Host 与 SNI 必须一致。sing-box / NekoBox 的 WS 自签证书通过 certRaw 下发，Mihomo 通过插件 fingerprint 固定校验；无法表达单独 SNI 或不安全校验选项的格式会排除该节点。通用链接的插件参数也需要客户端本身支持，不能仅根据客户端能导入 `ss://` 就判断插件可用。Stash、Surge、Surfboard 的算法和插件限制仍会参与过滤。

### HTTP 路径与 MPTCP

HTTP 路径是实际请求路径，默认 `/`。填写自定义路径后，客户端必须发送相同路径；不接受查询参数、空白或插件分隔符。Xray JSON、v2rayN 和 v2rayNG 会下发 `tcpSettings.header.request.path`，通用链接同时使用 `obfs-uri` 和 `path`；Surge / Surfboard 使用 `obfs-uri`。sing-box、Clash/Mihomo 等内置 simple-obfs 实现仅发送 `/`，自定义 HTTP 路径节点会从这些模板中排除；通用链接需要使用支持 `obfs-uri` 的插件客户端。

“启用 MPTCP”默认关闭，仅对本协议包含 TCP 的节点生效，UDP 不变。开启后入站监听和到目标站点的 TCP 拨号优先使用 Linux MPTCP；本机内核不支持、禁用或沙箱不允许创建 MPTCP socket 时回退 TCP。对端或网络不支持时由内核协商回退；出站 MPTCP 建连超过 3 秒或失败时，在发送任何应用数据之前尝试普通 TCP，仍受原有总超时限制。端口被占用和文件描述符耗尽等真实错误不会伪装成成功。

Xray JSON、sing-box 和 Clash Meta/Mihomo 模板会下发各自的 MPTCP 开关。其他模板没有统一可移植的开关时保持普通 TCP；服务器仍可接受这些客户端。支持 MPTCP 的客户端也必须具备对应操作系统和内核支持。

MPTCP 按每一段 TCP 连接协商：中转或反向代理终止 TCP 时，不会自动让整个中转链变成 MPTCP。HTTP/WS/TLS 可叠加在 MPTCP 字节流上，但不是启用后就自动获得多网卡带宽聚合。多路径需要系统路由与端点管理配合；程序不修改服务器全局 sysctl、路由或端点配置。物理链路完全不可达时，回退 TCP 也无法保证连接。

参考：[Linux MPTCP](https://docs.kernel.org/networking/mptcp.html)、[Xray Sockopt](https://xtls.github.io/config/transports/sockopt.html)、[sing-box Dial Fields](https://sing-box.sagernet.org/configuration/shared/dial/)。

### Windows v2rayN 与 Android v2rayNG

在“客户端订阅”选择对应入口：

- **v2rayN（Windows）**：`?format=v2rayn`，对照 v2rayN 7.25.6 的导入代码验证。
- **v2rayNG**：`?format=v2rayng`，对照 v2rayNG 2.3.13 的导入代码验证。

两个入口按 Xray 的协议和算法能力筛选节点；它们不将 sing-box 独有协议强行改写为 Xray。将地址加入客户端订阅分组，再更新订阅。两种入口返回逐节点的完整 Xray JSON 配置数组，每个节点可独立选择，保留原名称、连接信息、HTTP/WS 路径、Host、TLS/SNI、证书校验与 MPTCP。启用私有 DoH 时也为每个配置下发该用户的解析服务。

v2rayNG 2.3.13 的 `ss://` 导入器不解析 `v2ray-plugin` WS/TLS 参数；WS+TLS 请使用专用入口。原“Xray JSON”仍是通用内核配置，不等于客户端逐节点订阅。

完整配置使用默认的本地代理端口 **10808**，只监听 `127.0.0.1`；v2rayN 的 HTTP 系统代理和 SOCKS 均使用此端口。如自行修改了客户端的本地端口，需要保持一致。自定义配置由客户端按原文加载：系统代理、Android VPN、Windows TUN、后台保活及系统权限仍由客户端控制，不能假定完整配置会继承客户端全部路由和 DNS 覆写设置。

### Shadowsocks Rust 传输兼容范围

这里的 HTTP 指 simple-obfs 风格的 HTTP 伪装，不是 HTTP/2 或 HTTP CONNECT。默认路径为 `/`，自定义路径严格匹配；不将路径静默替换为 `/`。Mihomo 的 WS 实现会规范化百分号编码（例如 `%2F` 会改变实际请求路径），因此包含 `%` 的 WS Path 暂不下发到 Clash / Mihomo / Stash 模板；普通 `/proxy/ws` 路径不受影响，Xray 与 sing-box 保留编码路径。

| 客户端 / 格式 | 普通 TCP | HTTP `/` | HTTP 自定义 Path | WS `/` / 自定义 Path | TLS |
| --- | --- | --- | --- | --- | --- |
| v2rayN / v2rayNG 专用 | 支持 | 支持 | 支持 | 支持 | TCP、HTTP、WS 均可；支持独立 SNI、证书信任 |
| sing-box / NekoBox 1.4.2 | 支持 | 支持 | 内置插件不支持 | 支持 | WS 支持；Host 与 SNI 相同，自签证书通过 certRaw |
| Clash Meta / Mihomo / Verge | 支持 | 支持 | 内置插件不支持 | 支持 | WS 支持；Host 与 SNI 相同，自签证书通过 fingerprint |
| Stash | 支持 | 支持 | 未下发 | 支持 | WS；严格自签证书暂排除，避免丢失信任配置 |
| Surge / Surfboard | 支持 | 支持 | 使用 obfs-uri | 不下发 | 不将 Shadowsocks TLS 冒充 simple-obfs TLS |
| 通用 Base64 | 依赖客户端算法支持 | 依赖插件 | 依赖插件对 obfs-uri / path 的支持 | 依赖 v2ray-plugin 支持 | 依赖客户端插件实现 |

算法能力也参与过滤，表中的支持不代表所有加密算法均被所有客户端支持。HTTP Host 留空时使用连接地址；TLS 下优先使用 SNI。WS 关闭 mux，避免与不支持 V2Ray mux 帧的服务端混用。订阅响应的 `X-YJunit-Excluded-Nodes` 返回排除数量；不会通过关闭证书验证来强行兼容。显式选择“允许不安全连接”的节点只下发给能表达该设置的格式。新版 Xray 已移除 `allowInsecure=true`；对于主控持有证书指纹的此类节点，专用模板改用 `pinnedPeerCertSha256` 固定服务器证书，不跳过身份校验。如果没有已知指纹（例如证书仅由 agent 申请），则从 Xray 格式排除，需关闭不安全连接或提供证书。

NekoBox 的通用 Base64 导入可保留插件参数，但内置 HTTP-obfs 仍不读取自定义 HTTP Path。此组合请使用 v2rayN / v2rayNG，或为 NekoBox 选择默认 HTTP 路径、WS 节点。其 sing-box 完整配置导入会抽取节点，不会自动继承完整配置中的 DNS / 路由；私有解析需遵循客户端自身 DNS 配置方式。

验证分层记录客户端导入代码、内核检查和真实上传/下载；缺少 Windows / Android / Apple 真机时，不将 Linux 内核连接测试宣称为这些平台的 GUI/VPN 实测。Stash、Surge、Surfboard 的专有内核按官方文档核对，仍需对应设备验收。

参考：[v2rayN 导入代码](https://github.com/2dust/v2rayN/blob/7.25.6/v2rayN/ServiceLib/Handler/Fmt/V2rayFmt.cs)、[NekoBox 导入代码](https://github.com/MatsuriDayo/NekoBoxForAndroid/blob/1.4.2/app/src/main/java/io/nekohasekai/sagernet/fmt/shadowsocks/ShadowsocksFmt.kt)、[Mihomo Shadowsocks](https://github.com/MetaCubeX/mihomo/blob/v1.19.33/adapter/outbound/shadowsocks.go)、[Stash 插件说明](https://stash.wiki/proxy-protocols/proxy-types)、[Surge Shadowsocks](https://manual.nssurge.com/policies/shadowsocks.html)、[Surfboard Shadowsocks](https://getsurfboard.com/docs/profile-format/proxy/external-proxy/shadowsocks/)。

## 授权、计量与审计

- HTTP、WS、TLS 入口在同一 Rust 进程内处理，保留原始连接 IP，再调用 agent 私有 Unix socket 的设备准入。
- 累计上传、下载计数接入原有持久化上报、去重和倍率处理，不在内核中重复乘倍率。
- 用户过期、账户停用、流量耗尽、设备限制和配置租约继续生效。控制租约停止续期时，新内核关闭转发并保留末尾计数，等待 agent 收集。
- 支持现有域名、域名后缀、正则、IP/CIDR、端口及载荷审计。载荷规则检查连接前 16 KiB，不是无限制的全流量内容检测；已转发的前缀不能收回。
- SS2022 AES 使用 EIH 区分用户。其他算法采用认证头匹配；用户很多时，应优先选择 SS2022 AES，避免线性试钥成本。
- UDP 使用按用户隔离的会话和重放窗口，丢弃未认证或重复数据包。并发任务、队列和会话数量均有上限。

## 构建

依赖 Rust 稳定版（本次验证使用 1.99.1）、Python 3.10+、`patch`、C/C++ 编译工具。跨架构静态构建另外需要 Zig 和 cargo-zigbuild。

```bash
rustup target add x86_64-unknown-linux-musl aarch64-unknown-linux-musl
cargo install cargo-zigbuild --locked
python3 deploy/build-ssrust.py --zigbuild --target x86_64-unknown-linux-musl --output node-artifacts
python3 deploy/build-ssrust.py --zigbuild --target aarch64-unknown-linux-musl --output node-artifacts
```

构建脚本固定上游提交 `e697eb292b66e1e3977c4e7dbd2026ca7842ef1e`，校验下载摘要后应用 `deploy/ssrust/upstream.patch`。该提交的包版本为 1.25.0，不能将此开发快照误认为上游正式发行。

产物包含两个架构的二进制、对应源码、MIT 许可和构建清单。准备好相同版本的 agent、sing-box、Xray 产物后，使用 `deploy/package-node-release.py` 生成独立节点包。主控二进制包不携带节点内核。

安装路径为 `/usr/local/lib/yjunit-agent/shadowsocks-rust`，配置与内部控制 socket 位于 `/var/lib/yjunit-agent/ssrust`，只允许服务账户访问。内核不开放额外的公网管理端口。
