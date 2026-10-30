# 节点维护

默认路径：

| 内容 | 路径 |
| --- | --- |
| agent、sing-box、Xray | `/usr/local/lib/yjunit-agent/` |
| 私有接入配置 | `/etc/yjunit-agent/config.json` |
| 节点配置与计量日志 | `/var/lib/yjunit-agent/` |
| 更新回退文件 | `/var/lib/yjunit-agent-maintenance/` |

查看服务：

```sh
systemctl status yjunit-agent yjunit-agent-updater
journalctl -u yjunit-agent -n 100 --no-pager
journalctl -u yjunit-agent-updater -n 100 --no-pager
```

主控只需开放 HTTPS；不需要保存节点 SSH 密码。客户端代理端口需要按节点配置自行开放。代理内核只使用本项目的程序路径与 systemd 服务，不替换其他面板安装的同名内核。

agent 每分钟检测网卡和出口 IP。出口 IP 通过 ipify 获取，供应商信息通过 ipwho.is 获取；查询不包含用户流量或接入令牌。查询失败保留上次结果并显示状态，不阻断代理转发。供应商信息最多按小时刷新，IP 变化时重新查询。

在线删除需要维护服务支持 `retirement-v1`。清理任务只能停止和删除本项目固定路径，保留其他服务、主控维护程序和系统 BBR/FQ 设置。若回执未成功送达，面板可能仍显示等待结果，应检查机器上的卸载日志，不能将等待状态视为清理成功。

离线机器请先确认主控已撤销接入，再在机器恢复后自行清理本项目服务。不要将 `rm -rf` 用于不确定归属的目录。更换服务器时应在原机器停止旧被控，避免其持续尝试连接。
