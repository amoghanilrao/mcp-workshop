# Resilience & Production Patterns

???+ abstract "The Reality Check"
    Your MCP server works great on your laptop. Now make it bulletproof for production. This guide covers graceful degradation, retry logic, timeout handling, structured logging, and observability patterns that keep systems running at 3am.

---

## Why Resilience Matters

Production MCP servers face challenges dev environments don't:

- **External APIs fail** - Third-party services go down
- **Networks are unreliable** - Packets get lost, connections timeout
- **Resources are limited** - Memory, CPU, rate limits
- **Traffic spikes** - Sudden load from many agents
- **Partial failures** - Some tools work, others don't

**Resilient servers:**

- Retry transient failures automatically
- Degrade gracefully when dependencies fail
- Log enough to debug issues quickly
- Expose metrics for monitoring
- Fail fast on unrecoverable errors

---

## Retry Logic with Exponential Backoff

### Basic retry decorator

```python
# src/my_server/resilience.py
import asyncio
import logging
from functools import wraps
from typing import TypeVar, Callable

logger = logging.getLogger(__name__)

T = TypeVar('T')

def retry_with_backoff(
    max_retries: int = 3,
    initial_delay: float = 1.0,
    max_delay: float = 60.0,
    exponential_base: float = 2.0,
    exceptions: tuple = (Exception,)
):
    """
    Retry a function with exponential backoff.

    Args:
        max_retries: Maximum number of retry attempts
        initial_delay: Initial delay in seconds
        max_delay: Maximum delay between retries
        exponential_base: Multiplier for delay (2.0 = double each time)
        exceptions: Tuple of exceptions to catch and retry
    """
    def decorator(func: Callable[..., T]) -> Callable[..., T]:
        @wraps(func)
        async def wrapper(*args, **kwargs) -> T:
            delay = initial_delay

            for attempt in range(max_retries + 1):
                try:
                    return await func(*args, **kwargs)

                except exceptions as e:
                    if attempt == max_retries:
                        logger.error(
                            f"{func.__name__} failed after {max_retries} retries",
                            extra={
                                "function": func.__name__,
                                "attempts": attempt + 1,
                                "error": str(e)
                            }
                        )
                        raise

                    logger.warning(
                        f"{func.__name__} failed (attempt {attempt + 1}/{max_retries}), "
                        f"retrying in {delay:.1f}s",
                        extra={
                            "function": func.__name__,
                            "attempt": attempt + 1,
                            "delay": delay,
                            "error": str(e)
                        }
                    )

                    await asyncio.sleep(delay)
                    delay = min(delay * exponential_base, max_delay)

        return wrapper
    return decorator
```

### Usage in MCP tools

```python
# src/my_server/tools/api.py
from fastmcp import FastMCP
import httpx
from ..resilience import retry_with_backoff

mcp = FastMCP("resilient-server")

@mcp.tool(description="Fetch data from external API with retry")
@retry_with_backoff(
    max_retries=3,
    exceptions=(httpx.HTTPError, httpx.TimeoutException)
)
async def fetch_data(url: str) -> dict:
    """Fetch data with automatic retry on failure."""
    async with httpx.AsyncClient() as client:
        response = await client.get(url, timeout=10.0)
        response.raise_for_status()
        return response.json()
```

### Test retry behavior

```python
# tests/test_retry.py
import pytest
from unittest.mock import AsyncMock, patch
import httpx
from my_server.tools.api import fetch_data

@pytest.mark.asyncio
async def test_retry_succeeds_on_second_attempt():
    """Test that retry succeeds after one failure."""
    mock_client = AsyncMock()

    # Fail once, then succeed
    mock_client.get.side_effect = [
        httpx.TimeoutException("Timeout"),
        AsyncMock(status_code=200, json=lambda: {"data": "success"})
    ]

    with patch("httpx.AsyncClient") as mock:
        mock.return_value.__aenter__.return_value = mock_client
        result = await fetch_data("https://api.example.com/data")

    assert result == {"data": "success"}
    assert mock_client.get.call_count == 2

@pytest.mark.asyncio
async def test_retry_fails_after_max_attempts():
    """Test that retry gives up after max attempts."""
    mock_client = AsyncMock()
    mock_client.get.side_effect = httpx.TimeoutException("Timeout")

    with patch("httpx.AsyncClient") as mock:
        mock.return_value.__aenter__.return_value = mock_client

        with pytest.raises(httpx.TimeoutException):
            await fetch_data("https://api.example.com/data")

    assert mock_client.get.call_count == 4  # Initial + 3 retries
```

