# 06 · Gaps, prior research, and the research questions

*Context file for a deep-research brief on data quality tools for Globe. As of 2026-10-18.*
*Sources: the program knowledge base, raw research (plan-b set, landscape sweep), the DQ Ops stabilization and future-state packs, the LEAP requirements shelf, and the team memory pool. Statements marked **[Inference]** are this file's reading, not a sourced fact.*

**How to use this file.** Section 1 says where we stand. Section 2 lists the gaps, each with an ID (G1 to G13). Section 3 summarises research already done so it is not repeated. Section 4 lists open decisions a tool choice would affect. Section 5 is the question list, each question tied to a gap ID. Section 6 is a scoring rubric.

**Framing that shapes every answer.** DQ Ops is a second-line assurance function: it audits, flags, and drives remediation; it does not build or enforce checks inside producer pipelines. Since 2026-08-11 DQ Ops does not author enterprise DQ rules (the rules function under data governance does; pipeline engineering implements). Governance scope is Silver; Gold transfers to Data Management. The target platform is GCP and BigQuery (the LEAP migration), with ServiceNow as the incident system of record and Clairvoyance (Grafana-based) as the enterprise-mandated alerting tool. InfoSec bars third-party apps that open their own connection to a production database; dbt connecting to the warehouse is approved. File 04 carries the full constraint set.

---

## 1. Current maturity position

### 1.1 The 6 x 8 maturity model read

The team built a vendor-neutral model: six levels (0 Ad Hoc, 1 Localized, 2 Centralized, 3 Monitored, 4 Measured, 5 Governed Control / Validate-at-Use) across eight axes (Standards, Coverage, Detection, Measurement, Ownership, Enforcement, Automation, Consumption). Ownership and Enforcement gate the headline: a domain sits at most one level above the weaker of the two. It crosswalks to EDM Council DCAM v3 (lead) and DAMA-DMBOK; CMMI-DMM is kept only as a legacy column (retired 2022). Two principles run through it: **Observability is not Quality** and **Detection is not Control**.

| Assessment | Capability avg | Gating floor | Headline |
|---|---|---|---|
| DQ Ops rule-based capability, 2026-06-06 | 3.63 | 3 | **Level 4, fragile** |
| Same, projected after the "Plan B" engineering roadmap | 4.00 | 4 | Level 4, solid |
| Golden KPIs domain (worked example) | n/a | n/a | Level 4 |

- Strong axes (4): Standards, Coverage, Detection, Measurement, Ownership.
- Trailing axes (3): **Enforcement** (detective, not preventive), **Automation** (manual rule lifecycle), **Consumption** (no validate-at-use).
- The research conclusion: more detection tooling deepens the Detection axis but does not move the headline. Level 5 needs block-before-impact at source, validate-at-use (contracts), and durable, funded ownership. Ownership is "a people/RACI move; no tool fixes it."
- Two altitudes are kept apart: DQ Ops's own capability (target solid L4) versus enterprise DQ (target L5, which DQ Ops drives through four levers: the standard, assurance at scale, flag-and-recommend, accountability).

**[Inference]** The June scoring predates the org move, the Silver-only decision, and the Elastic decommission. It has not been re-run. Bronze coverage has since dropped (see G1), so a fresh assessment would likely score Coverage and Detection lower for the full estate.

### 1.2 The control ladder (from the September scorecard work)

Reproducible from the 481-check registry. It shows where controls sit in the flow.

| Rung | State | Count | Where it runs |
|---|---|---|---|
| 1 Schema | Enforced | 1 control | In the pipeline at ingestion, the only rung that blocks |
| 2 Arrival | Partial | 29 of ~45 platforms | Control files, after landing |
| 3 Presence | Observed only | 394 of 481 | After the fact, Silver and Gold |
| 4 Validity and reconciliation | Observed only | 87 of 481 | After the fact |
| 5 Plausibility | Absent | 0 of 481 | Nowhere |

TDG's own pipeline engineering plan concedes value ranges and freshness are "discovered in production" today and targets a quality-contract standard with at least 90% contract coverage.

---

## 2. Capability gaps by area

