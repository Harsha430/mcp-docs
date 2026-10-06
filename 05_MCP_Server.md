# 05: Building MCP Servers with Python SDK v2

**Version/Source Basis:** Official MCP Python SDK v2, July 28, 2026. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

## What Is an MCP Server?

### Simple Explanation

An **MCP server** is software that:
1. Exposes capabilities (tools, resources, prompts) through MCP protocol
2. Receives requests from MCP clients
3. Executes the requested operation
4. Returns results

**You build servers** when you have functionality (business logic, database, API) you want to make available to MCP hosts.

### Real-World Analogy

A restaurant server (waiter) exposes capabilities:
- "I can take orders" (tool: place_order)
- "I have today's menu" (resource: menu.txt)
- "I can suggest wine pairings" (prompt: wine_suggestion)

Customers (MCP clients) discover these and use them.

## Simple Server: One Tool

### Complete Code

```python
from mcp.server import Server
from mcp.transport import StdioTransport
import asyncio

# Create server instance
server = Server("customer-service")

# Define a tool
@server.tool()
def get_customer(customer_id: str) -> str:
    """
    Retrieve customer information.
    
    Args:
        customer_id: The unique customer identifier
    
    Returns:
        Customer details as a formatted string
    """
    # In real app, query database
    customers = {
        "12345": {"name": "Alice", "status": "active", "balance": 5000},
        "67890": {"name": "Bob", "status": "inactive", "balance": 0}
    }
    
    if customer_id not in customers:
        raise ValueError(f"Customer {customer_id} not found")
    
    customer = customers[customer_id]
    return f"Customer: {customer['name']}, Status: {customer['status']}, Balance: ${customer['balance']}"

# Run the server
async def main():
    transport = StdioTransport()
    await server.run(transport)

if __name__ == "__main__":
    asyncio.run(main())
```

### Line-by-Line Explanation

```python
from mcp.server import Server  # Import Server class
from mcp.transport import StdioTransport  # Import stdio transport
import asyncio  # Import async support
```
Imports the required modules for server creation and execution.

```python
server = Server("customer-service")
```
Creates a new MCP server named "customer-service". This is the instance that will handle all MCP protocol messages.

```python
@server.tool()
def get_customer(customer_id: str) -> str:
```
The `@server.tool()` decorator registers this function as an MCP tool. The function name becomes the tool name automatically.

```python
    if customer_id not in customers:
        raise ValueError(f"Customer {customer_id} not found")
```
Validation: If the customer doesn't exist, raise an exception. The SDK converts this to an MCP error response automatically.

```python
    return f"Customer: {customer['name']}, ..."
```
The return value becomes the tool result sent back to the client.

```python
async def main():
    transport = StdioTransport()
    await server.run(transport)
```
Sets up stdio transport (for local process communication) and runs the server. `await server.run()` is blocking—the server runs until interrupted.

### Execution

```bash
python server.py
```

The server starts and waits for client messages on stdin.

## Server with Multiple Tools

### Complete Code

```python
from mcp.server import Server
from mcp.transport import StdioTransport
import asyncio
import json

server = Server("customer-service")

# In-memory database
customers = {
    "12345": {"name": "Alice", "status": "active", "balance": 5000},
    "67890": {"name": "Bob", "status": "inactive", "balance": 0}
}

# Tool 1: Read
@server.tool()
def get_customer(customer_id: str) -> str:
    """Retrieve customer information by ID"""
    if customer_id not in customers:
        raise ValueError(f"Customer {customer_id} not found")
    customer = customers[customer_id]
    return json.dumps(customer)

# Tool 2: Create
@server.tool()
def create_customer(name: str, status: str = "active") -> str:
    """Create a new customer"""
    customer_id = str(max(int(cid) for cid in customers.keys()) + 1)
    customers[customer_id] = {
        "name": name,
        "status": status,
        "balance": 0
    }
    return f"Created customer {customer_id}: {name}"

# Tool 3: Update
@server.tool()
def update_customer_balance(customer_id: str, amount: float) -> str:
    """Update customer balance"""
    if customer_id not in customers:
        raise ValueError(f"Customer {customer_id} not found")
    customers[customer_id]["balance"] += amount
    return f"Updated customer {customer_id} balance to ${customers[customer_id]['balance']}"

# Tool 4: Delete (soft)
@server.tool()
def deactivate_customer(customer_id: str) -> str:
    """Deactivate a customer account"""
    if customer_id not in customers:
        raise ValueError(f"Customer {customer_id} not found")
    customers[customer_id]["status"] = "inactive"
    return f"Deactivated customer {customer_id}"

async def main():
    transport = StdioTransport()
    await server.run(transport)

if __name__ == "__main__":
    asyncio.run(main())
```