---

## Timeout Handling

### Set appropriate timeouts

```python
import httpx
import asyncio

# ❌ No timeout - can hang forever
async def bad_fetch(url: str):
    async with httpx.AsyncClient() as client:
        return await client.get(url)  # Dangerous!

# ✅ Request timeout
async def good_fetch(url: str):
    async with httpx.AsyncClient() as client:
        return await client.get(url, timeout=10.0)

# ✅ Per-operation timeout with asyncio
async def better_fetch(url: str):
    try:
        async with httpx.AsyncClient() as client:
            return await asyncio.wait_for(
                client.get(url, timeout=10.0),
                timeout=15.0  # Overall operation timeout
            )
    except asyncio.TimeoutError:
        raise TimeoutError(f"Request to {url} timed out after 15s")
```

### Timeout configuration

```python
# src/my_server/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    """Application settings with sensible defaults."""

    # Timeouts (in seconds)
    HTTP_TIMEOUT: float = 10.0
    DATABASE_TIMEOUT: float = 5.0
    TOOL_TIMEOUT: float = 30.0

    # Retry configuration
    MAX_RETRIES: int = 3
    RETRY_INITIAL_DELAY: float = 1.0
    RETRY_MAX_DELAY: float = 60.0

    # Rate limiting
    RATE_LIMIT_PER_MINUTE: int = 60

    class Config:
        env_file = ".env"

settings = Settings()
```

### Tool-level timeout enforcement

```python
from fastmcp import FastMCP
import asyncio
from .config import settings

mcp = FastMCP("timeout-enforced")

@mcp.tool(description="Tool with enforced timeout")
async def expensive_operation(data: str) -> dict:
    """Operation that must complete within configured timeout."""
    try:
        return await asyncio.wait_for(
            _do_expensive_work(data),
            timeout=settings.TOOL_TIMEOUT
        )
    except asyncio.TimeoutError:
        return {
            "error": f"Operation timed out after {settings.TOOL_TIMEOUT}s",
            "status": "timeout"
        }

async def _do_expensive_work(data: str) -> dict:
    """The actual work (could be slow)."""
    # Simulate expensive operation
    await asyncio.sleep(5)
    return {"result": "processed", "data": data}
```

---

## Graceful Degradation

### Fallback strategies

```python
# src/my_server/tools/weather.py
from fastmcp import FastMCP
import httpx
from typing import Optional

mcp = FastMCP("weather-server")

# Primary and fallback weather APIs
PRIMARY_API = "https://api.weather.com/v1"
FALLBACK_API = "https://api.openweathermap.org/data/2.5"

@mcp.tool(description="Get weather with fallback to secondary API")
async def get_weather(city: str) -> dict:
    """Get weather data with automatic fallback."""

    # Try primary API
    try:
        return await _fetch_from_primary(city)
    except Exception as e:
        logger.warning(f"Primary API failed: {e}, trying fallback")

        # Try fallback API
        try:
            return await _fetch_from_fallback(city)
        except Exception as fallback_error:
            logger.error(f"Both APIs failed: {e}, {fallback_error}")

            # Return degraded response
            return {
                "city": city,
                "status": "unavailable",
                "error": "Weather data temporarily unavailable",
                "message": "Please try again later"
            }

async def _fetch_from_primary(city: str) -> dict:
    """Fetch from primary weather API."""
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"{PRIMARY_API}/weather",
            params={"city": city},
            timeout=5.0
        )
        response.raise_for_status()
        return response.json()

async def _fetch_from_fallback(city: str) -> dict:
    """Fetch from fallback weather API."""
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"{FALLBACK_API}/weather",
            params={"q": city},
            timeout=5.0
        )
        response.raise_for_status()
        return response.json()
```

