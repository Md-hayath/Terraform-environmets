# FOCUS Use Cases — Complete Map vs Our Implementation

> **Based on**: FOCUS Use Case Library at `https://focus.finops.org/use-cases/`
> **Versions analyzed**: v1.0 (36) → v1.1 (40) → v1.2 (52) → v1.3 (59) → v1.4 (70)
> **Our codebase at**: Alembic head 0023, FOCUS 1.2 partial (37 of 107 columns)
> **Date**: July 2026

---

## 1. Version Evolution Summary

| Version | Use Cases | New | Key Additions |
|---------|-----------|-----|---------------|
| v1.0 | 36 | 36 | Foundation: cost-by-service, cost-by-region, month-over-month, tag coverage |
| v1.1 | 40 | 4 | Capacity reservations, SKU metered costs, service category+subcategory, capacity reservation anomaly |
| v1.2 | 52 | 12 | Invoice reconciliation, commitment tracking, virtual currency, unit economics, corrections, effective savings rate, forecasting |
| v1.3 | 59 | 7 | Multi-currency allocation, participating entities, shared cost allocation, marketplace tracking, contract commitment burn-down, initial commitments, service costs by subaccount |
| v1.4 | 70 | 11 | Detailed invoice reconciliation (3), Rate Optimization (6 new: coverage rate, cost breakdown, cross-provider comparison, discount by service, eligible capacity spend, eligible uncovered spend), effective cost by region, cash vs accrual |

**Total growth**: 36 → 70 (94% increase from v1.0 to v1.4)

---

## 2. Complete v1.4 Use Case Inventory (70 Total)

### 2.1 Allocation (12)

| # | Use Case | Slug | Added | Columns Required | Our Status |
|---|----------|------|-------|-----------------|------------|
| 1 | Allocate multi-currency charges per application | `allocate-multi-currency-charges-per-application-2` | v1.3 | `BilledCost`, `EffectiveCost`, `BillingCurrency`, `PricingCurrency`, `Tags`, `ResourceId` | ⚠️ Partial — multi-currency aware but PricingCurrency missing |
| 2 | Analyze cost by participating entities | `analyze-cost-by-participating-entities` | v1.3 | `ServiceProviderName`, `HostProviderName`, `PublisherName`, `InvoiceIssuerName`, `EffectiveCost` | ❌ Missing — ServiceProviderName/HostProviderName not in schema |
| 3 | Analyze effective cost by pricing currency | `analyze-effective-cost-by-pricing-currency-2` | v1.3 | `EffectiveCost`, `PricingCurrency`, `BillingCurrency`, `ChargePeriodStart` | ❌ Missing — PricingCurrency not in schema |
| 4 | Analyze purchase of virtual currency | `analyze-purchase-of-virtual-currency-2` | v1.3 | `ChargeCategory`, `ChargeClass`, `EffectiveCost`, `SkuId`, `SkuPriceId`, `Tags` | ⚠️ Partial — ChargeCategory/ChargeClass exist but SKU fields are sparse |
| 5 | Analyze tag coverage | `analyze-tag-coverage` | v1.0 | `Tags`, `ResourceId`, `EffectiveCost` | ✅ Supported — tags as JSONB, queryable via GIN |
| 6 | Analyze total cost by allocated resource | `analyze-total-cost-by-allocated-resource` | v1.3 | `AllocatedResourceId`, `AllocatedResourceName`, `EffectiveCost`, `ChargePeriodStart` | ❌ Missing — AllocatedResource columns not in schema |
| 7 | Determine target of virtual currency usage | `determine-target-of-virtual-currency-usage-2` | v1.3 | `ChargeCategory`, `ChargeClass`, `SkuId`, `Tags`, `ResourceId` | ⚠️ Partial — needs `ChargeFrequency` column |
| 8 | Identify resources with shared cost allocation | `identify-resources-with-shared-cost-allocation` | v1.3 | `AllocatedMethodId`, `AllocatedMethodDetails`, `EffectiveCost` | ❌ Missing — AllocatedMethod columns not in schema |
| 9 | Identify sources of billed cost | `identify-sources-of-billed-cost-2` | v1.3 | `BilledCost`, `ServiceCategory`, `ServiceName`, `SubAccountId`, `RegionName` | ✅ Supported — all columns exist |
| 10 | Report corrections by subaccount for a previously invoiced billing period | `report-corrections-by-subaccount-for-a-previously-invoiced-billing-period` | v1.3 | `ChargeCategory`, `ChargeClass`, `BilledCost`, `SubAccountId`, `BillingPeriodStart`, `BillingPeriodEnd` | ⚠️ Partial — needs `ChargeFrequency` and corrections classification |
| 11 | Track marketplace purchases across providers | `track-marketplace-purchases-across-providers` | v1.3 | `ServiceProviderName`, `HostProviderName`, `PublisherName`, `EffectiveCost`, `ServiceCategory` | ❌ Missing — ServiceProviderName/HostProviderName not in schema |
| 12 | Verify accuracy of provider invoices aka invoice reconciliation | `verify-accuracy-of-provider-invoices-aka-invoice-reconciliation` | v1.2 | `InvoiceId`, `InvoiceDetailId`, `BilledCost`, `EffectiveCost`, `ChargeCategory`, `BillingPeriodStart` | ❌ Missing — InvoiceId/InvoiceDetailId not in schema |

