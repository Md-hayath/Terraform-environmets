# FOCUS Explorer — Enterprise Implementation & Feature Roadmap

> How Digitomics InfraOps implements the FOCUS specification, what's built, what's missing, and how to evolve toward a FOCUS-conformant enterprise FinOps platform.

---

## Table of Contents

1. [What We Built](#1-what-we-built)
2. [What FOCUS Actually Solves](#2-what-focus-actually-solves)
3. [The 4 FOCUS Datasets](#3-the-4-focus-datasets)
4. [107 FOCUS Columns — What We Have vs What We Need](#4-107-focus-columns--what-we-have-vs-what-we-need)
5. [Gap Analysis by Feature Area](#5-gap-analysis-by-feature-area)
6. [Enterprise Architecture](#6-enterprise-architecture)
7. [Ingestion Pipeline — How It Works](#7-ingestion-pipeline--how-it-works)
8. [Normalization — The FOCUS Schema Layer](#8-normalization--the-focus-schema-layer)
9. [Grouping & Aggregation Engine](#9-grouping--aggregation-engine)
10. [Allocation — Virtual Tags & Chargeback](#10-allocation--virtual-tags--chargeback)
11. [Unit Economics — How to Build It Right](#11-unit-economics--how-to-build-it-right)
12. [Enterprise Gating & Multi-Tenancy](#12-enterprise-gating--multi-tenancy)
13. [Feature Roadmap — Phase by Phase](#13-feature-roadmap--phase-by-phase)
14. [How FOCUS Solves Problems We Haven't Tackled Yet](#14-how-focus-solves-problems-we-havent-tackled-yet)
15. [Database Migrations Required](#15-database-migrations-required)
16. [API Surface — Current & Proposed](#16-api-surface--current--proposed)
17. [Frontend — What Exists & What's Missing](#17-frontend--what-exists--whats-missing)
18. [Testing Strategy](#18-testing-strategy)
19. [Competitive Position](#19-competitive-position)

---

## 1. What We Built

### Core Components

| Layer | File | What It Does | Lines |
|-------|------|-------------|-------|
| **Schema** | `backend/services/focus/schema.py` | Normalizes any provider row to FOCUS 1.2 canonical fields. 50+ service→category mappings. SHA-1 hash for idempotent IDs. | 211 |
| **Store** | `backend/services/focus/charge_store.py` | Bulk upsert, filtered query, group-by aggregation, time-series, dimension autocomplete. Top-N + "Other" overflow. Multi-currency aware. | 575 |
| **Ingest** | `backend/services/focus/ingest.py` | 7 adapters: AWS CE API, AWS CUR CSV, GCP BigQuery export, GCP billing lines, Azure Blob FOCUS CSV, Azure billing lines, SaaS BillingRecords. | 833 |
| **Allocation** | `backend/services/allocation/virtual_tag_engine.py` | 12 FOCUS dimensions, 5 rule operators (eq/contains/regex/in/exists), first-match-wins funnel, shared cost reallocation (even/manual%/telemetry). | 380 |
| **Business Logic** | `backend/services/allocation_service.py` | CRUD for virtual tags, showback, chargeback, unit economics with trends, target variance. | 829 |
| **Routes** | `backend/routes/focus_routes.py` | 7 API endpoints: summary, timeseries, charges, dimension-values, export, export-config, ingest. | 413 |
| **Model** | `backend/app/db/models.py` | CostCharge (~40 columns), VirtualTag, SharedCostReallocation, OptimizationRecommendation, AnomalyRoutingEvent. | ~140 lines for these 5 models |
| **Migration** | `backend/alembic/versions/0020_focus_cost_charges_and_finops.py` | Creates all 5 tables. Idempotent. | — |
| **Worker** | `backend/workers/focus_sync_worker.py` | Scheduled/manual sync. | — |
| **Frontend** | `frontend/src/components/finops/cost-explorer.tsx` | Filter sidebar (7 dimensions), group-by, cost field toggle, granularity, area chart, breakdown table, line items, CSV export, saved views. | 918 |
| **Frontend Service** | `frontend/src/services/finops-allocation-service.ts` | All API calls + TypeScript interfaces. | 601 |

### What's Working

1. **Ingestion**: AWS Cost Explorer API + CUR CSV from S3, GCP BigQuery billing export, Azure Blob FOCUS 1.2 CSV, SaaS BillingRecords — all orchestrated by `run_focus_ingestion_for_tenant()`
2. **Normalization**: `normalize_charge_row()` maps 3 provider-specific input shapes to FOCUS 1.2, generates stable hash IDs for dedup
3. **Persistence**: Bulk upsert via `INSERT ... ON CONFLICT DO UPDATE`, 250 rows/chunk, insert/update counting via `xmax`
4. **Querying**: 7-dimension filtering with overlap semantics on date ranges, Top-N (50) + "Other" overflow, multi-currency awareness
5. **Allocation**: Virtual tag rules with 12 FOCUS dimensions, shared cost reallocation, showback/chargeback generation
6. **Unit Economics**: Cost-per-unit with trends, target variance, scoped allocation, revenue margin
7. **Frontend**: Full explorer with charts, tables, drill-down, CSV export, saved views (localStorage)
8. **Enterprise Gating**: `FOCUS_EXPLORER` feature flag, `costs.analysis` permission, sales-lockable surface

### Enterprise Gating Details

```
Feature Flag: FOCUS_EXPLORER (feature_flag_catalog.py:115)
Permission: costs.analysis
Sales Lock: ("focus-explorer", "FOCUS Explorer") (enterprise_entitlements.py:18)
Frontend Gate: EnterpriseGate component with surfaceKey="focus-explorer"
```

---

## 2. What FOCUS Actually Solves

FOCUS (FinOps Open Cost and Usage Specification) is an open standard (v1.4, June 2026) backed by the Linux Foundation. It normalizes billing data across all technology vendors — cloud, SaaS, AI, data centers — into a single canonical schema.

### The Problem

Every vendor bills differently:
- AWS: `unblendedCost`, `amortizedCost`, `blendedRate`
- GCP: `cost`, `currency`, `usage_amount`
- Azure: `BilledCost`, `EffectiveCost`, `CostInBillingCurrency`
- SaaS: `total_cost`, `amount`, `price`

FinOps practitioners spend 60-70% of their time normalizing data instead of analyzing it.

### The Solution

FOCUS creates one schema with 107 columns (v1.4) so:
- `BilledCost` from AWS = `BilledCost` from GCP = `BilledCost` from Azure
- One SQL query works across all providers
- New vendors adopting FOCUS = instant onboarding
- Skills and queries are portable across organizations

### Who Uses It

GitLab, STMicroelectronics, European Parliament, UnitedHealth Group, Zoom, Australian Retirement Trust, Heineken — all report faster time-to-insights, single source of truth, simplified multi-cloud management.

---

## 3. The 4 FOCUS Datasets

| Dataset | Purpose | Version | We Have It? |
|---------|---------|---------|------------|
| **Cost and Usage** | Core charge-line items (~40 required columns) | v1.0+ | Yes |
| **Invoice Detail** | Invoice-level charges, payment terms, PO numbers, tax | v1.4 | No |
| **Contract Commitment** | Commitment terms, lifecycle, discount rates, fulfillment | v1.3+ | No |
| **Billing Period** | Billing period boundaries, recency, completeness status | v1.4 | No |

### Cost and Usage (What We Implement)

The primary dataset. Every charge line item with columns for:
- **Cost**: BilledCost, EffectiveCost, ListCost, ContractedCost
- **Time**: ChargePeriodStart/End, BillingPeriodStart/End
- **Identity**: ProviderName, SubAccountId, ResourceId, SkuId
- **Classification**: ServiceCategory, ChargeCategory, ChargeClass
- **Usage**: ConsumedQuantity, PricingQuantity

### Invoice Detail (What We're Missing)

Carries charges as they appear on issued invoices. Enables:
- Tie consumption records to invoice line items
- Catch invoice errors before payment
- Shorten monthly close process

Key columns: InvoiceId, InvoiceDetailId, InvoiceIssueDate, PaymentDueDate, PaymentTerms, PurchaseOrderNumber

### Contract Commitment (What We're Missing)

Describes commitment-based purchases in provider-agnostic format. Enables:
- Compare Reserved Instances (AWS) vs CUDs (GCP) vs Reserved VMs (Azure)
- Track commitment lifecycle and utilization
- Identify unused commitments (waste)

Key columns: ContractCommitmentId, ContractCommitmentType, ContractCommitmentDiscountPercentage, ContractCommitmentLifecycleStatus

### Billing Period (What We're Missing)

Provides billing period boundaries and status. Enables:
- Know if data is final or subject to revision
- Period-over-period analysis when billing periods don't align with calendar months

Key columns: BillingPeriodCreated, BillingPeriodLastUpdated, BillingPeriodStatus

---

## 4. 107 FOCUS Columns — What We Have vs What We Need

### We Have (37 columns mapped)

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
CommitmentDiscountId, CommitmentDiscountType, CommitmentDiscountStatus, CommitmentDiscountCategory,
Tags, InvoiceIssuerName, PublisherName
```

### We're Missing (70 columns)

| Category | Missing Columns | Priority | Why They Matter |
|----------|----------------|----------|-----------------|
| **Account** | BillingAccountId, BillingAccountName, BillingAccountType, SubAccountType | P0 | Distinguish billing accounts from sub-accounts; map provider constructs |
| **Invoice** | InvoiceId, InvoiceDetailId, InvoiceIssueDate, PaymentDueDate, PaymentTerms, PurchaseOrderNumber, PaymentCurrency, PaymentCurrencyBilledCost | P0 | Invoice reconciliation — tie every charge to an invoice line item |
| **Contract** | ContractId, ContractApplied, ContractCommitmentId, ContractCommitmentCost, ContractCommitmentStartDate, ContractCommitmentEndDate, + 20 more | P1 | Cross-provider commitment comparison |
| **Commitment** | CommitmentProgramEligibilityDetails, CommitmentDiscountName, CommitmentDiscountQuantity, CommitmentDiscountUnit | P0 | Coverage rate calculation — know which programs a charge qualifies for |
| **Pricing** | PricingCategory, PricingCurrency, ListUnitPrice, ContractedUnitPrice, PricingCurrencyListUnitPrice | P0 | Understand pricing model (on-demand vs committed vs negotiated) |
| **SKU** | SkuMeter, SkuPriceDetails | P1 | Metering granularity — understand what's being metered |
| **Charge** | ChargeFrequency | P1 | Distinguish one-time vs recurring |
| **Location** | AvailabilityZone | P2 | Finer-grained location analysis |
| **Provider** | ServiceProviderName, HostProviderName, ReferenceInvoiceId | P1 | Marketplace tracking; distinguish service provider from host |
| **Allocation** | AllocatedMethodDetails, AllocatedMethodId, AllocatedResourceId, AllocatedResourceName, AllocatedTags | P2 | Provider-side cost splitting |
| **Billing Period** | BillingPeriodCreated, BillingPeriodLastUpdated, BillingPeriodStatus | P1 | Data freshness — know if data is final |

---

## 5. Gap Analysis by Feature Area

### Reporting & Analytics (16 use cases)

| Use Case | Status | What's Missing |
|----------|--------|---------------|
| Report costs by service category | Working | — |
| Analyze service costs month over month | Working | — |
| Analyze service costs by subaccount | Working | — |
| Analyze resource costs by SKU | Partial | Need SkuMeter, SkuPriceDetails |
| Analyze marketplace vendors costs | Missing | Need ServiceProviderName |
| Analyze capacity reservations on compute | Missing | Need CapacityReservationId, CapacityReservationStatus |
| Calculate unit economics | Partial | Need better metric scoping |
| Determine Effective Savings Rate | Partial | Need ListCost populated consistently |
| Report costs by service category and subcategory | Working | — |
| Report service costs by providers subaccount | Missing | Need BillingAccountId |
| Report on initial contract commitments | Missing | Need ContractCommitment dataset |
| Report spending across billing periods | Missing | Need BillingPeriodStatus |
| Report corrections for a previously invoiced billing period | Missing | Need ChargeClass populated |
| Analyze the different metered costs for a particular SKU | Missing | Need SkuMeter |
| Report application cost month over month | Partial | Need virtual tag by application |

### Allocation (12 use cases)

| Use Case | Status | What's Missing |
|----------|--------|---------------|
| Analyze tag coverage | Working | — |
| Identify resources with shared cost allocation | Working | — |
| Allocate multi-currency charges per application | Partial | Need currency normalization |
| Analyze total cost by allocated resource | Missing | Need AllocatedResourceId |
| Identify sources of billed cost | Working | — |
| Verify accuracy of provider invoices | Missing | Need InvoiceId + Invoice Detail dataset |
| Analyze cost by participating entities | Missing | Need BillingAccountType, SubAccountType |
| Analyze effective cost by pricing currency | Missing | Need PricingCurrency |
| Report corrections by subaccount | Missing | Need ChargeClass + correction logic |
| Track marketplace purchases | Missing | Need ServiceProviderName |
| Analyze purchase of virtual currency | Missing | Need AI/SAAS token columns |

### Anomaly Management (7 use cases)

| Use Case | Status | What's Missing |
|----------|--------|---------------|
| Analyze costs by service name | Working | — |
| Analyze service costs by region | Working | — |
| Compare resource usage month over month | Partial | No detection logic |
| Identify anomalous daily spending by subaccount | Missing | Need daily aggregation + z-score |
| Identify anomalous daily spending by subaccount and region | Missing | — |
| Identify anomalous daily spending by subaccount, region, and service | Missing | — |
| Identify unused capacity reservations | Missing | Need CapacityReservationId |

### Budgeting (5 use cases)

| Use Case | Status | What's Missing |
|----------|--------|---------------|
| Compare billed cost per subaccount to budget | Missing | Need budgets table |
| Update budgets with billed costs | Missing | — |
| Update budgets for each application | Missing | — |
| Track contract commitment burn-down over time | Missing | Need ContractCommitment dataset |
| Calculate consumption of virtual currency | Missing | Need token columns |

### Forecasting (4 use cases)

| Use Case | Status | What's Missing |
|----------|--------|---------------|
| Get historical usage and rates | Partial | Need historical aggregation |
| Forecast amortized costs month over month | Missing | Need trend projection |
| Forecast cashflow month over month by service | Missing | — |
| Calculate average rate of a component resource | Missing | Need SKU-level pricing |

### Rate Optimization (2 use cases)

| Use Case | Status | What's Missing |
|----------|--------|---------------|
| Identify unused commitments | Missing | Need CommitmentDiscountStatus = "Unused" detection |
| Report commitment discount purchases | Missing | Need purchase row tracking |

---

## 6. Enterprise Architecture

### Current Architecture

```
                    ┌─────────────────────────┐
                    │   Frontend (Next.js)     │
                    │   cost-explorer.tsx       │
                    │   - Filter sidebar       │
                    │   - Group-by selector     │
                    │   - Area chart            │
                    │   - Breakdown table       │
                    │   - Line items            │
                    └──────────┬──────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │   API Routes             │
                    │   focus_routes.py         │
                    │   /summary, /timeseries   │
                    │   /charges, /export       │
                    │   /ingest                 │
                    └──────────┬──────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼─────────┐  ┌──▼──────────┐  ┌──▼──────────────┐
    │  charge_store.py   │  │ schema.py   │  │ allocation_     │
    │  - persist          │  │ - normalize │  │ service.py      │
    │  - query            │  │ - map       │  │ - virtual tags  │
    │  - summarize        │  │ - hash      │  │ - showback      │
    │  - timeseries       │  │             │  │ - chargeback    │
    └─────────┬─────────┘  └──┬──────────┘  │ - unit econ     │
              │                │              └──┬──────────────┘
              └────────────────┼────────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │   PostgreSQL             │
                    │   cost_charges (~40 col) │
                    │   virtual_tags           │
                    │   shared_cost_realloc    │
                    │   optimization_recs      │
                    │   anomaly_routing        │
                    └─────────────────────────┘

Ingestion Paths:
  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐
  │ AWS CE API   │  │ GCP BigQuery │  │ Azure Blob   │
  │ + CUR CSV    │  │ Export SQL   │  │ FOCUS CSV    │
  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘
         │                │                  │
         └────────────────┼──────────────────┘
                          │
              ┌───────────▼───────────┐
              │ ingest.py              │
              │ run_focus_ingestion    │
              │ _for_tenant()          │
              └───────────┬───────────┘
                          │
              ┌───────────▼───────────┐
              │ normalize_charge_row() │
              │ schema.py              │
              └───────────┬───────────┘
                          │
              ┌───────────▼───────────┐
              │ persist_cost_charges() │
              │ charge_store.py        │
              └───────────────────────┘
```

### Enterprise Architecture (Target State)

```
                    ┌─────────────────────────────────┐
                    │   Frontend (Next.js)             │
                    │   ┌──────────────┐               │
                    │   │ Explorer v2  │               │
                    │   │ - Sankey     │               │
                    │   │ - Saved Views│               │
                    │   │ - NL Query   │               │
                    │   └──────────────┘               │
                    │   ┌──────────────┐               │
                    │   │ Invoices     │               │
                    │   │ - Reconcile  │               │
                    │   │ - Payment    │               │
                    │   └──────────────┘               │
                    │   ┌──────────────┐               │
                    │   │ Commitments  │               │
                    │   │ - Coverage   │               │
                    │   │ - Comparison │               │
                    │   └──────────────┘               │
                    │   ┌──────────────┐               │
                    │   │ AI Costs     │               │
                    │   │ - Tokens     │               │
                    │   │ - Models     │               │
                    │   └──────────────┘               │
                    │   ┌──────────────┐               │
                    │   │ Anomalies    │               │
                    │   │ - Detection  │               │
                    │   │ - Routing    │               │
                    │   └──────────────┘               │
                    └──────────┬──────────────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │   API Layer              │
                    │   focus_routes.py         │
                    │   invoice_routes.py (new) │
                    │   commitment_routes (new) │
                    │   anomaly_routes.py (new) │
                    └──────────┬──────────────┘
                               │
              ┌────────────────┼────────────────────┐
              │                │                    │
    ┌─────────▼─────────┐  ┌──▼──────────┐  ┌──────▼──────────┐
    │  charge_store.py   │  │ schema.py   │  │ allocation_     │
    │  + new columns     │  │ + 1.4 cols  │  │ service.py      │
    │  + invoice join    │  │ + invoice   │  │ + virtual tags  │
    │  + commitment join │  │ + contract  │  │ + showback      │
    │  + anomaly detect  │  │ + eligibility│ │ + chargeback    │
    └─────────┬─────────┘  └──┬──────────┘  │ + unit econ     │
              │                │              └──┬──────────────┘
              └────────────────┼────────────────┘
                               │
              ┌────────────────┼────────────────────┐
              │                │                    │
    ┌─────────▼─────────┐  ┌──▼──────────┐  ┌──────▼──────────┐
    │ cost_charges       │  │ focus_      │  │ focus_          │
    │ (65+ columns)      │  │ invoices    │  │ commitments     │
    │                    │  │ (new)       │  │ (new)           │
    └───────────────────┘  └─────────────┘  └─────────────────┘
                               │                    │
    ┌───────────────────┐  ┌──▼──────────┐  ┌──────▼──────────┐
    │ virtual_tags       │  │ budgets     │  │ anomaly_        │
    │                    │  │ (new)       │  │ events          │
    └───────────────────┘  └─────────────┘  └─────────────────┘
```

---

## 7. Ingestion Pipeline — How It Works

### Entry Points

Three paths trigger FOCUS ingestion:

1. **Manual**: `POST /api/v1/finops/focus/ingest` (admin-only, `focus_routes.py:350`)
2. **Scheduled**: `focus_sync_worker.py` (standalone/cron)
3. **Post-billing-sync**: `billing_sync_service.py:1112` (auto-trigger after sync)

### Orchestrator

`run_focus_ingestion_for_tenant()` (`ingest.py:719`) runs all adapters in sequence:

```python
# For each provider, ingest from all connected accounts:
for aws_connection in aws_connections:
    await ingest_aws_cost_explorer_lines(db, ...)

for gcp_connection in gcp_connections:
    bq_result = await ingest_gcp_bigquery_export(db, ...)
    # If BigQuery export not configured, fall back to billing lines

for azure_connection in azure_connections:
    az_export = await ingest_azure_focus_export(db, ...)
    # If FOCUS export not configured, fall back to billing lines

# Always ingest SaaS
await ingest_saas_from_billing_records(db, ...)

await db.commit()
```

### Provider Adapters

| Adapter | Source | Method | Limit |
|---------|--------|--------|-------|
| `ingest_aws_cost_explorer_lines()` | AWS Cost Explorer API | `GROUP BY SERVICE` with UnblendedCost/AmortizedCost | Current month, 5000 rows from CUR CSV |
| `_try_read_aws_cur_csv()` | S3 CUR CSV | Reads first `.csv` under configured S3 prefix | 5000 rows |
| `ingest_gcp_bigquery_export()` | BigQuery SQL | Parameterized query against `project.dataset.table` | 5000 rows |
| `ingest_gcp_billing_lines()` | GCP billing snapshot | `cost_by_service` + `cost_by_project` payloads | Current month |
| `ingest_azure_focus_export()` | Azure Blob CSV | FOCUS 1.2 native export from Azure Cost Management | 5000 rows |
| `ingest_azure_billing_lines()` | Azure billing snapshot | `cost_by_service` + `cost_by_resource_group` | Current month |
| `ingest_saas_from_billing_records()` | BillingRecord table | Maps non-cloud BillingRecord rows to FOCUS Usage lines | 500 rows |

### Multi-Account Support

Each provider accepts a `_connections` list — all active connections are ingested, not just one:

```python
# _connections_with_credentials() at focus_routes.py:321
aws_connections = await _connections_with_credentials(
    db, tenant_id=tenant_id, provider=ConnectionProvider.aws,
    get_credentials=lambda conn: get_aws_connection_credentials_by_id(...)
)
# Returns [{"connection_id": uuid, "credentials": dict}, ...]
```

---

## 8. Normalization — The FOCUS Schema Layer

### `normalize_charge_row()` (schema.py:97)

Maps raw provider fields to FOCUS 1.2:

```python
def normalize_charge_row(raw: dict, *, provider_name: str) -> dict:
    # 1. Resolve service_name from multiple input shapes
    service_name = raw.get("service_name") or raw.get("ServiceName") or raw.get("service")

    # 2. Resolve costs (BilledCost, EffectiveCost, ListCost, ContractedCost)
    billed = _dec(raw.get("billed_cost") or raw.get("BilledCost") or raw.get("cost"))
    effective = _dec(raw.get("effective_cost") or raw.get("EffectiveCost") or billed)

    # 3. Resolve time periods (ChargePeriod, BillingPeriod)
    period_start = _dt(raw.get("charge_period_start") or raw.get("ChargePeriodStart"))

    # 4. Resolve line-item ID (or generate stable hash)
    line_id = raw.get("provider_line_item_id") or raw.get("line_item_id")
    if not line_id:
        canonical = "|".join([provider_name, service_name, period_start.isoformat(), ...])
        line_id = f"{provider_name}:hash:{hashlib.sha1(...).hexdigest()[:24]}"

    # 5. Map service to category
    service_category = raw.get("service_category") or map_service_category(service_name)

    # 6. Return FOCUS 1.2 dict with ~40 fields
    return {
        "provider_line_item_id": str(line_id)[:512],
        "provider_name": provider_name,
        "billed_cost": billed,
        "effective_cost": effective,
        "service_category": service_category,
        "service_name": str(service_name)[:255] if service_name else None,
        "charge_category": raw.get("charge_category") or "Usage",
        # ... 30+ more fields
    }
```

### Service Category Mapping

`SERVICE_CATEGORY_MAP` has 50+ mappings:

```python
SERVICE_CATEGORY_MAP = {
    "amazonec2": "Compute",
    "ec2": "Compute",
    "awslambda": "Compute",
    "amazonrds": "Databases",
    "amazondynamodb": "Databases",
    "amazons3": "Storage",
    "amazoncloudfront": "Networking",
    "amazonsagemaker": "AI and Machine Learning",
    "amazonbedrock": "AI and Machine Learning",
    "compute engine": "Compute",
    "bigquery": "Analytics",
    "kubernetes engine": "Compute",
    "virtual machines": "Compute",
    "openai": "AI and Machine Learning",
    "anthropic": "AI and Machine Learning",
    # ... more
}
```

### Persistence

`persist_cost_charges()` (charge_store.py:74):

```python
async def persist_cost_charges(db, *, tenant_id, provider_name, rows, connection_id=None):
    # 1. Normalize all rows
    values = [normalize_charge_row(raw, provider_name=provider_name) for raw in rows]

    # 2. Bulk upsert in chunks of 250
    for chunk in chunks(values, 250):
        stmt = pg_insert(CostCharge).values(chunk)
        stmt = stmt.on_conflict_do_update(
            index_elements=["tenant_id", "provider_name", "provider_line_item_id"],
            set_={col: getattr(stmt.excluded, col) for col in update_columns},
        ).returning(CostCharge.id, literal_column("(xmax = 0)").label("was_insert"))

        # 3. Count inserts vs updates
        for row in result.all():
            if row.was_insert: inserted += 1
            else: updated += 1

    return {"inserted": inserted, "updated": updated, "total": inserted + updated}
```

---

## 9. Grouping & Aggregation Engine

### FOCUS Dimensions

```python
FOCUS_DIMENSIONS = {
    "service_category": CostCharge.service_category,
    "service_name": CostCharge.service_name,
    "provider_name": CostCharge.provider_name,
    "region_name": CostCharge.region_name,
    "sub_account_id": CostCharge.sub_account_id,
    "charge_category": CostCharge.charge_category,
    "resource_type": CostCharge.resource_type,
}
```

### Summary Aggregation (`summarize_focus()`)

```python
async def summarize_focus(db, *, tenant_id, group_by="service_category",
                          cost_field="effective_cost", period_start=None, period_end=None,
                          provider_name=None, service_category=None, ...):
    # 1. Build query with group-by dimension + currency
    dim_col = FOCUS_DIMENSIONS.get(group_by, CostCharge.service_category)
    cost_col = CostCharge.effective_cost if cost_field == "effective_cost" else CostCharge.billed_cost

    stmt = select(
        dim_col.label("dimension"),
        CostCharge.billing_currency.label("currency"),
        func.coalesce(func.sum(cost_col), 0).label("total_cost"),
        func.count(CostCharge.id).label("line_count"),
    )

    # 2. Apply filters (7 dimensions + date range with overlap semantics)
    stmt = _apply_filters(stmt, ...)

    # 3. Group by dimension + currency
    stmt = stmt.group_by(dim_col, CostCharge.billing_currency).order_by(desc("total_cost"))

    # 4. Top-N (50) + "Other" overflow
    top = merged[:50]
    overflow = merged[50:]
    if overflow:
        groups.append({"dimension": "Other", "total_cost": sum(o["total_cost"] for o in overflow)})

    # 5. Multi-currency awareness
    if len(currencies_seen) > 1:
        # Report per-currency breakdown, not blended total
        result["mixed_currency"] = True
        result["total_by_currency"] = grand_total_by_currency

    return result
```

### Time-Series Aggregation (`timeseries_focus()`)

```python
async def timeseries_focus(db, *, tenant_id, granularity="day", ...):
    # Bucket by day/week/month using date_trunc()
    if granularity == "month":
        bucket = func.date_trunc("month", CostCharge.charge_period_start)
    elif granularity == "week":
        bucket = func.date_trunc("week", CostCharge.charge_period_start)
    else:
        bucket = func.date_trunc("day", CostCharge.charge_period_start)

    # Returns: {"points": [{"period": "2026-01-01", "total_cost": 1234.56, "line_count": 42}, ...]}
```

### Filter Semantics

Date range uses **overlap** semantics (not containment):

```python
# A charge spanning Jul 1 - Aug 1 must show up for any range that touches July
if period_start is not None:
    stmt = stmt.where(CostCharge.charge_period_end >= period_start)
if period_end is not None:
    stmt = stmt.where(CostCharge.charge_period_start <= period_end)
```

### Dimension Autocomplete

```python
async def get_focus_dimension_values(db, *, tenant_id, dimension, limit=100):
    # Returns distinct values ordered by total cost descending
    # Powers filter dropdowns in the UI
    col = FOCUS_DIMENSIONS.get(dimension)
    stmt = select(col, func.sum(CostCharge.effective_cost).label("total_cost"))
    stmt = stmt.group_by(col).order_by(desc("total_cost")).limit(limit)
```

---

## 10. Allocation — Virtual Tags & Chargeback

### Virtual Tag Engine

12 FOCUS dimensions usable for tagging:

```python
FOCUS_FIELD_KEYS = [
    "service_name", "service_category", "provider_name",
    "region_name", "region_id", "sub_account_id", "sub_account_name",
    "resource_id", "resource_name", "resource_type", "sku_id",
    "charge_category",
]
```

Also supports `tag.*` keys (provider-native resource tags).

### Rule Operators

| Operator | Example | Match Logic |
|----------|---------|-------------|
| `eq` | `{"field": "service_name", "op": "eq", "value": "Amazon EC2"}` | Exact match |
| `contains` | `{"field": "service_name", "op": "contains", "value": "ec2"}` | Case-insensitive substring |
| `regex` | `{"field": "resource_id", "op": "regex", "value": "i-[a-f0-9]+"}` | Regex match |
| `in` | `{"field": "provider_name", "op": "in", "value": ["aws", "gcp"]}` | Value in list |
| `exists` | `{"field": "tag.Environment", "op": "exists"}` | Field is non-null and non-empty |

### First-Match-Wins Funnel

```python
def evaluate_virtual_tag(charge, virtual_tag):
    for rule in virtual_tag.rules:  # Ordered list
        if _match_rule(charge, rule):
            return rule.get("assign") or rule.get("value_label") or "matched"
    return virtual_tag.untagged_label or "untagged"
```

### Shared Cost Reallocation

Three strategies for redistributing shared costs:

```python
def _apply_reallocation(buckets, realloc):
    source = realloc.source_value
    amount = buckets[source]
    targets = realloc.targets

    if strategy == "even":
        share = amount / len(targets)
        for t in targets: result[t] += share

    elif strategy == "manual_percent":
        for t in targets:
            pct = Decimal(str(t["percent"])) / 100
            result[t["value"]] += amount * pct

    elif strategy == "telemetry_weight":
        total_w = sum(t["weight"] for t in targets)
        for t in targets:
            w = Decimal(str(t["weight"]))
            result[t["value"]] += amount * (w / total_w)
```

### Showback vs Chargeback

- **Showback**: Informational — read-only cost visibility (no billing)
- **Chargeback**: Billable — cost attribution to teams for actual billing

```python
async def build_chargeback_statement(db, *, tenant_id, virtual_tag_id, mode="showback"):
    allocation = await allocate_charges(db, ...)
    statements = [
        {
            "cost_center": a["value"],
            "amount": a["amount"],
            "billable": mode == "chargeback" and not a.get("is_untagged"),
            "status": "informational" if mode == "showback" else "chargeback",
        }
        for a in allocation.get("allocations")
    ]
```

---

## 11. Unit Economics — How to Build It Right

### Current Implementation

```python
async def build_unit_economics(db, *, tenant_id, user_id, dimension=None,
                               virtual_tag_id=None, revenue_metric_key=None):
    # 1. Get total spend from FOCUS (or fallback to billing overview)
    focus_total, focus_currency = await _focus_total_spend(db, tenant_id)
    total_spend = focus_total if focus_total > 0 else overview["actual_spend"]

    # 2. Load business metrics (user-defined metrics like "requests", "users", "GB")
    metrics = await list_business_metrics(db, tenant_id)

    # 3. For each metric, compute cost-per-unit
    for metric in metrics:
        metric_spend, _, _ = await _resolve_metric_spend(db, tenant_id, metric, ...)
        qty = Decimal(str(metric.quantity))
        cost_per = float(metric_spend / qty) if qty > 0 else None

        # 4. Compute trend vs prior period
        prev = find_prior_period(metric)
        if prev:
            prev_cpu = float(prev_spend / prev.quantity)
            trend_percent = ((cost_per - prev_cpu) / prev_cpu) * 100

        # 5. Compute margin (if revenue metric provided)
        if revenue is not None:
            margin = float(((revenue - metric_spend) / revenue) * 100)

        # 6. Compute target variance
        target = metric.extra.get("target_cost_per_unit")
        variance = compute_target_variance(cost_per, target)

    # 7. If dimension specified, compute per-segment unit economics
    if dimension or virtual_tag_id:
        allocation = await allocate_charges(db, ...)
        for alloc in allocation["allocations"]:
            segments.append({
                "cost_per_unit": float(alloc["amount"] / qty),
                # ...
            })
```

### How to Improve Unit Economics with FOCUS 1.4

FOCUS 1.4 adds columns that make unit economics more powerful:

| Column | Unit Economics Use |
|--------|-------------------|
| `ConsumedQuantity` + `ConsumedUnit` | Native usage metrics (tokens, GB, requests) |
| `PricingQuantity` + `PricingUnit` | Pricing-level metrics |
| `SkuPriceDetails` | Understand what's being metered (CoreCount, MemorySize) |
| `CommitmentDiscountQuantity` + `CommitmentDiscountUnit` | Commitment-adjusted unit cost |
| `ListUnitPrice` | On-demand unit price baseline |
| `ContractedUnitPrice` | Negotiated unit price |

### Recommended Unit Economics Architecture

```
┌─────────────────────────────────────────┐
│ Unit Economics Engine                    │
├─────────────────────────────────────────┤
│ 1. Metric Resolution                     │
│    - BusinessMetric table (user-defined) │
│    - FOCUS ConsumedQuantity (native)     │
│    - SKU Price Details (provider-defined)│
├─────────────────────────────────────────┤
│ 2. Cost Scoping                          │
│    - Total (all charges)                 │
│    - Virtual Tag (allocated subset)      │
│    - Provider (single provider)          │
│    - Service (single service)            │
├─────────────────────────────────────────┤
│ 3. Calculation                           │
│    - cost_per_unit = scoped_spend / qty  │
│    - trend = (current - prior) / prior   │
│    - margin = (revenue - spend) / revenue│
│    - target_variance = actual - target   │
├─────────────────────────────────────────┤
│ 4. Segmentation                          │
│    - By virtual tag value                │
│    - By provider                         │
│    - By service category                 │
│    - By region                           │
└─────────────────────────────────────────┘
```

---

## 12. Enterprise Gating & Multi-Tenancy

### Feature Flags

```python
# feature_flag_catalog.py:115
{
    "id": "focus-explorer",
    "name": "FOCUS Explorer",
    "description": "Multi-cloud cost analysis with FOCUS 1.2",
    "feature_flag": "FOCUS_EXPLORER",
    "section": "Cost Intelligence",
    "permission": "costs.analysis",
    "sales_lockable": True,
}
```

### Enterprise Entitlements

```python
# enterprise_entitlements.py:18
("focus-explorer", "FOCUS Explorer")
```

### Tenant Scoping

Every query is scoped to the authenticated user's tenant:

```python
# focus_routes.py:89
tenant_id = resolve_active_tenant_id(current_user)
return await summarize_focus(db, tenant_id=tenant_id, ...)
```

### Admin-Only Operations

```python
# focus_routes.py:356
admin: User = Depends(require_admin)
# Only admins can trigger ingestion
```

### Multi-Tenant Isolation

```sql
-- Every query includes tenant_id
WHERE cost_charges.tenant_id = $tenant_id

-- Unique constraint prevents cross-tenant collisions
UNIQUE(tenant_id, provider_name, provider_line_item_id)
```

---

## 13. Feature Roadmap — Phase by Phase

### Phase 1: FOCUS 1.4 Column Upgrade (Weeks 1-3)

**Goal**: Add missing columns to `cost_charges` and update normalization.

| Task | File | Effort |
|------|------|--------|
| Add 25+ new columns to CostCharge model | `models.py` | 1 day |
| Create migration 0021 | `alembic/versions/0021_focus_1_4_columns.py` | 1 day |
| Update `normalize_charge_row()` for new columns | `schema.py` | 2 days |
| Update AWS CUR adapter for invoice/contract columns | `ingest.py` | 1 day |
| Update Azure FOCUS adapter for all 1.4 columns | `ingest.py` | 1 day |
| Update GCP BigQuery adapter for billing account/SKU | `ingest.py` | 1 day |
| Add new indexes (invoice_id, billing_account_id, contract_id) | `charge_store.py` | 1 day |
| Tests for new column normalization | `test_focus_ingest.py` | 2 days |

**New columns added**:

```sql
-- Account hierarchy
billing_account_id VARCHAR(255)
billing_account_name VARCHAR(255)
billing_account_type VARCHAR(64)
sub_account_type VARCHAR(64)

-- Invoice
invoice_id VARCHAR(512)
invoice_detail_id VARCHAR(512)

-- Pricing
pricing_category VARCHAR(64)
list_unit_price NUMERIC(18,8)
contracted_unit_price NUMERIC(18,8)
pricing_currency VARCHAR(16)

-- SKU
sku_meter VARCHAR(255)
sku_price_details JSONB

-- Charge
charge_frequency VARCHAR(32)

-- Location
availability_zone VARCHAR(128)

-- Provider
service_provider_name VARCHAR(128)
host_provider_name VARCHAR(128)

-- Commitment
commitment_discount_name VARCHAR(255)
commitment_discount_quantity NUMERIC(18,6)
commitment_discount_unit VARCHAR(64)
commitment_program_eligibility JSONB

-- Contract
contract_id VARCHAR(255)
contract_applied JSONB

-- Allocation
allocated_method_id VARCHAR(255)
allocated_method_details JSONB
allocated_resource_id VARCHAR(512)
allocated_resource_name VARCHAR(512)
allocated_tags JSONB
```

### Phase 2: Invoice Reconciliation (Weeks 4-5)

**Goal**: New `focus_invoices` table + reconciliation endpoint + dashboard.

| Task | File | Effort |
|------|------|--------|
| Create `focus_invoices` model | `models.py` | 0.5 day |
| Create migration 0022 | `alembic/versions/0022_focus_invoices.py` | 0.5 day |
| Create invoice service | `services/focus/invoice_service.py` | 2 days |
| Create invoice routes | `routes/invoice_routes.py` | 1 day |
| Populate invoice_id from provider exports | `ingest.py` | 1 day |
| Reconciliation endpoint (variance detection) | `invoice_routes.py` | 1 day |
| Frontend reconciliation dashboard | `components/finops/invoice-reconciliation.tsx` | 3 days |
| Tests | `test_invoice_routes.py` | 1 day |

**New table**:

```sql
CREATE TABLE focus_invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    invoice_id VARCHAR(512) NOT NULL,
    invoice_issue_date TIMESTAMPTZ,
    payment_due_date TIMESTAMPTZ,
    payment_terms VARCHAR(64),
    payment_currency VARCHAR(16),
    billing_currency VARCHAR(16),
    billed_cost NUMERIC(20,8),
    line_item_count INTEGER,
    provider_name VARCHAR(64),
    connection_id UUID REFERENCES cloud_connections(id),
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, invoice_id)
);
```

**Reconciliation logic**:

```python
@router.get("/reconciliation")
async def reconcile_invoices(period_start, period_end, ...):
    # 1. JOIN cost_charges.invoice_id = focus_invoices.invoice_id
    # 2. Compare sum(EffectiveCost) per invoice vs billed_cost
    # 3. Flag variances > rounding tolerance (0.01)
    # 4. Return: {matched: [...], variances: [...], summary: {...}}
```

### Phase 3: Contract Commitment Intelligence (Weeks 6-8)

**Goal**: New `focus_commitments` table + coverage dashboard + cross-provider comparison.

| Task | File | Effort |
|------|------|--------|
| Create `focus_commitments` model | `models.py` | 0.5 day |
| Create migration 0023 | `alembic/versions/0023_focus_commitments.py` | 0.5 day |
| Create commitment service | `services/focus/commitment_service.py` | 3 days |
| Create commitment routes | `routes/commitment_routes.py` | 1 day |
| Parse commitment data from AWS (RI/SP), GCP (CUD), Azure (Ri) | `ingest.py` | 2 days |
| Coverage dashboard endpoint | `commitment_routes.py` | 1 day |
| Cross-provider comparison endpoint | `commitment_routes.py` | 1 day |
| Frontend commitment dashboard | `components/finops/commitment-dashboard.tsx` | 4 days |
| Tests | `test_commitment_routes.py` | 1 day |

**New table**:

```sql
CREATE TABLE focus_commitments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    contract_commitment_id VARCHAR(512) NOT NULL,
    provider_name VARCHAR(64),
    commitment_type VARCHAR(64),  -- Reservation, SavingsPlan, CUD
    commitment_category VARCHAR(32),  -- Usage or Spend
    start_date TIMESTAMPTZ,
    end_date TIMESTAMPTZ,
    quantity NUMERIC(20,8),
    unit VARCHAR(64),
    discount_percentage NUMERIC(5,2),
    fulfillment_interval VARCHAR(32),
    lifecycle_status VARCHAR(32),  -- Active, Expired, Cancelled
    payment_model VARCHAR(32),
    cost NUMERIC(20,8),
    currency VARCHAR(16),
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, contract_commitment_id)
);
```

### Phase 4: AI Cost Intelligence (Weeks 9-10)

**Goal**: Ingest from AI providers, build AI cost dashboard.

| Task | File | Effort |
|------|------|--------|
| Create AI ingestion adapter (OpenAI, Anthropic, Cohere) | `ingest.py` | 2 days |
| Map tokens to ConsumedQuantity + ConsumedUnit | `schema.py` | 0.5 day |
| Map models to SkuId + SkuPriceDetails | `schema.py` | 0.5 day |
| AI cost dashboard endpoint | `routes/focus_routes.py` | 1 day |
| Frontend AI cost dashboard | `components/finops/ai-cost-dashboard.tsx` | 3 days |
| Tests | `test_ai_ingest.py` | 1 day |

### Phase 5: Anomaly Detection (Week 11)

**Goal**: Z-score detection on daily spend, route to FinOps agent.

| Task | File | Effort |
|------|------|--------|
| Daily spend aggregation query | `services/focus/anomaly_service.py` | 1 day |
| Z-score detection logic | `services/focus/anomaly_service.py` | 1 day |
| Anomaly routing to FinOps agent | `services/focus/anomaly_service.py` | 1 day |
| Anomaly endpoint | `routes/anomaly_routes.py` | 0.5 day |
| Frontend anomaly feed | `components/finops/anomaly-feed.tsx` | 2 days |
| Tests | `test_anomaly_service.py` | 0.5 day |

### Phase 6: Budget vs Actuals (Week 12)

**Goal**: New `budgets` table + budget tracking.

### Phase 7: FOCUS Validator Integration (Week 13)

**Goal**: Validate `cost_charges` against FOCUS 1.4 requirements, return conformance score.

### Phase 8: FOCUS MCP for AI Agent (Week 14)

**Goal**: Connect FinOps agent to `https://focus.finops.org/wp-json/focus/v1/mcp` for spec-aware Q&A.

---

## 14. How FOCUS Solves Problems We Haven't Tackled Yet

### Invoice Reconciliation

**Problem**: "Do our provider invoices match what we're being charged?"

**FOCUS Solution**: Join Cost and Usage to Invoice Detail via `InvoiceId` and `InvoiceDetailId`.

```sql
SELECT
  i.InvoiceID,
  i.BilledCost as invoice_amount,
  SUM(c.EffectiveCost) as calculated_amount,
  ABS(i.BilledCost - SUM(c.EffectiveCost)) as variance
FROM focus_invoices i
JOIN cost_charges c ON c.invoice_id = i.InvoiceID
WHERE i.tenant_id = $tenant_id
GROUP BY i.InvoiceID, i.BilledCost
HAVING ABS(i.BilledCost - SUM(c.EffectiveCost)) > 0.01
```

### Commitment Coverage

**Problem**: "What percentage of our spend is covered by commitments? What's eligible but not covered?"

**FOCUS Solution**: Use `CommitmentProgramEligibilityDetails` column — identifies which commitment programs a charge qualifies for, whether or not one is applied.

```sql
SELECT
  ServiceName,
  SUM(EffectiveCost) as total,
  SUM(CASE WHEN CommitmentDiscountId IS NOT NULL THEN EffectiveCost ELSE 0 END) as covered,
  SUM(CASE WHEN CommitmentProgramEligibilityDetails IS NOT NULL
           AND CommitmentDiscountId IS NULL THEN EffectiveCost ELSE 0 END) as eligible_uncovered,
  ROUND(covered / total * 100, 2) as coverage_pct
FROM cost_charges
GROUP BY ServiceName
```

### Effective Savings Rate

**Problem**: "What's our real savings from commitments vs on-demand?"

**FOCUS Solution**: Compare `EffectiveCost` (with commitment) vs `ListCost` (without commitment).

```sql
SELECT
  SUM(CASE WHEN CommitmentDiscountStatus = 'Used' THEN EffectiveCost ELSE 0 END) as commitment_cost,
  SUM(CASE WHEN CommitmentDiscountStatus = 'Used' THEN ListCost ELSE 0 END) as list_cost,
  ROUND((1 - commitment_cost / list_cost) * 100, 2) as savings_rate
FROM cost_charges
WHERE CommitmentDiscountId IS NOT NULL
```

### AI Token Lifecycle

**Problem**: "How are we consuming AI credits/tokens? What's the cost per token?"

**FOCUS Solution**: Use `ConsumedQuantity` + `ConsumedUnit` for token tracking, `SkuPriceDetails` for model-level pricing.

```sql
SELECT
  ServiceName as provider,
  SkuPriceDetails->>'model' as model,
  SUM(ConsumedQuantity) as tokens_used,
  SUM(EffectiveCost) as total_cost,
  SUM(EffectiveCost) / NULLIF(SUM(ConsumedQuantity), 0) as cost_per_token
FROM cost_charges
WHERE ServiceCategory = 'AI and Machine Learning'
GROUP BY ServiceName, SkuPriceDetails->>'model'
```

### Split Cost Allocation

**Problem**: "How did the provider split shared costs across our workloads?"

**FOCUS Solution**: Use `AllocatedMethodDetails` and `AllocatedMethodId` columns.

```sql
SELECT
  ResourceName,
  AllocatedMethodId,
  AllocatedMethodDetails,
  SUM(EffectiveCost) as allocated_cost
FROM cost_charges
WHERE AllocatedMethodId IS NOT NULL
GROUP BY ResourceName, AllocatedMethodId, AllocatedMethodDetails
```

### Covering/Covered Charges

**Problem**: "How do we avoid double-counting commitment purchases and their usage?"

**FOCUS Solution**: A covering charge has `EffectiveCost = 0`. The covered charges draw from it. Total `EffectiveCost` across covering + covered = same total.

```sql
-- Covering charges (purchases)
SELECT * FROM cost_charges WHERE ChargeCategory = 'Purchase' AND EffectiveCost = 0

-- Covered charges (usage drawn from commitment)
SELECT * FROM cost_charges WHERE CommitmentDiscountStatus = 'Used'
```

---

## 15. Database Migrations Required

### Migration 0021: FOCUS 1.4 Columns

```sql
-- Account hierarchy
ALTER TABLE cost_charges ADD COLUMN billing_account_id VARCHAR(255);
ALTER TABLE cost_charges ADD COLUMN billing_account_name VARCHAR(255);
ALTER TABLE cost_charges ADD COLUMN billing_account_type VARCHAR(64);
ALTER TABLE cost_charges ADD COLUMN sub_account_type VARCHAR(64);

-- Invoice
ALTER TABLE cost_charges ADD COLUMN invoice_id VARCHAR(512);
ALTER TABLE cost_charges ADD COLUMN invoice_detail_id VARCHAR(512);

-- Pricing
ALTER TABLE cost_charges ADD COLUMN pricing_category VARCHAR(64);
ALTER TABLE cost_charges ADD COLUMN list_unit_price NUMERIC(18,8);
ALTER TABLE cost_charges ADD COLUMN contracted_unit_price NUMERIC(18,8);
ALTER TABLE cost_charges ADD COLUMN pricing_currency VARCHAR(16);

-- SKU
ALTER TABLE cost_charges ADD COLUMN sku_meter VARCHAR(255);
ALTER TABLE cost_charges ADD COLUMN sku_price_details JSONB;

-- Charge
ALTER TABLE cost_charges ADD COLUMN charge_frequency VARCHAR(32);

-- Location
ALTER TABLE cost_charges ADD COLUMN availability_zone VARCHAR(128);

-- Provider
ALTER TABLE cost_charges ADD COLUMN service_provider_name VARCHAR(128);
ALTER TABLE cost_charges ADD COLUMN host_provider_name VARCHAR(128);

-- Commitment
ALTER TABLE cost_charges ADD COLUMN commitment_discount_name VARCHAR(255);
ALTER TABLE cost_charges ADD COLUMN commitment_discount_quantity NUMERIC(18,6);
ALTER TABLE cost_charges ADD COLUMN commitment_discount_unit VARCHAR(64);
ALTER TABLE cost_charges ADD COLUMN commitment_program_eligibility JSONB;

-- Contract
ALTER TABLE cost_charges ADD COLUMN contract_id VARCHAR(255);
ALTER TABLE cost_charges ADD COLUMN contract_applied JSONB;

-- Allocation
ALTER TABLE cost_charges ADD COLUMN allocated_method_id VARCHAR(255);
ALTER TABLE cost_charges ADD COLUMN allocated_method_details JSONB;
ALTER TABLE cost_charges ADD COLUMN allocated_resource_id VARCHAR(512);
ALTER TABLE cost_charges ADD COLUMN allocated_resource_name VARCHAR(512);
ALTER TABLE cost_charges ADD COLUMN allocated_tags JSONB;

-- Indexes
CREATE INDEX ix_cost_charges_invoice ON cost_charges(tenant_id, invoice_id);
CREATE INDEX ix_cost_charges_billing_account ON cost_charges(tenant_id, billing_account_id);
CREATE INDEX ix_cost_charges_contract ON cost_charges(tenant_id, contract_id);
CREATE INDEX ix_cost_charges_service_provider ON cost_charges(tenant_id, service_provider_name);
```

### Migration 0022: Invoice Detail

```sql
CREATE TABLE focus_invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    invoice_id VARCHAR(512) NOT NULL,
    invoice_issue_date TIMESTAMPTZ,
    payment_due_date TIMESTAMPTZ,
    payment_terms VARCHAR(64),
    payment_currency VARCHAR(16),
    billing_currency VARCHAR(16),
    billed_cost NUMERIC(20,8),
    line_item_count INTEGER,
    provider_name VARCHAR(64),
    connection_id UUID REFERENCES cloud_connections(id),
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, invoice_id)
);
CREATE INDEX ix_focus_invoices_tenant ON focus_invoices(tenant_id);
CREATE INDEX ix_focus_invoices_provider ON focus_invoices(tenant_id, provider_name);
```

### Migration 0023: Contract Commitments

```sql
CREATE TABLE focus_commitments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    contract_commitment_id VARCHAR(512) NOT NULL,
    provider_name VARCHAR(64),
    commitment_type VARCHAR(64),
    commitment_category VARCHAR(32),
    start_date TIMESTAMPTZ,
    end_date TIMESTAMPTZ,
    quantity NUMERIC(20,8),
    unit VARCHAR(64),
    discount_percentage NUMERIC(5,2),
    fulfillment_interval VARCHAR(32),
    lifecycle_status VARCHAR(32),
    payment_model VARCHAR(32),
    cost NUMERIC(20,8),
    currency VARCHAR(16),
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, contract_commitment_id)
);
CREATE INDEX ix_focus_commitments_tenant ON focus_commitments(tenant_id);
CREATE INDEX ix_focus_commitments_provider ON focus_commitments(tenant_id, provider_name);
CREATE INDEX ix_focus_commitments_type ON focus_commitments(tenant_id, commitment_type);
```

### Migration 0024: Budgets

```sql
CREATE TABLE focus_budgets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(128) NOT NULL,
    scope_type VARCHAR(32) NOT NULL,  -- sub_account, service, provider
    scope_value VARCHAR(255) NOT NULL,
    amount NUMERIC(20,8) NOT NULL,
    currency VARCHAR(16) NOT NULL DEFAULT 'USD',
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    alert_threshold NUMERIC(5,2) DEFAULT 80.0,
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX ix_focus_budgets_tenant ON focus_budgets(tenant_id);
```

---

## 16. API Surface — Current & Proposed

### Current Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/finops/focus/summary` | GET | Group-by aggregation |
| `/api/v1/finops/focus/timeseries` | GET | Time-bucketed aggregation |
| `/api/v1/finops/focus/charges` | GET | Paginated line items |
| `/api/v1/finops/focus/dimension-values` | GET | Filter autocomplete |
| `/api/v1/finops/focus/export` | GET | CSV export |
| `/api/v1/finops/focus/export-config/{id}` | GET/PUT | Export config |
| `/api/v1/finops/focus/ingest` | POST | Trigger ingestion |

### Proposed New Endpoints

| Endpoint | Method | Purpose | Phase |
|----------|--------|---------|-------|
| `/api/v1/finops/focus/reconciliation` | GET | Invoice reconciliation | P2 |
| `/api/v1/finops/focus/reconciliation/{invoice_id}` | GET | Single invoice reconciliation | P2 |
| `/api/v1/finops/focus/commitments` | GET | List commitments | P3 |
| `/api/v1/finops/focus/commitments/coverage` | GET | Coverage analysis | P3 |
| `/api/v1/finops/focus/commitments/comparison` | GET | Cross-provider comparison | P3 |
| `/api/v1/finops/focus/ai-costs` | GET | AI cost breakdown | P4 |
| `/api/v1/finops/focus/ai-costs/by-model` | GET | Cost per model | P4 |
| `/api/v1/finops/focus/anomalies` | GET | Anomaly detection | P5 |
| `/api/v1/finops/focus/budgets` | GET/POST | Budget management | P6 |
| `/api/v1/finops/focus/budgets/actuals` | GET | Budget vs actuals | P6 |
| `/api/v1/finops/focus/validate` | GET | FOCUS conformance check | P7 |
| `/api/v1/finops/focus/explore` | POST | Dynamic query (NL→SQL) | P8 |

---

## 17. Frontend — What Exists & What's Missing

### Current Components

| Component | File | Lines | What It Does |
|-----------|------|-------|-------------|
| `CostExplorer` | `cost-explorer.tsx` | 918 | Main explorer: filter sidebar, group-by, chart, table, line items, CSV export, saved views |
| `FilterCombobox` | `filter-combobox.tsx` | — | Autocomplete filter dropdowns |
| `FocusPage` | `focus/page.tsx` | — | Page wrapper with EnterpriseGate |

### What's Missing

| Component | Phase | Priority |
|-----------|-------|----------|
| Invoice Reconciliation Dashboard | P2 | P1 |
| Commitment Coverage Dashboard | P3 | P1 |
| Cross-Provider Commitment Comparison | P3 | P1 |
| AI Cost Dashboard (by model, by token) | P4 | P1 |
| Anomaly Feed (daily spikes) | P5 | P2 |
| Budget vs Actuals Dashboard | P6 | P2 |
| FOCUS Conformance Score Display | P7 | P3 |
| Sankey Cost Flow Visualization | E03 | P2 |
| Saved Views (DB-backed) | E03 | P2 |
| NL→Query Interface | E10 | P3 |

---

## 18. Testing Strategy

### Backend Tests

| Test File | Coverage |
|-----------|----------|
| `test_focus_routes.py` | Route-level: tenant scoping, admin gating, pagination, dimension-values |
| `test_focus_ingest.py` | Normalization idempotency, BigQuery table validation, timestamp parsing, bulk upsert, tenant isolation |
| `test_finops_tool_result_formatter.py` | Tool result formatting |
| `test_azure_finops_fixes.py` | Azure integration |

### Proposed New Tests

| Test File | Coverage | Phase |
|-----------|----------|-------|
| `test_focus_1_4_columns.py` | New column normalization, migration upgrade | P1 |
| `test_invoice_reconciliation.py` | Invoice join, variance detection | P2 |
| `test_commitment_coverage.py` | Coverage calculation, cross-provider comparison | P3 |
| `test_ai_cost_ingestion.py` | Token parsing, model mapping | P4 |
| `test_anomaly_detection.py` | Z-score calculation, threshold detection | P5 |
| `test_budget_actuals.py` | Budget join, variance reporting | P6 |
| `test_focus_validator.py` | Conformance scoring | P7 |

### Validation Commands

```bash
# Backend
cd backend
uv run alembic upgrade head
uv run ruff check .
uv run ruff format --check .
uv run mypy main.py
uv run pytest

# Frontend
cd frontend
npm run lint
npm run type-check
npm test
npm run build
```

---

## 19. Competitive Position

### What Makes This Different

| Feature | Our Implementation | Vantage | Amnic | CloudHealth | Native CE |
|---------|-------------------|---------|-------|-------------|-----------|
| FOCUS 1.2 compliant | Yes | Partial | No | No | No |
| Multi-cloud ingestion | Yes (7 adapters) | Yes | Limited | Yes | Single |
| Virtual tag allocation | Yes (12 dimensions) | Limited | No | Yes | No |
| Invoice reconciliation | Planned (P2) | No | No | No | Limited |
| Commitment coverage | Planned (P3) | No | No | Limited | Yes (single) |
| AI cost tracking | Planned (P4) | No | No | No | No |
| Unit economics | Yes (existing) | Limited | Limited | Limited | No |
| Showback/chargeback | Yes (existing) | No | No | Yes | No |
| Enterprise RBAC | Yes | Yes | Yes | Yes | Yes |
| Open source | Yes | No | No | No | No |

### Enterprise Sales Angle

FOCUS conformance is becoming a procurement requirement. The European Parliament, UnitedHealth Group, Heineken, Zoom, GitLab, and STMicroelectronics all adopted FOCUS because:

1. **Standardization**: One schema across all vendors
2. **Automation**: Removes manual data exchange
3. **Accuracy**: Reduces errors and discrepancies
4. **Scalability**: Supports multi-cloud environments

### Key Differentiator

We're building the **FOCUS-native FinOps platform** — not just ingesting FOCUS data but deeply understanding all 4 datasets, 107 columns, and 59 use cases. This is the only platform that:

1. Ingests from 7 adapters (AWS CE + CUR, GCP BQ + billing, Azure Blob + billing, SaaS)
2. Normalizes to FOCUS 1.2 with idempotent upserts
3. Allocates via 12-dimension virtual tag engine with 5 rule operators
4. Computes unit economics with trends and target variance
5. Enterprise-gated with feature flags, permissions, and sales-lock surfaces
6. Planned: Invoice reconciliation, commitment coverage, AI cost tracking, anomaly detection

---

## Appendix: Key Resources

| Resource | URL |
|----------|-----|
| FOCUS Specification | https://focus.finops.org/focus-specification/ |
| Column Library | https://focus.finops.org/focus-columns/ |
| Use Case Library | https://focus.finops.org/use-cases/ |
| FOCUS Validator | https://focus.finops.org/focus-validator/ |
| FOCUS MCP Server | `https://focus.finops.org/wp-json/focus/v1/mcp` |
| Data Model Spreadsheet | https://docs.google.com/spreadsheets/d/1duuzCD4jovfKjfsfVBlWgxPCOOOqsWAXlSA2XLw-iuw |
| GitHub Repository | https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec |
| FinOps Converters | https://github.com/finopsfoundation/focus_converters |
| FinOps ToolKit | https://microsoft.github.io/finops-toolkit/ |
| E03 Epic | `docs/finops-strategy/epics/E03-cost-usage-explorer-v2.md` |