| ID | Area | Where we are | The gap |
|---|---|---|---|
| **G1** | Detection coverage (Bronze / Silver / Gold) | 481 column-level checks (Silver 194 verbatim, Gold 287 needing review). Bronze has zero catalog checks. Bronze anomaly detection ran on Elastic, now **decommissioned**: no Bronze flagging from Oct 2026 until a replacement exists. Bronze lands on AWS S3 with no Snowflake/dbt reach. 11 of 13 Elastic-watched source systems appear nowhere else in our detection estate. | Bronze is dark. Coverage is inversely correlated with financial materiality (peso metrics such as revenue and ARPU had no assurance on either surface). B2B and customer-experience populations have 0 checks. Siblings are unevenly covered (one protocol variant had 0 rules while two siblings had 5 each). |
| **G2** | Plausibility / accuracy vs presence | 82% of rules test presence (null/blank 155, attribute population 239). No range check type existed as of 2026-07-21. | A broadband usage feed ran ~83x inflated for 8.5+ months while it was both rule-checked (not-null passed) and anomaly-monitored. Missing: absolute bounds, per-entity ratio checks (volume per subscriber), and cross-field plausibility. LEAP story 4.8 (plausibility) is the only Critical story with no policy home. |
| **G3** | Anomaly detection | Snowflake forecast-band plus a Median+MAD day-of-week model on curated KPI series. Documented fragilities: ~9-12% recall and an IS_ACTIVE starvation issue. Gold has 0 anomaly series for the CEO-facing metrics set. | A 30-day rolling window is structurally blind to a persistent error (it becomes the new median). Breadth is curated KPIs, not estate. Elastic showed the architectural asks: decouple evaluation from notification interval; per-dataset detection from one templated model so onboarding a source is a registration, not a modelling project. LEAP's named Bronze tool (Dataplex AutoDQ) is rule-based; the target stack has no statistical layer. |
| **G4** | Proactive gating / shift-left | Gating Priority Index designed (Business Impact x Urgency x Risk, each banded 1-5, Urgency floored at 1, ranked by source table). LEAP mandates configurable DQ checks embedded inline at ingestion and transformation; pilot in October. | Only schema is enforced at source. No quarantine or block-before-impact. GPI inputs (lineage, readout history) are partial. DQ Ops recommends gates; producers must build them, so the tool question is what producers run and what evidence it emits. |
| **G5** | Reconciliation | Gold accuracy target = reconcile within 0.3%. Control files check Bronze completeness against manifests (99.7% target). 87 checks cover validity and reconciliation. | Google's own feedback log (FB-071) records **no reconciliation capability** in Dataplex (KPI recon, tolerance variance, grouped recon, row-level source-to-target). Migration requires Snowflake-to-BigQuery Gold to reconcile byte-for-byte and KPI variance within ±1% (±0.2 units for ratios). The pipeline engineering contract sample has no control-total or manifest reconciliation. |
| **G6** | Lineage | A curated "makeshift" source-to-KPI map. A column-level Bronze-Silver-Gold lineage explorer exists in dev with one report onboarded. LEAP names Knowledge Catalog column lineage plus bidirectional sync with the ISG-owned DataHub catalog. | No automated lineage feeding blast radius, alert routing, or GPI Risk. Table-level lineage on the Monitor was parked as the most complex item. |
| **G7** | Data contracts | Contract-first DQ spec written (ODCS v3.1.0-compatible, seven blocks incl. fallout records, per-run quality log, a 31-code error matrix, incident integration). Aligned 2026-09-30: contracts only at ownership-boundary crossings, one centralised ODCS template. Only one Globe contract is true ODCS v3.1.0 (a reference-data pilot, with lint errors, no quality block, no SLA). | No contract registry (GitLab proposed, owner unnamed). No enforcement-levels document. DQ thresholds belong in a DQ standard that is still a template. Pipeline engineering samples are not ODCS syntax and carry binary dbt tests without thresholds, severity, or rule IDs. DQ Ops holds no RACI role on contract runtime monitoring. |
| **G8** | Observability / freshness | "Data Freshness" shipped on the Monitor, but it is alert-based (an alert fires only when late), not a measured load timestamp. | Per-table freshness is blocked on a `MAX(load_ts)` feed and a critical-table SLA list. Schema-drift detection is paused (none implemented). Google's freshness monitoring lacks the timeliness-SLA rule workflow (FB-062). |
| **G9** | Incident workflow / MTTR | ServiceNow INC+PRB lifecycle, verdict taxonomy, RCA validation, Detection Gaps and Coverage Gaps surfaces. Clairvoyance will auto-raise ServiceNow incidents. RCA substantiation baseline 53.3%. | Today's attribution is inferred from free text. Alarm storms will become ticket storms unless alerts are correlated and deduped before raise. No **remediation ledger** (the one audit store with no counterpart anywhere). Our verdicts cannot express "resolved before the consumer's scheduled pull." MTTR must name its population; the old "15d to ~3d" claim is retired. Elastic produced ~40 emails per confirmed incident. |
| **G10** | Ownership | Ownership naming belongs to the policy function, not DQ Ops. LEAP targets 80% Owner/Steward/Custodian adoption. | In the enterprise WBS, 20 of 45 Critical rows have no settled owner. BSLA compliance monitoring is unowned. Master/reference data and remediation evidence are the two real unowned gaps. |
| **G11** | Scorecarding / evidence | Enterprise DQ Scorecard v3 locked (YTD composite, three of four areas scored, Network "No score"). LEAP WS3-WP9 (DG DQ Scorecard, target 2027-01-31) is DQ Ops's one outright work package. | No baselines exist for DQ Index, proactive detection rate, time-to-detect, time-to-resolve, or rule coverage (FB-103). None of the five BSC scored measures was computable as of mid-August. No immutable results-readout store yet (target T4: measured value, good/total counts, threshold, severity, timestamp per run). Google's feedback log asks for a weighted DQ index composite, separation of technical compliance vs certification vs residual risk, and dimension trends. |
| **G12** | Network DQ | Network (NTG) DQ is certified at source: 227 rules over IBM Maximo inventory across domains, published as per-domain Looker Studio dashboards. Execution runs in NTG's own stack (policy names Python or DataCube). | Our registry reaches network with 6 checks. Two network rules (NE-Name format, PSGC code format) look like scoping defects that would publish false ~100% failure. Headline metric differs across domains (dimension average vs compliance %). Network shows "No score" on the enterprise scorecard. |
| **G13** | Metadata / catalog / dimensions | DataHub is ISG-owned; we register, not compete. Business glossary was dropped from the Monitor (owned by the metadata tool). CDE register de-scoped to 2027. | **Four** live dimension sets (NTG ACCVU 5, DGIM 8, EDS-AG 4, our ODCS spec 7); the choice is the governance function's. No shared rule register is possible until one wins. Severity Matrix (policy-owned) gates every BSLA formula and is not ratified. |

