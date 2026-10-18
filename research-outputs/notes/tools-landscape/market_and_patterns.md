# Data Quality and Data Observability: Market Landscape, Architecture Patterns, and Build Lessons (2025-2026)

> **Raw research notes, not recommendations.** Evidence gathered on 2026-10-18 for the reports in `research-outputs/`. Vendor claims are marked as such; screenshots described here were not committed. Use these to check a claim's source, never as a decision or a plan.


Scope: the category as a whole, beyond Monte Carlo, Anomalo, DQOps, Great Expectations and dbt, which other researchers cover. Research date: 2026-10-18. Items marked **[background, not re-verified this session]** come from well-established public sources I know of but did not re-fetch; the report writer should treat them as lower confidence or verify them.

## Landscape: leading players, Gartner view, consolidation 2024-2026

### Takeaway
The market has split into three groups: (1) traditional "augmented DQ" suites (Informatica, Ataccama, Qlik/Talend), which lead Gartner's MQ; (2) modern observability vendors (Bigeye, Sifflet, Validio, Soda, Elementary, Acceldata, Lightup, Telmai, Qualytics and others); (3) native platform features (Databricks, Snowflake, AWS, Google, Microsoft) that are commoditizing basic freshness and volume monitoring. Consolidation has picked up a lot since 2024. Gartner now weights AI features and unstructured-data support heavily.

