# 01. Program, Org and Mandate

*Context file for a deep-research brief on data quality tools for Globe Telecom. Compiled 2026-10-18 from the DQ 2026 workspace knowledge base and team memory. Facts are as recorded in those sources. Items marked DRAFT, PROVISIONAL or OPEN are not settled and should not be treated as Globe policy.*

---

## 1. Who we are

### Org placement

| Level | Unit |
|---|---|
| Company | Globe Telecom (Philippines) |
| Group | **The Data Group (TDG)** |
| Division | **Data Assurance and Quality Operations (DAQO)** |
| Team | **Data Quality Operations (DQ Ops)** |

- Since **2026-06-30**. Before that the team was **Analytics Governance (AG)** under EDS (Enterprise Data and Strategic Services). Older documents say "AG". The framework keeps its legacy name, the **EDS-AG DQ Framework**.
- DAQO has two functions: **Data Assurance** (enables trusted adoption of priority data, KPI, report, pipeline and AI assets; runs release gates) and **Data Quality Operations** (us).
- DQ Ops charter line, from the org chart: *sustain enterprise trust in priority data flows: monitor quality, freshness, ownership, BSLAs, anomalies and recurring defects before they hit customers, KPIs, reports or AI.*

### Team shape (roles only)

- A small team: one head, two DQ Ops Managers (one joined June 2026, one September 2026), two Lead Experts. One expert (Bronze layer and CHM lead) left in mid-2026. That initiative is paused and Bronze BAU coverage is thin.
- Lead Experts are split by layer and initiative: one is the Gold-layer subject-matter expert and leads OPx; the other is the Silver anomaly-detection expert and leads SRG. One manager's mandate is to scale and optimize checks, especially anomaly detection.
- The team runs mostly BAU (business as usual: daily alert triage, incident follow-up, reporting) and is trying to shift toward strategic work. Low-maintenance tooling matters.
- The team holds 5 Claude Code seats (June to December 2026), used to author rules and interpret results. It does not use them to run checks against live data.

### The seven TDG divisions (for counterparty context)

| Division | Relevant functions |
|---|---|
| Data Governance & Management (DGM) | Data Policy & Governance (**DPG**); Data Foundation & Management (**DFM**, incl. Data Quality Design) |
| **Data Assurance & Quality Operations (DAQO)** | Data Assurance; **DQ Ops** |
| Data Architecture & Design | Solutions architecture, data architecture, discovery and analysis |
| Data Platform Management & Engineering | Platform management and engineering. **Holds the budget.** |
| Data Pipeline Engineering | Ingestion and integration, big data testing, solutions delivery |
| Master & Reference Data Services (MRD) | MR data and software engineering |
| Transformation Office (TMO, formerly DTO) | Portfolio orchestration, change management, demand intake and prioritization |

---

## 2. Counterparties: who owns what

DQ Ops is **downstream of nearly every input**. A governance tracker line puts it this way: *"if no DQ builds from Pipeline or DQ rules from DFM, then nothing to monitor."*

| Counterparty | Inside TDG? | What they own, relative to us |
|---|---|---|
| **DGM / DFM** | Yes | **Authors DQ rules** (DQ policy, business rules, metrics) and owns the source side overall. Owns the Enterprise DQ Standard and the future Data Contracts Standard. |
| **DPG** (under DGM) | Yes | Writes the **DQ policy stack** we assure against. **Names data owners.** DQ Ops raises owner-accountability demands but never names owners itself. |
| **Data Pipeline Engineering** | Yes | **Implements DQ rules as code** in pipelines. Endorses rules to DQ Ops at the BAU phase. Owns the data contract framework in practice. |
| **Platform team** | Yes | **Budget.** It funds in-year if the case warrants it, otherwise the spend goes into next year's cycle. As of August 2026 nobody had been identified who can provision a GCP project. |
| **TMO** | Yes | Demand intake framework, capacity model, prioritization framework. |
| **MRD / MRDE** | Yes | Master and reference data pipelines. Its BigQuery reference-data pipeline is our proposed pilot for contract-embedded DQ. |
| **Data Operations** | No | **First line of defense.** Runs 24/7 pipeline monitoring and SLA-based incident triage. Owns the **Report SLA Matrix (BSLA)**, which sits in a managed-services vendor contract, so its dimension definitions are contractual. |
| **Managed Services** (vendor) | No | Triages alerts, raises tickets, writes RCAs. |
| **EDS Data Management** | No | First-line producer. **Absorbs the Gold layer** (Gold transfers out of our governance, see section 6). |
| **EDS CDA** (Customer Data & Analytics) | No | The analytics community we serve. Primary consumer of the Gold layer. |
| **NTG** (Network Technical Group) | No | Network inventory data in IBM Maximo, with its own rulebook and steward. DQ for NTG domains is now a DQ Ops BAU stream. |

