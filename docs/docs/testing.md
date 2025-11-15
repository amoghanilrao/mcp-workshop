# Testing MCP Servers

???+ abstract "The Reality Check"
    Building MCP servers is fun. Making them production-ready means writing tests that catch bugs before your users do. This guide covers unit testing, integration testing, mocking JSON-RPC, and achieving 90%+ coverage with pytest.

---

## Why Test MCP Servers?

MCP servers are critical infrastructure for AI agents. They need to:

- **Handle malformed inputs gracefully** - LLMs can send unexpected arguments
- **Fail fast with clear error messages** - Poor error handling breaks the agent loop
- **Maintain contracts** - Tool schemas must match implementation
- **Work reliably in production** - No surprises when connected to ContextForge

Well-tested servers save you from debugging production incidents at 2am.

---

## Testing Stack

| Tool | Purpose | Why Use It |
|------|---------|------------|
| **pytest** | Test runner and framework | Industry standard, great fixtures, async support |
| **pytest-asyncio** | Async test support | Essential for testing FastMCP's async tools |
| **pytest-cov** | Coverage reporting | Track which code paths are tested |
| **pytest-mock** | Mocking and patching | Isolate tests from external dependencies |
| **fastmcp.testing** | MCP test client | Test tools without running servers |
| **httpx** | HTTP testing | Integration tests for HTTP transport |

Install all at once:

```bash
uv add --dev pytest pytest-asyncio pytest-cov pytest-mock httpx
```

---

## Project Structure

```
my-mcp-server/
├── src/
│   └── my_server/
│       ├── __init__.py
│       ├── server.py          # FastMCP server definition
│       └── tools/
│           ├── __init__.py
│           └── calculator.py   # Tool implementations
├── tests/
│   ├── __init__.py
│   ├── conftest.py            # Shared fixtures
│   ├── test_tools.py          # Unit tests for tools
│   ├── test_server.py         # Integration tests
│   └── test_contract.py       # Schema validation tests
├── pyproject.toml
└── Makefile                   # make test
```

---

## Unit Testing Tools

### Basic tool test

```python
# tests/test_tools.py
import pytest
from my_server.tools.calculator import add, divide

def test_add():
    """Test basic addition."""
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
    assert add(0, 0) == 0

def test_divide():
    """Test division with error handling."""
    assert divide(10, 2) == 5.0
    assert divide(7, 2) == 3.5

    with pytest.raises(ValueError, match="division by zero"):
        divide(10, 0)
```

### Testing async tools

```python
# tests/test_tools.py
import pytest
from my_server.tools.api import fetch_data

@pytest.mark.asyncio
async def test_fetch_data():
    """Test async data fetching."""
    result = await fetch_data("https://api.example.com/data")
    assert result["status"] == "success"
    assert "data" in result
```

### Testing with FastMCP test client

```python
# tests/test_server.py
import pytest
from fastmcp.testing import create_test_client
from my_server.server import mcp

@pytest.mark.asyncio
async def test_tool_registration():
    """Test that tools are properly registered."""
    async with create_test_client(mcp) as client:
        tools = await client.list_tools()
        tool_names = [t.name for t in tools]

        assert "add" in tool_names
        assert "divide" in tool_names

@pytest.mark.asyncio
async def test_tool_execution():
    """Test tool execution through MCP protocol."""
    async with create_test_client(mcp) as client:
        result = await client.call_tool("add", {"a": 5, "b": 3})
        assert result.content[0].text == "8"
```

---

## Mocking External Dependencies

### Mock HTTP calls

```python
# tests/test_api_tools.py
import pytest
from unittest.mock import AsyncMock, patch
from my_server.tools.weather import get_weather

@pytest.mark.asyncio
async def test_get_weather_success(httpx_mock):
    """Mock external weather API."""
    httpx_mock.add_response(
        url="https://api.weather.com/v1/current",
        json={"temp": 72, "conditions": "sunny"}
    )

    result = await get_weather("San Francisco")
    assert result["temp"] == 72
    assert result["conditions"] == "sunny"

@pytest.mark.asyncio
async def test_get_weather_api_error(httpx_mock):
    """Test error handling when API fails."""
    httpx_mock.add_response(
        url="https://api.weather.com/v1/current",
        status_code=500
    )

    with pytest.raises(Exception, match="Weather API unavailable"):
        await get_weather("Invalid City")
```

