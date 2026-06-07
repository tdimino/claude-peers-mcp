# claude-peers

Let your Claude Code instances find each other and talk. When you're running 5 sessions across different projects, any Claude can discover the others and send messages that arrive instantly.

```
  Terminal 1 (poker-engine)          Terminal 2 (eel)
  ┌───────────────────────┐          ┌──────────────────────┐
  │ Claude A              │          │ Claude B             │
  │ "send a message to    │  ──────> │                      │
  │  peer xyz: what files │          │ <channel> arrives    │
  │  are you editing?"    │  <────── │  instantly, Claude B │
  │                       │          │  responds            │
  └───────────────────────┘          └──────────────────────┘
```

## Quick start

### 1. Install

```bash
git clone https://github.com/louislva/claude-peers-mcp.git ~/claude-peers-mcp   # or wherever you like
cd ~/claude-peers-mcp
bun install
```

### 2. Register the MCP server

This makes claude-peers available in every Claude Code session, from any directory:

```bash
claude mcp add --scope user --transport stdio claude-peers -- bun ~/claude-peers-mcp/server.ts
```

Replace `~/claude-peers-mcp` with wherever you cloned it.

### 3. Run Claude Code with the channel

```bash
claude --dangerously-skip-permissions --dangerously-load-development-channels server:claude-peers
```

That's it. The broker daemon starts automatically the first time.

> **Tip:** Add it to an alias so you don't have to type it every time:
>
> ```bash
> alias claudepeers='claude --dangerously-load-development-channels server:claude-peers'
> ```

### 4. Open a second session and try it

In another terminal, start Claude Code the same way. Then ask either one:

> List all peers on this machine

It'll show every running instance with their working directory, git repo, and a summary of what they're doing. Then:

> Send a message to peer [id]: "what are you working on?"

The other Claude receives it immediately and responds.

## What Claude can do

| Tool             | What it does                                                                   |
| ---------------- | ------------------------------------------------------------------------------ |
| `list_peers`     | Find other Claude Code instances — scoped to `machine`, `directory`, or `repo` |
| `send_message`   | Send a message to another instance by ID (arrives instantly via channel push)  |
| `set_summary`    | Describe what you're working on (visible to other peers)                       |
| `check_messages` | Check message history — shows all recent messages with `[NEW]` markers, client-type tags, and timestamps |
| `kill_peer`      | Forcibly terminate an unresponsive peer's agent session                        |

## How it works

A **broker daemon** runs on a Unix domain socket (`~/.claude/run/claude-peers.sock`) backed by SQLite. Each agent session (Claude Code or Codex CLI) spawns an MCP server that registers with the broker. Claude Code peers get messages pushed instantly via [claude/channel](https://code.claude.com/docs/en/channels-reference); Codex peers poll via `check_messages`.

```
                    ┌───────────────────────────┐
                    │  broker daemon            │
                    │  Unix socket + SQLite     │
                    └──────┬───────────────┬────┘
                           │               │
                      MCP server A    MCP server B
                      (stdio)         (stdio)
                           │               │
                      Claude A         Codex B
```

The broker auto-launches when the first session starts. It cleans up dead and orphaned peers automatically (PPID-aware detection every 30s). MCP servers self-terminate when their parent agent exits. Everything is localhost-only.

### Message history

`check_messages` returns all recent messages (sent and received) with metadata:

```
3 message(s) (1 new):

[NEW] [codex] From abc123 (Thera-Knossos-Minos-Paper) — 2026-06-06T15:43:56Z
  Hey — Tom mentioned you created a translation resource...

[claude-code] From def456 (Programming) — 2026-06-06T15:37:33Z
  Goal 65 review handoff: The full-corpus final...

[codex] To abc123 (Thera-Knossos-Minos-Paper) — 2026-06-06T15:37:33Z
  Goal 65 review handoff: The full-corpus final...
```

Messages persist across reads — calling `check_messages` multiple times still shows history.

### Orphan detection

MCP server processes detect parent death via two mechanisms:
- **stdin close** — immediate detection when the parent agent exits
- **PPID monitoring** — 30-second fallback checks if the process was reparented to init/launchd

## Auto-summary

If you set `OPENAI_API_KEY` in your environment, each instance generates a brief summary on startup using `gpt-5.4-nano` (costs fractions of a cent). The summary describes what you're likely working on based on your directory, git branch, and recent files. Other instances see this when they call `list_peers`.

Without the API key, Claude sets its own summary via the `set_summary` tool.

## CLI

You can also inspect and interact from the command line:

```bash
cd ~/claude-peers-mcp

bun cli.ts status            # broker status + all peers (with orphan warnings)
bun cli.ts peers             # list peers
bun cli.ts orphans           # list orphaned server.ts processes (PPID=1)
bun cli.ts cleanup           # kill orphaned processes and remove from broker
bun cli.ts send <id> <msg>   # send a message into a Claude session
bun cli.ts kill <id>         # kill a peer's agent session
bun cli.ts kill-broker       # stop the broker
```

## Configuration

| Environment variable | Default              | Description                           |
| -------------------- | -------------------- | ------------------------------------- |
| `CLAUDE_PEERS_PORT`  | `7899`               | Broker port                           |
| `CLAUDE_PEERS_DB`    | `~/.claude-peers.db` | SQLite database path                  |
| `OPENAI_API_KEY`     | —                    | Enables auto-summary via gpt-5.4-nano |

## Requirements

- [Bun](https://bun.sh)
- Claude Code v2.1.80+
- claude.ai login (channels require it — API key auth won't work)
