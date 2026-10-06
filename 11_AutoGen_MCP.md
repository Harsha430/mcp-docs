# 11: AutoGen + MCP Integration

**Version/Source Basis:** MCP Protocol as of July 28, 2026. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

## AutoGen + MCP Architecture

### How They Connect

```
┌────────────────────────────────────────┐
│  AutoGen Agent / Multi-Agent Group     │
│  - Agent 1: User proxy                 │
│  - Agent 2: Assistant                  │
│  - Message flow between agents         │
└────────────────────────────────────────┘
           │
           │ Uses
           ▼
┌────────────────────────────────────────┐
│  AutoGen MCP Integration               │
│  - Wraps MCP clients                   │
│  - Exposes as tools to agents          │
│  - Handles tool execution              │
└────────────────────────────────────────┘
           │
           │ Uses
           ▼
┌────────────────────────────────────────┐
│  MCP Clients                           │
│  - One per server                      │
│  - HTTP/stdio transport                │
└────────────────────────────────────────┘
           │
           │ Connect to
           ▼
┌────────────────────────────────────────┐
│  MCP Servers                           │
│  - Customer, Banking, Document, etc.   │
└────────────────────────────────────────┘
```

## AutoGen + MCP Conceptually

**AutoGen** is a framework for multi-agent conversation. Agents communicate by exchanging messages. Tools are functions agents can call.

**MCP integration** makes MCP server capabilities available as tools to AutoGen agents.

```python
# Without MCP
assistant = AssistantAgent("assistant")
assistant.register_function(get_customer)  # Hardcoded tool

# With MCP
assistant = AssistantAgent("assistant")
adapter = AutoGenMCPAdapter()
adapter.register_server("http://localhost:8001")
# Now assistant discovers and can use tools dynamically
```

## Single Server Example

### Server Setup

```python
# server.py
from mcp.server import Server
from mcp.transport import HttpTransport
import asyncio

server = Server("info-server")

@server.tool()
def get_user_info(user_id: str) -> str:
    """Get user information"""
    users = {"123": {"name": "Alice", "email": "alice@example.com"}}
    return str(users.get(user_id, "Not found"))

@server.tool()
def get_order_history(user_id: str) -> str:
    """Get user's order history"""
    return f"User {user_id}: Order 1, Order 2, Order 3"

async def main():
    transport = HttpTransport(host="0.0.0.0", port=8001)
    await server.run(transport)

if __name__ == "__main__":
    asyncio.run(main())
```

### AutoGen Application

```python
import asyncio
from autogen import AssistantAgent, UserProxyAgent
from mcp.client import Client
from mcp.transport import HttpTransport

async def setup_mcp_tools(agent):
    """Setup MCP tools for an AutoGen agent"""
    # Connect to MCP server
    transport = HttpTransport(base_url="http://localhost:8001")
    mcp_client = Client(transport)
    await mcp_client.initialize()
    
    # Get tools from server
    mcp_tools = await mcp_client.list_tools()
    
    # Register as AutoGen functions
    for tool in mcp_tools:
        def create_tool_func(tool_name):
            def tool_func(**kwargs):
                import asyncio
                loop = asyncio.new_event_loop()
                result = loop.run_until_complete(
                    mcp_client.call_tool(tool_name, kwargs)
                )
                return result
            return tool_func
        
        agent.register_function(
            function=create_tool_func(tool.name),
            description=tool.description
        )
    
    return mcp_client

async def main():
    # Create agents
    user_proxy = UserProxyAgent(
        name="User",
        human_input_mode="TERMINATE"
    )
    
    assistant = AssistantAgent(
        name="Assistant",
        llm_config={"model": "gpt-4"}
    )
    
    # Setup MCP tools
    mcp_client = await setup_mcp_tools(assistant)
    
    # User initiates conversation
    user_proxy.initiate_chat(
        assistant,
        message="Get information for user 123 and their order history"
    )
    
    await mcp_client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

## Multiple Servers Example

### Server Setup

Three servers:
```python
# customer_server.py - Port 8001
@server.tool()
def get_customer(customer_id: str): ...

# banking_server.py - Port 8002  
@server.tool()
def check_balance(account_id: str): ...

# documents_server.py - Port 8003
@server.tool()
def search_documents(query: str): ...
```

### AutoGen Multi-Server Application

```python
import asyncio
from autogen import AssistantAgent, UserProxyAgent, GroupChat, GroupChatManager
from mcp.client import Client
from mcp.transport import HttpTransport