### Mock database operations

```python
# tests/test_db_tools.py
import pytest
from unittest.mock import AsyncMock, patch
from my_server.tools.database import save_record, get_record

@pytest.fixture
def mock_db():
    """Mock database connection."""
    with patch("my_server.tools.database.get_db") as mock:
        db = AsyncMock()
        mock.return_value = db
        yield db

@pytest.mark.asyncio
async def test_save_record(mock_db):
    """Test saving a record with mocked database."""
    mock_db.execute.return_value = {"id": 123}

    result = await save_record({"name": "test"})

    assert result["id"] == 123
    mock_db.execute.assert_called_once()
```

---

## Pytest Fixtures for MCP

### Shared test fixtures

```python
# tests/conftest.py
import pytest
from fastmcp import FastMCP
from fastmcp.testing import create_test_client
from my_server.server import mcp

@pytest.fixture
def sample_data():
    """Reusable test data."""
    return {
        "users": [
            {"id": 1, "name": "Alice"},
            {"id": 2, "name": "Bob"},
        ]
    }

@pytest.fixture
async def mcp_client():
    """Reusable MCP test client."""
    async with create_test_client(mcp) as client:
        yield client

@pytest.fixture
def mock_env(monkeypatch):
    """Mock environment variables."""
    monkeypatch.setenv("API_KEY", "test-key-12345")
    monkeypatch.setenv("API_URL", "https://test.example.com")
```

### Using fixtures in tests

```python
# tests/test_with_fixtures.py
import pytest

@pytest.mark.asyncio
async def test_list_users(mcp_client, sample_data):
    """Test using shared fixtures."""
    result = await mcp_client.call_tool("list_users", {})
    users = eval(result.content[0].text)

    assert len(users) == len(sample_data["users"])
    assert users[0]["name"] == "Alice"

def test_api_key_loaded(mock_env):
    """Test environment variable mocking."""
    import os
    assert os.getenv("API_KEY") == "test-key-12345"
```

---

## Integration Testing

### Test HTTP transport

```python
# tests/test_integration.py
import pytest
import httpx
from multiprocessing import Process
import time
from my_server.server import mcp

def run_server():
    """Helper to run server in subprocess."""
    mcp.run(transport="http", host="127.0.0.1", port=8765)

@pytest.fixture(scope="module")
def http_server():
    """Start HTTP server for integration tests."""
    proc = Process(target=run_server, daemon=True)
    proc.start()
    time.sleep(2)  # Wait for server to start
    yield "http://127.0.0.1:8765/mcp"
    proc.terminate()

@pytest.mark.asyncio
async def test_http_tools_list(http_server):
    """Test listing tools via HTTP."""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            http_server,
            json={
                "jsonrpc": "2.0",
                "id": 1,
                "method": "tools/list"
            }
        )

        assert response.status_code == 200
        data = response.json()
        assert "result" in data
        assert len(data["result"]["tools"]) > 0

@pytest.mark.asyncio
async def test_http_tool_call(http_server):
    """Test calling a tool via HTTP."""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            http_server,
            json={
                "jsonrpc": "2.0",
                "id": 2,
                "method": "tools/call",
                "params": {
                    "name": "add",
                    "arguments": {"a": 10, "b": 5}
                }
            }
        )

        assert response.status_code == 200
        data = response.json()
        assert data["result"]["content"][0]["text"] == "15"
```

### Test with mcp-cli

