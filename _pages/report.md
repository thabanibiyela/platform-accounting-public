---
layout: default
title: "Project Report"
permalink: /report
---

# Platform Accounting Automation — Project Report

*MSc Computer Science Capstone · University of London · 2025*

---

## Executive Summary

This project delivered a cloud-based automation platform for the revenue accounting function of a large-scale online food delivery marketplace. Operating across 17 European countries and processing 140+ million orders per year, the business generates financial data at a volume and granularity that its existing manual workflows could no longer reliably handle.

The solution replaces a monolithic, spreadsheet-driven operation with a modular system built on Google BigQuery, dbt, Python, and Google App Scripts. Journal entry turnaround was reduced from approximately 60 minutes to under 5 minutes per account. Account reconciliation time fell from approximately 180 minutes to under 5 minutes. The estimated annual labour saving exceeds 350 hours.

The project was completed as the capstone for an MSc in Computer Science at the University of London, applying enterprise architecture (TOGAF), object-oriented design, agile development practices, and cloud data engineering to a real production environment.

---

## 1. Business Context

### 1.1 The Organisation

The platform accounting team sits within the finance function of a leading European online food delivery company. The team is responsible for the revenue accounting of the company's platform business — the technology infrastructure that facilitates transactions between restaurants (partners) and consumers across all markets.

Platform revenue falls into two primary streams:

- **B2B Partner Revenue:** Commissions, service fees, and other charges invoiced to restaurant partners and processed through the company's invoicing system
- **B2C Consumer Revenue:** Delivery fees and other charges collected directly from consumers at point of order

Both streams generate financial data that must be extracted from the order management system, enriched with accounting dimensions, aggregated to the journal entry level, reconciled to posted ledger balances, and reported to internal and external stakeholders.

### 1.2 Scale and Complexity

With 140+ million orders processed annually across 17 countries, the platform generates financial microtransactions at massive scale. Each order may produce multiple revenue events — commission charges, delivery fee charges, accruals, refunds — each requiring classification by GL account, cost centre, business unit, revenue category, and delivery location. A single month-end closing period may involve millions of individual transaction lines.

### 1.3 The Existing System

Prior to this project, the team's workflow relied on:

- A **bespoke ETL script** that extracted data from the order management system and loaded it into a staging database
- A **customised accounting integration** that produced EIB (Enterprise Integration Builder) files for upload to the accounting system (Workday)
- **Extensive manual spreadsheet work** to perform reconciliations, produce analytics, and respond to stakeholder data requests

This approach had several material weaknesses:

| Issue | Impact |
|---|---|
| Manual reconciliation steps | High error risk; ~180 minutes per account per month |
| No automated data quality checks | Errors discovered downstream during review |
| Spreadsheet-based analytics | Ad hoc, inconsistent, non-reproducible |
| Monolithic ETL architecture | Difficult to maintain; no separation of concerns |
| No stakeholder self-service | Team fielded all data requests manually |

---

## 2. Problem Statement and Objectives

### 2.1 Gap Analysis

An internal gap analysis was performed using the TOGAF enterprise architecture framework to map the team's strategic objectives to its current IT architecture and identify where gaps existed. The analysis identified four primary gaps:

1. **Data Infrastructure:** No centralised, cloud-based data store accessible to multiple downstream processes
2. **Data Quality:** No systematic, automated data quality checks prior to financial posting
3. **Process Automation:** No programmatic execution of the end-to-end accounting workflow
4. **Stakeholder Reporting:** No self-service reporting capability for internal stakeholders

### 2.2 Functional Requirements

The following functional requirements were defined:

- **FR1:** The system must extract and store platform transaction data in BigQuery
- **FR2:** The system must apply accounting master data to enrich transaction data with GL accounts, cost centres, and business units
- **FR3:** The system must produce EIB journal entry files formatted for direct ingestion by Workday
- **FR4:** The system must produce audit-ready review workpapers for journal entry sign-off
- **FR5:** The system must reconcile platform data to posted Workday balances
- **FR6:** The system must run automated data quality checks prior to any downstream output
- **FR7:** The system must provide interactive stakeholder reporting dashboards
- **FR8:** The system must execute end-to-end within 5 minutes per process

### 2.3 Non-Functional Requirements

- **Latency:** Journal and reconciliation processes must complete within 5 minutes
- **Reliability:** Automated workflows must self-monitor and alert on failure
- **Auditability:** All outputs must be traceable to source data
- **Maintainability:** The codebase must be modular and unit-testable
- **Extensibility:** New GL accounts and revenue streams must be addable without architectural change

