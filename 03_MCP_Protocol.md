# 03: MCP Protocol

**Version/Source Basis:** MCP protocol as of **July 28, 2026**. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

## What Is the MCP Protocol?

### Real-World Analogy

Imagine a **letter-based communication system:**
- Everyone follows the same letter format (address, content, envelope type)
- You don't need to know the recipient personally
- The postal system (transport) delivers the letter
- The recipient understands the format and responds

**MCP Protocol is like the letter format** – a standard structure for messages that host and server understand, regardless of transport.

### Technical Definition

The **MCP Protocol** is a message-based specification built on **JSON-RPC 2.0**. It defines:
1. How messages are structured
2. What messages mean
3. When they're sent
4. How responses are formatted

It's **transport-independent** – messages can travel over HTTP, WebSocket, stdio, etc.

## JSON-RPC Foundation

### What Is JSON-RPC 2.0?

JSON-RPC is a **remote procedure call protocol** using JSON over any transport. A request/response pattern.

### Request Structure

```json
{
  "jsonrpc": "2.0",
  "method": "method_name",
  "params": {...},
  "id": 1
}
```

| Field | Meaning |
|-------|---------|
| `jsonrpc` | Version (always "2.0") |
| `method` | What action to perform |
| `params` | Arguments for the method |
| `id` | Request ID (for matching responses) |

### Response Structure

```json
{
  "jsonrpc": "2.0",
  "result": {...},
  "id": 1
}
```

Or error:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32600,
    "message": "Invalid Request"
  },
  "id": 1
}
```

### Why JSON-RPC?

- **Standard:** Already widely used and understood
- **Simple:** Request-response pattern is clear
- **Flexible:** Params can be any JSON structure
- **IDs:** Match responses to requests even with async messaging

## MCP Protocol Initialization Lifecycle

### Phase 1: Server Initialization Request

**Client sends:**
```json
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {
    "protocol_version": "2024-11-05",
    "capabilities": {
      "sampling": {}
    },
    "client_info": {
      "name": "my-app",
      "version": "1.0.0"
    }
  },
  "id": 1
}
```

**Server responds:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "protocol_version": "2024-11-05",
    "capabilities": {
      "tools": {},
      "resources": {},
      "prompts": {}
    },
    "server_info": {
      "name": "customer-server",
      "version": "1.0.0"
    }
  },
  "id": 1
}
```

### Phase 2: Initialization Complete Notification

After successful initialization, client sends:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
}
```

This is a **notification** (no `id`, no response expected). It signals "initialization is complete, you can start sending tool calls."

### Phase 3: Ready for Operations

Now the client can:
- Discover tools, resources, prompts
- Call tools
- Read resources

## Protocol Methods: Discovery

### List Tools

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "tools/list",
  "params": {},
  "id": 2
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "tools": [
      {
        "name": "get_customer",
        "description": "Retrieve customer information",
        "inputSchema": {
          "type": "object",
          "properties": {
            "customer_id": {
              "type": "string",
              "description": "The customer's ID"
            }
          },
          "required": ["customer_id"]
        }
      },
      {
        "name": "update_customer",
        "description": "Update customer information",
        "inputSchema": {
          "type": "object",
          "properties": {
            "customer_id": {"type": "string"},
            "data": {"type": "object"}
          },
          "required": ["customer_id", "data"]
        }
      }
    ]
  },
  "id": 2
}
```

### List Resources

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "resources/list",
  "params": {},
  "id": 3
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "resources": [
      {
        "uri": "documents://annual_report_2026.pdf",
        "name": "Annual Report 2026",
        "description": "Yearly financial report",
        "mimeType": "application/pdf"
      }
    ]
  },
  "id": 3
}
```

### List Prompts

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "prompts/list",
  "params": {},
  "id": 4
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "prompts": [
      {
        "name": "customer_summary",
        "description": "Generate a summary of a customer",
        "arguments": [
          {
            "name": "customer_id",
            "description": "Customer ID",
            "required": true
          }
        ]
      }
    ]
  },
  "id": 4
}
```

## Protocol Methods: Invocation

### Call Tool

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "get_customer",
    "arguments": {
      "customer_id": "12345"
    }
  },
  "id": 5
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Customer: Alice, ID: 12345, Balance: $5000, Status: Active"
      }
    ]
  },
  "id": 5
}
```

### Read Resource

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "resources/read",
  "params": {
    "uri": "documents://annual_report_2026.pdf"
  },
  "id": 6
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "contents": [
      {
        "uri": "documents://annual_report_2026.pdf",
        "mimeType": "application/pdf",
        "text": "[PDF content as text]"
      }
    ]
  },
  "id": 6
}
```

