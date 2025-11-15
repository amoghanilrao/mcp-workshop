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

## 2. Run Your First Server: Echo/Calculator

Let's start with a simple server to understand the basics.

### Create a minimal test server

```bash
mkdir ~/mcp-test && cd ~/mcp-test
```

Create `simple_server.py`:

```python
from fastmcp import FastMCP

mcp = FastMCP("demo-server")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers together."""
    return a + b

@mcp.tool()
def echo(text: str) -> str:
    """Echo text back to you."""
    return f"You said: {text}"

if __name__ == "__main__":
    mcp.run()
```

### Run it with STDIO

```bash
fastmcp run simple_server.py
```

The server is now running and waiting for JSON-RPC messages on stdin. Keep it running and open a new terminal.

### Test with the FastMCP client

Create `test_client.py`:

```python
import asyncio
from fastmcp import Client

async def main():
    # Connect to the running server
    async with Client("simple_server.py") as client:
        # List available tools
        tools = await client.list_tools()
        print(f"\n📋 Available tools:")
        for tool in tools:
            print(f"  • {tool.name}: {tool.description}")

        # Call the add tool
        result = await client.call_tool("add", {"a": 5, "b": 3})
        print(f"\n🔢 add(5, 3) = {result.content[0].text}")

        # Call the echo tool
        result = await client.call_tool("echo", {"text": "Hello MCP!"})
        print(f"\n💬 {result.content[0].text}")

asyncio.run(main())
```

Run it:

```bash
uv run python test_client.py
```

**Expected output:**
```
📋 Available tools:
  • add: Add two numbers together.
  • echo: Echo text back to you.

🔢 add(5, 3) = 8

💬 You said: Hello MCP!
```

???+ success "What just happened?"
    - The server exposed two tools via the MCP protocol
    - The client discovered them with `list_tools()`
    - The client called them with `call_tool(name, arguments)`
    - Everything communicated via JSON-RPC over STDIO

---

## 3. Run a Real Server: Synthetic Data Generator

Now let's try a more sophisticated server from the ContextForge collection.

```bash
cd ~/mcp-context-forge/mcp-servers/python/synthetic_data_server
```

### Install dependencies

```bash
uv venv
source .venv/bin/activate  # or `.venv\Scripts\activate` on Windows
uv pip install -e .
```

### Run the server (HTTP mode)

```bash
fastmcp run src/synthetic_data_server/server.py --transport http
```

The server starts on `http://localhost:8000/mcp`.

### Test it

Create `test_synthetic.py`:

```python
import asyncio
from fastmcp import Client

async def main():
    async with Client("http://localhost:8000/mcp") as client:
        # List tools
        tools = await client.list_tools()
        print(f"\n📋 Found {len(tools)} tools:")
        for tool in tools[:5]:  # Show first 5
            print(f"  • {tool.name}")

        # Generate sample data
        result = await client.call_tool(
            "generate_sample_data",
            {
                "num_rows": 10,
                "columns": ["name", "age", "email"],
                "seed": 42
            }
        )

        print(f"\n📊 Generated data:\n{result.content[0].text[:500]}")

asyncio.run(main())
```

Run it:

```bash
uv run python test_synthetic.py
```

You should see generated CSV data with names, ages, and emails!

---

## 4. Try Data Analysis: CSV Pandas Chat

This server lets you ask questions about CSV data in natural language.

```bash
cd ~/mcp-context-forge/mcp-servers/python/csv_pandas_chat_server
uv pip install -e .
```

### Set up OpenAI (required for this server)

```bash
export OPENAI_API_KEY="your-api-key-here"
```

### Run the server

```bash
fastmcp run src/csv_pandas_chat_server/server_fastmcp.py --transport http --port 8001
```

### Test with sample data

Create `test_csv_chat.py`:

```python
import asyncio
from fastmcp import Client
import os

async def main():
    csv_data = """product,sales,region
Widget A,1000,North
Widget B,1500,South
Widget C,800,East
Gadget X,2000,West
Gadget Y,1200,North"""

    async with Client("http://localhost:8001/mcp") as client:
        # Get CSV info
        result = await client.call_tool(
            "get_csv_info",
            {"csv_content": csv_data}
        )
        print(f"\n📊 CSV Info:\n{result.content[0].text[:300]}")

        # Ask a question (requires OpenAI API key)
        if os.getenv("OPENAI_API_KEY"):
            result = await client.call_tool(
                "chat_with_csv",
                {
                    "query": "What are the top 3 products by sales?",
                    "csv_content": csv_data
                }
            )
            print(f"\n💡 Query result:\n{result.content[0].text[:500]}")

asyncio.run(main())
```

Run it:

```bash
uv run python test_csv_chat.py
```

---

## 5. Visualization: Mermaid Diagrams

```bash
cd ~/mcp-context-forge/mcp-servers/python/mermaid_server
uv pip install -e .
fastmcp run src/mermaid_server/server.py --transport http --port 8002
```

Test it with `test_mermaid.py`:

```python
import asyncio
from fastmcp import Client

async def main():
    async with Client("http://localhost:8002/mcp") as client:
        # Create a flowchart
        result = await client.call_tool(
            "create_diagram",
            {
                "diagram_type": "flowchart",
                "content": """
                graph TD
                    A[Start] --> B{Decision}
                    B -->|Yes| C[Action 1]
                    B -->|No| D[Action 2]
                    C --> E[End]
                    D --> E
                """
            }
        )
        print(f"\n🎨 Mermaid diagram:\n{result.content[0].text}")

asyncio.run(main())
```

---

## 6. Understanding What You Learned

After running these servers, you now know:

### MCP Servers Expose:
- **Tools** - Functions that do work (calculations, data processing, API calls)
- **Resources** - Data that can be read (files, URLs, database queries)
- **Prompts** - Templates for common workflows

### Two Transport Modes:
- **STDIO** - Communicate via standard input/output (great for local tools)
- **HTTP** - Run as a web service (great for remote access, gateways)

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

## 7. Quick Reference: Testing Commands

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

## 8. Explore More Servers

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
