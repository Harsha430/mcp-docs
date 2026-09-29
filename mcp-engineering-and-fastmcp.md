# MCP Engineering & FastMCP: Complete Technical Reference

**Version Status:**
- Python: 3.13.7
- MCP Python SDK: 1.28.1 (anthropic/mcp-python-sdk)
- FastMCP: Requires separate installation (`pip install fastmcp`)
- CrewAI: 1.15.22

---

## PART 1: Why MCP Exists

### The Integration Problem Before MCP

Before MCP, the landscape looked like this:

```
AI Application A    AI Application B    AI Application C
     ↓                    ↓                    ↓
  GitHub              GitHub               GitHub
  Custom              Custom               Custom
  Integration         Integration          Integration
```

This multiplication problem gets worse:

```
                          External Systems
                          ├── GitHub
                          ├── Slack
                          ├── Databases
                          ├── Filesystems
                          ├── APIs
                          ├── Internal Enterprise Systems
                          └── ... 100s more

Applied to:
    ├── ChatGPT plugins
    ├── LangChain agents
    ├── CrewAI agents
    ├── Custom LLM applications
    ├── Multi-agent orchestration
    └── Enterprise AI platforms
```

**Consequences without MCP:**

| Problem | Impact |
|---------|--------|
| **Duplicate integrations** | Each AI system independently implements GitHub, database, filesystem adapters |
| **Custom adapters** | No standard interface; teams write their own glue code |
| **Fragile integrations** | Changes to one external system break multiple applications |
| **Inconsistent interfaces** | Each application exposes capabilities differently to models |
| **Hard to discover capabilities** | No standardized way for agents to learn what's available |
| **Difficult maintenance** | One system updates; N integrations must change |
| **Capability mismatch** | No clear contract between AI app and external system |

### Why MCP Exists: The Solution

MCP introduces a **standardized protocol and architecture** for AI applications to discover and use external capabilities.

```
                        MCP Protocol
                        ───────────

AI Application / LLM              External Systems
    ↓                                  ↓
  MCP Client ←──────────────────→  MCP Server
             standardized protocol     ├── GitHub
             request/response          ├── Slack
             discovery                 ├── Database
             execution                 ├── Filesystem
                                       └── Any capability
```

**Key insight:** Instead of n×m integrations, MCP provides:
- **1 standard protocol** that all AI apps speak
- **1 standard interface** that all external systems expose
- **Automatic discovery** so agents learn capabilities dynamically
- **Consistent execution** through a uniform request/response cycle

---

## PART 2: What Is MCP?

### Definition

**MCP = Model Context Protocol**

Breaking down the name:
- **Model**: Any AI model or agent system (LLM, Claude, custom agent framework)
- **Context**: The information and capabilities available to the model
- **Protocol**: Standardized rules for communication

### Precise Definition

**MCP is a protocol that enables standardized communication between:**
1. **Host/Client** (an AI application or agent)
2. **Server** (an external system exposing capabilities)

Through:
- **Tools** (actions/functions the model can invoke)
- **Resources** (readable context/data)
- **Prompts** (reusable instruction templates)

### What MCP Is NOT

| Is NOT | Actually |
|--------|----------|
| **An LLM** | A protocol; works with any LLM/agent |
| **An agent** | Agents use MCP; MCP doesn't think/reason |
| **A tool** | Defines how tools are discovered and executed |
| **An API** | Protocol layer above APIs; standardizes access |
| **A vector database** | Can provide context TO vector systems |
| **An orchestration framework** | Can support multi-agent systems |
| **A framework** | Protocol specification; frameworks implement it |

### MCP vs Related Concepts

| Concept | Definition | Relationship to MCP |
|---------|-----------|-------------------|
| **Protocol** | Rules for communication | MCP IS a protocol |
| **API** | Application Programming Interface | MCP abstracts over APIs |
| **Tool** | Something a model can call | MCP Tool = standardized tool |
| **Tool Calling** | LLM selecting and executing a tool | MCP handles execution; LLM handles selection |
| **Agent** | System that reasons, plans, acts | Agent uses MCP to act |
| **Workflow** | Sequence of steps | MCP provides capabilities for workflows |
| **LangChain Tool** | Tool in Python abstraction | MCP Tool is protocol-level; LangChain wraps MCP |

---

## PART 3: MCP Architecture

### High-Level Architecture

```
                        ┌─────────────────────┐
                        │  HOST APPLICATION   │
                        │  ┌───────────────┐  │
                        │  │  LLM / Agent  │  │
                        │  └───────────────┘  │
                        │                     │
                        │  ┌───────────────┐  │
                        │  │  MCP CLIENT   │  │
                        │  └───────┬───────┘  │
                        └──────────┼──────────┘
                                   │
                        MCP Protocol (JSON-RPC 2.0)
                        Over Transport (stdio, HTTP, etc.)
                                   │
                        ┌──────────▼──────────┐
                        │   MCP SERVER       │
                        │ (Capabilities)     │
                        ├───────────────────┤
                        │  • Tools           │
                        │  • Resources       │
                        │  • Prompts         │
                        └──────────┬─────────┘
                                   │
                        ┌──────────▼──────────┐
                        │  EXTERNAL SYSTEMS  │
                        │  ├── GitHub API    │
                        │  ├── Database      │
                        │  ├── Filesystem    │
                        │  └── Custom APIs   │
                        └────────────────────┘
```

### Component Responsibilities

#### HOST

**Definition:** The application running the LLM/agent

**Responsibility:**
- Integrates an MCP client
- Manages the reasoning loop
- Calls MCP client to discover capabilities
- Handles LLM tool-calling responses
- Presents final answers to users

**Examples:**
- LangChain Agent Application
- Claude with MCP integration
- CrewAI Agent
- Custom multi-agent system
- IDE with Claude integration

**Key point:** Host ≠ MCP Client. Host contains the client.

#### MCP CLIENT

**Definition:** Part of the host; handles protocol communication

**Responsibility:**
- Establishes connection to MCP server
- Sends ListTools/ListResources/ListPrompts requests
- Receives capability metadata
- Translates LLM tool-calls to CallToolRequest
- Receives ToolResult from server
- Handles errors and retries
- Manages session lifecycle

**Examples:**
- mcp.client.ClientSessionGroup (MCP Python SDK)
- fastmcp.Client (FastMCP)
- LangChain MCP integration

**Key point:** MCP Client does not reason. It communicates.

#### MCP SERVER

**Definition:** External system exposing capabilities

**Responsibility:**
- Registers tools (name, description, schema)
- Registers resources (URIs, descriptions)
- Registers prompts (names, arguments)
- Receives CallToolRequest
- Executes the requested tool
- Returns ToolResult
- Sends notifications (optional)

**Examples:**
- FastMCP server (e.g., `@mcp.tool()` decorated functions)
- MCP Python SDK server
- Custom MCP server implementation

---

## PART 4: Host vs Client vs Server

### Comparison Table

| Aspect | Host | MCP Client | MCP Server |
|--------|------|-----------|-----------|
| **Role** | Runs LLM/agent reasoning | Communicates with server | Exposes capabilities |
| **Where** | Local application | Part of host | Remote/local process |
| **Creates** | MCP Client instance | Connection to server | Tools, resources, prompts |
| **Receives** | User query | List of capabilities | Requests to execute |
| **Sends** | Decision to call tool | CallToolRequest | ToolResult |
| **Example** | LangChain app | ClientSessionGroup | FastMCP server |
| **Thinks** | Yes (reasoning) | No | No (executes only) |
| **Decides** | What tool to use | How to communicate | N/A |
| **Acts** | Coordinates | Relays messages | Executes tools |

### Critical Distinctions

**Host is NOT MCP Client:**
```python
# Host application
class MyAgent:
    def __init__(self):
        self.mcp_client = ClientSessionGroup()  # ← Host creates client
    
    async def reason(self, query):
        tools = await self.mcp_client.list_tools()  # ← Client communicates
        # Agent decides which tool to use
        result = await self.mcp_client.call_tool(...)  # ← Client executes
        return final_answer
```

**Agent is NOT MCP Client:**
```
Agent = Reasoning System (decides what to do)
MCP Client = Communication System (how to do it)

Agent asks: "What tools are available?"
MCP Client answers: "Here's what the server exposes"

Agent asks: "Call the GitHub tool"
MCP Client asks: "MCP Server, execute this tool"
MCP Client returns: "Here's the result"
```

---

## PART 5: MCP Protocol Fundamentals

### What Is a Protocol?

A **protocol** is a set of rules defining:
1. **Message structure** (what data is sent)
2. **Message semantics** (what messages mean)
3. **Sequencing** (what order messages arrive in)
4. **Error handling** (what happens when things fail)

**MCP Protocol** = Rules for communication between MCP Client and MCP Server

**Transport** = Physical mechanism for delivering messages (stdio, HTTP, WebSocket, etc.)

**Key distinction:** Protocol is independent of transport.

### MCP Protocol Fundamentals (MCP 1.28.1)

