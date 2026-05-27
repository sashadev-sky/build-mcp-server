# Building with MCP

## 📝 Notes

<details><summary><b>3️⃣ Distinct roles in MCP's spec: Host, Client, & Server</b></summary>

> The host **contains** the client(s). A host always wraps ≥1 client.

> The client abstracts away the complexity of server communication, letting u focus on building ur app logic.

1. **Host**: the app the user interacts with (e.g. **Claude for Desktop**, **Cursor**, **VS Code**, or a custom implementation like `cli_project/main.py`). It:
    - <u>Owns the LLM session</u>
      - For ex., the Anthropic client via `Anthropic()`
    - <u>Decides which servers to launch</u>
      - For ex., `MCPClient(command=..., args=...)`
    - <u>Manages user consent / input</u>: the loop
2. **Client**: a connector ***inside* the host**. The host spins up 1 client/server, & each client maintains a 1:1 stateful **JSON-RPC** session w/ its server, handling:
    - **`initialize`**
    - Capability negotiation
    - Message routing
3. **Server**: your process (e.g. `weather_mcp_server`). Exposes 3 capabilities (**Tools**, **Resources**, & **Prompts**), defined in the **FastMCP** note below ↓.

> 💡 In:
> - `weather_mcp_server`, **Claude for Desktop** is the host & spawns the client internally.
> - `cli_project`, `main.py` is the host & your wrapper class, for ex. `MCPClient`, is the client it embeds.

</details>

<details><summary><b>2️⃣ Standard transports: stdio & Streamable HTTP</b></summary>

> **MCP decouples the protocol (JSON-RPC 2.0) from the wire**: communication between the Client & Server can be done over many protocols.

The spec defines 2 standard transports:

1. **stdio**: client spawns the server as a subprocess & talks over `stdin`/`stdout`. Same machine, no network.
2. **Streamable HTTP**: client `POST`s JSON-RPC to **`/mcp`**; server streams responses back via **SSE** (Server-Sent Events). Local or remote.

Custom transports (WebSockets, Unix sockets, etc.) are allowed but non-standard. That has practical consequences:
- <u>Interoperability is limited to the 2 standard transports, stdio + Streamable HTTP</u>: a client written by someone else likely only speaks these. If you ship a WebSocket-only server, no off-the-shelf client (Claude for Desktop, Cursor, etc.) will connect to it.
- <u>Some spec features bind to specific transports</u>: e.g. MCP's authorization spec only applies to HTTP-based transports. stdio uses env vars instead. So "the protocol" isn't fully transport-blind in practice.

**When to pick what:**

|                   | **stdio**                       | **Streamable HTTP**               | **Custom**                            |
| ----------------- | ------------------------------- | --------------------------------- | ------------------------------------- |
| **Pick when**     | Local dev tools, IDE/CLI plugins | Remote/shared servers, multi-user SaaS | Niche needs (WebSockets, IPC pipes) |
| **Location**      | Same machine                    | Local or remote                   | Anywhere                              |
| **Lifecycle**     | Host spawns the server as a subprocess          | Long-running, you host it         | DIY                                   |
| **Auth**          | Env vars                        | OAuth, bearer tokens, headers     | DIY                                   |
| **Setup cost**    | None (no ports/network)         | Hosting + auth                    | High (build transport + clients)      |
| **Off-the-shelf clients** | ✅ Universal             | ✅ Universal                       | ❌ None                                |

> 📈 **The landscape is shifting**: stdio dominated 2024 to mid-2025 since every host (Claude for Desktop, Cursor, VS Code) launched local subprocess servers. 2026 is moving toward **remote Streamable HTTP**: Anthropic's MCP catalog, GitHub, Notion, Linear, etc. all ship as hosted HTTP servers. The split today is by use case: **developer tooling stays on stdio**, **production/SaaS integrations are moving to HTTP**.

> 💡 Both `cli_project` & `weather_mcp_server` use **stdio**.

</details>

<details><summary><b>🛠️ MCP server w/ <code>FastMCP</code></b></summary>

