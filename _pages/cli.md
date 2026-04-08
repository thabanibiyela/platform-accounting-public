---
layout: default
title: "CLI Application"
permalink: /cli
---

# CLI Application — `gforevpy`

`gforevpy` is a Python command line application that automates the production of financial accounting outputs for the platform accounting team. It retrieves data from Google BigQuery, performs calculations, and generates the files required for journal entry posting, account reconciliation, and audit support.

**Package:** `gforevpy` · **Language:** Python 3 · **Key dependencies:** `pandas`, `google-cloud-bigquery`, `openpyxl`, `jupyter`

---

## Getting Started

### Installation

Install the package in a Python virtual environment from the project root:

```bash
pip install .
```

### Authentication

Authorise Google Cloud access using Application Default Credentials:

```bash
gcloud auth application-default login
```

The application uses the BigQuery Python client, which picks up credentials automatically from the environment.

---

## Usage

```bash
gforevpy <process> <year> <month> <id>
```

| Argument | Type | Description |
|---|---|---|
| `process` | `journal` or `reconciliation` | The accounting process to run |
| `year` | Integer (e.g. `2025`) | The accounting year |
| `month` | Integer `1`–`12` | The accounting month |
| `id` | Integer | Process ID (journals) or GL account number (reconciliation) |

Once launched, the application presents an interactive menu. Enter the number of the command to run, or type `QUIT` to exit.

### Examples

```bash
# Journal entry process — Commission Revenue (process ID 1), August 2025
gforevpy journal 2025 8 1

# Journal entry process — Promoted Placement Revenue (process ID 3), August 2025
gforevpy journal 2025 8 3

# Account reconciliation — Commission Revenue GL accounts (4100/4200), August 2025
gforevpy reconciliation 2025 8 4100

# Account reconciliation — Consumer Delivery Fee GL account (4800), August 2025
gforevpy reconciliation 2025 8 4800
```

---

## Architecture

The application is built around a hierarchy of single-responsibility Python modules:

```
src/
├── app/
│   ├── app.py                      # CLI entry point (argparse)
│   └── process/
│       ├── runner.py               # Abstract base class: Process
│       ├── journals.py             # Concrete: ProcessMECJournal
│       └── reconciliation.py       # Concrete: ProcessRecon
├── models/
│   ├── accounting/
│   │   └── accounting.py           # Data model classes
│   ├── data/
│   │   ├── sql.py                  # SQL query builder classes
│   │   ├── datachecker.py          # Data quality check classes
│   │   └── localdb.py              # SQLite parameter store
│   └── prepares/
│       ├── prepjournal.py          # PreparesJournal — reconciliation logic
│       ├── prepeib.py              # PreparesAccountingJournal — EIB file export
│       └── preprecon.py            # PreparesRecon — reconciliation workpaper
├── workpapers/
│   ├── reviewjournal.ipynb         # Journal review workpaper template
│   └── reconciliation.ipynb        # Reconciliation workpaper template
├── config.py                       # Configuration (GL accounts, process IDs)
└── utils.py                        # Utility classes and functions
```

---

## OOP Design — The `Process` Abstract Base Class

The application's architecture is built around an abstract base class `Process` defined in `runner.py`. This class defines the interface that every accounting process must implement, ensuring a uniform contract regardless of the specific process type.

