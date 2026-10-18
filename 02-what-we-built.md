# 02. What DQ Ops has built (inventory as of 2026-10-18)

**Purpose of this file.** It lists everything the Data Quality Operations (DQ Ops) team has built or specified, so a tools study can judge what a vendor or open-source tool would **replace**, **complement**, or **duplicate**. Facts come from the workspace (project docs, the BigQuery lineage reference, the check registry) and the team's engineering memory. Status words mean:

- **Live**: in production use today.
- **WIP**: built or partly built, not in production.
- **Parked / paused**: built, deliberately stopped, waiting on a decision.
- **Decommissioned**: switched off.
- **Spec only**: a document or design, nothing runs.

Cost note: almost everything below runs on Google Sheets, Apps Script, a BigQuery sandbox and existing Snowflake access. There is no licensed DQ tool in the stack today.

---

## Summary table

| Build | What it does | Stack | Status |
|---|---|---|---|
| TDG Data Quality Monitor | One dashboard for DQ alerts, incidents, DQI, anomalies, governance risk, team workload | Single-file HTML/JS (Chart.js) served by an Apps Script Web App; reads BigQuery marts plus some Sheets | **Live** (v2 since 7 Oct 2026, current release @115) |
| Alert ingestion + incident pipeline | Scrapes alert emails, parses them, joins ServiceNow incidents, tags verdicts and RCA categories | Apps Script on a Google Sheet (scraper, parser, categorizer, problems, freshness, narrative) | **Live** |
| Verdict taxonomy + RCA tagging | Rule-based classification of every resolved incident (True Incident, Valid Expected Behavior, Timeliness, False Alert, Inconclusive) | Regex rule engine in Apps Script; offline Node/Python replay harness | **Live** (final model 25 Sep 2026) |
| RCA Substantiation Rate | Grades whether resolution notes give evidence or a cause | Node script over the incident corpus | **WIP** (metric measured, panel not deployed) |
| LLM narrative layer | Writes exec headliners and tooltips from computed facts | Apps Script calling a Gemini API (free tier) | **Live** |
| Incident Digest email | Twice-daily open-incident email with PRB list | Apps Script (notify) | **Paused** (built, triggers not installed) |
| BigQuery data layer | Warehouse copy of all Monitor data; marts the page reads | BigQuery sandbox: 4 datasets, scheduled queries, stored procedures | **Live** (sandbox project, not production) |
| AD KPI anomaly pipeline | Shows Snowflake anomaly-model output per KPI on the Monitor | Snowflake view → dbt-based laptop export (6-hourly) → Drive CSV → Apps Script loader → Web App → page | **Live** (laptop-fed) |
| IN Summary Median+MAD mirror | Own robust anomaly model for brand/channel cuts | Apps Script on a mirror Sheet | **Stale** (source frozen 26 Jul 2026; drilldowns removed) |
| Elastic anomaly detection (Bronze) | Hourly record-count anomaly alerts on ingestion | Elastic ML job (operated outside DQ Ops) | **Decommissioned** (Oct 2026) |
| DQ-check catalog / registry | Machine-readable list of every rule check, thresholds, dbt mapping | `checks.yml` in GitLab, Python seed/coverage scripts, dbt design doc | **Live as catalogue**; dbt build **deferred** |
| Check registry + intake plan | One registry for all check families, results in BigQuery, AI-drafted intake | Plan: Snowflake TASKs, GCS export, BigQuery, GitLab MR approval | **Spec only** (Phase 0 not started) |
| Data Lineage explorer | Column-level Bronze→Silver→Gold trace from a report number to source | Python builders over mapping docs + view SQL; HTML explorer on its own Apps Script Web App | **Live dev release, paused** for team review |
| Control Files tracker + agent team | Tracks ~29 platforms onboarding to control-file monitoring | Static HTML dashboard; Claude Code PM + 23 platform agents reading Gmail | **Live** (run on demand) |
| Data Contracts DQ spec | Contract-first DQ framework on ODCS v3.1.0 | Markdown/YAML spec | **Spec only** (WIP) |
| Contract DQ score monitoring (MRD) | Shows a producer's per-row contract DQ score on the Monitor | BigQuery scheduled query + marts; Apps Script notifier; GitLab poller | **Live** (pilot table, shown but not graded) |
| Enterprise DQ Scorecard | One-page leadership DQ scorecard (YTD) | Python build scripts → static HTML | **v3 locked** 10 Sep 2026; new version in **WIP** |
| Cerberus (Genesis agent spec) | DQ Ops agent persona + guardrails for a vendor agent platform | Markdown spec; platform provisioned 2 Oct 2026 | **Spec submitted**; no blueprint running |
| DQ Maturity model + skill | 6-level × 8-dimension maturity scoring | Claude skill, Excel instruments, HTML scorecard | **Live** (used on demand) |
| Claude Code skills + agents | Repeatable ops tasks (pulls, exports, renders, refreshes, check authoring) | Claude Code skills, 4-agent dev team | **Live** |
| DQ Ops Unified Dashboard (OPX) | Incident analytics: PDR, coverage gaps, MTTR | Apps Script + BigQuery (built by the incident analyst) | **Being absorbed** into the Monitor as a native panel |

