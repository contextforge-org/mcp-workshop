# Developing Your MCP Server

???+ abstract "Scope"
    Use FastMCP to publish a tiny tool server, run it over STDIO or HTTP, and get it ready for ContextForge.

???+ info "Already ran some servers?"
    If you completed [Running MCP Servers](running-mcp-servers.md), you've seen how MCP works from a user perspective. Now you'll build your own server from scratch using those same patterns.

---

## 1. Create a project

```bash
mkdir fastmcp-echo && cd fastmcp-echo
uv init --python 3.11  # optional but nice for structured projects
touch server.py
```

If you skip `uv init`, just make sure a `pyproject.toml` or `requirements.txt` exists before you publish.

???+ info "Suggested layout"
    ```
    fastmcp-echo/
    ├── pyproject.toml
    ├── server.py
    ├── tests/              # optional, e.g., test_server.py
    └── README.md
    ```
    Keeping things flat makes it easy to deploy with `fastmcp run server.py` or export to FastMCP Cloud later.

## 2. Install FastMCP (and friends)

```bash
uv add fastmcp
# Optional extras for this workshop
touch requirements.txt  # if you prefer pip-compatible workflows
```

FastMCP ships with the `fastmcp` CLI, transports, storage helpers, testing utilities, and decorators for tools/prompts/resources.

## 3. Write a minimal server

```python title="server.py"
from fastmcp import FastMCP

mcp = FastMCP("echo-server", version="2025-03-26")

@mcp.tool(description="Echo text back with optional casing tweaks")
def echo(text: str, upper: bool = False) -> str:
    """Return the submitted text."""
    return text.upper() if upper else text

@mcp.tool
def word_count(text: str) -> int:
    """Count characters separated by whitespace."""
    return len(text.split())

if __name__ == "__main__":
    mcp.run()  # defaults to stdio transport for local clients
```

Key FastMCP features at play:

- `FastMCP(name, version)` registers metadata the Gateway will surface.
- `@mcp.tool` pulls argument schemas from Python type hints and docstrings.
- `mcp.run()` picks STDIO by default; pass `transport="http"` for Streamable HTTP.

## 4. Run and test locally

### Option A — FastMCP CLI

```bash
fastmcp run server.py                    # stdio
fastmcp run server.py --transport http   # http://localhost:8000/mcp
fastmcp run server.py --python 3.11 --with httpx
```

The CLI auto-detects the FastMCP instance (`mcp`, `server`, `app`). Add `--with` to install extra packages on the fly.

???+ tip "Auto-reload during development"
    Run `watchmedo auto-restart --pattern="*.py" --recursive -- fastmcp run server.py --transport http` OR `nodemon --exec "fastmcp run server.py --transport http` OR `ls *.py | entr -r fastmcp run server.py --transport http` to restart the server when files change. Great for iterating on tools quickly. 
    **Note**: Installing additional packages like `entr`, `watchmedo` or `nodemon` may be required.
### Option B — Python entrypoint

```bash
python server.py                                      # stdio
python -m server                                      # also works if packaged
python server.py --transport http  # add argparse to toggle transports
```

Inside the file you can choose HTTP directly:

```python
if __name__ == "__main__":
    mcp.run(
        transport="http",
        host="0.0.0.0",
        port=8000,
        path="/mcp",  # change to /api/mcp if you mount behind a reverse proxy
    )
```

For production you can expose an ASGI app:

```python
app = mcp.http_app(path="/mcp")
```

Run it with your server of choice, e.g. `uvicorn server:app --host 0.0.0.0 --port 8000`.

## 5. Extend the server (optional but recommended)

FastMCP includes quality-of-life helpers from the latest releases:

- **Custom routes** – add `/health` or metrics endpoints with `@mcp.custom_route("/health", methods=["GET"])`.
- **Storage backends** – use `mcp.storage` to access encrypted key-value stores powered by `py-key-value-aio`.
- **Response caching** – wrap tools with the caching middleware to avoid repeated API calls.
- **Authentication** – add Bearer or OAuth providers so ContextForge (or public clients) can connect securely.
- **Prompts & resources** – register reusable prompt templates or file-backed resources with `@mcp.prompt` / `@mcp.resource`.
- **Testing** – leverage the in-memory test harness (`fastmcp.testing`) to call `tools.call` without spinning up transports.

