# ContextForge MCP Gateway

???+ abstract "What it does"
    The ContextForge Gateway is an MCP-native proxy, registry, and federation layer. Point your clients at one endpoint and let the gateway handle auth, transport translation (STDIO ⇆ HTTP), health checks, and server discovery.

???+ info "Workshop progress"
    By now you should have:

    - ✅ [Run some sample MCP servers](running-mcp-servers.md) to understand how they work
    - ✅ [Built your own server](developing-your-mcp-server.md) with custom tools

    Now you'll set up the Gateway to centralize access to all your servers.

## 0. Prerequisites

- Python 3.11+ and `uv`
- Podman/Docker if you prefer containers
- Ports `4444` (prod) and/or `8000` (dev) available

## 1. Why use it?

- **Single entry point** – expose multiple FastMCP servers (local or remote) via one URL.
- **Transport bridge** – wrap STDIO servers with Streamable HTTP for IDEs or hosted LLMs.
- **Auth + tokens** – issue bearer tokens or plug in OAuth providers once instead of per server.
- **Observability** – built-in admin UI plus APIs for status checks.
- **Enterprise-ready** – federation, rate limiting, mTLS, RBAC for production deployments.

???+ info "Enterprise deployments"
    For production architecture patterns, security best practices, and enterprise deployment strategies, see [Architecting Secure Enterprise AI Agents with MCP](https://ibm.biz/enterprise-ai-with-mcp).

## 2. Quick start (source install)

```bash
git clone https://github.com/IBM/mcp-context-forge.git
cd mcp-context-forge
cp .env.example .env
make venv install install-dev
make serve  # exposes https://localhost:4444 by default
```

Use `make dev` for hot reloading on port 8000. Prefer containers? Run `podman run -p 4444:4444 ghcr.io/ibm/mcp-context-forge:latest` after creating a data volume for the SQLite database.

## 3. Authenticate

The server expects basic auth for the admin UI plus a bearer token for the API.

```bash
export BASIC_AUTH_USER=admin
export BASIC_AUTH_PASSWORD=changeme
export JWT_SECRET_KEY=my-test-key
```

Generate a token:

```bash
uv run python -m mcpgateway.tokens issue --subject local-dev
export MCPGATEWAY_BEARER_TOKEN=<value>
```

(Or use the built-in `/admin` UI to mint one.)

## 4. Smoke test the gateway

```bash
curl -k http://127.0.0.1:4444/version
curl -k -u admin:changeme http://127.0.0.1:4444/admin/
```

You should see JSON containing the version hash, and the admin UI should load once you accept the self-signed certificate in your browser (or switch to HTTP locally).

## 5. Register your FastMCP server

Once your FastMCP server is reachable at `http://localhost:8000/mcp`, add it via API:

```bash
curl -X POST http://127.0.0.1:4444/servers \
  -H "Authorization: Bearer $MCPGATEWAY_BEARER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "echo-server",
    "url": "http://127.0.0.1:8000/mcp",
    "transport": "streamablehttp"
  }'
```

Or use the Admin UI → **Servers** → **Add Server** and pick the transport that matches how you run FastMCP.

## 6. Use it from clients

```python
from fastmcp import Client
from fastmcp.client.auth import BearerAuth

async with Client(
    "http://localhost:4444/mcp",
    auth=BearerAuth(token=os.environ["MCPGATEWAY_BEARER_TOKEN"]),
) as client:
    await client.ping()
    print(await client.list_tools())
```

- Point the FastMCP Python client at `http://localhost:4444/mcp` (plus bearer token) to exercise every registered server through one endpoint.
- Claude Desktop / Cursor / other IDEs can now add a single “ContextForge Gateway” server instead of dozens of individual ones.
- Need retries, rate limits, or mTLS? Check the `docs/` folder inside the gateway repo for detailed configuration examples, Kubernetes manifests, and helm charts.

Keep the gateway running while you iterate. Each time you restart your FastMCP server the Gateway will reconnect automatically, so you can focus on building tools.

---

## 7. Production Deployment

For local development, the setup above is sufficient. For production deployments, consider:

### Security
- Enable mTLS for server-to-server communication
- Use OAuth providers instead of static bearer tokens
- Run behind a reverse proxy (nginx, Traefik)
- Enable audit logging

### Scalability
- Deploy with Docker/Kubernetes (Helm charts available)
- Use Redis for distributed caching
- Configure federation for multi-cluster setups
- Set up health checks and monitoring

### Resources
- **[ContextForge Production Docs](https://ibm.github.io/mcp-context-forge/deployment/production/)**
- **[Enterprise AI with MCP](https://ibm.biz/enterprise-ai-with-mcp)** - Architecture patterns and security best practices
- **[Kubernetes Deployment Guide](https://ibm.github.io/mcp-context-forge/deployment/kubernetes/)**

---

## 8. Troubleshooting

- **401/403 errors** – confirm `MCPGATEWAY_BEARER_TOKEN` matches the token returned by `python -m mcpgateway.tokens issue ...`.
- **Server shows `inactive`** – ensure the FastMCP server is running and reachable from the Gateway host. Use `curl http://localhost:8000/mcp` to verify.
- **Self-signed certificate warnings** – during local development you can run the gateway over plain HTTP by setting `USE_TLS=false` in `.env`.
- **Port conflicts** – override the port via `PORT=5555 make serve` or update your container mapping.