---

## 3. Architecture and Design

### 3.1 Solution Overview

The solution was delivered across three work packages:

| Work Package | Scope | Technology |
|---|---|---|
| **Project A** | Master Data Harmonisation | Google BigQuery, dbt |
| **Project B** | Stakeholder Reporting | Google Looker Studio |
| **Project C** | Automation Platform | Python, JupyterLab, SQLite |

A fourth cross-cutting component — **automated workflows** — connects the work packages using Google App Scripts running on scheduled triggers.

### 3.2 Architectural Approach: Modular ETL

The existing monolithic ETL architecture was replaced with a modular design, separating concerns across distinct layers and components. This approach aligns with the principle that each component should have a single, well-defined responsibility — making the system easier to test, maintain, and extend.

```
                    ┌─────────────────────────────────────┐
                    │         Google BigQuery              │
  Workday API ─────►│  Raw Journal Lines                  │
                    │  (workday_journal_lines_YYYY_MM_*)   │
                    └──────────────┬──────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │         dbt Data Model               │
  Platform Data ───►│  Staging → Intermediate →           │
                    │  Enrichment → Aggregation            │
                    └──────────────┬──────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
  ┌───────────▼──────┐  ┌─────────▼──────┐  ┌─────────▼──────┐
  │  gforevpy (CLI)  │  │ Looker Studio  │  │ App Scripts    │
  │  Journal Entry   │  │  Dashboards    │  │  Monitoring    │
  │  Reconciliation  │  │                │  │  Extraction    │
  └──────────────────┘  └────────────────┘  └────────────────┘
```

### 3.3 Development Methodology

The project was developed using an adapted Scrum methodology suited to a solo developer. Two-week sprints were organised around the three work packages, with each sprint producing a working, deployable increment. Given the real production context, a feature-flag approach was used to allow gradual migration from the legacy system.

---

## 4. Project A — Master Data Harmonisation

### 4.1 Data Model Overview

The dbt data model transforms raw platform transaction data into structured, auditable financial data through six conceptual layers:

```
Layer 0: Raw event-level data (order management system → BigQuery)
Layer 1: Staging — field selection, renaming, type casting
Layer 2: Intermediate — stream-specific integration and pre-enrichment
Layer 3: Enrichment — accounting master data application
Layer 4: Aggregation — grouping to posting dimensions
Layer 5: Presentation — consumption-ready views for CLI and Looker Studio
```

### 4.2 Revenue Streams

The data model maintains two parallel processing tracks reflecting the two revenue pillars:

**B2C Consumer Revenue (`2a_consumer_b2c/`):**
Processes delivery fees charged to consumers at point of order. Staged from order-level transaction data and enriched with consumer-specific GL accounts and cost centres.

**B2B Partner Revenue (`2b_partner_b2b/`):**
Processes commissions, service fees, and accruals relating to restaurant partners. More complex than the consumer stream — includes four distinct sub-processes:
- Invoiced partner revenue (billed and settled invoices)
- Commission refunds (credit notes)
- Invoice accruals (month-end accrued revenue for uninvoiced orders)
- TMS revenue accruals (weekly-reported revenue pending invoicing)

### 4.3 Journal Entry Models (`3_journals/`)

The journal entry layer is where accounting-ready data is produced. A view (`JE_revenue_journal_view`) unions all revenue streams — invoiced, credited, accrued consumer, accrued partner — into a single balanced journal structure. Each line carries full accounting dimensions:

| Dimension | Description |
|---|---|
| `acc_period` | Accounting period (YYYY-MM) |
| `ledger_account` | GL account number |
| `company_id` | Legal entity identifier |
| `cost_center_id` | Cost centre |
| `business_unit` | Business unit worktag |
| `del_location_id` | Delivery location (Workday dimension) |
| `debit_amount` / `credit_amount` | Double-entry amounts |
| `memo` | Human-readable posting description |

An assertion model (`x_assert_journal_balanced.sql`) checks that debits equal credits at every combination of period, company, journal, and process — and surfaces any imbalance as a row:

```sql
-- x_assert_journal_balanced.sql
SELECT acc_period, country_name, journal_id, company_id, process_map_id,
       SUM(debit_amount) AS debit_amount,
       SUM(credit_amount) AS credit_amount
FROM JE_revenue_journal_view
GROUP BY 1,2,3,4,5
HAVING (ROUND(debit_amount,2) - ROUND(credit_amount,2)) != 0
```

