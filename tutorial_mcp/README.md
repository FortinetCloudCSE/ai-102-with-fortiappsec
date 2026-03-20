# MCP Glass Box Demo

An interactive browser-based simulation that visualizes the **Model Context Protocol (MCP)** architecture in real time. Step through every JSON-RPC message, API call, and state change as an MCP host boots, connects to servers, and orchestrates tool calls.

## Quick Start

Open `index.html` in any modern browser. That's it — no install, no build step, no server required.

> The file loads React, Babel, and Tailwind from CDN, so you need an internet connection on first load.

## What You'll See

The demo walks through **4 acts**:

| Act | Title | What Happens |
|-----|-------|-------------|
| 1 | Boot Sequence | Host reads config, spawns MCP clients, connects to servers via stdio, runs initialize handshakes, discovers tools |
| 2 | Simple Question | User asks a question the model answers without tools — a clean baseline |
| 3 | Tool-Triggered Question | A question that triggers `read_file` — the full MCP round-trip through client, server, and back |
| 4 | Chained Tool Calls | Two tools, two different servers, three API calls — the host orchestration loop in full view |

Each step shows:
- **Architecture Diagram** — animated SVG with highlighted components and message flow packets
- **Chat Panel** — Claude-style conversation UI with typing indicators
- **Message Log** — every JSON-RPC and API message in sequence
- **Payload Inspector** — full JSON payloads with syntax highlighting and diff view
- **Step Narrator** — explanation of what's happening and why

## Controls

| Input | Action |
|-------|--------|
| **Next / Back** buttons | Step forward or backward |
| **Play** button | Auto-advance through steps |
| Arrow keys | Navigate steps |
| `1` `2` `3` `4` | Jump to act |
| `P` | Toggle play/pause |
| `Esc` | Reset to beginning |

## Project Structure

```
index.html              # Self-contained demo (open directly in browser)
prompts/                # Planning and architecture docs used to build the demo
```

## Tech Stack

- **React 18** — UI rendering (CDN)
- **Babel Standalone** — in-browser JSX transpilation (CDN)
- **Tailwind CSS** — utility styles (CDN)
- **Zero dependencies** — no npm, no node_modules, no bundler

## Disclaimer

This is an **educational demo** designed to illustrate how MCP works at a conceptual level. The JSON payloads, API messages, and protocol exchanges shown are simplified representations meant to teach the flow and architecture — not to serve as exact protocol references. Do not treat the specific payload contents as authoritative specifications.

## License

MIT
