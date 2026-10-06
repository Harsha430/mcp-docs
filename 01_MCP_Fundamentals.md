# 01: MCP Fundamentals

**Version/Source Basis:** MCP protocol and documentation as of **July 28, 2026**. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

## What is MCP?

### Real-World Analogy

Imagine you're using a restaurant delivery app (host). To add a new restaurant, the app doesn't rewrite its entire codebase. Instead, each restaurant (server) follows a standard contract: "Tell me your menu items and prices" (protocol). The app discovers what each restaurant offers and shows you those options. You order (invoke tool), the restaurant prepares (executes function), and returns your food (result).

**MCP is exactly that contract** – a standard protocol that lets applications discover and use functions and data from external services without knowing their internal implementation.

### Technical Definition

**Model Context Protocol (MCP)** is a standardized, message-based protocol for applications to discover and invoke capabilities (tools, resources, prompts) from external services. It decouples the application (host) from capability providers (servers) using a client-server architecture with JSON-RPC message exchange.

## Why MCP Exists

### Problem It Solves

1. **Coupling:** Without MCP, applications hardcode integrations to each service
2. **Scaling:** Adding a new service requires application changes
3. **Standards:** Every integration follows a different pattern
4. **Context:** AI models need dynamic access to changing tools/data

### Use Cases

- **IDE Plugins:** VS Code discovering MCP servers for code analysis, debugging
- **AI Agents:** Claude, LangChain agents discovering and using tools dynamically
- **Browser Extensions:** Discovering web capabilities
- **CLI Tools:** Invoking capabilities from multiple servers
- **LLM Applications:** AI models using up-to-date business data and operations

## MCP vs REST APIs

| Aspect | REST API | MCP |
|--------|----------|-----|
| **Discovery** | Manual docs | Automatic discovery |
| **Invocation** | HTTP request | Protocol message |
| **Authentication** | Headers/keys | Built-in auth support |
| **Statelessness** | Each request standalone | True statelessness by design |
| **Capabilities** | Endpoints | Tools, resources, prompts |
| **Use Case** | Web services | AI agents, tools, IDE integrations |
| **Coupling** | Application knows each API | Application knows MCP protocol |

### Example: REST vs MCP

**REST:** Application hardcodes "call GET /api/customers/{id}"
```
Host knows: URL, HTTP method, authentication, response format
Changes in API require application code changes
```

**MCP:** Host asks server "what capabilities do you have?" and uses them dynamically
```
Host knows: MCP protocol
Changes in server capabilities are discovered automatically
```

## MCP vs Function Calling

### Function Calling (in LLMs)

When you give Claude a function definition, it decides to call it based on the context. The function definition is part of the prompt.

```json
{
  "name": "get_weather",
  "description": "Get current weather",
  "parameters": {
    "type": "object",
    "properties": {"location": {"type": "string"}}
  }
}
```

### MCP Approach

The server provides function definitions through protocol messages. The host discovers them and invokes them through MCP messages.

**Key Difference:**
- **Function Calling:** Definitions embedded in prompt/request
- **MCP:** Definitions discovered from server, used via protocol

## MCP Architecture Overview

```
┌─────────────────────────────────────────┐
│         Host (Application)              │
│  (Claude, VS Code, AI Agent, etc.)      │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │    MCP Client (Inside Host)       │  │
│  │ - Sends protocol messages         │  │
│  │ - Discovers capabilities          │  │
│  │ - Invokes tools/resources/prompts │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
           ▲
           │ MCP Protocol
           │ (JSON-RPC Messages)
           ▼
┌─────────────────────────────────────────┐
│        MCP Server (External)            │
│  (Database, API, Service, etc.)         │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │   Tool: get_customer              │  │
│  │   Resource: customer_data.json    │  │
│  │   Prompt: analysis_template       │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

## Core Components

### MCP Host
**What it is:** Your application (Claude, IDE, AI framework)

**Responsibilities:**
- Runs the MCP client
- Discovers servers
- Invokes tools/resources
- Handles responses

### MCP Client
**What it is:** Software inside the host that implements the MCP protocol

**Responsibilities:**
- Sends/receives MCP messages
- Manages server connections
- Handles JSON-RPC exchange
- Deserializes responses

### MCP Server
**What it is:** External service exposing capabilities

**Responsibilities:**
- Receives MCP messages
- Exposes tools, resources, prompts
- Executes requested operations
- Sends results back

## MCP Protocol: The July 28, 2026 Update

### What Changed

The **July 28, 2026** MCP protocol revision introduced:

1. **Explicit Statelessness:** Protocol no longer assumes persistent connections; each message is independent
2. **Simplified Message Format:** Cleaner JSON-RPC structure
3. **Enhanced Discovery:** Better capability metadata
4. **Improved Authorization:** Protocol-level support for authorization context
5. **Resource Streaming:** More efficient handling of large resources

### Key Principle: Statelessness

**Stateless MCP means:**
- The server doesn't remember you from one call to the next
- Each request includes all necessary context
- Transport layer can be stateless (HTTP) or stateful (WebSocket)
- Simplifies scaling and deployment

**Why it matters:**
- One server instance can handle many concurrent clients
- Easy to restart servers without losing state
- Clean separation between transport layer and protocol layer

## Message Flow

```
Host                         Server
 │
 ├──────────── Initialize ─────────────>
 │
 <──────────── Capabilities ──────────┤
 │
 ├────── Discover Tools/Resources ──>
 │
 <────── Tool/Resource List ──────────┤
 │
 ├────────── Call Tool ──────────────>
 │
 <────────── Result ──────────────────┤
 │