Any row returned by this assertion represents a balanced journal violation — caught before the EIB file is exported.

### 4.4 Master Data

The model is governed by a set of seed files (CSV) that encode the accounting master data mappings maintained by the team:

| Seed | Purpose |
|---|---|
| `acc_md_company.csv` | Country code to company entity mapping |
| `acc_md_business_unit.csv` | Business unit worktag definitions |
| `acc_md_wd_details.csv` | Revenue category and GL account mapping |
| `acc_md_tms_cc_to_bu.csv` | TMS cost centre to business unit mapping |
| `acc_md_delivery_locations.csv` | City to Workday delivery location mapping |
| `acc_md_process_id.csv` | Process identifier to workflow mapping |

These seeds are version-controlled alongside the dbt project, ensuring that accounting master data changes are auditable and reproducible.

### 4.5 Performance Optimisation

dbt materialises models as SQL views by default — efficient for freshness but slow for deeply nested models. To meet the 5-minute latency requirement, the aggregation layer is materialised as a physical BigQuery table:

```sql
-- aggregated_qXX_partner_invoicing_table.sql
{{ config( materialized = 'table' ) }}

WITH invoicing AS (
  SELECT * FROM {{ ref('aggregated_qXX_partner_invoicing_view') }}
)
SELECT * FROM invoicing
```

The performance gain from this single change was dramatic:

| Query | Execution Time |
|---|---|
| View (dynamic) | ~8 minutes |
| Table (materialised) | ~1 second |

### 4.6 Data Quality Testing

Data quality is enforced at two levels:

**Development-time (dbt tests):** Standard dbt tests (not_null, unique, relationships) are defined in `schema.yml` files and run during development. These catch structural issues before models are promoted.

**Production monitoring (assertion models):** SQL assertion models are materialised as BigQuery views in the `X_test_monitoring/` directory. Each assertion selects rows that *violate* a data quality rule — so any non-empty result is a failure. Examples:

- `x_assert_journal_balanced` — debits must equal credits
- `x_assert_dr_cr_positive` — all debit and credit amounts must be non-negative
- `x_assert_null_ledger_account` — no journal line may have a null GL account
- `x_assert_null_cost_centre` — no journal line may have a null cost centre
- `x_assert_null_business_unit` — no journal line may have a null business unit

These assertions are evaluated daily by the App Scripts continuous monitoring workflow, which aborts materialisation and alerts the team if any assertion returns rows.

---

## 5. Project B — Stakeholder Reporting

### 5.1 Requirements Gathering

Prior to building the dashboards, requirements were gathered by:
1. Reviewing historical stakeholder data requests (email and messaging records)
2. Conducting walkthrough sessions with the finance business partner and management reporting teams
3. Mapping recurring information needs to data dimensions available in the BigQuery model

The result was a set of five standard reporting dimensions that appear across all dashboards: accounting period, country/market, ledger account, business unit, and revenue category.

### 5.2 BI Data Model

A dedicated presentation layer in dbt (`4_Reporting/bi_reporting/`) produces the views consumed by Looker Studio. The key view — `revenue_anayltics_end_user_bi` — includes:

- Current period revenue, order count, and GMV
- Prior period comparators for each metric
- Pre-computed variances: revenue, order count, GMV
- Revenue-per-order and GMV-per-order metrics
- Price and volume variance decomposition (using standard variance analysis)

```sql
-- Excerpt from revenue_anayltics_end_user_bi.sql
ROUND((
  COALESCE(pp.revenue_per_order,0) *
  (COALESCE(cp.order_count,0) - COALESCE(pp.order_count,0))
),2) AS volume_variance_curr,

ROUND((
  COALESCE(cp.revenue_per_order,0) - COALESCE(pp.revenue_per_order,0)) *
  COALESCE(cp.order_count,0)
,2)  AS price_variance_curr
```

### 5.3 Dashboard Design

Each Looker Studio dashboard provides four standard panels:

1. **Periodic Overview** — Headline account balances for the selected period, with prior period and variance columns
2. **Month-on-Month Trend** — Time-series chart of revenue across the trailing 12 months
3. **Variance Analysis** — Tabular breakdown of variance by market, with price/volume split
4. **Geographic Detail** — Country-level card view with key metrics and period-on-period change

Filters are applied at the dashboard level and cascade across all panels — allowing stakeholders to drill into a specific country, account, or business unit without switching views.

### 5.4 Impact

