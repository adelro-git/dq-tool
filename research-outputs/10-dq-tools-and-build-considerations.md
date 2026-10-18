# Build Globe's data quality monitor inside AWS

> **Status: RECOMMENDATIONS, NOT DECISIONS.** Written 2026-10-18 by an AI research session from public sources and the context pack (files 01–06). **Not verified against the Monitor's code.**
>
> **Read this before acting on anything below (people and agents alike):**
> 1. Every item is a *candidate* to map against what DQ Ops has already built ([02](../02-what-we-built.md)) and its constraints ([04](../04-constraints.md)), not an instruction.
> 2. Look the item up in [07-reco-crosswalk.md](07-reco-crosswalk.md) first. It records the existing build piece each item touches, its status per the context pack, and the team's decision.
> 3. Confirm the current state in the code before proposing a change. The crosswalk statuses come from documents and may be stale.
> 4. If an item conflicts with an existing Monitor convention, a constraint in 04, or a recorded team decision, **the existing build wins**. Flag the conflict; do not "fix" the build to match this document.
> 5. Do not implement an item whose crosswalk decision is blank or "Reject".

> **Superseded in part.** This was the first report in the series, written before the context pack was shared. Its Globe section (hosting in AWS next to Snowflake, Vercel as an option) and its R/C lists assume a new build on a target platform. **They do not reflect the current stack or the open status of LEAP.** The vendor and tool findings still stand. For current-stack recommendations use [08](08-dq-monitor-robustness-plan.md); for how each R/C item maps to the existing build see [07](07-reco-crosswalk.md).


The leading data quality and observability tools all work the same way underneath. A control plane holds the check definitions, the metric history, the lineage graph and the incident state. A lightweight executor sits next to the data and pushes SQL down to the warehouse. Monitoring comes in two tiers. A near-free metadata tier (freshness, volume, schema) covers every table, and a more expensive query tier (column metrics, rules, reconciliation, ML on samples) is reserved for critical assets. What separates the products is how they control noise, not how clever their detection is. Monte Carlo runs an ML ensemble with weekly seasonality and an explicit "mark as normal" feedback loop. Anomalo runs a gradient-boosted "is this today's data?" classifier with SHAP attribution. DQOps, Great Expectations and dbt give you the rule and test primitives but leave history, anomaly detection and incident workflow mostly to you. All the 2025–2026 AI features (triage, root-cause, rule-recommendation agents, MCP servers) sit on top of that same metadata, history and lineage layer.

For Globe Telecom, the evidence points to building the tool inside Globe's own AWS estate, next to its Snowflake data warehouse. The design should be aggregates-first, with row samples off by default and masked when enabled. SIM-registration and subscriber tables should get aggregate-only checks. Vercel can host at most the UI, and only if the UI serves aggregates. Under the Data Privacy Act, any third-party host is a processor that Globe remains accountable for. Vercel's DPA allows worldwide data transfer, and its functions cap at 800 seconds when private networking is on. The regulatory points below are considerations to validate with Globe Legal, the DPO office and InfoSec. They are not legal advice. The design recommendations and the checklist at the end carry stable IDs, so they can be scored line by line against the team's existing repository in the planned gap analysis.

## Every leading tool splits a cheap metadata tier from an expensive query tier

Monte Carlo is the clearest example of the control-plane/executor split. It is a multi-tenant SaaS on AWS with three components:

- a Cloud Service (UI, monitoring, notification routing)
- an optional customer-hosted Data Store for row samples
- an optional customer-hosted Agent that holds credentials and runs operations against sources

