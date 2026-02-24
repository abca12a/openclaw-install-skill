---
name: openclaw-install
description: 通过 SSH 远程在 Windows WSL 上安装和配置 openclaw-cn 项目
invocation: user
---

# OpenClaw-CN 安装技能

通过 SSH 远程访问 Windows 下的 WSL，安装 openclaw-cn 项目。

## 使用方式

```
/openclaw-install <ssh地址>
```

示例：
```
/openclaw-install root@pengyou.tail39ed0c.ts.net
```

## 安装流程

### 阶段1：环境准备

1. **检查系统信息**
   ```bash
   ssh <地址> "uname -a && cat /etc/os-release"
   ```

2. **检查 Node.js 版本**（要求 >= 22）
   ```bash
   ssh <地址> "node --version"
   ```

3. **升级 Node.js**（如版本不足）
   ```bash
   ssh <地址> "curl -fsSL https://deb.nodesource.com/setup_22.x | bash -"
   ssh <地址> "apt install nodejs -y"
   ```

### 阶段2：安装 openclaw-cn

```bash
ssh <地址> "npm install -g openclaw-cn@latest"
```

### 阶段3：配置

1. **运行 doctor 修复配置**
   ```bash
   ssh <地址> "openclaw-cn doctor --fix"
   ```

2. **设置 gateway mode**
   ```bash
   ssh <地址> "openclaw-cn config set gateway.mode local"
   ```

3. **设置 auth token**
   ```bash
   ssh <地址> "openclaw-cn config set gateway.auth.token <token>"
   ```

4. **修复权限**
   ```bash
   ssh <地址> "chmod 700 ~/.openclaw && chmod 600 ~/.openclaw/openclaw.json"
   ```

### 阶段4：启动 Gateway

```bash
ssh <地址> "nohup openclaw-cn gateway --port 18789 --bind lan > /tmp/openclaw-gateway.log 2>&1 &"
```

### 阶段5：配置 Nginx 反向代理（可选）

浏览器无法直接添加 Authorization header，需要 nginx 代理：

```bash
# 安装 nginx
ssh <地址> "apt install nginx -y"

# 创建配置
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

# 启用并重载
ssh <地址> "ln -sf /etc/nginx/sites-available/openclaw /etc/nginx/sites-enabled/"
ssh <地址> "nginx -t && systemctl reload nginx"
```

## 常见问题与解决方案

### 问题1：Node.js 版本不足
- **现象**：安装失败或运行报错
- **原因**：openclaw-cn 要求 Node.js >= 22
- **解决**：使用 NodeSource 仓库安装 Node.js 22

### 问题2：配置格式过时
- **现象**：`agent.model` 配置无效
- **原因**：新版本使用 `agents.defaults.model.primary`
- **解决**：运行 `openclaw-cn doctor --fix` 自动迁移

### 问题3：Gateway 启动失败 - 缺少 mode
- **现象**：`gateway.mode is unset; gateway start will be blocked`
- **解决**：`openclaw-cn config set gateway.mode local`

### 问题4：Gateway 启动失败 - 缺少 token
- **现象**：`Gateway auth is set to token, but no token is configured`
- **解决**：`openclaw-cn config set gateway.auth.token <随机token>`

### 问题5：浏览器访问 Unauthorized
- **现象**：访问 http://127.0.0.1:18791 显示 Unauthorized
- **原因**：需要 `Authorization: Bearer <token>` header
- **解决**：安装 nginx 反向代理自动添加 header

### 问题6：onboard 向导无法交互
- **现象**：SSH 运行 `openclaw-cn onboard` 卡住
- **原因**：SSH 无 TTY，无法交互
- **解决**：手动创建配置文件或运行 `doctor --fix`

## 访问地址

| 服务 | 地址 | 说明 |
|------|------|------|
| Dashboard | `http://127.0.0.1:18789/?token=<token>` | 主控制台 |
| Gateway WS | `ws://0.0.0.0:18789` | WebSocket 连接 |
| Browser Control | `http://127.0.0.1:18791` | 需要 Bearer token |
| Browser Control (代理) | `http://127.0.0.1:8080` | 通过 nginx 免认证 |

## 后续配置（待完善）

- [ ] Telegram 机器人配置
- [ ] 飞书机器人配置
- [ ] AI API 配置（Anthropic/OpenAI）
- [ ] 安全审计配置

## 版本信息

- 当前版本：0.1.6
- 最新版本检查：`openclaw-cn update`
- 文档：https://docs.clawd.bot/cli

---

## 第三方 API 配置（重要）

