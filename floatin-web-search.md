# Beyond Limits Search — Implementation Mindmap

> Going beyond internal data: Web search + Cross-source verification + Real-time external intelligence

---

## The Problem

```
┌─────────────────────────────────────────────────────────────────┐
│  CURRENT LIMITATIONS                                            │
│                                                                 │
│  ❌ AI only searches internal knowledge base (RAG)             │
│  ❌ No web search — can't answer external questions            │
│  ❌ No cross-source verification — single source of truth      │
│  ❌ Blind to market conditions, pricing, competitor moves      │
│  ❌ Can't answer: "Is our cost normal?" (no benchmark data)    │
│  ❌ Can't answer: "What's the best practice for X?" (no web)   │
│  ❌ Can't answer: "How do competitors handle Y?" (no intel)    │
└─────────────────────────────────────────────────────────────────┘
```

---

## The Solution: Beyond Limits Search

```
┌─────────────────────────────────────────────────────────────────┐
│  BEYOND LIMITS = Internal + External + Comparison              │
│                                                                 │
│  ✅ Web Search (Tavily/DuckDuckGo) — external intelligence     │
│  ✅ Source Comparison — DB vs RAG vs Web — verify answers       │
│  ✅ Real-time Market Data — pricing, benchmarks, trends        │
│  ✅ Cross-Source Citations — [DB] [KB] [Web] with confidence   │
│  ✅ Discrepancy Detection — flag when sources disagree         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Mindmap: Implementation Architecture

```
                    ┌──────────────────────────┐
                    │  BEYOND LIMITS SEARCH     │
                    │  (Context-Aware AI)       │
                    └────────────┬─────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  1. WEB SEARCH  │    │  2. SOURCE      │    │  3. CONTEXT     │
│  LAYER          │    │  COMPARISON     │    │  INTELLIGENCE   │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ • Tavily API    │    │ • DB result     │    │ • Tab context   │
│ • DuckDuckGo    │    │ • RAG result    │    │ • User history  │
│ • Fetch URL     │    │ • Web result    │    │ • Tenant data   │
│ • Redis cache   │    │ • LLM verdict   │    │ • Intent flow   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## Component 1: Web Search Layer

### What It Does
Enables the AI to search the internet for real-time external information.

### Implementation

```
┌─────────────────────────────────────────────────────────────────┐
│  WEB SEARCH SERVICE (backend/services/web_search.py)           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Provider Chain (fallback)                              │   │
│  │                                                         │   │
│  │  Tavily (prod) ──► DuckDuckGo (fallback) ──► Cache     │   │
│  │                                                         │   │
│  │  • LLM-optimized results (Tavily)                      │   │
│  │  • Free tier available (DuckDuckGo)                    │   │
│  │  • Redis cache (1 hour TTL)                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  API                                                    │   │
│  │                                                         │   │
│  │  search(query, max_results=5) → [WebResult]             │   │
│  │  fetch_url(url) → markdown text                         │   │
│  │                                                         │   │
│  │  WebResult = {title, url, snippet, score}               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Safety                                                 │   │
│  │                                                         │   │
│  │  • SSRF guard (block internal hosts)                    │   │
│  │  • No credentials forwarded                             │   │
│  │  • Per-tenant rate limits                               │   │
│  │  • Result cache to cut cost                             │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### File Structure
```
backend/services/
└─ web_search.py              # Provider chain + cache

