# build-mcp-server

A server that exposes two tools: `get_alerts` and `get_forecast`. Then we’ll connect the server to an MCP host (in this case, Claude for Desktop).

## Installation

Install uv

```shell
curl -LsSf https://astral.sh/uv/install.sh | sh
```

## Running the server

```shell
cd weather && uv run main.py
```