```

## Tools, Resources, and Prompts

### Tool
**What:** A callable function the server exposes

**Example:** `get_customer_info(customer_id)`

```python
Tool {
  name: "get_customer_info",
  description: "Retrieve customer details",
  input_schema: {
    customer_id: "string"
  }
}
```

### Resource
**What:** Data the server offers to read

**Example:** `/documents/annual_report.pdf`

```python
Resource {
  uri: "documents://annual_report.pdf",
  name: "Annual Report 2026",
  mime_type: "application/pdf"
}
```

### Prompt
**What:** Templates or guidance the server provides

**Example:** A template for analyzing customer behavior

```python
Prompt {
  name: "customer_analysis",
  description: "Template for analyzing a customer",
  arguments: [
    {"name": "customer_id", "description": "The customer ID"}
  ]
}
```

## Transports

MCP messages can travel over different transports:

| Transport | Usage | Stateless? |
|-----------|-------|-----------|
| **stdio** | Local process communication | Can be |
| **SSE** | Server-Sent Events (HTTP) | Yes |
| **HTTP** | Standard HTTP requests | Yes |
| **WebSocket** | Two-way connection | No |

Each transport is just a vehicle for messages. The MCP protocol is transport-agnostic.

## Capabilities

When a server connects, it declares capabilities:

```python
{
  "tools": {
    "list": [...],  # Available tools
    "call": "..."   # Tool calling mechanism
  },
  "resources": {
    "list": [...],  # Available resources
    "read": "..."   # Resource reading mechanism
  },
  "prompts": {
    "list": [...],  # Available prompts
    "get": "..."    # Prompt retrieval mechanism
  }
}
```

The host knows what it can do with the server based on capabilities.

## When to Use MCP

✅ **Use MCP When:**
- You need dynamic tool/capability discovery
- You're building an AI agent or AI-powered application
- You want decoupling between application and services
- You need to support multiple heterogeneous services
- You're building IDE plugins or extensions

❌ **Don't Use MCP When:**
- You need real-time bidirectional communication (consider WebSocket directly)
- Your use case is simple REST API calling (use HTTP client)
- You're integrating with a single, well-known API (direct integration is simpler)

## Interview Summary

**Key Takeaways:**
- MCP is a **protocol for discovering and invoking capabilities**
- It solves **coupling and scaling problems**
- Built on **JSON-RPC** over various transports
- **Stateless by design** (July 2026 revision)
- Applications (hosts) discover what servers offer without hardcoding

**One-Liner Definition:**
"MCP is an open protocol that lets applications dynamically discover and use tools, resources, and prompts from external services, decoupling the application from the services it uses."

## Common Interview Questions

**Q1: What is MCP and why does it exist?**
A: MCP is a protocol for discovering and invoking capabilities from external services. It exists to decouple applications from the services they use and to enable dynamic capability discovery—critical for AI agents that need to work with changing tools and data.

**Q2: What's the difference between MCP and REST APIs?**
A: REST APIs require the application to know the endpoint, method, and authentication upfront. MCP allows automatic discovery—the server tells the host what capabilities it has, and the host invokes them through a standard protocol.

**Q3: How is MCP stateless? Isn't stateless bad for complex operations?**
A: MCP is stateless at the protocol level, not the application level. Each message includes all context needed to process it. This makes servers scalable—one instance can handle many concurrent clients. Complex operations still work; the server executes them within a single request/response cycle.

**Q4: When should you use MCP instead of direct function calling?**
A: Use MCP when you need dynamic discovery, multiple heterogeneous services, or decoupling. Use direct function calling when integrating a single well-known service into a simpler application.

**Q5: What did the July 28, 2026 protocol revision change?**
A: It formalized statelessness, simplified the message format, enhanced capability discovery, improved authorization support, and added resource streaming for efficiency.

---

**Next:** [02_MCP_Architecture.md](02_MCP_Architecture.md) – Deep dive into Host, Client, Server components