backend/agents/tools/
└─ web_search_tools.py        # LLM-bindable tool wrapper
```

### Example Queries Enabled

| Query | Before (Internal Only) | After (With Web Search) |
|-------|----------------------|------------------------|
| "Is our EC2 cost normal?" | ❌ Can't answer (no benchmark) | ✅ Compares with AWS pricing page + industry data |
| "What's the best practice for RDS optimization?" | ❌ Can't answer (not in KB) | ✅ Fetches AWS docs + blog posts |
| "How do competitors handle FinOps?" | ❌ Can't answer | ✅ Fetches case studies + benchmarks |
| "What's the current AWS pricing for m5.xlarge?" | ❌ Can't answer | ✅ Fetches live pricing page |

---

## Component 2: Source Comparison Engine

### What It Does
Cross-references answers from multiple sources (DB, RAG, Web) and provides a verified answer with citations.

### Implementation

```
┌─────────────────────────────────────────────────────────────────┐
│  SOURCE COMPARISON (backend/services/source_comparison.py)     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Input: Query + Multi-Source Results                    │   │
│  │                                                         │   │
│  │  {                                                     │   │
│  │    "query": "Is our EC2 spend normal?",                │   │
│  │    "db_result": {...},      # FOCUS tables              │   │
│  │    "rag_result": {...},     # Knowledge base            │   │
│  │    "web_result": {...}      # AWS pricing articles      │   │
│  │  }                                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  LLM Comparison Step                                   │   │
│  │                                                         │   │
│  │  "Compare these 3 sources. Do they agree?              │   │
│  │   Identify: agree / disagree / unknown per source.     │   │
│  │   Flag discrepancies."                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Output: Verified Answer + Citations                   │   │
│  │                                                         │   │
│  │  {                                                     │   │
│  │    "answer": "EC2 spend is +12% vs benchmark...",      │   │
│  │    "sources": [                                         │   │
│  │      {"kind": "db", "title": "FOCUS table", "score": 1.0}, │
│  │      {"kind": "kb", "title": "EC2 Review Doc", "score": 0.9},│
│  │      {"kind": "web", "title": "AWS Pricing", "url": "...", "score": 0.8} │
│  │    ],                                                  │   │
│  │    "comparison": {                                      │   │
│  │      "verdict": "agree",                               │   │
│  │      "confidence": 0.95,                               │   │
│  │      "discrepancies": []                               │   │
│  │    }                                                   │   │
│  │  }                                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### File Structure
```
backend/services/
└─ source_comparison.py       # Multi-source verification

backend/agents/tools/
└─ comparison_tools.py        # LLM-bindable comparison tool
```

### Example: Cross-Source Verification

```
Query: "Is our Azure spend normal for this workload?"

┌─────────────────────────────────────────────────────────────────┐
│  SOURCE 1: DATABASE (FOCUS tables)                              │
│  ─────────────────────────────────                              │
│  Result: Azure spend = $45,000 this month                       │
│  Confidence: 1.0 (ground truth)                                 │
├─────────────────────────────────────────────────────────────────┤
│  SOURCE 2: KNOWLEDGE BASE (Internal docs)                       │
│  ────────────────────────────────────────                       │
│  Result: "Azure spend typically $40-50K for this workload"      │
│  Confidence: 0.9 (historical pattern)                           │
├─────────────────────────────────────────────────────────────────┤
│  SOURCE 3: WEB (Azure pricing page + benchmarks)                │
│  ───────────────────────────────────────────────                │
│  Result: "Industry avg for this workload: $42-48K"             │
│  Confidence: 0.8 (external benchmark)                           │
├─────────────────────────────────────────────────────────────────┤
│  COMPARISON VERDICT: AGREE                                      │
│  ─────────────────────────────                                  │
│  All 3 sources agree: $45K is within normal range               │
│  Confidence: 0.95                                               │
│  Discrepancies: None                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component 3: Context Intelligence

### What It Does
Injects tab context (what user is viewing) + user history + tenant data into the query for smarter responses.

### Implementation

```
┌─────────────────────────────────────────────────────────────────┐
│  CONTEXT INTELLIGENCE                                          │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Tab Context (from browser extension)                  │   │
│  │                                                         │   │
│  │  {                                                     │   │
│  │    "url": "https://console.aws.amazon.com/ec2",        │   │
│  │    "title": "EC2 — AWS Console",                        │   │
│  │    "selection": "instance i-0abcd1234"                  │   │
│  │  }                                                     │   │
│  │                                                         │   │
│  │  → Agent knows user is looking at EC2 instance         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  User History (from PostgreSQL)                         │   │
│  │                                                         │   │
│  │  • Recent queries (last 7 messages)                     │   │
│  │  • Previous intents (FinOps vs DevOps)                  │   │
│  │  • Session context (multi-turn)                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Tenant Data (from PostgreSQL + Redis)                  │   │
│  │                                                         │   │
│  │  • Cloud provider connections (AWS/GCP/Azure)           │   │
│  │  • Integration tokens (Gmail, Slack, GitHub)            │   │
│  │  • RAG documents (knowledge base)                       │   │
│  │  • FOCUS cost data                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Complete Flow: Beyond Limits Search

