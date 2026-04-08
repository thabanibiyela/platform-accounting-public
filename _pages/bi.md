---
layout: default
title: "Stakeholder Reporting"
permalink: /bi
---

# Stakeholder Reporting — Google Looker Studio

As custodians of platform accounting data, the platform accounting team is responsible for providing stakeholders with timely, accurate insight into account balances, revenue trends, and variance drivers. Before this project, that meant fielding ad hoc data requests by email — processing each one manually and responding within 24–48 hours.

This component replaces that reactive model with **interactive, self-service dashboards** built in Google Looker Studio, connected directly to the BigQuery data model.

**Technology:** Google Looker Studio · Google BigQuery · dbt (presentation layer)

---

## Requirements Gathering

Dashboard requirements were gathered by:

1. **Reviewing historical request logs** — analysing recurring themes in stakeholder data requests (email and messaging records over a 12-month period)
2. **Walkthrough sessions** — structured sessions with the finance business partner and management reporting teams to understand how financial data is consumed and acted upon
3. **Dimension mapping** — translating business information needs into data dimensions available in the BigQuery model

The requirements exercise identified five standard dimensions that appear across all reporting use cases:

| Dimension | Description |
|---|---|
| Accounting period | Month and year of the reported data |
| Country / market | Geographic breakdown by entity |
| Ledger account | GL account number and description |
| Business unit | Workday business unit worktag |
| Revenue category | Revenue type (e.g. commission, service fee, accrual) |

---

## Data Model for BI

A dedicated presentation layer in the dbt project (`4_Reporting/bi_reporting/`) produces the views consumed by Looker Studio. These models join the aggregated revenue data with accounting master data to produce human-readable dimension labels alongside the financial figures.

### Key View: `revenue_anayltics_end_user_bi`

The primary BI view includes:

- **Current period metrics:** revenue, order count, GMV
- **Prior period comparators:** same metrics for the prior accounting period
- **Pre-computed variances:** revenue variance, order count variance, GMV variance
- **Rate metrics:** revenue per order, GMV per order
- **Variance decomposition:** price and volume components of the revenue variance

```sql
-- revenue_anayltics_end_user_bi.sql (key columns)
SELECT
    cp.acc_period,
    md.country_name,
    cp.ledger_account,
    bu.bu                              AS business_unit,
    acc_md.revenue_category            AS rev_cat_spc,

    -- Current period
    cp.order_count,
    ROUND(cp.revenue, 2)               AS revenue,
    ROUND(cp.gmv, 2)                   AS gmv,
    ROUND(cp.revenue_per_order, 2)     AS revenue_per_order,

    -- Prior period
    ROUND(pp.order_count, 2)           AS pp_order_count,
    ROUND(pp.revenue, 2)               AS pp_revenue,

    -- Variances
    ROUND(cp.revenue - pp.revenue, 2)  AS revenue_variance,

    -- Price and volume decomposition
    ROUND(
      pp.revenue_per_order * (cp.order_count - pp.order_count), 2
    ) AS volume_variance_curr,
    ROUND(
      (cp.revenue_per_order - pp.revenue_per_order) * cp.order_count, 2
    ) AS price_variance_curr

FROM report AS cp
LEFT JOIN report AS pp
  ON cp.prior_period = pp.acc_period
  AND cp.company_id  = pp.company_id
  AND cp.business_unit = pp.business_unit
  ...
```

This pre-computation in SQL minimises the calculation burden on Looker Studio and ensures consistent variance logic across all dashboards.

### Revenue Reconciliation View: `revenue_reconciliation`

A separate view supporting the reconciliation dashboard joins partner and consumer revenue streams, enriched with accounting master data:

```sql
-- revenue_reconciliation.sql (structure)
WITH
  partner_revenue  AS ( SELECT * FROM stg_partner_revenue_analytics  ),
  consumer_revenue AS ( SELECT * FROM stg_consumer_revenue_analytics ),
  total_revenue    AS (
    SELECT * FROM partner_revenue
    UNION ALL
    SELECT * FROM consumer_revenue
  )
SELECT
    acc_period,
    company_md.country_name,
    ledger_account,
    bu.bu            AS business_unit,
    revenue_category AS rev_cat_spc,
    accrued,
    ROUND(order_count, 0)  AS order_count,
    ROUND(gmv, 2)          AS gmv,
    ROUND(revenue, 2)      AS revenue
FROM total_revenue
LEFT JOIN accounting_md ...
LEFT JOIN bu ...
LEFT JOIN company_md ...
```

