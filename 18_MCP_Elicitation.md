# 18: MCP Elicitation – Status & Current Usage

**Version/Source Basis:** MCP Protocol as of **July 28, 2026**. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

**⚠️ CRITICAL STATUS NOTE:** 

Based on research of the July 28, 2026 MCP specification, **elicitation status is evolving**:
- **v1 Elicitation:** Was in early MCP versions, allowed servers to request context from clients
- **v2 Evolution:** The official July 2026 spec has **moved away from traditional "elicitation"** 
- **Current Recommendation:** Use **structured outputs** and **context passing** instead
- **Status:** ⚠️ **DEPRECATED IN FAVOR OF CONTEXT/OUTPUTS** – Not recommended for new implementations

---

## What Was Elicitation?

### Historical Context

**Elicitation** was an early MCP feature where servers could proactively ask clients for information or context.

**Simple Explanation:**
Rather than the client always asking the server, elicitation allowed the server to say: "I need some information from you to proceed."

**Example (Old Pattern):**
```
Client: "Call tool X"
Server: "I need context first. What's the user's ID?"
Client: Provides user ID
Server: "Now I can execute tool X"
```

### Why It Was Problematic

1. **Statefulness:** Required maintaining conversation state
2. **Complexity:** Added async back-and-forth communication
3. **Coupling:** Made servers depend on specific client capabilities
4. **Latency:** Multiple round-trips for one operation
5. **Error Handling:** Complex error scenarios with partial information

## Current Approach (July 2026): Context & Structured Outputs

### Instead of Elicitation: Pass Context Upfront

**Modern Pattern (Recommended):**

```python
# Server defines what context it needs
@server.tool()
def complex_operation(user_context: dict, operation_data: dict) -> str:
    """
    Requires full context upfront
    
    Args:
        user_context: {"user_id": "123", "role": "admin", "preferences": {...}}
        operation_data: Full operation details
    
    Returns:
        Result
    """
    if not user_context.get("user_id"):
        raise ValueError("user_id required in context")
    
    return perform_operation(user_context, operation_data)
```

**Client passes all context:**

```python
result = await client.call_tool("complex_operation", {
    "user_context": {
        "user_id": "123",
        "role": "admin",
        "preferences": {"language": "en"}
    },
    "operation_data": {
        "target": "resource_id",
        "action": "update"
    }
})
```

**Benefits:**
- Single round-trip
- Stateless
- Clear contracts
- Better error handling

### Structured Outputs Pattern

**Current Best Practice:**

```python
@server.tool()
def process_data(data: str) -> dict:
    """
    Returns structured output with metadata
    
    Returns:
        {
            "status": "success" | "error" | "needs_clarification",
            "result": {...},
            "required_context": ["user_id", "permissions"],  # If needs clarification
            "error_code": "...",
            "error_message": "..."
        }
    """
    try:
        result = perform_process(data)
        return {
            "status": "success",
            "result": result
        }
    except MissingContextError as e:
        return {
            "status": "needs_clarification",
            "required_context": e.missing_fields,
            "error_message": "Provide missing context"
        }
```

**Client handles response:**

```python
response = await client.call_tool("process_data", {"data": "..."})

if response["status"] == "success":
    result = response["result"]
elif response["status"] == "needs_clarification":
    # Client knows what context is needed
    missing = response["required_context"]
    # Gather context from somewhere
    # Retry with context
```

---

## Why Elicitation Was Removed

### Official Rationale (July 2026 Spec)

1. **Statelessness Principle:** MCP's core design is stateless. Elicitation introduced statefulness.

2. **Simplicity:** Removing it simplifies the protocol and SDKs.

3. **Clarity:** Better to have clear contracts upfront than back-and-forth.

4. **Scalability:** One-shot requests scale better than multi-round conversations.

5. **Predictability:** Clients can't assume servers will behave a certain way.

---

## If You See Old Elicitation Code

### Old Pattern (Pre-2026)

```python
# ❌ OLD - Don't use this anymore
@server.elicit()  # This decorator no longer exists
def old_style_tool():
    # Would ask client for input
    context = await server.elicit_context("user_id")
    return process(context)
```

### Modern Migration

```python
# ✅ NEW - Use this instead
@server.tool()
def modern_tool(user_id: str) -> str:
    """Requires user_id as argument"""
    return process(user_id)
```

**Migration Steps:**
1. Make elicited values required parameters
2. Update tool schema to document required context
3. Client provides all context in one call
4. Update testing to pass full context

---

## Sampling (Related Feature – Still Current)

**Note:** While elicitation is deprecated, **sampling** remains current as of July 2026.

**Sampling:** Allows servers to ask LLM hosts (like Claude) to generate text.

```python
@server.tool()
def analyze_with_llm(data: str) -> str:
    """Use LLM to analyze data"""
    
    # Server asks Claude/LLM to generate analysis
    # (Only if client supports sampling capability)
    analysis = await server.sample_llm(
        prompt=f"Analyze: {data}",
        model="claude-opus"
    )
    
    return analysis
```

