# How DQ testing and ML quality tools present results: Anomalo, Soda, Elementary, Great Expectations, Metaplane (as of Oct 2026)

> **Raw research notes, not recommendations.** Evidence gathered on 2026-10-18 for the reports in `research-outputs/`. Vendor claims are marked as such; screenshots described here were not committed. Use these to check a claim's source, never as a decision or a plan.


Evidence labels used throughout:
- **[SEEN]** I looked at a downloaded public screenshot or GIF frame. Local copies were not committed to this repo (vendor-owned images; see each finding's source link).
- **[DOC]** The vendor docs or blog describe it in text, but I did not see it.
- **[INF]** My own inference.

Screenshot dates matter. Many public images are old, and each one is flagged with its date.

Research budget note: about 20 tool calls. Coverage is strongest for Elementary and Soda, partial for Anomalo and GX, and thin for Metaplane.

---

## 1. Overview and dashboard views (KPIs, tiles, health summary, time range, filters)

### Takeaway
The common pattern has four layers, from top to bottom:
1. A headline health number: Soda shows a "Data health score %"; Elementary shows donuts of test counts.
2. Pass/warn/fail counts colour-coded green, amber and red.
3. Per-category tiles, such as freshness, volume, schema, dbt tests and anomalies, each with a green|amber|red split.
4. Daily trend charts.

Every tool exposes a global "Last N days" selector and attribute filters (schema, dataset, tag, owner).

### Cited Findings
- **Elementary OSS report, Dashboard tab [SEEN]** (frames from the official `report_ui.gif`; report timestamp 02/20/2023, so the screenshot is **2023**):
  - **Top bar:** tabs for Dashboard | Test Results | Test Runs | Model Runs | Lineage, then "Report time: …", then help, GitHub and Slack icons.
  - **Header row:** the project name, a "Dev" environment badge, "Filter by" dropdowns, and a "Last 14 Days" range button.
  - **Row 1, three donuts:** "Test results breakdown" (177 tests split 163 / 3 / 11), "Tables health" (8 tables split passed / warning / failed / error), and "Monitored tables" (9 tables: monitored vs unmonitored). The total sits in the donut centre and segment counts are labelled outside. The legend reads Passed (green), Warning (orange), Failed (red), Error (grey).
  - **Row 2, five category tiles:** Freshness, Volume, Schema changes, dbt tests, Anomalies. Each has a large total and a small "green | orange | red" triple underneath (for example, dbt tests 21 → 16 | 3 | 2).
  - **Row 3, three small area charts over the date range:** "Unique tests executions" (green), "Test failures" (red), "Test warnings" (orange).
  - **Below the fold:** tables headed "Monitored tables" and "Tables with failed tests".
  - Sources: [report_ui.gif](https://raw.githubusercontent.com/elementary-data/elementary/master/static/report_ui.gif) and [Elementary OSS intro](https://docs.elementary-data.com/oss/oss-introduction).
- **Elementary Cloud Dashboard [SEEN]** (image filename says "dec_2024"):
  - It has the same three donuts, the same five category tiles, and the same three trend charts. The time-range button reads "Last 7 Days".
  - A persistent left nav replaces the top tabs. Each nav item has a subtitle verb: Dashboard ("Review data health"), Lineage ("Analyze root cause & impact"), Catalog ("Explore datasets"), DATA TESTS → Results ("Triage data issues"), Test Execution History ("Find frequently failing tests"), Test Configuration ("Add, delete & edit tests"), DATA PIPELINES → Model Duration ("Detect pipeline bottlenecks"), Environments.
  - A freshness stamp "Updated 21 min ago" sits top right.
  - Source: [image](https://res.cloudinary.com/diuctyblm/image/upload/v1701701243/elementary_cloud_data_observability_dashboard_dec_2024_oaos2y.png), [Elementary dashboard docs](https://docs.elementary-data.com/features/data-observability-dashboard).
- **Soda Cloud, classic Organization Dashboard [SEEN]** (screenshot undated, roughly 2025):
  - **Three KPI cards:** "32 CHECKS FAILED (LAST 90 DAYS)" with a red "Review" pill, "0 CHECKS WARNED (LAST 90 DAYS)" with a yellow "Review" pill, and "72% DATA HEALTH SCORE (LAST 90 DAYS)" with an info icon.
  - **"All Checks – Last 90 days":** a daily stacked bar chart with PASS (green), WARN (yellow) and FAIL (red). Days with no runs show grey placeholder bars, so gaps are visible.
  - **"Activity" strip:** active users 47, active datasets 379, active checks 1107, average daily alerts 6.
  - **Controls:** a top-right "All checks" saved-view dropdown, plus link and refresh icons. A "Try the new dashboards" button is also present.
  - Source: [Soda org dashboard docs](https://docs.soda.io/manage-issues/organization-dashboard).
- **Soda health score definition [DOC]:** "the number of failing checks out of the total number of checks". Filters are Schema, Dataset, dataset-level attributes and check-level attributes. Filter sets can be saved as named "collections" that can be shared or deleted. The page notes this classic dashboard is being replaced by "Soda Cloud dashboards". — [Soda org dashboard](https://docs.soda.io/manage-issues/organization-dashboard)
- **Soda customizable dashboards (public beta) [DOC, via search snippet]:** single-number health-score tiles and line charts with a 7/30/90-day range. Results can be grouped by user tags, such as data quality dimension. — [Soda blog: Customizable Dashboards](https://soda.io/blog/customizable-dashboards)
- **Anomalo [DOC]:** an April 2026 post introduces an "Agentic Platform" with a conversational interface (AIDA) that returns SQL and visualisations. "Continuously updated dashboards built from natural-language intent" are marked *coming soon*. "Insights reports" are *pushed* to stakeholders rather than pulled from a dashboard (preview). — [Anomalo blog, 2 Apr 2026](https://www.anomalo.com/blog/introducing-self-driving-data-and-the-new-anomalo-agentic-platform/)

### Inferences
- [INF] **Status palette.** Elementary and Soda both use the same palette: green for pass, amber/yellow for warn, red for fail, grey for error, skipped or no-run. Counts are shown as coloured inline triples ("16 | 3 | 2"). This is a cheap, borrowable convention.
- [INF] **Coverage beside health.** Putting "Monitored vs unmonitored tables" next to health (Elementary) means coverage gaps are not hidden behind a high pass rate.
- [INF] **Explicit time windows.** Soda labels every KPI with its window ("LAST 90 DAYS"). This avoids ambiguity about what a % refers to.

### Gaps
- I did not see a screenshot of Soda's new customizable dashboards.
- I found no public screenshot of Anomalo's current (2026) home or overview screen.

---

## 2. Test and check results views (status, measured value vs threshold, history, failing rows)

### Takeaway
Results are almost always shown as a filterable table or list with one row per check. Each row shows:
- a coloured status pill or icon,
- the measured value (Soda shows "value: 880"; Elementary shows the result message "Got 2 results, configured to fail if != 0"),
- a mini run-history strip.

Expanding a row reveals the query, sample failing rows and a history chart.

### Cited Findings
- **Elementary OSS "Test Results" tab [SEEN, 2023]:**
  - **Left pane:** a tree browser with toggles for dbt (project/folder/model), Tags and Owners, plus search and filter. Each model node carries small green/amber count badges, for example "customers 54 1".
  - **Right pane:** a summary header "All Tables" with Owners, Tags and Path, and coloured count chips top right (163 green, 3 amber, 11 red).
  - **Table columns:** Table name, Column name, Test name (with an icon distinguishing dbt-native tests from Elementary anomaly tests), Test type, Last test run, and Last status (a "Failure" pill in light red). Each column has sort and filter icons. A "+" on each row expands it.
  - The view also has a "Last 14 Days" range button.
  - Source: [report_ui.gif](https://raw.githubusercontent.com/elementary-data/elementary/master/static/report_ui.gif).
- **Elementary report filtering by run [DOC]:** `edr report --select last_invocation`, `invocation_time:…` or `invocation_id:…` filter the Test Results and Lineage screens to a specific dbt run. — [Elementary generate report](https://docs.elementary-data.com/oss/guides/generate-report-ui)
- **Soda Cloud dataset page, "Checks" tab [SEEN]** (Soda v4 UI, roughly 2025):
  - **Header:** the dataset name with a "v4" contract-version badge, and breadcrumb chips (datasource / database / schema).
  - **Tabs:** Checks | Metric Monitors | Columns | Scans | Incidents | Requests.
  - **Check rows:** a green check-circle icon, the check name, attribute chips ("Department: engineering", "Layer: Gold"), and the measured value in small grey text below ("value: 0", "value: 880").
  - **History strip on the right of each row:** 7 small square pass/fail boxes. A chevron opens the detail view.
  - **Right side panel:** a dark card "70% passed, Last verified – Yesterday, 08:44". Below it, "Column check coverage 76%" and a "Checks" chip pair (33 green / 14 red).
  - **Controls:** Filter, Save, an "Unsaved collection" dropdown, search, and a "Verify" run button.
  - Source: [Soda dataset dashboard](https://docs.soda.io/manage-issues/dataset-dashboard).
- **Soda Checks tab behaviour [DOC]:**
  - Checks are grouped by column, and columns with failed checks auto-expand.
  - There is a "show only failed" toggle.
  - The health score panel recomputes for the active filter and says "Results for this filter only". Never-evaluated checks are excluded from it.
  - Scope pills switch between All checks / Base contract / scopes / Data Standards.
  - The Scans tab lists every scan, with results and execution logs.
  - Source: [Soda dataset dashboard](https://docs.soda.io/manage-issues/dataset-dashboard).
- **GX Data Docs (GX Core, static HTML) [DOC, mostly 0.14–0.17-era sources]:**
  - Each Validation Results page shows a table per section with the columns "Status", "Expectation" and "Observed Value".
  - Clicking a failed row shows a "Sampled Unexpected Values" table.
  - Sources: [GX legacy docs: check out Data Docs](https://legacy.017.docs.greatexpectations.io/docs/0.14.13/tutorials/getting_started/check_out_data_docs), [DEV tutorial](https://dev.to/samuelearl/data-pipelines-with-great-expectations-step-4-validate-data-582m).
- **GX Cloud [DOC]:**
  - Results open from the Validations tab via a "Batches & run history" pane.
  - Older docs (0.17) describe a timeline chart of validation runs where you hover a circle to see observed values, plus a "Failures Only" toggle.
  - Sources: [GX Cloud manage validations](https://docs.greatexpectations.io/docs/cloud/validations/manage_validations) (now 404s, so the content comes from the search snippet), [legacy 0.17](https://legacy.017.docs.greatexpectations.io/docs/cloud/validations/manage_validations).
- **GX Cloud result-detail levels [DOC, Feb 2026]:** users pick one of three result formats per suite, checkpoint or asset:
  1. "Status only": status, % failed rows for `mostly` expectations, and investigation queries, with no data shown.
  2. "Observed values" (default): min/max for aggregates, plus samples of missing or unexpected values.
  3. "Sample unexpected rows".

  Rendering differs by expectation type (aggregate, row-level, custom SQL, multi-source). — [GX blog, 12 Feb 2026](https://greatexpectations.io/blog/From-Detection-to-Diagnosis-Root-cause-analysis-for-Expectation-failures-in-GX-Cloud/)
- **Anomalo check status page [SEEN, blog dated Nov 2021, images re-uploaded 2024]:** a "Sample of Bad Rows" panel ("A random sample of 100 rows that caused the check to fail"). It is laid out as follows:
  - Buttons "Download Sample CSV" and "Show SQL".
  - A **transposed** single-record view with Column | Row Value. The offending cell (`priceperticket = None`) is highlighted with a red row band.
  - A "sample row number" slider to scrub through the 100 samples, and a "column sort order" dropdown ("database order").
  - The page also shows a sample of *good* rows for contrast. The rule result in the example states 793 failing rows (about 0.5% of the table).
  - Source: [Anomalo: Root Causing Data Failures](https://www.anomalo.com/blog/root-causing-data-failures/).

### Inferences
- [INF] **Measured value next to the check name.** Soda's "value: N", or Elementary's plain-English "Got N results, configured to fail if != 0", is the minimum needed to show measured vs threshold. None of the screenshots seen show a dedicated "expected vs actual" column.
- [INF] **Fixed-width history strips.** A per-row run-history strip of N coloured squares (Soda) is a compact way to show flakiness without opening a chart. Elementary Cloud puts this in a separate "Test Execution History" page aimed at "frequently failing tests".
- [INF] **Data exposure levels.** GX's three-level result format is a governance-friendly pattern for PII-sensitive teams: let the owner choose whether failing rows are shown at all.
- [INF] **Transposed bad-row view.** Anomalo's one-record-at-a-time view with the bad cell highlighted works well for wide tables.

### Gaps
- I did not obtain a current (2025–26) GX Data Docs or GX Cloud screenshot.
- I could not confirm the Elementary test-detail expanded panel (history chart, sample rows) from a screenshot. It is visible only as "+" expanders in the GIF frames I extracted.

---

## 3. Anomaly displays (expected band, markers, "why" explanation)

### Takeaway
Two complementary patterns:
1. **Tile with expected range in text.** Soda's metric monitors show the current value large, then "Expected: 7.95k – 8.15k rows" with a sparkline, and a red marker on the sparkline for the anomalous point.
2. **Explanatory segment chart.** Anomalo's Root Cause Analysis ranks `column = value` segments by the % of bad rows they capture against their prevalence in good rows.

### Cited Findings
- **Soda "Metric Monitors" tab [SEEN]** (screenshot filename dated 2025-05-29):
  - **Header:** "Dataset Monitors – Last 7 days", plus "Last scan: about 15 hours ago" and a "Configure Dataset Monitors" button.
  - **Card groups:** "BASED ON METADATA" (Total row count, Total row count change, Last insertion time, Schema changes) and "BASED ON DATA – TIME PARTITION COLUMN (CREATED_DATE) – DAILY PARTITION INTERVAL" (Partition row count, Most recent timestamp).
  - **Each card contains:**
    - an icon with a green check badge,
    - a title and a one-line description ("The total number of rows in the dataset at scan time"),
    - a big value ("8,02k", "21h 2s ago AT SCAN TIME (UTC)", "0 changes"),
    - an "Expected: <low> – <high>" line,
    - a dotted sparkline of recent scans (red triangle marker on the anomalous point in "Total row count change"; schema changes shown as a row of small check boxes),
    - a bell icon to toggle alerting.
  - Source: [Soda dataset dashboard](https://docs.soda.io/manage-issues/dataset-dashboard).
- **Anomalo Root Cause Analysis chart [SEEN, 2021 blog / 2024 images]:**
  - Title "Root Cause Analysis – The top data segments that contain a disproportionate amount of bad records".
  - A horizontal chart with one row per segment ("venuename = Metropolitan Opera", "catname = Opera", "venuecity = New York City", "catgroup = Shows", "eventname = Il Trovatore"). On the x-axis, "% of bad rows in segment" runs 0–100%.
  - Each row has a grey bar (share of good rows in the segment) and a coloured line/tick (share of bad rows), coloured from red (strong) to yellow (weak).
  - A right column, "similar data segments", lists correlated segments (e.g., "venuestate = NY").
  - **[DOC]** The article labels segments as "probable" (captures all bad rows, rare in good rows), "directional" (captures all bad rows but common in good data) or "insufficient" (captures only part of the bad rows).
  - Source: [Anomalo RCA blog](https://www.anomalo.com/blog/root-causing-data-failures/).
- **Anomalo 2026 [DOC]:** the Data Insights Agent produces analyst-style reports covering "what changed, the likely cause, affected data, and suggested actions". No visual format is specified. — [Anomalo Apr 2026](https://www.anomalo.com/blog/introducing-self-driving-data-and-the-new-anomalo-agentic-platform/)

### Inferences
- [INF] **Text before charts.** Putting the expected range as text under the KPI ("Expected: X – Y") is easier to build in a static report than a shaded-band chart, and conveys the same information.
- [INF] **Simple root-cause ranking.** The Anomalo segment chart can be approximated in-house: for each candidate `column=value`, compute % of failing rows covered and % of passing rows covered, then rank by the difference.

### Gaps
- No full-size time-series anomaly chart with a shaded expected band was seen in this pass, for Elementary, Soda or Anomalo. I believe Elementary's anomaly test detail has one, but I did not confirm it.
- I found no Anomalo SHAP-style "what changed" panel screenshot.
- I found no training-window visual convention.

---

## 4. Coverage and test-inventory views

### Takeaway
Coverage is shown as a ratio next to health, not on a separate screen:
- Elementary: a "Monitored tables" donut (monitored vs unmonitored).
- Soda: a "Column check coverage 76%" chip on the dataset page.

### Cited Findings
- **Elementary donuts [SEEN]:** the OSS report shows 8 monitored / 1 unmonitored. The Cloud dashboard shows 59 monitored / 181 unmonitored of 240 tables, where grey marks the unmonitored share. — [OSS GIF](https://raw.githubusercontent.com/elementary-data/elementary/master/static/report_ui.gif), [Cloud image](https://res.cloudinary.com/diuctyblm/image/upload/v1701701243/elementary_cloud_data_observability_dashboard_dec_2024_oaos2y.png)
- **Elementary Cloud nav [SEEN]:** includes "Catalog – Explore datasets" and "Test Configuration – Add, delete & edit tests" (inventory management). The Elementary OSS intro also says the report helps track "test coverage". — [Elementary OSS intro](https://docs.elementary-data.com/oss/oss-introduction)
- **Soda dataset page [SEEN]:** shows "Column check coverage 76%". The Columns tab [DOC] lists columns with data types, distinct counts and profiling stats. — [Soda dataset dashboard](https://docs.soda.io/manage-issues/dataset-dashboard)

### Inferences
- [INF] **A coverage heatmap.** A tables × check-type matrix is a natural in-house extension. No vendor screenshot of such a matrix was found.

### Gaps
- I found no dedicated coverage-gap screen for GX, Anomalo or Metaplane.

---

## 5. Incident and issue views

### Takeaway
Incidents appear as a simple table with these columns:
- Severity
- Incident ID and title
- Status (Reported/Resolved)
- Lead (avatar)
- Reported (relative time)

The table is filterable by lead, status and severity. Anomalo is moving triage to an agent that opens Jira or ServiceNow tickets.

### Cited Findings
- **Soda dataset "Incidents" tab [SEEN]:** a search box and a "RESULTS: 2" counter. The columns are SEVERITY (Major), INCIDENT ("INC-262: V4 GA Incident – Jarek"), STATUS (Resolved / Reported), LEAD (initials avatar plus name), and REPORTED ("about 1 month ago (Jan 19, 6:26 AM)"), with a kebab menu per row. [DOC] It can be filtered by lead, status and severity, and an incident can be created for any check. It also lists incidents that link checks from other datasets. — [Soda dataset dashboard](https://docs.soda.io/manage-issues/dataset-dashboard)
- **Soda Requests tab [DOC]:** contract change requests with statuses Open / Done / Won't Do, comments, and proposed changes for owners to approve. — same source
- **Anomalo [DOC, 2026]:** the "Data Issue First Responder" (coming soon) assesses impact and criticality, follows runbooks, opens tickets in ServiceNow or JIRA, escalates, and logs every action for audit. — [Anomalo Apr 2026](https://www.anomalo.com/blog/introducing-self-driving-data-and-the-new-anomalo-agentic-platform/)
- **Elementary Cloud nav [SEEN]:** labels Results as "Triage data issues". I did not see the incident screen itself.

### Inferences
- [INF] **Minimum incident record.** The smallest useful record is severity + ID + title + status + owner + age. Soda's display of both relative and absolute time ("about 1 month ago (Jan 19, 6:26 AM)") is a good convention.

### Gaps
- I did not see Elementary Cloud incident grouping, comments or Jira links, or the GX Cloud incident UI.

---

## 6. Lineage with test results overlaid

### Takeaway
Elementary's lineage view is the clearest example:
- each node shows inline green/amber/red test-count badges,
- a filter bar toggles Show All / Failures / Warnings and Upstream & Downstream,
- clicking a node opens a bottom drawer with that table's test-results table.

### Cited Findings
- **Elementary OSS "Lineage" tab [SEEN, 2023]:**
  - **Toolbar:** "All Nodes" dropdown, a segmented control [Show All | Failures | Warnings], an "All Tests" dropdown, and an "Upstream & Downstream" direction dropdown, each with an (i) tooltip.
  - **Canvas:** a left-to-right DAG with zoom +/−/reset controls. Model nodes are grey rounded rectangles with count badges (e.g., "stg_signups 28 [9 red]", "orders 6 [1 amber]"). Exposure/dashboard nodes such as "customers_dashboard" are tinted purple. The selected node is outlined blue.
  - **Bottom drawer for the selected node:** a header "orders – 6 Passed • 1 Warning • 0 Failures", then the same results table as the Test Results tab, with status pills: amber "Warning" for accepted_values, green "Passed" for column_anomalies zero_count and schema_changes.
  - Source: [report_ui.gif](https://raw.githubusercontent.com/elementary-data/elementary/master/static/report_ui.gif).
- **Metaplane alerts [DOC]:** reported to include column-level lineage / downstream impact. — [Metaplane blog: Slack alerts](https://metaplane.dev/blog/how-to-manage-metaplane-monitor-alerts-in-slack)

### Inferences
- [INF] **Inline count badges.** Badges on each node, plus a "failures only" filter, let users trace blast radius quickly.

### Gaps
- The current Elementary Cloud lineage (column-level, BI nodes) was not seen.

---

## 7. Data quality scores and dimensions

### Takeaway
Soda's built-in score is a simple pass ratio ("72% data health score", "70% passed"). Dimensions are user-defined check attributes, used for grouping and filtering rather than as a fixed scorecard. Elementary's dimension and health-score feature could not be verified in this pass.

### Cited Findings
- **Soda score definition [DOC]:** "failing checks out of total checks" on the org dashboard. On the dataset page, a filter-aware pass % excludes never-run checks. — [Soda org dashboard](https://docs.soda.io/manage-issues/organization-dashboard), [Soda dataset dashboard](https://docs.soda.io/manage-issues/dataset-dashboard)
- **Soda dimensions [DOC]:** the docs home names completeness, accuracy, timeliness, uniqueness and consistency, but does not show them in the UI. The customizable-dashboards blog lists grouping by tag, e.g., dimension. A Soda guide for building a Sigma dashboard off the Reporting API shows a *weighted* score by dimension using a per-check "Weight" attribute; this is a DIY integration, not a native view. — [docs.soda.io](https://docs.soda.io/), [Soda blog](https://soda.io/blog/customizable-dashboards), [Soda reporting API guide](https://docs.soda.io/soda-documentation/soda-v3/use-case-guides/reporting-api-to-overview-dashboards)
- **Anomalo [DOC]:** has a blog post on "six (or more) dimensions of data quality". It is conceptual, and no UI was found. — [Anomalo blog](https://www.anomalo.com/blog/the-six-or-more-dimensions-of-data-quality/)

### Inferences
- [INF] **Score recipe.** A defensible in-house score is: tag each check with a dimension and an optional weight, compute a pass % per dimension, then roll up a weighted average. This mirrors Soda's suggested API approach.

### Gaps
- **Elementary data-health-score and dimension feature: unverified.** The `llms.txt` grep returned no matching doc pages and I ran out of budget to search further.
- No screenshot of a dimension scorecard was found for any tool.

---

## 8. Notification design (Slack, Teams, email)

### Takeaway
Elementary's Slack alert is the most concrete reference design:
- a red left border,
- the headline "Failure: "<test>" test failed on <db.schema.table.column>",
- a meta line,
- a two-column field grid,
- "Result" and "Configuration" sections with code blocks for the result message, sample rows and SQL.

Metaplane alerts embed a small trend chart and feedback buttons.

### Cited Findings
- **Elementary Slack alert [SEEN]** (example timestamp 2024-02-12):
  - **Title:** `Failure: "not_null" test failed on ella_raw.elementary.customers.first_order`
  - **Meta line:** `Test: not_null - Generic | Status: fail | 2024-02-12 12:21:15`
  - **Red vertical attachment bar on the left.**
  - **Field grid:** Table; Column / Tags (PII); Owners / Subscribers (Slack @-mentions highlighted); Description (the test's doc string).
  - **"🔍 Result" section:**
    - "Result message" code block: "Got 2 results, configured to fail if != 0"
    - "Test results sample" code block: `[{'first_order': None}, …]`
    - "Test query" SQL block
  - **"🛠 Configuration" section:** "Test parameters" code block.
  - Source: [image](https://res.cloudinary.com/dvmtceykh/image/upload/v1707742279/od5lminwcmktkjfh0lao.png), [Elementary alerts docs](https://docs.elementary-data.com/oss/guides/alerts/elementary-alerts).
- **Elementary alert scope [DOC]:**
  - Alert types: dbt test failures and warnings, Elementary test failures and warnings, model run failures, and source freshness failures.
  - Enrichment via YAML `owner`, `subscribers`, `description` and `tags`.
  - Configurable channel, suppression interval and filters.
  - Slack and MS Teams are supported.
  - Source: [Elementary alerts](https://docs.elementary-data.com/oss/guides/alerts/elementary-alerts).
- **Metaplane [DOC]:**
  - Slack alerts show the monitor name, a short issue summary, a small trend chart of recent values, and action buttons for feedback or more detail.
  - Some alerts include column-level lineage and downstream impact.
  - Destinations: Slack, MS Teams, email and PagerDuty, with routing per monitor. The UI shows each monitor's alert destinations.
  - Sources: [Metaplane: manage alerts in Slack](https://metaplane.dev/blog/how-to-manage-metaplane-monitor-alerts-in-slack), [Metaplane: alert destinations](https://www.metaplane.dev/blog/more-visibility-into-alert-destinations).
- **Anomalo [DOC]:** an Anomalo blog recommends notifications include "samples of raw data that highlight good and bad values". This is guidance, not a confirmed alert layout. — [Anomalo: effective data monitoring](https://www.anomalo.com/blog/effective-data-monitoring/)

### Inferences
- [INF] **Alert template.** For a Teams or Slack webhook, mirror the Elementary structure:
  - headline with status and fully-qualified object,
  - owners as @-mentions,
  - plain-English result,
  - up to 5 sample rows,
  - SQL to reproduce,
  - a link to the report page.

### Gaps
- I did not see Soda, GX or Anomalo Slack/email alert screenshots.
- I did not see the Metaplane alert image itself, so its description comes from blog text only.

---

## 9. Static and self-hosted report options (zero-cost)

### Takeaway
Two free self-hosted options exist:
- **Elementary OSS:** `edr report` produces a single self-contained HTML file. It can be sent to Slack or hosted on S3 or GCS (Azure blob is also supported per my prior knowledge, but this pass did not confirm it).
- **GX Core Data Docs:** a static HTML site written to `uncommitted/data_docs/local_site/` (or any filesystem path). It is rebuilt by a checkpoint action and has pages for Expectation Suites and Validation Results.

### Cited Findings
- **Elementary generation and sharing [DOC]:** install the dbt package, create an `elementary` profile, `pip install elementary-data`, then `edr report`, which writes the HTML file. The sharing guide lists "Send via Slack", "Host on Amazon S3" and "Host on Google Cloud Storage". — [generate report](https://docs.elementary-data.com/oss/guides/generate-report-ui), [share report](https://docs.elementary-data.com/oss/guides/share-report-ui)
- **Elementary report tabs [SEEN, 2023]:** Dashboard, Test Results, Test Runs, Model Runs, Lineage. The page header shows a "Report time" stamp. — [GIF](https://raw.githubusercontent.com/elementary-data/elementary/master/static/report_ui.gif)
- **GX Data Docs [DOC]:**
  - Data Docs are "human-readable documentation" saved as static web pages.
  - `base_directory` can be absolute or relative; the default is `uncommitted/data_docs/local_site/`.
  - `context.build_data_docs()` or an `UpdateDataDocsAction` in a Checkpoint rebuilds them, and `context.open_data_docs()` opens them.
  - The current page only documents filesystem sites; S3/GCS/Azure hosting is not on this page.
  - Source: [GX configure Data Docs](https://docs.greatexpectations.io/docs/core/configure_project_settings/configure_data_docs/).
- **GX Data Docs page anatomy [DOC, legacy 0.14]:** contains pages for Expectation Suites and Validation Results. Validation tables show Status / Expectation / Observed Value, and failed rows reveal "Sampled Unexpected Values". By default only results in the local directory are compiled. — [GX legacy Data Docs](https://legacy.016.docs.greatexpectations.io/docs/0.14.13/terms/data_docs)

### Inferences
- [INF] **Static stamp.** A static report should carry a prominent "Report time" stamp, as Elementary does, because readers cannot tell staleness otherwise.
- [INF] **Hosting.** Either the Elementary HTML or GX Data Docs can be hosted for free on an internal file share, SharePoint, or a static bucket.

### Gaps
- I found no current public live demo URL of an Elementary OSS report.
- I did not view a GX Data Docs screenshot from 2025–26.

---

## 10. User UX complaints (reviews and GitHub issues)

### Takeaway
I found very little first-hand UX complaint evidence in this pass. The only signals are third-party remarks about Metaplane's roadmap uncertainty after the Datadog acquisition.

### Cited Findings
- **Metaplane:** review sites advise buyers to "separately validate roadmap continuity now that Metaplane is Metaplane by Datadog". A Sifflet (competitor) review calls Metaplane a "lightweight option", which signals competitive framing, so treat it with caution. — [rfp.wiki](https://www.rfp.wiki/artificial-intelligence/augmented-data-quality-solutions/metaplane), [Sifflet review](https://www.siffletdata.com/blog/metaplane-review)
- **Metaplane UI history:** a dashboard and menu redesign shipped in September 2023. — [Metaplane blog](https://www.metaplane.dev/blog/metaplane-data-quality-dashboard)

### Inferences
- None drawn from this evidence.

### Gaps
- I did not reach G2 or Gartner Peer Insights reviews for Soda, Anomalo, Elementary or GX, nor the Elementary or GX GitHub issues on report UX (for example, HTML report size or performance with large projects). This needs a follow-up pass.
- Metaplane's post-acquisition (Datadog-native) UI was not found.