Working rule: **DQ Ops specifies, others implement.** DQ Ops owns the check specification and traceability, not the build or the fix. Remediation, pipeline reruns, backfills, deep RCA, KPI-definition changes and source-system fixes all belong to the first line.

---

## 3. The 2026 DQ strategy

### The pivot: reactive to proactive

| | 2025 (built) | 2026 (building) |
|---|---|---|
| Posture | Reactive: detect and fix | **Proactive: prioritize and prevent at source** |
| Audit output | Findings and alerts | A **business-value-ranked "gate at source first" list** |
| Measured by | Coverage, DQI, MTTD | **Prevented Impact** (revenue protected + analyst time saved) |
| DQ Ops role | Run checks, surface issues | **Control tower**: set the standard, audit, flag and recommend, hold producers accountable |
| Engine | Hand-written Snowflake stored procedures | **DQ-as-code**: check registry, dbt, GitLab CI/CD |

Thesis line: *"Data quality must start at the source, not be retrofitted at the Data Lake."* Targeting principle: **report-specific CDE monitoring**, which means focusing on the Critical Data Elements behind the reports analysts actually use instead of policing every data point.

### 2025 foundation (what exists)

- Automated checks and anomaly detection across Bronze, Silver and Gold.
- The **Data Quality Index (DQI)**.
- The **TDG Data Quality Monitor**, an executive dashboard on a zero-cost Google Sheets + Apps Script stack, with an LLM narrative layer.
- Recorded baseline: about 95% coverage of about 128 brand KPIs and about 98% average DQI.

### DQR umbrella and the three initiatives

The **Data Quality Registry (DQR)** is the governance hub, housed in the Monitor. It contains three FY2026 initiatives, each of which feeds one factor of the Gating Priority Index (section 4).

| Initiative | What it is | Feeds | Status (as recorded) |
|---|---|---|---|
| **CHM**: CDE Health Monitor | Analyst self-service portal for CDE health and certification (RAG, Trust Score) | Business Impact, part of Risk | Q1 scoping done. Portal and CDE register not built. **Paused** after the lead left. |
| **OPx**: Operational Excellence | Proves the pivot with numbers: Proactive Detection Ratio (PDR), Time-to-Insight, Prevented Impact. Mandatory gap analysis on every ServiceNow ticket, classified Detection / Process / Ownership. | Urgency, outcome loop | Incident registry tagging is live and continuous. PDR baseline and Prevented-Impact model not built. |
| **SRG**: Systemic Remediation Governance | Governed change-request intake, risk scoring, Definition of Done, RACI | Routing and the scoring model | Q1 framework, RACI and risk model complete. DoD flow and SLA dashboard in progress. SRG DoD completion is about 23%. |

Timeline: Q1 build and align, Q2 launch and detect, Q3 scale and automate, Q4 institutionalize.

---

## 4. The canonical 2026 plan: targets T1 to T9

Source: the canonical plan v1.0 (2026-06-09). The T-IDs are internal workstream references.

| # | Workstream | Delivers | Horizon |
|---|---|---|---|
| T1 | Registry and authoring | Refactor the check registry (481 column-level checks: 194 Silver verbatim, 287 Gold needing review). Add owner, detection pattern and lineage fields. | H1 |
| T2 | dbt engine | 6 generic tests plus a registry-to-tests generator. dbt-duckdb on exported files (zero credentials) **and** approved dbt-on-Snowflake. | H1 |
| T3 | Rule-governance CI gate | warn/error severity bands, a GitLab CI/CD quality gate, regression fixtures, MR-diff review (Recce). Change control on **our own** registry. | H1 |
| T4 | Results-readout history store | An immutable, pullable record per check run: measured value, good/total counts, severity, timestamp. **This is the governance audit trail.** | H1 |
| T5 | Scale anomaly detection | Python AD on KPIs. Elementary OSS optional. Median+MAD stays in Snowflake ML. | H1 to H2 |
| T6 | Block / quarantine before impact | DQ Ops **recommends** staging-gate and quarantine criteria. Producers implement. | H2, cross-team |
| T7 | Validate-at-use + data contracts | ODCS v3.1.0 Definition of Done as a recommendation artifact routed via SRG | H2, cross-team |
| T8 | Durable ownership | Turnover-proof RACI extended to TDG actors, funded stewardship, runbooks | H2, continuous |
| T9 | **Proactive Gating Prioritization** | The **Gating Priority Index (GPI)**: one score per source table | H1 v0, H2 full |

