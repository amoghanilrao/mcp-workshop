# Advanced Topics

???+ abstract "Beyond the basics"
    Once you have a working FastMCP server connected to ContextForge, you can add prompts, resources, middleware, authentication, and more sophisticated features.

---

## Prompts

Prompts are reusable templates that help LLMs interact with your tools more effectively.

### Basic prompt

```python
from fastmcp import FastMCP

mcp = FastMCP("advanced-server")

@mcp.prompt(description="Analyze CSV data with guidance")
def analyze_csv_prompt(file_path: str) -> str:
    """Return a structured prompt for CSV analysis."""
    return f"""Please analyze the CSV file at: {file_path}

Follow these steps:
1. Get basic information using get_csv_info
2. Identify key columns and data types
3. Look for missing values or anomalies
4. Suggest relevant analysis questions
"""

@mcp.tool
def get_csv_info(file_path: str) -> dict:
    """Get information about a CSV file."""
    # Implementation here
    return {"columns": [], "rows": 0}
```

### Dynamic prompts

Prompts can include parameters and conditional logic:

```python
@mcp.prompt(description="Generate analysis strategy based on data type")
def data_analysis_strategy(data_type: str, goal: str) -> str:
    """Customize analysis approach."""
    strategies = {
        "sales": "Focus on trends, seasonality, and revenue metrics",
        "customer": "Analyze demographics, behavior patterns, and segmentation",
        "product": "Examine performance, inventory, and market fit"
    }

    strategy = strategies.get(data_type, "Perform general exploratory analysis")
    return f"Goal: {goal}\nStrategy: {strategy}\n\nRecommended tools: ..."
```

???+ tip "When to use prompts"
    - Provide guidance for complex multi-step workflows
    - Standardize common analysis patterns
    - Help LLMs understand domain-specific context
    - Chain multiple tool calls together

---

## Resources

Resources expose data (files, URLs, database queries) that LLMs can read and reference.

### File resources

```python
@mcp.resource(uri="file://data/sample.csv", description="Sample dataset")
def sample_data() -> str:
    """Return sample CSV data."""
    return """name,value,category
Item A,100,Electronics
Item B,200,Books
Item C,150,Electronics"""

@mcp.resource(uri="file://config/{name}.json", description="Configuration files")
def get_config(name: str) -> str:
    """Load configuration by name."""
    import json
    with open(f"configs/{name}.json") as f:
        return f.read()
```

### Dynamic resources

```python
import aiofiles

@mcp.resource(
    uri="report://{report_id}",
    description="Fetch generated reports",
    mime_type="text/markdown"
)
async def get_report(report_id: str) -> str:
    """Load a report by ID."""
    async with aiofiles.open(f"reports/{report_id}.md", "r") as f:
        return await f.read()
```

### Directory resources (FastMCP 2.13+)

```python
@mcp.resource(uri="docs://", description="Documentation directory")
async def docs_directory() -> list:
    """List all documentation files."""
    # Returns list of file metadata
    pass
```

???+ info "Resource URIs"
    Use RFC 3986 URI format. Common patterns:
    - `file://path/to/file`
    - `http://example.com/api/data`
    - `custom://resource/{id}`
    - Template variables in `{brackets}` become function parameters

---

## Middleware & Caching

### Response caching (FastMCP 2.13+)

Cache expensive tool calls to improve performance:

```python
from fastmcp import FastMCP
from fastmcp.middleware import CachingMiddleware

mcp = FastMCP("cached-server")

# Add caching middleware
mcp.add_middleware(
    CachingMiddleware(
        ttl=300,  # Cache for 5 minutes
        max_size=100  # Keep up to 100 responses
    )
)

@mcp.tool
def expensive_analysis(data: str) -> dict:
    """This will be cached automatically."""
    # Expensive computation here
    return {"result": "..."}
```

### Custom middleware