The dashboards eliminated the team's ad hoc reporting backlog. Stakeholders moved from raising data requests by email (typically answered within 24–48 hours) to accessing live, filtered data on demand at any time. The team's reporting workload shifted from reactive production to proactive analysis.

---

## 6. Project C — Automation Platform

### 6.1 Architecture

The automation platform is a Python package (`gforevpy`) structured around a hierarchy of single-responsibility components:

```
gforevpy/
├── app/
│   ├── app.py                  # CLI entry point (argparse)
│   └── process/
│       ├── runner.py           # Abstract base class Process
│       ├── journals.py         # ProcessMECJournal
│       └── reconciliation.py   # ProcessRecon
├── models/
│   ├── accounting/
│   │   └── accounting.py       # Data model classes (EIBJournalLines, PlatformReconData, etc.)
│   ├── data/
│   │   ├── sql.py              # SQL query builders
│   │   ├── datachecker.py      # Data quality check classes
│   │   └── localdb.py          # SQLite parameter store
│   └── prepares/
│       ├── prepjournal.py      # PreparesJournal — reconciliation logic
│       ├── prepeib.py          # PreparesAccountingJournal — EIB file generation
│       └── preprecon.py        # PreparesRecon — reconciliation workpaper
└── workpapers/
    ├── reviewjournal.ipynb     # Journal review workpaper template
    └── reconciliation.ipynb   # Reconciliation workpaper template
```

### 6.2 OOP Design

The application is built around an abstract base class `Process` that defines the interface all accounting processes must implement. Concrete subclasses provide process-specific implementations:

```python
class Process(ABC):
    """Base class for all CLI accounting processes."""

    @abstractmethod
    def run(self, command_id: int) -> object: ...

    @abstractmethod
    def export_data_files(self, verbose: bool = False) -> None: ...

    @abstractmethod
    def export_review_file(self, verbose: bool = False) -> None: ...

    @abstractmethod
    def get_data_quality_tests(self, verbose: bool = False) -> pd.DataFrame: ...

    @abstractmethod
    def write_parameters_to_db(self, verbose: bool = False) -> tuple[int]: ...

    @property
    @abstractmethod
    def cli_commands_map(self) -> dict: ...
```

This contract ensures that any new process type (e.g. an audit module) can be added by implementing the same interface — without touching the CLI entry point or any existing code.

### 6.3 Journal Entry Workflow

When `gforevpy journal <year> <month> <id>` is invoked, `ProcessMECJournal` orchestrates the following:

1. Connects to BigQuery via the Google Cloud Python client
2. Downloads EIB import data and journal lines for the period
3. Downloads posted Workday data for reconciliation
4. Builds reconciliation summaries (invoiced, credited, consumer, accruals)
5. Runs data quality assertions on the EIB data
6. Exports the EIB file (Excel) formatted for Workday ingestion
7. Executes the JupyterLab review workpaper and exports it as an HTML file

The CLI exposes 13 commands covering each step:

| # | Command | Description |
|---|---|---|
| 1 | Display Invoiced Revenue | Platform vs Workday reconciliation — invoiced partner revenue |
| 2 | Display Credited Revenue | Platform vs Workday reconciliation — credit notes |
| 3 | Display Consumer Revenue | Platform vs Workday reconciliation — consumer delivery fees |
| 4 | Display Accrued Revenue | Current vs prior month accrual summary |
| 5 | Display Total Revenue | Combined revenue summary across all streams |
| 6 | Review Relative Accrual | Plausibility check — accrual as % of total revenue vs historical expectation |
| 7–9 | Review [Stream] Recon | Detailed reconciliation with threshold flags |
| 10 | View Data Quality Tests | Results of EIB data assertions |
| 11 | Write Parameters to DB | Persist run parameters for workpaper execution |
| 12 | Export EIB File | Generate and export the Workday integration file |
| 13 | Export Review File | Execute and export the HTML review workpaper |

### 6.4 Reconciliation Workflow

`ProcessRecon` automates the monthly account reconciliation process. Given a GL account number, it:

1. Downloads platform data for the account from BigQuery
2. Downloads posted Workday journal lines (equivalent to Workday's *Find Journal Lines* report)
3. Downloads a month-on-month analytical summary
4. Downloads the Workday trial balance for the period
5. Runs data quality assertions on the reconciliation data
6. Produces a structured reconciliation workpaper (HTML/Excel)

### 6.5 Data Quality in the CLI

The CLI implements its own layer of data quality checks, independent of the dbt/App Scripts monitoring. Before exporting any file, the application:

- Verifies that debit and credit amounts are non-negative
- Verifies that journal totals balance within rounding tolerance
- Performs Workday vs Platform reconciliation across each revenue stream
- Flags any reconciling item that exceeds a 0.05% threshold

```python
# From PreparesJournal.checks_invoiced()
revenue['Threshold'] = 0.0005
revenue['Within Threshold'] = abs(revenue['Difference / Amount']) < revenue['Threshold']
```

The relative accrual plausibility check is particularly noteworthy — it builds a statistical expectation of the current month accrual as a percentage of total revenue based on historical data, then checks whether the current period falls within a 2% tolerance:

```python
partner_revenue['Rel. Accrual'] = abs(
    partner_revenue['C.Month Accrual'] / partner_revenue['Adj. Part. Revenue']
)
partner_revenue['Difference'] = (
    partner_revenue['Rel. Accrual'] - partner_revenue['Exp. Rel. Accrual']
)
partner_revenue['Within Threshold'] = (
    abs(partner_revenue['Difference']) < 0.02
)
```

### 6.6 Workpapers

JupyterLab notebooks serve as review workpaper templates. When `export_review_file()` is called, the CLI:

1. Writes the run parameters (year, month, process ID) to a local SQLite database
2. Executes the notebook using `jupyter nbconvert --execute`
3. Exports the result as an HTML file
4. Opens it in the default browser for review

The notebooks read their parameters from the SQLite database, ensuring they always run against the same period as the CLI session that invoked them. The exported HTML becomes the audit attachment uploaded to Workday.

### 6.7 Testing

The test suite covers all isolated logic and all BigQuery integrations:

| Type | Scope | Result |
|---|---|---|
| Unit | `accounting.py`, `datachecker.py`, `localdb.py`, `utils.py` | All passed |
| Integration | `prepeib.py`, `prepjournal.py`, `preprecon.py` (BigQuery) | All passed |
| UAT | End-to-end against functional requirements | All passed |

---

## 7. Automated Workflows

### 7.1 Continuous Monitoring

A daily Google App Scripts trigger performs the data integrity gate:

1. Runs each assertion model by querying BigQuery for row counts — any non-zero count is a failure
2. **All pass:** Materialises the aggregated tables from their views; sends a success notification
3. **Any fail:** Aborts materialisation; sends a detailed failure report by email

This ensures that only tested, verified data ever reaches the CLI application or Looker Studio dashboards.

### 7.2 Accounting Data Extraction

A second workflow uses the Workday REST API to extract posted journal lines into BigQuery. It runs per company entity, creating a timestamped table for each extraction window:

```
workday_journal_lines_YYYY_MM_<company_id>
```

The workflow applies relevance filters (company, GL account prefix, posting status, ledger type) to exclude irrelevant lines before insertion, keeping BigQuery storage lean.

**SLA targets:**
- First 5 working days of the month: maximum 30-minute latency (peak month-end)
- All other periods: maximum 24-hour latency

---

## 8. Results

### 8.1 Quantitative Outcomes

| Metric | Before | After | Improvement |
|---|---|---|---|
| Journal entry (per account) | ~60 min | < 5 min | 92% reduction |
| Reconciliation (per account) | ~180 min | < 5 min | 97% reduction |
| Partner invoicing query | ~8 min | < 1 sec | 99.8% reduction |
| Annual labour saving | — | 350+ hours | — |
| Data quality coverage | 0 automated checks | 27 assertion models | — |

### 8.2 Qualitative Outcomes

- **Auditability:** Every output is traceable to its source BigQuery model and seed data, with a full dbt lineage graph
- **Reliability:** Automated monitoring catches data issues before they reach downstream consumers
- **Scalability:** The modular architecture supports adding new GL accounts or revenue streams without restructuring the system
- **Self-service reporting:** Stakeholders access live financial data on demand, reducing the team's reporting burden

### 8.3 Limitations and Future Work

- The audit support module (`ProcessAudit`) was scoped but not implemented in the project timeframe
- The current deployment relies on manual `gforevpy` invocation; full automation would require a scheduler (e.g. Cloud Scheduler + Cloud Run)
- The reconciliation workflow assumes Workday data has been loaded by the extraction workflow prior to use

---

📄 [Back to Overview](/) · [Data Model →](data-model) · [CLI Application →](cli) · [Workflows →](workflows) · [BI Reporting →](bi)
