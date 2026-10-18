# Monte Carlo (montecarlo.ai): how the data and AI observability product works, as of October 2026

> **Raw research notes, not recommendations.** Evidence gathered on 2026-10-18 for the reports in `research-outputs/`. Vendor claims are marked as such; screenshots described here were not committed. Use these to check a claim's source, never as a decision or a plan.


Scope and method notes:
- The main sources are Monte Carlo's own docs at docs.getmontecarlo.com. I pulled each page as raw markdown through the site's `llms.txt` index. Most pages carry `updatedAt` stamps from June to October 2026, and I flag older pages where they appear. I also used the public GitHub org (monte-carlo-data), the pricing page, G2 summaries and 2025 press coverage.
- I inspected the product tour with REA (`capture_browser_scenario`, launch mode, headless Chrome, public pages only). I submitted no forms and did not log in. The tour was not gated.
  - Evidence ID `ev_c48e6dc17c1c9fd8c4320f3ad5781b9ca14c99b2f6aa1b178b3b6189931ccced`: the info.montecarlo.ai landing page, with its network and frame events.
  - Evidence ID `ev_c73f04f3a82c190ea78f73a20c0317d27c4f5ed7fc6ac2b7d35a04b573e7faa4`: the Walnut player, with response bodies, including the GraphQL `demoSettings` payload that holds all of the tour's step text.
  - I then downloaded the public screen snapshots and screenshot PNGs with curl/urllib from the asset URLs that REA observed. That download step is not REA evidence.
- "Marketing claim" means a statement made on montecarlo.ai, in press, or in tour narration. "Docs" means behaviour described in product documentation.

## 1. Deployment architecture, what data leaves the customer environment, and credentials

### Takeaway
Monte Carlo is a multi-tenant SaaS control plane that runs on AWS. You can optionally move two pieces into your own cloud:
- A **Data Store**: an object-storage bucket that holds row-level samples.
- An **Agent**: a thin, egress-only executor that runs commands against your sources and holds their credentials.

Even in the hybrid models, metadata, metrics, query logs and aggregated statistics always go to Monte Carlo's cloud. Only row samples, and AI-agent trace content, can be kept entirely in the customer's environment.