MCP is built on **JSON-RPC 2.0** specification.

**Core message types:**

1. **Request** - Client asks server to do something
2. **Response** - Server answers with result or error
3. **Notification** - Server sends unsolicited information
4. **Error** - Structured error response

### Simplified Message Flow

```
Client                                  Server
  │                                       │
  ├─────── InitializeRequest ────────────►
  │                                       │
  │◄─────── InitializeResponse ──────────┤
  │                                       │
  ├────── ListToolsRequest ──────────────►
  │                                       │
  │◄────── ListToolsResponse ────────────┤
  │ [Tool 1, Tool 2, ...]                │
  │                                       │
  ├─────── CallToolRequest ──────────────►
  │ {tool_name, arguments}                │
  │                                       │
  │◄────── CallToolResponse ──────────────┤
  │ {result}                              │
  │                                       │
  └────── CloseConnection ──────────────►
```

### Protocol Operations (MCP 1.28.1)

| Operation | Sender | Receiver | Purpose |
|-----------|--------|----------|---------|
| **initialize** | Client | Server | Begin session, exchange capabilities |
| **list_tools** | Client | Server | Discover available tools |
| **call_tool** | Client | Server | Execute a tool |
| **list_resources** | Client | Server | Discover available resources |
| **read_resource** | Client | Server | Fetch resource content |
| **list_prompts** | Client | Server | Discover available prompts |
| **get_prompt** | Client | Server | Fetch prompt messages |

---

## PART 6: JSON-RPC & Message Anatomy

### JSON-RPC 2.0 Request

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

**Field meanings:**

| Field | Required | Type | Meaning |
|-------|----------|------|---------|
| `jsonrpc` | Yes | String | Always "2.0" (JSON-RPC version) |
| `id` | Yes* | String/Number | Request ID to match response; not required for notifications |
| `method` | Yes | String | The operation to invoke (e.g., "tools/list") |
| `params` | No | Object | Arguments to the method |

*Omitted for notifications (server→client unsolicited messages)

### JSON-RPC 2.0 Response (Success)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "add",
        "description": "Add two numbers",
        "inputSchema": {
          "type": "object",
          "properties": {
            "a": {"type": "integer"},
            "b": {"type": "integer"}
          },
          "required": ["a", "b"]
        }
      }
    ]
  }
}
```

**Field meanings:**

| Field | Present When | Meaning |
|-------|--------------|---------|
| `jsonrpc` | Always | "2.0" |
| `id` | Always | Matches the request ID |
| `result` | Success | The result data; structure depends on method |

### JSON-RPC 2.0 Response (Error)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32600,
    "message": "Invalid Request",
    "data": {
      "description": "The request is malformed"
    }
  }
}
```

**Error field meanings:**

| Field | Meaning |
|-------|---------|
| `code` | Standard JSON-RPC error code (-32700 to -32600) or custom code |
| `message` | Human-readable error message |
| `data` | Optional additional error details |

### Common JSON-RPC Error Codes

| Code | Meaning |
|------|---------|
| -32700 | Parse error (invalid JSON) |
| -32600 | Invalid Request |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |
| -32099 to -32000 | Server error (reserved) |

### JSON-RPC Notification (Server → Client)

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/list_changed",
  "params": {}
}
```

**Characteristics:**
- No `id` field
- Server sends unsolicited
- Client doesn't send response
- Example: Resource updated, logging message

---

## PART 7: MCP Lifecycle

### Connection & Session Initialization Flow

```
Client                          Server
  │                               │
  ├─ connect via transport ──────►│
  │ (stdio, HTTP, etc.)            │
  │                                │
  ├─ InitializeRequest ───────────►│
  │ {clientInfo, capabilities}     │
  │                                │
  │◄────── InitializeResponse ────┤
  │ {serverInfo, capabilities}    │
  │                                │
  │◄────────── Resources ──────────┤
  │ (ResourceUpdated notification) │
  │                                │
  ├─ Initialized notification ───►│
  │                                │
  ├─ ListTools ──────────────────►│
  │                                │
  │◄─ ListToolsResult ────────────┤
  │ [Tool 1, Tool 2, ...]         │
  │                                │
  ├─ CallTool ───────────────────►│
  │ {name, arguments}             │
  │                                │
  │◄─ CallToolResult ─────────────┤
  │ {result}                       │
  │                                │
  └─ Close connection ───────────►│
```

### Detailed Initialization Sequence

1. **Transport Connection**
   - Client connects via transport (stdio subprocess, HTTP endpoint, etc.)
   - Establishes reliable message channel

2. **InitializeRequest** (Client → Server)
   ```python
   {
     "jsonrpc": "2.0",
     "id": 1,
     "method": "initialize",
     "params": {
       "protocolVersion": "2024-11-05",  # MCP version
       "capabilities": {},                # Client capabilities
       "clientInfo": {
         "name": "my-agent",
         "version": "1.0.0"
       }
     }
   }
   ```

3. **InitializeResponse** (Server → Client)
   ```python
   {
     "jsonrpc": "2.0",
     "id": 1,
     "result": {
       "protocolVersion": "2024-11-05",
       "capabilities": {              # Server capabilities
         "tools": {},
         "resources": {}
       },
       "serverInfo": {
         "name": "knowledge-server",
         "version": "1.0.0"
       }
     }
   }
   ```

4. **Initialized Notification** (Client → Server)
   - Acknowledges initialization complete

5. **Resource Updates** (Server → Client, optional)
   - Server may send ResourceUpdated notifications
   - Tells client about available resources

6. **Session Active**
   - Client can now list and call tools
   - Client can read resources
   - Client can get prompts

7. **Close**
   - Client closes connection
   - Server cleans up resources
   - Subprocess terminates (for stdio)

---

## PART 8: Capabilities

### What Are Capabilities?

**Capabilities** declare what each side can do and what features they support.

**Client capabilities** = What the client can handle
- Sampling (can the LLM sample from server?)
- Roots (can the server specify filesystem roots?)
- Logging (can the server send logs?)

**Server capabilities** = What the server exposes
- Tools (server exposes tools)
- Resources (server exposes resources)
- Prompts (server exposes prompts)

### Server Capabilities (MCP 1.28.1)

```json
{
  "capabilities": {
    "tools": {},
    "resources": {},
    "prompts": {},
    "logging": {},
    "roots": {
      "listChanged": true
    }
  }
}
```

| Capability | Meaning |
|-----------|---------|
| **tools** | Server implements tools that clients can call |
| **resources** | Server provides resources (readable data) |
| **prompts** | Server provides prompts (instruction templates) |
| **logging** | Server can send log messages to client |
| **roots** | Server can specify filesystem roots (multi-tenancy) |

### Client Capabilities (MCP 1.28.1)

```json
{
  "capabilities": {
    "sampling": {},
    "roots": {}
  }
}
```

| Capability | Meaning |
|-----------|---------|
| **sampling** | Client LLM can sample from server (e.g., for verification) |
| **roots** | Client can specify allowed filesystem roots |

### Important: Not All Servers Have All Capabilities

A server implementation chooses what to expose:

```
Server A: Supports tools only
  └── Capabilities: {tools: {}}

Server B: Supports tools and resources
  └── Capabilities: {tools: {}, resources: {}}

Server C: Supports all
  └── Capabilities: {tools: {}, resources: {}, prompts: {}}
```

Client must check what server actually supports before using it.

---

## PART 9: Three Core MCP Primitives

### The Three Building Blocks

MCP defines three core primitives for extending AI capabilities:

| Primitive | Purpose | Side Effects | Execution |
|-----------|---------|--------------|-----------|
| **Tool** | Action/function | Yes (modifies state) | Synchronous call |
| **Resource** | Data/context | No (read-only) | Fetch content |
| **Prompt** | Instruction template | No (information only) | Retrieve template |

### Tool

**Definition:** An action the model can invoke that typically has side effects.

**Mental model:** "Do something for me"

**Examples:**
- Create GitHub issue
- Query database
- Write file
- Send Slack message
- Deploy application
- Execute SQL

**Characteristics:**
- Exposes input schema
- Takes arguments
- Returns result
- May fail
- May modify external state

### Resource

**Definition:** Readable context/data that provides information to the model.

**Mental model:** "Tell me about this"

**Examples:**
- GitHub pull request content
- Project documentation
- Database query result
- File contents
- Configuration

**Characteristics:**
- Addressed by URI (e.g., `github://pull/123`)
- Read-only
- Returns content
- Cannot fail (should handle gracefully)
- Discovered via resource templates

### Prompt

**Definition:** Reusable instruction template that guides model behavior.

**Mental model:** "How should I approach this?"

**Examples:**
- "Summarize a GitHub issue"
- "Review code for security"
- "Generate test cases"
- "Explain database schema"

**Characteristics:**
- Takes arguments
- Returns structured messages
- Guides model reasoning
- Can reference tools and resources

---

## PART 10: Tools Deep Dive

### What Is an MCP Tool?

An **MCP Tool** is a standardized way to expose a callable action.