**GPI formula:** `Business Impact × Urgency × Risk`, each banded 1 to 5, with Urgency floored at 1 so a high-value source that has never failed still ranks. It reuses SRG's change-request scoring so the two can never disagree. The top N sources go to producers each month via SRG as an ODCS Definition of Done. Risk (blast radius) relies on a **makeshift lineage**: a curated source-to-KPI map, because DQ Ops has no real lineage tooling.

**Maturity:** DQ Ops's own capability benchmarks at **Level 4 (Measured)** on a 6-level, 8-dimension vendor-neutral model. Enterprise target is Level 5, which DQ Ops drives but does not implement. The weakest dimensions are Enforcement, Automation and Consumption.

**Plan success metrics:** Prevented Impact (not yet quantifiable); PDR target 90% Golden KPI / 75% non-Golden; DQI sustained with policy aspiration **Silver 98.7% / Gold 99.7%**; GPI top-N reviewed monthly.

**Honest readiness:** the CDE register, the PDR baseline and a quantified Prevented Impact are not built. Plans written after mid-2026 (section 7) shifted emphasis toward GCP/BigQuery and source-side rules, so T2's Snowflake targets now carry an expiry date.

---

## 5. Scorecards we are graded on

There are two target sets, and **all targets count** toward the 2026 record:

### A. PPE 2026 (set by EDS/CDA, pre-move)

Tiered goals (Meets / Partially Exceeds / Exceeds):
- DQI at policy aspiration (Silver 98.7 / Gold 99.7).
- Share of Golden KPIs above aspiration.
- Better proactive detection and resolution than 2025.
- Automated DQI beyond Golden KPIs across all three layers.
- Remediate sources lacking control files.
- End-to-end DQ certification ("DQ4A", not yet tracked).

Report-level DQ targets were **retired** at the move.

### B. DAQO OGSM (TDG, from the move onward): the current BSC basis

| Row | Measure (2H 2026 target) | Later targets | Owner |
|---|---|---|---|
| **R4** Proactive DQ assurance | 50% of priority datasets covered + baselines (PDR, control effectiveness, monthly score distribution) | 1H27: 75% covered, ≥75% caught before business escalation. 2H27: 100% / ≥90%. | Officially all of TDG. In practice DQ Ops owns it. |
| **R5** Issue accountability, RCA, systemic prevention | One deviation register. 80% of priority-dataset issues carry a named owner + expected resolution date. Baselines for ageing, closure, repeat rate, cost of failure. | 1H27: 90%, ≥20% fewer repeats. 2H27: 100%, ≥30%. | DAQO (owner naming depends on DPG) |
| **R6** Network QUAD | Broadband, wireless, monthly assessment, Network dashboard | No measure yet | |
| **R7** DAQO shared automation | Baseline effort hours | 1H27: ≥30% hours released, ≥50% automated | Shared |

Notes:
- **"Priority datasets" is undefined.** Every R4 and R5 percentage waits on that definition. Interim scope is the LEAP Wave 0/1/2 list, marked provisional.
- **Credit rule:** an issue raised and routed to the right owner, with evidence, counts as DQ Ops delivery. Another team's closure rate does not.
- An earlier TDG BSC 2H 2026 rubric targeted 90% adherence to data standards, 100% of priority datasets assessed monthly, ≥95% of scores reported to owners, and tracking issues against SLA. It is superseded for scoring but shows the direction of travel.

---

## 6. LEAP: the GCP migration and our Workstream 3 seat

**LEAP** (Data Lakehouse Enabled AI Platform) is a 15-month "strangler fig" migration of about **3,585 TB** from on-prem plus AWS (Databricks, Snowflake) to a federated data mesh on **Google Cloud / BigQuery**.