---

## 1. TDG Data Quality Monitor (the dashboard)

**Purpose.** The team's single control surface. It turns scattered alert emails, ServiceNow incidents, problem tickets (PRBs), the monthly DQI report and anomaly-model output into one view across the medallion layers (and now a fourth layer for master/reference data).

**Users.** DQ Ops analysts and managers; executives (Executive Dashboard is the landing page); Managed Reports Ops (IN Summary pre-publish check); data analysts who consume Silver/Gold tables. Team Hub pages are restricted to DQ Ops.

**How it works.**
- **Frontend**: one HTML file with all CSS/JS inline (about 1.5 MB), Chart.js. Served by a single domain-restricted Apps Script Web App, embedded in Google Sites.
- **Data path**: the page calls the backend with `google.script.run` (same-origin). Source Sheets are Restricted, so the Web App is the only way in. Two earlier designs (public-sheet fetch; cross-origin proxy) were rejected, the first on an InfoSec finding.
- **Since v2 (7 Oct 2026)**: most panels read BigQuery marts (`dqops_mart`) through one shared read-only query helper in the backend. Some panels still read Sheets or Drive (see section 4).
- **Deployment**: `clasp` push from the repo, versioned deployments, rollback by version number. Every release is an md5-gated patch chain.
- **Speed work (Plan 30, 15-16 Oct)**: slim per-page payloads, cache warmed every minute with the Executive page first, whole-slot cache swap after each 2-hour refresh. Executive page load went from about 1 minute to about 10 seconds.

**Status.** Live. Current release @115 (16 Oct 2026).

**Panels (current navigation).**

| Panel | What it shows |
|---|---|
| Executive Dashboard | Catch → Confirm → Fix → Trust questions: caught by a check first, alerts that were real, incidents resolved, days to close, report health cards, timeliness, DQI summary, alerts per month by layer |
| IN Summary Daily | One row per alert tagged to the IN Summary report, for Managed Reports Ops before they publish; "Data as of" stamp tied to the 2-hourly update slot |
| Anomaly Monitor | Snowflake anomaly-model output per KPI (see section 3); "by KPI" and "by table" views (table lens added 1 Oct) |
| Alert Monitoring | Layer health, verdict mix, detection rate, Reports Impact tiles per report, timeliness by table, SLA on alert tickets, MRD contract view |
| DQI Report | Monthly Data Quality Index, Silver and Gold composites, MRD tab |
| Incident Response (OPX) | Incident flow, user-raised vs check-raised, "were the tickets real?", incident registry, table view of sources with partition dates, verdicts, RCA, linked PRBs |
| Governance & Risk (WIP badge) | Resolver-team lanes, an R1 to R11 governance risk register with "how we know" evidence drawers, "Tables to add DQ checks to" priority queue, table hotspots |
| SRG | Placeholder, not available yet |
| DQ Ops workload (Team Hub, locked) | Jira tickets mapped to the talent allocation register, workload board |
| About the DQ Team | Static: service catalogue, "how we grade" (targets, SLA tables) |

