# Codex 远程连接修复：不用 TUN，只代理桌面程序

Windows 上 Codex 普通聊天正常，但连接 Linux / Termux 一直停在“初始化安全连接”，随后出现 `ECONNRESET` 或 `Opening handshake has timed out`？这份教程记录一次已恢复连接的排查过程，并提供可复制给编码助手的提示词。

> 记录日期：2026-10-23。本文是一次 Windows 环境下的实测经验，不是 OpenAI 官方方案，也不保证适用于所有版本或所有连接故障。

## 已验证的环境与结果

| 项目 | 实测情况 |
| --- | --- |
| 桌面程序 | Windows Store 包 `OpenAI.Codex 26.1016.8622.0`，界面日志版本 `26.1016.62342` |
| 本地代理 | v2rayN / Xray，监听 `127.0.0.1:10808` |
| 目标设备 | Linux 与 Termux 的 Codex Remote Control 主机 |
| 故障 | 桌面客户端控制其他设备时，WebSocket 直接出站并失败 |
| 转发工具 | ProxyBridge 4.3.0（Windows） |
| 验证 | 应用按程序转发后，Linux 与 Termux 均进入 `connected`，用户确认可以连接 |

实测生效的是 ProxyBridge CLI 加载限定规则的过程。教程中的 GUI 导入方式来自工具官方文档；开机启动和长期稳定性没有在这次排查中单独验证。

## 先分清连接方向

1. **其他设备控制这台 Windows**：本机作为被控主机，日志主要在 `.codex/logs_2.sqlite`，连接端点常包含 `/wham/remote/control/server`。
2. **这台 Windows 控制 Linux / Termux**：本机作为控制端，桌面日志位于 `%LOCALAPPDATA%\Codex\Logs`，连接端点常包含 `/codex/remote/control/client`。
3. **通过 SSH 连接项目**：检查 SSH 登录、远端登录 shell、Codex 命令是否可用；这是另一条连接路径。

本案例最初只检查了第一种方向。即使它显示 `Connected`，也不能证明第二种方向恢复。排查时先确认用户真正失败的方向，可以少走很多弯路。

## 为什么 `.env` 不一定能解决

本机安装包中的控制端代码用 Node 的 `ws` 创建 WebSocket。在检查到的调用处，没有显式提供代理 `agent` 或 `createConnection`。同时，Windows 的桌面主进程存在直连外部 `443` 的 `SynSent`，桌面日志反复报握手超时或连接重置。

普通聊天、本机被控与控制其他设备使用的网络实现可能不同。因此，“浏览器能打开 ChatGPT”“普通聊天正常”“本机远程控制服务 Connected”都不足以证明控制端 WebSocket 可用。

这次修改 `.codex/.env` 与 Rust 侧的代理配置后，本机被控连接恢复，但主动连接远端仍失败。最终有效的措施是把**实际发起连接的桌面程序**按规则转到本地代理。不能把此结果推广成“所有版本都忽略代理环境变量”，也不能断定某个配置项单独造成了恢复。

## 诊断步骤

### 1. 确认代理监听和协议

在 PowerShell 中检查实际端口。以下 `10808` 只是示例，使用你自己的值。

```powershell
Get-NetTCPConnection -State Listen -LocalPort 10808
```

通过代理访问远程控制地址：

```powershell
curl.exe --socks5-hostname 127.0.0.1:10808 --max-time 15 `
  -sS -o NUL -w "HTTP=%{http_code}\n" `
  https://chatgpt.com/backend-api/wham/remote/control/server
```

普通 HTTPS 请求返回 `426 Upgrade Required`，说明请求到达了需要 WebSocket Upgrade 的端点；这**不等于认证后的 WebSocket 已连接成功**。如果你的端口只支持 HTTP 代理，改用 `--proxy http://127.0.0.1:<端口>`。

### 2. 检查正确的日志和进程

从 `%LOCALAPPDATA%\Codex\Logs` 的最新日志中搜索：

```text
remote_control_websocket.connect_failed_before_open
Opening handshake has timed out
read ECONNRESET
remote_connections.manager_state_set
```

查出桌面主进程的 PID，再检查连接。安装包中的程序名可能是 `ChatGPT.exe`，不要仅凭产品界面名称猜测进程名。

```powershell
Get-CimInstance Win32_Process -Filter "Name='ChatGPT.exe'" |
  Select-Object ProcessId, ParentProcessId, ExecutablePath

# 用实际主进程 PID 替换 <PID>
Get-NetTCPConnection -OwningProcess <PID> |
  Select-Object RemoteAddress, RemotePort, State
```