| Phase | Months | DQ-relevant scope |
|---|---|---|
| 0–1 | 1–2 | Landing zone, GCS Bronze dual-write |
| 2 | 3–4 | Gold migrates first (~400 TB, ~14k tables). **Snowflake Gold decommissioned after parallel-run verification.** |
| 3 | 5–8 | Silver (~685 TB). Databricks to Dataproc. **dbt-bigquery adapter.** |
| 4 | 9–11 | Historical raw S3 to GCS. **Dataplex AutoDQ named as phase tooling.** |
| 5 | 12–15 | Decommission Snowflake and Databricks |

- **Litmus test "Trustworthiness & Quality"** (Guiding Principle 6, "Quality at the Source", owned by our division): every migrated dataset must enforce automated DQ checks, column-level lineage and real-time observability before cutover. Named GCP enforcement is Dataplex AutoDQ, Knowledge Catalog lineage and bidirectional DataHub sync. Legacy and BigQuery Gold must reconcile **byte for byte**.
- **WS3, "Trustworthy Data & Governance"**, is our seat. The only work package DAQO owns is **WS3-WP9, DG DQ Scorecard**: *"a board-level DQ posture and policy compliance report… independent, continuous, auditable assurance of data quality health"*. Adoption is due October 2026 and the WP target is 2027-01-31. WP4 (DQ Framework), WP3 (catalog/lineage) and WP11 (data contracts) sit with DGM.
- **October 2026 obligations:** build and test the DQ dashboard. Validate that Wave 0 DQ rules were actually implemented. (DQ rules for Waves 0/1/2 were reassigned to DFM on 2026-08-11.)
- **Ongoing obligations:** track vendor/partner SLAs and fallout-ticket SLAs; take part in wave assessments and governance readiness gates.
- **Numbers floated in LEAP:**
  - 98% completeness and 98% validity.
  - 100% of pilot workloads pass the Governance Readiness Gate.
  - "Minimum viable DQ" of ±1% KPI variance. A standard that shifts a business KPI more than 5% is wrong.
  - No baselines exist yet for DQ Index, PDR, time to detect, time to resolve or rule coverage.
- **Google gap log (~90 items, most "not evidenced")** against Dataplex / Knowledge Catalog. Google lacks:
  - An issue-management lifecycle.
  - **Reconciliation** of any kind.
  - Rule approval workflow and rule versioning or promotion.
  - A weighted, composite DQ Index.
  - Configurable anomaly detection (lookback, seasonality, false-positive feedback).
  - Quarantine and repair approvals.
  - Agentic DQ.

  The proposed partner stack also leans on Databricks Labs DQX, Great Expectations, Soda and DataHub, with component accountability undefined.
- **LEAP DQ Requirements track** (kickoff 2026-09-01): configurable DQ checks are **mandated inline in the pipeline** during ingestion and transformation. Control-file reconciliation becomes framework scope. A pilot launches in October. Our consolidated requirements are **76 user stories across 16 epics**. The enterprise WBS has 104 rows, and **20 of 45 Critical rows have no settled owner**. The LEAP master WBS had **no DQ lane** as of September 2026. "DQ Portal" appears only as an undated Feature Set 2 item for December 2026.
- **Layer scope, PROVISIONAL:** TDG governs to **Silver**. Gold transfers to Data Management (internally decided 2026-08-11, not ratified at Mancom level).

---

## 7. Platform direction from the TDG head (2026-08-11)

- Systems of record are **ServiceNow** for incidents, **GCP** as the monitoring project, **BigQuery** for all data, and visualizations on GCP. The current Sheets/Apps Script build is "a very good proof of concept and interim state". Pursue the correct systems and secure funding in advance.
- **Clairvoyance** (a Grafana-based tool) is the **enterprise-mandated observability and alerting tool for data**. It **auto-raises ServiceNow incidents**. DQ Ops can author its alert rules. Whether rules can be provisioned as code was still open.
- Target architecture is two surfaces on one BigQuery semantic layer: Clairvoyance for alarms, the DQ Monitor for audit, governance, RCA/verdicts, scorecard and evidence. Looker Studio is dropped.
- **Lens shift from consumption to source of record.** Quality requirements move to source partners before ingestion. Rules execute on pipelines. DQ Ops owns monitoring and **audit**.
- Status as at early October 2026: BigQuery is live in a **sandbox** (refresh every 2 hours). Monitor v2 is routed to it but not deployed. Production is expected to ride LEAP.
- **Elastic** (Bronze anomaly detection) was **decommissioned in October 2026**. There is no Bronze anomaly flagging until a replacement exists. The replacement is expected to come through LEAP (AD on ingest).

