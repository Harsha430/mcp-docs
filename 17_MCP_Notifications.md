# 17: MCP Notifications System

**Version/Source Basis:** MCP Protocol as of **July 28, 2026**. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

**Current Status:** ✅ **ACTIVE AND CURRENT** - Notifications are a core feature of MCP as of July 2026.

---

## What Are Notifications?

### Simple Explanation

**Notifications** are one-way messages that don't expect a response. Think of them like notifications on your phone—you get a message but don't have to reply.

In MCP, both clients and servers can send notifications to signal events or state changes.

### Real-World Analogy

A restaurant (server) sends you a notification: "Your table is ready" or "Your order is complete." You don't reply to these messages; you just receive them and react.

## Notifications vs Requests

### Request (Expects Response)

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {...},
  "id": 1  // Has ID → expects response
}

Response:
{
  "jsonrpc": "2.0",
  "result": {...},
  "id": 1
}
```

### Notification (No Response)

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
  // NO ID → no response expected
}
```

**Key Difference:** Notifications have no `id` field. No response is sent or expected.

## Notification Types

### 1. Client → Server Notifications

#### `notifications/initialized`

Sent by client after initialization handshake completes.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
}
```

**When:** After client receives server's initialize response. Signals "I'm ready to use your tools."

### 2. Server → Client Notifications

#### `notifications/resources/list_changed`

Server notifies client that resources list has changed.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/list_changed",
  "params": {}
}
```

**When:** New resources added, resources removed, or resource metadata changed.

**Client Action:** Should call `resources/list` again to refresh.

#### `notifications/tools/list_changed`

Server notifies client that tools list has changed.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed",
  "params": {}
}
```

**When:** New tools registered, tools removed, tool definitions changed.

**Client Action:** Should call `tools/list` again to refresh.

#### `notifications/prompts/list_changed`

Server notifies client that prompts list has changed.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/prompts/list_changed",
  "params": {}
}
```

**When:** New prompts added, prompts modified, prompts removed.

**Client Action:** Should call `prompts/list` again to refresh.

## Complete Notification Lifecycle

### Scenario: Server Adds a New Tool

```
Timeline:
T0: Client calls tools/list
    Server responds: [get_customer, list_customers]
    Client caches these tools
    
T1: Server developer adds new tool: update_customer
    
T2: Server sends notification: tools/list_changed
    
T3: Client receives notification
    Client invalidates cache
    Client calls tools/list
    
T4: Server responds: [get_customer, list_customers, update_customer]
    Client updates cache
    
T5: Client can now use update_customer
```

## Implementation: Server Sending Notifications

### Server Code

```python
from mcp.server import Server
from mcp.transport import StdioTransport
import asyncio

server = Server("dynamic-server")

# Track registered tools
registered_tools = ["get_customer"]

@server.tool()
def get_customer(customer_id: str) -> str:
    """Get customer info"""
    return f"Customer {customer_id}"

async def add_new_tool():
    """Simulate adding a new tool dynamically"""
    await asyncio.sleep(5)  # Wait 5 seconds
    
    # Register new tool
    registered_tools.append("create_customer")
    
    # Notify clients
    # Send notification that tools changed
    await server.send_notification("notifications/tools/list_changed", {})

@server.tool()
def list_tools():
    """Return current tools"""
    return registered_tools

async def main():
    # Start background task to add tool later
    asyncio.create_task(add_new_tool())
    
    transport = StdioTransport()
    await server.run(transport)

if __name__ == "__main__":
    asyncio.run(main())
```

## Implementation: Client Handling Notifications

### Client Code

```python
import asyncio
from mcp.client import Client
from mcp.transport import StdioTransport
import subprocess

class NotificationAwareClient:
    def __init__(self, transport):
        self.client = Client(transport)
        self.tool_cache = None
        self.resource_cache = None
        self.prompt_cache = None
        
        # Register notification handlers
        self.client.on_notification("notifications/tools/list_changed", 
                                    self.handle_tools_changed)
        self.client.on_notification("notifications/resources/list_changed",
                                    self.handle_resources_changed)
        self.client.on_notification("notifications/prompts/list_changed",
                                    self.handle_prompts_changed)
    
    async def handle_tools_changed(self, params):
        """Called when server notifies tools changed"""
        print("Tools list changed - refreshing cache...")
        self.tool_cache = None  # Invalidate
        self.tool_cache = await self.client.list_tools()
        print(f"New tools: {[t.name for t in self.tool_cache]}")
    
    async def handle_resources_changed(self, params):
        """Called when resources changed"""
        print("Resources changed - refreshing...")
        self.resource_cache = None
        self.resource_cache = await self.client.list_resources()
    
    async def handle_prompts_changed(self, params):
        """Called when prompts changed"""
        print("Prompts changed - refreshing...")
        self.prompt_cache = None
        self.prompt_cache = await self.client.list_prompts()
    
    async def get_tools(self):
        """Get tools (with caching)"""
        if self.tool_cache is None:
            self.tool_cache = await self.client.list_tools()
        return self.tool_cache
    
    async def initialize(self):
        await self.client.initialize()

async def main():
    proc = subprocess.Popen(["python", "dynamic_server.py"],
                           stdin=subprocess.PIPE,
                           stdout=subprocess.PIPE)
    
    transport = StdioTransport(proc)
    client = NotificationAwareClient(transport)
    await client.initialize()
    
    # Initial tool list
    tools = await client.get_tools()
    print(f"Initial tools: {[t.name for t in tools]}")
    
    # Wait - server will add new tool and send notification
    print("Waiting for notifications...")
    await asyncio.sleep(10)
    
    # Check updated tools
    tools = await client.get_tools()
    print(f"Final tools: {[t.name for t in tools]}")
    
    await client.client.close()
    proc.terminate()

if __name__ == "__main__":
    asyncio.run(main())
```

