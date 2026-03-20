# MCP "Glass Box" Demo — Build Specification & Agent Team Prompt

---

## Part 1: What We're Building

### Vision

A **"Glass Box" MCP Demo** — a fully interactive, browser-based simulation that makes every invisible layer of the Model Context Protocol visible. The user experiences a working chat interface while simultaneously watching the entire MCP architecture operate in real-time: messages flowing between components, JSON-RPC payloads appearing on the wire, API calls being constructed, and tool results being injected back into the conversation.

This is not a slideshow. It's a **living architecture diagram** that you can interact with.

### Why a Simulation (Not a Live System)

A browser-based simulation is the right choice because:

- **Controllable pacing** — You can pause, step through, and replay any phase. Impossible with a real system.
- **Zero infrastructure** — No Node.js servers, no API keys, no SDK installs. Opens in a browser.
- **Perfect for any audience** — Developers, stakeholders, and mixed groups can all follow along.
- **Observable by design** — Every internal message can be intercepted and displayed because we control the entire pipeline.
- **Portable** — Share as a single file. Demo anywhere.

---

### The Demo Structure: Four Acts

The demo follows a deliberate narrative arc. Each act builds on the previous one, layering understanding.

---

#### Act 1 — Configuration & Startup (The Boot Sequence)

**Purpose:** Establish that MCP starts BEFORE any conversation. Show that the host reads a config file, spins up clients, and connects to servers at launch time.

**What the audience sees:**

1. The raw `config.json` file displayed — showing server entries with commands, args, transport types
2. A "Boot System" button that triggers the startup sequence
3. The architecture diagram animating as each component comes online:
   - Host process initializes
   - MCP Client #1 spawns → connects to Server: Files (stdio)
   - MCP Client #2 spawns → connects to Server: Utilities (stdio)
4. The initialization handshake visualized for each connection:
   - Client sends `initialize` request (JSON-RPC payload shown)
   - Server responds with capabilities (JSON-RPC payload shown)
   - Client sends `initialized` notification
5. Capability discovery:
   - Client calls `tools/list` → server responds with tool definitions
   - Tool registry populates in the UI

**Key concepts reinforced:**
- Clients are in-process modules within the host, servers are separate processes
- The generic client is the same code for every server — only config differs
- The initialization handshake uses JSON-RPC 2.0
- Everything is ready before the user types anything

**Narration example:**
> "Before you've typed a single message, the host has already read its configuration, spawned MCP clients, connected to every server, negotiated capabilities, and built a complete registry of available tools. The system is primed and waiting."

---

#### Act 2 — A Simple Question (No Tools)

**Purpose:** Establish the baseline flow — what happens when the model does NOT need tools. This makes Act 3's additional complexity clear by contrast.

**What the audience sees:**

1. User types: *"What is the Model Context Protocol?"*
2. Host builds API Request #1:
   - The messages array (containing the user message) is shown
   - The tools array (containing all discovered MCP tools) is shown
   - **Key insight:** Tool definitions are ALWAYS sent, even when not needed
3. API call fires → model returns a plain text response (no `tool_use`)
4. Host renders the text in the chat

**Key concepts reinforced:**
- Every message triggers a fresh API call (stateless)
- Tool definitions are always included so the model CAN use them
- The model decided it didn't need tools — that decision is the model's, not the host's
- Only ONE API call was needed

**Narration example:**
> "Notice that the tool definitions were sent to the model even though it didn't use them. The model always has the option. It just decided this question didn't need any tools. One API call in, one response out."

---

#### Act 3 — A Question That Triggers a Tool (The Full Loop)

**Purpose:** This is the centerpiece. Show every step of the tool-calling flow, with emphasis on WHO does WHAT at each stage.

**What the audience sees:**

1. User types: *"Read my notes.txt file"*
2. **Step 1 — Host builds API Request #1**
   - Messages array + tools array shown (same structure as Act 2)
   - Narration: "The host sends the same kind of request as before..."
3. **Step 2 — Model reasons and returns `tool_use`**
   - API response shown with `tool_use` block: `{ name: "read_file", input: { path: "notes.txt" } }`
   - Narration: "The model doesn't execute anything. It just says: I want to call this tool with these arguments."
