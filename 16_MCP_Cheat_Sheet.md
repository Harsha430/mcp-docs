# 16: MCP Cheat Sheet – Quick Reference

**Version/Source Basis:** MCP Protocol July 28, 2026. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

---

## Core Concepts at a Glance

| Concept | Definition | Use Case |
|---------|-----------|----------|
| **MCP** | Model Context Protocol – standard for discovering and invoking capabilities | AI agents, IDE plugins, framework integrations |
| **Host** | Application making decisions (Claude, LangChain, your code) | Orchestrates MCP clients |
| **Client** | Protocol handler inside host | Sends/receives MCP messages |
| **Server** | Capability provider (database, API) | Exposes tools, resources, prompts |
| **Tool** | Callable function server exposes | Remote function invocation |
| **Resource** | Readable data server offers | Static/semi-static data |
| **Prompt** | Template guidance server provides | LLM instruction templates |
| **Transport** | How messages travel (HTTP, stdio, WebSocket) | Physical layer |
| **Stateless** | Each message independent | Scalability |
| **JSON-RPC** | Request/response message format | Protocol foundation |

---

## Installation Quick Start

### Official SDK v2

```bash
pip install mcp              # Basic
pip install "mcp[cli]"       # With CLI tools
pip install "mcp[dev]"       # With dev tools
```

### Verify

```bash
python -c "import mcp; print(mcp.__version__)"
```

---

## Server: Quick Template

### Minimal Server

```python
from mcp.server import Server
from mcp.transport import StdioTransport
import asyncio

server = Server("my-server")

@server.tool()
def my_tool(arg: str) -> str:
    """Tool description"""
    return "result"

async def main():
    transport = StdioTransport()
    await server.run(transport)

if __name__ == "__main__":
    asyncio.run(main())
```

### With Resources

```python
@server.resource()
def get_resources():
    return {"resources": [{"uri": "...", "name": "..."}]}

@server.resource_read()
def read_resource(uri: str) -> str:
    return "content"
```

### With Prompts

```python
@server.prompt()
def my_prompt(arg: str) -> str:
    return "prompt template"
```

### Error Handling

```python
@server.tool()
def process(data: str):
    if not data:
        raise ValueError("data required")
    return result
```

### Transports

```python
# Stdio (local)
transport = StdioTransport()

# HTTP (remote)
transport = HttpTransport(host="0.0.0.0", port=8000)

# HTTPS (secure)
transport = HttpTransport(host="0.0.0.0", port=8443,
                         ssl_certfile="cert.pem",
                         ssl_keyfile="key.pem")
```

---

## Client: Quick Template

### Minimal Client

```python
import asyncio
import subprocess
from mcp.client import Client
from mcp.transport import StdioTransport

async def main():
    # Start server
    proc = subprocess.Popen(["python", "server.py"],
                           stdin=subprocess.PIPE,
                           stdout=subprocess.PIPE)
    
    # Create client
    transport = StdioTransport(proc)
    client = Client(transport)
    
    # Initialize
    await client.initialize()
    
    # Use server
    tools = await client.list_tools()
    result = await client.call_tool("my_tool", {"arg": "value"})
    
    # Cleanup
    await client.close()
    proc.terminate()

asyncio.run(main())
```

### Discovery

```python
tools = await client.list_tools()
resources = await client.list_resources()
prompts = await client.list_prompts()
```

### Invocation

```python
result = await client.call_tool("tool_name", {"arg": "value"})
content = await client.read_resource("resource://uri")
prompt = await client.get_prompt("prompt_name", {"arg": "value"})
```

### Remote Server

```python
from mcp.transport import HttpTransport

transport = HttpTransport(base_url="http://localhost:8000")
client = Client(transport)
await client.initialize()
```

### With Auth

```python
transport = HttpTransport(
    base_url="https://api.example.com",
    headers={"Authorization": "Bearer token_xyz"}
)
```

---

## Multi-Server Pattern