## Use Cases for Notifications

### 1. Dynamic Tool Discovery

Server adds/removes tools without client reconnection.

```
Client discovers initial tools
Server adds new tool
Server sends notification
Client refreshes without reconnecting
```

### 2. Resource Updates

Server indicates that resources have changed.

```
Client caches resource list
Resource file updated on server
Server sends notification
Client knows to refresh
```

### 3. Server State Changes

Server signals operational changes.

```
Server enters maintenance mode
Server sends notification
Client can handle gracefully
```

### 4. Real-Time Alerts

Server sends alerts or important notifications.

```
Monitoring: System health changes
Server sends notification
Host/client can react immediately
```

## Notification Pattern: Pub/Sub Style

```python
class NotificationSubscriber:
    def __init__(self, client):
        self.client = client
        self.handlers = {}
    
    def subscribe(self, notification_type, handler):
        """Subscribe to notification"""
        if notification_type not in self.handlers:
            self.handlers[notification_type] = []
        self.handlers[notification_type].append(handler)
        
        # Register with client
        self.client.on_notification(notification_type, handler)
    
    def unsubscribe(self, notification_type, handler):
        """Unsubscribe from notification"""
        if notification_type in self.handlers:
            self.handlers[notification_type].remove(handler)

# Usage
subscriber = NotificationSubscriber(client)

async def on_tools_changed(params):
    print("Tools changed!")

subscriber.subscribe("notifications/tools/list_changed", on_tools_changed)
```

## Best Practices

### 1. Always Invalidate Cache on Notification

```python
async def handle_tools_changed(params):
    # ✅ Good: Invalidate cache
    self.tool_cache = None
    self.tool_cache = await self.client.list_tools()
    
    # ❌ Bad: Ignore notification
    # Just continue with old cache
```

### 2. Handle Notifications Gracefully

```python
async def handle_notification(params):
    try:
        # Refresh data
        await refresh_cache()
    except Exception as e:
        # Don't crash; log and continue
        logger.error(f"Failed to handle notification: {e}")
```

### 3. Don't Send Notifications Too Often

```python
# ❌ Bad: Send notification on every change
@server.tool()
def update_data(data):
    db.save(data)
    await server.send_notification(...)  # Too often!

# ✅ Good: Batch notifications
async def batch_notification_sender():
    changes = []
    while True:
        if changes:
            await server.send_notification(...)
            changes = []
        await asyncio.sleep(1)
```

## Interview Summary

**Key Notification Concepts:**
- **Notifications:** One-way messages (no response)
- **Current Status:** Active in July 2026 MCP
- **Types:** `notifications/initialized`, `notifications/tools/list_changed`, etc.
- **Use Cases:** Dynamic tool discovery, cache invalidation, alerts
- **Pattern:** Pub/sub style subscription
- **Cache Invalidation:** Invalidate when notification received

**One-Liner:**
"MCP notifications are one-way messages (no response expected) that servers send to clients when capabilities change, enabling dynamic tool/resource/prompt discovery without reconnection."

## Common Interview Questions

**Q1: What's the difference between a request and a notification?**
A: A request has an `id` field and expects a response. A notification has no `id` and is fire-and-forget. Use notifications for state changes that don't need acknowledgment.

**Q2: When would a server send `notifications/tools/list_changed`?**
A: When tools are added, removed, or modified. The client should refresh its cache after receiving this.

**Q3: What should a client do when it receives a notification?**
A: Invalidate any cached capability list (tools, resources, prompts) and refresh by calling the corresponding `list` method.

**Q4: Can notifications be lost?**
A: Depends on transport. HTTP is request-response; WebSocket is bidirectional. The protocol doesn't guarantee delivery—implement retry logic if critical.

**Q5: Are notifications part of the July 2026 spec?**
A: Yes, notifications are a core MCP feature as of July 28, 2026.

---

**Status Check:** ✅ Notifications are **CURRENT** and actively used in MCP as of July 28, 2026.
