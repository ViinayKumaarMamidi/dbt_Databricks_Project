# dbt_Databricks_Project
This repo contains details about leveraging Dbt tool with the help of Databricks and building Dbt concepts end to end. Thanks

**Complete documentation:** https://deepwiki.com/ViinayKumaarMamidi/dbt_Databricks_Project

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/ViinayKumaarMamidi/dbt_Databricks_Project)


# dbt + Databricks — Step-by-step Implementation Guide

This guide walks you through designing, developing, testing, and deploying a dbt project that targets Databricks (Delta Lake). It covers setup, common patterns, configuration snippets, CI/CD, and best practices so you can get a production-ready dbt + Databricks workflow.

Table of contents
- Overview & architecture
- Prerequisites
- Repo layout (recommended)
- Install dbt & adapter
- Databricks workspace & cluster setup
- dbt profile for Databricks (examples)
- dbt_project.yml example
- Models: basic, incremental, ephemeral, and snapshots
- Sources, seeds, and tests
- Macros and hooks
- Documentation (dbt docs) and lineage
- CI/CD / GitHub Actions example
- Deploying to Databricks (Jobs API)
- Versioning, branching & releases
- Monitoring, observability & troubleshooting
- Security & permissions
- Performance & cost tips
- Appendix (useful commands & examples)

---

Overview & architecture
- Purpose: author and maintain data transformations with dbt, run them on Databricks (Delta Lake), and ship tested, documented models into production.
- Pattern:
  - Raw data ingested into bronze Delta tables (streaming or batch)
  - dbt models transform bronze -> silver (clean/enriched) -> gold (aggregates, marts)
  - dbt runs scheduled via Databricks Jobs or external scheduler (Airflow/GitHub Actions)
  - CI runs tests and static checks on PRs; only merge to main after passing
  - Documentation published with dbt docs; lineage visible in Databricks and dbt docs

Prerequisites
- Databricks workspace with an account and access to create clusters or Jobs.
- Databricks token (for API calls / dbt profile).
- Optional: Unity Catalog configured (recommended for governance).
- Python 3.8+ for local development.
- dbt Core and dbt-databricks adapter (versions that match your dbt version).
- GitHub repository to store dbt project.
- CLI tools: git, aws/azure/gcloud CLIs if needed for storage access.
- (Optional) Databricks CLI configured for job deployment and file upload.

Recommended repo layout
- dbt_project.yml
- /models/
  - /staging/ (from sources to cleaned staging)
  - /intermediate/
  - /marts/ (gold)
- /macros/
- /seeds/
- /snapshots/
- /tests/ (custom data tests)
- /analysis/
- /docs/
- README.md

Install dbt & adapter
- Use a virtual environment. Example:
  - python -m venv .venv && source .venv/bin/activate
  - pip install --upgrade pip
- Install dbt core and the databricks adapter (choose matching versions):
  - pip install dbt-core dbt-databricks
  - Alternatively install a pinned version:
    - pip install "dbt-core==1.4.0" "dbt-databricks==1.4.0"
- Verify installation:
  - dbt --version

Databricks workspace & cluster setup
- Create or identify:
  - A SQL endpoint or job cluster suitable for dbt runs. For production, use job clusters configured via Databricks Jobs.
  - An isolation policy or network/security configuration (VPC injection / VNet).
- Configure cluster/runtime:
  - Use a Spark runtime compatible with dbt adapter (check dbt-databricks docs).
  - For Delta performance: enable Auto Optimize / Optimize writes if needed.
  - Attach appropriate IAM roles / policies for S3 or cloud storage access (AWS/GCP/Azure).
- Recommended cluster sizing:
  - Small dev/test: 2–4 cores
  - Prod: size according to data volume — allow autoscaling and use spot instances where appropriate.

dbt profile(s) for Databricks
- dbt connects to Databricks either via JDBC-like HTTP path + token (for Databricks SQL endpoints or clusters) or through the Databricks SQL connector. Put your profile in `~/.dbt/profiles.yml` or CI secrets.

Example profiles.yml — Databricks serverless / SQL endpoint (AWS):
```yaml
my_databricks_profile:
  target: dev
  outputs:
    dev:
      type: databricks
      catalog: hive_metastore            # or your Unity Catalog catalog
      schema: analytics_dev              # target schema/database
      host: <adb-xxxxxxxxxx.XX.azuredatabricks.net or <region>.azuredatabricks.net>
      http_path: /sql/1.0/endpoints/<endpoint-id>   # SQL endpoint path or cluster HTTP path
      token: "{{ env_var('DATABRICKS_TOKEN') }}"
      threads: 4
      connect_retries: 3
      # optional:
      organization: "<databricks-org-id>"   # when required
      timeout_seconds: 120
```

If using Unity Catalog:
```yaml
    dev:
      type: databricks
      catalog: main_catalog               # Unity Catalog catalog
      schema: analytics_dev               # Unity Catalog schema
      host: <databricks-host>
      http_path: /sql/1.0/endpoints/<endpoint-id>
      token: "{{ env_var('DATABRICKS_TOKEN') }}"
```

