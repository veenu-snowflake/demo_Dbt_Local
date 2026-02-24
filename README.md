# OpenFlow Analytics - dbt Core (Local)

A dbt project for transforming data loaded by OpenFlow into Snowflake. This project uses **dbt Core** -- the open-source dbt engine runs locally on your machine (or a CI/CD runner) and sends SQL to Snowflake for execution.

## How This Differs from demo_Dbt (Snowflake-Native)

| Aspect | This Project (dbt Core) | demo_Dbt (Snowflake-Native) |
|--------|------------------------|----------------------------|
| **dbt engine runs on** | Your laptop / CI runner | Inside Snowflake |
| **Command to run** | `dbt run` | `snow dbt execute ... run` |
| **Visible in Snowflake UI** | No | Yes (under dbt Projects) |
| **Authentication** | Credentials via env vars | Snowflake session (automatic) |
| **Network access** | Machine must reach Snowflake | Already inside Snowflake |
| **CI/CD** | GitHub Actions + `dbt run` | GitHub Actions + `snow dbt` |
| **Scheduling** | External (cron, Airflow) | Snowflake Tasks |

## Project Structure

```
demo_Dbt_Local/
├── dbt_project.yml              # Project configuration
├── profiles.yml.template        # Connection template (copy to profiles.yml)
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

## Setup

1. Install dbt:
   ```bash
   pip install dbt-core dbt-snowflake
   ```

2. Copy the template profile:
   ```bash
   cp profiles.yml.template profiles.yml
   ```

3. Set environment variables:
   ```bash
   export SNOWFLAKE_ACCOUNT="SFSEAPAC-VYADAV_AWS_AU"
   export SNOWFLAKE_USER="your-user"
   export SNOWFLAKE_PASSWORD="your-password"
   ```

4. Verify connection:
   ```bash
   dbt debug --profiles-dir .
   ```

5. Run models:
   ```bash
   dbt run --profiles-dir .
   ```

6. Run tests:
   ```bash
   dbt test --profiles-dir .
   ```

7. Run everything (models + tests):
   ```bash
   dbt build --profiles-dir .
   ```

## CI/CD

This project uses GitHub Actions for CI/CD. The pipeline runs dbt Core on the runner.

| Trigger | Action |
|---------|--------|
| Push to `develop` | Lint + Build & Test (CI schema) |
| PR to `main` | Lint + Build & Test (CI schema) |
| Push to `main` | Lint + Build & Test + Deploy to Production (with approval) |

### Required GitHub Secrets

| Secret | Description |
|--------|-------------|
| `SNOWFLAKE_ACCOUNT` | Snowflake account identifier (e.g., `SFSEAPAC-VYADAV_AWS_AU`) |
| `SNOWFLAKE_USER` | Snowflake username |
| `SNOWFLAKE_PASSWORD` | Snowflake password |
| `SNOWFLAKE_ROLE` | Snowflake role (e.g., `ACCOUNTADMIN`) |
| `SNOWFLAKE_WAREHOUSE` | Snowflake warehouse name |

### Network Policy Note

The CI/CD runner must be able to reach your Snowflake account. If your account has a network policy restricting IP access, you'll need to either:
- Whitelist the runner's IP (not practical for GitHub-hosted runners)
- Use a self-hosted runner on an allowed network

## Tests

| Type | Count | Location |
|------|-------|----------|
| Generic Tests | 15 | `models/generic_tests.yml` |
| Singular Tests | 4 | `tests/*.sql` |

## Version

v1.0.0
