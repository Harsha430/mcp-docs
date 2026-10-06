# 06: Building MCP Clients with Python SDK v2

**Version/Source Basis:** Official MCP Python SDK v2, July 28, 2026. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

## What Is an MCP Client?

### Simple Explanation

An **MCP client** is software that:
1. Connects to an MCP server
2. Discovers what tools/resources/prompts the server has
3. Invokes tools and reads resources
4. Processes results

**You build clients** when you want to use capabilities from MCP servers.

### Real-World Analogy

A customer in a restaurant is a client:
1. Sees the menu (discovery)
2. Decides what to order (tool invocation)
3. Receives the meal (result)

## Simple Client: Calling a Tool

### Complete Code

```python
import asyncio
import subprocess
from mcp.client import Client
from mcp.transport import StdioTransport

async def main():
    # Step 1: Start the server as a subprocess
    # Assuming server.py exists and implements the customer service
    server_process = subprocess.Popen(
        ["python", "server.py"],
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE
    )
    
    # Step 2: Create stdio transport
    transport = StdioTransport(server_process)
    
    # Step 3: Create client
    client = Client(transport)
    
    try:
        # Step 4: Initialize (handshake)
        print("Initializing...")
        await client.initialize()
        print("Connected!")
        
        # Step 5: List available tools
        print("\nDiscovering tools...")
        tools = await client.list_tools()
        for tool in tools:
            print(f"  - {tool.name}: {tool.description}")
        
        # Step 6: Call a tool
        print("\nCalling get_customer...")
        result = await client.call_tool("get_customer", {"customer_id": "12345"})
        print(f"Result: {result}")
        
    finally:
        # Step 7: Close connection
        await client.close()
        server_process.terminate()

if __name__ == "__main__":
    asyncio.run(main())
```

### Line-by-Line Explanation

```python
server_process = subprocess.Popen(
    ["python", "server.py"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE
)
```
Starts the server as a subprocess. This makes it a child process of the client, allowing stdio communication.

```python
transport = StdioTransport(server_process)
```
Creates a transport that sends/receives messages on the subprocess's stdin/stdout.

```python
client = Client(transport)
```
Creates an MCP client that will use this transport to communicate with the server.

```python
await client.initialize()
```
Performs the initialization handshake. The client sends an initialize message, the server responds with capabilities. This is blocking—doesn't proceed until server responds.

```python
tools = await client.list_tools()
```
Asks the server: "What tools do you have?" Returns a list of Tool objects.

```python
result = await client.call_tool("get_customer", {"customer_id": "12345"})
```
Calls the tool named "get_customer" with the argument `customer_id` set to "12345". The server executes and returns the result.

```python
await client.close()
server_process.terminate()
```
Closes the connection and stops the server process.

### Expected Output

```
Initializing...
Connected!

Discovering tools...
  - get_customer: Retrieve customer information by ID
  - create_customer: Create a new customer
  - update_customer_balance: Update customer balance
  - deactivate_customer: Deactivate a customer account

Calling get_customer...
Result: {"name": "Alice", "status": "active", "balance": 5000}
```

## Client: Discovery Operations

### Complete Code

```python
import asyncio
import subprocess
from mcp.client import Client
from mcp.transport import StdioTransport

async def main():
    # Start server
    server_process = subprocess.Popen(
        ["python", "server.py"],
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE
    )
    
    transport = StdioTransport(server_process)
    client = Client(transport)
    
    try:
        await client.initialize()
        
        # ===== DISCOVER TOOLS =====
        print("=== Tools ===")
        tools = await client.list_tools()
        for tool in tools:
            print(f"Tool: {tool.name}")
            print(f"  Description: {tool.description}")
            print(f"  Input Schema: {tool.input_schema}")
            print()
        
        # ===== DISCOVER RESOURCES =====
        print("=== Resources ===")
        resources = await client.list_resources()
        for resource in resources:
            print(f"Resource: {resource.uri}")
            print(f"  Name: {resource.name}")
            print(f"  MIME Type: {resource.mime_type}")
            print()
        
        # ===== DISCOVER PROMPTS =====
        print("=== Prompts ===")
        prompts = await client.list_prompts()
        for prompt in prompts:
            print(f"Prompt: {prompt.name}")
            print(f"  Description: {prompt.description}")
            print(f"  Arguments: {prompt.arguments}")
            print()
        
    finally:
        await client.close()
        server_process.terminate()

if __name__ == "__main__":
    asyncio.run(main())
```

### What This Does

1. **List Tools:** Discovers all callable tools
2. **List Resources:** Discovers all readable resources
3. **List Prompts:** Discovers all available prompts