### Get Prompt

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "prompts/get",
  "params": {
    "name": "customer_summary",
    "arguments": {
      "customer_id": "12345"
    }
  },
  "id": 7
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "Analyze customer 12345: Alice, long-term customer, premium tier..."
        }
      }
    ]
  },
  "id": 7
}
```

## Notifications vs Requests

### Request (Expects Response)

```json
{
  "jsonrpc": "2.0",
  "method": "tools/list",
  "params": {},
  "id": 1  // Has ID → expects response
}
```

**Must wait for response.**

### Notification (No Response Expected)

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
  // No ID → it's a notification
}
```

**Fire and forget.** No response expected.

## Error Handling

### Tool Call Errors

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "get_customer",
    "arguments": {
      "customer_id": "invalid"
    }
  },
  "id": 8
}
```

**Error Response:**
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Internal error",
    "data": {
      "details": "Customer not found"
    }
  },
  "id": 8
}
```

### Common Error Codes

| Code | Meaning |
|------|---------|
| -32700 | Parse error |
| -32600 | Invalid Request |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |
| -32000 to -32099 | Server error (reserved) |

## Content Types

Responses can contain different content types:

### Text Content

```json
{
  "type": "text",
  "text": "This is plain text"
}
```

### Image Content

```json
{
  "type": "image",
  "data": "base64-encoded-image-data",
  "mimeType": "image/png"
}
```

### Multiple Content Items

```json
{
  "content": [
    {"type": "text", "text": "Analysis: ..."},
    {"type": "image", "data": "...", "mimeType": "image/png"}
  ]
}
```

## July 28, 2026 Revision Highlights

### What Changed

1. **Explicit Statelessness:** Protocol explicitly states each message is independent
2. **Improved Discovery:** Better metadata in capability lists
3. **Authorization Context:** Protocol-level support for authorization
4. **Resource Streaming:** Efficient handling of large resources
5. **Simplified Messages:** Cleaner JSON structure

### Authorization Support

Requests can include authorization context:

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "transfer_money",
    "arguments": {"amount": 1000},
    "authorization": {
      "scheme": "Bearer",
      "credentials": "token_xyz"
    }
  },
  "id": 9
}
```

Server validates the token before executing the tool.

### Resource Streaming

For large resources, streaming is supported:

```json
{
  "method": "resources/read",
  "params": {
    "uri": "documents://large_file.pdf",
    "streaming": true  // Request streaming
  },
  "id": 10
}
```

Server streams content in chunks instead of one large response.

## Complete Message Flow Example

```
Step 1: Client Initialization
Client → Server:
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {"protocol_version": "2024-11-05", ...},
  "id": 1
}

Server → Client:
{
  "jsonrpc": "2.0",
  "result": {"capabilities": {...}, ...},
  "id": 1
}

Step 2: Initialization Complete
Client → Server:
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
}

Step 3: Discover Tools
Client → Server:
{
  "jsonrpc": "2.0",
  "method": "tools/list",
  "params": {},
  "id": 2
}

Server → Client:
{
  "jsonrpc": "2.0",
  "result": {"tools": [...]},
  "id": 2
}

Step 4: Call Tool
Client → Server:
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {"name": "get_customer", "arguments": {"customer_id": "123"}},
  "id": 3
}

Server → Client:
{
  "jsonrpc": "2.0",
  "result": {"content": [{"type": "text", "text": "Customer data..."}]},
  "id": 3
}
```

## Interview Summary

**Key Concepts:**
- **JSON-RPC 2.0:** Foundation for message structure
- **Stateless:** Each message is independent
- **Notifications:** Fire-and-forget messages
- **Requests:** Always expect a response
- **Discovery:** Methods to list tools, resources, prompts
- **Invocation:** Methods to call tools, read resources, get prompts
- **Errors:** Structured error responses with codes
- **July 2026:** Explicit statelessness, authorization, streaming

**One-Liner:**
"MCP Protocol is a JSON-RPC 2.0-based specification for stateless message exchange between hosts and servers, supporting discovery and invocation of tools, resources, and prompts."

## Common Interview Questions

**Q1: What is JSON-RPC and why does MCP use it?**
A: JSON-RPC is a request/response protocol for remote calls. MCP uses it because it's standard, simple, and provides automatic request/response matching via IDs.

**Q2: What's the difference between a notification and a request?**
A: A request has an `id` field and expects a response. A notification has no `id` and is fire-and-forget (used for initialization complete, server ready, etc.).

**Q3: How does the July 2026 revision enforce statelessness?**
A: The protocol explicitly requires each message to include all context needed for processing. Servers don't maintain session state between messages, making them horizontally scalable.

**Q4: What happens if a tool call fails?**
A: The server responds with a JSON-RPC error object containing an error code and message, plus optional data field with details.

**Q5: Can a tool response contain multiple types of content?**
A: Yes. Tools can return arrays of content objects—text, images, etc. This lets tools provide rich, multimodal responses.

---

**Next:** [04_MCP_Python_SDK_v2.md](04_MCP_Python_SDK_v2.md) – Installing and using the official SDK