**Key capabilities a tool would need to match.**
- **PRB-aware coverage math**: an alert covered by an ongoing problem ticket is not counted as a tracking gap.
- **Cross-surface consistency rule**: one helper decides each report's RAG band, mirrored line for line in the narrative backend, so no two surfaces disagree.
- **Detection Gaps intake**: analysts log incidents the checks missed; they show as caveats and never enter detection counts.
- **Timeliness status v3**: "fresh" means the data arrived, not "no open ticket". Three states: Overdue, Unverified, Fresh.
- **Honest grading conventions**: unmeasurable shows grey, never green; every number names its base; no composite risk score; Network or onboarding scopes show "No score", never zero.
- **KPI canonical map**: collapses messy KPI spellings into three parents (Subscriber, Reload, Usages) and sub-flows.

**Known limitations.**
- Apps Script quotas and payload size drive a lot of engineering (caching, warmers, slicing).
- The intake is still Gmail alert emails and Sheets; BigQuery snapshots them every run.
- Bronze is greyed from October 2026 ("Paused, no AD tool") after Elastic was switched off.
- Gold DQI reads a flat 100% since Feb 2026 because two failing tables left the check population. It is survivorship, not improvement.
- The DQI depends on two spreadsheets owned outside DQ Ops.

---

## 2. Alert ingestion, incidents, verdicts and narrative

### 2.1 Alert scraper, parser and ServiceNow consolidation

**Purpose.** Turn every DQ alert email into a structured alert row, and attach the ServiceNow incident (INC) that was raised for it.

**How it works.**
- An Apps Script scraper reads alert emails from Gmail (month by month, to get around a 500-thread search cap). Sources: Snowflake rule checks, the Snowflake anomaly model, and formerly Elastic.
- A parser splits threads per incident, recovers the true source table, and writes the All Alerts tab (21 columns).
- Since 31 Jul 2026, incident status comes from **one consolidated "All INCs" sheet**: the incident team's ServiceNow export ("Report" tab) joined with a ServiceNow snapshot tab. Layer became a column. Live-verified: 850 INCs, 0 duplicates.
- Refresh: 8 intraday runs, every 2 hours from 05:30 to 19:30.

**Status.** Live.

**Known limitations.**
- No direct ServiceNow API connection; incident data arrives through a Sheet export every few hours, so status can lag by hours.
- A latent bug: the ServiceNow snapshot sometimes appends a new row per change and the code keeps the first. State was fixed (22 Sep); resolution notes and resolved date still come from the oldest row.
- About 227 user-raised or infrastructure tickets are deliberately out of the alert scope.

### 2.2 Verdict taxonomy and RCA tagging

**Purpose.** Classify every resolved incident so detection rate and noise are measurable. Analysts tend to type "false alert" on almost everything, so the label itself carries no information.

**How it works.** A stateless rule engine (regex over resolution notes only, never the machine-written description) assigns a verdict and an RCA category/sub-category. It re-tags the whole sheet on every sweep, so a rule change restates every month. Verdicts:
- **True Incident (TI)**: a real fault, including a delay with a named cause, or another ticket cited.
- **Valid Expected Behavior (VEB)**: a real movement confirmed valid (holiday, campaign, source confirmed, reconciliation passed).
- **Timeliness**: data arrived late against D-1 expectation; still a valid detection.
- **False Alert (FA)**: proven noise only (alert defect, flagged value unchanged, or high count shown on trend in words).
- **Inconclusive**: resolved but unclassifiable. **Non-DQ issue** for user-raised platform, access or enhancement tickets.

**Status.** Live. Final model deployed 25 Sep 2026: on 1,165 rows, FA 46, Inconclusive 5, detection rate 95.5%. A replay harness reproduces stored verdicts with 0 mismatches before any change.

