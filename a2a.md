# A2A

## The Problem

AI agents are are built with different frameworks (Strands, Langchain, LangGraph, CrewAI, Google ADK, custom solutions), by different vendors, and deployed on separate infrastructure. Without a **shared communication standard**, each pair of agents needs a custom translation code to integrate - an approach that does not scale.

The Agent2Agent (A2A) protocol addresses this challenge: it gives agents a **common language** so any agent can talk to any other agent without custom code in between — all without exposing their internal state, memory, or tools.

Amazon Web Services **Bedrock AgentCore** natively supports A2A

## Origin and Governance

A2A has been donated to the Linux Foundation as an open-source project. It is licensed under **Apache 2.0** and is open to community contributions. The current released specification is **v1.0.0**.

### A2A Spec

> The A2A spec is an **open specification document** — a written agreement on how agents should communicate ([a2a-protocol.org](https://a2a-protocol.org)).

It defines the data model, the methods, the transport, the agent card format, and the expected behaviors. 

The A2A SDK is the reference implementation that other frameworks build on or use directly. [A2A Python SDK](https://github.com/a2aproject/a2a-python)


## Key Features at a Glance

- **Standardized Communication** — JSON-RPC 2.0 over HTTP(S), gRPC over HTTP/2, and HTTP+JSON/REST.
- **Agent Discovery** — Via "Agent Cards" that detail capabilities, skills, and connection info.
- **Flexible Interaction** — Synchronous request/response, streaming (SSE / gRPC server streaming), and asynchronous push notifications (webhooks).
- **Enterprise-Ready Security** — OAuth 2.0, OpenID Connect, API keys, mTLS, and JWS-signed Agent Cards.


## Frameworks

Currently **Strands and ADK** have the most mature built-in A2A support natively.
The major frameworks have been adding A2A support natively, so developers do not have to wire up the a2a-sdk separately.

No A2A built-in currently: LangChain, LangGraph, CrewAI, OpenAI SDK

More about:
[Strands A2A](https://strandsagents.com/docs/user-guide/concepts/multi-agent/agent-to-agent/)


## The Three-Layer Architecture

A2A's specification is organized into three layers that build on each other:

```
┌─────────────────────────────────────────────────┐
│  Layer 3: Protocol Bindings                     │
│  JSON-RPC  ·  gRPC  ·  HTTP+JSON/REST  · Custom│
├─────────────────────────────────────────────────┤
│  Layer 2: Abstract Operations                   │
│  SendMessage · StreamMessage · GetTask          │
│  ListTasks · CancelTask · SubscribeToTask       │
│  Push Notification CRUD · GetExtendedAgentCard  │
├─────────────────────────────────────────────────┤
│  Layer 1: Canonical Data Model                  │
│  Task · Message · Part · Artifact · AgentCard   │
│  Extension · Streaming Events                   │
└─────────────────────────────────────────────────┘
```

- **Layer 1 (Data Model):** Protocol-agnostic definitions expressed as Protocol Buffer messages. This is the normative source of truth (`spec/a2a.proto`).
- **Layer 2 (Operations):** The fundamental capabilities agents must support, independent of how they are exposed over a wire protocol.
- **Layer 3 (Bindings):** Concrete mappings of operations to JSON-RPC methods, gRPC RPCs, and REST endpoints.

This layered approach means core semantics stay consistent across all bindings, and new bindings can be added without changing the data model.


*Re: [A2A GitHub README](https://github.com/a2aproject/A2A), [A2A Protocol Specification](https://a2a-protocol.org/latest/specification/), [A2A Protocol Home](https://a2a-protocol.org/latest/).*
