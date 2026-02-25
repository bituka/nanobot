# Nanobot Remake Guide: Comprehensive Architecture Documentation

## Introduction

This guide provides a detailed blueprint for recreating nanobot, an ultra-lightweight personal AI assistant. Nanobot delivers core agent functionality in just ~4,000 lines of code, making it an excellent project for learning and customization.

## 🏗️ Architecture Overview

Nanobot follows a modular, event-driven architecture:

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   Chat Channels  │────▶│   Message Bus    │────▶│   Agent Loop     │
│ (Telegram/Discord│     │                  │     │                  │
│  WhatsApp/etc.)  │     │                  │     └──────────────────┘
└──────────────────┘     └──────────────────┘              ▲
                                                             │
                                                             │
┌──────────────────┐     ┌──────────────────┐              │
│    LLM Providers │────▶│    Context       │──────────────┘
│ (OpenAI/Anthropic│     │    Builder       │
│  OpenRouter etc.)│     │                  │
└──────────────────┘     └──────────────────┘
         ▲                      ▲
         │                      │
┌──────────────────┐     ┌──────────────────┐
│   Skill System   │────▶│  Tool Registry   │
│                  │     │                  │
└──────────────────┘     └──────────────────┘
         ▲                      ▲
         │                      │
┌──────────────────┐     ┌──────────────────┐
│   Memory Store   │────▶│  Session Mgr     │
│                  │     │                  │
└──────────────────┘     └──────────────────┘
```

## 1. Core Components

### 1.1 Agent Loop (Brain of the System)

**File:** [`nanobot/agent/loop.py`](nanobot/agent/loop.py)

The Agent Loop is the core processing engine that orchestrates all interactions. It:

1. Receives messages from the bus
2. Builds context with history, memory, skills
3. Calls the LLM
4. Executes tool calls
5. Sends responses back

Key Features:
- Async processing with timeout handling
- Tool call execution and error recovery
- Session management
- Memory consolidation
- MCP server connection
- Progress streaming

Key Methods:
- `run()` - Main loop that processes incoming messages
- `_run_agent_loop()` - Core iteration cycle
- `_process_message()` - Handles individual messages
- `_consolidate_memory()` - Manages memory archiving

### 1.2 Context Builder (Prompt Assembler)

**File:** [`nanobot/agent/context.py`](nanobot/agent/context.py)

The Context Builder assembles the system prompt for the LLM by combining:

1. **Bootstrap files** (AGENTS.md, SOUL.md, USER.md, TOOLS.md)
2. **Memory context** (from MEMORY.md)
3. **Skills information** (always-loaded vs available skills)
4. **Conversation history**
5. **Runtime context** (current time, channel info)

Key Features:
- Dynamic prompt construction
- Media support (base64-encoded images)
- Runtime context injection
- Message history management
- Tool result formatting

Key Methods:
- `build_system_prompt()` - Creates complete system prompt
- `build_messages()` - Assembles full message list for LLM
- `add_tool_result()` - Formats tool results for messages
- `add_assistant_message()` - Adds assistant responses to history

### 1.3 Message Bus (Event Hub)

**File:** [`nanobot/bus/queue.py`](nanobot/bus/queue.py)

The Message Bus provides async communication between channels and the agent core. It decouples the input/output channels from the processing logic.

Key Features:
- Inbound queue for messages from channels
- Outbound queue for responses to channels
- Async message passing
- Session key management

Classes:
- `MessageBus` - Main bus implementation
- `InboundMessage` - Data class for incoming messages
- `OutboundMessage` - Data class for outgoing messages

### 1.4 Session Management (Conversation History)

**File:** [`nanobot/session/manager.py`](nanobot/session/manager.py)

Session Manager handles conversation history persistence using JSONL files.

Key Features:
- Session creation/retrieval
- Message storage in JSONL format
- History retrieval with memory window
- Session clearing (/new command)
- Legacy session migration

Classes:
- `Session` - Individual conversation session
- `SessionManager` - Manages all sessions

### 1.5 Memory System (Long-Term Storage)

**File:** [`nanobot/agent/memory.py`](nanobot/agent/memory.py)

Dual-layer memory system:
1. **MEMORY.md** - Long-term fact storage (markdown)
2. **HISTORY.md** - Grep-searchable conversation log

Key Features:
- Automatic memory consolidation
- LLM-powered summarization
- Persistent storage in workspace
- Memory context injection into prompts

Key Methods:
- `consolidate()` - Archives old messages using LLM
- `read_long_term()` / `write_long_term()` - Memory file operations
- `append_history()` - Adds entries to history log

### 1.6 Tools & Tool Registry

**File:** [`nanobot/agent/tools/base.py`](nanobot/agent/tools/base.py), [`nanobot/agent/tools/registry.py`](nanobot/agent/tools/registry.py)

Tools are capabilities that the agent can use to interact with the environment.

Key Features:
- Tool abstraction layer
- JSON Schema validation
- Dynamic tool registration
- Execution with error handling
- OpenAI function schema compatibility

Core Tools:
1. **File System Tools** - Read, write, edit, list files
2. **Shell Tool** - Execute commands (with safety guards)
3. **Web Tools** - Search, fetch web content
4. **Message Tool** - Send messages to channels
5. **Spawn Tool** - Create subagents
6. **Cron Tool** - Manage scheduled tasks

### 1.7 LLM Providers (Model Integration)

**File:** [`nanobot/providers/base.py`](nanobot/providers/base.py), [`nanobot/providers/registry.py`](nanobot/providers/registry.py)

Unified interface for multiple LLM providers with provider registry.

Key Features:
- Provider registry for easy extensibility
- Support for 20+ providers (OpenAI, Anthropic, OpenRouter, etc.)
- OAuth login support
- Prompt caching support
- Model parameter overrides

Provider Types:
- **Direct** - Custom OpenAI-compatible endpoints
- **Gateway** - Model routing (OpenRouter, AiHubMix)
- **Standard** - Direct provider APIs (Anthropic, OpenAI)
- **Local** - vLLM, Ollama, etc.

### 1.8 Chat Channels (User Interface)

**File:** [`nanobot/channels/base.py`](nanobot/channels/base.py), individual channel implementations

Channels handle communication with different chat platforms.

Key Features:
- BaseChannel abstract interface
- Permission checking (allowFrom list)
- Message handling and sending
- Media support (images, voice, files)
- Command handlers

Supported Channels:
1. **Telegram** - Long polling with python-telegram-bot
2. **Discord** - WebSocket gateway
3. **WhatsApp** - Node.js bridge + WebSocket
4. **Feishu** - WebSocket long connection
5. **Mochat** - Socket.IO WebSocket
6. **DingTalk** - Stream mode
7. **Slack** - Socket mode
8. **Email** - IMAP/SMTP
9. **QQ** - BotPy SDK

## 2. Configuration System

**File:** [`nanobot/config/schema.py`](nanobot/config/schema.py), [`nanobot/config/loader.py`](nanobot/config/loader.py)

Uses Pydantic for validation and configuration management.

Key Features:
- YAML/JSON configuration file support
- Environment variable overrides
- Per-provider configuration
- Per-channel configuration
- Agent defaults (model, temperature, etc.)

Config File Structure (~/.nanobot/config.json):
```json
{
  "providers": {
    "openrouter": {
      "apiKey": "sk-or-v1-xxx"
    }
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-5",
      "temperature": 0.1
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "token": "YOUR_BOT_TOKEN",
      "allowFrom": ["YOUR_USER_ID"]
    }
  }
}
```

## 3. CLI Interface

**File:** [`nanobot/cli/commands.py`](nanobot/cli/commands.py)

Provides command-line interface for interacting with nanobot.

Key Commands:
- `nanobot onboard` - Initial setup
- `nanobot agent` - Chat with agent (interactive or direct)
- `nanobot gateway` - Start the gateway with all channels
- `nanobot status` - Show system status
- `nanobot cron` - Manage scheduled tasks
- `nanobot provider login` - OAuth login for providers
- `nanobot channels login` - WhatsApp login (QR scan)

## 4. Skills System

**File:** [`nanobot/agent/skills.py`](nanobot/agent/skills.py)

Skills extend the agent's capabilities. They are stored in the workspace and loaded dynamically.

Key Features:
- Skill discovery and loading
- Always-loaded skills vs available skills
- Skill summary for LLM context
- Skill dependencies management

Skill Structure:
```
skills/
├── my-skill/
│   ├── SKILL.md          # Skill description
│   └── requirements.txt  # Optional dependencies
└── ...
```

## 5. Heartbeat & Scheduled Tasks

### 5.1 Heartbeat (Periodic Tasks)

**File:** [`nanobot/heartbeat/service.py`](nanobot/heartbeat/service.py)

The heartbeat service wakes up every 30 minutes and checks `HEARTBEAT.md` in the workspace for tasks to execute.

### 5.2 Cron Service (Scheduled Tasks)

**File:** [`nanobot/cron/service.py`](nanobot/cron/service.py)

Cron service manages scheduled tasks using cron syntax or interval notation.

Key Features:
- Add/remove/list cron jobs
- Execute commands at specified times
- Persistent storage of jobs

## 6. MCP (Model Context Protocol) Support

**File:** [`nanobot/agent/tools/mcp.py`](nanobot/agent/tools/mcp.py)

MCP allows connecting external tool servers. Nanobot supports two transport modes:

1. **Stdio** - Local processes (npx, ux)
2. **HTTP** - Remote endpoints (SSE)

Key Features:
- Automatic tool discovery
- Tool registration on startup
- Integrated with agent loop
- Timeout configuration

## 7. Docker & Deployment

**File:** [`Dockerfile`](Dockerfile), [`docker-compose.yml`](docker-compose.yml)

Docker support for easy deployment.

Key Features:
- Multi-stage build for small image
- Volume mount for config and workspace
- Healthcheck endpoint
- docker-compose configuration

## 8. Key Design Principles

### 8.1 Simplicity First

- Minimal dependencies
- Readable code
- Flat architecture
- Avoid over-engineering

### 8.2 Extensibility

- Provider registry for easy integration
- Channel interface for new platforms
- Tool interface for new capabilities
- Skill system for domain-specific knowledge

### 8.3 Reliability

- Error handling and recovery
- Session persistence
- Memory consolidation
- Safety guards (command restrictions, path checks)

### 8.4 Performance

- Async processing
- Memory window management
- Prompt caching
- Efficient message handling

## 9. Step-by-Step Implementation Guide

### Phase 1: Foundation

1. **Set up project structure**
2. **Create core data models** (messages, sessions)
3. **Implement message bus**
4. **Create basic CLI framework**

### Phase 2: Agent Core

1. **Implement context builder**
2. **Create agent loop**
3. **Implement session management**
4. **Add memory system**

### Phase 3: Tools & Skills

1. **Implement base tool class**
2. **Create core tools** (file, shell, web, message)
3. **Implement tool registry**
4. **Add skills system**

### Phase 4: LLM Integration

1. **Create provider interface**
2. **Implement provider registry**
3. **Add provider implementations** (OpenAI, Anthropic, etc.)
4. **Implement litellm integration**

### Phase 5: Channels

1. **Create base channel interface**
2. **Implement first channel** (Telegram recommended)
3. **Add more channels** (Discord, WhatsApp, etc.)
4. **Implement channel manager**

### Phase 6: Advanced Features

1. **Add heartbeat service**
2. **Implement cron service**
3. **Add MCP support**
4. **Implement Docker configuration**

### Phase 7: Polish & Testing

1. **Add error handling**
2. **Implement safety guards**
3. **Add documentation**
4. **Test all components**

## 10. Technology Stack

- **Language:** Python 3.11+
- **Async Framework:** asyncio
- **LLM Integration:** litellm
- **Configuration:** Pydantic
- **Logging:** loguru
- **Chat Platforms:** Various SDKs (python-telegram-bot, discord.py, etc.)
- **Containerization:** Docker

## 11. Development Tips

1. **Start small** - Implement core functionality first
2. **Test incrementally** - Test each component before integration
3. **Document as you go** - Keep code readable and well-documented
4. **Use async/await** - All I/O operations should be async
5. **Implement safety checks** - Prevent dangerous operations
6. **Handle errors gracefully** - Provide meaningful error messages

## 12. Future Enhancements

Potential features to add:
- Multi-modal support (images, voice, video)
- Long-term memory improvements
- Better reasoning capabilities
- Calendar integration
- Self-improvement mechanisms
- More chat platform integrations

## Conclusion

Nanobot's architecture is designed for simplicity, extensibility, and reliability. By following this guide, you can recreate a powerful AI assistant with core agent functionality. The modular design makes it easy to customize and expand with new features.

Remember, the key to success is to start small, test thoroughly, and build incrementally. Nanobot's minimal footprint allows for rapid iteration and experimentation.
