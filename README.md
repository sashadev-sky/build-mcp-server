# Building with MCP

## 📝 Notes

<details><summary><b>3️⃣ distinct roles in MCP's spec: Host, Client, & Server</b></summary>

1. **Host**: the app the user interacts with (e.g. **Claude for Desktop**, **Cursor**, **VS Code**). It:
     - Owns the LLM session
     - Manages user consent
     - Decides which servers to launch
2. **Client**: a connector *inside* the host. The host spins up 1 client/server, & each client maintains a 1:1 stateful **JSON-RPC** session w/ its server, handling:
     - **`initialize`**
     - Capability negotiation
     - Message routing
3. **Server**: your process (e.g. `weather_mcp_server`) exposing:
     - Tools
     - Resources
     - Prompts

</details>

## Components

- MCP host: **[Claude for Desktop](https://claude.com/download)**
- Python package installer & resolver: **[uv](https://github.com/astral-sh/uv)**

## Setup

1. Download **Claude for Desktop**
    Then register your server by editing `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

    ```json
    {
      "mcpServers": {
        "weather": {
          "command": "uv",
          "args": ["--directory", "/ABSOLUTE/PATH/TO/weather_mcp_server", "run", "main.py"]
        }
      }
    }
    ```

    Restart Claude Desktop. The hammer icon in the chat input confirms the server loaded.

2. Install **uv**

    ```shell
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```
---

## Server

---

### 1) [weather_mcp_server](./weather_mcp_server)

---

## Server & Client

---

> 📝 **Important Note!**: a real-world project typically implements either an **MCP client** or an **MCP server**, not both. You might create:
>
> - An MCP server to expose your service to other developers
> - An MCP client to connect to existing MCP servers

### 1) [cli_project](./cli_project)

---
