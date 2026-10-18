# Anomalo: architecture, detection approach and workflow (as of Oct 2026)

> **Raw research notes, not recommendations.** Evidence gathered on 2026-10-18 for the reports in `research-outputs/`. Vendor claims are marked as such; screenshots described here were not committed. Use these to check a claim's source, never as a decision or a plan.


Scope note: no public interactive product tour or ungated docs site was found. The best technical evidence comes from three places: (1) the public `anomalo` Python SDK/CLI on PyPI (v0.55.1, Apache-2.0), downloaded and read directly, (2) Anomalo's own "Chapter N" blog series excerpted from the CTO's O'Reilly book, and (3) older (2021-2022) engineering blog posts. Most other material is marketing. REA was not used because I found no public interactive demo or JS-gated page that needed it. Evidence labels used below: **[TECH]** = code or engineering-level description; **[MKT]** = vendor marketing claim; **[3P]** = third party.

## 1. Deployment, data residency and security posture

### Takeaway
Anomalo ships the same product four ways: multi-tenant SaaS (hosted on `*.anomalo.io`), fully in-VPC on AWS/GCP/Azure (containerized, and seen running on OpenShift), a Snowflake Native App in Snowpark Container Services, and a "hybrid" mode. The SDK points to a remote "Query Runner" agent model for the hybrid case. With SaaS, query results and samples sit in Anomalo's cloud. With in-VPC or the Native App, the vendor says nothing leaves the customer environment. Public security evidence is limited to SOC 2 Type II and HIPAA monitoring, both announced in 2023.