Each discovery operation queries the server and returns the full list with metadata.

## Client: Tool Invocation

### Simple Tool Call

```python
# Call a tool with arguments
result = await client.call_tool("get_customer", {
    "customer_id": "12345"
})
```

### Multiple Tool Calls

```python
# Call multiple tools
customer = await client.call_tool("get_customer", {"customer_id": "123"})
balance = await client.call_tool("check_balance", {"customer_id": "123"})
history = await client.call_tool("get_transaction_history", {
    "customer_id": "123",
    "limit": 10
})

print(f"Customer: {customer}")
print(f"Balance: {balance}")
print(f"History: {history}")
```

### Tool with Complex Arguments

```python
# Some tools need complex input
result = await client.call_tool("search_customers", {
    "filters": {
        "status": "active",
        "min_balance": 1000
    },
    "sort": "name",
    "limit": 50
})
```

### Error Handling

```python
try:
    result = await client.call_tool("get_customer", {"customer_id": "invalid"})
except Exception as e:
    print(f"Tool call failed: {e}")
    # Handle error
```

## Client: Resource Operations

### Reading a Resource

```python
# List resources
resources = await client.list_resources()
print(f"Available resources: {[r.uri for r in resources]}")

# Read a specific resource
document = await client.read_resource("doc://customers/policies.txt")
print(f"Document content:\n{document}")
```

### Complete Example

```python
import asyncio
import subprocess
from mcp.client import Client
from mcp.transport import StdioTransport

async def main():
    server_process = subprocess.Popen(
        ["python", "document_server.py"],
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE
    )
    
    transport = StdioTransport(server_process)
    client = Client(transport)
    
    try:
        await client.initialize()
        
        # Search for documents
        results = await client.call_tool("search_documents", {
            "query": "customer"
        })
        print(f"Search results: {results}")
        
        # Read first resource
        resources = await client.list_resources()
        if resources:
            content = await client.read_resource(resources[0].uri)
            print(f"First document:\n{content}")
    
    finally:
        await client.close()
        server_process.terminate()

if __name__ == "__main__":
    asyncio.run(main())
```

## Client: Prompt Operations

### Using Prompts

```python
# List available prompts
prompts = await client.list_prompts()
for prompt in prompts:
    print(f"Prompt: {prompt.name}")
    print(f"  Arguments: {prompt.arguments}")

# Get a specific prompt
prompt_text = await client.get_prompt("customer_analysis", {
    "customer_id": "123"
})

print(f"Prompt template:\n{prompt_text}")
```

### Complete Example with Claude Integration

```python
import asyncio
import subprocess
from mcp.client import Client
from mcp.transport import StdioTransport

async def main():
    # Start MCP server
    server_process = subprocess.Popen(
        ["python", "analysis_server.py"],
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE
    )
    
    transport = StdioTransport(server_process)
    client = Client(transport)
    
    try:
        await client.initialize()
        
        # Get prompt template from MCP server
        prompt_template = await client.get_prompt("customer_analysis", {
            "customer_id": "123"
        })
        
        # Use prompt with Claude (simulated)
        print(f"Prompt from MCP server:\n{prompt_template}")
        
        # In a real app, you'd send this to Claude:
        # claude_response = await claude_client.complete(prompt_template)
    
    finally:
        await client.close()
        server_process.terminate()

if __name__ == "__main__":
    asyncio.run(main())
```

## HTTP Transport for Remote Servers

### Client with HTTP Transport

```python
from mcp.client import Client
from mcp.transport import HttpTransport
import asyncio

async def main():
    # Connect to remote HTTP server
    transport = HttpTransport(base_url="http://localhost:8000")
    client = Client(transport)
    
    try:
        await client.initialize()
        
        # Use tools/resources/prompts from remote server
        result = await client.call_tool("get_customer", {"customer_id": "123"})
        print(result)
    
    finally:
        await client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

### Difference from Stdio

| Aspect | Stdio | HTTP |
|--------|-------|------|
| **Connection** | Local subprocess | Remote URL |
| **Startup** | Need to start subprocess | Server already running |
| **Port** | N/A | Configured port |
| **Use Case** | IDE plugins, local tools | Cloud servers, remote APIs |

## Client Error Handling

### Handling Tool Errors

```python
try:
    result = await client.call_tool("process_order", {
        "order_id": "nonexistent"
    })
except ValueError as e:
    print(f"Tool returned error: {e}")
except Exception as e:
    print(f"Communication error: {e}")
```

### Handling Connection Errors

```python
try:
    await client.initialize()
