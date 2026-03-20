# MCP Architecture Deep Dive

A comprehensive guide to how the Model Context Protocol works — from high-level architecture down to the exact message flow between components.

---

## 1. The Core Architecture

MCP follows a **client-server architecture** with three distinct roles:

```
┌──────────────────────────────────────────────────────┐
│                    HOST APPLICATION                    │
│               (Claude Desktop / IDE / App)             │
│                                                        │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐         │
│   │MCP Client│   │MCP Client│   │MCP Client│  ← 1:1  │
│   └────┬─────┘   └────┬─────┘   └────┬─────┘         │
│        │              │              │                 │
│   ┌────┴────────┐┌────┴────────┐┌────┴──────────┐     │
│   │  AI Model   ││             ││               │     │
│   │  (Claude)   ││             ││               │     │
│   └─────────────┘│             ││               │     │
└──────────────────┼─────────────┼┼───────────────┼─────┘
     JSON-RPC      │  JSON-RPC   ││   JSON-RPC    │
     (stdio)       │  (stdio)    ││   (HTTP)      │
┌──────────────┐┌──┴───────────┐┌┴┴──────────────┐
│ MCP Server:  ││ MCP Server:  ││ MCP Server:    │
│ File System  ││ Database     ││ External APIs  │
└──────────────┘└──────────────┘└────────────────┘
```

### Host
The AI application the user interacts with (e.g., Claude Desktop). Manages the UI, security, user consent, and orchestrates everything. It creates and manages MCP clients, makes API calls to the model provider, and routes tool calls.

### Client
A protocol-level connector that maintains a 1:1 connection with a single MCP server. Handles protocol negotiation, capability discovery, and JSON-RPC message routing. Clients are generic — the same client code is used regardless of which server it connects to.

### Server
A lightweight program that exposes specific capabilities (tools, resources, prompts) via the MCP protocol. Servers are focused, modular, and can be written in any language.

---

## 2. The Three Primitives

Every MCP server can expose any combination of three capability types:

| Primitive | Controlled By | Description | Example |
|-----------|--------------|-------------|---------|
| **Tools** | Model | Functions the AI can call | `query_db()`, `create_file()` |
| **Resources** | Application (Host) | Data the model can read | File contents, DB records |
| **Prompts** | User | Reusable prompt templates | "Summarize code", "Debug error" |

The distinction in control is important:
- **Tools**: The model autonomously decides when to invoke them based on reasoning.
- **Resources**: The host decides when to surface them as context to the model.
- **Prompts**: The user explicitly selects and triggers them.

---

## 3. The Transport Layer

MCP uses **JSON-RPC 2.0** as its wire protocol. Two transport mechanisms:

- **stdio** — The host spawns the server as a child process, communicating over stdin/stdout. Used for local servers.
- **Streamable HTTP** — For remote/cloud-hosted servers. Uses HTTP with Server-Sent Events (SSE) for server-to-client streaming.

Because the protocol is JSON-RPC over a transport, **the client and server can be written in completely different languages**. A TypeScript client can talk to a Python server, a Rust server, a Go server — it doesn't matter. JSON is the contract.

---

## 4. Language Independence

```
┌──────────────────┐         ┌──────────────────┐
│  MCP Client      │  JSON   │  MCP Server      │
│  (TypeScript)    │◄───────►│  (Python)         │
└──────────────────┘  over   └──────────────────┘
                      stdio
                      or HTTP

The protocol is the contract, not the implementation language.
Any spec-compliant client works with any spec-compliant server.
```

Anthropic maintains official SDKs in TypeScript and Python, but servers can be built in any language that can read/write JSON over stdio or HTTP.

---

## 5. Startup & Initialization Sequence

Everything below happens **before the user sends their first message**.