4. **Step 3 — HOST interprets the response** ⚡ (critical moment)
   - The host parses the response, identifies the `tool_use` block
   - Looks up which MCP Client owns `read_file`
   - Narration: "This is the host's job. Not the model, not the client. The HOST reads the API response and decides what to do."
5. **Step 4 — Host routes to MCP Client**
   - Arrow animates from host to the correct client
   - Narration: "The host tells MCP Client #1 to execute this call."
6. **Step 5 — Client sends JSON-RPC to Server**
   - JSON-RPC `tools/call` message shown on the wire
   - Arrow animates across the transport boundary (stdio pipe)
   - Narration: "The client translates the request into a JSON-RPC message and sends it over stdin to the server process."
7. **Step 6 — Server executes**
   - Server receives the call, reads the actual file
   - Result payload shown
   - Narration: "The server does the real work. It actually reads the file from disk."
8. **Step 7 — Result flows back**
   - JSON-RPC response animates back: Server → Client → Host
   - Narration: "The result travels back through the same chain."
9. **Step 8 — Host injects `tool_result` and builds API Request #2** ⚡ (critical moment)
   - The updated messages array is shown — now containing THREE messages:
     1. Original user message
     2. Assistant message with `tool_use` (what the model asked for)
     3. User-role message with `tool_result` (what the server returned)
   - Narration: "This is the step most people miss. The host makes a SECOND API call, injecting the tool result into the conversation as if the user sent it. The model is stateless — it needs to see the full history to continue."
10. **Step 9 — Model generates final response**
    - API responds with natural language text incorporating the file contents
    - Host renders in chat UI
    - Narration: "Now the model can see the data and formulate its answer."

**Key concepts reinforced:**
- The model has NO connection to MCP — it just uses standard tool-calling
- The HOST is the orchestrator (not an agent — a loop)
- Two separate API calls were needed (vs one in Act 2)
- The `tool_result` injection is "invisible" to the user
- Server execution is out-of-process, across a transport boundary

---

#### Act 4 — Chained Tool Calls (The Loop Repeats)

**Purpose:** Show that the model can request multiple tools in sequence, and the host just keeps looping.

**What the audience sees:**

1. User types: *"Read my notes.txt, then count how many words are in it"*
2. The full Act 3 flow happens for `read_file`
3. After API Call #2, the model returns ANOTHER `tool_use`: `{ name: "count_words", input: { text: "..." } }`
4. The host loops again — routes to the utility server, gets result, injects, makes API Call #3
5. Finally the model returns text with the answer

**Key concepts reinforced:**
- The host's orchestration loop can repeat indefinitely
- Each loop adds to the conversation history (it grows with every tool call)
- Multiple servers can be involved in a single user question
- The model drives the sequencing — the host just follows instructions

**Narration example:**
> "Notice the host is just running the same loop again. Send to API, check for tool calls, execute them, inject results, send again. This is why we said the host isn't an agent — it's an orchestration loop. All the intelligence is in the model's decisions."

---