**Known limitations.** Pure pattern matching, maintained by hand through periodic sweeps. Quality depends on what resolvers write. An LLM classifier has not been adopted; the rule engine is the source of truth.

### 2.3 RCA Substantiation Rate

Grades whether resolution notes give evidence (for dismissals) or a cause (for true incidents). Baseline **53.3%** of 831 resolved alerts (TI 93.8%, Timeliness 42.1%, Bronze 23.9%). Kept outside the verdict block so it never moves the detection rate. **Status: WIP**, preview awaiting sign-off, not deployed.

### 2.4 LLM narrative layer

Apps Script sends computed facts to a Gemini model (free tier) and writes exec headliners and card tooltips into a NARRATIVES tab, which BigQuery reads back every 15 minutes. False "all clear" phrasings are hard-banned. **Status: live.** The target architecture names Vertex AI as the future narrative layer; Vertex is not enabled on the sandbox.

### 2.5 Incident Digest email

**Purpose.** Twice-daily (start and end of day) email of open incidents: one table sorted by age, longest standing and most recent callouts, open problem tickets, detections with no ticket, CSV attachment.

**Status.** **Paused.** Code deployed and tested (v2, 3 Jul 2026); recipients not confirmed and triggers not installed, waiting on team feedback. BigQuery marts for it are built but unread.

---

## 3. Anomaly detection

### 3.1 AD KPI anomaly pipeline (Anomaly Monitor panel)

**Purpose.** Show the Snowflake anomaly-detection model's daily verdict for each top KPI, so analysts can see "what is anomalous today" without opening Snowflake.

**Who owns what.** The model itself runs in Snowflake and is not ours. DQ Ops built the distribution and visualisation layer.

**How it works (four layers).**
1. A scheduled laptop task (every 6 hours) pulls 90 days of the model's output view through the approved dbt path and writes a CSV to a Drive folder. A Claude skill runs the same export on demand and handles VPN and Drive-mount hurdles.
2. A container-bound Apps Script loader picks the freshest source (falls back to the legacy daily dump if the CSV is stale), filters active rows and 90 days, and precomputes a compact "served" tab.
3. The Web App reads the served tab (falls back to a full scan), with a cache and a 10-minute warmer.
4. The page draws one bar chart per KPI (30-day window): actual, forecast, translucent expected band, amber anomaly dots. About 52 curated KPIs across IN SUMMARY, DELPHI and CWN, grouped by the three parents. A "by table" lens maps each KPI to its source table and column.

**Status.** Live. A plan to move the feed from laptop to Snowflake → GCS → BigQuery is **on hold**. A BigQuery copy of the metrics already accrues history from the Drive CSV.

**Known limitations.**
- Depends on one person's laptop, VPN and Drive mount.
- **A rolling-window model cannot see a persistent error.** A broadband usage table was about 83× inflated for 8.5+ months while it was a monitored series: after 30 days the bad level becomes the baseline. The monitor answers "did it change?", never "is it plausible?".
- Gold has zero anomaly series; nothing watches a computed Gold number move.

### 3.2 IN Summary Median+MAD mirror

The team's own robust model (median + MAD, 8 same-weekday lookback) for brand, channel and data-brand cuts of the IN Summary report. **Status: stale.** The report team moved to long-format tabs after 26 Jul 2026; the mirror kept "succeeding" with no new dates because it checks columns, not freshness. Five drilldowns were removed on 22 Sep. Lesson: our own pipelines also lack freshness checks.

### 3.3 Elastic anomaly detection (Bronze) and its decommissioning

**What it was.** One Elastic ML job on hourly record counts across 13 Bronze sources, operated outside DQ Ops, sending identical-subject emails.

**What the team measured (Jan to Aug 2026).**