### What This Does

- **Tool 1 (get_customer):** Read operation
- **Tool 2 (create_customer):** Create operation
- **Tool 3 (update_customer_balance):** Update operation
- **Tool 4 (deactivate_customer):** Delete operation (soft delete)

All tools are automatically discovered and invoked by clients.

## Server with Resources

### Complete Code

```python
from mcp.server import Server
from mcp.transport import StdioTransport
import asyncio

server = Server("document-service")

# Documents storage
documents = {
    "doc://customers/2026_analysis.txt": "Annual customer analysis report...",
    "doc://policies/terms.txt": "Terms of service..."
}

@server.tool()
def search_documents(query: str) -> str:
    """Search for documents by keyword"""
    results = []
    for uri, content in documents.items():
        if query.lower() in content.lower():
            results.append(uri)
    return f"Found {len(results)} documents: {results}"

# Resource: Define what resources are available
@server.resource()
def get_resources():
    """List available resources"""
    return {
        "resources": [
            {
                "uri": uri,
                "name": uri.split("/")[-1],
                "description": "Document resource",
                "mimeType": "text/plain"
            }
            for uri in documents.keys()
        ]
    }

# Resource handler: Read a specific resource
@server.resource_read()
def read_resource(uri: str) -> str:
    """Read a specific resource by URI"""
    if uri not in documents:
        raise ValueError(f"Resource {uri} not found")
    return documents[uri]

async def main():
    transport = StdioTransport()
    await server.run(transport)

if __name__ == "__main__":
    asyncio.run(main())
```

### How Resources Work

**Tools** are executable functions. **Resources** are data/documents that can be read.

```
Tool: get_customer(id) → executes, returns data
Resource: doc://customers.txt → read file content
```

## Server with Prompts

### Complete Code

```python
from mcp.server import Server
from mcp.transport import StdioTransport
import asyncio

server = Server("analysis-service")

@server.prompt()
def customer_analysis_prompt(customer_id: str) -> str:
    """
    Generate a prompt template for analyzing a customer.
    
    Args:
        customer_id: The customer to analyze
    
    Returns:
        A formatted prompt template
    """
    template = f"""
Analyze the following customer:

Customer ID: {customer_id}
Instructions:
1. Review customer history
2. Identify spending patterns
3. Suggest recommendations
4. Assess risk level

Please provide a comprehensive analysis.
"""
    return template

@server.prompt()
def risk_assessment_prompt(customer_id: str, balance: float) -> str:
    """Generate a prompt for risk assessment"""
    template = f"""
Risk Assessment for Customer {customer_id}

Current Balance: ${balance}
Task: Determine risk level based on:
- Balance amount
- Transaction history
- Customer tenure
- Account status

Categories: Low, Medium, High
"""
    return template

async def main():
    transport = StdioTransport()
    await server.run(transport)

if __name__ == "__main__":
    asyncio.run(main())
```

### How Prompts Work

Prompts are templates or guidance. A client:
1. Requests a prompt with arguments
2. Receives a formatted template
3. Uses the template to instruct an LLM

Example client usage:
```python
# Get the prompt
prompt = await client.get_prompt("customer_analysis_prompt", {"customer_id": "123"})

# Use it with Claude
response = await claude.complete(prompt)
```

## Combined: Tools, Resources, Prompts

### Complete Server Example

```python
from mcp.server import Server
from mcp.transport import StdioTransport
import asyncio
import json

server = Server("integrated-service")

# In-memory storage
customers = {
    "123": {"name": "Alice", "balance": 5000}
}
documents = {
    "doc://customers/policies.txt": "Customer policies..."
}

# ===== TOOLS =====

@server.tool()
def get_customer(customer_id: str) -> str:
    """Get customer information"""
    if customer_id not in customers:
        raise ValueError(f"Not found")
    return json.dumps(customers[customer_id])

# ===== RESOURCES =====

@server.resource()
def get_resources():
    """List available documents"""
    return {
        "resources": [
            {"uri": uri, "name": uri.split("/")[-1], "mimeType": "text/plain"}
            for uri in documents.keys()
        ]
    }

@server.resource_read()
def read_resource(uri: str) -> str:
    """Read a document"""
    if uri not in documents:
        raise ValueError(f"Not found")
    return documents[uri]

# ===== PROMPTS =====

@server.prompt()
def customer_summary(customer_id: str) -> str:
    """Prompt template for summarizing customer"""
    return f"Summarize customer {customer_id}: Name, Balance, Risk Level"

async def main():
    transport = StdioTransport()
    await server.run(transport)

if __name__ == "__main__":
    asyncio.run(main())
```