**Unlike an API:**
- APIs require custom client code per API
- MCP tools are auto-discovered
- MCP tools have standardized schema
- MCP tools are callable by any MCP client

**A tool may internally:**
- Call an API
- Execute a function
- Query a database
- Interact with a service
- Modify files/systems

### Tool Components

Every tool consists of:

1. **Name** - Unique identifier (e.g., "create_issue")
2. **Description** - Human-readable explanation
3. **Input Schema** - JSON Schema defining arguments
4. **Output Semantics** - What the result represents

### Example: GitHub Issue Tool

```
Tool: create_github_issue

Description:
  Create a new GitHub issue in a repository

Input Schema:
  {
    "repo": "string (required)",
    "title": "string (required)",
    "body": "string (optional)",
    "labels": "array of strings (optional)"
  }

Internal Implementation:
  1. Validate inputs
  2. Call GitHub API
  3. Parse response
  4. Return issue data

Result:
  {
    "issue_id": 123,
    "url": "https://github.com/..."
  }
```

### Tool Execution Flow

```
LLM/Agent                MCP Client              MCP Server
    │                         │                      │
    ├─ "I need to create ─────┼─ ListTools ────────►│
    │  an issue"              │                      │
    │                         │◄─ [create_issue] ───┤
    │◄─ "I'll use ────────────┤                      │
    │  create_issue tool"     │                      │
    │                         │                      │
    ├─ "Call create_issue ────┼─ CallTool ─────────►│
    │  with title='bug'"      │ {arguments...}      │
    │                         │                      │
    │                         │◄─ ToolResult ───────┤
    │◄─ "Issue created ───────┤ {issue_id: 123}    │
    │  #123"                  │                      │
    │                         │                      │
```

---

## PART 11: Tool Discovery

### What Is Tool Discovery?

**Tool Discovery** = Client queries server for available tools

### ListToolsRequest Flow

```
Client                          Server
  │                               │
  ├─ ListToolsRequest ───────────►│
  │ {jsonrpc, id, method, params}  │
  │                                │
  │◄─ ListToolsResponse ──────────┤
  │ {tools: [Tool1, Tool2, ...]}   │
  │                                │
```

### Complete Discovery Sequence

```
User Input
    ↓
Agent/LLM
    ↓
MCP Client.list_tools()
    ↓
Transport (stdio/HTTP)
    ↓
MCP Server (Python function receives request)
    ↓
Server.list_tools() handler
    ↓
Returns tool definitions
    ↓
Transport (response back)
    ↓
MCP Client receives response
    ↓
Agent/LLM sees: [tool_name, description, schema]
    ↓
Agent stores in context
    ↓
Agent can select these tools for future calls
```

### ListToolsRequest (MCP 1.28.1)

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list",
  "params": {}
}
```

**Parameters:** (None - server returns all tools it exposes)

### ListToolsResponse

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "multiply",
        "description": "Multiply two numbers",
        "inputSchema": {
          "type": "object",
          "properties": {
            "a": {
              "type": "integer",
              "description": "First number"
            },
            "b": {
              "type": "integer",
              "description": "Second number"
            }
          },
          "required": ["a", "b"]
        }
      },
      {
        "name": "search_db",
        "description": "Search the knowledge database",
        "inputSchema": {
          "type": "object",
          "properties": {
            "query": {
              "type": "string",
              "description": "Search query"
            },
            "limit": {
              "type": "integer",
              "description": "Max results (default 10)"
            }
          },
          "required": ["query"]
        }
      }
    ]
  }
}
```

### Tool Definition Structure

| Field | Type | Required | Purpose |
|-------|------|----------|---------|
| `name` | string | Yes | Unique tool identifier |
| `description` | string | Yes | Human-readable explanation |
| `inputSchema` | JSON Schema | Yes | Argument schema |

### Input Schema

Input schema uses **JSON Schema** format to describe tool arguments:

```json
{
  "type": "object",
  "properties": {
    "param1": {
      "type": "string",
      "description": "What this parameter does"
    },
    "param2": {
      "type": "integer",
      "description": "What this parameter does"
    },
    "optional_param": {
      "type": "boolean",
      "description": "This is optional"
    }
  },
  "required": ["param1", "param2"]
}
```

---

## PART 12: Tool Schema Generation

### From Python to Schema

This is where **FastMCP shines**. It automatically generates schemas from Python code.

### FastMCP Tool Schema Generation

**Python Function:**
```python
@mcp.tool()
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b
```

**Generated Schema:**
```json
{
  "name": "multiply",
  "description": "Multiply two numbers.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "a": {
        "type": "integer"
      },
      "b": {
        "type": "integer"
      }
    },
    "required": ["a", "b"]
  }
}
```

### Schema Generation Rules (FastMCP)

| Python Type | JSON Schema Type | Example |
|-------------|-----------------|---------|
| `int` | `"integer"` | `{"type": "integer"}` |
| `float` | `"number"` | `{"type": "number"}` |
| `str` | `"string"` | `{"type": "string"}` |
| `bool` | `"boolean"` | `{"type": "boolean"}` |
| `list` | `"array"` | `{"type": "array", "items": {...}}` |
| `dict` | `"object"` | `{"type": "object", "properties": {...}}` |
| `Optional[T]` | `T` (not in required) | Not included in `required` array |

### Optional Parameters

```python
@mcp.tool()
def search(query: str, limit: Optional[int] = 10) -> list:
    """Search the database.
    
    Args:
        query: Search terms
        limit: Maximum results
    """
    pass
```

**Generated Schema:**
```json
{
  "name": "search",
  "description": "Search the database.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search terms"
      },
      "limit": {
        "type": "integer",
        "description": "Maximum results"
      }
    },
    "required": ["query"]
  }
}
```

### Complex Types

```python
from pydantic import BaseModel

class SearchResult(BaseModel):
    id: int
    title: str
    score: float

@mcp.tool()
def search(query: str) -> list[SearchResult]:
    """Search and return results."""
    pass
```

**Generated Schema:**
```json
{
  "name": "search",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search query"
      }
    },
    "required": ["query"]
  }
}
```

### Validation

When MCP client calls tool with arguments, validation occurs:

```
CallToolRequest
{
  "arguments": {
    "a": "not a number"  ← Wrong type!
  }
}
    ↓
FastMCP/MCP Server
    ↓
Validate against schema
    ↓
Type mismatch detected
    ↓
Return error: Invalid params
```

---

## PART 13: Tool Execution

### What Happens When a Tool Is Called?

**CallToolRequest** = Client asks server to execute a specific tool

### Call Flow

```
LLM Decision:
  "I need to multiply 3 × 4"
    ↓
MCP Client creates CallToolRequest
    ↓
{
  "method": "tools/call",
  "params": {
    "name": "multiply",
    "arguments": {
      "a": 3,
      "b": 4
    }
  }
}
    ↓
Transport to MCP Server
    ↓
Server receives request
    ↓
Finds tool named "multiply"
    ↓
Validates arguments against schema
    ↓
Executes tool (Python function)
    ↓
Tool returns: 12
    ↓
Server wraps result
    ↓
Returns CallToolResponse
    ↓
{
  "content": [
    {
      "type": "text",
      "text": "12"
    }
  ]
}
    ↓
Transport to Client
    ↓
MCP Client unpacks response
    ↓
Host application receives result
    ↓
LLM processes result
    ↓
Final answer: "3 × 4 = 12"
```

### Critical Point: LLM Does NOT Directly Call MCP Tool

```
WRONG WAY:
┌─────────────────────────────────────┐
│ LLM                                 │
│ ├─ Selects "multiply"               │
│ └─ Directly calls MCP Server ✗      │
│    (LLMs cannot initiate network)   │
└─────────────────────────────────────┘

CORRECT WAY:
┌─────────────────────────────────────┐
│ LLM                                 │
│ └─ "I want to call multiply tool"   │
└─────────────────────────────────────┘
         ↓ LLM returns text decision
┌─────────────────────────────────────┐
│ Host Application                    │
│ ├─ Parses LLM response              │
│ ├─ Creates CallToolRequest          │
│ └─ Calls MCP Client                 │
└─────────────────────────────────────┘
         ↓ Client uses protocol
┌─────────────────────────────────────┐
│ MCP Server                          │
│ ├─ Receives CallToolRequest         │
│ ├─ Validates arguments              │
│ ├─ Executes tool                    │
│ └─ Returns result                   │
└─────────────────────────────────────┘
```

### CallToolRequest (MCP 1.28.1)

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "multiply",
    "arguments": {
      "a": 3,
      "b": 4
    }
  }
}
```

**Parameters:**

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `name` | string | Yes | Name of tool to call |
| `arguments` | object | Yes | Arguments matching input schema |

### CallToolResponse

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "12"
      }
    ],
    "isError": false
  }
}
```

---

## PART 14: Tool Result Anatomy

### ToolResult Structure (MCP 1.28.1)

```python
{
  "content": [
    {
      "type": "text",
      "text": "result content"
    }
  ],
  "isError": False
}
```