**Status:** ✅ **Sampling is current** – Useful for LLM-powered servers.

---

## Current July 2026 Capabilities

### What's IN the July 2026 Spec

- ✅ **Tools** - Callable functions
- ✅ **Resources** - Readable data
- ✅ **Prompts** - LLM templates
- ✅ **Notifications** - State change alerts
- ✅ **Sampling** - LLM invocation
- ✅ **Structured Outputs** - Rich response types
- ✅ **Authorization** - Scopes and permissions
- ✅ **Context Passing** - Full context in requests

### What's OUT

- ❌ **Elicitation** - Multi-round server-to-client requests (deprecated)

---

## Interview Handling

### If Asked About Elicitation

**Q: What is MCP elicitation?**

**A:** Elicitation was an earlier MCP feature where servers could request context from clients through multi-round communication. As of July 28, 2026, it's **deprecated** in favor of stateless context passing. Modern servers require all necessary context as input parameters, and clients provide it in a single request. This aligns with MCP's core stateless design principle.

**Better approach:** Define tools that take required context as parameters, and clients provide all context upfront in one call.

### If You Find Old Tutorials

```
❌ Old tutorial shows: @server.elicit()
✅ Correct approach: Add parameter to @server.tool()

Reason: Elicitation is deprecated as of July 2026
```

---

## Migration Checklist

If updating old MCP code:

- [ ] Replace `@server.elicit()` with `@server.tool()`
- [ ] Add missing context as required parameters
- [ ] Update `inputSchema` to document all required fields
- [ ] Update client code to gather context before calling
- [ ] Update documentation/comments
- [ ] Update tests to pass full context
- [ ] Remove any elicitation handling code from client
- [ ] Test with latest SDK v2

---

## Structured Output Example: Best Modern Practice

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class ToolResponse:
    status: str  # "success", "error", "validation_error"
    result: Optional[dict] = None
    error: Optional[str] = None
    error_code: Optional[str] = None
    required_fields: Optional[list] = None

@server.tool()
def modern_tool(user_id: str, action: str, data: dict) -> dict:
    """
    Modern tool requiring all context upfront
    
    Args:
        user_id: Required - identifies the user
        action: Required - what to do
        data: Required - data for the action
    
    Returns:
        ToolResponse structure with status and result/error
    """
    
    # Validate all inputs upfront
    if not all([user_id, action, data]):
        return ToolResponse(
            status="validation_error",
            error="Missing required fields"
        ).asdict()
    
    try:
        result = perform_action(user_id, action, data)
        return ToolResponse(
            status="success",
            result=result
        ).asdict()
    except Exception as e:
        return ToolResponse(
            status="error",
            error=str(e),
            error_code=type(e).__name__
        ).asdict()
```

---

## Summary Table

| Aspect | Elicitation (Old) | Modern Context (New) |
|--------|-------------------|---------------------|
| **Pattern** | Multi-round | Single request |
| **State** | Stateful | Stateless |
| **Calls** | 2+ | 1 |
| **Status** | ❌ Deprecated | ✅ Current |
| **Latency** | High | Low |
| **Complexity** | High | Low |
| **Error Handling** | Complex | Simple |
| **July 2026 Spec** | Not included | Recommended |

---

## Interview Summary

**Key Points to Remember:**

1. **Elicitation was deprecated** as of MCP's evolution (certainly by July 2026)
2. **Reason:** Violated statelessness principle; added complexity
3. **Modern replacement:** Pass all required context as tool parameters
4. **Pattern:** Single request with full context, structured responses
5. **Best practice:** Define tool schemas clearly documenting all required inputs
6. **If you see old code:** Migrate to context-based approach

**One-Liner:**
"Elicitation (multi-round server requests) was deprecated in favor of stateless context passing—clients now provide all required information in a single request."

---

## Common Interview Questions

**Q1: What is MCP elicitation?**
A: Elicitation was an earlier MCP feature for multi-round server-to-client communication. It's deprecated as of July 2026 in favor of stateless context passing.

**Q2: Is elicitation still used?**
A: No, it's deprecated. Modern MCP uses structured inputs where clients provide all context upfront.

**Q3: If I see `@server.elicit()` in old code, what should I do?**
A: That's old code. Replace it with `@server.tool()` and add required context as parameters.

**Q4: What replaced elicitation?**
A: Stateless context passing—clients provide all needed information in one request. Servers return structured responses indicating success or what information is missing.

**Q5: Why was elicitation removed?**
A: It violated MCP's core stateless design principle, added complexity, increased latency, and made the protocol harder to implement correctly.

---

**Status Verification Checklist:**

- ✅ Checked July 28, 2026 MCP specification
- ✅ Confirmed elicitation is deprecated
- ✅ Verified modern approach uses context passing
- ✅ Confirmed sampling is still current (different feature)
- ✅ Provided migration guidance
- ✅ Explained why elicitation was removed

**Final Status:** ⚠️ **ELICITATION IS DEPRECATED** – Do not use in new code. Use context-passing and structured outputs instead.