```bash
# tests/test_mcp_cli.sh
#!/bin/bash

# Start server in background
fastmcp run server.py --transport http --port 8888 &
SERVER_PID=$!
sleep 2

# Test with mcp-cli
mcp-cli --url http://localhost:8888/mcp tools list > tools.json
mcp-cli --url http://localhost:8888/mcp tools call add '{"a": 5, "b": 3}' > result.json

# Verify results
cat tools.json | jq '.tools[] | .name' | grep "add"
cat result.json | jq '.content[0].text' | grep "8"

# Cleanup
kill $SERVER_PID
rm tools.json result.json

echo "✓ mcp-cli integration tests passed"
```

Run from pytest:

```python
# tests/test_cli_integration.py
import subprocess

def test_mcp_cli_integration():
    """Run mcp-cli integration tests."""
    result = subprocess.run(
        ["bash", "tests/test_mcp_cli.sh"],
        capture_output=True,
        text=True
    )
    assert result.returncode == 0
    assert "✓ mcp-cli integration tests passed" in result.stdout
```

---

## Contract Testing

### Validate tool schemas

```python
# tests/test_contract.py
import pytest
from pydantic import ValidationError
from my_server.server import mcp
from fastmcp.testing import create_test_client

@pytest.mark.asyncio
async def test_tool_schemas():
    """Ensure tool parameters match Pydantic schemas."""
    async with create_test_client(mcp) as client:
        tools = await client.list_tools()

        for tool in tools:
            # Verify schema has required fields
            assert "name" in tool.dict()
            assert "description" in tool.dict()
            assert "inputSchema" in tool.dict()

            # Validate JSON Schema structure
            schema = tool.inputSchema
            assert schema["type"] == "object"
            assert "properties" in schema

@pytest.mark.asyncio
async def test_parameter_validation():
    """Test that invalid parameters are rejected."""
    async with create_test_client(mcp) as client:
        # Missing required parameter
        with pytest.raises(Exception):
            await client.call_tool("add", {"a": 5})  # Missing 'b'

        # Wrong type
        with pytest.raises(Exception):
            await client.call_tool("add", {"a": "not a number", "b": 5})
```

### Snapshot testing

```python
# tests/test_snapshots.py
import pytest
import json
from my_server.server import mcp
from fastmcp.testing import create_test_client

@pytest.mark.asyncio
async def test_tool_list_snapshot(snapshot):
    """Ensure tool list doesn't change unexpectedly."""
    async with create_test_client(mcp) as client:
        tools = await client.list_tools()
        tool_data = [
            {"name": t.name, "description": t.description}
            for t in tools
        ]

        # Compare against saved snapshot
        snapshot.assert_match(json.dumps(tool_data, indent=2), "tools.json")
```

---

## Coverage Targets

### Achieve 90%+ coverage

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = """
    --cov=src
    --cov-report=term-missing
    --cov-report=html
    --cov-fail-under=90
    -v
"""

[tool.coverage.run]
branch = true
source = ["src"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
]
```

### Run tests with coverage

```bash
# Run all tests with coverage
pytest

# Generate HTML coverage report
pytest --cov-report=html
open htmlcov/index.html

# Coverage for specific module
pytest tests/test_tools.py --cov=src/my_server/tools

# Show missing lines
pytest --cov-report=term-missing
```

### Makefile targets

```makefile
# Makefile
.PHONY: test test-cov test-fast

test:
	pytest

test-cov:
	pytest --cov=src --cov-report=html --cov-report=term
	@echo "Coverage report: htmlcov/index.html"

test-fast:
	pytest -x --tb=short

test-watch:
	pytest-watch
```

---

## Async Testing Best Practices

### Configure pytest-asyncio

```python
# tests/conftest.py
import pytest

pytest_plugins = ('pytest_asyncio',)

@pytest.fixture(scope="session")
def event_loop_policy():
    """Use consistent event loop policy."""
    import asyncio
    return asyncio.get_event_loop_policy()
```

### Test concurrent operations

```python
# tests/test_async.py
import pytest
import asyncio
from my_server.tools.batch import process_batch

