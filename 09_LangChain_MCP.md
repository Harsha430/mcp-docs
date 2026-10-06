# 09: LangChain + MCP Integration

**Version/Source Basis:** MCP Protocol as of July 28, 2026 and LangChain current documentation. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

## LangChain + MCP Architecture

### How They Connect

```
┌─────────────────────────────────────┐
│  LangChain Agent (Host)             │
│  - Thinks/reasons                   │
│  - Decides which tool to use        │
└─────────────────────────────────────┘
           │
           │ Owns
           ▼
┌─────────────────────────────────────┐
│  LangChain MCP Adapter/Integration  │
│  - Wraps MCP client(s)              │
│  - Exposes tools to agent           │
│  - Translates LangChain ↔ MCP       │
└─────────────────────────────────────┘
           │
           │ Uses
           ▼
┌─────────────────────────────────────┐
│  MCP Client (Official SDK v2)       │
│  - Communicates with servers        │
│  - Discovers capabilities           │
│  - Invokes tools                    │
└─────────────────────────────────────┘
           │
           │ Connects to
           ▼
┌─────────────────────────────────────┐
│  MCP Servers (Customer, Banking,   │
│   Documents, etc.)                  │
└─────────────────────────────────────┘
```

### Real-World Analogy

LangChain is a coordinator that plans work. MCP servers are specialists. The LangChain adapter is the liaison who:
1. Knows what each specialist can do (discovers from MCP servers)
2. Tells the coordinator about available options
3. Relays the coordinator's requests to specialists
4. Reports back specialist answers

## LangChain + MCP Conceptually

**LangChain without MCP:**
```python
agent = Agent()
agent.add_tool(get_customer_tool)  # Hardcoded tool
result = agent.run("Get customer 123")
```

**LangChain with MCP:**
```python
agent = Agent()
adapter = LangChainMCPAdapter()
adapter.connect_to_server("http://localhost:8001")  # Discover tools dynamically
agent.use_adapter(adapter)
result = agent.run("Get customer 123")
```

The agent doesn't know about specific tools anymore—it discovers them dynamically from MCP servers.

## Integration Pattern

### Step 1: Create MCP Client(s)

```python
from mcp.client import Client
from mcp.transport import HttpTransport

# Create clients for each server
customer_transport = HttpTransport(base_url="http://localhost:8001")
customer_client = Client(customer_transport)
await customer_client.initialize()

banking_transport = HttpTransport(base_url="http://localhost:8002")
banking_client = Client(banking_transport)
await banking_client.initialize()
```

### Step 2: Create LangChain Adapter

```python
# Hypothetical LangChain adapter (implementation depends on current LangChain version)
from langchain.adapters import MCPAdapter

adapter = MCPAdapter()
adapter.add_client("customer_server", customer_client)
adapter.add_client("banking_server", banking_client)

# The adapter discovers tools from all servers
tools = await adapter.get_tools()
# tools = [get_customer, create_customer, transfer_money, ...]
```

### Step 3: Create LangChain Agent with MCP Tools

```python
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_openai import ChatOpenAI

# Create LLM
llm = ChatOpenAI(model="gpt-4")

# Create agent with MCP tools
agent = create_openai_tools_agent(
    llm,
    tools,  # These are MCP tools via adapter
    prompt  # Agent prompt
)

# Create executor
executor = AgentExecutor(agent=agent, tools=tools)

# Run agent
result = await executor.invoke({"input": "Get customer 123 and check their balance"})
```

### Step 4: Agent Uses MCP

```
User: "Get customer 123 and check their balance"
       ↓
LangChain Agent: "I need to call get_customer and check_balance"
       ↓
Adapter: "Routing get_customer to customer_server"
       ↓
MCP Client: "Sending JSON-RPC call_tool message"
       ↓
Customer Server: "Executing get_customer(123)"
       ↓
Response travels back up through client → adapter → agent → user
```

## Complete Example: LangChain + One MCP Server

### Setup: Start Customer Server

```python
# customer_server.py (runs separately)
from mcp.server import Server
from mcp.transport import HttpTransport
import asyncio

server = Server("customer-server")

@server.tool()
def get_customer(customer_id: str) -> str:
    """Get customer information"""
    customers = {
        "123": {"name": "Alice", "status": "active"},
        "456": {"name": "Bob", "status": "inactive"}
    }
    if customer_id in customers:
        return str(customers[customer_id])
    raise ValueError(f"Customer {customer_id} not found")

async def main():
    transport = HttpTransport(host="0.0.0.0", port=8001)
    await server.run(transport)

if __name__ == "__main__":
    asyncio.run(main())
```

### LangChain Application

```python
import asyncio
from mcp.client import Client
from mcp.transport import HttpTransport
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain.tools import Tool

async def main():
    # 1. Connect to MCP server
    transport = HttpTransport(base_url="http://localhost:8001")
    mcp_client = Client(transport)
    await mcp_client.initialize()
    
    # 2. Get tools from MCP server
    mcp_tools = await mcp_client.list_tools()
    
    # 3. Create LangChain tools from MCP tools
    langchain_tools = []
    for mcp_tool in mcp_tools:
        def create_tool_func(tool_name):
            async def tool_func(**kwargs):
                result = await mcp_client.call_tool(tool_name, kwargs)
                return result
            return tool_func
        
        langchain_tool = Tool(
            name=mcp_tool.name,
            description=mcp_tool.description,
            func=create_tool_func(mcp_tool.name)
        )
        langchain_tools.append(langchain_tool)
    
    # 4. Create LangChain agent with MCP tools
    llm = ChatOpenAI(model="gpt-4")
    agent = create_openai_tools_agent(llm, langchain_tools)
    executor = AgentExecutor(agent=agent, tools=langchain_tools)
    
    # 5. Run agent
    result = await executor.ainvoke({
        "input": "Get information about customer 123"
    })
    
    print(f"Result: {result}")
    await mcp_client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

## Multiple MCP Servers with LangChain

### Server Setup

```python
# customer_server.py
from mcp.server import Server
@server.tool()
def get_customer(customer_id: str):
    ...