### 2.2 Anomaly Management (7)

| # | Use Case | Slug | Added | Columns Required | Our Status |
|---|----------|------|-------|-----------------|------------|
| 13 | Analyze costs by service name | `analyze-costs-by-service-name` | v1.0 | `ServiceName`, `EffectiveCost`, `ChargePeriodStart`, `ChargePeriodEnd` | ✅ Fully supported — our primary aggregation |
| 14 | Analyze service costs by region | `analyze-service-costs-by-region-2` | v1.0 | `ServiceName`, `RegionName`, `EffectiveCost`, `ChargePeriodStart` | ✅ Supported |
| 15 | Compare resource usage month over month | `compare-resource-usage-month-over-month-2` | v1.0 | `ResourceId`, `ResourceName`, `ConsumedQuantity`, `ConsumedUnit`, `ChargePeriodStart` | ⚠️ Partial — no period-over-period comparison query; no MoM % change in UI |
| 16 | Identify anomalous daily spending by subaccount | `identify-anomalous-daily-spending-by-subaccount` | v1.0 | `EffectiveCost`, `SubAccountId`, `ChargePeriodStart` (daily) | ⚠️ Partial — anomalies exist in cost-intelligence but not FOCUS-native; daily aggregation not optimized |
| 17 | Identify anomalous daily spending by subaccount and region | `identify-anomalous-daily-spending-by-subaccount-and-region` | v1.0 | `EffectiveCost`, `SubAccountId`, `RegionName`, `ChargePeriodStart` (daily) | ⚠️ Partial — same gap |
| 18 | Identify anomalous daily spending by subaccount, region, and service | `identify-anomalous-daily-spending-by-subaccount-region-and-service` | v1.0 | `EffectiveCost`, `SubAccountId`, `RegionName`, `ServiceName`, `ChargePeriodStart` (daily) | ⚠️ Partial — same gap |
| 19 | Identify unused capacity reservations | `identify-unused-capacity-reservations-2` | v1.1 | `CommitmentDiscountId`, `CommitmentDiscountType`, `CommitmentDiscountStatus`, `EffectiveCost`, `ChargeCategory` | ⚠️ Partial — commitment columns exist but Capacity Reservation tracking is not in schema |

### 2.3 Budgeting (5)

| # | Use Case | Slug | Added | Columns Required | Our Status |
|---|----------|------|-------|-----------------|------------|
| 20 | Calculate consumption of virtual currency within a billing period | `calculate-consumption-of-virtual-currency-within-a-billing-period-2` | v1.3 | `ChargeCategory`, `ChargeClass`, `ChargeFrequency`, `ConsumedQuantity`, `BillingPeriodStart`, `BillingPeriodEnd` | ❌ Missing — ChargeFrequency not in schema |
| 21 | Compare billed cost per subaccount to budget | `compare-billed-cost-per-subaccount-to-budget-2` | v1.0 | `BilledCost`, `SubAccountId`, `BillingPeriodStart`, `BillingPeriodEnd` | ⚠️ Partial — budget UIs exist but budget-vs-actual isn't FOCUS-native; uses separate table |
| 22 | Track contract commitment burn-down over time | `track-contract-commitment-burn-down-over-time` | v1.3 | `ContractCommitmentId`, `ContractCommitmentQuantity`, `ContractCommitmentUnit`, `EffectiveCost` | ❌ Missing — no focus_commitments table |
| 23 | Update budgets for each application | `update-budgets-for-each-application` | v1.0 | `BilledCost`, `Tags` (application), `SubAccountId` | ⚠️ Partial — budget UIs exist but write-back not implemented via API |
| 24 | Update budgets with billed costs | `update-budgets-with-billed-costs` | v1.0 | `BilledCost`, `BillingPeriodStart`, `BillingPeriodEnd` | ⚠️ Partial — same gap |

### 2.4 Data Ingestion (3)