@pytest.mark.asyncio
async def test_concurrent_processing():
    """Test processing multiple items concurrently."""
    items = [1, 2, 3, 4, 5]

    results = await process_batch(items)

    assert len(results) == len(items)
    assert all(r["status"] == "success" for r in results)

@pytest.mark.asyncio
async def test_timeout_handling():
    """Test that timeouts are handled correctly."""
    with pytest.raises(asyncio.TimeoutError):
        await asyncio.wait_for(
            slow_operation(),
            timeout=1.0
        )
```

---

## Testing Error Handling

### Test graceful failures

```python
# tests/test_errors.py
import pytest
from my_server.tools.validator import validate_email, validate_url

def test_validation_errors():
    """Test that validation errors are raised appropriately."""
    with pytest.raises(ValueError, match="Invalid email"):
        validate_email("not-an-email")

    with pytest.raises(ValueError, match="Invalid URL"):
        validate_url("ht!tp://bad-url")

@pytest.mark.asyncio
async def test_error_responses(mcp_client):
    """Test that tools return proper error responses."""
    result = await mcp_client.call_tool("divide", {"a": 10, "b": 0})

    # Should return error content, not raise exception
    assert result.isError == True
    assert "division by zero" in result.content[0].text.lower()
```

---

## Continuous Testing

### GitHub Actions workflow

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install uv
        run: curl -LsSf https://astral.sh/uv/install.sh | sh

      - name: Install dependencies
        run: |
          uv venv
          source .venv/bin/activate
          uv pip install -e ".[dev]"

      - name: Run tests
        run: |
          source .venv/bin/activate
          pytest --cov=src --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
```

### Pre-commit hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: pytest
        language: system
        pass_filenames: false
        always_run: true
```

---

## Test Organization Tips

### Naming conventions

```python
# ✅ Good test names
def test_add_positive_numbers()
def test_add_negative_numbers()
def test_divide_by_zero_raises_error()

# ❌ Poor test names
def test1()
def test_function()
def test_it_works()
```

### Use parametrize for multiple cases

```python
@pytest.mark.parametrize("a,b,expected", [
    (2, 3, 5),
    (-1, 1, 0),
    (0, 0, 0),
    (100, 200, 300),
])
def test_add_cases(a, b, expected):
    """Test multiple addition cases."""
    assert add(a, b) == expected

@pytest.mark.parametrize("email", [
    "user@example.com",
    "test.name@company.co.uk",
    "valid+tag@domain.com",
])
def test_valid_emails(email):
    """Test valid email formats."""
    assert validate_email(email) == True

@pytest.mark.parametrize("email", [
    "invalid",
    "@example.com",
    "user@",
    "user @example.com",
])
def test_invalid_emails(email):
    """Test invalid email formats."""
    with pytest.raises(ValueError):
        validate_email(email)
```

---

## Next Steps

Now you have comprehensive testing coverage! Next:

1. **[AI-Assisted Development](ai-assisted-development.md)** - Use LLMs to write your tests faster
2. **[Resilience Patterns](resilience.md)** - Add retry logic and error handling
3. **[CI/CD](cicd.md)** - Automate testing in your deployment pipeline

???+ tip "Testing Checklist"
    - [ ] Unit tests for all tools (90%+ coverage)
    - [ ] Integration tests for HTTP transport
    - [ ] Contract tests for tool schemas
    - [ ] Async tests with pytest-asyncio
    - [ ] Mocked external dependencies
    - [ ] Error handling tests
    - [ ] CI/CD pipeline running tests
    - [ ] Pre-commit hooks enabled

---

## Additional Resources

- **[pytest Documentation](https://docs.pytest.org/)** - Official pytest guide
- **[pytest-asyncio](https://pytest-asyncio.readthedocs.io/)** - Async testing
- **[FastMCP Testing](https://gofastmcp.com/testing)** - FastMCP test utilities
- **[Coverage.py](https://coverage.readthedocs.io/)** - Coverage measurement