### Content Types

#### Text Content

```python
{
  "type": "text",
  "text": "Plain text response"
}
```

Use for:
- Text results
- Error messages
- Any string output

#### Image Content

```python
{
  "type": "image",
  "data": "base64-encoded-image-data",
  "mimeType": "image/png"
}
```

Use for:
- Screenshot results
- Generated images
- Visualizations

#### Embedded Resource

```python
{
  "type": "resource",
  "uri": "file://path/to/file",
  "mimeType": "text/plain"
}
```

Use for:
- Referencing files
- Large documents
- External content

### Error Results

```python
{
  "content": [
    {
      "type": "text",
      "text": "Error: Invalid input"
    }
  ],
  "isError": True
}
```

### Accessing Results in Code

**With MCP Python SDK:**
```python
result = await client.call_tool("multiply", {"a": 3, "b": 4})
# result.content = [TextContent(text="12", type="text")]
# result.isError = False
```

**With FastMCP Client:**
```python
result = await client.call_tool("multiply", {"a": 3, "b": 4})
# Similar structure
```

---

## PART 15: Resources

### What Is a Resource?

**Resource** = Readable, structured data accessible via a URI

**Mental model:** "Give me information about this thing"

**Key characteristic:** Resources are **read-only**. They provide context to the model.

### Resource vs Tool

| Aspect | Tool | Resource |
|--------|------|----------|
| **Purpose** | Action | Information |
| **Side effects** | Usually yes | No |
| **Execution** | Invoked actively | Retrieved passively |
| **Schema** | Input schema required | URI-based addressing |
| **Example** | "Create issue" | "Show issue details" |
| **Mental model** | "Do this" | "Tell me this" |

### Resource Examples

| Resource | URI Pattern | Content |
|----------|-------------|---------|
| GitHub Pull Request | `github://pr/123` | PR details, comments, diff |
| Database Query | `database://employees/select-all` | Query results |
| File | `file:///path/to/file.txt` | File contents |
| Slack Conversation | `slack://channel/general` | Message history |
| Project Docs | `docs://api-reference` | API documentation |

---

## PART 16: Resource URIs

### URI Scheme

Resources are addressed by **URI** (Uniform Resource Identifier):

```
scheme://authority/path/to/resource

Examples:
  github://pull/123
  file:///home/user/document.txt
  database://employees
  docs://api/endpoints
```

### Static vs Dynamic Resources

**Static Resource:**
```
URI: github://user/john
Always refers to the same resource
```

**Dynamic/Template Resource:**
```
URI Template: github://pr/{id}
Client substitutes {id} with actual value
Example: github://pr/123
```

### Resource Discovery

Server declares what resources it exposes via **resource templates**:

```
Template: "github://pr/{id}"
Description: "GitHub pull request details"
Match pattern: /^github:\/\/pr\/\d+$/

When client requests github://pr/123:
  ├─ Does it match template pattern?
  ├─ Yes → Server handles it
  └─ Server fetches PR #123
```

---

## PART 17: Resource Discovery

### ListResourcesRequest

**Client asks:** "What resources are available?"

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "resources/list",
  "params": {}
}
```

### ListResourcesResponse

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "resources": [
      {
        "uri": "company://acme",
        "name": "ACME Company",
        "description": "ACME Corp information",
        "mimeType": "text/plain"
      },
      {
        "uri": "company://acme/employees/{id}",
        "name": "Employee",
        "description": "Information about an employee",
        "mimeType": "application/json"
      }
    ]
  }
}
```

### Resource Template Field

```
"uri": "company://acme/employees/{id}"
        ┌────────────────────────────┘
        │ {id} = placeholder
```

When server sees a request for `company://acme/employees/123`, it:
1. Checks if it matches a template
2. Extracts {id} = "123"
3. Fetches the resource
4. Returns content

### Discovery Flow

```
Client                          Server
  │                               │
  ├─ ResourcesListRequest ───────►│
  │                                │
  │◄─ ListResourcesResponse ──────┤
  │ [                              │
  │   {uri, name, description},    │
  │   {uri: template/{id}, ...},   │
  │   ...                          │
  │ ]                              │
  │                                │
  ├─ ReadResourceRequest ────────►│
  │ {uri: "company://acme"}        │
  │                                │
  │◄─ ReadResourceResponse ───────┤
  │ {uri, contents, mimeType}      │
  │                                │
```

---

## PART 18: Resource Result Anatomy

### ReadResourceRequest

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "resources/read",
  "params": {
    "uri": "company://acme"
  }
}
```

### ReadResourceResponse

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "contents": [
      {
        "uri": "company://acme",
        "mimeType": "text/plain",
        "text": "ACME Corporation\n...(content)..."
      }
    ]
  }
}
```

### Resource Content Structure

| Field | Type | Meaning |
|-------|------|---------|
| `uri` | string | The resource URI |
| `mimeType` | string | Content type (text/plain, application/json, etc.) |
| `text` | string | Text content (for text MIME types) |
| `blob` | bytes | Binary content (for binary MIME types) |

### Text Content Example

```json
{
  "uri": "github://pr/123",
  "mimeType": "text/markdown",
  "text": "# Pull Request #123\n\nDescription: Add new feature..."
}
```

### JSON Content Example

```json
{
  "uri": "database://employees/123",
  "mimeType": "application/json",
  "text": "{\"id\": 123, \"name\": \"John\", \"department\": \"Engineering\"}"
}
```

### Accessing Resources in Code

**Python SDK:**
```python
resource = await client.read_resource("company://acme")
# resource.contents = [ResourceContent(...)]
# resource.contents[0].uri = "company://acme"
# resource.contents[0].text = "ACME Corporation..."
# resource.contents[0].mimeType = "text/plain"
```

---

## PART 19: Prompts

### What Is an MCP Prompt?

**Prompt** = Reusable instruction template for the model

**Mental model:** "Here's how you should approach this task"

**Different from:**
- **System prompt** - Model's overall instructions
- **User input** - The actual task
- **MCP Prompt** - Reusable template for specific task patterns

### MCP Prompt Example

**Prompt Name:** `summarize_document`

**Arguments:** 
- `document_id` - Which document to summarize
- `summary_style` - "brief", "detailed", "executive"

**Returns:** Structured messages guiding the model

### Prompt vs Tool

| Aspect | Tool | Prompt |
|--------|------|--------|
| **Purpose** | Execute action | Guide reasoning |
| **Inputs** | Arguments | Context parameters |
| **Output** | Result | Instruction messages |
| **Side effects** | Usually yes | No |
| **Used by** | Model executes | Model reads |

### Prompt vs System Prompt

| Aspect | MCP Prompt | System Prompt |
|--------|-----------|---------------|
| **Scope** | Specific task template | Overall model behavior |
| **Discovery** | Server exposes prompts | Host defines system prompt |
| **Reusability** | Designed to be reused | Usually fixed |
| **Dynamic** | Can take arguments | Static |
| **Composition** | Can combine multiple | Single |

---

## PART 20: Prompt Discovery & Execution

### ListPromptsRequest

```json
{
  "jsonrpc": "2.0",
  "id": 6,
  "method": "prompts/list",
  "params": {}
}
```

### ListPromptsResponse

```json
{
  "jsonrpc": "2.0",
  "id": 6,
  "result": {
    "prompts": [
      {
        "name": "summarize_document",
        "description": "Summarize a document",
        "arguments": [
          {
            "name": "document_id",
            "description": "ID of document to summarize",
            "required": true
          },
          {
            "name": "style",
            "description": "Summary style (brief/detailed/executive)",
            "required": false
          }
        ]
      }
    ]
  }
}
```

### GetPromptRequest

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "prompts/get",
  "params": {
    "name": "summarize_document",
    "arguments": {
      "document_id": "doc-123",
      "style": "executive"
    }
  }
}
```

### GetPromptResponse

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "description": "Summarize document doc-123 in executive style",
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "Summarize this document in executive style:\n\n[document content here]\n\nProvide:\n1. Key findings\n2. Recommendations\n3. Next steps"
        }
      }
    ]
  }
}
```

### Prompt Message Structure

Each message in a prompt is an instruction to the model:

```json
{
  "role": "user",  // or "assistant"
  "content": {
    "type": "text",
    "text": "The actual instruction"
  }
}
```

### Prompt Execution Flow

```
Model/Agent
  ↓
"I want to summarize document 123"
  ↓
MCP Client.get_prompt("summarize_document", {document_id: "123"})
  ↓
MCP Server
  ↓
Prompt handler executes
  ↓
Returns: [UserMessage("Summarize...")]
  ↓
MCP Client receives
  ↓
Host application receives messages
  ↓
Model processes messages
  ↓
Model generates summary
```

---

## PART 21: Tools vs Resources vs Prompts Comparison

### Master Comparison Table