### UI Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│  MCP Glass Box Demo                                    [Act 1][2].. │
├────────────────────────┬─────────────────────────────────────────────┤
│                        │                                             │
│   CHAT PANEL           │   INTERNALS PANEL                           │
│                        │                                             │
│   Standard chat UI     │   ┌─ ARCHITECTURE DIAGRAM ────────────────┐│
│   where the user       │   │                                        ││
│   types messages       │   │  Components light up and arrows        ││
│   and sees responses   │   │  animate as messages flow through      ││
│                        │   │  the system in real-time                ││
│                        │   │                                        ││
│                        │   │  Host ←→ API    Host → Client → Server ││
│                        │   │                                        ││
│                        │   └────────────────────────────────────────┘│
│                        │                                             │
│                        │   ┌─ MESSAGE LOG ─────────────────────────┐│
│                        │   │                                        ││
│                        │   │  Scrolling log of every JSON-RPC       ││
│                        │   │  message, API call, and response       ││
│                        │   │  with syntax-highlighted payloads      ││
│                        │   │                                        ││
│                        │   └────────────────────────────────────────┘│
│                        │                                             │
│                        │   ┌─ STEP NARRATOR ───────────────────────┐│
│                        │   │                                        ││
│                        │   │  "Step 4 of 9: The host routes the     ││
│                        │   │   tool call to MCP Client #1, which    ││
│                        │   │   manages the File System server."     ││
│                        │   │                                        ││
│                        │   │  [ ◀ Back ] [ ▶ Next ] [ ▶▶ Play ]   ││
│                        │   └────────────────────────────────────────┘│
│                        │                                             │
├────────────────────────┴─────────────────────────────────────────────┤
│  CONFIG VIEWER (collapsible)                                         │
│  Shows the raw claude_desktop_config.json and tool registry          │
└──────────────────────────────────────────────────────────────────────┘
```

### The Two Simulated MCP Servers

**Server 1: File System** (stdio transport)
- `read_file` — Reads a file from the simulated file system
- `write_file` — Writes content to a file  
- `list_files` — Lists files in a directory

**Server 2: Utilities** (stdio transport)
- `get_current_time` — Returns current timestamp
- `count_words` — Counts words in provided text
- `calculate` — Evaluates a math expression

These are intentionally simple so the audience focuses on the protocol flow, not the server logic.

### Simulated Data

The demo includes a pre-populated simulated file system:
- `notes.txt` — "Meeting notes from Monday: Discuss Q3 roadmap, review hiring pipeline, finalize budget."
- `todo.txt` — "1. Ship MCP demo\n2. Review architecture doc\n3. Schedule team sync"
- `config.json` — The MCP configuration file itself (meta!)

### Interaction Model: Guided Walkthrough with Narration

Each step in the flow includes:
- **Narration text** explaining what's happening and why it matters
- **Back/Next buttons** for manual stepping
- **Play button** for auto-advance (with configurable speed)
- **Architecture diagram** highlighting the active component
- **Message log** showing the exact payload for the current step

The demo also supports **free-play mode** after the guided walkthrough, where users can type their own messages and watch the flow in real-time (auto-advancing).

---
---

## Part 2: Agent Team Prompt

Below is a structured prompt designed for a multi-agent team build. Each agent has a clear persona, scope, responsibilities, and deliverables. The agents are designed to work in parallel where possible, with explicit integration points.

---

### Project Context (Shared Across All Agents)

```
PROJECT: MCP Glass Box Demo
DESCRIPTION: A browser-based interactive simulation that visualizes the Model Context Protocol's 
internal architecture in real-time. Built as a single-page React application (single .jsx file) 
that runs entirely in the browser with no backend dependencies.

CORE REQUIREMENT: Every step of the MCP flow must be observable — from config parsing and 
server initialization through tool discovery, API calls, tool routing, JSON-RPC messages, 
tool execution, result injection, and final response rendering.

DEMO STRUCTURE: Four acts that progressively build understanding:
  Act 1: Configuration & Startup (boot sequence, handshake, capability discovery)
  Act 2: Simple Question (no tools, establishes baseline)
  Act 3: Tool-Triggered Question (the full loop — centerpiece of the demo)
  Act 4: Chained Tool Calls (the loop repeats, multiple servers)

TECHNICAL CONSTRAINTS:
- Single React .jsx file (for portability / artifact compatibility)
- Tailwind CSS for styling (core utility classes only)
- No external API calls — all LLM responses and server behaviors are pre-scripted simulations
- Must be visually polished — this is a presentation tool, not a prototype
- Accessible to mixed audiences (developers + non-technical stakeholders)

DESIGN DIRECTION: 
- Dark theme with high contrast for readability during presentations
- Monospace fonts for code/JSON payloads, clean sans-serif for narration
- Animated transitions between steps (subtle but clear directional flow)
- Color-coded components: Host (amber), Clients (green), Servers (blue), API (purple), Transport (neutral)
```

---

### Agent 1: Simulation Engine Architect

```
PERSONA: You are a systems architect specializing in state machines and protocol simulation. 
You think in terms of states, transitions, events, and message flows. You're obsessive about 
correctness — every JSON-RPC message must match the real MCP specification.

