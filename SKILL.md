---
name: openclaw-install
description: 通过 SSH 远程安装和配置 openclaw 项目（支持 macOS 和 Linux/WSL）
invocation: user
arguments:
  - name: ssh_address
    description: SSH 连接地址，如 root@your-server.ts.net
    required: true
---

# OpenClaw 远程安装与配置

通过 SSH 在远程机器上安装和配置 [openclaw](https://github.com/openclaw/openclaw)。
支持 macOS (arm64/x86_64) 和 Linux (WSL/原生)。

## 使用方式

```
/openclaw-install <ssh地址>
```

示例：
```
/openclaw-install root@your-server.ts.net
```

## 前置条件

- 本地已配置 SSH 免密登录到目标机器
- 目标机器有网络访问能力

## 安装流程

### 阶段1：环境检测

```bash
ssh <地址> "uname -s; uname -m; which node; node -v 2>/dev/null; which openclaw 2>/dev/null"
```

根据输出判断：
- 操作系统类型（Darwin = macOS, Linux = Linux/WSL）
- 架构（arm64/x86_64）
- Node.js 是否已安装及版本（需要 >= 22）
- openclaw 是否已安装

### 阶段2：安装 Node.js（如需要）

Node.js >= 22 是必须的。

**macOS (Homebrew):**
```bash
ssh <地址> "brew install node@22"
```

**Linux/WSL (NodeSource):**
```bash
ssh <地址> "curl -fsSL https://deb.nodesource.com/setup_22.x | bash -"
ssh <地址> "apt install nodejs -y"
```

### 阶段3：安装 openclaw

```bash
ssh <地址> "npm install -g openclaw@latest"
```

或从源码构建：
```bash
ssh <地址> "git clone https://github.com/openclaw/openclaw.git && cd openclaw && npm install && npm link"
```

验证安装：
```bash
ssh <地址> "openclaw --version"
```

### 阶段4：配置

配置文件路径：`~/.openclaw/openclaw.json`

```bash
ssh <地址> "mkdir -p ~/.openclaw && chmod 700 ~/.openclaw"
```

#### 4.1 模型配置

用户需提供 API provider 的 baseUrl、API Key 和模型列表。

配置示例（阿里百炼 coding plan）：
```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "bailian": {
        "baseUrl": "https://coding.dashscope.aliyuncs.com/v1",
        "apiKey": "<YOUR_API_KEY>",
        "api": "openai-completions",
        "models": [
          {
            "id": "qwen3.5-plus",
            "name": "qwen3.5-plus",
            "reasoning": false,
            "input": ["text", "image"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 1000000,
            "maxTokens": 65536
          }
        ]
      }
    }
  }
}
```

第三方 Claude API 代理配置示例：
```json
{
  "models": {
    "providers": {
      "custom-anthropic": {
        "baseUrl": "https://your-proxy.com/v1",
        "apiKey": "<YOUR_API_KEY>",
        "api": "anthropic-messages",
        "models": [
          { "id": "claude-opus-4-6", "name": "Claude Opus 4.6" }
        ]
      }
    }
  }
}
```

写入后验证 JSON：
```bash
ssh <地址> "cat ~/.openclaw/openclaw.json | python3 -m json.tool > /dev/null && echo valid || echo invalid"
```

#### 4.2 设置 agent 默认模型

**格式必须是 `provider/model-id`（斜杠分隔）：**
```bash
ssh <地址> 'openclaw config set agents.defaults.model "bailian/qwen3.5-plus"'
```

> ⚠️ **不要用 `provider:model` 格式**，否则会被错误解析为 `anthropic/provider:model`。

#### 4.3 设置 Gateway

```bash
ssh <地址> "openclaw config set gateway.mode local"
```

#### 4.4 修复权限和目录

```bash
ssh <地址> "chmod 600 ~/.openclaw/openclaw.json && mkdir -p ~/.openclaw/agents/main/sessions"
```

#### 4.5 运行 doctor

```bash
ssh <地址> "openclaw doctor --fix"
```

### 阶段5：配置聊天渠道

#### Telegram

**必须先启用插件，再添加 channel：**
```bash
ssh <地址> "openclaw plugins enable telegram"
ssh <地址> 'openclaw channels add --channel telegram --token "<BOT_TOKEN>"'
ssh <地址> 'openclaw config set channels.telegram.groupPolicy open'
```

> 默认 groupPolicy 是 allowlist，会丢弃所有群消息。设为 open 可接收群消息。

#### 其他渠道

查看支持的渠道：
```bash
ssh <地址> "openclaw channels add --help"
ssh <地址> "openclaw plugins list"
```

所有渠道插件默认 disabled，使用前需先 `openclaw plugins enable <插件名>`。

### 阶段6：启动 Gateway

#### macOS - SSH 环境（LaunchDaemon）

SSH 环境下 `openclaw gateway install` 会失败（需要 GUI session），必须手动创建 LaunchDaemon：

```bash
ssh <地址> 'cat > /Library/LaunchDaemons/ai.openclaw.gateway.plist << "PLIST"
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.gateway</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/openclaw</string>
        <string>gateway</string>
        <string>run</string>
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
        <string>/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/opt/homebrew/bin</string>
    </dict>
</dict>
</plist>
PLIST'
```

加载服务：
```bash
ssh <地址> "launchctl load /Library/LaunchDaemons/ai.openclaw.gateway.plist"
```

重启服务：
```bash
ssh <地址> "launchctl unload /Library/LaunchDaemons/ai.openclaw.gateway.plist && sleep 2 && launchctl load /Library/LaunchDaemons/ai.openclaw.gateway.plist"
```

> ⚠️ macOS SSH 环境下 `nohup` 和 `screen -dm` 均无法可靠启动 gateway，必须用 LaunchDaemon。

#### macOS - 本地 GUI 环境

```bash
openclaw gateway install
```

#### Linux/WSL - systemd

```bash
ssh <地址> "openclaw gateway install"
ssh <地址> "openclaw gateway start"
```

或手动创建 systemd 服务：
```bash
ssh <地址> 'mkdir -p ~/.config/systemd/user && cat > ~/.config/systemd/user/openclaw-gateway.service << "SERVICE"
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/env openclaw gateway run --port 18789
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
SERVICE'
ssh <地址> "systemctl --user daemon-reload && systemctl --user enable --now openclaw-gateway.service"
```

### 阶段7：验证

```bash
ssh <地址> "openclaw channels status"
```

预期输出：
```
Gateway reachable.
- Telegram default: enabled, configured, running, mode:polling, token:config
```

### 阶段8：用户配对

用户首次在 Telegram 发消息时会收到配对码，批准：
```bash
ssh <地址> "openclaw pairing approve telegram <PAIRING_CODE>"
```

## 常见问题与解决方案

### 模型报错 "Unknown model: anthropic/xxx"
- **原因**：`agents.defaults.model` 格式错误
- **解决**：必须用 `provider/model-id`（斜杠），不是 `provider:model-id`（冒号）
```bash
openclaw config set agents.defaults.model "bailian/qwen3.5-plus"
```

### "Unrecognized key" 配置错误
- **原因**：使用了不存在的配置键
- **解决**：`openclaw doctor --fix` 自动清理无效键

### macOS SSH 下 gateway 无法启动
- **现象**：`openclaw gateway install` 报 `Bootstrap failed: Domain does not support specified action`
- **原因**：LaunchAgent 需要 GUI session，SSH 环境不可用
- **解决**：创建 LaunchDaemon（见阶段6）

### macOS SSH 下 nohup 失败
- **现象**：`nohup: can't detach from console: Inappropriate ioctl for device`
- **解决**：不要用 nohup/screen，用 LaunchDaemon

### Telegram "Unknown channel"
- **原因**：telegram 插件默认 disabled
- **解决**：先 `openclaw plugins enable telegram`，再 `channels add`

### Telegram 群消息被丢弃
- **原因**：默认 groupPolicy 是 allowlist
- **解决**：`openclaw config set channels.telegram.groupPolicy open`

### Node.js 版本不足
- **现象**：安装失败或运行报错
- **解决**：macOS 用 `brew install node@22`，Linux 用 NodeSource

### HTTP 401 API 认证失败
- **现象**：`HTTP 401 authentication_error: Invalid bearer token`
- **原因**：第三方 API 需要配置自定义 provider
- **解决**：在 `models.providers` 中配置自定义供应商

### Gateway 端口被占用
- **解决**：`openclaw gateway stop` 或 `pkill -f openclaw`

## 访问地址

| 服务 | 地址 | 说明 |
|------|------|------|
| Dashboard | `http://127.0.0.1:18789/?token=<token>` | 主控制台 |
| Gateway WS | `ws://127.0.0.1:18789` | WebSocket 连接 |
| Browser Control | `http://127.0.0.1:18791` | 需要 Bearer token |

## 配置 Nginx 反向代理（可选，Linux）

```bash
ssh <地址> "apt install nginx -y"
ssh <地址> "cat > /etc/nginx/sites-available/openclaw << 'EOF'
server {
    listen 8080;
    server_name localhost;
    location / {
        proxy_pass http://127.0.0.1:18791;
        proxy_set_header Authorization \"Bearer <token>\";
        proxy_set_header Host \$host;
    }
}
EOF"
ssh <地址> "ln -sf /etc/nginx/sites-available/openclaw /etc/nginx/sites-enabled/"
ssh <地址> "nginx -t && systemctl reload nginx"
```

## 参考文档

- GitHub：https://github.com/openclaw/openclaw
- 官方文档：https://docs.openclaw.ai/cli
