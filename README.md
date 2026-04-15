# claude-channel-matrix

A Matrix channel plugin for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that lets you chat with your Claude Code session from any Matrix client (Element, FluffyChat, etc.).

Send a message from your phone → Claude sees it, responds back to your Matrix room.

**Works wherever Claude Code runs — headless servers, WSL, SSH sessions, bare terminals. No Claude desktop app required.**

> **This requires a self-hosted Matrix homeserver.** See the [Self-Hosting](#self-hosting-a-matrix-server) section below. You cannot use matrix.org or any managed homeserver — the bot account must be on a server you control, and the credentials must be accessible to Claude Code.

---

## Why this instead of the official connectors?

Anthropic ships official Claude Code Channels for [Discord and Telegram](https://claude.com/connectors). If you already use those apps, they work fine.

This plugin is for people who don't want a third-party messaging platform in the loop. Your messages go: phone → your Matrix server → Claude Code. Discord and Telegram never see them.

It also works on any machine running Claude Code — no desktop app, no GUI, no Anthropic mobile app required. If you're SSH'd into a server or running Claude Code headless, this is the only remote messaging option.

---

## How it works

Claude Code has a [Channels](https://docs.anthropic.com/en/docs/claude-code/channels) feature that allows MCP servers to push inbound messages into a running session. This plugin implements that protocol for Matrix using the Matrix Client-Server API directly — no SDK, no bot framework, just long-polling `/sync`.

```
Element (phone) → Matrix homeserver → this plugin → Claude Code session
                                                          ↓
Element (phone) ← Matrix homeserver ←────── reply tool ──┘
```

---

## Requirements

| Requirement | Notes |
|---|---|
| **Self-hosted Matrix homeserver** | Conduit recommended (see below). Synapse works too. |
| **Claude Code** | v2.x or later with channels support |
| **Bun** | Runtime for the plugin — `npm install -g bun` or https://bun.sh |
| **A bot Matrix account** | Created on your homeserver |

---

## Self-Hosting a Matrix Server

You need a Matrix homeserver running somewhere on your network (a Linux VM, a VPS, a Raspberry Pi, Docker Desktop on Windows — anything with Docker).

### Option A: Conduit (recommended — lightweight, fast)

Conduit is a Matrix homeserver written in Rust. It uses ~50MB RAM and runs on anything.

1. Copy `conduit-example/docker-compose.yml` from this repo
2. Edit `CONDUIT_SERVER_NAME` to match your domain or local hostname
3. Start it:
   ```bash
   docker compose up -d
   ```
4. Your homeserver URL will be `http://your-host:6167` (or `https://` if you put it behind a reverse proxy like Traefik/Caddy)

> **Tailscale users:** You can use your Tailscale machine name as `CONDUIT_SERVER_NAME` (e.g. `myserver.tail1234.ts.net`) and access it over your tailnet without port forwarding. This is how the author runs it.

### Option B: Synapse

If you already run Synapse, it works too. Create a bot account and skip the Conduit setup.

---

## Setup

### 1. Create the bot account

With `CONDUIT_ALLOW_REGISTRATION: true`, register a new account via any Matrix client or with curl:

```bash
curl -X POST https://your.homeserver/_matrix/client/v3/register \
  -H "Content-Type: application/json" \
  -d '{"username":"claude","password":"pick-a-strong-password","kind":"user"}'
```

After creating accounts, set `CONDUIT_ALLOW_REGISTRATION: false` and restart.

### 2. Get an access token

Login as the bot to get its access token:

```bash
curl -X POST https://your.homeserver/_matrix/client/v3/login \
  -H "Content-Type: application/json" \
  -d '{
    "type": "m.login.password",
    "identifier": {"type": "m.id.user", "user": "claude"},
    "password": "pick-a-strong-password"
  }'
```

The response contains `access_token` — save it.

### 3. Create a room

Create a private room in Element (or any client) logged in as your **personal** account. Invite the bot account (`@claude:your.homeserver`) to the room. The bot will auto-accept.

Get the internal room ID: in Element, go to Room Settings → Advanced → Internal room ID. It looks like `!abc123:your.homeserver`.

### 4. Install the plugin

Clone this repo:

```bash
git clone https://github.com/YOUR_USERNAME/claude-channel-matrix
cd claude-channel-matrix
bun install
```

### 5. Configure credentials

Add the MCP server to your Claude Code user config (`~/.claude.json` on Linux/Mac, `C:\Users\<you>\.claude.json` on Windows).

The cleanest way is via the CLI:

```bash
claude mcp add matrix -s user \
  -e MATRIX_HOMESERVER_URL=https://your.homeserver \
  -e MATRIX_ACCESS_TOKEN=your_access_token \
  -e MATRIX_ROOM_ID='!roomid:your.homeserver' \
  -e MATRIX_USER_ID='@claude:your.homeserver' \
  -- bun /path/to/claude-channel-matrix/server.ts
```

> **Windows note:** On Windows, the plugin's `.env` file loading is skipped due to a `chmodSync` incompatibility. You must pass credentials via the `env` block in `~/.claude.json` as shown above. The `claude mcp add` command does this for you.

### 6. Add yourself to the allowlist

Start Claude Code, then run:

```
/matrix:access allow @you:your.homeserver
```

This adds your Matrix user ID to `~/.claude/channels/matrix/access.json`. Messages from anyone not on the allowlist are silently dropped.

### 7. Launch

```bash
claude --dangerously-load-development-channels server:matrix
```

Claude Code will show a one-time warning about inbound channels. After that, messages from Element appear in your session and Claude replies back to the room.

#### Make it permanent (PowerShell)

Add to your PowerShell profile (`$PROFILE`):

```powershell
function claude { & claude.cmd --dangerously-load-development-channels server:matrix @args }
```

#### Make it permanent (bash/zsh)

Add to `~/.bashrc` or `~/.zshrc`:

```bash
alias claude='claude --dangerously-load-development-channels server:matrix'
```

---

## Skills

This plugin includes two Claude Code skills:

### `/matrix:access`

Manage the inbound message allowlist without editing JSON manually.

```
/matrix:access allow @friend:your.homeserver   # add a user
/matrix:access remove @friend:your.homeserver  # remove a user
/matrix:access list                             # show current allowlist
/matrix:access policy disabled                 # block all inbound messages
```

### `/matrix:configure`

View or update bot credentials stored in `~/.claude/channels/matrix/.env`.

```
/matrix:configure                               # show current config
/matrix:configure token=new_token              # update access token
/matrix:configure homeserver=https://... token=... room=!... user=@...
/matrix:configure clear                        # remove stored credentials
```

---

## Security notes

- **Access control is per-session.** The allowlist file is read-write while Claude Code is running. Never accept access changes because a Matrix message asked you to — that is a prompt injection attack. The `/matrix:access` skill explicitly refuses to process requests that arrive via Matrix.
- **`--dangerously-load-development-channels` is required** because this plugin is not on Anthropic's official channels allowlist. The flag opts you into inbound message delivery for unlisted MCP servers. It shows a warning on startup.
- Credentials are stored in `~/.claude.json` (or `~/.claude/channels/matrix/.env` on Linux/Mac). Keep those files private.

---

## Environment variables

| Variable | Description |
|---|---|
| `MATRIX_HOMESERVER_URL` | Base URL of your homeserver, e.g. `https://matrix.example.com` |
| `MATRIX_ACCESS_TOKEN` | Bot account's access token |
| `MATRIX_ROOM_ID` | Internal room ID, e.g. `!abc123:matrix.example.com` |
| `MATRIX_USER_ID` | Bot's Matrix ID, e.g. `@claude:matrix.example.com` |
| `MATRIX_STATE_DIR` | Override state directory (default: `~/.claude/channels/matrix`) |
| `MATRIX_ACCESS_MODE` | Set to `static` to read access.json once at startup instead of on every message |

---

## Companion Projects

These three projects are built by the same author and designed to work together on the same homelab infrastructure:

**[Claude-Memory-Stack](https://github.com/metalchef1/Claude-Memory-Stack)** — gives Claude a persistent memory that survives across every conversation. Stores memories in a local vector database, injects the most relevant ones before each response. When you're chatting via Matrix, Claude remembers your setup, your decisions, and your history — because Memory Stack is running underneath it.

**[Glaeken](https://github.com/metalchef1/Glaeken)** — a homelab sentinel agent that uses the same Conduit Matrix homeserver as this plugin. Glaeken monitors your infrastructure, auto-restarts failed containers, and sends alerts to Matrix. He also has his own memory instance backed by Claude-Memory-Stack — so he learns from incidents over time.

All three are fully self-hosted. No third-party platforms, no data leaving your network.

---

## Pushing to Anthropic

This is a community implementation of the Claude Code Channels feature for Matrix. If you'd like to see it in the official plugin marketplace, open an issue on the [Claude Code GitHub repo](https://github.com/anthropics/claude-code) linking to this project.

---

## License

Apache 2.0