```python
# runner.py (simplified)
from abc import ABC, abstractmethod
import pandas as pd

class Process(ABC):
    """Base (abstract) class for process classes exposed to the CLI."""

    def __init__(self, year: int, month: int, run_id: object):
        self.__year = year
        self.__month = month
        self.__run_id = run_id

    # --- Abstract methods — must be implemented by all subclasses ---

    @abstractmethod
    def run(self, command_id: int) -> object:
        """Execute a CLI command by ID."""
        pass

    @abstractmethod
    def display_run_options(self) -> str:
        """Return a formatted string listing available CLI commands."""
        pass

    @abstractmethod
    def get_data_quality_tests(self, verbose: bool = False) -> pd.DataFrame:
        """Return data quality test results as a DataFrame."""
        pass

    @abstractmethod
    def write_parameters_to_db(self, verbose: bool = False) -> tuple:
        """Persist run parameters to local SQLite for workpaper use."""
        pass

    @abstractmethod
    def export_data_files(self, verbose: bool = False) -> None:
        """Export the primary data output (EIB file or reconciliation data)."""
        pass

    @abstractmethod
    def export_review_file(self, verbose: bool = False) -> None:
        """Execute and export the HTML review workpaper."""
        pass

    @property
    @abstractmethod
    def cli_commands_map(self) -> dict:
        """Dict mapping command IDs to process names and methods."""
        pass
```

**Design rationale:** This contract means the CLI entry point (`app.py`) works identically for any process type — it simply calls `process.run(command_id)` without needing to know whether it's running a journal or reconciliation. Adding a new process type (e.g. an audit module) requires implementing these methods and registering the new class in `app.py`'s `process_list` dict — no other code changes needed.

### CLI Entry Point

```python
# app.py
process_list = {
    "journal":        ProcessMECJournal,
    "reconciliation": ProcessRecon
}

def app() -> None:
    parser = argparse.ArgumentParser(prog="gforevpy Revenue Automation Platform")
    parser.add_argument("process", type=str)
    parser.add_argument("year",    type=int)
    parser.add_argument("month",   type=int)
    parser.add_argument("id",      type=int)
    args = parser.parse_args()

    process = process_list[args.process](args.year, args.month, args.id)

    while True:
        command = input(process.display_run_options())
        if command.upper() == 'QUIT':
            break
        process.run(int(command))
```

---

## Journal Entry Process — `ProcessMECJournal`

`ProcessMECJournal` automates the month-end close journal entry workflow for a given revenue stream (identified by `process_id`).

### Data Loaded at Initialisation

When `ProcessMECJournal(year, month, process_id)` is instantiated, it downloads from BigQuery:

| Attribute | Class | Data |
|---|---|---|
| `__eib_import_creates` | `EIBJournalLines` | EIB import header data (Workday integration format) |
| `__eib_journal_creates` | `EIBJournalLines` | EIB journal line data (double-entry journal lines) |
| `__wd_data_creates` | `WDJournalsData` | Posted Workday journal lines for the period |
| `__wd_accrual_rev_creates` | `WDJournalsData` | Prior month accrual reversals from Workday |
| `__journal_reconciler` | `PreparesJournal` | Reconciliation engine (built from the above) |

### CLI Commands

| # | Command | Description |
|---|---|---|
| 1 | Display Invoiced Revenue | Platform vs Workday reconciliation — invoiced partner revenue by market |
| 2 | Display Credited Revenue | Platform vs Workday reconciliation — credit notes by market |
| 3 | Display Consumer Revenue | Platform vs Workday reconciliation — consumer delivery fees by market |
| 4 | Display Accrued Revenue | Current vs prior month accrual summary by market |
| 5 | Display Total Revenue | Combined revenue across all streams (invoiced + credited + consumer + accruals) |
| 6 | Review Relative Accrual | Plausibility check — current accrual as % of revenue vs historical expectation |
| 7 | Review Invoiced Revenue Recon | Detailed invoiced reconciliation with threshold flags (0.05%) |
| 8 | Review Credited Revenue Recon | Detailed credited reconciliation with threshold flags (0.05%) |
| 9 | Review Consumer Revenue Recon | Detailed consumer reconciliation with threshold flags (0.05%) |
| 10 | View Data Quality Tests | EIB data quality assertion results |
| 11 | Write Parameters to DB | Persist year/month/process_id to SQLite for workpaper execution |
| 12 | Export EIB File | Generate and export the Workday integration file (Excel) |
| 13 | Export Review File | Execute JupyterLab workpaper and export as HTML |

### Reconciliation Logic