### Cited Findings
- [MKT] Anomalo says it works "fully in your VPC or as SaaS". In a true in-VPC deployment, the vendor's business logic and data management run in the customer's cloud. Containerization is credited with making that practical. Anomalo argues against hybrid models for DQ in this post (Mar 28, 2023, older). — [Anomalo blog: In-VPC deployment](https://www.anomalo.com/blog/data-quality-control-your-own-data-with-in-vpc-deployment/)
- [MKT] With SaaS, setup is light, data is stored in the vendor's cloud, and customers hand over warehouse credentials so the vendor can run queries. — [same](https://www.anomalo.com/blog/data-quality-control-your-own-data-with-in-vpc-deployment/)
- [3P] G2 lists the deployment modes as SaaS, hybrid, in-VPC and Snowflake Native App. An RFP comparison lists in-VPC on AWS, GCP and Azure. — [G2 discussion](https://g2.com/products/anomalo/discuss) (via search snippet); [rfp.wiki](https://www.rfp.wiki/artificial-intelligence/augmented-data-quality-solutions/anomalo/metaplane)
- [MKT] Snowflake Native App: Anomalo calls it the "first fully containerized" data quality native app, running in Snowpark Container Services (Sep 2024). A later post says it "runs entirely inside your Snowflake environment. No data leaves." There is also a separate "Connected App" variant, and it is eligible for Snowflake Marketplace Capacity Drawdown. — [Press release](https://www.globenewswire.com/news-release/2024/09/25/2953203/0/en/Anomalo-Launches-First-Fully-Containerized-Native-Data-Quality-App-for-Observability-on-Snowflake-Marketplace.html); [MCD blog](https://www.anomalo.com/blog/anomalo-is-now-mcd-eligible-on-snowflake-marketplace/)
- [MKT] AWS Marketplace listing "Anomalo Data Quality" is contract-based: units at $1/unit for 12 months. The unit is undefined and real pricing comes through private offers. Purchases count toward AWS EDP/PPA commitments. — [AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-sodj3n3wjs67s); [Anomalo AWS blog](https://www.anomalo.com/blog/anomalo-now-available-on-aws-marketplace/)
- [MKT, case study ~Jun 2026] A global financial-services customer ran a three-phase rollout: (1) Snowflake Native App on critical tables, (2) API-embedded quality gates in a medallion pipeline (Databricks/Kafka/Airflow), (3) a "containerized deployment on OpenShift" where one instance monitors cloud and on-prem systems together. Bad records are quarantined before reaching consumers. The team claims 30 days from kickoff to live despite security gates. — [Anomalo case study](https://www.anomalo.com/case-study/a-global-financial-services-leader-builds-the-foundation-for-agentic-data-operations/?llm_view=1)
- [TECH, SDK] The client connects to `https://<host>/api/public/v1/<endpoint>` and authenticates with an `X-Anomalo-Token` header (legacy) or `Authorization: Bearer`. The host comes from `ANOMALO_INSTANCE_HOST`. The README example host is `https://your-instance.anomalo.io`, which suggests per-customer instance hostnames. Server versions are semver like `v0.190.4`, read from `/version.txt`. — `anomalo/client.py`, `anomalo/cli/state/README.md` in [PyPI anomalo](https://pypi.org/project/anomalo/)
- [TECH, SDK] The SDK exposes **Query Runner keys** (`create/rotate/revoke_query_runner_key`, scoped by `pool_id`, with ISO-8601 expiry and a deployment-default expiry). The internal docs also mention "airgapped networks", and the PyPI update check is designed to fail silently there. — `anomalo/client.py` lines ~816-870; `cli/state/context.md`
- [TECH, SDK] The governance model includes a sensitive-columns registry (`get/add/remove_sensitive_column`), access groups, access-group policies attached to groups, table-level access groups, roles, service accounts, per-user API keys with expiry, and data domains. — `anomalo/client.py`
- [MKT] SOC 2 Type II clean report (audited by Sensiba San Filippo, controls monitored with Drata; May 2023). HIPAA continuous monitoring with reports on request (Oct 2023). — [Security blog](https://www.anomalo.com/blog/anomalos-commitment-to-secure-data-for-our-customers/); [HIPAA blog](https://www.anomalo.com/blog/anomalo-prioritizes-patient-data-protection-with-rigorous-hipaa-compliance-for-todays-healthcare-tech-leaders/)
- [MKT] The 2026 agentic platform is offered "in-VPC or SaaS, on any cloud". Agents can use "any frontier LLM you choose" (bring-your-own LLM). — [Self-Driving Data blog, Apr 2, 2026](https://www.anomalo.com/blog/introducing-self-driving-data-and-the-new-anomalo-agentic-platform/)

### Inferences
- "Query Runner keys" grouped into pools strongly suggest a hybrid architecture: a customer-side agent pulls work from the Anomalo control plane, runs SQL near the data, and returns results. This resembles Monte Carlo's or Collate's agents. It is not documented publicly, so treat it as an inference.
- For Globe: an OpenShift/in-VPC pattern with one instance spanning on-prem (e.g., Oracle/Teradata/Hadoop) and cloud warehouses is the reference shape. Data-residency concerns (Philippine Data Privacy Act) are handled by keeping all compute and sample storage in-network, plus a sensitive-column registry that masks samples.
- SaaS still stores sample rows ("bad rows CSV", good/bad row samples in alerts) on the vendor side. In-house designs should treat sample storage as the main residency risk, not the aggregate metrics.

### Gaps
- No public Helm chart, architecture diagram, component list (DB, queue, workers), or update mechanism for in-VPC was found.
- No public data-retention policy for samples or metrics, and no ISO 27001 evidence.
- No GCP or Azure marketplace listing was verified in this pass.
- The Query Runner architecture is undocumented publicly.

## 2. Detection approach (unsupervised ML, rules, key metrics, observability)

### Takeaway
The core technical bet is "drift detection as classification". Sample about 10k random rows from "today" and from several reference days (yesterday, the same day last week, other seasonal lookbacks). Encode the columns automatically, then train gradient-boosted trees (XGBoost-like) to predict whether a row is from today. If the model can tell today apart, the data has structurally changed. SHAP values attribute that change to cells and columns, and are used to score severity, cluster correlated columns into one issue, and drive root-cause segment analysis. Beside the ML sit system checks (freshness, volume), a constrained NULL-spike model, time-series "key metric" checks, and no-code or SQL validation rules.

### Cited Findings
- [TECH] Sampling: about 10,000 random rows even for huge tables, drawn from today, yesterday, a day from last week and "other days needed for seasonal trends". Today is labeled 1 and the rest 0. Don't ask the warehouse for random rows directly. Instead pull per-day partitions with `TABLESAMPLE` (or equivalent), over-fetch a few times, then sample randomly from that chunk. — [Anomalo blog: Chapter 4, How to build an ML model for DQ monitoring](https://www.anomalo.com/blog/chapter-4-how-to-build-a-machine-learning-model-for-data-quality-monitoring/) (book excerpt, roughly mid-2026 repost of 2024 book content)
- [TECH] Feature encoding is chosen automatically per column: numeric, frequency (how common a string is), isNull, secondOfDay/timeDelta for timestamps, and OneHot. Meaningless digit strings like ZIP codes are encoded as categorical. More complex encoders find more issues but are harder to interpret. — [same](https://www.anomalo.com/blog/chapter-4-how-to-build-a-machine-learning-model-for-data-quality-monitoring/)
- [TECH] Model: gradient-boosted decision trees, with XGBoost suggested as the starting library. They were chosen because they train on small samples, scale, fit tabular data, infer fast, and have few important hyperparameters (learning rate, tree complexity). Evaluation uses held-out data. — [same](https://www.anomalo.com/blog/chapter-4-how-to-build-a-machine-learning-model-for-data-quality-monitoring/)
- [TECH] SHAP measures each row-column cell's contribution to the "is today" prediction. Severity is aggregated across thousands of points and shown on a scale from minimal to extreme. The design targets issues affecting 1% or more of rows, and lookback days are meant to suppress seasonal spikes like Cyber Monday. — [same](https://www.anomalo.com/blog/chapter-4-how-to-build-a-machine-learning-model-for-data-quality-monitoring/)
- [TECH] SHAP values are used to cluster correlated column alerts into one interpretable issue. Each alert carries a human-readable explanation and a ranked list of affected columns and values. — Anomalo blog posts surfaced via [search for Anomalo SHAP posts](https://www.anomalo.com/blog/unsupervised-data-monitoring/) (snippet-level evidence; the clustering post itself was not fetched)
- [TECH, Jan 26, 2021, older] Two table-level ML checks run by default: "no increase in NULL values" (a constrained, high-priority model) and "no anomalous records" (the full model, covering continuous distributions, categorical values, time durations and cross-column relationships). Each anomaly gets a severity score against a per-table **learned threshold** that "accounts for how much background noise there is in each column". Root cause analysis works from the raw data to find concentrated segments (e.g., NY = 100% of anomalous rows vs. 29.8% of the population). Visualizations are chosen by anomaly type. A "data chaos library" injects synthetic anomalies (e.g., `TimeColumnZero`) for benchmarking. The freshness check decides when each day's data is complete and then triggers the other checks. — [Anomalo blog: Unsupervised data monitoring](https://www.anomalo.com/blog/unsupervised-data-monitoring/)
- [TECH, SDK] The system check type `MLTableTimeZeroStatic` appears alongside `DataFreshnessStatic`. System checks (`data_freshness`, `data_volume`, and ML table checks) are created by the server when a table is configured. They can only be modified, never created or deleted by users. User checks are upserted by `ref`. — `cli/state/context.md`, `cli/state/machine.py`
- [MKT] Models are calibrated on benchmark datasets "that closely resemble the size and complexity of real business data" with realistic injected anomalies. "Automatic secondary checks" weed out false positives. The ML needs at least about 100 rows/day and runs daily after freshness confirms an update. — Anomalo pages via [search results](https://www.anomalo.com/anomaly-detection-software/)
- [TECH, May 2022] Checks out of the box: freshness, volume, missing data (dropped segments, null spikes), and table anomalies (duplicates, schema changes, distribution shifts). Custom checks can be no-code rules or arbitrary SQL. — [Anomalo + Databricks blog](https://www.anomalo.com/blog/detecting-stale-missing-corrupted-and-anomalous-data-with-anomalo-and-databricks/)
- [TECH, SDK] The table config knobs show how time and seasonality are modelled. `time_columns` (more than one allowed) plus `time_column_type` define daily intervals, or `timeColumnsIsNull` means full-table scans. `scope_of_data_to_check` is `incremental` or `all`. `interval_skip_expr` is a cron expression (e.g., skip weekends). `custom_holidays` takes a date list. `get_check_intervals` lists the per-interval history, and `run_checks(interval_id=..., respect_data_freshness_gate=...)` backfills or reruns a given interval. — `anomalo/client.py` `configure_table`, `cli/state/README.md`
- [TECH, SDK] Key-metric checks use `checkType "metric"` with params like `metric_name: row_count`, and they can be `timeBased`. `download_entity_time_series_csv(quality_check_run_id, entity_filter, metric_name)` implies segment-level ("entity") time-series metrics. — `cli/state/README.md`, `client.py`
- [TECH, SDK] The `anomalo_view` relation type lets a "table" be defined by SQL (`anomalo_view_sql`) so checks can run on derived datasets or joins. — `client.py` `create_table`

### Inferences
- Design lesson: classifier-based two-sample testing (today vs. a reference window) is a single model that catches distribution, null, categorical and cross-column relationship drift at once. SHAP gives explainability almost for free. Fixed sample size keeps compute flat no matter how big the table is.
- Alert-fatigue controls come in layers: a minimum effect size (1% of rows), per-table learned noise thresholds, seasonal lookback in the reference set, secondary confirmation checks, SHAP-based column clustering into one issue, freshness gating (don't evaluate partial data), and holiday/skip calendars. That is a useful checklist for Globe.
- The 2021 "learned threshold per table" and the 2024 book description are consistent, so the core algorithm appears stable from 2021 to 2026.

### Gaps
- Exact threshold calibration, the "secondary checks" algorithm, the stopping criteria and hyperparameters were not public. They are deferred to the book's chapters (paywalled O'Reilly, [TOC](https://www.oreilly.com/library/view/-/9781098145927)).
- The time-series model used for key metrics and volume is not publicly specified.
- Medium posts (anomalo-hq and Snowflake Medium) returned 403 and weren't read.

## 3. Unstructured data quality and AI/GenAI agents (2024-2026)

### Takeaway
Unstructured monitoring went from private beta (Jun 2024) to GA with "Workflows" (May 2025). It uses LLMs to score document collections for PII, duplicates, errors, abusive language and sentiment, at up to 100k documents per run. In April 2026 the product was repositioned as "Self-Driving Data", with nine agents on top of the profiling engine. Four of those are "coming soon".

### Cited Findings
- [MKT/3P, Jun 2024] Unstructured Data Monitoring in private beta, with consumption-based pricing planned. It detects inconsistencies, errors, duplication, sentiment and PII. Audio via transcription is on the roadmap. The LLM is unnamed. — [TechTarget](https://www.techtarget.com/searchdatamanagement/news/366588671/Anomalo-unveils-quality-monitoring-for-unstructured-data)
- [3P, May 2025] GA with Workflows: finds duplicates, errors, PII and abusive language, up to 100,000 documents per operation. — [SiliconANGLE](https://siliconangle.com/2025/05/29/anomalo-expands-data-quality-insights-unstructured-information/); [BigDATAwire](https://www.bigdatawire.com/this-just-in/anomalo-expands-unstructured-data-monitoring-with-new-workflows-hub/)
- [MKT] The Atlan integration was extended to unstructured data. — [Anomalo blog](https://www.anomalo.com/blog/anomalo-atlan-expand-integration-to-unstructured-data-with-app-framework/)
- [MKT, Apr 2, 2026] Agents:
  - GA: Table Observability (freshness, availability, schema), Data Quality (the core engine; users can "describe good data in natural language"), and AIDA conversational analytics (NL to SQL plus charts, grounded in profiling; also the platform's chat UI for configuration).
  - Preview: Data Insights (an unprompted investigation that writes a report on what changed, why, what's affected and suggested actions) and Data Documentation (pulls from chat, wikis, runbooks and profiling; doesn't overwrite human docs; traceable sources).
  - Coming soon: Data Issue First Responder (triage via runbooks, opens ServiceNow/JIRA tickets, escalates, "logged and auditable"), Dashboarding, Business KPI Monitoring, and Experiment Evaluation.
  
  — [Self-Driving Data blog](https://www.anomalo.com/blog/introducing-self-driving-data-and-the-new-anomalo-agentic-platform/); [BigDATAwire](https://www.hpcwire.com/bigdatawire/this-just-in/anomalo-launches-agentic-self-driving-data-system-to-automate-data-operations/)
- [MKT, case study] AIDA was used for "faster rule creation and profiling insights". — [case study](https://www.anomalo.com/case-study/a-global-financial-services-leader-builds-the-foundation-for-agentic-data-operations/?llm_view=1)

### Inferences
- The LLM features sit on top of the statistical profiling store (profiles, check history, documentation). They don't replace it. For an in-house tool, build the profile and metric history first. NL rule authoring and auto-investigation are cheap additions once that store exists.

### Gaps
- No technical detail on LLM orchestration, prompt grounding, guardrails, approval gates for agent actions, or cost of unstructured scans.

## 4. Workflow: configuration, alerting, triage, integrations

### Takeaway
Configuration is per table: connect a warehouse, crawl schemas, enable a table, set its cadence, and the system checks appear automatically. Users then add rule and metric checks through the UI, API, or a declarative YAML/Turtle "config-as-code" CLI with pull/plan/apply. Alerts go to notification channels (Slack, Teams, email, PagerDuty, webhooks) with priority levels and include samples of good and bad rows. A triage history exists per run. Catalog integrations are webhook-based (e.g., Atlan).

### Cited Findings
- [TECH, SDK] The data model hierarchy is Organization → Warehouse (with `schema_crawl_inclusion_list`/`exclusion_list`/`priority` and `refresh`/`refresh/new`) → Table (types include real tables and `ANOMALO_VIEW`) → TableConfiguration → Checks (system and user). Around these sit Labels (scope: table, check or everywhere), Notification Channels, Data Domains, Lineage edges (`add_table_lineage_edge`, upstream/downstream lists), Table Documentation, Table Profile (`get_table_profile(date=...)`) and async Tasks (`poll_for_task_completion`). — `anomalo/client.py`
- [TECH, SDK] Execution model: `run_checks(table_id, interval_id, check_ids, force, respect_data_freshness_gate, execution_priority="high")` returns a job. `get_run_result(run_checks_job_id | quality_check_run_id)` returns results. Also available: `get_run_result_triage_history(job_id)` and `download_bad_rows_csv(quality_check_run_id)`. This means the hierarchy is job → per-check run → bad-row samples, plus triage state. — `client.py`
- [TECH, SDK] Cadence types are `automatic` (freshness-gated: run when new data lands), `scheduled` (e.g., `checkCadenceRunAtDuration "PT2H"` after midnight UTC) and `observability` (metadata-only monitoring). The flags `enable_data_quality` and `enable_observability` toggle separately. SLA settings are `use_auto_sla`, `fresh_after` (ISO duration) and `notify_after` (time of day). Also `always_alert_on_errors`, and the `keep_alerting_on_stale_data` / `alert_default_notif_channel` check params. On automatic tables, priority for freshness and volume is managed automatically. — `cli/state/README.md`, `cli/state/machine.py`
- [TECH, SDK] Config-as-code: `anomalo pull/examine/apply --dryrun/destroy`, using YAML (`AnomaloVersionID: 2`) or SHACL/Turtle validated by W3C SHACL shapes. There is no state file: live state is re-read from the API on each run, then diffed. The plan is sorted topologically, and the docs suggest a GitHub Actions CI/CD flow. — `cli/state/context.md`, `README.md`
- [TECH, May 2022] Alerts go to email, Slack, Microsoft Teams or PagerDuty, and an API is available. Each alert shows the percentage of affected rows, where the failure sits, and good/bad row samples. The "Pulse" dashboard covers coverage, arrival times, trends and repeat offenders. — [Databricks blog](https://www.anomalo.com/blog/detecting-stale-missing-corrupted-and-anomalous-data-with-anomalo-and-databricks/)
- [TECH, SDK] `configure_table(slack_users=...)` maps Slack user mentions per table. SDK examples use the channel names "pagerduty", "slack-alerts" and "ops-alerts". — `client.py`, `anomalo_complete_shapes.ttl`
- [3P] Atlan connector: Anomalo sends webhook events to Atlan on check runs, and Atlan links the checks to assets. — [Atlan docs](https://docs.atlan.com/apps/connectors/observability/anomalo/how-tos/integrate-anomalo)
- [MKT] There is a native Alation integration, and dbt/Airflow orchestration integration. dbt lists Anomalo as a DQ partner. — [Anomalo page](https://www.anomalo.com/?p=1342); [dbt integrations](https://www.getdbt.com/product/integrations)
- [MKT] Auto-generated upstream and downstream lineage; ticketing integrations; "sophisticated alert routing". ServiceNow and JIRA are named for the First Responder agent. — [Product overview](https://www.anomalo.com/product-overview/?llm_view=1); [Self-Driving Data blog](https://www.anomalo.com/blog/introducing-self-driving-data-and-the-new-anomalo-agentic-platform/)

### Inferences
- The way the SDK designs `run_checks` + `respect_data_freshness_gate` + result polling is what enables "quality gates" in Airflow and dbt. The case study's quarantine pattern is likely built on this, not a native feature.
- Collibra integration was not confirmed.

### Gaps
- The UI was never seen directly (no tour or demo found). There are no details on triage states (acknowledge, resolve, mute) beyond the existence of triage history.
- `include_mc_fields` on `get_run_result` is unexplained (possibly multi-column or Monte-Carlo-compatible fields). This is unknown.

## 5. Compute cost model, scale claims, pricing

### Takeaway
Compute is pushed down to the customer's warehouse as SQL. Bounded random samples (about 10k rows per day per table) feed the ML, while metadata-only "observability" monitoring covers the long tail cheaply. Pricing is sales-led with no published price list. Buyer-reported contracts are around $58k-$165k/yr.

### Cited Findings
- [TECH] Fixed-size per-day samples via `TABLESAMPLE`, regardless of table size. — [Chapter 4 blog](https://www.anomalo.com/blog/chapter-4-how-to-build-a-machine-learning-model-for-data-quality-monitoring/)
- [MKT] "Efficient queries that run hourly" and "cost-effective monitoring of millions of tables". — [Product overview](https://www.anomalo.com/product-overview/?llm_view=1)
- [TECH, SDK] `scope_of_data_to_check: incremental|all` and the cadence type `observability` (metadata-only) are cost levers. `bulk_update_table_configuration(filter, configuration)` configures at scale. — `client.py`
- [3P] Vendr buyer data shows $58,000 to $164,688/yr, median about $115k (self-reported). The AWS listing is unit-based private-offer pricing. — [Vendr](https://www.vendr.com/marketplace/anomalo); [AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-sodj3n3wjs67s)
- [3P] G2 is about 4.4 from 41 reviews and Gartner Peer Insights about 4.7 from 21 reviews (an aggregator figure, low confidence). Reviewer cons include a learning curve, opaque pricing, and weaker real-time/streaming coverage than batch. — [rfp.wiki](https://www.rfp.wiki/artificial-intelligence/data-observability-tools/anomalo); [Gartner](https://www.gartner.com/reviews/product/anomalo)
- [3P, suspect] rfp.wiki claims Anomalo was acquired by Handshake in Jan 2026. I could not verify this. Tracxn lists Handshake's acquisitions as Uplimit, **Cleanlab AI** and TalentSpace, and Anomalo's own Apr-Jun 2026 posts show it operating independently. This looks like an AI-aggregator error confusing it with Cleanlab, so treat it as false unless confirmed. — [rfp.wiki](https://www.rfp.wiki/artificial-intelligence/data-observability-tools/anomalo); [Tracxn Handshake acquisitions](https://tracxn.com/d/acquisitions/acquisitions-by-handshake/__nQJR1AWdTc_wNgj5ufNOPBMjvT8oqR3xR6Doegtefpw)

### Inferences
- Cost architecture lesson: split into two tiers. A cheap metadata tier (freshness, volume and schema from information_schema or query history) covers everything. An expensive deep tier (sample-based ML plus rules) is reserved for critical tables, with freshness gating so each partition is scanned once.

### Gaps
- No public warehouse-credit benchmarks and no definition of the pricing unit (tables? checks?).

## 6. Public demos, API/SDK and what it reveals

### Takeaway
No interactive demo or public product tour exists. Demos are gated behind "request a demo". The richest public artifact is the `anomalo` PyPI package (v0.55.1, Apache-2.0, published from the private GitHub repo `datagravity-ai/dquality`). Its bundled internal developer docs show the domain model, config-as-code engine and execution API in detail.

### Cited Findings
- [TECH] Package: `pip install anomalo` installs a REST client plus a `fire`-based CLI. Dependencies are casefy, dataclasses-json, fire, pyyaml, requests and tenacity, with retry using exponential jitter and 5 attempts; pyshacl and rdflib are an optional extra. The package was published via trusted publishing from `datagravity-ai/dquality`. — [PyPI anomalo](https://pypi.org/project/anomalo/0.45.2/); local read of v0.55.1 wheel
- [TECH] The bundled `context.md` leaks internal ticket IDs (ONC-923, QLTY-3288, QLTY-2670), a Django backend (`site_apps.main.models.schema`), and a `www/` monorepo. It also exposes a design invariant: `POST configure_table` is a **full replace**, so partial edits must use `PATCH tables/<id>/config`. — `anomalo/cli/state/context.md`
- [TECH] API endpoint inventory (about 120 methods) covers warehouses, tables, configured_tables, check intervals, checks (create/update/clone/delete), run_checks/get_run_result/triage history/bad rows, labels, users, roles, API keys, query-runner keys, lineage edges, table configuration (+ bulk), documentation, sensitive columns, access groups and policies, tasks, data domains and notification channels. — `anomalo/client.py`
- [3P] Older talks: Data+AI Summit 2022 (CTO on how the algorithms are calibrated) and the SE Daily podcast (Feb 2022). — [Anomalo blog](https://www.anomalo.com/blog/anomalo-to-speak-about-how-unsupervised-machine-learning-can-scale-data-quality-monitoring-in-databricks-at-data-ai-summit-2022/); [SE Daily](https://softwareengineeringdaily.com/podcasts/data-quality-using-anomalo-with-jeremy-stanley/)
- [3P] Book: Stanley & Schwartz, *Automating Data Quality Monitoring* (O'Reilly, Jan 2024). Chapter 4 covers sampling, feature encoding, model development and explainability; another section is titled "Data Quality Monitoring Is Not Outlier Detection". — [O'Reilly](https://www.oreilly.com/library/view/-/9781098145927)

### Inferences
- Data-model template for Globe: Warehouse → Table/View → TableConfig (time column, cadence, SLA, calendar, scope) → SystemChecks (auto-created, modifiable only) + UserChecks (keyed by stable `ref`) → Interval → Job → CheckRun → {result, severity, bad-row sample, entity time series, triage history}, plus Labels, Domains, AccessGroups/Policies, SensitiveColumns, NotificationChannels and Lineage. Config-as-code with stateless diffing against the live API is worth copying.

### Gaps
- No public API reference (OpenAPI) was found. Response schemas are unknown because the client passes JSON through.
- The YouTube demos were not reviewed in this pass.
