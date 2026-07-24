# Focus Explorer — Enterprise Architecture, Gap Analysis & Implementation Roadmap

> **Version**: 1.0  
> **Based on**: Codebase at Alembic head `0023`, FOCUS spec v1.4, competitive analysis  
> **Scope**: The complete end-to-end architecture, current implementation, gaps, and phased roadmap for building an enterprise-grade Focus Explorer

---

## 1. Executive Summary

Digitomics InfraOps has a **solid FOCUS 1.2 foundation** that is more advanced than most FinOps startups at this stage. The current Focus Explorer (`frontend/src/components/finops/cost-explorer.tsx`) provides 7-dimension filtering, group-by aggregation, time-series charts, paginated line items, CSV export, and saved views — all powered by a normalized `cost_charges` table.

However, the gap between "solid foundation" and "enterprise FinOps platform" is substantial. The current implementation:

- **Works for single-currency, single-provider scenarios** but breaks down with multi-currency, multi-provider, and enterprise-scale data
- **Has no real ingestion for AI, SaaS, data platforms, or on-prem** — only cloud providers + a thin SaaS adapter
- **Has no invoice reconciliation, commitment tracking, forecasting, or anomaly detection** powered by the FOCUS layer
- **Has 37 of 107 FOCUS 1.4 columns** — missing 70 columns that enable invoice reconciliation, commitment coverage, allocation method tracking, and pricing analysis
- **Has no asynchronous query layer** — every aggregation runs synchronously over the entire dataset
- **Has no partitioned or materialized view strategy** — performance will degrade badly above ~10M rows per tenant

This document provides the complete analysis and a phased roadmap to close these gaps.

---

## 2. Current Codebase Architecture

### 2.1 Top-Level Layout

```
digitomics-infraops/
├── frontend/              # Next.js 16, React 19, TypeScript
│   └── src/
│       ├── app/           # App Router pages
│       ├── components/    # React components
│       ├── services/      # API client wrappers
│       ├── stores/        # Zustand stores
│       ├── config/        # API endpoint definitions
│       └── types/         # TypeScript types
├── backend/               # FastAPI + LangGraph + SQLAlchemy
│   ├── app/db/            # SQLAlchemy models
│   ├── alembic/           # Migrations (23 revisions, linear)
│   ├── routes/            # FastAPI route handlers
│   ├── services/          # Business logic
│   │   ├── focus/         # FOCUS normalization, ingest, query
│   │   ├── allocation/    # Virtual tag engine
│   │   ├── cost_intelligence/ # Forecasting, anomalies
│   │   └── optimization/  # Recommendation engine
│   ├── workers/           # Background workers
│   ├── Tools/             # Provider connectors
│   │   ├── connections/   # Connection services (per-provider)
│   │   └── integrations/  # Integration framework
│   └── agents/            # LangGraph agent definitions
├── docs/                  # Documentation
│   ├── focus_Explorer.md  # Current Focus Explorer doc
│   ├── FOCUS.md           # FOCUS spec analysis doc
│   └── finops-strategy/   # Product strategy & epics
└── packages/              # Shared workspace packages
```

### 2.2 Focus Explorer Component Architecture

```
Frontend (cost-explorer.tsx)
  ├── Filter sidebar (7 dimensions + date range + saved views)
  │   └── FilterCombobox (autocomplete from tenant data)
  ├── Group-by selector (7 dimensions)
  ├── Cost field toggle (effective_cost / billed_cost)
  ├── Granularity selector (day / week / month)
  ├── Area chart (recharts)
  ├── Breakdown table (sortable, clickable for drill-down)
  ├── Line items table (paginated, sortable)
  └── CSV export + Ingest button

API Layer (focus_routes.py)
  ├── GET /summary          → summarize_focus()
  ├── GET /timeseries       → timeseries_focus()
  ├── GET /charges          → query_cost_charges()
  ├── GET /dimension-values → get_focus_dimension_values()
  ├── GET /export           → CSV streaming
  ├── GET|PUT /export-config → get/set_export_config()
  └── POST /ingest          → run_focus_ingestion_for_tenant()

Backend Services (services/focus/)
  ├── schema.py             → normalize_charge_row(), map_service_category()
  ├── charge_store.py       → CRUD, aggregation, filtering
  └── ingest.py             → Provider adapters (AWS, GCP, Azure, SaaS)

Database
  └── cost_charges table    → FOCUS 1.2 normalized rows (~37 columns)

Allocation Engine (services/allocation/)
  └── virtual_tag_engine.py → Rule matching, allocation, reallocation
```

---

## 3. Current Focus Explorer Implementation — Feature Audit

### 3.1 Feature Reality Table

| Feature | Frontend | API | Backend | Database | Working? | Limitations |
| ------- | -------- | --- | ------- | -------- | -------- | ----------- |
| **Summary/Group-by** | `cost-explorer.tsx:336` summary query | `/api/v1/finops/focus/summary` | `summarize_focus()` in `charge_store.py:336` | `cost_charges` | ✅ Yes | Top-50 + Other; multi-currency aware but single dimension only |
| **Time-series chart** | `cost-explorer.tsx:360` area chart from timeseries query | `/api/v1/finops/focus/timeseries` | `timeseries_focus()` in `charge_store.py:479` | `cost_charges` | ✅ Yes | Single series only; no stacked/grouped view; mixed-currency flagged but shown blended |
| **Line items table** | `cost-explorer.tsx:255` charges query | `/api/v1/finops/focus/charges` | `query_cost_charges()` in `charge_store.py:211` | `cost_charges` | ✅ Yes | Paginated; sortable by 10 columns; max 1000 per page |
| **7-dimension filtering** | `cost-explorer.tsx:190` filter object | All `/focus/*` endpoints | `_apply_filters()` in `charge_store.py:152` | `cost_charges` | ✅ Yes | Overlap semantics on date range; exact-match on dimensions |
| **Dimension autocomplete** | `FilterCombobox` in 7 filter queries | `/api/v1/finops/focus/dimension-values` | `get_focus_dimension_values()` in `charge_store.py:307` | `cost_charges` | ✅ Yes | Top-100 by cost; 5-min staleTime cache |
| **CSV export** | `cost-explorer.tsx:369` | `/api/v1/finops/focus/export` | Server-side CSV streaming | `cost_charges` | ✅ Yes | Server-side; client-side fallback |
| **Saved views** | `cost-explorer.tsx:79` localStorage | None (client-only) | None | None | ✅ Yes | Browser localStorage only; max 20 views; not synced to server |
| **Drill-down** | `cost-explorer.tsx:746` click row → set drill dimension | Uses existing `/charges` with dimension filter | `query_cost_charges()` | `cost_charges` | ✅ Yes | Single dimension drill; no hierarchical drill path |
| **Ingest trigger** | `cost-explorer.tsx:335` ingest mutation | `POST /api/v1/finops/focus/ingest` | `run_focus_ingestion_for_tenant()` in `ingest.py:719` | Multi-provider | ✅ Yes | Admin-only; best-effort multi-connection; SaaS falls through on error |
| **Multi-account AWS** | N/A | `_connections_with_credentials()` in `focus_routes.py:321` | `ingest_aws_cost_explorer_lines()` | `cost_charges` | ✅ Yes | Previously capped at 1; now iterates all active connections |
| **Auto-trigger after sync** | N/A | N/A | `billing_sync_service.py:1112` | `cost_charges` | ✅ Yes | SaaS + cloud when credentials available; AWS only in sync hook |
| **Focus sync worker** | N/A | N/A | `workers/focus_sync_worker.py` | `cost_charges` | ✅ Yes | Single-tenant; AWS-only (uses old API); limited at 1 connection |
| **Virtual tags** | Via `virtual-tags` page | `/api/v1/finops/virtual-tags/*` | `allocation_service.py` + `virtual_tag_engine.py` | `virtual_tags`, `cost_charges` | ✅ Yes | 12 FOCUS dimensions + tag.* keys; 5 operators; first-match-wins |
| **Showback/Chargeback** | Via `allocation` page | `/api/v1/finops/showback`, `/api/v1/finops/chargeback` | `build_showback()` + `build_chargeback()` | `cost_charges`, `virtual_tags` | ✅ Yes | Virtual tag mode + weighted-rules + provider fallback |
| **Unit economics** | Via `unit-economics` page | `/api/v1/finops/unit-economics` | `build_unit_economics()` in `allocation_service.py:540` | `business_metrics`, `cost_charges` | ✅ Partial | Needs manual metric entry; no telemetry API; single period comparison |
| **Cost Intelligence** | Separate pages | `/api/v1/cost-intelligence/*` | `CostIntelligenceService` in `cost_intelligence/` | Separate tables | ✅ Partial | Connection-scoped; not FOCUS-native |
| **Optimization** | Via `optimization` page | `/api/v1/finops/optimization/*` | `recommendation_engine.py` | `optimization_recommendations` | ✅ Yes | Advisory only; HITL via PendingAction |
| **Anomaly routing** | Via settings | `/api/v1/finops/anomaly-routing/*` | `anomaly_routing_service.py` | `anomaly_routing_events` | ✅ Yes | Slack/Jira routing; dedup; no FOCUS-native detection |
| **GCP BigQuery ingestion** | N/A | Via `/ingest` | `ingest_gcp_bigquery_export()` in `ingest.py:270` | `cost_charges` | ✅ Yes | Requires export config; 5000 row limit; usage_start_time filtering |
| **Azure FOCUS CSV ingestion** | N/A | Via `/ingest` | `ingest_azure_focus_export()` in `ingest.py:503` | `cost_charges` | ✅ Yes | Requires storage config; 5000 row limit; FOCUS 1.2 native |
| **AWS CUR CSV ingestion** | N/A | Via `/ingest` | `_try_read_aws_cur_csv()` in `ingest.py:190` | `cost_charges` | ✅ Partial | First .csv under S3 prefix; 5000 rows; best-effort |
| **SaaS BillingRecord ingestion** | N/A | Via `/ingest` | `ingest_saas_from_billing_records()` in `ingest.py:671` | `cost_charges` via `billing_records` | ✅ Yes | 500 record limit; excludes AWS/GCP/Azure |
| **Export config management** | N/A | `/api/v1/finops/focus/export-config/{id}` | `get/set_export_config()` in `charge_store.py:34-71` | `CloudConnection.provider_metadata` | ✅ Yes | S3/BQ/Azure storage per connection |
| **Enterprise gating** | `EnterpriseGate` in `frontend/src/components/enterprise/` | `require_feature()` / `require_permission()` | `feature_flag_catalog.py` + `enterprise_entitlements.py` | Feature flags + sales locks | ✅ Yes | FOCUS_EXPLORER flag; costs.analysis permission; sales-lockable surface |

### 3.2 What Actually Works End-to-End

1. **User connects AWS/GCP/Azure** → credentials stored in `cloud_connections` → `POST /ingest` triggers all adapters → data flows through `normalize_charge_row()` → upserted into `cost_charges` → queryable via `/summary`, `/timeseries`, `/charges`
2. **User opens Focus Explorer** → 7 filter comboboxes populate from `/dimension-values` → `/summary` returns Top-50 groups → `/timeseries` returns daily/weekly/monthly points → area chart renders → click a row to drill → `/charges` returns paginated line items
3. **User creates virtual tags** → rules stored in `virtual_tags` → preview via `preview_virtual_tag()` → allocate via `allocate_charges()` → showback/chargeback via `build_chargeback_statement()`
4. **User defines business metrics** → stored in `business_metrics` → unit economics computed via `build_unit_economics()` → cost-per-unit with trends

### 3.3 What Does NOT Actually Work

1. **AWS CUR CSV ingestion**: Only reads first `.csv` under S3 prefix — not the latest, not all files, no recursive directory support. 5000 row limit.
2. **Focus sync worker**: Uses old single-connection API; only fetches AWS; ignores GCP/Azure connections beyond the first.
3. **Auto-trigger after billing sync**: Only passes AWS credentials; GCP and Azure connections are silently not fetched.
4. **Multi-currency rendering**: The frontend shows blended totals in the time-series chart when currencies mix — only the KPI and breakdown table correctly show `total_by_currency`.
5. **AI/ML service mapping**: The `SERVICE_CATEGORY_MAP` has only 3 AI entries (openai, anthropic, gemini) — no Azure AI, AWS Bedrock models, or other AI providers.

---

## 4. Complete Data-Flow Analysis

### 4.1 Connection → Cost Charge Lifecycle

