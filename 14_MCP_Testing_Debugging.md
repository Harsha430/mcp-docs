# 14: MCP Testing and Debugging

**Version/Source Basis:** MCP Protocol July 28, 2026. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

## Testing Layers

```
Unit Tests (Test tool logic)
     ↓
Integration Tests (Test tool + transport)
     ↓
End-to-End Tests (Test client + server + integration)
```

## Unit Tests: Tool Logic

### Server Tool Testing

```python
import pytest
from calculator_server import add, multiply, divide

def test_add():
    result = add(2, 3)
    assert "2 + 3 = 5" in result

def test_multiply():
    result = multiply(4, 5)
    assert "4 * 5 = 20" in result

def test_divide_by_zero():
    with pytest.raises(ValueError):
        divide(10, 0)

def test_divide():
    result = divide(10, 2)
    assert "10 / 2 = 5" in result
```

**Tests:** Logic, error handling, edge cases

**Doesn't test:** MCP protocol, transport, connection

## Integration Tests: Tool + Server

### Mock Server Testing

```python
import pytest
import asyncio
from mcp.server import Server
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_server_tool_execution():
    # Create server with tool
    server = Server("test-server")
    
    @server.tool()
    def get_data(query: str) -> str:
        return f"Results: {query}"
    
    # Get tool definition
    tools = await server.list_tools_impl()  # May vary by SDK
    
    assert len(tools) == 1
    assert tools[0].name == "get_data"
    assert tools[0].description

@pytest.mark.asyncio
async def test_server_error_handling():
    server = Server("test-server")
    
    @server.tool()
    def failing_tool():
        raise ValueError("Expected error")
    
    # Error should be catchable
    with pytest.raises(ValueError):
        await server.handle_call("failing_tool", {})
```

**Tests:** Server setup, tool discovery, error conversion

**Doesn't test:** Client, full message exchange

## End-to-End Tests: Client ↔ Server

### Complete Client-Server Test

```python
import pytest
import asyncio
import subprocess
from mcp.client import Client
from mcp.transport import StdioTransport
from pathlib import Path
import tempfile

@pytest.mark.asyncio
async def test_client_server_communication(tmp_path):
    # Create server script
    server_code = '''
from mcp.server import Server
from mcp.transport import StdioTransport
import asyncio

server = Server("test-server")

@server.tool()
def echo(text: str) -> str:
    """Echo back text"""
    return f"Echo: {text}"

async def main():
    transport = StdioTransport()
    await server.run(transport)

asyncio.run(main())
'''
    
    server_file = tmp_path / "test_server.py"
    server_file.write_text(server_code)
    
    # Start server
    server_process = subprocess.Popen(
        ["python", str(server_file)],
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE
    )
    
    try:
        # Create client
        transport = StdioTransport(server_process)
        client = Client(transport)
        
        # Initialize
        await client.initialize()
        
        # List tools
        tools = await client.list_tools()
        assert len(tools) > 0
        assert tools[0].name == "echo"
        
        # Call tool
        result = await client.call_tool("echo", {"text": "hello"})
        assert "Echo: hello" in result
        
        # Cleanup
        await client.close()
    
    finally:
        server_process.terminate()
        server_process.wait()
```

**Tests:** Full protocol flow, real transport, client-server interaction

## Testing Tool Invocation

### Valid Input Test

```python
@pytest.mark.asyncio
async def test_tool_call_success(client):
    result = await client.call_tool("get_customer", {"customer_id": "123"})
    assert result is not None
    assert "123" in str(result)
```

### Invalid Input Test

```python
@pytest.mark.asyncio
async def test_tool_call_with_invalid_input(client):
    with pytest.raises(Exception):  # Tool should raise/error
        await client.call_tool("get_customer", {"customer_id": ""})
```

### Missing Required Arguments

```python
@pytest.mark.asyncio
async def test_tool_call_missing_args(client):
    with pytest.raises(Exception):  # Should fail
        await client.call_tool("get_customer", {})  # Missing customer_id
```

## Testing Error Scenarios

### Server Error Handling

```python
@pytest.mark.asyncio
async def test_server_connection_error():
    # Try connecting to non-existent server
    transport = HttpTransport(base_url="http://localhost:99999")
    client = Client(transport)
    
    with pytest.raises(ConnectionError):
        await client.initialize()
```

### Tool Execution Failure

```python
@pytest.mark.asyncio
async def test_tool_execution_error(client):
    # Call a tool with inputs that cause it to fail
    with pytest.raises(Exception):
        await client.call_tool("process_data", {"data": "invalid"})
```

### Timeout Handling

```python
@pytest.mark.asyncio
async def test_client_timeout():
    transport = HttpTransport(
        base_url="http://localhost:8000",
        timeout=0.1  # Very short timeout
    )
    client = Client(transport)
    
    with pytest.raises(asyncio.TimeoutError):
        await client.initialize()
```

## Testing Multiple Servers

