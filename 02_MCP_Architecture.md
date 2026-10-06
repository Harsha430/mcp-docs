# 02: MCP Architecture

**Version/Source Basis:** MCP protocol as of **July 28, 2026**. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

## The MCP Triad: Host, Client, Server

### Real-World Analogy

Think of a **smart assistant system:**
- **You (User)** speak to the assistant in natural language
- **Assistant (Host)** understands your intent
- **Dispatcher (Client)** inside the assistant knows how to contact services
- **Restaurant/Bank/Library (Server)** provides specific services

When you ask "reserve a table," the dispatcher contacts the restaurant server and relays your request.

### Technical Structure

```
┌──────────────────────────────────────────────────────┐
│  Host (Claude, IDE, AI Agent Framework)              │
│                                                       │
│  ┌────────────────────────────────────────────────┐  │
│  │  Application Logic                             │  │
│  │  (What the host does—reasoning, coding, etc.)  │  │
│  └────────────────────────────────────────────────┘  │
│                      ▲                                │
│                      │                                │
│  ┌────────────────────────────────────────────────┐  │
│  │  MCP Client Layer                              │  │
│  │  (Discovers & invokes capabilities)            │  │
│  │                                                 │  │
│  │  - Initialize servers                          │  │
│  │  - Discover tools/resources/prompts            │  │
│  │  - Invoke tools                                │  │
│  │  - Handle responses                            │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
              │                           │
              │ MCP Protocol              │ MCP Protocol
              │ (JSON-RPC Messages)       │ (JSON-RPC Messages)
              ▼                           ▼
   ┌──────────────────┐        ┌──────────────────┐
   │  MCP Server 1    │        │  MCP Server 2    │
   │  (e.g., DB)      │        │  (e.g., API)     │
   │                  │        │                  │
   │ - Tools          │        │ - Tools          │
   │ - Resources      │        │ - Resources      │
   │ - Prompts        │        │ - Prompts        │
   └──────────────────┘        └──────────────────┘
```

## Component 1: MCP Host

### What Is It?

The **host** is your primary application. It's the decision-maker, the orchestrator.

**Examples of hosts:**
- Claude (AI assistant)
- VS Code (code editor)
- LangChain agent
- CrewAI application
- Your custom Python application

### What Does It Do?

1. **Reason & Decide:** Makes decisions about what capabilities to use
2. **Initialize:** Starts MCP client, connects to servers
3. **Discover:** Learns what tools/resources/prompts are available
4. **Invoke:** Calls tools and processes results
5. **Integrate:** Incorporates results into its logic

### Example Host Workflow

```
Claude (Host) receives: "What's the customer status for ID 12345?"

1. Claude recognizes this needs customer data
2. Claude's MCP client contacts the Customer Server
3. Discovers: "I have a tool called get_customer"
4. Invokes: "get_customer(id=12345)"
5. Receives: Customer data
6. Responds: "The customer is Active, balance is $5,000"
```

## Component 2: MCP Client

### What Is It?

The **client** is software running inside the host that speaks the MCP protocol. It's the translator and dispatcher.

**The client is NOT:** A separate application. It's embedded in the host.

### What Does It Do?

1. **Connection Management:** Opens/maintains connections to servers (via configured transport)
2. **Message Formatting:** Converts host requests into MCP protocol messages
3. **Discovery:** Queries servers for capabilities
4. **Invocation:** Sends tool invocation requests
5. **Response Handling:** Deserializes responses into data the host understands
6. **Error Handling:** Manages protocol errors and timeouts

### The Client's Responsibility Boundary

**Client OWNS:**
- Protocol message formatting
- Transport layer communication
- Server capability discovery
- Serialization/deserialization

**Client DOESN'T OWN:**
- What tools to call (that's the host's logic)
- Tool result interpretation (that's the host's logic)
- Application-level decisions (that's the host's logic)

### Client Pseudo-code

```python
class MCPClient:
    def __init__(self, server_address, transport_type):
        # Initialize connection to server
        self.connection = establish_connection(server_address, transport_type)
    
    def discover_capabilities(self):
        # Send: What capabilities do you have?
        # Receive: List of tools, resources, prompts
        return self.send_message({
            "method": "capabilities_request"
        })
    
    def invoke_tool(self, tool_name, arguments):
        # Send: Please call this tool with these arguments
        # Receive: Tool result
        return self.send_message({
            "method": "call_tool",
            "params": {
                "name": tool_name,
                "arguments": arguments
            }
        })
```

## Component 3: MCP Server

### What Is It?

