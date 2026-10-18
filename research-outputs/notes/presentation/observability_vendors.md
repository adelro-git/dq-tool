# Presentation layer of data observability vendors (Monte Carlo, Bigeye, Sifflet), 2025–2026

> **Raw research notes, not recommendations.** Evidence gathered on 2026-10-18 for the reports in `research-outputs/`. Vendor claims are marked as such; screenshots described here were not committed. Use these to check a claim's source, never as a decision or a plan.


Evidence labels used throughout:
- **[SCREENSHOT]**: I looked at a public docs screenshot myself. Local copies were not committed to this repo (vendor-owned images; see each finding's source link).
- **[DOCS-TEXT]**: the vendor's docs or blog say this in text.
- **[THIRD-PARTY]**: a review site, integration partner, or customer handbook.
- **Inference**: my own reasoning. These appear only under "Inferences".

Scope note: Bigeye and Sifflet docs have usable, recent screenshots. For Monte Carlo, the docs URLs I tried returned 404 (`/docs/alerts`), and one page had no screenshots (`/docs/data-reliability-dashboard`). Monte Carlo evidence is therefore mostly docs text and third-party material. Acceldata was not covered because I ran out of time.

## Navigation and information architecture

### Takeaway
All three vendors use a left icon rail with top-level sections (search/catalog, issues or incidents, monitors, lineage, dashboard, settings). Two scoping devices recur: domain/tag filters (Monte Carlo, Sifflet) and "Collections" (Bigeye). These narrow the views and also route notifications.

### Cited Findings
- [SCREENSHOT] Bigeye: a narrow left icon rail with roughly 9 icons (search, catalog/sources, warning-triangle "Issues" highlighted, documents, A/B, lineage graph, apps grid, tools). Settings, a terminal, help and collapse icons sit at the bottom. The page header shows the section name ("Issues") above a large count ("18 issues"). — [Bigeye View Issues](https://docs.bigeye.com/docs/view-issues) (image `issue_inbox.png`, undated)
- [DOCS-TEXT] Bigeye: you can open issues from anywhere in the platform, including the catalog and collection views. — [Bigeye Issues](https://docs.bigeye.com/docs/issues)
- [DOCS-TEXT] Bigeye: Slack channels and @users are configured on a Collection's notification tab, so Collections are the unit for ownership and routing. — [Bigeye Slack docs](https://docs.bigeye.com/docs/slack)
- [DOCS-TEXT] Bigeye: you can create a Collection that groups the metrics behind one BI dashboard, to monitor "dashboard health" with email, Slack or webhook notifications. — [Bigeye: Monitor Dashboard Health](https://docs.bigeye.com/docs/monitoring-dashboard-health-with-an-sla)
- [DOCS-TEXT] Sifflet: the dashboard is the landing page after login. Its top filters for domain, tags and date range apply to the whole view. — [Sifflet changelog: new dashboard (app v508)](https://docs.siffletdata.com/changelog/new-sifflet-dashboard-a-central-hub-for-your-datas-health)
- [DOCS-TEXT] Monte Carlo: the Data Reliability Dashboard breaks many metrics out by **Domain**, and filters by Tags (OR-combined) and "Key Assets Only" (Importance Score ≥ 0.75). — [MC Data Reliability Dashboard](https://docs.getmontecarlo.com/docs/data-reliability-dashboard)

### Inferences
- A practical scoping pattern for an in-house tool: a global domain/tag filter bar, plus a saved "collection" of checks per owner team that drives both the view and alert routing.

### Gaps
- I could not get Monte Carlo's full top-level nav list from public docs, and I found no MC screenshots.

## Overview / home / executive views

### Takeaway
Sifflet's home is a card grid with these elements, top to bottom: pinned assets, catalog counts, an incidents stacked bar by status, table-uptime % with sparklines per monitor type, a coverage gauge, and a monitor-type pie. Monte Carlo's dashboard is a manager view built around incident counts, response metrics (median time to response and to "Fixed"), table uptime as coverage, and adoption.

### Cited Findings
- [SCREENSHOT] Sifflet dashboard, taken around June–July 2025 (chart axis runs 2025-06-16 to 2025-06-30). Layout from top to bottom:
  1. **Filter bar**: a time-range dropdown ("14 days") and a "Select tags" multi-select.
  2. **Pinned** carousel with Recent/Data tabs. Cards show asset type (Dashboard/Connector/Table), name, and source/platform icon.
  3. **Monitoring overview → Data Catalog Summary**: four big-number tiles with circular icons ("85 Sources", "908 Tables & Views", "18 Pipelines", "216 Dashboards").
  4. **Incidents** card: "Latest incidents by detection date, grouped by current status", a Daily/(weekly) granularity dropdown, and a stacked bar chart colored red=Open, yellow=In progress, green=Closed.
  5. **Table uptime** card: "Average of tables without failures from active daily monitors". Three rows, each a big % plus an area sparkline: 97% Volume monitors, 97% Freshness monitors, 90% All monitors.
  6. **Monitor coverage** card: a half-donut gauge reading "65% Coverage", segmented Unmonitored (grey) / Sifflet monitors (blue) / dbt tests (light blue).
  7. **Monitor types** pie: 87% table-level health, 3% Metrics, 2% Format Validation, 4% Field Profiling, 3% dbt tests, 1% Custom.
  8. **"Sifflet AI Assistant – Monitors (Beta)"** carousel: each card shows a monitor, its asset, and a suggestion such as "Lower model sensitivity from 50 → 25", with Dismiss/Accept buttons.
  — [Sifflet changelog](https://docs.siffletdata.com/changelog/new-sifflet-dashboard-a-central-hub-for-your-datas-health) (image `new_dashboard.jpg`)
- [DOCS-TEXT] Sifflet: the date range runs 7–90 days. Customizable dashboards are described as the next planned step. — [Sifflet changelog](https://docs.siffletdata.com/changelog/new-sifflet-dashboard-a-central-hub-for-your-datas-health)
- [DOCS-TEXT] Monte Carlo Data Reliability Dashboard, described as a "manager overview":
  - **Incidents**: All Incidents (table monitors, custom monitors, dbt), and Incidents from Custom Monitors.
  - **Incident Response**: Status Update Rate (incidents with a status ÷ total), Time to Response (median from creation to first status update), Time to 'Fixed' (median).
  - **Table Uptime**: cards per monitor type showing tables with no incident out of total tables.
  - **Domains**: breakdown by domain.
  - **Users/Monitors**: activity and adoption, now moved to a separate "Activity" dashboard.
  - **Controls**: a Lookback Range filter, an "Include current month/week" toggle (off by default because partial periods skew trendlines), and an Incident Types exclusion (usually Schema Changes). Data refreshes every 12 hours.
  — [MC Data Reliability Dashboard](https://docs.getmontecarlo.com/docs/data-reliability-dashboard)
- [DOCS-TEXT] Monte Carlo: the big uptime numbers cover the whole lookback range, so they run lower than the adjacent period-by-period charts. The docs explicitly warn about this. — [MC Data Reliability Dashboard](https://docs.getmontecarlo.com/docs/data-reliability-dashboard)

### Inferences
- Both vendors show "uptime" (the share of tables without failures) rather than a composite score as the headline health KPI. Both split it by monitor type (freshness/volume/all).
- MC's choice to exclude the current partial period by default is a cheap pattern worth copying to avoid misleading trend dips.

### Gaps
- No Bigeye home/overview screenshot found. I also did not verify whether a Bigeye "dashboard" page exists in 2026.
- No Monte Carlo dashboard screenshot. Chart types are not confirmed.

## Alert / incident feed and detail pages

### Takeaway
The feeds share a common design: a left filter panel with counts per facet, a search bar, a sort dropdown, and rows that pair a title and an asset breadcrumb with status/severity chips, an assignee avatar and relative time. Bigeye also adds an inline sparkline per row, and Sifflet a "compromised dashboards" count. Sifflet's detail page has a sticky header (title, status chip, severity, Status/Create Jira/Edit buttons) and tabs for Overview, Monitors, Downstream Impacts, Lineage and Notifications, with a right sidebar for details and activity.

### Cited Findings
- [SCREENSHOT] **Bigeye Issues inbox** (undated docs screenshot):
  - **Left panel, "Group by"**: radio buttons for Issue status / Assignee / Metric category / Table.
  - **Left panel, "Filter issues"**: collapsible facets with counts. Issue status: Triage 18, Acknowledged 0, Monitoring 0, Closed, Merged. Type: Issues 18 / Incidents 0. Root cause: Is root cause / Not root cause. Assignee: with initials avatars. Metric status: Alerting 16, Failed 0. There is a "Reset all filters" link.
  - **Main area**: a search bar, "Showing 1 of 1 Issue status groups", and "Sort by: Recent issues". A group header ("Triage") has a red pie-clock status icon.
  - **Each row**, left to right: a checkbox, a signal-bar priority icon (1–3 bars, red/orange when high, grey when low), a status icon, a two-line title, an inline sparkline, the issue ID, relative time, an assignee avatar and a "…" menu.
  - **Title format**: "{Metric type} - {plain-English problem} on {COLUMN}", e.g. "Null (#) - Too many null values on ORDER_ID". Underneath is the breadcrumb source / schema / table / column.
  - **Sparkline**: shows the metric line, a grey expected band, and a red vertical drop marking the anomalous point.
  - **Issue ID and time**: e.g. "TU-10996", "Created 3d ago".
  — [Bigeye View Issues](https://docs.bigeye.com/docs/view-issues)
- [DOCS-TEXT] Bigeye issue statuses are Triage, Acknowledged, Monitoring and Closed. The default sort is highest priority first. Hovering the priority icon shows a priority score based on table popularity, the number of existing issues on the table, and how far the value is from the threshold. — [Bigeye View Issues](https://docs.bigeye.com/docs/view-issues)
- [DOCS-TEXT] Bigeye grouping logic: an issue opens on the first anomalous run, and later out-of-bounds runs become alerts threaded on the same issue timeline. Users can:
  - Acknowledge, resolve or close with one click.
  - Mark alerts good or bad as feedback.
  - Add notes on closed issues.
  - See previous issues for the same metric.
  - View lineage.
  - Use the "metric query" and "debug query" to troubleshoot in the source.
  — [Bigeye Issues](https://docs.bigeye.com/docs/issues)
- [SCREENSHOT] **Sifflet Incidents list** (screenshot dated 2025-08-08 in the filename):
  - **Left Filters panel**: Save/Clear links. Status chips (Open = orange up-arrow, In progress = blue right-arrow, Closed = grey check). Severity uses chevron icons: Critical = triple red, High = double orange, Moderate = single yellow, Low = dash. Assignment has a search box.
  - **Main area**: "Search incidents", a sort dropdown ("Creation date") with a direction toggle, and "1–3 of 3 incidents".
  - **Each card**: a red flag avatar, a bold title (e.g. "Daily Revenue"), and a linked asset with a platform icon (fct_orders / Snowflake QUERY_HISTORY). The meta line reads: status chip · "High severity" · "Last failure a month ago" · "5 Compromised dashboards".
  — [Sifflet Incidents](https://docs.siffletdata.com/docs/incidents)
- [SCREENSHOT] **Sifflet incident detail, Overview tab** (screenshot dated 2026-05-28):
  - **Header**: flag icon and title ("Metrics Monitor on int_value with Sum"), an "Open" chip and a "— Low" severity. On the right: a blue "Status ▾" button, "Create Jira issue", "Edit" and a kebab menu.
  - **Tabs**: Overview / Monitors / Downstream Impacts / Lineage / Notifications.
  - **Left column**: an AI-generated (sparkle icon) "Overview" in prose. It states the metric value vs the expected range (e.g. "SUM … was 10,501, which is lower than the expected range of 10,713 to 16,515"), the detection window, and the implication. Below is a "Root cause analysis and debugging recommendations" card with "Potential root causes" and "Suggested investigations".
  - **Right sidebar**: Details/Activity tabs, "1 Failing monitor" (link), "1 Impacted dataset" (link), and "Created 4 days ago by: {monitor}".
  — [Sifflet Incidents](https://docs.siffletdata.com/docs/incidents)
- [DOCS-TEXT] Sifflet incident fields:
  - **Name**: auto-generated, updated as failing monitors are added, and editable.
  - Data assets.
  - **Severity**: inherited from the monitor.
  - **Status**: Open / In Progress / Closed.
  - Assignees.
  - Last failure timestamp.
  - Compromised dashboards count.
  - A "Copy issue number" menu action.
  — [Sifflet Incidents](https://docs.siffletdata.com/docs/incidents)
- [DOCS-TEXT] Sifflet tabs:
  - **Monitors tab**: columns for type, name, status (Failing/Passing), severity, asset, first and last failure, last run and failed run count. Bulk "Qualify as passing" and "Reassign monitors" actions, plus a "Failing monitors only" toggle.
  - **Impacted Dashboards**: lists Tableau/Looker/Power BI assets.
  - **Lineage**: a graph centered on the involved assets, with failing assets highlighted.
  - **Notifications tab**: shows active destinations, or an empty state.
  - **Activity feed**: logs creation, status changes, added monitors, assignee changes, comments, monitor fixes, and Jira/ServiceNow events.
  — [Sifflet Incidents](https://docs.siffletdata.com/docs/incidents)
- [THIRD-PARTY] Monte Carlo statuses as exposed through a webhook integration: investigating, work_in_progress, fixed, expected, no_action_needed, false_positive. Severity runs SEV-1 to SEV-4 when an alert is marked as an incident. Events include alert_feedback, declared_alert_severity and owner. — [incident.io: Monte Carlo alerts](https://docs.incident.io/alerts/monte-carlo)
- [THIRD-PARTY] A Monte Carlo reviewer says severity, "a core concept of MC incidents", is not actually enforced. This is a single opinion. — [AWS Marketplace reviews](https://aws.amazon.com/marketplace/reviews/reviews-list/prodview-hikicsfohm3gg?page=29)
- [DOCS-TEXT] Monte Carlo blog: Incident IQ shows the anomaly's historical timeline and manages status, ownership and severity. — [MC blog: 10 quick tips](https://montecarlo.ai/blog-10-quick-tips-for-getting-started-with-monte-carlo)
- [DOCS-TEXT] Monte Carlo announced Observability Agents in April 2025. The Troubleshooting Agent investigates and explains root cause and recommends next steps. — [BusinessWire, Apr 2025](https://www.businesswire.com/news/home/20250417869977/en)

### Inferences
- Patterns worth copying:
  - **Status design**: icon-plus-color status chips that stay distinct even without color (arrow / check / clock shapes).
  - **Severity design**: chevron counts, so severity still reads in greyscale.
  - **Row layout**: a plain-English generated title plus an asset breadcrumb; an inline sparkline with the expected band in the list row (Bigeye); and a downstream-impact count on the card (Sifflet).
  - **Detail page**: the detail-page prose states the observed value and the expected range explicitly.
  - **Grouping**: grouping many alerts into one issue/incident is universal.
- Sifflet and Monte Carlo both place AI-generated root-cause narrative at the top of the detail page as of 2025–2026.

### Gaps
- No public screenshot of a Monte Carlo alert detail page. I could not verify its layout (e.g. chart placement or the "Incident IQ" panels).
- No screenshot of a Bigeye issue detail page.
- I found no evidence of failing-row sample display for any of the three.

## Table / asset health pages and anomaly chart drawing

### Takeaway
There is very little direct evidence. The only chart I saw is Bigeye's list-row sparkline (a metric line inside a shaded grey expected band, with a red marker on the anomaly).

### Cited Findings
- [SCREENSHOT] Bigeye inbox sparklines show a grey shaded threshold band around a dark metric line. The anomalous point ends in a red vertical line or segment. — [Bigeye View Issues](https://docs.bigeye.com/docs/view-issues)
- [SCREENSHOT] Sifflet's "Table uptime" uses a light-blue filled area sparkline next to a big % for each monitor type. — [Sifflet changelog](https://docs.siffletdata.com/changelog/new-sifflet-dashboard-a-central-hub-for-your-datas-health)

### Gaps
- I did not retrieve per-table pages (freshness, volume, schema history, column metrics) for any vendor.
- I found no evidence on how vendors draw excluded periods or annotations. This needs a follow-up on the vendors' monitor-detail docs pages.

## Coverage views

### Takeaway
Coverage is shown as a single percentage plus a split between monitored and unmonitored assets (Sifflet's gauge), or as "tables with no incident / total tables" per monitor type (Monte Carlo's uptime cards).

### Cited Findings
- [SCREENSHOT] Sifflet "Monitor coverage — Datasets grouped by monitor coverage" is a half-donut gauge: "65% Coverage", with Unmonitored in grey and native monitors vs dbt tests in two blues. — [Sifflet changelog](https://docs.siffletdata.com/changelog/new-sifflet-dashboard-a-central-hub-for-your-datas-health)
- [DOCS-TEXT] Monte Carlo's Table Uptime cards are labelled "(Coverage)" and show tables with no incident out of total tables, per monitor type. — [MC Data Reliability Dashboard](https://docs.getmontecarlo.com/docs/data-reliability-dashboard)

### Inferences
- Showing unmonitored assets in neutral grey, rather than red, separates "gap" from "failure". This is a useful convention.

### Gaps
- No drill-down list of unmonitored assets was found for any vendor.

## Scorecards / data quality scores

### Takeaway
None of the three vendors' public material I found shows a dimension-based DQ score with roll-up. They lead with uptime %, incident counts and coverage instead.

### Cited Findings
- [DOCS-TEXT] Monte Carlo uses an "Importance Score" (≥ 0.75 counts as a key asset) as a filter, not as a quality score. — [MC Data Reliability Dashboard](https://docs.getmontecarlo.com/docs/data-reliability-dashboard)
- [DOCS-TEXT] Bigeye's priority score combines table popularity, existing issue count and distance from threshold. It is shown on hover over the priority icon. — [Bigeye View Issues](https://docs.bigeye.com/docs/view-issues)

### Gaps
- The Sifflet `/docs/data-quality-score` URL returned 404, and I did not confirm whether a per-asset score exists.
- I found no evidence on how vendors display "no data / not run".
- Monte Carlo data products and domain scorecards were not checked.

## Monitor/check authoring UI

### Takeaway
I found little evidence on this. The only item seen is the AI-suggestion card pattern on Sifflet's home page, which proposes threshold or sensitivity changes as before→after values with Accept/Dismiss buttons.

### Cited Findings
- [SCREENSHOT] Sifflet AI Assistant cards show "AI ASSISTANT SUGGESTION: Lower model sensitivity from 50 to 25", rendered as "50 → 25", with a "Learn about AI model sensitivity" link and Dismiss/Accept buttons. — [Sifflet changelog](https://docs.siffletdata.com/changelog/new-sifflet-dashboard-a-central-hub-for-your-datas-health)
- [THIRD-PARTY] A Monte Carlo reviewer wants fewer dropdown menus and the ability to templatize custom alerts. — [G2 Monte Carlo reviews](https://www.g2.com/products/monte-carlo/reviews?page=4)

### Gaps
- No public screenshots or docs of monitor-creation forms, templates or test-before-save were retrieved.

## Notification design

### Takeaway
Bigeye is the best-documented case: one Slack thread per issue, whose root message is updated in place to show current status, with lifecycle events posted as replies and interactive buttons. Monte Carlo supports triage from Slack, where Slack thread comments sync to the incident.

### Cited Findings
- [DOCS-TEXT] Bigeye:
  - State changes and alert updates are posted as threaded replies to the original issue notification, and the root message is updated to reflect current status.
  - Users can mute Collections, acknowledge or close issues, and bind their Slack identity, all from Slack. Closing an issue opens a feedback dialog.
  - Tagging a user prefixes the message with a mention.
  - A Test button validates the configuration.
  — [Bigeye Slack](https://docs.bigeye.com/docs/slack); [Bigeye Slack interactions](https://docs.bigeye.com/docs/slack-interactions)
- [DOCS-TEXT] Bigeye notification triggers are: a metric starts alerting, it returns to healthy, the issue state changes, or a config change affects the issue. Feedback on whether thresholds need updating can be given in Slack. — [Bigeye blog, Mar 31 2023 (older)](https://bigeye.com/blog/streamlined-data-reliability-engineering-notifications-with-bigeye)
- [THIRD-PARTY] Monte Carlo, as summarized in search results from GitLab's handbook: teams can triage via Slack by assigning a status, and comments in the Slack thread are added to the MC incident automatically. — [GitLab Handbook: Monte Carlo Guide](https://handbook.gitlab.com/handbook/enterprise-data/platform/monte-carlo) (I could not re-confirm this on fetch because the page was truncated)
- [DOCS-TEXT] Sifflet notifications are sent on incident created, updated and closed. Each incident has a Notifications tab listing active destinations, and rules are managed under Settings → Notification rules. — [Sifflet Incidents](https://docs.siffletdata.com/docs/incidents)

### Inferences
- "One thread per issue, root message edited to current status" is the key anti-noise pattern for Slack and Teams.

### Gaps
- I found no screenshot or field-level spec of any vendor's actual Slack, Teams or email message body (title, metric vs expected, buttons).

## Reports / SLA views

### Takeaway
Monte Carlo reports response KPIs as medians: Time to Response (to first status), Time to Fixed, and Status Update Rate. Sifflet charts daily incidents stacked by status.

### Cited Findings
- [DOCS-TEXT] Monte Carlo: Status Update Rate, median Time to Response and median Time to 'Fixed', with domain breakdown. — [MC Data Reliability Dashboard](https://docs.getmontecarlo.com/docs/data-reliability-dashboard)
- [SCREENSHOT] Sifflet: an Incidents stacked bar by detection date, colored by current status, with Daily/period granularity. — [Sifflet changelog](https://docs.siffletdata.com/changelog/new-sifflet-dashboard-a-central-hub-for-your-datas-health)

### Gaps
- Bigeye SLA/report views were not found.

## Known UX complaints

### Takeaway
For Monte Carlo, the recurring complaints are alert overload and tuning effort, a cluttered UI ("not sure where to look"), slow incident detail pages, and a UI that keeps changing. I gathered no review data for Bigeye or Sifflet.

### Cited Findings
- [THIRD-PARTY] G2's AI-generated summary lists alert overload from automated monitors, an inefficient alert system, a lack of manual threshold settings, slow performance and disorganized features as cons. Ease of use is the top pro. — [G2 Monte Carlo reviews](https://www.g2.com/products/monte-carlo/reviews?page=4)
- [THIRD-PARTY] Individual G2 reviewers:
  - One describes the UI as "a lot is going on and I'm not sure where to look at".
  - Another says "it's a bit slow to open the details about the incident".
  - One says default monitors were overloaded with unimportant alerts until customized.
  — [G2 review 4659766](https://www.g2.com/survey_responses/monte-carlo-review-4659766); [G2 reviews](https://www.g2.com/products/monte-carlo/reviews?page=4)
- [THIRD-PARTY] A user review says the MC UI keeps changing, which makes it confusing. — [AWS Marketplace reviews](https://aws.amazon.com/marketplace/reviews/reviews-list/prodview-hikicsfohm3gg?page=2) (this came from a search summary, and I did not confirm the exact review page)

### Gaps
- Bigeye, Sifflet and Acceldata review complaints, and Reddit threads, were not collected.