| Engine | Alerts → confirmed incidents | Actionable rate | Detection rate (1 − proven FA) |
|---|---|---|---|
| Elastic (Bronze) | 527 → 11 (48:1) | 6.2% | 91.0% |
| Snowflake AD (KPI anomaly) | 820 → 54 (15:1) | 11.9% | 92.5% |
| Snowflake rule checks | 136 → 59 (2.3:1) | 60.2% | 92.9% |

- About 40 emails per confirmed incident; bursts up to 37 hourly emails in a row; no per-source threshold; no daily trigger or re-notify suppression.
- 6 of 8 cross-referenced Elastic incidents were opened after another monitor's ticket.
- All Bronze alerts carried a blank category, so no dimension, KPI or lineage.
- Conclusion: Elastic was not inaccurate but **unactionable**, and its architecture could not be tuned without hundreds of jobs. False negatives were not measurable.

**Status.** **Decommissioned** (no Bronze flagging from Oct 2026). The replacement is expected from the enterprise migration program (anomaly detection on ingestion). Until then, Bronze has **neither rule checks nor anomaly detection**.

---

## 4. BigQuery data layer (sandbox)

**Purpose.** Move the Monitor's data from Sheets into a warehouse, following the TDG head's direction (ServiceNow as system of record, GCP and BigQuery for all data).

**How it works.**
- A TDG GCP sandbox project (asia-southeast1). DQ Ops datasets: `dqops` (226 objects: external tables over Sheets/Drive, raw snapshots, staging, reference, core facts, engine, UDFs, procedures), `dqops_mart` (45 marts, the only layer the page reads), `dqops_landing` (rows written by Apps Script), `dqops_restore` (149 point-in-time snapshots).
- Scheduled queries: full refresh every 2 hours, narratives every 15 minutes, an hourly freshness watchdog, a daily contract-score snapshot. It also reads the incident team's own BigQuery tables (ServiceNow incident report, built every 2 hours).
- Rules: one base table per entity, everything else derived as views; a mart stays a table only with a stated reason. Every change is gated on two harnesses plus byte-identical served responses.

