# Multi-Instance Telegram Bot Setup

This fork adds multi-instance support to [linuz90/claude-telegram-bot](https://github.com/linuz90/claude-telegram-bot). Three bots run from a single codebase, each with its own Telegram token, Claude model, working directory, and memory scope — isolated by one environment variable: `BOT_ID`.

## Architecture

```
Single codebase (~/jayef_bots/)
  ├── BOT_ID=general  → Haiku model,  ~/assistant/,         general memory
  ├── BOT_ID=oto      → Sonnet model, ~/repos/oto/,         oto-planner memory
  └── BOT_ID=youtube  → Opus model,   ~/repos/youtube-channel/, youtube-planner memory
```

Each instance gets its own:
- Telegram bot token
- Claude model
- Working directory (with its own CLAUDE.md)
- MCP config (memory scope, tools)
- Session/restart/audit files in `/tmp/` (namespaced by BOT_ID)
- launchd service and log files

## Prerequisites

- **macOS** (launchd for process management)
- **Bun 1.0+** — [Install Bun](https://bun.sh/)
- **Claude Code CLI** — Run `claude` once to authenticate
- **Telegram Bot Tokens** — Create bots via [@BotFather](https://t.me/BotFather)
- **Your Telegram User ID** — Message [@userinfobot](https://t.me/userinfobot)

## Setup Steps

### 1. Clone and install

```bash
git clone https://github.com/JayefBuild/claude-telegram-bot.git ~/jayef_bots
cd ~/jayef_bots
bun install
```

### 2. Create Telegram bots

Message [@BotFather](https://t.me/BotFather) three times to create three bots. For each, send `/setcommands` and paste:

```
start - Show status and user ID
new - Start a fresh session
resume - Resume from a past session
stop - Interrupt current query
status - Check what Claude is doing
restart - Restart the bot
retry - Retry last message
```

Save the three tokens.

### 3. Configure instances

Edit the env files in `instances/` with your real values:

```bash
# instances/general.env
TELEGRAM_BOT_TOKEN=1234567890:ABC-DEF...
TELEGRAM_ALLOWED_USERS=your_telegram_id
CLAUDE_WORKING_DIR=/Users/you/assistant
CLAUDE_MODEL=claude-haiku-4-5
RATE_LIMIT_REQUESTS=40

# instances/oto.env
TELEGRAM_BOT_TOKEN=0987654321:GHI-JKL...
TELEGRAM_ALLOWED_USERS=your_telegram_id
CLAUDE_WORKING_DIR=/Users/you/repos/oto
CLAUDE_MODEL=claude-sonnet-4-5
RATE_LIMIT_REQUESTS=20

# instances/youtube.env
TELEGRAM_BOT_TOKEN=1122334455:MNO-PQR...
TELEGRAM_ALLOWED_USERS=your_telegram_id
CLAUDE_WORKING_DIR=/Users/you/repos/youtube-channel
CLAUDE_MODEL=claude-opus-4-6
RATE_LIMIT_REQUESTS=20
```

### 4. Configure MCP servers

Each bot has its own MCP config: `mcp-config.general.ts`, `mcp-config.oto.ts`, `mcp-config.youtube.ts`. Edit these to add tools specific to each bot (memory scopes, APIs, etc.).

The `ask-user` MCP server is included by default and provides interactive inline buttons in Telegram. Uncomment the `mem0` block once your memory stack is running.

### 5. Create working directories and CLAUDE.md files

```bash
mkdir -p ~/assistant ~/repos/oto ~/repos/youtube-channel
```

Each directory should have a `CLAUDE.md` with identity and context for that bot.

### 6. Test locally

Run each bot in a separate terminal to verify everything works:

```bash
BOT_ID=general bun run src/index.ts
BOT_ID=oto bun run src/index.ts
BOT_ID=youtube bun run src/index.ts
```

You should see output like:

```
Loaded 1 MCP servers from mcp-config.general.ts
Config loaded [general]: 1 allowed users, model: claude-haiku-4-5, working dir: /Users/you/assistant
==================================================
Claude Bot [general]
==================================================
```

Verify in `/tmp/` that files are namespaced:

```bash
ls /tmp/claude-bot-*
# claude-bot-general/  claude-bot-general-session.json  claude-bot-general-audit.log
# claude-bot-oto/      claude-bot-oto-session.json      ...
```

### 7. Deploy as launchd services

The launchd plists are in `launchagent/`. Before loading, update the paths in each plist if your repo isn't at `~/jayef_bots/`.

Symlink into LaunchAgents and bootstrap:

```bash
ln -s ~/jayef_bots/launchagent/com.agent.general.plist ~/Library/LaunchAgents/
ln -s ~/jayef_bots/launchagent/com.agent.oto.plist ~/Library/LaunchAgents/
ln -s ~/jayef_bots/launchagent/com.agent.youtube.plist ~/Library/LaunchAgents/

launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.agent.general.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.agent.oto.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.agent.youtube.plist
```

Verify they're running:

```bash
launchctl list | grep com.agent
```

Prevent your Mac from sleeping: **System Settings → Battery → Options → "Prevent automatic sleeping when the display is off"**.

### 8. Add shell aliases

Add these to `~/.zshrc` on the Mac Mini for easy management:

```bash
# ============== Telegram Bot Management ==============

# Status of all bots
alias bots='launchctl list | grep com.agent'

# General bot
alias gbot-restart='launchctl kickstart -k gui/$(id -u)/com.agent.general && echo "Restarted"'
alias gbot-stop='launchctl bootout gui/$(id -u)/com.agent.general 2>/dev/null && echo "Stopped"'
alias gbot-start='launchctl bootstrap gui/$(id -u) ~/jayef_bots/launchagent/com.agent.general.plist && echo "Started"'
alias gbot-logs='tail -f /tmp/claude-bot-general.log'

# Oto bot
alias obot-restart='launchctl kickstart -k gui/$(id -u)/com.agent.oto && echo "Restarted"'
alias obot-stop='launchctl bootout gui/$(id -u)/com.agent.oto 2>/dev/null && echo "Stopped"'
alias obot-start='launchctl bootstrap gui/$(id -u) ~/jayef_bots/launchagent/com.agent.oto.plist && echo "Started"'
alias obot-logs='tail -f /tmp/claude-bot-oto.log'

# YouTube bot
alias ybot-restart='launchctl kickstart -k gui/$(id -u)/com.agent.youtube && echo "Restarted"'
alias ybot-stop='launchctl bootout gui/$(id -u)/com.agent.youtube 2>/dev/null && echo "Stopped"'
alias ybot-start='launchctl bootstrap gui/$(id -u) ~/jayef_bots/launchagent/com.agent.youtube.plist && echo "Started"'
alias ybot-logs='tail -f /tmp/claude-bot-youtube.log'

# Restart all
alias bots-restart='gbot-restart; obot-restart; ybot-restart'
```

## Alias Reference

| Alias | What it does |
|-------|-------------|
| `bots` | List status of all three bot services |
| `gbot-start` | Register and start the general bot service |
| `gbot-stop` | Unregister the service (bot stops, won't auto-restart) |
| `gbot-restart` | Kill and relaunch the bot (launchd restarts it immediately) |
| `gbot-logs` | Tail the general bot's log file in real-time |
| `obot-*` | Same commands for the Oto bot |
| `ybot-*` | Same commands for the YouTube bot |
| `bots-restart` | Restart all three bots at once |

## Updating

```bash
cd ~/jayef_bots
git pull
bots-restart    # All three bots pick up the new code
```

To pull upstream improvements:

```bash
git remote add upstream https://github.com/linuz90/claude-telegram-bot.git
git fetch upstream
git merge upstream/main
```

## Adding a New Bot

1. Create `instances/new-bot.env` with token, model, working dir
2. Create `mcp-config.new-bot.ts` with the right memory scope
3. Write a `CLAUDE.md` in the working directory
4. Create the bot with @BotFather
5. Copy an existing plist to `launchagent/com.agent.new-bot.plist`, update `Label`, `BOT_ID`, and log paths
6. Symlink and bootstrap:
   ```bash
   ln -s ~/jayef_bots/launchagent/com.agent.new-bot.plist ~/Library/LaunchAgents/
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.agent.new-bot.plist
   ```

No code changes needed — just config files.

## File Layout

```
~/jayef_bots/
  src/                          ← Bot source code
  ask_user_mcp/                 ← Interactive buttons MCP server
  instances/
    general.env                 ← Tokens, model, working dir (gitignored)
    oto.env
    youtube.env
  mcp-config.general.ts         ← MCP servers per bot (gitignored)
  mcp-config.oto.ts
  mcp-config.youtube.ts
  launchagent/
    com.agent.general.plist     ← launchd service definitions
    com.agent.oto.plist
    com.agent.youtube.plist
```

Runtime files (all namespaced by BOT_ID):

```
/tmp/claude-bot-general/            ← Temp directory
/tmp/claude-bot-general-session.json ← Session persistence
/tmp/claude-bot-general-audit.log    ← Audit log
/tmp/claude-bot-general.log          ← stdout (from launchd)
/tmp/claude-bot-general.err          ← stderr (from launchd)
/tmp/ask-user-general-*.json         ← Interactive button requests
```
