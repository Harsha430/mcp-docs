# MCP Production, Evaluation & Interviews: System Design & Deep Dives

**Document Scope:**
- Production deployment patterns
- Reliability & observability
- Evaluation frameworks
- 150+ Interview questions
- System design scenarios
- Real-world architectures

---

## PART 1: MCP + LangChain Integration

### Architecture

```
LangChain Agent
    ├─ Reasoning engine
    ├─ Tool selection logic
    │
    ├─ Tool Executor
    │  ├─ Converts LangChain tool calls
    │  └─ Calls MCP Client
    │
    ├─ MCP Client adapter
    │  ├─ Connects to MCP Server
    │  └─ Translates LangChain ↔ MCP
    │
    └─ MCP Servers
       ├─ Server A (tools)
       ├─ Server B (resources)
       └─ Server C (prompts)
```

### Integration Flow

```
1. LangChain agent receives query
   ↓
2. Agent inspects available tools
   (These come from MCP servers)
   ↓
3. Agent selects: "Use tool X"
   ↓
4. LangChain Tool Executor
   receives tool selection
   ↓
5. LangChain MCP adapter
   calls: client.call_tool("X", args)
   ↓
6. MCP Client talks to MCP Server
   ↓
7. Server executes tool
   ↓
8. MCP Client returns result
   ↓
9. LangChain Tool Executor
   processes result
   ↓
10. Agent receives result
    ↓
11. Agent continues reasoning
```

### LangChain Code Example

```python
from langchain.agents import AgentExecutor
from langchain_mcp import MCPToolkit

# Create MCP toolkit
mcp_toolkit = MCPToolkit(
    servers={
        "github": {
            "command": "python",
            "args": ["/path/to/github_server.py"]
        }
    }
)

# Get tools from MCP
tools = mcp_toolkit.get_tools()

# Create agent
agent = create_tool_calling_agent(
    llm=model,
    tools=tools,
    ...
)

# Execute
result = await agent.invoke({"input": "Create a GitHub issue..."})
```

---

## PART 2: Multi-Server Composition

### Architecture

```
Host Application
    │
    ├─ MCP Client 1 → GitHub Server
    │                  ├─ create_issue
    │                  ├─ list_issues
    │                  └─ get_issue
    │
    ├─ MCP Client 2 → Database Server
    │                  ├─ query_db
    │                  ├─ insert
    │                  └─ update
    │
    ├─ MCP Client 3 → Filesystem Server
    │                  ├─ read_file
    │                  ├─ write_file
    │                  └─ list_files
    │
    └─ MCP Client 4 → Search Server
                       ├─ web_search
                       ├─ docs_search
                       └─ image_search
```

### Aggregation Pattern

```
Agent receives query:
"Find the latest GitHub issues and compare
 them with project documentation"

Flow:
├─ Client 1 lists GitHub issues
│  └─ returns [Issue1, Issue2, ...]
│
├─ Client 3 reads project docs
│  └─ returns [Doc content]
│
└─ Agent reasons:
   "Issue1 is about database migration,
    docs show we use PostgreSQL 15,
    this aligns."
```

### Namespace & Naming Conflicts

**Problem:**
```
GitHub Server has: create_issue
Jira Server has:   create_issue
```

**Solutions:**

1. **Prefixing**
   ```
   github_create_issue
   jira_create_issue
   ```

2. **Server grouping**
   ```
   github.create_issue
   jira.create_issue
   ```

3. **Unique naming**
   ```
   create_github_issue
   create_jira_issue
   ```

### Server Isolation

Each MCP server is independent:

```
If Server A fails:
  └─ Client A reports error
  └─ Client B, C, D continue working
  └─ Agent can fall back to alternatives

Example:
  GitHub server down
  └─ Agent can't list issues
  └─ But can still query database
  └─ And can search docs
```

---

## PART 3: Reliability Patterns

### Failure Taxonomy

```
Layer                  Failures
┌─────────────────────────────────┐
│ LLM/Agent            │ Timeout, error
│ MCP Client           │ Connection, protocol
│ Transport            │ Network, stdio pipe
│ MCP Server           │ Crash, hang
│ Tool Execution       │ Exception, resource
│ External System      │ API error, auth
└─────────────────────────────────┘
```

### Retry Strategy

**Transient failures (retry):**
- Network timeout
- Temporary server unavailable
- Rate limit (with backoff)

**Permanent failures (don't retry):**
- Invalid authentication
- Resource not found (404)
- Invalid parameters
- Authorization denied (403)

### Retry Implementation

```python
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
async def call_tool_with_retry(client, tool_name, args):
    try:
        return await client.call_tool(tool_name, args)
    except TransientError:
        raise  # Retry
    except PermanentError:
        return error_response()  # Don't retry
```

### Fallback Strategy

```
Primary: GitHub Server
  └─ Fails
  └─ Fallback: Cached GitHub data
     └─ Fails
     └─ Fallback: Local mirror
        └─ Fails
        └─ Return: "Unavailable"
```

### Timeout Management

```
Tool call flow:
├─ Client timeout: 30s
│  └─ Aborts if server slow
│
├─ Server timeout: 25s
│  └─ Server aborts long operations
│
└─ External API timeout: 20s
   └─ API call aborts
```

---

## PART 4: Observability & Monitoring

### Three Pillars: Logs, Metrics, Traces

#### Logs

```python
import logging
from fastmcp import Context

@mcp.tool()
async def search(query: str, context: Context) -> list:
    context.log(f"[SEARCH] Query: {query}")
    
    try:
        results = await db.search(query)
        context.log(f"[SEARCH] Found {len(results)} results")
        return results
    except Exception as e:
        context.log(f"[ERROR] Search failed: {str(e)}")
        raise
```

**Logs capture:**
- Request inputs
- Execution steps
- Errors and exceptions
- Timing information

#### Metrics

```python
import time
from prometheus_client import Counter, Histogram

tool_calls = Counter('mcp_tool_calls', 'MCP tool calls', ['tool_name'])
tool_duration = Histogram('mcp_tool_duration_seconds', 'Duration', ['tool_name'])
tool_errors = Counter('mcp_tool_errors', 'Errors', ['tool_name', 'error_type'])

@mcp.tool()
async def my_tool(arg: str) -> str:
    start = time.time()
    tool_calls.labels(tool_name='my_tool').inc()
    
    try:
        result = do_work(arg)
        duration = time.time() - start
        tool_duration.labels(tool_name='my_tool').observe(duration)
        return result
    except Exception as e:
        tool_errors.labels(tool_name='my_tool', error_type=type(e).__name__).inc()
        raise
```

**Metrics track:**
- Request counts
- Latency (p50, p95, p99)
- Error rates
- Resource usage

#### Traces

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

@mcp.tool()
async def my_tool(arg: str) -> str:
    with tracer.start_as_current_span("my_tool") as span:
        span.set_attribute("arg", arg)
        
        with tracer.start_as_current_span("prepare") as inner:
            prepared = prepare(arg)
        
        with tracer.start_as_current_span("execute") as inner:
            result = execute(prepared)
        
        span.set_attribute("result", str(result))
        return result
```

**Traces show:**
- Complete request path
- Latency at each step
- Dependencies between operations
- Error locations

### Key Metrics

| Metric | Type | Alert Threshold |
|--------|------|-----------------|
| **Tool call rate** | Counter | Track usage |
| **Tool success rate** | Percentage | < 95% alert |
| **Tool latency (p95)** | Histogram | > 5s alert |
| **Tool latency (p99)** | Histogram | > 10s alert |
| **Error rate by type** | Counter | Any > 1% alert |
| **Server availability** | Uptime % | < 99.5% alert |
| **Resource usage (CPU)** | Gauge | > 80% alert |
| **Resource usage (Memory)** | Gauge | > 85% alert |

---

## PART 5: Deployment Patterns

### Docker Deployment

**Dockerfile:**
```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "-m", "mcp.server", "--stdio"]
```

**Compose for multi-server:**
```yaml
services:
  knowledge-server:
    build: ./servers/knowledge
    environment:
      - DATABASE_URL=${DB_URL}
      - LOG_LEVEL=INFO
    ports:
      - "8000:8000"
  
  search-server:
    build: ./servers/search
    environment:
      - SEARCH_API_KEY=${SEARCH_KEY}
    ports:
      - "8001:8001"
  
  agent:
    build: ./agent
    depends_on:
      - knowledge-server
      - search-server
    environment:
      - KNOWLEDGE_URL=http://knowledge-server:8000/mcp
      - SEARCH_URL=http://search-server:8001/mcp
```

### Environment Variables

```bash
# secrets.env (not in git)
MCP_AUTH_TOKEN=secret-token-here
DATABASE_URL=postgresql://...
API_KEY_GITHUB=ghp_...

# Deployment
docker run \
  --env-file secrets.env \
  -p 8000:8000 \
  mcp-server:latest
