# Open-source / open-core DQ tools as reference designs: DQOps, Great Expectations, dbt (as of Oct 2026)

> **Raw research notes, not recommendations.** Evidence gathered on 2026-10-18 for the reports in `research-outputs/`. Vendor claims are marked as such; screenshots described here were not committed. Use these to check a claim's source, never as a decision or a plan.


Research date: 2026-10-18. Sources are official docs, GitHub repos (read through the `gh` API), and vendor press. The repo-level facts (licenses, release tags, file layouts) come straight from the GitHub API in this session.

**Three things changed in 2025-26 and invalidate older comparisons:**
1. DQOps is no longer Apache 2.0. The repo now ships under a Business Source License 1.1 with restrictions.
2. dbt is now a Rust rewrite (dbt v2.0, GA, at 2.0.8 as of 14 Oct 2026). It comes in an Apache-2.0 `dbt-oss` distribution and a proprietary `dbt` distribution. The separate `dbt-fusion` repo is archived.
3. Fivetran and dbt Labs closed their merger on 1 June 2026.

---

## Q1. DQOps: architecture, sensor/rule model, check types, storage, anomaly detection, deployment, licensing

### Takeaway
DQOps is the closest off-the-shelf match to a "DQ monitoring platform" design, and its internals are a strong blueprint:
- A Java 17 / Spring Boot server with a TypeScript UI.
- Jinja2 SQL "sensors" that push measurements down to the database.
- Separate Python "rules" that judge each reading against thresholds or its history.
- YAML rules-as-code.
- A local Hive-partitioned Parquet "data lake" of results.

However, its license is now BSL 1.1, which forbids embedding it in a commercial or for-profit product or hosted offering. The project is also small: 194 GitHub stars, with the last push on 5 Jan 2026. Copy its patterns rather than depend on it.

### Cited Findings