| Aspect | Tool | Resource | Prompt |
|--------|------|----------|--------|
| **Primary purpose** | Execute action | Provide data | Guide reasoning |
| **Initiation** | Model selects & calls | Model reads | Model receives |
| **Side effects** | Usually yes | Never | Never |
| **Required params** | Yes (arguments) | No (URI only) | No (optional args) |
| **Output type** | ToolResult | ResourceContent | Messages |
| **Discovery method** | ListTools | ListResources | ListPrompts |
| **Execution method** | CallTool | ReadResource | GetPrompt |
| **Callable by LLM** | Yes (tool use) | No (context) | No (instruction) |
| **Can fail** | Yes | Yes (gracefully) | No |
| **Caching** | Usually no | Can cache | Can cache |
| **Example** | create_issue() | github://pr/123 | summarize docs |
| **Mental model** | "Do this" | "Tell me this" | "Approach it this way" |

### When to Use Each

**Use Tool when:**
- You want the model to **take action**
- The operation has **side effects**
- You need **arguments and validation**
- You want the model to **decide when** to call it

**Use Resource when:**
- You want to provide **context/information**
- The data is **read-only**
- The data is **addressed by URI**
- You want the model to **reference** specific data

**Use Prompt when:**
- You want to provide **reusable instructions**
- The guidance applies to **multiple scenarios**
- You want to **guide model reasoning**
- The instructions are **parameterized**

---

## PART 22: Protocol vs Transport

### Critical Distinction

**Protocol** and **Transport** are completely different concepts.

| Aspect | Protocol | Transport |
|--------|----------|-----------|
| **Definition** | Rules for messages | How messages travel |
| **Level** | Semantic (what & why) | Physical (how) |
| **Independent** | Yes | Yes |
| **Changes affect** | Message structure | Connectivity |
| **MCP Protocol** | JSON-RPC 2.0 messages | Doesn't care |
| **Example** | "Tool request has name and arguments" | "Over HTTP or stdio" |

### Analogy

```
Protocol = The language and grammar (English, JSON structure)
Transport = The delivery method (postcard, email, phone call)

You can speak English via:
  ├── Postcard (written mail)
  ├── Email
  └── Phone call

The language doesn't change; the delivery method does.

Similarly, MCP Protocol stays the same over:
  ├── stdio (subprocess)
  ├── HTTP
  └── WebSocket (future)
```

### MCP Protocol Over Different Transports

```
MCP Client                    MCP Server
    │                              │
    ├── stdio ─────────────────────┤
    │ (process communication)       │
    │ (stdout/stdin)                │
    │                               │
    ├── HTTP ───────────────────────┤
    │ (network communication)        │
    │ (request/response)             │
    │                               │
    ├── WebSocket (future) ─────────┤
    │ (bidirectional streaming)      │
    │                               │
```

**All use the same MCP Protocol (JSON-RPC 2.0 messages)**

---

## PART 23: STDIO Transport

### What Is STDIO?

**STDIO** = Standard Input/Output stream

**Usage in MCP:** Local process-to-process communication

### STDIO Architecture

```
Host Application
    ↓
MCP Client
    ↓
[Spawn subprocess]
    ↓
MCP Server (subprocess)
    ├── stdin (reads requests)
    ├── stdout (writes responses)
    └── stderr (logs, not protocol)
```

### How STDIO Works

1. **Host starts MCP server as subprocess**
   ```python
   import subprocess
   
   process = subprocess.Popen(
       ["python", "my_mcp_server.py"],
       stdin=subprocess.PIPE,
       stdout=subprocess.PIPE,
       stderr=subprocess.PIPE
   )
   ```

2. **Client writes JSON to subprocess stdin**
   ```json
   {"jsonrpc": "2.0", "id": 1, "method": "initialize", ...}
   ```

3. **Server reads from stdin**
4. **Server processes request**
5. **Server writes JSON to stdout**
   ```json
   {"jsonrpc": "2.0", "id": 1, "result": {...}}
   ```

6. **Client reads from subprocess stdout**

### Subprocess Lifecycle

```
Host starts server
    ↓
Server initializes
    ↓
Server waits for requests on stdin
    ↓
Client connects, sends InitializeRequest
    ↓
Server responds
    ↓
Client sends ListToolsRequest
    ↓
Server responds with tools
    ↓
... (normal operation)
    ↓
Client closes connection
    ↓
Server reads EOF on stdin
    ↓
Server terminates gracefully
    ↓
Subprocess exits
```

### Key STDIO Rules

1. **One request per line**
   - Each message is newline-terminated JSON
   - Each message is complete and independent

2. **STDOUT is protocol only**
   - All MCP protocol messages go to stdout
   - Never mix logs or other output in stdout

3. **STDERR is safe for logging**
   - Server can write logs to stderr
   - Logs don't corrupt protocol communication
   - Client typically captures/ignores stderr

4. **Blocking I/O**
   - stdin/stdout are blocking streams
   - Server reads, processes, writes response
   - Then waits for next request

### Configuration (MCP 1.28.1)

In host configuration, STDIO server is specified:

```json
{
  "servers": {
    "knowledge": {
      "command": "python",
      "args": ["/path/to/my_mcp_server.py"]
    }
  }
}
```

This tells the host:
1. Run `python /path/to/my_mcp_server.py`
2. Connect via stdio
3. Use that as MCP server

---

## PART 24: Streamable HTTP Transport

### What Is Streamable HTTP?

**Streamable HTTP** = Network-based transport using HTTP and Server-Sent Events (SSE)

**Usage in MCP:** Network communication between remote client and server

### HTTP Architecture

```
Host Application (Client)          Network          MCP Server
        ↓                             ↑↓                 ↓
    MCP Client ──HTTP POST────────► /mcp/message ────────┤
        ↑                                              Process
        └────HTTP GET(SSE)◄───────── /mcp/stream ◄──────┘
                    (responses & notifications)
```

### Streamable HTTP Features

1. **POST /mcp/message** - Client sends requests
2. **GET /mcp/stream** - Server streams responses (SSE)
3. **Network-based** - Works over internet
4. **Asynchronous** - Requests and responses decoupled

### How Streamable HTTP Works

1. **Client connects to server URL**
   ```
   mcp_url = "https://mcp-server.example.com/mcp"
   ```

2. **Client opens SSE stream for responses**
   ```
   GET https://mcp-server.example.com/mcp/stream
   
   Server streams:
   data: {"jsonrpc": "2.0", "id": 1, "result": {...}}
   ```

3. **Client POSTs requests**
   ```
   POST https://mcp-server.example.com/mcp/message
   Body: {"jsonrpc": "2.0", "id": 1, "method": ...}
   ```

4. **Server processes and streams response**
   ```
   data: {"jsonrpc": "2.0", "id": 1, "result": {...}}
   ```

### Server-Sent Events (SSE) Format

```
data: {"jsonrpc": "2.0", "id": 1, "result": {...}}
\n
data: {"jsonrpc": "2.0", "method": "resources/updated"}
\n
```

Each message is prefixed with `data: ` and newline-terminated.

### Authentication

Streamable HTTP typically includes:

```
GET /mcp/stream
Authorization: Bearer <token>

POST /mcp/message
Authorization: Bearer <token>
Content-Type: application/json
```

### Use Cases

**Streamable HTTP:**
- Remote MCP servers
- Cloud-hosted servers
- Multi-user SaaS platforms
- Network segregation scenarios

**vs STDIO:**
- Local development
- Single-user applications
- Tightly coupled systems

### Configuration (MCP 1.28.1)

```json
{
  "servers": {
    "remote-knowledge": {
      "command": "http",
      "url": "https://api.example.com/mcp",
      "auth": {
        "type": "bearer",
        "token": "${KNOWLEDGE_API_KEY}"
      }
    }
  }
}
```

---

## PART 25: FastMCP

### What Is FastMCP?

**FastMCP** = Python framework/library for building MCP servers with minimal boilerplate

**MCP** = Protocol and specification
**FastMCP** = Framework implementing MCP

### Why FastMCP Exists

**Without FastMCP**, building an MCP server requires:
- Manual JSON-RPC message handling
- Schema generation from Python functions
- Request/response serialization
- Transport management
- Error handling boilerplate

**With FastMCP**:
- Decorators (`@mcp.tool()`, `@mcp.resource()`)
- Automatic schema generation from type hints
- Automatic serialization/deserialization
- Transport abstraction
- Less boilerplate

### FastMCP vs MCP SDK

| Aspect | MCP Python SDK | FastMCP |
|--------|---|---|
| **Level** | Lower-level protocol | Higher-level framework |
| **Boilerplate** | More | Less |
| **Decorators** | No | Yes |
| **Schema generation** | Manual | Automatic |
| **Learning curve** | Steeper | Gentler |
| **Control** | More fine-grained | Less |
| **Performance** | Same (FastMCP uses SDK) | Same |

### Installation

```bash
pip install fastmcp
```

Note: FastMCP is NOT currently available in PyPI by default. It's typically installed from source or via Anthropic's provided packages.

---

## PART 26: FastMCP Minimum Server

### Smallest Possible FastMCP Server

```python
from fastmcp import FastMCP

mcp = FastMCP("My First Server")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

if __name__ == "__main__":
    mcp.run()
```

