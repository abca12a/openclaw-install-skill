# OpenClaw Install Skill

Claude Code skill for installing and configuring [openclaw](https://github.com/openclaw/openclaw) via SSH.

Supports macOS (arm64/x86_64) and Linux (WSL/native).

## Usage

```
/openclaw-install <ssh-address>
```

Example:
```
/openclaw-install root@your-server.ts.net
```

## Installation

Copy `SKILL.md` to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/openclaw-install
cp SKILL.md ~/.claude/skills/openclaw-install/
```

## Features

- Remote installation via SSH
- macOS and Linux/WSL support
- Node.js version check and upgrade
- Model provider configuration (Alibaba Bailian, Anthropic proxy, etc.)
- Agent default model setup
- Telegram/Discord/other channel plugin management
- macOS LaunchDaemon for SSH headless gateway deployment
- Linux systemd gateway service
- User pairing workflow
- Nginx reverse proxy setup (optional)
- Automatic `doctor --fix` for config validation

## License

MIT