### Cached fallback

```python
from fastmcp import FastMCP
from typing import Optional
import aiofiles
import json
from datetime import datetime, timedelta

mcp = FastMCP("cached-fallback")

CACHE_TTL = timedelta(hours=1)

@mcp.tool(description="Fetch data with cached fallback")
async def fetch_with_cache_fallback(resource_id: str) -> dict:
    """
    Fetch resource with cache fallback.

    If live fetch fails, return cached data if available.
    """
    cache_key = f"cache/{resource_id}.json"

    # Try live fetch
    try:
        data = await _fetch_live(resource_id)
        await _save_to_cache(cache_key, data)
        return data

    except Exception as e:
        logger.warning(f"Live fetch failed: {e}, checking cache")

        # Try cached data
        cached_data = await _load_from_cache(cache_key)
        if cached_data:
            cached_data["_cached"] = True
            cached_data["_cache_warning"] = "Using cached data due to API unavailability"
            return cached_data

        # No cache available
        raise ValueError(f"Resource {resource_id} unavailable and not in cache")

async def _save_to_cache(key: str, data: dict):
    """Save data to cache with timestamp."""
    cache_entry = {
        "data": data,
        "cached_at": datetime.utcnow().isoformat()
    }
    async with aiofiles.open(key, 'w') as f:
        await f.write(json.dumps(cache_entry))

async def _load_from_cache(key: str) -> Optional[dict]:
    """Load data from cache if not expired."""
    try:
        async with aiofiles.open(key, 'r') as f:
            cache_entry = json.loads(await f.read())

        cached_at = datetime.fromisoformat(cache_entry["cached_at"])
        age = datetime.utcnow() - cached_at

        if age < CACHE_TTL:
            return cache_entry["data"]
        else:
            logger.info(f"Cache expired (age: {age})")
            return None

    except FileNotFoundError:
        return None
```

---

## Structured Logging

### Configure structured logging

```python
# src/my_server/logging_config.py
import logging
import json
from datetime import datetime
from typing import Any

class StructuredFormatter(logging.Formatter):
    """Format logs as JSON for easy parsing."""

    def format(self, record: logging.LogRecord) -> str:
        log_data = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
        }

        # Add exception info if present
        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)

        # Add extra fields
        if hasattr(record, "extra_data"):
            log_data.update(record.extra_data)

        return json.dumps(log_data)

def setup_logging(level: str = "INFO"):
    """Configure application logging."""
    handler = logging.StreamHandler()
    handler.setFormatter(StructuredFormatter())

    root_logger = logging.getLogger()
    root_logger.addHandler(handler)
    root_logger.setLevel(level)
```

### Logging with context

```python
# src/my_server/tools/processor.py
import logging
from fastmcp import FastMCP

logger = logging.getLogger(__name__)
mcp = FastMCP("logging-demo")

@mcp.tool(description="Process data with detailed logging")
async def process_data(data: dict, user_id: str) -> dict:
    """Process data with structured logging context."""

    # Create context for this operation
    context = {
        "operation": "process_data",
        "user_id": user_id,
        "data_size": len(str(data))
    }

    logger.info("Starting data processing", extra={"extra_data": context})

    try:
        # Processing steps with logging
        validated = await _validate(data)
        context["validation"] = "passed"

        transformed = await _transform(validated)
        context["transform"] = "completed"

        result = await _save(transformed)
        context["saved"] = True
        context["result_id"] = result["id"]

        logger.info("Data processing completed", extra={"extra_data": context})
        return result

    except ValueError as e:
        context["error"] = str(e)
        context["error_type"] = "validation_error"
        logger.error("Validation failed", extra={"extra_data": context})
        raise

    except Exception as e:
        context["error"] = str(e)
        context["error_type"] = type(e).__name__
        logger.error("Processing failed", extra={"extra_data": context}, exc_info=True)
        raise
```