## Transport Options

### Stdio (Local Process)

```python
from mcp.transport import StdioTransport

transport = StdioTransport()
await server.run(transport)
```

**Use when:** Local communication (IDE plugin, local tool)

### HTTP (Remote)

```python
from mcp.transport import HttpTransport

transport = HttpTransport(host="0.0.0.0", port=8000)
await server.run(transport)
```

**Use when:** Remote server, cloud deployment

### SSE (Server-Sent Events)

```python
from mcp.transport import SSETransport

transport = SSETransport(host="0.0.0.0", port=8000)
await server.run(transport)
```

**Use when:** Stateless HTTP, web-based

## Error Handling

### Built-in Error Conversion

```python
@server.tool()
def process_order(order_id: str):
    # Any exception becomes an MCP error
    if not order_id:
        raise ValueError("order_id required")
    if order_id == "invalid":
        raise RuntimeError("Order not found")
    return "OK"
```

### Custom Error Handling

```python
from mcp.types import Error

@server.tool()
def sensitive_operation(data: str):
    try:
        result = process(data)
        return result
    except Exception as e:
        # Return Error object directly
        raise ValueError(f"Processing failed: {str(e)}")
```

## Server Best Practices

### 1. Input Validation

```python
@server.tool()
def transfer_money(from_id: str, to_id: str, amount: float) -> str:
    # Validate all inputs
    if not from_id or not to_id:
        raise ValueError("from_id and to_id required")
    if amount <= 0:
        raise ValueError("amount must be positive")
    if from_id == to_id:
        raise ValueError("Cannot transfer to same account")
    
    # Proceed with transfer
    ...
```

### 2. Logging

```python
import logging

logger = logging.getLogger(__name__)

@server.tool()
def get_customer(customer_id: str):
    logger.info(f"get_customer called with {customer_id}")
    ...
    logger.info(f"get_customer returned successfully")
```

### 3. Authentication/Authorization

```python
@server.tool()
def admin_operation(command: str, token: str):
    # Validate token
    if not validate_token(token):
        raise PermissionError("Unauthorized")
    
    # Execute operation
    ...
```

### 4. External Service Integration

```python
import requests

@server.tool()
def get_weather(location: str) -> str:
    """Get weather from external API"""
    response = requests.get(f"https://api.weather.com/location/{location}")
    if response.status_code != 200:
        raise RuntimeError(f"Weather API error: {response.status_code}")
    return response.json()
```

## Project Structure

```
my_server/
├── server.py              # Main server code
├── requirements.txt       # pip install -r requirements.txt
├── .env                   # Configuration
├── .env.example           # Template
├── tests/
│   ├── test_server.py     # Unit tests
│   └── conftest.py        # Test fixtures
└── README.md              # Documentation
```

### requirements.txt

```
mcp>=0.1.0
python-dotenv>=1.0.0
```

## Interview Summary

**Key Server Concepts:**
- **@server.tool():** Registers a function as an MCP tool
- **@server.resource():** Declares available resources
- **@server.prompt():** Provides prompt templates
- **Exceptions:** Automatically converted to MCP errors
- **Transport:** stdio, HTTP, SSE, WebSocket options
- **Async:** All handlers must be async-compatible

**One-Liner:**
"An MCP server exposes tools (callable functions), resources (readable data), and prompts (templates) via the MCP protocol, handling client requests through the SDK."

## Common Interview Questions

**Q1: How do you define a tool in an MCP server?**
A: Use the `@server.tool()` decorator on a function. The function name becomes the tool name, the docstring becomes the description, and return types become the result.

**Q2: What's the difference between tools and resources?**
A: Tools are executable functions. Resources are data/documents that can be read. Tools return computed results; resources return stored data.

**Q3: How does error handling work in servers?**
A: Any exception raised in a tool is automatically converted to an MCP error response with a code and message.

**Q4: Can a server use multiple transports?**
A: Typically, a server uses one transport (stdio, HTTP, etc.). To support multiple transports, run separate instances with different transports.

**Q5: How do you scale an MCP server?**
A: Deploy multiple instances behind a load balancer using HTTP or SSE transport. The stateless design allows easy horizontal scaling.

---

**Next:** [06_MCP_Client.md](06_MCP_Client.md) – Building MCP clients and consuming servers
