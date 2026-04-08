---
layout: default
title: "Automated Workflows"
permalink: /workflows
---

# Automated Workflows — Google App Scripts

Two Google App Scripts workflows keep the platform accounting system running reliably without manual intervention. Both run on scheduled triggers and communicate via email on completion or failure.

**Technology:** Google App Scripts (JavaScript) · Google BigQuery API · Workday REST API · Gmail

---

## Architecture Overview

```
                   ┌──────────────────────┐
                   │   Google App Scripts  │
                   │   (Scheduled Triggers)│
                   └──────┬───────────────┘
                          │
          ┌───────────────┼───────────────┐
          │                               │
┌─────────▼──────────┐       ┌────────────▼────────────┐
│  Accounting Data   │       │  Continuous Monitoring   │
│  Extraction        │       │  Workflow                │
│                    │       │                          │
│  Workday REST API  │       │  1. Run assertions       │
│        │           │       │  2a. Pass → Materialise  │
│        ▼           │       │  2b. Fail → Alert email  │
│  Google BigQuery   │       │                          │
│  (journal tables)  │       │  Google BigQuery         │
└────────────────────┘       └──────────────────────────┘
```

---

## Workflow 1 — Accounting Data Extraction

### Purpose

Extracts posted accounting journal lines from the Workday REST API and loads them into Google BigQuery. This eliminates manual data exports from the accounting system and ensures that the BigQuery dataset always has up-to-date posted data for reconciliation and reporting.

### Schedule

| Period | Trigger Frequency | Target Latency |
|---|---|---|
| First 5 working days of month | Every 30 minutes | ≤ 30 minutes |
| All other periods | Daily | ≤ 24 hours |

The tighter schedule during the first five working days reflects the month-end close cycle — this is when the team is actively reconciling and posting, and requires the most current data.

### Data Flow

For each company entity, the workflow:

1. **Computes the extraction window** using `getAccountingDates()` — returns `{current_month: {start_date, end_date}, prior_month: {start_date, end_date}}`
2. **Authenticates** against the Workday REST API using Basic auth (credentials stored in Google Apps Script Properties)
3. **Fetches journal line data** for the date range and company from the Workday reporting endpoint
4. **Applies relevance filters** — retains only rows matching the in-scope companies, GL account prefixes, posting status (`Posted`), and ledger (`Actuals`)
5. **Creates a timestamped BigQuery table** per company: `workday_journal_lines_YYYY_MM_<company_id>`
6. **Batch-inserts** the journal lines in batches of 1,000 rows

```javascript
// settings.js — Relevance filter
function rowRelevance(company, account, status, ledger) {
  const inScopeAccounts = ['4','2015','1210','2010','1240','5400','1050','1211','2450','1132','1220','1230','1290','2061','2062','2210','2211'];
  const compRel   = companies.includes(company);
  const accRel    = inScopeAccounts.some(prefix => account.startsWith(prefix));
  const statusRel = status.toLowerCase() === 'posted';
  const ledgerRel = ledger.toLowerCase() === 'actuals';
  return compRel && accRel && statusRel && ledgerRel;
}
```

### BigQuery Schema

Each extracted table has the following key columns (among ~50 total):

| Column | Type | Description |
|---|---|---|
| `journal_sequence_number` | STRING | Unique Workday journal identifier |
| `accounting_date` | DATE | Journal posting date |
| `ledger_account` | STRING | GL account number |
| `company` | STRING | Legal entity name |
| `status` | STRING | Posting status (filtered to `Posted`) |
| `ledger` | STRING | Ledger type (filtered to `Actuals`) |
| `transaction_debit_amount` | FLOAT64 | Transaction currency debit |
| `transaction_credit_amount` | FLOAT64 | Transaction currency credit |
| `ledger_debit_amount` | FLOAT64 | Ledger currency debit |
| `ledger_credit_amount` | FLOAT64 | Ledger currency credit |
| `cost_center` | STRING | Cost centre worktag |
| `business_unit` | STRING | Business unit worktag |
| `region` | STRING | Geographic region |
| `system_reference_code` | STRING | TMS integration reference code |
| `revenue_category` | STRING | Workday revenue category worktag |
| `run_id` | INT64 | Process run identifier |

