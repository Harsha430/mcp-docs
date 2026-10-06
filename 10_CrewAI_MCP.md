# 10: CrewAI + MCP Integration

**Version/Source Basis:** MCP Protocol as of July 28, 2026. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

## CrewAI + MCP Architecture

### How They Connect

```
┌────────────────────────────────────────┐
│  CrewAI Crew (Multi-agent system)      │
│  - Agent 1: Analyst                    │
│  - Agent 2: Coordinator                │
│  - Agent 3: Reporter                   │
└────────────────────────────────────────┘
           │
           │ Crew uses
           ▼
┌────────────────────────────────────────┐
│  CrewAI MCP Integration/Adapter        │
│  - Manages MCP clients                 │
│  - Exposes tools to agents             │
│  - Translates CrewAI ↔ MCP             │
└────────────────────────────────────────┘
           │
           │ Uses
           ▼
┌────────────────────────────────────────┐
│  MCP Clients (One per server)          │
│  - Customer client                     │
│  - Banking client                      │
│  - Documents client                    │
└────────────────────────────────────────┘
           │
           │ Connect to
           ▼
┌────────────────────────────────────────┐
│  MCP Servers                           │
│  - Customer Server                     │
│  - Banking Server                      │
│  - Document Server                     │
└────────────────────────────────────────┘
```

### Key Difference from LangChain

**LangChain:** One agent, multiple tools
**CrewAI:** Multiple agents, each with tools

```python
# LangChain
agent = Agent(tools=[get_customer, transfer_money, search_docs])

# CrewAI
crew = Crew(
    agents=[
        Agent(name="Analyst", tools=[get_customer, check_balance]),
        Agent(name="Processor", tools=[transfer_money]),
        Agent(name="Reporter", tools=[search_docs, generate_report])
    ]
)
```

## CrewAI + MCP Example: Single Server

### Server Setup

```python
# customer_server.py
from mcp.server import Server
from mcp.transport import HttpTransport
import asyncio

server = Server("customer-service")

@server.tool()
def get_customer(customer_id: str) -> str:
    """Get customer info"""
    return f"Customer {customer_id}: Alice, active"

@server.tool()
def update_customer_status(customer_id: str, status: str) -> str:
    """Update customer status"""
    return f"Updated customer {customer_id} to {status}"

async def main():
    transport = HttpTransport(host="0.0.0.0", port=8001)
    await server.run(transport)

if __name__ == "__main__":
    asyncio.run(main())
```

### CrewAI Application

```python
import asyncio
from mcp.client import Client
from mcp.transport import HttpTransport
from crewai import Agent, Task, Crew, LLM
from crewai.tools import Tool

async def main():
    # 1. Connect to MCP server
    transport = HttpTransport(base_url="http://localhost:8001")
    mcp_client = Client(transport)
    await mcp_client.initialize()
    
    # 2. Get MCP tools
    mcp_tools_list = await mcp_client.list_tools()
    
    # 3. Convert to CrewAI tools
    crewai_tools = []
    for mcp_tool in mcp_tools_list:
        def create_tool(tool_name):
            def tool_func(**kwargs):
                import asyncio
                loop = asyncio.get_event_loop()
                return loop.run_until_complete(
                    mcp_client.call_tool(tool_name, kwargs)
                )
            return tool_func
        
        crewai_tool = Tool(
            name=mcp_tool.name,
            description=mcp_tool.description,
            func=create_tool(mcp_tool.name)
        )
        crewai_tools.append(crewai_tool)
    
    # 4. Create CrewAI agents with MCP tools
    analyst = Agent(
        role="Customer Analyst",
        goal="Analyze customer data",
        tools=crewai_tools,
        llm=LLM(model="gpt-4")
    )
    
    processor = Agent(
        role="Status Updater",
        goal="Update customer status",
        tools=crewai_tools,
        llm=LLM(model="gpt-4")
    )
    
    # 5. Create tasks
    analyze_task = Task(
        description="Get customer 123 info and analyze their status",
        agent=analyst,
        expected_output="Customer analysis"
    )
    
    update_task = Task(
        description="Update customer 123 status to premium",
        agent=processor,
        expected_output="Confirmation of update"
    )
    
    # 6. Create crew
    crew = Crew(
        agents=[analyst, processor],
        tasks=[analyze_task, update_task],
        verbose=True
    )
    
    # 7. Execute
    result = crew.kickoff()
    print(f"Result: {result}")
    
    await mcp_client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

## CrewAI + Multiple MCP Servers

### Server Setup (Same as LangChain Example)

Three servers running on ports 8001, 8002, 8003:
- Customer Server (get_customer, create_customer)
- Banking Server (transfer_money, check_balance)
- Document Server (search_documents, read_resource)

### CrewAI Multi-Server Application

```python
import asyncio
from mcp.client import Client
from mcp.transport import HttpTransport
from crewai import Agent, Task, Crew, LLM
from crewai.tools import Tool