### Cited Findings
- **Three components.**
  - The "Cloud Service": UI, monitoring system, dashboards and notification routing.
  - The "Data Store": an "Object storage bucket for a small sample of individual values or data records … associated with a data reliability alert/incident".
  - The "Agent": "a bridge between the Monte Carlo Cloud Service and customer resources".
  - Source: [Architecture](https://docs.getmontecarlo.com/docs/platform-architecture) (updated 2026-06-25).
- **Three deployment models.**
  - (1) Cloud/SaaS: a customer-dedicated object store lives in Monte Carlo's cloud.
  - (2) Hybrid, cloud with a customer-hosted Data Store, so you "never store record-level data with Monte Carlo". Available on AWS S3, GCS and Azure Blob.
  - (3) Hybrid, customer-hosted Agent and Data Store: "the most control but requires the most maintenance". On AWS this is Lambda plus S3, on GCP Cloud Run plus GCS, and on Azure a Function plus a Storage Account.
  - Monte Carlo recommends options 1 or 2.
  - Source: [Architecture](https://docs.getmontecarlo.com/docs/platform-architecture).
- **What always leaves the environment.** "Metadata, Metrics, Query logs, and Aggregated / anonymized statistics are always stored in Monte Carlo's Cloud." Troubleshooting data (samples) can be switched off per integration. Source: [Architecture](https://docs.getmontecarlo.com/docs/platform-architecture).
- **Data inventory by type** ([Information Monte Carlo Collects](https://docs.getmontecarlo.com/docs/info-monte-carlo-collects), updated 2026-10-14):
  - Metadata: tables, schemas, freshness/volume, BI objects.
  - Metrics: row counts, byte counts, last-modified time.
  - Query logs: "History of queries, as well as metadata about them (timestamp, user performing the query, errors…)", used for lineage and usage.
  - Aggregated statistics: null rates, distinct values, percentiles.
  - Unstructured-data stats.
  - Application data.
  - Agent span metadata and evaluation scores.
  - All of the above are stored in the "Cloud service".
  - Data sampling ("individual values or data records … in clear text form") goes to the Data Store.
  - Agent trace content (prompts and completions) stays in the "Customer environment".
- **Privacy caveat.** Monte Carlo acknowledges that "the service may ingest and process personal data as part of query logs or through other data sampling". It "strongly encourages customers to mask or anonymize personal data at the data source". Source: [Privacy](https://docs.getmontecarlo.com/docs/privacy).
- **Features that depend on sampling.** Turning sampling off disables all of these: value-based SQL rules, `{{query_result:field}}` parameters in notifications, "Test your SQL", previews of validation sets, the Monitoring Agent's "advanced recommendations", RCA follow-up queries, breached-row viewing, metric investigation, and the Data Profiler's common values. Source: [Data Sampling](https://docs.getmontecarlo.com/docs/data-sampling).
- **Generic Agent design** ([Generic Agent Deployment](https://docs.getmontecarlo.com/docs/generic-agent-platforms), updated 2026-09-11, preview, needs Enterprise plus the Advanced Networking add-on):
  - Egress-only. The agent opens a persistent HTTPS Server-Sent-Events stream to the "MC Agent Service" (for example `artemis.getmontecarlo.com`), receives operations, and pushes results back over HTTPS. Polling is the fallback.
  - The Agent Service forwards results to the "MC Collection Service" for processing.
  - It runs on Docker Compose, Kubernetes, EKS, AKS or GKE, including on-premises.
  - Agent authentication is either a Key/Token pair (`mcd_id`/`mcd_token`) or OAuth2 client credentials with 1-hour JWTs.
  - Optional controls: IP allowlist, AWS/Azure PrivateLink, a forward proxy, and TLS-inspection support.
  - Source code is public at github.com/monte-carlo-data/hermes-agent.
- **Older cloud-native agent (Apollo).** It is a Flask service that receives a credentials block plus an "operation" made of chained client-method "commands". The README example is the BigQuery call `projects().list(maxResults=100).execute()`, which the agent executes and returns.
  - Integration modules in the repo: bigquery, snowflake, redshift, databricks, db, looker, tableau, powerbi, git, s3/gcs/azure_blob, otlp, mcp, custom, custom_bi, custom_etl.
  - Source: [apollo-agent README](https://github.com/monte-carlo-data/apollo-agent) and its `apollo/integrations` directory.
- **Credentials.**
  - Default flow: create "a read-only service account" on the warehouse and "Provide service account credentials on your Monte Carlo onboarding wizard". In other words, Monte Carlo stores the credentials. Source: [Data Warehouses](https://docs.getmontecarlo.com/docs/overview-data-warehouses).
  - With an agent, "Self Hosted Credentials" can be used instead. These live in AWS Secrets Manager or env vars (KMS recommended), GCP Secret Manager, Azure Key Vault, or files on Kubernetes/Docker, and the agent reads them at runtime. Source: [Self Hosted Credentials](https://docs.getmontecarlo.com/docs/self-hosted-credentials).
  - On the Generic Agent, "credentials remain on your infrastructure". The secret managers are reached through the External Secrets Operator or read directly. Source: [Secrets Management](https://docs.getmontecarlo.com/docs/secrets-management).
- **Collection scope is controlled at the warehouse.** To exclude schemas, "remove the service account's access to those schemas at the database level". Source: [Data Warehouses](https://docs.getmontecarlo.com/docs/overview-data-warehouses).
- **Hybrid deployments need a paid tier.** They require Scale or Enterprise plus an Enterprise Support Plan. Accounts created after 2024-04-24 are on "Platform V2". The deployment types are CLOUD_V1/V2 and REMOTE_V1/V1.5/V2, where "REMOTE_V1" is the "Legacy hybrid deployment (data collector)". Source: [Deployments](https://docs.getmontecarlo.com/docs/deployment-and-connecting).
- **Platform infrastructure.** AWS Lambda for serverless compute, API Gateway, WAF, private VPC endpoints, logically isolated tenants. SOC 2 Type II and ISO 27001. Source: [Infrastructure Security](https://docs.getmontecarlo.com/docs/infrastructure-security).
- **Regions.** Multi-tenant regional instances exist in the **US** (primary release region) and the **EU**. Source: [Regional Hosting](https://docs.getmontecarlo.com/docs/regional-hosting).
- **Business Critical tier.** It adds a "Dedicated Instance" and cross-region disaster recovery. Source: [Pricing](https://montecarlo.ai/pricing).
- **Agent observability data plane.** Traces are stored in your own cloud. An OpenTelemetry Collector plus ClickHouse run on EKS, AKS or GKE, deployed with Terraform, and evaluations use your own Bedrock, Foundry or Vertex AI. Monte Carlo "receives span metadata and evaluation scores" and queries content on demand. Source: [Agent Observability Overview](https://docs.getmontecarlo.com/docs/agent-monitors-overview).

### Inferences
- The architecture separates a **control plane** (SaaS: scheduling, ML, metadata store, UI) from a **data-plane executor** (the agent). The executor is deliberately "dumb": it runs the commands and SQL it is sent and returns results, and processing stays in Monte Carlo's cloud. For an in-house tool at Globe, the same split works well even when everything is internal. Keep a central metadata and metrics store and a scheduler, plus lightweight per-source executors that hold the credentials.
- Only US and EU regions are documented, with no APAC region. A Philippine telco subject to the Data Privacy Act (RA 10173) would therefore send query logs to a foreign region. Query text can embed subscriber identifiers such as MSISDNs in WHERE clauses, and the logs also carry user names. An in-house tool avoids that exposure. If Monte Carlo were evaluated, the customer-hosted Data Store and agent would only keep *samples* local, not logs or metadata.
- Design lesson: list explicitly which features need row-level access (sampling) and make each one toggleable per source. Monte Carlo's per-integration sampling opt-out and its "count-based vs value-based" SQL rule distinction are good patterns to copy.

### Gaps
- The Data Store retention period and sample sizes were not found on the pages I read.
- No current statement says exactly how Monte Carlo encrypts cloud-stored credentials (for example a KMS-per-tenant scheme). The application-security page did not mention credentials.
- Whether an APAC or Singapore region is on the roadmap: no source found.

## 2. How monitoring works: monitor types, thresholds, and metadata vs query cost

### Takeaway
There are five monitor families: Table, Metric, Validation, Job and Agent.
- **Table monitors** (freshness, volume, schema) run off hourly metadata and query-log collection. They cost the warehouse almost nothing and use ML thresholds that need about 7 to 14 days of history.
- **Metric, validation, custom-SQL and comparison monitors** run SQL in the customer's warehouse on a schedule or trigger. They support ML ("dynamic") or fixed thresholds, segmentation, and backfill.
- Threshold tuning works through sensitivity (low/medium/high), "mark as normal", and training-data exclusion. Alert status itself does **not** feed the models.
- Everything can be declared as YAML (monitors as code) and applied by the CLI with replace (PUT) semantics.

### Cited Findings
- **Families** ([Monitors overview](https://docs.getmontecarlo.com/docs/monitors-overview)):
  - Table: patterns of updates, size and growth. Mostly metadata, and they "start working immediately without user configuration".
  - Metric: field statistics or custom metrics, segment anomalies. Query-based.
  - Validation: row-level rules, business logic, cross-source comparisons. Query-based.
  - Job: query runtime and performance, plus alerts ingested from Airflow, dbt, ADF and Databricks.
  - Agent: trace-based quality and operational metrics.
- **Monitor metadata.** Every monitor carries Priority, Audience, Tags and a "Data quality dimension" (Accuracy, Completeness, Consistency, Timeliness, Validity, Uniqueness). Accounts can make these fields mandatory through the API (`updateRequiredMonitorFields`). Source: [Monitors overview](https://docs.getmontecarlo.com/docs/monitors-overview).
- **Collection cadence per integration** ([Data Collection details](https://docs.getmontecarlo.com/docs/data-collection-details-per-integration), page dated 2025-07-02):
  - Snowflake: metadata hourly from information_schema, query logs hourly from QUERY_HISTORY.
  - Redshift: metadata hourly from SVV views, query logs every 10 minutes from STL, and freshness is "Calculated from query logs, based on queries that are deemed to update tables".
  - BigQuery and Databricks: hourly.
  - Teradata: query logs every 15 minutes, and freshness also comes from logs.
  - Postgres, MySQL and Oracle: metadata every 12 hours, with no freshness or volume.
  - Looker API: every 4 days because of API limits.
- **Table monitor ML** ([Table / pipeline observability](https://docs.getmontecarlo.com/docs/pipeline-observability), updated 2026-10-07):
  - Thresholds usually appear "after 7 days of observing a table" and are "mature by 14 days".
  - Training uses at most "a 6-week rolling window … including handling of weekly seasonality".
  - Schema change monitors involve "no machine learning" and "compare the schema of the table hour-over-hour" ([Schema Change](https://docs.getmontecarlo.com/docs/schema-change)).
- **Table monitor compute cost.** Table monitors "use metadata instead of querying the table, keeping compute costs minimal". Where metadata row counts don't exist (views, some integrations), optional row-count collection runs `COUNT(*)` on "each qualifying asset … each hour", which "can increase warehouse compute costs". Sources: [Table Monitor MaC](https://docs.getmontecarlo.com/docs/table); [pipeline-observability](https://docs.getmontecarlo.com/docs/pipeline-observability).
- **Table monitor YAML.** It selects assets by database, schema, name pattern (STARTS_WITH, ENDS_WITH, CONTAINS, MATCH_PATTERN), tags (HAS_ANY/HAS_ALL), type, plus exclusions.
  - Metrics: `last_updated_on`, `total_row_count`, `total_row_count_last_changed_on`, `schema`, `schema_fields_added`, `schema_fields_removed`, `schema_fields_type_change`.
  - The only tuning lever is `sensitivity`. A `domains` field is required.
  - Coverage is re-evaluated on every run, so new tables are picked up automatically.
  - Sources: [Table Monitor](https://docs.getmontecarlo.com/docs/table); [pipeline-observability](https://docs.getmontecarlo.com/docs/pipeline-observability).
- **Freshness.**
  - Two signals: "Time since last update" and "Time since last row count change".
  - Explicit thresholds can be "Periodically" (for example, at least every 6 hours, checked every 3 hours), "On a schedule" (for example, check at 07:55 for an 08:00 SLA), or CRON.
  - Freshness is managed per table from Assets.
  - Source: [Freshness](https://docs.getmontecarlo.com/docs/freshness).
- **Volume.** Volume alerts are subtyped as size increase, size decrease, and unchanged size. Source: [Volume](https://docs.getmontecarlo.com/docs/volume).
- **Anomaly detection architecture.** "an ensemble of anomaly detection models", with "Model selection … based on patterns observed in that series". Both are managed by Monte Carlo. Training data on "a rolling window of recent data" and sensitivity are the two user-controlled parts. Source: [Anomaly Detection](https://docs.getmontecarlo.com/docs/anomaly-detection-overview).
- **Feedback loop** ([Tuning thresholds](https://docs.getmontecarlo.com/docs/tuning-thresholds)):
  - Anomalies are auto-excluded from training. "Mark as normal" adds them back and widens the threshold.
  - Users can exclude chart periods, and "exclusion windows" (one-off or recurring holidays) can apply at warehouse, database, schema or table scope.
  - Changes take "several hours" to show.
  - Auto-learning of recurring anomalies: "a daily Metric monitor usually needs to observe about 4 anomalous days within a 7-day period before treating the pattern as normal".
- **Status is not feedback.** "Alert statuses **do not** provide feedback to the models that generate thresholds." Source: [Alert Statuses](https://docs.getmontecarlo.com/docs/incident-statuses).
- **Metric monitors** ([Metric Monitors](https://docs.getmontecarlo.com/docs/metric-monitors), updated 2026-09-16):
  - Source: a table, or custom SQL as the source.
  - Options: WHERE filter; time-field bucketing (hour, day, week or month), with an option to ignore recent immature buckets; a time zone picker; up to 5 segment fields or a SQL segment expression; and a minimum segment size.
  - Backfill of history speeds up ML training. A "Track only" mode exists.
  - Schedule options: fixed schedule, manual or API trigger, "When a table is updated" (within 30 minutes of the update, at most every 2 hours, or 12 hours for high-segment monitors), or "When a job or task completes", with a 48-hour fallback.
- **Metric monitor limits and threshold behaviour.**
  - Up to 10,000 metric × field × segment combinations per monitor, raisable to 1,000,000.
  - Multi-table metric monitors cover up to 1,000 tables, with field pattern matching (for example, all string fields ending in `_id` for null %).
  - Null% and Unique% thresholds are "optimized for the extremes": very sensitive near 0 or 100 %, insensitive when volatile.
  - Sources: [Metric Monitors](https://docs.getmontecarlo.com/docs/metric-monitors); [Segmentation](https://docs.getmontecarlo.com/docs/segmentation).
- **Available metrics** ([Available metrics](https://docs.getmontecarlo.com/docs/available-metrics)):
  - Row count and relative row count.
  - Unique and duplicate counts.
  - Null, empty string, all-spaces, NaN, and "none"/"null" keyword (% and count).
  - Mean, median, min, max, percentiles (20/40/60/80/95/99), zero/negative, standard deviation, sum, true/false.
  - String length statistics and format checks (integer, float, UUID, email, ISO timestamp, US SSN/phone/state/ZIP).
  - In past/in future, Unix time 0.
  - Model-accuracy metrics: RMSE, MAE, MAPE.
- **Custom SQL monitor** ([Custom SQL](https://docs.getmontecarlo.com/docs/custom-sql), updated 2026-09-26):
  - The query must return exactly one row with one numeric column.
  - Threshold types: static `threshold`, a range, `dynamic_threshold` (ML; AUTO/AUTO_HIGH/AUTO_LOW), or `change` (compared with the previous run). The UI calls these "Automatic", "Absolute" and "Relative".
  - Noise controls: `event_rollup_count`, `event_rollup_until_changed`, `alert_grouping`.
- **Validation monitors.** These are row-level predicate trees (UNARY, BINARY, SQL, GROUP AND/OR, with `negated`), where the conditions "define INVALID data". Source: [Validation Monitor](https://docs.getmontecarlo.com/docs/validation).
- **Comparison (reconciliation) monitor.** `metric_comparison` checks a source table against a target table, which can be in different warehouses. Example: ROW_COUNT `comparison_delta` GT 100. Up to 5,000 combinations and one segment field. Source: [Comparison Monitor](https://docs.getmontecarlo.com/docs/metric-comparison); [Segmentation](https://docs.getmontecarlo.com/docs/segmentation).
- **Deprecated monitor names.** "Dimension tracking", "field_health", "field_quality", freshness/volume rules and the legacy "comparison" are deprecated in monitors as code. Field Health and Field Quality Rules were merged into Metric monitors in April 2024, and field_quality was "Auto-converted to custom_sql as of August 2025". Sources: [Monitors as Code](https://docs.getmontecarlo.com/docs/monitors-as-code); [Metric Monitors](https://docs.getmontecarlo.com/docs/metric-monitors). If older write-ups mention "dimension tracking" or "field health", treat them as **outdated**.
- **Monitors as code** ([Monitors as Code](https://docs.getmontecarlo.com/docs/monitors-as-code), updated 2026-07-30):
  - Project setup: `montecarlo.yml` with `version`, `default_resource`, `default_domain` and `namespace`.
  - YAML keys: `table`, `json_schema`, `metric`, `metric_comparison`, `bulk_monitor`, `validation`, `custom_sql`, `query_performance`, `agent_metric`, `agent_evaluation`, `agent_trajectory`, `agent_validation`.
  - Monitors can be embedded in dbt `schema.yml` under `meta`, with `[[ ref() ]]` resolved from the dbt manifest. Monitors can also be generated from existing dbt tests.
  - Commands: `montecarlo monitors compile` and `apply --namespace [--dry-run]`. "PUT semantics — updates replace the entire definition."
  - Renaming a monitor creates a new one and drops its incident history ([Table](https://docs.getmontecarlo.com/docs/table)).
  - The dry-run plan shows "an estimated daily credit cost per created or updated monitor".
- **CI/CD.** `pip install 'montecarlodata>=0.167.0'`, with API keys in `MCD_DEFAULT_API_ID` and `MCD_DEFAULT_API_TOKEN`. Examples cover GitHub Actions, GitLab and CircleCI. Source: [CI/CD Integration](https://docs.getmontecarlo.com/docs/ci-cd-integration).
- **Compute options.** Several query connections of the same type (for example, Snowflake warehouses of different sizes) can be registered to tune cost and performance for heavy custom monitors. Source: [Data Warehouses FAQ](https://docs.getmontecarlo.com/docs/overview-data-warehouses).
- **LLM transformations of text columns.** Metric monitors can apply sentiment, classification or custom-prompt transforms using the warehouse's native AI functions (Snowflake, Databricks, BigQuery with `gemini-2.5-flash`), with row sampling caps. Source: [Metric Monitors](https://docs.getmontecarlo.com/docs/metric-monitors).

### Inferences
- A useful model to copy is two tiers of checks with different cost profiles:
  - (a) Near-free, broad "pipeline health" from catalog metadata and query history (freshness, volume, schema) on every table, with ML thresholds.
  - (b) Targeted, query-based deep checks (metrics, rules, reconciliation) on critical tables only, with explicit schedules or triggers and caps on segments and rows.
  - Globe's warehouse (BigQuery, Teradata, etc.) system views can supply tier (a) cheaply.
- The threshold engine appears to be a forecasting-model ensemble with weekly seasonality, a rolling window of at most 6 weeks, auto-exclusion of anomalies, and user re-inclusion. Monte Carlo's GitHub org has a fork of Nixtla `statsforecast` ([monte-carlo-data/statsforecast](https://github.com/monte-carlo-data/statsforecast)), which hints that statistical forecasting models are used. That is an inference, not something the docs state.
- Separating *triage status* from *model feedback* is a deliberate design choice. It stops "no action needed" clicks from silently desensitising thresholds. An in-house tool should use an explicit "mark as normal / exclude period" action instead.
- Hard-won details worth copying: ignoring immature trailing buckets, time zone handling for naive timestamps, segment-cardinality limits plus a "Test" step, trigger-on-table-update with rate limits, and a fallback schedule.

### Gaps
- The specific model families in the ensemble and the exact sensitivity math (for example, prediction-interval width) are not published.
- Per-monitor-type credit consumption rates exist in docs Monte Carlo links from its order forms, but I did not find them in the public llms.txt index.

## 3. Lineage, incident workflow, data products and domains, SLOs

### Takeaway
- Table and field lineage is derived mainly by parsing warehouse **query logs**. It is enriched by push or pull integrations (dbt, Airflow, ADF, BI tools via APIs or via the warehouse query logs) and exposed through a GraphQL API keyed by "MCON" identifiers.
- Alerts follow a lifecycle with grouping by schema, time window and lineage. Users can mark an alert as an incident with a severity, track TTA/TTR SLO policies, and route alerts through "audiences" to Slack, Teams, PagerDuty, Jira, ServiceNow and others.
- Data Products (downstream assets plus their automatic upstream lineage) and Domains (RBAC-scoped ownership) organise coverage and reporting.

### Cited Findings
- **Query logs drive lineage.** Query logs are collected to "Track lineage, usage analytics and query history". Source: [Information Monte Carlo Collects](https://docs.getmontecarlo.com/docs/info-monte-carlo-collects).
- **BI metadata via query logs.** Mode, Periscope/Sisense and Sigma metadata "is extracted from the query logs of your data warehouse". Source: [Data Collection details](https://docs.getmontecarlo.com/docs/data-collection-details-per-integration).
- **Push vs pull integrations.** dbt Core, Airflow and Prefect push metadata on each run, while dbt Cloud, Fivetran, Informatica, ADF, Tableau and Power BI are pulled through their APIs. Source: [Data Collection details](https://docs.getmontecarlo.com/docs/data-collection-details-per-integration).
- **Field lineage** ([Field Lineage](https://docs.getmontecarlo.com/docs/field-lineage), updated 2026-06-25, though its limitations text references an old "data-collector v2530"):
  - Two kinds of relationships: "SELECT clause lineage" (field to field) and "Non SELECT lineage" (WHERE and similar clauses, field to table).
  - Limitations: wildcard tables, `JOIN … USING` without prefixes, and nested fields are not covered. BI field lineage is mainly Tableau.
  - The page advises using table prefixes and aliases and avoiding `SELECT *`. For `count(*)`, every column counts as a source.
- **Lineage API.** `getTableLineage(mcon, direction, hops)` returns `baseNode`, `connectedNodes` (with objectType values such as `table` and `looker-view`, plus `isCustom`) and `flattenedEdges`. Source: [Lineage API](https://docs.getmontecarlo.com/docs/lineage).
- **Alert feed and detail.** The feed filters on status, type, severity, priority, owner, table, schema, database, tag, audience and linked issues. The detail page shows impacted tables, downstream BI "Impact analysis" with their users, and the audiences notified. Source: [Interacting with Alerts](https://docs.getmontecarlo.com/docs/interacting-with-incidents) (updated 2026-09-09).
- **Grouping.**
  - Table-monitor alerts group when they come from the same monitor and the same schema within a 5-hour window.
  - Legacy grouping also grouped alerts "connected by lineage … in a 5-hour window".
  - dbt alerts group by run or by repetitive error.
  - Operational and data alerts are never mixed.
  - Source: [Interacting with Alerts](https://docs.getmontecarlo.com/docs/interacting-with-incidents).
- **Lifecycle.**
  - Statuses: No status → Acknowledged → Work in progress → Fixed, or Expected / No action needed.
  - "Mark as incident" asks for a name, a comment and a SEV-1 to SEV-4 severity based on an impact × stakeholder-count matrix. Alerts can be merged into an existing incident within ±7 days.
  - Recommended KPIs: incidents by severity, Time to Response, Time to Fixed.
  - Sources: [Marking Alerts as Incidents](https://docs.getmontecarlo.com/docs/marking-alerts-as-incidents); [Alert Statuses](https://docs.getmontecarlo.com/docs/incident-statuses).
- **RCA "insights"** ([RCA Insights](https://docs.getmontecarlo.com/docs/rca-detections), now "generally … moved into the Troubleshooting agent"; screenshots date from 2023):
  - Correlation insights look for categorical field values tied to a volume or metric anomaly, for example `status = pending`.
  - Query insights cover query change, failed query, and empty query.
  - They are built from "query logs, lineage information and the content of the table".
- **Notifications** ([Notifications](https://docs.getmontecarlo.com/docs/notifications-v2), updated 2026-08-14):
  - An "Audience" is a named group of recipients across Slack, Teams, Email, PagerDuty, Webex, OpsGenie, Jira, ServiceNow and webhooks.
  - Per-monitor audiences can be conditional on Triage Agent priority (High, Medium, Low, Not triaged). Some audience must cover "Not triaged".
  - When priority changes, notifications only escalate upward.
  - Failure audiences, for monitors that fail to run, are always immediate.
  - The integrations list also includes FireHydrant, Google Chat, Incident.io, Opsgenie and Webhooks ([llms.txt index](https://docs.getmontecarlo.com/llms.txt)).
- **SLOs.**
  - Policies cover TTA or TTR in hours, filtered by priority (P1, P2…) and by domain, with optional re-notification on breach.
  - The strictest policy wins. Breach detection can lag by up to 15 minutes.
  - Only domain-assigned monitors are supported.
  - Source: [SLOs](https://docs.getmontecarlo.com/docs/service-level-objectives-slos).
- **Data Products.**
  - You pick the most downstream tables or reports, and upstream lineage assets are included automatically and tagged with an auto-generated "Data product tag".
  - Table monitors can be enabled for the whole set, and new tables that join the lineage inherit the tag.
  - Tabs: Coverage (monitored, not monitored, not supported) and Data operations (incidents, TTR trends).
  - The Start tier is limited to 3 Data Products.
  - Source: [Data Products](https://docs.getmontecarlo.com/docs/using-data-product-dashboards).
- **Domains.** Authorization groups can be restricted to domains, and assets, monitors and alerts are domain-scoped. "No assignment does not mean visible to all." "Reference domains" give read-only cross-domain use, for example a shared `dim_customer`. Source: [Data Authorization — Domains](https://docs.getmontecarlo.com/docs/data-authorization).
- **Domains are now required on monitors.** "One domain per monitor" is effectively required on new accounts ([Monitors as Code](https://docs.getmontecarlo.com/docs/monitors-as-code)).

### Inferences
- A query-log SQL parser is the core lineage engine. For Globe, that means investing in a robust SQL parser for its warehouse dialects. Open-source options like sqlglot or OpenLineage-style emitters are my suggestion, not Monte Carlo's.
- Push hooks from orchestrators (Airflow, dbt run artifacts) let you attach job context to tables. The same lineage graph then powers alert grouping, impact analysis, RCA and data-product membership, so it should be treated as foundational infrastructure.
- The audience abstraction (monitor → audience → channels), combined with domain ownership, is a clean routing model. Priority-conditional routing and "escalate upward only" are good noise-control patterns.
- Operational KPIs (TTA/TTR, incidents by severity) only work if statuses are actually updated. Monte Carlo nudges this with SLOs and re-notification.

### Gaps
- There was no current doc on how table-level lineage resolves views, temp tables or cross-database lineage, and no coverage numbers for parser accuracy.
- I did not verify the exact Slack and Jira interaction set (for example, two-way status sync) from the integration pages.

## 4. AI features in 2025–2026: observability agents and agent/LLM monitoring

### Takeaway
- In 2025 Monte Carlo launched a Monitoring Agent (recommends monitors) and a Troubleshooting Agent (root cause through parallel hypotheses).
- By late 2026 the product has a fleet of agents: Operations (chat router), Triage (auto-scores every alert), Troubleshooting, Tuning (noise reduction, optional auto-apply), Cost, PII, PR/MC Prevent (code-change gate), plus an MCP server and Agent Toolkit for Claude Code and Cursor.
- In parallel, the company rebranded toward an "Agent Trust Platform". It now monitors customers' own AI agents through OpenTelemetry traces, LLM-as-judge or SQL evaluations, and trajectory monitors, with trace content kept in the customer's cloud.

### Cited Findings
- **2025 launch.** On April 17, 2025 Monte Carlo announced Observability Agents. The Monitoring Agent recommends rules and thresholds, deployable with one click, and the Troubleshooting Agent was slated for GA in Q2 2025. Vendor claims: an 80%+ reduction in resolution time and 30%+ deployment efficiency, neither independently verified. Sources: [BusinessWire](https://www.businesswire.com/news/home/20250417869977/en); [TechTarget](https://www.techtarget.com/searchdatamanagement/news/366622933/Monte-Carlo-launches-first-agents-for-data-observability); [BigDATAwire](https://www.bigdatawire.com/2025/04/17/monte-carlo-brings-ai-agents-into-the-data-observability-fold).
- **Monitoring Agent recommendations.** They come from "sampling data of up to 10k rows". Without sampling, only heuristic (non-AI) recommendations are made. Sources: [Metric Monitors](https://docs.getmontecarlo.com/docs/metric-monitors); [Data Sampling](https://docs.getmontecarlo.com/docs/data-sampling).
- **Troubleshooting Agent** ([Troubleshooting agent](https://docs.getmontecarlo.com/docs/troubleshooting-agent), updated 2026-10-07):
  - It works through "100s of hypothesis" across data changes, system issues (Airflow/dbt failures) and code changes (PRs), and "automatically traverse[s] lineage".
  - Hypothesis types: upstream row-count changes, modified, failed, missing, non-writing or new queries, job failures, validation failures, PRs, and data correlation (correlation is cloud deployments only).
  - It can be triggered from the alert page, a Slack button, or the Operations Agent.
  - Customer MCP servers can be connected to it.
  - Comparison alerts and merged alerts are not yet supported.
- **Triage Agent** ([Triage agent](https://docs.getmontecarlo.com/docs/triage-agent), updated 2026-08-31):
  - Runs automatically on every new alert.
  - Gathers context: anomaly data, failed queries, lineage impact, 60-day alert history, asset criticality, query changes, orchestrator issues.
  - Scores "incident likelihood" and "alert impact" as HIGH, MEDIUM or LOW, and can chain the Troubleshooting Agent.
  - Re-assesses when an alert grows or resolves, at most twice for growth and once for resolution, within 12 hours.
  - Identical repeat alerts within 24 hours inherit the earlier verdict at no credit cost.
  - Each alert is metered once as an "Alert Triaged" credit event.
- **Tuning Agent** ([Tuning agent](https://docs.getmontecarlo.com/docs/tuning-agent)):
  - Analyses alert volume and triage statuses (NO_ACTION_NEEDED vs FIXED vs EXPECTED) plus troubleshooting findings.
  - Proposes at most 3 changes per metric or custom-SQL monitor (10 per table monitor), each flagged if it would reset learned thresholds.
  - The user applies the changes, or "auto-apply" runs hourly scans.
  - It can't apply changes to monitors-as-code monitors.
- **Operations Agent.** A chat front-end that runs "with the same permissions as the user". Every MCP-server operation is available to it, and it routes requests to the Troubleshooting, Triage, Cost and PII agents. Source: [Operations agent](https://docs.getmontecarlo.com/docs/operations-agent).
- **Other agents.** Cost/Performance (combines no-read activity with no downstream lineage to find tables to drop), PII/Compliance, PR agent, and MC Prevent ("Code Change" skill in the IDE, CI gates). Sources: [llms.txt index](https://docs.getmontecarlo.com/llms.txt); repos [mc-prevent-action / mc-prevent-orb](https://github.com/monte-carlo-data).
- **MCP server.** First-party and hosted in Monte Carlo's AWS. Tools include `create_or_update_table_monitor(dry_run=True)` and `create_or_update_sql_monitor`. Sources: [AI Security & Governance](https://docs.getmontecarlo.com/docs/ai-security); [Table](https://docs.getmontecarlo.com/docs/table); [Custom SQL](https://docs.getmontecarlo.com/docs/custom-sql).
- **AI data handling (vendor claim).** Monte Carlo "does not use customer data or AI prompts to train external models". Source: [AI Security](https://docs.getmontecarlo.com/docs/ai-security).
- **Agent observability ("Agent Trust")** ([Agent Observability Overview](https://docs.getmontecarlo.com/docs/agent-monitors-overview)):
  - Instrumentation uses the `montecarlo-opentelemetry` SDK (OTLP), Google ADK, Snowflake Cortex Agents or Databricks Agent Bricks/Genie.
  - Traces flow into a self-hosted OTel Collector and ClickHouse, and an LLM worker runs evaluations through your own cloud LLM.
  - Monitor types: Agent Evaluation, Metric, Trajectory, Validation, and pre-prod.
  - "Reinforcement Loop" proposes fixes as draft PRs or tickets and "never changes a running agent" ([Security overview](https://docs.getmontecarlo.com/docs/security-compliance-overview)).

### Inferences
- The AI layer sits on the same metadata, query-log, lineage and metric substrate as the rest of the product. Monte Carlo itself says the agents work best with "data sampling enabled, full lineage instrumentation, query history". For Globe, the lesson is to build the substrate first (catalog, query history, lineage, metric history, alert history). LLM triage and RCA can come later as a thin layer over those APIs.
- Triage-before-notify plus priority-conditional routing is Monte Carlo's answer to alert fatigue, the top G2 complaint (see section 5). An in-house tool can get much of this with deterministic rules: impact from lineage and key-asset tags, likelihood from alert history and job failures. No LLM is needed for that.

### Gaps
- I found no independent benchmarks of the agents' accuracy. The 80% MTTR-reduction figure is a vendor claim.
- I did not find which LLM providers power Monte Carlo's internal agents. A subprocessors page exists but I did not retrieve it.

## 5. Integrations and pricing model

### Takeaway
- Integrations span cloud warehouses and lakehouses (Snowflake, BigQuery, Redshift, Databricks, Teradata, Synapse, S3/Glue/Athena/Hive/Presto lakes), transactional databases, ETL/orchestration, BI tools, git, Kafka, vector databases, notification and ITSM tools, and a custom/push-ingest API.
- Pricing is usage-based **credits**, consumed per monitor at published rates, across four tiers. List prices are not public. Order-form figures surfaced in search show roughly $0.18 to $0.45 per credit depending on tier.

### Cited Findings
- **Warehouses and lakes.** Redshift, Snowflake, BigQuery, Databricks, Teradata, S3 data lakes (Glue/Hive metastore; Hive, Presto and Athena logs), Azure Synapse, MotherDuck, Dremio. Source: [Data Collection details](https://docs.getmontecarlo.com/docs/data-collection-details-per-integration).
- **Transactional databases.** SQL Server, SAP HANA, Azure SQL, MySQL, Oracle, Postgres. Pinecone is supported as a vector database. Source: [Data Collection details](https://docs.getmontecarlo.com/docs/data-collection-details-per-integration).
- **ETL and orchestration.** dbt Cloud and Core, Airflow, Fivetran, Informatica, Kafka (Confluent), Kafka Connect, Prefect, ADF. Source: [Data Collection details](https://docs.getmontecarlo.com/docs/data-collection-details-per-integration).
- **BI.** Tableau, Looker (git and API), Power BI, Mode, Sisense, Sigma. Source: [Data Collection details](https://docs.getmontecarlo.com/docs/data-collection-details-per-integration).
- **Code repositories.** GitHub (including GHES), GitLab, Azure DevOps, Bitbucket. Source: [llms.txt index](https://docs.getmontecarlo.com/llms.txt).
- **Custom integrations.** A "Custom Warehouse, Database, and Lake Integrations (Public Preview)" option and a Push Ingest API for metadata, query logs and lineage. Source: [llms.txt index](https://docs.getmontecarlo.com/llms.txt).
- **Pricing page** ([Pricing](https://montecarlo.ai/pricing)):
  - Tiers: Start, Scale, Enterprise, Business Critical.
  - "Cost per credit depends on the tier".
  - Start: up to 10 users and 1,000 monitors, 10k API calls a day, 24-hour support SLA.
  - Scale: unlimited users, data products and domains, "Self-Hosted Storage", SSO/SCIM/audit logs, 50k API calls a day.
  - Enterprise: multi-workspace, chargebacks, EDW integrations and ServiceNow, 100k API calls a day.
  - Business Critical: dedicated instance and cross-region DR.
  - No dollar figures are shown.
- **Credit prices (from search results, not verified).** Order forms list $0.18 per credit (Start), $0.25 (Scale) and $0.45 (Enterprise). FitGap lists $0.15 for Start and $0.50 for Enterprise + Advanced Networking. These sources conflict, and I could not open the order forms directly because they redirect. Sources: [Start order form](https://www.montecarlodata.com/pricing/start-order-form/); [Scale order form](https://www.montecarlodata.com/pricing/scale-order-form/); [Enterprise order form](https://www.montecarlodata.com/pricing/enterprise-tier-order-form/); [FitGap](https://us.fitgap.com/products/monte-carlo).
- **Billing mechanics.**
  - A Billing page shows daily usage, commits, running statement and credit rate, and supports a daily credit-threshold email alert.
  - A per-monitor "Consumption" data export covers the last 30 days.
  - A table that sits in N table monitors has its cost split 1/N across them for chargebacks.
  - Sources: [Billing](https://docs.getmontecarlo.com/docs/billing); [Consumption of credits](https://docs.getmontecarlo.com/docs/consumption-of-mc-credits).
- **Independent sentiment.**
  - G2 average is 4.3/5 across 531 reviews.
  - G2's AI-summarised cons: "alert overload … requiring excessive tuning", limited manual threshold control and custom metrics, slow UI.
  - G2's AI-summarised pros: real-time alerts.
  - Sources: [G2 seller page](https://www.g2.com/sellers/monte-carlo); [G2 compare](https://www.g2.com/compare/monte-carlo-vs-validio).

### Inferences
- Billing per monitor or table with metered agent actions means Monte Carlo's costs scale with coverage. For a telco with tens of thousands of tables, broad metadata-based table monitoring is where vendor cost would concentrate. An in-house build can make broad coverage nearly free, because it is just catalog queries.
- The fractional-credit chargeback and the "credit estimate in the dry-run plan" are good ideas for an internal tool too. Exposing estimated warehouse cost per monitor before deployment would help.

### Gaps
- No official per-monitor-type credit consumption table was found publicly.
- I found no Teradata- or Oracle-specific depth notes beyond collection cadence. Teradata freshness comes from query logs.

## 6. What the public product tour reveals (REA)

### Takeaway
- The page at info.montecarlo.ai/product-tour/monte-carlo-overview is not gated. It embeds one **Walnut** (teamwalnut.com) interactive demo, "Agent Observability Walkthrough", with 26 screens and 24 guided steps.
- The screens are captured from a **Figma prototype** ("Agent Obs Demo"), not the live app. The tour still shows the real information architecture: top nav, Create-monitor taxonomy, and an agent-monitor form.
- It does **not** walk through data-quality (table, metric) workflows. The "Data Observability →" link only scrolls to the same embed.

### Cited Findings
- **The page and its embed** (REA `ev_c48e6dc1…`):
  - Accessibility tree: "Self-guided Product Tour", heading "The Agent Trust Platform.", the subheading "unifies data and agent observability to monitor, troubleshoot, and improve production AI systems", and links "Agent Observability →" and "Data Observability →" (the latter is `#product-tour`). The "Contact Us" link was not followed.
  - Network traffic shows the embed is the Walnut player and demo: `app.teamwalnut.com/player/?demoId=80c326a1-960a-49b4-a65a-f20bfd4bfff0`, with `api.teamwalnut.com/graphql`.
  - Many ad and intent trackers also load: 6sense, Qualified, HubSpot, LinkedIn, G2 tracking, Clearout with a Cloudflare Turnstile.
  - Only one `demoId` appears in the page HTML.
- **The tour payload** (REA `ev_c73f04f3…`, GraphQL `demoSettings` response):
  - `demo.name = "Agent Observability Walkthrough"`, `storyline.name = "Agent Observability Self Guided"`.
  - `activeVersion.guides[0].annotations[0..23]`: 24 narrated steps in Quill Delta rich text, each anchored to a `screen.id` with a CSS/DOM selector.
  - `activeVersion.screens`: 26 screens, each with a `screenshotImageUri`.
- **Narrated workflow, steps 0 to 23 (marketing narration):**
  - "monitor both your AI inputs and outputs within a single pane of glass … create monitors to detect issues across data quality, query performance, and now agent outputs."
  - "consolidates trace information within your trusted data platform so it never leaves your environment. Simply select the generated trace table."
  - "Evaluations can be deployed to specific steps in the agent's workflow … (span)".
  - The `support_bot` workflow tree has sub-tasks and multiple LLM calls, and the user selects the final LLM call.
  - Sample prompts and completions are shown with model and duration.
  - "pre-built templates … including relevancy … or completely customize the prompt", then "Test", which gives a score of 4.
  - "We can also deploy deterministic monitors using SQL" with an "output length template".
  - "monitor a sample number of spans per run … filter the spans … compare results across different segments … relevancy across different model types".
  - "anomaly detection capabilities make it easy to set valid alert conditions", plus multiple conditions, for example spans using "more than 400 tokens … keep an eye on costs".
  - Final step: "join the waitlist or request a demo!" The waitlist wording suggests the tour was made before agent observability reached GA.
- **Screen snapshot provenance.** The gzip-compressed screen HTML contains Figma meta tags such as `twitter:title "Agent Obs Demo"` and `figma.com/proto/...Agent-Obs-Demo`. This shows the screens are captured from a Figma prototype, not the production app. Source: screen assets observed in `ev_c48e6dc1…`, downloaded and decompressed outside REA.
- **UI observed in screenshots** (screen PNGs from the asset URLs in the payload):
  - Top nav: **Alerts, Monitors, Dashboards, Performance, Assets, Settings**, a notifications bell, a **Domains** selector and a user menu.
  - "Create monitor" page, with tabs **Monitors / Templates** and tiles:
    - **Agent**: "Monitor LLM response quality by evaluating relevancy, clarity, accuracy, and more".
    - **Table**: "freshness, volume, and schema changes".
    - **Validation**, badged "Monitor with AI": "Identify individual rows with data quality issues".
    - **Metric**, badged "Monitor with AI": "Identify anomalies in aggregated metrics".
    - **Custom SQL**.
    - **Comparison**: "Compare metrics from two data sources to find deltas".
    - **Query performance**.
    - **JSON schema**.
  - "Create agent monitor" form:
    - Choose data: "Select trace table" `[Databricks] support.agent_trace_events`, Agent `support_bot`, "Spans to monitor" breadcrumb (`compose_user_reply > compose_user_reply > ChatBedrockConverse.chat`).
    - Evaluation: "Evaluate relevancy to generate 123 relevancy_score".
    - "Evaluation sample size: Sample 200 spans|% per run, stratified by none".
    - "Group by: Hour of ingestion".
    - "Filter by duration_sec is greater than 400ms".
    - "Segment by", with options in the groups "Evaluation fields" (`relevancy_score`) and "Agent data" (`root_name`, `model_version`, `workflow_name`).
    - "Define alert conditions: Alert when [metric] is [operator]" with "+ Condition".
    - "Define schedule: Run monitor On a schedule | When the table is updated".
  - "Add evaluation" drawer:
    - Toggle between **LLM-as-a-judge** and **SQL**.
    - "Prompt template: Custom", with the helper text "Add {{reference}} to ingest agent data in prompt", an "Advanced" section and a "Test evaluation" button.
    - A sample grid of prompts, completions and results. Sample data uses span names such as `ChatBedrockConverse.chat` with synthetic customer-support tickets.
  - "Select agent span" modal: a span tree `support_bot` → `troubleshoot-user_request` (`generate_hypothesis`, `transform_results`), `escalate_case` (`classify_ticket`, `escalate_ticket`), `compose_user_reply`, each with leaf LLM calls.

### Inferences
- **Tour vs docs.** The tour's agent-monitor form reads traces from a *warehouse table* (Databricks) and calls them "never leaves your environment". The current docs (2026) describe two paths: warehouse-native (Snowflake/Databricks agents) and an OTel→ClickHouse platform in the customer's cloud for custom agents. The tour therefore looks like an earlier, prototype-era version (waitlist CTA, Figma screens) and should be treated as illustrative, not authoritative.
- **Reusable design patterns from the tour.** These carry over to data-quality monitors at Globe, not only LLM monitors:
  - One "Create monitor" catalog of typed monitors.
  - A form structure of Choose data → define metric/evaluation → sample size → group-by time → filter → segment-by → alert conditions (anomaly or explicit, multiple conditions) → schedule (cron or on-table-update) → notify.
  - "Test before save" with a preview grid.
- **Data model visible in the UI.** Monitor {type, data source/table, agent/span or field selection, evaluation(s) producing named numeric fields, sampling, group-by bucket, filters, segment field, alert conditions[], schedule, audiences}. This matches the YAML schema in the docs, which suggests the UI, the API and monitors as code share one schema.
- The tour reveals nothing about lineage graphs, incident objects or the data-quality alert UI. To understand those, use the docs (sections 3 and 4).

### Gaps
- The landing page links "Data Observability →" to the same embed. I found no separate public data-observability (table/metric/lineage/incident) interactive tour.
- I did not click through the tour in-browser. Step text and screens came from the full GraphQL payload instead, which covers all 24 steps. Hotspot click targets (`clickableElementId`, `cssSelector`) were present but not analysed.
- The Figma-based screens may differ from the shipping UI. No live-app screenshots were obtained, and none could be without logging in.