将“直连失败”“显式代理探测可达”“实际桌面进程直连”和对应日志结合起来，才能支持代理路径问题的判断。不要仅凭一个错误码认定原因。

## 修复：只代理 Codex，不开启 TUN

### 1. 安装 ProxyBridge

从 [官方发布页](https://github.com/InterceptSuite/ProxyBridge/releases) 下载 Windows 安装包，核对发布页 SHA256 和数字签名，然后安装。

ProxyBridge 会安装 Windows 服务与 WinDivert 驱动。它不需要创建 TUN 网卡，但依然是系统级网络组件；如果机器不允许安装驱动，应先取得管理员同意。不要因为规则仅匹配一个程序，就承诺驱动完全没有兼容性风险。

### 2. 配置本地代理与规则

从 [示例配置](examples/Codex-only.pbprofile) 开始，先修改再导入。不要直接导入不符合本机环境的端口或程序名。

| 设置 | 建议值 |
| --- | --- |
| 代理协议 | 现有代理支持的 SOCKS5 或 HTTP |
| 地址与端口 | 本机代理实际监听地址；示例为 `127.0.0.1:10808` |
| 应用程序 | 选择正在运行的 Codex 桌面程序，优先使用完整可执行文件路径 |
| 目标端口与协议 | `443` / TCP |
| 动作 | 通过现有代理转发 |
| 代理解析域名 | 启用，并在实际连接中验证解析与转发是否正确 |
| 本地回环连接 | 保持直连 |
| 其他程序 | 不添加匹配它们的代理规则 |

示例的 `ProcessName` 为 `ChatGPT.exe`，会匹配同名程序。请使用 ProxyBridge 的 Running / Browse 功能确认实际 Codex 路径，缩小匹配范围。完整路径可能随 Store 更新变化，升级后应重新核对。

GUI：打开 `Profile > Import Profile...` 导入修改后的文件。不要添加匹配所有程序的全局代理规则。

CLI：先退出 ProxyBridge GUI，然后在管理员 PowerShell 中运行：

```powershell
& 'C:\Program Files\ProxyBridge\ProxyBridge_CLI.exe' `
  --profile 'C:\path\to\Codex-only.pbprofile' --verbose 3
```

诊断日志可能记录其他程序的连接信息。验证完成后降低日志等级，避免长期保存不必要的网络记录。CLI 需要持续运行才能维持这条转发路径；不要把临时运行误认为已经配置了开机启动。

### 3. 验证实际连接

让 Codex 自动重连，或手动点击重试，检查：

- ProxyBridge 显示 Codex 桌面程序的连接走指定代理。
- 对应 Linux / Termux 主机的最新状态变为 `connected`，且 `nextError=null`。
- 实际打开远端项目或已有聊天，确认可用。

本案例中，限定规则生效后，两台远端主机都恢复连接。不能只靠 `curl` 返回值或本机被控状态宣布修复成功。

## 停用与恢复

CLI 模式按 `Ctrl+C` 正常退出；GUI 模式禁用这条规则或暂停 ProxyBridge。不要直接卸载原有 v2rayN、修改系统默认路由或关闭其他服务。卸载 ProxyBridge 请使用它自己的卸载程序。

如果本机有未保存工作、正在运行的远程任务或其他网络服务，先确认重启与断开连接的影响。本教程不要求开启 TUN，也不要求为了这次故障关闭防火墙或证书验证。

## 可复用提示词

复制 [PROMPT.md](PROMPT.md) 中的完整提示词给有本机诊断权限的编码助手。提示词要求先确认连接方向、脱敏排查、按证据修复，并以真实远程连接验证结果。

## 参考资料

- [OpenAI：Remote connections](https://learn.chatgpt.com/docs/remote-connections)
- [控制端 WebSocket 忽略系统代理的相关报告 #35495](https://github.com/openai/codex/issues/35495)（macOS 报告；与本案例检查到的控制端代码路径相关，不代表官方确认了 Windows 根因）
- [Windows WebSocket 与系统代理问题 #29958](https://github.com/openai/codex/issues/29958)
- [Windows Remote Control 连接问题 #38768](https://github.com/openai/codex/issues/38768)
- [ProxyBridge Windows 配置](https://proxybridge.interceptsuite.com/docs/configuration-windows/)
- [ProxyBridge CLI](https://proxybridge.interceptsuite.com/docs/windows-cli/)
- [ProxyBridge 安装内容](https://proxybridge.interceptsuite.com/docs/installation-windows/)

## 许可

本仓库原创教程、提示词和示例配置采用 [MIT License](LICENSE)。文中引用的软件及文档遵循各自许可。本仓库未附带第三方程序、账户凭据或原始诊断日志。
