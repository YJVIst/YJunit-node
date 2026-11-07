# YJunit Node

YJunit 的节点被控与代理内核二进制发行仓库，支持 Linux amd64 / arm64。

包含 Rust agent、sing-box、Xray、Shadowsocks Rust，以及安装、更新和卸载维护程序。主控面板在 [YJunit](https://github.com/YJVIst/YJunit) 下载。

当前 agent **v0.11.0**，请先升级主控到 **v0.20.0**，再在服务器管理中更新被控。

包含 sing-box 1.14.4-yjunit.5、Xray 26.10.29-yjunit.1 和独立 Shadowsocks Rust 1.25.0-yjunit.1。新 Rust 内核支持认证 AEAD/SS2022 算法、TCP/UDP、HTTP 伪装（可配置路径）、WS、TLS 与可选 MPTCP（不支持时回退 TCP）。已有 sing-box/Xray 节点继续使用原内核。详见[配置与客户端矩阵](docs/shadowsocks-rust.md)。

## 接入节点

1. 在支持独立节点仓库的主控中打开 **服务器管理 → 添加服务器**。
2. 填写服务器名称，按需选择 BBR / FQ，生成接入命令。
3. 在目标 Linux 服务器上以 root 执行命令。命令有效期为 10 分钟，包含私有接入凭据，请勿公开。
4. 等待面板显示服务器在线，再创建节点和绑定套餐。

安装命令从本仓库的正式 Release 下载对应架构文件，并校验 GitHub 资产摘要、SHA256SUMS 和包内文件清单。主控仅提供短期安装凭据与业务配置。

要求 systemd、出站 HTTPS 和可用的系统软件源。安装器支持 apt、dnf、yum、zypper；需要 Python 3.9 或更新版本。BBR 还取决于主机 Linux 内核支持，不保证提高每条线路的速度。

## 更新与故障切换

- 在主控服务器卡片中检查更新，或使用“一键更新全部服务器”。更新前校验版本与摘要，保留配置和流量日志；健康检查失败会尝试回退。
- 旧版维护服务使用主控仓库，须先通过新版主控的接入命令迁移一次。不要把新仓库的版本号交给旧维护服务。
- 故障切换会撤销原接入令牌，并保留逻辑服务器与节点绑定。在备用服务器执行新命令即可恢复配置。使用自定义连接域名时，还需把 DNS 指向新服务器。
- 在线删除会清理本项目的 agent、代理内核、配置和服务。离线服务器只能撤销授权，无法远程删除文件；已取得的授权最长在 60 秒租约到期后失效。

安装目录与日志位置见 [节点维护说明](docs/node-maintenance.md)。

## 许可

YJunit agent 和安装脚本使用 MIT 许可，允许修改、二次分发及商业使用，须保留许可声明。sing-box、Xray、Shadowsocks Rust 分别遵循 GPL-3.0-or-later、MPL-2.0 和 MIT；各自对应的修改后源码、许可和构建元数据随节点包提供。