**`FastMCP`** (**`mcp.server.fastmcp.FastMCP`**) is the high-level Python API bundled w/ the [`mcp` SDK](https://github.com/modelcontextprotocol/python-sdk). The `FastMCP()` instance **is** your server; decorators register Python functions as its tools, resources, or prompts. No **JSON-RPC** boilerplate.

- **`FastMCP(name, **kwargs)`**: the 1st positional arg is the **MCP server name** sent during `initialize`. It identifies the server in host UIs, logs, & JSON-RPC negotiation. Not a Python identifier, so spaces & casing are fine. Convention: short & domain-flavored (`"weather"`, `"github"`, `"docs"`). Common kwargs:

  | kwarg                                     | purpose                                                                                                                                         |
  | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
  | `instructions=`                           | High-level guidance the LLM sees about how to use this server (e.g. "Use `get_alerts` for severe weather, `get_forecast` for daily conditions") |
  | `log_level=`                              | `"DEBUG"`, `"INFO"`, `"WARNING"`, `"ERROR"`. `cli_project` sets `"ERROR"` to keep the stdio channel quiet                                       |
  | `host=`, `port=`, `streamable_http_path=` | Streamable HTTP transport config (no-op w/ stdio)                                                                                               |
  | `auth=`, `token_verifier=`                | OAuth for HTTP transport                                                                                                                        |

  > ⚠️ **stdio gotcha**: `stdout` IS the JSON-RPC channel. The 1st non-JSON byte on it (a stray `print()`, a `tqdm` bar, a debug log from a dep) silently disconnects the client, no crash, no error. Python's `logging` & `FastMCP` both default to `stderr`, which is safe for the protocol, but **high-volume stderr can still trip up hosts**: Claude for Desktop, for ex., surfaces `stderr` noise as red error indicators.
  > - That's why `cli_project` sets `log_level="ERROR"` in `mcp = FastMCP("docs", log_level="ERROR")` (drops `DEBUG`/`INFO`/`WARNING`, so only real errors reach stderr) on top of stdio's "never touch stdout" rule.

Servers can expose 3 main types of capabilities (**primitives**), distinguished by **who decides when to invoke them**:

| primitive     | controlled by                | results used by | used for                                                       |
| ------------- | ---------------------------- | --------------- | -------------------------------------------------------------- |
| **Tools**     | **Claude** (model decides)   | Claude          | Giving Claude additional functionality (read APIs, run code, mutate state) |
| **Resources** | **the host app** (app decides) | the host app    | Getting data into the app, adding context to messages          |
| **Prompts**   | **the user** (user decides)  | varies          | User-triggered workflows: slash commands, buttons, menu options |

FastMCP API for each:

1. **Tools** (**`@mcp.tool()`**): functions the LLM can call (host mediates user approval).
   - By default: type hints → schema, function name → tool name, docstring → description. Pass **`@mcp.tool(name="...", description="...")`** to override when you want the public API to differ from Python identifiers.
2. **Resources** (**`@mcp.resource(uri)`**): read-only data the host can fetch (e.g. file contents, API responses, DB rows).
    - Like GET handlers in an HTTP server: fetch information, never mutate state. Use **URI templates** (`docs://documents/{doc_id}`) to handle a family of resources w/ one decorator.
3. **Prompts** (**`@mcp.prompt()`**): reusable message templates the host can surface to the user.
   - Returns a `list[base.Message]` that the host can either send straight to Claude or display in a UI. Typically exposed as `/slash` commands in chat UIs.

Ex:

```python
from mcp.server.fastmcp import FastMCP
from pydantic import Field

mcp = FastMCP("docs")

@mcp.tool(
    name="read_doc_contents",
    description="Read the contents of a document and return it as a string.",
)
def read_document(
    doc_id: str = Field(description="Id of the document to read"),
):
    if doc_id not in docs:
        raise ValueError(f"Doc with id {doc_id} not found")

    return docs[doc_id]
    docs[doc_id] = docs[doc_id].replace(old_str, new_str)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

The decorator auto-generates JSON schemas & **`mcp.run()`** wires up the transport.

</details>

<details><summary><b>🔌 MCP client w/ <code>ClientSession</code></b></summary>

The [`mcp` Python SDK](https://github.com/modelcontextprotocol/python-sdk) provides the primitives; you wrap them in your own class.

**Core SDK pieces** (from `mcp` & `mcp.client.*`):

| piece                                                | purpose                                                                                                                                                              |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`ClientSession(read, write)`**                     | High-level JSON-RPC session. Exposes `list_tools()`, `call_tool()`, `list_resources()`, `read_resource()`, `list_prompts()`, `get_prompt()`                          |
| **`StdioServerParameters(command, args, env)`**      | How to launch the server subprocess for stdio                                                                                                                        |
| **`stdio_client(params)`**                           | Async context manager that spawns the server & returns `(read, write)` streams                                                                                       |
| **`streamablehttp_client(url, ...)`**                | Same idea, but for **Streamable HTTP** transport (`mcp.client.streamable_http`)                                                                                      |
| **`AsyncExitStack`**                                 | Composes transport + session lifecycles so cleanup happens in the right order (close session before transport)                                                       |

**Lifecycle** (4 steps):

1. **Set up transport** → `stdio_client(params)` (or `streamablehttp_client(url)`).
2. **Wrap streams in a session** → `ClientSession(read, write)`.
3. **Handshake** → `await session.initialize()` (negotiates capabilities w/ server).
4. **Cleanup** → close session, then transport. `AsyncExitStack` handles ordering.

Ex:

```python
import asyncio
from contextlib import AsyncExitStack
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    params = StdioServerParameters(command="uv", args=["run", "mcp_server.py"])

    async with AsyncExitStack() as stack:
        read, write = await stack.enter_async_context(stdio_client(params))
        session = await stack.enter_async_context(ClientSession(read, write))
        await session.initialize()

        tools = (await session.list_tools()).tools
        result = await session.call_tool("read_doc_contents", {"doc_id": "plan.md"})
        print(result)

asyncio.run(main())
```

> ⚠️ **`initialize()` is mandatory**: skip it & every subsequent call (`list_tools`, `call_tool`, etc.) fails. It's the JSON-RPC handshake where the client & server exchange capabilities.

> 💡 **Wrap it in a class** implementing Python's [**async context manager protocol**](https://docs.python.org/3/reference/datamodel.html#asynchronous-context-managers) (`__aenter__` / `__aexit__`, the async versions of `__enter__` / `__exit__`). That lets callers write `async with MCPClient(...) as client:` & get **automatic cleanup on exit (even on exception)**, no manual `try`/`finally` to close the session & subprocess. See [`cli_project/mcp_client.py`](./cli_project/mcp_client.py), which also adds typed helper methods (`call_tool`, `read_resource`, etc.) over the raw session.

> 📝 **Streamable HTTP variant**: swap `stdio_client(params)` for `streamablehttp_client(url)`. Everything else (session, `initialize()`, all `session.*` calls) is identical. The SDK fully abstracts transport at the `ClientSession` layer.

</details>

<details><summary><b>🔄 End-to-end: the tool-calling loop</b></summary>

**When the user asks a question**, the host runs a back-&-forth between Claude (decides what to do) & the MCP server (runs the code), w/ the client mediating every server interaction. Each turn of the loop:

1. **[client]** `await client.list_tools()` → JSON-RPC `tools/list` request to server.
2. **[server]** Returns the list of registered `@mcp.tool()` schemas (name, description, input schema).
3. **[host]** Sends user message + tool schemas to Claude (Anthropic API).
4. **[Claude]** Reasons about the request, decides it needs a tool.
5. **[Claude]** Picks one (e.g. `read_doc_contents`) & emits a `tool_use` block w/ args.
6. **[client]** `await client.call_tool(name, args)` → JSON-RPC `tools/call` request to server.
7. **[server]** Runs the decorated Python function, returns the result as a `CallToolResult`.
8. **[host]** Sends the tool result back to Claude as a `tool_result` message.
9. **[Claude]** Formulates a natural language response from the tool output.
10. **[host]** Displays the response to the user.

**Roles**:
- **[client]** = bridge. Steps 1 & 6 (sends JSON-RPC requests, returns parsed results).
- **[server]** = executor. Steps 2 & 7 (handles `tools/list`, `tools/call`, runs the tool code).
- **[host]** = orchestrator. Steps 3, 8, 10 (Anthropic API calls, message threading, UI).
- **[Claude]** = brain. Steps 4, 5, 9 (reasoning, tool selection, response generation).

> 💡 If Claude doesn't need a tool, steps 1-2 & 6-7 still happen (the host always advertises tools), but Claude skips steps 5-8 & goes straight to 9-10. Tool use is Claude's choice per turn.

</details>


## Components

- MCP host: **[Claude for Desktop](https://claude.com/download)**
- Python package installer & resolver: **[uv](https://github.com/astral-sh/uv)**

### Setup

1. Download **Claude for Desktop**
2. Install **uv**
    ```shell
      curl -LsSf https://astral.sh/uv/install.sh | sh
    ```

## Projects

### Server

---

#### ❥ [Weather MCP Server](./weather_mcp_server)
- **Server**: tools only


### Server & Client

---

> 📝 **Note**: real-world projects usually implement _one_:
> 1. **MCP Server** to expose your service
> 1. **MCP Client** to consume existing MCP servers
>
> Below implement both just for educational purposes.

#### ❥ [CLI Chat](./cli_project)
-  **Server**: all 3 primitives, plus uses `pydantic.Field` for richer parameter descriptions.


## Debugging

### The server inspector

From inside the project directory:

1) Make sure your Python environment is activated
    ```shell
    /opt/homebrew/bin/python3 -m venv .venv
    source .venv/bin/activate
    ```
2) Run the **server inspector** with:

    ```shell
    # mcp dev mcp_server.py or mcp dev weather.py
    mcp dev <server-file>
    ```

    This starts a development server on port 6277 and gives you a local URL to open in your browser. The inspector interface will load, showing the MCP Inspector dashboard.

3) Click the "Connect" button on the left side to start your MCP server.
    - To test your tools:
      1. Navigate to the Tools section
      2. Click "List Tools" to see all available tools
      3. Select a tool to open its testing interface
      4. Fill in the required parameters
      5. Click "Run Tool" to execute and see results

## Resources
- [Model Context Protocol Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Model Context Protocol Docs](https://modelcontextprotocol.io/docs)