```
App Launch
    │
    ▼
Host reads configuration file
(e.g., claude_desktop_config.json)
    │
    ▼
For EACH server entry in config:
    │
    ├──► Spawn MCP Client
    │        │
    │        ▼
    │    Establish transport
    │    (spawn subprocess for stdio, or open HTTP connection)
    │        │
    │        ▼
    │    Client sends: initialize request
    │    "Here's my protocol version and capabilities"
    │        │
    │        ▼
    │    Server responds: initialize response
    │    "Here's my protocol version and capabilities"
    │        │
    │        ▼
    │    Client sends: initialized notification
    │    (Handshake complete)
    │        │
    │        ▼
    │    Client calls: tools/list
    │    Client calls: resources/list
    │    Client calls: prompts/list
    │        │
    │        ▼
    │    Server responds with all available
    │    tools, resources, and prompts
    │        │
    │        ▼
    │    Client passes tool registry to Host
    │
    └──► (repeat for next server)
    │
    ▼
All servers connected. Tool registry complete.
Chat UI appears. System is ready.
```

**Key insight**: If a server is misconfigured or down, you'll know at startup — not mid-conversation.

---

## 6. The Complete Message Flow (What Really Happens When You Chat)

This is where the real understanding comes from. Here's exactly what happens when a user asks a question that requires a tool.

### Phase 1: User Sends a Message

```
USER types: "What tables are in my database?"
    │
    ▼
HOST APPLICATION receives the message
```

### Phase 2: Host Builds the API Request

The host constructs a request to the **Anthropic API** (e.g., `api.anthropic.com/v1/messages`). This request includes the conversation history AND all tool definitions from every connected MCP server:

```json
{
  "model": "claude-sonnet-4-5-20250514",
  "messages": [
    {
      "role": "user",
      "content": "What tables are in my database?"
    }
  ],
  "tools": [
    {
      "name": "list_tables",
      "description": "List all tables in the database",
      "input_schema": {
        "type": "object",
        "properties": {}
      }
    },
    {
      "name": "query",
      "description": "Run a SQL query",
      "input_schema": {
        "type": "object",
        "properties": {
          "sql": { "type": "string" }
        }
      }
    }
  ]
}
```

**Critical point**: The model has no idea about MCP. It just sees tool definitions in its context — the same tool-calling interface used by any LLM API.

### Phase 3: The Model Reasons and Responds

