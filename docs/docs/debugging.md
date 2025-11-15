# Debugging MCP Servers

???+ abstract "When things go wrong"
    Systematic approaches to diagnosing and fixing common issues with FastMCP servers, ContextForge Gateway, and MCP clients.

---

## Quick Diagnostic Checklist

When your MCP setup isn't working, check these in order:

- [ ] Python version is 3.11+ (`python3 --version`)
- [ ] FastMCP server starts without errors
- [ ] Server responds to basic health checks
- [ ] ContextForge Gateway is running and accessible
- [ ] Bearer token is valid and properly set
- [ ] Server is registered in the Gateway
- [ ] Network connectivity between components
- [ ] No port conflicts (8000, 4444, etc.)

---

## Common Errors & Solutions

### 1. Server Won't Start

#### "No module named 'fastmcp'"

**Cause**: FastMCP not installed in the active environment

**Solution**:
```bash
# Check which Python is active
which python3
python3 -c "import fastmcp; print(fastmcp.__version__)"

# Install FastMCP
uv add fastmcp
# or
pip install fastmcp
```

#### "Port already in use"

**Cause**: Another process is using port 8000 (or your configured port)

**Solution**:
```bash
# Find what's using the port (Linux/macOS)
lsof -i :8000
sudo netstat -tuln | grep 8000

# Kill the process or use a different port
fastmcp run server.py --transport http --port 8001
```

#### "No FastMCP instance found"

**Cause**: The CLI can't find your `mcp` variable

**Solution**:
```python
# Make sure your server.py has module-level instance
from fastmcp import FastMCP

mcp = FastMCP("my-server")  # Must be named 'mcp', 'server', or 'app'

# Not inside a function:
# def create_server():  # ❌ Won't work
#     mcp = FastMCP("my-server")
```

---

### 2. Connection Issues

#### "Connection refused" when calling tools

**Check if server is running**:
```bash
# Test STDIO server
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | python server.py

# Test HTTP server
curl http://localhost:8000/mcp
```

**Expected response**: JSON-RPC response, not an error

#### "401 Unauthorized" from Gateway

**Cause**: Missing or invalid bearer token

**Solution**:
```bash
# Generate new token
cd /path/to/mcp-context-forge
uv run python -m mcpgateway.tokens issue --subject debug-session

# Export token
export MCPGATEWAY_BEARER_TOKEN="eyJ..."

# Verify it's set
echo $MCPGATEWAY_BEARER_TOKEN

# Test with curl
curl -H "Authorization: Bearer $MCPGATEWAY_BEARER_TOKEN" \
     http://localhost:4444/servers
```

#### "Server inactive" in Gateway

**Cause**: Gateway can't reach your FastMCP server

**Debugging steps**:
```bash
# 1. Verify FastMCP is running
curl http://localhost:8000/mcp

# 2. Check from Gateway container (if using Docker/Podman)
podman exec -it <gateway-container> curl http://host.docker.internal:8000/mcp
# or for Linux
podman exec -it <gateway-container> curl http://172.17.0.1:8000/mcp

# 3. Check Gateway logs
podman logs <gateway-container> -f
# or
tail -f /path/to/gateway/logs/app.log
```

---

### 3. Tool Execution Errors

#### Tools listed but not callable

**Symptom**: `tools/list` works, `tools/call` fails

**Debug**:
```python
# Add logging to your tool
import logging
logger = logging.getLogger(__name__)

@mcp.tool
def debug_tool(input: str) -> str:
    """Debug tool with logging."""
    logger.info(f"Tool called with: {input}")
    try:
        result = process(input)
        logger.info(f"Tool returned: {result}")
        return result
    except Exception as e:
        logger.error(f"Tool failed: {e}", exc_info=True)
        raise
```

**Run with debug logging**:
```bash
FASTMCP_LOG_LEVEL=DEBUG fastmcp run server.py --transport http
```

#### "Validation error" on tool call

**Cause**: Arguments don't match the function signature

**Solution**:
```python
from pydantic import Field

@mcp.tool
def strict_tool(
    required_param: str,  # Must be provided
    optional_param: str = "default",  # Has default
    validated_param: int = Field(ge=0, le=100)  # Must be 0-100
) -> str:
    """Check parameter requirements match what you're sending."""
    return f"{required_param}, {optional_param}, {validated_param}"
```

