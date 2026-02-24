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