### Batch Processing

The workflow batches inserts to stay within Google Apps Script's execution limits. The schema is fully specified at creation time, allowing BigQuery to type-check every insert:

```javascript
// settings.js — batch insert logic
function getValuesText(values) {
  let records = [];
  let batch = "";
  let batchCount = 0;

  for (const row of values) {
    if (!rowRelevance(row.Company, row.Ledger_Account, row.Status, row.Ledger)) {
      continue; // skip irrelevant rows
    }
    const rowString = buildRowString(row); // format values by column type
    batch = batch ? `${batch},\n${rowString}` : rowString;
    batchCount++;

    if (batchCount >= 1000) {
      records.push(batch); // flush batch
      batch = ""; batchCount = 0;
    }
  }
  records.push(batch); // flush final batch
  return records;
}
```

### Error Handling

Each SQL execution is wrapped in try/catch. If a job fails, the workflow logs the error and returns a failure flag. After processing all companies, the workflow checks whether all jobs passed:

```javascript
function run(start, end) {
  const logs = [];
  for (const companyId of companyIds) {
    const createResult = createTable();
    const insertResult = insertValues(start, end, companyId);
    logs.push({company: companyId, result: createResult});
    logs.push({company: companyId, result: insertResult});
  }
  const allPassed = logs.every(log => log.result.pass);
  if (!allPassed) {
    WorkflowSettings.sendLogAsMail(JSON.stringify(logs), 'Workday Update Failed');
  }
  return {pass: allPassed, info: logs};
}
```

---

## Workflow 2 — Continuous Monitoring

### Purpose

Runs data quality assertions against the BigQuery dataset daily and controls whether aggregated tables are materialised. This acts as a gate — only tested, verified data is ever exposed to downstream consumers (the CLI application and Looker Studio dashboards).

### Daily Execution Flow

```
┌─────────────────────────────────────────────────┐
│              Scheduled Daily Trigger             │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
        ┌────────────────────────┐
        │  Materialise test      │
        │  source tables         │
        │  (cons + partner aggr) │
        └────────────┬───────────┘
                     │
                     ▼
        ┌────────────────────────┐
        │  Run all assertions    │
        │  (query each assertion │
        │   view for row count)  │
        └────────────┬───────────┘
                     │
           ┌─────────┴──────────┐
     All pass?                Fails?
           │                    │
           ▼                    ▼
  ┌────────────────┐  ┌────────────────────┐
  │  Materialise   │  │  Abort. Send       │
  │  production    │  │  failure report    │
  │  tables        │  │  by email          │
  │                │  │                   │
  │  Send success  │  └────────────────────┘
  │  notification  │
  └────────────────┘
```

### Assertion Evaluation

Each assertion is a BigQuery view that returns rows only when a data quality rule is violated. The monitoring workflow queries each assertion view and counts the returned rows:

```javascript
// run_tests.js
function getTestResults(testCaseName) {
  const request = {
    query: `SELECT COUNT(*) AS number_fails, '${testCaseName}' AS test_name
            FROM ${executionDatabase}.${testCaseName}`,
    useLegacySql: false
  };

  const queryResults = BigQuery.Jobs.query(request, executionProjectId);
  // ... (poll for job completion) ...

  let totalErrors = 0;
  rows.forEach(row => { totalErrors += parseInt(row.f[0].v); });

  return totalErrors === 0
    ? { value: 0, log: `${testCaseName}: PASS` }
    : { value: totalErrors, log: `${testCaseName}: FAIL — ${totalErrors} rows with errors` };
}
```