Pick one improvement, keep commits small, and document the behavior in your server README.

## 6. Validate with the FastMCP client

FastMCP ships with an async client that can connect directly to your running server or even to the Python file via STDIO. Create `test_client.py`:

```python
import asyncio
from fastmcp import Client

async def main():
    async with Client("http://localhost:8000/mcp") as client:
        await client.ping()
        tools = await client.list_tools()
        print("Tools:", [tool.name for tool in tools])

        result = await client.call_tool("echo", {"text": "hello", "upper": True})
        print(result.content[0].text)

asyncio.run(main())
```

Point the client at a file path (`Client("server.py")`) to exercise STDIO transport without opening a network port, or pass a `FastMCP` instance directly for in-memory testing. Once this succeeds you are ready to register the endpoint with the ContextForge Gateway.

### Option C — Test with Ollama + Granite 4

Use a local LLM to interact with your server:

```bash
# Install and run Ollama with IBM Granite 4
curl -fsSL https://ollama.com/install.sh | sh
ollama pull granite4:3b

# Start your MCP server
fastmcp run server.py --transport http

# Run Granite 4
ollama run granite4:3b

# Use through an MCP-compatible client
# or integrate with your custom application
```

**IBM Granite 4.0** (October 2025) is excellent for testing MCP tools locally with its hybrid architecture, strong tool-calling capabilities, and 70-80% lower memory usage than traditional models.

## 7. Common pitfalls

- **Forgot to export the server instance**: ensure the file defines `mcp = FastMCP(...)` at module scope so `fastmcp run` can find it.
- **Wrong transport string**: use `transport="http"` for Streamable HTTP (preferred) or omit the parameter for STDIO. The string `"streamable-http"` still works but `http` is the current shorthand.
- **Port already in use**: change `port=` in `mcp.run()` or stop the conflicting process. FastMCP will crash loudly if it cannot bind.
- **Missing dependencies**: rerun `uv add <package>` or use `fastmcp run server.py --with <package>` to ensure requirements are available when the CLI spins up a subprocess.

For more troubleshooting help, see the [Debugging Guide](debugging.md).

---

## 8. Next Steps

Now that you've built your MCP server, consider:

1. **[Testing](testing.md)** - Write comprehensive tests to ensure reliability
2. **[AI-Assisted Development](ai-assisted-development.md)** - Use LLMs to enhance your server faster
3. **[Resilience](resilience.md)** - Add production-grade error handling and retry logic
4. **[CI/CD](cicd.md)** - Automate testing and deployment
5. **[Contributing Your Server](contributing.md)** - Share with the community!

---

## 9. Additional Resources

### Getting Started
- **[Official FastMCP Documentation](https://gofastmcp.com/getting-started/welcome)** - Comprehensive guides and API reference
- **[MCP Specification](https://spec.modelcontextprotocol.io/)** - Official protocol documentation
- **[Advanced Topics](advanced-topics.md)** - Prompts, resources, middleware, and more

### Examples & Code
- **[Sample Servers](https://github.com/IBM/mcp-context-forge/tree/main/mcp-servers/python)** - 20+ production-ready examples
- **[MCP Servers Registry](https://github.com/modelcontextprotocol/servers)** - Community servers

### Production & Enterprise
- **[Enterprise MCP Guide](https://ibm.biz/enterprise-ai-with-mcp)** - Architecture patterns for secure enterprise deployments
- **[ContextForge Gateway Docs](https://ibm.github.io/mcp-context-forge/)** - Production deployment guides

### Contributing
- **[GitHub Guide](github-guide.md)** - Learn Git and GitHub for collaboration
- **[Contributing Your Server](contributing.md)** - Submit to mcp-context-forge repository
