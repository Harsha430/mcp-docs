# 08: Server vs Client vs Host vs Adapter – Critical Distinctions

**Version/Source Basis:** MCP Protocol as of July 28, 2026. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

## Why This Distinction Matters

Many interview questions hinge on understanding **who owns what responsibility**. Confusing these components will break your architecture explanations.

**Interview Question:** "In a LangChain + MCP architecture, what component initiates the tool discovery?"

**Wrong Answer:** "LangChain discovers tools."
**Right Answer:** "The MCP Client inside/used by LangChain discovers tools. LangChain doesn't know about MCP; the client does."

## The Four Components

### 1. MCP Host

**What is it?**
- Your primary application
- Makes decisions
- Orchestrates everything

**Examples:**
- Claude (AI Assistant)
- VS Code (IDE)
- A Python script
- LangChain application
- CrewAI application

**Responsibilities:**
- Decide what capabilities are needed
- Decide which servers to use
- Decide which tools to call
- Process tool results
- Make business decisions

**Does NOT own:**
- Protocol message formatting (client owns this)
- Connection establishment (client owns this)
- Server implementation (server owns this)

### 2. MCP Client

**What is it?**
- Software that implements the MCP protocol
- Lives inside (or is tightly integrated with) the host
- Handles all protocol details

**Examples:**
- `mcp.client.Client` class (SDK v2)
- LangChain's MCP adapter
- CrewAI's MCP integration

**Responsibilities:**
- Format JSON-RPC messages
- Manage transport (stdio, HTTP, etc.)
- Establish connections to servers
- Send/receive MCP protocol messages
- Deserialize responses
- Handle protocol-level errors

**Does NOT own:**
- What tools to call (host decides)
- How servers work (server owns this)
- Application logic (host owns this)

### 3. MCP Server

**What is it?**
- External service providing capabilities
- Independent from the host
- Implements specific business logic

**Examples:**
- Customer database server
- Banking transaction server
- Document search server
- Weather API server

**Responsibilities:**
- Expose tools, resources, prompts
- Implement tool logic
- Manage its own data/database
- Validate inputs
- Return results
- Handle its own errors

**Does NOT own:**
- How results are used (host uses them)
- Other servers (runs independently)
- Protocol-level routing (client handles this)

### 4. Framework Adapter (LangChain, CrewAI, etc.)

**What is it?**
- Bridge between a framework and MCP
- NOT the same as MCP Client
- Sometimes includes the MCP Client

**Examples:**
- LangChain's `MCPIntegration` or similar
- CrewAI's MCP adapter
- AutoGen's MCP connector

**Responsibilities:**
- Create/manage MCP clients
- Translate framework tool format → MCP format
- Translate MCP results → framework format
- Expose MCP tools to framework agents

**Does NOT own:**
- MCP Protocol (client owns this)
- Agent decision-making (framework owns this)

## Visual Responsibility Chart

```
┌─────────────────────────────────────────────────────────┐
│  HOST (Claude, LangChain, Your Script)                  │
│                                                          │
│  Owns: Decision-making, business logic, tool selection  │
│  Question: "Which tool should I call?"                  │
│  Does NOT own: Protocol, connection details             │
└─────────────────────────────────────────────────────────┘
         │
         │ Delegates to
         ▼
┌─────────────────────────────────────────────────────────┐
│  MCP CLIENT (SDK v2 Client, Adapter's MCP layer)        │
│                                                          │
│  Owns: Protocol messages, transport, connections        │
│  Question: "How do I send this over MCP?"               │
│  Does NOT own: Business logic, result interpretation    │
└─────────────────────────────────────────────────────────┘
         │
         │ Communicates via MCP Protocol
         ▼
┌─────────────────────────────────────────────────────────┐
│  MCP SERVER (Customer, Banking, Document Server)        │
│                                                          │
│  Owns: Tool implementation, data, its own logic         │
│  Question: "How do I execute this tool?"                │
│  Does NOT own: How results are used, other servers      │
└─────────────────────────────────────────────────────────┘
```