```

### Health Checks

```python
@mcp.tool()
async def health_check() -> dict:
    """Internal tool for deployment health checks."""
    return {
        "status": "healthy",
        "timestamp": datetime.now().isoformat(),
        "dependencies": {
            "database": await check_db(),
            "external_api": await check_api(),
            "cache": await check_cache()
        }
    }
```

**Kubernetes probe:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mcp-server
spec:
  containers:
  - name: server
    image: mcp-server:latest
    livenessProbe:
      httpGet:
        path: /health
        port: 8000
      initialDelaySeconds: 10
      periodSeconds: 30
    readinessProbe:
      httpGet:
        path: /ready
        port: 8000
      initialDelaySeconds: 5
      periodSeconds: 10
```

---

## PART 6: Versioning & Backwards Compatibility

### Semantic Versioning for MCP

```
Version: MAJOR.MINOR.PATCH
         (MCP spec).(Server spec).(Bug fix)

1.0.0 → 1.1.0 (new tool added) = Minor
1.0.0 → 1.0.1 (bug fix) = Patch
1.0.0 → 2.0.0 (tool removed) = Major
```

### Breaking Changes

**Adding a tool:**
```python
# Version 1.0.0
@mcp.tool()
def search(query: str) -> list:
    pass

# Version 1.1.0 (non-breaking)
@mcp.tool()
def search(query: str, limit: int = 10) -> list:  # Added optional param
    pass
```

**Changing a tool schema:**
```python
# Version 1.0.0
def search(query: str) -> list:
    pass

# Version 2.0.0 (breaking)
def search(query: str, filters: dict) -> dict:  # Changed return type
    pass
```

### Backwards Compatibility

**Keep old tool, add new:**
```python
# v1 (deprecated)
@mcp.tool()
def search_v1(query: str) -> list:
    """DEPRECATED: Use search_v2 instead."""
    return search_v2(query)

# v2 (current)
@mcp.tool()
def search_v2(query: str, filters: dict = None) -> dict:
    """Improved search with filters."""
    pass
```

---

## PART 7: Security Deep Dive

### Authentication

**Token-based:**
```python
from fastmcp import Context

@mcp.tool()
async def delete_database(database_id: str, context: Context) -> dict:
    """Delete a database (dangerous operation)."""
    # Get token from context
    token = context.auth_token  # Set by server from request
    
    # Verify token
    user = await verify_token(token)
    if not user:
        raise AuthenticationError("Invalid token")
    
    # Check authorization
    if not user.can_delete_database:
        raise AuthorizationError("User cannot delete databases")
    
    # Execute
    await db.delete(database_id)
    return {"deleted": True}
```

### Prompt Injection Attack

**Vulnerable:**
```python
@mcp.resource()
def get_document(doc_id: str) -> str:
    """Get document content."""
    content = read_file(f"/docs/{doc_id}")
    # If doc_id comes from user input:
    # doc_id = "../../../etc/passwd" → LFI
    return content
```

**Safe:**
```python
@mcp.resource("document://{id}")
def get_document(id: str) -> str:
    """Get document content."""
    # Validate doc_id format
    if not re.match(r"^[a-zA-Z0-9_-]+$", id):
        raise ValueError(f"Invalid document ID: {id}")
    
    content = read_file(f"/docs/{id}")
    return content
```

### Input Validation

```python
from pydantic import BaseModel, validator

class DeleteUserRequest(BaseModel):
    user_id: int
    confirm: bool = False
    
    @validator('user_id')
    def user_id_positive(cls, v):
        if v <= 0:
            raise ValueError('user_id must be positive')
        return v

@mcp.tool()
async def delete_user(request: DeleteUserRequest) -> dict:
    """Delete a user."""
    if not request.confirm:
        raise ValueError("Must confirm deletion")
    
    await db.delete_user(request.user_id)
    return {"deleted": True}
```

### PII Detection

```python
import re
from presidio_analyzer import AnalyzerEngine

analyzer = AnalyzerEngine()

@mcp.resource()
def get_user(user_id: int) -> str:
    """Get user details."""
    user_data = fetch_user(user_id)
    
    # Detect PII
    pii_results = analyzer.analyze(text=str(user_data))
    
    if pii_results:
        # Log detection
        logging.warning(f"PII detected in response: {pii_results}")
        
        # Redact or anonymize
        user_data = anonymize_pii(user_data)
    
    return str(user_data)
```

---

## PART 8: Enterprise Architecture

### Multi-Tenant Architecture

```
Enterprise Host
    │
    ├─ MCP Client (shared)
    │   ├─ multi-tenant token management
    │   └─ request context (tenant ID, user ID)
    │
    ├─ MCP Server A (multi-tenant)
    │   ├─ tenant isolation at tool level
    │   ├─ tenant-scoped resources
    │   └─ tenant-scoped prompts
    │
    ├─ MCP Server B (single-tenant per deployment)
    │   └─ separate instance per tenant
    │
    └─ MCP Server C (multi-tenant)
        └─ database-level isolation
```

### Policy Enforcement

```
Policy Layer
    ├─ Tool access policy
    │  └─ Who can call what tool
    │
    ├─ Resource access policy
    │  └─ Who can read what resource
    │
    ├─ Data residency policy
    │  └─ Where data can be stored/processed
    │
    └─ Rate limit policy
       └─ How many calls per user/org
```

### Compliance

**GDPR:**
```python
@mcp.tool()
async def get_customer_data(customer_id: int, context: Context) -> dict:
    """Get customer data."""
    user = context.user
    
    # Check consent
    if not customer.has_consent_for(user.org):
        raise PermissionError("Insufficient consent")
    
    # Get data
    data = await db.get_customer(customer_id)
    
    # Add metadata for GDPR
    data['_extracted_at'] = datetime.now()
    data['_requested_by'] = user.id
    
    return data
```

---

## PART 9: Testing Strategy

### Unit Tests

```python
import pytest
from my_server import multiply

def test_multiply_positive():
    assert multiply(3, 4) == 12

def test_multiply_negative():
    assert multiply(-3, 4) == -12

def test_multiply_zero():
    assert multiply(0, 5) == 0

def test_multiply_large():
    assert multiply(999999, 999999) == 999999000001
```

### Integration Tests

```python
import pytest
from mcp.client.stdio import StdioClientTransport
from mcp.client import ClientSession

@pytest.mark.asyncio
async def test_multiply_via_mcp():
    transport = StdioClientTransport("python", "server.py")
    async with ClientSession(transport) as client:
        await client.initialize()
        
        result = await client.call_tool("multiply", {"a": 3, "b": 4})
        
        assert result.isError is False
        assert "12" in result.content[0].text
```

### End-to-End Tests

```python
@pytest.mark.asyncio
async def test_agent_with_mcp():
    # Setup
    agent = MyAgent()
    
    # Execute
    result = await agent.run("Multiply 3 and 4")
    
    # Assert
    assert "12" in result
```

### Test Matrix

```
┌─────────────────────────────────────┐
│ Test Dimension                      │
├─────────────────────────────────────┤
│ Tool                                │
│  ├─ Valid inputs                    │
│  ├─ Invalid types                   │
│  ├─ Missing required params          │
│  ├─ Extra params                     │
│  ├─ Boundary values                  │
│  ├─ Timeout behavior                 │
│  ├─ Error handling                   │
│  └─ Permission checks                │
│                                      │
│ Resource                            │
│  ├─ Valid URIs                      │
│  ├─ Invalid URIs                    │
│  ├─ Non-existent resources          │
│  ├─ Permission checks                │
│  ├─ Cache behavior                  │
│  └─ Error handling                   │
│                                      │
│ Prompt                              │
│  ├─ Valid arguments                 │
│  ├─ Missing arguments               │
│  ├─ Invalid argument types          │
│  └─ Output validation                │
│                                      │
│ Protocol                            │
│  ├─ Initialize handshake            │
│  ├─ Capability negotiation          │
│  ├─ Message serialization           │
│  ├─ Error responses                 │
│  └─ Notifications                    │
│                                      │
│ Transport                           │
│  ├─ Connection success              │
│  ├─ Connection failure              │
│  ├─ Reconnection                    │
│  └─ Timeout handling                 │
└─────────────────────────────────────┘
```

---

## PART 10: Evaluation Framework

### Layer 1: Protocol Evaluation

```
Tests:
  ✓ Connection established
  ✓ InitializeRequest sent
  ✓ InitializeResponse valid
  ✓ Capabilities received
  ✓ Initialization confirmed

Metrics:
  • Connection success rate
  • Initialization time
  • Protocol compliance
  • Schema validation
```

### Layer 2: Tool Evaluation