| # | Use Case | Slug | Added | Columns Required | Our Status |
|---|----------|------|-------|-----------------|------------|
| 25 | Verify accuracy of service provider invoices | `verify-accuracy-of-service-provider-invoices` | v1.0 | `InvoiceId`, `BilledCost`, `EffectiveCost`, `ServiceProviderName` | ❌ Missing — InvoiceId not in schema |
| 26 | Verify accuracy of services charges across service providers | `verify-accuracy-of-services-charges-across-service-providers` | v1.0 | `ServiceProviderName`, `PublisherName`, `BilledCost`, `EffectiveCost` | ❌ Missing — ServiceProviderName not in schema |
| 27 | Verify discount accuracy for a previously invoiced billing period (corrections excluded) | `verify-discount-accuracy-for-a-previously-invoiced-billing-period-corrections-excluded` | v1.0 | `CommitmentDiscountId`, `CommitmentDiscountType`, `ListCost`, `EffectiveCost`, `BilledCost`, `BillingPeriodStart` | ⚠️ Partial — commitment columns exist but no verification query |

### 2.5 Forecasting (4)

| # | Use Case | Slug | Added | Columns Required | Our Status |
|---|----------|------|-------|-----------------|------------|
| 28 | Calculate average rate of a component resource | `calculate-average-rate-of-a-component-resource-2` | v1.2 | `EffectiveCost`, `ConsumedQuantity`, `ConsumedUnit`, `ResourceId`, `ChargePeriodStart` | ✅ Supported — can compute cost/quantity ratio |
| 29 | Forecast amortized costs month over month based on historical trends | `forecast-amortized-costs-month-over-month-based-on-historical-trends-2` | v1.0 | `EffectiveCost`, `ChargePeriodStart` (monthly) | ⚠️ Partial — forecast exists in cost-intelligence but not FOCUS-native; separate table/model |
| 30 | Forecast cashflow month over month based on historical trends by service | `forecast-cashflow-month-over-month-based-on-historical-trends-by-service-2` | v1.0 | `BilledCost`, `ServiceName`, `ChargePeriodStart` (monthly) | ⚠️ Partial — same gap |
| 31 | Get historical usage and rates to enable cost forecasting | `get-historical-usage-and-rates-to-enable-cost-forecasting` | v1.0 | `ConsumedQuantity`, `ConsumedUnit`, `EffectiveCost`, `PricingQuantity`, `PricingUnit`, `ChargePeriodStart` | ✅ Supported — all columns exist |

### 2.6 Governance, Policy & Risk (1)

| # | Use Case | Slug | Added | Columns Required | Our Status |
|---|----------|------|-------|-----------------|------------|
| 32 | Report subaccounts by region | `report-subaccounts-by-region` | v1.0 | `SubAccountId`, `SubAccountName`, `RegionName` | ✅ Supported — both columns exist |

### 2.7 Invoicing & Chargeback (4)

| # | Use Case | Slug | Added | Columns Required | Our Status |
|---|----------|------|-------|-----------------|------------|
| 33 | Reconcile Cost and Usage to Invoice Detail by Invoice ID | `reconcile-cost-and-usage-to-invoice-detail-by-invoice-id` | v1.4 | `InvoiceId`, `InvoiceDetailId`, `BilledCost`, `EffectiveCost`, `ChargeCategory`, `BillingPeriodStart` | ❌ Missing — InvoiceId not in schema; no focus_invoices table |
| 34 | Reconcile Multi-Currency Settlement using Lineage IDs | `reconcile-multi-currency-settlement-using-lineage-ids` | v1.4 | `InvoiceId`, `BilledCost`, `BillingCurrency`, `PricingCurrency`, `LineageId`, `SettlementCurrency` | ❌ Missing — LineageId, SettlementCurrency not in schema; no invoice table |
| 35 | Understand the billing account or sub account entity | `understand-the-billing-account-or-sub-account-entity-2` | v1.2 | `BillingAccountId`, `BillingAccountName`, `BillingAccountType`, `SubAccountId`, `SubAccountName`, `SubAccountType` | ❌ Missing — BillingAccountId/Name/Type not in schema; SubAccountType missing |
| 36 | Validate Tax Variance | `validate-tax-variance` | v1.4 | `InvoiceId`, `BilledCost`, `EffectiveCost`, `TaxAmount`, `ChargeCategory` | ❌ Missing — TaxAmount not in schema; no invoice table |

### 2.8 Planning & Estimating (4)

| # | Use Case | Slug | Added | Columns Required | Our Status |
|---|----------|------|-------|-----------------|------------|
| 37 | Determine contracted savings by virtual currency | `determine-contracted-savings-by-virtual-currency` | v1.2 | `ContractedCost`, `ListCost`, `EffectiveCost`, `ChargeCategory`, `SkuId`, `Tags` | ❌ Missing — virtual currency concept not mapped |
| 38 | Join contract commitment details with usage charges | `join-contract-commitment-details-with-usage-charges` | v1.3 | `ContractCommitmentId`, `CommitmentDiscountId`, `EffectiveCost`, `ChargePeriodStart` | ❌ Missing — no focus_commitments table |
| 39 | Quantify usage of a component resource | `quantify-usage-of-a-component-resource-2` | v1.2 | `ConsumedQuantity`, `ConsumedUnit`, `EffectiveCost`, `ResourceId`, `ChargePeriodStart` | ✅ Supported |
| 40 | Report effective cost of compute | `report-effective-cost-of-compute` | v1.0 | `EffectiveCost`, `ServiceCategory` (Compute), `ChargePeriodStart` | ✅ Supported |