**Test with FastMCP client**:
```python
async with Client("http://localhost:8000/mcp") as client:
    # This will fail - missing required_param
    # result = await client.call_tool("strict_tool", {})

    # This works
    result = await client.call_tool("strict_tool", {
        "required_param": "test",
        "validated_param": 50
    })
```

---

## Debugging Techniques

### 1. Enable verbose logging

**FastMCP server**:
```bash
# Environment variable
export FASTMCP_LOG_LEVEL=DEBUG
fastmcp run server.py

# Or in code
import logging
logging.basicConfig(level=logging.DEBUG)
```

**ContextForge Gateway**:
```bash
# In .env file
LOG_LEVEL=DEBUG

# Or environment variable
LOG_LEVEL=DEBUG make serve
```

### 2. Test with curl

**List tools**:
```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list"
  }'
```

**Call a tool**:
```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "echo",
      "arguments": {"text": "hello"}
    }
  }'
```

### 3. Use the FastMCP test client

```python
import asyncio
from fastmcp import Client

async def debug():
    """Interactive debugging session."""
    async with Client("http://localhost:8000/mcp") as client:
        # Test connectivity
        await client.ping()
        print("✓ Server is reachable")

        # List available tools
        tools = await client.list_tools()
        print(f"✓ Found {len(tools)} tools:")
        for tool in tools:
            print(f"  - {tool.name}: {tool.description}")

        # Test tool call
        result = await client.call_tool("echo", {"text": "test"})
        print(f"✓ Tool result: {result.content[0].text}")

asyncio.run(debug())
```

### 4. Inspect MCP messages

**Add request/response logging**:
```python
from fastmcp import FastMCP
from fastapi import Request, Response
import json

mcp = FastMCP("debug-server")

@mcp.middleware("http")
async def log_requests(request: Request, call_next):
    """Log all MCP requests and responses."""
    body = await request.body()
    print(f"Request: {body.decode()}")

    response = await call_next(request)

    # Log response (requires buffering)
    print(f"Response status: {response.status_code}")

    return response
```

### 5. Use MCP Inspector

