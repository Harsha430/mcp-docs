# 07: Multi-Server Architecture

**Version/Source Basis:** MCP Protocol as of July 28, 2026. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

## What Is Multi-Server Architecture?

### Simple Explanation

A single application (host) can connect to **multiple MCP servers** simultaneously, each providing different capabilities.

**Example:**
- App → Customer Server (get_customer, create_customer)
- App → Banking Server (transfer_money, check_balance)
- App → Document Server (search_docs, read_file)

The app orchestrates which server to use based on the task.

### Real-World Analogy

A personal assistant (app) works with multiple specialists:
- Accountant (Financial Server)
- Librarian (Document Server)
- Travel Agent (Booking Server)

The assistant knows which specialist to contact for each request.

## Architecture Diagram

```
┌────────────────────────────────────────────┐
│  Application / Host                        │
│  (Makes decisions about what to do)        │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │ Application Logic                    │  │
│  │ - Process user requests              │  │
│  │ - Decide which servers to use        │  │
│  │ - Combine results                    │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
         ▼              ▼              ▼
    ┌────────┐    ┌────────┐    ┌────────┐
    │ MCP    │    │ MCP    │    │ MCP    │
    │Client 1│    │Client 2│    │Client 3│
    └────────┘    └────────┘    └────────┘
         ▼              ▼              ▼
    ┌────────┐    ┌────────┐    ┌────────┐
    │Server 1│    │Server 2│    │Server 3│
    │Cust.   │    │Banking │    │Docs    │
    └────────┘    └────────┘    └────────┘
```

## Client Per Server Pattern

The standard pattern is: **One MCP Client per MCP Server**.

```python
import asyncio
from mcp.client import Client
from mcp.transport import StdioTransport
import subprocess

class MultiServerApp:
    def __init__(self):
        self.customers_client = None
        self.banking_client = None
        self.documents_client = None
    
    async def setup(self):
        # Setup customer server connection
        customer_proc = subprocess.Popen(
            ["python", "customer_server.py"],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE
        )
        self.customers_client = Client(StdioTransport(customer_proc))
        await self.customers_client.initialize()
        
        # Setup banking server connection
        banking_proc = subprocess.Popen(
            ["python", "banking_server.py"],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE
        )
        self.banking_client = Client(StdioTransport(banking_proc))
        await self.banking_client.initialize()
        
        # Setup document server connection
        document_proc = subprocess.Popen(
            ["python", "document_server.py"],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE
        )
        self.documents_client = Client(StdioTransport(document_proc))
        await self.documents_client.initialize()
    
    async def process_customer_order(self, customer_id, amount):
        # Get customer info
        customer = await self.customers_client.call_tool(
            "get_customer",
            {"customer_id": customer_id}
        )
        
        # Check balance
        balance = await self.banking_client.call_tool(
            "check_balance",
            {"account_id": customer_id}
        )
        
        # Read policy
        policy = await self.documents_client.read_resource(
            "doc://policies/order_policy.txt"
        )
        
        # Combine and process
        print(f"Customer: {customer}")
        print(f"Balance: {balance}")
        print(f"Policy: {policy}")
        
        if float(balance) >= amount:
            result = await self.banking_client.call_tool(
                "transfer_money",
                {"account_id": customer_id, "amount": amount}
            )
            return f"Order processed: {result}"
        else:
            return "Insufficient balance"

async def main():
    app = MultiServerApp()
    await app.setup()
    
    result = await app.process_customer_order("12345", 500)
    print(result)

if __name__ == "__main__":
    asyncio.run(main())
```

## Centralized Client Manager

For complex applications, use a manager to handle multiple clients:

```python
import asyncio
from mcp.client import Client
from mcp.transport import HttpTransport
from typing import Dict

class MCPClientManager:
    def __init__(self):
        self.clients: Dict[str, Client] = {}
    
    async def add_server(self, name: str, server_url: str):
        """Add a new server connection"""
        transport = HttpTransport(base_url=server_url)
        client = Client(transport)
        await client.initialize()
        self.clients[name] = client
        print(f"Connected to {name}")
    
    def get_client(self, name: str) -> Client:
        """Get a client by server name"""
        if name not in self.clients:
            raise ValueError(f"Server {name} not connected")
        return self.clients[name]
    
    async def call_tool(self, server_name: str, tool_name: str, args: dict):
        """Call a tool on a specific server"""
        client = self.get_client(server_name)
        return await client.call_tool(tool_name, args)
    
    async def close_all(self):
        """Close all connections"""
        for client in self.clients.values():
            await client.close()

# Usage
async def main():
    manager = MCPClientManager()
    
    # Connect to multiple servers
    await manager.add_server("customers", "http://localhost:8001")
    await manager.add_server("banking", "http://localhost:8002")
    await manager.add_server("documents", "http://localhost:8003")
    
    # Use servers through manager
    customer = await manager.call_tool(
        "customers",
        "get_customer",
        {"customer_id": "123"}
    )
    print(f"Customer: {customer}")
    
    balance = await manager.call_tool(
        "banking",
        "check_balance",
        {"account_id": "123"}
    )
    print(f"Balance: {balance}")
    
    await manager.close_all()

if __name__ == "__main__":
    asyncio.run(main())
```

## Orchestration Patterns

### Sequential Orchestration

Call servers one after another:

```python
async def sequential_workflow(app):
    # Step 1: Get customer
    customer = await app.customers_client.call_tool(
        "get_customer",
        {"customer_id": "123"}
    )
    
    # Step 2: Check balance (depends on customer ID)
    balance = await app.banking_client.call_tool(
        "check_balance",
        {"account_id": "123"}
    )
    
    # Step 3: Process based on results
    if balance > 1000:
        status = "premium"
    else:
        status = "standard"
    
    return f"Customer: {customer}, Status: {status}"
```

### Parallel Orchestration

Call multiple servers concurrently:

```python
async def parallel_workflow(app):
    # Call all servers concurrently
    customer_task = app.customers_client.call_tool(
        "get_customer",
        {"customer_id": "123"}
    )
    
    balance_task = app.banking_client.call_tool(
        "check_balance",
        {"account_id": "123"}
    )
    
    # Wait for all results
    customer, balance = await asyncio.gather(customer_task, balance_task)
    
    return f"Customer: {customer}, Balance: {balance}"
```

### Conditional Orchestration

Use one server's result to decide which next server to call:

```python
async def conditional_workflow(app):
    # Check customer status
    customer = await app.customers_client.call_tool(
        "get_customer",
        {"customer_id": "123"}
    )
    
    # Based on status, call different servers
    if customer["status"] == "premium":
        # Call premium services
        result = await app.banking_client.call_tool(
            "apply_premium_discount",
            {"customer_id": "123", "percentage": 20}
        )
    else:
        # Call standard services
        result = await app.banking_client.call_tool(
            "apply_standard_discount",
            {"customer_id": "123", "percentage": 5}
        )
    
    return result
```

## Service Isolation

Each server is isolated from others:

```
Server 1          Server 2          Server 3
(Customers)       (Banking)         (Documents)
├─ Database       ├─ Database       ├─ File System
├─ Auth           ├─ Auth           ├─ Auth
└─ Logging        └─ Logging        └─ Logging

Each has its own:
- Data
- Authentication
- Error handling
- Logging
```

Benefits:
- **Failure Isolation:** One server crash doesn't affect others
- **Independent Scaling:** Scale each server separately
- **Security:** Each server manages its own data
- **Teams:** Different teams can own different servers

## Tool Name Conflicts

When using multiple servers, tool names might conflict:

```
Customer Server: get_customer()
Document Server: get_customer() ← Same name!
```

**Solution:** Always specify which server to use:

```python
customer_info = await app.customers_client.call_tool(
    "get_customer",  # Can be the same name
    {"customer_id": "123"}
)

doc_customer = await app.documents_client.call_tool(
    "get_customer",  # Different server, so no conflict
    {"doc_id": "abc"}
)
```

Or namespace the tools:

```
Customer Server:  customers_get_customer()
Document Server:  documents_get_customer()
Banking Server:   banking_transfer_money()
```

## Error Handling with Multiple Servers