### 2.9 Rate Optimization (8)

| # | Use Case | Slug | Added | Columns Required | Our Status |
|---|----------|------|-------|-----------------|------------|
| 41 | Calculate Commitment Discount Coverage Rate with Eligibility-Adjusted Denominator | `calculate-commitment-discount-coverage-rate-with-eligibility-adjusted-denominator` | v1.4 | `CommitmentDiscountId`, `CommitmentProgramEligibility`, `EffectiveCost`, `ChargeCategory`, `ServiceCategory` | ❌ Missing — CommitmentProgramEligibility not in schema |
| 42 | Commitment Discount Effective Cost Breakdown | `commitment-discount-effective-cost-breakdown` | v1.4 | `CommitmentDiscountId`, `CommitmentDiscountName`, `EffectiveCost`, `ListCost`, `ChargePeriodStart` | ❌ Missing — CommitmentDiscountName not in schema |
| 43 | Compare Commitment Opportunities Across Providers (Cross-Provider with SaaS) | `compare-commitment-opportunities-across-providers-cross-provider-with-saas` | v1.4 | `CommitmentDiscountId`, `ServiceProviderName`, `EffectiveCost`, `ListCost`, `ChargeCategory` | ❌ Missing — ServiceProviderName not in schema; no commitment comparison capability |
| 44 | Discount Effectiveness by Service | `discount-effectiveness-by-service` | v1.4 | `CommitmentDiscountId`, `ServiceName`, `ListCost`, `EffectiveCost`, `ChargePeriodStart` | ⚠️ Partial — commitment columns exist but no effectiveness query |
| 45 | Identify Eligible Capacity Reservation Spend | `identify-eligible-capacity-reservation-spend` | v1.4 | `CommitmentProgramEligibility`, `EffectiveCost`, `ServiceName`, `ChargeCategory`, `RegionName` | ❌ Missing — CommitmentProgramEligibility not in schema |
| 46 | Identify Eligible Uncovered Spend by Program Type | `identify-eligible-uncovered-spend-by-program-type` | v1.4 | `CommitmentProgramEligibility`, `EffectCost`, `ServiceCategory`, `CommitmentDiscountType`, `ChargeCategory` | ❌ Missing — CommitmentProgramEligibility not in schema |
| 47 | Identify unused commitments | `identify-unused-commitments-2` | v1.2 | `CommitmentDiscountId`, `CommitmentDiscountType`, `CommitmentDiscountStatus`, `EffectiveCost`, `ChargePeriodStart` | ⚠️ Partial — commitment columns exist but no unused-commitment query |
| 48 | Report commitment discount purchases | `report-commitment-discount-purchases` | v1.2 | `ChargeCategory` (Purchase), `CommitmentDiscountId`, `EffectiveCost`, `CommitmentDiscountType` | ⚠️ Partial — charge_category exists but purchase vs usage distinction not explicit |

### 2.10 Reporting & Analytics (18)

