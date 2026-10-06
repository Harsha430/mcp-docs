# 04: MCP Python SDK v2

**Version/Source Basis:** Official MCP Python SDK v2 available as of **July 28, 2026**. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/) and [https://github.com/modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk)

## What Is the Official MCP Python SDK v2?

### Simple Explanation

The **official MCP Python SDK v2** is a Python library (`mcp` package) that lets you:
- **Write MCP servers** (expose capabilities)
- **Write MCP clients** (discover and use capabilities)
- Handle all protocol complexity for you

**Without the SDK:** You'd manually format JSON-RPC messages, handle transports, manage connections.

**With the SDK:** You write Python code; the SDK handles MCP protocol details.

### Real-World Analogy

Think of building a house:
- **Without SDK:** You'd mine ore, smelt metal, cut wood, mill lumber, etc.
- **With SDK:** You buy materials from a store and focus on building

## Installation

### Basic Installation

```bash
pip install mcp
```

### With CLI Tools

```bash
pip install "mcp[cli]"
```

This adds the `mcp` command-line tool for testing and debugging.

### With Development Tools

```bash
pip install "mcp[dev]"
```

Includes testing utilities, linting tools, type checking.

### Verify Installation

```bash
python -c "import mcp; print(mcp.__version__)"
```

Expected output: Version info like `0.1.0` or similar.

## SDK Structure

### Main Modules

```
mcp/
├── server/          # For writing servers
│   ├── Server       # Base server class
│   ├── Tool         # Tool definition
│   ├── Resource     # Resource definition
│   └── Prompt       # Prompt definition
├── client/          # For writing clients
│   └── Client       # Client implementation
├── types/           # Protocol types
│   ├── Tool         # Tool type definitions
│   ├── Resource     # Resource type definitions
│   └── TextContent  # Content types
└── transport/       # Transport implementations
    ├── StdioTransport
    ├── HttpTransport
    └── WebSocketTransport
```

### Key Imports

```python
# Server
from mcp.server import Server, Tool, Resource, Prompt
from mcp.types import ToolDefinition, ResourceDefinition

# Client
from mcp.client import Client

# Types
from mcp.types import TextContent, ImageContent

# Transport
from mcp.transport import StdioTransport, HttpTransport

# Decorators (simplified API)
from mcp.server import tool, resource, prompt
```

## SDK Patterns: Server and Client

### Pattern Overview

**Server-side:**
1. Create a server instance
2. Define tools, resources, prompts
3. Implement handlers
4. Run the server

**Client-side:**
1. Create a client instance
2. Connect to server
3. Discover capabilities
4. Invoke tools/resources/prompts

## Example Domain

For all SDK examples, we use:
- **Server:** Customer service server
- **Tools:** `get_customer`, `update_customer`
- **Resources:** Customer documents
- **Prompts:** Analysis templates

## Python SDK v2 vs SDK v1

### Key Differences (v1 → v2)

| Aspect | SDK v1 | SDK v2 |
|--------|--------|--------|
| **Installation** | `pip install mcp-server` | `pip install mcp` |
| **Base Class** | Custom server class | `Server` class |
| **Decorators** | Custom/inconsistent | `@tool`, `@resource`, `@prompt` |
| **Transport** | Manual setup | `StdioTransport`, `HttpTransport` |
| **Type Hints** | Partial | Full typing support |
| **Async/Sync** | Limited async | Full async/await support |

### Why This Documentation Uses SDK v2

- **Current Standard:** v2 is the current recommended approach
- **Better DX:** Cleaner decorators and patterns
- **Better Types:** Full type hint support
- **Active Development:** v2 receives updates; v1 is maintained only for compatibility

## Common Misconception: FastMCP

**⚠️ Important:** FastMCP is NOT part of the official SDK v2. See [12_FastMCP_vs_Official_SDK.md](12_FastMCP_vs_Official_SDK.md) for detailed comparison.

**In this document:** We focus on the official `mcp` package SDK v2.

## SDK Lifecycle Workflow

### Server Lifecycle

```
1. Create Server instance
   ↓
2. Define tools/resources/prompts (decoration)
   ↓
3. Configure transport (stdio, HTTP, etc.)
   ↓
4. Run/serve (blocking call)
   ↓
5. Handle incoming requests
   ↓
6. Send responses
```

### Client Lifecycle

```
1. Create Client instance
   ↓
2. Connect to server (establish transport)
   ↓
3. Initialize (handshake)
   ↓
4. Discover capabilities
   ↓
5. Invoke tools/read resources/get prompts
   ↓
6. Disconnect
```

## Minimal Server Example: Pseudo-code Overview

```python
from mcp.server import Server, tool
from mcp.transport import StdioTransport

# 1. Create server
server = Server("customer-server")

# 2. Define a tool
@server.tool()
def get_customer(customer_id: str) -> str:
    """Get customer information"""
    return f"Customer {customer_id}: Active"

# 3. Run
if __name__ == "__main__":
    transport = StdioTransport()
    server.run(transport)
```

**What this does:**
- Creates a server named "customer-server"
- Defines one tool: `get_customer`
- Runs over stdio (standard input/output)
- Listens for client requests

## Minimal Client Example: Pseudo-code Overview

```python
from mcp.client import Client
from mcp.transport import StdioTransport
import subprocess

# 1. Start server process
server_process = subprocess.Popen(["python", "server.py"])

# 2. Create transport
transport = StdioTransport(server_process)

# 3. Create client
client = Client(transport)

# 4. Initialize
await client.initialize()

# 5. List tools
tools = await client.list_tools()
print(f"Available tools: {[t.name for t in tools]}")

# 6. Call tool
result = await client.call_tool("get_customer", {"customer_id": "123"})
print(result)

# 7. Disconnect
await client.close()
```

**What this does:**
- Starts the server as a subprocess
- Creates stdio transport to communicate with it
- Discovers available tools
- Calls a tool
- Closes the connection

## Type System in SDK v2

The SDK provides strong typing:

```python
from mcp.types import Tool, ToolDefinition, TextContent

# Tool definition with input schema
tool_def = ToolDefinition(
    name="transfer_money",
    description="Transfer money between accounts",
    inputSchema={
        "type": "object",
        "properties": {
            "from_account": {"type": "string"},
            "to_account": {"type": "string"},
            "amount": {"type": "number"}
        },
        "required": ["from_account", "to_account", "amount"]
    }
)

# Text content
content = TextContent(
    type="text",
    text="Transfer completed successfully"
)
```

## Async/Await Pattern

SDK v2 is built on `asyncio`. All I/O operations are async:

```python
import asyncio
from mcp.client import Client

async def main():
    # All operations are async
    client = Client(transport)
    await client.initialize()
    tools = await client.list_tools()
    result = await client.call_tool("get_customer", {"customer_id": "123"})
    await client.close()

# Run the async function
asyncio.run(main())
```

**Why async?**
- Non-blocking I/O (messages can take time)
- Can handle multiple concurrent requests
- Better resource utilization

## Key Classes and Methods

### Server Class

```python
class Server:
    def __init__(self, name: str, version: str = "1.0.0"):
        """Create a server instance"""
        pass
    
    async def run(self, transport):
        """Start the server with a transport"""
        pass
    
    @tool()
    def my_tool(self, arg: str) -> str:
        """Define a tool"""
        return result
    
    @resource()
    def my_resource(self) -> str:
        """Define a resource"""
        return resource_data
    
    @prompt()
    def my_prompt(self, arg: str) -> str:
        """Define a prompt"""
        return prompt_text
```

### Client Class

```python
class Client:
    def __init__(self, transport):
        """Create a client instance"""
        pass
    
    async def initialize(self):
        """Initialize connection with server"""
        pass
    
    async def list_tools(self) -> List[Tool]:
        """List available tools"""
        pass
    
    async def call_tool(self, name: str, arguments: dict) -> str:
        """Call a tool"""
        pass
    
    async def list_resources(self) -> List[Resource]:
        """List available resources"""
        pass
    
    async def read_resource(self, uri: str) -> str:
        """Read a resource"""
        pass
    
    async def list_prompts(self) -> List[Prompt]:
        """List available prompts"""
        pass
    
    async def get_prompt(self, name: str, arguments: dict) -> str:
        """Get a prompt"""
        pass
    
    async def close(self):
        """Close connection"""
        pass
```

### Transport Classes

```python
from mcp.transport import StdioTransport, HttpTransport

# Stdio (for local process communication)
transport = StdioTransport(process)

# HTTP (for remote servers)
transport = HttpTransport(base_url="http://localhost:8000")
```

## Configuration and Environment

### Common Configuration Patterns

```python
import os
from mcp.server import Server

# Read from environment
DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///default.db")
AUTH_TOKEN = os.getenv("AUTH_TOKEN")

server = Server("my-server")

# Use configuration in tools
@server.tool()
def get_data(query: str) -> str:
    # Connect using DATABASE_URL
    conn = connect(DATABASE_URL)
    result = conn.execute(query)
    return result
```

### .env Files

```bash
# .env
DATABASE_URL=postgresql://user:pass@localhost/db
AUTH_TOKEN=secret_token_here
DEBUG=true
```

Load with:

```python
from dotenv import load_dotenv
import os

load_dotenv()
database_url = os.getenv("DATABASE_URL")
```

## Error Handling in SDK v2

### Tool Errors

```python
from mcp.types import Error

@server.tool()
def get_customer(customer_id: str):
    if not customer_id:
        raise ValueError("customer_id is required")
    
    result = find_customer(customer_id)
    if not result:
        raise ValueError(f"Customer {customer_id} not found")
    
    return result
```

The SDK automatically converts exceptions to MCP error responses.

### Connection Errors

```python
try:
    await client.initialize()
except ConnectionError as e:
    print(f"Failed to connect: {e}")
except Exception as e:
    print(f"Unexpected error: {e}")
```

## Project Structure

### Typical Server Project

```
my_mcp_server/
├── server.py           # Main server implementation
├── tools.py            # Tool implementations
├── requirements.txt    # Dependencies
├── .env               # Configuration
└── README.md          # Documentation
```

### Typical Client Project

```
my_mcp_client/
├── client.py          # Client implementation
├── main.py            # Application logic
├── requirements.txt   # Dependencies
└── README.md          # Documentation
```

## Testing with SDK v2

### Simple Server Test

```python
import pytest
from mcp.server import Server

@pytest.mark.asyncio
async def test_get_customer():
    server = Server("test-server")
    
    @server.tool()
    def get_customer(customer_id: str):
        return f"Customer {customer_id}"
    
    # Test directly
    result = await server.call_tool("get_customer", {"customer_id": "123"})
    assert "123" in result
```

## Interview Summary

**Key SDK v2 Concepts:**
- **Official Package:** `mcp` (installed via `pip install mcp`)
- **Base Classes:** `Server`, `Client`, `Tool`, `Resource`, `Prompt`
- **Decorators:** `@server.tool()`, `@server.resource()`, `@server.prompt()`
- **Transport:** `StdioTransport`, `HttpTransport`, etc.
- **Async:** All I/O is async/await
- **Type Hints:** Full typing support

**One-Liner:**
"The official MCP Python SDK v2 (`mcp` package) provides `Server` and `Client` classes with decorators for tools, resources, and prompts, handling JSON-RPC protocol and transport layers."

## Common Interview Questions

**Q1: What's the difference between SDK v1 and v2?**
A: v2 uses cleaner decorators, better typing, full async support, and is the current recommended approach. v1 is older and has less clean APIs.

**Q2: How do you install the official MCP SDK?**
A: `pip install mcp` (basic) or `pip install "mcp[cli]"` (with CLI tools).

**Q3: What's the relationship between the SDK and the protocol?**
A: The SDK is a Python implementation of the protocol. It handles JSON-RPC messages, transport, and connection management so you write Python instead of working with raw protocol messages.

**Q4: Why is everything async in SDK v2?**
A: Async allows non-blocking I/O, which is crucial for handling multiple concurrent requests efficiently.

**Q5: Can you use SDK v2 for both servers and clients?**
A: Yes. The SDK provides both `Server` class for building servers and `Client` class for building clients.

---

**Next:** [05_MCP_Server.md](05_MCP_Server.md) – Building MCP servers with tools, resources, prompts
