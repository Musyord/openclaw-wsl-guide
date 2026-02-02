# Windows 11 环境下 WSL 的初始化与安装

微软在 Windows 11 中极大地简化了安装流程，通过单一命令即可完成核心组件的启用。

## 一键安装流程

在具有管理员权限的 PowerShell 或 Windows 终端中，执行以下命令：

```powershell
wsl --install
```

此操作将自动执行以下逻辑：启用虚拟机平台（Virtual Machine Platform）和适用于 Linux 的 Windows 子系统功能；下载并安装最新的 Linux 内核；默认下载并安装 Ubuntu 发行版。如果系统中已存在旧版 WSL，建议运行 `wsl --update` 以确保获取镜像网络模式所需的内核组件。

## 验证安装状态

安装完成后，必须重启计算机以使 Hyper-V 管理程序生效。重启后，可以通过以下命令验证状态：

```powershell
wsl --status
wsl --version
```

分析结果应确认当前 WSL 版本已切换至 2.0 以上，且默认版本设置为 2。

## Ubuntu 发行版的深度配置与包管理

Ubuntu 作为 WSL 的首选分发版，为 OpenClaw 提供了广泛的社区支持与库依赖。

### 用户初始化与账户安全

首次运行 Ubuntu 时，系统会提示创建 UNIX 用户名和密码。此账户拥有 `sudo` 权限，是后续执行安装操作的核心身份。

### 软件源优化与包索引更新

为了确保下载速度及网络通信的可靠性，建议配置国内镜像源（如清华大学 TUNA 镜像或阿里云镜像）。随后，执行全系统的软件更新：

```bash
sudo apt update && sudo apt upgrade -y
```

这一步骤至关重要，因为它同步了本地包管理器的索引，确保在编译 OpenClaw 时引用的开发库（如 SDL2）处于最新稳定状态。

## 网络通信革命：镜像模式（Mirrored Mode）配置

传统的 WSL2 采用 NAT（网络地址转换）架构，这导致 Linux 环境处于独立的子网中。对于需要与局域网设备深度互通、或在使用 VPN 时保持连接的 OpenClaw 项目，切换至"镜像网络模式"是最佳选择。

### 镜像模式的技术优势分析

镜像模式通过直接将 Windows 宿主机的网络接口状态同步到 Linux 内核中，消除了虚拟子网带来的复杂性。

| 维度 | NAT 模式 (默认) | 镜像模式 (推荐) |
|------|----------------|----------------|
| IP 地址一致性 | 独立虚拟 IP (172.x.x.x) | 与宿主机共享相同局域网 IP |
| IPv6 支持 | 极其有限 | 全面原生支持 |
| localhost 行为 | 跨平台回环复杂 | 原生双向回环支持 |
| VPN 兼容性 | 经常导致 DNS 丢包 | 显著提升兼容性 |
| 局域网可见性 | 需要手动端口转发 | 局域网内其他设备可直接发现 |

### 通过.wslconfig 实施配置

镜像模式属于全局配置，需在 Windows 用户目录（`%UserProfile%`）下创建或修改 `.wslconfig` 文件。

使用记事本打开 `C:\Users\<您的用户名>\.wslconfig`，添加以下内容：

```ini
[wsl2]
# 启用镜像网络模式
networkingMode=mirrored
# 启用 DNS 隧道，防止 VPN 环境下域名解析失效
dnsTunneling=true
# 强制 WSL 使用 Windows 的 HTTP 代理设置
autoProxy=true
# 启用集成防火墙支持
firewall=true

[experimental]
# 自动回收闲置内存，优化性能
autoMemoryReclaim=gradual
# 支持主机回环地址访问
hostAddressLoopback=true
```

配置保存后，必须在 Windows 终端中执行 `wsl --shutdown` 命令。等待约 8 秒钟以确保虚拟机彻底关闭，随后重新启动 Ubuntu 即可加载新的网络架构。

### 防火墙与安全策略调整

在镜像模式下，WSL2 应用将直接暴露在 Windows 防火墙规则中。为了确保 OpenClaw 的网络端口能够被正确访问，需要配置 Windows 防火墙规则。

执行以下 PowerShell 指令以允许必要的入站连接：

```powershell
# 创建入站防火墙规则，允许 OpenClaw 服务端口
New-NetFirewallRule -DisplayName "OpenClaw-Service" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 18789

# 验证规则是否创建成功
Get-NetFirewallRule -DisplayName "OpenClaw-Service" | Format-Table
```