---

## 8. Governance policy stack (ALL DRAFT, NOT RATIFIED)

Owned by DPG under DGM. All four documents have blank approvals and signatures. Read them as **the direction and the field-level shape**, not as settled thresholds.

| Doc | What it sets |
|---|---|
| **A. Data Quality Policy** | 8 dimensions with per-layer baselines. Records-based pass rate: `records passed ÷ eligible records tested`. Promotion gates (Source→Bronze→Silver→Gold→Published). Four failure treatments (Block, Quarantine, Publish with Warning, Log & Remediate). Alerts must emit asset, dimension, rule, timestamp, **measured vs expected**, severity, and **auto-create ServiceNow tickets**. 5-stage incident lifecycle. Immutable audit logs kept at least 1 year. Exceptions route through DAQO. |
| **B. Enterprise DQ Standard v2** | Still an **unlocalized vendor template**. Requirements DQ-01 to DQ-13: CDE identification, ownership, rule metadata, testability, traceability, change control, consumer transparency. 28-field rule spec (eligible population, numerator, denominator, failure treatment, evidence location, rule version). 4 control tiers. Rule→CDE→product score roll-up. Critical failures must stay visible. **No score when a test did not run.** |
| **C. DQ Framework** (created 2026-09-14) | Hub-and-spoke: TDG runs the **central observability plane** and audits; domain spokes embed tests in CI/CD. 3 tiers: Tier 1 = mandatory automated pipeline blocking; Tier 2 = alert, fix within 48 business hours; Tier 3 = passive drift and volume monitoring. Execution standards: pipeline-native testing (Airflow, dbt), automated data contracts, ML anomaly detection on 90/180-day baselines. Metrics: pass rate ≥97% (Gold reconciliation 99.7%), 0 unhandled critical breaches, MTTR Sev1 <4h / Sev2 <48h, 100% CDE coverage for Tier 1. |
| **D. Data Contracts Policy** | Contracts are mandatory at operational→analytical ingress, Bronze→Silver, cross-domain interfaces and regulatory outputs. Six mandatory components, including DQ thresholds as automated assertions and SLAs. SemVer with a 60-day deprecation notice for breaking changes. CI/CD schema-registry blocking. Runtime circuit breakers with a dead-letter queue. **DAQO runs quarterly contract-compliance audits**, approves DLQ extensions and 24h RCA waivers. Audit logs kept at least 3 years. |

**RACI reading for C's metrics** (the DQ Ops head, 2026-10-07): Data Operations is **Responsible** for pass rate, critical breaches and MTTR. DAQO is **Accountable** (monitoring). DGM is Responsible for CDE coverage.

**Known contradictions inside the stack:**
- Dimension counts differ (8 vs 7).
- Tier models differ (4 vs 3).
- Log retention differs (1 vs 3 years).
- Three different pass-rate thresholds.
- "Acknowledge" and "resolve" are used interchangeably for the 4h target.

Practical rule: **build to the shape, keep thresholds as configuration.**

---

## 9. The EDS-AG DQ Framework (our current operating standard)

**Four pillars:** Discovery & Cataloging → DQ Check Implementation → DQ Scoring / DQI → DataOps Monitoring.

**Four dimensions:**
- **Timeliness**
- **Completeness**
- **Accuracy:** trends within expected behavioural thresholds
- **Validity & Consistency**

**Medallion estate:**

| Layer | Platform | Monitoring |
|---|---|---|
| Bronze | AWS S3 | Was Elastic AD (now off) |
| Silver (FDM) | Snowflake | DQ Ops + DataOps |
| Gold (CDAP) | Snowflake | DQ Ops + DataOps |

Our check reach today is **Snowflake only**.

| Dimension | Bronze | Silver | Gold |
|---|---|---|---|
| Timeliness | vs source OLA | ≤1 hr vs average SLA | 100%, zero tolerance |
| Completeness | Control-file count vs loaded **99.7%**; AD vs 90-day baseline | AD on record counts vs 180-day trend | ≥97% non-null on critical attributes |
| Accuracy | AD on volumes vs 90-day baseline | AD on KPIs vs 180-day trend | Reconciliation to Silver within **99.7%** |
| Validity & Consistency | AD on schema-error counts | Null/blank, invalid, duplicate each ≤3% → >97% | ≥97% pass |

