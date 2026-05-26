# Weather MCP Server

A server that exposes 2 tools:
1. `get_alerts`
2. `get_forecast`

Then we'll connect the server to an **MCP host** (in this case, **Claude for Desktop**).

## Running the server

```shell
cd weather_mcp_server && uv run main.py
```

## Project structure

- `main.py` calls **`mcp.run(transport="stdio")`**, which starts an MCP server speaking **JSON-RPC** over **stdin/stdout**.
