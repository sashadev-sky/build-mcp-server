# MCP Chat

MCP Chat is a **command-line interface application** that enables interactive chat capabilities with AI models through the Anthropic API. The application supports document retrieval, command-based prompts, and extensible tool integrations via the MCP (Model Control Protocol) architecture.

- CLI based chatbot that allows users to chat with a set of documents
  - _"what's 1+1?"_
  - _"can you please summarize the contents of @"_
- Claude should be able to read a document
- Claude should be able to edit a document
- Users can "mention" a document by writing out "@doc_name"
- The doc's contents will automatically be included as context
- Users can run a "command" with "/command_name"

## Implementation

**Host, client, & server**:
> `main.py` is the host & **`MCPClient`** is the client it embeds.
- **MCP host**: `main.py`
  - <u>Owns the LLM session</u>: the Anthropic client lives in `main.py` via **`Anthropic()`**
  - <u>Decides which servers to launch</u>: `main.py` calls **`MCPClient(command=..., args=...)`**
  - <u>Mediates user consent / input</u>: the CLI loop in `main.py`
- **MCP server**: an MCP server that manages documents stored in memory. Provides:
  1. **Tools**: 2 essential tools with custom `name`s & `description`s + uses **`pydantic.Field`** for richer parameter descriptions.
     1. Read document contents
     2. Update document contents through find-and-replace operations.
  1. **Resources**: with MIME types
  1. **Prompts**: that return message lists.
- **MCP client**: `MCPClient`


## Prerequisites

- Python >=3.10,\<3.14
- Anthropic API Key

## Setup

### Configure the environment variables

Create or edit the `.env` file in this project root and verify that the following variables are set correctly:

```sh
ANTHROPIC_API_KEY=""  # Enter your Anthropic API secret key
```

### Install dependencies

Pin the project to Python 3.13 so `uv` grabs a prebuilt wheel

```shell
cd cli_project
uv python install 3.13
uv venv --python 3.13
uv sync
```

### Running the server

```shell
uv run main.py
```

## Usage

### Basic Interaction

Simply type your message and press Enter to chat with the model.

### Document Retrieval

Use the @ symbol followed by a document ID to include document content in your query:

```
> Tell me about @deposition.md
```

### Commands

Use the / prefix to execute commands defined in the MCP server:

```
> /summarize deposition.md
```

Commands will auto-complete when you press Tab.

## Development

### Adding New Documents

Edit the `mcp_server.py` file to add new documents to the `docs` dictionary.

### Implementing MCP Features

To fully implement the MCP features:

1. Complete the TODOs in `mcp_server.py`
2. Implement the missing functionality in `mcp_client.py`

### Linting and Typing Check

There are no lint or type checks implemented.
