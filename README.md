# Local AI Doc Assistant

A local AI application that answers questions about Microsoft products and technologies using on-device inference with [Foundry Local](https://learn.microsoft.com/azure/foundry-local/) and grounded documentation retrieval from the [Microsoft Learn MCP Server](https://learn.microsoft.com/training/support/mcp).

## How it works

```
User Question
       │
       ▼
┌────────────────┐   search query   ┌───────────────────────┐
│  Foundry Local  │ ──────────────► │  Learn MCP Server      │
│  (phi-4-mini)   │ ◄────────────── │  learn.microsoft.com   │
│  on-device      │   doc content   │  /api/mcp              │
└────────────────┘                  └───────────────────────┘
       │
       ▼
Grounded Answer
```

1. You ask a question about a Microsoft product or service
2. The local model (`phi-4-mini`) calls a `search_docs` tool via tool calling
3. The tool queries Learn MCP Server (`microsoft_docs_search`) for relevant documentation
4. Documentation content is fed back to the model as tool results
5. The model synthesizes an answer grounded in official Microsoft docs

All inference runs **locally on your device** — no API keys, no cloud LLM costs. Only the doc search query goes to Learn MCP Server (no auth required).

![Screenshot of the Learn Doc Assistant answering "What is Foundry Local?"](Screenshot%202026-04-10%20125611.png)

## Prerequisites

- **Windows 11** (Foundry Local SDK does not yet work on Linux)
- **Node.js** 20+
- **Foundry Local** — `winget install Microsoft.FoundryLocal`

## Getting started

```bash
npm install
node app.js
```

On first run, the `phi-4-mini` model (~2-3 GB) and execution providers will be downloaded automatically.

```
Learn Doc Assistant ready! Ask about any Microsoft product or technology.
Type 'quit' to exit.

You: What is Foundry Local?
  Tool call: search_docs({"query":"Foundry Local"})
  [Searching Learn MCP Server for: "Foundry Local"]

A: Foundry Local is an on-device AI solution that allows you to build and run
AI models directly within your application...
```

## Changes from original sample

Adapted from the [Foundry Local tutorial-tool-calling sample](https://github.com/microsoft/Foundry-Local/tree/main/samples/js/tutorial-tool-calling). Key changes:

1. **Tool**: Replaced `get_weather`/`calculate` with `search_docs` — calls Learn MCP Server via HTTP (JSON-RPC)
2. **Model**: `qwen2.5-0.5b` → `phi-4-mini` (Microsoft, better reasoning)
3. **System prompt**: Instructs the model to always search docs before answering
4. **`tool_choice: required`**: Forces the model to call the search tool
5. **Response truncation**: MCP results truncated to ~2000 chars to stay within memory limits
6. **Async tool loop**: Tool implementations are async to support the HTTP call

## Design decisions

- **No source citations** — the model doesn't reliably include URLs in answers; a production app could post-process citations from the MCP response
- **2000 char context limit** — MCP results are truncated to keep memory usage safe on consumer hardware; could be increased with larger models
- **MCP via raw HTTP** — keeps the sample simple and close to the original tutorial; a proper MCP SDK client (`@modelcontextprotocol/sdk`) would be more robust

## Known limitations

- **Windows only** — Linux SDK crashes on `catalog.getModel()` ([#626](https://github.com/microsoft/Foundry-Local/issues/626)), and there's no Foundry CLI for Linux yet ([#625](https://github.com/microsoft/Foundry-Local/issues/625))
- **CUDA EP mismatch** — falls back to CPU (slower inference) due to ORT version mismatch with the CUDA execution provider

## References

- [Foundry Local docs](https://learn.microsoft.com/azure/foundry-local/)
- [Foundry Local GA blog post](https://devblogs.microsoft.com/foundry/foundry-local-ga/)
- [Foundry Local samples](https://github.com/microsoft/Foundry-Local/tree/main/samples)
- [Learn MCP Server overview](https://learn.microsoft.com/training/support/mcp)
- [Learn MCP Server developer reference](https://learn.microsoft.com/training/support/mcp-developer-reference)