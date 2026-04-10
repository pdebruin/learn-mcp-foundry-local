# Exploration Log — Foundry Local + Learn MCP Server

## Goal

Build a local AI app using Foundry Local that connects to Learn MCP Server to answer questions grounded in Microsoft product docs.

---

## Step 1: MCP Client Approach

**Question:** How should our app communicate with Learn MCP Server?

### Options

| Option | Description | Pros | Cons |
|---|---|---|---|
| **1. MCP SDK** (`pip install mcp`) | Official MCP Python SDK with streamable HTTP transport | Future-proof; handles protocol correctly; lightweight | Extra dependency; async code |
| **2. Agent framework** (Semantic Kernel, LangChain) | Frameworks with built-in MCP connectors | Higher-level orchestration; tool chaining | Heavy dependency for a simple app |
| **3. Direct HTTP** | Raw POST requests to `https://learn.microsoft.com/api/mcp` | Minimal dependencies; simple | Learn docs warn interface may change; need to reverse-engineer request format |

### Decision

**Option 1 — MCP SDK.** The Learn MCP Server [developer docs](https://learn.microsoft.com/training/support/mcp-developer-reference) explicitly recommend using an agent framework or MCP client, not direct HTTP. The MCP Python SDK is purpose-built and lightweight. Agent frameworks are overkill for this prototype.

### Key details

- Install: `pip install mcp`
- Endpoint: `https://learn.microsoft.com/api/mcp`
- No authentication required
- Tools available: `microsoft_docs_search`, `microsoft_docs_fetch`, `microsoft_code_sample_search`
- Client discovers tools dynamically via `session.list_tools()`

---

## Step 2: Language Choice

**Decision: JavaScript/Node.js.** Good SDK support and solid samples in the Foundry Local repo.

---

## Step 3: Are the Foundry Local tool-calling samples useful for MCP?

**Question:** Can we use the existing tool-calling samples as a base for our Learn MCP integration?

### Samples reviewed

| Sample | Pattern | Key detail |
|---|---|---|
| [tutorial-tool-calling](https://github.com/microsoft/Foundry-Local/blob/main/samples/js/tutorial-tool-calling/app.js) | Native SDK (`model.createChatClient()`) + tool loop | Simpler, no web server needed. Has `processToolCalls` loop. |
| [tool-calling-foundry-local](https://github.com/microsoft/Foundry-Local/blob/main/samples/js/tool-calling-foundry-local/src/app.js) | OpenAI SDK + Foundry web server + streaming | Shows streaming responses. Uses `openai` npm package. |

### How this maps to our app

The tool-calling pattern is exactly what we need:

1. Define a `search_docs` tool (OpenAI function schema) that the local model can call
2. When the model calls it, our code hits Learn MCP Server to fetch documentation
3. Feed the doc content back as the tool result
4. The model synthesizes a grounded answer from the docs

### Key insight: MCP client may not be needed

The **Foundry Local model acts as the agent**. It decides when to call tools. We just need to implement the tool functions that talk to Learn MCP Server. This could be done via:

- **MCP SDK** (`@modelcontextprotocol/sdk`) — proper MCP client, discovers tools dynamically
- **Direct HTTP** — simpler, just call the MCP endpoint with the right payload for `microsoft_docs_search`

Since the model's tool-calling drives everything, the MCP client question from Step 1 becomes less critical — it's just an implementation detail of how our `search_docs` function fetches data.

### Recommended base

**tutorial-tool-calling** — cleaner, uses native SDK, has the `processToolCalls` loop we can adapt.

---

## Step 4: Model Selection

**Question:** Which Foundry Local model should we use?

### Hardware

- **GPU:** NVIDIA RTX A2000 Laptop (4 GB VRAM)
- **CPU:** Intel i7-11370H
- **RAM:** 16 GB
- **OS:** Ubuntu 24.04 on WSL2, Windows 11, Surface Laptop Studio

### Constraints

1. Must support **tool calling** — our architecture needs the model to invoke a `search_docs` tool
2. Must fit in **4 GB VRAM**
3. Prefer **Microsoft models** (Phi family)

### Models with tool calling that fit 4GB VRAM

| Model | GPU Variant | Size | Tools |
|---|---|---|---|
| `qwen2.5-0.5b` (Alibaba) | OpenVINO | 0.36 GB | ✅ |
| `qwen2.5-1.5b` (Alibaba) | OpenVINO | 1.00 GB | ✅ |
| **`phi-4-mini` (Microsoft)** | **OpenVINO** | **2.15 GB** | **✅** |

### Decision

**`phi-4-mini`** — Microsoft's own model, best reasoning quality of the options, supports tool calling, fits in 4GB VRAM (OpenVINO variant at 2.15 GB). Falls back to CPU (4.80 GB) if GPU has issues.

---

## Step 5: Context Window

**Phi-4-mini has 128K token context.** Learn MCP Server returns up to 10 chunks of ~500 tokens each (~5K tokens). No constraint here.

---

## Step 6: Streaming

**Decision: Yes, use streaming.** The `tutorial-tool-calling` sample already uses non-streaming, but `tool-calling-foundry-local` uses streaming with the OpenAI SDK. We'll base our app on the streaming sample to reuse as much code as possible. Can fall back to non-streaming if issues arise.

---

## Summary of Decisions

| Question | Decision |
|---|---|
| MCP client approach | MCP SDK (`@modelcontextprotocol/sdk`) or direct HTTP — implementation detail of the tool function |
| Language | JavaScript/Node.js |
| Base sample | `tool-calling-foundry-local` (streaming, OpenAI SDK pattern) |
| Model | `phi-4-mini` (Microsoft, 2.15 GB OpenVINO GPU, tool calling) |
| Context window | Not a constraint (128K tokens vs ~5K from Learn MCP) |
| Streaming | Yes, reuse from sample |

## Next: Build It

### Attempt 1 — Results

**Environment:** Windows 11, PowerShell, Node.js 20.10.0, `foundry-local-sdk` + `foundry-local-sdk-winml`

1. ✅ `native-chat-completions` sample works — model loads, answers questions, streams
2. ✅ `tutorial-tool-calling` sample works — `get_weather` tool called successfully with `qwen2.5-0.5b`
3. ❌ Adapted app with `search_docs` tool — model **never calls the tool** (finish_reason: stop, tool_calls: 0)
   - Tried `phi-4-mini` — no tool calls
   - Tried `qwen2.5-0.5b` — no tool calls
   - Even asking about weather (no weather tool defined) — no tool calls
   - Stronger system prompts didn't help
4. ❌ `phi-4-mini` loaded but used ~11GB memory (expected ~2-3GB)
5. ❌ CUDA EP failed with API version mismatch (ORT 1.23.2 vs requested API 24)

### Attempt 2 — Working! 🎉

Key fixes that got it working:

1. **`tool_choice: { type: 'required' }`** — forces the model to call the tool (without this, `qwen2.5-0.5b` ignores tools entirely)
2. **Truncate MCP response to ~2000 chars** — the raw response is too large and causes OOM (8.7GB allocation failure)
3. **Parse MCP JSON results** — Learn MCP Server returns JSON text containing `results[]` with `title`, `content`, `contentUrl`. Need to extract and format as clean markdown.
4. **Clear `toolChoice` on follow-up calls** — prevents infinite tool-calling loop after results are returned

### Working flow

```
User question
  → Model calls search_docs (forced via tool_choice: required)
    → HTTP POST to Learn MCP Server (JSON-RPC: tools/call microsoft_docs_search)
      → Returns top 3 doc chunks with titles + URLs
    → Fed back as tool result (~2000 chars max)
  → Model synthesizes answer from docs (tool_choice cleared)
→ Grounded answer displayed
```

### phi-4-mini Results

Switched from `qwen2.5-0.5b` to `phi-4-mini`. Noticeably better:

- ✅ Tool calling works with `tool_choice: required`
- ✅ Answers are well-structured and grounded in retrieved docs
- ✅ "What is Foundry Local?" — accurate answer pulled from Learn docs
- ⚠️ "How to create azure storage account using az cli?" — Learn MCP returned the `azd` tab content, not the `az` CLI section. Model faithfully answered from docs it received (correct behavior, retrieval limitation)
- ⚠️ No source URL citations in answers (model doesn't include them)
- ⚠️ CUDA EP still fails, running on CPU fallback

### Conclusion

**Proof of concept is successful.** Foundry Local + Learn MCP Server can work together to create a local AI doc assistant. The architecture works:

1. Foundry Local runs `phi-4-mini` on-device for inference
2. Learn MCP Server provides grounded Microsoft documentation via HTTP
3. Tool calling connects the two — model decides when to search docs
4. Answers are grounded in real docs, not hallucinated

### What would improve this

- **Larger context window / truncation budget** — 2000 chars is tight; more doc content = better answers
- **Better model** — a 7B+ model would reason better over docs and cite sources
- **CUDA fix** — GPU acceleration would make inference much faster
- **MCP SDK client** — proper MCP client instead of raw HTTP for protocol correctness
- **Source citations** — prompt engineering or post-processing to include URLs
- **Linux support** — blocked by SDK segfault ([#626](https://github.com/microsoft/Foundry-Local/issues/626))

### Blockers on Linux (WSL2)

- No Foundry CLI for Linux ([#625](https://github.com/microsoft/Foundry-Local/issues/625))
- JS SDK segfaults on `catalog.getModel()` ([#626](https://github.com/microsoft/Foundry-Local/issues/626))
- `foundry-local-sdk-winml` in optionalDependencies blocks Linux native binary download (must remove from package.json)
