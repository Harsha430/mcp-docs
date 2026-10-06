# 15: MCP Interview Questions & Answers

**Version/Source Basis:** MCP Protocol July 28, 2026. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

---

## Beginner Level

**Q1: What is MCP?**
A: MCP (Model Context Protocol) is an open, standardized protocol that lets applications discover and use tools, resources, and prompts from external services. It decouples the host application from the services it uses by defining a common message format and interaction pattern.

**Q2: Why does MCP exist?**
A: Without MCP, applications hardcode integrations to each service. MCP solves this by providing a standard contract: any server following MCP can expose capabilities, and any host understanding MCP can discover and use them. This is critical for AI agents that need to work with changing tools and data.

**Q3: What are the three core components of MCP?**
A: 
1. **Host** - The application (Claude, LangChain, your code) 
2. **Client** - The protocol handler inside the host
3. **Server** - The capability provider (database, API, service)

**Q4: What's an MCP tool?**
A: A tool is a function a server exposes. For example, `get_customer(customer_id)` is a tool. The host can discover it and call it.

**Q5: What's an MCP resource?**
A: A resource is data the server offers to read. For example, `file://data.txt` is a resource. The host can read it via the protocol.

**Q6: What's an MCP prompt?**
A: A prompt is a template or guidance the server provides. For example, a customer analysis template. The host can request it and use it to instruct an LLM.

**Q7: What's the difference between MCP and REST APIs?**
A: REST requires the app to know each API's endpoints, methods, authentication. MCP allows automatic discovery—the server tells the host what it can do.

**Q8: Is MCP synchronous or asynchronous?**
A: The protocol is inherently request-response, but transport can be synchronous (HTTP) or asynchronous (WebSocket). The official SDK v2 is async (uses async/await).

**Q9: What transport does MCP use?**
A: MCP is transport-agnostic. Common transports: stdio (local), HTTP/HTTPS (remote), WebSocket (persistent connections), SSE (Server-Sent Events).

**Q10: What did the July 28, 2026 MCP revision change?**
A: Formalized statelessness (each message is independent), simplified protocol messages, enhanced authorization support, and improved capability discovery.

---

## Intermediate Level

**Q11: What does "stateless" mean in MCP?**
A: Stateless means each message includes all context needed to process it. The server doesn't remember you between requests. This lets one server instance handle many concurrent clients. Transport can be stateful (WebSocket) but protocol is stateless.

**Q12: Can one application connect to multiple MCP servers?**
A: Yes. The typical pattern is one MCP Client per server. The host orchestrates which server to use based on its logic.

**Q13: How does tool discovery work?**
A: The client sends a `tools/list` message. The server responds with a list of available tools including their names, descriptions, and input schemas. The client caches this.

**Q14: What is JSON-RPC 2.0?**
A: JSON-RPC is a request/response protocol using JSON. A request has a method, params, and ID. The response has a result or error, plus the matching ID. MCP uses this as its foundation.

**Q15: What's the initialization handshake?**
A: 
1. Client sends `initialize` with protocol version and capabilities
2. Server responds with its capabilities
3. Client sends `notifications/initialized` (fire-and-forget)
4. Now client can discover tools and call them

**Q16: What happens if a tool call fails?**
A: The server sends a JSON-RPC error response with an error code and message. The client converts this to an exception. The host handles it.

**Q17: What's the difference between a notification and a request?**
A: A request has an `id` field and expects a response. A notification has no `id` and is fire-and-forget. `notifications/initialized` is a notification.

**Q18: How do you handle errors in MCP?**
A: At protocol level, servers respond with JSON-RPC error objects. At SDK level, exceptions in tool functions are converted to errors. At application level, use try/except.

**Q19: What does the Official SDK v2 provide?**
A: The `mcp` package provides `Server` and `Client` classes, decorators (@tool, @resource, @prompt), transport implementations (StdioTransport, HttpTransport), and async/await support.

**Q20: What is FastMCP?**
A: FastMCP is a higher-level convenience framework built on the official SDK v2. It reduces boilerplate for rapid prototyping. Less control than official SDK but faster to develop with.

---