**Status.** Live on the sandbox. Not a production project. No service account (runs under a person's identity). Cloud Run deploy rights are missing.

**Known limitations.** The incident register lacks a report on about half of Silver incidents (339 of 666) and a KPI on 288; one parity check is broken.

**Still Sheet or Drive sourced.** Alert emails, notes, problems and detection gaps (Sheet is the intake); narratives; DQI scores (computed by hand monthly in a Sheet); rule-result workbooks; AD KPI metrics; the digest; the v1 Sheet engine and categorizer (kept for parity).


---

## 5. DQ-check catalog and registry

**Purpose.** A version-controlled list of every rule-based DQ check, so adding or scaling a check is a one-entry edit and thresholds come from the framework, never invented.

**How it works.**
- `projects/dq-checks/catalog/checks.yml` in the team's GitLab repo. One entry per layer, table, column, check type, threshold, intended dbt test, status, review flag, plus lineage tags (`source_system`, `feeds_kpi`) on new entries.
- Check types: NULL/BLANK, VALIDITY, DUPLICATE, ATTRIBUTE_POPULATION, RECONCILIATION, TIMELINESS, CUSTOM_SQL.
- Thresholds from the internal DQ framework (four dimensions × three layers).
- A Claude skill (`dq-author-check`) adds entries. A dbt design (six generic tests, catalogue→test generator, dummy-schema profiles, CI target) exists on paper.
- The checks themselves execute today as hand-written Snowflake stored procedures, one wrapper proc per table, owned outside the catalogue.

**Current content.** 506 entries: Silver 194, Gold 287 (all flagged `needs_review`, intent inferred from table naming), MRD 25 (contract checks run by the producer). **Bronze: zero.**

**Status.** Live as a catalogue. **dbt build deferred.** Catalogue status is not execution status: 394 of 481 checks were observed running after the fact.

**Known limitations.**
- 82% of rules test presence (null/blank, attribute population). **None test an absolute range**; no plausibility or ratio checks existed as of July 2026.
- Coverage is inversely related to financial materiality: peso metrics (revenue, ARPU) had no assurance on either surface.
- 60% of the catalogue sits in Gold, the layer transferring to another team.
- ML/anomaly checks are documented in `ml-checks/` and are not in dbt.

**Prior research.** A June 2026 benchmark recommended finishing a dbt-native build rather than adopting DQOps (BSL licence, needs live JDBC). It found dbt-duckdb can run checks on exported files with zero database credentials, and that GX OSS and Soda Core also run file-only (Soda Core v4 moved to the Elastic License 2.0).

### 5.1 Check registry and intake plan

**Purpose.** One registry for every check family DQ Ops runs (rules, anomaly, CHM logs, control files, timeliness, retired Elastic), with results in BigQuery and AI-assisted intake.

**Design (decided D1 to D5).** Checks run in a new schema in the group sandbox database on the production Snowflake account; results exported to GCS and loaded to BigQuery; intake = AI drafts a check, a person approves via GitLab merge request, Snowflake runs it. Proposed (D6): dbt Core compiles, Snowflake TASKs execute. Checks run by other teams are out of scope.

**Status.** **Spec only.** Phase 0 not started; a production Snowflake connection is not yet established. These are explicitly **interim monitoring checks**, not enterprise standards (another team will set those).

---

## 6. Data Lineage explorer

**Purpose.** One column-level lineage tool so operations can trace any alerted report number to its source: platform → table → column → transform.

**Users.** DQ Ops; target consumer is DataOps for issue resolution.

**How it works.** Python builders read official mapping workbooks (about 1,744 column rows) and view-definition SQL, standardise them into a Bronze / Silver / Gold model (JSON), and generate an interactive HTML explorer. Open ticketed incidents are overlaid on the impacted nodes. Served by its own domain-gated Apps Script Web App, separate from the Monitor. A Claude skill (`lineage-onboard`) adds the next report into the same model.

**Coverage.** IN Summary onboarded (about 107 nodes after a July refresh); a CWN (CEO-facing scorecard) lineage package built separately.

**Key capabilities.** Filter-aware impact (sources dropped before Gold do not light up Gold); column-level upstream trace; search by INC number.

**Status.** Live dev release (2 Jul 2026), **paused** since 3 Jul for team review. Eventual home is a panel in the Monitor.

**Known limitations.** Built from documents, not from query logs or a catalog; one report fully onboarded. It is **unofficial**: the official lineage tool is DataHub (enterprise Data Hub), and the team decided the Monitor must not present itself as a lineage tool. Measured build cost of the first report was about $352 in model usage.

---

## 7. Control Files tracker and agent team

**Purpose.** Track about 29 source platforms (target about 45) onboarding to control-file-based proactive monitoring under an internal policy. A control file ships with each ingestion and declares row count and send time, so DataOps can alert on count or timing drift. In-scope incident turnaround reportedly fell from about 15 days (2025) to 1 to 4 days (Feb 2026).

**How it works.** A static HTML dashboard (the inline row array is the source of truth). A Claude Code skill pre-scans a Gmail label for activity, spawns only the platform agents with new threads (from a persistent team of 1 program manager + 23 platform agents), and proposes numbered edits that the team lead approves. Rules: every claim cites a thread; incidents are out of scope.

**Status.** Live, refreshed on demand.

**Known limitations.** Email-driven and manual-approval by design; tracks implementation progress, not data quality results. The control-file checks themselves are run by DataOps, not DQ Ops.

---

## 8. Data Contracts

### 8.1 Contract-first DQ specification

**Purpose.** Make the data contract (ODCS v3.1.0) the place where DQ rules, thresholds, fallout capture and monitoring config are defined, so an AI can generate checks from it.

**Content.** About 1,200-line spec: principles, six contract blocks, seven quality dimensions, threshold guidance, an AI rule-generation spec, a 30-code error matrix, a catalogue of 60+ dbt tests, fallout and quality-log rules, a worked template; plus a DQ-only sample contract.

**Status.** **Spec only.** Enterprise direction has since narrowed contracts to ownership boundaries (source→Bronze, domain→domain, external→local), so internal hops fall to DQ policy gates. There is no central contract registry; GitLab is proposed. The contract standards document is owned by another department and not yet written.

### 8.2 Contract DQ score monitoring (MRD layer)

**Purpose.** Show a producer team's per-row `dq_score` from a contract-governed master/reference data table on the Monitor, daily, without recomputing it.

**How it works.** The producer's dbt converter turns the ODCS contract into 25 rule flags and a weighted score on every row of a reference geography table (about 49,555 rows) in the BigQuery sandbox. DQ Ops reads it daily (own scheduled query), stores daily rule and score facts, keeps failing-record evidence for the latest run (capped at 1,000 per check), and serves it on Alert Monitoring and DQI as a fourth layer, "MRD". A GitLab poller detects contract changes; an Apps Script notifier sends one daily alert email.

**Status.** **Live** on the pilot table, shown as "Onboarding: shown, not graded yet", outside all totals. Read-only on the producer's data.

**Known limitations.** One table; the producer table keeps no history (we snapshot it); dev branch, not production.

---

## 9. Enterprise DQ Scorecard

**Purpose.** One leadership page scoring enterprise data quality across Subscriber, Reload, Usage and Network.

**How it works.** Python build scripts compute YTD scores from the DQI sheet and incident tabs and render a static TDG-branded HTML page. Four dimensions run today (Completeness, Timeliness, Accuracy, Consistency). Network shows "No score" and is excluded from the composite. Bands: Good ≥97, Fair 95 to 97, Poor <95 (provisional).

**Status.** v3 locked 10 Sep 2026 after two earlier versions were rejected by leadership as too technical. A new version measured against the TDG DQ framework is in **WIP preview**.

**Known limitations.** Hand-run build; no mapping yet from scores to the three business focus areas (shown as asks); band conflict with the DQI sheet unresolved.

---

## 10. Agents: Cerberus (Genesis) and DAQO saved prompts

**Genesis** is a vendor agent platform TDG is onboarding through Data Platform Engineering. An agent is knowledge + tools + guardrails; guardrails are blueprint phases and scoring thresholds that force human review; OAuth runs on individual user credentials, not service accounts; a validation UI ("Prover") works issue lists to zero.

**Cerberus** is the DQ Ops agent spec (persona and guardrails). Scope: rank coverage gaps by consequence, triage exposure, judge control effectiveness, screen RCA for evidenced vs asserted cause, explain scores, and draft interim monitoring checks for human approval. Hard rules: it never computes or publishes a score (those stay in dbt/BigQuery), never asserts a remediation SLA, and stays out of check execution.

**Status.** Platform provisioned for DQ Ops on 2 Oct 2026. Spec submitted. No blueprint built; connections and scheduling not confirmed. Overlap with a Vertex AI narrative layer is undecided.

**DAQO saved prompts.** The Data Assurance side runs two saved Gemini instructions (business-rule ownership acceptance; assessment artifact triage) behind Apps Script intake and email. They are candidates to port to Genesis; the decision belongs to that team.

---

## 11. Maturity Scorecard, Claude Code assets, other team tools

**DQ Maturity model.** Vendor-neutral 6 levels × 8 dimensions (Standards, Coverage, Detection, Measurement, Ownership, Enforcement, Automation, Consumption), crosswalked to DCAM v3, DAMA-DMBOK, ISO 8000, Gartner (CMMI-DMM as legacy). A Claude skill produces a markdown report and HTML scorecard from Excel instruments. Self-assessment: the Golden-KPI domain scores **Level 4 (Measured)**, held back by Enforcement 3 (detective, not preventive), Automation 3 (manual rule authoring) and Consumption 3 (no validate-at-use). **Live**, used on demand.

**Claude Code skills (in the repo).**

| Skill | Use |
|---|---|
| `snowflake-pull` | Read-only SQL pulls, column profiles, freshness spot-checks via the approved dbt path |
| `ad-kpi-export` | On-demand AD KPI export to Drive |
| `kpi-anomaly-render` | Static PNG of any anomaly series |
| `dq-author-check` | Add checks to the registry |
| `lineage-onboard` | Add a report to the lineage model |
| `control-files-refresh` | Activity-gated tracker refresh |
| `dq-owner-request` | Map INC/PRB tickets to tables/attributes for the data-owner request template, from BigQuery |
| `dq-teamhub-refresh` | Weekly Jira + allocation refresh with checks |
| `dq-maturity-assessment`, `dq-deck`, `dq-wins-refresh` | Maturity scoring, exec slides, wins tracker |

Plus a 4-agent development team (lead, frontend, Apps Script backend, read-only design reviewer) and a `dq-analyst` agent for quick data investigations. The InfoSec rule throughout: AI authors and interprets; engines execute; AI never runs against live production databases except through the approved read-only path.

**DQ Ops Unified Dashboard (OPX).** Built by the team's incident analyst: Apps Script + BigQuery over the incident team's ServiceNow tables. Tabs: Summary (Proactive Detection Ratio, totals, coverage gaps, MTTR), Detection & Coverage, MTTR. Being absorbed into the Monitor's Incident Response panel; the standalone retires at cutover.

**Clairvoyance onboarding pack.** Clairvoyance (Grafana) is the enterprise-mandated observability and alerting tool. DQ Ops prepared an onboarding brief (current data flows, BigQuery lineage, users, signals), approved but **not yet sent**. It asks Clairvoyance to read anomaly results directly from Snowflake.

---

## What a tool would need to beat or plug into

**What we already have that a tool would duplicate (high bar to replace):**
- A working ops dashboard with honest grading rules, PRB-aware coverage, detection-gap caveats, a verdict model with a 0-mismatch replay harness, and exec narratives. A tool's default dashboards would need to reproduce these rules or feed our marts, not replace them blindly.
- A BigQuery mart layer the Monitor reads. A tool should write results into BigQuery (or be readable from it), not become a second source of a number.
- A rule catalogue (`checks.yml`) with framework thresholds and a dbt design. A tool should import or compile from it, not force a parallel registry.

**Gaps where a tool could add real value:**
1. **Bronze detection.** Zero rule checks and no anomaly detection since Elastic was switched off. Needs per-source thresholds, sane alert grouping (daily digest, re-notify suppression) and categories on every alert.
2. **Plausibility, not just presence.** Absolute range bounds, per-entity ratio checks, and detection of persistent (step-change) errors that rolling windows absorb. Peso metrics first.
3. **Check execution engine.** Something that runs the catalogue on a schedule in Snowflake and BigQuery, stores time-series results (not just pass/fail), and replaces one-proc-per-table. Must fit the InfoSec path (no app-to-production-database connections beyond approved routes; ideally runs on exported files or inside the warehouse).
4. **Gold monitoring.** Zero anomaly series on Gold.
5. **Freshness at table level.** Measure `MAX(load_ts)` per table, including tables that never alert, instead of inferring from tickets.
6. **ServiceNow integration.** Direct API in and out (auto-raised incidents, status sync) instead of Sheet exports.
7. **Lineage.** Query-log or catalog-based column lineage that integrates with DataHub (the official tool) rather than our document-built explorer.
8. **Contract enforcement.** Compile ODCS contract quality blocks into checks across producers, with a contract registry.
9. **Removing single points of failure.** The laptop AD export, person-credential scheduled queries, and hand-run DQI.

**Integration points a tool must plug into:** Snowflake (several accounts; read via the approved dbt path), BigQuery (`dqops` / `dqops_mart`), ServiceNow (INC/PRB), Clairvoyance/Grafana (alarms), GitLab (registry and MR approval), Gmail-based alert intake (current), Google Workspace auth (domain-restricted access), and Genesis/Cerberus as a read-only analysis layer. Any tool must accept that DQ Ops **specifies and audits**; pipeline owners implement checks at source, and another department sets the enterprise standards.
