# Prerequisites

???+ abstract "Goal"
    Make sure your workstation can run a FastMCP server and launch the ContextForge Gateway. Everything below works on macOS, Linux, and Windows (via WSL2).

## 1. Core tools

| Tool | Why you need it | Install tips |
| --- | --- | --- |
| **Python 3.11+** | Runs FastMCP servers and the gateway | Use your OS package manager or [python.org/downloads](https://www.python.org/downloads/) |
| **uv** | Fast dependency + virtualenv manager used by FastMCP (`uvx`, `uv run`) | `curl -LsSf https://astral.sh/uv/install.sh | sh` |
| **git** | Clone this workshop and your server repo | `brew install git`, `sudo apt install git`, or download for Windows |
| **Podman or Docker** (optional) | Needed only if you want to run the Gateway or servers in containers | Install from [podman.io](https://podman.io) or [docker.com](https://www.docker.com/get-started/) |

???+ tip "Windows"
    Install [WSL2](https://learn.microsoft.com/windows/wsl/install) with Ubuntu, then follow the Linux instructions inside the WSL shell. Podman Desktop or Docker Desktop also work if you prefer a GUI runtime.

## 2. Quick OS notes

=== "Linux"
    ```bash
    sudo apt update && sudo apt install git podman python3 python3-venv -y
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```
    Log out and back in (or source the shell profile) so `uv` is on your PATH.

=== "macOS"
    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    brew install git python@3.11 podman
    curl -LsSf https://astral.sh/uv/install.sh | sh  # or `brew install uv`
    podman machine init --memory 4096 --cpus 2
    podman machine start
    ```

=== "Windows (WSL2)"
    ```powershell
    wsl --install -d Ubuntu
    wsl --status
    ```
    Inside WSL:
    ```bash
    sudo apt update && sudo apt install git podman python3 python3-venv -y
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```

## 3. Verify your environment

```bash
python3 --version
uv --version
git --version
podman --version   # optional
```

If `uv` is on your PATH you can run the rest of the workshop commands exactly as shown.

## 4. Helpful extras

- **fastmcp CLI** – ships with the FastMCP package; once installed you can run `fastmcp --help`.
- **ContextForge Gateway** – clone [IBM/mcp-context-forge](https://github.com/IBM/mcp-context-forge) to use the proxy locally.
- **MCP Inspector** – official GUI testing tool: `npx @modelcontextprotocol/inspector`

### Local LLM with Ollama (Optional but Recommended)

Run LLMs locally to test your MCP servers offline:

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull IBM Granite 4 model (recommended)
ollama pull granite4:3b

# Or try larger variant
ollama pull granite4:8b

# Or other models
ollama pull llama3.2
ollama pull qwen2.5
```

**Why Granite 4?** IBM's Granite 4.0 models (released October 2025) feature a breakthrough hybrid Mamba-2/transformer architecture with:
- **70-80% less memory** usage vs traditional transformers
- **Strong tool-calling** and instruction-following capabilities
- **Apache 2.0 licensed** and ISO 42001 certified
- **Perfect for local testing** - runs efficiently on laptops

Test your MCP server with Ollama:
```bash
# Start your MCP server
fastmcp run server.py --transport http

# Run Granite 4 in another terminal
ollama run granite4:3b

# Or use through an MCP-compatible client
```

### Learn More

- **[MCP Official Site](https://modelcontextprotocol.io/)** - Protocol overview and getting started
- **[MCP Specification](https://spec.modelcontextprotocol.io/)** - Technical specification
- **[FastMCP Docs](https://gofastmcp.com/getting-started/welcome)** - Framework documentation
- **[Enterprise MCP Guide](https://ibm.biz/enterprise-ai-with-mcp)** - Production architecture and security

## 5. Next steps

Once the commands above work you are ready to:

1. **[Run existing MCP servers](running-mcp-servers.md)** to see MCP in action
2. **[Build your own server](developing-your-mcp-server.md)** with FastMCP
3. **[Launch ContextForge Gateway](mcp-gateway.md)** for central routing and management