The **server** is an external service that exposes capabilities (tools, resources, prompts) through the MCP protocol.

**Examples of servers:**
- Database query server
- REST API wrapper
- File system server
- Specialized business logic server

### What Does It Do?

1. **Listen:** Receives MCP messages from clients
2. **Expose:** Declares available tools, resources, prompts
3. **Execute:** Runs tools when requested
4. **Return Results:** Sends back tool results or data

### The Server's Responsibility Boundary

**Server OWNS:**
- What capabilities it exposes
- How tools are implemented
- Data it manages
- Internal business logic

**Server DOESN'T OWN:**
- How the host uses the results
- What decisions the host makes
- What other servers the host connects to
- The host's overall logic

### Server Pseudo-code

```python
class MCPServer:
    def __init__(self):
        self.tools = {
            "get_customer": self.get_customer,
            "update_customer": self.update_customer
        }
    
    def handle_message(self, message):
        if message["method"] == "capabilities_request":
            return self.respond_capabilities()
        elif message["method"] == "call_tool":
            tool_name = message["params"]["name"]
            args = message["params"]["arguments"]
            result = self.tools[tool_name](**args)
            return self.respond_result(result)
    
    def get_customer(self, customer_id):
        # Query database
        return fetch_from_db(customer_id)
    
    def update_customer(self, customer_id, data):
        # Update database
        return save_to_db(customer_id, data)
```

## Message Flow: Complete Cycle

### Scenario: Host Calls a Tool

```
Step 1: Host initialization
Host: "I need to connect to servers"
MCP Client: "Let me establish connections"

Step 2: Capability discovery
Host: "What can you do?"
MCP Client: Sends discovery message to Server
Server: "I have tools: get_customer, update_customer"
MCP Client: Receives and caches capabilities
Host: "Good, I know what's available"

Step 3: Tool invocation
Host: "I need customer data for ID 123"
MCP Client: Sends {"method": "call_tool", "params": {"name": "get_customer", "arguments": {"id": 123}}}
Server: Receives, executes get_customer(123)
Server: Sends back {"result": {"name": "Alice", "balance": 5000}}
MCP Client: Deserializes response
Host: Receives {"name": "Alice", "balance": 5000}

Step 4: Host uses result
Host: Processes the customer data, makes decisions
```

## Transports: How Messages Travel

### What Is Transport?

Transport is the mechanism for moving MCP messages between client and server. The protocol doesn't care how; these are just wires.

### Common Transports

#### 1. **stdio (Standard Input/Output)**
- **Use Case:** Local process-to-process communication
- **How:** Messages are JSON lines on stdout/stdin
- **Stateless:** Can be (each message is independent)
- **Example:** IDE plugin to local analysis server

```
Host Process
    ▼
┌─────────────┐
│   stdin     │
└─────────────┘
    ▼
Server Process
    ▲
┌─────────────┐
│   stdout    │
└─────────────┘
```

#### 2. **HTTP/SSE (Server-Sent Events)**
- **Use Case:** Remote servers, web-based
- **How:** Client sends POST requests, server sends responses
- **Stateless:** Yes (each request is independent)
- **Example:** Cloud API server

```
GET /api/capabilities → Server sends back capabilities
POST /api/tools/call → Server executes tool, sends result
```

#### 3. **WebSocket**
- **Use Case:** Long-lived connections, real-time
- **How:** Bidirectional persistent connection
- **Stateless:** Can be (protocol layer is still stateless)
- **Example:** Live collaborative tools

```
┌──────────────────────────────┐
│   WebSocket Connection       │
│   (Persistent)               │
│                              │
│ Message 1 → Tool call        │
│ ← Result 1                   │
│ Message 2 → Resource read    │
│ ← Resource data              │
└──────────────────────────────┘
```

### Transport Doesn't Matter to Protocol

**Important:** The MCP protocol is **transport-agnostic**. Whether messages travel via HTTP or WebSocket, the protocol message structure is the same.

```python
# Message is the same regardless of transport
message = {
    "jsonrpc": "2.0",
    "method": "call_tool",
    "params": {
        "name": "get_customer",
        "arguments": {"id": 123}
    },
    "id": 1
}

# Transport chooses how to send it:
# - stdio: print(json.dumps(message))
# - HTTP: POST body
# - WebSocket: ws.send(json.dumps(message))
```

## Statelessness in Practice

### What "Stateless" Means in MCP

The protocol assumes **each message is complete and independent**. The server doesn't remember you from one message to the next.

### Why Stateless?