The assertions evaluated daily cover three areas:

| Assertion Group | Count | Checks |
|---|---|---|
| Journal Entry (`JE_revenue_journal`) | 5+ | Balance, non-negative amounts, null GL/CC/BU |
| Consumer Aggregation | 9 | Null dimensions across all accounting fields |
| Partner Invoicing Aggregation | 9 | Null dimensions across all accounting fields |

### Materialisation

If all assertions pass, the workflow materialises the production aggregated tables using DDL executed via the BigQuery Jobs API:

```javascript
// materialise_views.js
const views = new Map([
  ['aggregated_cons_del_analytical_end_user_bi',    'aggregated_cons_del_analytical_table'],
  ['aggregated_qXX_partner_invoicing_end_user_bi',  'aggregated_qXX_partner_invoicing_table'],
  ['JE_revenue_journal_table',                      'JE_revenue_journal_view'],
]);

function runQuery() {
  const testResults = checkTests(); // run assertions first
  if (testResults.results) {
    for (const [targetTable, sourceView] of views) {
      const sql = WorkflowSettings.generateSqlString(targetTable, sourceView);
      WorkflowSettings.createMaterialiseJob(targetTable, sql);
    }
  } else {
    // Tests failed — do not materialise, send alert
    WorkflowSettings.sendLogAsMail(testResults.log, 'Materialisation Aborted');
  }
}
```

The generated DDL follows this pattern:

```sql
-- Generated by WorkflowSettings.generateSqlString()
CREATE OR REPLACE TABLE project.dataset.aggregated_qXX_partner_invoicing_table
OPTIONS() AS (
  WITH tb AS (
    SELECT * FROM project.dataset.aggregated_qXX_partner_invoicing_view
  )
  SELECT * FROM tb
);
```

### Email Notifications

Both outcomes trigger email notifications sent via Gmail's Apps Script API:

- **Pass:** Subject `Table Materialisation Logs: YYYY-MM-DD` — includes per-table status
- **Fail:** Subject `Test results: YYYY-MM-DD` — includes per-assertion pass/fail detail

---

## Configuration — `WorkflowSettings`

Both workflows share a common settings module (`WorkflowSettings`) that centralises configuration:

| Setting | Description |
|---|---|
| `getExecutionProjectId()` | GCP project ID used for billing |
| `getExecutionDatabase()` | BigQuery dataset (test vs production toggle) |
| `getTestCases()` | List of assertion model names to evaluate |
| `getDate()` | Current date string for log subjects |
| `generateSqlString()` | Builds the `CREATE OR REPLACE TABLE ... AS (...)` DDL |
| `waitedMaterialiseTable()` | Executes a materialisation job synchronously with polling |
| `createMaterialiseJob()` | Executes a materialisation job asynchronously |
| `sendLogAsMail()` | Sends a log string as an HTML email |

### Test vs Production Toggle

The execution database is configurable — switching `inProduction = false` routes all queries to a test dataset, enabling safe testing of configuration changes before promoting to production:

```javascript
// settings.js
const inProduction = true;
let executionDatabase = 'test_dataset';
if (inProduction) {
  executionDatabase = 'production_dataset';
}
```

---

## Deployment

Both workflows are deployed as Google Apps Script projects bound to a Google Cloud project. Scheduled triggers are set via Apps Script's built-in trigger management:

- **Accounting Data Extraction:** Time-based trigger, frequency set per the SLA schedule above
- **Continuous Monitoring:** Daily trigger, executes overnight outside business hours

Authentication to BigQuery uses the Apps Script project's service account credentials. Authentication to Workday uses Basic auth credentials stored securely in Apps Script Properties (not hardcoded).

---

[Back to Overview]({{ site.baseurl }}/) · [Data Model](data-model) · [Workflows](workflows) · [CLI Application](cli) · [BI Reporting](bi) · [Extended Report](report) 
