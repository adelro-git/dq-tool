# Platform-native and enterprise DQ tools: how results are presented, plus executive scorecard practice (as of Oct 2026)

> **Raw research notes, not recommendations.** Evidence gathered on 2026-10-18 for the reports in `research-outputs/`. Vendor claims are marked as such; screenshots described here were not committed. Use these to check a claim's source, never as a decision or a plan.


Evidence labels used throughout:
- **[SEEN]**: I viewed the screenshot myself (files were not committed to this repo (vendor-owned images; see each finding's source link)).
- **[DOC]**: the feature is described in text in official docs. I did not see it.
- **[INF]**: my own inference.

Screenshot dates: the Snowflake shots come from a Classmethod blog dated 2025-09-10, while the tab was still in PREVIEW. The Databricks shots come from current docs. One Databricks dashboard shows "Last updated: Jan 26, 2026".

## 1. Snowflake (Snowsight) data quality presentation: views, states, values vs expectations, anomalies, notifications

### Takeaway
Snowflake has two levels of DQ UI:
- **Per-object Data Quality tab** (Catalog » Explorer » object » Data Quality, with sub-tabs Data Profile and Monitoring). It groups checks by quality dimension and shows a status pill per group ("All checks passed" / "N check(s) failed"). Each check row shows the latest value, a trend, and "x/y passed". A side panel adds the expectation expression, a Failed/Passed status, Run History, Impacted Assets (downstream lineage) and "View failed records".
- **Account-level "Data quality" dashboard** (Governance & security » Data quality). It shows "Healthy tables" % and "Monitoring coverage" % cards, incident counts by type (Volume anomaly / Freshness anomaly / Failed expectations), and a per-schema table with coverage and % unhealthy bars.

Neither level has a composite numeric "DQ score". Snowflake reports health as counts and percentages of tables or checks passing.

### Cited Findings
**Per-object Data Quality tab ([DOC] plus [SEEN] on 2025-09 preview shots)**
- **Navigation and setup.** Path is Catalog » Explorer » object » **Data Quality** tab » **Monitoring**. If no DMFs exist, **Set up** opens a prepopulated worksheet. Users see only the DMFs their privileges allow. — [Snowflake: Monitoring data quality checks in Snowsight](https://docs.snowflake.com/en/user-guide/data-quality-ui-monitor)
- **Page layout.** [DOC]
  - A failed-check count sits at the top.
  - A **"Checks by dimension"** widget turns a group red if at least one DMF in it failed.
  - **Quality Dimensions** groups the DMFs. System DMFs are grouped by category, e.g. **Accuracy** (NULL_COUNT, BLANK_COUNT) and **Volume**. Custom DMFs go under **Custom**.
  - There is one row per DMF association, and the same DMF can appear on several columns.
  - Each row shows "the latest result and a seven-day trend".
  - A **Run Schedule** widget shows the DATA_METRIC_SCHEDULE.
  - Source: [Snowflake docs](https://docs.snowflake.com/en/user-guide/data-quality-ui-monitor)
- **Group cards (screenshot, Sep 2025 preview, dark theme).** [SEEN]
  - Each dimension group is a collapsible card: "Accuracy · 1 metric", "Statistics · 0 metrics", "Uniqueness · 1 metric", "Custom · 7 metrics".
  - The card header shows a green-check "All checks passed" or a red-X "1 check failed" / "4 checks failed".
  - Inside the card, metric chips (e.g. "Null count", "• Duplicate count" with a red dot) or a dropdown for many custom DMFs, then "1 column associated".
  - Table columns are COLUMN | <METRIC VALUE, e.g. NULL COUNT 6,653,934> | TREND HISTORY (a sparkline; only one point here) | QUALITY CHECKS ("✓ 1/1 passed" or red "• 0/1 passed").
  - The raw metric value and the check outcome sit side by side. A large null count can still be "passed" because the expectation decides pass/fail, not the value itself. — [Classmethod blog, 2025-09-10](https://dev.classmethod.jp/articles/snowsight-data-quality-tab/)
- **Side panel (screenshot, same date).** [SEEN]
  - Title "Duplicate count for BIKEID".
  - Buttons **View Lineage** and **View failed records**.
  - Two KPI tiles: "17,678 · Current duplicate count" and "Sep 10, 2025, 10:40:17 AM · Last run".
  - **Quality Checks** table with NAME | EXPRESSION | STATUS, e.g. "EXPECT_ZERO_DUPLI… | VALUE = 0 | ✗ Failed".
  - **Impacted Assets** with search and filters, listing NAME/DATABASE/SCHEMA. Empty state reads "No downstream assets impacted". — [Classmethod blog](https://dev.classmethod.jp/articles/snowsight-data-quality-tab/)
- **Side panel details from docs.** [DOC]
  - **View failed records** appears only for system DMFs whose value is > 0. It opens a worksheet calling SYSTEM$DATA_METRIC_SCAN.
  - **Arguments** links to the Columns tab.
  - **Impacted Assets** lists only downstream objects that contain the column's data, for single-column DMFs.
  - **Run History** graphs results over time.
  - Source: [Snowflake docs](https://docs.snowflake.com/en/user-guide/data-quality-ui-monitor)
- **Data Profile sub-tab.** Per-column NULL COUNT, NULL %, MIN, MAX and TOP VALUES (the top 4). — [Classmethod](https://dev.classmethod.jp/articles/snowsight-data-quality-tab/). The tab launched in preview on 2025-09-09. — [Snowflake release note](https://docs.snowflake.com/en/release-notes/2025/other/2025-09-11-dq-ui)
- **Setup UX.** [DOC]
  - **Suggested quality checks** (Cortex) appear in an editable column, "What should the result be?".
  - **Build checks manually** opens a "Set up a quality check" dialog.
  - Schedule options are "Interval-based timing", "Select schedule" and "Trigger-based execution".
  - Source: [Snowflake setup docs](https://docs.snowflake.com/en/user-guide/data-quality-ui-setup)

**Expectations (pass/fail model)**
- **How an expectation works.** It is a Boolean expression over `VALUE`, the DMF result, e.g. `VALUE = 0`. TRUE means pass.
- **Where results are stored.** In the event table `SNOWFLAKE.LOCAL.DATA_QUALITY_MONITORING_RESULTS_RAW`, record types `EVALUATION_RESULT` and `EXPECTATION_VIOLATION_STATUS`. The flattened view is `DATA_QUALITY_MONITORING_EXPECTATION_STATUS`.
- **States.** Only pass and fail (violated TRUE/FALSE). I found no documented "warn" tier.
- Source: [Snowflake expectations docs](https://docs.snowflake.com/en/user-guide/data-quality-expectations)

**Anomaly detection**
- **Coverage and settings.** Anomaly detection applies to ROW_COUNT and FRESHNESS DMFs (`ANOMALY_DETECTION = TRUE`). Sensitivity can be LOW, MEDIUM (default) or HIGH.
- **Training state.** While training, `anomaly_detection_status = TRAINING_IN_PROGRESS`. Training needs at least 2 weeks of data, and 60 days is recommended.
- **Stored fields.** Each result stores `evaluation_result` (value plus a boolean anomaly flag), `upper_bound`, `lower_bound` and `forecast`.
- **Not documented.** The docs page does not describe a Snowsight chart rendering for anomalies.
- Source: [Snowflake anomaly docs](https://docs.snowflake.com/en/user-guide/data-quality-anomaly)

**Account-level dashboard ([DOC], preview, Enterprise Edition)**
- Path is Governance & security » Data quality. It needs the DATA_QUALITY_MONITORING_VIEWER or _ADMIN app role.
- **Filters and context.** A Time range filter (default 7 days) and the warehouse shown in the top right.
- **Summary cards:**
  - **Healthy tables**: % of monitored tables with no open incidents, with a healthy vs unhealthy bar.
  - **Monitoring coverage**: % of account tables with at least one active DMF association, with a monitored vs unmonitored bar.
- **"Data quality incidents" section.** Open counts by type (**Volume anomaly**, **Freshness anomaly**, **Failed expectations**) plus a recent-incidents preview. An **Investigate** button opens Cortex Code with the incident context, and multi-select is possible. The incidents page columns are Metric name, Database, Schema, Object, Column ("—" for table-level metrics).
- **"Monitored schemas" table.** Columns are Database, Schema, Volume anomaly, Freshness anomaly, Failing expectations, Monitoring coverage (bar), % unhealthy tables (bar).
- **Manage Monitors and recommendations.** The Manage Monitors dialog has per-schema Freshness/Volume toggles. A recommendations banner reads "New schemas recommended for monitoring", with statuses "Recommended" or "No access". Recommendations are based on table popularity.
- Source: [Snowflake dashboard docs](https://docs.snowflake.com/en/user-guide/data-quality-centralized-dashboard)

**Notifications and alerts**
- **Triggers.** A notification is sent "whenever an expectation is violated or Snowflake identifies an anomaly".
- **Scope and throttling.** Enabled per database, with opt-out per DMF association. A `cooldown_hours` throttle applies (the docs example is 4 h).
- **Channels.** Email (verified recipients) or a WEBHOOK integration (Slack example body `{"text": "SNOWFLAKE_WEBHOOK_MESSAGE"}`). The 9.40 release notes also mention Teams and PagerDuty.
- **Content.** The docs say only that notifications "include metadata that identifies the object and its associated DMF". The payload is not documented.
- Sources: [Snowflake notifications](https://docs.snowflake.com/en/user-guide/data-quality-notifications); [9.40 release notes](https://docs.snowflake.com/release-notes/2025/9_40)
- **Alert templates.** Snowsight Alerts include a DATA_QUALITY template group with an "anomaly detection" alert and an "expectation violations" alert. — [Alerts in Snowsight](https://docs.snowflake.com/en/user-guide/alerts-ui)

### Inferences
- [INF] Snowflake's convention to borrow is a dimension-grouped card with a single roll-up pill ("All checks passed" / "N failed"). Inside each card, the raw metric value, a trend sparkline and the "x/y passed" check verdict are separate columns. Drill-down goes into a side panel with the expression, status, history, impacted downstream assets, and a failed-records button.
- [INF] The account dashboard pairs **Healthy %** with **Coverage %** as the two headline numbers. This is a deliberate guard against a high health % over a small monitored footprint.
- [INF] An in-house tool on Snowflake can reproduce Snowflake's semantics by reading `DATA_QUALITY_MONITORING_EXPECTATION_STATUS` and `..._ANOMALY_DETECTION_STATUS`, including the forecast and upper/lower bounds needed for band charts.

### Gaps
- No public screenshot found for the 2026 account-level dashboard or for any Snowsight anomaly band chart.
- The exact notification email or webhook payload is undocumented.
- Not-run and stale states in the per-object UI are undocumented.
- The 2025-09 screenshots are from the preview, so the current GA UI may differ.

## 2. Databricks data quality monitoring (anomaly detection) and Lakehouse/data profiling dashboards

### Takeaway
Databricks presents table health as a categorical status with six values: Healthy, Unhealthy, Training, Error, Excluded, Not enabled. The status appears as a dot or pill wherever the table appears: schema table lists, lineage-graph nodes, and the table Quality tab. Each check is drawn as an "observed vs predicted range" chart. The roll-ups are % healthy tables per schema, catalog and metastore. Incident lists carry Impact (High/Medium/Low, from downstream lineage and query volume), a lineage-derived Root Cause, and triage actions.

### Cited Findings
- **Status vocabulary.** [DOC]
  - **Healthy**: all checks passed in the latest scan.
  - **Unhealthy**: freshness or completeness flagged.
  - **Training**: baseline still being built, so no verdict yet.
  - **Error**: monitoring hit an error on this table.
  - **Excluded**: table explicitly excluded.
  - **Not enabled**: monitoring is off for the schema.
  - The first indicator can take up to 2 weeks to appear.
  - Indicators show on schema/table pages and on lineage graph nodes.
  - Source: [Databricks anomaly detection docs](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/)
- **Table Quality tab (screenshot).** [SEEN]
  - Tabs: Overview, Sample Data, Details, Permissions, Policies, History, Lineage, Insights, **Quality**.
  - Header: "Data Quality Monitoring", with Configure.
  - Status line: "Anomaly detection ● Healthy as of 1 hour ago".
  - **Freshness section** ("● Commit freshness is as expected", "Last commit Fri Jun 26 06:20 AM"). A strip chart of past commits as green ticks, a blue **Predicted commit window** and a yellow **Stale window**, with a dashed "Last evaluated" marker. Text below reads "Predicted commit window Jun 26 06:20 AM—Jun 26 01:50 PM".
  - **Completeness section** ("● Completeness is as expected"). A row-count line chart, "15,392 rows in the last 24 hours". A green past-volume line sits inside a light-blue shaded **Predicted daily volume range** band, with a "Last evaluated" dot. Footer reads "Predicted daily volume 14,538 rows".
  - **Not-measured state.** The "Null values (Beta)" section shows "No data yet. Only high impact tables are monitored for null value anomalies."
  - Source: [Databricks docs image](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/)
- **Data Quality Monitoring results page (screenshot).** [SEEN]
  - Filters: Catalog and Schema (All Schemas).
  - Card "Data Quality 99.95% healthy", with a green/red bar and the legend "4,371 tables healthy / 2 tables unhealthy".
  - Card "Schemas Monitored 8", with a bar labelled "0.11% of all schemas". This is the coverage tile.
  - "All Monitored Tables", with the note "Tables are automatically scanned at the same frequency as their updates".
  - Filter chips: "Unhealthy 2 | Healthy 4,371 | Error 568", plus an "Excluded tables" button.
  - Table columns: Schema | Table | Status (red "Unhealthy" pill) | First Detected | Reason (e.g. "Stale") | Root Cause | Impact (Medium) | Scan Frequency | Results ("Review").
  - Source: [Databricks docs](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/)
- **Column meanings and triage actions.** [DOC]
  - Impact = High/Medium/Low, "based on affected downstream tables and queries".
  - Root Cause = the upstream source contributing to the issue, e.g. a failing job or an upstream table issue, derived from Unity Catalog lineage.
  - Triage actions: **"Assign to me"** keeps the table Unhealthy for 7 days. **"Not an issue"** marks it Healthy for 7 days and sets Resolution = "Not an issue".
  - A **Recently Resolved Incidents** list has Resolution = "Auto-resolved" or "Not an issue".
  - Source: [Databricks docs](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/)
- **Metastore-level dashboard (screenshot).** [SEEN] Opened from Govern » Data Quality Monitoring.
  - Header card "Data Quality 99.21% healthy tables", with a bar for "8,791 healthy tables / 70 unhealthy tables".
  - "All monitored catalogs — The most unhealthy catalogs appear first".
  - Columns: Catalog | Tables monitored | # Unhealthy critical tables | Healthy % (inline green/red bar) | Owner.
  - A **Manage alerts (Beta)** button.
  - Source: [Databricks docs](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/)
- **Importable AI/BI dashboard template (screenshot, "Last updated: Jan 26, 2026, v1.0").** [SEEN] Built from `system.data_quality_monitoring.table_results`.
  - KPI tiles: "% Healthy tables 87.86%" (green), "# Healthy tables 16.34K" (green), "# Unhealthy tables 16" (red).
  - Line chart "Healthy tables over time" (% healthy by evaluation date, Oct 2025–Jan 2026).
  - Section "Quality incidents by impact — Impact is based on downstream lineage and query volume", with status and impact filters.
  - Table "Unhealthy tables" with Table | Status | Last Scanned | Impact | Scan Frequency | Reason | Root Cause. Root cause is a hyperlink, e.g. "Failing job: … (RUN_EXECUTION_ERROR)", or *null*.
  - Source: [Databricks docs](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/)
- **Governance Hub.** Shows the "percentage of healthy tables" for admins.
- **Data profiling (formerly Lakehouse Monitoring).** Covers summary stats, distribution and drift, and custom metrics, with alerts on metric changes. This overview page did not describe the auto-generated dashboard layout.
- Source for both: [Databricks DQ monitoring overview](https://docs.databricks.com/aws/en/data-quality-monitoring/)

### Inferences
- [INF] Patterns worth borrowing:
  - An explicit **Training / Error / Excluded / Not enabled** state separate from Healthy/Unhealthy, so "no verdict" never renders as green.
  - **Impact-ranked** incident lists, sorted "most unhealthy first".
  - A **coverage tile** next to the health tile.
  - **Predicted-band plus observed-line** charts with a "Last evaluated" marker.
  - Root cause shown as a link to the failing upstream job.
  - Lightweight triage states (Assign to me / Not an issue / Auto-resolved).
- [INF] Databricks rolls up as **% healthy tables** (a binary per table) rather than a weighted score. That avoids composite-score ambiguity but hides severity. "# Unhealthy critical tables" partially compensates.

### Gaps
- Alert or notification content for Databricks anomaly alerts is not on these pages. The docs say root cause also appears in alerts.
- The auto-generated data profiling dashboard layout was not verified in this pass.

## 3. Google Dataplex (Knowledge Catalog) data quality scans (brief)

### Takeaway
Dataplex shows a 0–100 **score per job, overall and per dimension** (% of rules passed), plus a boolean pass. Per-rule thresholds are expressed as "passing threshold" percentages. Results can be published to a "Data quality" tab on the BigQuery or catalog table page. Email alerts trigger on "Quality score (<=)", job failure, or job completion.

### Cited Findings
- **Score fields.** The overall result has `passed` (true only if all rules pass) and `score` 0–100. Dimension results have `passed` and `score`, and appear only if at least one rule is tagged with that dimension. Custom uppercase dimension names are allowed. — [Dataplex DataQualityResult API](https://docs.cloud.google.com/dataplex/docs/reference/rest/v1/DataQualityResult)
- **Console views.** The scan page has **Overview** (recent jobs: run time, records scanned, whether all checks passed, number of failed checks) and **Jobs history** (per job: DQ score, failed rules, logs). "Publish results to Knowledge Catalog" surfaces the latest results on the table's **Data quality** tab. Results can be exported to BigQuery with fields such as `job_dimension_result`, `rule_rows_passed` and `rule_rows_passed_percent`. — [Dataplex auto data quality docs](https://docs.cloud.google.com/dataplex/docs/use-auto-data-quality)
- **Email notifications.** Up to 5 addresses, with triggers "Quality score (<=)" (0–100), "Job failures" and "Job completion (success or failure)". — [same](https://docs.cloud.google.com/dataplex/docs/use-auto-data-quality)

### Inferences
- [INF] Dataplex's score is a % of rules passed, so it is unweighted by business criticality. Alerting on a score floor is a simple pattern to borrow.

### Gaps
- No console screenshot was viewed. Email layout is not documented.

## 4. Enterprise DQ suites: scorecards, dimension scores, roll-ups, thresholds

### Takeaway
The enterprise suites present a **numeric % score** with **three-band status**: Good/Acceptable/Not Acceptable (Informatica) or Passing/Warning/Failing (Collibra). They show **per-dimension scores** as donut or ring charts with trend over runs, and **roll up by averaging** (Informatica CDGC) or by **ratio of passing monitors** (Collibra DQ&O). Ataccama uses an "Overall quality" % = share of records passing all rules in dimensions that are configured to *contribute*.

### Cited Findings
**Informatica CDGC / IDMC**
- **Overview and Data Quality tabs.** The asset **Overview** tab shows a composite score as a **donut** in a Data Quality panel, with the last run time and whether the score rose or fell since the previous run. The **Data Quality** tab contains:
  - "Dimensions Latest Scores" (Validity, Completeness, Accuracy, Consistency, Timeliness).
  - A "Rules Against Target" bar chart with a band legend.
  - A "Total Rule Occurrence" table (rule name, dimension, score, rows passed, rows failed).
  - Source: [Trailhead: Review and interpret DQ results in CDGC](https://trailhead.salesforce.com/content/learn/modules/data-asset-quality-and-classification-in-informatica-cdgc/review-and-interpret-data-quality-results)
- **Bands.** Each rule template has a **target** and a **threshold**:
  - "Good" (green): score ≥ target.
  - "Acceptable" (amber): between threshold and target.
  - "Not Acceptable" (red): below threshold.
  - Clicking a rule opens its Score tab: donut, "Score Trend" across runs, acceptance status, and a failed-rows preview.
  - Source: [Trailhead](https://trailhead.salesforce.com/content/learn/modules/data-asset-quality-and-classification-in-informatica-cdgc/review-and-interpret-data-quality-results)
- **Roll-up.** The parent composite is the **average of child element scores**, e.g. columns 94/86/78/82 give a table score of 85%. Stakeholders are notified when a score drops below threshold. — [Trailhead](https://trailhead.salesforce.com/content/learn/modules/data-asset-quality-and-classification-in-informatica-cdgc/review-and-interpret-data-quality-results)
- **Overall score definition.** The CDGC overall score is the mean of the per-dimension scores. A parent asset's per-dimension score is an aggregated roll-up of its children (the method is unspecified). — [Informatica CDGC Data Quality Score MCP server doc](https://onlinehelp.informatica.com/IICS/prod/agentic-ui/en/cc-aiae-gs-mcp-servers/CDGC_Data_Quality_Score_server.html) (via search snippet)
- **Classic Informatica Analyst (10.5, older) scorecard dashboard.** Panes:
  - "Scorecards by Project" (bar).
  - "Scorecard Run Trend" (6-month line).
  - "Data Objects with Scorecards" (a coverage doughnut: with vs without scorecard).
  - "Cumulative Metrics Trend" (monthly stacked columns of good / acceptable / unacceptable metrics).
  - Example bands are unacceptable 0–40%, acceptable 41–89%, good 90–100%. Weighted metric groups existed.
  - Sources: [Informatica Analyst 10.5 scorecard dashboard](https://onlinehelp.informatica.com/analyst/10.5/en/data-discovery-guide/Scorecard_Dashboard_in_Informatica_Analyst.html); [IDQ 10.5 view reference (weights)](https://docs.informatica.com/data-quality-and-governance/informatica-data-quality/10-5/database-view-reference/profiling-warehouse-views/idpv_score_smry.html)

**Collibra Data Quality & Observability (current, docs "latest" / 2026.08)**
- **Score formula.** Score = (passing monitors / total active monitors) × 100.
  - "Passing" and "Learning" monitors count as passing.
  - Suppressed rules are excluded.
  - Exception status counts as failing.
- **Bands (adjustable).** Passing 90–100, Warning 76–89, Failing 0–75. The docs caution that a passing score does not guarantee there are no issues.
- **Where scores appear:**
  - The asset **Quality tab** shows an overall score, its history, and **ring charts per dimension**.
  - **Monitoring Overview** shows a roll-up score for a schema or table and a per-job Score column.
  - Job Details shows a run history chart.
  - The score can appear as an attribute in asset lists, diagrams and search.
- **Aggregation paths** walk the knowledge graph from an asset down to columns. External tool scores are averaged.
- Source: [Collibra: Data quality scores](https://productresources.collibra.com/docs/collibra/latest/Content/UnifiedDataQuality/co_about-data-quality-scores.htm)
- **Legacy Collibra DQ (2023.04 docs).** Score starts at 100, and each dimension or finding **deducts points**. Dimensions are custom-weighted and rules carry severity, e.g. 100 − 59 = 41. — [Collibra DQ scoring (2023.04)](https://productresources.collibra.com/docs/collibra/2023.04/Content/DataQuality/DQScorecards/Overview/Scoring.htm)

**Ataccama ONE**
- **Overall quality.** "Overall quality" % = share of records passing all applied DQ rules. It is shown for sources, terms, catalog items and attributes. "Not applicable results" can be excluded from the calculation.
- **Dimension tabs and settings.** DQ Settings controls which dimensions **contribute** to overall quality. A dimension gets its own results tab only if it has an active rule on the asset and contributes.
- **Monitoring project DQ reports.** Filterable, with per-dimension results, custom alerts and notifications, and **Invalid samples** of failing records that can be sent to remediation.
- **Anomalies and structure checks.** AI anomaly detection results can be accepted or rejected. Structure checks (missing columns, type changes) alert but **do not affect the DQ metric**.
- Source: [Ataccama ONE DQ overview](https://docs.ataccama.com/one/latest/data-quality/data-quality-overview.html)
- **Worked example (15.2 docs, search snippet).** Dimension A passes 25% and B passes 50%. With both contributing, Overall = 25%. This is record-level "passes all rules", not an average. — [Ataccama run DQ evaluation](https://docs.ataccama.com/one/latest/data-quality/run-dq-evaluation.html)

### Inferences
- [INF] **Roll-up logic varies widely across vendors, so whatever an in-house tool uses must be stated on screen.** The approaches found:
  - Arithmetic mean: Informatica CDGC, IBM IA.
  - Record-level all-rules-pass: Ataccama, which is stricter and approximates "weakest link".
  - Monitor pass ratio: Collibra DQ&O and Dataplex.
  - Deduction from 100: legacy Collibra.
  - Binary % healthy tables: Snowflake and Databricks.
- [INF] Common presentation primitives across these suites:
  - A donut or ring for the headline score.
  - Per-dimension small multiples.
  - A three-band traffic light defined by **target** plus **threshold** (Informatica's two-parameter model is clean).
  - A run-over-run trend line.
  - A rule table with rows passed/failed.
  - A failed-record sample.

### Gaps
- No current (2025–2026) screenshots of the Informatica CDGC, Collibra or Ataccama dashboards were viewed. Descriptions are from text docs and Trailhead.
- Ataccama's monitoring project report-tab layout and notification content were not found (that docs URL returned 404).
- CDGC dimension or child weighting is undocumented.

## 5. Executive scorecard design practice (cross-cutting synthesis)

### Takeaway
Practitioner and vendor sources broadly agree on five points:
- Give executives a **small number of consolidated, business-tied measures with trend and targets**.
- Keep **dimension, rule and record decompositions** one click away for operators.
- Avoid a single enterprise-wide average. Prefer **fitness-for-use scores per business use or critical data elements (CDEs)**.
- Treat critical failures as **non-compensating gates**.
- Always show **coverage and freshness** so that an unmeasured or stale estate does not look healthy.

Authoritative DAMA, DCAM and Gartner primary guidance on dashboard layout was not accessible: it is paywalled or was not found.

### Cited Findings
- **Dylan Jones, "23 Tips for Creating Effective Data Quality Scorecards" (Data Quality Pro; year not shown on page):**
  - Use four presentation levels: summary scores → decompositions → intermediate error reports → atomic record detail (Tip 5).
  - "A simple aggregate score for the entire database is usually rather meaningless" (Tip 11).
  - "The most valuable scores measure data fitness for various business uses" (Tip 13).
  - "scores not tied to a meaningful business objective are useless" (Tip 10).
  - Know whether errors are historic or recent (Tip 16).
  - A "dimensional time-dependent data quality scorecard must be one of the first priorities" (Tip 23).
  - Source: [Data Quality Pro](https://www.dataqualitypro.com/blog/how-to-create-data-quality-scorecards)
- **DataKitchen TestGen, a CDE score as a separate headline.**
  - Scorecards show a Total Score, a **CDE Score** (critical data elements only) and category breakdowns (by quality dimension, business domain or impact dimension) as dashboard tiles with history sparklines.
  - Color bands: green ≥96%, yellow 91–95%, orange 86–90%, red <86%.
  - Optional weighting: entity tables up to 10×, ID columns 3×, descriptions 0.5×, plus a PII multiplier.
  - Overall score = profiling score × testing score.
  - Source: [DataKitchen TestGen quality scores](https://docs.datakitchen.io/testgen/quality-scores/)
- **Acceldata, a disclosed methodology.**
  - The score shows a tooltip naming the aggregation methodology. Options are simple average by policy, simple average by rule, or row-weighted.
  - "The score is not scoped to a time period" (it is the latest execution).
  - "Changing the score aggregation methodology does not recalculate historical scores."
  - Source: [Acceldata score aggregation methodology](https://documentation.acceldata.io/docs/acceldata/adoc/documentation/score-aggregation-methodology)
- **Non-compensating gates.**
  - "Some failures should be non-compensating gates; strong completeness must not cancel prohibited data… or stale inputs."
  - Use the aggregate "as a navigation and release signal, not as a substitute for inspecting critical failures."
  - Source: [VarenyaZ glossary (low-authority source)](https://varenyaz.com/glossary/cognitive-science-ai-data-quality-score/)
- **Aggregation choices.** Weighted average, minimum ("weakest link") or grade buckets. Weights reflect business consequence, e.g. a missing currency field is minor for analytics but a hard stop for payments. — [Gravitee overview (low-authority source)](https://gravitee.io/corpus/gen-220/data-curation/data-quality-scoring.html)
- **Simple-average example.** IBM Information Analyzer averages column scores, (60+50+60)/3 = 57%, so every column counts equally. — [IBM docs](https://www.ibm.com/docs/en/SSZJPZ_11.5.0/com.ibm.swg.im.iis.ia.product.doc/topics/c_quality_score.html)
- **Tiered scorecard (DEV Community practitioner article).**
  - Tiers are enterprise, source and field.
  - At the enterprise tier, a single aggregated score answers how trustworthy the data is.
  - Field scores roll up, weighted by importance.
  - Calculate daily for operations and publish an executive roll-up monthly.
  - Source: [DEV Community: How to build an effective DQ scorecard](https://dev.to/kapusto/how-to-build-an-effective-data-quality-scorecard-66)
- **Executive view sizing and context.** Keep the executive view to a "max 6 visuals", with a data freshness indicator and context (targets, trends). — [Mastech Fabric quality dashboards guide (vendor)](https://digihub.mastechdigital.com/Fabric_Assets/Microsoft%20Fabric%20Asset%20Library/10_Data_Quality/MS_Fabric_Quality_Monitoring_Dashboards.docx)
- **DAMA DMBOK2 (secondary citation).** DQ levels should be "measured objectively and consistently". The methodology should be shared with stakeholders, and DQ reporting and issue management should be built into SLAs. — [NATO DCRA page citing DMBOK2](https://nhqc3s.hq.nato.int/apps/DCRA_Report/id-29d4122b072148f5aaf4882ecc5d963c/elements/id-12970cf328c94e18ad280baa959ead9b.html)
- **Gartner (abstracts only, paywalled):**
  - Jan 2026: quality leaders struggle to show business impact because "existing metrics fail to capture strategic value". The recommendation is to move from tactical reporting to strategically aligned, actionable metrics.
  - Jun 2025: universal quality levels are costly, so thresholds should be set relative to business benefit.
  - Jul 2024: use a "metrics cascade" to tie data and analytics governance to outcomes.
  - Sources: [Gartner 7337830](https://www.gartner.com/en/documents/7337830); [Gartner 6558302](https://www.gartner.com/en/documents/6558302); [Gartner 5567427](https://www.gartner.com/en/documents/5567427). Which abstract maps to which ID was reported by search summarization and was not individually verified.
- **Platform precedent for coverage beside health.**
  - Snowflake: "Healthy tables" next to "Monitoring coverage" ([docs](https://docs.snowflake.com/en/user-guide/data-quality-centralized-dashboard)).
  - Databricks: "Schemas Monitored 8 · 0.11% of all schemas" next to "99.95% healthy" ([docs](https://docs.databricks.com/aws/en/data-quality-monitoring/anomaly-detection/)).
  - Informatica Analyst: "Data Objects with/without Scorecard" doughnut ([docs](https://onlinehelp.informatica.com/analyst/10.5/en/data-discovery-guide/Scorecard_Dashboard_in_Informatica_Analyst.html)).

### Inferences
[INF] Synthesized patterns for an in-house Snowflake-based DQ dashboard and executive scorecard:
1. **Separate the views.**
   - **Executive scorecard**: about 6 tiles maximum, monthly cadence, with trend vs target.
   - **Operator console**: daily, incident- and check-level detail.
   - Each executive number should drill into dimensions → rules → failed records, the 4-level hierarchy in Jones Tip 5.
2. **Headline pair.** Show **% healthy (or score)** together with **% coverage**: in-scope CDEs or tables with at least one active check, and checks that ran on schedule. Never show health without coverage (Snowflake and Databricks precedent).
3. **Explicit non-verdict states.** Render "Training", "Not run / stale", "Error", "Excluded" and "Not monitored" as neutral grey or hatched, never green (Databricks status set).
4. **Scoring transparency.**
   - State the roll-up method in a tooltip (Acceldata precedent). Freeze historical scores when the method changes.
   - Prefer a separate **CDE score** (TestGen) and **non-compensating gates**: any critical-rule failure caps the tile at red, regardless of the average.
   - Avoid one enterprise-wide mean (Jones Tip 11). Roll up by business domain or data product.
5. **Banding.** Use a per-rule target and threshold with three bands: Good / Acceptable / Not acceptable (Informatica, Collibra).
6. **Value vs expectation display.**
   - Show the raw metric value, the expectation expression, and the pass/fail verdict as separate columns (Snowflake).
   - For anomaly checks, plot the observed line over a shaded predicted band, with a "last evaluated" marker (Databricks). This uses Snowflake's stored forecast and upper/lower bounds.
7. **Operator incident list.**
   - Columns: status, first detected, reason, root cause (linked to the upstream job or table), impact (from downstream lineage and query volume), owner, and actions (assign / not-an-issue / ticket link).
   - Sort by impact, most unhealthy first (Databricks).
   - Include "Impacted Assets" from lineage (Snowflake).
8. **Notifications.** Include object, check or DMF, observed value vs expectation or bounds, first-detected time, impact, owner, and links to the dashboard and failed records. Apply a cooldown or dedup window (Snowflake `cooldown_hours`). This content list is a synthesis: vendor payloads are largely undocumented.

### Gaps
- No primary DAMA-DMBOK or EDM Council DCAM text on scorecard *presentation* was obtained.
- Gartner research is paywalled, and only abstracts were seen.
- Several best-practice sources are vendor blogs or low-authority glossaries. Treat their thresholds (e.g. TestGen's 96/91/86 bands, Collibra's 90/76) as examples, not standards.
- No source quantified executive comprehension of composite vs dimension-level displays.
