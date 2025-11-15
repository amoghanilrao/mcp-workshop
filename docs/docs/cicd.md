# CI/CD & Deployment

???+ abstract "The Ship-It Skills"
    You've built an awesome MCP server. Now let's automate testing, linting, building, and deploying it. This guide covers the full pipeline from git push to production using GitHub Actions, Docker, and modern Python tooling.

---

## The Modern Python Stack

| Tool | Purpose | Why Use It |
|------|---------|------------|
| **[ruff](https://github.com/astral-sh/ruff)** | Linting + formatting | 100x faster than pylint, replaces black/flake8/isort |
| **[mypy](https://mypy-lang.org/)** | Type checking | Catch type errors before runtime |
| **[uv](https://github.com/astral-sh/uv)** | Package/env manager | 100x faster than pip, handles venvs |
| **[pytest](https://pytest.org/)** | Testing | Industry standard, great ecosystem |
| **[Docker](https://www.docker.com/)** | Containerization | Consistent deployment across environments |
| **[GitHub Actions](https://github.com/features/actions)** | CI/CD | Free for public repos, integrated with GitHub |

---

## Project Structure

```
my-mcp-server/
├── .github/
│   └── workflows/
│       ├── test.yml              # Run tests on PRs
│       ├── lint.yml              # Linting and type checking
│       └── release.yml           # Build and publish releases
├── src/
│   └── my_server/
│       ├── __init__.py
│       ├── server.py
│       └── tools/
├── tests/
│   └── test_server.py
├── Containerfile                 # Multi-stage Docker build
├── .pre-commit-config.yaml       # Git hooks
├── pyproject.toml                # All configuration in one file
├── Makefile                      # Development commands
└── README.md
```

---

## Linting with Ruff

### Install and configure

```bash
uv add --dev ruff
```

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
    "ARG", # flake8-unused-arguments
    "SIM", # flake8-simplify
]
ignore = [
    "E501",  # line too long (handled by formatter)
    "B008",  # do not perform function calls in argument defaults
]

[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]  # Allow unused imports in __init__.py
"tests/*" = ["ARG"]       # Allow unused arguments in tests

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

### Run ruff

```bash
# Check for issues
ruff check .

# Auto-fix issues
ruff check --fix .

# Format code
ruff format .

# Check + format in one command
ruff check --fix . && ruff format .
```

---

## Type Checking with mypy

### Configure mypy

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
no_implicit_optional = true

# Per-module options
[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
```

### Install and run

```bash
uv add --dev mypy types-aiofiles types-httpx

# Type check
mypy src/

# Type check with coverage report
mypy --html-report mypy-report src/
```

### Add type hints

```python
# ❌ Before - no type hints
def process_data(data, options):
    result = transform(data)
    return result

# ✅ After - full type hints
from typing import Dict, List, Optional

def process_data(
    data: List[Dict[str, str]],
    options: Optional[Dict[str, any]] = None
) -> Dict[str, any]:
    """Process data with options."""
    result: Dict[str, any] = transform(data)
    return result
```

---

## Packaging with uv

### pyproject.toml

```toml
# pyproject.toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-mcp-server"
version = "1.0.0"
description = "Production-ready MCP server"
authors = [{name = "Your Name", email = "you@example.com"}]
license = {text = "Apache-2.0"}
readme = "README.md"
requires-python = ">=3.11"

dependencies = [
    "fastmcp>=2.13.0",
    "httpx>=0.27.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=5.0.0",
    "pytest-mock>=3.14.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
]

[project.scripts]
my-server = "my_server.server:main"

[project.urls]
Homepage = "https://github.com/yourusername/my-mcp-server"
Issues = "https://github.com/yourusername/my-mcp-server/issues"
```

### Build and publish

```bash
# Install build dependencies
uv pip install build twine

# Build distribution
python -m build

# Check distribution
twine check dist/*

# Upload to PyPI
twine upload dist/*

# Or use uv (coming soon)
uv publish
```

---

## Docker Multi-Stage Builds

### Containerfile (Podman/Docker compatible)

```dockerfile
# Containerfile
# Stage 1: Build dependencies
FROM registry.access.redhat.com/ubi9/python-311:latest AS builder

WORKDIR /app

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

# Copy dependency files
COPY pyproject.toml .
COPY README.md .

# Install dependencies to /app/.venv
RUN uv venv /app/.venv && \
    uv pip install --no-cache -e .

# Stage 2: Runtime
FROM registry.access.redhat.com/ubi9/python-311:latest

WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder /app/.venv /app/.venv

# Copy application code
COPY src/ ./src/

# Create non-root user
RUN useradd -m -u 1000 mcpuser && \
    chown -R mcpuser:mcpuser /app

USER mcpuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Run server
ENV PATH="/app/.venv/bin:$PATH"
CMD ["python", "-m", "my_server.server"]
```

### Build and run

```bash
# Build image
podman build -t my-mcp-server:latest .

# Run container
podman run -d \
    --name my-mcp-server \
    -p 8080:8080 \
    -e API_KEY=secret \
    my-mcp-server:latest

# Test
curl http://localhost:8080/health

# View logs
podman logs -f my-mcp-server

# Stop
podman stop my-mcp-server
```

### Optimize for size

```dockerfile
# Use slim base image
FROM python:3.11-slim

# Multi-stage build to exclude build tools
# Install only runtime dependencies
# Clean up apt cache
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# Result: 200MB vs 1GB+ for full image
```

---

## GitHub Actions CI/CD

### Test workflow

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: "latest"

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: |
          uv venv
          uv pip install -e ".[dev]"

      - name: Run tests
        run: |
          source .venv/bin/activate
          pytest --cov=src --cov-report=xml --cov-report=term

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
          token: ${{ secrets.CODECOV_TOKEN }}
```

### Lint workflow

```yaml
# .github/workflows/lint.yml
name: Lint

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          uv venv
          uv pip install ruff mypy

      - name: Run ruff
        run: |
          source .venv/bin/activate
          ruff check .
          ruff format --check .

      - name: Run mypy
        run: |
          source .venv/bin/activate
          mypy src/
```

### Release workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - "v*"

jobs:
  build-and-publish:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Build package
        run: |
          uv venv
          uv pip install build
          python -m build

      - name: Publish to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
        with:
          password: ${{ secrets.PYPI_API_TOKEN }}

      - name: Build Docker image
        run: |
          podman build -t ghcr.io/${{ github.repository }}:${{ github.ref_name }} .
          podman build -t ghcr.io/${{ github.repository }}:latest .

      - name: Push to GitHub Container Registry
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | podman login ghcr.io -u ${{ github.actor }} --password-stdin
          podman push ghcr.io/${{ github.repository }}:${{ github.ref_name }}
          podman push ghcr.io/${{ github.repository }}:latest
```

---

## Pre-commit Hooks

### Configure pre-commit

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.13.0
    hooks:
      - id: mypy
        additional_dependencies: [types-aiofiles, types-httpx]

  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: pytest
        language: system
        pass_filenames: false
        always_run: true
```

### Install pre-commit

```bash
uv add --dev pre-commit

# Install hooks
pre-commit install

# Run manually
pre-commit run --all-files

# Update hooks
pre-commit autoupdate
```

---

## Semantic Versioning

### Version scheme

```
MAJOR.MINOR.PATCH

1.0.0 -> Initial release
1.0.1 -> Bug fix (backward compatible)
1.1.0 -> New feature (backward compatible)
2.0.0 -> Breaking change
```

### Automate versioning

```toml
# pyproject.toml
[tool.bumpversion]
current_version = "1.0.0"
commit = true
tag = true

[[tool.bumpversion.files]]
filename = "pyproject.toml"
search = 'version = "{current_version}"'
replace = 'version = "{new_version}"'

[[tool.bumpversion.files]]
filename = "src/my_server/__init__.py"
search = '__version__ = "{current_version}"'
replace = '__version__ = "{new_version}"'
```

```bash
# Install bump-my-version
uv add --dev bump-my-version

# Bump patch version (1.0.0 -> 1.0.1)
bump-my-version bump patch

# Bump minor version (1.0.1 -> 1.1.0)
bump-my-version bump minor

# Bump major version (1.1.0 -> 2.0.0)
bump-my-version bump major
```

---

## Makefile for Development

```makefile
# Makefile
.PHONY: help install test lint format type-check docker-build docker-run clean

help:  ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

install:  ## Install dependencies
	uv venv
	uv pip install -e ".[dev]"

test:  ## Run tests
	pytest --cov=src --cov-report=html --cov-report=term

test-fast:  ## Run tests without coverage
	pytest -x --tb=short

lint:  ## Run linters
	ruff check .

format:  ## Format code
	ruff format .

fix:  ## Fix linting issues and format
	ruff check --fix .
	ruff format .

type-check:  ## Run type checker
	mypy src/

check: lint type-check test  ## Run all checks

docker-build:  ## Build Docker image
	podman build -t my-mcp-server:latest .

docker-run:  ## Run Docker container
	podman run -d --name my-mcp-server -p 8080:8080 my-mcp-server:latest

docker-stop:  ## Stop Docker container
	podman stop my-mcp-server
	podman rm my-mcp-server

clean:  ## Clean build artifacts
	rm -rf .venv dist build *.egg-info .coverage htmlcov .pytest_cache .mypy_cache .ruff_cache
	find . -type d -name __pycache__ -exec rm -rf {} +
	find . -type f -name "*.pyc" -delete

release-patch:  ## Release patch version
	bump-my-version bump patch
	git push && git push --tags

release-minor:  ## Release minor version
	bump-my-version bump minor
	git push && git push --tags

release-major:  ## Release major version
	bump-my-version bump major
	git push && git push --tags
```

---

## Deployment Strategies

### Option 1: Direct deployment

```bash
# SSH to server
ssh user@server

# Pull latest code
cd /opt/my-mcp-server
git pull

# Update dependencies
uv pip install -e .

# Restart service
sudo systemctl restart my-mcp-server
```

### Option 2: Docker deployment

```bash
# Pull latest image
podman pull ghcr.io/username/my-mcp-server:latest

# Stop old container
podman stop my-mcp-server
podman rm my-mcp-server

# Run new container
podman run -d \
    --name my-mcp-server \
    -p 8080:8080 \
    -e API_KEY=${API_KEY} \
    --restart unless-stopped \
    ghcr.io/username/my-mcp-server:latest
```

### Option 3: Kubernetes deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-mcp-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-mcp-server
  template:
    metadata:
      labels:
        app: my-mcp-server
    spec:
      containers:
      - name: server
        image: ghcr.io/username/my-mcp-server:latest
        ports:
        - containerPort: 8080
        env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: mcp-secrets
              key: api-key
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: my-mcp-server
spec:
  selector:
    app: my-mcp-server
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

```bash
# Deploy
kubectl apply -f deployment.yaml

# Check status
kubectl get pods
kubectl logs -f deployment/my-mcp-server

# Scale
kubectl scale deployment my-mcp-server --replicas=5
```

---

## Deployment Checklist

### Before deploying

- [ ] All tests pass (`make test`)
- [ ] Linting passes (`make lint`)
- [ ] Type checking passes (`make type-check`)
- [ ] Version bumped (`bump-my-version bump patch`)
- [ ] CHANGELOG.md updated
- [ ] Git tag created
- [ ] Docker image built and tested
- [ ] Environment variables documented
- [ ] Health check endpoint working
- [ ] Secrets configured (API keys, tokens)

### After deploying

- [ ] Health check returns 200
- [ ] Readiness check passes
- [ ] Logs look normal
- [ ] Metrics are being collected
- [ ] Error rate is low
- [ ] Response times are acceptable
- [ ] End-to-end tests pass
- [ ] Monitor for 15+ minutes

---

## Monitoring in Production

### Prometheus metrics

```python
# src/my_server/metrics.py
from prometheus_client import Counter, Histogram, Gauge, generate_latest
from fastapi import Response

# Define metrics
requests_total = Counter(
    "mcp_requests_total",
    "Total requests",
    ["method", "status"]
)

request_duration = Histogram(
    "mcp_request_duration_seconds",
    "Request duration",
    ["method"]
)

active_connections = Gauge(
    "mcp_active_connections",
    "Active connections"
)

@mcp.custom_route("/metrics", methods=["GET"])
async def metrics():
    """Expose Prometheus metrics."""
    return Response(
        content=generate_latest(),
        media_type="text/plain"
    )
```

### Grafana dashboard

```json
{
  "dashboard": {
    "title": "MCP Server Metrics",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(mcp_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Request Duration",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(mcp_request_duration_seconds_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(mcp_requests_total{status=~\"4..|5..\"}[5m])"
          }
        ]
      }
    ]
  }
}
```

---

## Next Steps

1. **[Testing](testing.md)** - Ensure your CI tests are comprehensive
2. **[Resilience](resilience.md)** - Add production-grade error handling
3. **[GitHub Guide](github-guide.md)** - Learn Git workflows for collaboration
4. **[Contributing](contributing.md)** - Share your server with the community
5. **[Register with ContextForge](register-server.md)** - Deploy to the gateway

???+ tip "Automation is Key"
    The less manual work in your deployment, the fewer mistakes you'll make. Aim for:
    - Single command deploys (`make deploy`)
    - Automated testing (GitHub Actions)
    - Automated versioning and tagging
    - Automated rollbacks on failure

---

## Additional Resources

- **[ruff Documentation](https://docs.astral.sh/ruff/)** - Linting and formatting
- **[mypy Documentation](https://mypy.readthedocs.io/)** - Type checking
- **[uv Documentation](https://docs.astral.sh/uv/)** - Package management
- **[GitHub Actions Docs](https://docs.github.com/en/actions)** - CI/CD
- **[Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)** - Container optimization
- **[Semantic Versioning](https://semver.org/)** - Versioning strategy