| # | Use Case | Slug | Added | Columns Required | Our Status |
|---|----------|------|-------|-----------------|------------|
| 49 | Analyze capacity reservations on compute costs | `analyze-capacity-reservations-on-compute-costs-2` | v1.1 | `CommitmentDiscountType`, `ServiceCategory` (Compute), `EffectiveCost`, `RegionName` | ⚠️ Partial — needs capacity reservation type distinction |
| 50 | Analyze marketplace vendors costs | `analyze-marketplace-vendors-costs` | v1.0 | `PublisherName`, `ServiceCategory`, `EffectiveCost`, `ChargePeriodStart` | ✅ Supported |
| 51 | Analyze resource costs by SKU | `analyze-resource-costs-by-sku` | v1.0 | `SkuId`, `SkuPriceId`, `EffectiveCost`, `ResourceId`, `ChargePeriodStart` | ✅ Supported — both SKU columns exist |
| 52 | Analyze service costs by subaccount | `service-costs-subaccount` | v1.3 | `ServiceName`, `SubAccountId`, `EffectiveCost`, `ChargePeriodStart` | ✅ Supported |
| 53 | Analyze service costs month over month | `analyze-service-costs-month-over-month` | v1.0 | `ServiceName`, `EffectiveCost`, `ChargePeriodStart` (monthly) | ✅ Supported — timeseries with month granularity |
| 54 | Analyze the different metered costs for a particular SKU | `analyze-the-different-metered-costs-for-a-particular-sku-2` | v1.1 | `SkuId`, `SkuMeter`, `SkuPriceDetails`, `ConsumedQuantity`, `EffectiveCost`, `PricingQuantity` | ❌ Missing — SkuMeter, SkuPriceDetails not in schema |
| 55 | Calculate unit economics | `calculate-unit-economics` | v1.2 | `EffectiveCost`, `ConsumedQuantity`, `ConsumedUnit`, `Tags`, `ResourceId` | ✅ Supported — uses business_metrics table |
| 56 | Cash vs. Accrual Comparison by Billing Period | `cash-vs-accrual-comparison-by-billing-period` | v1.4 | `BilledCost`, `EffectiveCost`, `ChargeClass`, `BillingPeriodStart`, `BillingPeriodEnd`, `ChargePeriodStart` | ❌ Missing — accrual vs cash distinction not implemented |
| 57 | Determine Effective Savings Rate | `determine-effective-savings-rate` | v1.2 | `EffectiveCost`, `ListCost`, `CommitmentDiscountId`, `ChargePeriodStart` | ⚠️ Partial — can compute; no formal rate query |
| 58 | Determine Effective Savings Rate by Service | `determine-effective-savings-rate-by-service` | v1.2 | `EffectiveCost`, `ListCost`, `CommitmentDiscountId`, `ServiceName`, `ChargePeriodStart` | ⚠️ Partial — same gap |
| 59 | Effective Cost by Service and Region | `effective-cost-by-service-and-region` | v1.4 | `EffectiveCost`, `ServiceName`, `RegionName`, `ChargePeriodStart` | ✅ Supported |
| 60 | Report application cost month over month | `application-cost` | v1.0 | `EffectiveCost`, `Tags` (application), `ChargePeriodStart` | ✅ Supported — tags as JSONB |
| 61 | Report corrections for a previously invoiced billing period | `report-corrections-for-a-previously-invoiced-billing-period` | v1.0 | `ChargeClass` (correction), `BilledCost`, `BillingPeriodStart`, `BillingPeriodEnd` | ❌ Missing — no corrections-specific logic |
| 62 | Report costs by service category | `report-costs-by-service-category` | v1.0 | `ServiceCategory`, `EffectiveCost`, `ChargePeriodStart` | ✅ Supported |
| 63 | Report costs by service category and subcategory | `report-costs-by-service-category-and-subcategory-2` | v1.1 | `ServiceCategory`, `ServiceSubcategory`, `EffectiveCost` | ✅ Supported — both columns exist |
| 64 | Report on initial contract commitments | `report-on-initial-contract-commitments` | v1.3 | `ContractCommitmentId`, `CommitmentDiscountId`, `EffectiveCost`, `ChargePeriodStart`, `BillingPeriodStart` | ❌ Missing — no focus_commitments table |
| 65 | Report service costs by providers subaccount | `report-service-costs-by-providers-subaccount-2` | v1.3 | `ServiceProviderName`, `SubAccountId`, `EffectiveCost`, `ChargePeriodStart` | ❌ Missing — ServiceProviderName not in schema |
| 66 | Report spending across billing periods for a service provider by service category | `report-spending-across-billing-periods-for-a-service-provider-by-service-category` | v1.0 | `ServiceProviderName`, `ServiceCategory`, `BilledCost`, `BillingPeriodStart` | ❌ Missing — ServiceProviderName not in schema |

### 2.11 Unit Economics (1)

| # | Use Case | Slug | Added | Columns Required | Our Status |
|---|----------|------|-------|-----------------|------------|
| 67 | Analyze credit memos | `analyze-credit-memos-2` | v1.2 | `ChargeClass` (credit), `BilledCost`, `EffectiveCost`, `InvoiceId`, `ChargePeriodStart` | ❌ Missing — credit memos not tracked; InvoiceId not in schema |

### 2.12 Usage Optimization (3)

| # | Use Case | Slug | Added | Columns Required | Our Status |
|---|----------|------|-------|-----------------|------------|
| 68 | Analyze cost per compute service for a subaccount | `cost-compute-subaccount` | v1.0 | `ServiceCategory` (Compute), `ServiceName`, `SubAccountId`, `EffectiveCost` | ✅ Supported |
| 69 | Analyze costs by availability zone for a subaccount | `analyze-costs-by-availability-zone-for-a-subaccount` | v1.0 | `AvailabilityZone`, `SubAccountId`, `EffectiveCost`, `ChargePeriodStart` | ❌ Missing — AvailabilityZone not in schema |
| 70 | Analyze costs of components of a resource | `resource-component-costs` | v1.0 | `ResourceId`, `ResourceName`, `ServiceName`, `EffectiveCost`, `ConsumedQuantity` | ✅ Supported |

---

## 3. Implementation Status Summary

### 3.1 Overall Stats

