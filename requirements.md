# Local AI Doc Assistant — Requirements

## Goal

Build a local AI application using **Foundry Local** that connects to the **Learn MCP Server** to answer user questions grounded in official Microsoft product documentation. The app runs inference on-device (private, zero-cost) and retrieves up-to-date docs from Learn MCP Server as its knowledge source.

## How It Works

```
User Question
     │
     ▼
┌──────────────┐    search/fetch     ┌────────────────────┐
│ Foundry Local │ ──────────────────► │  Learn MCP Server  │
│ (local LLM)  │ ◄────────────────── │  learn.microsoft.com/api/mcp │
└──────────────┘    doc content       └────────────────────┘
     │
     ▼
Grounded Answer
```

1. User asks a question about a Microsoft product or service.
2. The app calls Learn MCP Server (`microsoft_docs_search`) to retrieve relevant documentation.
3. Optionally calls `microsoft_docs_fetch` to get full article content for the most relevant result.
4. Passes the retrieved documentation + user question to a Foundry Local model for on-device inference.
5. The local model synthesizes an answer grounded in the retrieved docs.

## Components

### Foundry Local (on-device inference)

| Attribute | Detail |
|---|---|
| Status | Generally Available (April 2026) |
| Repo | <https://github.com/microsoft/Foundry-Local> |
| Docs | <https://learn.microsoft.com/azure/foundry-local/> |
| SDKs | Python, JavaScript, C#, Rust (~20 MB runtime) |
| Install (Python) | `pip install foundry-local-sdk` (macOS/Linux) or `pip install foundry-local-sdk-winml` (Windows) |
| Install (JS) | `npm install foundry-local-sdk` (macOS/Linux) or `npm install foundry-local-sdk-winml` (Windows) |
| API format | OpenAI chat completions request/response format |
| Suggested models | `qwen2.5-0.5b` (lightweight), or larger models from the catalog for better quality |

### Learn MCP Server (documentation retrieval)

| Attribute | Detail |
|---|---|
| Status | Generally Available (November 2025) |
| Endpoint | `https://learn.microsoft.com/api/mcp` |
| Auth | None required |
| Protocol | Streamable HTTP (MCP) |
| Key tools | `microsoft_docs_search` — semantic search returning up to 10 content chunks |
| | `microsoft_docs_fetch` — fetch full article as markdown |
| | `microsoft_code_sample_search` — find code samples by query |

## Requirements

### Functional

1. **Accept a natural language question** from the user (CLI or simple UI).
2. **Search Learn MCP Server** for relevant documentation using `microsoft_docs_search`.
3. **Optionally fetch** full articles via `microsoft_docs_fetch` when deeper context is needed.
4. **Build a prompt** that includes the retrieved doc content as context and the user's question.
5. **Run inference locally** via Foundry Local to produce a grounded answer.
6. **Display the answer** along with source URLs from Learn so the user can verify.

### Non-Functional

7. **Privacy** — User questions and answers stay on-device; only the doc search query goes to Learn MCP Server.
8. **No API keys required** — Neither Foundry Local nor Learn MCP Server require authentication.
9. **Cross-platform** — Should work on Windows, macOS, and Linux.
10. **Lightweight** — Minimal dependencies beyond Foundry Local SDK and an MCP/HTTP client.

## Open Questions

1. **MCP client** — Should we use a full MCP client library to talk to Learn MCP Server, or simply make HTTP requests to the streamable HTTP endpoint directly?
2. **Language choice** — Python is the simplest to prototype; JavaScript has good Foundry Local samples too. Which to start with?
3. **Model selection** — Which Foundry Local catalog model gives the best quality for doc-grounded Q&A within reasonable hardware constraints?
4. **Context window** — How much retrieved doc content fits in the local model's context window? May need to truncate or summarize.
5. **Streaming** — Should responses stream token-by-token for better UX?

## References

- [Foundry Local GA blog post](https://devblogs.microsoft.com/foundry/foundry-local-ga/)
- [Foundry Local samples](https://github.com/microsoft/Foundry-Local/tree/main/samples)
- [Learn MCP Server overview](https://learn.microsoft.com/training/support/mcp)
- [Learn MCP Server developer reference](https://learn.microsoft.com/training/support/mcp-developer-reference)