**Architecture**
- The runtime is "mostly developed as a Java application based on Spring Boot". It runs as a JVM process with an embedded web server on default port 8888 — [DQOps architecture](https://dqops.com/docs/dqo-concepts/architecture/dqops-architecture/)
- The JVM starts two Python processes, `evaluate_rules.py` and `evaluate_templates.py`, both under `$DQO_HOME/lib`. Sensors are Jinja2 SQL templates and rules are Python functions — [DQOps architecture](https://dqops.com/docs/dqo-concepts/architecture/dqops-architecture/)
- Language mix by bytes (GitHub languages API): Java ~51.3 MB, Python ~8.3 MB, Jinja ~3.6 MB, TypeScript ~2.3 MB. TypeScript is the web UI; the docs say the web UI talks to an embedded REST API — [GitHub dqops/dqo](https://github.com/dqops/dqo); [architecture](https://dqops.com/docs/dqo-concepts/architecture/dqops-architecture/)
- Other interfaces are a CLI, a Python client wrapping the REST API (e.g. `run_checks`), and Airflow operators `run_checks_operator` and `assert_table_status_operator` — [DQOps architecture](https://dqops.com/docs/dqo-concepts/architecture/dqops-architecture/)
- The YAML files can be edited directly. DQOps watches for file changes (including from `git pull`) and invalidates its in-memory cache — [DQOps architecture](https://dqops.com/docs/dqo-concepts/architecture/dqops-architecture/)

**Sensor vs. rule separation**
- A check combines a sensor and a rule. The sensor "reads the value from the data source at a given point in time". The rule is "a set of conditions (thresholds) that the sensor readout must meet" — [DQOps check definition](https://dqops.com/docs/dqo-concepts/definition-of-data-quality-checks/)
- A sensor's SQL must return a column named `actual_value` — [DQOps check definition](https://dqops.com/docs/dqo-concepts/definition-of-data-quality-checks/)
- Source code shows each sensor has one Jinja2 template per dialect. Example: `home/sensors/table/volume/row_count/` holds `{bigquery,clickhouse,databricks,db2,duckdb,hana,mariadb,mysql,oracle,postgresql,presto,questdb,redshift,snowflake,spark,sqlserver,teradata,trino}.sql.jinja2`. Each has a matching `.dqoprovidersensor.yaml` and a shared `sensordefinition.dqosensor.yaml` — [GitHub dqops/dqo home/sensors](https://github.com/dqops/dqo/tree/develop/home/sensors)
- The PostgreSQL row-count sensor is just this, with grouping and time-partitioning injected by shared dialect macros:
  ```
  SELECT COUNT(*) AS actual_value
    {{ lib.render_data_grouping_projections(...) }} {{ lib.render_time_dimension_projection(...) }}
  FROM {{ lib.render_target_table() }} ... {{ lib.render_where_clause() }} {{ lib.render_group_by() }}
  ```
  — [postgresql.sql.jinja2](https://github.com/dqops/dqo/blob/develop/home/sensors/table/volume/row_count/postgresql.sql.jinja2)
- Rules live as `.py` + `.dqorule.yaml` pairs under `home/rules/{averages,change,comparison,percentile,stdev}` — [GitHub home/rules](https://github.com/dqops/dqo/tree/develop/home/rules)
- Each rule exposes `evaluate_rule(rule_parameters) -> RuleExecutionResult(passed, expected_value, lower_bound, upper_bound)`. The inputs include `actual_value`, `parameters`, `previous_readouts: Sequence[HistoricDataPoint]` and `time_window` (`prediction_time_window`, `min_periods_with_readouts`) — [anomaly_stationary_percentile_moving_average.py](https://github.com/dqops/dqo/blob/develop/home/rules/percentile/anomaly_stationary_percentile_moving_average.py)

**Check types and severities**
- Profiling checks assess a new source.
- Monitoring checks keep one result per table or column, in daily or monthly variants. The daily variant keeps the latest status per day, the monthly one the latest status per month.
- Partition checks store one result per time partition. They need a time-partitioning column and enable incremental evaluation of new or changed partitions.
- Source: [DQOps check definition](https://dqops.com/docs/dqo-concepts/definition-of-data-quality-checks/)
- There are three severity levels: warning, error and fatal. Warnings "are not treated as data quality issues" and don't lower the KPI. Error is the default for monitoring checks and counts against the KPI. Fatal is meant to stop pipelines. When several rules fail, the most severe one is reported — [DQOps check definition](https://dqops.com/docs/dqo-concepts/definition-of-data-quality-checks/)
- YAML example: `columns: <col>: profiling_checks: profile_nulls_percent: {warning: {max_percent: 1.0}, error: {max_percent: 5.0}, fatal: {max_percent: 30.0}}`. It validates against the JSON schema `https://cloud.dqops.com/dqo-yaml-schema/TableYaml-schema.json` — [DQOps check definition](https://dqops.com/docs/dqo-concepts/definition-of-data-quality-checks/)
- DQOps ships "around 150 predefined data quality checks". The README lists BigQuery, Snowflake, PostgreSQL, Redshift, SQL Server and MySQL among its sources, and the repo has templates for ~18 dialects (above) — [GitHub README](https://github.com/dqops/dqo)
- Checks can be configured "from the UI or in YAML files". It has built-in job scheduling, plus notifications and incidents through Slack or a webhook — [GitHub README](https://github.com/dqops/dqo)

**Result storage (local Parquet data lake)**
- The `.data` folder holds six Parquet tables: `check_results`, `error_samples`, `errors`, `incidents`, `sensor_readouts` and `statistics` — [DQOps data storage](https://dqops.com/docs/dqo-concepts/data-storage-of-data-quality-results/)
- The tables use a Hive-style layout partitioned by connection, table and month: `c=<conn>/t=<table>/m=2023-02-01`. `incidents` has no `t=` level — [DQOps data storage](https://dqops.com/docs/dqo-concepts/data-storage-of-data-quality-results/)
- Each sensor readout row is copied to `check_results` with extra columns such as `severity`. The tables share `time_period`, `time_period_utc`, `check_hash`, `data_group_hash`, `time_series_id` and `id`.
- `id` is a hash of (`time_series_id`, `check_hash`, `time_period`), so a rerun overwrites the row (an idempotent upsert).
- The files can be queried by Spark, Trino or Athena as external tables.
- Source for the three points above: [DQOps data storage](https://dqops.com/docs/dqo-concepts/data-storage-of-data-quality-results/)
- DQOps Cloud replicates `.data` into a data bucket and the configuration (`sources`, `sensors`, `rules`, `checks`, `settings`, `.credentials`) into a config bucket. It "continuously loads these parquet files ... to BigQuery native tables". Dashboards are Looker Studio through a community connector. Direct warehouse access needs ENTERPRISE — [DQOps architecture](https://dqops.com/docs/dqo-concepts/architecture/dqops-architecture/)

**Anomaly detection**
- The free version "detects anomalies by calculating the Z-score over rolling quadrilles" (sic) and does not detect seasonality. The commercial version "uses an advanced AI algorithm that considers seasonality" (no details given) — [DQOps anomaly docs](https://dqops.com/docs/categories-of-data-quality-checks/how-to-detect-anomaly-data-quality-issues/)
- The check families are `min/max/mean/median/sum_anomaly` and `*_change` (suffixes `_1_day`, `_7_days`, `_30_days`). They use a 90-day window and need at least 30 days of history — [DQOps anomaly docs](https://dqops.com/docs/categories-of-data-quality-checks/how-to-detect-anomaly-data-quality-issues/)
- In code, `anomaly_stationary_percentile_moving_average.py` uses numpy/scipy:
  1. Take the median and `scipy.stats.tstd` of previous readouts.
  2. Transform the readouts to relative multiples of the median.
  3. Compute two-tailed bounds from `anomaly_percent`.
  4. Return expected value and lower/upper bounds.

  Other rule files include `anomaly_partition_row_count`, `anomaly_timeliness_delay`, `anomaly_differencing_percentile_moving_average(_30_days)`, `percentile_moving_{7,30,60}_days` and `change_percentile_moving_{7,30,60}_days` — [GitHub home/rules/percentile](https://github.com/dqops/dqo/tree/develop/home/rules/percentile)

**Deployment**
- Local: `python -m pip install --user dqops`, then `python -m dqops`. The first start downloads the distribution and a Java 17 JRE — [GitHub README](https://github.com/dqops/dqo); [architecture](https://dqops.com/docs/dqo-concepts/architecture/dqops-architecture/)
- Production: Docker image `dqops/dqo`. Mount the user home at `/dqo/userhome`, expose port 8888, run headless, and pass a Cloud API key.
- Hybrid: on-premise instances sync to the Cloud data lake.
- Source for the two points above: [DQOps architecture](https://dqops.com/docs/dqo-concepts/architecture/dqops-architecture/)

**Licensing and pricing**
- The LICENSE is now the "DQOPS BUSINESS SOURCE LICENSE AGREEMENT, Business Source License 1.1", from licensor Documati sp. z o.o., covering DQOps 1.13.2.
- Its Additional Use Grant allows production use. But you may not distribute a derivative work, or offer it as a hosted service, "for a fee or otherwise on a commercial or other for-profit basis". You also may not "link the Licensed Work to, or otherwise include [it] in or with, any product ... offered ... on a commercial or other for-profit basis".
- Each file converts to Apache 2.0 four years after its last modification. GitHub reports the license as `NOASSERTION`.
- Source for the three points above: [dqops/dqo LICENSE](https://github.com/dqops/dqo/blob/develop/LICENSE)
- Releases: v1.12.0 (23 Jul 2025), v1.13.0 (28 Dec 2025), v1.13.1 (5 Jan 2026). The latest commit (5 Jan 2026) bumps the version to 1.13.2 — [GitHub releases / API](https://github.com/dqops/dqo/releases)
- Paid SaaS tiers: Personal at $600/mo billed annually (200 tables, 1 user, 2 concurrent jobs) and Team at $2,000/mo (2,000 tables, 5 users, 20 jobs, includes the DQ data warehouse). Enterprise is custom, and only Enterprise lists on-premise or hybrid deployment. The pricing page lists no free tier — [DQOps pricing](https://dqops.com/pricing/)

### Inferences
- **Patterns worth copying:** a Jinja2 template per dialect for each metric, all returning `actual_value`; a separate stateless Python rule given (current value, history window) that returns (passed, expected, lower, upper); three-tier severity, with warnings excluded from KPIs; a deterministic result `id` hash so reruns are idempotent; and Hive-partitioned Parquet results that a lakehouse engine (Trino/Spark/BigQuery/DuckDB) can read.
- **Embedding risk for Globe Telecom:** internal use is probably covered by the Additional Use Grant. However, clause (ii) is broad about including DQOps "in or with" any for-profit product, and Globe is a for-profit company. Legal review is needed before embedding it in anything customer- or partner-facing.
- **Project health:** no commits since Jan 2026 and 194 stars suggest low community momentum. This is a vendor-lock and continuity risk.
- **Limitations vs. a telco need:** the free anomaly detection is Z-score/percentile with no seasonality, which matters for strongly weekly or daily-seasonal telco volumes such as CDRs and recharges. Dashboards depend on DQOps Cloud plus Looker Studio. Lineage is not a core feature.

### Gaps
- Whether DQOps uses DuckDB internally to query its own Parquet files was not confirmed by the pages fetched.
- The exact date of the switch from Apache 2.0 to BSL was not confirmed. My recollection that earlier DQOps releases were Apache-2.0 is unverified in this session.
- Which features are "paid-only" in the self-hosted build, beyond seasonality-aware anomaly detection and the data warehouse, is not listed explicitly.
- The scheduler internals (cron expressions per check type, job queue) were not reviewed in source.

---

## Q2. Great Expectations (GX Core 1.x and GX Cloud)

### Takeaway
GX Core (Apache-2.0, Python, v1.24.0 released 7 Oct 2026) is a validation library rather than a monitoring platform. Its object model is: Data Context → Data Source/Asset/Batch Definition → Expectation Suite → Validation Definition → Checkpoint → Actions, with Data Docs as an HTML output. It pushes metric computation down to the backend through SQLAlchemy, Spark or Pandas execution engines.

Scheduling, anomaly detection, ExpectAI (AI-recommended expectations, beta) and a hosted UI are GX Cloud features. Telemetry is on by default in Core.

### Cited Findings

**Repo facts**
- License Apache-2.0, ~11.9k stars. Recent releases: 1.23.1 (18 Sep 2026), 1.23.2 (28 Sep 2026), 1.24.0 (7 Oct 2026) — [GitHub great_expectations](https://github.com/great-expectations/great_expectations)

**Object model**
- The Data Context is the entry point to configs, metadata and validation results.
- A Data Source is "the GX representation of a data store". A Data Asset is a collection of records, and a Batch Definition organises records into Batches.
- An Expectation is "a verifiable assertion about data", and Suites group Expectations.
- A Validation Definition "explicitly associates" a Batch Definition with a Suite.
- A Checkpoint runs multiple Validation Definitions and triggers Actions, such as email, Slack or Teams notifications.
- Data Docs host Suites and Validation Results.
- Source for the six points above: [GX overview](https://docs.greatexpectations.io/docs/core/introduction/gx_overview)

**Execution engines and pushdown**
- The `great_expectations/execution_engine/` source contains `pandas_execution_engine.py`, `sparkdf_execution_engine.py`, `sqlalchemy_execution_engine.py`, `sqlalchemy_dialect.py`, `redshift_execution_engine.py`, `sqlite_execution_engine.py` and `partition_and_sample/` — [GitHub execution_engine](https://github.com/great-expectations/great_expectations/tree/develop/great_expectations/execution_engine)
- Metrics are implemented as metric providers: `column_aggregate_metric_provider.py`, `map_metric_provider/`, `column_pair_map_metrics/`, `multicolumn_map_metrics/`, `query_metric_provider.py`, `table_metric_provider.py` — [GitHub expectations/metrics](https://github.com/great-expectations/great_expectations/tree/develop/great_expectations/expectations/metrics)

**0.x → 1.x changes (0.18 is no longer maintained)**
- `BatchRequest` was removed, and the Validation Definition's `data` field now references the asset.
- Profilers (including the Rule-Based Profiler) were removed ("may return later").
- The Validation Definition became its own domain object.
- Fluent datasources live under `fluent_datasources` in `gx/great_expectations.yml`, and arbitrary batching regexes were dropped.
- The Checkpoint file was split into a checkpoint file plus a validation-definition file.
- Expectations are added as class instances, e.g. `suite.add_expectation(gxe.ExpectColumnValuesToBeBetween(...))`.
- "V1 has no default actions", so `update_data_docs` must be added explicitly.
- The config version is 4.0, and the database/S3/GCS/Azure store backends were dropped: file contexts use the filesystem only.
- Source for all points in this list: [GX migration guide](https://docs.greatexpectations.io/docs/reference/learn/migration_guide)

**Telemetry**
- Analytics events are on by default and include "which GX features are used with what OS and Python version".
- To disable, set `GX_ANALYTICS_ENABLED=False` (read when the context initializes) or set `context.variables.analytics_enabled = False` and then `save()` (file context).
- Source for the two points above: [GX usage statistics](https://docs.greatexpectations.io/docs/reference/learn/usage_statistics)

**GX Cloud**
- ExpectAI (BETA) generates AI-recommended Expectations "personalized based on an analysis of a sample of your data". It can also "generate a SQL query based on a natural language prompt" for custom SQL Expectations — [GX Cloud Manage Expectations](https://docs.greatexpectations.io/docs/cloud/expectations/manage_expectations/)
- Anomaly detection: choosing the Schema, Volume or Completeness issue type gives an "Automatic" option. Generated Expectations default to warning severity.
- Profiling: Cloud introspects the schema on asset creation and offers one-click descriptive metrics that are used to suggest parameters.
- Source for the two points above: [GX Cloud Manage Expectations](https://docs.greatexpectations.io/docs/cloud/expectations/manage_expectations/)
- The "Anomaly Detection Model" was announced 2 Jul 2025 for volume and completeness. Users toggle "Anomaly Detection | Explicit", and auto-generated checks are marked with a sparkles icon — [GX blog](https://greatexpectations.io/blog/why-anomaly-detection-matters-in-data-quality-and-how-gx-just-made-it-easier/)
- A Jan 2026 incident blocked validation of assets that use a "forecasting" anomaly-detection parameter. This shows Cloud anomaly checks use forecasting-based parameters — [GX status page](https://status.greatexpectations.io/incidents/wk7xnv9l448h)
- Pricing: a free "Developer" option plus paid "Team" and "Enterprise". No prices or limits are published — [GX pricing](https://greatexpectations.io/pricing/)

### Inferences
- **Pattern to copy:** a declarative, typed "expectation" object (name + kwargs + severity + id) that is independent of any engine. Metric providers then compile it into SQLAlchemy, Spark or Pandas. This is the same sensor-vs-rule split as DQOps, but the metric and the threshold are bundled in one Expectation class.
- **What GX Core does not do:** no scheduler, no results warehouse (results go to JSON/Data Docs unless you persist them yourself), no anomaly detection or ML in Core, no lineage, no incident management. The profilers were removed in 1.x. Orchestration (Airflow, Dagster, etc.) and result storage are up to you.
- **Embedding:** GX Core is the safest library to embed of the three. It is Apache-2.0, Python and has an active release cadence (roughly every 1-3 weeks). The custom-Expectation API allows telco-specific checks.
- **Telemetry should be disabled** in a Globe deployment for data-governance reasons.

### Gaps
- GX Cloud tier prices and limits, and exactly which features (ExpectAI, anomaly detection, scheduling, alerts, SSO, agent runner) are gated to Team/Enterprise, were not published on the pages fetched.
- The algorithm behind GX Cloud's anomaly detection (forecast model type, seasonality) is not documented publicly in the pages fetched.
- How GX Cloud executes against private networks (an agent vs. a hosted runner) was not reviewed.

---

## Q3. dbt: data tests, unit tests, contracts, artifacts, packages, v2/Fusion licensing, merger

### Takeaway
dbt's DQ model is SQL-as-test: each test is a SELECT that returns failing rows. Results are judged by `severity`/`warn_if`/`error_if` on the failure count. Failed rows can optionally be persisted (`store_failures`), and results are surfaced through `run_results.json`.

In 2026 dbt became a Rust engine (v2.0). The Apache-2.0 `dbt-oss` distribution is built from `dbt-labs/dbt` (formerly dbt-core). The proprietary `dbt` distribution is free to use under the dbt Product License, with premium features unlocked by login or a paid platform account. v2 adds queryable Parquet artifacts. Fivetran + dbt Labs closed their merger on 1 Jun 2026.

### Cited Findings

**v2 / Fusion and repo state**
- The `dbt-labs/dbt-core` repo `main` branch "now contains all the Apache 2.0 source code of dbt v2.0 — a ground-up rewrite of dbt in Rust". v1 Python development moved to the `1.latest` branch.
- v2 "produces Parquet artifacts that can be easily queried" and still emits JSON (`manifest.json`, etc.) for backward compatibility.
- v2 ships as a single self-contained binary with no Python runtime. Windows ARM is not yet supported.
- Source for the three points above: [dbt repo README](https://github.com/dbt-labs/dbt-core)
- Releases on 14 Oct 2026: v2.0.8, v1.12.6, v1.11.16 and v1.13.0b1. Two lines are being maintained in parallel — [GitHub releases](https://github.com/dbt-labs/dbt-core/releases)
- The v2.0.8 notes mention an `analyze` phase and `dbt build --write-catalog` — [v2.0.8 release](https://github.com/dbt-labs/dbt-core/releases/tag/v2.0.8)
- The `dbt-labs/dbt-fusion` repo is archived ("ARCHIVE: code & issue tracking in dbt-core") — [GitHub dbt-fusion](https://github.com/dbt-labs/dbt-fusion)
- Announcement: "dbt Core 2.0 is Apache 2.0 licensed and built on the same foundations as the dbt Fusion engine". The shared runtime, "previously committed to ELv2", is now Apache 2.0.
- Fusion is a precompiled Rust binary that includes proprietary code, is "free to use", and unlocks premium features with a free login or paid account. It includes a "built-in high-performance SQL linter".
- Adapters are built on "ADBC and the Arrow ecosystem". The Parquet artifacts are "directly queried through DuckDB".
- Migration check: `dbt parse --use-v2-parser`; the `dbt-autofix` tool is also available. "You don't have to move to v2.x today, tomorrow, or ever."
- Source for the four points above: [dbt blog: dbt core v2 is here](https://docs.getdbt.com/blog/dbt-core-v2-is-here)
- Licensing page: dbt v1 is Apache 2.0, and "dbt OSS ... the Apache 2.0 open-source distribution for v2 ... is now available in `dbt-oss`". The `dbt` v2 distribution "is proprietary to dbt Labs, made available under the dbt Product Licensing Agreement", and "all [distributions are] free to install and run". The dbt platform is a separate hosted product — [dbt licensing](https://docs.getdbt.com/docs/dbt-licensing)
- **Dates conflict.** The blog fetch reported the alpha as 1 Jun 2026, while a third-party digest says July 2026 alpha and September 2026 GA. The 2.0.8 tag (14 Oct 2026) confirms v2 is released — [dbt blog](https://docs.getdbt.com/blog/dbt-core-v2-is-here); [releases.sh digest](https://releases.sh/collections/data-engineering/digest/2026-09-14)
- Merger: Fivetran and dbt Labs completed an all-stock merger, announced 13 Oct 2025 and closed 1 Jun 2026. George Fraser is CEO and Tristan Handy is President. TechTarget reports dbt Core v2.0 open-sources the Fusion runtime under Apache 2.0 — [Fivetran press](https://www.fivetran.com/press/fivetran-dbt-labs-complete-merger-to-create-the-data-infrastructure-for-trusted-ai-agents); [TechTarget](https://www.techtarget.com/data-technologies/news/366643590/Fivetran-DBT-Labs-complete-merger-to-form-data-layer-for-AI)

**Data tests, severity and failure storage**
- `severity` is `error` (default) or `warn`. `error_if` and `warn_if` default to `!=0`.
- With `severity: error`, `error_if` is checked first and then `warn_if`. With `severity: warn`, `error_if` is skipped.
- Conditions compare an integer failure count, e.g. `">1000"` or `between 5 and 10`. `--warn-error` and `--warn-error-options` promote warnings to errors.
- Configs can be set per test, in singular test SQL (`{{ config(error_if='>50') }}`), or project-wide (`data_tests: +severity: warn`).
- Source for the four points above: [dbt severity](https://docs.getdbt.com/reference/resource-configs/severity)
- `store_failures` saves failing records (up to `limit`) "in a new table with the name of the test". Results "always replace previous failures", which means there is no history.
- The default schema is `{{ profile.schema }}_dbt_test__audit`. A config value of true or false overrides the `--store-failures` flag, and `store_failures_as` overrides `store_failures: false`.
- Source for the two points above: [dbt store_failures](https://docs.getdbt.com/reference/resource-configs/store_failures)

**Unit tests**
- Unit tests have been available since v1.8. They are YAML `unit_tests:` with `model`, `given` (inputs as `dict`, `csv` or `sql`; fixtures in `tests/fixtures/`) and `expect`.
- They support SQL models only, in the current project only. Materialized views, recursive SQL and introspective queries are not supported.
- They cannot verify incremental merge behaviour.
- dbt "strongly recommends only running unit tests in development or CI". Exclude them in prod with `--exclude-resource-type` or `DBT_ENGINE_EXCLUDE_RESOURCE_TYPES` (v1.11+), or select only them with `dbt test --select "test_type:unit"`.
- Source for the four points above: [dbt unit tests](https://docs.getdbt.com/docs/build/unit-tests)

**Model contracts**
- `contract: enforced: true` requires every column's `name` and `data_type` to match the output, plus `constraints` "as supported for this materialization and data platform". This is a preflight check at compile time, and a mismatch raises a compilation error so the model is not built.
- Supported on `table`, `view` (names and types only), and `incremental` (only with `on_schema_change: append_new_columns`/`fail`). Not supported on Python models, materialized views, ephemeral models, sources, seeds or snapshots.
- Source for the two points above: [dbt contracts](https://docs.getdbt.com/reference/resource-configs/contract)

**Artifacts**
- `run_results.json` (schema v6) contains `metadata`, `args`, `elapsed_time` and `results[]`.
- Each result has `unique_id`, `status`, `thread_id`, `execution_time`, `timing[]` (compile/execute), `message`, `failures`, `adapter_response`, `compiled_code` and `relation_name`.
- It is produced by `build`, `test`, `run`, `seed`, `snapshot` and other commands.
- `dbt freshness` writes `freshness.json`, with `sources.json` kept for backward compatibility.
- Source for the four points above: [run_results.json](https://docs.getdbt.com/reference/artifacts/run-results-json)

**Elementary (DQ package)**
- Elementary OSS (Apache-2.0) is a dbt package plus a CLI. The package collects run and test metadata into the warehouse; the CLI produces an observability report and sends Slack/Teams alerts.
- Anomaly detection runs as native dbt tests, with freshness, volume and schema monitors, lineage enriched with test results, and an artifacts uploader.
- Elementary Cloud (paid) adds automated ML monitoring, column-level lineage from ingestion to BI, a catalog and AI agents.
- Source for the three points above: [GitHub elementary-data/elementary](https://github.com/elementary-data/elementary)

### Inferences
- **Pattern to copy:** "a test is a SELECT that returns bad rows". The failure count is compared against `warn_if`/`error_if`, failing rows are optionally kept in an audit schema, and status is machine-readable in run artifacts. This is the simplest, most portable rules-as-code model, and it lives in the same Git repo as the transformations.
- **What dbt alone does not do:**
  - It has no history of results. `run_results.json` is per invocation, and `store_failures` replaces previous failures.
  - It has no anomaly detection or ML, no alerting, and no monitoring of tables outside dbt beyond source freshness.
  - It has no scheduler outside the dbt platform or an external orchestrator.
- Elementary (or a custom artifact uploader) fills the history, anomaly and alert gaps. The v2 Parquet artifacts make a custom results warehouse easier to build: load them into the lakehouse and query them with DuckDB, Trino or BigQuery.
- **Licensing for Globe:** `dbt-oss` v2 and dbt v1 are Apache-2.0 and safe to embed. The default `dbt` binary is proprietary (free), so pin to `dbt-oss` if license purity matters. The Fivetran ownership adds roadmap uncertainty: commercial features are likely to keep moving into the platform.
- **Unchanged since earlier versions (from training knowledge, not re-verified this session):** dbt-expectations (now maintained by Metaplane after Calogica handed it over in 2024) ports GX-style tests as dbt macros. dbt Explorer was renamed "Catalog", and the dbt platform offers data health tiles/signals and the Semantic Layer. These need verification.

### Gaps
- An authoritative feature split of `dbt-oss` vs. the proprietary `dbt` binary was not fetched: whether column-level lineage, static SQL analysis/type-checking, state-aware orchestration and LSP features are open or premium. See the linked "dbt-availability" page.
- v2 Parquet artifact schemas, and any v2 changes to test semantics, were not reviewed.
- The current state of dbt-expectations/Metaplane, dbt Catalog health tiles and the Semantic Layer was not verified in this session.
- Techzine reports Fivetran + dbt Labs released "dbt v2 and dbt State". What "dbt State" is was not verified — [Techzine](https://www.techzine.eu/news/analytics/144381/fivetran-and-dbt-labs-release-dbt-v2-and-dbt-state/)

---

## Q4. Common patterns worth copying, and what each tool does not do (cross-tool synthesis)

### Takeaway
All three converge on the same patterns:
- Declarative YAML (or Python-object) rule definitions under Git.
- Metrics computed in-database through SQL pushdown (dialect Jinja templates, SQLAlchemy compilation, or plain SELECTs).
- Severity tiers on a numeric result.
- A machine-readable result record per check execution.

They differ most in result history and anomaly detection. Only DQOps natively stores a time series of readouts (Parquet), which is what anomaly rules need. GX Core and dbt require you to build or buy that layer.

### Cited Findings

**Rules-as-code**
- DQOps: YAML per table, validated by a JSON schema — [DQOps](https://dqops.com/docs/dqo-concepts/definition-of-data-quality-checks/)
- GX: suites are stored in the `gx/` file context and built from Expectation classes — [GX migration](https://docs.greatexpectations.io/docs/reference/learn/migration_guide)
- dbt: YAML `data_tests`, `unit_tests` and `contract` blocks — [dbt severity](https://docs.getdbt.com/reference/resource-configs/severity); [contracts](https://docs.getdbt.com/reference/resource-configs/contract)

**SQL pushdown**
- DQOps: per-dialect Jinja2 templates returning `actual_value` — [repo](https://github.com/dqops/dqo/tree/develop/home/sensors)
- GX: SQLAlchemy/Spark/Pandas execution engines with metric providers — [repo](https://github.com/great-expectations/great_expectations/tree/develop/great_expectations/execution_engine)
- dbt: the test is a SQL SELECT whose failure count is evaluated — [dbt severity](https://docs.getdbt.com/reference/resource-configs/severity)

**Severity models**
- DQOps: warning/error/fatal, with warnings excluded from KPIs — [DQOps](https://dqops.com/docs/dqo-concepts/definition-of-data-quality-checks/)
- dbt: warn/error with count thresholds — [dbt](https://docs.getdbt.com/reference/resource-configs/severity)
- GX Cloud: auto-generated expectations default to warning severity — [GX Cloud](https://docs.greatexpectations.io/docs/cloud/expectations/manage_expectations/)

**Result storage**
- DQOps: `sensor_readouts`/`check_results`/`errors`/`incidents`/`statistics`/`error_samples` Parquet, partitioned `c=/t=/m=`, with an idempotent hash `id` — [DQOps storage](https://dqops.com/docs/dqo-concepts/data-storage-of-data-quality-results/)
- dbt: `run_results.json` plus an audit schema for failing rows, and Parquet artifacts in v2 — [run_results](https://docs.getdbt.com/reference/artifacts/run-results-json); [repo README](https://github.com/dbt-labs/dbt-core)
- GX: Validation Results rendered to Data Docs — [GX overview](https://docs.greatexpectations.io/docs/core/introduction/gx_overview)

**Orchestration**
- DQOps: built-in scheduler plus Airflow operators — [DQOps architecture](https://dqops.com/docs/dqo-concepts/architecture/dqops-architecture/)
- GX Core: Checkpoints run from your orchestrator — [GX overview](https://docs.greatexpectations.io/docs/core/introduction/gx_overview)
- dbt: `dbt build`/`test` from an orchestrator or the dbt platform — [run_results](https://docs.getdbt.com/reference/artifacts/run-results-json)

**Alerting**
- DQOps: Slack/webhook notifications and incidents — [GitHub](https://github.com/dqops/dqo)
- GX: Checkpoint Actions (email, Slack, Teams) — [GX overview](https://docs.greatexpectations.io/docs/core/introduction/gx_overview)
- dbt: none natively; Elementary adds Slack/Teams — [Elementary](https://github.com/elementary-data/elementary)

**Anomaly detection**
- DQOps OSS: Z-score/percentile on 90-day windows; seasonality only in the commercial version — [DQOps](https://dqops.com/docs/categories-of-data-quality-checks/how-to-detect-anomaly-data-quality-issues/)
- GX: Cloud only (volume, completeness, schema; forecasting-based) — [GX Cloud](https://docs.greatexpectations.io/docs/cloud/expectations/manage_expectations/); [status](https://status.greatexpectations.io/incidents/wk7xnv9l448h)
- dbt: none natively; Elementary has anomaly tests — [Elementary](https://github.com/elementary-data/elementary)

**Licenses (as of Oct 2026)**
- DQOps: BSL 1.1 with a non-commercial embedding restriction — [LICENSE](https://github.com/dqops/dqo/blob/develop/LICENSE)
- GX Core: Apache-2.0 — [GitHub](https://github.com/great-expectations/great_expectations)
- dbt: v1 and `dbt-oss` v2 are Apache-2.0; the `dbt` v2 binary is proprietary — [dbt licensing](https://docs.getdbt.com/docs/dbt-licensing)
- Elementary OSS: Apache-2.0 — [Elementary](https://github.com/elementary-data/elementary)

### Inferences
**Recommended reference design for an in-house Globe tool (synthesis):**
1. Check definitions in YAML under Git, validated by a JSON schema (DQOps / dbt style).
2. Metric "sensors" as per-dialect Jinja SQL templates that return `actual_value`, plus optional grouping (data_group) and time-partition columns, all pushed down to BigQuery, Hive or Trino (DQOps pattern).
3. Stateless Python "rules" that receive (actual, params, history) and return (passed, expected, lower, upper). This keeps thresholds and anomaly models swappable (DQOps pattern), so seasonal models can be plugged in where DQOps paywalls them.
4. warning/error/fatal severities, with fatal able to block pipelines and warnings excluded from KPI scoring.
5. An append-only results store (readouts, check results, errors, incidents, error samples) in Parquet/Iceberg, partitioned by source/table/month, with a deterministic hash `id` for idempotent reruns.
6. Partition-level (incremental) checks keyed by date, for large telco fact tables.
7. Integrate rather than rebuild for transformation-layer tests: ingest dbt `run_results.json` or the v2 Parquet artifacts (and GX Validation Results) into the same results store, so dashboards and alerts see one unified history.

**Embed vs. build:**
- GX Core (Apache-2.0) is embeddable as a validation library.
- dbt `dbt-oss` (Apache-2.0) is embeddable for transformation-time tests.
- DQOps should be treated as a design reference, not an embeddable component, because of BSL clause (ii) and low project activity.

### Gaps
- No independent 2026 benchmark comparing the three on performance or scale was found.
- Community comparison articles from 2024-2026 were not reviewed because of the tool-call budget. The synthesis above rests on primary docs and source.
