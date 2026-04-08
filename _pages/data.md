---
layout: default
title: "Data Model"
permalink: /data-model
---

# Data Model — dbt on Google BigQuery

The data model is the backbone of the platform accounting automation system. Built in **Google BigQuery** and powered by **dbt (Data Build Tool)**, it transforms raw event-level platform transaction data into structured, auditable financial accounting data ready for journal posting, reconciliation, and stakeholder reporting.

---

## Architecture Overview

Data flows through six conceptual layers, from raw event data to consumption-ready outputs:

```
Raw Events (order management system)
    │
    ▼
Layer 1: Staging
    Renaming, type casting, field selection, basic filtering
    │
    ▼
Layer 2: Intermediate (Pre-Aggregation)
    Stream-specific integration — one model per revenue type
    │
    ▼
Layer 3: Enrichment
    Accounting master data applied — GL accounts, cost centres, business units
    │
    ▼
Layer 4: Aggregation
    Grouped to posting dimensions — views and physical tables
    │
    ▼
Layer 5: Presentation
    Consumption-ready — for the CLI application and Looker Studio
    │
    ▼
Layer X: Monitoring
    Assertion models — data quality checks materialised daily
```

---

## Project Structure

```
dbt/
├── models/
│   └── 1_core/
│       ├── 1_staging/
│       │   ├── 1_order_level/          # Order-level staging
│       │   ├── 2_invoice_level/        # Invoice-level staging
│       │   ├── 3_restaurant_master_data/   # Restaurant master data
│       │   └── 4_other_master_data/    # Accounting dates, couriers, locations
│       ├── 2a_consumer_b2c/            # Consumer delivery fee revenue stream
│       │   ├── 1_pre_aggregation/
│       │   └── aggregated/
│       ├── 2b_partner_b2b/             # Partner invoicing revenue stream
│       │   ├── 1_pre_aggregation/
│       │   └── aggregated/
│       ├── 3_journals/                 # Journal entry models and EIB templates
│       │   ├── 1_journal_data/
│       │   └── 2_eib_template/
│       ├── 4_Reporting/                # BI and reconciliation views
│       │   ├── bi_reporting/
│       │   └── monitoring/
│       └── X_test_monitoring/          # Data quality assertion models
│           ├── aggregated_cons_del_analytical_view/
│           ├── aggregated_qXX_partner_invoicing_view/
│           └── JE_revenue_journal/
├── seeds/
│   ├── master_data/                    # Accounting master data CSVs
│   ├── eib_template/                   # EIB import template
│   └── workday/                        # Workday mapping tables
└── tests/                              # Development-phase dbt tests
```

---

## Revenue Streams

The model processes two distinct revenue pillars in parallel tracks.

### B2C — Consumer Delivery Fee (`2a_consumer_b2c/`)

Processes delivery fees charged directly to consumers at point of order. The consumer stream is structurally simpler than the partner stream — revenue is recognised at the order level and the primary transformation is enriching order data with accounting dimensions.

**Key models:**
- `q1_230_consumer_delivery_fee.sql` — Pre-enrichment: applies accounting master data to consumer delivery fee orders
- `q1_231_consumer_delivery_fee_refund.sql` — Pre-enrichment: consumer delivery fee refunds
- `enriched_consumer_delivery_fee.sql` — Enriched: full accounting dimension set applied
- `aggregated_cons_del_analytical_view.sql` — Aggregated view: grouped by posting dimensions
- `aggregated_cons_del_analytical_table.sql` — Materialised table (performance)

### B2B — Partner Invoicing (`2b_partner_b2b/`)

Processes commissions and service charges invoiced to restaurant partners. Significantly more complex than the consumer stream — the partner stream encompasses four distinct sub-processes:

| Sub-process | Description |
|---|---|
| Invoiced revenue | Settled invoices — partner commission and fees |
| Commission refunds | Credit notes issued against previously invoiced amounts |
| Invoice accruals | Month-end accrual for orders not yet invoiced |
| TMS accruals | Weekly-reported revenue from the Transport Management System |

**Key models:**
- `q1_commission_pre_invoicing.sql` — Pre-enrichment: invoiced commission by restaurant
- `q2_232_commission_refund_details.sql` — Pre-enrichment: refund amounts by invoice
- `q2_tr_accrual_pre_invoicing.sql` — Pre-enrichment: TMS weekly revenue accruals
- `q3A_tms_revenue_weekly.sql` — Pre-enrichment: TMS weekly aggregation
- `enriched_q1_invoice_accruals.sql` — Enriched: accrual amounts with full GL dimensions
- `aggregated_qXX_partner_invoicing_view.sql` — Aggregated view (union of all sub-processes)
- `aggregated_qXX_partner_invoicing_table.sql` — Materialised table (performance)

---

## Layer 1 — Staging

Staging models apply minimal transformations to raw source data. Each model selects only the fields relevant to the platform accounting team, applies type casting, and filters to the relevant accounting periods.