```
┌─────────────────────┐
│ 1. Connection Setup  │
│ Frontend: /dashboard │
│ /connections          │
│ API: connections_     │
│ routes.py             │
│ Model: CloudConnection│
│ Storage: encrypted    │
│ credentials in DB     │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│ 2. Sync / Ingest     │
│ 3 paths:              │
│ a) POST /ingest      │
│    (admin, manual)    │
│ b) billing sync tail │
│    (auto after sync)  │
│ c) focus_sync_worker  │
│    (scheduled)        │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│ 3. Provider Adapter   │
│ ingest.py:            │
│ - AWS CE API          │
│ - AWS CUR CSV (S3)    │
│ - GCP BigQuery Export  │
│ - GCP billing lines   │
│ - Azure FOCUS CSV     │
│ - Azure billing lines │
│ - SaaS BillingRecords  │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│ 4. Normalization      │
│ schema.py:            │
│ normalize_charge_row()│
│ - Maps provider fields│
│   → FOCUS 1.2         │
│ - Generates hash IDs  │
│ - Maps service→cat    │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│ 5. Persistence        │
│ charge_store.py:      │
│ persist_cost_charges()│
│ - Bulk upsert (250)   │
│ - ON CONFLICT UPDATE  │
│ - xmax insert/update   │
│ tracking              │
│ - Expire ORM identity  │
│ map                   │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│ 6. Query / Aggregate  │
│ charge_store.py:      │
│ summarize_focus()     │
│ timeseries_focus()    │
│ query_cost_charges()  │
│ get_dimension_values()│
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│ 7. API Response       │
│ focus_routes.py:      │
│ JSON responses        │
│ CSV streaming         │
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│ 8. Frontend Render    │
│ cost-explorer.tsx:    │
│ - Area chart          │
│ - Breakdown table     │
│ - Line items          │
│ - Filters             │
└─────────────────────┘
```

### 4.2 Key Observations

- **No caching layer**: Every `/summary` and `/timeseries` call does a full table scan. No Redis caching of aggregates.
- **No partitioning**: `cost_charges` has no time-based partitioning. Queries filter by `charge_period_start` but scan all partitions.
- **No async queries**: All queries run synchronously in the HTTP request. No job queue for large aggregations.
- **Single aggregation pattern**: Can only group by one dimension at a time. No cross-tabulation.
- **No pre-computation**: Every aggregation is computed live. No materialized views or rollup tables.

---

## 5. Current Connector Analysis

### 5.1 Provider Capability Matrix

| Provider | Auth | Billing | Usage | Resources | Tags | Discounts | Commitments | Invoice | FOCUS Native | Data Freshness | Row Limit |
| -------- | ---- | ------- | ----- | --------- | ---- | --------- | ----------- | ------- | ------------ | -------------- | --------- |
| **AWS (CE API)** | IAM keys | ✅ Unblended/Amortized | ❌ Service-level only | ❌ Not at row level | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | Current month | Unlimited (grouped) |
| **AWS (CUR CSV)** | IAM keys | ✅ Full detail | ✅ Line-item usage | ✅ ResourceId | ✅ resourceTags/user:* | ✅ LineItem-level | ✅ Via CUR columns | ✅ bill/InvoiceId | Partial | Depends on export | 5000 rows |
| **GCP (BigQuery)** | Service account key | ✅ Cost/Currency | ✅ Usage amount/unit | ✅ Resource labels | ✅ Labels as tags | ❌ Not mapped | ❌ Not mapped | ❌ Not mapped | ❌ No | Custom range | 5000 rows |
| **GCP (billing lines)** | Service account key | ✅ Cost summary | ❌ Service-level only | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | Current month | Unlimited |
| **Azure (FOCUS CSV)** | App registration | ✅ Billed/Effective/List/Contracted | ✅ Usage quantity/unit | ✅ ResourceId/Name/Type | ✅ Tags field | ❌ Not mapped | ❌ Not mapped | ❌ Not mapped | ✅ Yes (v1.2) | Depends on export | 5000 rows |
| **Azure (billing lines)** | App registration | ✅ Cost summary | ❌ Resource group only | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | Current month | Unlimited |
| **SaaS (BillingRecord)** | Per-tool credentials | ✅ Total cost | ✅ Usage quantity | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | Varies | 500 records |

### 5.2 Connector Inventory

Each connector service lives in `backend/Tools/connections/`:

| File | Provider | What It Does |
| ---- | -------- | ------------ |
| `cloud_connections_service.py` | AWS/GCP | Generic cloud connection credential resolution |
| `gcp_billing_service.py` | GCP | GCP billing snapshot fetching (cost_by_service, cost_by_project) |
| `gcp_connections_service.py` | GCP | GCP-specific connection management |
| `azure_billing_service.py` | Azure | Azure billing snapshot fetching (cost_by_service, cost_by_resource_group) |
| `azure_connections_service.py` | Azure | Azure connection credential resolution |
| `aws_dashboard_service.py` | AWS | AWS dashboard data (not FOCUS) |
| `azure_dashboard_service.py` | Azure | Azure dashboard data (not FOCUS) |
| `gcp_dashboard_service.py` | GCP | GCP dashboard data (not FOCUS) |
| `openai_integration_service.py` | OpenAI | OpenAI cost data (not mapped to FOCUS) |
| `gemini_billing_service.py` | Gemini | Gemini billing (not mapped to FOCUS) |
| `google_maps_billing_service.py` | Google Maps | G Maps billing (not mapped to FOCUS) |
| `databricks_integration_service.py` | Databricks | Databricks cost data (not mapped to FOCUS) |
| `godaddy_cost_service.py` | GoDaddy | Domain cost data (not mapped to FOCUS) |

### 5.3 SaaS Connectors (BillingRecord-based)

These connectors sync into `billing_records` but are NOT mapped to FOCUS `cost_charges` unless `ingest_saas_from_billing_records()` runs:

- OpenAI, Gemini, Google Maps, Google Workspace, Twilio, Fast2SMS, MongoDB, WHAPI, 360Dialog, Slack, Slackbot, Cursor, Namecheap, Trello, Jira, Zoom, Databricks, GoDaddy, Airtel

### 5.4 Critical Connector Gaps

1. **No incremental sync**: Every adapter fetches the full current month. No `last_synced_at` watermark tracking.
2. **No historical backfill strategy**: `_resolve_window()` defaults to current month. Backfill requires manual `period_start`/`period_end` parameters.
3. **No deduplication across adapters**: AWS CE API + CUR CSV are both ingested — same charges may appear from both paths.
4. **No OpenAI/Anthropic/Cohere AI API cost ingestion**: These are the fastest-growing cost categories with zero FOCUS coverage.
5. **No Databricks/Snowflake data platform cost ingestion**: Credits/DBUs not mapped to FOCUS.
6. **No on-prem/data center cost support**: No model or adapter for private infrastructure costs.

---

## 6. Current Database Model

### 6.1 Core Tables

#### `cost_charges` — FOCUS 1.2 normalized charge line items

- **Purpose**: Canonical FOCUS-cost storage for all providers
- **Row count estimate**: Variable per tenant; single query may scan millions
- **Primary key**: `id` (UUID)
- **Unique constraint**: `(tenant_id, provider_name, provider_line_item_id)`
- **Indexes**: 3 composite indexes (tenant_period_category, tenant_resource, tenant_service)
- **Missing indexes**: `(tenant_id, charge_period_start)` for time-range filtering; `(tenant_id, billing_currency)` for multi-currency queries; `(tenant_id, charge_category)` for charge type filtering

**Columns (~37 mapped):**

| Category | Columns | Source |
| -------- | ------- | ------ |
| Identity | `id`, `tenant_id`, `connection_id`, `provider_name`, `provider_line_item_id` | Generated + mapped |
| Publisher | `publisher_name`, `invoice_issuer_name` | Mapped |
| Cost | `billed_cost`, `effective_cost`, `list_cost`, `contracted_cost`, `billing_currency` | Mapped |
| USD | `x_billed_cost_usd`, `x_effective_cost_usd` | Converted |
| Time | `billing_period_start/end`, `charge_period_start/end` | Mapped |
| Service | `service_category`, `service_name`, `service_subcategory` | Mapped + derived |
| Account | `sub_account_id`, `sub_account_name` | Mapped |
| Location | `region_id`, `region_name` | Mapped |
| Resource | `resource_id`, `resource_name`, `resource_type` | Mapped |
| Charge | `charge_category`, `charge_class`, `charge_description` | Mapped |
| Pricing | `pricing_quantity`, `pricing_unit`, `consumed_quantity`, `consumed_unit` | Mapped |
| SKU | `sku_id`, `sku_price_id` | Mapped |
| Commitment | `commitment_discount_id/type/status/category` | Mapped |
| Tags | `tags` (JSONB) | Mapped |
| Extensions | `x_provider_extensions` (JSONB) | Mapped |
| Audit | `created_at`, `updated_at` | Auto |

#### `virtual_tags` — Rule-based allocation definitions

- **Purpose**: Finout-style virtual tag rules for cost allocation
- **Rows per tenant**: Typically 5-50
- **Unique constraint**: `(tenant_id, name)`

#### `shared_cost_reallocations` — Shared cost redistribution

- **Purpose**: Redistribute shared costs across allocation targets
- **Parent**: `virtual_tags`

#### `business_metrics` — User-defined unit economics metrics

- **Purpose**: Track quantity of business outputs (users, requests, GB)
- **Unique constraint**: `(tenant_id, metric_key, period_start, period_end)`

#### `cost_allocation_rules` — Weighted allocation rules (legacy)

- **Purpose**: Showback allocation rules predating virtual tags
- **Status**: Legacy; virtual tags are the preferred approach

#### `optimization_recommendations` — Cost optimization suggestions

- **Purpose**: Advisory recommendations with HITL approval

#### `anomaly_routing_events` — Anomaly alert deduplication

- **Purpose**: Track anomaly routing dispatches

### 6.2 Missing Tables (vs Target Architecture)

| Missing Table | Purpose | Priority |
| ------------- | ------- | -------- |
| `focus_invoices` | Invoice-level data for reconciliation | P0 |
| `focus_commitments` | Commitment/RIs/SPs/CUDs tracking | P0 |
| `focus_budget` | Budget vs actual | P1 |
| `focus_forecast` | ML-generated forecasts | P1 |
| `focus_data_quality` | FOCUS conformance scores | P1 |
| `focus_enrichments` | Business context (team, app, env, product) | P1 |
| `focus_allocation_results` | Pre-computed allocation outputs | P2 |

### 6.3 Scalability Concerns

1. **No partitioning**: `cost_charges` has no `PARTITION BY RANGE (charge_period_start)`. At 10M+ rows, queries degrade.
2. **No time-series compression**: The same data could be compressed for historical months.
3. **Composite indexes are limited**: Two of three indexes start with `tenant_id` which is always filtered — but no covering indexes for the most common query patterns.
4. **Sidecar tables missing**: No aggregate/materialized view tables for common group-by patterns.
5. **No data retention policy**: The model has no TTL or archival strategy.

---

## 7. Current API Analysis

### 7.1 Focus-Specific Endpoints

| Endpoint | Method | Purpose | Request Params | Response |
| -------- | ------ | ------- | -------------- | -------- |
| `/api/v1/finops/focus/summary` | GET | Group-by aggregation | group_by, cost_field, period_start/end, 7 filter dims | Groups with cost, count, share%, currency breakdown |
| `/api/v1/finops/focus/timeseries` | GET | Time-bucketed cost | granularity, cost_field, period_start/end, 7 filter dims | Points with period, cost, line_count |
| `/api/v1/finops/focus/charges` | GET | Paginated line items | limit, offset, sort_by/dir, 7 filter dims | Charges array, count, total_count, has_more |
| `/api/v1/finops/focus/dimension-values` | GET | Filter autocomplete | dimension | Values list ordered by total cost |
| `/api/v1/finops/focus/export` | GET | CSV download | Same as summary | Streaming CSV response |
| `/api/v1/finops/focus/export-config/{id}` | GET/PUT | Manage export config | Connection ID + config body | Export config object |
| `/api/v1/finops/focus/ingest` | POST | Trigger ingestion | period_start/end (optional) | Results array |

### 7.2 Allocation-Focused Endpoints

| Endpoint | Method | Purpose |
| -------- | ------ | ------- |
| `/api/v1/finops/virtual-tags` | GET/POST | List/create virtual tags |
| `/api/v1/finops/virtual-tags/{id}` | PATCH/DELETE | Update/delete virtual tag |
| `/api/v1/finops/virtual-tags/preview` | POST | Dry-run rules against charges |
| `/api/v1/finops/virtual-tags/dimensions` | GET | Available dimension fields |
| `/api/v1/finops/virtual-tags/suggest` | GET | Auto-suggest from tag data |
| `/api/v1/finops/shared-reallocations` | GET/POST | Manage shared cost rules |
| `/api/v1/finops/showback` | GET | Showback allocation |
| `/api/v1/finops/chargeback` | GET | Chargeback statements |
| `/api/v1/finops/unit-economics` | GET | Cost-per-unit calculations |
| `/api/v1/finops/business-metrics` | GET/POST | User-defined metrics |

