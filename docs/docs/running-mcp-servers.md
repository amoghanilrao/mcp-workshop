# Running Existing MCP Servers

???+ abstract "Goal"
    Get hands-on experience with MCP by running real servers from the ContextForge collection. You'll test tools, see how MCP works in practice, and understand what you'll be building later.

---

## Why Start Here?

Before building your own server, it's helpful to:

1. **See MCP in action** - understand what tools and resources look like
2. **Learn the client workflow** - how to list and call tools
3. **Get familiar with transports** - STDIO vs HTTP modes
4. **Explore different patterns** - data analysis, visualization, file processing

Once you've run a few servers, building your own will make much more sense.

---

## 1. Clone the Sample Servers

```bash
git clone https://github.com/IBM/mcp-context-forge.git
cd mcp-context-forge/mcp-servers/python
```

This directory contains 20+ production-ready MCP servers that demonstrate real-world MCP implementations. Browse all samples here: [https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers](https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers)

We'll run a few interesting ones to understand how MCP works.

---

## 2. Run Your First Server: Git over stdio

Let's start with a simple server to understand the basics.

```
# Install uv/uvx
pip install uv # to install uvx, if not already installed

# Run the Git MCP server as stdio
uvx mcp-server-git

# List capabilities
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"demo","version":"0.0.1"}}}
```

### 3. Convert stdio to SSE or Streamable HTTP using ContextForge

```bash
# Install ContextForge
uv pip install mcp-contextforge-gateway

# Convert stdio to remote streamable HTTP
python3 -m mcpgateway.translate --stdio "uvx mcp-server-git" --expose-streamable-http --port 9000

# .. or legacy SSE
python3 -m mcpgateway.translate --stdio "uvx mcp-server-git" --expose-sse --port 9000
```

---

## 4. Understanding What You Learned

After running these servers, you now know:

### MCP Servers Expose:
- **Tools** - Functions that do work (calculations, data processing, API calls)
- **Resources** - Data that can be read (files, URLs, database queries)
- **Prompts** - Templates for common workflows

### Two Transport Modes:
- **STDIO** - Communicate via standard input/output (great for local tools)
- **HTTP** - Run as a web service (great for remote access, gateways) (SSE available as well, but deprecated)

### The Client Workflow:
1. Connect to a server (file path for STDIO, URL for HTTP)
2. Discover capabilities (`list_tools()`, `list_resources()`, `list_prompts()`)
3. Call tools with arguments (`call_tool(name, args)`)
4. Get structured responses back

### Common Patterns:

| Server Type | Example | Use Case |
|-------------|---------|----------|
| **Data Processing** | csv_pandas_chat | Analyze data with natural language |
| **Visualization** | mermaid, plotly | Create charts and diagrams |
| **File Operations** | xlsx, docx, pptx | Read/write office documents |
| **Code Execution** | python_sandbox | Run code safely |
| **API Wrappers** | Custom servers | Expose any API as MCP tools |

---

## 5. Quick Reference: Testing Commands

### STDIO server (file path)
```python
async with Client("path/to/server.py") as client:
    tools = await client.list_tools()
```

### HTTP server (URL)
```python
async with Client("http://localhost:8000/mcp") as client:
    tools = await client.list_tools()
```

### List and call a tool
```python
# Discovery
tools = await client.list_tools()
print([t.name for t in tools])

# Execution
result = await client.call_tool("tool_name", {"arg1": "value1"})
print(result.content[0].text)
```

### With authentication (for Gateway)
```python
from fastmcp.client.auth import BearerAuth

async with Client(
    "http://localhost:4444/mcp",
    auth=BearerAuth(token=os.environ["MCPGATEWAY_BEARER_TOKEN"])
) as client:
    # ... use client
```

---

## 6. Explore More Servers

The ContextForge collection includes 20+ servers across Python, Go, and other languages:

**Python Servers:**

- **data_analysis_server** - Statistical analysis and pandas operations
- **plotly_server** - Interactive visualizations
- **xlsx_server** / **docx_server** / **pptx_server** - Office document manipulation
- **graphviz_server** - Graph diagrams
- **latex_server** - LaTeX document generation
- **url_to_markdown_server** - Web scraping and conversion
- **chunker_server** - Text chunking for LLMs
- **code_splitter_server** - Parse and analyze code

**Go Servers:**

- **fast-time-server** - High-performance time operations

Browse all samples: [https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers](https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers)

Each has a README with installation and usage examples.

---

## Next Steps

Now that you've run several MCP servers and understand how they work, you're ready to:

1. **[Build your own server](developing-your-mcp-server.md)** - Create custom tools for your use case
2. **[Set up ContextForge Gateway](mcp-gateway.md)** - Centralize multiple servers
3. **[Learn advanced features](advanced-topics.md)** - Prompts, resources, authentication

The skills you just learned (listing tools, calling them, understanding transports) are the foundation for everything else in the workshop.

???+ tip "Pro tip"
    Keep one of these servers running in the background while you build your own. You can reference how they structure tools, handle errors, and validate inputs.

---

## Testing with Local LLMs

Want to test your MCP servers with a local LLM? Use Ollama with IBM Granite 4:

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull IBM Granite 4 model
ollama pull granite4:3b

# Run it
ollama run granite4:3b
```

**Why Granite 4?** IBM's Granite 4.0 models (October 2025) are optimized for enterprise use with:
- **Hybrid Mamba-2/transformer architecture** - 70-80% less memory than traditional models
- **Strong tool-calling** and instruction-following capabilities
- **Efficient resource usage** - 3B model runs smoothly on laptops
- **Enterprise-grade** - Apache 2.0 licensed, ISO 42001 certified
- **Perfect for testing MCP servers locally**

Available sizes: `granite4:3b` (recommended), `granite4:8b`, or tiny variants for edge devices.

You can integrate Ollama with your MCP servers through MCP-compatible clients or custom integrations.

---

## Additional Resources

- **[MCP Specification](https://spec.modelcontextprotocol.io/)** - Official protocol documentation
- **[MCP Official Site](https://modelcontextprotocol.io/)** - Getting started guides
- **[MCP Servers Registry](https://github.com/modelcontextprotocol/servers)** - Community-maintained servers
- **[FastMCP Docs](https://gofastmcp.com/getting-started/welcome)** - FastMCP framework documentation
- **[Enterprise MCP Guide](https://ibm.biz/enterprise-ai-with-mcp)** - Production architecture and security patterns
- **[Ollama](https://ollama.com)** - Run LLMs locally
- **[IBM Granite Models](https://www.ibm.com/granite)** - Enterprise-grade open models