```
Tests:
  ✓ ListTools returns all tools
  ✓ Tool schemas are valid JSON Schema
  ✓ Tool descriptions are present
  ✓ Required parameters specified
  ✓ Tool can be executed
  ✓ Invalid inputs rejected
  ✓ Results are well-formed
  ✓ Errors are handled gracefully

Metrics:
  • Tool count
  • Schema compliance
  • Execution success rate
  • Latency (p50, p95, p99)
  • Error rate
  • Tool coverage
```

### Layer 3: Agent Evaluation

```
Tests:
  ✓ Agent discovers tools
  ✓ Agent selects appropriate tool
  ✓ Agent calls tool correctly
  ✓ Agent processes results
  ✓ Agent recovers from errors
  ✓ Agent reaches correct conclusion

Metrics:
  • Tool selection accuracy
  • Argument correctness
  • Trajectory quality
  • Final answer accuracy
  • Error recovery rate
  • Token efficiency
```

### Evaluation Dataset

```python
test_cases = [
    {
        "input": "Multiply 3 and 4",
        "expected_tool": "multiply",
        "expected_args": {"a": 3, "b": 4},
        "expected_result": "12"
    },
    {
        "input": "Find user John in database",
        "expected_tool": "search_db",
        "expected_args": {"query": "user John"},
        "expected_result": contains("id", "name", "email")
    },
    {
        "input": "What's ACME's mission?",
        "expected_tool": "read_resource",
        "expected_args": {"uri": "company://acme"},
        "expected_result": contains("mission", "ACME")
    }
]

for test_case in test_cases:
    result = await agent.run(test_case["input"])
    
    if matches(result, test_case["expected_result"]):
        print("✓ PASS")
    else:
        print("✗ FAIL", result)
```

---

## PART 11: Interview Questions (150+)

### SECTION 1: Fundamentals (30 questions)

**Q1: Define MCP in one sentence.**  
**A:** Model Context Protocol is a standardized protocol enabling AI applications to discover and use external capabilities (tools, resources, prompts) through a common interface.  
**Keywords:** Protocol, standardized, discovery, external capabilities  
**Follow-up:** How does standardization solve the integration problem?

**Q2: Why was MCP created? Name the main problem it solves.**  
**A:** Before MCP, each AI application had to build custom integrations with each external system (n×m problem). MCP introduces a single standard protocol.  
**Keywords:** Integration problem, custom adapters, standardization  
**Follow-up:** Can you describe a real-world scenario without MCP?

**Q3: What are the three core MCP primitives?**  
**A:** Tools (actions), Resources (read-only data), Prompts (instruction templates).  
**Keywords:** Tool, Resource, Prompt  
**Follow-up:** When would you use each?

**Q4: What is the difference between a Tool and a Resource?**  
**A:** Tools have side effects and are actively invoked; Resources are read-only and accessed by URI.  
**Keywords:** Side effects, read-only, URI-addressed  
**Follow-up:** What if a Tool doesn't have side effects?

**Q5: What is the relationship between Host and MCP Client?**  
**A:** The Host is the application (contains LLM/agent); the MCP Client is the communication layer within the Host.  
**Keywords:** Host contains Client, communication layer  
**Follow-up:** Can a Host be an MCP Server?

**Q6: Does an LLM directly call an MCP tool?**  
**A:** No. The LLM decides which tool to use; the Host/MCP Client handles communication with the server.  
**Keywords:** Indirection, LLM decision, Host execution  
**Follow-up:** Who decides to retry a failed tool call?

**Q7: What is a Protocol?**  
**A:** A set of rules defining message structure, semantics, sequencing, and error handling.  
**Keywords:** Rules, message structure, semantics  
**Follow-up:** Is HTTP a protocol?

**Q8: What is a Transport?**  
**A:** The physical mechanism for delivering messages (stdio, HTTP, WebSocket, etc.).  
**Keywords:** Physical mechanism, delivery method  
**Follow-up:** Can you change transport without changing protocol?

**Q9: Name two transports that MCP supports.**  
**A:** Stdio (local subprocess) and Streamable HTTP (remote network).  
**Keywords:** Stdio, HTTP, local, remote  
**Follow-up:** What are pros and cons of each?

**Q10: What is JSON-RPC?**  
**A:** A stateless, lightweight RPC protocol using JSON for requests and responses.  
**Keywords:** RPC, JSON, stateless, lightweight  
**Follow-up:** What does the "2.0" in JSON-RPC 2.0 mean?

**Q11: What fields are required in a JSON-RPC request?**  
**A:** jsonrpc ("2.0"), id (request identifier), method (operation name), params (arguments).  
**Keywords:** jsonrpc, id, method, params  
**Follow-up:** Can params be optional?

**Q12: What is the difference between a Response and a Notification in JSON-RPC?**  
**A:** Response has an id (matching request) and result/error; Notification has no id and is unsolicited.  
**Keywords:** ID, result, unsolicited  
**Follow-up:** Can a server send a Notification to client?

**Q13: List the MCP lifecycle phases.**  
**A:** Connect → Initialize → Discover capabilities → Execute tools → Close.  
**Keywords:** Initialization, discovery, execution  
**Follow-up:** What happens if initialization fails?

**Q14: What does "capabilities" mean in MCP?**  
**A:** Declaration of what features/functionality each side supports (e.g., server supports tools, resources).  
**Keywords:** Feature declaration, handshake  
**Follow-up:** Can a client and server have mismatched capabilities?

**Q15: What is the purpose of ListToolsRequest?**  
**A:** Client queries server to discover available tools and their schemas.  
**Keywords:** Discovery, schema, client query  
**Follow-up:** When is ListTools called?

**Q16: Describe the flow from LLM decision to tool execution.**  
**A:** LLM decides → Host receives decision → MCP Client sends CallToolRequest → Server executes → Result returns → Host receives → LLM processes.  
**Keywords:** Flow, indirection, multiple layers  
**Follow-up:** Where can this flow fail?

**Q17: What is a Tool Schema?**  
**A:** JSON Schema describing a tool's input parameters (types, requirements, descriptions).  
**Keywords:** JSON Schema, parameters, types, descriptions  
**Follow-up:** Can schema include constraints like min/max length?