## Advanced Level

**Q21: How would you design a multi-server architecture?**
A: 
1. Identify services: customer, banking, documents, etc.
2. Each becomes an MCP server exposing relevant tools
3. The host maintains a client per server
4. The host orchestrates which server to use
5. Use parallel calls when independent, sequential when dependent
Example: Customer agent queries customer server, financial agent queries banking server.

**Q22: How do you scale an MCP system?**
A: 
- Deploy multiple instances of each server behind a load balancer (HTTP/HTTPS transport allows this)
- Use stateless design (part of the protocol)
- Cache tool discovery results
- Use parallel calls where possible
- Monitor and profile tool execution times

**Q23: What's the relationship between MCP and LLM function calling?**
A: Function calling embeds function definitions in LLM prompts. MCP dynamically discovers function definitions from servers. MCP is more flexible for changing tools; function calling is simpler for fixed functions.

**Q24: How would you handle authorization in MCP?**
A: 
1. Client includes credentials in MCP messages (Bearer token, API key)
2. Server validates credentials (authentication)
3. Server checks permissions (authorization)
4. Use scopes or roles for granular access control
5. Enforce least privilege

**Q25: What security considerations apply to remote MCP servers?**
A: Use HTTPS/TLS for transport encryption, implement authentication (JWT/tokens), enforce authorization (scopes), store secrets in environment variables (never hardcode), validate and sanitize all inputs, use certificate pinning for critical systems.

**Q26: How do you test MCP systems?**
A: 
- Unit tests: test tool logic
- Integration tests: test server + tools
- E2E tests: test client + server together
- Error tests: connection failures, timeouts, invalid inputs
- Multi-server tests: coordination and isolation

**Q27: What's the difference between MCP and a Message Queue (RabbitMQ, Kafka)?**
A: Message queues are for async publish-subscribe. MCP is for synchronous request-response discovery and tool invocation. They solve different problems.

**Q28: Can MCP servers call other MCP servers?**
A: Not directly via MCP. A server can call another service as an external dependency, but the host is the orchestrator between MCP servers.

**Q29: How would you implement caching for tool results?**
A: 
- Cache tool discovery results (rarely change during runtime)
- Cache resource reads if data is stable
- Invalidation: refresh if server indicates changes
- Trade-off: staleness vs performance

**Q30: What does "one MCP Client per server" mean?**
A: Each MCP server connection requires its own client instance. If connecting to 3 servers, you have 3 clients. The host manages all of them.

---

## Architecture Level

**Q31: In a LangChain + MCP system, what component initiates tool discovery?**
A: The LangChain MCP adapter creates MCP clients, which initiate discovery by sending `tools/list` messages. LangChain doesn't know about MCP; the adapter translates for it.

**Q32: How does CrewAI differ from LangChain when using MCP?**
A: LangChain has one agent with all tools. CrewAI has multiple agents, each specialized. The MCP adapter distributes tools from shared servers to different agents.

**Q33: What's the difference between an MCP adapter and an MCP client?**
A: The MCP client implements the protocol. The adapter is framework-specific and uses/wraps the client to integrate with the framework (LangChain, CrewAI, AutoGen).

**Q34: Can multiple frameworks (LangChain, CrewAI, AutoGen) share the same MCP servers?**
A: Yes. This is the ideal pattern. Each framework has its own adapter and clients, but all talk to the same shared MCP servers.

**Q35: How would you design a system where Claude, LangChain, and CrewAI all access the same business data?**
A: 
1. Business data → MCP Servers (customer, banking, documents)
2. Claude → MCP Client → Servers
3. LangChain → MCP Adapter → MCP Clients → Servers
4. CrewAI → MCP Adapter → MCP Clients → Servers
Each host independently discovers and uses tools, but all servers are shared.

---

## Scenario-Based

**Q36: Scenario - A tool call is slow. How would you debug it?**
A:
1. Profile the tool function: time its execution
2. Check network latency: ping the server
3. Check server load: is it CPU-bound or I/O-bound?
4. Enable logging: see protocol messages
5. Implement caching: avoid redundant calls
6. Parallelize: call independent tools concurrently

