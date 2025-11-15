# AI-Assisted Development

???+ abstract "The AI Magic"
    Why write code manually when AI can help? This guide shows you how to use LLMs (via Cursor, Copilot, or local models) to build MCP servers faster, better, and with fewer bugs.

---

## The Meta Loop: AI Building AI Tools

You're building tools for AI agents. Why not use AI to build those tools? This creates a powerful feedback loop:

1. **Describe** what you want the MCP tool to do
2. **Generate** the implementation with an LLM
3. **Test** the tool with an AI agent
4. **Iterate** based on how the agent uses it

The best part? You can test your MCP servers with local LLMs (like Granite 4 via Ollama), creating a completely offline development workflow.

---

## Development Tools

### AI Coding Assistants

| Tool | Best For |
|------|----------|
| **[Cursor](https://cursor.sh/)** | Full IDE with AI chat, multi-file edits |
| **[GitHub Copilot](https://github.com/features/copilot)** | Inline suggestions in VS Code, JetBrains |
| **[Claude Code](https://www.anthropic.com/claude)** | Complex refactoring, architecture decisions |
| **[Continue](https://continue.dev/)** | Open-source, works with local LLMs |

### Local LLMs for Testing

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull IBM Granite 4 (best for tool calling)
ollama pull granite4:3b

# Or try other models
ollama pull qwen2.5-coder:7b
ollama pull deepseek-coder-v2:16b
```

**Why Granite 4?** IBM's Granite 4.0 models (October 2025) excel at:

- **Tool calling** - Strong function calling and schema understanding
- **Efficiency** - 70-80% less memory than traditional transformers
- **Local development** - 3B model runs great on laptops
- **Enterprise-grade** - Apache 2.0 licensed, ISO 42001 certified

---

## Workflow 1: Cursor + FastMCP

### Initial setup

```bash
# Install Cursor
# Download from https://cursor.sh/

# Clone your MCP project
git clone <your-mcp-repo>
cd <your-mcp-repo>
code .  # Opens in Cursor
```

### Generate a tool with Cursor

**Prompt in Cursor Chat (Cmd+K or Ctrl+K):**

```
Create a FastMCP tool called `summarize_text` that:

1. Takes a `text` parameter (string) and optional `max_length` (int, default 100)
2. Uses an LLM to generate a summary
3. Returns the summary as a string
4. Includes proper error handling and type hints
5. Add unit tests using pytest

Use async/await and include a docstring.
```

**Cursor will generate:**

```python
# src/my_server/tools/summarizer.py
from fastmcp import FastMCP
import httpx

mcp = FastMCP("summarizer")

@mcp.tool(description="Summarize text using an LLM")
async def summarize_text(text: str, max_length: int = 100) -> str:
    """
    Generate a concise summary of the provided text.

    Args:
        text: The text to summarize
        max_length: Maximum length of summary in characters

    Returns:
        A summarized version of the input text
    """
    if not text.strip():
        raise ValueError("Text cannot be empty")

    if max_length < 10:
        raise ValueError("max_length must be at least 10 characters")

    # Call OpenAI or local LLM
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://localhost:11434/api/generate",  # Ollama
            json={
                "model": "granite4:3b",
                "prompt": f"Summarize this in {max_length} chars:\n\n{text}",
                "stream": False
            },
            timeout=30.0
        )
        response.raise_for_status()
        return response.json()["response"][:max_length]

# tests/test_summarizer.py
import pytest
from my_server.tools.summarizer import summarize_text

@pytest.mark.asyncio
async def test_summarize_text():
    """Test text summarization."""
    text = "This is a long text that needs to be summarized. " * 10
    summary = await summarize_text(text, max_length=50)

    assert len(summary) <= 50
    assert summary  # Not empty

@pytest.mark.asyncio
async def test_empty_text_raises_error():
    """Test that empty text raises ValueError."""
    with pytest.raises(ValueError, match="cannot be empty"):
        await summarize_text("", max_length=100)
```

### Multi-file refactoring with Cursor

**Prompt:**

```
Refactor this MCP server to:

1. Move all tools from server.py into separate files in tools/
2. Create a tools/__init__.py that imports and registers all tools
3. Add type hints everywhere
4. Extract configuration into a config.py file
5. Update tests to match the new structure

Preserve all functionality.
```

Cursor will analyze your codebase and make changes across multiple files, maintaining consistency.

---

## Workflow 2: GitHub Copilot

### Inline suggestions

Copilot excels at completing code as you type:

```python
# Type this comment:
# Create a tool that validates email addresses using regex

# Copilot suggests:
@mcp.tool(description="Validate email format")
def validate_email(email: str) -> bool:
    """Check if email has valid format."""
    import re
    pattern = r'^[\w\.-]+@[\w\.-]+\.\w+$'
    return re.match(pattern, email) is not None
```

### Generate tests from implementation

```python
# In test file, type:
# Test the validate_email function

# Copilot suggests:
def test_validate_email_valid():
    assert validate_email("test@example.com") == True
    assert validate_email("user.name@company.co.uk") == True

def test_validate_email_invalid():
    assert validate_email("invalid") == False
    assert validate_email("@example.com") == False
    assert validate_email("user@") == False
```

### Copilot Chat for architecture

Open Copilot Chat panel and ask:

```
How should I structure an MCP server that:

- Fetches data from multiple APIs
- Caches responses for 5 minutes
- Handles rate limiting
- Supports authentication

Show me the project structure and key classes.
```

---

## Workflow 3: Continue + Local LLMs

Continue is open-source and works great with Ollama for fully local development.

### Setup Continue

```bash
# Install Continue extension in VS Code
# Configure .continue/config.json
{
  "models": [
    {
      "title": "Granite 4",
      "provider": "ollama",
      "model": "granite4:3b",
      "apiBase": "http://localhost:11434"
    }
  ]
}
```

### Use Continue for code generation

Highlight code, press `Cmd+Shift+M`, and prompt:

```
Add retry logic with exponential backoff to this HTTP call.
Retry up to 3 times with delays of 1s, 2s, 4s.
```

Continue + Granite 4 will modify the code:

```python
import asyncio
import httpx

async def fetch_with_retry(url: str, max_retries: int = 3) -> dict:
    """Fetch URL with exponential backoff retry."""
    for attempt in range(max_retries):
        try:
            async with httpx.AsyncClient() as client:
                response = await client.get(url, timeout=10.0)
                response.raise_for_status()
                return response.json()
        except (httpx.HTTPError, httpx.TimeoutException) as e:
            if attempt == max_retries - 1:
                raise
            delay = 2 ** attempt  # 1s, 2s, 4s
            await asyncio.sleep(delay)
```

---

## Prompt Engineering for Code Generation

### Effective prompts for MCP tools

**❌ Vague prompt:**
```
Create a tool for working with CSV files
```

**✅ Specific prompt:**
```
Create a FastMCP tool called `analyze_csv` that:

Input:

- csv_content: str (CSV data as string)
- operation: str (one of: "summary", "column_stats", "missing_values")

Output:

- JSON object with analysis results

Implementation:

- Use pandas for CSV parsing
- Handle malformed CSV gracefully
- Return error message if CSV is invalid
- Include type hints and docstrings
- Write pytest tests covering all 3 operations

Error handling:

- Raise ValueError for invalid operation
- Return error details for malformed CSV
```

### Prompts for testing

```
Generate pytest tests for the analyze_csv tool that:

1. Test all three operations (summary, column_stats, missing_values)
2. Test with valid CSV data
3. Test with malformed CSV (should handle gracefully)
4. Test with invalid operation parameter (should raise ValueError)
5. Mock pandas DataFrame operations
6. Achieve >90% code coverage
7. Use pytest fixtures for sample CSV data

Include both positive and negative test cases.
```

### Prompts for refactoring

```
Refactor this tool to follow best practices:

1. Extract hardcoded values into constants
2. Add input validation with pydantic
3. Improve error messages to be more user-friendly
4. Add structured logging with context
5. Add type hints for all parameters and return values
6. Split complex logic into smaller helper functions
7. Add docstrings in Google style

Maintain backward compatibility.
```

---

## Testing with Local LLMs

### Set up Ollama for testing

```bash
# Start Ollama service
ollama serve

# In another terminal, pull models
ollama pull granite4:3b

# Test the model
ollama run granite4:3b
>>> Use the multiply_numbers tool to calculate 15 * 23
```

### Create a test harness

```python
# tests/test_with_ollama.py
import pytest
import httpx
import json
from my_server.server import mcp
from fastmcp.testing import create_test_client

@pytest.mark.asyncio
async def test_tool_with_llm():
    """Test MCP tool using Ollama + Granite 4."""

    # Start MCP server
    async with create_test_client(mcp) as mcp_client:
        # Get available tools
        tools = await mcp_client.list_tools()

        # Format tools for LLM
        tool_descriptions = [
            f"{t.name}: {t.description}"
            for t in tools
        ]

        # Ask Ollama to use a tool
        async with httpx.AsyncClient() as client:
            response = await client.post(
                "http://localhost:11434/api/generate",
                json={
                    "model": "granite4:3b",
                    "prompt": f"""
Available tools:
{chr(10).join(tool_descriptions)}

Task: Calculate the sum of 42 and 58.

Which tool should you call and with what parameters?
Respond in JSON format: {{"tool": "tool_name", "params": {{}}}}
                    """,
                    "stream": False,
                    "format": "json"
                }
            )

            llm_response = response.json()["response"]
            action = json.loads(llm_response)

            # Execute the tool the LLM selected
            result = await mcp_client.call_tool(
                action["tool"],
                action["params"]
            )

            assert "100" in result.content[0].text
```

---

## AI-Assisted Debugging

### Use AI to explain errors

When you hit an error:

```python
# Error output:
# pydantic_core._pydantic_core.ValidationError: 1 validation error for ToolInput
# field required (type=value_error.missing)
```

**Prompt to Cursor/Copilot:**

```
I'm getting this error:
[paste error]

In this code:
[paste code]

What's wrong and how do I fix it?
```

### Generate debugging code

```
Add comprehensive logging to this function to help debug:

- Log input parameters
- Log each step of processing
- Log the final result
- Include timing information
- Use structured logging with context

Make it easy to trace execution flow.
```

---

## AI-Powered Code Review

### Pre-commit review with AI

```python
# .git/hooks/pre-commit
#!/bin/bash

# Get changed Python files
CHANGED_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep '\.py$')

if [ -z "$CHANGED_FILES" ]; then
    exit 0
fi

# Use LLM to review
for FILE in $CHANGED_FILES; do
    echo "Reviewing $FILE..."

    # Send to Claude Code or Continue
    REVIEW=$(llm "Review this code for:
    - Security issues
    - Error handling
    - Type safety
    - MCP best practices

    $(cat $FILE)")

    echo "$REVIEW"
done
```

### Automated test generation

```
Analyze this MCP tool and generate missing test cases:

[paste tool code]

Generate tests for:

- All code paths (aim for 100% coverage)
- Edge cases and boundary conditions
- Error conditions
- Invalid inputs
- Async behavior
- Resource cleanup

Use pytest and pytest-asyncio.
```

---

## Copilot/Cursor Workflows

### Workflow: Build a new tool in 5 minutes

1. **Describe the tool** (Cmd+K in Cursor):
   ```
   Create a FastMCP tool `geocode_address` that converts an address string
   to latitude/longitude using the Nominatim API. Include error handling.
   ```

2. **Generate tests** (in test file):
   ```
   # Generate pytest tests for geocode_address with mocked HTTP calls
   ```

3. **Add to server** (in server.py):
   ```
   # Import and register the geocode_address tool
   ```

4. **Run tests**:
   ```bash
   pytest tests/test_geocode.py
   ```

5. **Fix issues** - Copilot will suggest fixes as you edit

### Workflow: Refactor for production

**Prompt:**

```
Refactor this prototype MCP server for production:

1. Add proper error handling with custom exceptions
2. Add retry logic for external API calls
3. Add request/response logging
4. Add input validation with pydantic
5. Add rate limiting
6. Add health check endpoint
7. Extract configuration to environment variables
8. Add comprehensive docstrings
9. Add type hints everywhere
10. Create a complete test suite

Maintain the same public API.
```

---

## Best Practices

### Do's

✅ **Be specific in prompts** - Include requirements, constraints, and examples
✅ **Generate tests first** - Let AI write tests, then implement to pass them
✅ **Iterate in small steps** - Make one change at a time, test, repeat
✅ **Review generated code** - AI makes mistakes; understand what it generates
✅ **Use AI for boilerplate** - Let AI handle repetitive code
✅ **Test with local LLMs** - Use Granite 4 to validate your MCP tools work with AI agents

### Don'ts

❌ **Don't blindly accept suggestions** - AI can introduce bugs
❌ **Don't skip testing** - Generated code needs tests
❌ **Don't over-prompt** - Break complex tasks into smaller prompts
❌ **Don't ignore errors** - If AI code doesn't work, ask AI to fix it
❌ **Don't commit without review** - Always review AI-generated code

---

## Example: Full Tool Development

### Step 1: Describe what you want

**Cursor prompt:**

```
Create a complete MCP tool for sentiment analysis:

Tool name: analyze_sentiment
Input:
  - text: str (required)
  - model: str (optional, default "local", options: "local", "openai")
Output:
  - sentiment: str ("positive", "negative", "neutral")
  - score: float (0.0 to 1.0, confidence)
  - explanation: str (why this sentiment)

Implementation:
  - For model="local": use transformers library with distilbert
  - For model="openai": call OpenAI API
  - Handle errors gracefully
  - Add caching to avoid re-analyzing the same text
  - Include comprehensive logging

Also generate:
  - Unit tests (>90% coverage)
  - Integration test with both models
  - README section documenting the tool
```

### Step 2: AI generates the implementation

Cursor creates:

- `src/tools/sentiment.py` - Tool implementation
- `tests/test_sentiment.py` - Comprehensive tests
- Updated `README.md` - Documentation

### Step 3: Test with local LLM

```bash
# Start your server
fastmcp run server.py --transport http

# Test with Granite 4
ollama run granite4:3b
>>> Use the analyze_sentiment tool to check if "I love this product!" is positive
```

### Step 4: Iterate based on feedback

Granite 4 tries your tool and you see it fail. **Ask Cursor:**

```
The LLM is calling analyze_sentiment but getting an error:
"model parameter must be 'local' or 'openai', got 'gpt-4'"

Make the tool more flexible:

- Accept any OpenAI model name if model starts with "gpt-"
- Default to "local" for any other value
- Update tests to cover this behavior
```

---

## Resources

### AI Coding Tools

- **[Cursor](https://cursor.sh/)** - AI-first IDE
- **[GitHub Copilot](https://github.com/features/copilot)** - Code completion
- **[Continue](https://continue.dev/)** - Open-source AI coding assistant
- **[Claude Code](https://www.anthropic.com/claude)** - AI pair programmer

### Local LLMs

- **[Ollama](https://ollama.com)** - Run LLMs locally
- **[IBM Granite Models](https://www.ibm.com/granite)** - Enterprise-grade open models
- **[DeepSeek Coder](https://github.com/deepseek-ai/DeepSeek-Coder)** - Code-specialized LLM

### Guides

- **[Prompt Engineering Guide](https://www.promptingguide.ai/)** - Learn effective prompting
- **[FastMCP Documentation](https://gofastmcp.com/getting-started/welcome)** - FastMCP framework
- **[Testing Guide](testing.md)** - Write tests for your AI-generated code

---

## Next Steps

1. **[Testing](testing.md)** - Test your AI-generated code thoroughly
2. **[Resilience](resilience.md)** - Add production-grade error handling
3. **[CI/CD](cicd.md)** - Automate testing and deployment

???+ tip "Start Small"
    Don't try to generate an entire MCP server at once. Start with one tool, test it, iterate, then add more.
