# Floating Tab-Aware AI Chat + MCP Context Layer — Architecture Report

> Status: Proposal / Design Document
> Scope: Existing chat agent → floating chat in every browser tab, tab-context awareness, and an MCP context-fetching layer (web search + backend DB)

---

## Table of Contents

- [1. Executive Summary](#1-executive-summary)
- [2. How the Existing Chat Agent Works (Verified from Code)](#2-how-the-existing-chat-agent-works-verified-from-code)
- [3. Requirement Mapping vs. Current State (Gap Analysis)](#3-requirement-mapping-vs-current-state-gap-analysis)
- [4. Target Architecture Overview](#4-target-architecture-overview)
- [5. MCP Explained in This Context (What We Add vs. What Exists)](#5-mcp-explained-in-this-context-what-we-add-vs-what-exists)
- [6. Component 1: Floating Chat in Every Tab (Browser Extension)](#6-component-1-floating-chat-in-every-tab-browser-extension)
- [7. Component 2: Tab Context Awareness](#7-component-2-tab-context-awareness)
- [8. Component 3: MCP Server Layer for Context Fetching](#8-component-3-mcp-server-layer-for-context-fetching)
- [9. Component 4: Web Search + Source Comparison](#9-component-4-web-search--source-comparison)
- [10. API Contract Changes](#10-api-contract-changes)
- [11. Security & Tenancy](#11-security--tenancy)
- [12. Suggested Implementation Phases](#12-suggested-implementation-phases)
- [13. Risks & Mitigations](#13-risks--mitigations)
- [14. Key File References](#14-key-file-references)

---

## 1. Executive Summary

Digitomics InfraOps already has a **production chat agent** (FastAPI + LangGraph agents + SSE streaming + RAG + PostgreSQL persistence + a floating chat bubble inside the dashboard). What it does **not** have today:

| Need | Current state |
|---|---|
| Chat agent in every browser tab (not just the dashboard) | Only in-app `FloatingChatBubble` |
| Tab context awareness (knows what the user is viewing) | None (VS Code extension does this only for the IDE) |
| MCP **server** exposing app context (DB, RAG, tools) to any MCP host | Backend is only an MCP **client** (consumes AWS/Azure/GCP MCP servers) |
| Web search + cross-source comparison (web vs RAG vs DB) | No web search tool exists anywhere |
| Context window strategy | Streaming with in-memory windowing happens per-agent; needs explicit FIFO truncation |

This report proposes:

1. A **browser extension** (Chrome/Edge MV3) that mounts the existing chat UI in a floating panel on **every tab**, authenticated via a hand-off token from the dashboard session.
2. A **tab-context resource** (`tab://active`) pushed to the agent, mirroring the `editor://context` pattern already used by the opencode VS Code extension — the agent sees URL, title, and selected text of the current tab.
3. A **new MCP server layer** (`mcp_server.py`) that exposes Digitomics internals — PostgreSQL query tools, RAG retrieval, the 41 integration tools, and web search — as MCP tools/resources over Streamable HTTP and stdio.
4. A **web search tool + answer-comparison workflow** (web results vs. RAG knowledge base vs. live DB facts) with cited, rankable sources.

---

## 2. How the Existing Chat Agent Works (Verified from Code)

### 2.1 End-to-end request flow

```
Browser tab / dashboard
  │  fetch POST /api/v1/chat/stream  (SSE, Bearer JWT cookie)
  ▼
backend/routes/agent_routes.py            HTTP entry, permission gates
  ▼
backend/services/agent_chat_service.py    AgentChatService
  ├─ run_orchestration()        (non-stream, line 494)
  ├─ stream_orchestration()     (SSE, line 638)
  ├─ context assembly (session history, attachments, tenant)
  ├─ intent classification (3-tier: sticky session → keyword → LLM)
  ├─ credential resolution (IntegrationToken table)
  ├─ RAG retrieval (pgvector, MMR + RRF + cosine gate)
  ├─ _execute_agent_by_intent() (line 1636)
  └─ _save_final_response()     (line 1424)
  ▼
backend/workflows/intent_router.py + intent_classifier.py   intent cascade
  ▼
backend/agents/                                            LangGraph agents
  ├─ GenericAgent  ─ tools: RAG, file, web? (no web today)
  ├─ FinOpsAgent   ─ AWSToolsWrapper, GCP/Azure cost, FOCUS tables
  ├─ DevOpsAgent   ─ AWS/ Azure/ GCP MCP clients (stdio subprocess)
  └─ shared tools  ─ rag_search_knowledge_base, hitl, etc.
  ▼
SSE events:  heartbeat | token | status | reset | done | error
  ▼
frontend/src/services/chat-service.ts      fetch + ReadableStream parser
frontend/src/components/chat/chat-container.tsx  render
```

### 2.2 Verified key facts

| Area | File | What it does |
|---|---|---|
| HTTP entry | `backend/routes/agent_routes.py` | 12 endpoints incl. `POST /chat/stream`, `GET /chat/sessions`, `GET /chat/sessions/{id}/messages`; SSE via `StreamingResponse` + `_timeout_stream` / `_disconnect_aware_stream` |
| Orchestration | `backend/services/agent_chat_service.py` | `run_orchestration` :494, `stream_orchestration` :638, `_execute_agent_by_intent` :1636, `_save_final_response` :1424 |
| Intent | `backend/workflows/intent_router.py` :77, `intent_classifier.py` :361 | sticky session → keyword → LLM cascade |
| Agents | `backend/agents/` | LangGraph `StateGraph` + `ToolNode`; Generic / FinOps / DevOps / GCP / Azure |
| Persistence | `backend/app/db/models.py` | `chat_sessions`, `chat_messages`, `rag_document_chunks` (pgvector 1536-dim, `text-embedding-3-small`), `IntegrationToken`, FOCUS tables |
| RAG | `backend/agents/tools/rag_tools.py` `RagToolWrapper` :32 | MMR + RRF + cosine gate retrieval, hallucination-check answering |
| Streaming client | `frontend/src/services/chat-service.ts` :303 | fetch + ReadableStream, `\n\n` split, `Last-Event-ID` replay, 5 retries exponential backoff |
| Floating chat (in-app) | `frontend/src/components/chat/floating-chat-bubble.tsx` :23 | FAB `fixed bottom-6 right-6 z-40`, resizeable, mounted in `dashboard-shell.tsx` :357/373, same `useChatStore` as full page |
| Chat state | `frontend/src/stores/chat-store.ts` :157 | per-session messages, agents, abort controller |
| MCP client layer | `backend/agents/tools/Devops_{aws,azure,gcp}_tools/mcp/` | consumes AWS/Azure/GCP MCP servers via `mcp` Python package stdio |
| "MCP" facade (misnamed) | `backend/integrations/mcp/server.py` | **not** a protocol server — internal tool-schema registry + `handle_mcp_call` → `gateway.execute_with_fallback` |
| Integration tools | `backend/integrations/schemas/*.json` | 41 tool schemas (gmail 5, slack 7, github 8, googledocs 6, googledrive 7, googlesheets 4, teams 4) |
| DB plumbing | `backend/app/db/database.py` | SQLAlchemy async engine, RLS `set_tenant_context`, `tenant_scoped_session` |
| Redis | `backend/services/redis_client.py` | singleton async client, `int_cache:*`, `slack_channel:*`, `int_ctx:*`, revocation keys |

### 2.3 LLM inference (matches your "Serverless LLM APIs" row)

- Per-agent model names in `backend/core/config.py` (`finops_agent_model_name` = `gpt-4o`, `intent_classifier_model_name` = `gpt-4o-mini`, etc.)
- Providers: `core/container.py` wires `OpenAILLMProvider` / `AnthropicLLMProvider`; `openrouter_base_url` exists. **DeepSeek / Groq are OpenAI-compatible** → swap by pointing the OpenAI provider at `https://api.deepseek.com` or `https://api.groq.com/openai/v1` with respective key (env-driven). No code change needed if the provider honors `base_url`.

### 2.4 Your requirements table vs. existing code

| Your requirement | Exists today? | Where |
|---|---|---|
| Inference: serverless LLM APIs (OpenAI/DeepSeek/Groq) | ✅ OpenAI/Anthropic/OpenRouter wired; DeepSeek/Groq = config change only | `core/config.py`, `core/container.py` |
| In-memory sliding window (FIFO pair truncation) | ⚠️ Partial — history passed per-turn; explicit truncation policy not centralized | `agent_chat_service.py` context assembly |
| Full history in SQL/NoSQL DB (audit/analytics) | ✅ PostgreSQL `chat_sessions`/`chat_messages` | `app/db/models.py`, `ChatRepository` |
| MCP for context fetching | ❌ Only MCP *client* today; no MCP *server* | — |
| SSE / WebSockets low latency | ✅ SSE (`/chat/stream`); WS not used for chat | `agent_routes.py` |

---

## 3. Requirement Mapping vs. Current State (Gap Analysis)

| Requirement | Current | Gap | Needed |
|---|---|---|---|
| Floating chat per browser tab | In-app bubble only | No browser-extension surface | Extension (MV3) + hand-off auth |
| Tab context awareness | None | Agent blind to current tab | `tab://` context resource push |
| MCP for context fetching | Client-only | No server exposing our DB/RAG/tools | MCP server (stdio + Streamable HTTP) |
| Web search comparison | No web search | Web vs RAG vs DB compare | WebSearch tool + compare workflow |
| Sliding-window context | Ad-hoc | No central truncation | FIFO pair-truncation module |

---

## 4. Target Architecture Overview

```
┌────────────────────────────── Browser (every tab) ──────────────────────────────┐
│  Extension: content script injects floating panel (React, same as dashboard)    │
│  ├─ TabContextProvider  ──► URL/title/selection ──► background SW               │
│  ├─ FloatingChatPanel   ──► existing ChatContainer (surface="floating")          │
│  └─ Auth hand-off: reads dashboard cookie / token from app                        │
└──────────────────────────────┬───────────────────────────────────────────────────┘
                               │ HTTPS + Bearer (extension -> backend directly)
                               ▼
┌────────────────────────── Digitomics Backend ──────────────────────────────────┐
│  POST /api/v1/chat/stream (existing, + tab context field in request)           │
│                                                                                │
│  ┌────────────┐   ┌──────────────────┐   ┌───────────────────────────────┐     │
│  │AgentChat   │──►│ Intent classifier│──►│ Generic/FinOps/DevOps agents  │     │
│  │Service     │   └──────────────────┘   │  ┌─────────────────────────┐  │     │
│  └────────────┘                          │  │ LangGraph tool loop      │  │     │
│                                          │  └─────────────────────────┘  │     │
│  ┌─────────────── MCP Server (NEW: mcp_server.py) ────────────────┐      │     │
│  │  tools: db_query, rag_search, integration_* (41), web_search   │◄─────┘     │
│  │  resources: tab://active, chat://session/{id}, rag://kb         │            │
│  │  transport: Streamable HTTP (remote) + stdio (local CLI)        │            │
│  └─────────────────────────────────────────────────────────────────┘           │
│                                                                                │
│  ┌────────────── Web Search (NEW: web_search.py) ───────────────────┐          │
│  │  provider (Tavily/Serper/DuckDuckGo) → results → source compare  │          │
│  └──────────────────────────────────────────────────────────────────┘          │
└────────────────────────────────────────────────────────────────────────────────┘
                               │
          ┌────────────────────┼─────────────────────┐
          ▼                    ▼                     ▼
   PostgreSQL (history,   Redis (state,           External MCP servers
   RAG vectors, FOCUS,    cache, revocations)     (AWS/Azure/GCP — existing
   IntegrationToken)                                mcp client usage)
```

---

## 5. MCP Explained in This Context (What We Add vs. What Exists)

**MCP (Model Context Protocol)** = a standard JSON-RPC 2.0 contract so any AI host (opencode CLI, Claude Desktop, our agents) can fetch tools/resources from any server.

- **Today**: our backend is an **MCP client** — `agents/tools/Devops_aws_tools/mcp/aws_mcp_client.py`, `azure_mcp_client.py`, `gcp_mcp_client.py` spawn MCP servers via `uvx`/`npx` over stdio and call their tools. There is **zero MCP server code** (no `FastMCP`/`mcp.server` import). `backend/integrations/mcp/server.py` is only an internal schema registry + gateway facade (its name is misleading).
- **Proposal**: add an MCP **server** in `backend/integrations/mcp_server/` that exposes:

| MCP server (name) | Tools / Resources | Backed by (existing) |
|---|---|---|
| `digitomics_db` | `db_query(sql, params)` (read-only, tenant-scoped, SELECT-gated), `chat_history(session_id)`, `cost_summary()`, `focus_tables()` | SQLAlchemy engine + RLS `set_tenant_context` (app/db/database.py) |
| `digitomics_rag` | `rag_search_knowledge_base(query)` | `RagToolWrapper` + pgvector |
| `digitomics_integrations` | 41 tools from `integrations/schemas/*.json` | `gateway.execute_with_fallback` + `RedisIntegrationCache` |
| `digitomics_web` | `web_search(query)`, `fetch_url(url)` | new provider (Tavily/Serper) |
| `digitomics_tab` | resource `tab://active` (url, title, selection) | pushed from extension → Redis → served as resource |

Transports:
- **stdio** → so `opencode` / local CLI hosts can `npx digitomics-mcp` and get DB/RAG/web tools.
- **Streamable HTTP** → so the browser extension and remote hosts can authenticate with Bearer + OAuth and stream.

This is the "Modular Interface: MCP for context fetching" requirement from your table — the agent (and external hosts) fetch context from DB/RAG/web/tab through one standard interface instead of bespoke endpoints.

---

## 6. Component 1: Floating Chat in Every Tab (Browser Extension)

### 6.1 Why a browser extension (not an iframe)

`frontend/next.config.ts` sets `X-Frame-Options: SAMEORIGIN` and CSP `frame-ancestors 'self'` — third-party iframe embedding is blocked today. A MV3 extension with a **content script** avoids this entirely.

### 6.2 Proposed structure (new `extension/` directory at repo root)

```
extension/
├─ manifest.json                 # MV3, permissions: tabs, activeTab, scripting, storage
├─ background/
│  └─ service-worker.ts          # tab context publisher, token broker, SSE keep-alive
├─ content/
│  ├─ inject.ts                  # mounts floating panel into page
│  └─ tab-context.ts             # observes URL/title/selection → posts to background
├─ panel/                        # React app: reuse existing chat components
│  └─ FloatingChatPanel.tsx      # wraps frontend/src/components/chat/chat-container.tsx
└─ auth/
   └─ token-handoff.ts           # reads dashboard cookie / sessionStorage token
```

### 6.3 Auth hand-off (key design decision)

The dashboard stores an access token in a cookie (`frontend/src/lib/auth-cookie.ts`) and refreshes via `POST /api/v1/auth/refresh`. Options:

1. **Cookie read by content script** (if cookie is same-origin readable — currently `HttpOnly`? verify). If not readable, use option 2.
2. **Extension popup asks user to sign in once** — extension calls `/api/v1/auth/login` (or refresh) with credentials and stores a scoped token in `chrome.storage.local`, then reuses the backend refresh flow.
3. **Token hand-off via a dedicated endpoint** `POST /api/v1/auth/extension-broker` returning a short-lived, device-scoped token (issued to a registered extension client_id), keeping the dashboard session untouched.

**Recommendation:** (3) — new endpoint issues a short-TTL token bound to extension instance id; extension refreshes it with the same `/auth/refresh` flow. Reuses existing JWT + Redis revocation (`services/token_service.py`).

### 6.4 Panel reuse

`ChatContainer({ surface: 'floating' })` + `useChatStore` + `useChatSession` are already shared between the full chat page and the in-app floating bubble — the extension panel can import the same components (bundled with Vite) with zero backend changes for basic chat.

---

## 7. Component 2: Tab Context Awareness

### 7.1 Pattern (proven in this ecosystem)

The opencode VS Code extension runs a small MCP server exposing `editor://context` (active file + selection), the CLI discovers it via a lock file + Bearer auth, and the model sees "current selection" automatically. We mirror that: **`tab://active`**.

### 7.2 Data model

```json
{
  "url": "https://console.aws.amazon.com/ec2",
  "title": "EC2 — AWS Console",
  "selection": "instance i-0abcd1234 is running",
  "tab_id": 7,
  "domain": "console.aws.amazon.com",
  "pushed_at": "2026-08-14T10:00:00Z"
}
```

### 7.3 Flow

```
content script (every tab)
   │ observe: tabs.onUpdated / onActivated / selectionchange (debounced ~150ms)
   ▼
background service worker
   │ POST /api/v1/chat/tab-context   (or MCP resource update)
   ▼
backend tab_context service
   │ SETEX redis:tabctx:{user_id}:{tab_id} 60s  +  notify agents
   ▼
agent chat turn
   │ context assembly reads active tab context → prepends "User is viewing: ..."
   ▼
LLM answers with awareness of the page
```

- Per-user, per-tab isolation; Redis key `tabctx:{user_id}:{tab_id}` TTL 60s (re-pushed on change).
- Privacy: `tab_context_enabled` tenant flag; selection is optional (`alt-click to send selection`), default sends URL+title only.
- Also exposed as MCP resource `tab://active` so external MCP hosts can read it.

---

## 8. Component 3: MCP Server Layer for Context Fetching

### 8.1 New module layout

```
backend/integrations/mcp_server/
├─ server.py             # FastMCP app; registers tools/resources; Streamable HTTP + stdio entrypoints
├─ tools/
│  ├─ db_tools.py        # read-only SELECT executor with RLS + allowlist + row/time limits
│  ├─ rag_tools.py       # thin wrapper over RagToolWrapper
│  ├─ integration_tools.py  # wraps 41 schemas via gateway.execute_with_fallback
│  └─ web_tools.py       # web_search / fetch_url
├─ resources/
│  ├─ tab_context.py     # tab://active
│  └─ chat_history.py    # chat://session/{id}
├─ auth.py               # Bearer/JWT verification → user/tenant identity
└─ runner.py             # uvicorn entry for HTTP; console script for stdio
```

### 8.2 Tool registration (example)

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("digitomics")

@mcp.tool()
async def db_query(sql: str, limit: int = 50) -> str:
    """Read-only, tenant-scoped SQL query (SELECT only)."""
    # enforce: single statement, starts with SELECT/WITH, allowlist tables,
    # tenant via RLS set_tenant_context, max rows 200, timeout 10s
    ...

@mcp.tool()
async def web_search(query: str, sources: int = 5) -> str:
    """Search the web and return ranked results with snippets + URLs."""
    ...
```

### 8.3 Critical safety rules for `db_query`

- Single statement only (`sqlparse`), must start with `SELECT` or `WITH`.
- Tenant enforced through RLS (`set_tenant_context`) — never bypass.
- Row cap (e.g. 200), statement timeout, table allowlist derived from existing model set.
- Full audit log row per query (who, tenant, SQL, latency) — matches "data persistence for audit/analytics".

### 8.4 Serving transports

- `uvicorn mcp_server.runner:app` → Streamable HTTP at `/mcp` (Bearer auth).
- stdio entrypoint for local hosts: `python -m integrations.mcp_server.server --stdio` (can be registered in opencode config as a local MCP: `"command": ["uv", "run", "python", "-m", "integrations.mcp_server.server", "--stdio"]`).

---

## 9. Component 4: Web Search + Source Comparison

### 9.1 Web search provider (new)

None exists in the repo today. Add `backend/services/web_search.py`:

- Provider: **Tavily** (recommended — LLM-optimized results, cheap) or **Serper** (Google SERP) or free **DuckDuckGo** (`ddgs` package) for dev.
- `web_search(query, max_results, country)` → `[{title, url, snippet, score}]`.
- `fetch_url(url)` → readable markdown text (readability extraction) for the agent to cite directly.
- Redis cache `web_cache:{sha256(query)}` TTL 1h to cut cost.

### 9.2 Cross-source comparison workflow

New intent-level flow in `AgentChatService` for queries that need verification (e.g. "is this AWS cost anomaly real?" or "best price for X"):

```
user query
   │
   ├── RAG  ──► kb_chunks + scores          (internal knowledge)
   ├── DB   ──► live facts (FOCUS/cost data) (ground truth for numbers)
   └── Web  ──► web results + snippets       (external context)
   │
   ▼
source_compare step (LLM prompt)
   → verdict per source: agree / disagree / unknown
   → final answer with citations [Web: url] [KB: doc] [DB: table]
   → if web disagrees with DB → surface discrepancy to user
```

Deliverable shape in `done` SSE metadata: `sources: [{kind: web|kb|db, title, url, score}]` — frontend renders citations + a "Compared across N sources" badge.

### 9.3 Example end-to-end

> User (on AWS console tab): "Is our EC2 spend normal this month?"
> 1. Tab context → agent knows user is looking at EC2.
> 2. DB tool → FOCUS/cost tables: EC2 spend, MoM delta.
> 3. RAG → internal KB doc about last month's EC2 review.
> 4. Web → AWS pricing/announcement articles.
> 5. Compare → "Spend is +12% vs last month (DB). KB says we expected +10–15% due to migration. No conflicting web signals. Main driver: instance i-0abcd1234 (36%)."

---

## 10. API Contract Changes

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/auth/extension-broker` | POST | issue short-TTL extension token (device-bound) |
| `/api/v1/chat/tab-context` | POST/PUT | push current tab context (Redis-backed) |
| `/api/v1/chat/stream` (extended) | POST | request body gains optional `tab_context` + `context_sources` |
| `/mcp` | POST (Streamable HTTP) | MCP server (DB/RAG/integrations/web tools) |
| `/api/v1/web/search` | POST | web search proxy (permission-gated) |
| `/api/v1/web/search/cache` | DELETE | admin purge of web cache |

SSE `done` event `metadata` gains: `sources: [...], comparison: {...}` — frontend `chat-message.tsx` renders citation chips; `chat-service.ts` `parseSseEvent` needs no protocol change (extend payload only).

---

## 11. Security & Tenancy

| Concern | Control |
|---|---|
| Extension auth | Short-TTL device-scoped tokens; reuse Redis revocation (`token_service.py`) |
| Tab context privacy | Per-user Redis keys, tenant flag `tab_context_enabled`, URL-only by default |
| MCP db_query | RLS tenancy, SELECT-only, allowlist, row/time caps, full audit logging |
| MCP remote | Bearer auth (JWT) + optional OAuth 2.1 (MCP spec); same user identity for tool execution |
| Web search | No credentials forwarded; result cache is tenant-scoped; `fetch_url` blocks internal hosts (SSRF guard) |
| XSS/embedding | Extension keeps CSP of host page; panel is isolated `<shadow-root>` |

---

## 12. Suggested Implementation Phases

| Phase | Scope | Est. effort |
|---|---|---|
| **P0 — Web search + compare** | `web_search.py`, Tavily/DDG provider, web tools wired into GenericAgent, source metadata in SSE `done` | 2–3 days |
| **P0 — Sliding window** | `context_window.py`: FIFO pair-truncation (drop oldest user/assistant pairs when over token budget), unit-tested, wired into `agent_chat_service.py` context assembly | 1 day |
| **P1 — MCP server (stdio first)** | `mcp_server/` with db_query + rag + integrations tools; register locally in opencode config; smoke-test from CLI | 3–4 days |
| **P1 — MCP HTTP transport** | Streamable HTTP at `/mcp` + Bearer auth + audit logging | 2 days |
| **P2 — Extension MVP** | MV3 scaffold, floating panel reusing `ChatContainer`, token hand-off endpoint, SSE chat from any tab | 4–5 days |
| **P2 — Tab context** | content-script observer, background publisher, Redis store, `tab://active` MCP resource, context injection into chat | 2–3 days |
| **P3 — Comparison UX** | citation chips, source comparison cards in `chat-message.tsx`, settings for tab-context privacy | 2 days |

**Order rationale**: P0 delivers immediate agent value (web + windowing). P1 unlocks "MCP for context fetching" from any host. P2 delivers the headline feature (floating per-tab chat) reusing everything above.

---

## 13. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Token in extension storage stolen | Short TTL + device binding + rotation; revoke via existing Redis revocation |
| `db_query` SQL injection | Strict parser, allowlist, RLS, audit — treat as read-only reporting surface |
| Web search cost | Redis cache 1h, provider rate limits, per-tenant quota |
| Extension vs CSP of third-party sites | Shadow DOM + minimal permissions; graceful fail (no panel on `chrome://`, PDFs) |
| Context window blowup with tab context + web | FIFO pair truncation (P0) caps tokens before injection |
| MCP tool spam (41 integration tools) | Enable per-agent via existing `tools` config globs; lazy load schemas |

---

## 14. Key File References

Existing (do not rewrite):
- `backend/routes/agent_routes.py` — chat endpoints + SSE
- `backend/services/agent_chat_service.py` — orchestration (`run_orchestration` :494, `stream_orchestration` :638)
- `backend/workflows/intent_router.py`, `intent_classifier.py` — intent cascade
- `backend/agents/` — LangGraph agents + tool bindings
- `backend/integrations/mcp/server.py` — existing tool registry facade (reuse `handle_mcp_call`)
- `backend/integrations/gateway.py` — `execute_with_fallback` :302 (reuse for integration tools)
- `backend/integrations/schemas/*.json` — 41 tool schemas (reuse for MCP registration)
- `backend/app/db/database.py` — `set_tenant_context` :77, `tenant_scoped_session` :86
- `backend/services/redis_client.py`, `services/token_service.py` — Redis + revocation
- `backend/agents/tools/rag_tools.py` — RAG wrapper (reuse for MCP rag tool)
- `frontend/src/components/chat/{chat-container,floating-chat-bubble}.tsx` — reusable chat UI
- `frontend/src/stores/chat-store.ts`, `frontend/src/services/chat-service.ts` — state + SSE client
- `frontend/src/lib/auth-cookie.ts`, `frontend/src/lib/api-client.ts` — auth/token flow
- `frontend/next.config.ts` — proxy rewrites (note iframe-blocking headers)

New (proposed):
- `backend/integrations/mcp_server/` — MCP server layer
- `backend/services/web_search.py` — web provider + cache
- `backend/services/context_window.py` — FIFO pair truncation
- `backend/services/tab_context.py` — tab context store
- `backend/routes/extension_routes.py` — broker + tab-context endpoints
- `extension/` — MV3 browser extension (manifest, service worker, content script, React panel)