### 7.3 API Gaps

1. **No multi-dimension group-by**: Can only group by one dimension at a time.
2. **No drill-path API**: No endpoint returns the hierarchical dimension tree (Provider → Account → Service → Resource).
3. **No saved views server-side**: All saved views are in localStorage — no sync, no sharing, no backup.
4. **No export of raw FOCUS line items**: Only group-by summary export exists.
5. **No async query support**: Large aggregations time out or block the connection pool.
6. **No budget/forecast integration**: Budgets and forecasts are not queryable through the focus API.
7. **No comparison/period-over-period endpoint**: Comparisons are computed client-side (or not at all).

---

## 8. Current Frontend Analysis

### 8.1 Component Inventory

| Component | File | Lines | Purpose |
| --------- | ---- | ----- | ------- |
| `CostExplorer` | `frontend/src/components/finops/cost-explorer.tsx` | 918 | Main Focus Explorer component |
| `FilterCombobox` | `frontend/src/components/finops/filter-combobox.tsx` | 154 | Autocomplete filter dropdowns |
| Focus page | `frontend/src/app/dashboard/focus/page.tsx` | 62 | Page wrapper with EnterpriseGate + URL params |
| `EnterpriseGate` | `frontend/src/components/enterprise/enterprise-gate.tsx` | 100 | Feature gate with sales lock |
| API client | `frontend/src/services/finops-allocation-service.ts` | 601 | All FinOps API calls + TypeScript types |

### 8.2 State Management

- **React Query** (TanStack Query): All API data fetching and caching
- **Zustand** (`useAuthStore`): User/auth state for role/permission checks
- **React state** (`useState`): Filter values, drill state, pagination, UI state
- **localStorage**: Saved views (no server persistence)

### 8.3 Data Flow Per Interaction

```
User selects filter → useState updates → filters memo changes
  → React Query refetches /summary and /timeseries
    → Area chart updates
    → Breakdown table updates
  → FilterCombobox queries /dimension-values (5-min cache)

User clicks group-by → /summary with new group_by
  → Breakdown table re-renders

User clicks a row → drillDimension set → /charges with dimension filter
  → Line items table shows filtered rows

User clicks Export → /export CSV downloaded
User clicks Ingest → POST /ingest → refresh caches
```

### 8.4 Frontend Gaps

1. **No stacked/grouped chart**: The area chart shows a single series — no multi-provider stacking, no group-by visualization.
2. **No comparison mode**: Can't compare "this period vs last period" or "budget vs actual".
3. **No hierarchical navigation**: No breadcrumb or path showing filter state (Provider → Service → Resource).
4. **No Sankey/allocation flow visualization**: Can't visualize how costs flow from providers to teams.
5. **No async job status**: Ingest is fire-and-forget; no progress indicator or completion notification.
6. **No server-side saved views**: Views are lost on browser clearing.
7. **No custom date presets**: Only free-form date inputs; no "Last 7 days", "This month", "Last quarter" buttons.
8. **No keyboard navigation**: Filter comboboxes require mouse for option selection.
9. **No mobile responsiveness**: The filter sidebar (220px) + chart + table layout doesn't adapt well to narrow viewports.
10. **No accessibility annotations**: Missing aria labels, roles, and keyboard support on interactive elements.

---

## 9. Current Limitations and Bugs

### 9.1 Confirmed Bugs

1. **Focus sync worker uses old single-connection API** (`workers/focus_sync_worker.py:28-74`): Fetches only the first AWS connection; does not use `_connections_with_credentials()` or pass GCP/Azure connections.

2. **Billing sync auto-trigger only passes AWS** (`billing_sync_service.py:1122-1144`): GCP and Azure connections are not resolved, so those providers are silently skipped after every billing sync.

3. **AWS CUR CSV reader only gets first file** (`ingest.py:201-203`): `list_objects_v2` returns keys sorted alphabetically; first `.csv` may not be the most recent export.

4. **GCP BigQuery fallback not triggered**: `run_focus_ingestion_for_tenant()` tries BigQuery then billing lines for GCP, but the billing lines adapter requires a pre-fetched `gcp_payload` dict — there's no fallback that auto-fetches billing lines when BigQuery isn't configured.

5. **Azure fallback identical gap**: Same pattern — `azure_payload` must be pre-fetched for `ingest_azure_billing_lines()` to work.

6. **Multi-day charges with same hash**: The hash-based line-item ID (`schema.py:130-143`) includes `period_start.isoformat()`. A daily-chargeable resource gets a new hash each day (correct), but monthly charges with the same content hash to the same ID every ingestion run (could be correct or could silently drop the second month's data if the first hasn't been cleaned).

### 9.2 Design Limitations

1. **Single dimension aggregation**: `summarize_focus()` groups by exactly one dimension + currency. No `GROUP BY GROUPING SETS` or multi-dimensional rollup.

2. **Top-N + Other overflow**: 50 groups maximum. At enterprise scale with thousands of resources, 50 is insufficient for resource-level analysis.

3. **Charge-level allocation**: `allocate_charges()` (`virtual_tag_engine.py:216`) loads ALL charges into memory for the tenant. At 500K+ charges, this is a memory OOM risk.

4. **No Redis caching for aggregates**: Every `/summary` and `/timeseries` call does a full SQL aggregation. No short-term TTL cache.

5. **Mixed currency in time-series**: `timeseries_focus()` blends currencies in the chart — the `mixed_currency` flag is set but the chart still shows a blended number.

6. **Pagination max 1000 rows**: `query_cost_charges()` hard-limits at 1000 rows (`min(max(limit, 1), 1000)`). Large tenants can't view all their line items.

7. **No full-text search on line items**: Filtering by resource name, charge description, or tag values requires exact matches.

---

## 10. FOCUS Specification Analysis

### 10.1 What FOCUS 1.4 Provides

The FOCUS 1.4 specification defines **4 datasets** with **107 columns**:

| Dataset | Purpose | Columns | Our Status |
| ------- | ------- | ------- | ---------- |
| Cost and Usage | Core charge line items | ~40 mandatory | ✅ 37 mapped |
| Invoice Detail | Invoice-level charges | ~11 | ❌ Missing |
| Contract Commitment | Commitment terms/lifecycle | ~27 | ❌ Missing |
| Billing Period | Billing period status | ~3 | ❌ Missing |

### 10.2 Column Gap Analysis: We Have 37/107

**We have (37 columns):**
```
BilledCost, EffectiveCost, ListCost, ContractedCost, BillingCurrency,
ChargePeriodStart, ChargePeriodEnd, BillingPeriodStart, BillingPeriodEnd,
ServiceCategory, ServiceName, ServiceSubcategory,
SubAccountId, SubAccountName,
RegionId, RegionName,
ResourceId, ResourceName, ResourceType,
ChargeCategory, ChargeClass, ChargeDescription,
PricingQuantity, PricingUnit,
ConsumedQuantity, ConsumedUnit,
SkuId, SkuPriceId,
CommitmentDiscountId, CommitmentDiscountType, CommitmentDiscountStatus,
CommitmentDiscountCategory, Tags, InvoiceIssuerName, PublisherName,
x_billed_cost_usd, x_effective_cost_usd
```

**We're missing (70 columns) — prioritized:**

**P0 — Required for foundation (22 columns):**
- `billing_account_id`, `billing_account_name`, `billing_account_type` — Account hierarchy
- `sub_account_type` — Sub-account classification
- `invoice_id`, `invoice_detail_id` — Invoice reconciliation
- `pricing_category`, `list_unit_price`, `contracted_unit_price`, `pricing_currency` — Pricing model
- `sku_meter`, `sku_price_details` — Metering granularity
- `availability_zone` — Location granularity
- `charge_frequency` — One-time vs recurring
- `service_provider_name`, `host_provider_name` — Marketplace/provider tracking
- `commitment_discount_name`, `commitment_discount_quantity`, `commitment_discount_unit` — Commitment details
- `commitment_program_eligibility` — Coverage analysis
- `contract_id`, `contract_applied` — Contract tracking

**P1 — Required for enterprise use cases (15+ columns):**
- Invoice dates/terms: `invoice_issue_date`, `payment_due_date`, `payment_terms`, `purchase_order_number`, `payment_currency`, `payment_currency_billed_cost`
- Pricing: `pricing_currency_list_unit_price`, `pricing_currency_effective_cost`, `pricing_currency_contracted_unit_price`
- Allocation: `allocated_method_id`, `allocated_method_details`, `allocated_resource_id`, `allocated_resource_name`, `allocated_tags`
- Billing period: `billing_period_created`, `billing_period_last_updated`, `billing_period_status`

**P2 — Contract Commitment dataset (27 columns):**
Full contract commitment tracking: `contract_commitment_id`, `contract_commitment_type`, `contract_commitment_start_date`, `contract_commitment_end_date`, `contract_commitment_cost`, `contract_commitment_quantity`, `contract_commitment_unit`, `contract_commitment_discount_percentage`, `contract_commitment_lifecycle_status`, `contract_commitment_payment_model`, etc.

### 10.3 What FOCUS Model We Should Adopt Directly

1. **Cost and Usage dataset**: Adopt all ~40 required columns as-is. Our existing model is close but needs 10+ P0 column additions.
2. **ChargeCategory/ChargeClass/ChargeFrequency**: Adopt the FOCUS enum values exactly.
3. **ProviderName/PublisherName/InvoiceIssuerName**: Already mapped; keep the FOCUS definition.
4. **CommitmentDiscount fields**: Already have the base set; need 3 more.
5. **Tags as `ResourceTags`**: Already implemented as JSONB.

### 10.4 What We Should Extend

1. **`x_provider_extensions`**: Our extension mechanism is correct — keep provider-native fields here.
2. **`x_billed_cost_usd` / `x_effective_cost_usd`**: Our USD conversion columns are a valid extension for single-currency reporting.
3. **`connection_id`**: Not a FOCUS column, but essential for multi-connection management.
4. **Virtual tag allocation columns**: We need `x_allocated_value`, `x_allocated_virtual_tag_id` as enrichment columns on `cost_charges`.

---

## 11. Competitor/Product Research

### 11.1 Competitive Landscape

| Product | Core Model | Focus Explorer | Strengths | Weaknesses |
| ------- | ---------- | -------------- | --------- | ---------- |
| **CloudZero** | Custom normalized schema with Unit Economics at core | Dimension-based explorer with unit costs | Unit economics, engineering context, multi-cloud | No HITL automation; limited commitment tracking |
| **Finout** | Virtual tags on any dimension | Many dimensions, pivot tables, saved views | Virtual tags (copied here), cost allocation, metric forest | No AI agents; limited budget support |
| **Vantage** | FOCUS-compatible, provider-agnostic | Clean explorer, segmentation, resources | Best UX, resource-level cost, FOCUS native | Limited allocation; no chargeback; fewer integrations |
| **Harness CCM** | Custom + Kubernetes | Explorer with perspectives, budgets | K8s cost, CI/CD integration, perspectives | Limited multi-cloud; weak SaaS support |
| **IBM Cloudability** | Market-settled schema | Legacy explorer, rightsizing | Mature rightsizing; commitment management | Legacy UX; slow innovation |
| **CAST AI** | Kubernetes-focused | Cluster/compute explorer | K8s cost optimization, auto-scaling | Cloud-only; no SaaS/AI; no FOCUS |
| **Amnic** | K8s + cloud cost | K8s context explorer | K8s workload-level cost; engineering attribution | Cloud-only; early stage |
| **ProsperOps** | Commitment API | No explorer (optimization only) | Commitment automation; RI/SP management | Single use-case; no full platform |

### 11.2 What We Should Copy

1. **CloudZero's unit economics depth**: Cost-per-unit as the primary metric, not just an add-on. Their business metric telemetry API is the gold standard.
2. **Finout's virtual tag implementation**: Our virtual tags are already similar — good. Their "Metric Forest" and nested allocation UI is worth studying.
3. **Vantage's UX**: Clean, fast, resource-grouped exploration. Their "no query language" approach.
4. **Vantage's FOCUS commitment**: They ship FOCUS-native data from AWS/GCP/Azure with saved filters and views.
5. **Harness's perspectives**: Named, persisted filter+group configurations that can be shared across teams.
6. **CAST AI's cluster-level cost views**: For Kubernetes cost allocation (Epic E01).