```python
clients = {}
servers = {
    "customer": "http://localhost:8001",
    "banking": "http://localhost:8002"
}

# Connect
for name, url in servers.items():
    transport = HttpTransport(base_url=url)
    client = Client(transport)
    await client.initialize()
    clients[name] = client

# Use
customer = await clients["customer"].call_tool("get_customer", {})
balance = await clients["banking"].call_tool("check_balance", {})
```

---

## Security Essentials

### Authentication: JWT

```python
import jwt
from datetime import datetime, timedelta

# Generate
token = jwt.encode({
    "user_id": "123",
    "exp": datetime.utcnow() + timedelta(hours=1)
}, "secret_key")

# Validate
try:
    payload = jwt.decode(token, "secret_key")
except jwt.ExpiredSignatureError:
    raise PermissionError("Token expired")
```

### Authorization: Scopes

```python
@server.tool()
def sensitive_op():
    user_scopes = get_user_scopes()
    if "admin:*" not in user_scopes:
        raise PermissionError("Insufficient scope")
    return result
```

### Never Hardcode Secrets

```python
import os

API_KEY = os.getenv("API_KEY")  # ✅ Good
# API_KEY = "secret123"          # ❌ Bad
```

---

## JSON-RPC Messages

### Request

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {"name": "tool_name", "arguments": {}},
  "id": 1
}
```

### Response

```json
{
  "jsonrpc": "2.0",
  "result": "value",
  "id": 1
}
```

### Error

```json
{
  "jsonrpc": "2.0",
  "error": {"code": -32603, "message": "Internal error"},
  "id": 1
}
```

---

## Protocol Lifecycle

```
1. Client sends: initialize
2. Server responds: capabilities
3. Client sends: notifications/initialized
4. Client sends: tools/list (or resources/list, prompts/list)
5. Server responds: tool definitions
6. Client sends: tools/call
7. Server responds: result
8. Repeat 6-7 as needed
```

---

## Common Commands

### Test Connection

```bash
# Ping MCP server
curl -X POST http://localhost:8000/tools/list \
  -H "Content-Type: application/json"
```

### View Logs

```bash
# Enable debug logging
export DEBUG=mcp:*
python client.py
```

### Run Server

```bash
python server.py
```

### Test with CLI

```bash
# If installed with [cli]
mcp --help
mcp server run server.py
```

---

## Architecture Patterns

### Single Server

```
App → Client → Server
```

### Multi-Server

```
App → Client₁ → Server₁
    → Client₂ → Server₂
    → Client₃ → Server₃
```

### With Framework

```
LangChain Agent → Adapter → Client → Server
```

### Enterprise

```
Claude → Shared Servers ← LangChain
           ← CrewAI
           ← AutoGen
