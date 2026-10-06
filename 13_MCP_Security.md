# 13: MCP Security and Authorization

**Version/Source Basis:** MCP Protocol July 28, 2026 security features. Reference: [https://modelcontextprotocol.io/docs/2026-07-28/learn/](https://modelcontextprotocol.io/docs/2026-07-28/learn/)

## Security Principles

### Three Layers

```
Layer 1: Transport Security (How messages travel)
   └─ HTTPS/TLS, secure stdin, authorized connections

Layer 2: Authentication (Who are you?)
   └─ Credentials, tokens, API keys

Layer 3: Authorization (What can you do?)
   └─ Permissions, scopes, access control
```

## Transport Layer Security

### Stdio Transport (Local)

```python
# Stdio is local process-to-process
# Security: Limited to local machine
# Assumption: If process can access stdio, it's trusted

server_process = subprocess.Popen(
    ["python", "server.py"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE
)
```

**Risks:**
- Anyone on the machine can intercept
- No network, so limited exposure

**Use when:** Local tools, IDE plugins, development

### HTTP Transport (Remote)

```python
# HTTP over plain text
# Risk: Messages visible in transit

transport = HttpTransport(base_url="http://localhost:8000")  # ❌ NOT SAFE
```

### HTTPS Transport (Secure Remote)

```python
# HTTPS encrypts messages in transit
# Recommended for all remote

transport = HttpTransport(base_url="https://api.company.com")  # ✅ SAFE
```

**Always use HTTPS in production.**

## Authentication Layer

### What Authentication Does

Authentication answers: **"Who are you?"**

### Bearer Token Pattern

```python
# Server-side
@server.tool()
def transfer_money(amount: float):
    # Extract token from request context
    token = get_auth_context().token
    
    if not validate_token(token):
        raise PermissionError("Invalid token")
    
    # Execute tool
    return process_transfer(amount)

# Client-side
from mcp.transport import HttpTransport

transport = HttpTransport(
    base_url="https://api.company.com",
    headers={"Authorization": "Bearer secret_token_xyz"}
)
```

### JWT (JSON Web Tokens)

```python
import jwt
from datetime import datetime, timedelta

# Server generates JWT
def generate_token(user_id: str):
    payload = {
        "user_id": user_id,
        "exp": datetime.utcnow() + timedelta(hours=1)
    }
    token = jwt.encode(payload, "secret_key", algorithm="HS256")
    return token

# Client uses JWT
@server.tool()
def get_data():
    token = get_auth_context().token
    
    try:
        payload = jwt.decode(token, "secret_key", algorithms=["HS256"])
        user_id = payload["user_id"]
    except jwt.ExpiredSignatureError:
        raise PermissionError("Token expired")
    except jwt.InvalidTokenError:
        raise PermissionError("Invalid token")
    
    # Proceed with user_id
    return fetch_user_data(user_id)
```

### API Key Pattern

```python
# Simple API key
VALID_KEYS = {
    "key_abc123": {"name": "client_1", "permissions": ["read", "write"]},
    "key_def456": {"name": "client_2", "permissions": ["read"]}
}

@server.tool()
def get_data():
    api_key = get_auth_context().api_key
    
    if api_key not in VALID_KEYS:
        raise PermissionError("Invalid API key")
    
    # Get permissions
    permissions = VALID_KEYS[api_key]["permissions"]
    
    if "read" not in permissions:
        raise PermissionError("Read not permitted")
    
    return fetch_data()
```

**Risk:** Don't hardcode keys. Use environment variables.

```python
import os

VALID_KEYS = {
    os.getenv("CLIENT_API_KEY"): {"name": "client_1", "permissions": ["read", "write"]}
}
```

## Authorization Layer

### What Authorization Does

Authorization answers: **"What can you do?"**

### Role-Based Access Control (RBAC)

```python
ROLES = {
    "admin": {
        "tools": ["get_data", "write_data", "delete_data", "manage_users"],
        "resources": ["all"]
    },
    "analyst": {
        "tools": ["get_data", "search_documents"],
        "resources": ["public", "analytics"]
    },
    "user": {
        "tools": ["get_data"],
        "resources": ["public"]
    }
}

@server.tool()
def delete_data(data_id: str):
    # Get user from token
    user = get_current_user()
    
    # Get user's role
    role = get_user_role(user)
    
    # Check authorization
    if "delete_data" not in ROLES[role]["tools"]:
        raise PermissionError(f"Role {role} cannot delete data")
    
    # Proceed
    return delete_from_db(data_id)
```

### Scope-Based Authorization

```python
SCOPES = {
    "customer:read": "Read customer data",
    "customer:write": "Write customer data",
    "banking:transfer": "Execute transfers",
    "banking:read": "Read banking data",
    "admin:*": "All admin operations"
}

def validate_scope(required_scope: str):
    user_scopes = get_user_scopes()  # From token
    
    # Check if user has required scope
    if required_scope not in user_scopes:
        # Check for wildcard
        if "admin:*" not in user_scopes:
            raise PermissionError(f"Scope {required_scope} required")

@server.tool()
def transfer_money(to_account: str, amount: float):
    validate_scope("banking:transfer")
    return process_transfer(to_account, amount)

@server.tool()
def get_customer(customer_id: str):
    validate_scope("customer:read")
    return fetch_customer(customer_id)
```

## Complete Example: Secure MCP Server

```python
from mcp.server import Server
from mcp.transport import HttpTransport
import os
import jwt
from datetime import datetime, timedelta
import asyncio

server = Server("secure-api")

# Configuration
SECRET_KEY = os.getenv("SECRET_KEY")  # Never hardcode!
ALLOWED_ORIGINS = os.getenv("ALLOWED_ORIGINS", "").split(",")

def authenticate_request():
    """Extract and validate token from request context"""
    auth_header = os.getenv("HTTP_AUTHORIZATION", "")
    
    if not auth_header.startswith("Bearer "):
        raise PermissionError("Missing authorization header")
    
    token = auth_header[7:]  # Remove "Bearer "
    
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return payload
    except jwt.ExpiredSignatureError:
        raise PermissionError("Token expired")
    except jwt.InvalidTokenError:
        raise PermissionError("Invalid token")

def check_authorization(user_id: str, required_scope: str):
    """Check if user has required scope"""
    user_scopes = get_user_scopes(user_id)  # From database
    
    if required_scope not in user_scopes:
        raise PermissionError(f"User lacks scope: {required_scope}")

@server.tool()
def get_customer(customer_id: str) -> str:
    """Get customer data (requires customer:read scope)"""
    payload = authenticate_request()
    user_id = payload["user_id"]
    
    check_authorization(user_id, "customer:read")
    
    # Proceed
    return f"Customer {customer_id} data"

@server.tool()
def transfer_money(to_account: str, amount: float) -> str:
    """Transfer money (requires banking:transfer scope)"""
    payload = authenticate_request()
    user_id = payload["user_id"]
    
    check_authorization(user_id, "banking:transfer")
    
    # Additional validation
    if amount > 100000:
        # High amount - require admin approval
        if "banking:approve_high_transfers" not in get_user_scopes(user_id):
            raise PermissionError("High transfers require approval")
    
    # Proceed
    return f"Transferred ${amount} to {to_account}"

@server.tool()
def admin_reset_password(user_id: str, new_password: str) -> str:
    """Admin operation (requires admin:users scope)"""
    payload = authenticate_request()
    admin_id = payload["user_id"]
    
    check_authorization(admin_id, "admin:users")
    
    # Proceed
    return f"Password reset for user {user_id}"

def get_user_scopes(user_id: str):
    """Fetch user scopes from database (implementation)"""
    # In real app: database.query("SELECT scopes FROM users WHERE id = ?", user_id)
    return ["customer:read", "customer:write", "banking:transfer"]

async def main():
    # Setup HTTPS transport
    transport = HttpTransport(
        host="0.0.0.0",
        port=8443,
        ssl_certfile="/path/to/cert.pem",
        ssl_keyfile="/path/to/key.pem"
    )
    await server.run(transport)

if __name__ == "__main__":
    asyncio.run(main())
```

## Secure Client Example

```python
import os
from mcp.client import Client
from mcp.transport import HttpTransport

# Get credentials from environment, NOT from code
API_KEY = os.getenv("MCP_API_KEY")
SERVER_URL = os.getenv("MCP_SERVER_URL")

if not API_KEY or not SERVER_URL:
    raise RuntimeError("Set MCP_API_KEY and MCP_SERVER_URL environment variables")

async def main():
    # Create transport with authentication
    transport = HttpTransport(
        base_url=SERVER_URL,
        headers={"Authorization": f"Bearer {API_KEY}"}
    )
    
    client = Client(transport)
    await client.initialize()
    
    # Call tools
    result = await client.call_tool("get_customer", {"customer_id": "123"})
    print(result)
    
    await client.close()
```

## Deployment: Remote MCP Server

### Setting Up HTTPS

```bash
# Generate self-signed certificate (dev only)
openssl req -x509 -newkey rsa:4096 -nodes -out cert.pem -keyout key.pem -days 365

# In production: Use certificates from trusted CA
```

### Environment Configuration

```bash
# .env (never commit to version control)
SECRET_KEY=super_secret_key_xyz
DATABASE_URL=postgresql://user:pass@localhost/db
ALLOWED_ORIGINS=https://trusted-app.com
MCP_PORT=8443
MCP_CERTFILE=/etc/certs/server.pem
MCP_KEYFILE=/etc/certs/server-key.pem
```

### Deployment Pattern

```python
import os
from pathlib import Path

# Load from environment
SECRET_KEY = os.getenv("SECRET_KEY")
DATABASE_URL = os.getenv("DATABASE_URL")
CERT_FILE = os.getenv("MCP_CERTFILE")
KEY_FILE = os.getenv("MCP_KEYFILE")
PORT = int(os.getenv("MCP_PORT", "8443"))

if not all([SECRET_KEY, CERT_FILE, KEY_FILE]):
    raise RuntimeError("Missing required environment variables")

# Verify files exist
if not Path(CERT_FILE).exists():
    raise RuntimeError(f"Certificate file not found: {CERT_FILE}")

# Setup with TLS
transport = HttpTransport(
    host="0.0.0.0",
    port=PORT,
    ssl_certfile=CERT_FILE,
    ssl_keyfile=KEY_FILE
)
```

## Security Best Practices

### 1. Never Hardcode Secrets

```python
# ❌ Bad
API_KEY = "secret_abc123"

# ✅ Good
API_KEY = os.getenv("API_KEY")
if not API_KEY:
    raise RuntimeError("API_KEY environment variable required")
```

### 2. Use HTTPS in Production

```python
# ❌ Bad
transport = HttpTransport(base_url="http://localhost:8000")

# ✅ Good
transport = HttpTransport(base_url="https://secure-api.company.com")
```

### 3. Validate All Inputs

```python
@server.tool()
def process_data(data: str) -> str:
    # Validate input
    if not data:
        raise ValueError("data is required")
    if len(data) > 10000:
        raise ValueError("data too large")
    
    # Sanitize (prevent injection)
    safe_data = sanitize_input(data)
    
    # Process
    return process(safe_data)
```

### 4. Token Expiration

```python
# Tokens should expire
payload = {
    "user_id": user_id,
    "exp": datetime.utcnow() + timedelta(hours=1)  # Expires in 1 hour
}
token = jwt.encode(payload, SECRET_KEY)

# Client must refresh before expiration
```

### 5. Least Privilege

```python
# Give users minimum required permissions

# ❌ Bad: Admin can do anything
user_scopes = ["admin:*"]

# ✅ Good: User has specific scopes
user_scopes = ["customer:read", "customer:write"]
```

## Interview Summary

**Key Security Concepts:**
- **Transport Layer:** HTTPS/TLS for encryption
- **Authentication:** Who are you? (tokens, keys, JWT)
- **Authorization:** What can you do? (roles, scopes, permissions)
- **Secrets:** Environment variables, never hardcode
- **Validation:** Sanitize all inputs
- **Expiration:** Tokens should expire
- **Least Privilege:** Minimum permissions needed

**One-Liner:**
"MCP security involves transport encryption (HTTPS), authentication (tokens/JWT), authorization (scopes/roles), and never hardcoding secrets."

## Common Interview Questions

**Q1: How do you secure a remote MCP server?**
A: Use HTTPS with valid certificates, implement token-based authentication (JWT or API keys), enforce authorization via scopes or roles, store secrets in environment variables, and validate all inputs.

**Q2: Should authentication happen in the MCP client or server?**
A: Authentication happens on both sides. The client includes credentials (token, API key) in requests. The server validates those credentials.

**Q3: What's the difference between authentication and authorization?**
A: Authentication verifies identity ("who are you?"). Authorization grants permissions ("what can you do?"). You authenticate first, then authorize.

**Q4: How do you handle token expiration?**
A: Include an `exp` field in JWT tokens. The server checks it before processing requests. Clients must refresh tokens before they expire.

**Q5: Can you hardcode API keys in your application?**
A: Never. Use environment variables, secret managers, or configuration files. Hardcoded secrets are security risks if code is committed to version control.

---

**Next:** [14_MCP_Testing_Debugging.md](14_MCP_Testing_Debugging.md) – Testing and debugging MCP systems