### Breakdown

```python
from fastmcp import FastMCP
# Import FastMCP class for building servers

mcp = FastMCP("My First Server")
# Create server instance
# - "My First Server" = server name
# - Server not yet listening

@mcp.tool()
# Decorator registering function as MCP tool
# - Generates schema from function signature
# - Registers with server
# - Parameters become tool arguments
# - Return value becomes tool result

def add(a: int, b: int) -> int:
# Function definition
# - 'a' and 'b' are parameters (int type)
# - Return type is int
# - FastMCP uses these for schema generation

    """Add two numbers."""
# Docstring used as tool description

    return a + b
# Function logic

if __name__ == "__main__":
    mcp.run()
# Start server
# - Reads from stdin (STDIO transport)
# - Listens for MCP protocol messages
# - Processes requests
# - Writes responses to stdout
```

### What FastMCP Generates

**When client calls `list_tools()`, it receives:**

```json
{
  "tools": [
    {
      "name": "add",
      "description": "Add two numbers.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "a": {
            "type": "integer"
          },
          "b": {
            "type": "integer"
          }
        },
        "required": ["a", "b"]
      }
    }
  ]
}
```

---

## PART 27: @mcp.tool() Decorator Deep Dive

### Tool Decorator Parameters

```python
@mcp.tool(
    name="optional_name",                    # Auto-generated if omitted
    description="Tool description",          # Uses docstring if omitted
)
def my_tool(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b
```

### Automatic Schema Generation

**From function signature to schema:**

```python
@mcp.tool()
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b
```

**Schema generated:**

```json
{
  "name": "multiply",
  "description": "Multiply two numbers.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "a": {"type": "integer"},
      "b": {"type": "integer"}
    },
    "required": ["a", "b"]
  }
}
```

### Type Hints and Schema Types

FastMCP maps Python type hints to JSON Schema types:

| Python Type | JSON Schema Type |
|-------------|-----------------|
| `int` | `{"type": "integer"}` |
| `float` | `{"type": "number"}` |
| `str` | `{"type": "string"}` |
| `bool` | `{"type": "boolean"}` |
| `list` | `{"type": "array", "items": ...}` |
| `dict` | `{"type": "object"}` |
| `Optional[T]` | Same as T (not required) |

### Optional Parameters

```python
@mcp.tool()
def search(query: str, limit: int = 10) -> list:
    """Search the database.
    
    Args:
        query: Search terms
        limit: Maximum results
    """
    return perform_search(query, limit)
```

**Generated schema:**

```json
{
  "inputSchema": {
    "properties": {
      "query": {"type": "string"},
      "limit": {"type": "integer"}
    },
    "required": ["query"]
  }
}
```

**Key point:** `limit` has a default, so it's not required.

### Parameter Descriptions

FastMCP uses docstring parsing to extract parameter descriptions:

```python
@mcp.tool()
def search(query: str, limit: int = 10) -> list:
    """Search the database.
    
    Args:
        query: Search terms
        limit: Maximum results (default 10)
    """
    pass
```

**Generated:**

```json
{
  "properties": {
    "query": {
      "type": "string",
      "description": "Search terms"
    },
    "limit": {
      "type": "integer",
      "description": "Maximum results (default 10)"
    }
  }
}
```

### Return Type

Return type annotation controls output semantics:

```python
@mcp.tool()
def get_user(id: int) -> dict:
    """Get user details."""
    return {"id": 1, "name": "John"}
```

The return type (`dict`) tells FastMCP the output is a dictionary (will be JSON-serialized).

---

## PART 28: @mcp.resource() for Resources

### Basic Resource

```python
@mcp.resource()
def company_info() -> str:
    """Company information resource."""
    return "ACME Corporation...(content)"
```

**Exposes resource:** `resource://company_info`

### Parameterized Resource

```python
@mcp.resource("employee://{id}")
def get_employee(id: str) -> str:
    """Get employee information.
    
    Args:
        id: Employee ID
    """
    return fetch_employee(id)
```

**Exposes resource template:** `employee://{id}`

**Can be accessed as:** `employee://123`, `employee://john-doe`, etc.

### Resource Return Type

```python
@mcp.resource("document://{id}")
def get_document(id: str) -> str:
    """Fetch document."""
    return read_file(f"/docs/{id}.txt")
```

Must return:
- `str` for text content
- `bytes` for binary content
- Path to file (string pointing to file)

---

## PART 29: @mcp.prompt() for Prompts

### Basic Prompt

```python
@mcp.prompt()
def summarize() -> str:
    """Prompt to summarize content."""
    return """Summarize the following in 3 bullet points:
- Key findings
- Implications
- Next steps"""
```

**Exposes prompt:** `summarize`

### Parameterized Prompt

```python
@mcp.prompt()
def code_review(language: str) -> str:
    """Review code for issues.
    
    Args:
        language: Programming language (python/javascript/go)
    """
    return f"""Review this {language} code for:
1. Security issues
2. Performance problems
3. Best practice violations"""
```

### Prompt Return Format

```python
@mcp.prompt()
def analyze(query: str) -> dict:
    """Analysis prompt."""
    return {
        "system": "You are a data analyst",
        "messages": [
            {"role": "user", "content": f"Analyze: {query}"}
        ]
    }
```

Or return plain string (gets wrapped automatically).

---

## PART 30: FastMCP Context API

### What Is Context?

**Context** = Runtime information about the current request/session

**Provides access to:**
- Logging
- Progress reporting
- Request information
- Sampling (if supported)
- Other runtime state

### Accessing Context

```python
from fastmcp import Context

@mcp.tool()
async def my_tool(query: str, context: Context) -> str:
    """Tool that uses context."""
    context.log(f"Processing query: {query}")
    return process(query)
```

**Key point:** Make the parameter `context: Context` (type-annotated)

FastMCP automatically injects it; it's not part of the client request.

### Context Methods

| Method | Purpose |
|--------|---------|
| `log(message)` | Log a message |
| `set_progress(progress, total)` | Report progress |
| `set_status(status)` | Report status |

### Logging Example

```python
@mcp.tool()
async def search(query: str, context: Context) -> list:
    """Search database."""
    context.log(f"Starting search for: {query}")
    
    results = await database.search(query)
    context.log(f"Found {len(results)} results")
    
    return results
```

---

## PART 31: FastMCP Server Lifecycle

### Server Initialization

```python
mcp = FastMCP("Server Name")
```

Creates server instance but doesn't start it.

### Startup Sequence

```
1. Import FastMCP
    ↓
2. Create FastMCP instance
    ↓
3. Register tools with @mcp.tool()
    ↓
4. Register resources with @mcp.resource()
    ↓
5. Register prompts with @mcp.prompt()
    ↓
6. Call mcp.run()
    ↓
7. Server starts listening on stdin
    ↓
8. Server receives InitializeRequest
    ↓
9. Server responds with capabilities
    ↓
10. Server waits for client requests
    ↓
11. Client sends ListToolsRequest
    ↓
12. Server responds with tool definitions
    ↓
13. Client sends CallToolRequest
    ↓
14. Server executes tool
    ↓
15. Server responds with result
    ↓
... (repeat 13-15)
    ↓
16. Connection closes
    ↓
17. Server terminates
```

---

## PART 32: MCP Inspector

### What Is MCP Inspector?

**MCP Inspector** = Developer tool for testing and debugging MCP servers

**Purpose:**
- Connect to MCP server
- Inspect capabilities
- List tools, resources, prompts
- Test tool execution
- View requests and responses
- Debug issues

### Why MCP Inspector?

Without inspector:
- Manual protocol message creation
- Hard to test without client
- Can't see protocol details easily

With inspector:
- Visual UI for testing
- See all messages
- Inspect schemas
- Click-to-execute tools
- Understand server state

### Installation & Usage

```bash
# Check if inspector is available
mcp inspector

# Or via Python
python -m mcp.inspector
```

### Inspector Workflow

1. **Start your MCP server**
   ```bash
   python my_mcp_server.py
   ```

2. **Start Inspector**
   ```bash
   mcp inspector
   ```

3. **Configure server connection**
   - Choose stdio or HTTP
   - For stdio: point to server process
   - For HTTP: enter server URL

4. **Click "Initialize"**
   - Sends InitializeRequest
   - Shows response

5. **View capabilities**
   - Shows what server exposes

6. **List tools**
   - Shows all available tools with schemas

7. **Execute tool**
   - Fill in arguments
   - Click execute
   - See result and timing

8. **View requests/responses**
   - Inspect raw JSON
   - See protocol details
   - Debug issues

---

## PART 33: Testing MCP Servers

### Why Inspector Isn't Enough

Inspector is good for:
- Manual testing
- One-off checks
- Understanding behavior

Inspector is bad for:
- Regression testing
- CI/CD automation
- Comprehensive coverage
- Performance testing

### Testing Strategy

**Three levels:**

1. **Unit tests** - Test Python functions directly
2. **Integration tests** - Test tool execution
3. **Protocol integration tests** - Test full MCP flow