- Anomaly flag threshold: score ≥75.
- Checks are **binary pass/fail** today. Moving to warn/error bands is a Level 4→5 step.
- Silver checks evaluate incremental (new) records.

**Registry reality:** 481 rule checks. 394 of the 481 only test **presence** (null/blank or populated). **0 of 481 test magnitude or plausibility.** For example, a broadband usage defect with absurd volumes passed every check because the values were not blank. Anomaly detection is the only accuracy surface. It runs as three engines: Snowflake ML on Silver KPIs, our own Median+MAD model on a key report, and formerly Elastic on Bronze.

---

## 10. Competing dimension sets across the enterprise

Every score has to declare which set it uses. Choosing the standard is **DGM's decision, not ours**: separation of duties keeps DAQO out of standard definition.

| Set | Origin | Count | Dimensions |
|---|---|---|---|
| **EDS-AG** | Our framework (pre-move, analytics estate) | 4 | Timeliness, Completeness, Accuracy, Validity & Consistency |
| **NTG ACCVU** | NTG DQ Framework v1.0 (network / Maximo) | 5 | Accuracy, Completeness, Consistency, Validity, Uniqueness (rulebook: 227 rules; Completeness/Validity 0.98, Accuracy/Consistency 0.95, Uniqueness 1.0) |
| **DGIM / DPG Policy** | Broadband RI v4.0, now carried into draft Policy A | 8 | Accuracy, Completeness, Consistency, Timeliness, Validity, Uniqueness, Accessibility, Integrity (self-declared provisional) |

Two further live sets: the draft Enterprise Standard (7, without Accessibility) and our own ODCS-based contract spec (7, adds Coverage). Our recommendation is ODCS 7 with a crosswalk. The authors of the NTG and DGIM frameworks now sit inside TDG, so harmonizing the sets is an internal decision.

---

## 11. Operating model

- **Second-line assurance ("control tower").** DQ Ops audits, flags, recommends and measures. Producers (first line) implement and enforce. *Audit, not embed.*
- **Target model**, written into the LEAP risk register: DFM defines rules → Pipeline/Platform Engineering implements them as code → DQ Ops validates and monitors results.
- **Separation of duties:** assurance excludes technical solution design and standard definition.
- **Mandate is audit-led.** Every source describes DQ Ops as doing more than "monitoring". Audit needs things monitoring does not:
  1. Independent recomputation from raw rule-run and fallout detail.
  2. Rule provenance (versioned rules).
  3. A **control coverage register**: a source with no rules emits no failures and looks perfect.
  4. Chain of custody through correlation IDs.
  5. A **remediation ledger** with a clock.
  6. **Recurrence detection**: did the fix hold?
- **Two intake types.** Type 1: pipeline teams pass down in-pipe DQ builds. Type 2: CDA-raised needs that DQ Ops builds under a time-boxed exception because pipeline builds are too slow.
- **InfoSec invariant (hard).** AI authors rules and interprets results. Engines execute. No AI against live production databases. No third-party native app opening its own production-DB connection without lengthy security approval (this is why DQOps over live JDBC was gated). Approved data path: **Snowflake → GCP folder → Google Sheet → CSV**. dbt is approved on files (duckdb) and on Snowflake.

---

## 12. Key metrics and their caveats

