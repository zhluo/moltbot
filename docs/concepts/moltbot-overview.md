---
summary: "Comprehensive overview of Moltbot architecture and end-to-end message flow"
read_when:
  - Understanding how Moltbot works
  - Learning the architecture
  - Onboarding new developers
---
# Moltbot Architecture Overview

This document provides a comprehensive overview of how Moltbot works, including the architecture, key components, and an end-to-end example of message processing.

## What is Moltbot?

**Moltbot** is a personal AI assistant that you run on your own devices. It connects to multiple messaging channels (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, etc.) and provides a unified AI assistant experience across all of them. The assistant can execute tools, browse the web, manage files, control devices, and more.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            MESSAGING CHANNELS                                │
│  WhatsApp │ Telegram │ Slack │ Discord │ Signal │ iMessage │ WebChat │ ...  │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              GATEWAY                                         │
│                    (WebSocket Control Plane)                                 │
│                    ws://127.0.0.1:18789                                      │
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Channel    │  │   Session   │  │    Agent    │  │   Tools     │         │
│  │  Manager    │  │   Manager   │  │   Runtime   │  │   Registry  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │    Cron     │  │   Hooks     │  │   Plugins   │  │   Config    │         │
│  │  Scheduler  │  │   Engine    │  │   System    │  │   Manager   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CLIENT CONNECTIONS                                 │
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  macOS App  │  │    CLI      │  │  WebChat    │  │  iOS/Android│         │
│  │  (Menu Bar) │  │  Commands   │  │     UI      │  │    Nodes    │         │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Gateway (Control Plane)

The **Gateway** is the central hub of Moltbot. It's a long-running process that:

- **Maintains provider connections** to all messaging channels (WhatsApp, Telegram, etc.)
- **Exposes a WebSocket API** for clients to connect and interact
- **Routes messages** between channels and the AI agent
- **Manages sessions** and conversation state
- **Coordinates tools** and executes agent actions

**Key files:**
- `src/gateway/server.impl.ts` - Main gateway server implementation
- `src/gateway/server-methods/` - WebSocket RPC method handlers
- `src/gateway/protocol/` - Protocol schemas and types

### 2. Channels

Channels are the messaging integrations. Each channel:
- Connects to a specific messaging platform
- Receives inbound messages
- Sends outbound replies
- Handles platform-specific features (reactions, threads, media)

**Supported channels:**
- **WhatsApp** (`src/whatsapp/`) - via Baileys library
- **Telegram** (`src/telegram/`) - via grammY library
- **Slack** (`src/slack/`) - via Bolt SDK
- **Discord** (`src/discord/`) - via discord.js
- **Signal** (`src/signal/`) - via signal-cli
- **iMessage** (`src/imessage/`) - macOS only
- **Extension channels** (`extensions/`) - MS Teams, Matrix, Zalo, etc.

**Key files:**
- `src/channels/plugins/` - Channel plugin system
- `src/channels/registry.ts` - Channel registration

### 3. Agent Runtime (Pi Agent)

The agent runtime executes the AI assistant logic:

- **Session management** - Loads/saves conversation history
- **System prompt assembly** - Builds context from workspace, skills, bootstrap files
- **Model inference** - Calls the LLM (Anthropic Claude, OpenAI, etc.)
- **Tool execution** - Runs tools when the model requests them
- **Streaming** - Emits partial responses for real-time feedback

**Key files:**
- `src/agents/pi-embedded-runner/run.ts` - Main agent execution loop
- `src/commands/agent.ts` - Agent command entry point
- `src/agents/tools/` - Built-in tool implementations

### 4. Sessions

Sessions track conversation state:
- **Session keys** identify conversations (e.g., `agent:default:main`, `agent:default:telegram:group:123`)
- **Session store** (`~/.clawdbot/agents/<agentId>/sessions/sessions.json`) persists metadata
- **Transcripts** (`*.jsonl`) store full conversation history

**Session scoping:**
- DMs can be scoped to `main` (shared), `per-peer`, or `per-channel-peer`
- Groups/channels get isolated sessions
- Identity links can unify users across channels

### 5. Tools

Tools extend the agent's capabilities:

| Tool | Description | Location |
|------|-------------|----------|
| `bash` | Execute shell commands | Built into pi-agent-core |
| `browser` | Control Chromium browser | `src/agents/tools/browser-tool.ts` |
| `canvas` | Agent-controlled visual workspace | `src/agents/tools/canvas-tool.ts` |
| `cron` | Schedule recurring tasks | `src/agents/tools/cron-tool.ts` |
| `message_send` | Send messages to channels | `src/agents/tools/message-tool.ts` |
| `web_fetch` | Fetch web pages | `src/agents/tools/web-fetch.ts` |
| `web_search` | Search the web | `src/agents/tools/web-search.ts` |
| `memory_*` | Persistent memory operations | `src/agents/tools/memory-tool.ts` |
| `sessions_*` | Cross-session communication | `src/agents/tools/sessions-*.ts` |
| `nodes_*` | Control connected devices | `src/agents/tools/nodes-tool.ts` |

