---
summary: "Comprehensive overview of Moltbot architecture, capabilities, and how it works"
read_when:
  - Understanding how Moltbot works
  - Learning the architecture
  - Onboarding new developers
  - Comparing Moltbot to other AI agents
---
# Moltbot: Complete Overview

## TL;DR

> **Moltbot is an OS-level AI agent with remote channels and autonomy.**

- **OS-Level Agent**: Direct access to filesystem, shell, processes, browser (not screen control)
- **Remote Channels**: Reachable via WhatsApp, Telegram, Slack, Discord, Voice, Web, CLI
- **Autonomy**: Acts on its own via cron jobs, heartbeats, and webhooks
- **Self-Hosted**: Runs on your machine, data stays yours, you just rent the LLM brain

---

## Table of Contents

1. [What is Moltbot?](#what-is-moltbot)
2. [Architecture Overview](#architecture-overview)
3. [The Agent Runtime](#the-agent-runtime)
4. [Tools Available](#tools-available)
5. [Can It Build Software?](#can-it-build-software)
6. [Security Model](#security-model)
7. [Autonomous Capabilities](#autonomous-capabilities)
8. [How It Differs From Other Agents](#how-it-differs-from-other-agents)
9. [End-to-End Example](#end-to-end-example)
10. [Key Source Directories](#key-source-directories)

---

## What is Moltbot?

**Moltbot** is a personal AI assistant that:

1. **Runs on your own devices** (Mac, Linux, Raspberry Pi, VPS)
2. **Connects to messaging channels** (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, etc.)
3. **Has full OS access** (filesystem, shell, browser, network)
4. **Acts autonomously** (scheduled tasks, webhooks, proactive check-ins)
5. **Controls multiple devices** (Mac, iOS, Android nodes)

### The One-Liner

```
Moltbot = OS-Level Agent + Remote Channels + Autonomy + Multi-Device
```

### The Formula

```
Cloud Agent = Brain + Interface + Storage (all theirs)
Moltbot     = Brain (LLM API) + Interface (yours) + Storage (yours) + Actions (yours)
                    │
                    └── You just rent the thinking part
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            REMOTE CHANNELS                                  │
│    WhatsApp  Telegram  Slack  Discord  Signal  Voice  WebChat  CLI          │
│        │         │       │       │        │       │       │      │          │
│        └─────────┴───────┴───────┴────────┴───────┴───────┴──────┘          │
│                                    │                                        │
│                                    ▼                                        │
│                           ┌───────────────┐                                 │
│                           │    GATEWAY    │  ◄── Always running (daemon)    │
│                           │    (24/7)     │                                 │
│                           └───────┬───────┘                                 │
│                                   │                                         │
│                    ┌──────────────┼──────────────┐                          │
│                    ▼              ▼              ▼                          │
│              ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│              │  Agent   │  │  Tools   │  │ Sessions │                       │
│              │ Runtime  │  │ Registry │  │ Manager  │                       │
│              └──────────┘  └──────────┘  └──────────┘                       │
│                                   │                                         │
│                                   ▼                                         │
│                         OS-LEVEL ACCESS                                     │
│    ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│    │   File   │ │  Shell   │ │ Process  │ │ Browser  │ │  Nodes   │        │
│    │  System  │ │ Commands │ │   Mgmt   │ │   CDP    │ │ (Devices)│        │
│    └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Core Components

| Component | Location | Purpose |
|-----------|----------|---------|
| **Gateway** | `src/gateway/` | WebSocket server, message routing, RPC methods |
| **Agent Runtime** | `src/agents/` | LLM calls, tool execution, session management |
| **Channels** | `src/telegram/`, `src/discord/`, etc. | Platform-specific messaging integrations |
| **Tools** | `src/agents/tools/` | Browser, cron, message, nodes, web, etc. |
| **Sessions** | `src/config/sessions.ts` | Conversation state persistence |

---

## The Agent Runtime

Moltbot uses **`@mariozechner/pi-agent-core`** as the underlying agent runtime.

### Agent Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│  @mariozechner/pi-agent-core     ← Core agent loop + tool execution  │
├─────────────────────────────────────────────────────────────────────┤
│  @mariozechner/pi-coding-agent   ← Session management + coding tools │
├─────────────────────────────────────────────────────────────────────┤
│  @mariozechner/pi-ai             ← LLM provider abstraction          │
├─────────────────────────────────────────────────────────────────────┤
│  Moltbot Tools + Integration Layer                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### The Agent Loop

```
1. QUEUE REQUEST
   └─ Serialize per-session (prevent race conditions)

2. RESOLVE AUTH
   └─ Get API key from auth profiles (OAuth, API key, etc.)

3. BUILD SYSTEM PROMPT
   └─ Identity + Skills + Context Files + Runtime Info + Tool Summaries

4. LOAD SESSION
   └─ Read JSONL transcript (conversation history)

5. CALL LLM
   └─ Stream tokens, emit "assistant" events

6. HANDLE TOOL CALLS
   └─ If model requests tool → execute → return result → loop

7. FINALIZE
   └─ Save transcript, update session, emit "lifecycle:end"
```

### System Prompt Components

The system prompt is built dynamically and includes:

| Section | Content |
|---------|---------|
| **Identity** | "You are Moltbot, a personal AI assistant" |
| **Runtime Info** | Host, OS, model, channel, sandbox status |
| **Tool Summaries** | List of available tools with descriptions |
| **Skills** | Available skills from workspace |
| **Context Files** | AGENTS.md, SOUL.md, USER.md, TOOLS.md |
| **Messaging** | How to send messages across channels |
| **Memory** | How to use memory_search/memory_get |

### Context Files (Workspace)

| File | Purpose |
|------|---------|
| `AGENTS.md` | Workspace rules, conventions, behavior guidelines |
| `SOUL.md` | Personality, values, identity |
| `USER.md` | Information about the user |
| `TOOLS.md` | Local tool notes (SSH hosts, camera names, etc.) |
| `MEMORY.md` | Long-term curated memory |
| `memory/*.md` | Daily notes and logs |

---

## Tools Available

### Core Tools (from pi-coding-agent)

| Tool | Description |
|------|-------------|
| `read` | Read file contents |
| `write` | Create or overwrite files |
| `edit` | Make precise edits to files |
| `apply_patch` | Apply multi-file patches |
| `grep` | Search file contents |
| `find` | Find files by pattern |
| `ls` | List directory contents |
| `exec` | Run shell commands (with PTY support) |
| `process` | Manage background processes |

### Moltbot Tools

| Tool | Description |
|------|-------------|
| `browser` | CDP browser automation |
| `canvas` | Visual workspace control |
| `web_search` | Web search (Brave API) |
| `web_fetch` | Fetch & extract web content |
| `message` | Send to any messaging channel |
| `sessions_list` | List active sessions |
| `sessions_history` | Get session transcript |
| `sessions_send` | Cross-session messaging |
| `sessions_spawn` | Create subagents |
| `nodes` | Device control (camera, screen, notify) |
| `cron` | Schedule jobs and reminders |
| `gateway` | Restart, update Moltbot |
| `image` | Analyze images with vision |
| `tts` | Text-to-speech (ElevenLabs) |
| `memory_search` | Search memory files |
| `memory_get` | Get memory content |

---

## Can It Build Software?

**Yes.** Moltbot has full software development capabilities:

### What It Can Do

```bash
# File operations
read("/path/to/file")
write("/path/to/file", content)
edit("/path/to/file", old, new)

# Shell commands (anything you could run)
exec("git clone https://github.com/...")
exec("npm install && npm run build")
exec("docker compose up -d")
exec("python train_model.py")

# Browser automation
browser.open("https://example.com")
browser.click("#submit")
browser.fill("input[name=email]", "test@example.com")
```

### Coding Agent Integration

Moltbot can orchestrate other coding agents:

```bash
# Run Codex CLI
bash pty:true command:"codex exec --full-auto 'Build a REST API'"

# Run Claude Code
bash pty:true command:"claude 'Refactor the auth module'"

# Parallel issue fixing with git worktrees
git worktree add -b fix/issue-78 /tmp/issue-78 main
bash pty:true workdir:/tmp/issue-78 background:true command:"codex --yolo 'Fix issue #78'"
```

---

## Security Model

### Default: Full Trust for Your DMs

```
Main session (your direct chats) → Full host access
Groups/channels → Can be sandboxed
```

### Security Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: WHO CAN TALK TO IT                                    │
│  • DM pairing (strangers get pairing code)                      │
│  • Allowlists per channel                                       │
│  • Group mention requirements                                   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: WHAT TOOLS ARE AVAILABLE                              │
│  • Tool policy (allow/deny lists)                               │
│  • Tool profiles: minimal, coding, messaging, full              │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: WHERE CODE RUNS                                       │
│  • Sandbox mode (Docker container)                              │
│  • Gateway host (your machine)                                  │
│  • Node (companion device)                                      │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4: EXEC APPROVALS                                        │
│  • deny: block all host commands                                │
│  • allowlist: only pre-approved commands                        │
│  • ask: prompt before running                                   │
│  • full: allow everything                                       │
└─────────────────────────────────────────────────────────────────┘
```

### Sandbox Configuration

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main"  // Sandbox everything except your DMs
      }
    }
  }
}
```

---

## Autonomous Capabilities

Moltbot can act without being asked:

### 1. Heartbeat (Periodic Check-ins)

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "enabled": true,
        "every": "10m",
        "prompt": "Check system events and pending tasks."
      }
    }
  }
}
```

### 2. Cron Jobs (Scheduled Tasks)

```bash
# Morning briefing every day at 7am
moltbot cron add \
  --name "Morning status" \
  --cron "0 7 * * *" \
  --message "Summarize inbox + calendar" \
  --deliver --channel whatsapp --to "+15551234567"

# One-shot reminder
moltbot cron add --name "Reminder" --at "20m" \
  --system-event "Call the dentist!" --wake now
```

### 3. Webhooks (External Triggers)

```bash
# External systems can trigger Moltbot
curl -X POST http://127.0.0.1:18789/hooks/wake \
  -H 'Authorization: Bearer SECRET' \
  -d '{"text":"New email from boss","mode":"now"}'
```

### 4. Voice Wake

```json
{ "triggers": ["clawd", "claude", "hey assistant"] }
```

### Autonomy Summary

| Feature | Trigger | What It Does |
|---------|---------|--------------|
| **Heartbeat** | Timer (e.g., every 10m) | Check events, proactive alerts |
| **Cron** | Schedule (cron/at/every) | Run any task, deliver anywhere |
| **Webhooks** | External HTTP call | React to email, CI, etc. |
| **Voice Wake** | Wake word spoken | Activate and listen |
| **Background Monitor** | Process completion | Notify when tasks finish |

---

## How It Differs From Other Agents

### Moltbot vs Terminal Agents (Codex, Claude Code)

| Aspect | Terminal Agent | Moltbot |
|--------|---------------|---------|
| **Lifecycle** | Starts → runs → exits | Always running (daemon) |
| **Interface** | Terminal only | Any messaging app |
| **Access** | At computer only | From anywhere (phone) |
| **Session** | Lost when closed | Persistent across days |
| **Proactive** | No | Yes (cron, heartbeat) |

### Moltbot vs Cloud Agents (ChatGPT, Claude.ai)

| Aspect | Cloud Agent | Moltbot |
|--------|-------------|---------|
| **Data** | Their servers | Your machine |
| **File access** | ❌ | ✅ Full filesystem |
| **Run code** | ❌ | ✅ Any command |
| **Device control** | ❌ | ✅ Mac, iOS, Android |
| **Custom integrations** | ❌ | ✅ Anything |
| **Cost** | Monthly subscription | Pay per token |

### Moltbot vs Computer Use Agents (Anthropic Computer Use, Manus)

| Aspect | Computer Use | Moltbot |
|--------|--------------|---------|
| **Control method** | Screen + mouse (vision) | Direct APIs, CLI, browser CDP |
| **Speed** | Slow (screenshot → think → act) | Fast (direct calls) |
| **Reliability** | Fragile (UI changes break it) | Robust (APIs are stable) |
| **Interface** | Their desktop app | Any messaging app |
| **Autonomous** | ❌ | ✅ (cron, webhooks) |
| **Remote access** | ❌ | ✅ (phone, anywhere) |

### Category Definition

**Moltbot is NOT a computer use agent.** It's an:

> **OS-level AI agent with remote channels and autonomy**

- **OS-Level**: Direct filesystem, shell, process, network access
- **Remote Channels**: WhatsApp, Telegram, Slack, Discord, Voice, Web, CLI
- **Autonomy**: Cron, heartbeat, webhooks, background monitoring

---

## End-to-End Example

### WhatsApp Message: "What's the weather in NYC?"

```
Step 1: MESSAGE ARRIVAL
        User sends WhatsApp message
        ↓
        Baileys library receives via WebSocket
        ↓
        Channel handler validates allowlist

Step 2: SESSION RESOLUTION
        Resolve session key → "agent:default:main"
        Load conversation history from JSONL
        Check reset conditions

Step 3: AGENT EXECUTION
        agentCommand() orchestrates the run
        ↓
        runEmbeddedPiAgent({
          sessionId, sessionKey,
          prompt: "What's the weather in NYC?",
          provider: "anthropic",
          model: "claude-opus-4-5"
        })

Step 4: PI AGENT RUNTIME
        Queue in session lane
        Build system prompt
        Load history
        Call LLM API
        Stream response

Step 5: TOOL EXECUTION (if needed)
        Model decides: { "tool": "web_fetch", "args": { "url": "https://wttr.in/NYC" } }
        ↓
        Execute tool
        Return result to model
        Model generates final response

Step 6: RESPONSE DELIVERY
        Chunk if needed (WhatsApp limits)
        Format for channel
        Send via Baileys
        ↓
        User receives: "The weather in NYC is 45°F and partly cloudy..."

Step 7: EVENT BROADCASTING
        broadcast("agent", { stream: "lifecycle", phase: "end" })
        ↓
        macOS app, WebChat, etc. update in real-time
```

---

## Key Source Directories

| Directory | Purpose |
|-----------|---------|
| `src/gateway/` | WebSocket server, RPC methods, protocol |
| `src/agents/` | Agent runtime, tools, skills, model auth |
| `src/agents/tools/` | Tool implementations |
| `src/agents/pi-embedded-runner/` | Core agent loop |
| `src/auto-reply/` | Message processing, commands |
| `src/channels/` | Channel plugin system |
| `src/telegram/`, `src/discord/`, etc. | Channel implementations |
| `src/config/` | Configuration loading |
| `src/cron/` | Scheduled jobs |
| `src/infra/` | Heartbeat, webhooks, utilities |
| `apps/macos/` | macOS menu bar app |
| `apps/ios/`, `apps/android/` | Mobile companion apps |
| `extensions/` | Plugin channels (MS Teams, Matrix, etc.) |

---

## Can I Build This Myself?

**Yes, in principle.** Moltbot is:

```
Codex/Claude Code (shell access)
    + WhatsApp/Telegram/Slack API wrappers
    + Cron daemon
    + Webhook server
    + Session persistence
    + Message routing
    + Security model
    + Edge case handling
    + ~2 years of integration work
```

The value is in the **integration, polish, and features** - not unique magic.

### What You'd Build

```python
# Pseudocode for DIY Moltbot
whatsapp = Baileys()
telegram = Grammy()
slack = Bolt()

def on_message(channel, sender, text):
    session = load_session(sender)
    response = codex.run(text, context=session)
    save_session(sender, response)
    send_reply(channel, sender, response)

schedule.every(10).minutes.do(heartbeat)

@app.post("/webhook")
def webhook(payload):
    codex.run(f"Handle: {payload}")
```

Then spend 6-12 months handling edge cases. 😅

---

## Summary

| Question | Answer |
|----------|--------|
| **What is Moltbot?** | OS-level AI agent with remote channels and autonomy |
| **Can it build software?** | Yes - full filesystem, shell, browser access |
| **Can it use the computer freely?** | Yes, with configurable security controls |
| **Can it act on its own?** | Yes - cron, heartbeat, webhooks |
| **How does it differ from terminal agents?** | Always-on, multi-channel, remote access, persistent sessions |
| **Is it like a cloud agent on your machine?** | Yes - same convenience, self-hosted control |
| **Is it a computer use agent?** | No - OS-level access (APIs), not screen control (vision) |
| **What agent runtime does it use?** | `@mariozechner/pi-agent-core` |
| **What tools does it have?** | read, write, edit, exec, browser, message, cron, nodes, web_*, sessions_*, etc. |

---

## Quick Start

```bash
# Install
npm install -g moltbot@latest

# Onboard (wizard walks you through setup)
moltbot onboard --install-daemon

# Start gateway
moltbot gateway --port 18789 --verbose

# Send a message
moltbot agent --message "Hello, Moltbot!"
```

For full documentation: [https://docs.molt.bot](https://docs.molt.bot)