```
User Query: "Is our EC2 spend normal? How do competitors handle this?"
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. CONTEXT ASSEMBLY                                            │
│  ──────────────────                                             │
│  • Tab context: EC2 instance page (from extension)             │
│  • User history: Previous FinOps queries                       │
│  • Tenant data: AWS credentials, FOCUS tables                  │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. INTENT CLASSIFICATION                                       │
│  ──────────────────────                                         │
│  Keywords: "EC2" + "spend" → FINOPS                            │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. MULTI-SOURCE RETRIEVAL (parallel)                          │
│  ─────────────────────────────────                              │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ DB Query     │  │ RAG Search   │  │ Web Search   │         │
│  │              │  │              │  │              │         │
│  │ FOCUS tables │  │ Internal KB  │  │ Tavily API   │         │
│  │ EC2 costs    │  │ Cost docs    │  │ AWS pricing  │         │
│  │ MoM delta    │  │ Best pract.  │  │ Benchmarks   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. SOURCE COMPARISON                                           │
│  ────────────────────                                           │
│  • DB: "+12% vs last month"                                    │
│  • KB: "Expected +10-15% due to migration"                     │
│  • Web: "Industry avg: +8-12% for similar workloads"           │
│  • Verdict: AGREE (all sources consistent)                     │
│  • Confidence: 0.92                                            │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. RESPONSE SYNTHESIS                                          │
│  ────────────────────                                           │
│                                                                 │
│  "EC2 spend increased 12% this month (DB). This is expected    │
│   due to the microservices migration (KB: +10-15% projected).   │
│   Industry benchmark shows +8-12% for similar migrations       │
│   (Web: AWS Architecture Blog).                                │
│                                                                 │
│   For competitor practices, I found:                            │
│   • Company A uses reserved instances for 40% savings          │
│   • Company B uses spot instances for batch workloads          │
│   • Company C rightsizes monthly based on utilization          │
│                                                                 │
│   Sources: [DB: FOCUS] [KB: Migration Plan] [Web: AWS Blog]   │
│   [Web: FinOps Case Studies]"                                  │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. SSE STREAMING                                               │
│  ────────────────                                               │
│  event: token  → "EC2 spend increased 12%..."                  │
│  event: token  → "Industry benchmark shows..."                 │
│  event: done   → {sources: [...], comparison: {...}}           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation Phases

### Phase 1: Web Search Foundation (3 days)

```
┌─────────────────────────────────────────────────────────────────┐
│  Files to Create                                                │
│  ───────────────                                                │
│  backend/services/web_search.py                                 │
│  backend/agents/tools/web_search_tools.py                       │
│                                                                 │
│  Tasks                                                          │
│  ─────                                                          │
│  □ Implement Tavily provider                                    │
│  □ Implement DuckDuckGo fallback                                │
│  □ Add Redis cache (1h TTL)                                     │
│  □ Create LLM-bindable tool wrapper                             │
│  □ Wire to GenericAgent                                         │
│  □ SSRF guard for fetch_url                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Phase 2: Source Comparison (2 days)

```
┌─────────────────────────────────────────────────────────────────┐
│  Files to Create                                                │
│  ───────────────                                                │
│  backend/services/source_comparison.py                          │
│  backend/agents/tools/comparison_tools.py                       │
│                                                                 │
│  Tasks                                                          │
│  ─────                                                          │
│  □ Implement multi-source retrieval orchestrator                │
│  □ LLM comparison prompt (agree/disagree/unknown)               │
│  □ Citation format: [DB] [KB] [Web]                             │
│  □ Discrepancy detection + flagging                             │
│  □ Wire to agent flow                                           │
└─────────────────────────────────────────────────────────────────┘
```

### Phase 3: Context Intelligence (3 days)

```
┌─────────────────────────────────────────────────────────────────┐
│  Files to Create                                                │
│  ───────────────                                                │
│  backend/services/tab_context.py                                │
│  backend/routes/extension_routes.py                             │
│                                                                 │
│  Tasks                                                          │
│  ─────                                                          │
│  □ Redis tab context store (60s TTL)                            │
│  □ Tab context API endpoint                                     │
│  □ Context injection into agent flow                            │
│  □ Privacy controls (opt-in, tenant flag)                       │
└─────────────────────────────────────────────────────────────────┘
```