### Unit Test Example

```python
def test_multiply():
    from my_server import multiply
    result = multiply(3, 4)
    assert result == 12
```

### Integration Test Example

```python
import pytest
from fastmcp import FastMCP
import asyncio

mcp = FastMCP("test-server")

@mcp.tool()
def add(a: int, b: int) -> int:
    return a + b

@pytest.mark.asyncio
async def test_add_via_mcp():
    from mcp.client import ClientSessionGroup
    
    async with ClientSessionGroup() as client:
        # Test tool discovery
        tools = await client.list_tools()
        assert any(t.name == "add" for t in tools)
        
        # Test tool execution
        result = await client.call_tool("add", {"a": 3, "b": 4})
        assert "7" in str(result.content[0].text)
```

### Test Categories

| Category | Tests | Examples |
|----------|-------|----------|
| **Valid inputs** | Tool works correctly | multiply(2, 3) == 6 |
| **Invalid inputs** | Type validation | multiply("a", "b") → error |
| **Missing inputs** | Required params | multiply(a=2) → error |
| **Edge cases** | Boundaries | multiply(0, 999) |
| **External failures** | API errors | Database query timeout |
| **Resource discovery** | Resource templates | Resource URIs match patterns |
| **Prompt retrieval** | Prompt arguments | Arguments substitute correctly |

---

## PART 34: MCP Client Building Blocks

### MCP Client Role

**MCP Client** = Communication abstraction

Handles:
- Connection to server
- Request/response serialization
- Capability discovery
- Tool execution
- Error handling

### Client Operations

| Operation | Request | Response | Purpose |
|-----------|---------|----------|---------|
| `initialize()` | InitializeRequest | InitializeResponse | Establish session |
| `list_tools()` | ListToolsRequest | ListToolsResponse | Discover tools |
| `call_tool()` | CallToolRequest | CallToolResponse | Execute tool |
| `list_resources()` | ListResourcesRequest | ListResourcesResponse | Discover resources |
| `read_resource()` | ReadResourceRequest | ReadResourceResponse | Fetch resource |
| `list_prompts()` | ListPromptsRequest | ListPromptsResponse | Discover prompts |
| `get_prompt()` | GetPromptRequest | GetPromptResponse | Fetch prompt |

---

## PART 35: Building a Basic MCP Client (MCP SDK)

### Minimum Python Client

```python
import asyncio
from mcp.client.stdio import StdioClientTransport
from mcp.client import ClientSession

async def main():
    # Create transport (stdio)
    transport = StdioClientTransport(
        "python",           # Command
        "/path/to/server.py"  # Script
    )
    
    # Create client session
    async with ClientSession(transport) as session:
        
        # 1. Initialize
        await session.initialize()
        print("✓ Connected")
        
        # 2. List tools
        tools = await session.list_tools()
        for tool in tools:
            print(f"- {tool.name}: {tool.description}")
        
        # 3. Call tool
        result = await session.call_tool("add", {"a": 3, "b": 4})
        print(f"Result: {result}")
        
        # 4. List resources
        resources = await session.list_resources()
        for resource in resources:
            print(f"- {resource.uri}")
        
        # 5. Read resource
        content = await session.read_resource("company://acme")
        print(f"Content: {content.contents[0].text}")

if __name__ == "__main__":
    asyncio.run(main())
```

### Breakdown

```python
from mcp.client.stdio import StdioClientTransport
# Import for subprocess-based servers

from mcp.client import ClientSession
# Main client interface

async with ClientSession(transport) as session:
# Context manager: connects, initializes, cleans up

await session.initialize()
# Send InitializeRequest, receive capabilities

tools = await session.list_tools()
# Send ListToolsRequest, get [Tool, ...]

result = await session.call_tool("tool_name", {"arg": value})
# Send CallToolRequest, get ToolResult

resources = await session.list_resources()
# Send ListResourcesRequest, get [Resource, ...]

content = await session.read_resource("uri")
# Send ReadResourceRequest, get ResourceContent
```

### Client Return Objects

#### list_tools() returns

```python
[
  Tool(
    name="add",
    description="Add two numbers",
    inputSchema={...}
  ),
  ...
]
```

#### call_tool() returns

```python
ToolResult(
  content=[
    TextContent(
      type="text",
      text="12"
    )
  ],
  isError=False
)
```

#### list_resources() returns

```python
[
  Resource(
    uri="company://acme",
    name="ACME",
    description="ACME Corp",
    mimeType="text/plain"
  ),
  ...
]
```

#### read_resource() returns

```python
ResourceContents(
  contents=[
    ResourceContent(
      uri="company://acme",
      mimeType="text/plain",
      text="ACME Corporation..."
    )
  ]
)
```

---

## PART 36: MCP Client Output Anatomy

### list_tools() Output Structure

```python
result = await client.list_tools()

# result is List[Tool]
for tool in result:
    # tool.name: str = "multiply"
    # tool.description: str = "Multiply two numbers"
    # tool.inputSchema: dict = {
    #   "type": "object",
    #   "properties": {
    #     "a": {"type": "integer"},
    #     "b": {"type": "integer"}
    #   },
    #   "required": ["a", "b"]
    # }
```

### call_tool() Output Structure

```python
result = await client.call_tool("multiply", {"a": 3, "b": 4})

# result.content: List[ToolResultContent]
#   ├─ type: "text", "image", or "resource"
#   ├─ text: str (for text content)
#   ├─ data: bytes (for image)
#   └─ uri: str (for resource reference)

# result.isError: bool = False

# Access:
for content in result.content:
    if content.type == "text":
        print(content.text)
```

### list_resources() Output Structure

```python
result = await client.list_resources()

# result is List[Resource]
for resource in result:
    # resource.uri: str = "company://acme"
    # resource.name: str = "ACME"
    # resource.description: str = "ACME information"
    # resource.mimeType: str = "text/plain"
```

### read_resource() Output Structure

```python
result = await client.read_resource("company://acme")

# result.contents: List[ResourceContent]
for content in result.contents:
    # content.uri: str
    # content.mimeType: str
    # content.text: str (for text content)
    # content.blob: bytes (for binary content)
```

### list_prompts() Output Structure

```python
result = await client.list_prompts()

# result is List[Prompt]
for prompt in result:
    # prompt.name: str
    # prompt.description: str
    # prompt.arguments: List[PromptArgument]
    for arg in prompt.arguments:
        # arg.name: str
        # arg.description: str
        # arg.required: bool
```

### get_prompt() Output Structure

```python
result = await client.get_prompt("summarize_document", {
    "document_id": "123",
    "style": "executive"
})

# result.description: str
# result.messages: List[Message]
for msg in result.messages:
    # msg.role: str = "user" or "assistant"
    # msg.content: MessageContent
    #   ├─ type: "text", "image", etc.
    #   ├─ text: str (for text)
    #   └─ ...
```

---

## PART 37: MCP Request/Response Reference

### Complete Reference Table

| Operation | Request Method | Request Params | Response Fields | Use Case |
|-----------|---|---|---|---|
| **Initialize** | initialize | protocolVersion, clientInfo, capabilities | protocolVersion, serverInfo, capabilities | Begin session |
| **List Tools** | tools/list | (none) | tools: [Tool] | Discover capabilities |
| **Call Tool** | tools/call | name, arguments | content: [Content], isError: bool | Execute action |
| **List Resources** | resources/list | (none) | resources: [Resource] | Discover data |
| **Read Resource** | resources/read | uri | contents: [Content] | Fetch data |
| **List Prompts** | prompts/list | (none) | prompts: [Prompt] | Discover templates |
| **Get Prompt** | prompts/get | name, arguments | description, messages: [Message] | Fetch template |

### Notification Table

| Notification | From | Purpose |
|---|---|---|
| **resources/list_changed** | Server → Client | Resources list was updated |
| **resources/updated** | Server → Client | Specific resource was updated |
| **logging/message** | Server → Client | Server logged a message |

---

## PART 38: Error Handling

### Protocol Error Types

| Layer | Error Type | Example | Handling |
|-------|---|---|---|
| **JSON-RPC** | Parse error | Malformed JSON | Connection reset |
| **Protocol** | Method not found | Unknown method name | Return error response |
| **Protocol** | Invalid params | Wrong argument types | Return error response |
| **Tool** | Execution error | Tool throws exception | ToolResult isError=True |
| **Resource** | Not found | URI doesn't exist | Graceful not found |
| **Transport** | Connection error | Network failure | Retry/fail |
| **Auth** | Unauthorized | No valid credentials | Return 401/403 |

### Tool Execution Error Example

**Client sends:**
```json
{
  "method": "tools/call",
  "params": {
    "name": "divide",
    "arguments": {
      "a": 10,
      "b": 0
    }
  }
}
```

**Server executes, gets error:**
```
ZeroDivisionError: division by zero
```

**Server responds:**
```json
{
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Error: division by zero"
      }
    ],
    "isError": true
  }
}
```