```python
from fastmcp.middleware import Middleware

class LoggingMiddleware(Middleware):
    async def on_tool_call(self, tool_name: str, arguments: dict) -> None:
        """Log every tool call."""
        print(f"Tool called: {tool_name} with {arguments}")

    async def on_tool_result(self, tool_name: str, result: any) -> any:
        """Log and optionally modify results."""
        print(f"Tool {tool_name} returned: {result}")
        return result

mcp.add_middleware(LoggingMiddleware())
```

---

## Authentication

### Bearer token auth

```python
from fastmcp import FastMCP
from fastmcp.auth import BearerAuth

mcp = FastMCP("secure-server")

# Require authentication
mcp.set_auth(BearerAuth(token="your-secret-token"))

# Or use environment variable
import os
mcp.set_auth(BearerAuth(token=os.environ.get("MCP_AUTH_TOKEN")))
```

### OAuth providers (FastMCP 2.13+)

```python
from fastmcp.auth import GitHubOAuth

mcp.set_auth(
    GitHubOAuth(
        client_id=os.environ["GITHUB_CLIENT_ID"],
        client_secret=os.environ["GITHUB_CLIENT_SECRET"],
    )
)
```

Supported providers:
- GitHub
- Google
- Azure (Entra ID)
- AWS Cognito
- Auth0
- WorkOS/AuthKit
- Descope
- Scalekit