**Q18: What is a Resource URI?**  
**A:** Address scheme for accessing resources (e.g., github://pull/123, file:///path/to/file).  
**Keywords:** Addressing, URI scheme, URI template  
**Follow-up:** What's the difference between static and templated URIs?

**Q19: What is a Prompt in MCP?**  
**A:** Reusable instruction template that guides model behavior, parameterized by arguments.  
**Keywords:** Template, instruction, parameterized  
**Follow-up:** How is Prompt different from System Prompt?

**Q20: What is the purpose of FastMCP?**  
**A:** Python framework for building MCP servers with decorators and automatic schema generation.  
**Keywords:** Framework, decorators, automatic schema  
**Follow-up:** What problem does FastMCP solve?

**Q21: What does @mcp.tool() decorator do?**  
**A:** Registers a Python function as an MCP tool and generates JSON Schema from type hints and docstring.  
**Keywords:** Registration, schema generation, type hints  
**Follow-up:** What if the function has no docstring?

**Q22: How does FastMCP generate schema from Python code?**  
**A:** Uses type annotations (int → integer, str → string) and parses docstring for parameter descriptions.  
**Keywords:** Type annotations, docstring parsing  
**Follow-up:** What about complex types like Pydantic models?

**Q23: What is an MCP Server?**  
**A:** Application that exposes capabilities (tools, resources, prompts) through MCP protocol.  
**Keywords:** Capability exposure, protocol interface  
**Follow-up:** Can an MCP Server also be a Client?

**Q24: What is an MCP Client?**  
**A:** Component within a Host that communicates with MCP Server (discovery, execution, resource fetching).  
**Keywords:** Communication layer, discovery, execution  
**Follow-up:** Is the MCP Client responsible for retries?

**Q25: Explain: "MCP is not an LLM, not an agent, not an API".**  
**A:** MCP is protocol for standardized communication; LLMs and agents are consumers; APIs are what MCP abstracts over.  
**Keywords:** Protocol, abstraction layer  
**Follow-up:** What does MCP abstract away?

**Q26: What problem do Resources solve that Tools don't?**  
**A:** Resources provide read-only context without side effects; Tools execute actions.  
**Keywords:** Read-only context, side effects  
**Follow-up:** Can you use Tool instead of Resource?

**Q27: How many servers can a Host connect to?**  
**A:** As many as needed; typical architectures have 3-10 servers.  
**Keywords:** Multiple servers, composition  
**Follow-up:** How does the agent choose which server to query?

**Q28: What happens if two servers expose tools with the same name?**  
**A:** Naming conflict; resolved by prefixing, namespacing, or unique naming convention.  
**Keywords:** Namespace collision, resolution strategies  
**Follow-up:** Can a client handle this automatically?

**Q29: Explain stdio transport.**  
**A:** Local process-to-process communication; server runs as subprocess; client writes to stdin, reads from stdout.  
**Keywords:** Subprocess, stdin/stdout, local  
**Follow-up:** Why is stderr important?

**Q30: Explain HTTP transport (Streamable HTTP).**  
**A:** Network communication; client POSTs requests, server streams responses via HTTP/SSE.  
**Keywords:** Network, HTTP, SSE, remote  
**Follow-up:** When would you choose HTTP over stdio?

---

### SECTION 2: Protocol Deep Dive (25 questions)

**Q31: What fields are in a Tool definition?**  
**A:** name, description, inputSchema (required for all; also version, deprecated flag if applicable).  
**Keywords:** Metadata, schema, naming  
**Follow-up:** What if description is missing?

**Q32: What is the structure of inputSchema?**  
**A:** JSON Schema object with type, properties (parameter definitions), required array.  
**Keywords:** JSON Schema, properties, required  
**Follow-up:** Can inputSchema define object properties?

**Q33: How does client know if a server supports Resources?**  
**A:** During initialization, server returns capabilities object including {resources: {}}.  
**Keywords:** Capabilities, negotiation, features  
**Follow-up:** What if client doesn't check capabilities?

**Q34: What is a ResourceTemplate?**  
**A:** URI pattern with placeholders (e.g., github://pr/{id}) for discovering dynamic resources.  
**Keywords:** URI pattern, template, dynamic discovery  
**Follow-up:** How are templates matched to requests?

**Q35: How does CallToolRequest differ from ListToolsRequest?**  
**A:** ListTools queries available tools; CallTool executes a specific tool with arguments.  
**Keywords:** Discovery vs execution, parameters  
**Follow-up:** Can you call a tool without listing first?

**Q36: What does ToolResult contain?**  
**A:** content (array of text/image/resource content) and isError (boolean).  
**Keywords:** Content, error flag  
**Follow-up:** How does client know if tool failed?

**Q37: What is a TextContent object?**  
**A:** {type: "text", text: "string content"} - represents text result from tool.  
**Keywords:** Content type, text field  
**Follow-up:** What about binary content?

**Q38: How does client handle malformed JSON from server?**  
**A:** Transport layer detects parse error; JSON-RPC error returned to client.  
**Keywords:** Parse error, transport detection  
**Follow-up:** What happens to the connection?

**Q39: What is the InitializeRequest/Response handshake?**  
**A:** Client sends protocol version and capabilities; Server responds with version and capabilities.  
**Keywords:** Handshake, version negotiation, capabilities  
**Follow-up:** What if versions don't match?

**Q40: Can a client have different capabilities than the server?**  
**A:** Yes; client may support sampling, server may support resources—each declares what it supports.  
**Keywords:** Independent capabilities, feature matching  
**Follow-up:** What if capability is missing on both sides?

**Q41: What is the purpose of Notification?**  
**A:** Server sends unsolicited message to client (no response expected).  
**Keywords:** Unsolicited, one-way, no response  
**Follow-up:** What are common notification types?

**Q42: Name three types of Notifications in MCP 1.28.1.**  
**A:** resources/list_changed, resources/updated, logging/message.  
**Keywords:** Resource updates, logging  
**Follow-up:** Can client send notifications to server?

**Q43: What is a resource/list_changed notification?**  
**A:** Server tells client the resource list has changed; client should call ListResources again.  
**Keywords:** Resource update, re-discovery  
**Follow-up:** Why not send new resources directly?

**Q44: What is resource/updated notification?**  
**A:** Server tells client a specific resource was modified (URI included).  
**Keywords:** Specific resource, update event  
**Follow-up:** Does client automatically re-read the resource?

**Q45: Explain the error response structure.**  
**A:** {jsonrpc, id, error: {code, message, data}}.  
**Keywords:** Error object, code, message, data  
**Follow-up:** What is error.data used for?

**Q46: What error code means "Method not found"?**  
**A:** -32601 (MCP Server received unknown method name).  
**Keywords:** JSON-RPC standard error, invalid method  
**Follow-up:** Should client retry on -32601?

**Q47: What error code means "Invalid params"?**  
**A:** -32602 (arguments don't match schema or are wrong type).  
**Keywords:** Validation error, type mismatch  
**Follow-up:** Should client retry on -32602?

**Q48: How does server respond to tool execution error?**  
**A:** Returns ToolResult with isError=true and error description in content.  
**Keywords:** Tool error, content field, error flag  
**Follow-up:** Can server also return JSON-RPC error instead?

**Q49: What is "resource not found" error?**  
**A:** Client requests ReadResource for non-existent URI; server returns error or empty content.  
**Keywords:** 404-like, URI invalid  
**Follow-up:** Should server crash if resource not found?

**Q50: Explain the complete CallToolRequest message.**  
**A:** {jsonrpc: "2.0", id: X, method: "tools/call", params: {name: "tool", arguments: {...}}}.  
**Keywords:** Complete structure, params, method  
**Follow-up:** Is method always "tools/call"?

**Q51: What is the complete ReadResourceRequest message?**  
**A:** {jsonrpc: "2.0", id: X, method: "resources/read", params: {uri: "..."}}.  
**Keywords:** URI parameter, method name  
**Follow-up:** Can you read multiple resources in one request?

**Q52: Explain the complete GetPromptRequest message.**  
**A:** {jsonrpc: "2.0", id: X, method: "prompts/get", params: {name: "...", arguments: {...}}}.  
**Keywords:** Prompt name, arguments, method  
**Follow-up:** Are arguments required?

**Q53: What is a Prompt message?**  
**A:** {role: "user" or "assistant", content: {type, text}}.  
**Keywords:** Role, content, message structure  
**Follow-up:** How does this differ from LLM messages?

**Q54: Can a tool return both text and image content?**  
**A:** Yes; ToolResult.content is array that can contain multiple content types.  
**Keywords:** Mixed content, array  
**Follow-up:** When would you do this?

**Q55: Explain Pagination in MCP (if supported).**  
**A:** (MCP 1.28.1 does not have built-in pagination; server returns all results).  
**Keywords:** Limitation, full results  
**Follow-up:** What if tool returns 1M results?

---

### SECTION 3: FastMCP & Implementation (30 questions)

**Q56: Write a minimal FastMCP server.**  
**A:**
```python
from fastmcp import FastMCP

mcp = FastMCP("Server")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

if __name__ == "__main__":
    mcp.run()
```
**Keywords:** Decorator, type hints, docstring  
**Follow-up:** What does mcp.run() do?

**Q57: How does @mcp.tool() know the parameter names?**  
**A:** From the function signature; Python's inspect module extracts parameter names and types.  
**Keywords:** Inspect module, signature reflection  
**Follow-up:** What about default values?

**Q58: How does FastMCP generate parameter descriptions?**  
**A:** Parses the function docstring looking for Args section (Google/NumPy format).  
**Keywords:** Docstring parsing, standard format  
**Follow-up:** What if docstring doesn't follow the format?

**Q59: How does FastMCP handle Optional parameters?**  
**A:** Optional[T] parameters are not included in inputSchema "required" array.  
**Keywords:** Required array, optional schema  
**Follow-up:** What about default values?

**Q60: What happens when a tool receives invalid input type?**  
**A:** FastMCP validates against schema; returns error if mismatch.  
**Keywords:** Validation, error response  
**Follow-up:** Can client send type validation error back to LLM?

**Q61: Can a FastMCP tool be async?**  
**A:** Yes; FastMCP supports async tool functions with `async def`.  
**Keywords:** Async support, async def  
**Follow-up:** Do you need await inside the tool?

**Q62: How do you log from a FastMCP tool?**  
**A:** Inject Context parameter; use context.log(message).  
**Keywords:** Context injection, logging  
**Follow-up:** Where do logs appear?

**Q63: Can a tool access request context?**  
**A:** Yes; add `context: Context` parameter; inject automatically by FastMCP.  
**Keywords:** Context injection, dependency injection  
**Follow-up:** What's in Context object?

**Q64: What is a FastMCP Resource?**  
**A:** @mcp.resource() decorated function returning data accessed by URI.  
**Keywords:** Resource decorator, URI-addressed  
**Follow-up:** How do you define resource URI?

**Q65: How does @mcp.resource("document://{id}") work?**  
**A:** Registers templated resource; {id} is placeholder; replaced by actual value in URI.  
**Keywords:** Template, placeholder substitution  
**Follow-up:** What validates the URI format?

**Q66: What should a FastMCP resource return?**  
**A:** String (text content) or bytes (binary).  
**Keywords:** Return type, content  
**Follow-up:** Can resource return JSON?

**Q67: What is a FastMCP Prompt?**  
**A:** @mcp.prompt() decorated function returning instruction template.  
**Keywords:** Prompt decorator, template  
**Follow-up:** What does prompt return?

**Q68: How do you pass arguments to a prompt?**  
**A:** Add parameters to prompt function; client sends in GetPromptRequest.arguments.  
**Keywords:** Parameters, arguments  
**Follow-up:** Are arguments required?

**Q69: Can a prompt return structured messages?**  
**A:** Yes; return dict with system, messages keys or list of message dicts.  
**Keywords:** Message structure, formatting  
**Follow-up:** What format does client expect?

**Q70: How do you start a FastMCP server?**  
**A:** Call mcp.run() which opens stdin for protocol messages.  
**Keywords:** run() method, stdin listening  
**Follow-up:** Can you customize transport?

**Q71: Can FastMCP server handle multiple clients?**  
**A:** No; stdio is one client per process. Multiple clients need separate processes or HTTP transport.  
**Keywords:** Single client, transport limitation  
**Follow-up:** How would you support multi-client?

**Q72: How does FastMCP handle tool errors?**  
**A:** Exception in tool caught; ToolResult returned with isError=true.  
**Keywords:** Exception handling, error flag  
**Follow-up:** How do you customize error messages?

**Q73: Can you add metadata to a tool?**  
**A:** Yes; @mcp.tool(name="...", description="...") or use docstring.  
**Keywords:** Metadata, decorator params  
**Follow-up:** What other metadata can you add?

**Q74: How do you test a FastMCP server without MCP client?**  
**A:** Call tool function directly as Python function.  
**Keywords:** Unit test, direct call  
**Follow-up:** How do you test protocol?

**Q75: Can FastMCP server call other MCP servers?**  
**A:** Yes; server can be host + client; can call other servers.  
**Keywords:** Composition, nested clients  
**Follow-up:** Any limitations?

**Q76: How does FastMCP handle type validation?**  
**A:** Uses schema; validates CallToolRequest.arguments against inputSchema.  
**Keywords:** Schema validation, JSON Schema  
**Follow-up:** Can you customize validation?

**Q77: What is FastMCP Context API?**  
**A:** Provides runtime information (logging, progress, request data) to tools.  
**Keywords:** Context, runtime information  
**Follow-up:** What methods does Context have?

**Q78: How do you report progress in FastMCP?**  
**A:** Use context.set_progress(progress, total) in tool.  
**Keywords:** Progress reporting, context method  
**Follow-up:** Where does progress appear?

**Q79: Can you customize tool execution behavior?**  
**A:** FastMCP hides implementation; customize inside tool function.  
**Keywords:** Implementation detail, tool body  
**Follow-up:** Can you add middleware?

**Q80: How do you handle tool dependencies (tool A calls tool B)?**  
**A:** Tool A receives client from context or imports server module.  
**Keywords:** Dependency injection, composition  
**Follow-up:** Any limitations?

**Q81: Can FastMCP server hot-reload tools?**  
**A:** No; tools are registered at startup; would need server restart.  
**Keywords:** Runtime limitation  
**Follow-up:** How would you support hot-reload?

**Q82: How do you version a FastMCP tool?**  
**A:** Add version to tool name (search_v2) or docstring; maintain backwards compatibility.  
**Keywords:** Versioning strategy, naming  
**Follow-up:** Can clients use old versions?

**Q83: How do you secure a FastMCP tool?**  
**A:** Validate inputs, check authorization, return safe errors, use context for auth info.  
**Keywords:** Security, validation, auth  
**Follow-up:** Where does auth token come from?

**Q84: How do you handle secrets in FastMCP?**  
**A:** Use environment variables; never hardcode; load at startup.  
**Keywords:** Environment variables, secrets  
**Follow-up:** Can you inject secrets via context?

**Q85: Can FastMCP server handle batch requests?**  
**A:** No; each request is independent; JSON-RPC doesn't support batching in MCP 1.28.1.  
**Keywords:** Single request, no batching  
**Follow-up:** How do you optimize multiple calls?

---

### SECTION 4: Debugging & Testing (20 questions)

**Q86: What is MCP Inspector?**  
**A:** Developer tool for testing MCP servers; visual UI for discovery, execution, debugging.  
**Keywords:** Developer tool, UI, testing  
**Follow-up:** How do you start it?

**Q87: What can you do in MCP Inspector?**  
**A:** Connect to server, initialize, list tools/resources/prompts, execute tools, view raw messages.  
**Keywords:** Discovery, execution, debugging  
**Follow-up:** Can you see latency?

**Q88: How do you connect to stdio server in Inspector?**  
**A:** Select stdio; specify command and args (e.g., python server.py).  
**Keywords:** Transport selection, subprocess  
**Follow-up:** How do you connect to HTTP?

**Q89: How do you connect to HTTP server in Inspector?**  
**A:** Select HTTP; enter URL (e.g., http://localhost:8000/mcp).  
**Keywords:** HTTP transport, URL  
**Follow-up:** How do you handle authentication?

**Q90: What does "Initialize" button do in Inspector?**  
**A:** Sends InitializeRequest to server; displays capabilities.  
**Keywords:** Initialization, handshake  
**Follow-up:** What if init fails?

**Q91: How do you execute a tool in Inspector?**  
**A:** Click tool name; fill in arguments; click execute; see result.  
**Keywords:** GUI execution, argument input  
**Follow-up:** How do you see raw messages?

**Q92: What debugging information does Inspector show?**  
**A:** Request/response JSON, timing, errors, capabilities, schemas.  
**Keywords:** Protocol debugging, timing  
**Follow-up:** Can you export logs?

**Q93: Why is Inspector not sufficient for production testing?**  
**A:** Manual only; can't automate, can't run in CI, can't test complex flows.  
**Keywords:** Manual limitation, automation need  
**Follow-up:** What's the alternative?

**Q94: How do you write integration tests for MCP server?**  
**A:** Use mcp.client.StdioClientTransport; connect; initialize; call tools.  
**Keywords:** ClientSession, test harness  
**Follow-up:** How do you assert results?

**Q95: What test categories should you cover?**  
**A:** Tool discovery, tool execution, valid/invalid inputs, edge cases, errors, resources, prompts.  
**Keywords:** Comprehensive coverage  
**Follow-up:** What's the minimum coverage?

**Q96: How do you test tool timeout behavior?**  
**A:** Tool with intentional delay; set client timeout shorter; expect timeout error.  
**Keywords:** Timeout test, timing  
**Follow-up:** How do you set timeout?

**Q97: How do you test tool error handling?**  
**A:** Tool that throws exception; expect ToolResult with isError=true.  
**Keywords:** Exception testing, error flag  
**Follow-up:** How do you validate error message?

**Q98: How do you test resource not found?**  
**A:** Call ReadResource with invalid URI; expect error response.  
**Keywords:** Error scenario, URI validation  
**Follow-up:** What should server return?

**Q99: How do you test concurrent tool calls?**  
**A:** Create multiple concurrent requests; verify all complete correctly.  
**Keywords:** Concurrency, async  
**Follow-up:** Can one failure affect others?

**Q100: How do you test tool schema?**  
**A:** Call ListTools; validate each tool has name, description, inputSchema.  
**Keywords:** Schema validation, metadata  
**Follow-up:** What makes a valid schema?

**Q101: How do you stress test an MCP server?**  
**A:** Send large number of requests; vary payload sizes; measure latency, errors, resources.  
**Keywords:** Load test, performance  
**Follow-up:** What tools exist?

**Q102: How do you debug a tool that works in tests but fails in production?**  
**A:** Check environment differences, logging, external dependencies, auth, rate limits.  
**Keywords:** Debugging methodology, env differences  
**Follow-up:** What are common causes?

**Q103: How do you trace a request through the full system?**  
**A:** Implement distributed tracing; use request IDs across layers.  
**Keywords:** Tracing, request ID, observability  
**Follow-up:** What tools support this?

**Q104: What should you log for debugging?**  
**A:** Request inputs, execution steps, results, errors, timing, external API calls.  
**Keywords:** Logging strategy, verbosity  
**Follow-up:** What NOT to log?

**Q105: How do you handle logs from a stdio subprocess?**  
**A:** Redirect server stderr to log file; never mix with stdout (protocol).  
**Keywords:** stderr handling, stdout discipline  
**Follow-up:** What if server logs to stdout?

---

### SECTION 5: Architecture & System Design (15 questions)

**Q106: Design a multi-server MCP architecture for a research assistant.**  
**A:**
```
Agent
├─ GitHub MCP Server (list issues, create issues)
├─ Filesystem MCP Server (read docs, create reports)
├─ Search MCP Server (web search, vector search)
└─ Database MCP Server (query knowledge base)
```
**Keywords:** Composition, separation of concerns  
**Follow-up:** How do you handle name conflicts?

**Q107: How would you handle server A being down in a multi-server setup?**  
**A:** Other servers remain available; agent falls back to alternative tools; retry logic on transient failures.  
**Keywords:** Fault isolation, fallback  
**Follow-up:** How does agent know which server is down?

**Q108: Design an enterprise MCP system with multi-tenancy.**  
**A:**
```
Enterprise Host
├─ Authentication Layer (tenant validation)
├─ MCP Client (tenant context injection)
├─ Multi-tenant MCP Server A
├─ Multi-tenant MCP Server B
└─ Audit logging (all tenant actions)
```
**Keywords:** Isolation, context, audit  
**Follow-up:** How do you prevent cross-tenant data leaks?

**Q109: How would you design an MCP system for GDPR compliance?**  
**A:** Tool approval for PII access; audit logging; consent tracking; data minimization.  
**Keywords:** Privacy, compliance, audit  
**Follow-up:** How do you handle "right to be forgotten"?

**Q110: Design a high-availability MCP server deployment.**  
**A:** Multiple instances; load balancer; health checks; graceful shutdown; state management.  
**Keywords:** HA, scaling, statelessness  
**Follow-up:** What about session state?

**Q111: How would you optimize latency for remote MCP servers?**  
**A:** HTTP/2 connection pooling; caching; batch requests; local proxies; CDN for resources.  
**Keywords:** Performance, caching, batching  
**Follow-up:** What's the trade-off?

**Q112: Design MCP + RAG integration.**  
**A:** MCP Resource provides documents; RAG system retrieves relevant chunks; Agent queries via MCP.  
**Keywords:** Integration, document access  
**Follow-up:** Where does vector DB fit?

**Q113: Design MCP for real-time collaboration (e.g., multi-agent).**  
**A:** Shared MCP Server for state; versioning; conflict resolution; notifications.  
**Keywords:** Shared state, versioning, events  
**Follow-up:** How do you handle concurrent updates?

**Q114: How would you migrate from custom integrations to MCP?**  
**A:** Incrementally wrap existing integrations in MCP Servers; test compatibility; deprecate old code.  
**Keywords:** Migration strategy, incremental  
**Follow-up:** What's the rollback plan?

**Q115: Design a discovery service for MCP servers in large organizations.**  
**A:** Central registry; server metadata; health status; authentication requirements.  
**Keywords:** Service discovery, registry, metadata  
**Follow-up:** How do you prevent outdated registrations?

**Q116: How would you implement request rate limiting for MCP?**  
**A:** Token bucket at client or server; track requests per user/org; return 429 error.  
**Keywords:** Rate limiting, throttling  
**Follow-up:** What metrics would you track?

**Q117: Design MCP security for zero-trust architecture.**  
**A:** Every request authenticated; encrypt transport; audit everything; validate responses.  
**Keywords:** Zero trust, auth, audit  
**Follow-up:** How do you handle key rotation?

**Q118: How would you design MCP for compliance with HIPAA (healthcare)?**  
**A:** Encryption at rest/transit; access controls; audit logging; data isolation; breach notification.  
**Keywords:** Healthcare compliance, security  
**Follow-up:** What about HITECH?

**Q119: Design MCP versioning strategy for API compatibility.**  
**A:** Semver; tool versioning; backwards compatibility; deprecation timeline; clear upgrade path.  
**Keywords:** Versioning, compatibility, deprecation  
**Follow-up:** How long do you support old versions?

**Q120: How would you design MCP for offline-first applications?**  
**A:** Offline cache; local MCP server replica; sync when online; conflict resolution.  
**Keywords:** Offline, caching, sync  
**Follow-up:** How do you handle stale data?

---

### SECTION 6: Scenarios & Problem-Solving (40 questions)

**Q121: GitHub MCP server suddenly returns 503. What happens?**  
**A:** Client timeout or gets error response; agent can't list issues; fallback to cached data or skip GitHub tools.  
**Keywords:** Failure, fallback, robustness  
**Follow-up:** How do you detect this proactively?

**Q122: Tool executes successfully but returns corrupted JSON. What happens?**  
**A:** Client fails to deserialize; returns protocol error; agent receives error; no result for that tool call.  
**Keywords:** Data corruption, parsing error  
**Follow-up:** How do you prevent this?

**Q123: Client sends CallTool with wrong argument type (string instead of int). What happens?**  
**A:** Server validates against schema; returns -32602 (Invalid params) error.  
**Keywords:** Validation, type checking  
**Follow-up:** Should client try again?

**Q124: MCP Server process crashes. What happens?**  
**A:** (Stdio) Transport detects EOF; client fails; agent reports server unavailable.  
**Keywords:** Crash detection, connection loss  
**Follow-up:** How do you implement auto-restart?

**Q125: Client connects but initialization never completes. What happens?**  
**A:** Client timeout; connection reset; agent cannot proceed with this server.  
**Keywords:** Hanging connection, timeout  
**Follow-up:** How do you debug hanging init?

**Q126: Two agents both try to modify the same resource. What happens?**  
**A:** (Depends on implementation) Last write wins; OR conflict error; OR versioning.  
**Keywords:** Concurrency, conflict  
**Follow-up:** How do you handle this?

**Q127: Resource contains prompt injection (e.g., "Ignore your instructions..."). What happens?**  
**A:** Agent reads resource; injection string goes to LLM; potential security risk.  
**Keywords:** Injection attack, trust boundary  
**Follow-up:** How do you prevent this?

**Q128: Tool tries to access database but auth token expired. What happens?**  
**A:** Database returns 401; tool returns error in ToolResult.  
**Keywords:** Auth failure, error handling  
**Follow-up:** Should client retry?

**Q129: Client sends ListTools to unresponsive server (no response for 30s). What happens?**  
**A:** Client timeout triggers; connection reset; agent reports server unavailable.  
**Keywords:** Timeout, unresponsiveness  
**Follow-up:** How do you configure timeout?

**Q130: Tool description is 10,000 characters long. Any problem?**  
**A:** Technically no; practical issue: agent may ignore long descriptions; context overflow.  
**Keywords:** Scale, practical limits  
**Follow-up:** What's a reasonable limit?

**Q131: Server exposes 1,000 tools. Any problem?**  
**A:** Agent context overflow; LLM struggles with choice; performance impact; usability.  
**Keywords:** Scaling, practical limits  
**Follow-up:** How do you solve this?

**Q132: Tool argument is complex nested object. How is schema represented?**  
**A:** JSON Schema nested properties; type: object, properties with nested definitions.  
**Keywords:** Complex schema, nesting  
**Follow-up:** Can you use references?

**Q133: Client accidentally kills server process (signal -9). What happens?**  
**A:** (Stdio) Client reads EOF; protocol broken; must reconnect.  
**Keywords:** Process termination  
**Follow-up:** How do you handle this gracefully?

**Q134: Server experiences high CPU while executing tool. What happens?**  
**A:** Tool takes long time; client may timeout; other clients waiting.  
**Keywords:** Resource exhaustion  
**Follow-up:** How do you mitigate?

**Q135: Network packet lost between client and server. What happens?**  
**A:** (HTTP) Retries or timeout; (Stdio) May cause protocol desync; need reconnect.  
**Keywords:** Network failure, transport  
**Follow-up:** How resilient is MCP to packet loss?

**Q136: LLM hallucinates tool name that doesn't exist. What happens?**  
**A:** Host creates CallTool with non-existent name; server returns -32601 (Method not found).  
**Keywords:** Hallucination, error handling  
**Follow-up:** How does agent recover?

**Q137: Tool succeeds but returns result with wrong type (string instead of int). What happens?**  
**A:** Agent receives result; if using statically typed code, may have type error; if not, works as-is.  
**Keywords:** Type mismatch, runtime  
**Follow-up:** Should server validate return type?

**Q138: Client configured with wrong server URL. What happens?**  
**A:** Connection fails immediately; agent reports server unreachable.  
**Keywords:** Configuration error  
**Follow-up:** How do you detect this early?

**Q139: Tool success but returns 500MB of data. What happens?**  
**A:** Possible memory issues; slow network transfer; context overflow in agent.  
**Keywords:** Data size, scaling  
**Follow-up:** How do you handle large results?

**Q140: Server responds with Resource of type "image/png" but client expects text. What happens?**  
**A:** Client receives MIME type; agent may fail to process; error or fallback.  
**Keywords:** MIME type, content negotiation  
**Follow-up:** Should client validate?

**Q141: Tool calls another tool as part of execution. Can this create infinite loops?**  
**A:** Yes; tool A calls tool B calls tool A = infinite recursion; need depth limiting.  
**Keywords:** Recursion, safety limits  
**Follow-up:** How do you prevent infinite recursion?

**Q142: Multiple agents race to increment a counter via MCP tool. What value is final?**  
**A:** Depends on implementation; last write wins; OR needs locking.  
**Keywords:** Race condition, atomicity  
**Follow-up:** How do you ensure consistency?

**Q143: Tool leaks credentials in error message. What happens?**  
**A:** Credentials visible in ToolResult; potentially exposed to agent/logs/UI.  
**Keywords:** Security, credential leakage  
**Follow-up:** How do you prevent this?

**Q144: Server under DDoS attack. How does MCP help?**  
**A:** MCP doesn't inherently protect; need rate limiting, auth, network security.  
**Keywords:** Security, resilience  
**Follow-up:** What's your defense?

**Q145: Client unable to deserialize response due to unknown field in JSON. What happens?**  
**A:** (Strict parsing) Parse error; (Lenient parsing) Skip unknown fields.  
**Keywords:** Forward compatibility, versioning  
**Follow-up:** How do you maintain compatibility?

**Q146: Tool returns same result for different inputs. Agent doesn't realize. What happens?**  
**A:** Agent may make wrong decisions based on cached result; no inherent caching in MCP.  
**Keywords:** Determinism, correctness  
**Follow-up:** Should you add caching?

**Q147: Server claims to support Tool capability but ListTools is empty. What happens?**  
**A:** Inconsistency; agent has no tools to use; likely implementation bug.  
**Keywords:** Implementation bug, consistency  
**Follow-up:** How do you validate?

**Q148: Client receives notification before initialization completes. What happens?**  
**A:** Depends on client implementation; likely queued or dropped.  
**Keywords:** Message ordering, sequencing  
**Follow-up:** Is this allowed by spec?

**Q149: Tool takes 59.9 seconds with 60 second timeout. Last 0.1s spent serializing response. What happens?**  
**A:** Response serialization may exceed timeout; client times out; receives no result.  
**Keywords:** Timing, edge cases  
**Follow-up:** How do you add buffer?

**Q150: Agent calls tool, server processes it, network hiccup loses response. Agent waits for timeout. What happens?**  
**A:** Tool actually executed; but agent doesn't know; may retry and double-execute.  
**Keywords:** Idempotency, retry safety  
**Follow-up:** How do you make tools idempotent?

**Q151: Three servers all expose resource "document://overview". How does client handle?**  
**A:** (Depends on architecture) First match wins; OR error; OR namespace it.  
**Keywords:** Namespace, disambiguation  
**Follow-up:** How would you fix?

**Q152: Tool result contains PII. Agent logs it. What's the compliance issue?**  
**A:** Logs may be stored insecurely; violates GDPR/HIPAA; audit trail records PII.  
**Keywords:** Compliance, logging, PII  
**Follow-up:** How do you prevent?

**Q153: MCP protocol version changed. Old clients connect to new server. What happens?**  
**A:** Mismatch detected in Initialize; client/server may reject; error returned.  
**Keywords:** Version negotiation, compatibility  
**Follow-up:** What's the upgrade strategy?

**Q154: Server supports tools, client sends ListResources. What happens?**  
**A:** Server responds with empty resources (or no resources capability); client proceeds normally.  
**Keywords:** Partial capabilities  
**Follow-up:** Is this expected?

**Q155: Agent too slow; generates response before tool result arrives. What happens?**  
**A:** Agent misses result; LLM generates response without complete information.  
**Keywords:** Timing, race condition  
**Follow-up:** How do you enforce ordering?

---

## PART 12: 150+ Rapid-Fire Interview Questions

### Quick Answers (Test Your Knowledge)

**Core Concepts (5-10 sec each):**

1. **MCP = ?** Model Context Protocol
2. **Why MCP?** Standardize AI ↔ external system communication
3. **3 primitives?** Tools, Resources, Prompts
4. **Tool vs Resource?** Tool = action (side effects); Resource = read-only data
5. **Host vs Client?** Host = app; Client = communication layer
6. **LLM calls tool?** No; Host does
7. **Protocol vs Transport?** Protocol = rules; Transport = delivery method
8. **Stdio vs HTTP?** Stdio = local subprocess; HTTP = remote network
9. **JSON-RPC?** Request/response/notification messages with IDs
10. **ListTools does?** Queries server for available tools
11. **CallTool does?** Executes a specific tool with arguments
12. **ToolResult has?** content (array) + isError (bool)
13. **Resource URI?** Addressing scheme (e.g., github://pull/123)
14. **Prompt?** Reusable instruction template
15. **InitializeRequest?** Handshake to establish session
16. **Capabilities?** Feature declarations (tools, resources, prompts, etc.)
17. **FastMCP?** Python framework for building MCP servers
18. **@mcp.tool?** Decorator registering function as tool + generates schema
19. **Schema from?** Type hints + docstring parsing
20. **MCP Inspector?** Developer tool for testing MCP servers

**Protocol Details (5-10 sec):**

21. **JSON-RPC fields?** jsonrpc, id, method, params, result, error
22. **ListToolsRequest?** Empty params; returns tools array
23. **CallToolRequest?** params.name + params.arguments
24. **ReadResourceRequest?** params.uri
25. **GetPromptRequest?** params.name + params.arguments
26. **Error code -32602?** Invalid params
27. **Error code -32601?** Method not found
28. **Notification?** No id; unsolicited; one-way
29. **Resource template?** URI with {placeholder}
30. **Content types?** text, image, resource (in ToolResult)

**FastMCP (5-10 sec):**

31. **@mcp.tool() requires?** Function + type hints
32. **Optional param?** Not in required array
33. **Tool docstring?** Becomes description
34. **Tool error?** ToolResult with isError=true
35. **FastMCP resource?** @mcp.resource(uri)
36. **FastMCP prompt?** @mcp.prompt()
37. **Context injection?** Add context: Context parameter
38. **mcp.run()?** Start server, listen on stdin
39. **Tool async?** Yes; async def supported
40. **Tool logging?** context.log(msg)

**Debugging (5-10 sec):**

41. **Inspector connects to?** stdio or HTTP
42. **Inspector execute?** Tool call via UI
43. **Why test?** Inspector is manual; need automation
44. **Integration test?** Use ClientSession + Client Transport
45. **Stress test?** Many concurrent requests
46. **Timeout test?** Intentional delay; expect timeout
47. **Error test?** Exception in tool; expect error result
48. **Schema validation?** jsonSchema validation
49. **Logging location?** stderr (not stdout)
50. **What not to log?** Secrets, PII, passwords

**Architecture (5-10 sec):**

51. **Multi-server?** Multiple clients, each to a server
52. **Namespace conflict?** Prefix tool names
53. **Server down?** Other servers continue; fallback
54. **Fault isolation?** Server A failure doesn't affect B
55. **Caching strategy?** Depends on data freshness
56. **Rate limiting?** Token bucket; track per user/org
57. **Health check?** Dedicated tool or HTTP endpoint
58. **Versioning?** Semver; tool versioning
59. **Backwards compat?** Keep old tool versions; deprecate
60. **High availability?** Multiple instances; load balancer

**Security (5-10 sec):**

61. **Authentication?** Token/creds; validate before tool execution
62. **Authorization?** Check user permissions for tool/resource
63. **Input validation?** Schema validation + business logic checks
64. **Output validation?** Check response doesn't leak secrets
65. **Prompt injection?** Don't treat resource content as instructions
66. **TLS?** Required for HTTP transport
67. **Secrets?** Environment variables; never hardcode
68. **Audit logging?** Log all tool calls (no secrets)
69. **PII handling?** Detect, redact, log access
70. **GDPR?** Consent, audit, "right to be forgotten"

**Reliability (5-10 sec):**

71. **Transient failure?** Network timeout; retry
72. **Permanent failure?** Invalid auth; don't retry
73. **Retry strategy?** Exponential backoff; max attempts
74. **Fallback?** Alternative tool or cached data
75. **Timeout?** Client timeout < Server timeout < External API timeout
76. **Circuit breaker?** Stop calling after N failures
77. **Idempotency?** Tool can be safely retried
78. **Concurrency?** Multiple clients OK; race conditions need locking
79. **Resource exhaustion?** Monitor CPU/memory; implement limits
80. **Network loss?** HTTP: retry; Stdio: reconnect

**Monitoring (5-10 sec):**

81. **Logs track?** Inputs, steps, errors, timing
82. **Metrics?** Counts, latency, error rates
83. **Traces?** Request path; latency per step
84. **Alert threshold?** Error rate > 1%; latency p95 > limit
85. **Request ID?** Trace across layers
86. **Server availability?** Uptime %; alert < 99.5%
87. **Latency metric?** p50, p95, p99
88. **Error rate?** Errors / total requests
89. **Resource usage?** CPU, memory, disk
90. **Tool latency?** Track per-tool; identify slow tools

**Deployment (5-10 sec):**

91. **Docker?** Containerize; pass env vars for secrets
92. **Compose?** Multi-service; define dependencies
93. **Health probe?** Liveness (process alive); Readiness (accepting requests)
94. **Load balancer?** Route to multiple instances
95. **Kubernetes?** StatelessProbes; HPA; rolling deploy
96. **Config management?** Environment variables; secrets manager
97. **Logging?** Stdout captured; stderr for debug
98. **Graceful shutdown?** Complete in-flight requests; timeout
99. **Rollback?** Keep old image; redeploy if new fails
100. **Blue-green?** Two environments; switch traffic

**Real-World (5-10 sec):**

101. **Slow tool?** Cache results; async execution; optimize query
102. **Tool timeout?** Increase timeout or optimize tool
103. **High error rate?** Debug logs; check external API; validate inputs
104. **Connection drops?** Add retry logic; check network
105. **Resource leak?** Monitor memory; add limits; implement cleanup
106. **Server crashes?** Check logs; add try-catch; add health check
107. **Agent hallucination?** Improve prompt; limit tool list; verify selections
108. **Context overflow?** Trim tool descriptions; limit tools offered
109. **Latency spikes?** Check external dependencies; add caching; profile
110. **Data inconsistency?** Add versioning; implement locking; add validation

---

## PART 13: Final Cheat Sheets

### Architecture Cheat Sheet

```
┌────────────────────────────────────┐
│ AGENT / LLM                         │
│ (decides which tool to use)         │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│ HOST APPLICATION                   │
│ (your app; contains client)         │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│ MCP CLIENT                          │
│ (manages protocol communication)    │
│ • Connects to server                │
│ • ListTools / CallTool / ReadResource│
│ • Error handling                    │
└────────────────────────────────────┘
           ↓ MCP Protocol (JSON-RPC 2.0)
           ↓ Transport (stdio or HTTP)
           ↓
┌────────────────────────────────────┐
│ MCP SERVER                          │
│ (exposes capabilities)              │
│ • Tools (actions)                   │
│ • Resources (data)                  │
│ • Prompts (templates)               │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│ EXTERNAL SYSTEMS                    │
│ • APIs                              │
│ • Databases                         │
│ • Services                          │
└────────────────────────────────────┘
```

### Tool Lifecycle Cheat Sheet

```
1. DISCOVERY
   Client.list_tools()
   → Server returns [Tool1, Tool2, ...]
   → Agent sees available tools

2. DECISION
   Agent analyzes query
   → Selects: "I need Tool1"

3. CALL
   Host sends CallToolRequest
   {method: "tools/call", params: {name: "Tool1", arguments: {...}}}
   → MCP Client sends to Server

4. EXECUTION
   Server receives request
   → Validates arguments against schema
   → Executes Python function
   → Captures result

5. RETURN
   Server sends CallToolResponse
   {result: {content: [...], isError: false}}
   → MCP Client receives
   → Host processes result

6. REASONING
   Agent sees result
   → Continues reasoning
   → May select another tool or generate final answer
```

### FastMCP Quick Reference

```python
# Minimal Server
from fastmcp import FastMCP

mcp = FastMCP("Server")

# Tool
@mcp.tool()
def multiply(a: int, b: int) -> int:
    """Multiply two numbers.
    
    Args:
        a: First number
        b: Second number
    """
    return a * b

# Resource
@mcp.resource("data://user/{id}")
def get_user(id: str) -> str:
    """Get user data."""
    return fetch_user(id)

# Prompt
@mcp.prompt()
def analyze(text: str) -> str:
    """Analyze text."""
    return f"Analyze this: {text}"

# Run
if __name__ == "__main__":
    mcp.run()
```

### Protocol Message Cheat Sheet

```
INITIALIZE
Request:
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {},
    "clientInfo": {"name": "client", "version": "1.0"}
  }
}

Response:
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {"tools": {}, "resources": {}},
    "serverInfo": {"name": "server", "version": "1.0"}
  }
}

LIST TOOLS
Request:
{"jsonrpc": "2.0", "id": 2, "method": "tools/list", "params": {}}

Response:
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [{
      "name": "multiply",
      "description": "Multiply two numbers",
      "inputSchema": {
        "type": "object",
        "properties": {
          "a": {"type": "integer"},
          "b": {"type": "integer"}
        },
        "required": ["a", "b"]
      }
    }]
  }
}

CALL TOOL
Request:
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "multiply",
    "arguments": {"a": 3, "b": 4}
  }
}

Response:
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [{"type": "text", "text": "12"}],
    "isError": false
  }
}
```

### Error Codes Cheat Sheet

| Code | Meaning | Retry? |
|------|---------|--------|
| -32700 | Parse error | No |
| -32600 | Invalid Request | No |
| -32601 | Method not found | No |
| -32602 | Invalid params | No |
| -32603 | Internal error | Yes |
| Tool Error | Tool execution failed | Maybe |

### Best Practices Checklist

```
✓ Protocol
  [ ] Validate InitializeResponse
  [ ] Check server capabilities
  [ ] Handle all JSON-RPC error codes
  [ ] Implement retry with backoff
  [ ] Set reasonable timeouts

✓ Tools
  [ ] All tools have descriptions
  [ ] All parameters documented
  [ ] Input validation
  [ ] Error handling
  [ ] Timeout protection

✓ Resources
  [ ] URIs documented
  [ ] MIME types correct
  [ ] Access control verified
  [ ] Caching strategy

✓ Security
  [ ] Authentication enabled
  [ ] Authorization enforced
  [ ] Input validation
  [ ] No secrets in logs
  [ ] TLS for remote

✓ Observability
  [ ] Structured logging
  [ ] Metrics collected
  [ ] Tracing enabled
  [ ] Health checks
  [ ] Alerts configured

✓ Reliability
  [ ] Retry logic implemented
  [ ] Fallback strategy
  [ ] Circuit breaker
  [ ] Timeout management
  [ ] Error handling

✓ Testing
  [ ] Unit tests
  [ ] Integration tests
  [ ] Error scenarios
  [ ] Performance tested
  [ ] Security tested

✓ Deployment
  [ ] Environment variables
  [ ] Secrets management
  [ ] Health checks
  [ ] Monitoring
  [ ] Graceful shutdown
```

---

## PART 14: Summary & Next Steps

### What You Now Understand

✓ **Why MCP exists** - Solves n×m integration problem  
✓ **What MCP is** - Protocol for standardized AI ↔ system communication  
✓ **How it works** - Client-server with Protocol over Transport  
✓ **Three primitives** - Tools (actions), Resources (data), Prompts (templates)  
✓ **Full lifecycle** - Discovery → Decision → Call → Execution → Return  
✓ **Protocol details** - JSON-RPC 2.0 message structure  
✓ **Transport** - Stdio (local) vs HTTP (remote)  
✓ **FastMCP** - Framework for building servers with minimal code  
✓ **Testing & Debugging** - Inspector + automated tests  
✓ **Production** - Deployment, monitoring, reliability, security  
✓ **System design** - Multi-server, enterprise, compliance  
✓ **Interview readiness** - 150+ questions with answers  

### Interview Preparation Roadmap

**Week 1: Fundamentals**
- Read Part 1-5 of this document
- Build simple FastMCP server
- Connect with MCP Inspector
- Run basic tests

**Week 2: Protocol Deep Dive**
- Study JSON-RPC structure
- Understand message flow
- Build MCP client
- Debug with Inspector

**Week 3: Implementation**
- Build multi-tool server
- Add resources and prompts
- Implement error handling
- Write integration tests

**Week 4: Production & Design**
- Study reliability patterns
- Design multi-server system
- Implement observability
- Review security checklist

**Week 5: Interview Prep**
- Review rapid-fire questions
- Practice system design scenarios
- Articulate trade-offs
- Explain with confidence

### Key Takeaways for Interviews

1. **Define MCP clearly** - "Standardized protocol for AI ↔ external system communication"
2. **Explain the problem** - Before MCP: n×m integrations; After: 1 standard protocol
3. **Know the architecture** - Host → Client → Server → External systems
4. **Understand primitives** - Tools (action), Resources (data), Prompts (template)
5. **Protocol vs Transport** - Independent concepts; can mix and match
6. **Tool execution** - LLM doesn't call directly; Host handles via client
7. **Reliability** - Retry, fallback, timeout, circuit breaker
8. **Security** - Auth, validation, audit, secrets
9. **Scale** - Multi-server, isolation, fallback
10. **Production** - Monitoring, observability, deployment

### Resources

- **Official MCP Docs:** https://modelcontextprotocol.io/
- **MCP Python SDK:** https://github.com/anthropics/mcp-python-sdk
- **FastMCP:** (Check MCP ecosystem)
- **This Documentation:** Reference for all concepts

---

## End of Documentation

**Total Coverage:**
- 2 comprehensive documents
- 97 major sections
- 150+ interview questions
- 10+ architecture diagrams
- Cheat sheets and references
- Production patterns
- Security frameworks
- Testing strategies
- System design scenarios

**You are now interview-ready for:**
✓ GenAI / Agentic AI roles  
✓ MCP Engineering positions  
✓ LLM Application Design  
✓ System Design interviews  
✓ Production deployment discussions  