### Input Validation Error

**Client sends:**
```json
{
  "method": "tools/call",
  "params": {
    "name": "multiply",
    "arguments": {
      "a": "not a number",
      "b": 5
    }
  }
}
```

**Server validates against schema, finds type mismatch:**

**Server responds:**
```json
{
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": {
      "description": "Argument 'a' must be an integer, got string"
    }
  }
}
```

---

## PART 39: Common MCP Mistakes

### Mistake 1: Confusing Host and Client

**Wrong:**
```python
class MyAgent:
    def __init__(self):
        self = MCP Client  # ❌ Wrong mental model
```

**Right:**
```python
class MyAgent:  # ← This is the Host
    def __init__(self):
        self.mcp_client = ClientSession()  # ← This is the Client
```

### Mistake 2: LLM Calling Server Directly

**Wrong:**
```
LLM → MCP Server directly
```

LLMs cannot initiate network connections. Only the host can.

**Right:**
```
LLM → Host (via function call)
Host → MCP Client
MCP Client → MCP Server
```

### Mistake 3: Treating Resource as Tool

**Wrong:**
```python
# Treating read-only data as an action
resource = await client.call_tool("get_document", ...)
```

**Right:**
```python
# Resources are read-only
resource = await client.read_resource("document://123")
```

### Mistake 4: Confusing Protocol and Transport

**Wrong:**
```python
# Thinking HTTP is the protocol
"We're using HTTP, so MCP must support it"
```

**Right:**
```
MCP Protocol (JSON-RPC) is independent of transport.
MCP can use: stdio, HTTP, WebSocket, etc.
```

### Mistake 5: Stdout Logging Corruption

**Wrong server:**
```python
@mcp.tool()
def my_tool():
    print("Debug message")  # ❌ Corrupts stdout!
    return result
```

**Right server:**
```python
@mcp.tool()
def my_tool(context: Context):
    context.log("Debug message")  # ✓ Goes to stderr
    return result
```

---

## PART 40: Security Checklist

```
[ ] Authentication
  [ ] Server validates client identity
  [ ] API keys/tokens required
  [ ] TLS/HTTPS enabled for remote

[ ] Authorization
  [ ] Tools have appropriate access controls
  [ ] Resources restricted by user/role
  [ ] Prompts don't reveal sensitive info

[ ] Input Validation
  [ ] Tool arguments validated against schema
  [ ] Type checking enforced
  [ ] String length limits applied
  [ ] Array size limits applied

[ ] Output Validation
  [ ] Tool results don't leak secrets
  [ ] No PII in responses
  [ ] Resources validated before return

[ ] Prompt Injection Defense
  [ ] Resource content not treated as instructions
  [ ] User input sanitized
  [ ] Prompt arguments escaped

[ ] Secret Management
  [ ] No secrets in tool descriptions
  [ ] API keys in environment variables
  [ ] Credentials never logged
  [ ] Secrets rotation capability

[ ] Audit Logging
  [ ] All tool calls logged
  [ ] Tool execution results logged (no secrets)
  [ ] Authorization failures logged
  [ ] Errors logged for debugging

[ ] Transport Security
  [ ] TLS for HTTP transport
  [ ] Subprocess token validation
  [ ] Message signing where appropriate

[ ] Isolation
  [ ] Servers isolated from each other
  [ ] Tool execution sandboxed where possible
  [ ] Resource access controlled
```

---

## PART 41: Production Checklist

```
[ ] SETUP
  [ ] Python version verified (3.13.7)
  [ ] MCP SDK version pinned (1.28.1)
  [ ] FastMCP version pinned
  [ ] Dependencies locked

[ ] TRANSPORT
  [ ] Stdio or HTTP chosen
  [ ] Subprocess limits set (timeouts)
  [ ] Connection pooling configured
  [ ] Retry logic implemented

[ ] TOOLS
  [ ] All tools have descriptions
  [ ] Input schema complete
  [ ] Type hints on all parameters
  [ ] Error handling present
  [ ] Timeouts implemented

[ ] RESOURCES
  [ ] URIs documented
  [ ] MIME types correct
  [ ] Access control verified
  [ ] Caching strategy defined

[ ] PROMPTS
  [ ] Prompts tested
  [ ] Arguments validated
  [ ] No secrets in templates

[ ] OBSERVABILITY
  [ ] Logging configured (stderr)
  [ ] Metrics collected
  [ ] Distributed tracing setup
  [ ] Health checks implemented

[ ] TESTING
  [ ] Unit tests for all tools
  [ ] Integration tests for MCP flow
  [ ] Error cases tested
  [ ] Performance tested (p95, p99)

[ ] DEPLOYMENT
  [ ] Docker image created
  [ ] Environment variables documented
  [ ] Secrets management setup
  [ ] Rollback plan exists
  [ ] Versioning strategy defined

[ ] SECURITY
  [ ] Authentication enabled
  [ ] Authorization enforced
  [ ] Input validation complete
  [ ] Secrets never logged
  [ ] TLS for remote

[ ] MONITORING
  [ ] Error rates tracked
  [ ] Latency monitored
  [ ] Resource usage tracked
  [ ] Alerts configured
```

---

## PART 42: Key Concepts Summary

### MCP Primitives

```
Tool = "Do something"
  └─ Has side effects
  └─ Invoked actively
  └─ Example: create_issue()

Resource = "Tell me something"
  └─ Read-only
  └─ Addressed by URI
  └─ Example: github://pull/123

Prompt = "Here's how to approach this"
  └─ Instruction template
  └─ Parameterized
  └─ Example: summarize_document(doc_id, style)
```

### Protocol vs Transport

```
Protocol = MCP (JSON-RPC 2.0)
  └─ Same regardless of transport

Transport = How protocol messages move
  ├─ Stdio (local subprocess)
  └─ HTTP (remote network)
```

### Architecture

```
Host
  ├─ MCP Client (part of host)
  └─ Your reasoning loop

MCP Server
  ├─ Tools
  ├─ Resources
  └─ Prompts

External Systems
  ├─ APIs
  ├─ Databases
  └─ Services
```

---

## PART 43: FastMCP Quick Reference

### Creating a Server

```python
from fastmcp import FastMCP

mcp = FastMCP("Server Name")

@mcp.tool()
def tool_name(param: type) -> type:
    """Description"""
    return result

if __name__ == "__main__":
    mcp.run()
```

### Tool Anatomy

```python
@mcp.tool()
def my_tool(
    required_param: str,
    optional_param: int = 10
) -> dict:
    """One-line description.
    
    Args:
        required_param: Description
        optional_param: Description (default 10)
    
    Returns:
        Description of return value
    """
    return {"result": "value"}
```

### Testing

```python
import asyncio
from mcp.client.stdio import StdioClientTransport
from mcp.client import ClientSession

async def test():
    transport = StdioClientTransport("python", "server.py")
    async with ClientSession(transport) as session:
        await session.initialize()
        result = await session.call_tool("tool_name", {})
        print(result)

asyncio.run(test())
```

---

## Rapid-Fire Interview Questions

**Q: What is MCP?**  
A: Model Context Protocol—a standardized protocol for communication between AI applications and external systems (servers), enabling tools, resources, and prompts discovery and execution.

**Q: Why was MCP created?**  
A: To solve the n×m integration problem: instead of each AI app implementing custom integrations with each external system, MCP provides a single standard protocol.

**Q: What's the difference between Protocol and Transport?**  
A: Protocol = rules for messages (JSON-RPC 2.0). Transport = how messages move (stdio, HTTP, etc.). Protocol is independent of transport.

**Q: What are the three core MCP primitives?**  
A: Tools (actions), Resources (data), and Prompts (instruction templates).

**Q: How is Tool different from Resource?**  
A: Tools have side effects and are actively invoked; Resources are read-only and addressed by URI.

**Q: Does the LLM directly call MCP tools?**  
A: No. The LLM decides to use a tool; the Host/MCP Client handles the communication with the server.

**Q: What does the MCP Client do?**  
A: Communicates with MCP servers—handles connections, serialization, tool execution, resource fetching, etc.

**Q: Is Host the same as MCP Client?**  
A: No. Host = the application (contains LLM/agent logic). MCP Client = the communication layer (part of the host).

**Q: What is FastMCP?**  
A: A Python framework for building MCP servers with decorators (`@mcp.tool()`) and automatic schema generation from type hints.

**Q: What does @mcp.tool() do?**  
A: Registers a Python function as an MCP tool and generates JSON Schema from the function signature and docstring.

---

## End of Part 1 Document

This document covers:
✓ Why MCP exists
✓ What MCP is
✓ Architecture (Host, Client, Server)
✓ Protocol fundamentals
✓ JSON-RPC structure
✓ MCP lifecycle
✓ Capabilities
✓ Tools (discovery, schema, execution)
✓ Resources
✓ Prompts
✓ Transport (stdio, HTTP)
✓ FastMCP basics
✓ Testing & debugging
✓ Security & production

**Next document:** Production evaluation, interviews, and system design scenarios.