| Metric | Definition / status | Caveat |
|---|---|---|
| **DQI** | Cumulative monthly score per layer × dimension, rolled to KPI, built from binary check pass/fails. YTD view tabs Silver and Gold. Bronze excluded. | Silver has been below its 98.7% aspiration most of 2026: a broadband fixed-line usage source problem pulled Usage completeness from 98 to about 63–67 from April. Gold's near-100% is partly survivorship. The draft policy wants a **records-based** pass rate, but ours is check-based. |
| **Enterprise DQ Scorecard** (v3, Sept 2026) | Subscriber, Reload, Usage areas + Network. YTD composite about 94.6%. Network shows "No score", never zero. | Earlier versions were vetoed as too technical. It is being rebuilt as **the** DQ Scorecard, benchmarked to the draft TDG Framework. No enterprise threshold is ratified; 97% is quoted as provisional. |
| **MTTR** | Must **name its population** (incident, alert or DQC issue) on every line. Incident basis 2025: 16.0 days mean (median 6). 2026 Jan–Sep: 6.3 days mean (median 1). | The gain is mostly a mix effect. Check-caught incidents resolve in about 1 day median; user-raised in about 6–7 days. "15 → 3 days" is **retired**. |
| **PDR** (proactive detection) | Only same-definition baseline is Nov–Dec 2025: about 73% all-KPI (Golden about 96%). 2026 YTD: about 77% all-KPI. | Older 82.5% / 80% baselines have no source and are retired. |
| **Verdict taxonomy** | Each alert-ticket is tagged True Incident, Valid Expected Behaviour, Timeliness Issue, False Alert (proven noise only) or Inconclusive. Detection rate = 1 − provable false alerts, about 92–95% after the September 2026 final model. User-raised non-DQ issues sit outside DQ counts. | The categorizer re-tags the whole history each sweep, so months restate. |
| **RCA Substantiation Rate** | Share of resolved alerts whose **resolution notes** carry evidence: **53.3%** baseline (Gold 67%, Silver 59%, Bronze 24%). | Reported beside detection, never folded in. No RAG target yet. |
| **Engine yield** (Jan–Aug 2026) | Alerts per confirmed incident: rule checks 2.3:1; Snowflake AD 15:1; Elastic 48:1. | The engines are similar on accuracy but differ on **actionability**. |
| **Operational RAG guardrails** | Team-set Monitor tile bands, e.g. resolution ≥97%, average resolution ≤3 days, PDR ≥84%. | Guardrails only, not framework policy. |

---

## 13. What this means for tool selection

1. **The buyer is a second-line assurance team, not a pipeline team.** Tools must support **independent audit**: ingest raw rule-run and fallout detail, recompute scores, version rules, keep immutable evidence (1–3 years), and track coverage and remediation. Pure "observability" or pipeline-embedded test frameworks cover only part of the mandate.
2. **Rules are authored by DFM and executed by Pipeline Engineering.** A tool that assumes the DQ team writes and runs every check misfits. Favor tools that consume a shared spec (contracts, YAML) and report results from wherever checks run.
3. **The landing zone is GCP / BigQuery, via LEAP.** Snowflake has a decommission date. Dataplex AutoDQ, Knowledge Catalog and DataHub are already named in LEAP. Recommendations should say what they add on top of Dataplex. The Google gap log is the checklist: reconciliation, issue lifecycle, rule approval and versioning, a weighted composite index, configurable anomaly detection, quarantine workflow.
4. **Fixed enterprise anchors:** ServiceNow (incident system of record, auto-raised tickets), Clairvoyance/Grafana (mandated alerting), dbt (approved on Snowflake and on files, with dbt-bigquery planned), GitLab CI/CD (approved git). Tools must integrate with these, not replace them.
5. **Strong InfoSec constraint.** Any third-party tool opening its own production-DB connection faces a long security approval. Tools that run inside the warehouse, inside dbt, or on exported files clear fastest.
6. **Specific capability gaps to fill:**
   - Bronze anomaly detection (Elastic is gone, nothing has replaced it yet).
   - **Magnitude and plausibility rules** (0 of 481 today).
   - Reconciliation (source vs target, KPI tolerance).
   - A control coverage register.
   - A remediation ledger with SLA clocks.
   - Recurrence detection.
   - Alert deduplication and correlation **before** ServiceNow tickets are raised.
   - Master and reference data checks (uniqueness, referential integrity, code sets).
   - Data contract validation (ODCS).
   - Lineage for blast-radius ranking.
7. **Configurable thresholds and dimensions.** The policy stack is DRAFT, with conflicting dimension sets, tiers and thresholds. Tools must treat dimensions, tiers, severity bands and treatments (block / quarantine / warn / log) as configuration and support a crosswalk.
8. **Estate is broader than the lake.** Network data lives in IBM Maximo and DataCube with its own rulebook. A single-warehouse tool will not cover it.
9. **Small, BAU-heavy team, with budget held by another division.** Prefer low-maintenance, managed or dbt-native options. Avoid tools that need one ML job per dataset to tune. Funding asks must state in-year vs next-cycle.
10. **Timing:** LEAP pilot and production releases land in October 2026. The DQ Portal is a December 2026 feature set. WP9 runs to January 2027. 2026 is the foundation year and 2027 the cutover.