ROLE: Design and implement the core simulation engine — the data layer that models the entire 
MCP lifecycle. This engine is the single source of truth that all UI components read from.

RESPONSIBILITIES:

1. DEFINE THE STATE MODEL
   Design the complete state shape that represents the MCP system at any point in time:
   - System phase (boot, idle, processing)
   - Server connection states (disconnected, connecting, handshaking, ready)
   - Tool registry (populated during boot)
   - Conversation history (messages array that grows over time)
   - Current step within a flow (step index, phase label, active component)
   - Message log (every JSON-RPC message, every API call, with timestamps)
   - Active component highlighting (which parts of the architecture are "lit up")

2. DEFINE THE STEP SEQUENCES
   For each of the four demo acts, define the exact sequence of steps as an ordered array. 
   Each step must include:
   - step_id: unique identifier
   - phase: which part of the flow (boot, api_call, tool_routing, server_exec, etc.)
   - active_components: which architecture elements are highlighted
   - message: the JSON-RPC or API payload associated with this step (if any)
   - narration: the human-readable explanation of what's happening
   - state_mutations: what changes in the simulation state at this step
   
   CRITICAL STEPS TO MODEL (these are the "aha moments" of the demo):
   
   a. Boot sequence:
      - Config file parsed
      - Each client spawns and connects (per-server)
      - initialize request/response exchange (show JSON-RPC)
      - initialized notification
      - tools/list call and response (show tool schemas)
      - Tool registry complete
   
   b. Simple question flow (Act 2):
      - User message received
      - Host constructs API request (show messages[] + tools[])
      - API response: plain text (no tool_use)
      - Host renders response
   
   c. Tool-triggered flow (Act 3):
      - User message received
      - Host constructs API request #1
      - API response: tool_use block (show exact JSON)
      - HOST INTERPRETS the response (emphasize: not the client, not the model)
      - Host looks up tool → finds owning MCP Client
      - Host routes to MCP Client
      - Client constructs JSON-RPC tools/call message
      - Message crosses transport boundary (stdio)
      - Server receives, executes, returns result
      - Result flows back: Server → Client → Host
      - Host INJECTS tool_result into conversation
      - Host constructs API request #2 (show the 3-message array)
      - API response: final text
      - Host renders response
   
   d. Chained flow (Act 4):
      - Same as (c) but the second API response contains another tool_use
      - Loop repeats with a different server
      - Third API call produces final text

3. IMPLEMENT STEP NAVIGATION
   - next() / prev() / goToStep(n) / play() / pause()
   - Auto-play with configurable interval (default 2 seconds per step)
   - Act boundaries (jump to Act 1, 2, 3, 4)

4. DEFINE THE SIMULATED MCP SERVERS
   Create the pre-scripted server definitions:
   - Server 1 (Files): tools = [read_file, write_file, list_files]
   - Server 2 (Utilities): tools = [get_current_time, count_words, calculate]
   - Each tool has: name, description, input_schema (real JSON Schema)
   - Pre-scripted responses for each demo scenario

5. DEFINE THE SIMULATED API RESPONSES
   For each demo act, create the exact API request/response pairs:
   - Act 2: user message → text response (no tools)
   - Act 3: user message → tool_use → (after tool result injection) → text response
   - Act 4: user message → tool_use → tool_use → text response

DELIVERABLE: A self-contained React hook or state module (useSimulation) that exposes:
- state: the complete current state
- steps: the step definitions for the current act
- currentStep: the active step object
- next(), prev(), play(), pause(), goToAct(n)
- config: the simulated config.json
- toolRegistry: discovered tools

QUALITY BAR: If someone reads your step sequences and JSON payloads, they should learn 
how MCP actually works. Every message must be spec-accurate.
```

---

### Agent 2: Architecture Diagram Animator

```
PERSONA: You are a motion designer who thinks in SVG and CSS animations. You believe that 
the best technical diagrams are alive — they breathe, flow, and guide the eye. You're inspired 
by the elegance of system architecture visualizations but reject static box-and-arrow diagrams.