async def main():
    # 1. Connect to all servers
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
    
    # 2. Collect tools from all servers
    all_tools = {}
    for server_name, client in clients.items():
        tools = await client.list_tools()
        all_tools[server_name] = tools
    
    # 3. Create tool wrappers
    def create_tool_wrapper(server_name, tool_name, client):
        def tool_func(**kwargs):
            import asyncio
            loop = asyncio.get_event_loop()
            return loop.run_until_complete(
                client.call_tool(tool_name, kwargs)
            )
        return tool_func
    
    crewai_tools_by_server = {}
    for server_name, tools in all_tools.items():
        crewai_tools_by_server[server_name] = []
        for tool in tools:
            crewai_tool = Tool(
                name=f"{server_name}_{tool.name}",
                description=tool.description,
                func=create_tool_wrapper(
                    server_name,
                    tool.name,
                    clients[server_name]
                )
            )
            crewai_tools_by_server[server_name].append(crewai_tool)
    
    # 4. Create specialized agents
    customer_agent = Agent(
        role="Customer Specialist",
        goal="Handle customer-related queries",
        tools=crewai_tools_by_server["customer"],
        llm=LLM(model="gpt-4")
    )
    
    financial_agent = Agent(
        role="Financial Analyst",
        goal="Handle banking and financial operations",
        tools=crewai_tools_by_server["banking"],
        llm=LLM(model="gpt-4")
    )
    
    research_agent = Agent(
        role="Research Specialist",
        goal="Find and analyze documents",
        tools=crewai_tools_by_server["documents"],
        llm=LLM(model="gpt-4")
    )
    
    coordinator_agent = Agent(
        role="Coordinator",
        goal="Coordinate across teams",
        # Coordinator doesn't have tools; delegates to specialists
        llm=LLM(model="gpt-4")
    )
    
    # 5. Create tasks
    task1 = Task(
        description="Get customer 123 information",
        agent=customer_agent,
        expected_output="Customer details"
    )
    
    task2 = Task(
        description="Check customer 123's balance and propose transfer",
        agent=financial_agent,
        expected_output="Balance check and transfer proposal"
    )
    
    task3 = Task(
        description="Search for customer policies and documents",
        agent=research_agent,
        expected_output="List of relevant documents"
    )
    
    task4 = Task(
        description="Summarize findings from all teams",
        agent=coordinator_agent,
        expected_output="Executive summary"
    )
    
    # 6. Create crew
    crew = Crew(
        agents=[
            customer_agent,
            financial_agent,
            research_agent,
            coordinator_agent
        ],
        tasks=[task1, task2, task3, task4],
        verbose=True
    )
    
    # 7. Execute
    result = crew.kickoff(
        inputs={
            "customer_id": "123",
            "operation": "analyze_and_propose_transfer"
        }
    )
    print(f"Result: {result}")
    
    # Cleanup
    for client in clients.values():
        await client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

## Architecture Comparison: LangChain vs CrewAI + MCP

| Aspect | LangChain | CrewAI |
|--------|-----------|--------|
| **Structure** | One agent with many tools | Multiple agents, each with tools |
| **Tool Distribution** | All tools available to agent | Tools assigned per agent |
| **Workflow** | Linear or branching | Sequential tasks with agent transitions |
| **Collaboration** | LLM orchestrates | Task-based orchestration |
| **Specialization** | One LLM makes all decisions | Each agent can specialize |
| **MCP Integration** | Adapter wraps clients | Adapter distributes tools to agents |

## CrewAI Workflow with MCP

```
1. Crew initialized
   ↓
2. Each agent receives assigned MCP tools
   ↓
3. Task 1 → Agent 1 (customer agent) uses customer tools
   ↓
4. Task 2 → Agent 2 (financial agent) uses banking tools
   ↓
5. Task 3 → Agent 3 (research agent) uses document tools
   ↓
6. Task 4 → Agent 4 (coordinator) summarizes
   ↓
7. Result to user
```

## Context Passing Between Agents

Agents in CrewAI can pass context:

```python
task1_result = await customer_agent.execute(task1)

task2 = Task(
    description=f"Based on this customer: {task1_result}, check balance",
    agent=financial_agent,
    expected_output="Balance confirmation"
)
```

## Error Handling in CrewAI + MCP

```python
try:
    result = crew.kickoff()
except Exception as e:
    print(f"Crew error: {e}")
    # Handle errors:
    # - MCP connection error
    # - Tool execution error
    # - Agent decision error
    # - Task failure
```

## Interview Summary

**Key Concepts:**
- **Multi-Agent System:** CrewAI manages multiple specialized agents
- **Task-Based:** Tasks flow between agents
- **Tool Assignment:** Each agent gets specific MCP tools
- **Context Passing:** Agents communicate via task output
- **MCP Integration:** Adapter manages clients, distributes tools
- **Shared Servers:** All agents access same MCP servers

**One-Liner:**
"CrewAI + MCP connects multiple specialized agents to dynamically discovered tools from shared MCP servers, orchestrating complex workflows across teams."

## Common Interview Questions

**Q1: How is CrewAI different from LangChain when using MCP?**
A: LangChain uses one agent with all tools. CrewAI uses multiple agents, each specialized with specific MCP tools. CrewAI orchestrates workflows as tasks flowing between agents.

**Q2: Can agents in CrewAI use tools from multiple MCP servers?**
A: Yes. The adapter can assign tools from different servers to different agents, or assign multiple servers' tools to one agent based on the design.

**Q3: How do agents in CrewAI communicate?**
A: Through task output. One task's output becomes context for the next task. Agents don't directly call each other; the crew orchestrates the flow.

**Q4: If an MCP server fails, what happens?**
A: The agent trying to use that tool gets an error. The crew can handle it gracefully (retry, skip, or fail the task depending on error handling configuration).

**Q5: How do you scale CrewAI + MCP?**
A: Add more agents with specialized tools, add more MCP servers, configure task dependencies wisely. Each MCP server independently scales.

---

**Next:** [11_AutoGen_MCP.md](11_AutoGen_MCP.md) – AutoGen + MCP integration
