# MCP Chat

MCP Chat is a **command-line interface application** that enables interactive chat capabilities with AI models through the Anthropic API. The application supports document retrieval, command-based prompts, and extensible tool integrations via the MCP (Model Control Protocol) architecture.

- CLI based chatbot that allows users to chat with a set of documents
  - _"what's 1+1?"_
  - _"tell me about @"_
- Claude should be able to read a document **[tool: `read_doc_contents`]**
- Claude should be able to edit a document **[tool: `edit_document`]**
- Users can "mention" a document by writing out "@doc_name" **[resource: `docs://documents/{doc_id}`]**
  - The doc's contents will automatically be included as context
- Users can run a "command" with "/command_name" **[prompt: `format`, prompt: `summarize`]**
  - _"/format {doc_id}"_
  - _"/summarize {doc_id}"_

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
  1. **Resources**: 2 resources w/ explicit `mime_type=`s & URI templates.
     1. `docs://documents` (**`application/json`**): list all doc IDs.
     2. `docs://documents/{doc_id}` (**`text/plain`**): fetch a single doc's contents. The `{doc_id}` placeholder is a URI template parameter, bound to the function's `doc_id` arg at call time.
  1. **Prompts**: that return message lists.
- **MCP client**: `MCPClient`
  - The client acts as the bridge between our application logic and the MCP server, making it easy to access server functionality without worrying about the underlying connection details.


### Understanding Resources Through an Example
Let's say you want to build a document mention feature where users can type `@document_name` to reference files. This requires two operations:
1. Getting a list of all available documents (for autocomplete): when a user types @, you need to show available documents.
2. Fetching the contents of a specific document (when mentioned): when they submit a message with a mention, you automatically inject that document's content into the prompt sent to Claude.


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

## Testing Prompts in the CLI

Once implemented, you can test prompts through the command-line interface. When you type a forward slash, available prompts appear as commands. Selecting a prompt may prompt you to choose from available options (like document IDs), and then the complete prompt gets sent to Claude.

The workflow looks like this:
- User selects a prompt (like "format")
- System prompts for required arguments (like which document to format)
- The prompt gets sent to Claude with the interpolated values
- Claude can then use tools to fetch additional data and complete the task

## Testing the Client

To test our implementation, we can run the client directly. The file includes a **testing harness** that connects to our MCP server and calls our methods:

```python
async with MCPClient(
    command="uv", args=["run", "mcp_server.py"]
) as client:
    result = await client.list_tools()
    print(result)
```

When we run this test, we should see our tool definitions printed out, including the `read_doc_contents` and `edit_document tools` we created earlier.

```shell
uv run mcp_client.py
# [Tool(name='read_doc_contents', title=None, description='Read the contents of a document and return it as a string.', inputSchema={'properties': {'doc_id': {'description': 'Id of the document to read', 'title': 'Doc Id', 'type': 'string'}}, 'required': ['doc_id'], 'title': 'read_documentArguments', 'type': 'object'}, outputSchema=None, icons=None, annotations=None, meta=None, execution=None), Tool(name='edit_document', title=None, description='Edit a document by replacing a string in the documents content with a new string', inputSchema={'properties': {'doc_id': {'description': 'Id of the document that will be edited', 'title': 'Doc Id', 'type': 'string'}, 'old_str': {'description': 'The text to replace. Must match exactly, including whitespace', 'title': 'Old Str', 'type': 'string'}, 'new_str': {'description': 'The new text to insert in place of the old text', 'title': 'New Str', 'type': 'string'}}, 'required': ['doc_id', 'old_str', 'new_str'], 'title': 'edit_documentArguments', 'type': 'object'}, outputSchema=None, icons=None, annotations=None, meta=None, execution=None)]
```