### Log output (JSON format)

```json
{
  "timestamp": "2025-01-15T10:30:45.123Z",
  "level": "INFO",
  "logger": "my_server.tools.processor",
  "message": "Starting data processing",
  "module": "processor",
  "function": "process_data",
  "line": 15,
  "operation": "process_data",
  "user_id": "user_123",
  "data_size": 1024
}
```

---

## Error Handling

### Proper error codes

```python
# src/my_server/errors.py
from enum import Enum

class ErrorCode(str, Enum):
    """Standard error codes for MCP tools."""

    # Client errors (4xx equivalent)
    INVALID_INPUT = "invalid_input"
    MISSING_PARAMETER = "missing_parameter"
    VALIDATION_FAILED = "validation_failed"
    UNAUTHORIZED = "unauthorized"
    RATE_LIMITED = "rate_limited"

    # Server errors (5xx equivalent)
    INTERNAL_ERROR = "internal_error"
    SERVICE_UNAVAILABLE = "service_unavailable"
    TIMEOUT = "timeout"
    DEPENDENCY_FAILED = "dependency_failed"

class MCPError(Exception):
    """Base exception for MCP tools."""

    def __init__(
        self,
        message: str,
        code: ErrorCode,
        details: dict = None
    ):
        self.message = message
        self.code = code
        self.details = details or {}
        super().__init__(self.message)

    def to_dict(self) -> dict:
        """Convert error to dict for JSON response."""
        return {
            "error": self.code.value,
            "message": self.message,
            "details": self.details
        }

# Specific error types
class ValidationError(MCPError):
    def __init__(self, message: str, details: dict = None):
        super().__init__(message, ErrorCode.VALIDATION_FAILED, details)

class ServiceUnavailableError(MCPError):
    def __init__(self, message: str, details: dict = None):
        super().__init__(message, ErrorCode.SERVICE_UNAVAILABLE, details)
```

### Use in tools

```python
from fastmcp import FastMCP
from .errors import ValidationError, ServiceUnavailableError, ErrorCode

mcp = FastMCP("error-handling")

@mcp.tool(description="Tool with proper error handling")
async def validate_and_process(email: str, age: int) -> dict:
    """Process user data with validation."""

    # Input validation
    if not email or "@" not in email:
        raise ValidationError(
            "Invalid email address",
            details={"field": "email", "value": email}
        )

    if age < 0 or age > 150:
        raise ValidationError(
            "Age must be between 0 and 150",
            details={"field": "age", "value": age}
        )

    # External dependency
    try:
        result = await external_service.process(email, age)
        return result

    except httpx.HTTPError as e:
        raise ServiceUnavailableError(
            "External service unavailable",
            details={
                "service": "external_service",
                "status_code": getattr(e.response, "status_code", None)
            }
        )
```

---

## Observability Hooks

### Custom middleware for metrics

```python
# src/my_server/middleware.py
import time
from fastapi import Request, Response
from prometheus_client import Counter, Histogram

# Metrics
tool_calls = Counter(
    "mcp_tool_calls_total",
    "Total number of tool calls",
    ["tool_name", "status"]
)

tool_duration = Histogram(
    "mcp_tool_duration_seconds",
    "Tool execution duration",
    ["tool_name"]
)

async def observability_middleware(request: Request, call_next):
    """Middleware to track tool calls and performance."""

    start_time = time.time()

    # Extract tool name from JSON-RPC request
    body = await request.body()
    try:
        import json
        data = json.loads(body)
        tool_name = data.get("params", {}).get("name", "unknown")
    except:
        tool_name = "unknown"

    # Process request
    try:
        response = await call_next(request)
        status = "success" if response.status_code < 400 else "error"

    except Exception as e:
        status = "exception"
        raise

    finally:
        # Record metrics
        duration = time.time() - start_time
        tool_calls.labels(tool_name=tool_name, status=status).inc()
        tool_duration.labels(tool_name=tool_name).observe(duration)

    return response
```