| Status | Count | % |
|--------|-------|---|
| ✅ Fully Supported | 17 | 24% |
| ⚠️ Partially Supported | 20 | 29% |
| ❌ Missing | 33 | 47% |

### 3.2 By Capability

| Capability | Total | ✅ | ⚠️ | ❌ | Coverage |
|------------|-------|-----|------|-----|----------|
| Allocation | 12 | 2 | 3 | 7 | 17% |
| Anomaly Management | 7 | 1 | 6 | 0 | 14% |
| Budgeting | 5 | 0 | 4 | 1 | 0% |
| Data Ingestion | 3 | 0 | 1 | 2 | 0% |
| Forecasting | 4 | 2 | 2 | 0 | 50% |
| Governance | 1 | 1 | 0 | 0 | 100% |
| Invoicing & Chargeback | 4 | 0 | 0 | 4 | 0% |
| Planning & Estimating | 4 | 2 | 0 | 2 | 50% |
| Rate Optimization | 8 | 0 | 3 | 5 | 0% |
| Reporting & Analytics | 18 | 9 | 3 | 6 | 50% |
| Unit Economics | 1 | 0 | 0 | 1 | 0% |
| Usage Optimization | 3 | 2 | 0 | 1 | 67% |

### 3.3 Critical Gaps (P0 — Blockers)

| Missing Component | Blocks These Use Cases | Priority |
|------------------|----------------------|----------|
| **`InvoiceId` / `InvoiceDetailId` columns** | #12, #25, #33, #34, #36, #67 | P0 |
| **`focus_invoices` table** | #12, #33, #34, #36 | P0 |
| **`ServiceProviderName` / `HostProviderName` columns** | #2, #11, #26, #43, #65, #66 | P0 |
| **`BillingAccountId` / `BillingAccountName` / `BillingAccountType` columns** | #35 | P0 |
| **`AvailabilityZone` column** | #69 | P0 |
| **`CommitmentProgramEligibility` column** | #41, #45, #46 | P0 |
| **`AllocatedResourceId` / `AllocatedResourceName` / `AllocatedMethodId` / `AllocatedMethodDetails` columns** | #6, #8 | P0 |
| **`focus_commitments` table** | #22, #38, #64 | P0 |

### 3.4 Important Gaps (P1 — High Value)

| Missing Component | Blocks These Use Cases | Priority |
|------------------|----------------------|----------|
| `PricingCurrency` column | #1, #3, #34 | P1 |
| `ChargeFrequency` column | #7, #10, #20 | P1 |
| `SkuMeter` / `SkuPriceDetails` columns | #54 | P1 |
| `CommitmentDiscountName` column | #42 | P1 |
| `SubAccountType` column | #35 | P1 |
| `TaxAmount` column | #36 | P1 |
| Period-over-period comparison query | #15 | P1 |
| FOCUS-native anomaly detection | #16, #17, #18 | P1 |
| Budget-vs-actual FOCUS integration | #21, #23, #24 | P1 |

---

## 4. Column Gap Analysis for Use Case Support

### 4.1 New Columns Required (Priority Order)

| Column | FOCUS Dataset | Use Cases Enabled | Migration | Priority |
|--------|---------------|-------------------|-----------|----------|
| `invoice_id` | Cost and Usage | #12, #25, #33, #34, #36, #67 | 0021 | P0 |
| `invoice_detail_id` | Cost and Usage | #12, #33 | 0021 | P0 |
| `service_provider_name` | Cost and Usage | #2, #11, #26, #43, #65, #66 | 0021 | P0 |
| `host_provider_name` | Cost and Usage | #2, #11 | 0021 | P0 |
| `billing_account_id` | Cost and Usage | #35 | 0021 | P0 |
| `billing_account_name` | Cost and Usage | #35 | 0021 | P0 |
| `billing_account_type` | Cost and Usage | #35 | 0021 | P0 |
| `sub_account_type` | Cost and Usage | #35 | 0021 | P0 |
| `availability_zone` | Cost and Usage | #69 | 0021 | P0 |
| `commitment_program_eligibility` (JSONB) | Cost and Usage | #41, #45, #46 | 0021 | P0 |
| `allocated_method_id` | Cost and Usage | #8 | 0021 | P1 |
| `allocated_method_details` (JSONB) | Cost and Usage | #8 | 0021 | P1 |
| `allocated_resource_id` | Cost and Usage | #6 | 0021 | P1 |
| `allocated_resource_name` | Cost and Usage | #6 | 0021 | P1 |
| `allocated_tags` (JSONB) | Cost and Usage | #6 | 0021 | P1 |
| `pricing_currency` | Cost and Usage | #1, #3, #34 | 0021 | P1 |
| `charge_frequency` | Cost and Usage | #7, #10, #20 | 0021 | P1 |
| `sku_meter` | Cost and Usage | #54 | 0021 | P1 |
| `sku_price_details` (JSONB) | Cost and Usage | #54 | 0021 | P1 |
| `commitment_discount_name` | Cost and Usage | #42 | 0021 | P1 |