### 11.3 What We Should Improve

1. **Multi-provider normalization depth**: Go beyond FOCUS 1.2 to FOCUS 1.4 — most competitors are still on 1.0-1.2.
2. **Explainable AI agents**: No competitor has FOCUS-aware AI agents that can answer "why did my bill go up?" with attribution.
3. **HITL optimization**: CloudZero/Finout don't execute — we can close the loop with PendingAction.
4. **SaaS+AI cost coverage**: Most competitors ignore non-cloud spend. This is our wedge.

### 11.4 What We Should Deliberately Avoid

1. **Over-engineering the query language**: Don't build a SQL-like query DSL. Use the FOCUS dimensions as the exploration schema.
2. **Real-time ingestion**: No competitor provides true real-time billing. Stick with hourly/daily sync.
3. **Commitment purchasing**: ProsperOps and CAST AI buy RI/SPs on your behalf. We should advise, not execute (HITL).
4. **Thick client**: Don't move aggregation to the browser. Server-side query + thin client is correct.
5. **Generic BI integration**: Don't build a generic SQL export. Focus on FOCUS-native exports.

---

## 12. Target Product Definition

The enterprise Focus Explorer should enable every user to:

1. **Connect any technology provider** (cloud, SaaS, AI, data, on-prem) in under 5 minutes.
2. **Import billing/usage data** automatically on a schedule, with manual backfill available.
3. **Normalize all data** into a single FOCUS 1.4-compatible schema automatically.
4. **Explore costs** across all providers with flexible filtering, grouping, and drill-down.
5. **Filter by any FOCUS dimension** with autocomplete, multi-select, and saved filters.
6. **Group by any dimension** (or combination of dimensions) with sub-totals and rollups.
7. **View costs as** Billed, Effective, List, Amortized, Contracted, or Allocated.
8. **Compare periods**: Current vs previous, month-over-month, year-over-year.
9. **Drill from provider → account → service → resource → usage/charge**.
10. **See unit costs**: Cost per API call, GB, user, transaction, request, token, or workload.
11. **Apply tags and business context**: environment, team, application, product, cost center.
12. **Create allocation rules**: Virtual tags with shared cost redistribution.
13. **Allocate shared costs**: Even split, weighted, usage-based, or manual.
14. **Save views** server-side, share with team, set as default.
15. **Export data** as CSV, Excel, or programmatic JSON — scheduled or on-demand.
16. **Build reports** from saved views with chart configurations.
17. **Set budgets** and track actual vs budget.
18. **Forecast costs** using ML models trained on historical data.
19. **Detect anomalies** with explainable root cause analysis.
20. **Track commitments** (RI/SP/CUD) coverage and utilization.
21. **Reconcile invoices** against consumption.
22. **Use the same normalized data** for all FinOps features: allocation, showback, chargeback, budgeting, forecasting, anomaly detection, and optimization.

---

## 13. Target Enterprise Architecture

