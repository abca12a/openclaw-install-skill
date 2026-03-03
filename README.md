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
- macOS and Linux/WSL cross-platform support
- Automatic platform detection (Darwin/Linux, HOME path, PATH)
- Node.js version check and upgrade
- Model provider configuration (Alibaba Bailian, SiliconFlow, etc.)
- Agent default model setup
- Telegram / Feishu channel configuration
- macOS LaunchDaemon for SSH headless gateway deployment
- Linux systemd gateway service
- User pairing workflow
- Automatic `doctor --fix` for config validation
- Cross-platform path handling (macOS `/var/root` vs Linux `/root`)

## Key Lessons Encoded

This skill encodes hard-won lessons from real deployments:

1. **macOS root HOME is `/var/root`**, not `/root` - workspace and all config paths must adapt
2. **npm global bin may not be in SSH PATH** - always detect prefix and prepend to PATH
3. **`nohup` fails via SSH on macOS** - use LaunchDaemon instead
4. **`openclaw gateway install` fails in SSH** - needs GUI domain on macOS, use LaunchDaemon
5. **Cross-platform config upload** - must fix `workspace` and `browser.executablePath` after scp
6. **SSH quoting matters** - use single quotes to prevent local PATH/variable leakage

## License

MIT