Notes:
- Keep secrets out of repo: use environment variables in CI (DATABRICKS_TOKEN, DBT_TARGET).
- For Databricks on AWS, http_path can be a cluster http path (jobs) or SQL endpoint path.

dbt_project.yml (example)
```yaml
name: my_dbt_databricks_project
version: '1.0'
config-version: 2

profile: my_databricks_profile

source-paths: ["models"]
analysis-paths: ["analysis"]
test-paths: ["tests"]
data-paths: ["data"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]
target-path: "target"
clean-targets:
  - "target"
  - "dbt_modules"

models:
  +materialized: view
  staging:
    +schema: staging
    +tags: ['staging']
  marts:
    +schema: marts
    +materialized: table
```

Models: conventions & examples
- Use layered approach:
  - staging: simple SQL that selects + cast + basic cleaning from source
  - intermediate / intermediate marts: joins and business logic
  - marts: final tables, aggregates, or BI consumption tables

Example simple model (models/staging/stg_orders.sql):
```sql
{{ config(materialized='view') }}

with raw as (
  select
    id as order_id,
    customer_id,
    parse_timestamp(order_ts) as order_ts,
    total_amount
  from {{ source('raw', 'orders') }}
)

select
  order_id,
  customer_id,
  order_ts,
  cast(total_amount as double) as total_amount
from raw
where order_ts is not null
```

Incremental model (Delta) example (models/marts/orders_agg.sql)
- Use `is_incremental()` to apply incremental logic; use MERGE if writing to Delta (recommended).
```sql
{{ config(materialized='incremental', unique_key='order_id') }}

with source_data as (
  select
    order_id,
    customer_id,
    order_ts,
    total_amount
  from {{ ref('stg_orders') }}
  where order_ts >= date_sub(current_date(), 7)   -- example incremental filter
)

{% if is_incremental() %}

  -- When incremental, merge new/updated rows into the target delta table
  merge into {{ this }} t
  using source_data s
  on t.order_id = s.order_id
  when matched then update set
    t.customer_id = s.customer_id,
    t.order_ts = s.order_ts,
    t.total_amount = s.total_amount
  when not matched then insert (order_id, customer_id, order_ts, total_amount)
  values (s.order_id, s.customer_id, s.order_ts, s.total_amount)

{% else %}

  select * from source_data

{% endif %}
```

Notes:
- dbt-databricks supports writing Delta with MERGE operations. Ensure the target table is Delta.
- Use `unique_key` config to document uniqueness; but actual MERGE logic is left to SQL macro or model.

Snapshots
- Use snapshots to capture slowly changing dimensions (SCD).
- Example snapshot config (snapshots/snap_customers.sql):
```yaml
# snapshots/snapshots.yml
snapshots:
  - name: customers_snapshot
    target_schema: snapshots
    strategy: timestamp
    updated_at: updated_at_column
    unique_key: customer_id
```
- Then SQL snapshot file `snapshots/customers_snapshot.sql` contains a `select * from {{ source('raw', 'customers') }}`

Sources, seeds, and tests
- Sources: declare raw tables in `models/src.yml` using `sources:` blocks to document lineage and enable source freshness tests.
- Seeds: small lookup tables or mapping CSVs under `/seeds/`. Load with `dbt seed`.
- Tests:
  - Use schema tests in `.yml` model files: `unique`, `not_null`, `relationships`.
  - Implement custom tests for complex constraints if needed.
- Example source + test (models/sources.yml):
```yaml
version: 2
sources:
  - name: raw
    tables:
      - name: orders
        description: "Raw orders table"
        freshness:
          warn_after: {count: 1, period: day}
          error_after: {count: 2, period: day}
```

Macros & hooks
- Use macros for reusable SQL (merge patterns, upsert macros).
- Use on-run-start / on-run-end hooks for environment setup or post-run notifications.
- Example macro for Delta MERGE (macros/merge_delta.sql):
```sql
{% macro merge_delta(target, source, key_columns, update_columns) %}
merge into {{ target }} t
using {{ source }} s
on {{ " and ".join(["t.%s = s.%s" | format(c,c) for c in key_columns]) }}
when matched then update set {{ ", ".join(["t.%s = s.%s" | format(c,c) for c in update_columns]) }}
when not matched then insert *
{% endmacro %}
```

Documentation & lineage
- Generate docs: `dbt docs generate`
- Serve locally: `dbt docs serve` (dev only)
- Publish docs artifact as part of CI/CD and host via GitHub Pages or upload to an S3 bucket or Databricks Workspace (via REST API).
- Use `description:` fields in `.yml` files for columns, sources, and models.

Common dbt CLI commands
- dbt debug
- dbt deps
- dbt seed
- dbt run --models tag:staging
- dbt test
- dbt snapshot
- dbt docs generate && dbt docs serve