---

## 3. Prior research already done (do not redo)

### 3.1 Plan B benchmark, 2026-06-06 (two verified passes)

Method: deep-research runs with three-vote adversarial verification. Pass 1: 25 claims verified, 23 confirmed, 2 killed. Pass 2: 25 claims, all confirmed.

| Tool | Verdict |
|---|---|
| **dbt (Core) tests + dbt-utils** | The committed path. Registry compiles to 6 generic tests, one config row per check. |
| **dbt-duckdb** | Pick for the zero-credential path: reads CSV/Parquet/JSON exports with no DB connection; runs in CI with no credentials; experimental GCS and Google Sheets readers. |
| **dbt-on-Snowflake** | Approved warehouse execution. **Note:** the target platform is now BigQuery; the architecture review says target dbt-bigquery from the start. |
| **Snowflake Data Metric Functions** | Native serverless upgrade path from hand-written procs; needs Enterprise Edition. Cost vs dbt left open. |
| **DQOps** | OSS engine, ~150 templates, BSL 1.1, needs live JDBC. Gated by InfoSec and largely redundant. Parked; borrow its architecture (sensor / rule / check split, store readouts, check taxonomy, partitioned checks). |
| **Great Expectations OSS** | Runs file-only via pandas; Apache-2.0; has Data Docs. Option for an offline report UI. |
| **Soda Core** | Runs file-only via DuckDB/Dask; SodaCL YAML. **Licence in flux**: v4 reportedly moved to Elastic License 2.0 (the landscape table lists Apache-2.0). Unresolved, verify. |
| **Elementary OSS** | dbt-native anomaly tests; complements Median+MAD; Cloud tier paid. File-only operation not confirmed. |
| **metaplane/dbt_expectations** | Maintained fork (calogica is dead). Optional only. |
| **Recce** | dbt PR-diff review for our own check codebase; replaces the abandoned data-diff. |
| **datacontract-cli** | Optional ODCS lint on received exports. |