**Q37: Scenario - MCP server crashes while your application is running. What happens?**
A:
- The MCP client throws a connection error
- The host receives the error and can handle it
- Other servers continue working
- Implement retry logic or fallback
- The error is isolated to that server

**Q38: Scenario - You need to call Tool A, then Tool B with Tool A's result. Design this.**
A:
```
1. Call Tool A: result_a = await client.call_tool("tool_a", {})
2. Wait for result
3. Call Tool B with result: result_b = await client.call_tool("tool_b", {"input": result_a})
This is sequential orchestration. Dependencies dictate the order.
```

**Q39: Scenario - You need to call Tool A, Tool B, Tool C independently. Design this.**
A:
```
Parallel orchestration:
results = await asyncio.gather(
    client.call_tool("tool_a", {}),
    client.call_tool("tool_b", {}),
    client.call_tool("tool_c", {})
)
All three run concurrently.
```

**Q40: Scenario - You need to authorize certain users to certain tools. Design this.**
A:
```
1. Extract user token from request
2. Validate token (authentication)
3. Get user scopes/roles from database
4. Before executing tool, check if user has permission
5. Example scope model: "customer:read", "banking:transfer"
6. Raise PermissionError if unauthorized
```

---

## Protocol Deep-Dive

**Q41: What does an MCP initialize message look like?**
A:
```json
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {
    "protocol_version": "2024-11-05",
    "capabilities": {"sampling": {}},
    "client_info": {"name": "my-app", "version": "1.0"}
  },
  "id": 1
}
```

**Q42: What does a tools/call message look like?**
A:
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "get_customer",
    "arguments": {"customer_id": "123"}
  },
  "id": 2
}
```

**Q43: What does an error response look like?**
A:
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Internal error",
    "data": {"details": "Customer not found"}
  },
  "id": 2
}
```

**Q44: What are common MCP error codes?**
A:
- -32700: Parse error (malformed JSON)
- -32600: Invalid Request (malformed)
- -32601: Method not found
- -32602: Invalid params
- -32603: Internal error

**Q45: What's in a tool definition?**
A:
```json
{
  "name": "transfer_money",
  "description": "Transfer funds between accounts",
  "inputSchema": {
    "type": "object",
    "properties": {
      "from": {"type": "string"},
      "to": {"type": "string"},
      "amount": {"type": "number"}
    },
    "required": ["from", "to", "amount"]
  }
}
```

---

## Common Mistakes & Edge Cases

**Q46: What's a common mistake when defining tool arguments?**
A: Forgetting to include all required arguments in the `required` array. If a tool needs `customer_id` and `options`, mark both as required. If `options` is optional, omit it from `required`.

**Q47: What happens if you call a tool that doesn't exist?**
A: The server returns a method-not-found error. The client converts it to an exception. Always call `list_tools()` first to verify tool availability.

**Q48: What if a client sends the wrong argument type?**
A: Depends on validation. The server should validate against `inputSchema`. If it doesn't, it may fail silently or throw an error. Always validate inputs.

**Q49: What if authentication fails?**
A: The server returns an authorization error. The client throws an exception. The host receives it and can retry with different credentials or fail gracefully.

**Q50: What if you forget to await an async operation?**
A: The promise/coroutine doesn't execute. The tool never runs. Always `await` async calls:
```
❌ result = client.call_tool(...)  # Wrong
✅ result = await client.call_tool(...)  # Correct
```

---

## Interview Summary

**What to Emphasize:**
- MCP is a protocol, not a framework
- Understand the architecture: host → client → server
- Statelessness and scalability
- Protocol vs SDK vs adapter
- How frameworks integrate
- Security (authentication/authorization)
- Multi-server coordination

**Interview Approach:**
1. Start with fundamentals (what is MCP?)
2. Show you understand the architecture
3. Discuss design decisions (stateless, JSON-RPC, transport-agnostic)
4. Show practical knowledge (SDK, tools, resources)
5. Demonstrate systems thinking (scaling, security, errors)

---

**Next:** [16_MCP_Cheat_Sheet.md](16_MCP_Cheat_Sheet.md) – Quick reference guide