```

---

## Error Codes Reference

| Code | Meaning | Solution |
|------|---------|----------|
| -32700 | Parse error | Check JSON syntax |
| -32600 | Invalid Request | Check message format |
| -32601 | Method not found | Verify tool name |
| -32602 | Invalid params | Check arguments against schema |
| -32603 | Internal error | Check server logs |

---

## Debugging Checklist

- [ ] Server running? `ps aux | grep python`
- [ ] Port open? `netstat -tuln | grep 8000`
- [ ] Firewall blocking? `sudo ufw status`
- [ ] Tool exists? `await client.list_tools()`
- [ ] Arguments valid? Check `inputSchema`
- [ ] Authentication correct? Check token/API key
- [ ] HTTPS certificate valid? Check cert file
- [ ] Network reachable? `ping server`
- [ ] Timeout set appropriately? Try `timeout=30`
- [ ] Logs showing? Enable `DEBUG=mcp:*`

---

## Performance Tips

| Optimization | Effect | When |
|--------------|--------|------|
| Cache `list_tools()` | 50-100ms saved | Tool list rarely changes |
| Use `asyncio.gather()` | 3x faster | Calling independent tools |
| Implement connection pooling | 20% faster | Many sequential calls |
| Profile tool execution | Identify bottlenecks | Slow operations |
| Add pagination | Memory savings | Large result sets |

---

## Interview Soundbites

**"What is MCP?"**
> MCP is a standardized protocol for discovering and invoking capabilities from external services, enabling decoupling and dynamic tool discovery for AI agents.

**"Why MCP vs REST?"**
> REST requires hardcoding endpoints; MCP allows automatic discovery. Hosts don't need to know specific APIs upfront.

**"How do you scale MCP?"**
> Stateless design allows horizontal scaling. Deploy multiple server instances behind a load balancer using HTTP transport.

**"What's the protocol?"**
> JSON-RPC 2.0 over configurable transports (HTTP, WebSocket, stdio). Stateless request-response design.

**"How do frameworks integrate?"**
> Each framework has an adapter that wraps MCP clients. LangChain, CrewAI, AutoGen all use the same shared servers.

**"How to secure MCP?"**
> HTTPS transport, JWT/token authentication, scope-based authorization, never hardcode secrets.

---

## Terminology

- **Adapter:** Framework-specific bridge to MCP
- **Capability:** Tool, resource, or prompt a server exposes
- **Discovery:** Process of learning what capabilities are available
- **Endpoint:** URL or address of an MCP server
- **Invocation:** Act of calling a tool
- **JSON-RPC:** Message format used by MCP
- **Notification:** Fire-and-forget message (no response)
- **Protocol:** Specification (transport-independent)
- **Request:** Message expecting a response
- **Resource:** Readable data from a server
- **Scope:** Permission level (e.g., `customer:read`)
- **Stateless:** Each request independent
- **Token:** Credential for authentication
- **Transport:** Physical layer (HTTP, stdio, WebSocket)

---

## File Structure Template

```
my_mcp_project/
├── server.py           # MCP server code
├── client.py           # MCP client code
├── requirements.txt    # pip install -r requirements.txt
├── .env               # Environment variables (don't commit)
├── tests/
│   └── test_server.py # Test code
└── README.md          # Documentation
```

---

## .env Template

```bash
# .env
SECRET_KEY=your_secret_key_here
DATABASE_URL=postgresql://user:pass@localhost/db
MCP_PORT=8000
MCP_CERTFILE=/path/to/cert.pem
MCP_KEYFILE=/path/to/key.pem
DEBUG=mcp:*
```

---

## Testing Template

```python
import pytest

@pytest.mark.asyncio
async def test_tool():
    # Setup
    server = Server("test")
    
    # Execute
    result = await server.call_tool(...)
    
    # Assert
    assert result == expected
```

---

## Resources

- **Official Docs:** https://modelcontextprotocol.io/docs/2026-07-28/learn/
- **Python SDK:** https://github.com/modelcontextprotocol/python-sdk
- **JSON-RPC Spec:** https://www.jsonrpc.org/
- **JWT.io:** https://jwt.io/

---

## One-Page Summary

**What:** Protocol for discovering and invoking tools from external services.

**Why:** Decouples applications from services; enables dynamic tool discovery for AI agents.

**Components:** Host (app), Client (protocol handler), Server (capability provider).

**Protocol:** JSON-RPC 2.0, stateless, transport-independent.

**Transports:** HTTP/HTTPS, WebSocket, stdio.

**Core Concepts:** Tools (functions), Resources (data), Prompts (templates).

**SDK:** `mcp` package; use official v2 for production.

**Security:** HTTPS, JWT/tokens, scopes, never hardcode secrets.

**Frameworks:** LangChain, CrewAI, AutoGen integrate via adapters.

**Multi-Server:** One host can use multiple servers; pattern is one client per server.

**Interview Ready:** Know what MCP is, why it exists, how it works, how frameworks use it, security considerations, and real-world applications.

---

**End of Documentation**

For detailed information, see the complete documentation set (00_README.md through 15_MCP_Interview_Questions.md).