```python
@pytest.fixture
async def multi_server_setup(tmp_path):
    """Setup multiple test servers"""
    servers = {}
    processes = {}
    clients = {}
    
    # Create and start multiple servers
    server_configs = [
        ("customer", "get_customer"),
        ("banking", "transfer_money"),
        ("documents", "search_docs")
    ]
    
    for server_name, tool_name in server_configs:
        # Create server script
        server_code = f'''
from mcp.server import Server
from mcp.transport import StdioTransport
import asyncio

server = Server("{server_name}-server")

@server.tool()
def {tool_name}(**kwargs):
    return "OK"

async def main():
    transport = StdioTransport()
    await server.run(transport)

asyncio.run(main())
'''
        
        server_file = tmp_path / f"{server_name}_server.py"
        server_file.write_text(server_code)
        
        # Start
        processes[server_name] = subprocess.Popen(
            ["python", str(server_file)],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE
        )
        
        # Create client
        clients[server_name] = Client(StdioTransport(processes[server_name]))
        await clients[server_name].initialize()
    
    yield clients
    
    # Cleanup
    for process in processes.values():
        process.terminate()

@pytest.mark.asyncio
async def test_multi_server_coordination(multi_server_setup):
    clients = multi_server_setup
    
    # Call tools from different servers
    result1 = await clients["customer"].call_tool("get_customer", {})
    result2 = await clients["banking"].call_tool("transfer_money", {})
    result3 = await clients["documents"].call_tool("search_docs", {})
    
    assert result1 == "OK"
    assert result2 == "OK"
    assert result3 == "OK"
```

## Debugging: Protocol-Level

### Log All Messages

```python
import logging

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger("mcp")

# This will log all protocol messages
logger = logging.getLogger("mcp.client")
logger.setLevel(logging.DEBUG)
```

### Inspect JSON-RPC Messages

```python
@pytest.mark.asyncio
async def test_inspect_messages(client):
    """Inspect the actual JSON-RPC messages"""
    
    # Monkey-patch to log messages
    original_send = client.transport.send
    original_receive = client.transport.receive
    
    async def logged_send(message):
        print(f"Sending: {json.dumps(message, indent=2)}")
        return await original_send(message)
    
    async def logged_receive():
        message = await original_receive()
        print(f"Received: {json.dumps(message, indent=2)}")
        return message
    
    client.transport.send = logged_send
    client.transport.receive = logged_receive
    
    # Now run a tool call and see messages
    result = await client.call_tool("get_customer", {"customer_id": "123"})
```

### Expected Output

```json
Sending: {
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "get_customer",
    "arguments": {"customer_id": "123"}
  },
  "id": 1
}

Received: {
  "jsonrpc": "2.0",
  "result": "Customer: Alice",
  "id": 1
}
```

## Debugging: Common Issues

### Issue 1: "Connection Refused"

```
Error: Connection refused on localhost:8000

Solution:
1. Verify server is running: ps aux | grep python
2. Verify port: netstat -tuln | grep 8000
3. Check firewall: sudo ufw status
4. Try different port: HttpTransport(base_url="http://localhost:9000")
```

### Issue 2: "Tool Not Found"

```
Error: Tool 'get_customer' not found

Solution:
1. Verify tool is decorated: @server.tool()
2. List available: tools = await client.list_tools()
3. Check tool name matches exactly (case-sensitive)
4. Restart server after adding tool
```

### Issue 3: "Invalid Arguments"

```
Error: Invalid arguments for tool

Solution:
1. Check schema: tool.inputSchema
2. Verify required fields are provided
3. Check argument types match schema
4. Use @dataclass or TypedDict for clarity
```

### Issue 4: "Timeout"

```
Error: Timeout waiting for response

Solution:
1. Increase timeout: HttpTransport(timeout=30)
2. Check if server is responding: curl -X POST http://localhost:8000/tools/call
3. Check network latency: ping server
4. Profile tool execution: time the tool function
```

## Test Coverage Example

```python
def test_suite():
    """Complete test suite structure"""
    
    # Unit tests
    test_calculator_logic()
    
    # Integration tests
    test_server_tool_registration()
    test_server_error_handling()
    
    # E2E tests
    test_client_server_initialization()
    test_client_tool_invocation()
    test_client_error_handling()
    
    # Multi-server tests
    test_multi_server_coordination()
    test_multi_server_failure_isolation()
    
    # Security tests
    test_authentication_required()
    test_authorization_enforced()
    
    # Performance tests
    test_tool_call_latency()
    test_concurrent_calls()
```

## Interview Summary

**Key Testing Concepts:**
- **Unit Tests:** Test tool logic independently
- **Integration Tests:** Test tool + server
- **E2E Tests:** Test client + server together
- **Error Scenarios:** Connection failures, timeouts, invalid inputs
- **Multi-Server:** Test coordination across servers
- **Debugging:** Log messages, inspect protocol, diagnose issues

**One-Liner:**
"Test MCP at three levels: tool logic (unit), tool + server (integration), and client + server (E2E), covering error scenarios and debugging protocol issues."

## Common Interview Questions

**Q1: How do you test an MCP server?**
A: At multiple levels. Unit tests verify tool logic. Integration tests verify the server setup and tool registration. E2E tests verify the full client-server communication including message exchange.

**Q2: What do you test for error handling?**
A: Connection errors (server down), tool execution errors (invalid arguments), timeouts (slow operations), and MCP protocol errors (invalid responses).

**Q3: How do you debug when tool calls fail?**
A: Log protocol messages to see the JSON-RPC exchange. Verify the server is running and accepting connections. Check tool arguments against the schema. Profile the tool execution to catch timeouts.

**Q4: How do you test multiple MCP servers together?**
A: Use fixtures to start multiple servers, create clients for each, and test tool calls across servers. Verify that failures in one don't break others.

**Q5: What's a common debugging pattern?**
A: Start with a working simple case, add one complexity at a time, log protocol messages to understand what's happening, and verify assumptions about tool arguments and return types.

---

**Next:** [15_MCP_Interview_Questions.md](15_MCP_Interview_Questions.md) – Comprehensive interview Q&A