```
Stateful (Traditional):
Message 1: "Set session context"
Message 2: "Call tool" (uses context from Message 1)
⚠️  Problem: Server must remember; harder to scale

Stateless (MCP):
Message 1: "Call tool with ALL context included"
Message 2: "Call another tool with ALL context included"
✅ Benefit: Server doesn't remember; scales easily
```

### Example: Stateless Tool Invocation

```python
# Each request includes all needed context
Request 1: {
    "method": "call_tool",
    "params": {
        "name": "get_customer",
        "arguments": {"id": 123},
        "context": {"user": "alice", "auth_token": "..."}  # All context here
    }
}

# Next request is independent
Request 2: {
    "method": "call_tool",
    "params": {
        "name": "update_customer",
        "arguments": {"id": 123, "status": "premium"},
        "context": {"user": "alice", "auth_token": "..."}  # Context included again
    }
}

# Server doesn't need to remember Request 1 to handle Request 2
```

## One Host, Multiple Servers

A single host (application) can connect to multiple MCP servers simultaneously:

```
┌──────────────────────────────────┐
│  Host (Claude, LangChain, etc.)  │
│                                  │
│  ┌────────────────────────────┐  │
│  │  MCP Client 1              │  │
│  │  (Talks to Customer Server)│  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │  MCP Client 2              │  │
│  │  (Talks to Banking Server) │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │  MCP Client 3              │  │
│  │  (Talks to Document Server)│  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘
        ▼         ▼          ▼
    Customer  Banking   Document
    Server    Server    Server
```

**Key Point:** Each server connection is typically managed by a separate client instance, but all are orchestrated by the host application.

## Capabilities: What Servers Declare

When a server connects, it declares what it can do:

```python
{
    "protocol_version": "2024-11-05",  # or current version
    "capabilities": {
        "tools": {
            "listChanged": true  # Tools can change dynamically
        },
        "resources": {
            "subscribe": true  # Can subscribe to resource changes
        },
        "prompts": {
            "listChanged": true
        }
    }
}
```

The host knows what's possible based on capabilities.

## Architecture Patterns

### Pattern 1: Single Server

```
Application → MCP Client → MCP Server
             (tools discovery)
```

### Pattern 2: Multiple Servers

```
Application → MCP Client 1 → Server 1
           → MCP Client 2 → Server 2
           → MCP Client 3 → Server 3
```

### Pattern 3: Shared Servers (Used in Practical Project)

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ MCP Server: Cust │     │ MCP Server: Bank │     │ MCP Server: Docs │
│ (get_customer)   │     │ (transfer_money) │     │ (search_docs)    │
└────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
         │                        │                        │
         └────────────┬───────────┴──────────┬─────────────┘
                      │                      │
                      ▼                      ▼
            ┌─────────────────┐   ┌─────────────────┐
            │ Python Host     │   │ LangChain Agent │
            │ (MCP Client)    │   │ (MCP Client)    │
            └─────────────────┘   └─────────────────┘
```

All frameworks use the same servers.

## Interview Summary

**Key Concepts:**
- **Host:** The application making decisions
- **Client:** The protocol translator inside the host
- **Server:** The capability provider
- **Stateless:** Each message is independent
- **Transports:** HTTP, WebSocket, stdio—all carry the same protocol

**One-Liner:**
"MCP architecture separates the host (decision-maker), client (protocol handler), and server (capability provider), communicating via stateless JSON-RPC messages over various transports."

## Common Interview Questions

**Q1: What's the role of the MCP client?**
A: The MCP client is software running inside the host that implements the MCP protocol. It discovers capabilities, formats requests into MCP messages, sends them to servers, deserializes responses, and handles errors. It's the translator between the host's logic and the protocol.

**Q2: Can one host connect to multiple servers?**
A: Yes. A host typically maintains separate client connections to each server. The host orchestrates which server to contact based on its logic.

**Q3: Why is statelessness important in MCP?**
A: Statelessness means each request includes all needed context. This lets servers scale—one server instance can handle many concurrent clients without maintaining session state for each one.

**Q4: What does transport-agnostic mean?**
A: The MCP protocol message structure is the same regardless of how it travels. Whether messages go over HTTP, WebSocket, or stdio, the JSON-RPC message is identical. Transport is just the vehicle.

**Q5: Who decides what tools to call—the host or the client?**
A: The host decides. The client just implements the protocol. If the host wants to call a tool, the client formats the request and sends it.

---

**Next:** [03_MCP_Protocol.md](03_MCP_Protocol.md) – JSON-RPC, message structure, lifecycle
