# The Three-Layer Architecture — Deep Dive

This page unpacks the three-layer architecture of the A2A specification, clarifies what "client" and "server" actually mean in practice, explains how enforceable the spec really is, and walks through the Agent Card in concrete implementation terms.

---

## What Are We Talking About When We Say "A2A Client" or "A2A Server"?

1. An **A2A Server** is any software that exposes an HTTP (or gRPC) endpoint and responds to A2A protocol messages. It is the agent that *receives work*. It could be a Python app running on a cloud VM, a Docker container on AWS Bedrock, a serverless function — anything. The only requirement is that it speaks the A2A protocol on the wire. Internally, it can use any LLM, any framework, any tools. The outside world never sees those internals.

2. An **A2A Client** is any software that *sends work* to an A2A Server. It could be a chat UI, a CLI tool, an orchestrator agent, or another A2A Server that is delegating a sub-task. The client discovers the server's capabilities (via the Agent Card), sends messages, and receives results.

3. The same piece of software can be **both** a client and a server at the same time. For example, an orchestrator agent acts as a *server* to the user (it receives the user's request) and as a *client* to downstream specialist agents (it delegates sub-tasks to them). This is a common pattern in multi-agent systems:

```
User (human)
  │
  ▼
┌──────────────────────┐
│  Orchestrator Agent   │  ← A2A Server (to the user)
│                       │  ← A2A Client (to the specialists)
│  Receives user request│
│  Delegates sub-tasks  │
└───┬──────────┬────────┘
    │          │
    ▼          ▼
┌────────┐  ┌────────┐
│Research│  │ Code   │     ← A2A Servers (to the orchestrator)
│ Agent  │  │ Agent  │
└────────┘  └────────┘
```

4. There is no special binary or runtime you install to become "an A2A Client" or "an A2A Server." You just implement the protocol in your application. The official SDKs (Python, JS, Go, Java, .NET) give you typed models and helpers so you don't have to hand-roll JSON-RPC framing, but they are convenience libraries, not mandatory runtimes.

5. In short: **server = the agent doing the work**, **client = whoever asked it to do the work**. The labels describe a role in a particular interaction, not a permanent identity.

---

## The Three-Layer Architecture: Guideline or Enforcement?

### What the three layers are

6. The A2A specification organizes itself into three layers. This is the structure of *the spec document itself* — it's how the protocol designers chose to separate concerns so that the standard is clean and extensible.

7. **Layer 1 — Canonical Data Model.** This defines the data structures: Task, Message, Part, Artifact, AgentCard, Extension, and so on. These are defined as Protocol Buffer messages in a file called `spec/a2a.proto`. This `.proto` file is the single normative (authoritative) source of truth for the entire protocol. Everything else — JSON schemas, SDK types, documentation — is derived from it.

8. **Layer 2 — Abstract Operations.** This defines *what you can do* with those data structures: Send a Message, Get a Task, Cancel a Task, Subscribe to updates, etc. These operations are described in plain language, independent of any wire protocol. They say things like "the Send Message operation takes a SendMessageRequest and returns either a Task or a Message." They don't say *how* that request travels over the network.

9. **Layer 3 — Protocol Bindings.** This is where the abstract operations get mapped to concrete wire protocols. The spec defines three official bindings:
   - **JSON-RPC 2.0** — the most common, uses HTTP POST with JSON-RPC framing
   - **gRPC** — high-performance, uses Protocol Buffers over HTTP/2
   - **HTTP+JSON/REST** — RESTful endpoints with standard HTTP verbs

10. The reason for this separation is practical: if someone wants to add a WebSocket binding or a MQTT binding in the future, they only need to define Layer 3 for that transport. Layers 1 and 2 stay the same. The core semantics don't change just because the bytes travel differently.

### Is it enforced?

11. The A2A specification is a **voluntary open standard**, not a law or a compiled library that forces compliance. Nobody will arrest you for ignoring it. It is similar in nature to HTTP (RFC 9110), JSON-RPC (the JSON-RPC 2.0 spec), or OpenAPI — these are documents that describe how things *should* work so that independent implementations can interoperate.

12. However, the spec uses formal requirement language from RFC 2119: words like **MUST**, **SHOULD**, and **MAY** have precise meanings. When the spec says "Agents MUST authenticate every incoming request," it means that a conforming implementation is required to do this. If you skip it, your implementation is non-conforming and other A2A clients/servers may not work correctly with it.

13. Enforcement happens through **interoperability**. If your server doesn't serve a valid Agent Card, clients can't discover it. If your server doesn't return proper JSON-RPC responses, clients will get parse errors. If you don't implement the required operations, clients that depend on them will fail. The protocol is self-enforcing in the same way that HTTP is: you can technically send garbage over a TCP socket, but no browser will understand it.

14. The official SDKs provide **typed models and validation** that help you stay conformant. If you use the Python SDK's `Task` class, it will enforce required fields. If you use the gRPC binding, the `.proto` file enforces the schema at compile time. So while the spec itself is a document, the tooling around it provides practical enforcement.

15. To summarize: the three-layer architecture is the **design of the specification**, not something you "install." You implement it by building software that follows the rules. The SDKs and proto definitions help you get it right. Interoperability with other A2A implementations is your test of conformance.

---

## The Agent Card: What It Is, Where It Comes From, How You Implement It

### What is it?

16. The Agent Card is a **JSON document** that describes everything a client needs to know about a server before talking to it. Think of it as a combination of a business card and an API spec. It tells clients: "Here's who I am, here's what I can do, here's how to authenticate, and here's where to send requests."

17. The Agent Card is **part of the A2A protocol**. It is not optional. The spec says: "A2A Servers MUST make an Agent Card available." If you're building an A2A server, you need to serve an Agent Card.

18. It is always a **JSON object** — not YAML, not XML, not a binary format. The structure is defined by the `AgentCard` message in `spec/a2a.proto`, and when serialized to JSON it uses camelCase field names.

### Where does it come from?

19. **You create it.** The Agent Card is authored by the developer or team that builds the A2A server. It is a static (or semi-static) configuration artifact that describes your agent. You write it by hand, generate it from your agent's configuration, or let your framework produce it.

20. There is no central registry that issues Agent Cards. Each server is responsible for its own card. You define the content and serve it from your server.

### Where is it served?

21. The standard discovery mechanism is a **well-known URI**. Clients expect to find the Agent Card at:

```
https://{your-domain}/.well-known/agent-card.json
```

22. This follows the same pattern as other well-known URIs on the web (like `/.well-known/openid-configuration` for OIDC or `/.well-known/security.txt`). A client that knows your domain can fetch the card without any prior configuration.

23. There are two other discovery paths: **registries** (curated catalogs where agents are listed, like an app store for agents) and **direct configuration** (someone gives you the card URL or the card content directly). But the well-known URI is the default.

### How do you implement it?

24. At its simplest, you add a GET endpoint to your web server that returns a JSON object. Here is a minimal but complete example:

```json
{
  "name": "Invoice Processing Agent",
  "description": "Extracts data from invoice PDFs and returns structured JSON.",
  "version": "1.0.0",
  "supportedInterfaces": [
    {
      "url": "https://invoices.example.com/a2a",
      "protocolBinding": "JSONRPC",
      "protocolVersion": "1.0"
    }
  ],
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "defaultInputModes": ["application/pdf", "image/png"],
  "defaultOutputModes": ["application/json"],
  "skills": [
    {
      "id": "extract-invoice-data",
      "name": "Invoice Data Extraction",
      "description": "Accepts an invoice PDF or image and returns structured line items, totals, and vendor info as JSON.",
      "tags": ["invoice", "extraction", "pdf", "ocr"],
      "examples": [
        "Extract line items from this invoice PDF",
        "Parse this scanned invoice image"
      ]
    }
  ]
}
```

25. In a Python FastAPI server, serving this looks like:

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()

AGENT_CARD = {
    "name": "Invoice Processing Agent",
    "description": "Extracts data from invoice PDFs and returns structured JSON.",
    "version": "1.0.0",
    "supportedInterfaces": [
        {
            "url": "https://invoices.example.com/a2a",
            "protocolBinding": "JSONRPC",
            "protocolVersion": "1.0"
        }
    ],
    "capabilities": {
        "streaming": False,
        "pushNotifications": False
    },
    "defaultInputModes": ["application/pdf", "image/png"],
    "defaultOutputModes": ["application/json"],
    "skills": [
        {
            "id": "extract-invoice-data",
            "name": "Invoice Data Extraction",
            "description": "Accepts an invoice PDF or image and returns structured line items.",
            "tags": ["invoice", "extraction", "pdf", "ocr"],
            "examples": ["Extract line items from this invoice PDF"]
        }
    ]
}

@app.get("/.well-known/agent-card.json")
async def agent_card():
    return JSONResponse(content=AGENT_CARD)
```

26. That's it. When a client hits `https://invoices.example.com/.well-known/agent-card.json`, it gets back the JSON, reads the `supportedInterfaces` to find the endpoint URL and protocol, reads `skills` to understand what the agent can do, and then starts sending messages to `https://invoices.example.com/a2a`.

### What does the client do with it?

27. The client uses the Agent Card to answer three questions before sending any work:
   - **Can this agent do what I need?** → Check `skills`, `tags`, `description`
   - **How do I talk to it?** → Check `supportedInterfaces` for the URL and protocol binding
   - **How do I authenticate?** → Check `securitySchemes` and `security`

28. In a multi-agent system, an orchestrator might fetch Agent Cards from several agents, compare their skills, and route the user's request to the best match. The `tags` and `examples` fields on skills are designed to make this programmatic matching possible.

### Does the Agent Card change at runtime?

29. Generally, no. The Agent Card is a relatively static document — it changes when you deploy a new version of your agent, add a skill, or update your auth requirements. It does not change per-request or per-user.

30. The one exception is the **Extended Agent Card**. If your card declares `capabilities.extendedAgentCard: true`, authenticated clients can call a separate endpoint (`GET /extendedAgentCard`) to get a richer version of the card. This extended card might include additional skills or configuration that are only available to authenticated users. But even this is a semi-static document, not a per-request dynamic response.

31. The spec recommends using standard HTTP caching (`Cache-Control`, `ETag`) on the Agent Card endpoint so clients don't re-fetch it unnecessarily.

---

## Putting It All Together

32. Here is the full picture of how the three layers, the client/server roles, and the Agent Card connect in a real interaction:

```
Step 1: DISCOVERY
─────────────────
Client                                         Server
  │  GET /.well-known/agent-card.json            │
  │─────────────────────────────────────────────▶│
  │◀─────────────────────────────────────────────│
  │  200 OK { "name": "...", "skills": [...],    │
  │           "supportedInterfaces": [...] }     │
  │                                              │
  │  Client reads the card:                      │
  │  - Picks a protocol binding (Layer 3)        │
  │  - Understands the data model (Layer 1)      │
  │  - Knows which operations to call (Layer 2)  │

Step 2: INTERACTION
───────────────────
  │  POST /a2a  (JSON-RPC binding)               │
  │  { "method": "SendMessage",                  │
  │    "params": { "message": {                  │
  │      "role": "ROLE_USER",                    │
  │      "parts": [{"text": "Parse invoice"}]    │  ← Layer 1 objects
  │    }}}                                       │  ← Layer 2 operation
  │─────────────────────────────────────────────▶│  ← Layer 3 binding
  │                                              │
  │◀─────────────────────────────────────────────│
  │  { "result": { "task": {                     │
  │      "id": "task-123",                       │
  │      "status": {"state": "TASK_STATE_COMPLETED"},
  │      "artifacts": [{"parts": [{"data": {...}}]}]
  │  }}}                                         │

Step 3: (optional) FOLLOW-UP
────────────────────────────
  │  GET /tasks/task-123  (polling)              │
  │  or POST /tasks/task-123:subscribe (stream)  │
  │  or receive webhook POST (push notification) │
```

33. Every A2A interaction follows this pattern: discover via the Agent Card, then interact using the operations and data model through a specific protocol binding. The three layers aren't separate systems you deploy — they're three aspects of every single request and response flowing between client and server.

---

*Sources: [A2A Specification §1.3, §2, §4.4, §5, §8](https://a2a-protocol.org/latest/specification/), [A2A GitHub README](https://github.com/a2aproject/A2A). Content was rephrased for compliance with licensing restrictions.*