### Health check endpoint

```python
# src/my_server/server.py
from fastmcp import FastMCP
from fastapi import Request
import httpx

mcp = FastMCP("production-server")

@mcp.custom_route("/health", methods=["GET"])
async def health_check():
    """Health check endpoint for load balancers."""
    return {
        "status": "healthy",
        "version": "1.0.0",
        "timestamp": datetime.utcnow().isoformat()
    }

@mcp.custom_route("/ready", methods=["GET"])
async def readiness_check():
    """Readiness check - verify dependencies."""
    checks = {
        "database": await check_database(),
        "external_api": await check_external_api()
    }

    all_healthy = all(checks.values())

    return {
        "ready": all_healthy,
        "checks": checks
    }, 200 if all_healthy else 503

async def check_database() -> bool:
    """Check if database is accessible."""
    try:
        # Attempt database query
        return True
    except:
        return False

async def check_external_api() -> bool:
    """Check if external API is reachable."""
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(
                "https://api.example.com/health",
                timeout=2.0
            )
            return response.status_code == 200
    except:
        return False
```

---

## Circuit Breaker Pattern

```python
# src/my_server/circuit_breaker.py
from datetime import datetime, timedelta
from enum import Enum
from typing import Callable, TypeVar
import asyncio

T = TypeVar('T')

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing, reject requests
    HALF_OPEN = "half_open"  # Testing if service recovered

class CircuitBreaker:
    """Circuit breaker to prevent cascading failures."""

    def __init__(
        self,
        failure_threshold: int = 5,
        timeout: float = 60.0,
        expected_exception: type = Exception
    ):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.expected_exception = expected_exception

        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED

    async def call(self, func: Callable[..., T], *args, **kwargs) -> T:
        """Execute function through circuit breaker."""

        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result

        except self.expected_exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        """Reset failure count on success."""
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        """Increment failure count and open circuit if threshold exceeded."""
        self.failure_count += 1
        self.last_failure_time = datetime.utcnow()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

    def _should_attempt_reset(self) -> bool:
        """Check if enough time has passed to try again."""
        return (
            self.last_failure_time is not None
            and datetime.utcnow() - self.last_failure_time
            > timedelta(seconds=self.timeout)
        )
```

### Use circuit breaker

```python
from .circuit_breaker import CircuitBreaker

external_api_breaker = CircuitBreaker(
    failure_threshold=5,
    timeout=60.0,
    expected_exception=httpx.HTTPError
)

@mcp.tool(description="Call external API with circuit breaker")
async def call_external_api(endpoint: str) -> dict:
    """Call API through circuit breaker."""
    return await external_api_breaker.call(
        _make_api_call,
        endpoint
    )

async def _make_api_call(endpoint: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(endpoint, timeout=5.0)
        response.raise_for_status()
        return response.json()
```

---

## Production Checklist

- [ ] **Retry logic** - Exponential backoff for transient failures
- [ ] **Timeouts** - All external calls have timeouts
- [ ] **Graceful degradation** - Fallbacks when dependencies fail
- [ ] **Structured logging** - JSON logs with context
- [ ] **Error codes** - Consistent error codes and messages
- [ ] **Health checks** - `/health` and `/ready` endpoints
- [ ] **Metrics** - Prometheus metrics for monitoring
- [ ] **Circuit breakers** - Prevent cascading failures
- [ ] **Rate limiting** - Protect against abuse
- [ ] **Resource limits** - Memory and CPU limits configured

---

## Next Steps

1. **[Testing](testing.md)** - Test your resilience patterns
2. **[CI/CD](cicd.md)** - Automate deployment with these patterns
3. **[Debugging](debugging.md)** - Use structured logs to debug production issues

---

## Additional Resources

- **[12-Factor App](https://12factor.net/)** - Best practices for production apps
- **[Site Reliability Engineering](https://sre.google/books/)** - Google's SRE practices
- **[Prometheus](https://prometheus.io/)** - Monitoring and metrics
- **[Structured Logging](https://www.structlog.org/)** - Python structured logging
