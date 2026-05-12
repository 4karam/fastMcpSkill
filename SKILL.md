---
name: fastmcp-server
description: Concise blueprint for building MCP servers with FastMCP (Python). Covers core primitives (tools, resources, prompts), Context injection, lifespan management, server composition, transports, deployment, and security. Use when creating, reviewing, or deploying any MCP server.
---

# FastMCP Server — Production Blueprint

> **Design philosophy**: Every tool is atomic. Every resource is read-only. Every prompt is reusable. If the LLM can misuse it, the design is wrong.

---

## 1. What is FastMCP

Python framework that abstracts MCP protocol into decorators. Handles JSON-RPC, schema generation, transport negotiation. You write functions — FastMCP exposes them as MCP primitives.

**Install**: `pip install fastmcp` — **Docs**: [gofastmcp.com](https://gofastmcp.com)

---

## 2. Core Primitives

| Primitive | Decorator | Purpose | LLM Interaction |
|---|---|---|---|
| **Tool** | `@mcp.tool` | Action with side effects | LLM calls it |
| **Resource** | `@mcp.resource(uri)` | Read-only data by URI | LLM fetches it |
| **Prompt** | `@mcp.prompt` | Reusable prompt template | LLM uses as instruction |

### Design Rules

| Rule | Applies To |
|---|---|
| One function = one action — never bundle multiple operations | Tools |
| Always type-hint all parameters AND return type | All |
| Docstring = description shown to LLM — be precise, not verbose | All |
| Raise `ToolError` for user-facing failures — never raw exceptions | Tools |
| URI must be semantic and stable (`config://app`, `users://{id}`) | Resources |
| Dynamic URIs use path parameters (`{param}`) — auto-extracted | Resources |
| Return `str` or `list[Message]` — never raw dicts | Prompts |

### Tool vs Resource Decision

| Question | Tool | Resource |
|---|---|---|
| Does it modify state? | ✅ | ❌ |
| Is it idempotent read? | Possible | ✅ Preferred |
| Does it need parameters beyond URI? | ✅ | ❌ |
| Should LLM trigger it actively? | ✅ | ❌ |

---

## 3. Context

Injected via type hint `ctx: Context`. Request-scoped. Auto-excluded from tool schema — LLM never sees it.

| Capability | Method | When to Use |
|---|---|---|
| Logging | `ctx.info()` / `debug()` / `warning()` / `error()` | Always — structured logs to client |
| Progress | `ctx.report_progress(current, total)` | Operations > 2 seconds |
| Resource access | `ctx.read_resource(uri)` | Tool needs data from another resource |
| LLM sampling | `ctx.sample()` | Tool needs LLM completion mid-execution |
| User input | `ctx.elicit()` | Tool needs structured input from user |
| Session state | `ctx.set_state()` / `ctx.get_state()` | State that persists across requests in same session |

**Rules:**
- Never use Context for business logic parameters — it's infrastructure only
- Context is request-scoped — do NOT store references outside the handler
- If you need server-scoped state → use Lifespan, not Context

---

## 4. Lifespan

Manages shared resources (DB pools, HTTP clients, caches) across server lifetime.

| Pattern | When |
|---|---|
| Use lifespan | Resource is expensive to create AND shared across tools |
| Don't use lifespan | Resource is cheap or request-specific |

**Access pattern**: `ctx.request_context.lifespan_context["key"]`

**Rule**: Lifespan yields a dict. Tools access it via Context. Never use globals.

---

## 5. Error Handling

| Error Type | Mechanism | Exposed to LLM? |
|---|---|---|
| Business logic failure | Raise `ToolError("message")` | ✅ Yes — structured, safe |
| Unexpected crash | Unhandled exception | ❌ No — logged server-side only |
| Validation failure | Raise `ToolError` with clear message | ✅ Yes |

**Rules:**
- Never leak stack traces, secrets, or internal paths in `ToolError`
- Never catch-all exceptions silently — let them crash and log
- Validate inputs explicitly inside the tool — type hints alone are not enough

---

## 6. Server Composition

| Pattern | Method | Use Case |
|---|---|---|
| Local sub-server | `main.mount(sub, prefix="x")` | Modular architecture, same process |
| Remote server | `main.mount(create_proxy(url), prefix="x")` | Federated / microservice MCP |

**Rules:**
- Prefix acts as namespace — avoids tool name collisions
- Parent middleware applies to ALL mounted sub-servers
- Sub-server-specific middleware: add before mounting
- Mount is live — changes in sub-server reflect immediately

---

## 7. Middleware

Intercepts all requests. Use for cross-cutting concerns only.

| Use Case | Implementation |
|---|---|
| Audit logging | Log tool name + result in `on_call_tool` |
| Error normalization | Catch exceptions, return safe `ToolError` |
| Auth enforcement | Reject unauthorized requests before tool execution |
| Rate limiting | Track call frequency, reject excess |

**Rules:**
- Add error-handling middleware FIRST — catches everything after it
- Middleware must call `call_next(context)` to continue the chain
- Never put business logic in middleware — that belongs in tools

---

## 8. Transports

| Transport | When | Config |
|---|---|---|
| **stdio** | Local CLI, IDE integration (Claude Desktop, Cursor) | `mcp.run()` — default |
| **Streamable HTTP** | Remote / production / multi-client | `mcp.run(transport="streamable-http")` |
| **SSE** | Legacy compatibility only — avoid for new servers | `mcp.run(transport="sse")` |

**Production rule**: Always use Streamable HTTP for anything beyond local single-user.

---

## 9. Sync vs Async

| Function Type | FastMCP Behavior | Use When |
|---|---|---|
| `async def` | Runs on event loop directly | I/O-bound: HTTP, DB, file, network |
| `def` | Auto-dispatched to threadpool | CPU-bound or simple sync operations |

**Rule**: Default to `async def`. Use sync only for truly simple operations with no I/O.

---

## 10. Deployment

### Run Modes

| Mode | Method | Workers |
|---|---|---|
| Dev (local) | `mcp.run()` | Single |
| Dev (HTTP) | `mcp.run(transport="streamable-http")` | Single |
| Production | `app = mcp.http_app()` → uvicorn | Multi-worker |

### Production Stack

```
Client → Reverse Proxy (TLS) → uvicorn (ASGI) → FastMCP app
```

### Docker Rules

- Base: `python:3.12-slim`
- Run as non-root: `USER 1000`
- Secrets via environment variables only — never baked in image
- Expose health endpoint at `/health` (unauthenticated) for orchestrator probes

### Client Config (stdio)

```json
{
  "mcpServers": {
    "server-name": {
      "command": "python",
      "args": ["server.py"]
    }
  }
}
```

### Client Config (remote)

```json
{
  "mcpServers": {
    "server-name": {
      "url": "http://host:8000/mcp",
      "transport": "streamable-http"
    }
  }
}
```

---

## 11. Security

| Concern | Rule |
|---|---|
| TLS | Terminate at reverse proxy — never plain HTTP in production |
| Authentication | Bearer/JWT at gateway or middleware — never skip |
| Authorization | Per-tool access control — not every client gets every tool |
| Input validation | Type hints + explicit checks inside tool — never trust LLM input |
| Error exposure | `ToolError` with safe messages only — never leak internals |
| Rate limiting | Infrastructure level (Nginx / API Gateway) |
| Secrets | Environment variables — never hardcoded, never in tool responses |
| CORS | Restrict to trusted origins if browser-accessible |

---

## 12. Testing

| Method | Purpose |
|---|---|
| `fastmcp dev server.py` | MCP Inspector — interactive web UI for manual testing |
| `Client(mcp)` | Programmatic testing — call tools, assert results |

**Rule**: Every tool must be testable via `Client(mcp)` without external dependencies (mock I/O in tests).

---

## Quick Reference

| Decision | Rule |
|---|---|
| Tool vs Resource? | Side effects → Tool. Read-only → Resource |
| Sync vs Async? | I/O → async. CPU/simple → sync (auto-threadpooled) |
| Error handling? | Business failure → `ToolError`. Crash → unhandled exception to logs |
| Shared state? | Server-scoped → Lifespan. Session-scoped → `ctx.set_state()` |
| Composition? | Same process → `mount()`. Remote → `create_proxy()` |
| Transport? | Local/CLI → stdio. Remote/prod → Streamable HTTP |
| Production? | `http_app()` + uvicorn + Docker + reverse proxy |
| Middleware? | Cross-cutting only. Error middleware first. Never business logic |