([Monte Carlo Architecture](https://docs.getmontecarlo.com/docs/platform-architecture)). The newer Generic Agent only makes outbound connections. It holds a Server-Sent-Events stream to Monte Carlo's agent service, receives operations, and returns results over HTTPS. It runs on Docker or Kubernetes and is open source ([Generic Agent Deployment](https://docs.getmontecarlo.com/docs/generic-agent-platforms)).

The important caveat is that **metadata, metrics, query logs and aggregated statistics are "always stored in Monte Carlo's Cloud,"** even in hybrid mode. Only row samples and AI-agent trace content can stay in the customer's environment ([Information Monte Carlo Collects](https://docs.getmontecarlo.com/docs/info-monte-carlo-collects)). Monte Carlo documents only **US and EU regions** ([Regional Hosting](https://docs.getmontecarlo.com/docs/regional-hosting)). It also acknowledges that query logs and samples may contain personal data, and asks customers to mask at the source ([Privacy](https://docs.getmontecarlo.com/docs/privacy)). For a Philippine telco, query text can carry MSISDNs in WHERE clauses, so this matters directly.

Monte Carlo's monitors fall into Table, Metric, Validation, Job and Agent families ([Monitors overview](https://docs.getmontecarlo.com/docs/monitors-overview)). Table monitors read Snowflake `information_schema` and `QUERY_HISTORY` hourly ([Data Collection details](https://docs.getmontecarlo.com/docs/data-collection-details-per-integration)), "keeping compute costs minimal" ([Table Monitor](https://docs.getmontecarlo.com/docs/table)). Metric, custom-SQL, validation and comparison monitors run SQL in the customer's warehouse. They offer schedules, triggers on table update, up to five segment fields, and backfill ([Metric Monitors](https://docs.getmontecarlo.com/docs/metric-monitors)). Monitors can be written as YAML and applied through a CLI with replace-the-whole-definition (PUT) semantics. The dry-run plan prints **an estimated daily credit cost per monitor** before anything is deployed ([Monitors as Code](https://docs.getmontecarlo.com/docs/monitors-as-code)).

Anomalo ships the same product four ways: SaaS, fully in-VPC on AWS/GCP/Azure, a Snowflake Native App running inside Snowpark Container Services, and a hybrid mode ([Anomalo in-VPC blog](https://www.anomalo.com/blog/data-quality-control-your-own-data-with-in-vpc-deployment/); [Native App press release](https://www.globenewswire.com/news-release/2024/09/25/2953203/0/en/Anomalo-Launches-First-Fully-Containerized-Native-Data-Quality-App-for-Observability-on-Snowflake-Marketplace.html)). Its public Python SDK shows the domain model directly ([PyPI anomalo](https://pypi.org/project/anomalo/)):

- Warehouse → Table → TableConfiguration → Checks, where system checks are created by the server and can only be modified, and user checks are keyed by a stable `ref`
- run jobs, bad-row samples and triage history
- a registry of sensitive columns
- "Query Runner keys" grouped in pools, which suggests a customer-side executor for the hybrid mode (an inference; it is not documented publicly)

One documented case study is a financial-services firm that ran a single containerized instance on OpenShift across cloud and on-prem systems and quarantined bad records before they reached consumers. The firm claims it went live in 30 days ([Anomalo case study](https://www.anomalo.com/case-study/a-global-financial-services-leader-builds-the-foundation-for-agentic-data-operations/?llm_view=1)). That is a useful reference shape for Globe, whose estate is also mixed.

The open-source tools are building blocks rather than platforms. DQOps is the closest to a complete platform. It is a Java/Spring Boot server with a TypeScript UI. Its "sensors" are Jinja2 SQL templates written once per dialect (about 18 dialects, Snowflake and Teradata among them), and every sensor returns a column named `actual_value`. Its "rules" are separate stateless Python functions that take the current value, the parameters and the history, and return passed/expected/lower/upper ([DQOps architecture](https://dqops.com/docs/dqo-concepts/architecture/dqops-architecture/); [sensor templates](https://github.com/dqops/dqo/tree/develop/home/sensors); [rule source](https://github.com/dqops/dqo/blob/develop/home/rules/percentile/anomaly_stationary_percentile_moving_average.py)). Results go to Hive-partitioned Parquet tables (`sensor_readouts`, `check_results`, `errors`, `incidents`, `error_samples`, `statistics`) partitioned by connection, table and month. Each row's ID is a hash, so a rerun overwrites the earlier row instead of duplicating it ([DQOps data storage](https://dqops.com/docs/dqo-concepts/data-storage-of-data-quality-results/)). However, **DQOps is now under Business Source License 1.1**. It forbids including the code "in or with" any for-profit product, and its last commit was on 5 Jan 2026 ([LICENSE](https://github.com/dqops/dqo/blob/develop/LICENSE)). Treat it as a design reference, not a dependency.

Great Expectations Core (Apache-2.0, v1.24.0 released 7 Oct 2026) is a validation library. Its chain is Data Context → Data Source/Asset/Batch Definition → Expectation Suite → Validation Definition → Checkpoint → Actions. It pushes work down through SQLAlchemy, Spark or Pandas engines ([GX overview](https://docs.greatexpectations.io/docs/core/introduction/gx_overview); [GitHub](https://github.com/great-expectations/great_expectations)). Version 1.x removed the profilers and the database-backed stores ([GX migration guide](https://docs.greatexpectations.io/docs/reference/learn/migration_guide)). Its **telemetry is on by default** and is disabled with `GX_ANALYTICS_ENABLED=False` ([GX usage statistics](https://docs.greatexpectations.io/docs/reference/learn/usage_statistics)).

dbt's model is that a test is a SELECT that returns failing rows. The failure count is judged by `warn_if`/`error_if`, and `store_failures` keeps only the latest failures, with no history ([dbt severity](https://docs.getdbt.com/reference/resource-configs/severity); [store_failures](https://docs.getdbt.com/reference/resource-configs/store_failures)). In 2026 dbt became a Rust engine (v2.0.8 tagged 14 Oct 2026). It now has an Apache-2.0 `dbt-oss` distribution and a proprietary-but-free `dbt` binary, and it emits Parquet artifacts that DuckDB can query ([dbt repo README](https://github.com/dbt-labs/dbt-core); [dbt licensing](https://docs.getdbt.com/docs/dbt-licensing)).

| Tool | Where checks execute | What leaves the customer environment | Check model | Result history | License / commercial model |
|---|---|---|---|---|---|
| Monte Carlo | Customer warehouse via SaaS-held credentials or customer-hosted agent | Metadata, metrics, query logs, aggregates always; samples optional | Table / Metric / Validation / Custom SQL / Comparison / Job / Agent monitors; YAML MaC | Vendor-hosted | SaaS, usage-based credits, four tiers |
| Anomalo | Customer warehouse (SaaS, in-VPC, Snowflake Native App, hybrid) | SaaS: results and samples; in-VPC / Native App: vendor says nothing | Auto system checks + ML table checks + rule/metric checks; YAML/Turtle config-as-code | Vendor- or customer-hosted | Sales-led, unpublished |
| DQOps | Customer DB via Jinja SQL sensors | Nothing if self-hosted; Cloud sync optional | Sensor + rule, profiling/monitoring/partition checks, warning/error/fatal | Local Parquet lake | BSL 1.1, SaaS tiers from $600/mo ([pricing](https://dqops.com/pricing/)) |
| GX Core | Customer engine (SQLAlchemy/Spark/Pandas) | Telemetry events unless disabled | Expectation suites + checkpoints + actions | None built-in (JSON / Data Docs) | Apache-2.0 |
| dbt | Customer warehouse during builds | Nothing (OSS) | SELECT-returns-bad-rows tests, unit tests, contracts | None built-in (per-run artifacts) | Apache-2.0 `dbt-oss`; proprietary free `dbt` |
| Snowflake DMFs | Snowflake serverless | Nothing | System + custom Data Metric Functions, optional anomaly detection | Event table | Enterprise Edition, billed per run ([Snowflake docs](https://docs.snowflake.com/en/user-guide/data-quality-schema-level)) |

The last row matters for Globe. **Snowflake Data Metric Functions already provide ROW_COUNT, FRESHNESS, null and duplicate metrics, plus lower-bound anomaly detection, inside Snowflake** ([Snowflake DMFs](https://docs.snowflake.com/en/user-guide/data-quality-schema-level); [anomaly status view](https://docs.snowflake.com/en/sql-reference/local/data_quality_monitoring_anomaly_detection_status)). Databricks does the same from commit history ([Databricks docs](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/)). An in-house tool's value therefore lies above basic freshness and volume: cross-platform coverage, telco business rules, source-to-target reconciliation, ownership, and incident workflow.

## Anomaly detection is mostly classical statistics wrapped in noise controls

Monte Carlo describes "an ensemble of anomaly detection models" and selects a model per series ([Anomaly Detection](https://docs.getmontecarlo.com/docs/anomaly-detection-overview)). Thresholds appear after **7 days and mature by 14 days**, trained on a rolling window of at most 6 weeks with weekly seasonality ([pipeline observability](https://docs.getmontecarlo.com/docs/pipeline-observability)). Detected anomalies are automatically excluded from training. A user can click "mark as normal" to add a point back. Exclusion windows can be set for holidays, and a daily metric needs about **4 anomalous days within 7** before the model learns a recurring pattern ([Tuning thresholds](https://docs.getmontecarlo.com/docs/tuning-thresholds)). Monte Carlo deliberately keeps triage status out of model feedback: "Alert statuses do not provide feedback to the models" ([Alert Statuses](https://docs.getmontecarlo.com/docs/incident-statuses)). That stops a habit of clicking "no action needed" from quietly loosening thresholds. Monte Carlo's GitHub org also has a fork of Nixtla's `statsforecast` ([monte-carlo-data/statsforecast](https://github.com/monte-carlo-data/statsforecast)), which hints at statistical forecasting models. That is an inference, not something the docs state.

Anomalo takes a different approach. It samples about **10,000 random rows** from today and from reference days (yesterday, the same weekday last week, and other seasonal lookbacks). It encodes each column automatically and trains gradient-boosted trees (XGBoost is suggested) to predict whether a row comes from today. If the model can tell today apart, the data has changed. SHAP values then attribute the change to columns and cells, set the severity, and group correlated columns into one issue. The design targets issues that affect about 1% or more of rows ([Anomalo Chapter 4](https://www.anomalo.com/blog/chapter-4-how-to-build-a-machine-learning-model-for-data-quality-monitoring/)). Its older engineering post adds three things: a constrained model just for NULL spikes, a learned per-table noise threshold, and a freshness gate so partial data is never evaluated ([Unsupervised data monitoring](https://www.anomalo.com/blog/unsupervised-data-monitoring/)). The fixed sample size keeps compute flat regardless of table size. That suits Globe's multi-billion-row CDR tables.

The open-source tools are much simpler. DQOps OSS uses a Z-score or percentile over a 90-day window, needs 30 days of history, and **handles seasonality only in its commercial edition** ([DQOps anomaly docs](https://dqops.com/docs/categories-of-data-quality-checks/how-to-detect-anomaly-data-quality-issues/)). GX Core has no anomaly detection. GX Cloud has forecasting-based "Automatic" checks for schema, volume and completeness ([GX Cloud Manage Expectations](https://docs.greatexpectations.io/docs/cloud/expectations/manage_expectations/); [GX status incident](https://status.greatexpectations.io/incidents/wk7xnv9l448h)). dbt has none, and Elementary OSS adds anomaly tests as dbt tests ([Elementary](https://github.com/elementary-data/elementary)).

The best-documented in-house system is Uber's DQM. It compresses per-partition column metrics with PCA and forecasts each component with **Holt-Winters additive smoothing**. The prediction intervals give a 0–4 severity, and the table score is the sum of the top three components. Uber's main operational lesson was that **metric-level alerts caused alert fatigue, and table-level aggregation fixed it** ([Uber Engineering](https://www.uber.com/en-NZ/blog/monitoring-data-quality-at-scale/)). Bigeye's summary of that work adds two lessons. Plain standard-deviation thresholds miss incidents on seasonal metrics. Thresholds must also not adapt to bad values during an incident, so they keep alerting until it is resolved ([Bigeye on Uber](https://bigeye.com/blog/lessons-learned-from-uber-designing-an-intelligent-data-quality-monitor)). Snowflake and Databricks both default to **one-sided lower-bound detection for volume**, which cuts false positives from normal growth ([Snowflake](https://docs.snowflake.com/en/sql-reference/local/data_quality_monitoring_anomaly_detection_status); [Databricks](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/)).

Taken together, a robust Globe baseline would be:

- a rolling median/MAD score
- day-of-week and hour-of-day seasonal buckets
- an STL or Holt-Winters forecast with prediction intervals
- automatic exclusion of flagged points from training, plus an explicit "mark as normal" action
- one-sided bounds for volume by default
- severity aggregated at table level

Philippine holidays and the 15th/30th payday cycle will shape recharge and usage volumes. That is an inference for the team to test, not a cited finding. None of the sources benchmark Prophet, STL, Isolation Forest or changepoint methods against each other on data-quality metric series. Vendor accuracy claims cannot be verified.

## Lineage, grouping and ownership decide whether alerts get acted on

Monte Carlo builds table and field lineage mainly by **parsing warehouse query logs**. It distinguishes lineage from the SELECT clause (field to field) from lineage via WHERE and similar clauses (field to table). It struggles with wildcard tables, `JOIN … USING` without table prefixes, and nested fields ([Field Lineage](https://docs.getmontecarlo.com/docs/field-lineage)). It enriches the graph with metadata pushed from dbt Core and Airflow and pulled from Fivetran, Tableau and Power BI APIs ([Data Collection details](https://docs.getmontecarlo.com/docs/data-collection-details-per-integration)). That one graph then drives several features:

- alert grouping: same monitor and same schema within a 5-hour window
- downstream BI impact analysis
- "Data Products": pick the most downstream assets and their upstream lineage is included automatically
- the troubleshooting agent's walk through upstream tables

([Interacting with Alerts](https://docs.getmontecarlo.com/docs/interacting-with-incidents); [Data Products](https://docs.getmontecarlo.com/docs/using-data-product-dashboards)). Anomalo also exposes lineage edges and data domains through its SDK, and its alerts carry the percentage of affected rows plus good and bad row samples ([Anomalo + Databricks blog](https://www.anomalo.com/blog/detecting-stale-missing-corrupted-and-anomalous-data-with-anomalo-and-databricks/)).

Monte Carlo's incident model is a good template. Alert statuses run No status → Acknowledged → Work in progress → Fixed, or Expected / No action needed. "Mark as incident" asks for a SEV-1 to SEV-4 severity drawn from an impact × stakeholder matrix, and alerts within ±7 days can be merged into one incident ([Marking Alerts as Incidents](https://docs.getmontecarlo.com/docs/marking-alerts-as-incidents)). An "Audience" is a named set of recipients across Slack, Teams, email, PagerDuty, Jira, ServiceNow and webhooks. Routing can depend on priority, notifications only escalate upward, and alerts about monitors that failed to run are always sent immediately ([Notifications](https://docs.getmontecarlo.com/docs/notifications-v2)). SLO policies track time-to-acknowledge and time-to-resolve per priority and domain ([SLOs](https://docs.getmontecarlo.com/docs/service-level-objectives-slos)). Domains scope assets, monitors and alerts under RBAC, and "No assignment does not mean visible to all" ([Data Authorization — Domains](https://docs.getmontecarlo.com/docs/data-authorization)).

Organisational failures sink more programs than algorithmic ones. Monte Carlo's own guidance says:

- every monitor should map to an action
- monitors that are never acted on should be pruned
- alerts go to the team that can fix them, not a central team
- severity is tiered by layer (a bronze-layer schema change is informational, a gold-layer revenue table is SEV-1)
- delivery is tiered too: real-time for critical tables, digests for the rest

([Monte Carlo alert-fatigue guide](https://montecarlo.ai/blog-alert-fatigue-monitoring-strategy)). Even so, G2's summary of reviews names "alert overload … requiring excessive tuning" as Monte Carlo's top complaint ([G2](https://www.g2.com/sellers/monte-carlo)). Airbnb's Midas expert-certification process was criticised as unscalable and was followed by a lightweight YAML check framework ([Airbnb DQ Part 1](https://medium.com/airbnb-engineering/data-quality-at-airbnb-e582465f3ef7); [Packt CloudPro, secondary](https://packtcloudpro.substack.com/p/the-hidden-platform-lesson-behind)). The lesson for Globe is to require an owner and a tier on every asset, derive severity from tier and downstream impact, and group alerts so that one upstream break produces one incident.

## AI agents sit on the substrate, and pricing scales with coverage

In April 2025 Monte Carlo launched a Monitoring Agent (recommends monitors) and a Troubleshooting Agent (root cause), with a vendor-claimed 80%+ cut in resolution time ([BusinessWire](https://www.businesswire.com/news/home/20250417869977/en)). By late 2026 the set had grown:

- a Triage Agent that scores every new alert HIGH/MEDIUM/LOW for incident likelihood and impact, using lineage, 60 days of alert history and orchestrator failures ([Triage agent](https://docs.getmontecarlo.com/docs/triage-agent))
- a Troubleshooting Agent that tests "100s of hypothesis" across data, system and code changes ([Troubleshooting agent](https://docs.getmontecarlo.com/docs/troubleshooting-agent))
- a Tuning Agent that proposes threshold changes and can apply them automatically ([Tuning agent](https://docs.getmontecarlo.com/docs/tuning-agent))
- an Operations chat agent and an MCP server ([Operations agent](https://docs.getmontecarlo.com/docs/operations-agent); [AI Security](https://docs.getmontecarlo.com/docs/ai-security))

The AI features depend on row access. Monitor recommendations sample up to 10k rows, and when sampling is turned off, AI recommendations, root-cause follow-up queries and breached-row viewing all stop working ([Data Sampling](https://docs.getmontecarlo.com/docs/data-sampling)). That is the central trade-off for a privacy-constrained telco.

Anomalo repositioned in April 2026 as "Self-Driving Data". Of its nine agents, three are GA (Table Observability, Data Quality, and AIDA, a natural-language-to-SQL assistant), two are in preview (Data Insights and Data Documentation), and four are "coming soon", including a First Responder that opens ServiceNow and Jira tickets. Customers can bring their own LLM ([Self-Driving Data blog](https://www.anomalo.com/blog/introducing-self-driving-data-and-the-new-anomalo-agentic-platform/)). Elsewhere:

- GX Cloud's ExpectAI (beta) recommends expectations from a data sample and writes SQL from a prompt ([GX Cloud](https://docs.greatexpectations.io/docs/cloud/expectations/manage_expectations/)).
- Actian shipped observability agents with an MCP server in February 2026 ([BigDATAwire](https://www.hpcwire.com/bigdatawire/this-just-in/actian-introduces-data-observability-agents-for-the-agentic-ai-era/)).
- Soda 4.0 pairs an open-source contracts engine with AI contract generation ([Soda launch](https://launch.soda.io/blog/one-more-thing)).
- Gartner's 2025 Magic Quadrant (Leaders: Informatica, Ataccama, Qlik) reportedly gave more weight to AI capabilities and unstructured-data support. That reading comes from a vendor ([Ataccama](https://www.ataccama.com/blog/whats-new-in-the-2025-gartner-magic-quadrant-for-augmented-data-quality-solutions)).

Every vendor builds its AI layer on top of the profile, metric-history, lineage and alert-history stores. None replaces them. For Globe, LLMs belong in rule authoring, incident summaries and an MCP/API endpoint, not in the alert trigger. Keeping detection deterministic keeps alerts reproducible.

### The Monte Carlo product tour shows agent observability, not data quality

The public tour at info.montecarlo.ai was inspected with REA, a reverse-engineering MCP tool, using headless capture of public pages only (REA Evidence ID `ev_c48e6dc17c1c9fd8c4320f3ad5781b9ca14c99b2f6aa1b178b3b6189931ccced` for the landing page and network; `ev_c73f04f3a82c190ea78f73a20c0317d27c4f5ed7fc6ac2b7d35a04b573e7faa4` for the Walnut player and its GraphQL `demoSettings` payload). It embeds one Walnut demo, "Agent Observability Walkthrough", with 26 screens and 24 narrated steps. The screen assets carry Figma meta tags ("Agent Obs Demo"), so **the tour is an illustrative Figma prototype of agent observability only**. It is not the live app, and it does not cover data-quality workflows. Its final call to action is a waitlist, which suggests it predates GA. The screenshot PNGs were downloaded outside REA from asset URLs that REA observed.

Even so, it shows a reusable information architecture:

- top navigation of Alerts / Monitors / Dashboards / Performance / Assets / Settings, with a Domains selector
- one "Create monitor" catalog: Agent, Table, Validation, Metric, Custom SQL, Comparison, Query performance, JSON schema
- a monitor form that runs: choose data → evaluation → sample size → group-by time → filter → segment-by → alert conditions → schedule (cron or on-table-update), with "Test" before saving

That structure matches the YAML schema in the docs, which suggests Monte Carlo's UI, API and monitors-as-code share one definition model. Globe's tool should do the same.

### Pricing rewards whoever makes broad coverage cheap

Monte Carlo sells usage-based credits consumed per monitor across the Start, Scale, Enterprise and Business Critical tiers, with no published dollar figures ([Pricing](https://montecarlo.ai/pricing)). It splits a table's cost 1/N across the N monitors that cover it for chargeback, and exports consumption per monitor ([Consumption of credits](https://docs.getmontecarlo.com/docs/consumption-of-mc-credits)). Anomalo is sales-led. Its AWS Marketplace listing uses undefined $1 units through private offers ([AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-sodj3n3wjs67s)). DQOps publishes $600/month (200 tables) and $2,000/month (2,000 tables) SaaS tiers ([DQOps pricing](https://dqops.com/pricing/)). GX lists Developer, Team and Enterprise tiers without prices ([GX pricing](https://greatexpectations.io/pricing/)). DataKitchen, a vendor, puts the average enterprise list price at about $172K a year ([DataKitchen](https://datakitchen.io/blog/the-2026-data-quality-and-data-observability-commercial-software-landscape/)).

Because vendor cost grows with the number of tables monitored, a telco with tens of thousands of tables pays most for the broad metadata tier. An in-house build can make that tier almost free, since it is just catalog queries.

## Several headline claims remain single-source or unverified

Treat the following as open questions, not facts, when citing this report.

| Claim | Status | Evidence and reconciliation |
|---|---|---|
| GX Cloud shut down on 1 Jun 2026, assets went to FICO, and GX Core continues "under Fivetran stewardship" | **Unverified, single source** | Comes only from DataKitchen, a competing vendor ([DataKitchen](https://datakitchen.io/blog/the-2026-data-quality-and-data-observability-commercial-software-landscape/)). The open-source research did not corroborate it. In October 2026 the GX Cloud docs ([Manage Expectations](https://docs.greatexpectations.io/docs/cloud/expectations/manage_expectations/)) and the pricing page with Developer/Team/Enterprise tiers ([GX pricing](https://greatexpectations.io/pricing/)) were still reachable, a GX Cloud status incident is logged in Jan 2026 ([status](https://status.greatexpectations.io/incidents/wk7xnv9l448h)), and GX Core is still shipping (1.24.0 on 7 Oct 2026). Live pages could be stale, so the shutdown is neither confirmed nor ruled out. "Fivetran stewardship" may be confused with the Fivetran–dbt Labs merger, which closed on the same date, 1 Jun 2026 ([Fivetran press](https://www.fivetran.com/press/fivetran-dbt-labs-complete-merger-to-create-the-data-infrastructure-for-trusted-ai-agents)). Do not plan around GX Cloud either way. GX Core (Apache-2.0) is unaffected. |
| Anomalo acquired by Handshake (Jan 2026) | **Likely false** | Appears only on an AI-aggregated site ([rfp.wiki](https://www.rfp.wiki/artificial-intelligence/data-observability-tools/anomalo)). Tracxn lists Handshake's acquisitions as Uplimit, Cleanlab AI and TalentSpace ([Tracxn](https://tracxn.com/d/acquisitions/acquisitions-by-handshake/__nQJR1AWdTc_wNgj5ufNOPBMjvT8oqR3xR6Doegtefpw)), and Anomalo's Apr–Jun 2026 posts show it operating independently. Probably confused with Cleanlab. |
| Monte Carlo credit prices of $0.18 / $0.25 / $0.45 (Start / Scale / Enterprise) | **Unverified, conflicting** | From search snippets of order forms that redirected and could not be opened ([Start](https://www.montecarlodata.com/pricing/start-order-form/); [Scale](https://www.montecarlodata.com/pricing/scale-order-form/); [Enterprise](https://www.montecarlodata.com/pricing/enterprise-tier-order-form/)). FitGap gives $0.15 and $0.50 ([FitGap](https://us.fitgap.com/products/monte-carlo)). Per-monitor credit consumption rates are not public, so total cost cannot be modelled. |
| Anomalo contracts of $58K–$164,688/yr, median about $115K | **Self-reported buyer data** | From Vendr ([Vendr](https://www.vendr.com/marketplace/anomalo)). Small sample, undefined scope. Use only as an order of magnitude. |
| dbt v2 release dates | **Conflicting** | The dbt blog fetch gave an alpha date of 1 Jun 2026. A third-party digest says July alpha and September GA ([dbt blog](https://docs.getdbt.com/blog/dbt-core-v2-is-here); [releases.sh](https://releases.sh/collections/data-engineering/digest/2026-09-14)). Only the v2.0.8 tag on 14 Oct 2026 is confirmed ([releases](https://github.com/dbt-labs/dbt-core/releases)). |
| BSP Circular 1140 amended the IT Risk Management rules | **Unverified** | From background knowledge only. Searches surfaced Circular 1137 on outsourcing ([Philstar](https://www.philstar.com/business/2022/02/21/2162141/rules-outsourcing-tightened)) but not 1140. Confirm on bsp.gov.ph, and only if GCash or fintech data is in scope. |
| A lean internal data quality platform needs about 3–6 FTE | **Heuristic, unsourced** | A researcher's rule of thumb (1 lead, 2–3 data/backend engineers, 1 front-end, 0.5–1 SRE, plus a part-time DPO/InfoSec liaison). No Globe-specific or benchmark source. |
| Vendor AI effect claims (e.g., 80% MTTR reduction) | **Vendor claims** | No independent benchmarks were found ([BusinessWire](https://www.businesswire.com/news/home/20250417869977/en)). |

## Globe's constraints point to an AWS-resident, aggregates-first design

Globe has said publicly that it is AWS-first. It renewed a multi-year AWS agreement on 6 Nov 2025 covering infrastructure modernisation, real-time data and AI ([AWS press release](https://press.aboutamazon.com/aws/2025/11/globe-collaborates-with-aws-to-accelerate-digital-transformation)). Its enterprise data warehouse moved from Teradata (about 500 TB) to **Snowflake on AWS** ([Blazeclan case study](https://blazeclan.com/studies/blazeclans-data-ingestion-framework-helps-globe-telecom-inc-migrate-its-data-platform-to-snowflake-on-aws-within-3-months/)), and it moved 7.2 PB of Hadoop data to AWS ([AWS blog](https://aws.amazon.com/jp/blogs/news/how-globe-telecom-used-aws-datasync-at-scale-to-migrate-7-2-pb-of-hadoop-data)). Two footprints sit on Google: the Amdocs AI and Data Platform on Google Cloud ([Amdocs press release](https://www.nasdaq.com/press-release/globe-accelerates-digital-transformation-amdocs-ai-and-data-platform-2024-08-07)) and Google Workspace with Gemini ([Google Workspace story](https://workspace.google.com/customers/globe)). GCash/Mynt is a separate BSP-regulated entity that also uses Alibaba Cloud ([Alibaba Cloud](https://www.alibabacloud.com/en/customers/gcash?_p_lc=1)). It should stay out of scope for version 1.

In August 2026 Globe broadened its ISO 27001:2022 certification to cover corporate information security governance and core data-centre management ([Globe newsroom](https://www.globe.com.ph/about-us/newsroom/corporate/globe-iso-27001-2022-certification-digital-trust)). Any new system will therefore face ISMS controls on assets, access reviews, change management and suppliers. Several things remain unconfirmed: Globe's identity provider, its ITSM tool, its chat standard, and the AWS region of its Snowflake account. Singapore (ap-southeast-1) is likely but unverified.

On regulation, the Philippines has **no general data localisation mandate**. Cross-border processing is allowed, but Globe as personal information controller stays accountable for every processor ([NDV Law](https://ndvlaw.com/how-do-philippine-companies-comply-with-cross-border-data-transfers/); [Chambers 2025](https://gpg-pdf.chambers.com/technology-and-outsourcing-2025/47/)). The considerations to validate with Legal and the DPO office are:

- NPC Circular 2023-06 sets the security baseline for controllers and processors ([Baker McKenzie](https://resourcehub.bakermckenzie.com/en/resources/global-data-and-cyber-handbook/asia-pacific/philippines/topics/security-requirements-and-breach-notification)).
- Notifiable breaches must be reported to the NPC and affected subjects **within 72 hours** ([NDV Law](https://ndvlaw.com/when-must-philippine-companies-report-data-breaches/)).
- Under NPC Circular 2022-04, a new data processing system must be registered **within 20 days** ([NPC](https://privacy.gov.ph/?p=43514)). The DQ tool will likely count as a new processing system and need a privacy impact assessment (PIA).
- Under the SIM Registration Act (RA 11934), registration data is "absolutely confidential", and attacks on the SIM register must be reported to DICT **within 24 hours** ([Grant Thornton PH](https://www.grantthornton.com.ph/insights/articles-and-updates1/line-of-sight/smishing-no-more-an-overview-of-the-sim-card-registration-act/)).

For the tool's design, this means:

- Compute metrics in Globe's VPC and persist aggregates.
- Keep failed-row samples off by default. Where they are enabled, mask or tokenise MSISDN, IMSI and names, keep retention short, and gate access by role.
- Use aggregate-only checks on SIM-registration tables.
- Log who viewed which sample, who changed which check, and who exported what, to support the breach-response clocks.

Vercel is a good host for a Next.js UI but a poor fit for running checks. Functions cap at **800 seconds** (1,800 seconds in beta, and not with Secure Compute), and cron is only a trigger, limited to 100 jobs per project ([Vercel Functions Limits](https://vercel.com/docs/functions/limitations); [Vercel Cron](https://vercel.com/docs/cron-jobs/usage-and-pricing)). Private connectivity to AWS data requires the Enterprise Secure Compute add-on with VPC peering, priced by "contact sales" ([Secure Compute](https://vercel.com/docs/networking/secure-compute)). Vercel holds SOC 2 Type 2 and ISO 27001:2022 and has a Singapore region (sin1). There is no Philippine region. Its DPA permits transfer "to and in the United States and anywhere else in the world", and its default function region is in the US ([Vercel Security & Compliance](https://vercel.com/docs/security/compliance); [Vercel regions](https://vercel.com/docs/regions)).

Buying is not a shortcut around these constraints. Monte Carlo's hybrid mode still sends query logs to US or EU regions. Anomalo's in-VPC and Snowflake Native App modes avoid that, but any vendor is still a processor that needs a vendor risk review.

| Dimension | (a) On-prem containers | (b) Vercel UI + workers in Globe VPC | (c) Fully in Globe AWS (recommended default) |
|---|---|---|---|
| Data access | Good for DC sources; awkward for Snowflake on AWS | Workers native; UI only via API or Secure Compute peering | Native: Snowflake PrivateLink, S3, IAM roles |
| Long-running checks | Unlimited | Unlimited in workers; Vercel cron only as trigger | Unlimited (MWAA/Airflow, Step Functions, EventBridge + Fargate) |
| Compliance / procurement | No new vendor | New processor: vendor risk review, DPA, PIA update, possible NPC registration update | Reuses AWS/Snowflake contracts and ISMS scope |
| Residency | In-country | UI metadata may leave region per Vercel DPA; PII must be excluded | Region-pinned under Globe's AWS controls |
| Ops / DX | Highest ops burden, weakest DX | Best DX, two perimeters and IAM models | Medium-low ops with managed services; previews via ephemeral envs |

Option (c) is the reference pattern. It runs a Next.js standalone container behind an internal ALB with OIDC, a Postgres metadata database on RDS for definitions, results and audit, and Python workers on Fargate or EKS scheduled by Airflow/MWAA or EventBridge. Secrets live in Secrets Manager with KMS. Snowflake is reached over PrivateLink with a least-privilege `DQ_READER` role governed by masking policies. Option (b) is defensible only if the Vercel UI calls a Globe-hosted API that returns aggregates and holds no warehouse credentials.

## Design recommendations framed for gap analysis

Each recommendation has an ID and a question to ask of the existing repository, so the coming gap analysis can score it as present, partial or absent.

| ID | Recommendation | Grounding | Question to ask of the existing repo |
|---|---|---|---|
| R1 | Separate the control plane (definitions, scheduler, metric history, incidents, UI/API) from stateless executors that hold credentials and run pushed-down SQL inside Globe's VPC | Monte Carlo agent, Anomalo Query Runner, DQOps sensors | Where do warehouse credentials live, and does any UI or serverless tier connect to Snowflake directly? |
| R2 | Two monitoring tiers: metadata-only freshness, volume and schema on every table from Snowflake `INFORMATION_SCHEMA`/`ACCOUNT_USAGE` and query history; query-based checks only on tiered critical tables. Evaluate Snowflake DMFs as a cheap substrate for tier 1 | Monte Carlo table monitors; Snowflake DMFs; Databricks | Does coverage of all tables require per-table scans, and what does one full run cost in credits? |
| R3 | One versioned check-definition schema (YAML in Git, JSON-schema validated) shared by UI, API and CLI, with plan/dry-run, stable IDs, and an estimated warehouse cost per check | Monte Carlo MaC; Anomalo config-as-code; DQOps YAML | Can checks be reviewed in a PR and diffed before apply? Does renaming a check lose its history? |
| R4 | Sensor/rule separation: per-dialect SQL templates return `actual_value`; stateless rules take (value, params, history) and return (passed, expected, lower, upper) | DQOps | Is threshold logic coupled to the SQL, or can a new anomaly model be swapped in without re-scanning? |
| R5 | Append-only, partitioned metric-history and result store with deterministic hashed IDs for idempotent reruns, partitioned by source/table/month | DQOps Parquet lake; dbt v2 Parquet artifacts | Does a rerun duplicate results? Is history kept long enough (≥ 90 days) for seasonal models? |
| R6 | Partition-level incremental checks with a freshness gate (evaluate only complete partitions) and immature-bucket exclusion | DQOps partition checks; Anomalo freshness gate; Monte Carlo metric monitors | Are checks run on partial loads, and do large CDR tables get full scans? |
| R7 | Seasonal statistical baseline (median/MAD, weekday/hour buckets, STL or Holt-Winters), anomalies excluded from training, explicit "mark as normal" and exclusion windows (PH holidays, payday), one-sided volume bounds by default | Monte Carlo tuning; Uber DQM; Snowflake/Databricks | What model is used, does it learn from incident values, and can users exclude holidays? |
| R8 | Severity tiers (warning/error/fatal) where warnings don't count against KPIs and fatal can block pipelines; severity derived from asset tier × layer × downstream impact | DQOps; dbt `warn_if`/`error_if`; Monte Carlo guide | Is severity static per check, or derived from impact? |
| R9 | Incident model: grouping by schema/time/lineage, statuses kept separate from model feedback, SEV levels, merge, owner, TTA/TTR SLOs, monitor-health report to prune unactioned monitors | Monte Carlo incidents and SLOs; Uber | Does one upstream break produce one incident or dozens of alerts? |
| R10 | Ownership and domain RBAC: mandatory owner and tier per asset; audiences route by domain and priority; digests for low tiers | Monte Carlo audiences/domains | Can an asset exist unowned, and who receives its alerts? |
| R11 | Lineage from Snowflake query history parsed with a SQL parser (sqlglot suggested, not verified), plus Airflow/dbt artifacts, powering impact analysis and grouping | Monte Carlo query-log lineage; dbt artifacts | Is there any lineage, and is it used in alerting? |
| R12 | Privacy by design: aggregates by default; sampling toggle per source and column; masking/tokenisation of MSISDN/IMSI; sensitive-column registry; aggregate-only on SIM-registration tables; short sample retention | Monte Carlo sampling opt-out; Anomalo sensitive columns; DPA/RA 11934 considerations | Can any code path write raw subscriber rows to the app DB, logs or UI? |
| R13 | Enterprise controls: corporate IdP SSO + SCIM-mapped roles (Viewer, Steward, time-boxed PII-Sample Viewer, Admin); warehouse-enforced masking; append-only audit log to SIEM; Secrets Manager/KMS; IaC; SAST/container scanning; disabled third-party telemetry (`GX_ANALYTICS_ENABLED=False`) | Globe ISO 27001 scope; GX telemetry docs | Are audit events immutable and is telemetry off? |
| R14 | Host execution in Globe AWS (option c); if Vercel is used, UI-only in sin1 calling a Globe API that returns aggregates, no credentials in Vercel env, SSO + Private Deployments, logs drained to SIEM | Vercel limits and DPA | Does anything long-running depend on Vercel Functions or Cron? |
| R15 | Integrate, don't rebuild: ingest dbt `run_results.json`/v2 Parquet artifacts and GX validation results into the same history; embed only Apache-2.0 components (GX Core, `dbt-oss`, Soda Core, Elementary OSS); avoid embedding BSL DQOps | License findings | Which dependencies carry non-permissive licenses? |
| R16 | Telco-specific check library: CDR/xDR counts per switch/site/hour, MSISDN/IMSI format validity, subscriber-to-usage referential integrity, mediation-to-billing reconciliation (counts, sums, hashed diffs) | Market notes (inference); Monte Carlo comparison monitors | Are reconciliation and telco formats first-class, or ad-hoc SQL? |
| R17 | AI as a thin, optional layer after the substrate: rule suggestions from profiles with human approval, NL-to-SQL authoring, incident summaries, an MCP/read API; detection stays deterministic; LLM calls use Globe-approved endpoints (e.g., Bedrock in-region) | Monte Carlo/Anomalo/GX patterns | Does any alert depend on a non-reproducible LLM call, and where do prompts containing data go? |

## Capability checklist for the in-house tool

Priority P0 means version 1 is not credible without it. P1 is needed for an enterprise rollout. P2 is differentiating or later.

| # | Capability | Priority | Reference implementation |
|---|---|---|---|
| C-01 | Source connectors: Snowflake (primary), S3/Glue, plus Oracle/Hadoop/Amdocs-GCP as needed | P0 | Monte Carlo integration list; DQOps dialects |
| C-02 | Asset inventory with owner, domain, tier, sensitivity tags | P0 | Monte Carlo domains; Anomalo labels |
| C-03 | Metadata freshness, volume, schema-change monitors on all tables | P0 | Monte Carlo table monitors; Snowflake DMFs |
| C-04 | Column metrics: null %, distinct, min/max/mean, percentiles, top-k, format validity | P0 | Monte Carlo available metrics; Uber DQM |
| C-05 | Custom SQL and row-level validation rules mapped to DQ dimensions | P0 | Monte Carlo validation/custom SQL; dbt tests |
| C-06 | Checks-as-code (YAML in Git, schema validation, plan/dry-run, CI) | P0 | Monte Carlo MaC; Anomalo CLI |
| C-07 | Scheduler with cron, on-table-update and on-job-complete triggers, retries, backfill | P0 | Monte Carlo metric schedule options |
| C-08 | Partition-level incremental execution with a freshness gate | P0 | DQOps partition checks; Anomalo |
| C-09 | Append-only metric/result history, idempotent IDs, ≥ 90-day retention | P0 | DQOps storage |
| C-10 | Static, relative-change and seasonal-anomaly thresholds; training exclusion; mark-as-normal; holiday windows | P0 | Monte Carlo tuning; DQOps rules |
| C-11 | Severity levels with KPI semantics; pipeline-blocking "fatal" gate | P0 | DQOps; dbt |
| C-12 | Alert routing to Teams/Slack/Google Chat/email and Jira/ServiceNow, by owner and priority | P0 | Monte Carlo audiences |
| C-13 | SSO via corporate IdP, RBAC, warehouse-enforced masking | P0 | Globe ISMS expectations |
| C-14 | Immutable audit log (config changes, runs, sample views, exports) to SIEM | P0 | DPA/RA 11934 breach-response support |
| C-15 | Sampling off by default; per-source/column toggles; masking/tokenisation; sample retention | P0 | Monte Carlo sampling; Anomalo sensitive columns |
| C-16 | Secrets in Secrets Manager/KMS; no credentials in UI tier | P0 | Monte Carlo self-hosted credentials |
| C-17 | Incident lifecycle: grouping, statuses, SEV, merge, assignee, comments | P1 | Monte Carlo incidents |
| C-18 | TTA/TTR SLOs and incident KPIs per domain | P1 | Monte Carlo SLOs |
| C-19 | Source-to-target reconciliation (cross-system counts, sums, hashed diffs) | P1 | Monte Carlo comparison monitor; reladiff |
| C-20 | Segmented metrics with cardinality caps and "test before save" | P1 | Monte Carlo segmentation |
| C-21 | Table- and column-level lineage from query history and orchestrator artifacts | P1 | Monte Carlo lineage |
| C-22 | Downstream impact analysis and lineage-aware alert grouping | P1 | Monte Carlo alert detail |
| C-23 | Scorecards per domain/data product by DQ dimension, surfaced in catalog/BI | P1 | Airbnb Metis; Uber score |
| C-24 | Monitor-health report and alert feedback (useful/false/redundant) | P1 | Monte Carlo guide; Tuning agent concept |
| C-25 | Cost meter: estimated and actual warehouse credits per check and per run | P1 | Monte Carlo dry-run cost; Snowflake DMF billing |
| C-26 | Ingest dbt and GX results into the unified history | P1 | dbt artifacts; GX validation results |
| C-27 | Public API (and optional MCP endpoint) for check status and asset health | P1 | Monte Carlo MCP; Anomalo SDK |
| C-28 | Telco check library (CDR volumes per site/hour, MSISDN/IMSI validity, revenue-assurance reconciliation) | P1 | Market notes (inference) |
| C-29 | Data products (downstream asset + inherited upstream coverage) | P2 | Monte Carlo Data Products |
| C-30 | Multivariate/distribution drift on samples (classifier + SHAP style) for top-tier tables | P2 | Anomalo |
| C-31 | Deterministic triage scoring (impact from lineage/tier, likelihood from history and job failures) | P2 | Monte Carlo Triage agent (deterministic variant) |
| C-32 | LLM-assisted rule suggestion, NL-to-SQL, incident summaries with human approval | P2 | Monte Carlo/Anomalo/GX ExpectAI |
| C-33 | Unstructured-data and AI-agent output monitoring | P2 | Anomalo unstructured; Monte Carlo agent observability |
| C-34 | Compliance artefacts: PIA, NPC system registration entry, vendor risk files for any third-party host | P0 (non-code) | NPC Circular 2022-04; DPA accountability |

## Conclusion

The leading products differ less in detection than in what they make cheap and what they make quiet. Monte Carlo and Anomalo charge for coverage and hold metadata or samples in their own cloud. Globe's advantage in building is the reverse of that: broad metadata coverage is nearly free inside its own Snowflake and AWS estate, and data minimisation can be guaranteed by architecture rather than by contract. The real cost of building lies in the parts the open-source stack leaves empty: metric history, seasonal thresholds that learn the right lessons, lineage-aware incident grouping, and ownership routing. The gap analysis should weight those parts most heavily.

The 2026 market churn shows that check definitions and metric history should stay in portable formats Globe controls. That churn includes DQOps relicensing to BSL, dbt's split into open and proprietary distributions under Fivetran, Metaplane's absorption into Datadog, and the GX Cloud situation, which remains unresolved. With those in Globe's hands, any engine, vendor or AI layer can be swapped in later without re-instrumenting the estate. Before the design is finalised, the open items to confirm internally are the Snowflake region and edition, the corporate IdP and ITSM tools, whether SIM-registration tables fall in scope, and the DPO's view on sample handling.
