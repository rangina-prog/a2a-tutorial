# Protocol & Transport

A2A separates its abstract operations from the wire protocols used to carry them. The spec defines three official **protocol bindings** — JSON-RPC, gRPC, and HTTP+JSON/REST — plus a framework for custom bindings. An agent can support one or more of these simultaneously.

---

## Core Operations

Every A2A implementation must support these operations, regardless of binding:

| Operation | Purpose |
|-----------|---------|
| **Send Message** | Primary way to initiate or continue agent interactions |
| **Send Streaming Message** | Same as above, but with real-time streaming of updates |
| **Get Task** | Retrieve current state of a previously initiated task |
| **List Tasks** | Query tasks with filtering and pagination |
| **Cancel Task** | Request cancellation of an ongoing task |
| **Subscribe to Task** | Open a stream to receive updates for an existing task |
| **Push Notification CRUD** | Create / Get / List / Delete webhook configurations |
| **Get Extended Agent Card** | Retrieve a richer Agent Card after authentication |

---

## Binding 1: JSON-RPC 2.0

The most common binding. Uses standard HTTP(S) with JSON-RPC 2.0 message framing.

### Basics

- **Transport:** HTTP(S)
- **Content-Type:** `application/json`
- **Method naming:** PascalCase (e.g., `SendMessage`, `GetTask`)
- **Streaming:** Server-Sent Events (`text/event-stream`)

### Request / Response Shape

Every JSON-RPC request follows this structure:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "SendMessage",
  "params": {
    "message": {
      "role": "ROLE_USER",
      "parts": [{ "text": "What is the weather today?" }],
      "messageId": "msg-001"
    }
  }
}
```

A successful response wraps the result:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "task": {
      "id": "task-abc",
      "contextId": "ctx-123",
      "status": { "state": "TASK_STATE_COMPLETED" },
      "artifacts": [{
        "artifactId": "art-001",
        "parts": [{ "text": "Sunny, high of 75°F." }]
      }]
    }
  }
}
```

### Streaming (SSE)

For `SendStreamingMessage` and `SubscribeToTask`, the server responds with `Content-Type: text/event-stream`. Each SSE `data:` line contains a JSON-RPC result wrapping a `StreamResponse`:

```
data: {"jsonrpc":"2.0","id":1,"result":{"task":{"id":"task-abc","status":{"state":"TASK_STATE_WORKING"}}}}

data: {"jsonrpc":"2.0","id":1,"result":{"artifactUpdate":{"taskId":"task-abc","artifact":{"parts":[{"text":"# Report\n\n..."}]}}}}

data: {"jsonrpc":"2.0","id":1,"result":{"statusUpdate":{"taskId":"task-abc","status":{"state":"TASK_STATE_COMPLETED"}}}}
```

### Method Mapping

| Operation | JSON-RPC Method |
|-----------|----------------|
| Send Message | `SendMessage` |
| Stream Message | `SendStreamingMessage` |
| Get Task | `GetTask` |
| List Tasks | `ListTasks` |
| Cancel Task | `CancelTask` |
| Subscribe to Task | `SubscribeToTask` |
| Create Push Config | `CreateTaskPushNotificationConfig` |
| Get Push Config | `GetTaskPushNotificationConfig` |
| List Push Configs | `ListTaskPushNotificationConfigs` |
| Delete Push Config | `DeleteTaskPushNotificationConfig` |
| Get Extended Card | `GetExtendedAgentCard` |

### Error Codes

JSON-RPC uses numeric error codes. A2A defines custom codes in the `-32001` to `-32099` range:

| Code | A2A Error |
|------|-----------|
| `-32001` | TaskNotFoundError |
| `-32002` | TaskNotCancelableError |
| `-32003` | PushNotificationNotSupportedError |
| `-32004` | UnsupportedOperationError |
| `-32005` | ContentTypeNotSupportedError |
| `-32006` | InvalidAgentResponseError |
| `-32007` | ExtendedAgentCardNotConfiguredError |
| `-32008` | ExtensionSupportRequiredError |
| `-32009` | VersionNotSupportedError |

Standard JSON-RPC errors (`-32700` parse error, `-32600` invalid request, etc.) also apply.

---

## Binding 2: gRPC

High-performance, strongly-typed interface using Protocol Buffers over HTTP/2.

### Basics

- **Transport:** gRPC over HTTP/2 with TLS
- **Definition:** `spec/a2a.proto` (the normative source)
- **Serialization:** Protocol Buffers v3
- **Service:** `A2AService`

### Service Definition (summary)

```protobuf
service A2AService {
  rpc SendMessage(SendMessageRequest) returns (SendMessageResponse);
  rpc SendStreamingMessage(SendMessageRequest) returns (stream StreamResponse);
  rpc GetTask(GetTaskRequest) returns (Task);
  rpc ListTasks(ListTasksRequest) returns (ListTasksResponse);
  rpc CancelTask(CancelTaskRequest) returns (Task);
  rpc SubscribeToTask(SubscribeToTaskRequest) returns (stream StreamResponse);
  rpc CreateTaskPushNotificationConfig(...) returns (TaskPushNotificationConfig);
  rpc GetTaskPushNotificationConfig(...) returns (TaskPushNotificationConfig);
  rpc ListTaskPushNotificationConfigs(...) returns (...);
  rpc DeleteTaskPushNotificationConfig(...) returns (google.protobuf.Empty);
  rpc GetExtendedAgentCard(...) returns (AgentCard);
}
```