## Concrete Example: LangChain + MCP

### Component Breakdown

```
┌──────────────────────────────────────────┐
│  LangChain Agent (Host)                  │
│  - Reasons about a task                  │
│  - Decides: "I need customer data"       │
│  - Calls MCP tools through adapter       │
└──────────────────────────────────────────┘
           │
           │ Uses
           ▼
┌──────────────────────────────────────────┐
│  LangChain MCP Adapter                   │
│  - Includes/uses MCP Client              │
│  - Translates LangChain tools ↔ MCP      │
│  - Manages MCP client lifecycle          │
└──────────────────────────────────────────┘
           │
           │ Uses
           ▼
┌──────────────────────────────────────────┐
│  MCP Client (SDK v2)                     │
│  - Formats JSON-RPC messages             │
│  - Manages HTTP/stdio connections        │
│  - Discovers capabilities                │
└──────────────────────────────────────────┘
           │
           │ Communicates via MCP Protocol
           ▼
┌──────────────────────────────────────────┐
│  MCP Servers (Remote or Local)           │
│  - Customer Server (tools, resources)    │
│  - Banking Server (tools, resources)     │
└──────────────────────────────────────────┘
```

### Question: "Who discovers tools?"

**Wrong:** "LangChain discovers tools"
**Right:** "The MCP Client (via LangChain's adapter) sends a discovery message to the MCP Server. The server responds with the tool list. LangChain's adapter receives this and exposes it to the agent."

### Question: "Who makes the decision to call a tool?"

**Wrong:** "The MCP Server decides"
**Right:** "The LangChain Agent (host) decides which tool to call. It passes the request to the adapter, which uses the MCP Client to send the call to the server."

## Concrete Example: Custom Python App + MCP

### Component Breakdown

```
┌──────────────────────────────────────────┐
│  My Python App (Host)                    │
│  async def main():                       │
│    result = await process_order(...)     │
└──────────────────────────────────────────┘
           │
           │ Owns
           ▼
┌──────────────────────────────────────────┐
│  Business Logic                          │
│  - Check if customer exists              │
│  - Decide to transfer money              │
│  - Make business decisions               │
└──────────────────────────────────────────┘
           │
           │ Uses
           ▼
┌──────────────────────────────────────────┐
│  MCP Client (SDK v2: Client class)       │
│  - Sends: "list_tools" message           │
│  - Sends: "call_tool" message            │
│  - Receives: tool results                │
└──────────────────────────────────────────┘
           │
           │ Over Transport (HTTP, stdio)
           ▼
┌──────────────────────────────────────────┐
│  MCP Servers                             │
│  - Customer Server                       │
│  - Banking Server                        │
└──────────────────────────────────────────┘
```

### Code Showing Responsibility

```python
# Host: Makes decision
async def main():  # This is the HOST
    result = await app.process_order(customer_id="123", amount=500)
    # Host logic: decide what to do with result
    if "success" in result:
        print("Order processed")

# Application logic (still Host)
async def process_order(self, customer_id, amount):
    # Host decides: I need customer info, I need banking info
    customer = await self.customers_client.call_tool(...)  # Host decides
    balance = await self.banking_client.call_tool(...)     # Host decides
    
    # Host logic: combine and decide next step
    if balance > amount:
        return "Can process"
    else:
        return "Insufficient funds"

# MCP Client: Handles protocol (inside client.py)
async def call_tool(self, tool_name, arguments):
    # Client responsibility: format MCP message
    message = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "params": {"name": tool_name, "arguments": arguments}
    }
    # Client responsibility: send over transport
    await self.transport.send(message)
    # Client responsibility: receive response
    response = await self.transport.receive()
    return response

# Server: Implements tool (in server.py)
@server.tool()
def get_customer(customer_id: str):
    # Server responsibility: implement the tool
    result = database.query(customer_id)
    # Server responsibility: return result
    return result
```

## Common Confusion Points

