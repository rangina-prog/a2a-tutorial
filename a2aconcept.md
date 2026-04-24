# A2A Concepts & Task Lifecycle

---

## Table of Contents

- [A2A Client and Server](#a2a-client-and-server)
  - [A2A Client](#1-a2a-client)
  - [A2A Server (Remote Agent)](#2-a2a-server-remote-agent)
  - [Client Side — Application Code vs Library Internals](#client-side--application-code-vs-library-internals)
  - [Server Side — Application Code vs Framework Internals](#server-side--application-code-vs-framework-internals)
  - [An Agent Can Be Both Client and Server](#an-agent-can-be-both-client-and-server)
- [Quick Reference: A2A JSON-RPC Methods](#quick-reference-a2a-json-rpc-methods)
- [Task Lifecycle](#task-lifecycle)
  - [Task State Machine](#task-state-machine)
  - [State Definitions](#state-definitions)
  - [TaskStatus Object](#taskstatus-object)
- [Multi-Turn Interactions](#multi-turn-interactions)
  - [Context ID](#context-id-contextid)
  - [Task ID](#task-id-taskid)
  - [Conversation Patterns](#conversation-patterns)
  - [Rules for Multi-Turn](#rules-for-multi-turn)
- [Task Update Delivery](#task-update-delivery)
  - [Polling (Get Task)](#1-polling-get-task)
  - [Streaming (SSE / gRPC)](#2-streaming-sse--grpc)
  - [Push Notifications (Webhooks)](#3-push-notifications-webhooks)
- [Streaming Event Types](#streaming-event-types)
- [Execution Mode](#execution-mode)
- [History Length](#history-length)

---

## A2A Client and Server

### 1. A2A Client

The agent that **initiates requests** to an A2A Server. A client is an application, an agent, or any software that needs to delegate work to another remote agent.

Responsibilities:

- Discover agents via their Agent Cards
- Send messages to initiate or continue tasks
- Receive results via polling, streaming, or push notifications

### 2. A2A Server (Remote Agent)

An agent that **exposes an A2A-compliant endpoint**. It processes incoming messages, manages tasks, and returns results. The server's internal implementation is opaque — it could be powered by any LLM, framework, or custom logic.

An A2A Server responds to A2A protocol messages. The only requirement is that it speaks the A2A protocol on the wire.

| | Client Library | Server Framework |
|:---|:---|:---|
| **Purpose** | Send requests to another agent | Receive requests from other agents |
| **Direction** | Outbound — the agent calls out | Inbound — other agents call in |
| **What it handles** | Building JSON-RPC requests, parsing responses | Routing JSON-RPC requests to agent logic, building responses |



Let's walk through what's happening:

**Step 1 — Discovery (plain HTTP GET, no JSON-RPC).** Agent A knows Agent B's domain. It fetches the Agent Card with a regular GET request. This is not a JSON-RPC call — it's just a standard HTTP GET that returns JSON. From the card, Agent A learns:
- The JSON-RPC endpoint URL (from `supportedInterfaces`)
- What Agent B can do (from `skills`)
- How to authenticate (from `securitySchemes`)

**Step 2 — The actual call (JSON-RPC over HTTP POST).** Agent A constructs a JSON-RPC request with `method: "SendMessage"` and the message content in `params`. It sends this as an HTTP POST to the URL it learned from the Agent Card. The `Content-Type` is `application/json`. Auth credentials go in the `Authorization` header.

Agent B receives the POST, parses the JSON-RPC envelope, sees the method is `SendMessage`, extracts the message from `params`, processes it (using whatever internal logic it has), and sends back a JSON-RPC response with the result.

### Who Needs What

This is the key question — who needs to have the JSON-RPC endpoint, and who needs to be able to send JSON-RPC requests:

- **Agent B (the server / the one receiving work)** needs:
  - An HTTP server running
  - A `/.well-known/agent-card.json` endpoint (plain GET)
  - A JSON-RPC endpoint (the URL declared in the Agent Card) that accepts POST requests and routes them to the right method handlers (`SendMessage`, `GetTask`, etc.)

- **Agent A (the client / the one sending work)** needs:
  - The ability to make HTTP GET requests (to fetch the Agent Card)
  - The ability to make HTTP POST requests with JSON-RPC payloads (to call methods)
  - Knowledge of Agent B's domain (so it knows where to look)

Agent A does **not** need to run a server. Agent B does **not** need to make outbound requests (unless it's doing push notifications via webhooks). The relationship is client → server, just like a browser talking to a web API.

### There Is No "Handshake"

Unlike protocols like WebSocket or TLS, there is no formal handshake in A2A. There's no connection negotiation, no capability exchange before the first message. The flow is:

1. Client reads the Agent Card (this is the closest thing to a "handshake" — the client learns what the server supports)
2. Client sends a JSON-RPC request
3. Server responds

That's it. Every request is independent. There's no persistent connection (unless you're using SSE streaming, which is a one-way server-to-client stream after the initial POST). The client can send one request and walk away, or send a hundred. Each one is a standalone HTTP POST with a JSON-RPC payload.

### Client Side — Application Code vs Library Internals

```python
# Application code (using a2a-sdk client):
from a2a.client import A2AClient

client = A2AClient(agent_url="https://my-agent.example.com")
response = await client.send_message("Check my CloudWatch logs for errors")
print(response.artifacts[0].parts[0].text)

# What the library does behind the scenes:
# 1. POST https://my-agent.example.com/
#    Body: {"jsonrpc":"2.0","method":"message/send","id":"...","params":{...}}
# 2. Parses the JSON-RPC response into typed Python objects
# 3. Returns a Task object with status, artifacts, history
```

With Strands SDK, the client is further abstracted — a remote agent is declared and Strands handles the A2A communication:

```python
# Application code (using Strands A2AClient):
from strands.multiagent.a2a import A2AClient

client = A2AClient(
    agent_card_url="https://my-agent.example.com/.well-known/agent-card.json",
    bearer_token="eyJ..."
)
response = await client.send_message("Check my CloudWatch logs for errors")

# Strands automatically:
# 1. Fetches the agent card (GET /.well-known/agent-card.json)
# 2. Sends JSON-RPC message/send to the agent's URL
# 3. Parses the response and returns structured results
```

### Server Side — Application Code vs Framework Internals

```python
# Application code (using a2a-sdk server):
from a2a.server.agent_execution import AgentExecutor, RequestContext
from a2a.server.events import EventQueue

class MyAgentExecutor(AgentExecutor):
    async def execute(self, context: RequestContext, event_queue: EventQueue):
        user_message = context.get_user_input()  # "Check my CloudWatch logs"
        
        # Agent logic — call an LLM, use tools, run any custom code
        result = await my_llm.generate(user_message)
        
        # Return the result as a task artifact
        updater = TaskUpdater(event_queue, task.id, task.context_id)
        await updater.add_artifact([TextPart(text=result)])
        await updater.complete()

# Wiring it up:
from a2a.server.apps import A2AStarletteApplication
from a2a.types import AgentCard

agent_card = AgentCard(name="My Agent", url="...", skills=[...])
server = A2AStarletteApplication(agent_card=agent_card, http_handler=handler)
app = server.build()  # Returns a Starlette ASGI app

# What the framework does behind the scenes:
# 1. Listens for POST / with JSON-RPC body
# 2. Parses {"method":"message/send","params":{...}}
# 3. Calls the AgentExecutor.execute() method with the parsed message
# 4. Wraps the response into {"jsonrpc":"2.0","result":{"status":"completed",...}}
# 5. Serves GET /.well-known/agent-card.json automatically
```

### An Agent Can Be Both Client and Server

In a multi-agent system, the orchestrator agent is both:

```
User
  |
  |  (A2A client call)
  v
+---------------------------+
|  Orchestrator Agent       |
|                           |
|  Runs an HTTP server      |  <-- Server role (receives from user)
|  with JSON-RPC endpoint   |
|                           |
|  Also makes HTTP POSTs    |  <-- Client role (sends to specialists)
|  to other agents          |
+----+--------------+-------+
     |               |
     | POST          | POST
     | (JSON-RPC)    | (JSON-RPC)
     v               v
+-----------+  +-----------+
|  Agent X  |  |  Agent Y  |   <-- Server role only
|  (server) |  |  (server) |
+-----------+  +-----------+
```

The orchestrator has both an HTTP server (to receive requests) and an HTTP client (to send requests to other agents). Each downstream agent only needs a server. The JSON-RPC format is the same in every direction — the only difference is who is sending and who is receiving.

---

## Quick Reference: A2A JSON-RPC Methods

| Method | What It Does | Client Sends | Server Returns |
|:-------|:-------------|:-------------|:---------------|
| `SendMessage` | Start or continue a task | Message with parts | Task or Message |
| `SendStreamingMessage` | Same, but with SSE streaming | Message with parts | SSE stream of events |
| `GetTask` | Check task status | Task ID | Task with current state |
| `ListTasks` | Query multiple tasks | Filters, pagination | Array of tasks |
| `CancelTask` | Cancel a running task | Task ID | Updated task |
| `SubscribeToTask` | Stream updates for existing task | Task ID | SSE stream of events |
| `GetExtendedAgentCard` | Get richer card after auth | (nothing) | Extended AgentCard |

Each of these is a different value in the `method` field of the JSON-RPC request, with different `params` and different `result` shapes.

---

## Task Lifecycle

Tasks are the fundamental unit of work in A2A. They are stateful, server-managed, and progress through a well-defined lifecycle. This section covers the state machine, multi-turn interactions, and the three mechanisms for receiving task updates.

---

### Task State Machine

```
                    +---------------+
                    |  SUBMITTED    |
                    +-------+-------+
                            |
                            v
                    +---------------+
              +---->|   WORKING     |<----+
              |     +--+----+----+--+     |
              |        |    |    |        |
              |        |    |    |        |
     (resume) |        |    |    |        | (resume)
              |        |    |    |        |
    +---------+--+     |    |    |   +----+----------+
    |  INPUT     |<----+    |    +-->|  AUTH          |
    |  REQUIRED  |          |        |  REQUIRED      |
    +------------+          |        +----------------+
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
       +-----------+  +----------+  +----------+
       | COMPLETED |  |  FAILED  |  | CANCELED |
       +-----------+  +----------+  +----------+

                    +----------+
                    | REJECTED |  (can occur from any non-terminal state)
                    +----------+
```

### State Definitions

| State | Type | Description |
|:------|:-----|:------------|
| `TASK_STATE_SUBMITTED` | Initial | Task acknowledged and queued |
| `TASK_STATE_WORKING` | Active | Agent is actively processing |
| `TASK_STATE_INPUT_REQUIRED` | Interrupted | Agent needs more info from the client |
| `TASK_STATE_AUTH_REQUIRED` | Interrupted | Agent needs authorization to proceed |
| `TASK_STATE_COMPLETED` | Terminal | Task finished successfully |
| `TASK_STATE_FAILED` | Terminal | Task finished with an error |
| `TASK_STATE_CANCELED` | Terminal | Task was canceled before completion |
| `TASK_STATE_REJECTED` | Terminal | Agent decided not to perform the task |

Once a task reaches a **terminal state**, it cannot accept further messages. Sending a message to a terminal task returns `UnsupportedOperationError`.

### TaskStatus Object

Each task carries a `status` object:

```json
{
  "state": "TASK_STATE_WORKING",
  "message": {
    "role": "ROLE_AGENT",
    "parts": [{ "text": "Analyzing your document..." }]
  },
  "timestamp": "2025-10-28T10:30:00.000Z"
}
```

The optional `message` field lets the agent communicate context about the current state (progress info, error details, clarification requests).

---

## Multi-Turn Interactions

A2A supports multi-turn conversations through two identifiers:

### Context ID (`contextId`)

Groups related tasks and messages into a conversational session.

- Server may generate a `contextId` if the client does not provide one
- All tasks and messages with the same `contextId` belong to the same session
- Agents can use it to maintain LLM conversation history across interactions

### Task ID (`taskId`)

References a specific task for follow-up messages.

- Always server-generated
- Client includes `taskId` in subsequent messages to continue that task
- Server returns `TaskNotFoundError` if the ID does not exist

### Conversation Patterns

**Pattern 1: Input Required, then Client Responds**

```
Client --> Server:  "Book me a flight"
Server --> Client:  Task { state: INPUT_REQUIRED, message: "Where from and to?" }
Client --> Server:  Message { taskId: "task-1", text: "SFO to JFK" }
Server --> Client:  Task { state: COMPLETED, artifacts: [...] }
```

**Pattern 2: New Task in Same Context**

```
Client --> Server:  Message { text: "What's the weather in NYC?" }
Server --> Client:  Task { id: "task-1", contextId: "ctx-A", state: COMPLETED }

Client --> Server:  Message { contextId: "ctx-A", text: "And in London?" }
Server --> Client:  Task { id: "task-2", contextId: "ctx-A", state: COMPLETED }
```

The agent can use the shared `contextId` to understand "And in London?" refers to weather.

**Pattern 3: Auth Required, then Out-of-Band Resolution**

```
Client --> Server:  "Deploy the staging environment"
Server --> Client:  Task { state: AUTH_REQUIRED, message: "Need OAuth token for AWS" }
  ... client obtains credentials out-of-band ...
  ... agent receives credentials and resumes ...
Server --> Client:  Task { state: COMPLETED }
```

### Rules for Multi-Turn

- If both `taskId` and `contextId` are provided, they must match (the context of the referenced task)
- If only `taskId` is provided, the server infers `contextId` from the task
- Agents must reject messages with mismatching `taskId` and `contextId`
- Client-provided `taskId` for creating new tasks is not supported — task IDs are always server-generated

---

## Task Update Delivery

A2A provides three complementary mechanisms for receiving task updates:

### 1. Polling (Get Task)

The simplest approach. Client periodically calls `GetTask` to check status.

```
GET /tasks/task-abc?historyLength=5 HTTP/1.1
Host: agent.example.com
Authorization: Bearer token
```

**Pros:** Simple, works everywhere, no persistent connections needed.
**Cons:** Higher latency, potential for unnecessary requests.
**Best for:** Simple integrations, infrequent updates, restrictive firewalls.

### 2. Streaming (SSE / gRPC)

Real-time delivery of events as they occur. Two entry points:

- `SendStreamingMessage` — Send a message and stream the response
- `SubscribeToTask` — Open a stream for an existing task

```
POST /message:stream HTTP/1.1
Content-Type: application/a2a+json

{ "message": { "role": "ROLE_USER", "parts": [{"text": "Write a report"}] } }
```

Response (SSE):

```
data: {"task":{"id":"task-1","status":{"state":"TASK_STATE_WORKING"}}}

data: {"artifactUpdate":{"taskId":"task-1","artifact":{"parts":[{"text":"# Report\n\n..."}]},"append":true}}

data: {"artifactUpdate":{"taskId":"task-1","artifact":{"parts":[{"text":"## Section 2\n\n..."}]},"append":true,"lastChunk":true}}

data: {"statusUpdate":{"taskId":"task-1","status":{"state":"TASK_STATE_COMPLETED"}}}
```

**Stream patterns:**

- **Message-only stream:** Single `Message` object, then close. No task tracking.
- **Task lifecycle stream:** `Task` object first, then zero or more `TaskStatusUpdateEvent` / `TaskArtifactUpdateEvent`, close on terminal state.

**Pros:** Low latency, efficient for frequent updates.
**Cons:** Requires persistent connection.
**Best for:** Interactive apps, real-time dashboards, live progress monitoring.
**Requires:** `AgentCard.capabilities.streaming: true`

### 3. Push Notifications (Webhooks)

Agent sends HTTP POST requests to a client-registered endpoint when task state changes.

**Setup:** Include push config when sending a message:

```json
{
  "message": { "role": "ROLE_USER", "parts": [{"text": "Generate Q1 report"}] },
  "configuration": {
    "taskPushNotificationConfig": {
      "url": "https://client.example.com/webhook/a2a",
      "token": "my-secret-token",
      "authentication": { "scheme": "Bearer" }
    }
  }
}
```

**Webhook payload:** Uses the same `StreamResponse` format as streaming:

```json
{
  "statusUpdate": {
    "taskId": "task-abc",
    "contextId": "ctx-123",
    "status": {
      "state": "TASK_STATE_COMPLETED",
      "timestamp": "2025-03-15T18:30:00Z"
    }
  }
}
```

**Pros:** No persistent connection needed, great for long-running tasks.
**Cons:** Client must be reachable via HTTP, more complex setup.
**Best for:** Server-to-server integrations, long-running tasks, event-driven architectures.
**Requires:** `AgentCard.capabilities.pushNotifications: true`

---

## Streaming Event Types

### TaskStatusUpdateEvent

Notifies the client of a state change:

```json
{
  "taskId": "task-abc",
  "contextId": "ctx-123",
  "status": {
    "state": "TASK_STATE_WORKING",
    "message": {
      "role": "ROLE_AGENT",
      "parts": [{ "text": "Processing step 2 of 5..." }]
    }
  }
}
```

### TaskArtifactUpdateEvent

Delivers artifact content (potentially in chunks):

```json
{
  "taskId": "task-abc",
  "contextId": "ctx-123",
  "artifact": {
    "artifactId": "art-001",
    "parts": [{ "text": "...chunk of content..." }]
  },
  "append": true,
  "lastChunk": false
}
```

- `append: true` — content should be appended to a previously sent artifact with the same ID
- `lastChunk: true` — this is the final chunk of the artifact

---

## Execution Mode

The `returnImmediately` field in `SendMessageConfiguration` controls blocking behavior:

| Value | Behavior |
|:------|:---------|
| `false` (default) | **Blocking** — waits until task reaches terminal or interrupted state |
| `true` | **Non-blocking** — returns immediately after creating the task |

Non-blocking mode requires the client to poll, stream, or use push notifications to get updates.

---

## History Length

The `historyLength` parameter controls how much conversation history is returned:

| Value | Behavior |
|:------|:---------|
| Unset | Server returns its default amount (implementation-defined) |
| `0` | No history returned |
| `> 0` | At most this many recent messages |

---

*Re: [A2A Specification](https://a2a-protocol.org/latest/specification/), [A2A Key Concepts](https://a2a-protocol.org/latest/topics/key-concepts/).*