```sql
-- stg_orders.sql (simplified)
WITH dates AS ( SELECT * FROM {{ ref('accounting_dates_view') }} )

SELECT
    orders.datum,
    orders.restaurantrest_id,
    orders.ordertype,
    orders.totaalexbzg,      -- order total excl. VAT
    orders.bezorgkosten,     -- delivery fee
    orders.order_id,
    orders.betaalmethode     -- payment method
FROM source_schema.order AS orders
WHERE DATE(orders.datum) IN (SELECT datum FROM dates)
```

**Staging sub-groups:**

| Sub-group | Content |
|---|---|
| `1_order_level/` | Order details, payments, order fees, top-rank bids, VAT breakdown |
| `2_invoice_level/` | Accounting entries, invoices, order invoicing costs |
| `3_restaurant_master_data/` | Restaurant details, country settings, restaurant settings |
| `4_other_master_data/` | Accounting dates, couriers, TMS countries, accounting master data views |

---

## Layer 2–3 — Intermediate and Enrichment

Intermediate models join staging entities to produce the inputs required for each revenue stream. Enrichment models apply accounting master data — GL accounts, cost centres, business units, and revenue categories — to produce accounting-ready records.

The enrichment step is where platform data is mapped to the financial chart of accounts. For example, a restaurant commission in Germany maps to a specific GL account, cost centre, and business unit depending on the revenue type, payment service provider, and delivery arrangement.

**Key accounting dimensions applied at enrichment:**

| Dimension | Source |
|---|---|
| `ledger_account` | `acc_md_wd_details.csv` |
| `cost_center_id` | `acc_md_tms_cc_to_bu.csv` |
| `business_unit` | `acc_md_business_unit.csv` |
| `del_location_id` | `acc_md_delivery_locations.csv` |
| `seg_id` | Revenue category mapping |

---

## Layer 4 — Aggregation

Aggregated models group enriched data to the minimum set of dimensions required for posting and downstream processes. The partner invoicing aggregated view unions all four sub-processes (invoiced, refunds, invoice accruals, TMS accruals) into a single model:

```sql
-- aggregated_qXX_partner_invoicing_view.sql (structure)
WITH
    invoicing    AS ( ... FROM aggregated_q3A_tms_revenue_weekly_view  ),
    refunds      AS ( ... FROM aggregated_q2_232_com_refund             ),
    inv_accruals AS ( ... FROM aggregated_q1_invoice_accruals           ),
    tr_accruals  AS ( ... FROM aggregated_q2_tr_accrual_pre_invoicing   ),
    cons_invoicing AS (
        SELECT * FROM invoicing
        UNION ALL SELECT * FROM refunds
        UNION ALL SELECT * FROM inv_accruals
        UNION ALL SELECT * FROM tr_accruals
    )
SELECT
    acc_period, country_name, journal_id, company_id,
    ledger_account, currency, cost_center_id, business_unit,
    del_location_id, memo, accrued, refund_flag,
    debit_amount, credit_amount
FROM cons_invoicing
LEFT JOIN locations ...
LEFT JOIN company ...
WHERE (debit_amount - credit_amount) != 0
```

**Materialisation strategy:**

By default, dbt materialises models as SQL views (computed at query time). For the deeply nested partner invoicing model, this resulted in unacceptable query latency:

| Materialisation | Execution Time |
|---|---|
| View (dynamic) | ~8 minutes |
| Table (physical) | ~1 second |

The solution is to materialise the aggregation layer as a BigQuery table and refresh it via the App Scripts monitoring workflow (after all data quality checks pass):

```sql
-- aggregated_qXX_partner_invoicing_table.sql
{{ config( materialized = 'table' ) }}

WITH invoicing AS (
  SELECT * FROM {{ ref('aggregated_qXX_partner_invoicing_view') }}
)
SELECT * FROM invoicing
```

---

## Layer 5 — Journals and Presentation

### Journal Entry Models (`3_journals/`)

The journal layer assembles all revenue streams into a single balanced double-entry journal structure.

```
aggregated_cons_del_analytical_table  ─┐
aggregated_qXX_partner_invoicing_table ─┼─► JE_revenue_journal_view ──► EIB template
(other revenue streams)               ─┘
```

`JE_revenue_journal_view` is built in three steps via intermediate temp models:
1. `temp_je_1_*` — Revenue lines per stream
2. `temp_je_2_credit_lines_view` — Credit-side journal lines
3. `temp_je_3_debit_lines_view` — Debit-side journal lines (anchor lines)

The final view unions debits and credits, producing a complete balanced journal:

```sql
-- JE_revenue_journal_view.sql
SELECT * FROM temp_je_3_debit_lines_view

UNION ALL

SELECT
    acc_period, country_name, journal_id, company_id,
    ledger_account, account_set, currency, memo,
    cost_center_id, seg_id, business_unit, del_location_id,
    accrued, refund_flag, process_map_id,
    CASE
        WHEN SUM(debit_amount) - SUM(credit_amount) > 0
        THEN ROUND(ABS(SUM(debit_amount) - SUM(credit_amount)), 2)
        ELSE 0
    END AS debit_amount,
    CASE
        WHEN SUM(debit_amount) - SUM(credit_amount) < 0
        THEN ROUND(ABS(SUM(debit_amount) - SUM(credit_amount)), 2)
        ELSE 0
    END AS credit_amount
FROM temp_je_2_credit_lines_view
GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20
```

### EIB Template Models (`3_journals/2_eib_template/`)

EIB (Enterprise Integration Builder) models reformat the journal data to match the exact column structure required by Workday's integration import. Two EIB models are produced:

- `eib_import_accounting_journal.sql` — Header/summary sheet
- `eib_Journal_entry_line_replacement.sql` — Line detail sheet (one row per journal line)

---

## Seeds — Master Data

The dbt project's `seeds/` directory contains the accounting master data maintained by the team as version-controlled CSV files. Changes to chart of accounts mappings, cost centre structures, or business unit definitions are made by updating these files and re-running dbt.

### Master Data Seeds (`seeds/master_data/`)

| File | Description |
|---|---|
| `acc_md_company.csv` | Country code → legal entity mapping |
| `acc_md_business_unit.csv` | Business unit codes and descriptions |
| `acc_md_wd_details.csv` | Revenue category → GL account mapping |
| `acc_md_tms_cc_to_bu.csv` | TMS cost centre → business unit cross-reference |
| `acc_md_tms_spc_and_gl_to_cost_center.csv` | SPC/GL → cost centre mapping |
| `acc_md_tms_wd_process_mapping.csv` | TMS process → Workday process mapping |
| `acc_md_delivery_locations.csv` | City → Workday delivery location ID |
| `acc_md_courier_business_unit.csv` | Courier type → business unit |
| `acc_md_process_id.csv` | Process ID → workflow mapping |
| `acc_md_psp_tags.csv` | Payment service provider classification tags |
| `acc_md_de_admin_psp_split.csv` | Germany admin PSP distribution weights |
| `stg_delivery_locations.csv` | Delivery location reference data |

### Workday Mapping Seeds (`seeds/workday/`)

| File | Description |
|---|---|
| `rev_spc_pl_hierarchy.csv` | Revenue SPC P&L hierarchy |
| `wd_account_class_map.csv` | Workday account class mapping |
| `wd_tms_system_codes_map.csv` | TMS system code → Workday reference |

---

## Data Quality — X_test_monitoring

Assertion models in `X_test_monitoring/` are SQL `SELECT` statements that return rows only when data quality rules are violated. Any non-empty result represents a failure. These models are materialised as BigQuery views and queried daily by the App Scripts monitoring workflow.

### Journal Entry Assertions

| Assertion | Rule |
|---|---|
| `x_assert_journal_balanced` | Debits must equal credits for every journal |
| `x_assert_dr_cr_positive` | All debit and credit amounts must be ≥ 0 |
| `x_assert_null_ledger_account` | No journal line may have a null GL account |
| `x_assert_null_cost_centre` | No journal line may have a null cost centre |
| `x_assert_null_business_unit` | No journal line may have a null business unit |

```sql
-- x_assert_journal_balanced.sql
WITH journal AS ( SELECT * FROM {{ ref('JE_revenue_journal_view') }} )

SELECT
    acc_period, country_name, journal_id, company_id, process_map_id,
    SUM(debit_amount)  AS debit_amount,
    SUM(credit_amount) AS credit_amount
FROM journal
GROUP BY 1,2,3,4,5
HAVING (ROUND(debit_amount,2) - ROUND(credit_amount,2)) != 0
-- Any row returned = unbalanced journal = FAIL
```

### Consumer Aggregation Assertions (9 models)

Null-check assertions on key dimensions in `aggregated_cons_del_analytical_view`:
- `x_assert_null_company_id`
- `x_assert_null_cost_centre`
- `x_assert_null_currency`
- `x_assert_null_ledger_account`
- `x_assert_null_business_unit`
- `x_assert_null_period`
- `x_assert_null_journal_id`
- `x_assert_null_del_location`
- `x_assert_null_memo`

### Partner Invoicing Assertions (9 models)

Equivalent null-check assertions on `aggregated_qXX_partner_invoicing_view` — same dimensions as consumer stream.

---

## dbt Configuration

Models are configured via `dbt_project.yml` and inline `{{ config() }}` blocks. The default materialisation is `view`. Exceptions that require `table` materialisation are configured explicitly at the model level.

dbt Jinja templating is used throughout for:
- `{{ ref('model_name') }}` — cross-model references (builds the DAG)
- `{{ config(...) }}` — materialisation and metadata configuration
- `{{ source('schema', 'table') }}` — raw source references

---

[Back to Overview](..) · [Data Model](data-model) · [Workflows](workflows) · [CLI Application](cli) · [BI Reporting](bi) · [Extended Report](report)