### Confusion 1: "MCP discovers tools"
**Wrong.** MCP is a protocol (a specification). Clients implement it, servers respond to it.
**Right:** "The MCP Client sends a discovery message following the MCP protocol. The server responds with the tool list."

### Confusion 2: "The host implements tools"
**Wrong.** The host uses tools, doesn't implement them.
**Right:** "The server implements tools. The host decides which tools to use."

### Confusion 3: "The client makes business decisions"
**Wrong.** The client is a protocol translator, not a decision-maker.
**Right:** "The host makes business decisions. The client transmits the decision to the server."

### Confusion 4: "LangChain's adapter IS the MCP client"
**Partially right, but imprecise.** The adapter contains or uses the MCP client.
**Better:** "LangChain's adapter includes/wraps an MCP client to integrate MCP servers into LangChain."

## Responsibility Flow in a Tool Call

```
Step 1: Host makes decision
"I need customer data"
│
├─ Host: "Call get_customer on customer server"
│
Step 2: Client translates
├─ Client: Format as JSON-RPC message
├─ Client: Send over HTTP/stdio
│
Step 3: Server executes
├─ Server: Receive message
├─ Server: Execute get_customer tool
├─ Server: Return result as JSON-RPC response
│
Step 4: Client receives and translates back
├─ Client: Deserialize JSON-RPC response
├─ Client: Convert to Python object
│
Step 5: Host processes result
├─ Host: "Now I have the data"
├─ Host: Make next decision
```

## Authentication/Authorization Responsibility

```
Host:
- Decides: "I'm user alice"
- Provides: credentials/token

Client:
- Includes auth in message: "Authorization: Bearer token"
- Handles: HTTP headers, transport-level auth

Server:
- Receives: authorization header
- Validates: "Is this token valid?"
- Decides: "Can alice access customer data?"
- Returns: Result (if authorized) or error (if not)
```

## Testing: Understanding Responsibility

```python
# Test 1: Host logic
def test_process_order():
    # Test: Host makes correct decisions
    result = process_order(customer_id="123", amount=500)
    assert "success" in result  # Host decides what's successful

# Test 2: Client behavior
def test_client_call_tool():
    # Test: Client sends correct MCP message
    client = Client(mock_transport)
    await client.call_tool("get_customer", {...})
    assert mock_transport.sent == expected_json_rpc_message

# Test 3: Server behavior
def test_server_get_customer():
    # Test: Server implements tool correctly
    server = Server()
    result = server.get_customer(customer_id="123")
    assert result.name == "Alice"
```

## Interview Preparation

### Q1: "In your architecture, who decides which tool to call?"
**A:** "The host (my application) decides. It makes business decisions and requests specific tools. The client transmits the request to the server."

### Q2: "What does the MCP Client do?"
**A:** "The client handles the MCP protocol—formatting JSON-RPC messages, managing connections (transport), discovering capabilities from servers, and deserializing responses. It's the protocol translator."

### Q3: "Can a server call another server?"
**A:** "Not directly through MCP. The host is the orchestrator. If needed, one server can call another as external service integration (not via MCP)."

### Q4: "Where does authentication happen?"
**A:** "The host provides credentials (decides to authenticate). The client includes auth in MCP messages. The server validates the credentials and decides authorization."

### Q5: "What makes something a 'host'?"
**A:** "Something is a host if it orchestrates MCP clients to accomplish a goal. Claude is a host. LangChain agents are hosts. Your Python script using MCP is a host."

## Interview Summary

**Key Distinctions:**
- **Host:** Decision-maker, orchestrator
- **Client:** Protocol translator, connection manager
- **Server:** Capability provider, tool implementer
- **Adapter:** Framework-specific bridge to MCP

**One-Liner:**
"The host makes decisions, the client implements MCP protocol, servers provide capabilities, and adapters bridge frameworks to MCP."

---

**Next:** Framework Integrations: [09_LangChain_MCP.md](09_LangChain_MCP.md)