### Phase 4: Extension MVP (5 days)

```
┌─────────────────────────────────────────────────────────────────┐
│  Files to Create                                                │
│  ───────────────                                                │
│  extension/manifest.json                                        │
│  extension/background/service-worker.ts                         │
│  extension/content/inject.ts                                    │
│  extension/panel/FloatingChatPanel.tsx                          │
│                                                                 │
│  Tasks                                                          │
│  ─────                                                          │
│  □ MV3 scaffold                                                 │
│  □ Token hand-off endpoint                                      │
│  □ Content script + shadow DOM panel                            │
│  □ Tab context observer                                         │
│  □ SSE streaming from extension                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Query Examples: Before vs After

### Example 1: Cost Analysis

| Query | Before | After |
|-------|--------|-------|
| "Is our AWS spend normal?" | ❌ "I don't have external benchmark data" | ✅ Compares DB + KB + Web benchmarks, gives verdict with confidence |

### Example 2: Best Practices

| Query | Before | After |
|-------|--------|-------|
| "What's the best practice for RDS optimization?" | ❌ "I can only search our knowledge base" | ✅ Fetches AWS docs + blog posts + case studies |

### Example 3: Competitor Intelligence

| Query | Before | After |
|-------|--------|-------|
| "How do competitors handle FinOps?" | ❌ "I don't have that information" | ✅ Fetches case studies + benchmarks + industry reports |

### Example 4: Pricing Check

| Query | Before | After |
|-------|--------|-------|
| "What's the current EC2 m5.xlarge price?" | ❌ "I can only query our billing data" | ✅ Fetches live AWS pricing page |

### Example 5: Discrepancy Detection

| Query | Before | After |
|-------|--------|-------|
| "Is this cost data accurate?" | ❌ No way to verify | ✅ Cross-checks DB vs KB vs Web, flags discrepancies |

---

## Business Value

```
┌─────────────────────────────────────────────────────────────────┐
│  BEFORE BEYOND LIMITS                                          │
│  ─────────────────────                                          │
│  • AI answers only from internal data                          │
│  • User must manually verify with web searches                 │
│  • No cross-source confidence                                  │
│  • Limited to what's in the knowledge base                     │
└─────────────────────────────────────────────────────────────────┘

                              │
                              ▼

┌─────────────────────────────────────────────────────────────────┐
│  AFTER BEYOND LIMITS                                           │
│  ────────────────────                                           │
│  • AI searches internal + external simultaneously              │
│  • Cross-references DB, KB, Web for verified answers           │
│  • Provides confidence scores + citations                      │
│  • Flags discrepancies between sources                         │
│  • Answers questions about market, competitors, best practices │
└─────────────────────────────────────────────────────────────────┘
```

---

## Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Query coverage | 90%+ answerable | % of queries that get a verified answer |
| Source diversity | 2.5+ sources/query | Avg sources consulted per query |
| Verification confidence | >0.85 | Avg comparison confidence score |
| Discrepancy detection | 100% | % of real discrepancies flagged |
| User satisfaction | >4.5/5 | Survey rating |
| Response time | <30s | End-to-end latency |

---

## Key Files Summary

### New Files

```
backend/
├─ services/
│  ├─ web_search.py              # Web search provider + cache
│  └─ source_comparison.py       # Multi-source verification
├─ agents/tools/
│  ├─ web_search_tools.py        # LLM-bindable web tool
│  └─ comparison_tools.py        # LLM-bindable comparison tool
├─ integrations/mcp_server/
│  └─ tools/web_tools.py         # MCP web tools
└─ routes/
   └─ extension_routes.py        # Tab context + auth

extension/
├─ manifest.json
├─ background/service-worker.ts
├─ content/inject.ts
└─ panel/FloatingChatPanel.tsx
```

### Modified Files

```
backend/
├─ services/agent_chat_service.py   # + multi-source retrieval
├─ agents/generic_agent.py          # + web_search tool
└─ agents/finops_agent.py           # + comparison tool

frontend/
└─ src/components/chat/
   └─ chat-message.tsx              # + citation chips
```
