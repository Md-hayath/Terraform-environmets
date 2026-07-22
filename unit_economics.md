# Unit Economics Implementation — Complete Analysis & Roadmap

> Production-grade Unit Economics for the Digitomics InfraOps FinOps platform.
> Designed against the actual codebase, not a generic architecture.

---

## Table of Contents

1. [Current Architecture Trace](#1-current-architecture-trace)
2. [Current Cost Data Model](#2-current-cost-data-model)
3. [Current Virtual Tags](#3-current-virtual-tags)
4. [Current Focus Explorer](#4-current-focus-explorer)
5. [Current Unit Economics (Existing Implementation)](#5-current-unit-economics-existing-implementation)
6. [Market Reference Architectures](#6-market-reference-architectures)
7. [Unit Economics Domain Model for This Codebase](#7-unit-economics-domain-model-for-this-codebase)
8. [Architecture Mapping: What to Reuse vs Build New](#8-architecture-mapping-what-to-reuse-vs-build-new)
9. [Database Schema — Minimum Required Changes](#9-database-schema--minimum-required-changes)
10. [Cost Attribution Layer Design](#10-cost-attribution-layer-design)
11. [Business Metrics Design](#11-business-metrics-design)
12. [Unit Definition Design](#12-unit-definition-design)
13. [Calculation Engine Design](#13-calculation-engine-design)
14. [API Design](#14-api-design)
15. [Frontend Design](#15-frontend-design)
16. [Critical Architectural Decisions](#16-critical-architectural-decisions)
17. [Gap Analysis](#17-gap-analysis)
18. [Bugs and Issues Found](#18-bugs-and-issues-found)
19. [Implementation Roadmap](#19-implementation-roadmap)

---

## 1. Current Architecture Trace

### 1.1 System Architecture

```
┌──────────────────────────────────────────────────────────────── ─┐
│                    FRONTEND (Next.js 16)                         │
│  App Router │ Zustand Stores │ React Query │ Recharts            │
│                                                                  │
│  /dashboard/focus          → FOCUS Explorer (cost-explorer.tsx)  │
│  /dashboard/virtual-tags   → Virtual Tag Studio                  │
│  /dashboard/unit-economics → Unit Economics pages                │
│  /dashboard/allocation     → Showback/Chargeback                 │
│  /dashboard/costs/*        → Cost Hierarchy drill-down           │
│  /dashboard/overview       → Dashboard Grid (14 widgets)         │
└──────────────────────────┬────────────────────────────────────── ┘
                           │ apiFetch() (JWT + cookie auth)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    BACKEND (FastAPI)                              │
│                                                                  │
│  routes/focus_routes.py          → Focus Explorer endpoints      │
│  routes/allocation_routes.py     → Virtual Tags, Unit Econ,      │
│                                     Showback/Chargeback,         │
│                                     Optimization, Anomaly        │
│  routes/billing_routes.py        → Billing overview & sync       │
│                                                                  │
│  services/focus/                 → FOCUS normalization,          │
│                                     charge persistence, queries  │
│  services/allocation/            → Virtual tag engine            │
│  services/allocation_service.py  → Unit economics, showback,     │
│                                     chargeback, metric CRUD      │
│  services/billing_records_service.py → 20+ provider adapters     │
│  services/billing_sync_service.py    → Sync orchestration        │
└──────────────────────────┬───────────────────────────────────── ─┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              PostgreSQL (pgvector:pg16 + RLS)                   │
│                                                                 │
│  cost_charges        — FOCUS 1.2 line items (30+ columns)       │
│  billing_records     — per-tool/month aggregated snapshots      │
│  virtual_tags        — ordered rule sets (JSONB rules)          │
│  shared_cost_reallocations — even/manual%/telemetry strategies  │
│  cost_allocation_rules     — weighted allocation rules          │
│  business_metrics    — unit economics denominator inputs        │
│  cloud_connections   — provider credential store                │
│  tool_connections    — SaaS tool connection state               │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Complete Data Flow

```
Provider APIs (AWS CE, GCP BQ, Azure FOCUS, 20+ SaaS)
    ↓
billing_sync_service.py (orchestrator, sequential per provider)
    ↓
billing_records_service.py._extract_snapshot() (670-line provider switch)
    ↓
BillingRecord (deduplicated by provider_billing_key = tool:account:month)
    ↓
focus/ingest.py (provider-specific adapters)
    ↓
focus/schema.py.normalize_charge_row() (FOCUS 1.2 normalization)
    ↓
focus/charge_store.persist_cost_charges() (INSERT ON CONFLICT DO UPDATE)
    ↓
CostCharge table (FOCUS 1.2 line items, 30+ columns)
    ↓
┌──────────────────────────────────────────────────────┐
│                                                      │
│  ← FOCUS Explorer (summarize, timeseries, charges)   │
│  ← Virtual Tag Engine (rule evaluation, allocation)  │
│  ← Unit Economics (cost_per_unit = spend / quantity) │
│  ← Showback/Chargeback (allocation + statements)     │
│  ← Optimization (recommendations, savings)           │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 2. Current Cost Data Model

### 2.1 What CostCharge Contains (FOCUS 1.2)

Every `CostCharge` row in `cost_charges` table has these columns:

**Identity:**
- `id` (UUID PK)
- `tenant_id` (UUID FK, CASCADE)
- `connection_id` (UUID FK → cloud_connections, nullable)
- `provider_line_item_id` (String(512), unique per tenant+provider)
- `provider_name` (String(64): "aws", "gcp", "azure", "saas")

**Cost columns:**
- `billed_cost` (Numeric(18,6)) — amount as billed on invoice
- `effective_cost` (Numeric(18,6)) — amortized/effective cost
- `list_cost` (Numeric(18,6), nullable) — list price (Azure FOCUS)
- `contracted_cost` (Numeric(18,6), nullable) — contracted price (Azure FOCUS)
- `billing_currency` (String(16), default "USD")
- `x_billed_cost_usd` (Numeric(18,6), nullable) — USD equivalent
- `x_effective_cost_usd` (Numeric(18,6), nullable) — USD equivalent

**Time:**
- `billing_period_start`, `billing_period_end` (DateTime(tz))
- `charge_period_start`, `charge_period_end` (DateTime(tz))

**Dimensions:**
- `service_category` (String(128)) — Compute, Storage, Databases, etc.
- `service_name` (String(255)) — e.g. "Amazon EC2", "Cloud Storage"
- `service_subcategory` (String(128), nullable)
- `sub_account_id` (String(255)) — AWS account / GCP project / Azure subscription
- `sub_account_name` (String(255), nullable)
- `region_id` (String(128), nullable)
- `region_name` (String(128), nullable)
- `resource_id` (String(512), nullable) — instance/bucket ARN
- `resource_name` (String(512), nullable)
- `resource_type` (String(128), nullable)

**Classification:**
- `charge_category` (String(64)) — "Usage", "Tax", "Refund", "Credit", "Purchase"
- `charge_class` (String(64), nullable)
- `charge_description` (Text, nullable)

**Pricing/Usage:**
- `pricing_quantity` (Numeric(18,6), nullable)
- `pricing_unit` (String(64), nullable)
- `consumed_quantity` (Numeric(18,6), nullable)
- `consumed_unit` (String(64), nullable)

**SKU:**
- `sku_id` (String(255), nullable)
- `sku_price_id` (String(255), nullable)

**Commitment Discounts:**
- `commitment_discount_id` (String(255), nullable)
- `commitment_discount_type` (String(64), nullable)
- `commitment_discount_status` (String(64), nullable) — "Used" / "Unused"
- `commitment_discount_category` (String(64), nullable)

**Tags:**
- `tags` (JSONB) — provider-native resource tags (e.g. `{"Team": "Platform", "Env": "prod"}`)
- `x_provider_extensions` (JSONB) — provider-specific extra data

**Audit:**
- `created_at`, `updated_at` (DateTime(tz))

### 2.2 How billed_cost vs effective_cost Are Populated

| Provider | billed_cost source | effective_cost source | Notes |
|----------|-------------------|----------------------|-------|
| AWS CE | `UnblendedCost` | `AmortizedCost` | AmortizedCost includes RI/SP amortization |
| AWS CUR | `lineItem/UnblendedCost` or `BilledCost` | `lineItem/AmortizedCost` or `EffectiveCost` | Richer per-line data |
| GCP BQ | `cost` column | Same as billed_cost | GCP doesn't distinguish at line level |
| GCP Billing | `net_cost` or `cost` | Same as billed_cost | Same limitation |
| Azure FOCUS | `BilledCost` | `EffectiveCost` | Native FOCUS 1.2 columns |
| Azure Billing | `cost` or `amount` | Same as billed_cost | API-level, no distinction |
| SaaS | `BillingRecord.total_cost` | Same as billed_cost | No distinction |

### 2.3 What Is Available for Unit Economics

**Directly usable as cost numerator:**
- `effective_cost` — the primary field for unit economics (amortized)
- `billed_cost` — cash-basis alternative
- `list_cost`, `contracted_cost` — for pricing analysis (Azure only)
- `consumed_quantity`, `consumed_unit` — for usage-based unit economics

**Directly usable for attribution/dimension:**
- `provider_name` — which cloud/SaaS
- `service_category` — what type of service
- `service_name` — specific service
- `sub_account_id` — which account/project/subscription
- `region_name` — which region
- `resource_id`, `resource_name` — specific resource
- `resource_type` — resource classification
- `charge_category` — Usage, Tax, Refund, etc.
- `tags` (JSONB) — any provider tag (accessed via `tag.<key>` syntax)

**NOT available but would be useful:**
- No `invoice_id` link between CostCharge and BillingRecord
- `list_cost` and `contracted_cost` exist but are not aggregatable via `summarize_focus()`
- `consumed_quantity` exists but is not reliably populated by all providers
- `tags` cannot be used for filtering/grouping in FOCUS Explorer (only via Virtual Tags at app layer)

### 2.4 How Costs Are Aggregated

`charge_store.summarize_focus()`:
1. Groups by `(dimension, billing_currency)` to prevent cross-currency blending
2. Uses `func.coalesce(func.sum(cost_col), 0)` — supports `effective_cost` or `billed_cost` via `cost_field` param
3. Returns top 50 groups; remainder collapsed into "Other"
4. Reports `share_percent` per group
5. Multi-currency: `total_cost` only set when single currency; otherwise `total_by_currency`

### 2.5 Duplicate Prevention

- `cost_charges`: Unique constraint on `(tenant_id, provider_name, provider_line_item_id)`
- `billing_records`: Unique constraint on `(tenant_id, provider_billing_key)`
- Both use `INSERT ... ON CONFLICT DO UPDATE`
- `provider_line_item_id` is either natural (from provider) or SHA-1 hash of key fields

---

## 3. Current Virtual Tags

### 3.1 Data Model

```python
# VirtualTag (virtual_tags table)
{
    id: UUID,
    tenant_id: UUID,
    name: str (unique per tenant),
    description: str,
    rules: JSONB,  # Array of rule dicts
    untagged_label: str (default "untagged"),
    retroactive: bool (default True, stored but NOT enforced),
    enabled: bool
}

# Rules JSON structure:
[
    {
        "field": "tag.Team" | "service_name" | "region_name" | ...,
        "op": "eq" | "contains" | "regex" | "in" | "exists",
        "value": "string" | ["list", "of", "values"],
        "assign": "Platform"  # the label to assign on match
    },
    ...
]
```

### 3.2 Available Fields for Rules

12 FOCUS-native fields:
```
service_name, service_category, provider_name,
region_name, region_id, sub_account_id, sub_account_name,
resource_id, resource_name, resource_type,
sku_id, charge_category
```

Plus any provider tag via `tag.<key>` syntax (e.g. `tag.Team`, `tag.Environment`).

Tag keys are dynamically discovered from the actual `tags` JSONB data (up to 500 charges sampled).

### 3.3 How Rules Are Evaluated

**First-match-wins funnel:**
1. Rules are evaluated in order (array position = priority)
2. For each charge, iterate rules top-to-bottom
3. First matching rule's `assign` value becomes the charge's label
4. If no rule matches, the charge gets `untagged_label`

**Operator behavior:**
- `eq`: exact string equality
- `contains`: case-insensitive substring
- `regex`: Python `re.search`
- `in`: set membership
- `exists`: non-null and non-empty

### 3.4 How Allocation Works

```python
# virtual_tag_engine.allocate_charges():
1. Load VirtualTag from DB (by id, name, or first enabled)
2. Load ALL CostCharge rows for tenant (optionally filtered by period/provider)
3. For each charge:
   - Evaluate virtual tag rules → get assigned label
   - Accumulate: buckets[label] += charge.effective_cost
4. Load SharedCostReallocation rules for this tag
5. Apply each reallocation sequentially:
   - even: divide source equally among targets
   - manual_percent: amount * (percent/100)
   - telemetry_weight: amount * (weight / total_weight)
6. Compute coverage = (total - untagged) / total * 100
7. Return allocations sorted by amount descending
```

### 3.5 Can Virtual Tags Be the Attribution Layer for Unit Economics?

**Yes, they already are.**

The linkage exists:
- `BusinessMetric.extra.virtual_tag_id` points to a VirtualTag
- `BusinessMetric.extra.cost_scope` specifies which allocation bucket value
- `build_unit_economics()` calls `allocate_charges()` to get the scoped spend
- The scoped spend is divided by `BusinessMetric.quantity` to get `cost_per_unit`

**What works:**
- Any FOCUS dimension or provider tag can be the attribution dimension
- Shared cost reallocation redistributes untagged/shared costs
- Multiple metrics can reference different virtual tags
- Segments show per-bucket cost_per_unit breakdown

**What is missing for enterprise-grade:**
- No multi-dimensional cross-tabulation (e.g. team AND environment)
- No hierarchical allocation (BU → Team → Squad)
- No time-series allocation (monthly trend of per-team spend)
- No materialized allocation results (re-evaluated from scratch every time)
- All charges loaded into memory (OOM risk at scale)
- No Explorer drill-down by virtual tag value

---

## 4. Current Focus Explorer

### 4.1 Dimensions

**Groupable (7):** service_category, service_name, provider_name, region_name, sub_account_id, charge_category, resource_type

**Filterable (same 7):** Each backed by `FilterCombobox` with real values from `get_focus_dimension_values()`

**Sortable (10):** charge_period_start, billed_cost, effective_cost, service_name, resource_name, provider_name, region_name, sub_account_id, charge_category, resource_type

### 4.2 How It Works

1. User selects group_by dimension, cost_field (effective/billed), date range, filters
2. Frontend calls `GET /api/v1/finops/focus/summary?group_by=X&cost_field=Y`
3. Backend `summarize_focus()` groups charges by dimension, returns top 50 + Other
4. Frontend renders summary table + timeseries chart + line items table
5. Clicking a summary row drills down into line items filtered by that dimension value

### 4.3 What Is Missing for Unit Economics

- No way to overlay unit economics (cost_per_unit) on the Explorer
- No way to filter by virtual tag value in Explorer
- No way to see "this cost is attributed to customer X" in Explorer
- No connection between Explorer dimensions and business entities
- The "View in Explorer" link from allocation page deliberately does NOT pass allocation name as filter (known bug workaround)

---

## 5. Current Unit Economics (Existing Implementation)

### 5.1 What Exists Today

**Backend: `build_unit_economics()` in `allocation_service.py:532-657`**

The core formula:
```
cost_per_unit = metric_spend / quantity
where:
  metric_spend = SUM(CostCharge.effective_cost) for the metric's period
                 (optionally scoped to a virtual tag bucket)
  quantity     = BusinessMetric.quantity (user-provided)
```

**Execution trace:**
1. Get total spend from FOCUS (or fallback to BillingRecord overview)
2. Load all BusinessMetric rows for tenant
3. For each metric:
   - If `extra.virtual_tag_id` is set: run `allocate_charges()`, extract the `cost_scope` bucket
   - Otherwise: use total FOCUS spend
   - `cost_per_unit = metric_spend / quantity`
   - Compute trend: compare with prior period's same metric_key
   - Compute gross_margin if revenue_metric_key provided
   - Compute target variance if `extra.target_cost_per_unit` set
4. If `dimension`/`virtual_tag_id` passed: compute segments (per-allocation-bucket cost_per_unit)

**Frontend: 3 pages**
- `/dashboard/unit-economics` — overview with KPI cards + metrics table + segments grid
- `/dashboard/unit-economics/configure` — add/edit business metrics (manual entry)
- `/dashboard/unit-economics/[metricId]` — metric detail with target vs actual, gross margin, segments

### 5.2 What Works

- Basic cost_per_unit calculation
- Virtual tag scoping (cost per team, per product, per customer)
- Trend analysis (period-over-period %)
- Gross margin (if revenue metric provided)
- Target variance (if target set)
- Segment breakdown by virtual tag dimension

### 5.3 What Is Missing or Broken

| Issue | Location | Impact |
|-------|----------|--------|
| **Revenue is treated as quantity, not monetary value** | `allocation_service.py:593-596` | `revenue_by_period` uses `metric.quantity` (a count) as "revenue" — this is wrong for gross margin calculation |
| **All metrics use the same denominator for segments** | `allocation_service.py:630-635` | Segments all divide by the first non-revenue metric's quantity — no per-segment metric support |
| **No time-series unit economics** | `build_unit_economics()` | Returns a single aggregated result, not monthly/daily trend |
| **No reusable unit definitions** | `BusinessMetric` model | Each metric is a standalone row — no concept of "Cost per Active User" as a reusable definition |
| **No metric sources/automation** | Configure page | Only manual entry — no API, CSV, or integration ingestion |
| **No historical snapshots** | No `unit_economics_calculations` table | Every request recalculates from scratch |
| **No explainability** | No explanation field | User cannot see "why was this cost attributed here?" |
| **No drill-down from unit economics to Explorer** | `[metricId]/page.tsx:154` | Link goes to Explorer with period only — no attribution dimension filter |
| **BusinessMetric has no `revenue_amount` field** | `BusinessMetric` model | Revenue is modeled as `quantity` which is semantically wrong |
| **No multi-dimensional unit economics** | Single virtual tag per metric | Cannot see "cost per customer per product" |
| **No cost source selection** | Hardcoded to FOCUS total | Cannot specify "only AWS costs" or "only Compute costs" |
| **`_focus_total_spend` ignores period bounds on CostCharge** | `allocation_service.py:404-407` | Uses `charge_period_start >= period_start AND charge_period_end <= period_end` — correct but different from Explorer overlap semantics |
| **No caching of allocation results** | `allocate_charges()` called per request | Re-evaluates all charges from scratch every time |

---

## 6. Market Reference Architectures

### 6.1 CloudZero

**Core concept:** Three-layer Dimension model:
1. **Grouping Dimension** — organizes known spend via metadata rules (tags, accounts, K8s namespaces)
2. **Allocation Dimension** — splits shared/unallocated costs using telemetry
3. **Combined Dimension** — final view combining both

**Key innovation:** `CostFormation` YAML rules — allocation logic as code, version-controlled, retroactive.

**Business metrics:** Telemetry Streams — time-series data ingested via API from observability tools (Grafana, Prometheus, CloudWatch), databases, or CSV. Each stream has: timestamp, element name, allocation value, optional filter columns.

**Revenue:** Cloud Efficiency Rate (CER) = `(Revenue - Cloud Costs) / Revenue`. Enrichable per dimension.

**Pattern:** `Raw Spend → Grouping Dimension → Allocation Dimension → Combined Dimension → Telemetry Join → Unit Cost`

### 6.2 Finout

**Core concept:** Virtual Tags (patented) — rule-based allocation on MegaBill (FOCUS-normalized data). This codebase's Virtual Tag engine is directly inspired by Finout.

**Business metrics:** Telemetry Integrations — S3, Datadog, Snowflake, Prometheus, GCP BQ. Synced daily.

**Shared costs:** Telemetry-based (proportional by usage) or customized (even/manual percent).

**Pattern:** `Multi-Cloud Spend → MegaBill → Virtual Tags → Shared Cost Reallocation → Unit Economics Widget`

### 6.3 Vantage

**Core concept:** Cost Reports with Business Metrics and Segments. Four calculation types:
1. **Unit Cost** — cost / metric
2. **Usage Unit Cost** — cost / usage (from cloud usage data)
3. **Gross Margin** — (revenue - cost) / revenue
4. **Raw Value** — just the metric value

**Business metrics:** Imported from CloudWatch, Datadog, Metronome, Snowflake, CSV, or Vantage API. **Labels** enable sub-allocation (e.g. metric `requests` with labels `app1`, `app2`).

**Hierarchies:** Segments with bottom-up allocation — children claim costs, roll up to parents. Unallocated costs tracked.

**Pattern:** `Cloud Spend → Cost Reports (filters) → Virtual Tags → Segments → Business Metrics → Unit Cost / Gross Margin`

### 6.4 Amnic

**Core concept:** Self-serve cost dimensions (teams, environments, customers, products) with Meters for business metric ingestion.

**Key innovation:** Radix context graph — links every dollar to the change that drove it. Root cause tracing.

**Pattern:** `Multi-Cloud Spend → Cost Dimensions → Cost Buckets → Meters → Rule-based Unit Cost`

### 6.5 FOCUS Specification (v1.2 / v1.3)

**v1.3 introduces:**
- `AllocatedResourceId`, `AllocatedResourceName` — what cost is allocated to
- `AllocatedMethodId`, `AllocatedMethodDetails` (JSON with `AllocatedRatio`, `UsageUnit`, `UsageQuantity`) — how it was allocated
- `ContractApplied` JSON — links to Contract Commitment dataset

**Key insight:** FOCUS provides the cost data model but NOT the business metric model. Platforms must combine FOCUS data with external metrics.

### 6.6 Cross-Platform Pattern Summary

| Pattern | Best Implementation | Key Insight |
|---------|-------------------|-------------|
| Allocation as separate layer | CloudZero (3-dimension) | Grouping → Allocation → Combined. Inspectable and composable. |
| Virtual tagging | Finout (patented) | Rule-based allocation that doesn't touch infrastructure. Priority-ordered. |
| Business metric ingestion | Vantage (labels) | Labels enable same metric to serve multiple scopes. |
| Shared cost strategies | CloudZero + Finout | Even, proportional, telemetry-based, fixed-rate. |
| Hierarchical allocation | Vantage (Segments) | Bottom-up: children claim costs, roll up to parents. |
| Revenue connection | CloudZero (CER) | (Revenue - Cost) / Revenue. Per-dimension enrichment. |
| Auditability | FOCUS 1.3 | AllocatedMethodDetails JSON — methodology, ratio, usage unit. |

---

## 7. Unit Economics Domain Model for This Codebase

### 7.1 The Core Formula

```
Unit Economics = Allocated Cost / Business Metric

Where:
  Allocated Cost = SUM(CostCharge.effective_cost)
                   filtered by: period, provider, service, virtual_tag value
  Business Metric = user-defined quantity (customers, requests, transactions, etc.)
  Result = cost_per_unit (e.g. $12.50/customer)
```

### 7.2 What "Unit" Means

A "unit" is any business outcome that infrastructure cost serves. It is NOT hardcoded — users define it.

Examples:
```
Cost per Customer     = SUM(aws+gcp+azure costs for "customer_123") / 10,000 customers
Cost per Transaction  = SUM(compute+storage costs for "payments") / 500,000 transactions
Cost per API Request  = SUM(infrastructure costs for "api") / 2,000,000 requests
Cost per Active User  = SUM(all costs for "analytics") / 8,500 active users
Cost per AI Token     = SUM(AI service costs) / 50,000,000 tokens
Cost per GB Processed = SUM(storage costs) / 1,000,000 GB
```

### 7.3 Generic Data Model

```
UnitDefinition
├── name: "Cost per Active User"
├── numerator:
│   ├── cost_source: "all" | "provider:X" | "service:Y" | "virtual_tag:Z"
│   └── cost_field: "effective_cost" | "billed_cost"
├── denominator:
│   ├── metric_key: "active_users"
│   └── source: "manual" | "api" | "integration"
├── attribution:
│   └── virtual_tag_id → virtual_tag_value (optional scope)
├── time_period: monthly | weekly | daily | custom
└── → UnitCalculationResult
        ├── cost_per_unit: 12.50
        ├── total_cost: 125,000
        ├── total_quantity: 10,000
        ├── trend_percent: -5.2%
        ├── gross_margin_percent: 82.5%
        └── explanation: "..."
```

### 7.4 Cost Source Specification

The cost source should support these scopes:

| Scope | Example | How It Maps |
|-------|---------|-------------|
| All costs | `cost_source: "all"` | `SUM(effective_cost) WHERE tenant_id = X` |
| By provider | `cost_source: "provider:aws"` | `SUM(effective_cost) WHERE provider_name = 'aws'` |
| By service category | `cost_source: "service:Compute"` | `SUM(effective_cost) WHERE service_category = 'Compute'` |
| By service name | `cost_source: "service:Amazon EC2"` | `SUM(effective_cost) WHERE service_name = 'Amazon EC2'` |
| By virtual tag | `cost_source: "vtag:team:Platform"` | `allocate_charges()` → extract "Platform" bucket |
| By account | `cost_source: "account:123456789"` | `SUM(effective_cost) WHERE sub_account_id = '123456789'` |
| By charge category | `cost_source: "charge_category:Usage"` | `SUM(effective_cost) WHERE charge_category = 'Usage'` |

### 7.5 Business Metric Specification

```
BusinessMetric
├── name: "Monthly Active Users"
├── metric_key: "active_users"       (unique identifier)
├── unit_label: "users"              (display unit)
├── period_start: 2026-01-01
├── period_end: 2026-01-31
├── quantity: 8500                    (the denominator value)
├── revenue_amount: 100000.00        (optional, for margin calculation)
├── source: "manual" | "api" | "csv" | "integration"
├── source_config: {}                (API endpoint, schedule, etc.)
├── dimensions: {}                   (optional labels for sub-allocation)
└── extra: {
      virtual_tag_id: "uuid",
      cost_scope: "Platform",
      target_cost_per_unit: 15.00
    }
```

### 7.6 Calculation Result Specification

```
UnitCalculationResult
├── unit_definition_id
├── period_start, period_end
├── numerator:
│   ├── total_cost: 125000.00
│   ├── cost_field: "effective_cost"
│   ├── cost_source: "all"
│   ├── currency: "USD"
│   └── line_item_count: 45000
├── denominator:
│   ├── metric_key: "active_users"
│   ├── quantity: 8500
│   └── unit_label: "users"
├── result:
│   ├── cost_per_unit: 14.71
│   ├── trend_percent: -5.2
│   ├── gross_margin_percent: 82.5
│   ├── target_cost_per_unit: 15.00
│   ├── variance_amount: -0.29
│   └── variance_percent: -1.9
├── attribution:
│   ├── virtual_tag_id: "uuid"
│   ├── virtual_tag_name: "team"
│   └── cost_scope: "Platform"
├── explanation: {
│   ├── cost_source_description: "All FOCUS charges, effective_cost",
│   ├── period: "January 2026",
│   ├── attribution_method: "Virtual Tag 'team', value 'Platform'",
│   └── calculation: "$125,000 / 8,500 users = $14.71/user"
│   }
├── segments: [  # optional, per-virtual-tag-value breakdown
│   ├── { segment: "Platform", cost: 45000, cost_per_unit: 15.00 },
│   ├── { segment: "Data", cost: 35000, cost_per_unit: 14.00 },
│   └── { segment: "Other", cost: 45000, cost_per_unit: 15.00 }
│   ]
└── drill_down:
    └── focus_explorer_url: "/dashboard/focus?period_start=...&provider_name=..."
```

---

## 8. Architecture Mapping: What to Reuse vs Build New

| Unit Economics Concept | Existing Codebase Concept | Action | Why |
|----------------------|--------------------------|--------|-----|
| **Cost Source** | `CostCharge.effective_cost` + `_focus_total_spend()` | **Reuse** | Already queries FOCUS charges with period/provider filters |
| **Cost Dimensions** | `CostCharge` columns (12+ FOCUS fields) + `tags` JSONB | **Reuse** | All attribution dimensions already exist |
| **Cost Attribution** | `VirtualTag` + `allocate_charges()` | **Extend** | Core attribution mechanism works; needs multi-dimensional support |
| **Shared Cost Allocation** | `SharedCostReallocation` + `_apply_reallocation()` | **Reuse** | Three strategies already implemented |
| **Business Metric** | `BusinessMetric` table | **Extend** | Needs `revenue_amount`, `source`, `source_config`, `dimensions` columns |
| **Unit Definition** | Does not exist | **New** | No reusable definition concept — each metric is standalone |
| **Calculation Engine** | `build_unit_economics()` | **Extend** | Core formula works; needs explainability, caching, time-series |
| **Results Storage** | Does not exist | **New** | No historical calculation persistence |
| **Cost Exploration** | `summarize_focus()` + FOCUS Explorer | **Extend** | Needs unit economics overlay and virtual tag drill-down |
| **Showback/Chargeback** | `build_showback()` / `build_chargeback()` | **Reuse** | Already works with virtual tags |
| **Frontend Overview** | `unit-economics/page.tsx` | **Extend** | Needs trend charts, definition management |
| **Frontend Configure** | `unit-economics/configure/page.tsx` | **Extend** | Needs metric sources, API ingestion |
| **Frontend Detail** | `unit-economics/[metricId]/page.tsx` | **Extend** | Needs explainability, drill-down to Explorer |

### Key Insight: The Existing Architecture Is 70% There

The system already has:
- FOCUS 1.2 cost data with 30+ columns
- Virtual Tag engine that maps costs to business entities
- BusinessMetric table for denominator values
- `build_unit_economics()` with the core formula
- Showback/Chargeback with allocation
- FOCUS Explorer for cost exploration

What is missing:
- **Unit Definition** concept (reusable formula definitions)
- **Metric sources** (API/CSV/integration ingestion)
- **Historical calculation snapshots**
- **Explainability** (why was this cost attributed here?)
- **Time-series unit economics** (monthly trend)
- **Multi-dimensional attribution** (cost per customer per product)
- **Cost source selection** (which costs to include)
- **Revenue as monetary value** (not just quantity)

---

## 9. Database Schema — Minimum Required Changes

### 9.1 Extend `business_metrics` Table

Add columns to the existing table rather than creating a new one:

```sql
ALTER TABLE business_metrics ADD COLUMN revenue_amount NUMERIC(18,6);
ALTER TABLE business_metrics ADD COLUMN source VARCHAR(32) DEFAULT 'manual';
ALTER TABLE business_metrics ADD COLUMN source_config JSONB DEFAULT '{}';
ALTER TABLE business_metrics ADD COLUMN dimensions JSONB DEFAULT '{}';
```

**Why:** Revenue is currently modeled as `quantity` (a count), which is semantically wrong. `revenue_amount` stores the actual monetary revenue for margin calculations. `source` and `source_config` enable automated metric ingestion. `dimensions` enable label-based sub-allocation.

### 9.2 New Table: `unit_definitions`

Purpose: Reusable unit economics definitions that users create once and view over time.

```sql
CREATE TABLE unit_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants.id ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT DEFAULT '',

    -- Cost source (numerator)
    cost_source VARCHAR(64) NOT NULL DEFAULT 'all',
    -- 'all' = tenant-wide FOCUS total
    -- 'provider:aws' = specific provider
    -- 'service:Compute' = service category
    -- 'vtag:<tag_id>:<value>' = virtual tag scoped
    cost_field VARCHAR(32) NOT NULL DEFAULT 'effective_cost',
    -- 'effective_cost' or 'billed_cost'

    -- Business metric (denominator)
    metric_key VARCHAR(64) NOT NULL,

    -- Attribution (optional scope)
    virtual_tag_id UUID REFERENCES virtual_tags(id) ON DELETE SET NULL,
    cost_scope VARCHAR(255),

    -- Settings
    target_cost_per_unit NUMERIC(18,6),
    include_charge_categories JSONB DEFAULT '["Usage"]',

    -- Metadata
    enabled BOOLEAN DEFAULT TRUE,
    created_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE(tenant_id, name)
);

CREATE INDEX ix_unit_definitions_tenant ON unit_definitions(tenant_id);
```

**Why this is needed:** Currently each `BusinessMetric` row is a standalone data point. A `UnitDefinition` is a reusable formula: "Cost per Active User = (all effective_cost) / (active_users metric)". Users create it once, and it calculates automatically for each period.

### 9.3 New Table: `unit_calculation_results`

Purpose: Persist calculation results for historical comparison and audit.

```sql
CREATE TABLE unit_calculation_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants.id ON DELETE CASCADE,
    unit_definition_id UUID NOT NULL REFERENCES unit_definitions(id) ON DELETE CASCADE,

    -- Period
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,

    -- Numerator (cost)
    total_cost NUMERIC(18,6) NOT NULL,
    cost_field VARCHAR(32) NOT NULL,
    currency VARCHAR(16) NOT NULL DEFAULT 'USD',
    line_item_count INTEGER DEFAULT 0,

    -- Denominator (metric)
    metric_key VARCHAR(64) NOT NULL,
    metric_quantity NUMERIC(18,4) NOT NULL,
    unit_label VARCHAR(64) NOT NULL,

    -- Result
    cost_per_unit NUMERIC(18,6),
    trend_percent NUMERIC(8,4),
    gross_margin_percent NUMERIC(8,4),
    target_cost_per_unit NUMERIC(18,6),
    variance_amount NUMERIC(18,6),
    variance_percent NUMERIC(8,4),

    -- Attribution snapshot
    virtual_tag_name VARCHAR(128),
    cost_scope VARCHAR(255),

    -- Explainability
    explanation JSONB DEFAULT '{}',

    -- Segments (per-virtual-tag-value breakdown)
    segments JSONB DEFAULT '[]',

    -- Audit
    calculated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE(tenant_id, unit_definition_id, period_start, period_end)
);

CREATE INDEX ix_unit_results_tenant_period ON unit_calculation_results(tenant_id, period_start, period_end);
CREATE INDEX ix_unit_results_definition ON unit_calculation_results(unit_definition_id, period_start);
```

**Why:** Without this, every page load recalculates from scratch. With this, historical comparisons are instant and every calculation is auditable.

### 9.4 New Table: `metric_data_sources`

Purpose: Define automated metric ingestion sources.

```sql
CREATE TABLE metric_data_sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants.id ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    metric_key VARCHAR(64) NOT NULL,
    source_type VARCHAR(32) NOT NULL,
    -- 'manual' | 'csv_upload' | 'api_endpoint' | 'webhook' | 'integration'

    config JSONB NOT NULL DEFAULT '{}',
    -- For api_endpoint: { url, auth_type, auth_config, headers, schedule_cron }
    -- For csv_upload: { last_upload_filename, last_upload_at }
    -- For integration: { integration_type, connection_id }

    is_active BOOLEAN DEFAULT TRUE,
    last_synced_at TIMESTAMPTZ,
    last_sync_error TEXT,

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE(tenant_id, name)
);

CREATE INDEX ix_metric_sources_tenant ON metric_data_sources(tenant_id);
```

**Why:** Currently business metrics can only be entered manually via a form. This table enables automated ingestion from APIs, CSVs, and integrations.

### 9.5 Entity Relationship

```
tenants
  ├── business_metrics (extended with revenue_amount, source, dimensions)
  ├── unit_definitions (NEW)
  │     ├── references virtual_tags (attribution)
  │     └── references metric_key (links to business_metrics)
  ├── unit_calculation_results (NEW)
  │     └── references unit_definitions
  ├── metric_data_sources (NEW)
  │     └── references metric_key
  ├── virtual_tags (existing)
  ├── cost_charges (existing, FOCUS 1.2)
  └── shared_cost_reallocations (existing)
```

### 9.6 Historical Data Strategy

- `unit_calculation_results` stores one row per definition per period
- Recalculation: on-demand via API or scheduled via background worker
- Retention: keep all results (small table — one row per definition per month)
- Partitioning: not needed initially; add by `period_start` if table exceeds 1M rows

---

## 10. Cost Attribution Layer Design

### 10.1 How Attribution Should Work

Attribution should be **dynamic** (evaluated at query time), not materialized.

**Why dynamic:**
- Virtual Tags already evaluate dynamically — changing a rule retroactively affects all historical costs
- Materializing would require re-materialization on every rule change
- The existing `allocate_charges()` function already does this efficiently for moderate data volumes

**When to add materialization (future optimization):**
- When a tenant has >1M charges and allocation takes >5s
- Cache results in `unit_calculation_results.segments` column
- Refresh on schedule or on-demand

### 10.2 Attribution Flow

```
CostCharge rows
    ↓
Virtual Tag Rules (first-match funnel)
    ↓
Allocation Buckets: {label: Decimal(total_cost)}
    ↓
Shared Cost Reallocation (even/manual%/telemetry)
    ↓
Final Allocations: [{value, amount, share_percent}]
    ↓
Cost Source Filter (provider, service, charge_category)
    ↓
Filtered Allocations
    ↓
Unit Definition (cost_source + metric_key + period)
    ↓
Numerator (total_cost from filtered allocations)
÷
Denominator (BusinessMetric.quantity)
=
cost_per_unit
```

### 10.3 Cost Source Resolution

The `cost_source` field on `UnitDefinition` determines which costs flow into the numerator:

| cost_source | SQL/Logic |
|-------------|-----------|
| `"all"` | `SUM(effective_cost) WHERE tenant_id = X AND charge_period OVERLAPS period` |
| `"provider:aws"` | `...AND provider_name = 'aws'` |
| `"service:Compute"` | `...AND service_category = 'Compute'` |
| `"service:Amazon EC2"` | `...AND service_name = 'Amazon EC2'` |
| `"account:123456"` | `...AND sub_account_id = '123456'` |
| `"charge_category:Usage"` | `...AND charge_category = 'Usage'` |
| `"vtag:<tag_id>:<value>"` | `allocate_charges(tag_id)` → extract bucket `<value>` |

This is implemented as a query modifier that adds WHERE clauses to the CostCharge query, or delegates to `allocate_charges()` for virtual tag scopes.

### 10.4 Explainability

Every calculation result stores an `explanation` JSONB:

```json
{
  "cost_source": {
    "type": "all",
    "description": "All FOCUS charges, effective_cost field",
    "filters_applied": [],
    "total_cost": 125000.00,
    "line_items": 45000
  },
  "attribution": {
    "method": "virtual_tag",
    "virtual_tag_name": "team",
    "cost_scope": "Platform",
    "coverage_percent": 87.5
  },
  "denominator": {
    "metric_key": "active_users",
    "quantity": 8500,
    "unit_label": "users",
    "source": "manual",
    "period": "2026-01-01 to 2026-01-31"
  },
  "calculation": "$125,000.00 / 8,500 users = $14.71/user",
  "trend": {
    "previous_period": "2025-12",
    "previous_cost_per_unit": 15.52,
    "change_percent": -5.2
  }
}
```

---

## 11. Business Metrics Design

### 11.1 Metric Schema

Extend the existing `BusinessMetric` model:

```python
class BusinessMetric(Base):
    # Existing columns (keep):
    id, tenant_id, name, metric_key, unit_label,
    period_start, period_end, quantity, extra, created_at, updated_at

    # New columns:
    revenue_amount: Numeric(18,6)      # monetary revenue (not count)
    source: String(32)                 # 'manual', 'api', 'csv', 'integration'
    source_config: JSONB               # ingestion configuration
    dimensions: JSONB                  # labels for sub-allocation
```

### 11.2 Metric Dimensions

The `dimensions` JSONB enables label-based sub-allocation (inspired by Vantage):

```json
{
  "customer_id": "cust_123",
  "product": "analytics",
  "region": "us-east-1"
}
```

This allows the same `active_users` metric to be broken down by customer, product, or region when used in unit economics.

### 11.3 Metric Sources

| Source Type | Config Shape | Ingestion Method |
|-------------|-------------|-----------------|
| `manual` | `{}` | User enters quantity via UI |
| `csv_upload` | `{ last_filename, last_at }` | User uploads CSV, parsed server-side |
| `api_endpoint` | `{ url, auth_type, auth_config, headers, schedule_cron }` | Background worker polls endpoint |
| `webhook` | `{ secret, endpoint }` | External system POSTs metric values |
| `integration` | `{ integration_type, connection_id }` | Existing integration adapter |

### 11.4 Metric Validation

- `quantity` must be >= 0
- `period_end` must be >= `period_start`
- `metric_key` must be unique per tenant per period
- `revenue_amount` must be >= 0 if provided
- `source_config` must be valid JSON for the given source_type

---

## 12. Unit Definition Design

### 12.1 What a Unit Definition Is

A reusable formula that users create once and view over time:

```
Name: Cost per Active User
Cost Source: All FOCUS charges (effective_cost)
Business Metric: active_users
Attribution: Virtual Tag "team", scope "Platform"
Target: $15.00/user
Period: Monthly
```

### 12.2 How Users Create One

1. Navigate to Unit Economics → "Create Unit Definition"
2. Name: "Cost per Active User"
3. Cost Source: Select from dropdown (All, By Provider, By Service, By Virtual Tag)
4. Business Metric: Select existing metric key or create new
5. Attribution: Optionally select virtual tag + scope
6. Target: Optional target cost_per_unit
7. Save → system calculates for current and historical periods

### 12.3 How It Differs from BusinessMetric

| Concept | BusinessMetric | UnitDefinition |
|---------|---------------|----------------|
| What it is | A data point (quantity for a period) | A formula (how to calculate) |
| Contains | quantity, period, metric_key | cost_source, metric_key, attribution |
| Relationship | Multiple per metric_key (one per period) | One per formula, references metric_key |
| Purpose | Store denominator values | Define calculation logic |

---

## 13. Calculation Engine Design

### 13.1 Core Algorithm

```python
async def calculate_unit_economics(
    db, tenant_id, unit_definition_id, period_start, period_end
) -> UnitCalculationResult:

    # 1. Load the unit definition
    unit_def = await load_unit_definition(db, unit_definition_id)

    # 2. Resolve numerator (cost)
    cost = await resolve_cost_source(
        db, tenant_id,
        cost_source=unit_def.cost_source,
        cost_field=unit_def.cost_field,
        period_start=period_start,
        period_end=period_end,
    )

    # 3. Resolve denominator (metric)
    metric = await resolve_business_metric(
        db, tenant_id,
        metric_key=unit_def.metric_key,
        period_start=period_start,
        period_end=period_end,
    )

    # 4. Calculate
    cost_per_unit = cost.total_cost / metric.quantity if metric.quantity > 0 else None

    # 5. Calculate trend
    prev_result = await get_previous_period_result(db, unit_def.id, period_end)
    trend = calculate_trend(cost_per_unit, prev_result.cost_per_unit) if prev_result else None

    # 6. Calculate margin
    margin = calculate_gross_margin(metric.revenue_amount, cost.total_cost)

    # 7. Calculate variance
    variance = calculate_variance(cost_per_unit, unit_def.target_cost_per_unit)

    # 8. Generate segments (optional)
    segments = None
    if unit_def.virtual_tag_id:
        segments = await generate_segments(db, tenant_id, unit_def, period_start, period_end)

    # 9. Build explanation
    explanation = build_explanation(cost, metric, unit_def, cost_per_unit, trend, margin, variance)

    # 10. Persist result
    result = await persist_result(db, tenant_id, unit_def, period_start, period_end,
                                   cost, metric, cost_per_unit, trend, margin, variance,
                                   segments, explanation)

    return result
```

### 13.2 Cost Source Resolution

```python
async def resolve_cost_source(db, tenant_id, cost_source, cost_field, period_start, period_end):
    """Resolve the numerator for unit economics calculation."""

    if cost_source == "all":
        # Direct query on CostCharge
        total, currency, count = await _focus_total_spend_with_count(
            db, tenant_id, period_start, period_end, cost_field
        )
        return CostResolution(total=total, currency=currency, line_items=count,
                              source_type="all", description="All FOCUS charges")

    elif cost_source.startswith("provider:"):
        provider = cost_source.split(":", 1)[1]
        total, currency, count = await _focus_total_spend_with_count(
            db, tenant_id, period_start, period_end, cost_field, provider_name=provider
        )
        return CostResolution(total=total, currency=currency, line_items=count,
                              source_type="provider", description=f"Provider: {provider}")

    elif cost_source.startswith("service:"):
        service = cost_source.split(":", 1)[1]
        total, currency, count = await _focus_total_spend_with_count(
            db, tenant_id, period_start, period_end, cost_field, service_category=service
        )
        return CostResolution(total=total, currency=currency, line_items=count,
                              source_type="service", description=f"Service: {service}")

    elif cost_source.startswith("vtag:"):
        # Parse vtag:<tag_id>:<value>
        parts = cost_source.split(":", 2)
        tag_id = parts[1]
        scope = parts[2] if len(parts) > 2 else None
        allocation = await allocate_charges(db, tenant_id=tenant_id,
                                            virtual_tag_id=tag_id,
                                            period_start=period_start,
                                            period_end=period_end)
        total = resolve_scoped_spend_from_allocation(allocation, scope)
        return CostResolution(total=total, currency=allocation["currency"],
                              line_items=allocation["line_count"],
                              source_type="vtag",
                              description=f"Virtual Tag: {allocation['virtual_tag_name']} = {scope}")

    elif cost_source.startswith("account:"):
        account = cost_source.split(":", 1)[1]
        total, currency, count = await _focus_total_spend_with_count(
            db, tenant_id, period_start, period_end, cost_field, sub_account_id=account
        )
        return CostResolution(total=total, currency=currency, line_items=count,
                              source_type="account", description=f"Account: {account}")

    else:
        # Default: all costs
        total, currency, count = await _focus_total_spend_with_count(
            db, tenant_id, period_start, period_end, cost_field
        )
        return CostResolution(total=total, currency=currency, line_items=count,
                              source_type="all", description="All FOCUS charges")
```

### 13.3 Handling Edge Cases

| Edge Case | Handling |
|-----------|----------|
| Zero denominator | `cost_per_unit = None`, display "—" in UI |
| Missing metric for period | Look for closest prior period; if none, result is None |
| Multiple currencies | Use `x_effective_cost_usd` for USD normalization; store `currency: "USD"` |
| Late-arriving data | Recalculation on-demand; stale results marked with `calculated_at` |
| Cost corrections | Re-ingestion updates CostCharge via UPSERT; recalculation picks up changes |
| Partial period | Use actual charge_period bounds, not calendar month |
| No FOCUS data | Fallback to BillingRecord total (existing behavior) |
| Shared cost reallocation | Applied before numerator resolution for vtag sources |

### 13.4 Trend Calculation

```python
def calculate_trend(current_cpu, previous_cpu):
    """Calculate period-over-period change in cost_per_unit."""
    if previous_cpu is None or previous_cpu == 0:
        return None
    return ((current_cpu - previous_cpu) / previous_cpu) * 100
```

### 13.5 Gross Margin Calculation

```python
def calculate_gross_margin(revenue_amount, cost):
    """Calculate gross margin percentage."""
    if revenue_amount is None or revenue_amount <= 0:
        return None
    return ((revenue_amount - cost) / revenue_amount) * 100
```

**Important:** `revenue_amount` must be a monetary value, not a count. The existing implementation incorrectly uses `quantity` as revenue.

### 13.6 Segment Generation

```python
async def generate_segments(db, tenant_id, unit_def, period_start, period_end):
    """Generate per-virtual-tag-value breakdown."""
    allocation = await allocate_charges(
        db, tenant_id=tenant_id,
        virtual_tag_id=unit_def.virtual_tag_id,
        period_start=period_start, period_end=period_end
    )
    metric = await resolve_business_metric(db, tenant_id, unit_def.metric_key,
                                           period_start, period_end)
    segments = []
    for alloc in allocation.get("allocations", []):
        amount = Decimal(str(alloc.get("amount", 0)))
        cpu = float(amount / metric.quantity) if metric.quantity > 0 else None
        segments.append({
            "segment": alloc["value"],
            "cost": float(amount),
            "cost_per_unit": cpu,
            "share_percent": alloc.get("share_percent"),
            "is_untagged": alloc.get("is_untagged", False),
        })
    return segments
```

---

## 14. API Design

### 14.1 Unit Definition Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/finops/unit-definitions` | List all unit definitions for tenant |
| `POST` | `/api/v1/finops/unit-definitions` | Create a unit definition |
| `GET` | `/api/v1/finops/unit-definitions/{id}` | Get a unit definition |
| `PATCH` | `/api/v1/finops/unit-definitions/{id}` | Update a unit definition |
| `DELETE` | `/api/v1/finops/unit-definitions/{id}` | Delete a unit definition |

### 14.2 Calculation Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/finops/unit-economics/{definition_id}/calculate` | Calculate for a period |
| `GET` | `/api/v1/finops/unit-economics/{definition_id}/history` | Get historical results |
| `POST` | `/api/v1/finops/unit-economics/{definition_id}/recalculate` | Trigger recalculation |
| `GET` | `/api/v1/finops/unit-economics/{definition_id}/explain` | Get detailed explanation |

### 14.3 Business Metric Endpoints (Extended)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/finops/business-metrics` | List all (existing) |
| `POST` | `/api/v1/finops/business-metrics` | Create/upsert (extended with revenue_amount, source, dimensions) |
| `POST` | `/api/v1/finops/business-metrics/import` | CSV upload |
| `GET` | `/api/v1/finops/metric-sources` | List metric data sources |
| `POST` | `/api/v1/finops/metric-sources` | Create metric data source |
| `POST` | `/api/v1/finops/metric-sources/{id}/sync` | Trigger metric sync |

### 14.4 Enhanced Existing Endpoints

| Endpoint | Enhancement |
|----------|-------------|
| `GET /unit-economics` | Add `unit_definition_id` param; return `explanation` field |
| `GET /focus/summary` | Add optional `virtual_tag_id` + `virtual_tag_value` overlay |

### 14.5 Request/Response Examples

**Create Unit Definition:**
```json
POST /api/v1/finops/unit-definitions
{
  "name": "Cost per Active User",
  "description": "Infrastructure cost per monthly active user",
  "cost_source": "all",
  "cost_field": "effective_cost",
  "metric_key": "active_users",
  "virtual_tag_id": "uuid-of-team-tag",
  "cost_scope": "Platform",
  "target_cost_per_unit": 15.00,
  "include_charge_categories": ["Usage"]
}
```

**Calculate Response:**
```json
GET /api/v1/finops/unit-economics/{id}/calculate?period_start=2026-01-01&period_end=2026-01-31
{
  "unit_definition_id": "uuid",
  "name": "Cost per Active User",
  "period_start": "2026-01-01",
  "period_end": "2026-01-31",
  "numerator": {
    "total_cost": 125000.00,
    "cost_field": "effective_cost",
    "currency": "USD",
    "line_item_count": 45000,
    "source_description": "All FOCUS charges, effective_cost"
  },
  "denominator": {
    "metric_key": "active_users",
    "quantity": 8500,
    "unit_label": "users"
  },
  "result": {
    "cost_per_unit": 14.71,
    "trend_percent": -5.2,
    "gross_margin_percent": 82.5,
    "target_cost_per_unit": 15.00,
    "variance_amount": -0.29,
    "variance_percent": -1.9
  },
  "attribution": {
    "virtual_tag_name": "team",
    "cost_scope": "Platform"
  },
  "explanation": {
    "cost_source_description": "All FOCUS charges, effective_cost",
    "period": "January 2026",
    "attribution_method": "Virtual Tag 'team', value 'Platform'",
    "calculation": "$125,000.00 / 8,500 users = $14.71/user"
  },
  "segments": [
    {"segment": "Platform", "cost": 45000.00, "cost_per_unit": 15.00, "share_percent": 36.0},
    {"segment": "Data", "cost": 35000.00, "cost_per_unit": 14.00, "share_percent": 28.0},
    {"segment": "Other", "cost": 45000.00, "cost_per_unit": 15.00, "share_percent": 36.0}
  ],
  "drill_down": {
    "focus_explorer_url": "/dashboard/focus?period_start=2026-01-01&period_end=2026-01-31"
  }
}
```

---

## 15. Frontend Design

### 15.1 Updated Unit Economics Overview Page

```
/dashboard/unit-economics

┌─────────────────────────────────────────────────────────────┐
│  Unit Economics                                    [+ Create]│
│  Map infrastructure costs to business metrics               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │ Active   │ │ Total    │ │ Avg Cost │ │ Gross    │      │
│  │ Units: 4 │ │ Spend    │ │ Trend    │ │ Margin   │      │
│  │          │ │ $125K    │ │ -3.2%    │ │ 82.5%    │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
│                                                             │
│  Unit Definitions                                           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Name              │ Cost/unit │ Trend │ Margin │ ... │  │
│  │ Cost per Customer │ $12.50    │ -5%   │ 85%    │ →  │  │
│  │ Cost per Request  │ $0.0023   │ +2%   │ —      │ →  │  │
│  │ Cost per AI Token │ $0.000012 │ -8%   │ 72%    │ →  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Trend Chart (12-month)                                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  ▂▃▅▆▇█▇▆▅▄▃▃  (cost_per_unit over time)            │  │
│  │  — — — — — — — (target line)                         │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 15.2 Create/Edit Unit Definition Dialog

```
┌─────────────────────────────────────────────────────┐
│  Create Unit Definition                              │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Name: [Cost per Active User              ]          │
│  Description: [Infrastructure cost per MAU]          │
│                                                      │
│  ── Cost Source (Numerator) ──                        │
│  Source: [All FOCUS charges           ▼]              │
│    Options:                                           │
│    - All costs                                        │
│    - By Provider (select provider)                    │
│    - By Service Category (select category)            │
│    - By Virtual Tag (select tag + value)              │
│    - By Account (select account)                      │
│                                                      │
│  Cost Field: [effective_cost ▼]                       │
│                                                      │
│  ── Business Metric (Denominator) ──                  │
│  Metric: [active_users ▼]  (or create new)           │
│                                                      │
│  ── Attribution ──                                    │
│  Virtual Tag: [team ▼]  (optional)                   │
│  Scope: [Platform ▼]     (optional)                  │
│                                                      │
│  ── Targets ──                                        │
│  Target cost per unit: [$15.00        ]               │
│                                                      │
│  ── Preview ──                                        │
│  Current period: $14.71/user (vs target $15.00)      │
│  Trend: -5.2% vs prior period                        │
│                                                      │
│              [Cancel]  [Save Definition]              │
└─────────────────────────────────────────────────────┘
```

### 15.3 Unit Definition Detail Page

```
/dashboard/unit-economics/{definition_id}

┌─────────────────────────────────────────────────────────────┐
│  ← Back to Unit Economics                                   │
│                                                             │
│  Cost per Active User                              [Edit]   │
│  active_users · Jan 2026                                   │
│                                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │ Cost/    │ │ Quantity │ │ Attrib.  │ │ Trend    │      │
│  │ Unit     │ │          │ │ Spend    │ │          │      │
│  │ $14.71   │ │ 8,500    │ │ $125,000 │ │ -5.2%   │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
│                                                             │
│  Target vs Actual                                           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Target: $15.00  Actual: $14.71  Variance: -$0.29   │  │
│  │  ✓ Under target by 1.9%                              │  │
│  │                                                      │  │
│  │  [Drill down in FOCUS Explorer →]                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Gross Margin: 82.5%                                        │
│                                                             │
│  12-Month Trend                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  ▂▃▅▆▇█▇▆▅▄▃▃  (cost_per_unit)                      │  │
│  │  ▂▃▄▅▅▆▆▇▇▇▇▇  (volume bars)                        │  │
│  │  — — — — — — —  (target line)                        │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Allocated Segments                                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Segment    │ Amount    │ Cost/Unit │ Share           │  │
│  │ Platform   │ $45,000   │ $15.00    │ 36.0%           │  │
│  │ Data       │ $35,000   │ $14.00    │ 28.0%           │  │
│  │ Other      │ $45,000   │ $15.00    │ 36.0%           │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Explanation                                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Cost Source: All FOCUS charges, effective_cost        │  │
│  │ Period: January 2026                                  │  │
│  │ Attribution: Virtual Tag 'team', value 'Platform'     │  │
│  │ Calculation: $125,000.00 / 8,500 users = $14.71/user │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 15.4 Drill-Down Flow

```
Unit Economics
    ↓ click "Cost per Customer"
Unit Definition Detail
    ↓ click "Drill down in FOCUS Explorer"
Focus Explorer (pre-filtered by period + cost source)
    ↓ click a service row
Line Items (filtered by service)
    ↓ click a specific charge
Charge Detail (full FOCUS row with all 30+ columns)
```

The drill-down URL encodes the cost source filters:
```
/dashboard/focus?period_start=2026-01-01&period_end=2026-01-31&provider_name=aws&service_category=Compute
```

---

## 16. Critical Architectural Decisions

### 16.1 Should Unit Economics be a separate module, an extension of Focus Explorer, or an analytical layer?

**Decision: Extension of existing allocation architecture + new definition layer.**

- The calculation engine lives in `services/allocation_service.py` (extended)
- New `unit_definitions` table stores reusable formulas
- New `unit_calculation_results` table stores persisted results
- The FOCUS Explorer gets an optional unit economics overlay
- The frontend gets enhanced unit economics pages

**Rationale:** The existing architecture already has 70% of what's needed. Creating a separate module would duplicate the virtual tag engine, cost querying, and allocation logic.

### 16.2 Should Virtual Tags be the attribution layer?

**Decision: Yes, Virtual Tags are the primary attribution mechanism.**

- No new allocation abstraction needed
- Virtual Tags already support 12 FOCUS fields + any provider tag
- SharedCostReallocation handles shared costs
- The `cost_source` field on UnitDefinition adds provider/service/account scoping on top of virtual tags

**Rationale:** The Virtual Tag engine is already Finout-compatible and works correctly. Building a new allocation layer would be redundant.

### 16.3 Should calculations be on-demand, precomputed, or hybrid?

**Decision: Hybrid — on-demand with result persistence.**

- First calculation for a period: on-demand (evaluates charges from scratch)
- Result persisted to `unit_calculation_results`
- Subsequent reads: from persisted results (instant)
- Recalculation: on-demand via API or background worker
- Scheduled recalculation: optional daily/weekly worker

**Rationale:** On-demand ensures correctness. Persistence ensures performance. The existing `allocate_charges()` loads all charges into memory — persisting results avoids repeated this.

### 16.4 Should results be stored permanently, recalculated, or versioned?

**Decision: Stored permanently with recalculation support.**

- Results are stored in `unit_calculation_results`
- Each result has `calculated_at` timestamp
- Recalculation overwrites the existing result for that period
- Historical results are never deleted (audit trail)
- The `explanation` JSONB provides full audit context

### 16.5 How should shared costs be handled?

**Decision: Use existing SharedCostReallocation, no changes needed.**

The existing three strategies (even, manual_percent, telemetry_weight) are sufficient. They are applied within the virtual tag engine before unit economics calculation.

### 16.6 How should customers, products, teams, and environments be modeled?

**Decision: As Virtual Tag values, not as separate entities.**

- "Customer" = a value in a virtual tag with rules matching customer identifiers
- "Product" = a value in a virtual tag with rules matching product tags
- "Team" = a value in a virtual tag with rules matching team tags
- "Environment" = a value in a virtual tag with rules matching environment tags

No separate `customers`, `products`, `teams`, `environments` tables needed. The Virtual Tag engine IS the business entity mapping layer.

### 16.7 How should revenue connect to infrastructure cost?

**Decision: Add `revenue_amount` to BusinessMetric, not a separate revenue table.**

- `BusinessMetric.revenue_amount` stores the monetary revenue for the period
- Used for gross margin calculation: `(revenue_amount - cost) / revenue_amount * 100`
- Revenue can optionally be scoped to a virtual tag value via `extra.virtual_tag_id`

**Rationale:** A separate revenue table would add complexity without benefit for the initial implementation. If revenue dimensions become more complex later, it can be extracted.

### 16.8 How should AI costs be attributed?

**Decision: Same as any other cost — via Virtual Tags.**

AI costs (OpenAI, Gemini, etc.) are already ingested as SaaS billing records and normalized into FOCUS charges under `provider_name: "saas"`. Virtual Tag rules can match `service_name` or `tag.*` keys to attribute AI costs to products, customers, or features.

Example virtual tag rule:
```json
{"field": "service_name", "op": "contains", "value": "OpenAI", "assign": "AI Infrastructure"}
```

### 16.9 How should multi-cloud costs be normalized?

**Decision: Use `x_effective_cost_usd` for cross-currency normalization.**

The CostCharge model already stores `x_effective_cost_usd` (USD equivalent). The calculation engine should use this field when costs span multiple currencies.

### 16.10 How should users understand and audit every calculation?

**Decision: Store `explanation` JSONB on every calculation result.**

Every `unit_calculation_result` row includes an `explanation` field that documents:
- What cost source was used
- What filters were applied
- What attribution method was used
- What the denominator was
- The exact calculation formula
- How the trend was computed

This is inspired by FOCUS 1.3's `AllocatedMethodDetails` and CloudZero's CostFormation auditability.

---

## 17. Gap Analysis

| Area | Existing Capability | Required for Unit Economics | Gap | Change Type |
|------|-------------------|---------------------------|-----|-------------|
| **BusinessMetric model** | quantity, metric_key, period, extra | revenue_amount, source, source_config, dimensions | Missing revenue as monetary value; no metric sources | Extend existing table |
| **Unit Definition** | Does not exist | Reusable formula (cost_source + metric_key + attribution) | Entirely missing | New table + service |
| **Calculation persistence** | Recalculated on demand every time | Historical results stored | No persistence | New table |
| **Cost source selection** | Hardcoded to all FOCUS total | Provider/service/account/vtag scoped | No cost source concept | Extend calculation engine |
| **Explainability** | No explanation field | Full audit trail per calculation | Missing | Add explanation JSONB |
| **Time-series unit economics** | Single aggregated result | Monthly/daily trend chart | Missing time-series | Extend calculation engine |
| **Metric sources** | Manual entry only | API, CSV, webhook, integration | No automation | New table + worker |
| **Multi-dimensional** | Single virtual tag per metric | Cross-tabulation (team AND product) | Not supported | Future enhancement |
| **FOCUS Explorer overlay** | CostExplorer shows cost only | Show cost_per_unit on Explorer | Missing | Extend FOCUS Explorer |
| **Virtual tag drill-down** | "View in Explorer" link ignores allocation | Pass virtual tag value as filter | Known bug | Fix existing link |
| **Revenue modeling** | revenue treated as quantity | revenue as monetary amount | Wrong semantics | Fix build_unit_economics |
| **Segment denominators** | All segments use same metric | Per-segment metric support | Missing | Extend segments logic |
| **Allocation caching** | Re-evaluates all charges per request | Cache allocation results | No caching | Add in-memory + DB cache |
| **BillableMetric table** | `business_metrics` | `unit_definitions` + `unit_calculation_results` | Conceptual gap | New tables |

---

## 18. Bugs and Issues Found

### Critical

| # | File:Line | Issue | Impact |
|---|-----------|-------|--------|
| 1 | `allocation_service.py:593-596` | `revenue_by_period` uses `metric.quantity` (a count like "10000 customers") as the revenue amount for gross margin calculation. Gross margin = `(revenue - cost) / revenue` but revenue is actually a count, not dollars. | Gross margin calculation is financially incorrect. |
| 2 | `allocation_service.py:630-635` | All segments use the same denominator (first non-revenue metric's quantity). If segments represent different customers with different user counts, they all divide by the same number. | Per-segment cost_per_unit is incorrect when segments have different metric volumes. |

### High

| # | File:Line | Issue | Impact |
|---|-----------|-------|--------|
| 3 | `virtual_tag_engine.py:255` | All CostCharge rows loaded into memory for allocation. For tenants with millions of charges, this causes OOM. | Scalability limit. |
| 4 | `charge_store.py:406` | `summarize_focus` accumulates `float(r.total_cost)` — loses precision for large sums. | Incorrect totals at scale. |
| 5 | `allocation_service.py:542` | `build_unit_economics` unconditionally calls `build_billing_overview` even when FOCUS data is available. Expensive fallback. | Unnecessary latency on every unit economics request. |
| 6 | `billing_sync_service.py:1035-1044` | FOCUS ingest only uses first AWS connection (`.limit(1)`). GCP/Azure FOCUS not triggered from billing sync. | Multi-account tenants get incomplete FOCUS data. |

### Medium

| # | File:Line | Issue | Impact |
|---|-----------|-------|--------|
| 7 | `allocation_service.py` (build_unit_economics) | No historical caching — every request recalculates from scratch. | Slow for large tenants. |
| 8 | `allocation_service.py` | No time-series unit economics — single aggregated result only. | Cannot show monthly trend in unit economics. |
| 9 | `[metricId]/page.tsx:154` | "Drill down in FOCUS Explorer" link only passes period, not cost source or attribution filters. | User cannot drill into the specific costs that feed the unit calculation. |
| 10 | `virtual_tag_engine.py` | `retroactive` flag stored but never enforced. | Misleading — users think tag only applies retroactively but it applies to all data. |

---

## 19. Implementation Roadmap

### Phase 1 — Foundation (Weeks 1-2)

**Objective:** Fix data model issues that make unit economics unreliable.

**Features:**
1. Add `revenue_amount` column to `business_metrics` table
2. Add `source`, `source_config`, `dimensions` columns to `business_metrics`
3. Fix `build_unit_economics()` to use `revenue_amount` instead of `quantity` for gross margin
4. Fix segment denominator to use per-segment metrics when available
5. Add `cost_source` parameter to `_focus_total_spend()` for scoped queries
6. Remove unconditional `build_billing_overview()` call when FOCUS data exists

**Database changes:**
```sql
ALTER TABLE business_metrics ADD COLUMN revenue_amount NUMERIC(18,6);
ALTER TABLE business_metrics ADD COLUMN source VARCHAR(32) DEFAULT 'manual';
ALTER TABLE business_metrics ADD COLUMN source_config JSONB DEFAULT '{}';
ALTER TABLE business_metrics ADD COLUMN dimensions JSONB DEFAULT '{}';
```

**Backend changes:**
- `allocation_service.py`: Fix `build_unit_economics()` revenue logic
- `allocation_service.py`: Extend `_focus_total_spend()` with filter params
- `allocation_routes.py`: Extend POST /business-metrics schema with new fields

**Frontend changes:**
- `configure/page.tsx`: Add revenue_amount field to form
- `configure/page.tsx`: Add source/dimensions fields

**Acceptance criteria:**
- Gross margin uses monetary revenue, not count
- Cost source can be filtered to specific provider/service
- No unnecessary `build_billing_overview()` call

---

### Phase 2 — Unit Definitions (Weeks 3-5)

**Objective:** Allow users to define reusable unit economics formulas.

**Features:**
1. Create `unit_definitions` table
2. Create `unit_calculation_results` table
3. Implement UnitDefinition CRUD API
4. Implement calculation engine
5. Implement explanation generation
6. Create frontend Unit Definition management pages

**Database changes:**
```sql
CREATE TABLE unit_definitions (...);
CREATE TABLE unit_calculation_results (...);
```

**Backend changes:**
- New service: `services/unit_economics_service.py`
- New routes: `routes/unit_economics_routes.py`
- Extend `virtual_tag_engine.py` with `_focus_total_spend_with_count()`

**Frontend changes:**
- Enhance `/dashboard/unit-economics` overview page
- Add Unit Definition create/edit dialog
- Add Unit Definition detail page with explanation
- Add `MetricTrendChart` with 12-month history

**Acceptance criteria:**
- Users can create a Unit Definition with cost source, metric, and attribution
- Calculation produces correct cost_per_unit with explanation
- Historical results are persisted and queryable
- Trend analysis shows period-over-period change

---

### Phase 3 — Metric Sources (Weeks 6-7)

**Objective:** Enable automated business metric ingestion.

**Features:**
1. Create `metric_data_sources` table
2. Implement CSV upload for metrics
3. Implement API endpoint for metric ingestion
4. Create background worker for scheduled metric sync
5. Create metric source management UI

**Database changes:**
```sql
CREATE TABLE metric_data_sources (...);
```

**Backend changes:**
- New worker: `workers/metric_sync_worker.py`
- New routes: metric source CRUD + sync trigger
- CSV parsing service

**Frontend changes:**
- Metric source management page
- CSV upload component
- Metric source status indicators

**Acceptance criteria:**
- Users can upload metrics via CSV
- Users can configure API endpoints for metric ingestion
- Metrics are automatically synced on schedule

---

### Phase 4 — Focus Explorer Integration (Week 8)

**Objective:** Connect unit economics to cost exploration.

**Features:**
1. Add virtual tag overlay to FOCUS Explorer
2. Fix "View in Explorer" link to pass cost source filters
3. Add cost_per_unit column to Explorer summary when a unit definition is selected
4. Add unit economics context to Explorer line items

**Backend changes:**
- Extend `summarize_focus()` with optional virtual tag overlay
- Extend `charge_to_dict()` with attribution fields

**Frontend changes:**
- Add unit definition selector to FOCUS Explorer
- Add cost_per_unit overlay on summary table
- Fix drill-down links from allocation/unit-economics pages

**Acceptance criteria:**
- User can select a unit definition in Explorer and see cost_per_unit per dimension
- "View in Explorer" from unit economics page pre-filters correctly
- Explorer shows which costs are attributed to which business entity

---

### Phase 5 — Advanced Features (Weeks 9-12)

**Objective:** Add multi-dimensional, time-series, and advanced allocation features.

**Features:**
1. Time-series unit economics (monthly trend stored in `unit_calculation_results`)
2. Multi-dimensional unit economics (cost per customer per product)
3. Per-segment metric support (different denominator per segment)
4. Scheduled recalculation worker
5. Unit economics anomaly detection (cost_per_unit spike alerts)

**Backend changes:**
- Extend calculation engine with time-series support
- Add segment-level metric resolution
- New worker: `workers/unit_economics_recalc_worker.py`
- Extend anomaly detection with unit-economics-aware thresholds

**Frontend changes:**
- Time-series chart on unit definition detail page
- Multi-dimensional comparison view
- Segment detail with per-segment metrics

**Acceptance criteria:**
- Monthly trend chart shows 12-month cost_per_unit history
- Multi-dimensional view shows "cost per customer per product"
- Anomaly alerts fire when cost_per_unit exceeds threshold

---

### Phase 6 — Revenue and Margin (Weeks 13-14)

**Objective:** Complete the financial picture with revenue, margin, and cost-to-revenue analysis.

**Features:**
1. Revenue data ingestion (manual + CSV + API)
2. Gross margin calculation per unit definition
3. Cost-to-revenue ratio
4. Contribution margin analysis
5. Break-even analysis
6. Cloud Efficiency Rate (CER) per dimension

**Backend changes:**
- Extend `build_unit_economics()` with margin calculations
- Add CER calculation per virtual tag value
- Add break-even point calculation

**Frontend changes:**
- Revenue vs cost comparison chart
- Margin analysis components
- CER dashboard widget
- Break-even analysis tool

**Acceptance criteria:**
- Gross margin is financially accurate
- CER matches manual spreadsheet calculation
- Break-even analysis is mathematically correct

---

## End-to-End Flow Summary

```
CURRENT CODEBASE:
  CostCharge (FOCUS 1.2) + VirtualTag + BusinessMetric + SharedCostReallocation
         ↓
NEW ATTRIBUTION LAYER:
  UnitDefinition (cost_source + metric_key + virtual_tag + cost_scope)
         ↓
BUSINESS METRICS:
  BusinessMetric (quantity + revenue_amount + source + dimensions)
         ↓
CALCULATION ENGINE:
  resolve_cost_source() → numerator
  resolve_business_metric() → denominator
  cost_per_unit = numerator / denominator
  + trend + margin + variance + explanation
         ↓
PERSISTENCE:
  unit_calculation_results (historical, auditable, explainable)
         ↓
UNIT ECONOMICS:
  Overview → Detail → Segments → Trend → Explanation
         ↓
DRILL-DOWN:
  FOCUS Explorer (pre-filtered by cost source + attribution)
         ↓
ORIGINAL COST:
  CostCharge rows (full FOCUS 1.2 data with 30+ columns)
```

---

*Document generated from codebase analysis. All file references are to the actual repository at `C:\Users\Admin\Desktop\digitomics-infraops`.*