???+ warning "Production auth"
    - Store secrets in environment variables or secret managers
    - Use HTTPS in production
    - Enable token rotation
    - Review [FastMCP auth docs](https://gofastmcp.com/servers/auth/token-verification) for detailed configuration

---

## Storage & State

FastMCP 2.13+ includes pluggable storage backends via [py-key-value-aio](https://github.com/strawgate/py-key-value).

### Basic key-value storage

```python
from fastmcp import FastMCP

mcp = FastMCP("stateful-server")

@mcp.tool
async def save_preference(key: str, value: str) -> str:
    """Store user preference."""
    await mcp.storage.set(key, value)
    return f"Saved {key}={value}"

@mcp.tool
async def get_preference(key: str) -> str:
    """Retrieve user preference."""
    value = await mcp.storage.get(key, default="not found")
    return value
```

### Encrypted storage

```python
from py_key_value import EncryptedStore, FilesystemStore

# Configure encrypted filesystem storage
mcp.storage = EncryptedStore(
    backend=FilesystemStore(base_path="./data"),
    encryption_key=os.environ["STORAGE_KEY"]
)
```

### TTL and caching

```python
from py_key_value import TTLStore, InMemoryStore

# Add TTL wrapper
mcp.storage = TTLStore(
    backend=InMemoryStore(),
    default_ttl=3600  # 1 hour
)

await mcp.storage.set("session", data, ttl=300)  # Override to 5 minutes
```

---

## Server Lifespans (FastMCP 2.13+)

Proper initialization and cleanup hooks that run once per server instance:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app):
    """Initialize resources on startup, cleanup on shutdown."""
    # Startup
    db = await connect_database()
    cache = await init_cache()

    # Make available to tools
    app.state.db = db
    app.state.cache = cache

    yield  # Server runs

    # Shutdown
    await db.close()
    await cache.close()

mcp = FastMCP("db-server", lifespan=lifespan)

@mcp.tool
async def query_db(sql: str) -> list:
    """Run SQL query."""
    # Access via context
    result = await mcp.app.state.db.execute(sql)
    return result
```

???+ note "Breaking change in 2.13"
    Previously `lifespan` ran per-client. Now it runs once per server. Update your code if migrating from earlier versions.

---

## Input Validation

FastMCP uses Pydantic for automatic validation:

```python
from pydantic import Field, field_validator

@mcp.tool
def analyze_data(
    data_type: str = Field(
        ...,
        pattern="^(sales|customer|product)$",
        description="Type of data to analyze"
    ),
    threshold: float = Field(
        default=0.5,
        ge=0.0,
        le=1.0,
        description="Confidence threshold (0.0-1.0)"
    ),
    max_rows: int = Field(default=1000, le=10000)
) -> dict:
    """Analyze data with validated parameters."""
    return {"type": data_type, "threshold": threshold}
```

### Custom validators

```python
from pydantic import field_validator

@mcp.tool
def process_file(file_path: str) -> str:
    """Process a file with path validation."""

    @field_validator('file_path')
    def validate_path(cls, v):
        if not v.endswith('.csv'):
            raise ValueError("Only CSV files allowed")
        return v
```

---

## Custom Routes

Add health checks or metrics endpoints:

```python
from fastapi import Request

@mcp.custom_route("/health", methods=["GET"])
async def health_check():
    """Health check endpoint."""
    return {"status": "healthy", "version": "1.0.0"}

@mcp.custom_route("/metrics", methods=["GET"])
async def metrics(request: Request):
    """Expose Prometheus metrics."""
    return {
        "tool_calls": request.app.state.get("tool_call_count", 0),
        "uptime": request.app.state.get("uptime", 0)
    }
```

---

## Testing

FastMCP includes a test harness for unit testing:

```python
import pytest
from fastmcp.testing import create_test_client

@pytest.mark.asyncio
async def test_tool():
    """Test tools without running a server."""
    from server import mcp

    async with create_test_client(mcp) as client:
        # Test tool listing
        tools = await client.list_tools()
        assert any(t.name == "echo" for t in tools)

        # Test tool execution
        result = await client.call_tool("echo", {"text": "test"})
        assert result.content[0].text == "test"
```

---

## Enterprise Considerations

When deploying MCP in production:

### Security
- **Input validation** - Validate all tool inputs with Pydantic
- **Rate limiting** - Prevent abuse with middleware
- **Authentication** - Use OAuth providers with token rotation
- **Audit logging** - Track all tool invocations
- **Secret management** - Use vault solutions (AWS Secrets Manager, HashiCorp Vault)

### Scalability
- **Async operations** - Use `async def` for I/O-bound tools
- **Connection pooling** - Reuse database/API connections
- **Caching** - Cache expensive operations with TTLs
- **Horizontal scaling** - Deploy multiple instances behind a gateway

### Monitoring
- **OpenTelemetry** - Instrument with traces and metrics
- **Health checks** - Add `/health` endpoints
- **Error tracking** - Integrate Sentry or similar
- **Performance metrics** - Monitor tool execution times

???+ info "Enterprise Architecture Guide"
    For comprehensive guidance on architecting secure, scalable MCP deployments in enterprise environments, see [Architecting Secure Enterprise AI Agents with MCP](https://ibm.biz/enterprise-ai-with-mcp).

---

---

## ContextForge Integration

### Gateway architecture

ContextForge is a production MCP gateway that provides:

- **Centralized routing** - One endpoint for multiple MCP servers
- **Authentication** - Bearer token auth, OAuth integration
- **Rate limiting** - Prevent abuse and manage costs
- **Observability** - Centralized logging and metrics
- **Virtual servers** - Compose tools from multiple servers

### Register a server with ContextForge

```bash
# Set up authentication
export MCPGATEWAY_BEARER_TOKEN="your-token-here"

# Register your server
curl -X POST http://localhost:4444/servers \
  -H "Authorization: Bearer $MCPGATEWAY_BEARER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-server",
    "url": "http://localhost:8000/mcp",
    "description": "My production MCP server",
    "tags": ["production", "v1"]
  }'
```

### Virtual server composition

Combine tools from multiple servers into a virtual server:

```python
# Virtual server config
{
  "name": "data-pipeline",
  "description": "Complete data analysis pipeline",
  "servers": [
    {
      "name": "csv-server",
      "tools": ["read_csv", "write_csv"]
    },
    {
      "name": "analysis-server",
      "tools": ["analyze_data", "generate_report"]
    },
    {
      "name": "visualization-server",
      "tools": ["create_chart"]
    }
  ]
}
```

Agents see `data-pipeline` as a single MCP server with 5 tools from 3 different backends.

### Authentication with ContextForge

```python
# Client connecting to ContextForge
from fastmcp import Client
from fastmcp.client.auth import BearerAuth
import os