### 4.2 New Tables Required

| Table | Use Cases Enabled | Migration | Priority |
|-------|------------------|-----------|----------|
| `focus_invoices` | #12, #33, #34, #36, #67 | 0022 | P0 |
| `focus_commitments` | #22, #38, #64 | 0023 | P0 |
| `focus_budgets` | #21, #23, #24 | 0024 | P1 |

---

## 5. Implementation Priority Matrix

### Phase 1 — Foundation (P0, Weeks 1-4)
*Enable 17 use cases (24% → 48%)*

| Component | Use Cases Unlocked | Effort |
|-----------|-------------------|--------|
| Migration 0021: Add 18 P0 columns to `cost_charges` | #2, #6, #8, #11, #12, #25, #26, #33, #34, #35, #36, #41, #43, #45, #46, #65, #66, #67, #69 | 2 days |
| Migration 0022: Create `focus_invoices` table | #12, #33, #34, #36, #67 | 1 day |
| Migration 0023: Create `focus_commitments` table | #22, #38, #64 | 1 day |
| Invoice reconciliation service + route | #12, #33, #34 | 3 days |
| `ServiceProviderName`/`HostProviderName` in normalize_charge_row() | #2, #11, #26, #43, #65, #66 | 1 day |
| `BillingAccountId/Name/Type` in normalize_charge_row() | #35 | 0.5 day |
| `AvailabilityZone` in normalize_charge_row() | #69 | 0.5 day |
| `CommitmentProgramEligibility` in normalize_charge_row() | #41, #45, #46 | 1 day |

### Phase 2 — Allocation & Reporting (P1, Weeks 5-8)
*Enable 10 more use cases (48% → 62%)*

| Component | Use Cases Unlocked | Effort |
|-----------|-------------------|--------|
| Migration 0024: Create `focus_budgets` table | #21, #23, #24 | 1 day |
| PricingCurrency in normalize_charge_row() | #1, #3, #34 | 0.5 day |
| ChargeFrequency in normalize_charge_row() | #7, #10, #20 | 0.5 day |
| SkuMeter/SkuPriceDetails in normalize_charge_row() | #54 | 0.5 day |
| CommitmentDiscountName implementation | #42 | 0.5 day |
| Period-over-period comparison query | #15 | 2 days |
| FOCUS-native anomaly detection service | #16, #17, #18 | 3 days |
| Budget-vs-actual FOCUS integration | #21, #23, #24 | 2 days |

### Phase 3 — Advanced (P2, Weeks 9-12)
*Enable remaining use cases (62% → 80%+)*

| Component | Use Cases Unlocked | Effort |
|-----------|-------------------|--------|
| Effective Savings Rate query | #57, #58 | 1 day |
| Commitment coverage rate service | #41, #45, #46 | 2 days |
| Cross-provider commitment comparison | #43 | 2 days |
| Multi-currency settlement reconciliation | #34 | 3 days |
| Tax variance validation | #36 | 1 day |
| Cash vs accrual comparison | #56 | 1 day |
| Credit memo tracking | #67 | 1 day |
| Corrections tracking | #10, #61 | 1 day |
| Virtual currency support | #4, #7, #20, #37 | 3 days |

### Phase 4 — Polish & AI Integration (P3, Weeks 13-16)
*Enable AI agent to answer all 70 use cases*

| Component | Use Cases Unlocked | Effort |
|-----------|-------------------|--------|
| FOCUS MCP server tool for FinOps agent | All spec-level questions | 2 days |
| NL→FOCUS query engine | All | 5 days |
| FOCUS conformance validator | #25, #26, #27 | 2 days |
| Scheduled FOCUS conformance reports | All | 2 days |
| Use case library integration in agent | All | 2 days |

---

## 6. What Each Version Added (Use Case Evolution)

### v1.0 → v1.1 (4 new)

The 4 use cases added in v1.1 all relate to **finer-grained cost analysis**:
- `analyze-capacity-reservations-on-compute-costs` — understanding how reserved capacity affects compute costs
- `analyze-the-different-metered-costs-for-a-particular-sku` — breaking down SKU-level costs by meter
- `report-costs-by-service-category-and-subcategory` — two-level category reporting
- `identify-unused-capacity-reservations` — anomaly detection for reservation waste

**Impact on us**: #49 and #54 need new columns (SkuMeter, SkuPriceDetails). #63 is already supported.

### v1.1 → v1.2 (12 new)

Major expansion into **commitments, forecasting, unit economics, and invoice reconciliation**:
- Invoice verification and reconciliation (3 use cases)
- Commitment tracking (purchases, unused, savings rate)
- Forecasting (trend, cashflow, historical rates)
- Unit economics
- Virtual currency support
- Cost corrections
- Billing account/sub-account entity understanding