except ConnectionError as e:
    print(f"Failed to connect to server: {e}")
except TimeoutError as e:
    print(f"Connection timeout: {e}")
```

### Retry Logic

```python
import asyncio

async def call_with_retry(client, tool_name, arguments, max_retries=3):
    for attempt in range(max_retries):
        try:
            return await client.call_tool(tool_name, arguments)
        except Exception as e:
            if attempt < max_retries - 1:
                await asyncio.sleep(2 ** attempt)  # Exponential backoff
                continue
            else:
                raise

# Usage
result = await call_with_retry(client, "get_customer", {"customer_id": "123"})
```

## Client Best Practices

### 1. Always Close Connections

```python
async def safe_client_usage():
    client = Client(transport)
    try:
        await client.initialize()
        # Use client
        ...
    finally:
        await client.close()  # Always close!
```

### 2. Cache Capabilities

```python
class CachedClient:
    def __init__(self, client):
        self.client = client
        self._tools = None
        self._resources = None
    
    async def get_tools(self):
        if self._tools is None:
            self._tools = await self.client.list_tools()
        return self._tools
```

### 3. Validate Arguments

```python
async def call_tool_safely(client, tool_name, arguments):
    tools = await client.list_tools()
    tool = next((t for t in tools if t.name == tool_name), None)
    
    if not tool:
        raise ValueError(f"Tool {tool_name} not found")
    
    # Validate arguments against schema
    # (schema validation logic would go here)
    
    return await client.call_tool(tool_name, arguments)
```

### 4. Logging

```python
import logging

logger = logging.getLogger(__name__)

async def main():
    logger.info("Starting MCP client")
    client = Client(transport)
    await client.initialize()
    logger.info("Client initialized")
    
    result = await client.call_tool("get_customer", {"customer_id": "123"})
    logger.info(f"Tool result: {result}")
```

## Project Structure

```
my_client/
├── client.py          # Main client code
├── main.py            # Application logic
├── requirements.txt   # Dependencies
└── README.md          # Documentation
```

### requirements.txt

```
mcp>=0.1.0
```

## Testing

### Mock Server for Testing

```python
import pytest
from mcp.server import Server
from mcp.client import Client
from mcp.transport import StdioTransport
import subprocess
import asyncio

@pytest.mark.asyncio
async def test_client_tool_call(tmp_path):
    # Create a test server
    server_code = '''
from mcp.server import Server
from mcp.transport import StdioTransport
import asyncio

server = Server("test-server")

@server.tool()
def get_customer(customer_id: str):
    return f"Customer {customer_id}"

async def main():
    transport = StdioTransport()
    await server.run(transport)

asyncio.run(main())
'''
    
    server_file = tmp_path / "test_server.py"
    server_file.write_text(server_code)
    
    # Run server
    server_process = subprocess.Popen(
        ["python", str(server_file)],
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE
    )
    
    # Test client
    transport = StdioTransport(server_process)
    client = Client(transport)
    
    try:
        await client.initialize()
        result = await client.call_tool("get_customer", {"customer_id": "123"})
        assert "123" in result
    finally:
        await client.close()
        server_process.terminate()
```

## Interview Summary

**Key Client Concepts:**
- **Client Class:** Main interface for server communication
- **Initialize:** Handshake with server before using
- **Discovery:** List available tools, resources, prompts
- **Tool Invocation:** Call tools and handle results
- **Transports:** stdio for local, HTTP for remote
- **Error Handling:** Graceful error handling and retries
- **Resource Management:** Always close connections

**One-Liner:**
"An MCP client discovers and invokes capabilities (tools, resources, prompts) from an MCP server through the SDK, handling protocol details and connection management."

## Common Interview Questions

**Q1: What's the first thing a client must do when connecting to a server?**
A: Call `await client.initialize()`. This performs the initialization handshake and retrieves the server's capabilities.

**Q2: What's the difference between list_tools() and call_tool()?**
A: `list_tools()` discovers what tools exist (metadata). `call_tool()` actually executes a tool (invocation).

**Q3: How do you handle errors when calling tools?**
A: Use try/except. Tool execution errors from the server become exceptions. Connection errors are separate exceptions.

**Q4: Can a client talk to multiple servers?**
A: Yes, you create separate client instances for each server, each with its own transport.

**Q5: Should you call list_tools() every time you want to call a tool?**
A: No, for efficiency cache the results. Only refresh if you know the server's tools have changed.

---

**Next:** [07_MCP_Multi_Server.md](07_MCP_Multi_Server.md) – Connecting to multiple MCP servers