# banking_server.py
from mcp.server import Server
@server.tool()
def transfer_money(from_id: str, to_id: str, amount: float):
    ...

# document_server.py
from mcp.server import Server
@server.tool()
def search_documents(query: str):
    ...
```

### LangChain Client with Multiple Servers

```python
import asyncio
from mcp.client import Client
from mcp.transport import HttpTransport
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain.tools import Tool

async def main():
    # Connect to multiple servers
    clients = {}
    servers = {
        "customer": "http://localhost:8001",
        "banking": "http://localhost:8002",
        "documents": "http://localhost:8003"
    }
    
    for server_name, url in servers.items():
        transport = HttpTransport(base_url=url)
        client = Client(transport)
        await client.initialize()
        clients[server_name] = client
    
    # Collect all tools from all servers
    all_tools = []
    for server_name, client in clients.items():
        mcp_tools = await client.list_tools()
        
        for mcp_tool in mcp_tools:
            # Create a tool wrapper
            def create_tool_wrapper(srv_name, tool_name, cli):
                async def tool_func(**kwargs):
                    result = await cli.call_tool(tool_name, kwargs)
                    return result
                return tool_func
            
            langchain_tool = Tool(
                name=f"{server_name}_{mcp_tool.name}",  # Namespace tools
                description=mcp_tool.description,
                func=create_tool_wrapper(server_name, mcp_tool.name, client)
            )
            all_tools.append(langchain_tool)
    
    # Create agent with all tools
    llm = ChatOpenAI(model="gpt-4")
    agent = create_openai_tools_agent(llm, all_tools)
    executor = AgentExecutor(agent=agent, tools=all_tools)
    
    # Run complex query involving multiple servers
    result = await executor.ainvoke({
        "input": "Get customer 123, check their balance, and search for their recent documents"
    })
    
    print(f"Result: {result}")
    
    # Cleanup
    for client in clients.values():
        await client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

## Architecture Diagram: LangChain + Multiple MCP Servers

```
┌────────────────────────────────────────┐
│  LangChain Agent                       │
│  "Get customer 123 and their balance"  │
│                                        │
│  Reasoning:                            │
│  1. I need get_customer from customer  │
│  2. I need check_balance from banking  │
│  3. Call them in order                 │
└────────────────────────────────────────┘
           ▼ Agent decides which tool
    ┌─────────────────┐
    │  MCP Adapter    │
    │ - Dispatches to │
    │   correct client│
    └─────────────────┘
      ▼              ▼              ▼
   ┌──────┐      ┌──────┐      ┌──────┐
   │MCP   │      │MCP   │      │MCP   │
   │Cli 1 │      │Cli 2 │      │Cli 3 │
   └──────┘      └──────┘      └──────┘
      ▼              ▼              ▼
   ┌──────┐      ┌──────┐      ┌──────┐
   │Cust  │      │Bank  │      │Docs  │
   │Srvr  │      │Srvr  │      │Srvr  │
   └──────┘      └──────┘      └──────┘
```

## Tool Namespacing

When using multiple servers, avoid tool name collisions:

```python
# Without namespacing (bad):
# customer_server has: get_customer, list_customers
# document_server has: get_customer (document), list_customers (search)
# Collision! Which get_customer?

# With namespacing (good):
Tool(name="customer_get_customer", ...)
Tool(name="document_get_customer", ...)
```

## Error Handling in LangChain + MCP

```python
try:
    result = await executor.ainvoke({"input": "Get customer..."})
except Exception as e:
    print(f"Agent error: {e}")
    # Distinguish between:
    # - MCP client error (connection, protocol)
    # - Tool execution error (server returned error)
    # - Agent reasoning error (LLM issue)
```

## Interview Summary

**Key Concepts:**
- **LangChain Agent:** Decision-maker (host)
- **MCP Adapter:** Integration layer
- **MCP Client:** Protocol handler
- **MCP Servers:** Tool providers
- **Dynamic Discovery:** Tools discovered from servers
- **Multiple Servers:** One agent, multiple tool sources
- **Namespacing:** Avoid tool name conflicts

**One-Liner:**
"LangChain + MCP connects LangChain agents to dynamically discovered tools from MCP servers through an adapter that manages client connections."

## Common Interview Questions

**Q1: How does LangChain discover MCP tools?**
A: The LangChain adapter creates MCP clients for each server, calls list_tools on them, wraps the results as LangChain tools, and presents them to the agent.

**Q2: When you have multiple MCP servers, does the agent know which tool is on which server?**
A: Not internally. The adapter handles routing. If you namespace tool names (e.g., customer_get_customer), the agent can reason about which server to use based on the name.

**Q3: What happens if an MCP server goes down while the agent is running?**
A: The MCP client throws an error. The agent receives the error and can handle it (retry, use a different tool, fail gracefully).

**Q4: Can LangChain make parallel calls to multiple MCP servers?**
A: LangChain's orchestration determines this. If the agent decides two tools are independent, they can be called in parallel by the executor.

**Q5: How do you authenticate to MCP servers from LangChain?**
A: The MCP adapter creates clients with appropriate transports (HTTP with headers, etc.). Authentication credentials are provided when creating the client.

---

**Next:** [10_CrewAI_MCP.md](10_CrewAI_MCP.md) – CrewAI + MCP integration