### 6. Skills

Skills are modular capabilities loaded from the workspace:
- **Bundled skills** - Shipped with Moltbot
- **Managed skills** - Downloaded from ClawdHub registry
- **Workspace skills** - User-created in `~/clawd/skills/`

Skills inject tools and prompts into the agent context.

---

## End-to-End Example: WhatsApp Message Processing

Let's trace what happens when you send "What's the weather in NYC?" via WhatsApp:

### Step 1: Message Arrival (WhatsApp → Gateway)

```
User sends WhatsApp message: "What's the weather in NYC?"
         │
         ▼
┌─────────────────────────────────────┐
│  WhatsApp Client (Baileys)          │
│  src/whatsapp/                      │
│                                     │
│  1. Receives message via WebSocket  │
│  2. Extracts sender, text, media    │
│  3. Validates allowlist             │
└─────────────────┬───────────────────┘
                  │
                  ▼
```

**Code path:** The Baileys library (WhatsApp Web protocol) receives the message and triggers a handler in `src/whatsapp/` or the channel plugin system.

### Step 2: Inbound Processing

```
┌─────────────────────────────────────┐
│  Channel Handler                    │
│                                     │
│  1. Normalize message context       │
│  2. Check DM/group policy           │
│  3. Handle pairing if needed        │
│  4. Route to agent                  │
└─────────────────┬───────────────────┘
                  │
                  ▼
```

The channel handler:
1. Creates a `MsgContext` with normalized fields (`Body`, `From`, `To`, `ChatType`, etc.)
2. Checks if the sender is allowed (`allowFrom` config)
3. If pairing is required, sends a pairing code instead of processing
4. Otherwise, routes to the agent runtime

**Key type (from `src/auto-reply/templating.ts`):**
```typescript
type MsgContext = {
  Body?: string;
  From?: string;
  To?: string;
  SessionKey?: string;
  AccountId?: string;
  ChatType?: string;
  Provider?: string;
  // ... more fields
}
```

### Step 3: Session Resolution

```
┌─────────────────────────────────────┐
│  Session Manager                    │
│  src/config/sessions.ts             │
│                                     │
│  1. Resolve session key             │
│     → "agent:default:main"          │
│  2. Load or create session entry    │
│  3. Load JSONL transcript           │
│  4. Check reset conditions          │
└─────────────────┬───────────────────┘
                  │
                  ▼
```

**Session key resolution:**
- DM messages typically go to `agent:default:main` (shared session)
- Group messages go to `agent:default:whatsapp:group:<groupId>`
- The session store tracks metadata like `sessionId`, `thinkingLevel`, `lastChannel`

### Step 4: Agent Execution

```
┌─────────────────────────────────────┐
│  Agent Command                      │
│  src/commands/agent.ts              │
│                                     │
│  1. Validate parameters             │
│  2. Resolve model (claude-opus-4-5) │
│  3. Load skills snapshot            │
│  4. Call runEmbeddedPiAgent()       │
└─────────────────┬───────────────────┘
                  │
                  ▼
```

**Code:** `src/commands/agent.ts` orchestrates the agent run:
```typescript
const result = await runWithModelFallback({
  cfg,
  provider,
  model,
  run: (providerOverride, modelOverride) => {
    return runEmbeddedPiAgent({
      sessionId,
      sessionKey,
      prompt: body,
      provider: providerOverride,
      model: modelOverride,
      thinkLevel: resolvedThinkLevel,
      // ...
    });
  },
});
```

### Step 5: Pi Agent Runtime

```
┌─────────────────────────────────────┐
│  Pi Agent Runtime                   │
│  src/agents/pi-embedded-runner/     │
│                                     │
│  1. Queue in session lane           │
│  2. Resolve auth profile            │
│  3. Build system prompt             │
│  4. Load message history            │
│  5. Call LLM API                    │
│  6. Stream response                 │
│  7. Execute tools if requested      │
│  8. Save transcript                 │
└─────────────────┬───────────────────┘
                  │
                  ▼
```

**The embedded Pi agent:**
1. **Queues the run** - Serializes requests per session to prevent races
2. **Resolves auth** - Gets API key from profiles (OAuth, API key, etc.)
3. **Builds system prompt** - Combines base prompt + skills + bootstrap files + workspace context
4. **Loads history** - Reads previous turns from JSONL transcript
5. **Calls LLM** - Sends request to Anthropic/OpenAI/etc.
6. **Streams** - Emits `assistant` events as tokens arrive
7. **Handles tools** - If model calls a tool, executes it and loops
8. **Persists** - Appends user message and assistant response to JSONL

