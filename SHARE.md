# 分享文案

**主题：Codex 能聊天却连不上 Linux / Termux？不用 TUN 的修复记录**

这次遇到的情况是：Windows 上 Codex 普通聊天正常，但连接远端一直卡在“初始化安全连接”，出现 ECONNRESET 或握手超时。

排查发现，控制其他设备的桌面 WebSocket 与本机被控服务走的是不同网络路径。仅修改 .env，不能证明远端连接已经恢复。最终使用 ProxyBridge，把实际 Codex 桌面程序的 TCP 443 连接转到现有本地代理，Linux 和 Termux 都恢复连接，全程未启用 TUN。

整理了中文教程、可复制给编码助手的排查提示词，以及默认禁用、需按本机修改后启用的配置示例。教程包含验证步骤、停用方法和适用范围，不附带私人日志或凭据。

项目地址：https://github.com/gzweiqi001-afk/remote

关键词：Codex / Windows / Remote Control / Linux / Termux / WebSocket / v2rayN / ProxyBridge / 不使用 TUN
