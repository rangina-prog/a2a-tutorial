# 07 — A2A vs MCP

A2A and MCP (Model Context Protocol) are the two most prominent open standards in the agentic AI ecosystem. They solve **different problems** and are designed to be **complementary**, not competing.

---

## The Core Distinction

```
                    ┌─────────────────────────────────┐
                    │         Agent System             │
                    │                                  │
  ┌──────────┐     │  ┌──────────┐     ┌──────────┐  │     ┌──────────┐
  │  Remote   │◀───A2A───▶│  Agent   │◀───MCP───▶│  Tool   │  │     │ Database │
  │  Agent    │     │  │  (LLM)   │     │ Server  │  │     │  API     │
  └──────────┘     │  └──────────┘     └──────────┘  │     └──────────┘
                    │                                  │
                    └─────────────────────────────────┘

  ◀─── A2A ───▶                    ◀─── MCP ───▶
  Agent ↔ Agent                    Agent ↔ Tool
  (horizontal)                     (vertical)
```

- **MCP** operates **vertically** — it connects an agent to its tools, APIs, and data sources. It standardizes how an agent calls a function, queries a database, or reads a file.
- **A2A** operates **horizontally** — it connects independent agents to each other. It standardizes how agents discover, communicate, delegate tasks, and collaborate.

---

## Side-by-Side Comparison

| Aspect | A2A | MCP |
|--------|-----|-----|
| **Full name** | Agent-to-Agent Protocol | Model Context Protocol |
| **Created by** | Google (donated to Linux Foundation) | Anthropic |
| **Purpose** | Agent ↔ Agent communication | Agent ↔ Tool communication |
| **Relationship model** | Peer-to-peer | Client-server |
| **Communication style** | Task-based with state machine | Request-response |
| **Discovery** | Agent Cards (JSON manifests) | Tool manifests / capabilities |
| **Transport** | HTTP, gRPC, SSE, webhooks | JSON-RPC over stdio/HTTP/SSE |
| **State management** | Stateful tasks with lifecycle | Stateless tool calls |
| **Async support** | Native (streaming, push, polling) | Limited (primarily sync) |
| **Opacity** | Agents are opaque black boxes | Tools expose their interface |
| **Use case** | Agent delegates work to another agent | Agent calls an external API or tool |

---

## When to Use Each

### Use MCP When...

You need an agent to **use a tool or access a resource**:

- Query a database
- Call a REST API
- Read/write files
- Execute code
- Search the web
- Access a knowledge base

MCP is the standard way to give an agent capabilities. The agent knows exactly what the tool does (via its schema) and calls it directly.

### Use A2A When...

You need an agent to **collaborate with another agent**:

- Delegate a sub-task to a specialist agent
- Orchestrate multiple agents in a workflow
- Route requests to the best-suited agent
- Enable cross-vendor agent interoperability
- Support long-running, multi-turn collaborative work

A2A treats the remote agent as an opaque peer — the client doesn't know (or need to know) what tools, models, or logic the remote agent uses internally.

### Use Both Together

The most powerful architectures combine both:

```
User
  │
  ▼
┌─────────────────┐
│ Orchestrator     │
│ Agent            │
│                  │
│  Uses MCP to     │──── MCP ────▶ Calendar API
│  access tools    │──── MCP ────▶ Email Service
│                  │
│  Uses A2A to     │──── A2A ────▶ Research Agent
│  delegate work   │──── A2A ────▶ Code Review Agent
│                  │──── A2A ────▶ Data Analysis Agent
└─────────────────┘
```

The orchestrator agent uses **MCP** to directly access tools it controls, and **A2A** to delegate complex sub-tasks to specialized agents it doesn't control.

---

## Architectural Differences

### Discovery

| | A2A | MCP |
|-|-----|-----|
| **Mechanism** | Agent Card at `/.well-known/agent-card.json` | Tool manifest / `tools/list` |
| **Content** | Identity, skills, auth, interfaces, capabilities | Tool name, description, input schema |
| **Scope** | What the agent can do (high-level skills) | What the tool does (specific function) |

### Communication Pattern

**MCP: Direct function call**
```json
// Client calls a tool
{
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "city": "Seattle" }
  }
}
// Tool returns result immediately
{
  "result": { "temperature": 55, "condition": "rainy" }
}
```

**A2A: Task-based collaboration**
```json
// Client sends a message to an agent
{
  "method": "SendMessage",
  "params": {
    "message": {
      "role": "ROLE_USER",
      "parts": [{ "text": "Research the weather patterns in Seattle and write a summary" }]
    }
  }
}
// Agent creates a task, may take time, may ask for clarification
{
  "result": {
    "task": {
      "id": "task-123",
      "status": { "state": "TASK_STATE_WORKING" }
    }
  }
}
```

### State Management

- **MCP:** Stateless. Each tool call is independent. Context is managed by the calling agent.
- **A2A:** Stateful. Tasks have a lifecycle (submitted → working → completed). Context persists across multi-turn interactions via `contextId`.

### Opacity

- **MCP:** Tools are transparent — the agent sees the tool's schema, knows its parameters, and gets structured results.
- **A2A:** Agents are opaque — the client only sees the Agent Card's declared skills. The remote agent's internal tools, models, and reasoning are hidden.

---

## Complementary Standards in the Ecosystem

The A2A specification explicitly acknowledges MCP as complementary:

> **MCP** provides agent-to-tool communication — a standard for how an agent connects to its tools, APIs, and resources.
>
> **A2A** provides agent-to-agent communication — a universal, decentralized standard that allows AI agents to interoperate, collaborate, and share findings.

Other related standards:

| Standard | Focus | Relationship to A2A |
|----------|-------|---------------------|
| **MCP** (Anthropic) | Agent ↔ Tool | Complementary — vertical vs horizontal |
| **ACP** (IBM) | Agent communication | Concepts incorporated into A2A |
| **agntcy** (Cisco) | Internet of Agents framework | Uses A2A and MCP for communication |

---

## Decision Framework

```
Do you need to...

├── Call a specific function/API/tool?
│   └── Use MCP
│
├── Delegate work to another autonomous agent?
│   └── Use A2A
│
├── Build a multi-agent system with specialized agents?
│   └── Use A2A for agent coordination + MCP for tool access
│
├── Give your agent access to databases/files/APIs?
│   └── Use MCP
│
├── Enable cross-vendor agent interoperability?
│   └── Use A2A
│
└── Support long-running, multi-turn collaborative tasks?
    └── Use A2A
```

---

*Sources: [A2A Protocol Home](https://a2a-protocol.org/latest/), [A2A Specification Appendix B](https://a2a-protocol.org/latest/specification/), [A2A GitHub README](https://github.com/a2aproject/A2A), [DigitalOcean A2A vs MCP](https://www.digitalocean.com/community/tutorials/a2a-vs-mcp-ai-agent-protocols), [WorkOS MCP vs A2A](https://workos.com/blog/mcp-vs-a2a). Content was rephrased for compliance with licensing restrictions.*