async def main():
    async with Client(
        "http://localhost:4444/mcp",
        auth=BearerAuth(token=os.environ["MCPGATEWAY_BEARER_TOKEN"])
    ) as client:
        # All tools from all registered servers
        tools = await client.list_tools()
        print(f"Available tools: {len(tools)}")
```

---

## Rate Limiting

### Implement rate limiting middleware

```python
# src/my_server/rate_limiting.py
from fastapi import Request, HTTPException
from collections import defaultdict
from datetime import datetime, timedelta
import asyncio

class RateLimiter:
    """Simple in-memory rate limiter."""

    def __init__(self, requests_per_minute: int = 60):
        self.requests_per_minute = requests_per_minute
        self.requests = defaultdict(list)

    async def check_rate_limit(self, client_id: str):
        """Check if client has exceeded rate limit."""
        now = datetime.utcnow()
        minute_ago = now - timedelta(minutes=1)

        # Clean old requests
        self.requests[client_id] = [
            req_time for req_time in self.requests[client_id]
            if req_time > minute_ago
        ]

        # Check limit
        if len(self.requests[client_id]) >= self.requests_per_minute:
            raise HTTPException(
                status_code=429,
                detail=f"Rate limit exceeded: {self.requests_per_minute} requests/minute"
            )

        # Record this request
        self.requests[client_id].append(now)

# Add to FastMCP server
rate_limiter = RateLimiter(requests_per_minute=60)

@mcp.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    """Rate limit requests by IP."""
    client_ip = request.client.host
    await rate_limiter.check_rate_limit(client_ip)
    return await call_next(request)
```

### Production rate limiting with Redis

```python
import redis.asyncio as redis
from datetime import timedelta

class RedisRateLimiter:
    """Distributed rate limiter using Redis."""

    def __init__(self, redis_url: str, requests_per_minute: int = 60):
        self.redis = redis.from_url(redis_url)
        self.requests_per_minute = requests_per_minute

    async def check_rate_limit(self, client_id: str):
        """Check rate limit using Redis."""
        key = f"rate_limit:{client_id}"

        # Increment counter
        current = await self.redis.incr(key)

        # Set expiry on first request
        if current == 1:
            await self.redis.expire(key, 60)

        # Check limit
        if current > self.requests_per_minute:
            ttl = await self.redis.ttl(key)
            raise HTTPException(
                status_code=429,
                detail=f"Rate limit exceeded. Try again in {ttl} seconds."
            )
```

---

## Next Steps

### Production Guides
- **[Testing](testing.md)** - Comprehensive testing strategies
- **[Resilience](resilience.md)** - Production-grade error handling
- **[CI/CD](cicd.md)** - Automated deployment pipelines

### Documentation
- **[FastMCP Official Docs](https://gofastmcp.com/getting-started/welcome)** - Comprehensive guides and API reference
- **[MCP Specification](https://spec.modelcontextprotocol.io/)** - Official protocol documentation
- **[ContextForge Docs](https://ibm.github.io/mcp-context-forge/)** - Gateway deployment and configuration

### Code & Examples
- **[Sample Servers](https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers/python)** - 20+ production-ready examples
- **[ContextForge Gateway](https://github.com/IBM/mcp-context-forge)** - Gateway source code
- **[MCP Servers Registry](https://github.com/modelcontextprotocol/servers)** - Community servers

### Enterprise & Production
- **[Enterprise AI with MCP](https://ibm.biz/enterprise-ai-with-mcp)** - Architecture patterns and security
- **[Debugging Guide](debugging.md)** - Troubleshooting and diagnostics