**Killed claims, do not repeat:** (1) one LLM-authored rule compiles into multiple heterogeneous enforcement targets (refuted 0-3; the proven target is dbt tests); (2) Great Expectations OSS has no UI (refuted; it has Data Docs).

### 3.2 Monitor feature benchmark, 2026-06-20 (16 tools)

Monte Carlo, Bigeye, Anomalo, Acceldata, Great Expectations, dbt, AWS Glue DQ, Snowflake Horizon, Google Dataplex, Collibra, Alation, Atlan, Microsoft Purview, Informatica IDMC, Talend, Ataccama ONE. Soda and Databricks Lakehouse Monitoring timed out (judged category-redundant).

- Conclusion: the Monitor is competitive on scoring, incident lifecycle, root-cause classification, and executive reporting. The most-cited gaps were freshness/volume/schema primitives, column profiling, outbound alerting, glossary, ownership view, contracts/SLA surface, lineage.
- Excluded by our constraints: broad live connectors, in-pipeline remediation and circuit breakers, MDM, access governance/masking, column-level lineage via query-log parsing, estate-wide auto-ML, FinOps, paid SaaS licensing.
- Triage (2026-06-23): built time-to-detect, an incident digest, freshness. Sent recurrence, ownership, contracts to the governance initiatives. Parked schema drift, profiling, dimension alignment, lineage, sensitivity feedback. Dropped per-incident LLM RCA (cost) and glossary.

### 3.3 Other prior work

- **Datadog patterns (2026-06):** 59 ideas mined, Datadog itself out of scope (SaaS on live connections). Keystone: a results-readout history store. Lighter verification; re-confirm before leadership use.
- **Reconciliation decisions D1-D10:** adopt **ODCS v3.1.0** as the format for recommendations to producers (not an engine DQ Ops runs); Snowflake MCP parked (approval); builder-validator two-LLM pass for drafted checks; "check-regression fixtures" (never "golden"); GitLab CI as the zero-cost orchestrator.
- **AI-DQ landscape (2026-04 and the 2026 sweep):** Claude on Bedrock and Vertex; commercial AI-native vendors Anomalo, Monte Carlo, Sifflet, Bigeye, Acceldata, Telmai, Metaplane (acquired by Datadog 2025); LLM rule generation in dbt Copilot, GX ExpectAI, SodaGPT, Databricks DQX; OSS catalogs OpenMetadata (built-in MCP server), DataHub, OpenLineage/Marquez, Amundsen (low momentum); OSS checks Pandera, Deequ, Cuallee; ML/LLM eval tools (Evidently, RAGAS, DeepEval, promptfoo, Giskard; Cleanlab and Deepchecks are AGPL). Verdict: "no vendor or OSS ships an LLM-driven DQ control plane end to end." Vendor statistics are vendor-published; POC on own data.
- **Elastic AD evidence case (2026-08):** Elastic was not inaccurate (91.0% vs 92.5% for Snowflake AD and 92.9% for rules on our own metric) but unactionable: 527 alerts to 11 confirmed incidents (48:1) vs 15:1 for statistical AD and 2.3:1 for rule checks. Rules out-yield statistical AD about 7x. False negatives and the counterfactual are not measurable from the corpus. Proposed replacement: AD as a platform service in LEAP, e.g. porting the model to BigQuery ML (`ARIMA_PLUS` / `ML.DETECT_ANOMALIES`), with a 60-90 day parallel bake-off.
- **LEAP Google feedback log:** ~90 items against Google's statement of compliance, effectively a gap analysis of Dataplex / Knowledge Catalog. Google covers rule execution and alerting; roughly none of the lifecycle (issue management, rule approval workflow, AI-suggested rules, error-code registry, reconciliation, anomaly configurability, quarantine, baselines). The proposed partner stack leans on Databricks DQX with no managed-product SLA, plus Great Expectations, Soda, DataHub, Protegrity.