`PreparesJournal` performs the platform-to-Workday reconciliation. It separates revenue into streams by reading line memo descriptions and builds reconciliation DataFrames for each:

```python
# prepjournal.py — determining revenue flow from memo
def __determine_flow(self, line_memo: str) -> str:
    memo = str(line_memo).upper()
    if   'INVOICED' in memo: return 'INVOICED'
    elif 'CREDITED' in memo: return 'CREDITED'
    elif 'ACCRUED'  in memo: return 'ACCRUED'
    else:                    return 'CONSUMER'
```

Each reconciliation merges the Platform amount against the Workday amount by country and currency, computes the difference, and checks it against threshold:

```python
# prepjournal.py — invoiced revenue reconciliation
def summarise_invoiced(self) -> pd.DataFrame:
    results = pd.merge(
        self.__recons['wd_partner_invoiced'],
        self.__recons['platform_partner_invoiced'],
        on=['Country', 'Currency'],
        how='outer'
    )
    results['Difference Curr.']    = results['WD Amount'] - results['Platform Amount']
    results['Difference / Amount'] = results['Difference Curr.'] / results['WD Amount']
    return results

def checks_invoiced(self) -> pd.DataFrame:
    revenue = self.summarise_invoiced()
    revenue['Threshold']       = 0.0005  # 0.05%
    revenue['Within Threshold'] = abs(revenue['Difference / Amount']) < revenue['Threshold']
    return revenue
```

### Relative Accrual Plausibility

Command 6 performs a statistical plausibility check on the current period accrual:

1. Calculates current month accrual as a % of adjusted partner revenue
2. Compares against a historical expected ratio (`Exp. Rel. Accrual`) derived from prior periods
3. Flags any market where the difference exceeds 2%

```python
# prepjournal.py
partner_revenue['Rel. Accrual']     = abs(accrual / adjusted_revenue)
partner_revenue['Difference']       = rel_accrual - expected_rel_accrual
partner_revenue['Within Threshold'] = abs(partner_revenue['Difference']) < 0.02
```

---

## Reconciliation Process — `ProcessRecon`

`ProcessRecon` automates the monthly balance sheet account reconciliation for a given GL account (e.g. 4100 for Commission Revenue).

### Data Loaded at Initialisation

| Attribute | Class | Data |
|---|---|---|
| `__creates_platform_data` | `PlatformReconData` | Platform transaction data for the account |
| `__creates_workday_data` | `WDJournalsReconData` | Posted Workday journal lines (*Find Journal Lines* equivalent) |
| `__creates_mom_summary` | `WDJournalsReconData` | Month-on-month Workday analytical summary |
| `__creates_trial_balance` | `WDJournalsReconData` | Workday trial balance for the period |
| `__recon` | `PreparesRecon` | Reconciliation workpaper logic |

### CLI Commands

| # | Command | Description |
|---|---|---|
| 1 | Display Platform Data | Platform transaction data for the reconciled account |
| 2 | Display Workday Journal Lines | Posted Workday lines (equivalent to Find Journal Lines) |
| 3 | Display MoM Summary | Month-on-month comparative summary |
| 4 | Display Trial Balance | Workday trial balance |
| 5 | View Data Quality Tests | Reconciliation data quality assertion results |
| 6 | Write Parameters to DB | Persist parameters for workpaper execution |
| 7 | Export Reconciliation Data | Export platform and Workday data to Excel |
| 8 | Export Review File | Execute and export the reconciliation workpaper as HTML |

---

## Data Layer — SQL Query Builders

SQL queries are constructed programmatically using class-based builders in `sql.py`. Each SQL class encapsulates the query for a specific data need, accepting the run parameters as constructor arguments:

```python
# sql.py (pseudocode pattern)
class EIBImportSQL(AccountingSQLQuery):
    def __init__(self, year: int, month: int, run_id: int, test_flag: bool):
        self._sql_str = f"""
            SELECT ...
            FROM `{dataset}.eib_import_accounting_journal`
            WHERE acc_period = '{year}-{month:02d}'
              AND process_map_id = {run_id}
        """

class WDJournalTMSLinesSQL(AccountingSQLQuery):
    """Queries posted Workday journal lines for TMS reconciliation."""
    ...

class ReconDataSQL(AccountingSQLQuery):
    """Queries platform reconciliation data for a given GL account."""
    ...
```

This pattern decouples SQL logic from business logic and makes queries independently testable.

---

## Data Quality — `datachecker.py`

The `datachecker` module runs pre-export data quality assertions within the CLI. Three check classes are defined:

| Class | Scope | Key Assertions |
|---|---|---|
| `CheckJournalLines` | EIB journal lines | Debit/credit amounts non-negative; no null GL account |
| `CheckJournalImport` | EIB import data | Required fields present; amounts match journal totals |
| `CheckReconData` | Reconciliation data | Platform vs Workday completeness |

Results are collected as a `{True: {assertion: description}, False: {assertion: description}}` dict and surfaced to the CLI via `get_data_quality_tests()`.

---

## Local Parameter Store — `localdb.py`

The `ParametersDataBase` class manages a local SQLite database that bridges the CLI session and the JupyterLab workpaper templates:

```python
# localdb.py
class ParametersDataBase:
    """SQLite client for reading and writing run parameters."""

    def set_parameters(self, year: int, month: int, run_id: int) -> tuple:
        """Write run parameters to the local db."""
        ...

    def get_parameters(self) -> tuple:
        """Read the most recently written run parameters."""
        ...
```

When the CLI writes parameters with command 11 (`write_parameters_to_db`), the notebooks pick them up via `ParametersDataBase().get_parameters()` at execution time — ensuring workpapers always process the same period as the CLI session.

---

## Workpapers — JupyterLab

Two JupyterLab notebook templates serve as review workpapers:

| Notebook | Process | Output |
|---|---|---|
| `reviewjournal.ipynb` | Journal entry | HTML review workpaper with reconciliations, data quality results, and system code narrative |
| `reconciliation.ipynb` | Reconciliation | HTML reconciliation workpaper with platform vs Workday comparison, MoM trend, and data quality results |

The workpaper execution is triggered via command 13 (`Export Review File`):

```python
# journals.py — export_review_file()
conversion_str = """jupyter nbconvert --execute {template} \
    --no-input --to html \
    --output {name} --output-dir {output_path}"""

os.system(conversion_str)
webbrowser.open(export_path + file_name + '.html')
```

The `--no-input` flag suppresses all code cells in the exported HTML — the output is a clean, code-free document suitable for audit attachment.

---

## Testing

The test suite is split across unit and integration tests:

### Unit Tests (`tests/unit/`)

| Test Module | Scope |
|---|---|
| `test_accounting.py` | `EIBJournalLines`, `WDJournalsData`, `PlatformReconData` — data model classes |
| `test_datachecker.py` | `CheckJournalLines`, `CheckJournalImport`, `CheckReconData` — assertion logic |
| `test_localdb.py` | `ParametersDataBase` — SQLite read/write |
| `test_utils.py` | `GLAccount`, `get_period_string`, `get_export_path` — utility functions |

Unit tests cover all methods that do not connect to external systems. All unit tests passed.

### Integration Tests (`tests/integration/`)

| Test Module | Scope |
|---|---|
| `test_prepeib.py` | `PreparesAccountingJournal` — end-to-end EIB file generation against BigQuery |
| `test_prepjournal.py` | `PreparesJournal` — end-to-end journal reconciliation against BigQuery |
| `test_preprecon.py` | `PreparesRecon` — end-to-end reconciliation workpaper against BigQuery |

Integration tests verify that the application successfully interfaces with BigQuery and produces the expected outputs. All integration tests passed.

---

[Back to Overview](..) · [Data Model](data-model) · [Workflows](workflows) · [CLI Application](cli) · [BI Reporting](bi) · [Extended Report](report) 