### Service Parameters

A2A service parameters (like `A2A-Version`) are transmitted via **gRPC metadata** (headers):

```go
md := metadata.Pairs(
    "authorization", "Bearer token",
    "a2a-version", "1.0",
)
ctx := metadata.NewOutgoingContext(context.Background(), md)
response, err := client.SendMessage(ctx, request)
```

### Error Handling

gRPC maps A2A errors to standard gRPC status codes:

| A2A Error | gRPC Status |
|-----------|-------------|
| TaskNotFoundError | `NOT_FOUND` |
| TaskNotCancelableError | `FAILED_PRECONDITION` |
| UnsupportedOperationError | `FAILED_PRECONDITION` |
| ContentTypeNotSupportedError | `INVALID_ARGUMENT` |
| InvalidAgentResponseError | `INTERNAL` |

Error details use `google.rpc.ErrorInfo` with `domain: "a2a-protocol.org"`.

---

## Binding 3: HTTP+JSON/REST

A RESTful interface using standard HTTP methods and JSON payloads.

### Basics

- **Transport:** HTTP(S)
- **Content-Type:** `application/a2a+json` (recommended)
- **Methods:** Standard HTTP verbs (GET, POST, DELETE)
- **Streaming:** Server-Sent Events

### Endpoint Mapping

| Operation | Method | Endpoint |
|-----------|--------|----------|
| Send Message | POST | `/message:send` |
| Stream Message | POST | `/message:stream` |
| Get Task | GET | `/tasks/{id}` |
| List Tasks | GET | `/tasks` |
| Cancel Task | POST | `/tasks/{id}:cancel` |
| Subscribe to Task | POST | `/tasks/{id}:subscribe` |
| Create Push Config | POST | `/tasks/{id}/pushNotificationConfigs` |
| Get Push Config | GET | `/tasks/{id}/pushNotificationConfigs/{configId}` |
| List Push Configs | GET | `/tasks/{id}/pushNotificationConfigs` |
| Delete Push Config | DELETE | `/tasks/{id}/pushNotificationConfigs/{configId}` |
| Get Extended Card | GET | `/extendedAgentCard` |

### Example: Send Message

```http
POST /message:send HTTP/1.1
Host: agent.example.com
Content-Type: application/a2a+json
Authorization: Bearer token
A2A-Version: 1.0

{
  "message": {
    "messageId": "msg-001",
    "role": "ROLE_USER",
    "parts": [{ "text": "Book me a flight to NYC" }]
  }
}
```

Response:

```http
HTTP/1.1 200 OK
Content-Type: application/a2a+json

{
  "task": {
    "id": "task-xyz",
    "contextId": "ctx-456",
    "status": {
      "state": "TASK_STATE_INPUT_REQUIRED",
      "message": {
        "role": "ROLE_AGENT",
        "parts": [{ "text": "Where are you flying from?" }]
      }
    }
  }
}
```

### Error Handling

HTTP errors use `google.rpc.Status` JSON representation:

```json
{
  "error": {
    "code": 404,
    "status": "NOT_FOUND",
    "message": "Task not found",
    "details": [{
      "@type": "type.googleapis.com/google.rpc.ErrorInfo",
      "reason": "TASK_NOT_FOUND",
      "domain": "a2a-protocol.org"
    }]
  }
}
```

---

## Cross-Binding Comparison

| Aspect | JSON-RPC | gRPC | HTTP+JSON/REST |
|--------|----------|------|----------------|
| **Transport** | HTTP(S) | HTTP/2 + TLS | HTTP(S) |
| **Serialization** | JSON | Protobuf | JSON |
| **Streaming** | SSE | Server streaming RPC | SSE |
| **Content-Type** | `application/json` | `application/grpc` | `application/a2a+json` |
| **Typing** | Dynamic | Static (proto) | Dynamic |
| **Best for** | Web clients, simplicity | High-throughput, polyglot | REST-native ecosystems |

All three bindings must provide **identical functionality**, **consistent behavior**, **same error handling**, and **equivalent authentication**.

---

## Versioning

Clients send the protocol version via the `A2A-Version` service parameter (header or query param):

```
A2A-Version: 1.0
```

- Servers must process requests using the semantics of the requested version.
- If the version is unsupported, the server returns `VersionNotSupportedError`.
- An empty version header is interpreted as `0.3` (for backward compatibility).
- Only `Major.Minor` is used — patch versions don't affect protocol compatibility.

---

## JSON Field Naming Convention

All JSON serializations use **camelCase** field names, not the snake_case from Protocol Buffer definitions:

| Proto field | JSON field |
|-------------|-----------|
| `protocol_version` | `protocolVersion` |
| `context_id` | `contextId` |
| `default_input_modes` | `defaultInputModes` |

Enum values use their proto string names in **SCREAMING_SNAKE_CASE**:
- `TASK_STATE_COMPLETED`
- `ROLE_USER`

---

*Sources: [A2A Specification §5, §9, §10, §11](https://a2a-protocol.org/latest/specification/). Content was rephrased for compliance with licensing restrictions.*