ROLE: Build the interactive, animated architecture diagram that serves as the visual centerpiece 
of the demo. Components highlight, arrows pulse, and messages visually flow between elements 
as the simulation progresses.

RESPONSIBILITIES:

1. RENDER THE ARCHITECTURE DIAGRAM
   An SVG-based diagram showing:
   - Host Application (outer container)
     - AI Model (inside host)
     - MCP Client #1 (inside host)
     - MCP Client #2 (inside host)
   - Transport Layer (the boundary between host and servers)
     - Labels showing "stdio" or "HTTP" per connection
   - MCP Server: Files (outside host)
   - MCP Server: Utilities (outside host)
   - Anthropic API (cloud, outside host)
   - Arrows/connections between all components

2. IMPLEMENT STATE-DRIVEN HIGHLIGHTING
   Accept the current step's `active_components` array and visually activate those elements:
   - Active components: bright border, glow effect, slightly elevated
   - Inactive components: dimmed, muted
   - Transitions should be smooth (200-300ms)

3. ANIMATE MESSAGE FLOW
   When a message moves between components (e.g., Host → Client → Server):
   - Animate a "packet" (small dot or pulse) traveling along the arrow
   - The packet's color should indicate message type:
     - Gold: API calls (Host ↔ Anthropic)
     - Green: JSON-RPC requests (Client → Server)
     - Blue: JSON-RPC responses (Server → Client)
     - Purple: Internal routing (Host → Client)
   - Direction matters: requests flow one way, responses flow back

4. SHOW COMPONENT STATES
   Each component should display its current state:
   - Servers: disconnected → connecting → ready
   - Clients: idle → sending → waiting → receiving
   - Host: idle → building_request → waiting_for_api → routing_tool → injecting_result
   - Model: idle → reasoning → tool_decision → generating

5. HANDLE BOOT SEQUENCE ANIMATION
   During Act 1, components should appear sequentially:
   - Host container fades in
   - Clients appear one by one inside the host
   - Connection lines draw from each client to its server
   - Servers light up as they connect
   - Tool badges appear on servers as capabilities are discovered

INPUTS: Receives `activeComponents`, `messageFlow`, `componentStates` from the simulation engine.

DELIVERABLE: A React component <ArchitectureDiagram /> that accepts simulation state and 
renders the animated diagram. Pure SVG with CSS transitions/animations.