### 3.4 What prior research did NOT cover

- **BigQuery-native execution.** Plan B was Snowflake-centric. Dataplex AutoDQ, BigQuery ML anomaly detection, and dbt-bigquery were not benchmarked hands-on.
- **Clairvoyance** capabilities (rules-as-code, correlation before raise, chargeback) are unconfirmed.
- Soda and Databricks Lakehouse Monitoring were never completed; GX and Soda were verified only for the file-only question.
- **Reconciliation tools** (migration diff, KPI recon) were not surveyed.
- **Plausibility tooling** (range, ratio, cross-field) was not surveyed as a category.
- **Network / master-data DQ** tooling (Maximo, MDM-adjacent validation) was not surveyed.
- **ODCS v3.1.0 tool support** (which engines consume the quality block natively) was not surveyed.
- **Cost:** self-generated estimates were withdrawn on 2026-08-12; costing must be joint with the platform team.

---

## 4. Open decisions a tool choice would affect

| Decision | Status | Decided by (role) | How a tool choice interacts |
|---|---|---|---|
| Check runtime for DQ Ops's interim monitoring checks (D6: dbt Core compiles, Snowflake tasks run) | Proposed, open | DQ Ops head | Sets which engine we standardise on before the Snowflake decommission |
| Warehouse target timing | Snowflake Gold decommissioned LEAP months 3-4; Snowflake and Databricks fully by months 12-15 | LEAP program | Any Snowflake-only tool has an expiry; favour BigQuery-native or portable |
| Bronze anomaly detection replacement | Open since Elastic decommission | LEAP / platform; DQ Ops proposes | BQML vs Dataplex vs vendor; must meet the AD-10 and AD-12 asks |
| Clairvoyance rules-as-code, INC field mapping, dedup before raise | Open, time-boxed design window | Clairvoyance and integration owners | Whether contract-generated alert rules are possible; what a DQ tool must emit |
| Enterprise dimension set | Open | Data governance (DGM) | Tool's dimension model and scoring must map to whichever set wins |
| Contract template, registry owner, enforcement levels | Partly aligned (one ODCS template); registry and enforcement unassigned | Data governance / rules function; platform plus data engineering co-own the registry | ODCS-native tooling; registry in GitLab vs a catalog |
| Catalog and lineage home | DataHub (ISG) plus Dataplex Knowledge Catalog with bidirectional sync named by LEAP | ISG / LEAP WS3 | Whether we buy, use DataHub, or keep our explorer |
| Serving layer | Apps Script stays for 2026; no Cloud Run this year; Cloud Run plus IAP later | DQ Ops head | A UI-heavy SaaS would compete with Clairvoyance and the Monitor (a third surface is "unjustifiable") |
| Funding and GCP provisioning | Platform team holds budget (in-year if warranted, else next year); provisioning owner still unnamed | Platform team | Licensed tools need a funding case; ₱0 stance is a stated differentiator |
| Severity Matrix | Not ratified | Policy function | Severity mapping in any tool |
| Gold transfer and CEO-facing metric set disposition | Gold transfer decided; disposition of the CEO-facing set open | TDG leadership | Whether Gold tooling is in scope for us |
| Agent platform boundary (Genesis) | Agent drafts monitoring checks only; kept out of runtime (runs on a person's credential) | DQ Ops head | AI features in a DQ tool must respect "AI authors, engines execute" |
| DQ Portal in LEAP Feature Set 2 | Dated Dec 2026, unowned cell | LEAP PMO | Possible overlap with any scorecard or portal product |

---

## 5. Research questions (prioritised, grouped by theme)

Priority: **P1** = needed for decisions open now; **P2** = needed within LEAP's next phases; **P3** = useful context.

### A. Execution engine and BigQuery fit

1. **(P1, G1, G4, G7)** Which DQ engines run natively inside BigQuery with SQL pushdown (no data egress, no separate connection service), and which can read an ODCS v3.1.0 quality block or compile from a YAML registry? Compare dbt-bigquery tests, Dataplex AutoDQ, Soda, Great Expectations, Elementary, and any others.
2. **(P1, G1)** What exactly does Dataplex AutoDQ (and Dataplex data profiling) support today: rule types, custom SQL, thresholds, incremental/partitioned scans, result tables, alerting hooks, pricing model, and quotas? Where does it fall short against the Google feedback log items?
3. **(P1, G1)** For a portable registry, what is the least-rework path from Snowflake checks to BigQuery: dbt adapter dispatch, a neutral spec (ODCS, SodaCL), or an engine with both dialects? What breaks in translation?
4. **(P2, G1)** What tools assess Bronze landing zones on object storage (GCS, and AWS S3 during the migration) for completeness and arrival against manifests, without a warehouse table?

### B. Plausibility and anomaly detection

5. **(P1, G2)** Which tools support absolute range, per-entity ratio (e.g. volume per subscriber), and cross-field plausibility checks as first-class check types, with dimension tagging? Give syntax examples.
6. **(P1, G3)** How do BigQuery ML `ARIMA_PLUS` / `ML.DETECT_ANOMALIES`, Dataplex anomaly features, and Elementary compare for per-dataset anomaly detection from one templated definition, with evaluation decoupled from notification, configurable lookback and seasonality, and false-positive feedback?
7. **(P2, G2, G3)** What methods catch persistent errors that rolling-window detection misses (fixed baselines, long-horizon references, change-point detection, peer comparison across brands or cohorts)? Which tools ship them?
8. **(P2, G3)** What does the evidence say on alert yield (alerts per confirmed incident) for ML-first observability tools versus rule-based checks? Prefer independent or customer-reported figures over vendor claims.

### C. Contracts and proactive gating

9. **(P1, G7)** Which tools natively consume ODCS v3.1.0 (lint, test, generate checks, detect drift), and how mature is datacontract-cli against BigQuery? Which support contracts only at ownership boundaries plus promotion gates internally?
10. **(P1, G4)** What patterns and tools let producers run inline checks during ingestion and transformation on GCP (dbt, Dataflow, Dataform, Dataplex) with quarantine or block actions, and emit a per-run quality log and fallout records our audit can consume?
11. **(P2, G7, G13)** Where do contract registries live in practice (Git repo, catalog, dedicated registry), and what tooling connects a GitLab-held contract to runtime checks and alerting?

### D. Observability, alerting, and incident workflow

12. **(P1, G9)** Can Clairvoyance (Grafana-based) hold alert rules as code, correlate and dedupe alerts by run or pipeline ID before raising a ServiceNow incident, and pass structured fields (rule ID and version, dataset, table, column, records failed and evaluated, severity, assignment group)? What do Grafana alerting and its ServiceNow integration support natively?
13. **(P2, G8)** Which tools provide per-table freshness, volume, and schema-drift monitoring on BigQuery from metadata (INFORMATION_SCHEMA, audit logs) at near-zero compute, with SLA thresholds?
14. **(P2, G9)** What tools or patterns maintain a remediation ledger (issue to owner to fix to verified closure to recurrence window) integrated with ServiceNow, without duplicating it as a second ticket system?

### E. Lineage, metadata, catalog

15. **(P2, G6)** How complete is Dataplex Knowledge Catalog column-level lineage for dbt-on-BigQuery and scheduled queries, and how reliable is a bidirectional DataHub sync? Can lineage drive alert blast radius and routing?
16. **(P3, G13)** How do catalogs (DataHub, OpenMetadata, Dataplex) ingest DQ results as metadata and map rules to a chosen dimension set and CDEs?

### F. Reconciliation and migration assurance

17. **(P1, G5)** Which tools do source-to-target reconciliation across Snowflake and BigQuery (row counts, aggregates, row-level diff, tolerance bands, grouped KPI recon) during a migration dual-run, and what are their data-movement and connection requirements?
18. **(P2, G5)** Which tools support control-total or manifest reconciliation as a contract-level check at ingestion?

### G. Scorecarding and evidence

19. **(P2, G11)** Which tools persist an immutable per-run results history (measured value, good and total counts, threshold, severity, timestamp) queryable in BigQuery, suitable as audit evidence? Which compute a weighted DQ index across dimensions and roll up column to table to domain?
20. **(P3, G11)** What do industry scorecards use to separate technical compliance, business certification, operational health, and residual risk?

### H. Network and master data

21. **(P2, G12)** What tools support rule-based DQ on operational master data (asset inventory such as IBM Maximo, reconciled to network management systems and reference codes) and how would results federate into an enterprise scorecard alongside lakehouse DQ?

### I. AI assistance and governance

22. **(P2, G1, G2)** Which tools offer AI-suggested checks from profiling or schema that output reviewable code (YAML, SQL, dbt) rather than executing autonomously, and how do they handle human approval? Compare Dataplex rule recommendations, GX ExpectAI, SodaGPT, DQX, and MCP-based patterns.

### J. Commercial, licensing, InfoSec

23. **(P1, all)** For each shortlisted tool: licence (OSI open source, source-available such as BSL or Elastic License, AGPL, commercial), deployment model (in-warehouse, in-VPC, SaaS), whether it opens its own connection to production data, and data residency on GCP Asia regions.
24. **(P2, all)** What are realistic cost drivers at our scale (BigQuery scan cost for scheduled checks, per-table or per-asset licensing, Dataplex scan pricing)? Express as drivers and formulas, not a single number.

---

## 6. Suggested evaluation criteria and scoring rubric

### 6.1 Knock-out gates (fail any one, stop)

1. **InfoSec data path.** Does not require a third-party application to open its own connection to production data, or has a credible in-warehouse / in-project deployment that the security function can approve.
2. **GCP / BigQuery future.** Runs on BigQuery (or is platform-neutral) and does not depend on Snowflake or Databricks surviving past LEAP.
3. **Licence clarity.** Licence terms confirmed for internal enterprise use; AGPL or source-available terms flagged for legal.
4. **Mandate fit.** Supports a second-line model: checks can be authored by producers or the rules function and read by DQ Ops; does not force DQ Ops to own producer pipelines.

### 6.2 Weighted criteria (score each 0-3)

Scale: 0 = absent, 1 = partial or workaround, 2 = native with limits, 3 = native and proven at scale.

| # | Criterion | Weight | What "3" looks like |
|---|---|---|---|
| 1 | BigQuery-native execution and pushdown | 12 | Runs as SQL in BigQuery, incremental/partitioned, no egress |
| 2 | Plausibility and check breadth | 12 | Range, ratio, cross-field, reconciliation, freshness, schema, as typed checks |
| 3 | Anomaly detection fit | 10 | Per-dataset from one template, persistent-error handling, decoupled notification, feedback loop |
| 4 | Contract and registry compatibility | 10 | Reads or compiles from ODCS v3.1.0 or a YAML registry kept in GitLab |
| 5 | Evidence and results history | 10 | Immutable per-run readouts with counts, threshold, severity, queryable in BigQuery |
| 6 | Alerting and incident integration | 10 | Rules as code, dedupe before raise, structured ServiceNow fields, works with Clairvoyance |
| 7 | Reconciliation and migration diff | 8 | Cross-platform, tolerance bands, grouped KPI recon |
| 8 | Lineage and metadata interoperability | 6 | Emits or consumes OpenLineage / DataHub / Knowledge Catalog |
| 9 | Dimension and scoring flexibility | 6 | Configurable dimension set and weighted roll-up |
| 10 | AI assistance with human review | 4 | Suggests checks as reviewable code; no autonomous write |
| 11 | Operating cost and team load | 8 | Low maintenance for a small team; transparent cost drivers |
| 12 | Vendor viability and support in the Philippines / APAC | 4 | Stable roadmap, regional support, managed SLA where commercial |
| | **Total** | **100** | |

Weighted score = sum of (score / 3 x weight). Report each tool's score with a confidence label (verified in docs, verified hands-on, vendor claim only).

### 6.3 Reporting rules for the research output

- Separate execution (what runs checks) from lifecycle (issue, ownership, evidence). Google covers execution; the lifecycle is where our gaps sit.
- Mark every vendor statistic as vendor-published unless independently confirmed.
- Name the population on every MTTR or detection-rate figure.
- Do not describe any tool as "self-healing" or "autonomous"; describe what it automates and who approves.
- Do not repeat the two killed claims in 3.1.
- Prefer tools that strengthen Enforcement, Automation, and Consumption (the trailing axes) over tools that only add detection.
