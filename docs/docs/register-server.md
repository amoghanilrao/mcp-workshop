# Register Your Server with ContextForge

???+ info "Workshop progress"
    You should now have:

    - ✅ Run and tested [sample MCP servers](running-mcp-servers.md)
    - ✅ [Built your own server](developing-your-mcp-server.md)
    - ✅ [Launched the Gateway](mcp-gateway.md)

    Final step: register your server so it's accessible through the Gateway!

## 0. Before you start

- Gateway running on `http://localhost:4444`
- FastMCP server running on `http://localhost:8000/mcp`
- `MCPGATEWAY_BEARER_TOKEN` exported in your shell

```bash
export MCPGATEWAY_BEARER_TOKEN=$(uv run python -m mcpgateway.tokens issue --subject echo-dev | jq -r '.token')
```

## 1. API workflow

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

Verify:

```bash
curl -H "Authorization: Bearer $MCPGATEWAY_BEARER_TOKEN" \
     http://127.0.0.1:4444/servers
```

Expected snippet:

```json
{
  "servers": [
    {
      "name": "echo-server",
      "url": "http://127.0.0.1:8000/mcp",
      "transport": "streamablehttp",
      "status": "active"
    }
  ]
}
```

## 2. Admin UI workflow

1. Visit `http://localhost:4444/admin` and sign in with your basic-auth credentials.
2. Go to **Servers → Add Server**.
3. Enter the HTTP URL exposed by FastMCP (e.g., `http://localhost:8000/mcp`).
4. Pick the transport that matches your server (`streamablehttp` is recommended for new projects; STDIO-only servers can be proxied via `mcpgateway.wrapper`).
5. Save and confirm the status flips to **Active**.

Use the **Tools** and **Resources** tabs to make sure your FastMCP metadata propagated. If the server does not appear, double-check that it is reachable via `curl http://localhost:8000/mcp` and that the bearer token you provided is valid.

## 3. Validate through the gateway

```python
from fastmcp import Client
from fastmcp.client.auth import BearerAuth

async with Client(
    "http://localhost:4444/mcp",
    auth=BearerAuth(token=os.environ["MCPGATEWAY_BEARER_TOKEN"]),
) as client:
    await client.ping()
    result = await client.call_tool("echo-server-echo", {"text": "Gateway Success"})
    print(result.content[0].text)
```

If the tool name includes the namespace (e.g., `echo-server-echo`), that means the Gateway successfully namespaced and exposed it.

## 4. Troubleshooting

- **`409 server already exists`** – delete the entry via `DELETE /servers/{name}` or remove it from the Admin UI before re-registering.
- **`404 server not found`** – double-check the `name` value; it must match exactly when issuing `PUT`/`DELETE` calls.
- **Server stuck in `inactive`** – ensure the FastMCP process is still running and reachable from the gateway container/host. Use `docker exec` or `podman exec` to curl from inside the container if needed.
- **Auth header missing** – verify your shell exported `MCPGATEWAY_BEARER_TOKEN` (run `env | grep MCPGATEWAY`) and that your HTTP client isn't stripping it.

For more help, see the [Debugging Guide](debugging.md).

---

## 5. Next Steps

🎉 **Congratulations!** You now have a working MCP server connected to ContextForge Gateway.

### Explore More Sample Servers

You've already run a few servers in the [Running MCP Servers](running-mcp-servers.md) section. Now try registering them with the Gateway or explore others:

- **[csv_pandas_chat_server](https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers/python/csv_pandas_chat_server)** - Natural language CSV analysis (you may have run this!)
- **[data_analysis_server](https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers/python/data_analysis_server)** - Statistical analysis and visualization
- **[plotly_server](https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers/python/plotly_server)** - Interactive data visualization
- **[xlsx_server](https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers/python/xlsx_server)** - Excel file manipulation
- **[synthetic_data_server](https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers/python/synthetic_data_server)** - Generate test datasets (you may have run this!)
- **[mermaid_server](https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers/python/mermaid_server)** - Create diagrams from text (you may have run this!)
- **[python_sandbox_server](https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers/python/python_sandbox_server)** - Safe Python code execution

[View all 20+ sample servers →](https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers/python)

Try registering multiple servers with the Gateway and accessing them all through a single endpoint!

### Learn Advanced Features

- **[Advanced Topics](advanced-topics.md)** - Prompts, resources, middleware, authentication, storage
- **[Debugging](debugging.md)** - Troubleshooting tips and common issues
- **[FastMCP Documentation](https://gofastmcp.com/getting-started/welcome)** - Official FastMCP guides
- **[ContextForge Docs](https://ibm.github.io/mcp-context-forge/)** - Gateway configuration and deployment
- **[Enterprise MCP Guide](https://ibm.biz/enterprise-ai-with-mcp)** - Production architecture, security, and best practices

### Build Something

Ideas for your next MCP server:

- Integrate with your favorite API (GitHub, Slack, Jira, etc.)
- Create domain-specific analysis tools
- Build data transformation pipelines
- Add LLM-powered features to existing applications
- Expose internal tools to AI assistants

Happy building! 🚀
