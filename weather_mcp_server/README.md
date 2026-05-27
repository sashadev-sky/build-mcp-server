# Weather MCP Server

An **MCP server** that exposes 2 **tools**:

1. `get_alerts`
   - _"what are weather alerts for NY?"_
2. `get_forecast`

To use, run the server then connect it to an **MCP host** (in this case, **Claude for Desktop**).

- (No client, Claude for Desktop spwans the client internally).

**Implementation**:

- Demonstrates a simple tool decorator

## Setup

### Install dependencies

Pin the project to Python 3.13 so `uv` grabs a prebuilt wheel

```shell
cd weather_mcp_server
uv python install 3.13
uv venv --python 3.13
uv sync
```

### Running the server

```shell
uv run main.py
```

### Registering the server

Register your server to Claude for Desktop by editing **`claude_desktop_config.json`**:

- In `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows), add it to the `mcpServers` field:

    ```json
    {
        "mcpServers": {
            "weather": {
                "command": "uv",
                "args": [
                    "--directory",
                    "/Users/sashaboginsky/build-mcp/weather_mcp_server",
                    "run",
                    "main.py"
                ]
            }
        }
    }
    ```

- Restart Claude Desktop

## Project structure

- `main.py` calls `mcp.run(transport="stdio")`, which starts an MCP server speaking **JSON-RPC** over **stdin/stdout**