```python
async def robust_multi_server_call(app, calls):
    """
    Make multiple server calls, handling errors gracefully
    
    calls: list of (server_client, tool_name, arguments)
    """
    results = {}
    errors = {}
    
    for server_name, client, tool_name, args in calls:
        try:
            result = await client.call_tool(tool_name, args)
            results[server_name] = result
        except Exception as e:
            errors[server_name] = str(e)
    
    return {
        "results": results,
        "errors": errors,
        "success": len(errors) == 0
    }

# Usage
response = await robust_multi_server_call(
    app,
    [
        ("customers", app.customers_client, "get_customer", {"customer_id": "123"}),
        ("banking", app.banking_client, "check_balance", {"account_id": "123"}),
        ("documents", app.documents_client, "list_resources", {}),
    ]
)

if response["errors"]:
    print(f"Some calls failed: {response['errors']}")
```

## Logging Across Servers

```python
import logging

logger = logging.getLogger(__name__)

async def traced_server_call(app, server_name, client, tool_name, args):
    """Call a server tool with logging"""
    logger.info(f"Calling {server_name}.{tool_name} with {args}")
    
    try:
        result = await client.call_tool(tool_name, args)
        logger.info(f"{server_name}.{tool_name} returned: {result}")
        return result
    except Exception as e:
        logger.error(f"{server_name}.{tool_name} failed: {e}")
        raise
```

## Resource Discovery Across Servers

```python
async def discover_all_resources(app):
    """List resources from all servers"""
    all_resources = {}
    
    for server_name, client in [
        ("customers", app.customers_client),
        ("banking", app.banking_client),
        ("documents", app.documents_client),
    ]:
        try:
            resources = await client.list_resources()
            all_resources[server_name] = resources
        except Exception as e:
            all_resources[server_name] = {"error": str(e)}
    
    return all_resources
```

## Performance Considerations

### Connection Overhead
Each client connection has setup overhead. For HTTP:
```
Connection Time: ~50-100ms per server
Total for 3 servers: ~150-300ms
```

### Caching Discovery Results
```python
class CachedMultiServerApp:
    def __init__(self):
        self.clients = {}
        self.tool_cache = {}
    
    async def get_tools(self, server_name):
        """Get tools from cache or discover"""
        if server_name not in self.tool_cache:
            client = self.clients[server_name]
            self.tool_cache[server_name] = await client.list_tools()
        return self.tool_cache[server_name]
```

### Concurrency
Use `asyncio.gather()` for parallel calls:
```python
results = await asyncio.gather(
    client1.call_tool(...),
    client2.call_tool(...),
    client3.call_tool(...),
    return_exceptions=True
)
```

## Testing Multi-Server Setup

```python
import pytest

@pytest.mark.asyncio
async def test_multi_server_workflow():
    # Create mock servers
    # Connect clients
    # Test workflow
    
    # Example:
    result = await app.process_customer_order("123", 500)
    assert "processed" in result or "Insufficient" in result
```

## Interview Summary

**Key Concepts:**
- **One Client Per Server:** Standard pattern
- **Client Manager:** Centralized management for many servers
- **Orchestration:** Sequential, parallel, conditional
- **Isolation:** Each server independent
- **Tool Namespacing:** Handle name conflicts
- **Error Handling:** Graceful degradation
- **Performance:** Cache discovery, use parallel calls

**One-Liner:**
"Multi-server architecture connects one application to multiple MCP servers, each with separate client connections, allowing orchestrated access to diverse capabilities."

## Common Interview Questions

**Q1: Can one application connect to multiple MCP servers?**
A: Yes. The typical pattern is one MCP Client per server, all managed by the application.

**Q2: How do you handle tool name conflicts across servers?**
A: Always reference the tool through its server's client. Or namespace tool names (e.g., customers_get_customer).

**Q3: What happens if one server crashes?**
A: The other servers continue working. The failed server's client throws an error, which you handle gracefully.

**Q4: How do you orchestrate work across multiple servers?**
A: Use async/await. Sequential (await one, then another), parallel (asyncio.gather()), or conditional (if result1, then call server2).

**Q5: Should you cache discovery results (list_tools)?**
A: Yes, for performance. Tools rarely change during runtime, so cache them and only refresh if needed.

---

**Next:** [08_Server_vs_Client_vs_Host.md](08_Server_vs_Client_vs_Host.md) – Clear boundaries between components