### Workday MoM Analytics: `workday_mom_analytics`

A complementary view provides a month-on-month comparison of posted Workday data for the account reconciliation dashboards, enabling trend analysis directly from the accounting system's perspective.

---

## Dashboard Design

Each Looker Studio dashboard is structured around four standard panels. The filter bar at the top cascades across all panels — stakeholders select their market, account, and period once and all views update.

### Panel 1 — Periodic Overview

A summary table showing headline account balances for the selected accounting period. Columns:

| Column | Description |
|---|---|
| Country | Geographic market |
| Revenue Category | Revenue type (commission, accrual, etc.) |
| Business Unit | Workday business unit |
| Orders | Transaction volume |
| GMV | Gross merchandise value |
| Revenue | Net revenue for the period |
| Prior Period Revenue | Prior month comparator |
| Variance | Current vs prior period (absolute) |
| Variance % | Current vs prior period (percentage) |

### Panel 2 — Month-on-Month Trend

A time-series line chart plotting revenue across the trailing 12 months, broken down by revenue category. This panel helps stakeholders identify seasonal patterns, growth trends, and anomalies at a glance.

### Panel 3 — Variance Analysis

A tabular breakdown of the period-on-period variance, decomposed into its component drivers:

| Metric | Description |
|---|---|
| Revenue Variance | Total change in revenue vs prior period |
| Volume Variance | Variance attributable to change in order volume |
| Price Variance | Variance attributable to change in revenue per order |
| Volume Variance % | Volume component as % of total variance |
| Price Variance % | Price component as % of total variance |

The price/volume decomposition uses the standard variance analysis formula:
- **Volume effect:** Prior RPO × (Current orders − Prior orders)
- **Price effect:** (Current RPO − Prior RPO) × Current orders

where RPO = Revenue Per Order.

### Panel 4 — Geographic Detail

A country-level card grid showing each market's performance independently. Each card displays:

- Revenue (current period)
- Prior period revenue
- Period-on-period change
- Order count and GMV

Country-level filtering lets stakeholders drill into a single market across all panels simultaneously.

---

## Filtering and Interactivity

Looker Studio filters are configured at the report level and applied across all charts. Available filter dimensions:

| Filter | Options |
|---|---|
| Accounting Period | Select one or multiple months |
| Country / Market | Multi-select from all 17 markets |
| Ledger Account | Select specific GL accounts |
| Business Unit | Filter by Workday business unit |
| Revenue Category | Filter by revenue type (invoiced, accrued, consumer) |
| Accrued Flag | Toggle between accrued and non-accrued revenue |

---

## Data Freshness

Dashboard data is updated daily when the continuous monitoring workflow materialises the BigQuery tables. The monitoring workflow acts as a data quality gate — dashboards always reflect tested, verified data.

- **Normal periods:** Data refreshed daily (overnight materialisation)
- **Month-end peak (first 5 working days):** Data refreshed every 30 minutes (accounting data extraction SLA)

Looker Studio is configured to cache data for 12 hours to balance freshness with query costs.

---

## Impact

| Before | After |
|---|---|
| Stakeholders email data requests | Stakeholders access live dashboards directly |
| Team responds within 24–48 hours | Data available on demand, 24/7 |
| Static outputs, manual preparation | Interactive filters, self-service exploration |
| Ad hoc format, inconsistent methodology | Consistent methodology, pre-computed variances |
| Reporting backlog during month-end | No backlog — data always current |

The dashboards eliminated the team's ad hoc reporting workload entirely. The team shifted from reactive data production to proactive analysis and commentary — a qualitative improvement that freed time for higher-value work.

---

[Back to Overview](/) · [Data Model](data-model) · [Workflows](workflows) · [CLI Application](cli) · [BI Reporting](bi) · [Extended Report](report) 
