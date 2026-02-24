# OpenFlow Analytics - Snowflake-Native dbt Project

A dbt project for transforming data loaded by OpenFlow into Snowflake. This project runs **entirely inside Snowflake** using Snowflake-native dbt (`snow dbt`). No external CI/CD runners, no credential management, no network connectivity issues.

## Why Snowflake-Native dbt?

| Aspect | dbt Core (External) | Snowflake-Native dbt |
|--------|---------------------|----------------------|
| **Where it runs** | Your laptop or CI/CD runner | Inside Snowflake |
| **Authentication** | Credentials via env vars or secrets | Snowflake session (automatic) |
| **Network access** | Runner must reach Snowflake | Already inside Snowflake |
| **CI/CD pipeline** | GitHub Actions + secrets setup | Not needed |
| **Scheduling** | External scheduler (cron, Airflow) | Snowflake Tasks |
| **Cost** | Runner compute + Snowflake warehouse | Snowflake warehouse only |

## Project Structure

```
demo_Dbt_Snowflake/
├── dbt_project.yml              # Project configuration
├── profiles.yml                 # Snowflake session auth (placeholders)
│
├── models/
│   ├── sources.yml              # Source definitions
│   ├── generic_tests.yml        # Generic tests (not_null, unique, etc.)
│   │
│   ├── raw/                     # Raw layer (views)
│   │   ├── raw_postgres__country.sql
│   │   ├── raw_postgres__customer_loyalty.sql
│   │   └── raw_sharepoint__documents.sql
│   │
│   ├── int/                     # Intermediate layer (views)
│   │   └── int_sharepoint__documents.sql
│   │
│   └── reporting/               # Reporting layer (tables)
│       └── rpt_document_summary.sql
│
└── tests/                       # Singular tests (custom SQL)
    ├── assert_positive_file_sizes.sql
    ├── assert_no_future_dates.sql
    ├── assert_modified_after_created.sql
    └── assert_report_totals_match.sql
```

## Data Lineage

```
PostgreSQL ──► raw_postgres__country ──────────────────────► (standalone)
PostgreSQL ──► raw_postgres__customer_loyalty ─────────────► (standalone)
SharePoint ──► raw_sharepoint__documents ──► int_sharepoint__documents ──► rpt_document_summary
```

## Deployment & Execution

### Step 1: Deploy the project into Snowflake

```bash
snow dbt deploy openflow_analytics \
  --source . \
  --database POSTGRES_REPLICA \
  --schema DBT_PROJECTS \
  --connection <your-connection> \
  --force
```

This uploads the entire dbt project into Snowflake. No external files or dependencies remain outside.

### Step 2: Run all models

```bash
snow dbt execute \
  --connection <your-connection> \
  --database POSTGRES_REPLICA \
  --schema DBT_PROJECTS \
  openflow_analytics run
```

### Step 3: Run all tests

```bash
snow dbt execute \
  --connection <your-connection> \
  --database POSTGRES_REPLICA \
  --schema DBT_PROJECTS \
  openflow_analytics test
```

### Step 4 (Optional): Schedule with Snowflake Tasks

Once deployed, you can schedule recurring runs using Snowflake Tasks:

```sql
CREATE OR REPLACE TASK dbt_openflow_analytics_task
  WAREHOUSE = 'SNOWFLAKE_LEARNING_WH'
  SCHEDULE = 'USING CRON 0 6 * * * America/Los_Angeles'  -- Daily at 6 AM
AS
  EXECUTE DBT PROJECT openflow_analytics
    ARGS = 'run'
    DATABASE = 'POSTGRES_REPLICA'
    SCHEMA = 'DBT_PROJECTS';

-- Enable the task
ALTER TASK dbt_openflow_analytics_task RESUME;
```

## Tests

| Type | Count | Location |
|------|-------|----------|
| Generic Tests | 15 | `models/generic_tests.yml` |
| Singular Tests | 4 | `tests/*.sql` |

Run all tests:
```bash
snow dbt execute \
  --connection <your-connection> \
  --database POSTGRES_REPLICA \
  --schema DBT_PROJECTS \
  openflow_analytics test
```

## Key Differences from demo_Dbt (dbt Core version)

- **No `.github/workflows/`** -- no CI/CD pipeline needed
- **No `profiles.yml.template`** -- no external credentials to manage
- **No GitHub Secrets** -- Snowflake session handles authentication
- **Scheduling** -- use Snowflake Tasks instead of GitHub Actions triggers
- **Same models, tests, and sources** -- the dbt logic is identical

## Version

v1.0.0