### 13.1 Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                        FRONTEND (Next.js 16)                         │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                   Focus Explorer v2                           │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │  │
│  │  │ Explorer │ │Invoices  │ │Commitment│ │AI Cost Dashboard │ │  │
│  │  │ - Group   │ │ - Recon  │ │ - Tacker │ │ - Tokens/Models  │ │  │
│  │  │ - Filter  │ │ - Pay    │ │ - Compare│ │ - Token Budget   │ │  │
│  │  │ - Chart   │ │ - Terms  │ │ - Waste  │ │ - Model Cost     │ │  │
│  │  │ - Lines   │ │          │ │          │ │                  │ │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘ │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │  │
│  │  │ Budgets  │ │ Forecasts│ │Anomalies │ │Unit Economics    │ │  │
│  │  │ - Actual │ │ - ML     │ │ - Detect │ │ - Per-User       │ │  │
│  │  │ - Alert  │ │ - Trend  │ │ - RCA    │ │ - Per-Request    │ │  │
│  │  │ - Report │ │ - Season │ │ - Route  │ │ - Per-GB         │ │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘ │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────────┐
│                       API LAYER (FastAPI)                            │
│  /api/v1/finops/focus/*    /invoices/*   /commitments/*              │
│  /budgets/*  /forecasts/*  /anomalies/*  /unit-economics/*           │
│  /export/*    /allocations/*   /reports/*                             │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
┌──────────────────────────┼───────────────────────────────────────────┐
│  ┌───────────────────────▼──────────────────────────────────────┐   │
│  │                    SERVICE LAYER                              │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐  │   │
│  │  │Query/Explore │ │Allocation    │ │Analytics             │  │   │
│  │  │- Multi-dim   │ │- VirtualTags │ │- Budget vs Actual    │  │   │
│  │  │- Filter      │ │- Shared Cost │ │- Forecast (ML)       │  │   │
│  │  │- Aggregation │ │- Showback    │ │- Anomaly Detection   │  │   │
│  │  │- Drilldown   │ │- Chargeback  │ │- Commitment Analysis │  │   │
│  │  └──────────────┘ └──────────────┘ └──────────────────────┘  │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐  │   │
│  │  │Normalization │ │Enrichment    │ │Unit Economics        │  │   │
│  │  │- FOCUS 1.4   │ │- Tags→Owners │ │- Cost per Unit       │  │   │
│  │  │- Schema Map  │ │- Teams/Apps  │ │- Business Metrics    │  │   │
│  │  │- Validation  │ │- Products    │ │- Segment Analysis    │  │   │
│  │  └──────────────┘ └──────────────┘ └──────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────────┐
│                    DATA LAYER (PostgreSQL + Redis)                   │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ cost_charges  │  │ Raw Provider │  │ enrichements  │               │
│  │ (FOCUS 1.4   │  │ Store        │  │ (team, app,   │               │
│  │  canonical)  │  │ (parquet or  │  │  product)     │               │
│  │              │  │  JSONB)      │  │              │               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ Aggregated   │  │ focus_       │  │ focus_       │               │
│  │ Rollups      │  │ invoices     │  │ commitments  │               │
│  │ (materialized│  │              │  │              │               │
│  │  views)      │  │              │  │              │               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
│                                                                      │
│  Redis Cache: Aggregates, Dimension Values, Session State            │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 14. Canonical Data Model

### 14.1 Entity Definitions

#### `Provider`
- **Purpose**: Normalized source provider identifier
- **FOCUS mapping**: `ProviderName`, `ServiceProviderName`, `HostProviderName`
- **Source**: Derived from connection
- **Type**: String enum (aws, gcp, azure, saas, ai, datacenter)

#### `InvoiceIssuer`
- **Purpose**: Entity that issued the invoice
- **FOCUS mapping**: `InvoiceIssuerName`
- **Source**: Provider export
- **Enables**: Multi-provider invoice reconciliation

#### `BillingAccount`
- **Purpose**: Top-level billing account (e.g., AWS Master Account, GCP Billing Account)
- **FOCUS mapping**: `BillingAccountId`, `BillingAccountName`, `BillingAccountType`
- **Source**: Provider export
- **Missing**: ⚠️ Not on `cost_charges` yet

#### `SubAccount`
- **Purpose**: Sub-account within billing account (e.g., AWS Linked Account, GCP Project)
- **FOCUS mapping**: `SubAccountId`, `SubAccountName`, `SubAccountType`
- **Source**: Provider export
- **Status**: ✅ `sub_account_id`, `sub_account_name` exist; `sub_account_type` missing

#### `Service`
- **Purpose**: Service/Resource type being charged
- **FOCUS mapping**: `ServiceCategory`, `ServiceName`, `ServiceSubcategory`
- **Source**: Provider export → FOCUS category mapping
- **Status**: ✅ All 3 exist

#### `Resource`
- **Purpose**: Individual resource instance
- **FOCUS mapping**: `ResourceId`, `ResourceName`, `ResourceType`
- **Source**: Provider export
- **Status**: ✅ All 3 exist

#### `Charge` (line item)
- **Purpose**: Single charge line item
- **FOCUS mapping**: 40+ columns
- **Source**: Normalized from provider export
- **Status**: ✅ Stored in `cost_charges`

#### `Cost`
- **Purpose**: Multiple cost perspectives for each charge
- **FOCUS mapping**: `BilledCost`, `EffectiveCost`, `ListCost`, `ContractedCost`
- **Status**: ✅ All 4 exist

#### `Usage`
- **Purpose**: Usage/consumption metrics for each charge
- **FOCUS mapping**: `ConsumedQuantity`, `ConsumedUnit`, `PricingQuantity`, `PricingUnit`
- **Status**: ✅ All 4 exist

#### `Currency`
- **Purpose**: Billing and pricing currency
- **FOCUS mapping**: `BillingCurrency`, `PricingCurrency`, `PaymentCurrency`
- **Status**: ✅ `billing_currency` exists; `pricing_currency` and `payment_currency` missing

#### `Tags`
- **Purpose**: Resource tags/metadata
- **FOCUS mapping**: `Tags` (JSON)
- **Status**: ✅ `tags` column (JSONB) exists

#### `Sku`
- **Purpose**: Stock-keeping unit for the charge
- **FOCUS mapping**: `SkuId`, `SkuMeter`, `SkuPriceDetails`, `SkuPriceId`
- **Status**: ✅ `sku_id`, `sku_price_id` exist; `sku_meter`, `sku_price_details` missing

#### `CommitmentDiscount`
- **Purpose**: Applied commitment discount info
- **FOCUS mapping**: `CommitmentDiscountId/Type/Status/Category/Name/Quantity/Unit`
- **Status**: ✅ 4 base fields exist; 3 more needed

#### `Invoice`
- **Purpose**: Invoice-level charge grouping
- **FOCUS mapping**: `InvoiceId`, `InvoiceDetailId`, `InvoiceIssueDate`, `PaymentDueDate`, etc.
- **Status**: ❌ Entire `focus_invoices` table missing

#### `ContractCommitment`
- **Purpose**: Commitment-based purchase (RI/SP/CUD)
- **FOCUS mapping**: 27 columns
- **Status**: ❌ Entire `focus_commitments` table missing

#### `Allocation`
- **Purpose**: Provider-side allocation of shared costs
- **FOCUS mapping**: `AllocatedMethodDetails/Id/ResourceId/ResourceName/Tags`
- **Status**: ❌ All 5 columns missing

### 14.2 Enrichment Entities (Our Extension)

#### `BusinessEntity` (new)
- **Purpose**: Org unit mapping (Team → Department → Division → Company)
- **Source**: Virtual tags + enrichment table
- **Usage**: Showback/chargeback cost centers

#### `Product` (new)
- **Purpose**: Product/feature mapping
- **Source**: User-defined or imported
- **Usage**: Unit economics per product

#### `Environment` (new)
- **Purpose**: Environment classification (prod/staging/dev)
- **Source**: Tag-based or manual
- **Usage**: Environment cost analysis

#### `Application` (new)
- **Purpose**: Application-level cost grouping
- **Source**: Tag-based or manual
- **Usage**: Application cost trends

---

## 15. Data Ingestion Architecture

### 15.1 Target Architecture

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Provider     │   │ Incoming Raw │   │ Normalized   │   │ Enriched     │
│ API/Export   │→  │ Data Store   │→  │ FOCUS Store  │→  │ Cost Charges  │
│              │   │ (immutable)  │   │ (cost_charges)│   │ + Enrichments│
└──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
```

### 15.2 Raw Data Layer (New)

**Purpose**: Store immutable provider-native data before normalization. Enables re-normalization without re-fetching.

**Implementation**: Parquet files in object storage (S3-compatible), or a `raw_provider_data` table with JSONB columns.

**Schema**:
```sql
CREATE TABLE raw_provider_data (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    connection_id UUID REFERENCES cloud_connections(id),
    provider_name VARCHAR(64) NOT NULL,
    fetch_started_at TIMESTAMPTZ NOT NULL,
    fetch_completed_at TIMESTAMPTZ,
    source_type VARCHAR(64) NOT NULL,  -- ce_api, cur_csv, bq_export, azure_csv
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    data JSONB NOT NULL,
    row_count INTEGER,
    status VARCHAR(32) DEFAULT 'pending',
    error TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 15.3 Ingestion Pipeline (Enhanced)

```
Scheduler (cron/worker)
  ↓
For each tenant:
  For each provider connection:
    1. Fetch raw data from provider API/export
    2. Store raw data in raw_provider_data (immutable)
    3. Normalize to FOCUS 1.4 via normalize_charge_row()
    4. Upsert into cost_charges
    5. Update connection last_synced_at
```

### 15.4 Incremental Sync Strategy

- **Watermark**: Each connection stores `last_synced_at` and `last_synced_period_start`/`end`
- **Default window**: `last_synced_period_end` → `now()` (or end of current month)
- **Backfill**: Explicit `period_start`/`period_end` parameters for historical data
- **Full re-sync**: Clear cost_charges for a period and re-fetch from raw data

---

## 16. Normalization Architecture

### 16.1 Current (FOCUS 1.2) → Target (FOCUS 1.4)

The existing `normalize_charge_row()` in `schema.py` needs to be extended to map all 20+ new columns in Phase 1.

**New mapping logic for each provider**:

| New Column | AWS CUR | AWS CE | GCP BQ | Azure CSV | SaaS |
| ---------- | ------- | ------ | ------ | --------- | ---- |
| `billing_account_id` | `bill/PayerAccountId` | N/A | `billing_account_id` | `BillingAccountId` | N/A |
| `billing_account_name` | `bill/PayerAccountName` | N/A | N/A | `BillingAccountName` | N/A |
| `invoice_id` | `bill/InvoiceId` | N/A | N/A | `InvoiceId` | N/A |
| `pricing_category` | `lineItem/LegalEntity` | N/A | N/A | `PricingCategory` | N/A |
| `list_unit_price` | `pricing/PublicOnDemandCost` | N/A | `sku.rate` | `ListUnitPrice` | N/A |
| `sku_meter` | `product/ProductCode` | N/A | `sku.id` | `SkuMeter` | N/A |
| `availability_zone` | `lineItem/AvailabilityZone` | N/A | `location.zone` | `AvailabilityZone` | N/A |
| `service_provider_name` | `lineItem/ServiceProvider` | N/A | N/A | `ServiceProviderName` | N/A |
| `charge_frequency` | N/A | N/A | N/A | `ChargeFrequency` | N/A |

### 16.2 Service Category Mapping (Extended)

Current `SERVICE_CATEGORY_MAP` has ~55 entries. Target needs:

- **Cloud**: AWS (50+ services), GCP (40+), Azure (60+)
- **AI**: OpenAI models, Anthropic, Cohere, Azure AI, AWS Bedrock, GCP Vertex AI
- **SaaS**: Slack, Jira, GitHub, Datadog, New Relic, Mongo, Twilio, etc.
- **Data**: Snowflake, Databricks, Redshift, BigQuery
- **Network**: Cloudflare, Fastly, Akamai
- **Observability**: Grafana, Datadog, New Relic, Splunk

### 16.3 FOCUS Validator

New service to validate `cost_charges` against FOCUS 1.4:

```python
def validate_focus_conformance(rows: List[Dict]) -> Dict:
    """Validate rows against FOCUS 1.4 requirements.
    Returns: {score: float, errors: List[str], warnings: List[str]}
    """
```

---

## 17. Allocation Architecture

### 17.1 Current Architecture (Works Well, Keep)

- **Virtual tags**: 12 FOCUS dimensions + tag.* keys; 5 operators (eq/contains/regex/in/exists); first-match-wins funnel
- **Shared cost reallocation**: 3 strategies (even/manual%/telemetry); redistributes buckets
- **Showback/Chargeback**: Informational vs billable statements
- **Preview**: Dry-run rules against charges in memory

### 17.2 Target Additions

1. **Allocation result caching**: Pre-compute and store allocation results for common time periods
2. **Hierarchical allocation**: Tag value → parent tag value → cost center
3. **Tag propagation**: Inherit tags from parent resources
4. **Allocation breakdown on cost_charges**: Add `x_allocated_value` and `x_allocated_virtual_tag_id` columns so line items carry their allocation assignment
5. **Allocation dashboards**: Show allocation coverage, untagged %, shared cost impact

---

## 18. Semantic Layer

### 18.1 Metric Definitions

| Metric | FOCUS Column | Description |
| ------ | ------------ | ----------- |
| **BilledCost** | `billed_cost` | Amount charged by provider |
| **EffectiveCost** | `effective_cost` | Cost after commitments/discounts |
| **ListCost** | `list_cost` | On-demand/undiscounted cost |
| **ContractedCost** | `contracted_cost` | Negotiated rate cost |
| **Savings** | `list_cost - effective_cost` | Savings from commitments/discounts |
| **CoverageRate** | `covered / total * 100` | % of spend covered by commitments |
| **SavingsRate** | `savings / list_cost * 100` | % savings vs on-demand |
| **CostPerUnit** | `effective_cost / consumed_quantity` | Unit cost |
| **LineCount** | `COUNT(*)` | Number of charge lines |

### 18.2 Dimension Definitions

| Dimension | FOCUS Column | Values |
| --------- | ------------ | ------ |
| **Provider** | `provider_name` | aws, gcp, azure, saas, ai |
| **Service Category** | `service_category` | Compute, Storage, Databases, etc. |
| **Service Name** | `service_name` | Amazon EC2, Compute Engine, etc. |
| **Sub Account** | `sub_account_id` | AWS account ID, GCP project ID |
| **Region** | `region_name` | us-east-1, europe-west1 |
| **Charge Category** | `charge_category` | Usage, Purchase, Tax, Credit |
| **Resource Type** | `resource_type` | Instance, Bucket, Volume |
| **Resource** | `resource_id` | i-123, vm-456 |
| **SKU** | `sku_id` | Provider SKU identifier |
| **Commitment** | `commitment_discount_type` | Reserved, SavingsPlan |
| **Tag** | `tags[key]` | Dynamic tenant-specific values |
| **Virtual Tag** | (derived) | Dynamic allocation values |

---

## 19. Query Architecture

### 19.1 Current Limitations

1. Single-dimension group-by only
2. Synchronous query execution
3. No query result caching
4. No pre-computed aggregations
5. Full table scan for every query

### 19.2 Target Query Architecture

```
┌──────────────┐
│   Query      │
│   Request    │
└──────┬───────┘
       │
┌──────▼───────┐
│  Query       │
│  Router      │
│              │
│  - Parse dims│
│  - Validate  │
│  - Check     │
│    cache     │
└──────┬───────┘
       │
       ├────────────────────────────────────┐
       │                                    │
┌──────▼───────┐                    ┌───────▼────────┐
│  Cache Hit   │                    │  Cache Miss    │
│  (Redis)     │                    │                │
│  Return      │                    │  ┌──────────┐  │
│  cached      │                    │  │ SQL Gen  │  │
│  result      │                    │  └────┬─────┘  │
└──────────────┘                    │       │        │
                                    │  ┌────▼─────┐  │
                                    │  │ Query    │  │
                                    │  │ Executor │  │
                                    │  └────┬─────┘  │
                                    │       │        │
                                    │  ┌────▼─────┐  │
                                    │  │ Cache    │  │
                                    │  │ Result   │  │
                                    │  └──────────┘  │
                                    └────────────────┘
```

### 19.3 Caching Strategy

| Cache | Type | TTL | Invalidation |
| ----- | ---- | --- | ------------ |
| Dimension values | Redis | 5 min | On ingest |
| Summary (current month) | Redis | 5 min | On ingest |
| Summary (historical) | Redis | 1 hour | On ingest |
| Timeseries (current) | Redis | 5 min | On ingest |
| Dimension autocomplete | React Query | 5 min | Page refresh |

### 19.4 Materialized Aggregations

Create a `focus_aggregations` table for pre-computed rollups:

```sql
CREATE TABLE focus_aggregations (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    dimension1 VARCHAR(64),  -- group-by dim name
    dimension1_value VARCHAR(255),  -- group-by dim value
    dimension2 VARCHAR(64),
    dimension2_value VARCHAR(255),
    billing_currency VARCHAR(16),
    billed_cost NUMERIC(18,6),
    effective_cost NUMERIC(18,6),
    list_cost NUMERIC(18,6),
    line_count INTEGER,
    UNIQUE(tenant_id, period_start, period_end, dimension1, dimension1_value, billing_currency)
);
```

Refresh via scheduled job or trigger after each ingest.

---

## 20. Analytics Architecture

### 20.1 Budget vs Actual

**New table**:
```sql
CREATE TABLE focus_budgets (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    scope_type VARCHAR(64),  -- provider, service_category, sub_account, virtual_tag
    scope_value VARCHAR(255),
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    budget_amount NUMERIC(18,6) NOT NULL,
    currency VARCHAR(16) DEFAULT 'USD',
    alert_threshold NUMERIC(5,2),  -- e.g. 80% = alert at 80% spend
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 20.2 Forecasting (ML)

**Approach**: Use historical `cost_charges` data with `Prophet` or `statsmodels`:
- Daily aggregation per service/provider
- Seasonal decomposition (day-of-week, month-of-year)
- Linear trend + changepoint detection
- Store results in `focus_forecasts` table

### 20.3 Anomaly Detection (FOCUS-Native)

**Approach**: Z-score on daily aggregates from `cost_charges`:
```
daily_spend = SUM(effective_cost) GROUP BY tenant_id, sub_account_id, charge_period_start::date
z_score = (daily_cost - rolling_mean(30)) / rolling_std(30)
flag when z_score > 2.5 or z_score < -2.5
```

---

## 21. Unit Economics Architecture

### 21.1 Current State

- User manually enters business metrics (users, requests, etc.)
- Cost-per-unit = scoped spend / quantity
- Trend comparison to prior period
- Revenue margin when revenue metric provided
- Segmented allocation per dimension

### 21.2 Target Enhancements

1. **Telemetry API**: Accept metric data via API (not just manual entry)
2. **Native usage metrics**: Derive from `ConsumedQuantity` + `ConsumedUnit` where available
3. **Auto-scaling**: New connections auto-suggest metrics based on provider data
4. **Multi-period trends**: Show 3/6/12 month trend
5. **Forecast unit cost**: Predict future cost-per-unit based on trend
6. **Competitive benchmarking**: Compare unit costs against industry baselines

---

## 22. Enterprise Requirements

### 22.1 Multi-Tenancy

- **Current**: ✅ Every query scoped to `tenant_id`. `resolve_active_tenant_id()` on every route.
- **Gap**: No cross-tenant reporting for platform admins. No tenant comparison.
- **Target**: Platform admin can view aggregated (anonymized) statistics. Individual tenant data is always isolated.

### 22.2 RBAC

- **Current**: ✅ Feature flags, `costs.analysis` permission, sales-lockable surfaces.
- **Gap**: No row-level security (RLS). A user with `costs.analysis` can see all charges in their tenant.
- **Target**: PostgreSQL RLS on `cost_charges` by `tenant_id`. Fine-grained permissions (view by service, by account).

### 22.3 Auditability

- **Current**: Logger with structured `action` + `session_id` + `error_type` fields.
- **Gap**: No query audit log. No "who saw what data" tracking.
- **Target**: Log every non-trivial query with user_id, filters, response size, latency. Store in `audit_log` table.

### 22.4 Data Lineage

- **Current**: `connection_id` on cost_charges links back to `cloud_connections`.
- **Gap**: No provenance tracking. Can't answer "where did this charge originate?" end-to-end.

### 22.5 Performance at Scale

- **Current**: Synchronous queries, no partitioning, no materialized views.
- **Target**: Partitioned tables, aggregated rollups, async queries for large datasets, query timeout guardrails.

---

## 23. Current vs Target Gap Analysis

### 23.1 Prioritized Gap Matrix

| Area | Current State | Target State | Gap | Priority | Complexity |
| ---- | ------------- | ------------ | --- | -------- | ---------- |
| **FOCUS column coverage** | 37 columns | 60+ columns (v1.4) | 25+ missing P0/P1 columns | P0 | Medium |
| **Multi-dimension group-by** | Single dimension | Multi-dimension with rollup | New query pattern | P0 | Medium |
| **SaaS ingestion** | 500 BillingRecord limit | Full SaaS sync via adapters | No dedicated AI/SaaS adapters | P0 | Medium |
| **AI cost ingestion** | None | OpenAI/Anthropic/Cohere adapters | Missing entirely | P0 | Low |
| **Invoice reconciliation** | None | Invoice table + reconciliation | Missing entirely | P0 | High |
| **Commitment tracking** | 4 columns on cost_charges | Dedicated commitments table + dashboard | Missing entirely | P0 | High |
| **Data partitioning** | None | Time-based partitions | Missing | P1 | Medium |
| **Async queries** | Synchronous only | Async job queue for large queries | Missing | P1 | High |
| **Aggregation caching** | None | Redis + materialized views | Missing | P1 | Medium |
| **Server-side saved views** | localStorage only | DB-backed saved views + sharing | Missing | P1 | Low |
| **Budget vs actual** | None | Budget table + tracking | Missing | P1 | Medium |
| **Forecasting** | None (cost_intel is connection-scoped) | FOCUS-native ML forecasting | Missing | P1 | High |
| **Anomaly detection** | Routing only (no detection) | FOCUS-native z-score detection | Missing | P1 | Medium |
| **Unit economics telemetry** | Manual metric entry | API + auto-suggest | Gap | P1 | Low |
| **Multi-provider comparison** | Tabular only | Unified explorer with overlays | Partial | P1 | Medium |
| **Drill path hierarchy** | Click → filter | Breadcrumb path: Provider→Service→Resource | Partial | P1 | Low |
| **Stacked/grouped charts** | Single series | Multi-series stacked/grouped | Missing | P2 | Medium |
| **Period comparison** | None | MoM, YoY, custom comparison | Missing | P2 | Medium |
| **Sankey/cost flow** | None | Allocation flow visualization | Missing | P2 | High |
| **Allocation results on charges** | In-memory only | Persisted x_allocated columns | Missing | P2 | Medium |
| **Data quality/conformance** | None | FOCUS validator | Missing | P2 | Low |
| **Full-text search** | Exact match only | ILIKE/tsvector on charge descriptions | Missing | P2 | Low |
| **Custom reports** | CSV export | Scheduled reports + multiple formats | Missing | P3 | High |
| **Competitive benchmarking** | None | Industry comparisons | Missing | P3 | High |
| **On-prem/data center** | None | Legacy cost allocation | Missing | P3 | High |

---

## 24. Prioritized Roadmap

### P0 — Must Have (Foundation)

| # | Feature | Why P0 | Effort |
| - | ------- | ------ | ------ |
| 1 | Add 25+ FOCUS 1.4 columns to cost_charges | Blocking invoice/commitment/allocation feature | 1 week |
| 2 | Update normalize_charge_row() for new columns | Need to populate new columns | 1 week |
| 3 | Add AI ingestion adapters (OpenAI, Anthropic) | Fastest-growing cost category, zero coverage | 1 week |
| 4 | Add full SaaS ingestion (not just 500 records) | Most connectors not mapped to FOCUS | 1 week |
| 5 | Multi-dimension group-by API | Users need Provider+Service+Account rollups | 2 weeks |
| 6 | Data partitioning strategy | Without this, performance degrades catastrophically | 1 week |
| 7 | Replace old focus_sync_worker | Uses old single-connection API | 0.5 week |
| 8 | Fix billing_sync auto-trigger | GCP/Azure silently skipped | 0.5 week |

### P1 — Enterprise Usability

| # | Feature | Why P1 | Effort |
| - | ------- | ------ | ------ |
| 9 | Redis caching for aggregates | Dramatic latency reduction | 1 week |
| 10 | Materialized rollup views | Pre-compute common group-bys | 1 week |
| 11 | Server-side saved views | Sharing, persistence, team collaboration | 1 week |
| 12 | Budget vs actual | Operational necessity | 2 weeks |
| 13 | Focus-native anomaly detection | Unlock proactive cost management | 1 week |
| 14 | Invoice table + reconciliation | Enterprise sales differentiator | 2 weeks |
| 15 | Commitment table + coverage dashboard | Cross-provider commitment comparison | 2 weeks |
| 16 | Drill-path hierarchy UI | Navigate Provider→Account→Service→Resource | 1 week |
| 17 | Unit economics telemetry API | Let platforms push metrics programmatically | 1 week |
| 18 | Multi-currency UI improvements | Honest rendering of mixed currencies | 0.5 week |

### P2 — Advanced Capabilities

| # | Feature | Why P2 | Effort |
| - | ------- | ------ | ------ |
| 19 | Stacked/grouped time-series chart | Visual multi-provider comparison | 1 week |
| 20 | Period comparison (MoM/YoY) | Core FinOps requirement | 2 weeks |
| 21 | Allocation results on cost_charges | Persist allocation for query performance | 1 week |
| 22 | FOCUS conformance validator | Trust signal for enterprise sales | 1 week |
| 23 | Async query execution for large datasets | Prevent timeouts on 50M+ row datasets | 2 weeks |
| 24 | Full-text search on line items | Find charges by description/resource | 1 week |

### P3 — Future

| # | Feature | Why P3 | Effort |
| - | ------- | ------ | ------ |
| 25 | ML forecasting on FOCUS data | Predictive cost management | 3 weeks |
| 26 | Sankey cost flow chart | Allocation visualization | 2 weeks |
| 27 | Scheduled report delivery | Email/Slack reports | 2 weeks |
| 28 | Competitive benchmarking | Industry cost comparisons | 3 weeks |
| 29 | On-prem/data center cost support | Total cost of ownership | 3 weeks |

---

## 25. Exact Implementation Plan

### Phase 1: Foundation (Weeks 1-2)

**Objective**: Data model completeness + performance foundation.

**Step 1.1**: Add FOCUS 1.4 columns to `cost_charges`
- Files to modify:
  - `backend/app/db/models.py` — Add 25+ new columns to `CostCharge`
  - `backend/alembic/versions/0024_focus_1_4_columns.py` — New migration
- New columns: `billing_account_id/name/type`, `sub_account_type`, `invoice_id/detail_id`, `pricing_category`, `list_unit_price`, `contracted_unit_price`, `pricing_currency`, `sku_meter`, `sku_price_details` (JSONB), `availability_zone`, `charge_frequency`, `service_provider_name`, `host_provider_name`, `commitment_discount_name/quantity/unit`, `commitment_program_eligibility` (JSONB), `contract_id`, `contract_applied` (JSONB), `allocated_method_id/details/resource_id/resource_name/tags`
- Migration strategy: `ALTER TABLE ADD COLUMN IF NOT EXISTS` for idempotency
- Tests: Verify all new columns exist, types are correct

**Step 1.2**: Update normalization
- Files to modify:
  - `backend/services/focus/schema.py` — Update `normalize_charge_row()` to map new columns
  - `backend/services/focus/ingest.py` — Update all adapters to pass new fields
- Backward compatibility: Old rows get NULL for new columns
- Tests: Unit tests for each provider → FOCUS mapping

**Step 1.3**: Add time-based partitioning
- New migration: `ALTER TABLE cost_charges SET PARTITION BY RANGE (charge_period_start)`
- Create monthly partitions for current + next 3 months
- pg_partman or manual partition management

**Step 1.4**: Fix worker and sync bugs
- Files to modify:
  - `backend/workers/focus_sync_worker.py` — Use `_connections_with_credentials()` pattern
  - `backend/services/billing_sync_service.py:1112-1144` — Resolve GCP/Azure connections

**Acceptance criteria**:
- [ ] `uv run alembic upgrade head` succeeds
- [ ] All 25+ new columns present on `cost_charges`
- [ ] `normalize_charge_row()` can map all FOCUS 1.4 columns from at least one provider
- [ ] Focus sync worker ingests from all AWS/GCP/Azure connections
- [ ] Billing sync auto-trigger ingests GCP and Azure
- [ ] `uv run pytest tests/` passes (or N/A)
- [ ] `uv run ruff check .` passes

### Phase 2: Ingestion Completeness (Weeks 3-4)

**Objective**: All technology providers can push cost data to FOCUS.

**Step 2.1**: AI ingestion adapters
- New files:
  - Extend `backend/services/focus/ingest.py` with `ingest_openai_cost()`, `ingest_anthropic_cost()`
  - `backend/services/focus/ai_providers.py` — Provider-specific logic
- Each adapter maps: API costs → `ConsumedQuantity` (tokens), `SkuId` (model), `ServiceCategory` ("AI and Machine Learning")

**Step 2.2**: Full SaaS ingestion
- New service: `backend/services/focus/saas_ingestion.py`
- Removes the 500-record limit; iterates all `BillingRecord` rows
- Maps each non-cloud SaaS tool → FOCUS rows
- Adds Databricks, Snowflake, and data platform support

**Step 2.3**: Ingest scheduler
- New worker: `backend/workers/focus_sync_scheduler.py`
- Runs on a cron schedule (default: daily)
- Iterates all tenants with cloud/SaaS connections
- Respects `last_synced_at` for incremental sync

**Acceptance criteria**:
- [ ] OpenAI/Antropic API costs appear in Focus Explorer
- [ ] All SaaS tools with `BillingRecord` data appear in Focus Explorer
- [ ] Scheduled sync runs without errors
- [ ] `ingest_saas_from_billing_records()` has no row limit

### Phase 3: Query Performance (Weeks 5-6)

**Objective**: Sub-second aggregation queries even at 50M+ rows.

**Step 3.1**: Redis caching
- Files to modify:
  - `backend/services/redis_client.py` — Ensure available
  - `backend/services/focus/charge_store.py` — Add cache check before SQL; write after SQL
- Cache keys: `focus:summary:{tenant_id}:{hash(filters)}`, `focus:timeseries:{tenant_id}:{hash(filters)}`
- TTL: 5 min for current month, 1 hour for historical

**Step 3.2**: Materialized rollup views
- New migration: `focus_aggregations` table
- New service: `backend/services/focus/aggregation_service.py`
- Refresh after each successful ingest
- Query router checks materialized views first

**Step 3.3**: Async queries (optional in this phase)
- New table: `focus_query_jobs` — tracks async query status
- New endpoint: `POST /api/v1/finops/focus/query` — submit async query
- New endpoint: `GET /api/v1/finops/focus/query/{id}` — poll result
- Worker executes query and stores result

**Acceptance criteria**:
- [ ] Summary queries under 200ms for cached results
- [ ] Materialized aggregates refresh after ingest
- [ ] Rollup query returns same data as live query (within tolerance)

### Phase 4: Invoice Reconciliation (Weeks 7-8)

**Objective**: Match invoices to consumption with variance detection.

**Step 4.1**: New `focus_invoices` table
- New migration `0025_focus_invoices`
- Model in `models.py`

**Step 4.2**: Invoice parsing from provider exports
- AWS: `bill/InvoiceId` from CUR
- Azure: `InvoiceId` from native FOCUS export
- GCP: Limited — invoice data in BigQuery (requires mapping)

**Step 4.3**: Reconciliation endpoint
- New routes: `backend/routes/invoice_routes.py`
- `POST /api/v1/finops/invoices/reconcile` — runs reconciliation
- `GET /api/v1/finops/invoices` — list invoices
- `GET /api/v1/finops/invoices/{id}/line-items` — charges for an invoice

**Step 4.4**: Frontend reconciliation dashboard
- New component: `frontend/src/components/finops/invoice-reconciliation.tsx`
- Shows: invoice list, match status, variance %, line items

### Phase 5: Commitment Intelligence (Weeks 9-10)

**Objective**: Track and analyze commitments across providers.

**Step 5.1**: New `focus_commitments` table
- New migration `0026_focus_commitments`
- Model in `models.py`

**Step 5.2**: Commitment parsing
- AWS: CE API `get_reservation_utilization`, `get_savings_plans_utilization`
- GCP: BigQuery commitment data
- Azure: Reservation/RI data from Azure API

**Step 5.3**: Coverage dashboard
- New routes: `backend/routes/commitment_routes.py`
- `GET /api/v1/finops/commitments/coverage` — coverage by service/provider
- `GET /api/v1/finops/commitments/utilization` — waste tracking
- `GET /api/v1/finops/commitments/comparison` — cross-provider comparison

**Step 5.4**: Frontend commitment dashboard
- New component: `frontend/src/components/finops/commitment-dashboard.tsx`

### Phase 6: Analytics (Weeks 11-12)

**Objective**: Budgets, forecasts, anomaly detection from FOCUS data.

**Step 6.1**: Budgets
- New table: `focus_budgets`
- New routes: CRUD + budget vs actual query
- Frontend: budget management + actual tracking

**Step 6.2**: Forecasting
- New service: `backend/services/forecasting/focus_forecast_service.py`
- Prophet-based model trained on daily FOCUS aggregates
- Returns: predicted cost, confidence interval, trend components

**Step 6.3**: Anomaly detection
- New service: `backend/services/focus/anomaly_service.py`
- Daily z-score on cost_charges aggregates
- Routes to Slack/Jira via existing `anomaly_routing_service.py`

### Phase 7: Frontend Explorer v2 (Weeks 13-16)

**Objective**: Enterprise-grade exploration experience.

**Step 7.1**: Server-side saved views
- New table: `focus_saved_views`
- API: CRUD for saved views (not localStorage)
- Frontend: saved view manager with search, share, set as default

**Step 7.2**: Multi-dimension group-by
- API: Accept `group_by` as comma-separated list
- SQL: `GROUP BY GROUPING SETS` or multiple queries
- Frontend: Multi-select dimension picker

**Step 7.3**: Period comparison
- API: `compare_to` parameter (previous_period, same_period_last_year)
- Frontend: Comparison columns in table, overlay in chart

**Step 7.4**: Drill path hierarchy
- Frontend: Breadcrumb showing current path
- Click Provider → auto-filter + show Sub Accounts → click Account → show Services

**Step 7.5**: Enhanced chart
- Stacked area/bar chart for multi-dimension
- Chart selector: area, bar, line, pie

---

## 26. Database Migration Strategy

### 26.1 Guiding Principles

1. **Idempotent migrations**: All migrations should be safe to run multiple times
2. **Add-only patterns**: Prefer `ADD COLUMN IF NOT EXISTS` over destructive changes
3. **Backward compatibility**: Old application code must work after each migration
4. **Linear head**: Exactly one head at all times (per `AGENTS.md`)
5. **Migration per concern**: One migration per logical change, not one per day

### 26.2 Migration Sequence

| # | Revision | Description | Type |
| - | -------- | ----------- | ---- |
| Current | `0023` | Rename cost_observations unique | Existing |
| Next | `0024` | Add FOCUS 1.4 columns to cost_charges (25+ columns) | ALTER TABLE ADD COLUMN |
| Next | `0025` | Create focus_invoices table | CREATE TABLE |
| Next | `0026` | Create focus_commitments table | CREATE TABLE |
| Next | `0027` | Create focus_aggregations table + indexes | CREATE TABLE |
| Next | `0028` | Create focus_saved_views table | CREATE TABLE |
| Next | `0029` | Create focus_budgets table | CREATE TABLE |
| Next | `0030` | Create focus_forecasts table | CREATE TABLE |
| Next | `0031` | Add partitioning to cost_charges (if not using pg_partman) | ALTER TABLE...SET PARTITION |

### 26.3 Example Migration: 0024

```python
"""Add FOCUS 1.4 columns to cost_charges.

Revision ID: 0024
Revises: 0023
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import JSONB

def upgrade():
    # Account hierarchy
    for col in [
        ("billing_account_id", sa.String(255)),
        ("billing_account_name", sa.String(255)),
        ("billing_account_type", sa.String(64)),
        ("sub_account_type", sa.String(64)),
        ("invoice_id", sa.String(512)),
        ("invoice_detail_id", sa.String(512)),
        ("pricing_category", sa.String(64)),
        ("list_unit_price", sa.Numeric(18, 8)),
        ("contracted_unit_price", sa.Numeric(18, 8)),
        ("pricing_currency", sa.String(16)),
        ("sku_meter", sa.String(255)),
        ("availability_zone", sa.String(128)),
        ("charge_frequency", sa.String(32)),
        ("service_provider_name", sa.String(128)),
        ("host_provider_name", sa.String(128)),
        ("commitment_discount_name", sa.String(255)),
        ("commitment_discount_quantity", sa.Numeric(18, 6)),
        ("commitment_discount_unit", sa.String(64)),
        ("contract_id", sa.String(255)),
        ("allocated_method_id", sa.String(255)),
        ("allocated_resource_id", sa.String(512)),
        ("allocated_resource_name", sa.String(512)),
    ]:
        op.execute(f"ALTER TABLE cost_charges ADD COLUMN IF NOT EXISTS {col[0]} {col[1].compile()}")
    
    # JSONB columns
    for col in ["sku_price_details", "commitment_program_eligibility", "contract_applied",
                 "allocated_method_details", "allocated_tags"]:
        op.execute(f"ALTER TABLE cost_charges ADD COLUMN IF NOT EXISTS {col} JSONB")

    # New indexes
    op.create_index("ix_cost_charges_invoice_id", "cost_charges", ["tenant_id", "invoice_id"])
    op.create_index("ix_cost_charges_billing_account", "cost_charges", ["tenant_id", "billing_account_id"])
```

---

## 27. API Evolution Strategy

### 27.1 Versioning

Current API is under `/api/v1/`. Keep v1 stable. New endpoints should use `/api/v2/` when breaking changes are needed.

### 27.2 New Endpoints (Additive)

| Phase | Method | Endpoint | Purpose |
| ----- | ------ | -------- | ------- |
| 1 | GET | `/api/v1/finops/focus/summary` | Add multi-dimension group_by (comma-separated) |
| 1 | GET | `/api/v1/finops/focus/dimensions` | Return all available dimensions + metadata |
| 3 | GET | `/api/v1/finops/focus/query/{id}` | Poll async query result |
| 3 | POST | `/api/v1/finops/focus/query` | Submit async query |
| 4 | GET | `/api/v1/finops/invoices` | List invoices |
| 4 | POST | `/api/v1/finops/invoices/reconcile` | Run reconciliation |
| 5 | GET | `/api/v1/finops/commitments/coverage` | Coverage analysis |
| 5 | GET | `/api/v1/finops/commitments/comparison` | Cross-provider comparison |
| 6 | GET/POST | `/api/v1/finops/budgets` | Budget CRUD |
| 6 | GET | `/api/v1/finops/budgets/actual` | Budget vs actual |
| 6 | GET | `/api/v1/finops/forecasts` | Forecast data |
| 6 | POST | `/api/v1/finops/forecasts/generate` | Trigger forecast generation |
| 6 | GET | `/api/v1/finops/anomalies` | Detected anomalies |
| 7 | GET/POST | `/api/v1/finops/saved-views` | Server-side saved views |

### 27.3 Backward Compatibility Rules

1. Never remove or rename existing query parameters
2. New parameters are always optional (default to existing behavior)
3. Response can add new fields but never remove existing fields
4. Add `_v2` suffix to responses for breaking format changes

---

## 28. Frontend Evolution Strategy

### 28.1 Component Hierarchy (Target)

```
FocusExplorerPage
├── EnterpriseGate
├── ArchivedBanner (if data is old)
├── FilterBar
│   ├── DateRangePresets (7d, 30d, this month, custom)
│   ├── DimensionFilter (7+ auto-complete combos)
│   └── SavedViewSelector (server-side)
├── ActionBar
│   ├── GroupByMultiSelect (1-3 dimensions)
│   ├── CostFieldToggle (Billed/Effective/List)
│   ├── GranularitySelect (day/week/month)
│   ├── PeriodComparisonToggle (MoM/YoY/none)
│   └── ExportButton (CSV/JSON/Excel)
├── KPIRow
│   ├── TotalCost (with currency breakdown)
│   ├── LineItemCount
│   ├── CoveragePercent (if allocation active)
│   └── LastSyncTimestamp
├── ChartSection
│   ├── ChartTypeSelect (area/bar/line/stacked)
│   └── TimeSeriesChart (Recharts/Nivo)
├── DrillBreadcrumb (Provider → Account → Service → Resource)
├── BreakdownTable (sortable, paginated, click-to-drill)
├── LineItemsTable (sortable, searchable, paginated)
└── ComparisonPanel (side-by-side period comparison)
```

### 28.2 New Components Needed

| Component | File | Purpose | Phase |
| --------- | ---- | ------- | ----- |
| `focus/saved-views.tsx` | Server-side view CRUD | 7 |
| `focus/invoice-reconciliation.tsx` | Invoice dashboard | 4 |
| `focus/commitment-dashboard.tsx` | Commitment tracker | 5 |
| `focus/budget-tracker.tsx` | Budget vs actual | 6 |
| `focus/anomaly-feed.tsx` | Detected anomalies | 6 |
| `focus/forecast-chart.tsx` | ML forecast overlay | 6 |
| `focus/ai-cost-dashboard.tsx` | AI-specific cost view | 2 |
| `focus/sankey-flow.tsx` | Allocation flow viz | 7 |
| `focus/comparison-panel.tsx` | Period comparison | 7 |

### 28.3 State Management Evolution

- Current: React Query for API + localStorage for views
- Target: React Query + Zustand for persisted UI state + server-side views API
- Avoid: Redux, Context-heavy patterns

---

## 29. Testing Strategy

### 29.1 Backend Tests

| Test Suite | File | What It Tests | Phase |
| ---------- | ---- | ------------- | ----- |
| Schema normalization | `tests/test_focus_schema.py` | `normalize_charge_row()` for each provider input shape | 1 |
| Charge store | `tests/test_focus_charge_store.py` | `persist_cost_charges()`, `summarize_focus()`, `query_cost_charges()` | 1 |
| Ingest adapters | `tests/test_focus_ingest.py` | Each adapter with mock provider data | 1 |
| Allocation engine | `tests/test_virtual_tag_engine.py` | Rule matching, reallocation, showback/chargeback | 1 |
| Integration | `tests/test_focus_integration.py` | End-to-end ingest → normalize → query | 1 |
| Invoices | `tests/test_invoice_routes.py` | Invoice reconciliation | 4 |
| Commitments | `tests/test_commitment_routes.py` | Commitment coverage, comparison | 5 |
| Budgets | `tests/test_budget_routes.py` | Budget CRUD + actual queries | 6 |
| Forecasts | `tests/test_forecast_service.py` | Forecast generation | 6 |
| Anomalies | `tests/test_anomaly_service.py` | Z-score detection | 6 |
| API contracts | `tests/test_focus_contracts.py` | Request/response schema validation | 1 |

### 29.2 Frontend Tests

| Test Suite | File | What It Tests | Phase |
| ---------- | ---- | ------------- | ----- |
| CostExplorer | `__tests__/focus/cost-explorer.test.tsx` | Component rendering, filter changes | 7 |
| FilterCombobox | `__tests__/focus/filter-combobox.test.tsx` | Autocomplete behavior | 7 |
| API service | `__tests__/focus/finops-api.test.ts` | URL construction, response parsing | 1 |

### 29.3 Testing Principles

1. **Mock provider APIs**, never call real billing APIs in tests
2. **Use test database** with real PostgreSQL (not SQLite) for SQLAlchemy tests
3. **Factory fixtures** for test data (FOCUS rows, virtual tags, connections)
4. **Contract tests** for API request/response schemas
5. **Parallel execution** — tests should be independent

---

## 30. Performance and Scalability Strategy

### 30.1 Row Estimates

| Tenant Size | Monthly Charges | Annual Charges |
| ----------- | --------------- | -------------- |
| Small (1-2 providers) | 50K-200K | 600K-2.4M |
| Medium (3-5 providers) | 200K-1M | 2.4M-12M |
| Large (5+ providers, many resources) | 1M-10M | 12M-120M |
| Enterprise (multi-account, +SaaS) | 10M-50M | 120M-600M |

### 30.2 Performance Layers

| Layer | Strategy | Expected Latency |
| ----- | -------- | ---------------- |
| Redis cache | TTL-based aggregate cache | < 5ms |
| Materialized rollups | Pre-computed for common queries | < 50ms |
| Live query (partitioned) | Indexed + partitioned query | < 500ms |
| Live query (full scan) | Last resort for un-cached queries | < 5s |
| Async query | Job queue for complex/large queries | 5s-5min |

### 30.3 Scaling Bottlenecks

| Bottleneck | Risk | Mitigation |
| ---------- | ---- | ---------- |
| cost_charges full table scan | High | Partitioning + materialized views |
| In-memory allocation (virtual_tag_engine.py:248) | Medium | Batch processing + pagination |
| Synchronous HTTP aggregation | Medium | Async query for large datasets |
| No Redis for aggregates | High | Add caching (Phase 3) |
| Single Postgres instance | Medium | Read replicas for query workload |

---

## 31. Security and Multi-Tenancy Strategy

### 31.1 Current

- ✅ Tenant isolation via `tenant_id` on every query
- ✅ Feature flag gating (`FOCUS_EXPLORER`)
- ✅ Permission check (`costs.analysis`)
- ✅ Sales lock surface (`focus-explorer`)
- ✅ Connection-level credential encryption

### 31.2 Target Additions

| Feature | Implementation | Priority |
| ------- | -------------- | -------- |
| Row-Level Security (RLS) | PostgreSQL `ALTER TABLE cost_charges ENABLE ROW LEVEL SECURITY` | P1 |
| Audit logging for queries | Log every non-trivial query with user_id + filters + response size | P1 |
| Export access control | Permission check before CSV/JSON export | P1 |
| Query rate limiting | Existing `require_rate_limit()` already in place | P0 (done) |
| Cross-tenant isolation validation | Pen test: verify tenant A cannot query tenant B's data | P1 |

---

## 32. Observability Strategy

### 32.1 Current

- Structured logging with `action`, `session_id`, `error_type`, `duration_ms` fields
- Rate limiting via `require_rate_limit()`
- Worker health checks

### 32.2 Target

| Signal | Implementation | 
| ------ | -------------- |
| Query latency | Log every query with filters, duration, rows returned |
| Ingest latency | Track each adapter's fetch + normalize + persist time |
| Cache hit rate | Redis cache hit/miss ratio for aggregates |
| Error rate | Error count by adapter, query type |
| Row count growth | Daily count of cost_charges per tenant |
| Sync staleness | Per-connection hours since last successful sync |
| Query performance p50/p95/p99 | Latency percentiles for each endpoint |

---

## 33. Risks and Trade-offs

### 33.1 Key Risks

| Risk | Likelihood | Impact | Mitigation |
| ---- | ---------- | ------ | ---------- |
| FOCUS 1.4 migration breaks existing queries | Low | High | Add-only columns; NULL-safe queries |
| BigQuery query cost grows with row count | Medium | Medium | Limit scan range; filter early |
| Azure FOCUS CSV ingest fails for large tenants | Medium | Medium | Pagination + parallel blob reading |
| In-memory allocation crashes at >1M charges | High | High | Batch allocation + pagination |
| No historical data after migration | Low | High | Re-ingest after schema migration |

### 33.2 Trade-offs

| Decision | Option A (Chosen) | Option B (Rejected) | Rationale |
| -------- | ----------------- | ------------------- | --------- |
| Partitioning | Manual/pg_partman | Automatic (TimescaleDB) | Avoid new dependency |
| Async queries | Job queue approach | Websocket streaming | Simpler to implement |
| Cache invalidation | TTL-based | Event-driven | Avoid cache coherency complexity |
| Multi-dimension group-by | SQL GROUPING SETS | Application-level merge | Database handles correctly |
| Materialized views | Explicit table | PostgreSQL MATERIALIZED VIEW | More control over refresh |

---

## 34. Final Recommended Architecture

### 34.1 Data Flow (Phase 4+)

```
Provider API/Export
      ↓
Raw Data Layer (raw_provider_data / object storage)
      ↓ [normalize_charge_row() / FOCUS 1.4]
Focus Data Layer (cost_charges + partitions)
      ↓ [enrichment: tags, ownership, virtual tags]
Enriched Focus Data (cost_charges + x_allocated columns)
      ↓ [aggregation: pre-compute rollups]
Aggregation Layer (focus_aggregations)
      ↓ [caching: Redis]
API Layer (fast, cached responses)
      ↓
Focus Explorer UI (Next.js, React Query)
```

### 34.2 Key Design Decisions

1. **cost_charges remains the canonical table** — all FinOps features read from it
2. **Aggregations are pre-computed** — not computed live for common queries
3. **Raw data is immutable** — re-normalization never requires re-fetching
4. **FOCUS 1.4 is the internal contract** — all providers normalize to this schema
5. **Extensions prefixed with `x_`** — keep FOCUS columns pure, extend with `x_` columns
6. **Allocation is a column, not just a service** — `x_allocated_value` on cost_charges enables direct querying
7. **Async for large, sync for small** — split query strategy based on time window and tenant size

---

## 35. Final Recommended Implementation Sequence

### Immediate (Weeks 1-4)

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ FOCUS 1.4│  │ Fix      │  │ AI + SaaS│  │ Redis    │
│ columns  │→│ workers  │→│ adapters  │→│ caching  │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
```

### Short-term (Weeks 5-12)

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Partition│  │ Invoices │  │ Commit   │  │ Material.│
│ + rollout│→│ + Recon  │→│ -ments   │→│ views    │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
```

### Medium-term (Weeks 13-20)

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Budgets  │  │ Forecast │  │ Anomaly  │  │ Explorer │
│ + Actuals│→│ (ML)     │→│ Detection│→│ v2 Front │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
```

### First File to Modify

**`backend/app/db/models.py`** — Add 25+ FOCUS 1.4 columns to `CostCharge`.

### First Migration

**`backend/alembic/versions/0024_focus_1_4_columns.py`** — Idempotent ALTER TABLE.

### First Backend Service to Refactor

**`backend/services/focus/schema.py`** — Update `normalize_charge_row()` to map all new columns.

### First API Contract to Introduce

**`GET /api/v1/finops/focus/dimensions`** — Return available dimensions with metadata (type, allowed operators, category).

### First Frontend Component to Refactor

**`frontend/src/services/finops-allocation-service.ts`** — Add TypeScript interfaces for all new FOCUS 1.4 columns.

### First Provider Adapter to Standardize

**AWS CUR CSV adapter** (`ingest.py:190-267`) — It has the richest column set but reads only the first CSV. Add:
1. Multi-file support (sort by last modified, read most recent)
2. Full FOCUS 1.4 column mapping
3. Remove 5000-row limit (replace with streaming)
4. Incremental sync (compare `last_synced_at` with S3 object timestamps)

### First End-to-End Vertical Slice

1. Add columns to model ✅
2. Create migration ✅
3. Update normalize_charge_row() ✅
4. Update AWS CUR adapter to populate new columns ✅
5. Update summarize_focus() to group by new dimensions ✅
6. Add new dimensions to FOCUS_DIMENSIONS dict ✅
7. Update frontend to show new filter options ✅
8. Verify end-to-end with test data ✅

### Tests Required Before Continuing

1. `test_normalize_charge_row_maps_all_focus_1_4_columns()` — Verify every new column has a mapping path
2. `test_aws_cur_focus_14_mapping()` — Verify CUR CSV reader populates all new columns
3. `test_focus_summary_new_dimensions()` — Verify group-by on new dimensions works
4. `test_migration_0024_idempotent()` — Verify migration can run twice safely
5. `test_persist_cost_charges_new_columns()` — Verify upsert with new columns

---

## Appendix A: File Reference

### Core Focus Files (Backend)

| File | Lines | Purpose | Phase |
| ---- | ----- | ------- | ----- |
| `backend/services/focus/schema.py` | 211 | FOCUS normalization | 1 |
| `backend/services/focus/charge_store.py` | 575 | Persistence + query | 1 |
| `backend/services/focus/ingest.py` | 833 | Provider adapters | 1 |
| `backend/services/allocation/virtual_tag_engine.py` | 380 | Rule-based allocation | Existing |
| `backend/services/allocation_service.py` | 829 | Business logic | Existing |
| `backend/routes/focus_routes.py` | 413 | Focus API routes | 1 |
| `backend/routes/allocation_routes.py` | 646 | Allocation API routes | Existing |
| `backend/workers/focus_sync_worker.py` | 105 | Legacy sync worker | 1 (fix) |
| `backend/app/db/models.py` | 2537 | All SQLAlchemy models | 1 |

### Core Focus Files (Frontend)

| File | Lines | Purpose | Phase |
| ---- | ----- | ------- | ----- |
| `frontend/src/components/finops/cost-explorer.tsx` | 918 | Main explorer component | 7 |
| `frontend/src/components/finops/filter-combobox.tsx` | 154 | Autocomplete filter | 7 |
| `frontend/src/services/finops-allocation-service.ts` | 601 | API client + types | 1 |
| `frontend/src/app/dashboard/focus/page.tsx` | 62 | Focus page wrapper | Existing |
| `frontend/src/config/api.ts` | 298 | Endpoint definitions | 1 |

### Documentation Files

| File | Purpose |
| ---- | ------- |
| `docs/focus_Explorer.md` | Current Focus Explorer impl doc |
| `docs/FOCUS.md` | FOCUS spec analysis |
| `docs/finops-strategy/` | Product strategy + epics |
| `AGENTS.md` | Agent workflow guide |

---

## Appendix B: FOCUS 1.4 Column Reference

### Column Group: Account (6)
- **BillingAccountId** — P0 — NOT IMPLEMENTED
- **BillingAccountName** — P0 — NOT IMPLEMENTED
- **BillingAccountType** — P0 — NOT IMPLEMENTED
- **SubAccountId** — IMPLEMENTED (sub_account_id)
- **SubAccountName** — IMPLEMENTED (sub_account_name)
- **SubAccountType** — P0 — NOT IMPLEMENTED

### Column Group: Allocation (5)
- **AllocatedMethodDetails** — P2 — NOT IMPLEMENTED
- **AllocatedMethodId** — P2 — NOT IMPLEMENTED
- **AllocatedResourceId** — P2 — NOT IMPLEMENTED
- **AllocatedResourceName** — P2 — NOT IMPLEMENTED
- **AllocatedTags** — P2 — NOT IMPLEMENTED

### Column Group: Billing (9)
- **BilledCost** — IMPLEMENTED
- **BillingCurrency** — IMPLEMENTED
- **ConsumedQuantity** — IMPLEMENTED
- **ConsumedUnit** — IMPLEMENTED
- **ContractedCost** — IMPLEMENTED
- **ContractedUnitPrice** — P0 — NOT IMPLEMENTED
- **EffectiveCost** — IMPLEMENTED
- **ListCost** — IMPLEMENTED
- **ListUnitPrice** — P0 — NOT IMPLEMENTED

### Column Group: Billing Period (3)
- **BillingPeriodCreated** — P1 — NOT IMPLEMENTED
- **BillingPeriodLastUpdated** — P1 — NOT IMPLEMENTED
- **BillingPeriodStatus** — P1 — NOT IMPLEMENTED

### Column Group: Charge (4)
- **ChargeCategory** — IMPLEMENTED
- **ChargeClass** — IMPLEMENTED
- **ChargeDescription** — IMPLEMENTED
- **ChargeFrequency** — P1 — NOT IMPLEMENTED

### Column Group: Commitment Discount (8)
- **CommitmentDiscountCategory** — IMPLEMENTED
- **CommitmentDiscountId** — IMPLEMENTED
- **CommitmentDiscountName** — P0 — NOT IMPLEMENTED
- **CommitmentDiscountQuantity** — P0 — NOT IMPLEMENTED
- **CommitmentDiscountStatus** — IMPLEMENTED
- **CommitmentDiscountType** — IMPLEMENTED
- **CommitmentDiscountUnit** — P0 — NOT IMPLEMENTED
- **CommitmentProgramEligibilityDetails** — P0 — NOT IMPLEMENTED

### Column Group: Contract (27)
- **ContractApplied** — P0 — NOT IMPLEMENTED
- **ContractCommitmentId** — P2 — NOT IMPLEMENTED
- **(plus 25 more)** — P2 — NOT IMPLEMENTED

### Column Group: Invoice (11)
- **InvoiceId** — P0 — NOT IMPLEMENTED
- **InvoiceDetailId** — P0 — NOT IMPLEMENTED
- **InvoiceIssueDate** — P1 — NOT IMPLEMENTED
- **PaymentDueDate** — P1 — NOT IMPLEMENTED
- **PaymentTerms** — P1 — NOT IMPLEMENTED
- **PurchaseOrderNumber** — P1 — NOT IMPLEMENTED
- **PaymentCurrency** — P1 — NOT IMPLEMENTED
- **PaymentCurrencyBilledCost** — P1 — NOT IMPLEMENTED
- (+ 3 more) — P1 — NOT IMPLEMENTED

### Column Group: Location (3)
- **AvailabilityZone** — P1 — NOT IMPLEMENTED
- **RegionId** — IMPLEMENTED
- **RegionName** — IMPLEMENTED

### Column Group: Pricing (7)
- **PricingCategory** — P0 — NOT IMPLEMENTED
- **PricingCurrency** — P0 — NOT IMPLEMENTED
- **PricingQuantity** — IMPLEMENTED
- **PricingUnit** — IMPLEMENTED
- (+ 3 more) — P0 — NOT IMPLEMENTED

### Column Group: Resource (4)
- **ResourceId** — IMPLEMENTED
- **ResourceName** — IMPLEMENTED
- **ResourceType** — IMPLEMENTED
- **Tags** — IMPLEMENTED

### Column Group: Service (3)
- **ServiceCategory** — IMPLEMENTED
- **ServiceName** — IMPLEMENTED
- **ServiceSubcategory** — IMPLEMENTED

### Column Group: SKU (4)
- **SkuId** — IMPLEMENTED
- **SkuMeter** — P0 — NOT IMPLEMENTED
- **SkuPriceDetails** — P0 — NOT IMPLEMENTED
- **SkuPriceId** — IMPLEMENTED

### Column Group: Timeframe (4)
- **BillingPeriodEnd** — IMPLEMENTED
- **BillingPeriodStart** — IMPLEMENTED
- **ChargePeriodEnd** — IMPLEMENTED
- **ChargePeriodStart** — IMPLEMENTED

### Column Group: Charge Origination (7)
- **InvoiceIssuerName** — IMPLEMENTED
- **ServiceProviderName** — P0 — NOT IMPLEMENTED
- **HostProviderName** — P0 — NOT IMPLEMENTED
- (+ 4 more) — P0-P2 — NOT IMPLEMENTED

---

*End of focus-md.md — authoritative blueprint for Focus Explorer evolution.*