CI/CD: GitHub Actions example
- CI typically runs on PRs: lint, dbt deps, dbt compile, dbt run (optionally against a dev/schema), dbt test.
- Use ephemeral schema / prefix per run or run against a dev schema to avoid interfering with prod.

Example GitHub Actions workflow (.github/workflows/ci.yml):
```yaml
name: dbt CI

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  dbt-ci:
    runs-on: ubuntu-latest
    env:
      DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
      DATABRICKS_HTTP_PATH: ${{ secrets.DATABRICKS_HTTP_PATH }}
      DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          python -m venv .venv
          source .venv/bin/activate
          pip install --upgrade pip
          pip install dbt-core dbt-databricks

      - name: Configure dbt profile
        run: |
          mkdir -p ~/.dbt
          cat > ~/.dbt/profiles.yml <<EOF
          my_databricks_profile:
            target: ci
            outputs:
              ci:
                type: databricks
                catalog: hive_metastore
                schema: ci_${{ github.run_id }}
                host: $DATABRICKS_HOST
                http_path: $DATABRICKS_HTTP_PATH
                token: $DATABRICKS_TOKEN
                threads: 4
          EOF

      - name: dbt deps
        run: |
          source .venv/bin/activate
          dbt deps

      - name: dbt seed
        run: |
          source .venv/bin/activate
          dbt seed --profiles-dir ~/.dbt

      - name: dbt run
        run: |
          source .venv/bin/activate
          dbt run --profiles-dir ~/.dbt

      - name: dbt test
        run: |
          source .venv/bin/activate
          dbt test --profiles-dir ~/.dbt
```

Notes:
- CI creates an isolated schema `ci_${{ github.run_id }}` to avoid conflicts.
- Use GitHub Secrets to store host, http_path and token.
- Consider spinning ephemeral Databricks clusters using the Jobs API (start -> run -> terminate) to reduce cost.

Deploying dbt runs to Databricks Jobs
- Two common patterns:
  - Use Databricks Jobs to run a packaged dbt job on a cluster: Upload your project to DBFS (or use a repo-backed workspace item) and create a job that runs `dbt run`.
  - Use external orchestrator that calls Databricks Jobs API to start a job cluster and run dbt CLI.
- Example high-level steps:
  1. Build a wheel or package of your project or mount repo in Databricks Repos.
  2. Create a Databricks Job with a task that executes `python -m pip install .` and then `dbt run --profiles-dir /dbt_profile`.
  3. Configure job cluster with required libraries (Delta, cloud connectors).
  4. Trigger the Databricks Job from CI on merges to main.

Using Databricks REST API (example: run a job)
- Use the Databricks Jobs API with the token:
```bash
curl -X POST https://<databricks-host>/api/2.1/jobs/run-now \
  -H "Authorization: Bearer $DATABRICKS_TOKEN" \
  -d '{ "job_id": <job-id> }'
```

Versioning, branching & releases
- Use GitFlow or trunk-based workflows.
- Enforce PR checks (CI) so merges to main are tested.
- Tag releases and promote artifacts (manifest.json and compiled SQL) into a release artifact bucket/registry if needed.

Monitoring, observability & troubleshooting
- Capture dbt logs (target/run logs) and upload to centralized logging (S3, CloudWatch).
- Monitor Databricks Jobs: success/failure, run duration, driver logs, stderr/stdout.
- Use Cloud cost monitoring for Databricks clusters.
- Common troubleshooting:
  - `dbt debug` for connection issues
  - Validate cluster libraries and Spark runtime versions
  - Check DBFS or cloud storage permissions for seed/snapshot/checkpoint writes

Security & permissions
- Use least-privilege for Databricks tokens: create tokens with restricted scope and rotate regularly.
- Use Unity Catalog for table-level governance and ABAC if available.
- Store secrets in GitHub Secrets or use a secrets manager (Azure Key Vault, AWS Secrets Manager).
- Avoid committing credentials to repo.

Performance & cost tips
- Use Delta Lake features: ZORDER, OPTIMIZE for large tables
- Partition tables on frequently queried keys (date, customer region)
- Use appropriate cluster sizes; leverage spot workers for cost savings
- Use job clusters (start/stop per job) instead of long-running interactive clusters to save cost
- Use `threads` in dbt profiles to parallelize model runs where dependencies allow

Appendix: Useful commands & examples
- Initialize project (if starting fresh):
  - dbt init my_dbt_databricks_project
- Run single model:
  - dbt run --models marts.orders_agg
- Run tests:
  - dbt test --models +marts.orders_agg
- Compile project:
  - dbt compile
- Generate docs:
  - dbt docs generate
  - dbt docs serve
- Clean target:
  - dbt clean

Quick checklist before first production run
- [ ] Databricks token and host verified
- [ ] dbt profile configured in production with secrets protected
- [ ] Clusters/job configured with required libraries
- [ ] Tests written & passing locally
- [ ] CI pipeline in place (PR checks)
- [ ] Backups & retention policies for Delta / S3
- [ ] Monitoring & alerts for job failures and lag