**Impact on us**: This is the biggest gap — 8 of 12 new use cases require columns/tables we don't have.

### v1.2 → v1.3 (7 new)

Focus on **multi-entity, multi-provider, and allocation**:
- Multi-currency allocation
- Participating entities (ServiceProviderName/HostProviderName)
- Shared cost allocation
- Marketplace tracking
- Contract commitment burn-down
- Initial commitment reporting
- Service costs by subaccount

**Impact on us**: 5 of 7 need columns we don't have (ServiceProviderName, Allocated* columns, ContractCommitmentId).

### v1.3 → v1.4 (11 new)

Heavy focus on **Rate Optimization (6 use cases) and Invoicing (3)**:
- 6 commitment discount rate optimization use cases (coverage rate, cost breakdown, cross-provider comparison, discount by service, eligibility analysis)
- 3 invoice reconciliation use cases (by invoice ID, multi-currency settlement, tax validation)
- 2 reporting improvements (effective cost by region, cash vs accrual)

**Impact on us**: 10 of 11 require columns/tables not yet in our schema.

---

## 7. Key Observations

### 7.1 What We Do Well

1. **Basic cost exploration** — We support the 17 "Fully Supported" use cases well, covering basic cost-by-service, cost-by-region, cost-by-category, and month-over-month trending
2. **Tag-based allocation** — Our virtual tag engine covers the tag coverage and application cost use cases
3. **Unit economics** — Our business_metrics + unit_economics system maps to the FOCUS unit economics use case
4. **Multi-provider ingestion** — 7 adapters give us broad provider coverage for the ingestion use cases

### 7.2 Where We're Weak

1. **Invoice reconciliation** — 0 of 4 invoicing use cases are supported. No `InvoiceId`, no `focus_invoices` table, no reconciliation logic.
2. **Rate optimization** — 0 of 8 rate optimization use cases. No `CommitmentProgramEligibility`, no commitment coverage analysis.
3. **Commitment tracking** — All 4 commitment-related use cases across Planning and Reporting require the missing `focus_commitments` table.
4. **Provider/entity tracking** — Missing `ServiceProviderName` and `HostProviderName` blocks 6 use cases.
5. **Anomaly detection** — Our existing anomaly system is not FOCUS-native — it uses separate tables and connection-scoped logic, not FOCUS columns.

### 7.3 Version Trend

The FOCUS specification is **moving rapidly toward**:
1. **Invoice reconciliation** (3 new in v1.4) — procurement requirement
2. **Rate optimization** (6 new in v1.4) — savings visibility
3. **Multi-currency and cross-provider** — enterprise reality
4. **Commitment tracking and burn-down** — budget accountability

Our implementation priorities should follow this trend.

---

## 8. Frontend Component Gap Map

| Use Case Area | Existing Frontend | Missing |
|--------------|-------------------|---------|
| Cost by service/region/category | ✅ `CostExplorer` with 7-dim filter, group-by, chart, table | No comparison mode, no MoM % change |
| Tag coverage | ✅ Virtual tag management UI | No tag coverage % visualization |
| Capacity reservations | ❌ | No component |
| SKU metered costs | ❌ | No SKU drill-down component |
| Invoice reconciliation | ❌ | No invoice page, no recon dashboard |
| Commitment tracking | ❌ | No commitment dashboard |
| Rate optimization | ❌ | No savings rate, coverage, comparison UI |
| Budget vs actual | ⚠️ Budget dashboard exists | Not FOCUS-native, no commitment burn-down |
| Forecasting | ⚠️ Cost trend with forecast | Not FOCUS-native, separate from explorer |
| Anomaly detection | ⚠️ Anomaly list + routing | Not FOCUS-native, no daily anomaly by dim |
| Unit economics | ✅ Unit economics dashboard | Works — uses business_metrics |
| Corrections | ❌ | No corrections UI |
| Credit memos | ❌ | No credit memo tracking |
| Availability zone | ❌ | Not a filter dimension in explorer |
| Marketplace vendors | ❌ | No marketplace tracking |
| Cash vs accrual | ❌ | No comparison mode |

---

## Appendix: REST API Reference

The FOCUS Use Case Library can be queried programmatically:

```bash
# All use cases (v1.3/v1.4)
GET https://focus.finops.org/wp-json/focus/v1/use-cases

# Individual use case metadata
GET https://focus.finops.org/wp-json/wp/v2/use_case?slug=costs-service-name

# Focus versions taxonomy: 10=v1.0, 11=v1.1, 12=v1.2, 33=v1.3, 44=v1.4
```

The detailed use case content (Context, SQL Query, Columns, Personas, KPIs) is served via the FOCUS MCP Server at `https://focus.finops.org/wp-json/focus/v1/mcp` (requires MCP protocol client).