Claude (running on Anthropic's servers) sees the question and available tools. It decides `list_tables` is relevant. The API response:

```json
{
  "content": [
    {
      "type": "tool_use",
      "id": "call_abc123",
      "name": "list_tables",
      "input": {}
    }
  ]
}
```

### Phase 4: The Host Interprets and Routes (NOT the client, NOT the model)

**The HOST parses the API response**, sees a `tool_use` block, and takes action:

1. Looks up which MCP Client owns the `list_tables` tool
2. Tells that client to execute the call
3. The client sends a JSON-RPC `tools/call` message to the server
4. The server executes (runs the actual database query)
5. Results flow back: Server → Client → Host

```
HOST receives API response with tool_use
    │
    ▼
HOST looks up: "list_tables" belongs to → DB Client
    │
    ▼
MCP CLIENT sends JSON-RPC to server:
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "list_tables",
    "arguments": {}
  },
  "id": 1
}
    │
    ▼
MCP SERVER executes actual database query
    │
    ▼
MCP SERVER responds:
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      { "type": "text", "text": "users, orders, products, inventory" }
    ]
  },
  "id": 1
}
    │
    ▼
MCP CLIENT passes result to HOST
```

### Phase 5: Host Makes a SECOND API Call

This is the step most people miss. The host now makes **another API call** to Anthropic, injecting the tool result into the conversation history:

```json
{
  "messages": [
    {
      "role": "user",
      "content": "What tables are in my database?"
    },
    {
      "role": "assistant",
      "content": [
        {
          "type": "tool_use",
          "id": "call_abc123",
          "name": "list_tables",
          "input": {}
        }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "call_abc123",
          "content": "users, orders, products, inventory"
        }
      ]
    }
  ]
}
```

The `tool_result` message is "invisible" to the user — they never typed it. **The host injects it** into the conversation as a user-role message so the model can see the data.

### Phase 6: The Model Writes the Final Response

Claude now sees the original question plus the tool result. It generates:

> "Your database has four tables: users, orders, products, and inventory."

This comes back as a normal text response, and the host renders it in the chat UI.

### The Complete Flow Diagram

```
USER: "What tables are in my database?"
  │
  ▼
┌─────────────────────────────────────────────────┐
│                 HOST APPLICATION                  │
│                                                   │
│  1. Build API Request #1                          │
│     (user message + tool definitions)             │
│                    │                              │
│                    ▼                              │
│  2. Call Anthropic API ─────────► Anthropic API   │
│                                   Claude reasons  │
│                                   Returns tool_use│
│                    ◄─────────────                 │
│                    │                              │
│  3. Parse response: tool_use found!               │
│     Look up which MCP Client owns this tool       │
│                    │                              │
│  4. Route to MCP Client                           │
│                    │                              │
│         ┌──────────┘                              │
│         ▼                                         │
│    MCP Client ──── JSON-RPC ────► MCP Server      │
│                                   (executes work) │
│    MCP Client ◄─── JSON-RPC ──── MCP Server       │
│         │                         (returns result)│
│         └──────────┐                              │
│                    ▼                              │
│  5. Inject tool_result into conversation          │
│                                                   │
│  6. Build API Request #2                          │
│     (message + tool_use + tool_result)            │
│                    │                              │
│                    ▼                              │
│  7. Call Anthropic API ─────────► Anthropic API   │
│                                   Claude sees data│
│                                   Returns text    │
│                    ◄─────────────                 │
│                    │                              │
│  8. Render text response in chat UI               │
│                                                   │
└─────────────────────────────────────────────────┘
  │
  ▼
USER sees: "Your database has four tables: users, orders,
            products, and inventory."
```

---

## 7. Is the Host an "Agent"?

**No.** The host is an orchestration loop, not an autonomous agent. It follows a deterministic pattern:

```
loop:
    1. Send messages + tools to API
    2. Receive response
    3. If response contains tool_use:
         a. Route tool call to correct MCP Client
         b. Get result
         c. Inject result into conversation
         d. Go to step 1 (call API again)
    4. If response contains only text:
         a. Render in UI
         b. Wait for next user message
```

The "agentic" behavior emerges from this loop — the model might chain multiple tool calls — but the host itself is just mechanical orchestration code. **All reasoning comes from the model.** The host just follows instructions.

---

## 8. Scope: What's Shared vs. Per-Chat

| Component | Scope | Explanation |
|-----------|-------|-------------|
| MCP Client connections | **Shared across all chats** | Established at app startup, available to every conversation |
| Tool registry | **Shared across all chats** | All chats see the same available tools |
| Conversation history | **Per chat** | Each chat has its own independent message array |
| Tool results | **Per chat** | Results are injected into that specific chat's history |

Two open chats both have access to the same MCP tools, but each maintains its own independent conversation context.

---

## 9. The Generic Client

Because MCP is an RFC-like specification, the client implementation is essentially identical for every server connection. Anthropic's official SDKs (TypeScript and Python) implement the client side of the spec. When the host connects to any server, it uses the same client code — only the configuration differs:

- Which server to connect to
- Which transport to use (stdio vs HTTP)
- Environment variables or arguments

The spec defines optional capabilities (like `sampling`, where a server can request model completions). Hosts may choose which optional features to support. But the core protocol machinery — initialization, capability negotiation, tool listing, tool calling — is universal.

---

## 10. Key Takeaways

1. **The host is the orchestrator of everything.** It makes API calls, routes tool calls, injects results, and controls the UI.

2. **The model knows nothing about MCP.** It just sees tool definitions and decides to use them. Standard LLM tool-calling.

3. **MCP clients are generic.** Same code, different configuration. The protocol is the contract.

4. **Language doesn't matter.** JSON-RPC over a transport means any language can implement either side.

5. **Each user message can trigger multiple API calls.** One to get the model's reasoning, potentially more as tool results get injected and the model reasons again.

6. **The connection is stateful but idle.** After initialization, servers just wait for tool calls. They don't push data unless using optional subscription features.
