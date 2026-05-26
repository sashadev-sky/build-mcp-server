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

**Important Architecture Note!**
In real-world projects, you typically implement either an **MCP client** or an **MCP server**, not both. You might create:
- An MCP server to expose your service to other developers
- An MCP client to connect to existing MCP servers

Our project will implement both just so we understand how they work:
- **MCP server**: we created an MCP server that manages documents stored in memory. The server will provide two essential tools:
  1. One to read document contents and
  2. One to update them through find-and-replace operations.

## Prerequisites

- Python >=3.10,<3.14
- Anthropic API Key

## Setup

### Step 1: Configure the environment variables

1. Create or edit the `.env` file in the project root and verify that the following variables are set correctly:

```
ANTHROPIC_API_KEY=""  # Enter your Anthropic API secret key
```

### Step 2: Install dependencies

#### Option 1: Setup with uv

[uv](https://github.com/astral-sh/uv) is a fast Python package installer and resolver.

1) Install uv, if not already installed:

```bash
cd cli_project
pip install uv
```

2) Pin the project to Python 3.13 so `uv` grabs a prebuilt wheel

```shell
rm -rf uv.lock
uv python install 3.13
uv venv --python 3.13
uv sync
```

3) Run the project

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
