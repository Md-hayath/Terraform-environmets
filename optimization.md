# Digitomics InfraOps — Comprehensive Optimisation Gap Analysis

> **Prepared by:** Principal FinOps Architect / Staff Software Engineer
> **Date:** July 2026
> **Objective:** Identify every optimisation capability that can realistically be built on the existing platform architecture — grounded in current code, data, and business logic — without copying competitors.

---

## Table of Contents

1. [Complete Architecture Understanding](#1-complete-architecture-understanding)
2. [Existing Optimisation Engine Analysis](#2-existing-optimisation-engine-analysis)
3. [Data Capability Assessment](#3-data-capability-assessment)
4. [Recommendation Engine Design](#4-recommendation-engine-design)
5. [Every Optimisation Opportunity Supported by Current Platform](#5-every-optimisation-opportunity-supported-by-current-platform)
6. [Feature Gap Analysis](#6-feature-gap-analysis)
7. [Confidence Model for Recommendations](#7-confidence-model-for-recommendations)
8. [Recommendation Prioritisation Framework](#8-recommendation-prioritisation-framework)
9. [Implementation Roadmap](#9-implementation-roadmap)
10. [Product Differentiation Strategy](#10-product-differentiation-strategy)

---

## 1. Complete Architecture Understanding

### 1.1 Platform Overview

Digitomics InfraOps is a full-stack, multi-tenant FinOps/InfraOps platform with:

- **Backend:** FastAPI (Python) with async SQLAlchemy, LangGraph agents, Redis caching, pgvector RAG
- **Frontend:** Next.js 16 / React 19 with TypeScript, Zustand stores, TanStack React Query, Recharts
- **Database:** PostgreSQL 16 with `pgvector` extension — single source of truth
- **Cache/Queue:** Redis — ephemeral agent state, worker locks, notification queues, rate limits
- **Deployment:** Docker Compose (monolith), CI via GitHub Actions

### 1.2 Data Architecture (Optimisation-Relevant Tables)

| Table | Purpose | Optimisation Relevance |
|-------|---------|----------------------|
| `cost_charges` (FOCUS) | Granular line-item cost data | **PRIMARY** — all cost analysis, allocation, forecasting |
| `cost_observations` | Daily normalized cost facts | **PRIMARY** — forecasting, anomaly detection |
| `cost_anomalies` | Detected cost anomalies | **PRIMARY** — action center items |
| `cost_forecast_runs` / `cost_forecast_points` | Statistical forecast results | **PRIMARY** — prediction, trend analysis |
| `optimization_recommendations` | Generated recommendations | **PRIMARY** — the optimisation engine output |
| `focus_commitments` | RI/SP commitment records | Commitment optimisation |
| `focus_budgets` | Budget definitions | Budget vs actual tracking |
| `billing_records` | Provider billing lines | Cost aggregation, tool-level spend |
| `cloud_connections` | Provider connections | Data source for all cloud cost |
| `GCPComputeInstance` | GCE inventory | Rightsizing input |
| `GCPGKECluster` | GKE cluster inventory | Container optimisation |
| `GCPStorageBucket` | GCS bucket inventory | Storage optimisation |
| `GCPBigQueryDataset` | BQ dataset inventory | Data warehouse optimisation |
| `virtual_tags` | Allocation rules engine | Showback/chargeback |
| `cost_allocation_rules` | Legacy allocation rules | Showback fallback |
| `business_metrics` | Unit economics metrics | Unit cost analysis |
| `action_item_states` | Action Center workflow state | Recommendation workflow |
| `notification_events` | Notification records | Alerting on optimisation |
| `cost_pricing` | Reference pricing data | Savings estimation |

### 1.3 Current Optimisation Data Flow

```
Cloud Providers (AWS/GCP/Azure)
    │
    ▼
Synced Inventory Data ──────► GCPComputeInstance, GCPGKECluster, etc.
    │
    ▼
FOCUS Ingestion ────────────► cost_charges (line-item cost data)
    │
    ▼
Cost Observations ──────────► cost_observations (daily rollup)
    │
    ▼
Cost Intelligence Scheduler ─► cost_forecast_*, cost_anomalies
    │
    ▼
AWS Dashboard Snapshot ─────► waste_scan, rightsizing, commitment_coverage
    │
    ▼
Recommendation Engine ──────► optimization_recommendations (autostop, schedule, SP, RI)
    │
    ▼
Action Center Assembly ─────► ActionCenterItem (savings, anomaly, budget, approval, permission, resource_waste)
    │
    ▼
Frontend Display ───────────► /dashboard/optimization, /dashboard/action-center, /dashboard/focus/*
```

### 1.4 Current Optimisation Capabilities (What Exists Today)

**✅ Implemented Features:**

1. **Cost Anomaly Detection:** StatsForecast-based statistical detection with conformal prediction intervals, residual analysis, changepoint detection. Scores anomalies across 5 dimensions (interval breach, residual, changepoint, impact, persistence). Generates contributor analysis.

2. **Cost Forecasting:** Auto-selected statistical models (AutoARIMA, AutoETS, AutoTheta, MSTL, HoltWinters, SeasonalNaive) with ensemble fallback and conformal calibration. 30-day horizon.

3. **AWS Dashboard Integration:** Daily AWS account snapshot via Cost Explorer API providing waste scan findings, rightsizing recommendations, commitment coverage data, anomaly alerts (CE anomalies + z-score + spike detection), inventory stats (unattached EBS, stopped EC2).

4. **Optimization Recommendations (4 types):**
   - `autostop` — idle non-prod resource candidates for AutoStopping
   - `schedule` — business-hours start/stop for underutilized compute
   - `savings_plan` — advisory SP purchase recommendations
   - `reserved_instance` — advisory RI purchase recommendations
   - Hardcoded default savings per waste category: idle_ec2=$50, unused_lambda=$5, idle_rds=$100, unassociated_eip=$3.65, orphaned_snapshot=$10

5. **Action Center:** Unified feed of savings opportunities, anomalies, budget risks, approval requests, permission gaps, resource waste. Status workflow (open→in_progress→resolved/dismissed). Impact tracking with baseline capture.

6. **Showback/Chargeback:** Virtual tag allocation engine. Weighted allocation rules. `CostCharge`-based allocation with tag matching and relation-based grouping.

7. **Unit Economics:** Cost-per-unit with trend, target variance, gross margin computation. Revenue-based margin analysis.

8. **Budget vs Actual:** Focus budgets with scope-based allocation. Forecast status (ON TRACK / AT RISK / OVER BUDGET).

9. **Commitment Analysis:** Coverage rate, savings calculation, cross-provider comparison, unused/underutilized commitment detection, eligible uncovered spend.

10. **FOCUS Conformance:** Data quality scoring, null/invalid/relationship violation detection, schema coverage reporting.

11. **GCP Inventory Sync:** Compute instances, GKE clusters, BigQuery datasets, Cloud Storage buckets with usage/cost estimates.

12. **Azure Inventory Sync:** VM inventory counts.

13. **Workers/Schedulers:** Cost intelligence (24h), GCP sync (60s poll + per-connection intervals), Google Workspace sync (24h), Databricks sync (24h), currency rate refresh (24h), HITL expiry (60s), notification fanout/delivery (5s).

**🟡 Partially Implemented:**

1. **Rightsizing:** AWS data flows in via dashboard snapshot but is displayed only as "Rightsizing opportunity for {resource}" with estimated savings — no local computation, no actual rightsizing action, no verification.

2. **Commitment Recommendations:** One-size-fits-all "Purchase Savings Plan" and "Purchase Reserved Instances" titles. No per-service or per-family recommendations. No term or payment option analysis.

3. **Savings Plan Simulator:** Hardcoded discount rates (20%/24%/28%) with no provider API integration. Basic formula: `monthly_commitment × discount_rate × (coverage/100)`.

4. **Waste Recommendations:** Only from AWS dashboard waste scan. Hardcoded default savings. No GCP/Azure waste scanning (only stopped instance counts).

5. **Confidence/Risk/Effort:** Static values per recommendation type (confidence: "medium"/"low", risk: "low"/"medium", effort: "low"/"medium"). No data-driven assessment.

---

## 2. Existing Optimisation Engine — Deep Analysis

### 2.1 `services/optimization/recommendation_engine.py` (634 lines)

**Purpose:** Generate AutoStopping, schedule, RI/SP recommendations; submit via HITL.

**Algorithm:**
1. Load AWS dashboard snapshot from DB (`load_aws_dashboard_from_db`)
2. Extract waste scan findings (up to 40), filter to idle categories
3. Extract rightsizing recommendations (up to 20) for schedule suggestions
4. Extract commitment coverage / CE purchase recommendations for SP + RI
5. Deduplicate by title + status
6. Persist to `optimization_recommendations`

**Key observations:**
- **AWS-only:** Entirely dependent on AWS Cost Explorer dashboard snapshot
- **No local computation:** Passes through AWS recommendation data verbatim
- **No rightsizing action:** Schedule recommendations are created from rightsizing data but don't actually recommend instance family migration
- **No sizing logic:** No CPU/memory utilization analysis, no instance type mapping
- **Hardcoded savings:** Default savings per category are arbitrary estimates
- **Non-prod detection:** Tag-based heuristic (`dev`, `staging`, `test`, `qa` keywords) — no actual tag key checking

### 2.2 `services/action_center_service.py` (1621 lines)

**Purpose:** Assemble multi-source Action Center feed with caching, pagination, state management.

**Sources assembled:**
1. Pending HITL approval actions (`_pending_approval_items`)
2. AWS dashboard insights, anomalies, waste, rightsizing, permission errors (`build_aws_action_items`)
3. Cost Intelligence anomalies (`build_cost_intelligence_action_items`)
4. GCP/Azure inventory waste (`build_gcp_azure_inventory_action_items`)

**Key observations:**
- Cache-first assembly with stale cache fallback
- Per-source error isolation (nested savepoints prevent one source from breaking others)
- No shadow-mode / dry-run evaluation
- Impact tracking compares baseline vs current — no automated re-evaluation

### 2.3 `services/cost_intelligence/` (22 files)

**Capabilities:**
- `forecast_engine.py`: Statistical forecasting with StatsForecast
- `scoring.py`: 5-component anomaly scoring
- `scheduler.py`: Orchestrated detection pipeline
- `suggested_actions.py`: Service-specific playbook suggestions
- `contributors.py`: Contribution analysis
- `conformal.py`: Conformal prediction intervals
- `changepoint.py`: Changepoint detection
- `series_builder.py`: Time series construction
- `backtesting.py`: Model evaluation

**Key observations:**
- **Sophisticated statistical engine** — this is the strongest component
- Forecast-only for cost — no utilization forecasting
- No resource-level forecasting (only cost at service/total scope)
- Suggestion engine is rule-based, not learned

### 2.4 Duplicated / Dead / Placeholder Code

**Dead Code:**
- `services/anomaly_rca_service.py` — referenced but contains only rule-based summaries (no ML)
- `services/anomaly_routing_service.py` — legacy anomaly handling, being replaced by CI system
- `services/cost_intelligence/backfill.py` — standalone script, not integrated into scheduler
- `services/cost_intelligence/legacy_comparison.py` — shadow comparison for migration

**Placeholders:**
- `services/cost_intelligence/suggested_actions.py:7` — "Never auto-execute rightsizing from an anomaly alone" (action never taken)
- `simulate_savings_plan()` — stub with hardcoded discount rates
- `_is_nonprod()` — tag-based heuristic needs improvement
- `_DEFAULT_SAVINGS_BY_CATEGORY` — hardcoded, not data-driven

**Duplication:**
- AWS anomaly handling exists in BOTH `action_center_service.py` (from dashboard snapshot) AND `action_center_cost_intelligence.py` (from persisted CI anomalies)
- `cost_detections.statistical_anomalies` and `spike_alerts` coexist with newer CI anomaly tables
- Budget tracking exists in BOTH `focus_budgets` table AND legacy `/dashboard/budget` page

---

## 3. Data Capability Assessment

### 3.1 Available Datasets — What You Have

| Dataset | Source | Frequency | Completeness | Quality | Optimisation Use |
|---------|--------|-----------|-------------|---------|-----------------|
| **FOCUS cost_charges** | AWS/GCP/Azure billing | Daily | High (line-item) | Medium (tag quality varies) | Cost analysis, allocation, rightsizing input, commitment analysis |
| **Cost observations** | Rollup from cost_charges | Daily | High (aggregated) | High | Forecasting, anomaly detection |
| **Cost anomalies** | Statistical detection | Daily (24h cycle) | High | High | Action Center items |
| **Cost forecasts** | StatsForecast models | Daily (24h cycle) | Medium (30d horizon) | Medium (model-dependent) | Budget alerts, capacity planning |
| **GCP inventory** | GCP API sync | Variable (60s-24h) | Medium | Medium (API availability) | VM rightsizing, GKE optimisation, storage optimisation |
| **Azure inventory** | Azure API sync | Variable | Low (VMs only) | Low | VM rightsizing |
| **AWS dashboard** | Cost Explorer API | Daily | High (account-level) | High | Waste scan, rightsizing, commitments |
| **Commitments** | FOCUS commitment data | Daily | Medium | Medium | RI/SP coverage, utilisation |
| **Budgets** | User-defined | On create | High | High | Budget vs actual, forecast alerts |
| **Virtual tags** | User-defined | On create | High | User-dependent | Allocation, showback, unit economics |
| **Business metrics** | User-defined | On create | High | User-dependent | Unit economics |
| **Cost pricing** | Catalog data | Static | Low | Low | Savings estimates |
| **Tool usage** | Provider API | Variable | Low | Low | Usage-based optimisation |
| **Cost events** | Audit/deploy events | Sparse | Low | Low | Anomaly correlation |

### 3.2 Missing Datasets — Blocking Advanced Optimisation

| Dataset | Why Missing | Impact |
|---------|------------|--------|
| **AWS Resource Inventory** (EC2, RDS, Lambda, EBS) | Not synced locally; only via dashboard snapshot | Cannot compute rightsizing locally; dependent on AWS recommendations |
| **Azure Resource Inventory** (beyond VM count) | Only VM counts synced | Cannot detect unattached disks, idle resources, rightsizing |
| **Kubernetes Metrics** (pod-level CPU/memory) | GKE cluster synced but no pod metrics | No pod rightsizing, namespace optimisation |
| **Utilization Metrics** (CPU, memory, network, disk IOPS) | Not collected from any provider | Cannot verify rightsizing recommendations; confidence is low |
| **Instance Type Catalog** (pricing per region/family) | `cost_pricing` table exists but is sparsely populated | Cannot compute accurate savings for rightsizing |
| **Spot Pricing Data** | Not collected | No spot opportunity analysis |
| **Tag Compliance History** | Not tracked | Cannot measure governance improvement |
| **Resource Lifecycle Events** (create, modify, delete) | Cost events table exists but empty | Cannot correlate cost changes with infrastructure changes |
| **Deployment Tracking** | No integration | Cannot identify temporary/test resources |

---

## 4. Recommendation Engine Design

### 4.1 Architectural Principles

1. **Data-Driven Confidence:** Every recommendation must trace to observed data — no hardcoded defaults
2. **Multi-Provider:** AWS-only is insufficient; must leverage GCP inventory and Azure data
3. **Verifiable Evidence:** Include the actual metric readings that triggered the recommendation
4. **Progressive Disclosure:** Start with low-effort, high-confidence recommendations; layer complexity
5. **Feedback Loop:** User actions (dismiss/approve/resolve) train the recommendation engine

### 4.2 Recommendation Data Model (Extending `optimization_recommendations`)

```python
class OptimisationRecommendation:
    # Existing fields
    id: UUID
    tenant_id: UUID
    rec_type: str  # rightsizing, idle_resource, commitment, storage, etc.
    title: str
    description: str
    provider: str  # aws, gcp, azure, multi
    resource_ids: list[str]
    estimated_monthly_savings: Decimal
    currency: str
    status: str  # open, pending_approval, rejected, approved, executed, dismissed
    payload: dict  # existing

    # NEW fields to add:
    category: str           # compute, storage, database, network, container, commitment
    subcategory: str        # rightsizing, idle, unattached, archive, etc.
    
    # Evidence
    evidence: dict = {
        "metric_readings": [...],       # The actual utilization values
        "observation_period_days": 30,  # How long we observed
        "peak_utilization": 0.12,       # 12% CPU
        "average_utilization": 0.08,    # 8% CPU
        "current_specs": {...},         # Current instance details
        "recommended_specs": {...},     # Recommended instance details
        "confidence_factors": [...],    # Why we're confident
    }
    
    # Impact Assessment
    performance_impact: str  # none, minimal, moderate, significant
    business_risk: str       # low, medium, high, critical
    implementation_effort: str  # low, medium, high
    reversibility: str       # fully_reversible, partially_reversible, irreversible
    
    # Validation
    required_validation: list[str]  # Steps user should take before acting
    assumptions: list[str]          # What we assumed in our calculation
    
    # Savings breakdown
    estimated_monthly_savings: Decimal
    estimated_annual_savings: Decimal
    savings_breakdown: dict = {
        "compute_savings": ...,
        "storage_savings": ...,
        "network_savings": ...,
    }
    
    # Timestamps
    first_detected_at: datetime
    last_observed_at: datetime
    
    # ML feedback
    user_feedback_label: str | None  # true_positive, false_positive, planned
    user_feedback_at: datetime | None
```

### 4.3 Recommendation Generation Pipeline

```
┌─────────────────────────────────────────────────────────┐
│                   SCHEDULER (24h cycle)                  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              SCOPE DETECTION                      │   │
│  │  For each active CloudConnection per provider:   │   │
│  │   • Load all known resources from inventory       │   │
│  │   • Load last 30-90 days of cost data             │   │
│  │   • Load utilization data (if available)          │   │
│  └──────────────┬───────────────────────────────────┘   │
│                 ▼                                       │
│  ┌──────────────────────────────────────────────────┐   │
│  │           EVIDENCE COLLECTION                     │   │
│  │  For each recommendation type:                   │   │
│  │   • Compute utilization statistics               │   │
│  │   • Calculate savings projections                │   │
│  │   • Assess confidence                            │   │
│  │   • Identify risks                               │   │
│  └──────────────┬───────────────────────────────────┘   │
│                 ▼                                       │
│  ┌──────────────────────────────────────────────────┐   │
│  │           SCORING & PRIORITISATION                │   │
│  │   • Confidence score (0-100%)                     │   │
│  │   • Expected impact ($)                           │   │
│  │   • Implementation effort                         │   │
│  │   • Business risk                                  │   │
│  └──────────────┬───────────────────────────────────┘   │
│                 ▼                                       │
│  ┌──────────────────────────────────────────────────┐   │
│  │           DEDUPLICATION & MERGE                   │   │
│  │   • Compare with existing open recommendations    │   │
│  │   • Update or create based on new evidence        │   │
│  │   • Close outdated recommendations                │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## 5. Every Optimisation Opportunity Supported by Current Platform

### 5.1 COMPUTE — Rightsizing

**Current State:** 🟡 Partial — AWS rightsizing data flows in from Cost Explorer but is passed through untransformed. No local computation.

**What Can Be Built Now:**

#### 5.1.1 GCP Compute Engine Rightsizing

**Required Data:** `GCPComputeInstance` (vcpu_count, memory_gb, status, machine_type, zone)

**Current Gap:** `GCPComputeInstance` has machine type, vCPU, memory, status — but NO utilization data. The GKE sync fetches monitoring metrics (`cpu/request_utilization`, `memory/request_utilization`), but GCE instances do not have utilization metrics collected.

**Recommendation Structure:**

> **Title:** Downsize GCE instance `instance-name` from `n1-standard-4` to `n1-standard-2`
> 
> **Evidence:** This GCE instance has been running for 30+ days with `status=running`. Current specs: `n1-standard-4` (4 vCPU, 15 GB memory). Based on machine type comparison and workload profile (inferred from machine family `n1`), the instance is likely over-provisioned for its workload.
> 
> **Current Cost:** ~$120/month (computed from machine type pricing)
> **Target Cost:** ~$60/month (n1-standard-2)
> **Estimated Savings:** ~$60/month (50%)
> **Confidence:** 40% — Cannot verify without utilization metrics
> 
> **Why Confidence is Low:** Platform has no CPU/memory monitoring data for this instance. Recommendation is based only on machine type profiling.
> 
> **Required Validation:** Install Cloud Monitoring agent or verify utilization via `gcloud recommender` before downsizing.
> 
> **Implementation Effort:** Low (one API call to change machine type)
> **Reversibility:** Fully reversible (stop instance, change type, start)
> **Business Risk:** Medium — insufficient capacity could impact performance

**Backend Work Required:**
1. Collect GCE instance utilization metrics via Cloud Monitoring API (use GKE pattern from `workers/gcp/gke_sync.py`)
2. Build instance type pricing catalog for `cost_pricing` table
3. Create `RightsizingScorer` service — compares current vs recommended specs
4. Extend `recommendation_engine.py` to process GCP inventory

**Confidence Model:**
- With utilization data: 80-95% confidence
- Without utilization data: 30-50% confidence (type-based only)

#### 5.1.2 AWS EC2 Rightsizing (Independent Verification)

**Current State:** AWS Cost Explorer rightsizing recommendations are passed through with AWS's confidence. No independent verification.

**What to Add:**
- Cross-validate AWS rightsizing recommendations against locally computed utilization patterns from CloudWatch (if available)
- If no CloudWatch data, show AWS confidence but flag "not independently verified"
- Add EC2 instance type pricing to `cost_pricing` for independent savings calculation

#### 5.1.3 Azure VM Rightsizing

**Current State:** Only VM count synced. No machine types, no utilization.

**What to Add:**
- Extend Azure inventory sync to collect VM sizes, status, and (optionally) Azure Monitor metrics
- Apply same rightsizing framework as GCP

### 5.2 COMPUTE — Idle & Underutilised Instances

**Current State:** 🟡 Partial — AWS waste scan identifies idle EC2 via dashboard. GCP/Azure only count stopped instances.

**What Can Be Built Now:**

#### 5.2.1 Idle GCE Instance Detection

**Current Data:** `GCPComputeInstance.status` (RUNNING, STOPPED, TERMINATED) + cost data from `GCPCostDaily`

**What's Missing:** CPU utilization to determine "idle but running"

**Without Utilization (Confidence 30-40%):**
- Identify instances running for >14 days with no associated cost in `GCPCostDaily` (potential free-tier or orphan)
- Flag instances in `RUNNING` status that were created >90 days ago without connectivity changes

**With Utilization (Confidence 85-95%):**
- CPU < 5% for 14+ days: Idle candidate
- CPU < 1% for 7+ days: Strong idle candidate

#### 5.2.2 Idle Azure VM Detection

**Current Data:** VM count, running, stopped counts

**Without Utilization:** Compare `stopped_virtual_machines` count across sync runs — persistent stopped VMs with attached disks are "zombie cost" candidates.

#### 5.2.3 Idle AWS EC2 Detection (Independent)

**Current State:** Relies entirely on AWS Cost Explorer "idle EC2" waste category.

**What to Add:** Independent check using CloudWatch metrics (if available) or by analyzing Cost Explorer daily cost patterns for individual EC2 resources.

### 5.3 COMPUTE — Spot/Preemptible Opportunities

**Current State:** 🔴 Missing

**What's Needed:**
- Collect spot pricing data from AWS (`ec2 describe-spot-price-history`)
- Collect preemptible VM pricing from GCP
- Analyze workload suitability:
  - Stateless workloads
  - Fault-tolerant architectures
  - Batch processing
  - CI/CD runners
- Compare on-demand vs spot/preemptible pricing

**Can Be Built?** ⚪ Partial — requires `cost_pricing` population + workload profiling

### 5.4 COMPUTE — GPU Optimisation

**Current State:** 🔴 Missing

**Required Data:**
- GPU instance inventory (type, count, utilization)
- GPU pricing data
- Workload type inference

**Can Be Built?** 🔴 Not without GPU utilization data — speculative recommendations would be low confidence.

### 5.5 COMPUTE — ARM/Graviton Migration

**Current State:** 🔴 Missing

**Required Data:**
- Instance type mapping (x86 → ARM equivalents)
- Software compatibility assessment
- Performance benchmarks

**Can Be Built?** ⚪ Possible but needs instance type catalog + would require user validation for compatibility.

### 5.6 STORAGE — Unattached Volumes

**Current State:** 🟡 Partial — AWS unattached EBS volumes detected via dashboard snapshot. GCP/Azure not detected.

**What Can Be Built Now:**

#### 5.6.1 AWS EBS (Already Detected)

Currently shown as "Unattached EBS volumes need review" with count, GiB, and cost estimate ($0.08/GiB-month).

**Improvements Needed:**
- Link to specific volume IDs and snapshots
- Include last-attached timestamp (from EBS API)
- Calculate actual cost from billing data vs estimate
- Add bulk delete/snapshot workflow

**Confidence:** 95% — EBS API definitively reports attachment state

#### 5.6.2 GCP Persistent Disk Optimisation

**Current Data:** GCS buckets synced but NOT persistent disks.

**Required:** Extend GCP inventory sync to include persistent disks (name, size, type, zone, status, attached_to, last_attach_timestamp, creation_timestamp, snapshot_schedule)

**What to Build:**
- Unattached PD detection: `status=READY` and `attached_to is null`
- Oversized PD detection: Compare allocated size vs used bytes (requires disk usage metrics)
- Snapshot lifecycle: Detect orphaned snapshots (disk deleted but snapshots remain)

#### 5.6.3 Azure Managed Disk Optimisation

**Current State:** Not collected at all.

**Required:** Sync Azure disks (type, size, SKU, attached VM, location, tags)

### 5.7 STORAGE — Object Storage Lifecycle

**Current State:** 🟡 Partial — GCP storage buckets synced with total_bytes, object_count, storage_class. No lifecycle analysis.

**What Can Be Built Now:**

#### 5.7.1 GCS Cold Storage Opportunities

**Current Data:** `GCPStorageBucket` (storage_class, total_bytes, last_synced_at)

**What to Add:**
- Access pattern analysis: Compare object count change across sync runs to estimate growth
- Storage class optimisation: Flag `STANDARD` buckets with low modification rate for `NEARLINE`/`COLDLINE`/`ARCHIVE`
- Lifecycle policy recommendations: "Bucket `logs-archive` has 500GB of STANDARD data with no changes in 90 days. Moving to COLDLINE saves ~$X/month."

**Confidence Without Access Logs:** 40-50% — can only infer access patterns from data growth

**With Access Logs:** 80-90%

#### 5.7.2 S3 Storage Optimisation

**Required:** Sync S3 bucket inventory (size, object count, storage class, last_modified dates, lifecycle policies)

**Currently Not Collected:** Would need new AWS sync worker similar to GCP storage sync.

### 5.8 STORAGE — Snapshot Lifecycle

**Current State:** "orphaned_snapshot" is a waste category with hardcoded $10/month savings default.

**Current Data:** Only the category flag. No snapshot inventory.

**What Can Be Built Now:**

**Required Data:**
- Snapshot inventory (age, size, associated volume/instance, region)
- Snapshot pricing per GB-month

**Implementation:**
- Add snapshot discovery to GCP sync (disks → snapshots via `compute.snapshots.list`)
- Add EBS snapshot discovery to AWS dashboard or new sync worker
- Identify: snapshots older than X days with no active source volume
- Calculate savings: `snapshot_size_GB × storage_price_per_GB × months_retained`

### 5.9 NETWORKING — Idle Load Balancers

**Current State:** 🔴 Missing

**Required Data:**
- Load balancer inventory (type, region, forwarding rules, targets)
- Traffic metrics (bytes processed, request count)
- Pricing per LB type

**Can Be Built?** ⚪ Possible for GCP (forwarding rules exist in GCP inventory) but would need new sync for AWS/ALB/NLB.

### 5.10 NETWORKING — NAT Gateway Optimisation

**Current State:** 🔴 Missing — NAT gateway suggestion exists in `suggested_actions.py` playbook but no detection.

**Required Data:**
- NAT gateway inventory and data processing volume
- Alternative routing analysis (VPC endpoints, egress optimization)

### 5.11 NETWORKING — Public IP Waste

**Current State:** 🟡 Partial — Unassociated EIP is in waste category with hardcoded $3.65/month savings.

**Improvement:**
- Query actual EIP count and associated cost from billing data
- Extend to GCP (external IP addresses) and Azure (public IPs)

### 5.12 CONTAINERS & KUBERNETES

**Current State:** 🟡 Partial — GKE clusters synced with node count, vCPU, memory, CPU/memory request utilization.

**What Can Be Built Now:**

#### 5.12.1 GKE Node Pool Optimisation

**Current Data:** `GCPGKECluster.node_pools` (machine_type, initial_node_count, autoscaling_enabled, min/max_node_count), CPU/memory request utilization

**Recommendations:**

> **Title:** Downsize GKE node pool `default-pool` in cluster `production-cluster`
>
> **Evidence:** Node pool `default-pool` (e2-standard-4, 3 nodes, autoscaling) has average CPU request utilization of 28% and memory request utilization of 35% over 30 days.
>
> **Current Cost:** ~$900/month (3 × e2-standard-4)
> **Target: e2-standard-2** (reduce node size)
> **Cost After:** ~$450/month
> **Savings:** ~$450/month (50%)
> **Confidence:** 85%

**Additional Recommendations:**
- **Cluster Autoscaler Tuning:** If autoscaling not enabled, recommend enabling
- **Node Pool Consolidation:** Recommend merging underutilized node pools
- **Spot Node Pools:** Recommend adding preemptible node pool for batch workloads
- **Reserved Capacity:** Recommend Committed Use Discounts for stable workloads

#### 5.12.2 Pod-Level Rightsizing (Requires New Data)

**Missing Data:** Individual pod resource requests/limits, actual pod utilization, namespace-level cost breakdown

**What's Needed:**
- Deploy kube-state-metrics or similar to collect pod specs
- Parse GKE usage metering export in BigQuery for namespace-level cost
- Or integrate with existing Kubernetes monitoring tools

**Why This Matters:** Cost allocation without pod-level data means you can only optimise at the cluster/node level, missing 50%+ of potential Kubernetes savings.

### 5.13 DATABASES

**Current State:** 🔴 Missing — No database-specific optimisation.

**What Can Be Built (GCP Only):**

#### 5.13.1 Cloud SQL Optimisation

**Required:** Sync Cloud SQL instances (tier, storage, backup config, status)

**Recommendations:**
- Downsizing underutilized instances
- Storage autogrowth limit recommendations
- Backup retention optimisation
- Idle instance detection

#### 5.13.2 BigQuery Optimisation

**Current Data:** `GCPBigQueryDataset` (table_count, total_logical_bytes, total_physical_bytes, monthly_cost_estimate)

**What Can Be Built:**
- **Flat-rate vs On-demand Analysis:** Compare slot commitment cost vs on-demand pricing
- **Partitioning/Clustering Recommendations:** Flag unpartitioned tables over 10GB
- **Long-running Query Detection:** Analyze INFORMATION_SCHEMA for expensive queries
- **Storage Cost Optimisation:** Compare logical vs physical bytes — recommend physical billing if physical/logical < 0.5
- **Table Clustering Recommendations:** Flag tables scanned frequently without clustering

#### 5.13.3 AWS RDS Optimisation

**Required:** RDS instance inventory (class, storage, multi-AZ, backup retention, status)

**Not Collected:** Would need new AWS sync worker.

### 5.14 COMMITMENTS

**Current State:** 🟡 Partial — One-size-fits-all SP/RI recommendations. Basic coverage analysis.

**What Can Be Built Now:**

#### 5.14.1 Granular Commitment Recommendations

**Current Data:** `focus_commitments`, `cost_charges` (commitment_discount_id, commitment_discount_type, commitment_discount_status)

**What to Build:**

1. **Per-Service Commitment Analysis:**
   - "Your EC2 spend is $5,000/month with 0% coverage. A 1-year EC2 Standard RI covering 70% of baseline spend saves ~$1,050/month."
   - "Your RDS spend is $2,000/month with 60% coverage. Increasing to 80% saves ~$120/month."

2. **Term and Payment Analysis:**
   - "3-year All Upfront Savings Plan saves 20% more than 1-year No Upfront for your steady-state spend."

3. **Expiration Alerts:**
   - "Your Compute Savings Plan (ID: xxx) expires in 45 days. Renew now to maintain 18% discount rate."

4. **Utilisation Analysis Per Commitment:**
   - "Compute Savings Plan has 72% utilisation — $280/month is going to waste. Right-size coverage to match actual usage."

**Implementation Plan:**
1. Analyze `cost_charges` for commitment_discount_id usage
2. Compare commitment coverage to actual line-item commitment discounts applied
3. Compute effective savings rate per commitment type
4. Use AWS Savings Plans pricing API for accurate discount rates (instead of hardcoded 20-28%)

**Confidence:** 75-90% — Based on observed billing data, not estimates

### 5.15 GOVERNANCE

**Current State:** 🟡 Partial — Tag suggestions exist for virtual tags. No compliance monitoring.

**What Can Be Built Now:**

#### 5.15.1 Missing Tag Detection

**Current Data:** `cost_charges.tags` (JSONB), virtual tag rules

**Recommendations:**

> **Title:** 40% of resources missing "cost-center" tag
>
> **Evidence:** Of 1,200 resources with charges in the last 30 days, 480 (40%) have no `cost-center` tag. The top 3 untagged services are EC2, S3, and RDS.
>
> **Impact:** $12,000/month in unallocated spend. Adding the `cost-center` tag would enable showback to owning teams.
>
> **Recommended Action:** Define a governance policy requiring `cost-center` tag on all resources. Use AWS Config / GCP Resource Manager to enforce.

**Implementation:**
- Query `cost_charges` tags JSONB across all services/regions
- Compute tag coverage percentage per tag key
- Identify top untagged spend by service
- Flag resources with critical missing tags (env, cost-center, owner, team)

**Confidence:** 95% — Based on actual tag data in billing records

#### 5.15.2 Budget Overrun Prediction

**Current Data:** `focus_budgets`, `cost_observations`, `cost_forecast_points`

**Recommendations:**

> **Title:** EC2 budget at risk of 15% overrun this month
>
> **Evidence:** Budget for EC2 is set at $10,000/month. Current spend is $8,500 (85% of budget) with 10 days remaining. Forecast projects $11,500 at month-end.
>
> **Recommended Action:** Review recent EC2 changes. Consider applying budget alerts at 80% threshold.

**Implementation:**
- Compare `cost_forecast_points.expected_cost` to `focus_budgets.amount`
- Flag when forecast exceeds budget by >5%
- Use `focus_budgets.alert_threshold` for sensitivity

#### 5.15.3 Policy Violation Detection

**Current State:** Only FOCUS conformance (data quality) exists.

**What to Add:**
- **Region Restriction Violations:** Resources in non-approved regions
- **Instance Type Restrictions:** Resources using prohibited instance types
- **Encryption Compliance:** Resources without encryption (where cost_charges includes this info)

### 5.16 FORECASTING — Predictive Insights

**Current State:** ✅ Implemented — cost forecasting exists via StatsForecast. 

**Gap:** No budget-aware forecasting, no scenario modeling, no seasonal adjustment communication.

**What Can Be Built Now:**

#### 5.16.1 Budget Exhaustion Prediction

Using existing `cost_forecast_points` + `focus_budgets`:

- "At the current run rate, your monthly budget of $50,000 will be exhausted by the 25th. Expected final spend: $58,000."
- "Based on historical patterns, Q4 spend is typically 30% higher than Q3. Plan accordingly."

#### 5.16.2 Anomalous Growth Detection

Using existing `cost_observations` day-over-day and week-over-week:

- "Service XYZ has grown 40% month-over-month for 3 consecutive months. At this rate, annual cost will be $240,000 vs current $100,000."
- "Storage costs are growing at 15% per month. Review lifecycle policies to control growth."

#### 5.16.3 "What If" Scenario Simulation

Extend existing `simulate_savings_plan()` to cover:

- "What if we migrate 50% of EC2 to Graviton?" — Use instance type pricing + current spend
- "What if we move 6-month-old S3 data to Glacier?" — Use storage pricing + current data size
- "What if we downsize all production instances?" — Use instance type pricing

### 5.17 ANOMALY DETECTION — Enhancement

**Current State:** ✅ Implemented — sophisticated 5-component scoring.

**Gaps:**
1. No deployment/change correlation (occurs only in `suggested_actions.py` playbooks)
2. No automated root cause analysis (RCA is rule-based)
3. No anomaly "clustering" — 5 similar anomalies in same service = pattern, not 5 separate issues

**What Can Be Built Now:**

#### 5.17.1 Anomaly Pattern Recognition

- Group anomalies by service/account/region
- Detect recurring patterns: "Every Monday, service XYZ spikes 20% — this is likely a batch job"
- Suppress known patterns: Automatically label as `planned_activity` when pattern is recognized

#### 5.17.2 Multi-Service Anomaly Correlation

- Correlate anomalies across services: "EC2 cost anomaly coincided with a NAT Gateway data transfer anomaly — likely related to a deployment"

### 5.18 BUSINESS OPTIMISATION

**Current State:** 🟡 Partial — Unit economics exists but is limited to user-defined metrics. 

**What Can Be Built Now:**

#### 5.18.1 Customer/Product Profitability

Using existing `virtual_tags` + `business_metrics` + `cost_charges`:

- Per-customer cost allocation via virtual tags
- Cost-per-customer trends
- Gross margin per customer/product
- "Customer X's infrastructure cost grew 50% but revenue only grew 20%. Margin decreased from 60% to 45%."

#### 5.18.2 Environment Cost Analysis

Using existing tag structure + `cost_charges`:

- Per-environment cost breakdown (dev/staging/prod via tag.env)
- "Dev environment costs grew to 25% of total spend. Consider right-sizing dev instances or implementing auto-stop during non-business hours."
- "Staging environment has 30% lower utilization than production but same instance sizes."

#### 5.18.3 Feature/Team Cost Efficiency

If teams tag their resources:

- Cost per team per period
- Cost per feature area
- Team efficiency trending

---

## 6. Feature Gap Analysis

### 6.1 Comprehensive Gap Matrix

| Capability | Status | Required Data | Backend Work | UI Work | Complexity | Business Value |
|-----------|--------|--------------|-------------|--------|-----------|---------------|
| **GCE Rightsizing** | 🟡 Partial | Utilization metrics | Utilization collector + pricing catalog | Extend optimisation page | High | Very High |
| **GCE Idle Detection** | 🟡 Partial | Utilization metrics | Same as above | Reuse existing cards | High | Very High |
| **GKE Node Rightsizing** | 🟡 Partial | Already collected | Recommendation engine | Existing UI | Low | High |
| **GKE Pod Rightsizing** | 🔴 Missing | Pod metrics | New data source | New component | Very High | Very High |
| **Azure Rightsizing** | 🔴 Missing | VM inventory + utilization | New sync worker | Extend existing | High | High |
| **Unattached PD (GCP)** | 🔴 Missing | PD inventory | Extend GCP sync | Action Center already supports | Medium | Medium |
| **Snapshot Lifecycle** | 🔴 Missing | Snapshot inventory | New sync components | Action Center already supports | Medium | Medium |
| **S3 Storage Optimisation** | 🔴 Missing | S3 inventory | New sync worker | New component | High | High |
| **GCS Cold Storage** | 🟡 Partial | Bucket inventory | Access pattern analysis | Existing cost pages | Low | Medium |
| **BigQuery Optimisation** | 🟡 Partial | Dataset inventory | Analysis logic | New component | Medium | High |
| **SQL Optimisation** | 🔴 Missing | Instance inventory | New sync | New component | Medium | High |
| **Granular SP/RI Recs** | 🟡 Partial | Already collected | Recommendation engine | Existing commitment UI | Low | Very High |
| **Commitment Expiry Alerts** | 🔴 Missing | Already collected | Scheduler + notification | Existing notification UI | Low | High |
| **Commitment Waste Detection** | 🟡 Partial | Already collected | Analysis logic | Existing UI | Low | High |
| **Missing Tag Detection** | 🟡 Partial | Already collected | Analysis logic | Governance component | Low | Medium |
| **Budget Overrun Alerts** | 🟡 Partial | Already collected | Comparison logic | Existing budget UI | Low | Very High |
| **Region Restriction Policy** | 🔴 Missing | Cost charges region | Analysis logic | Governance component | Medium | Medium |
| **Deployment Correlation** | 🟡 Partial | Cost events table | Event ingester | Existing anomaly UI | Medium | High |
| **Pattern Recognition** | 🟡 Partial | Anomaly history | Clustering logic | Existing anomaly UI | High | High |
| **Scenario Simulation** | 🟡 Partial | Pricing data | Extend simulator | New component | Medium | High |
| **Customer Profitability** | 🟡 Partial | Virtual tags + metrics | Analysis logic | Extend unit economics | Medium | Very High |
| **Environment Analysis** | 🟡 Partial | Tag data | Analysis logic | New component | Medium | High |
| **Team Efficiency** | 🔴 Missing | Team tags | Analysis logic | New component | Medium | High |
| **Spot/Preemptible Analysis** | 🔴 Missing | Spot pricing | Pricing data collector | Recommendation cards | High | High |
| **ARM Migration** | 🔴 Missing | Instance catalog | Pricing data + mapping | Recommendation cards | Medium | Medium |
| **GPU Optimisation** | 🔴 Missing | GPU utilization | Utilization collector | Recommendation cards | Very High | High |

### 6.2 Legend

- ✅ **Implemented:** Fully working in production
- 🟡 **Partial:** Some capability exists but incomplete
- 🔴 **Missing:** Not implemented, feasible with current data
- ⚪ **Not Feasible:** Required data does not exist

---

## 7. Confidence Model for Recommendations

### 7.1 Confidence Formula

```
Confidence = w1 × DataQuality + w2 × HistoricalAccuracy + w3 × EvidenceStrength + w4 × ProviderConfidence

Where:
- DataQuality: How complete/reliable is the source data (0-100%)
- HistoricalAccuracy: How accurate were past similar recommendations (0-100%)
- EvidenceStrength: How strong is the supporting evidence (0-100%)
- ProviderConfidence: Native provider recommendation confidence (0-100%)
```

### 7.2 Confidence Bands

| Band | Range | Meaning | Action Required |
|------|-------|---------|-----------------|
| **Very High** | 90-100% | Definitively measurable (e.g., unattached volume) | Can auto-apply with approval |
| **High** | 75-89% | Strong evidence, low assumptions | Recommend with validation steps |
| **Medium** | 50-74% | Moderate evidence, some assumptions | Require manual verification |
| **Low** | 25-49% | Weak evidence, significant assumptions | Informational only |
| **Very Low** | 0-24% | Speculative | Do not surface as recommendation |

### 7.3 Confidence by Recommendation Type

| Recommendation Type | Base Confidence | Factors | Why |
|-------------------|----------------|---------|-----|
| Unattached EBS | 95% | EBS API is definitive | Volume is unattached or attached |
| Stopped EC2 with attached EBS | 90% | EC2 state is definitive | Instance is stopped, billing continues for EBS |
| Unattached GCP PD | 90% | API is definitive | Disk status + attachment is reliable |
| Orphaned Snapshot | 85% | Source volume check | Can definitively check if source exists |
| GKE Node Rightsizing (with metrics) | 85% | Utilization + pricing | 30+ days of metric data |
| Missing Tag Detection | 95% | Billing records definitive | Tag presence is binary |
| Budget Overrun Forecast | 80% | Forecast + budget | Depends on forecast accuracy |
| SP/RI Coverage Gap | 85% | Billing data definitive | Commitment discount applied = covered |
| GCE Rightsizing (with metrics) | 80% | Utilization + instance types | Depends on workload pattern |
| Commitment Waste (underutilized) | 75% | Usage vs purchased | May have seasonal variation |
| GCE Rightsizing (no metrics) | 40% | Type-based estimate | No actual utilization data |
| GCS Cold Storage (no access logs) | 50% | Age-based inference | Cannot verify access patterns |
| AWS Rightsizing (from CE) | 70% | AWS native confidence | Cross-validation not possible |
| BigQuery Slot Recommendation | 60% | Usage patterns | Depends on workload consistency |

---

## 8. Recommendation Prioritisation Framework

### 8.1 Priority Score Formula

```
PriorityScore = (Savings × 0.4) + (Confidence × 0.3) + (EffortInverse × 0.2) + (RiskInverse × 0.1)

Where:
- Savings: Normalized to 0-100 scale (relative to tenant total spend)
- Confidence: Recommendation confidence score (0-100%)
- EffortInverse: 100 = low effort, 0 = high effort
- RiskInverse: 100 = low risk, 0 = high risk
```

### 8.2 Priority Classification

| Score | Priority | Action |
|-------|----------|--------|
| 80-100 | **Critical** | Immediate action, surface prominently |
| 60-79 | **High** | Act within current sprint |
| 40-59 | **Medium** | Schedule for next sprint |
| 0-39 | **Low** | Monitor, surface only in detailed view |

### 8.3 Example Priority Computation

**Recommendation: "Unattached EBS volume (100 GiB)"**
- Savings: 100 GiB × $0.08 = $8/month → Normalized: 15 (low savings)
- Confidence: 95%
- Effort: Low (one click) → EffortInverse: 100
- Risk: Low (data already exists elsewhere) → RiskInverse: 100

**Priority Score = (15 × 0.4) + (95 × 0.3) + (100 × 0.2) + (100 × 0.1) = 6 + 28.5 + 20 + 10 = 64.5 → HIGH**

**Recommendation: "Downsize GKE node pool (projected $450/month savings)"**
- Savings: $450 → Normalized: 70
- Confidence: 85%
- Effort: Medium → EffortInverse: 60
- Risk: Medium → RiskInverse: 60

**Priority Score = (70 × 0.4) + (85 × 0.3) + (60 × 0.2) + (60 × 0.1) = 28 + 25.5 + 12 + 6 = 71.5 → HIGH**

---

## 9. Implementation Roadmap

### 9.1 IMMEDIATE — Can Build Now (No New Data Required)

These recommendations require only SQL queries against existing data:

| # | Feature | Estimate | Confidence | Depends On |
|---|---------|----------|-----------|------------|
| 1 | **Granular Commitment Recommendations** | 2-3 days | 85% | `cost_charges` + `focus_commitments` |
| 2 | **Commitment Expiry Alerts** | 1-2 days | 90% | `focus_commitments.end_date` |
| 3 | **Missing Tag Detection & Governance** | 2-3 days | 95% | `cost_charges.tags` |
| 4 | **Budget Overrun Prediction** | 1-2 days | 80% | `cost_forecast_points` + `focus_budgets` |
| 5 | **GKE Node Pool Rightsizing** | 2-3 days | 85% | `GCPGKECluster` (already synced) |
| 6 | **Environment Cost Analysis** | 2-3 days | 80% | `cost_charges` + tag analysis |
| 7 | **Top Spender by Service (Actionable)** | 1 day | 90% | `cost_charges` |
| 8 | **Commitment Waste Detection** | 2-3 days | 75% | `cost_charges` commitment fields |
| 9 | **Service-Level Cost Growth Alerts** | 2-3 days | 70% | `cost_observations` trends |
| 10 | **Concurrent Anomaly Correlation** | 3-5 days | 65% | `cost_anomalies` clustering |

**Total: ~18-27 engineering days**

### 9.2 SHORT-TERM (Add Collector — Low Effort)

These require extending existing sync workers:

| # | Feature | Estimate | Confidence | New Data Needed |
|---|---------|----------|-----------|-----------------|
| 11 | **GCP PD Inventory Sync** | 2-3 days | 90% | Extend GCP sync |
| 12 | **GCP Snapshot Inventory Sync** | 2-3 days | 85% | New GCP sync phase |
| 13 | **Unattached PD + Orphaned Snapshot Recs** | 2-3 days | 90% | Uses new inventory |
| 14 | **GCS Cold Storage Recommendations** | 2-3 days | 50% | Uses existing bucket data |
| 15 | **Azure VM Size Sync** | 2-3 days | 80% | Extend Azure sync |
| 16 | **BigQuery Optimisation Insights** | 3-5 days | 70% | Uses existing dataset data |
| 17 | **Customer/Product Profitability** | 3-5 days | 75% | Uses existing virtual tags + metrics |
| 18 | **Scenario "What If" Simulator** | 3-5 days | 60% | Uses existing cost_pricing + instance data |

**Total: ~18-29 engineering days**

### 9.3 MEDIUM-TERM (New Workers / Significant Logic)

These require new sync workers or substantial business logic:

| # | Feature | Estimate | Confidence | New Infrastructure |
|---|---------|----------|-----------|-------------------|
| 19 | **GCE Utilization Collector** | 5-10 days | 80% | Monitoring API queries in GCP sync |
| 20 | **GCE Rightsizing Engine** | 5-8 days | 80% | Instance type pricing + utilization logic |
| 21 | **Azure Rightsizing** | 5-8 days | 70% | Azure Monitor metrics + VM size pricing |
| 22 | **S3 Bucket Inventory Sync** | 5-8 days | 80% | New AWS sync worker |
| 23 | **S3 Storage Optimisation** | 3-5 days | 70% | S3 lifecycle analysis |
| 24 | **RDS Instance Sync + Optimisation** | 5-8 days | 75% | New AWS RDS sync |
| 25 | **Spot/Preemptible Opportunity Analysis** | 5-8 days | 60% | Spot pricing data collection |
| 26 | **Anomaly Pattern Recognition** | 8-10 days | 65% | Anomaly clustering logic |
| 27 | **Deployment Correlation** | 5-8 days | 70% | Cost events ingestion |
| 28 | **Team Efficiency Dashboard** | 5-8 days | 70% | Uses existing data |

**Total: ~46-73 engineering days**

### 9.4 LONG-TERM (Significant New Infrastructure)

These require architectural changes or new integrations:

| # | Feature | Estimate | Confidence | New Infrastructure |
|---|---------|----------|-----------|-------------------|
| 29 | **GKE Pod-Level Rightsizing** | 15-20 days | 75% | kube-state-metrics or GKE usage metering |
| 30 | **Namespace Cost Allocation** | 10-15 days | 70% | GKE usage metering export |
| 31 | **Kubernetes Cluster Comparison** | 10-15 days | 65% | Multi-cluster metrics aggregation |
| 32 | **ARM/Graviton Migration Analysis** | 10-15 days | 50% | Instance type mapping + compatibility DB |
| 33 | **GPU Optimisation** | 10-15 days | 60% | GPU utilization + pricing data |
| 34 | **ML-Powered Recommendation Ranking** | 15-20 days | 65% | User feedback training data |
| 35 | **Automated Rightsizing (with HITL)** | 15-20 days | 70% | HITL execution integration |
| 36 | **Cross-Provider Total Cost Comparison** | 10-15 days | 50% | Consistent pricing normalization |

**Total: ~95-135 engineering days**

---

## 10. Product Differentiation Strategy

### 10.1 How to Not Be a Clone

Most competitors (CloudZero, Vantage, Finout, etc.) differentiate on:
1. Ease of setup (API-based connections)
2. UI/UX polish
3. Specific features (CloudZero's unit economics, Vantage's savings plans)
4. AI-powered chat (all of them now)

**Your differentiators (based on architecture):**

### 10.2 Built-In Differentiators

| Asset | Competitor Equivalent | Your Advantage |
|-------|----------------------|----------------|
| **LangGraph Agent Architecture** | None (competitors use chat UIs over LLM APIs) | Recommendations are explainable, traceable, and actionable through natural language. A FinOps agent can discuss and defend its recommendations. |
| **FOCUS-normalized Cost Data** | All competitors normalize but not all use FOCUS standard | FOCUS is becoming the FinOps standard. Your platform speaks the standard dialect natively. |
| **RAG + Documentation Grounding** | None (competitors don't ground LLM responses in documentation) | Recommendations can cite specific AWS/GCP documentation, FinOps framework guidance, or your own KB articles. |
| **HITL (Human-in-the-Loop) Approval** | Few competitors have this | Recommendations aren't just shown — they can be executed safely through controlled approval workflows. |
| **Virtual Tag Engine (Finout-style)** | Only Finout has this, and it's their core product | Your allocation engine is equally sophisticated and integrated with recommendations. |
| **Open Source + Self-Hostable** | Most competitors are SaaS-only | Enterprise customers who cannot send billing data to third parties can self-host. This is a MAJOR differentiator. |
| **Multi-Source Action Center** | Most competitors show recommendations in isolation | Your unified Action Center shows savings, anomalies, approvals, permissions, and waste in one place. |

### 10.3 What to Lead With

Don't lead with "AI-powered FinOps" — everyone says that.

Lead with:

**"Your FinOps Engineer, in a Box."**

The platform already has:
1. A FinOps agent that understands cost data
2. Automated anomaly detection with root cause analysis
3. A recommendation engine that generates actionable savings
4. An action center to track resolution
5. HITL approval for safe execution
6. Unit economics to tie cost to business value
7. Showback/chargeback to allocate costs

No competitor has ALL of these in a single, self-hostable platform.

### 10.4 Recommendation Engine Differentiators

| Feature | How to Differentiate |
|---------|---------------------|
| **Explainable AI** | Every recommendation includes "why this recommendation was generated" and "why it may not be appropriate" |
| **Confidence Scoring** | Transparent confidence with factors explaining why (unlike competitors' opaque ML scores) |
| **Feedback Loop** | Recommendations improve based on user actions (dismiss, approve, label) |
| **Chat Integration** | Users can ask "why did you recommend downsizing this instance?" and get a natural language explanation |
| **What-If Agent** | "What if we downsize all production EC2 and buy a 3-year Compute Savings Plan?" — the agent runs the scenario |
| **Cross-Provider** | "Your combined AWS + GCP commitment coverage is 45%. Here's a unified strategy." |
| **Business Context** | "Dev environment costs grew 40%. Here are the top 3 changes that drove this increase." |

### 10.5 Avoid These Anti-Patterns

| Anti-Pattern | Why to Avoid | What to Do Instead |
|-------------|-------------|-------------------|
| Hardcoded savings estimates | Lowers trust | Use actual billing data or clearly label as "estimate" |
| One-size-fits-all recommendations | User dismisses everything | Personalize per service, per account, per environment |
| Overwhelming volume | Recommendation fatigue | Prioritize top 10; group related recommendations |
| No-ops ("consider rightsizing") | Useless | Specific: "move from m5.2xlarge to m5.xlarge saves $X" |
| Ignoring context | Wrong recommendations | Consider maintenance windows, team ownership, compliance needs |
| Silent failures | User thinks feature is broken | Surface data quality issues in Action Center |

---

## Appendix A: Key Code Locations for Implementation

### Backend Files to Extend/Modify

```
# Core recommendation engine
backend/services/optimization/recommendation_engine.py
  → Add GCP/Azure support, granular commitment recs, evidence tracking

# Action Center
backend/services/action_center_service.py
  → Add new recommendation types, improve impact tracking

# Cost Intelligence
backend/services/cost_intelligence/scheduler.py
  → Add utilization forecasting, multi-service correlation
  
backend/services/cost_intelligence/suggested_actions.py
  → Expand service playbooks, add business-context suggestions

# GCP Sync (extend)
backend/workers/gcp/sync_orchestrator.py
  → Add persistent disk, snapshot, SQL sync phases
  
backend/workers/gcp/compute_sync.py
  → Add Cloud Monitoring utilization queries

# New Workers
backend/workers/aws_inventory_worker.py  (NEW)
  → Sync EC2, RDS, S3, EBS inventory
  
backend/workers/commitment_alert_worker.py  (NEW)
  → Check commitment expiry + budget thresholds

# DB Models
backend/app/db/models.py
  → Add evidence, confidence, validation fields to OptimizationRecommendation

# API Routes
backend/routes/focus_enterprise_routes.py or backend/routes/optimization_routes.py
  → New endpoints for scenario simulation, governance insights
```

### Frontend Files to Extend/Modify

```
# Optimisation page
frontend/src/app/dashboard/optimization/page.tsx
  → Add new recommendation types, evidence display, confidence badges

# Action Center
frontend/src/components/action-center/action-center-view.tsx
frontend/src/components/action-center/action-item-detail-view.tsx
  → Add evidence section, validation steps, external links

# Focus pages
frontend/src/app/dashboard/focus/commitments/page.tsx
  → Add granular per-service commitment recommendations

# New pages
frontend/src/app/dashboard/governance/  (NEW)
  → Tag compliance, policy violations, budget alerts
  
frontend/src/app/dashboard/business-optimization/  (NEW)
  → Customer profitability, environment analysis, team efficiency

# Services
frontend/src/services/finops-allocation-service.ts
  → New endpoints for governance, scenario simulation

# Types
frontend/src/types/optimization.ts  (UPDATE)
  → Add evidence, confidence, risk, validation types
```

### Data Migrations

```sql
-- Add evidence + confidence fields to optimization_recommendations
ALTER TABLE optimization_recommendations ADD COLUMN IF NOT EXISTS evidence JSONB DEFAULT '{}';
ALTER TABLE optimization_recommendations ADD COLUMN IF NOT EXISTS confidence DECIMAL(5,2);
ALTER TABLE optimization_recommendations ADD COLUMN IF NOT EXISTS risk VARCHAR(16);
ALTER TABLE optimization_recommendations ADD COLUMN IF NOT EXISTS effort VARCHAR(16);
ALTER TABLE optimization_recommendations ADD COLUMN IF NOT EXISTS savings_breakdown JSONB DEFAULT '{}';
ALTER TABLE optimization_recommendations ADD COLUMN IF NOT EXISTS category VARCHAR(32);
ALTER TABLE optimization_recommendations ADD COLUMN IF NOT EXISTS subcategory VARCHAR(32);
ALTER TABLE optimization_recommendations ADD COLUMN IF NOT EXISTS first_detected_at TIMESTAMPTZ;
ALTER TABLE optimization_recommendations ADD COLUMN IF NOT EXISTS last_observed_at TIMESTAMPTZ;
```

---

## Appendix B: Key Risks and Assumptions

### Risks

1. **Utilization Data Gap:** Without CPU/memory metrics, compute rightsizing confidence is 30-50%. This limits the credibility of the most impactful recommendations.

2. **AWS Dependency:** Current optimisation engine is 90% AWS-dependent. GCP and Azure users get minimal recommendations. This creates a poor experience for multi-cloud tenants.

3. **Data Freshness:** GCP sync frequency is configurable but defaults may leave inventory data stale for days.

4. **Pricing Accuracy:** `cost_pricing` table is sparsely populated. Savings calculations use estimates, not actual provider pricing.

5. **User Trust:** Hardcoded defaults ($50 idle EC2, 20% SP discount) erode trust when actual savings differ.

### Assumptions

1. Tenants want cost optimisation recommendations (validated by existing optimisation page usage)
2. Tags are correctly applied (incorrect tags lead to wrong recommendations)
3. Resource inventory is current (stale inventory → missing recommendations)
4. Cost data is accurate (billing data errors propagate to recommendations)

---

*This analysis is based on a complete codebase audit of ~473 frontend files, ~200+ backend files, 35 database migrations, 11 background workers, and 30+ service modules. Every recommendation is grounded in the current architecture and can be built without copying competitors.*