### Cited Findings
- 2025 Gartner Magic Quadrant for Augmented Data Quality Solutions (dated 10 March 2025): 12 vendors evaluated and three Leaders, Informatica, Ataccama and Qlik. Informatica says it is a Leader for the 17th time, with the highest Ability to Execute. Qlik says sixth time. Ataccama says its fourth consecutive time. All of this comes from vendor press releases, not Gartner's paywalled text. — [Nasdaq/Informatica PR](https://www.nasdaq.com/press-release/informatica-named-leader-2025-gartnerr-magic-quadranttm-augmented-data-quality); [Ataccama news](https://www.ataccama.com/news/ataccama-named-a-leader-in-the-2025-gartner-magic-quadrant-for-augmented-data-quality-solutions); [Qlik LP](https://qlik.com/us/lp/sem/di/gartner-magic-quadrant-for-augmented-data-quality-solutions)
- Movements in the 2025 MQ, as Ataccama reads them:
  - Collibra and three other unnamed vendors dropped out. Ataccama attributes Collibra's exit to trouble integrating its OwlDQ acquisition.
  - New entrants: Ab Initio (Challenger), Anomalo and Irion.
  - Qlik rose to Leader (linked to its Kyndi acquisition). IBM fell from Leader to Challenger. SAS fell to Niche Player.
  - Gartner reportedly raised the weighting of AI capabilities (rule generation, anomaly detection, automated stewardship) and made unstructured data (text, images, audio) a scored criterion.
  - Vendor interpretation. — [Ataccama blog](https://www.ataccama.com/blog/whats-new-in-the-2025-gartner-magic-quadrant-for-augmented-data-quality-solutions)
- Datadog acquired Metaplane (announced 23 April 2025). Metaplane is an ML-based data observability tool with column-level lineage. It continues as "Metaplane by Datadog" and extends Datadog's Data Jobs Monitoring and Data Streams Monitoring. Snowflake had invested in Metaplane earlier (May 2024). — [TechCrunch](https://techcrunch.com/2025/04/23/datadog-acquires-ai-powered-observability-startup-metaplane); [Datadog blog](https://www.datadoghq.com/blog/datadog-acquires-metaplane.md)
- DataKitchen's 2026 landscape counts "more than 50" commercial vendors:
  - 28 modern DQ/observability vendors: Monte Carlo, Bigeye, Anomalo, Metaplane (Datadog), Validio, Soda, Datafold, Acceldata, Lightup, Synq, Pantomath, iCEDQ, RightData, FirstEigen DataBuck, Databand (IBM), Kensu, Telmai, DataKitchen, DQLabs, Rakuten SixthSense, Sifflet, Elementary, Evidently AI, PipeRider, Qualytics, AWS Glue DQ, GX.
  - 9 traditional suites: Informatica, SAP DS/IS, Oracle EDQ, Ataccama, Qlik Talend, IBM QualityStage, SAS, Precisely.
  - 15 catalog/governance tools with DQ features: Collibra, Atlan, Alation, Purview, DataHub, OpenMetadata, etc.
  - Ownership changes it lists: Select Star is now part of Snowflake, Stemma part of Teradata, Databand part of IBM, and "Coalesce Quality (formerly SYNQ)".
  - Its other claims: the category has absorbed more than $1B of VC; average enterprise list price is about $172K/year; feature sets have "largely converged".
  - Author bias: the author is DataKitchen's CEO. — [DataKitchen 2026 landscape](https://datakitchen.io/blog/the-2026-data-quality-and-data-observability-commercial-software-landscape/)
- The same DataKitchen piece claims GX Cloud shut down on 1 June 2026 with about 30 days' notice, its assets went to FICO, and the open-source GX Core continues under Fivetran stewardship. It calls this the category's first outright shutdown. This is a single, vendor-authored source and other researchers covering GX should verify it. — [DataKitchen](https://datakitchen.io/blog/the-2026-data-quality-and-data-observability-commercial-software-landscape/)
- A summary of the same market cites as consolidation examples Datadog-Metaplane, Coalesce-SYNQ, a Fivetran-dbt Labs combination, and IBM folding Databand into watsonx.data. It also mentions Snowflake agreeing to buy Observe (IT observability, about $1B reported). This is a secondary search summary; the dbt/Fivetran detail is for the dbt researcher to confirm. — [DataKitchen](https://datakitchen.io/blog/the-2026-data-quality-and-data-observability-commercial-software-landscape/); [Futurum on Snowflake/Observe](https://futurumgroup.com/insights/5-reasons-snowflake-acquiring-observe-sets-the-tone-for-2026/)
- Soda launch week (June, year not confirmed in snippet; likely 2025):
  - Announcements: "AI powered metrics observability", collaborative data contracts, a free-forever tier, transparent pricing, and the acquisition of NannyML (ML-model monitoring).
  - Soda 4.0 pairs Soda Core 4.0, described as an "open-source data contracts engine", with Soda Cloud 4.0 (AI contract generation, anomaly detection, diagnostics). — [Soda launch blog](https://launch.soda.io/blog/one-more-thing); [Soda data contracts](https://soda.io/blog/data-contracts-implement-and-enforce-with-soda)
- Actian launched "Data Observability Agents" in public preview on 24 February 2026, alongside an MCP server that lets AI agents read quality signals and trigger resolution steps. — [BigDATAwire](https://www.hpcwire.com/bigdatawire/this-just-in/actian-introduces-data-observability-agents-for-the-agentic-ai-era/)

**Native platform features (commoditizing the basics):**
- Databricks data quality monitoring (Unity Catalog):
  - One-click anomaly detection runs across all tables in a schema.
  - Freshness: a per-table model of commit history predicts the next commit; an unusually late commit marks the table "stale".
  - Completeness: rows committed in the last 24 hours are compared with a predicted range; falling below the lower bound flags the table.
  - "Intelligent scanning" prioritizes tables by popularity and downstream usage, and scans low-impact tables less often.
  - Requires serverless compute. Results land in `system.data_quality_monitoring.table_results`.
  - Status is Public Preview or Beta depending on the docs page. Event-time freshness is not supported in the current version. — [Databricks docs](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/); [GCP variant](https://docs.databricks.com/gcp/en/lakehouse-monitoring/anomaly-detection)
- Snowflake Data Metric Functions (DMFs):
  - Serverless functions log measurements to an event table.
  - System DMFs in `SNOWFLAKE.CORE` cover ROW_COUNT, FRESHNESS, null %, duplicates and more. Custom DMFs can encode business rules.
  - They attach at schema level, with per-object overrides.
  - Anomaly detection is turned on with `ANOMALY_DETECTION = TRUE` on a DMF association. It flags values below a predicted lower bound, and sensitivity is adjustable.
  - Edition and billing: Enterprise Edition feature; billed only for scheduled serverless runs, shown as a separate line item; one guide says the default schedule is hourly. — [Snowflake docs: schema-level](https://docs.snowflake.com/en/user-guide/data-quality-schema-level); [anomaly status view](https://docs.snowflake.com/en/sql-reference/local/data_quality_monitoring_anomaly_detection_status); [adesso guide](https://silicon-saxony.de/en/adesso-implementing-data-quality-monitoring-using-data-metric-functions-in-snowflake/)
- AWS Glue Data Quality is built on Deequ and uses the DQDL rule language. Google Dataplex offers auto data quality scans and profiling on BigQuery. Microsoft Purview has a Data Quality module with rules and scores. **[background, not re-verified this session]**

### Inferences
- For an in-house build, basic freshness and volume anomaly detection is now table stakes and is effectively free or cheap in Databricks, Snowflake and Glue. The tool's value should sit above that layer: cross-platform coverage (Globe likely runs a heterogeneous estate of Hadoop/Teradata/Oracle/cloud), business-rule checks, source-to-target reconciliation, ownership and incident workflow, scorecards, and lineage-based impact analysis.
- Vendor exit and shutdown risk is real (GX Cloud, Metaplane absorbed into Datadog). That strengthens the case for open, portable check definitions (YAML/SQL) and storing metric history in the tool's own warehouse tables.
- Gartner's weighting suggests enterprise buyers and auditors increasingly expect AI-assisted rule suggestion and unstructured-data coverage. Plan for both, even if v1 does neither.

### Gaps
- I could not access Gartner's full 2025 MQ text or a 2026 MQ/Market Guide. If a 2026 edition exists, I did not find it. Visionary and Niche placements beyond those named above are unconfirmed.
- I found no confirmed 2024-2026 acquisitions of Bigeye, Sifflet, Acceldata, Validio, Lightup, Telmai or Qualytics. The searches returned nothing either way.
- I did not obtain an exact Soda 4.0 GA date.

## Core capability taxonomy

### Takeaway
Mature tools combine two models: classic DQ dimensions with explicit rules (completeness, validity, uniqueness, timeliness, consistency, accuracy), and observability "pillars" with learned monitors (freshness, volume, schema, distribution, lineage). They wrap both in profiling, contracts, reconciliation, incident management and scoring. Leading practice is learned monitors for broad coverage plus explicit rules and contracts on critical assets.

### Cited Findings
- Practitioners and vendors describe observability (learned monitors) and quality testing (explicit tests) as complementary, not substitutes. Hand-authored tests limit coverage. — [DataKitchen](https://datakitchen.io/blog/the-2026-data-quality-and-data-observability-commercial-software-landscape/)
- Native platforms define the minimal pillars this way. Databricks: freshness (commit-time model) and completeness (24h row count against a predicted range). Snowflake: ROW_COUNT, FRESHNESS, nulls, duplicates, plus custom DMFs. — [Databricks](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/); [Snowflake](https://docs.snowflake.com/en/user-guide/data-quality-schema-level)
- Uber's DQM computes partition-level column metrics per date partition. Numeric columns get avg, median, min and max; string columns get unique count and missing count. These metrics feed per-column time series. — [Uber Eng: Monitoring Data Quality at Scale](https://www.uber.com/en-NZ/blog/monitoring-data-quality-at-scale/)
- Scoring and scorecards:
  - Uber gives each principal component a 0-4 severity and sums the top 3 into a table quality score, shown daily.
  - Airbnb ties data quality scores to each warehouse asset in its Metis catalog. Scores are computed offline and ingested into the metadata store.
  - Airbnb's "Midas" certified critical data models through reviews by designated Data/Metrics Architects. — [Uber](https://www.uber.com/en-NZ/blog/monitoring-data-quality-at-scale/); [Airbnb Metis](https://medium.com/airbnb-engineering/metis-building-airbnbs-next-generation-data-management-platform-d2c5219edf19); [Airbnb DQ Part 2](https://medium.com/airbnb-engineering/data-quality-at-airbnb-870d03080469)
- Data contracts have become a mainstream feature. Soda repositioned Soda Core 4.0 as an open-source "data contracts engine", with AI contract generation in Soda Cloud. — [Soda](https://soda.io/blog/data-contracts-implement-and-enforce-with-soda)
- Reconciliation and data diff: Datafold's open-source data-diff (cross-database table comparison via hashed segment bisection) was archived on 17 May 2024 and is no longer supported. reladiff is described as the continuation fork by the original engine author. — [GitHub datafold/data-diff](https://github.com/datafold/data-diff); [dataworkers.io (secondary)](https://dataworkers.io/resources/datafold-data-diff-alternatives/)
- A January 2026 telco case study (OTE, Greece) used the DATAMITE Horizon Europe framework to automate profiling and business-rule validation across heterogeneous systems. It reports catching schema changes, duplication and broken workflows before downstream impact. — [IntechOpen chapter](https://www.intechopen.com/chapters/1233458)

### Inferences
- Recommended capability set for Globe's tool, in priority order:
  1. Asset inventory and ownership.
  2. Metadata-based freshness, volume and schema-change monitors on all tables.
  3. Partition-level column profiling (null %, distinct, min/max/mean, and for categoricals the top-k value distribution) on tiered tables.
  4. Declarative rule checks (YAML/SQL) mapped to DQ dimensions.
  5. Source-to-target reconciliation (counts, sums, hashed diffs). This is critical in telco for CDR/billing/usage flows across mediation, BSS and the warehouse.
  6. Incident lifecycle (group, assign, ack, resolve, feedback).
  7. Dimension-based scorecards per domain or data product.
  8. Column-level lineage for impact analysis.
- Telco-specific checks the taxonomy implies: CDR/xDR record counts per switch/site per hour (partition-level volume), MSISDN/IMSI format validity, referential integrity between subscriber master and usage, and revenue-assurance reconciliation between mediation and billing totals.

### Gaps
- I found no authoritative standard (for example DAMA or ISO 8000) text in this session, so the dimension definitions are uncited.
- I found no published vendor-neutral telco case study from Soda, Telmai or Acceldata.

## Technical approaches: monitoring modes, anomaly detection, alert fatigue, RCA, cost

### Takeaway
There are three monitoring modes. Metadata-only monitoring (commit history, information_schema, query logs) is cheap and covers everything. SQL pushdown profiling scans partitions incrementally on tiered tables. In-pipeline checks act as circuit breakers on critical paths. Anomaly detection in production systems is mostly classical time-series methods (exponential smoothing/Holt-Winters, seasonal decomposition, PCA aggregation, robust z-score/MAD), not deep learning. The hard parts are seasonality, not learning from incident values, and aggregating alerts.

### Cited Findings
- **Metadata-first, impact-prioritized scanning.**
  - Databricks builds freshness models from table commit history alone.
  - It prioritizes scans by table popularity and downstream usage, and scans less critical tables less often. This is a direct cost-control pattern.
  - It runs on serverless compute and writes results to a system table. — [Databricks docs](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/)
- **Scheduled pushdown metrics.** Snowflake DMFs run on a schedule (default hourly per one guide) on serverless compute billed separately. Cost scales with schedule frequency × number of attached DMFs. — [adesso](https://silicon-saxony.de/en/adesso-implementing-data-quality-monitoring-using-data-metric-functions-in-snowflake/); [Kanerika (secondary)](https://kanerika.com/blogs/snowflake-data-quality/)
- **Uber DQM algorithm.**
  - Column metrics per partition are compressed with PCA. Top components explain more than 90% of variation.
  - Each component is forecast one step ahead with Holt-Winters additive exponential smoothing. Prediction intervals give a 0-4 severity.
  - Table score = unweighted sum of the top 3 components, so high-variance components don't mask others.
  - Backtested on hundreds of tables. Each point is projected with and without the latest value to amplify sudden changes.
  - Stack: PySpark on HDFS, tens of thousands of Hive/Vertica tables.
  - Key lesson: metric-level alerts were too noisy and caused alert fatigue. Uber recommends table-level alerts plus a curated set of metric-level alerts.
  - Uber also found that clustering quality scores over time can reconstruct lineage, because related tables degrade together. Automated RCA via lineage was listed as future work. — [Uber Eng](https://www.uber.com/en-NZ/blog/monitoring-data-quality-at-scale/)
- **Uber D3 (2023).** Automated column-level drift detection with a compute layer, an anomaly-detection component and an orchestrator. It prioritizes important columns from offline usage and needs no manual thresholds. I could not fetch the primary page (404 on two URLs), so this is from search summaries. — [Uber D3 blog](https://www.uber.com/blog/d3-an-automated-system-to-detect-data-drifts)
- **Threshold design lessons.**
  - Plain standard-deviation thresholds miss incidents on highly seasonal metrics.
  - For flat metrics that shift during an incident, thresholds must not adapt to the bad values; keep alerting until resolved, which means excluding anomalous points from training.
  - Detectors must tolerate missing data. — [Bigeye on Uber](https://bigeye.com/blog/lessons-learned-from-uber-designing-an-intelligent-data-quality-monitor)
- **Snowflake and Databricks detection shape.** Both default to one-sided lower-bound detection for volume (too few rows), with tunable sensitivity. This is a pragmatic way to cut false positives from benign growth. — [Snowflake](https://docs.snowflake.com/en/sql-reference/local/data_quality_monitoring_anomaly_detection_status); [Databricks](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/)
- **Alert-fatigue mitigations (Monte Carlo practice guide).**
  - Require every monitor to map to a concrete action.
  - Prune monitors that stay unactioned or are marked "expected".
  - Focus on key data products and their upstream lineage (example: one dashboard fed by about 50 upstream tables).
  - Route alerts to the team that can fix them: platform owns ingestion, domains own business metrics.
  - Tier severity by layer (bronze schema change = info; gold revenue table = SEV-1) and auto-escalate by downstream blast radius.
  - Tier delivery: real-time PagerDuty/Slack for critical tables, daily digest for important ones, weekly for experimental ones.
  - The article gives no empirical statistics. — [Monte Carlo](https://montecarlo.ai/blog-alert-fatigue-monitoring-strategy)
- **Typical false-positive sources.** Benign column reordering triggering schema alerts, weekend volume dips, and static manual thresholds going stale as tables grow. — [Medium DataOps](https://medium.com/@manik.ruet08/alert-fatigue-in-dataops-how-to-build-a-smarter-alerting-system-0e70750fb9cf); [Validio](https://validio.io/blog/avoid-data-quality-and-observability-alert-fatigue)
- **Academic foundations [background, not re-verified this session].**
  - Amazon Deequ: "Automating Large-Scale Data Quality Verification" (Schelter et al., VLDB 2018). Declarative constraints are computed as incrementally updatable metrics on Spark, with constraint suggestion from profiling and anomaly detection on metric history. — http://www.vldb.org/pvldb/vol11/p1781-schelter.pdf
  - Google TFDV: "Data Validation for Machine Learning" (Breck et al., SysML 2019). Schema inferred from data, then train/serve skew and drift detection. — https://mlsys.org/Conferences/2019/doc/2019/167.pdf

### Inferences
- Recommended architecture for Globe:
  - (a) A metadata collector (information_schema / commit logs / Hive metastore / query logs) on a short interval for freshness, volume and schema across all assets. This is cheap.
  - (b) Incremental partition-level SQL pushdown profiling (only the newest partition, with optional sampling on very large CDR tables) for tier-1 and tier-2 tables.
  - (c) In-pipeline circuit-breaker checks (block publish on failure) only for tier-1 outputs such as billing, regulatory reports and revenue dashboards.
  - (d) A metric-history store in the tool's own tables, which allows algorithm swaps without re-scanning.
- Recommended detection stack:
  - Start with robust baselines: rolling median/MAD z-score, day-of-week and hour-of-day seasonal buckets, and an STL or Holt-Winters forecast with prediction intervals.
  - Exclude flagged points from training.
  - Default to one-sided bounds for volume and add two-sided bounds where business logic requires them.
  - Aggregate to table-level severity (Uber-style).
  - Add Prophet or changepoint detection later for metrics with holidays. Philippine holidays and payday cycles (15th/30th) will matter for telco usage and load volumes. These are my inferences, not cited.
- Cost control: tiering, incremental partition scans, a shared single-pass profiling query per table (compute all column metrics in one SELECT), scheduling scans after pipeline completion events rather than on fixed cron, and budget caps per scan.

### Gaps
- I found no rigorous public benchmark comparing Prophet, STL, Isolation Forest and changepoint methods on DQ metric time series. Vendor ML claims are unverifiable.
- I found no primary source on segment drill-down RCA algorithms (for example, which dimension slice explains a metric shift). Commercial tools advertise this but did not document it in what I read.
- I could not retrieve the primary Uber D3 page or Airbnb Part 2 text (403/404).

## AI/LLM trends

### Takeaway
Through 2025-2026 the category has shifted toward AI-generated rules and contracts, natural-language check authoring, MCP servers that expose quality signals to AI agents, and agentic triage and root-cause explanation. Gartner now scores AI capabilities and unstructured-data support explicitly.

### Cited Findings
- Gartner's 2025 MQ reportedly increased the weighting of AI capabilities (rule generation, anomaly detection, automated stewardship) and added unstructured-data handling as a criterion. Vendor interpretation. — [Ataccama](https://www.ataccama.com/blog/whats-new-in-the-2025-gartner-magic-quadrant-for-augmented-data-quality-solutions)
- Actian's Data Observability Agents (public preview, February 2026) run checks as data lands, flag anomalies and explain root causes in plain language. Its MCP server supports reads and write actions that trigger resolution. — [BigDATAwire](https://www.hpcwire.com/bigdatawire/this-just-in/actian-introduces-data-observability-agents-for-the-agentic-ai-era/)
- Soda Cloud 4.0 advertises AI contract generation and "self-driving" anomaly detection and diagnostics. Soda acquired NannyML, which extends into ML-model monitoring. — [Soda launch](https://launch.soda.io/blog/one-more-thing)
- Monte Carlo's "Agent Observability" connects OpenTelemetry agent traces to the health of the data feeding agents (stale context traced to a late table or a schema change). Secondary comparison source. — [Confident AI](https://www.confident-ai.com/knowledge-base/compare/best-ai-agent-observability-tools-2026)
- DataKitchen recommends evaluating tools by their MCP tool count and how many of those tools can write, arguing that tools must be callable by AI assistants. Vendor-authored. — [DataKitchen](https://datakitchen.io/blog/the-2026-data-quality-and-data-observability-commercial-software-landscape/)
- In Elastic's 2026 report, 85% of organizations use some form of GenAI for observability (IT observability broadly). — [The New Stack](https://thenewstack.io/elastic-agentic-observability-enterprise-adoption/)

### Inferences
- Practical, low-risk LLM uses for an in-house tool:
  - Rule suggestion from profiles plus column names and comments, with human approval.
  - Natural-language-to-SQL check authoring.
  - Incident summaries that combine anomaly, lineage, recent schema and code changes.
  - An MCP or API endpoint so analysts' AI assistants can ask "is this table healthy?".
- Keep detection deterministic and statistical. Use LLMs for authoring and explanation, not as the alert trigger, to avoid non-reproducible alerts.

### Gaps
- I found no independent evidence on the accuracy or false-positive impact of agentic RCA in data observability. Claims are vendor-only.
- I found little concrete technical material on monitoring unstructured data or LLM outputs inside DQ tools. Gartner scores it, but the methods were not documented in the sources I found.

## Failure modes and lessons learned; what makes enterprise rollouts succeed

### Takeaway
Most failures are organizational, not algorithmic: too many low-value monitors producing noise, alerts routed to a central team that can't fix them, unclear ownership, certification processes that don't scale, and unbounded scan costs. Successful programs tier assets, route by ownership, aggregate alerts, and attach scores to the catalog where people already look.

### Cited Findings
- Uber: metric-level alerting caused alert fatigue; table-level aggregation reduced alert volume substantially. — [Uber](https://www.uber.com/en-NZ/blog/monitoring-data-quality-at-scale/)
- Airbnb's Midas certification (expert review of data models) was later criticized as non-scalable. It met resistance and put a disproportionate burden on frontline data teams. Airbnb followed with "Wall", a YAML-config check framework publishing results to Kafka. The Wall details come from a secondary source. — [Packt CloudPro (secondary)](https://packtcloudpro.substack.com/p/the-hidden-platform-lesson-behind); [Airbnb DQ Part 1](https://medium.com/airbnb-engineering/data-quality-at-airbnb-e582465f3ef7)
- Central platform teams receiving every alert is a common anti-pattern. Ownership should follow who can actually fix the issue. — [Monte Carlo](https://montecarlo.ai/blog-alert-fatigue-monitoring-strategy)
- Unowned alerts get ignored. Noisy alerts train responders to treat "no action" as normal, and periodic alert audits break the cycle. — [LogicMonitor](https://www.logicmonitor.com/blog/network-monitoring-avoid-alert-fatigue); [RunBook Academy](https://runbook.academy/courses/observability/lessons/04-alert-fatigue/)
- Every resolved alert should capture quick feedback (useful, false, redundant) to tune thresholds and ownership. — [Medium DataOps](https://medium.com/@manik.ruet08/alert-fatigue-in-dataops-how-to-build-a-smarter-alerting-system-0e70750fb9cf)
- Static, manually defined validations go stale and generate false positives as data grows. — [Validio](https://validio.io/blog/avoid-data-quality-and-observability-alert-fatigue)
- "Tools multiply good practice. They do not replace it." Since features have converged, ownership model, exit risk and portability matter more in evaluation. — [DataKitchen](https://datakitchen.io/blog/the-2026-data-quality-and-data-observability-commercial-software-landscape/)
- In IT observability broadly, 57% of telecom respondents experience high-impact outages weekly or more often. This is system observability, not DQ, but it shows telco operational pressure. — [New Relic report](https://www.businesswire.com/news/home/20240528864623/en/New-Relic-Report-Shows-IT-and-Telecommunications-Industries-are-Embracing-Observability-to-Capitalize-on-AI-and-Cloud)

### Inferences
- Design requirements derived from these lessons:
  - A mandatory owner and tier on each monitored asset.
  - Severity derived from tier × layer × downstream impact.
  - Incident grouping, so one upstream break produces one incident, not 50 alerts.
  - Digest delivery for non-critical tiers.
  - Feedback buttons that feed threshold tuning, plus a "monitor health" report (alerts never actioned → prune).
  - A per-scan cost meter.
  - Scores surfaced in the existing catalog or BI tools, not only in a new UI.
- Avoid Midas-style heavyweight certification as the main mechanism. Use lightweight, self-service YAML/SQL checks with automated baseline coverage instead.

### Gaps
- I found few practitioner-authored postmortems with quantified alert-fatigue or cost-blowup numbers. Most sources are vendor content.
- I found no public Philippine or Southeast Asian telco DQ platform case study (Globe, Smart, Telkomsel, Singtel and others).

## Open-source building blocks

### Takeaway
The reusable OSS stack in 2026: Soda Core (now a contracts engine), Deequ/PyDeequ (Spark-scale metrics, the basis of AWS Glue DQ), GX Core (continuing despite GX Cloud's reported shutdown), Elementary (dbt-native observability), Pandera (dataframe validation), OpenLineage/Marquez (runtime lineage), sqlglot (SQL parsing for column-level lineage), and reladiff (successor to the archived data-diff). Several older projects are archived or inactive.

### Cited Findings
- Soda Core:
  - Open source, about 2.4k GitHub stars.
  - Connectors: Snowflake, BigQuery, Databricks, PostgreSQL, Redshift, DuckDB and more.
  - 50+ built-in check types.
  - v4 is positioned as a data contracts engine. — [datalakehousehub (secondary)](https://datalakehousehub.com/knowledgebase/soda); [Soda](https://soda.io/blog/data-contracts-implement-and-enforce-with-soda)
- Datafold data-diff was archived on 17 May 2024 and is unsupported. reladiff is the fork by the original engine author. — [GitHub](https://github.com/datafold/data-diff); [dataworkers.io](https://dataworkers.io/resources/datafold-data-diff-alternatives/)
- GX Core reportedly continues under Fivetran stewardship after the GX Cloud closure. Single vendor-authored source. — [DataKitchen](https://datakitchen.io/blog/the-2026-data-quality-and-data-observability-commercial-software-landscape/)
- [background, not re-verified this session]:
  - Deequ (Scala/Spark) and PyDeequ (Python wrapper) are maintained by AWS Labs: constraint verification, suggestion, metric repositories, anomaly detection strategies.
  - OpenLineage is an LF AI & Data project defining a lineage event spec, with Spark, Airflow, dbt and Flink integrations; Marquez is its reference metadata server.
  - sqlglot provides a pure-Python SQL parser/transpiler across 20+ dialects with a `lineage` module for column-level lineage.
  - Apache Griffin (eBay-origin) has had little activity in recent years, and re_data's open-source development has largely stalled.
  - Verify the status of Griffin and re_data before relying on them.

### Inferences
- A pragmatic build for Globe:
  - Use Soda Core or Deequ as the check-execution engine. Deequ is the better fit if the heavy data is on Spark/Hadoop; Soda Core if it's SQL warehouses.
  - Write your own metric-history store, anomaly service, incident and ownership layer, and UI. These are the differentiated parts, and OSS coverage there is weak.
  - Emit and consume OpenLineage events from Airflow/Spark, and use sqlglot to parse warehouse query logs and view definitions for column-level lineage.
  - Use reladiff, or a home-grown hashed segment-checksum approach, for cross-system reconciliation.
- Avoid hard dependency on any single vendor-led OSS project's roadmap (data-diff archive, GX Cloud shutdown). Wrap engines behind your own check-definition schema.

### Gaps
- I did not verify in this session the 2026 maintenance status of Apache Griffin, re_data, Elementary OSS, Pandera or PyDeequ (release cadence, Spark 3.5/4 support).
- I did not find a confirmed current GitHub status for reladiff.
