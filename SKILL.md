---
name: openclaw-install
description: 通过 SSH 远程安装和配置 openclaw 项目（支持 macOS 和 Linux/WSL）
arguments:
  - name: ssh_address
    description: SSH 连接地址，如 root@your-server.ts.net
    required: true
---

# OpenClaw 远程安装与配置

通过 SSH 在远程机器上安装和配置 [openclaw](https://github.com/openclaw/openclaw)。
支持 macOS (arm64/x86_64) 和 Linux (WSL/原生)。

## 前置条件

- 本地已配置 SSH 免密登录到目标机器
- 目标机器有网络访问能力

## 安装流程

### 1. 检测远程环境（关键）

```bash
ssh <ssh_address> 'uname -s; uname -m; echo "HOME=$HOME"; which node 2>/dev/null; node -v 2>/dev/null; echo "PATH=$PATH"; which openclaw 2>/dev/null'
```

根据输出判断：
- **操作系统**：Darwin = macOS, Linux = Linux/WSL
- **架构**：arm64/x86_64
- **HOME 目录**：macOS root 是 `/var/root`，Linux root 是 `/root`（关键区别）
- **Node.js** 是否已安装及版本（需要 >= 22）
- **PATH 完整性**：npm 全局 bin 目录是否在 PATH 中
- **openclaw** 是否已安装

> **关键教训**：macOS 的 root 用户 HOME 目录是 `/var/root`，不是 `/root`。
> 所有涉及 `~` 或 `$HOME` 的路径在 macOS root 下解析为 `/var/root`。
> 配置文件中的硬编码路径（如 `workspace: "/root/.openclaw/workspace"`）必须改为实际 HOME。

### 2. 修复 PATH（如需要）

npm 全局安装的二进制可能不在 PATH 中，特别是非标准 Node.js 安装。

**检测 npm prefix：**
```bash
ssh <ssh_address> 'npm config get prefix'
```

**如果 prefix/bin 不在 PATH 中，永久修复：**

macOS:
```bash
ssh <ssh_address> 'grep -q "<prefix>/bin" /etc/profile || echo "export PATH=<prefix>/bin:\$PATH" >> /etc/profile'
```

Linux:
```bash
ssh <ssh_address> 'grep -q "<prefix>/bin" ~/.bashrc || echo "export PATH=<prefix>/bin:\$PATH" >> ~/.bashrc'
```

> **关键教训**：通过 SSH 执行命令时 PATH 可能不完整。
> 如果 `openclaw` 命令找不到，用 `PATH=<prefix>/bin:$PATH openclaw ...` 前缀执行。
> 后续所有命令建议都带上完整 PATH 前缀，避免 `command not found`。

### 3. 安装 Node.js（如需要）

Node.js >= 22 是必须的。

**macOS (Homebrew):**
```bash
ssh <ssh_address> "brew install node@22"
```

**Linux/WSL (NodeSource):**
```bash
ssh <ssh_address> "curl -fsSL https://deb.nodesource.com/setup_22.x | bash -"
ssh <ssh_address> "apt install nodejs -y"
```

### 4. 安装 openclaw

```bash
ssh <ssh_address> "npm install -g openclaw@latest"
```

验证安装（注意 PATH）：
```bash
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw --version'
```

### 5. 上传配置文件

如果本地有已准备好的 `openclaw.json`：

```bash
ssh <ssh_address> "mkdir -p ~/.openclaw && chmod 700 ~/.openclaw"
scp /path/to/openclaw.json <ssh_address>:~/.openclaw/openclaw.json
```

> **关键教训**：上传配置文件后必须检查和修复平台相关路径：
>
> **1. workspace 路径**：如果配置中有 `"workspace": "/root/.openclaw/workspace"`，
> 在 macOS 上必须改为 `"/var/root/.openclaw/workspace"`（或直接用 `$HOME/.openclaw/workspace`）。
>
> **2. browser.executablePath**：Linux 的 Chromium 路径在 macOS 上不存在，
> 需要清空让 openclaw 自动检测。
>
> **修复命令：**
> ```bash
> ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw config set agents.defaults.workspace "$HOME/.openclaw/workspace"'
> ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw config set browser.executablePath ""'
> ```

### 6. 配置 openclaw

#### 6.1 运行 doctor（优先）

```bash
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw doctor --fix'
```

#### 6.2 设置 Gateway

```bash
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw config set gateway.mode local'
```

#### 6.3 修复权限

```bash
ssh <ssh_address> "chmod 700 ~/.openclaw && chmod 600 ~/.openclaw/openclaw.json && chmod 700 ~/.openclaw/credentials 2>/dev/null"
```

#### 6.4 设置 agent 默认模型

格式为 `provider/model-id`：
```bash
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw config set agents.defaults.model.primary "bailian/qwen3.5-plus"'
```

**注意：** 不要用 `provider:model` 格式，否则会被错误解析。

### 7. 配置聊天渠道

#### Telegram

```bash
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw config set channels.telegram.enabled true'
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw config set channels.telegram.botToken "<TELEGRAM_BOT_TOKEN>"'
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw config set channels.telegram.dmPolicy pairing'
```

#### 飞书

```bash
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw config set channels.feishu.enabled true'
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw config set channels.feishu.appId "<APP_ID>"'
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw config set channels.feishu.appSecret "<APP_SECRET>"'
```

### 8. 启动 Gateway

#### macOS - SSH 环境（推荐 LaunchDaemon）

> **关键教训**：macOS 上通过 SSH 启动后台进程需要注意：
> - `nohup` 在 SSH 非 TTY 环境下会报 `Inappropriate ioctl for device` 并失败
> - `openclaw gateway install` 创建的 LaunchAgent 在 root SSH session 中无法 bootstrap（`Domain does not support specified action`）
> - **正确方案**：使用 LaunchDaemon（系统级服务），而不是 LaunchAgent（用户级服务）

**创建 LaunchDaemon：**

先获取路径信息：
```bash
OPENCLAW_PATH=$(ssh <ssh_address> 'which openclaw 2>/dev/null || echo $(npm config get prefix)/bin/openclaw')
NODE_BIN_DIR=$(ssh <ssh_address> 'dirname $(which node)')
```

然后创建 plist（替换 $OPENCLAW_PATH 和 $NODE_BIN_DIR 为实际值）：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.gateway</string>
    <key>ProgramArguments</key>
    <array>
        <string>$OPENCLAW_PATH</string>
        <string>gateway</string>
        <string>--port</string>
        <string>18789</string>
        <string>--bind</string>
        <string>lan</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>UserName</key>
    <string>root</string>
    <key>StandardOutPath</key>
    <string>/var/root/.openclaw/gateway-stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/var/root/.openclaw/gateway-stderr.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>HOME</key>
        <string>/var/root</string>
        <key>PATH</key>
        <string>$NODE_BIN_DIR:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/opt/homebrew/bin</string>
    </dict>
</dict>
</plist>
```

写入后加载：
```bash
ssh <ssh_address> "launchctl load /Library/LaunchDaemons/ai.openclaw.gateway.plist"
```

**临时快速启动（不推荐长期使用）：**
```bash
ssh -t <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw gateway --port 18789 --bind lan &'
```

#### Linux/WSL - systemd

```bash
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw gateway install'
```

或手动：
```bash
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw gateway start'
```

### 9. 验证

```bash
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw status'
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw channels status'
```

预期输出：
```
Gateway reachable.
- Telegram default: enabled, configured, running, mode:polling, token:config
```

### 10. 配对用户

用户首次在 Telegram 发消息时会收到配对码，批准：
```bash
ssh <ssh_address> 'PATH=$(npm config get prefix)/bin:$PATH openclaw pairing approve telegram <PAIRING_CODE>'
```

## 常见问题

### 问题1：macOS `/root` 路径不存在

- **现象**：`ENOENT: no such file or directory, mkdir '/root'`
- **原因**：macOS root 的 HOME 是 `/var/root`，不是 `/root`。配置文件中硬编码了 `/root` 路径
- **解决**：
  ```bash
  openclaw config set agents.defaults.workspace "$HOME/.openclaw/workspace"
  ```

### 问题2：`openclaw: command not found`

- **现象**：SSH 执行 `openclaw` 报 command not found
- **原因**：npm 全局 bin 目录不在 SSH session 的 PATH 中
- **解决**：
  ```bash
  # 查找 npm prefix
  npm config get prefix
  # 临时修复
  PATH=<prefix>/bin:$PATH openclaw ...
  # 永久修复
  echo 'export PATH=<prefix>/bin:$PATH' >> /etc/profile
  ```

### 问题3：macOS SSH 下 `nohup` 启动失败

- **现象**：`nohup: can't detach from console: Inappropriate ioctl for device`
- **原因**：macOS SSH non-TTY 环境下 nohup 不工作
- **解决**：使用 LaunchDaemon（见步骤 8），或用 `ssh -t` 分配 TTY

### 问题4：macOS `openclaw gateway install` 失败

- **现象**：`Bootstrap failed: 125: Domain does not support specified action`
- **原因**：`gateway install` 创建 LaunchAgent，但 root 的 SSH session 无 GUI domain
- **解决**：手动创建 LaunchDaemon（见步骤 8）

### 问题5：上传配置后 browser 路径错误

- **现象**：浏览器相关功能报错，路径指向 Linux 的 Chromium
- **原因**：`browser.executablePath` 是另一个平台的路径
- **解决**：
  ```bash
  openclaw config set browser.executablePath ''
  ```

### 问题6：API 认证失败 - HTTP 401

- **现象**：`HTTP 401: Incorrect API key provided`
- **解决**：
  ```bash
  openclaw config set env.DASHSCOPE_API_KEY "sk-xxx"
  openclaw config set models.providers.bailian.apiKey '${DASHSCOPE_API_KEY}'
  ```

### 问题7：Gateway 端口被占用

- **现象**：`Port 18789 is already in use`
- **解决**：
  ```bash
  openclaw gateway stop
  # 或
  pkill -f openclaw
  ```

### 问题8：SSH 命令中 PATH 污染

- **现象**：`-sh: Files/nodejs:/cmd:/c/Program: not a valid identifier`
- **原因**：Windows 本地的 PATH（含空格）通过双引号 SSH 命令泄漏到远程 shell
- **解决**：使用单引号包裹 SSH 命令，避免本地变量展开：
  ```bash
  # 错误：双引号会展开本地 $PATH
  ssh host "export PATH=/usr/local/bin:$PATH && ..."
  # 正确：单引号阻止本地展开
  ssh host 'PATH=/usr/local/bin:$PATH openclaw ...'
  ```

### 问题9：Gateway 重启

**macOS LaunchDaemon：**
```bash
launchctl unload /Library/LaunchDaemons/ai.openclaw.gateway.plist
sleep 2
launchctl load /Library/LaunchDaemons/ai.openclaw.gateway.plist
```

**Linux systemd：**
```bash
systemctl --user restart openclaw-gateway.service
```

## 跨平台路径对照表

| 项目 | macOS (root) | Linux (root) |
|------|-------------|-------------|
| HOME | `/var/root` | `/root` |
| 配置目录 | `/var/root/.openclaw/` | `/root/.openclaw/` |
| 配置文件 | `/var/root/.openclaw/openclaw.json` | `/root/.openclaw/openclaw.json` |
| workspace | `/var/root/.openclaw/workspace` | `/root/.openclaw/workspace` |
| 后台服务 | LaunchDaemon (`/Library/LaunchDaemons/`) | systemd (`~/.config/systemd/user/`) |
| Node.js 安装 | Homebrew / 官方 pkg | NodeSource apt |
| nohup via SSH | 不可用 | 可用 |
| gateway install | SSH 下不可用（需 LaunchDaemon） | 可用 |

## 参考文档

- 官方仓库：https://github.com/openclaw/openclaw
- 官方文档：https://docs.openclaw.ai/cli
- 故障排除：https://docs.clawd.bot/troubleshooting