### 问题7：API 认证失败 - HTTP 401

- **现象**：`HTTP 401 authentication_error: Invalid bearer token`
- **原因**：使用第三方中转 Claude API 时，默认的 `anthropic` 供应商无法识别自定义 baseUrl
- **解决**：需要配置自定义供应商

#### 配置步骤

**1. 设置 API Key 环境变量**
```bash
openclaw-cn config set env.CUSTOM_ANTHROPIC_KEY "sk-ant-xxx..."
```

**2. 配置自定义供应商**
```bash
openclaw-cn config set models.providers.custom-anthropic '{"baseUrl":"https://your-api-proxy.com/v1","apiKey":"${CUSTOM_ANTHROPIC_KEY}","api":"anthropic-messages","models":[{"id":"claude-opus-4-6","name":"Claude Opus 4.6"}]}'
```

**3. 设置默认模型**
```bash
openclaw-cn config set agents.defaults.model.primary "custom-anthropic/claude-opus-4-6"
```

**4. 重启网关**
```bash
systemctl --user restart openclaw-gateway.service
```

**5. 验证配置**
```bash
openclaw-cn models status
```

#### 配置说明

| 字段 | 说明 |
|------|------|
| `baseUrl` | 第三方 API 地址 |
| `apiKey` | 使用 `${ENV_VAR}` 引用环境变量 |
| `api` | 协议类型：`anthropic-messages` 或 `openai-completions` |
| `models[].id` | 模型 ID（不含供应商前缀） |


### 问题8：Telegram 渠道配置

#### 添加 Telegram Bot

```bash
openclaw-cn channels add telegram --token '<BOT_TOKEN>'
```

#### 用户配对

当用户首次使用 bot 时会收到配对请求：
```
配对码：XXXXXXXX
请让机器人所有者执行：
openclaw-cn pairing approve telegram XXXXXXXX
```

**批准配对：**
```bash
openclaw-cn pairing approve telegram <配对码>
```

#### 检查渠道状态

```bash
openclaw-cn channels status
```

正常输出：
```
- Telegram default: enabled, configured, 运行中, mode:polling, token:config
```


### 问题9：Gateway 进程管理

#### 创建 systemd 服务（推荐）

```bash
cat > ~/.config/systemd/user/openclaw-gateway.service << 'SERVICE'
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
Environment=ANTHROPIC_API_KEY=sk-xxx
Environment=ANTHROPIC_BASE_URL=https://your-proxy.com/v1
ExecStart=/usr/bin/env openclaw-cn gateway --port 18789
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
SERVICE
```

#### 启用服务

```bash
systemctl --user daemon-reload
systemctl --user enable openclaw-gateway.service
systemctl --user start openclaw-gateway.service
```

#### 常用命令

```bash
# 查看状态
systemctl --user status openclaw-gateway.service

# 重启
systemctl --user restart openclaw-gateway.service

# 查看日志
journalctl --user -u openclaw-gateway.service -f
```


### 问题10：认证配置文件不生效

- **现象**：手动创建 `auth-profiles.json` 但 `models status` 显示 `Missing auth`
- **原因**：openclaw-cn 不直接读取手动创建的认证文件，需要通过配置供应商方式
- **解决**：使用 `models.providers` 配置自定义供应商（见问题7）

### 问题11：Gateway 端口被占用

- **现象**：`Port 18789 is already in use`
- **解决**：
  ```bash
  # 停止现有服务
  openclaw-cn gateway stop
  # 或强制终止
  pkill -f openclaw
  ```

### 问题12：配置验证失败

- **现象**：`Config validation failed: Unrecognized key`
- **原因**：使用了不存在的配置键
- **解决**：查看官方文档确认正确的配置路径

## 完整配置示例

```json
{
  "env": {
    "CUSTOM_ANTHROPIC_KEY": "sk-ant-xxx..."
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "custom-anthropic/claude-opus-4-6"
      }
    }
  },
  "models": {
    "providers": {
      "custom-anthropic": {
        "baseUrl": "https://your-proxy.com/v1",
        "apiKey": "${CUSTOM_ANTHROPIC_KEY}",
        "api": "anthropic-messages",
        "models": [
          {"id": "claude-opus-4-6", "name": "Claude Opus 4.6"}
        ]
      }
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "123456:ABC..."
    }
  }
}
```

## 参考文档

- 官方文档：https://docs.openclaw.ai/cli
- 自定义供应商：https://clawd.org.cn/guides/custom-ai-providers.html
- GitHub：https://github.com/jiulingyun/openclaw-cn
