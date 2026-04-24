# Framework A2A Adoption

How major agent frameworks have adopted (or are adopting) the A2A protocol, what they provide out of the box, and how they handle the Agent Card.

---

## Summary Table

| Framework | Vendor | A2A Support | Agent Card | Binding Used | Status |
|-----------|--------|-------------|------------|--------------|--------|
| **Strands Agents** | AWS | Native since v1.0 | Auto-generated from tools | JSON-RPC | Production |
| **Google ADK** | Google | Native, built-in | Auto-generated via helper | JSON-RPC | Production |
| **CrewAI (AMP)** | CrewAI | Native on AMP platform | Auto-generated from crew/agent metadata | JSON-RPC + gRPC + REST | Early release |
| **Microsoft Agent Framework** | Microsoft | Native via NuGet/pip packages | Configured via `MapA2A()` or equivalent | JSON-RPC (SSE streaming) | Preview |
| **LangGraph / LangChain** | LangChain Inc. | Community + samples, no native built-in | Manual setup required | JSON-RPC | Community/samples |
| **AutoGen** | Microsoft | Via Microsoft Agent Framework migration | Via Agent Framework integration | JSON-RPC | Indirect |

---

## Agent Card: How Each Framework Handles It

The Agent Card is the entry point for all A2A interactions. Here's how each framework approaches it:

| Framework | Card Generation | Card Serving | Customization |
|-----------|----------------|--------------|---------------|
| **Google ADK** | `create_agent_card()` helper auto-generates from skills | Served by `a2a-sdk` server | Full control via `AgentSkill` objects |
| **Strands** | Auto-generated from agent name + tools | `A2AServer` serves at `/.well-known/agent.json` | Tools automatically become skills |
| **CrewAI AMP** | Auto-generated from agent roles, goals, tools | Platform serves at `/.well-known/agent-card.json` | Auth auto-populates security fields; supports extended cards |
| **Microsoft Agent Framework** | Manual config via `MapA2A()` | Framework serves at configured path | Set name, description, version; URL auto-assigned |
| **LangGraph** | Manual — you write the JSON | You serve it yourself (e.g., FastAPI endpoint) | Full manual control |
| **AutoGen** | N/A (use Agent Framework) | N/A | N/A |

> Frameworks that have native A2A support auto-generate the Agent Card from your agent's existing metadata (tools, skills, roles). 
Frameworks without native support require you to write and serve the card manually.

---

## Summary: Agent Card Commonalities Across Frameworks

Despite different APIs and developer experiences, every framework that implements A2A converges on the same Agent Card structure because they're all targeting the same spec. Here's what's common and what varies.

### Fields Every Framework Produces

These fields appear in every Agent Card regardless of which framework generates it. They are required by the A2A spec, so no framework can skip them:

| Field | What It Contains | Where Frameworks Get It |
|-------|-----------------|------------------------|
| `name` | Human-readable agent name | Agent name / role (ADK, Strands, CrewAI) or manually set (Microsoft, LangGraph) |
| `description` | What the agent does | Agent description / goal / backstory, or manually set |
| `version` | Semver string like `"1.0.0"` | Usually hardcoded or derived from deployment config |
| `supportedInterfaces` | Array of `{ url, protocolBinding, protocolVersion }` | Framework auto-fills based on the server it starts (URL + port + binding type) |
| `capabilities` | `{ streaming, pushNotifications, ... }` | Framework sets based on what it supports (e.g., CrewAI sets streaming + push; Strands sets based on config) |
| `defaultInputModes` | MIME types accepted (e.g., `["text/plain"]`) | Usually defaults to text; some frameworks let you override |
| `defaultOutputModes` | MIME types produced | Same as above |
| `skills` | Array of skill objects | This is where the biggest variation happens — see below |

### How Skills Get Populated

With skills each framework derives them differently from its own abstractions:

- **Google ADK:** You explicitly define `AgentSkill` objects with `id`, `name`, `description`, `tags`, and `examples`. These map 1:1 to the Agent Card's `skills` array. You have full control over every field.

- **Strands:** The framework auto-generates skills from the **tools** you give the agent. Each tool becomes a skill. The tool's name becomes the skill name, and the tool's docstring or description becomes the skill description. You don't write skill definitions separately.

- **CrewAI AMP:** Skills are derived from **agent roles and goals**. At the crew level, each agent with `A2AServerConfig` becomes a skill on the crew's aggregate card. The agent's `role` becomes the skill name, `goal` becomes the description. Tools attached to the agent inform the skill's tags.

- **Microsoft Agent Framework:** Skills are not auto-generated. You set `Name`, `Description`, and `Version` on the card object. If you need skills, you'd configure them manually. The framework focuses on getting the protocol layer right rather than card richness.

- **LangGraph:** Fully manual. You write the entire `skills` array in your Agent Card JSON. Nothing is derived from the graph structure.

### Fields That Only Some Frameworks Auto-Populate

| Field | Who Auto-Populates | How |
|-------|--------------------|-----|
| `securitySchemes` / `security` | **CrewAI AMP** | Derived from the auth scheme you configure (`EnterpriseTokenAuth`, `OIDCAuth`, etc.) |
| `securitySchemes` / `security` | **Google ADK** (on Agent Engine) | Platform injects based on deployment auth config |
| `provider` | None auto-populate | Always manual if you want it |
| `iconUrl` | None auto-populate | Always manual |
| `documentationUrl` | None auto-populate | Always manual |
| `signatures` (JWS) | None auto-populate | No framework currently auto-signs cards |
| `extensions` | **CrewAI AMP** | Populates if extensions are configured on the deployment |

### What the User Typically Controls

Across all frameworks, these are the things you as the developer decide:

1. **Agent identity** — name, description, version. Every framework asks you for these in some form, whether it's a constructor argument, a config object, or a raw JSON field.

2. **Skills / capabilities description** — what your agent can do. Frameworks with auto-generation (ADK, Strands, CrewAI) derive this from your tools, roles, or skill definitions. Manual frameworks (LangGraph, Microsoft) leave it to you.

3. **Auth requirements** — how clients authenticate. CrewAI auto-populates this from your auth config. Others require manual setup or rely on the hosting platform.

4. **Input/output modes** — what MIME types your agent accepts and produces. Most frameworks default to `text/plain` and let you override per-skill if needed.

5. **Capabilities flags** — whether you support streaming, push notifications, extended cards. Frameworks set these based on what they actually implement. You generally don't override these.

### What No Framework Provides (Currently)

A few Agent Card features from the spec aren't auto-populated by any framework today:

- **Multiple protocol bindings** — CrewAI AMP is the only framework that auto-generates cards with multiple `supportedInterfaces` entries (JSON-RPC + gRPC + REST). Others typically expose a single binding.
- **Card signing (JWS)** — no framework auto-signs Agent Cards. If you need signed cards for integrity verification, you'd implement this yourself.
- **Extended Agent Cards** — only CrewAI AMP supports this natively (role-based skill visibility for authenticated vs. unauthenticated clients). Others would require custom implementation.
- **Skill-level security requirements** — the spec allows per-skill auth, but no framework auto-generates this from its own abstractions.

### The Common Pattern

Regardless of framework, the Agent Card workflow follows the same three steps:

1. **Define** — the framework (or developer) creates the Agent Card JSON from agent metadata
2. **Serve** — the framework (or developer) exposes it at a well-known URL (usually `/.well-known/agent-card.json`)
3. **Consume** — clients fetch the card, read skills and interfaces, and start sending messages

---