### Step 6: Tool Execution (if needed)

If the model decides to use a tool (e.g., `web_fetch` to get weather):

```
┌─────────────────────────────────────┐
│  Tool Execution                     │
│  src/agents/tools/                  │
│                                     │
│  Model response:                    │
│  {                                  │
│    "tool": "web_fetch",             │
│    "args": {                        │
│      "url": "https://wttr.in/NYC"   │
│    }                                │
│  }                                  │
│                                     │
│  1. Validate tool access            │
│  2. Execute tool function           │
│  3. Return result to model          │
│  4. Model generates final response  │
└─────────────────┬───────────────────┘
                  │
                  ▼
```

**Tool result flows back to the model, which generates the final response.**

### Step 7: Response Delivery

```
┌─────────────────────────────────────┐
│  Outbound Delivery                  │
│  src/commands/agent/delivery.ts     │
│                                     │
│  1. Chunk long messages             │
│  2. Format for channel (markdown)   │
│  3. Send via WhatsApp client        │
└─────────────────┬───────────────────┘
                  │
                  ▼

User receives: "The weather in NYC is currently 45°F and partly cloudy..."
```

The response is:
1. **Chunked** if needed (WhatsApp has message length limits)
2. **Formatted** for the channel (markdown support varies)
3. **Sent** via the originating channel's outbound adapter

### Step 8: Event Broadcasting

Throughout this process, the Gateway broadcasts events to connected clients:

```typescript
// Events emitted during agent run
broadcast("agent", { runId, stream: "lifecycle", data: { phase: "start" } });
broadcast("agent", { runId, stream: "assistant", data: { delta: "The weather..." } });
broadcast("agent", { runId, stream: "tool", data: { name: "web_fetch", status: "start" } });
broadcast("agent", { runId, stream: "lifecycle", data: { phase: "end" } });
```

The **macOS app**, **WebChat**, and other clients receive these events to show real-time progress.

---

## Configuration

Moltbot configuration lives in `~/.clawdbot/moltbot.json`:

```json5
{
  // Model configuration
  agent: {
    model: "anthropic/claude-opus-4-5"
  },
  
  // Channel configuration
  channels: {
    whatsapp: {
      allowFrom: ["+1234567890"],
      groups: { "*": { requireMention: true } }
    },
    telegram: {
      botToken: "123456:ABCDEF",
      allowFrom: ["@username"]
    }
  },
  
  // Gateway settings
  gateway: {
    port: 18789,
    bind: "loopback"
  },
  
  // Session settings
  session: {
    dmScope: "main",
    reset: { daily: "04:00" }
  }
}
```

---

## Key Source Directories

| Directory | Purpose |
|-----------|---------|
| `src/gateway/` | WebSocket server, RPC methods, protocol |
| `src/agents/` | Agent runtime, tools, skills, model auth |
| `src/auto-reply/` | Message processing, commands, templating |
| `src/channels/` | Channel plugin system, routing |
| `src/telegram/`, `src/discord/`, etc. | Channel-specific implementations |
| `src/cli/` | CLI command wiring |
| `src/commands/` | Command implementations |
| `src/config/` | Configuration loading and validation |
| `src/infra/` | Infrastructure utilities |
| `apps/macos/` | macOS menu bar app |
| `apps/ios/`, `apps/android/` | Mobile companion apps |
| `extensions/` | Channel plugins (MS Teams, Matrix, etc.) |

---

## CLI Commands

Quick reference for common operations:

```bash
# Start the gateway
moltbot gateway --port 18789 --verbose

# Send a message to the agent
moltbot agent --message "Hello, Clawd!"

# Check channel status
moltbot channels status --probe

# Run the onboarding wizard
moltbot onboard --install-daemon

# Check system health
moltbot doctor
```

---

## Summary

Moltbot works by:

1. **Gateway** maintains connections to messaging channels and exposes a WebSocket API
2. **Channels** receive messages and route them to the agent
3. **Sessions** track conversation state per user/group
4. **Agent runtime** calls the LLM with context, history, and tools
5. **Tools** extend agent capabilities (bash, browser, web, etc.)
6. **Responses** are delivered back through the originating channel

The architecture is designed to be:
- **Local-first** - Runs on your own devices
- **Multi-channel** - One assistant, many messaging surfaces
- **Extensible** - Plugins, skills, and hooks
- **Secure** - Pairing, allowlists, and sandboxing options