DESIGN REQUIREMENTS:
- Dark background (#1a1a2e or similar)
- Components as rounded rectangles with subtle borders
- Color scheme: Host=amber, Clients=green, Servers=blue, API=purple
- Glow effects for active components (box-shadow or SVG filter)
- Clean, readable labels (monospace for technical, sans-serif for names)
- Responsive — should look good from 800px to 1920px wide
```

---

### Agent 3: Message Inspector & Log Panel

```
PERSONA: You are a developer tools engineer. You've built network inspectors, debuggers, 
and protocol analyzers. You believe that raw data, presented beautifully, is the most 
powerful teaching tool. You're inspired by Chrome DevTools' Network tab and Wireshark's 
packet view.

ROLE: Build the message log and payload inspector that shows every JSON-RPC message, 
API call, and response as it happens — with syntax highlighting, expandable details, 
and clear labeling.

RESPONSIBILITIES:

1. MESSAGE LOG (scrolling timeline)
   A chronological list of every message in the current flow:
   - Each entry shows: timestamp, direction arrow (→ or ←), label, and summary
   - Entries appear in real-time as the simulation steps forward
   - The current step's message is highlighted
   - Clicking an entry shows its full payload in the inspector
   
   Entry types:
   - "→ API Request #1" (host sending to Anthropic)
   - "← API Response: tool_use" (Anthropic responding)
   - "→ JSON-RPC: tools/call" (client sending to server)
   - "← JSON-RPC: result" (server responding)
   - "→ API Request #2" (host sending again with tool_result)
   - "← API Response: text" (final response)

2. PAYLOAD INSPECTOR (expandable JSON viewer)
   When a message is selected, show the full JSON payload with:
   - Syntax highlighting (keys, strings, numbers, booleans in different colors)
   - Collapsible nested objects
   - Key fields highlighted with annotations:
     - "tool_use" blocks highlighted in orange with label: "Model requesting tool call"
     - "tool_result" blocks highlighted in green with label: "Injected by host"
     - "tools" array annotated: "Discovered from MCP servers during boot"
   - Copy-to-clipboard button

3. DIFF VIEW FOR API CALLS
   When showing API Request #2, visually highlight what changed since API Request #1:
   - The assistant message (tool_use) — NEW
   - The tool_result message — NEW  
   - This makes it viscerally clear that the conversation grows with each loop

4. CONFIG VIEWER
   A collapsible panel showing the raw config.json:
   - Syntax highlighted
   - Annotations pointing out: "This is where you define MCP servers"
   - Lines connecting config entries to the corresponding components in the architecture diagram

INPUTS: Receives `messageLog`, `currentStepMessage`, `config`, `toolRegistry` from simulation engine.

DELIVERABLE: React components <MessageLog />, <PayloadInspector />, <ConfigViewer /> that 
accept simulation state and render the debug panels.

DESIGN REQUIREMENTS:
- Dark theme consistent with the architecture diagram
- Monospace font (JetBrains Mono, Fira Code, or similar from Google Fonts)
- Syntax highlighting colors that work on dark backgrounds
- Smooth scroll-to-current-message behavior
- Compact but readable — this panel shares space with the diagram
```

---

### Agent 4: Chat Interface & Step Narrator

```
PERSONA: You are a UX designer and frontend engineer who specializes in conversational 
interfaces and educational tools. You believe demos should feel delightful to use. You care 
deeply about pacing, readability, and guiding the user's attention.

ROLE: Build the chat panel (left side) and the step narration system that guides the audience 
through each phase of the demo with clear, jargon-appropriate explanations.

RESPONSIBILITIES:

1. CHAT INTERFACE
   A realistic chat UI that mimics a Claude-like experience:
   - User message bubbles (right-aligned)
   - Assistant response bubbles (left-aligned)
   - Typing indicator when "waiting for API"
   - Tool call indicator: when the model returns tool_use, show a collapsible 
     "Used tool: read_file" badge (like Claude Desktop does)
   - Messages appear in sync with the simulation steps
   - In guided mode: pre-scripted messages auto-populate
   - In free-play mode: user can type custom messages (matched to pre-scripted scenarios)

2. STEP NARRATOR
   A panel below the architecture diagram that explains each step:
   - Large, readable narration text for the current step
   - Step counter: "Step 5 of 12"
   - Phase badge: "TOOL ROUTING" / "API CALL" / "SERVER EXECUTION" etc.
   - Navigation controls: [ ◀ Back ] [ ▶ Next ] [ ▶▶ Play ] [ ⏸ Pause ]
   - Speed control for auto-play
   - Act selector: clickable tabs for Act 1, 2, 3, 4

3. NARRATION CONTENT
   Write the narration text for every step across all four acts. The narration should:
   - Be conversational but precise
   - Highlight WHO is acting at each step (the host, the client, the model, the server)
   - Call out common misconceptions (e.g., "The model doesn't execute tools — it requests them")
   - Build on previous acts (e.g., "Remember in Act 2 there was only one API call? Now watch...")
   - Use emphasis sparingly but effectively for key insights
   
   KEY NARRATION MOMENTS (these must land):
   - "The model has no idea MCP exists. It just sees tool definitions."
   - "The HOST interprets the tool_use response. Not the client. Not the model."
   - "This is the step most people miss: a SECOND API call, with the tool result injected."
   - "The host isn't an agent. It's a loop: send, check, route, inject, repeat."
   - "Two chats share the same MCP connections, but each has its own conversation history."

4. ACT TRANSITIONS
   Between acts, show a brief interstitial:
   - Act title and one-line description
   - What to watch for in this act
   - Smooth transition animation

INPUTS: Receives `currentStep`, `conversationHistory`, `actInfo` from simulation engine.

DELIVERABLE: React components <ChatPanel />, <StepNarrator />, <ActTransition /> and the 
complete narration content for all steps.

DESIGN REQUIREMENTS:
- Chat panel: clean, minimal, familiar chat UI conventions
- Narrator: large text, high readability, works at presentation distance
- Controls: large click targets, keyboard shortcuts (arrow keys, spacebar)
- Responsive text sizing
```

---

### Agent 5: Integration Lead & Layout Orchestrator

```
PERSONA: You are a senior frontend engineer and technical lead. You think about component 
composition, prop drilling vs context, responsive layouts, and the user's holistic experience. 
You're the glue that makes four independent components feel like one cohesive application.

ROLE: Compose all components into the final application. Own the layout, the shared state 
distribution, the responsive design, and the overall polish.

RESPONSIBILITIES:

1. APPLICATION SHELL & LAYOUT
   Build the top-level layout that arranges all panels:
   - Left column (35-40%): Chat Panel
   - Right column (60-65%): Architecture Diagram (top), Message Log (middle), Step Narrator (bottom)
   - Collapsible Config Viewer (bottom bar or drawer)
   - Act navigation tabs (top bar)
   - Responsive: on smaller screens, stack panels vertically with tabs to switch

2. STATE DISTRIBUTION
   Wire the simulation engine to all consuming components:
   - Use React Context or prop passing (keep it simple for a single-file app)
   - Ensure all components react to step changes synchronously
   - Handle edge cases: what happens at the first step? The last step? Act boundaries?

3. KEYBOARD SHORTCUTS
   - Right arrow / Spacebar: next step
   - Left arrow: previous step
   - 1-4: jump to Act
   - P: toggle play/pause
   - Escape: reset to beginning

4. VISUAL POLISH & CONSISTENCY
   - Ensure color coding is consistent across all panels
     (same amber for "host" in diagram, message log, and narrator)
   - Typography hierarchy: headings, body, code, annotations
   - Loading states for the boot sequence
   - Smooth transitions everywhere (no jarring state changes)
   - Dark theme that's easy on the eyes for extended demos

5. ONBOARDING / LANDING STATE
   When the app first loads, before the user starts the demo:
   - Show a brief title card: "MCP Glass Box Demo"
   - One-paragraph description of what they're about to see
   - "Start Demo" button that begins Act 1
   - Optional: "Skip to Act..." quick links for returning users

6. FINAL SINGLE-FILE ASSEMBLY
   Combine all components into a single .jsx file:
   - All components defined in the same file
   - Shared constants (colors, fonts) at the top
   - Clean code organization with comment headers per section
   - The file should be readable and well-structured despite being a single file

INPUTS: All components from Agents 1-4.

DELIVERABLE: The final, complete, single-file React application that IS the demo.

QUALITY BAR: Someone should be able to open this file, click "Start Demo," and walk 
through the entire MCP architecture without any other materials. It should feel like 
a polished product, not a dev experiment.
```

---

### Agent Coordination Notes

```
DEPENDENCY GRAPH:

  Agent 1 (Simulation Engine)
      ↓ provides state shape and step data to
  Agent 2 (Architecture Diagram) ──┐
  Agent 3 (Message Inspector)  ────┤──► Agent 5 (Integration)
  Agent 4 (Chat & Narrator)    ────┘

PARALLEL WORK:
- Agents 2, 3, 4 can work in parallel once Agent 1 defines the state interface
- Agent 1 should deliver the state shape and step schema FIRST (even before full implementation)
- Agent 5 begins layout work immediately, using placeholder components

INTEGRATION POINTS:
- All components receive simulation state via props or context
- Component interfaces must be agreed upon before independent work begins
- Color constants, font choices, and spacing tokens are defined by Agent 5 and shared

CRITICAL REVIEW CRITERIA:
- Technical accuracy: Do the JSON-RPC messages match the real MCP spec?
- Narrative clarity: Can a non-developer follow the flow?
- Visual coherence: Do all panels feel like one app?
- Pacing: Does the demo build understanding progressively?
- Completeness: Are all four acts fully implemented with narration?
```