The official [MCP Inspector](https://github.com/modelcontextprotocol/inspector) provides a GUI for testing:

```bash
# Install and run
npx @modelcontextprotocol/inspector python server.py

# Or for HTTP servers
npx @modelcontextprotocol/inspector http://localhost:8000/mcp
```

---

## Gateway-Specific Debugging

### Check Gateway health

```bash
# Version and status
curl -k http://localhost:4444/version

# List registered servers
curl -H "Authorization: Bearer $MCPGATEWAY_BEARER_TOKEN" \
     http://localhost:4444/servers

# Check specific server status
curl -H "Authorization: Bearer $MCPGATEWAY_BEARER_TOKEN" \
     http://localhost:4444/servers/echo-server
```

### View Gateway logs

```bash
# Docker/Podman
podman logs mcp-gateway -f --tail 100

# Local installation
tail -f logs/mcpgateway.log

# With filtering
podman logs mcp-gateway 2>&1 | grep ERROR
```

### Test Gateway → Server connectivity

```bash
# From inside Gateway container
podman exec -it mcp-gateway bash

# Test reaching your FastMCP server
curl http://host.docker.internal:8000/mcp
# or
curl http://172.17.0.1:8000/mcp
```

### Database issues

```bash
# Check SQLite database
sqlite3 mcp.db "SELECT * FROM servers;"
sqlite3 mcp.db "SELECT * FROM tools;"

# Reset database (⚠️ deletes all data)
rm mcp.db
make migrate  # Re-run migrations
```

---

## Transport-Specific Issues

### STDIO Transport

**Problem**: Server starts but client can't connect

**Debug**:
```bash
# Test directly with echo
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | python server.py

# Expected: Valid JSON response
# Not: Python errors, empty output, or non-JSON
```

**Common issue**: Print statements in tool code
```python
# ❌ Breaks STDIO transport
@mcp.tool
def bad_tool(x: int) -> int:
    print("Debug message")  # Interferes with JSON-RPC!
    return x * 2

# ✅ Use logging instead
import logging
logger = logging.getLogger(__name__)

@mcp.tool
def good_tool(x: int) -> int:
    logger.info("Debug message")
    return x * 2
```

### HTTP Transport

**Problem**: Server binds to wrong interface

```python
# ❌ Only localhost can connect
mcp.run(transport="http", host="127.0.0.1")

# ✅ Accept connections from containers/network
mcp.run(transport="http", host="0.0.0.0", port=8000)
```

**Problem**: Path mismatch

```python
# Server running at /api/mcp
mcp.run(transport="http", path="/api/mcp")

# ❌ Client connecting to wrong path
client = Client("http://localhost:8000/mcp")  # Wrong!

# ✅ Match the server path
client = Client("http://localhost:8000/api/mcp")
```

---

## Performance Debugging

### Slow tool execution

**Add timing**:
```python
import time
from functools import wraps

def timing_decorator(func):
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.time()
        result = await func(*args, **kwargs)
        duration = time.time() - start
        print(f"{func.__name__} took {duration:.2f}s")
        return result
    return wrapper

@mcp.tool
@timing_decorator
async def slow_tool(data: str) -> dict:
    """Debug slow operations."""
    # Your code here
    pass
```

### Memory issues

```python
import tracemalloc

# Start tracing
tracemalloc.start()

@mcp.tool
def memory_intensive(size: int) -> str:
    """Track memory usage."""
    current, peak = tracemalloc.get_traced_memory()
    print(f"Current: {current / 1024 / 1024:.1f} MB")
    print(f"Peak: {peak / 1024 / 1024:.1f} MB")

    # Your code
    big_list = [0] * size

    current, peak = tracemalloc.get_traced_memory()
    print(f"After allocation: {current / 1024 / 1024:.1f} MB")

    return "done"
```

---

## Environment Issues

### Python environment confusion

```bash
# Verify which Python/packages you're using
which python3
python3 -m pip list | grep fastmcp
python3 -c "import sys; print(sys.prefix)"

# Create clean venv
python3 -m venv .venv
source .venv/bin/activate
pip install fastmcp
```

### UV vs pip conflicts

```bash
# Prefer uv for consistency
uv venv
source .venv/bin/activate
uv pip install fastmcp

# Or use uv run to auto-manage
uv run fastmcp run server.py
```

---

## Getting Help

**Before asking for help, collect**:

1. **Error messages** (full traceback)
2. **FastMCP version**: `python -c "import fastmcp; print(fastmcp.__version__)"`
3. **Python version**: `python --version`
4. **Minimal reproduction** (simplest server.py that shows the issue)
5. **What you tried** (debugging steps from this guide)

**Community Resources**:

- **[FastMCP Discussions](https://github.com/jlowin/fastmcp/discussions)** - Ask questions and share solutions
- **[ContextForge Issues](https://github.com/IBM/mcp-context-forge/issues)** - Report Gateway bugs
- **[MCP Official](https://modelcontextprotocol.io/)** - Protocol overview and resources

**Documentation**:

- **[FastMCP Docs](https://gofastmcp.com/getting-started/welcome)** - Framework documentation
- **[MCP Specification](https://spec.modelcontextprotocol.io/)** - Protocol specification
- **[ContextForge Docs](https://ibm.github.io/mcp-context-forge/)** - Gateway guides
- **[Enterprise MCP Guide](https://ibm.biz/enterprise-ai-with-mcp)** - Production patterns and architecture

---

## Debugging Examples

### Example 1: Tool not appearing in Gateway

**Symptom**: Tool works in FastMCP but doesn't show in Gateway

**Debug steps**:
```bash
# 1. Verify tool locally
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'

# 2. Check Gateway registration
curl -H "Authorization: Bearer $TOKEN" \
     http://localhost:4444/servers/my-server

# 3. Force Gateway to refresh
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:4444/servers/my-server/refresh

# 4. Check Gateway can reach server
curl -X POST http://localhost:4444/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

### Example 2: Type validation failing

**Symptom**: Tool call returns validation error

**Debug**:
```python
# Add detailed validation
from pydantic import Field, field_validator

@mcp.tool
def validated_tool(
    email: str = Field(pattern=r"^[\w\.-]+@[\w\.-]+\.\w+$"),
    age: int = Field(ge=0, le=150)
) -> str:
    """Test with known good/bad inputs."""
    return f"Valid: {email}, {age}"

# Test systematically
test_cases = [
    ({"email": "test@example.com", "age": 25}, True),   # Should work
    ({"email": "invalid", "age": 25}, False),            # Bad email
    ({"email": "test@example.com", "age": -5}, False),   # Bad age
    ({"email": "test@example.com", "age": 200}, False),  # Age too high
]
```

---

## Next Steps

- **Advanced Topics**: [advanced-topics.md](advanced-topics.md)
- **FastMCP Docs**: https://gofastmcp.com/getting-started/welcome
- **Sample Servers**: https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers/python