async def main():
    # Connect to all servers
    servers = {
        "customer": "http://localhost:8001",
        "banking": "http://localhost:8002",
        "documents": "http://localhost:8003"
    }
    
    clients = {}
    all_tools = {}
    
    for server_name, url in servers.items():
        transport = HttpTransport(base_url=url)
        client = Client(transport)
        await client.initialize()
        clients[server_name] = client
        all_tools[server_name] = await client.list_tools()
    
    # Create specialized agents
    customer_agent = AssistantAgent(
        name="CustomerSpecialist",
        llm_config={"model": "gpt-4"}
    )
    
    financial_agent = AssistantAgent(
        name="FinancialSpecialist",
        llm_config={"model": "gpt-4"}
    )
    
    research_agent = AssistantAgent(
        name="ResearchSpecialist",
        llm_config={"model": "gpt-4"}
    )
    
    # Register tools per agent
    def create_tool_wrapper(server_name, tool_name):
        def tool_func(**kwargs):
            import asyncio
            loop = asyncio.new_event_loop()
            result = loop.run_until_complete(
                clients[server_name].call_tool(tool_name, kwargs)
            )
            return result
        return tool_func
    
    # Customer agent gets customer tools
    for tool in all_tools["customer"]:
        customer_agent.register_function(
            function=create_tool_wrapper("customer", tool.name),
            description=tool.description
        )
    
    # Financial agent gets banking tools
    for tool in all_tools["banking"]:
        financial_agent.register_function(
            function=create_tool_wrapper("banking", tool.name),
            description=tool.description
        )
    
    # Research agent gets document tools
    for tool in all_tools["documents"]:
        research_agent.register_function(
            function=create_tool_wrapper("documents", tool.name),
            description=tool.description
        )
    
    # Create group chat
    groupchat = GroupChat(
        agents=[customer_agent, financial_agent, research_agent],
        messages=[],
        max_round=10
    )
    
    manager = GroupChatManager(groupchat=groupchat)
    
    user_proxy = UserProxyAgent(name="User")
    
    # Initiate group conversation
    user_proxy.initiate_chat(
        manager,
        message="Analyze customer 123: Get their info, check balance, and find relevant documents"
    )
    
    # Cleanup
    for client in clients.values():
        await client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

## AutoGen Agent Conversation Flow

```
User: "Get customer 123 and check their balance"
   ↓
UserProxyAgent: Sends message to agents
   ↓
CustomerSpecialist: "I'll get the customer info"
   ├─ Calls MCP tool: get_customer(123)
   └─ Reports: "Customer is Alice, active"
   ↓
FinancialSpecialist: "I'll check the balance"
   ├─ Calls MCP tool: check_balance(123)
   └─ Reports: "Balance is $5000"
   ↓
GroupChatManager: "Summary: Alice's account is active with $5000"
   ↓
User: Sees result
```

## AutoGen vs LangChain vs CrewAI

| Aspect | AutoGen | LangChain | CrewAI |
|--------|---------|-----------|--------|
| **Model** | Multi-agent conversation | Single/multi-agent with tools | Multi-agent with tasks |
| **Communication** | Messages between agents | Tool use + reasoning | Task-based workflow |
| **Tool Invocation** | Agents call tools | Agent selects tools | Tasks assign tools |
| **MCP Pattern** | Adapter registers tools | Adapter wraps clients | Adapter distributes tools |
| **Scaling** | Conversation complexity | Parallel tool calls | Task dependencies |

## Nested Conversations in AutoGen + MCP

```python
# Agents can have sub-conversations
def sub_conversation_task(agents, initial_message):
    """Run a sub-conversation between agents"""
    group = GroupChat(agents=agents, messages=[], max_round=5)
    manager = GroupChatManager(groupchat=group)
    
    proxy = UserProxyAgent(name="Initiator")
    proxy.initiate_chat(manager, message=initial_message)

# Main conversation uses results from sub-conversations
@main_agent.tool()
def analyze_customer(customer_id):
    # Can trigger sub-conversation
    result = sub_conversation_task(
        [customer_agent, financial_agent],
        f"Analyze customer {customer_id}"
    )
    return result
```

## Error Handling in AutoGen + MCP

```python
def create_safe_tool_wrapper(server_name, tool_name):
    def tool_func(**kwargs):
        try:
            import asyncio
            loop = asyncio.new_event_loop()
            result = loop.run_until_complete(
                clients[server_name].call_tool(tool_name, kwargs)
            )
            return f"Success: {result}"
        except Exception as e:
            return f"Error: {str(e)}"
    return tool_func
```

## Interview Summary

**Key Concepts:**
- **Agent Conversation:** Agents exchange messages
- **Tool Registration:** MCP tools registered as callable functions
- **Group Chat:** Multiple agents with GroupChatManager
- **MCP Adapter:** Manages clients and tool registration
- **Multi-Server:** Tools from multiple servers assigned to agents
- **Nested Conversations:** Complex multi-level workflows

**One-Liner:**
"AutoGen + MCP integrates dynamically discovered MCP server tools into AutoGen's agent conversation model, enabling multi-agent systems to access shared capabilities."

## Common Interview Questions

**Q1: How does AutoGen use MCP differently from LangChain?**
A: LangChain uses one agent with all tools. AutoGen creates conversations between multiple agents, each calling MCP tools. AutoGen's focus is agent-to-agent communication, not single-agent tool use.

**Q2: In AutoGen, who decides which tool to call?**
A: The agent (via LLM reasoning). The agent sees available tools and decides which to use based on conversation context.

**Q3: Can AutoGen agents use tools from multiple MCP servers?**
A: Yes. The adapter can register tools from all servers to all agents, or distribute them selectively (customer tools to customer agent, financial tools to financial agent).

**Q4: What's a GroupChat in AutoGen + MCP?**
A: A GroupChat orchestrates communication between multiple agents. Each agent can use MCP tools. The GroupChatManager decides turn order and message flow.

**Q5: How do you handle tool errors in AutoGen?**
A: Wrap tool calls in try/except. Return error messages to the agent. The agent can then decide to retry, use a different tool, or report the error.

---

**Next:** [12_FastMCP_vs_Official_SDK.md](12_FastMCP_vs_Official_SDK.md) – Understanding FastMCP
