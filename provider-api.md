# Provider API Specification

This document outlines the closures required for implementing a provider in the cross.stream LLM framework. Each provider must implement this common interface to ensure compatibility with the command handler.

## Core Provider Interface

Each provider must export a record with the following closures:

## `models`

Retrieves available models from the provider.

**Input:** 
- `key: string` - API key for provider authentication

**Example Input:**
```nushell
"sk-ant-api03-key-example..." # API key string
```

**Output:**
- A list of available model records, each containing at least:
  - `id: string` - The model identifier
  - `created: datetime` - When the model was created/updated


**Example Output:**
```nushell
[
  {id: "claude-3-5-sonnet-20241022", created: 2024-10-22T00:00:00Z},
  {id: "claude-3-7-sonnet-20250219", created: 2025-02-19T00:00:00Z}
]
```

## `call`

Makes an API call to the provider to generate a response.

**Input:**
- `key: string` - API key for provider authentication
- `model: string` - Model identifier to use
- `tools?: list` - Optional list of tools definitions
- Messages records in standard format (from pipeline)

**Example Inputs:**
```nushell
# Key
"sk-ant-api03-key-example..."

# Model
"claude-3-7-sonnet-20250219"

# Tools (Optional)
[
  {
    name: "read_file",
    description: "Read the contents of a file",
    input_schema: {
      type: "object",
      properties: {
        path: {type: "string", description: "Path to the file"}
      },
      required: ["path"]
    }
  }
]
```

**Output:**
- Raw response events in the provider's format

**Example Call:**
```nushell
[
  {role: "user", content: [{type: "text", text: "Hello, how are you?"}]},
  {role: "assistant", content: [{type: "text", text: "I'm doing well, how can I help?"}]},
  {role: "user", content: [{type: "text", text: "Tell me about cross.stream"}]}
]
```

## `response_stream_aggregate`

Aggregates streaming response events into a final response.

**Input:**
- Stream of provider-specific events

**Output:**
- Complete response record with normalized structure

**Example Input (Stream of Events):**
```nushell
[
  {type: "message_start", message: {id: "msg_01ABCDabcd", model: "claude-3-7-sonnet-20250219"}},
  {type: "content_block_start", content_block: {type: "text", text: []}},
  {type: "content_block_delta", delta: {type: "text_delta", text: "Cross.stream"}},
  {type: "content_block_delta", delta: {type: "text_delta", text: " is an"}},
  {type: "content_block_delta", delta: {type: "text_delta", text: " event streaming"}}, 
  {type: "content_block_delta", delta: {type: "text_delta", text: " framework"}},
  {type: "content_block_stop"},
  {type: "message_stop"}
]
```

**Example Output:**
```nushell
{
  role: "assistant",
  mime_type: "application/json",
  message: {
    id: "msg_01ABCDabcd",
    content: [
      {type: "text", text: "Cross.stream is an event streaming framework..."}
    ],
    model: "claude-3-7-sonnet-20250219",
    stop_reason: "end_turn"
  }
}
```

## `response_stream_streamer`

Transforms provider-specific events into a normalized streaming format for real-time display. This closure should process the provider's native event format and emit a stream of standard records.

**Input:**
- Stream of provider-specific events

**Output:**
- Stream of normalized events in one of two formats:
  1. Content block type indicator: `{type: string}` - Signals the start of a new content block
  2. Content addition: `{content: string}` - Appends content to the current block

**Normalized Streaming Protocol:**
The streaming protocol follows these rules:
- A `{type: "text"}` record indicates the start of a text block
- A `{type: "tool_use", name: "tool_name"}` record indicates the start of a tool use block
- After a type indicator, subsequent `{content: "..."}` records append content to that block
- Content continues to be added to the current block until a new type indicator appears
- This protocol allows for consistent real-time display of different response types

**Example Input (Single Event):**
```nushell
{type: "content_block_delta", delta: {type: "text_delta", text: "Cross.stream is an"}}
```

**Example Output (Content Addition):**
```nushell
{
  content: "Cross.stream is an"
}
```

**Example Complete Stream:**
```nushell
# Start of text block
{type: "text"}
{content: "Cross.stream is "}
{content: "an event streaming framework."}

# Start of tool use block
{type: "tool_use", name: "read_file"}
{content: "{\"path\":"}
{content: "\"/example.txt\"}"}
```

## `convert-mcp-toolslist-to-provider`

Converts MCP tool definitions to provider-specific format.

**Input:**
- List of MCP tool definitions

**Output:**
- List of provider-specific tool definitions

**Example Input:**
```nushell
[
  {
    name: "read_file",
    description: "Read the contents of a file",
    inputSchema: {
      type: "object",
      properties: {
        path: {type: "string", description: "Path to the file"}
      },
      required: ["path"]
    }
  }
]
```

**Example Output (Anthropic format):**
```nushell
[
  {
    name: "read_file",
    description: "Read the contents of a file",
    input_schema: {
      type: "object",
      properties: {
        path: {type: "string", description: "Path to the file"}
      },
      required: ["path"]
    }
  }
]
```

## `response_to_toolscall_mcp`

Extracts tool calls from provider responses and converts to MCP format.

**Input:**
- Provider-specific response containing a tool call

**Example Input (Anthropic Response):**
```nushell
{
  id: "msg_01ABCDabcd",
  message: {
    content: [
      {
        type: "tool_use",
        id: "tu_01ABCDabcd",
        name: "read_file",
        input: {path: "/path/to/file.txt"}
      }
    ]
  }
}
```

**Example Output (MCP Format):**
```nushell
{
  "jsonrpc": "2.0",
  "id": "tu_01ABCDabcd",
  "method": "tools/call",
  "params": {
    "name": "read_file",
    "arguments": {path: "/path/to/file.txt"}
  }
}
```

## Implementation Details

The Anthropic provider implements these closures with specific adaptations for the Anthropic API:

1. For tools, it renames `inputSchema` to `input_schema` to match Anthropic's format
2. It adds proper headers including API version and beta features when tools are used
3. It handles streaming response formats specific to Anthropic's SSE format
4. It correctly processes tool usage, including extraction and conversion

When implementing a new provider, ensure all closures maintain the same interface signatures while adapting the internals to match the provider's specific requirements.