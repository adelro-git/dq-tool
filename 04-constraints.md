# 04 · Constraints a DQ tool recommendation must respect

*Context file for the DQ tools deep-research brief. As of 2026-10-18. Audience: the head of Data Quality Operations (DQ Ops) at Globe's The Data Group (TDG), and the research AI reading this file.*

**How to read this.** Every item below is a real constraint at Globe, drawn from internal program sources. Each one states what it is, where it comes from, and what it means for a candidate tool. Items marked **(inferred)** are our reading, not a stated rule. A checklist table at the end separates hard constraints (a tool that breaks one is out) from soft ones (a tool that breaks one needs a strong case).

**One-paragraph summary.** DQ Ops is a second-line, audit-led function. It specifies checks, it does not build them into producer pipelines. AI may author and interpret, but engines execute, and AI never runs against live production databases. Third-party apps that open their own connection to a production database face a long security approval. The estate is moving from Snowflake and AWS to Google Cloud (GCP) and BigQuery under the LEAP program over roughly 15 months, so anything Snowflake-only has an expiry date. Budget is tight and sits with another team. The policy stack that would set thresholds is still draft. The last purpose-bought detection tool (Elastic anomaly detection) was decommissioned because it could not be configured per dataset and produced unactionable alert volume.

---

## 1. Security and InfoSec

**1.1 AI authors and interprets; engines execute. AI never touches live production databases.**
- *What:* The workspace's non-negotiable InfoSec invariant. Claude or any AI may draft checks, contracts and analysis. Execution happens in cloud-native engines. Check design and dbt work happen on dummy schemas. Live data is never pasted into a prompt.
- *Source:* CONTRIBUTING.md (InfoSec invariant); memory note on the InfoSec data path.
- *Implication:* An "AI agent that connects to your warehouse and fixes data" is out as described. AI features are acceptable where the AI generates rules or explains results and a governed engine runs them. Any AI feature that queries production data directly needs a separate InfoSec case.

**1.2 Third-party native apps that open their own production DB connection need a lengthy security approval.**
- *What:* The real line is "dbt versus third-party native apps", not "files versus warehouse". dbt connecting to Snowflake is already approved (confirmed 2026-06-06). A third-party tool that opens its own JDBC or similar connection (DQOps was the worked example) still goes through long security review. Even a tightly scoped, read-only Snowflake MCP connection was judged to sit in the same approval bucket and was parked.
- *Source:* memory (InfoSec data path); research reconciliation decision D1; Plan B benchmark.
- *Implication:* Prefer tools that run **in-warehouse or by pushdown through an already-approved path** (dbt, warehouse-native features such as Snowflake Data Metric Functions or BigQuery-native checks). A tool that needs its own live connection should be scored "requires InfoSec approval, revisit", not rejected outright, but expect months of delay.

**1.3 SaaS that pulls Restricted data out to a vendor environment is likely not viable.**
- *What:* When the serving layer was reviewed, external SaaS was ruled out because it would put Restricted data on an unapproved third party. Datadog was studied for patterns only, because a commercial SaaS on live connections fails the invariant the same way live JDBC does.
- *Source:* research-and-benchmarks.md §3 and §7.
- *Implication:* Data-plane SaaS (vendor-hosted, ingests rows or samples) is likely blocked. Look for self-hosted, in-customer-cloud, or metadata-only deployment modes and say which one the recommendation assumes.

**1.4 The approved zero-approval substrate is files: Snowflake → GCP storage → Google Sheet → CSV.**
- *What:* The files-first export path is the zero-approval, zero-credential option. The current DQ Monitor runs on it. dbt-duckdb on exported files in CI was verified to need no database credentials.
- *Source:* memory (InfoSec data path); Plan B benchmark.
- *Implication:* Tools that can run file-only (on CSV or Parquet exports) get a fast, no-approval proof path. This matters for pilots. Note the direction of travel in §4: files are the proof path, BigQuery is the target store.

**1.5 Source sheets are Restricted; the domain-gated web app is the only sanctioned data path.**
- *What:* The DQ Monitor's source Google Sheets are Restricted. The only sanctioned read path is the Apps Script web app, gated to the Globe domain. Connector reads of production DQ sheets (for example via a Drive connector) are ad-hoc inspection only, never a pipeline source.
- *Source:* CLAUDE.md; KNOWLEDGE-BASE.md; memory (Drive extraction reference).
- *Implication:* A tool must not be designed to pull from these sheets directly. Its outputs should be Globe-owned and accessible only internally (no public links).

**1.6 No secrets in tools, prompts, or repos.**
- *What:* Real warehouse credentials live only in each user's git-ignored profile. Keys, tokens and passwords are never pasted into an AI session. A personal laptop is not an acceptable runner for Restricted production data; use a governed runner.
- *Source:* CONTRIBUTING.md (never-commit list); memory (Claude MTS note, runner caveat).
- *Implication:* Prefer tools that use federated or workload identity (for example service-account application default credentials on GCP, no keys). Tools that need long-lived keys stored in config are a negative.

**1.7 Per-person team data must not land where it is widely visible.**
- *What:* BigQuery datasets are visible across TDG to project viewers, and dataset ACLs cannot override project IAM. Per-person allocation and workload data therefore stays out of BigQuery.
- *Source:* memory (BQ is system of record; BQ sandbox note).
- *Implication:* A tool that writes user-level or people data into the shared warehouse needs its own isolation answer.

---

## 2. Budget and procurement

**2.1 Budget-constrained, with a ₱0 history.**
- *What:* The 2026 DQ Monitor was built at zero vendor cost on Google Sheets and Apps Script. That remains the right story for what shipped, but the TDG head has said it is a proof of concept and interim state, and the target is GCP and BigQuery with funding procured in advance.
- *Source:* memory (showcase framing; GCP/BQ architecture mandate).
- *Implication:* A paid tool is not ruled out, but it must justify itself against an OSS or GCP-native alternative that costs little or nothing. The target-state BigQuery result store was estimated at a small monthly cloud cost; the note says "the spend is never the obstacle; the org clock is" (provisioning, integration users, onboarding all take weeks to months).

**2.2 Budget sits with another team, and asks route to one of two cycles.**
- *What:* Budget sits with the Platform team (a sibling division inside TDG). They look for immediate in-year budget only if the case warrants it; otherwise it moves to the next year's cycle. A funding submission must say which ask it is making, or it defaults to the easier (later) route.
- *Source:* memory (DQ Ops counterparties).
- *Implication:* A recommendation should state whether it needs in-year money or can wait for the next cycle, and should work as either. DQ Ops cannot buy a tool on its own authority.

**2.3 Spend goes through a Memo to Spend (MTS) with a multi-level approval chain.**
- *What:* Purchases go through an MTS in Globe's memo system, routed through division, IT finance and CIO-level approvals. AI tools go through the AI group's access process, which has seat tiers and expects an architecture diagram when a new automated pipeline is introduced. Waiver-of-competition and funding-source fields are owned by finance and proponents, not DQ Ops.
- *Source:* memory (Claude MTS; Elastic MTS renewal).
- *Implication:* Factor procurement lead time into any roadmap. Single-vendor or sole-source tools may need a waiver of competition (inferred from the MTS form fields).

**2.4 Procurement text must use plain operations language.**
- *What:* MTS justifications for IT, finance and CIO audiences must avoid DQ-internal jargon (medallion layer names, internal tool names, acronyms). Mirror the plain language in precedent documents ("Data Lake", "ingestion", named platforms).
- *Source:* memory (MTS procurement plain-language feedback).
- *Implication:* A research output that will feed a funding case should describe the tool's value in operational terms (what it checks, where, what it replaces), not in vendor or framework jargon.

**2.5 Vendor renewals carry history and conditions.**
- *What:* The Elastic Cloud renewal was approved the prior year on the condition that it be the last extension. TDG co-signed the next year's justification on a coverage argument (Elastic was the only anomaly detection on raw ingestion). Elastic was then decommissioned (decision 2026-09-25; Bronze coverage stopped October 2026; see §8).
- *Source:* memory (Elastic MTS renewal; Elastic AD discontinuation case).
- *Implication:* Leadership has seen a tool renewed on a "last year" promise. A new tool recommendation should carry a dated exit or replacement path and avoid open-ended lock-in.

**2.6 AI tooling cost is driven by usage patterns, not seat price alone.**
- *What:* Measured Claude Code cost on a long build was dominated by cache reads of large context, not output; estimates from output alone undercounted about tenfold.
- *Source:* memory (Claude Code cost model).
- *Implication:* For any AI-assisted DQ tool, ask how cost scales with context and run frequency, not just list price.

---

## 3. Ownership and operating model

**3.1 DQ Ops specifies; pipeline and platform owners implement.**
- *What:* DQ Ops is second line. It specifies the check (rule, threshold, report traceability). Pipeline and platform owners implement it. Producers remediate. As of 2026-08-11, DQ Ops also does not author DQ rules for the migration: the Data Fabric Management (DFM) department under Data Governance and Management (DGM) defines rules, Pipeline Engineering implements them, DQ Ops validates and monitors.
- *Source:* memory (specifies-not-builds; 90-day re-plan; LEAP WS3); research reconciliation D10.
- *Implication:* A tool whose value depends on DQ Ops embedding checks inside producer pipelines does not fit. Tools DQ Ops would own are audit-side: independent re-computation, coverage audit, scoring, evidence. Pipeline-embedded enforcement tools are a recommendation to DFM and Pipeline Engineering, not to DQ Ops.

**3.2 The mandate is audit-led.**
- *What:* The mandate (still being finalised) is to audit enterprise data quality, push remediation, and drive the Data Quality Index (DQI) up over time. Audit needs things monitoring does not: raw fallout and rule-run detail (not just scores), rule provenance with versions, a control coverage register, chain-of-custody keys, a remediation ledger with a clock, and recurrence detection.
- *Source:* memory (GCP/BQ architecture mandate).
- *Implication:* Score tools on whether they emit and retain row-level counts, rule versions and run IDs, not just dashboards and pass/fail. The remediation ledger "has no counterpart anywhere" today and is the clearest gap a tool could fill.

**3.3 Naming owners is DPG's job.**
- *What:* Named issue or dataset owners are for Data Policy and Governance (DPG) to identify. DQ Ops raises owner-accountability demands. A ServiceNow assignment group is never "the owner".
- *Source:* memory (owner naming is DPG's).
- *Implication:* A tool that auto-assigns or infers accountable owners must not present them as authoritative. It can show resolver teams and flag missing owners.

**3.4 Metric RACI: Data Operations is Responsible, DAQO is Accountable.**
- *What:* For pass rate, critical rule breaches and incident MTTR, Data Operations is Responsible (they action failures) and Data Assurance and Quality Operations (DAQO) is Accountable for monitoring. CDE governance coverage belongs to DGM.
- *Source:* memory (framework metric RACI).
- *Implication:* Workflow features (ticket routing, remediation actions) belong to Data Operations' tooling. DQ Ops needs read and oversight access, not the operational console.

**3.5 Remediation closes, but slowly and outside our control.**
- *What:* The current remediation loop is not open: defects are validated with the source by gravity, and fixes go through L3 support. It is manual, gravity-triaged and paced by parties outside data engineering, and defects can recur.
- *Source:* memory (EDS remediation is not open-loop).
- *Implication:* Tools claiming "automated remediation" or "self-healing" do not match how fixes happen here. Value lies in tracking the loop (ageing, recurrence), not closing it.

**3.6 Mandated enterprise tools already exist for some functions.**
- *What:* Clairvoyance (a Grafana-based tool) is the enterprise-mandated observability and alerting tool for data, and it auto-raises ServiceNow incidents. ServiceNow is the incident system of record. The DQ Monitor stays the audit, governance and exec-scorecard surface. Looker Studio was dropped for the scorecard.
- *Source:* memory (GCP/BQ architecture mandate).
- *Implication:* A tool that brings its own alerting or incident system competes with a mandate. Prefer tools that feed Clairvoyance and ServiceNow rather than replace them. Whether Clairvoyance supports alert rules as code is still unconfirmed.

**3.7 Agent platform: Genesis is being onboarded; agents must not compute numbers.**
- *What:* Genesis is the vendor agent platform TDG is onboarding through Data Platform Engineering, and it is provisioned for DQ Ops. The DQ Ops agent spec reads computed results read-only and returns analysis; it is never a second source of a number. Computation stays in version-controlled, parity-tested dbt. Genesis authenticates as individual users, not service accounts, so unattended runs are an open question.
- *Source:* memory (Genesis / Cerberus; Gems to Genesis).
- *Implication:* AI-native DQ tools overlap with an already-chosen agent platform. Recommend them only where they do something Genesis plus dbt cannot.

---

## 4. Architecture mandates and timing

**4.1 GCP and BigQuery are the target. ServiceNow is the incident system of record.**
- *What:* Direction from the TDG head (2026-08-11): pull from the correct systems of record. ServiceNow for incidents, GCP as the monitoring project, BigQuery for all data needs, visualisation on GCP.
- *Source:* memory (GCP/BQ architecture mandate).
- *Implication:* A recommended tool must run on, or natively integrate with, GCP and BigQuery. Snowflake-only tools are short-lived (see 4.4).

**4.2 BigQuery is the DQ result-and-evidence store, not a copy of the estate.**
- *What:* Checks execute where the data lives (pipelines for DFM rules, the warehouse for DQ Ops' second-line checks). Only outcomes land in BigQuery. Machine pulls land in BigQuery directly; Sheets survive only as human entry forms. Keep one base table per entity and derive the rest as views or marts.
- *Source:* memory (architecture mandate; BQ is system of record).
- *Implication:* Prefer tools that push down to the warehouse and can **write results (with counts, rule IDs, run IDs) into our BigQuery tables**. Tools that keep results only in their own proprietary store are a negative.

**4.3 LEAP sets the clock.**
- *What:* LEAP is Globe's 15-month migration to a data mesh on GCP; Day 1 was August 2026. DQ Ops sits in Workstream 3 (Trustworthy Data and Governance) and owns one work package, the DG DQ Scorecard (target 2027-01-31). Production GCP for DQ Ops arrives with LEAP (decided 2026-09-15); until then work runs in a sandbox. Planning stance: 2026 is foundation, 2027 is cutover.
- *Source:* memory (LEAP WS3; BQ sandbox); leap-program-ws3.md.
- *Implication:* A tool needing a new production environment before LEAP's landing zone exists will wait for LEAP. Tools that can be proven in a sandbox now and moved later fit best.

**4.4 Snowflake today, with an external expiry.**
- *What:* Silver and Gold layers run on Snowflake today. In LEAP's plan, Snowflake Gold is decommissioned around months 3 to 4 after parallel-run verification, and Snowflake plus Databricks fully around months 12 to 15. The note says to target dbt-bigquery from the start. Mapping months to calendar (inferred from Day 1 = August 2026): Gold around late 2026, full exit around mid to late 2027.
- *Source:* memory (architecture mandate); leap-program-ws3.md.
- *Implication:* Snowflake-native features (for example Data Metric Functions, which were piloted and not pursued) are a poor long-term bet. Cross-warehouse tools (dbt-based, or supporting both Snowflake and BigQuery) bridge the transition.

**4.5 LEAP has already named GCP tooling for parts of the problem.**
- *What:* LEAP names Dataplex AutoDQ checks, Knowledge Catalog column lineage and bidirectional DataHub sync for its DQ framework, with Dataplex AutoDQ for raw data in months 9 to 11. The Google compliance gap list (about 90 items, most "not evidenced") shows no reconciliation, no issue-management lifecycle, no rule-approval workflow and no weighted DQ index in the proposed stack. Databricks Labs DQX is in the proposed stack with no managed-product SLA. Dataplex AutoDQ is rule-based; the target stack has no statistical anomaly layer.
- *Source:* memory (LEAP WS3; Elastic case); leap-program-ws3.md.
- *Implication:* Do not recommend a tool that duplicates Dataplex rule execution. The gaps worth researching are the lifecycle around execution (reconciliation, issue lifecycle, rule approval, weighted index), statistical anomaly detection on ingested data, and coverage audit.

**4.6 Raw ingestion (Bronze) has no current detection tool.**
- *What:* Raw ingestion sits on AWS S3 with no Snowflake or dbt reach. Since Elastic's decommission there is no Bronze flagging from October 2026 until a new anomaly-detection tool exists. The stated replacement direction is anomaly detection as a platform service on ingest in LEAP (for example BigQuery ML on the GCS landing zone).
- *Source:* memory (Elastic AD discontinuation case; architecture mandate).
- *Implication:* This is the largest open gap and the strongest funding argument. A recommended tool must work on GCS or BigQuery raw landing data, not on S3 alone.

**4.7 Data contracts are the intended enforcement vehicle.**
- *What:* The draft Data Quality Policy says all DQ requirements are enforced through data contracts. The program adopted ODCS v3.1.0 as the format for DQ Ops' recommendations to producers (not as an engine DQ Ops runs). DQ Ops has a written contract quality section (fallout records, per-run quality logs, error-code matrix, incident integration) for DGM to adopt.
- *Source:* tdg-dpg-dq-policy-stack.md; research reconciliation D8; memory (architecture mandate).
- *Implication:* Prefer tools that read ODCS contracts or can be generated from them. Tools with a proprietary rule language and no contract import add translation cost.

---

## 5. Policy and governance

**5.1 The DQ policy stack is draft and unratified.**
- *What:* DPG's four DQ documents (Policy, Enterprise Standard, Framework, Data Contracts Policy) all carry blank approval dates and signatures. The Standard is still an unlocalised vendor template. Working rule: build against their shape (fields, lifecycle stages, promotion gates, failure treatments, escalation ladder), never their numbers. Keep every threshold as configuration.
- *Source:* tdg-dpg-dq-policy-stack.md; memory (policy stack reference).
- *Implication:* A tool must hold thresholds, tiers and SLAs as editable configuration, not hardcoded defaults. Draft numbers conflict: three different pass-rate figures across documents, and the Framework's severity-1 MTTR (resolution) contradicts the Standard's (acknowledgement).

**5.2 Multiple dimension sets are live at once.**
- *What:* The source notes **five** live dimension sets, not three: the Policy's 8 (with Accessibility), the Standard's 7, the existing EDS-AG framework's 4, ODCS's 7 (with Coverage), and a provisional Network set. Every published score must declare which set it uses.
- *Source:* tdg-dpg-dq-policy-stack.md §3 (contradiction 1).
- *Implication:* A tool must support custom or multiple dimension taxonomies per check. A fixed vendor dimension list is a soft negative.

**5.3 Other unresolved contradictions a tool will meet.**
- *What:* Control tiers (Standard has 4, Framework has 3). Audit log retention (Policy says 1 year, Contracts Policy says 3). The Policy puts operational logs and telemetry out of scope while the Framework and Contracts Policy depend on structured pipeline logs.
- *Source:* tdg-dpg-dq-policy-stack.md §3.
- *Implication:* Retention and tiering must be configurable. Plan for the longer (3-year) retention to be safe **(inferred)**.

**5.4 Measurement rules already written.**
- *What:* The draft Standard requires that duplicate counting of the same defect be disclosed and controlled, critical failures stay visible separately and are not hidden by aggregates, and no score is reported when a test did not run or the population cannot be established. The Network scorecard rule is "no score, never zero".
- *Source:* tdg-dpg-dq-policy-stack.md; MEMORY.md index (Enterprise DQ Scorecard).
- *Implication:* A tool must distinguish "not run" from "passed" and from "0%", and must allow severity-separated reporting.

**5.5 Obligations written onto DQ Ops without an instrument.**
- *What:* The drafts give DQ Ops quarterly contract-compliance audits, approval authority over dead-letter-queue clearance extensions and the 24-hour RCA waiver, and deviation management. No remediation SLA is ratified; do not assert one.
- *Source:* tdg-dpg-dq-policy-stack.md; memory (Genesis spec rules).
- *Implication:* A tool offering an approvals register with clock and audit trail would meet a real need. It must not ship with a built-in SLA presented as Globe policy.

**5.6 Assurance must not set the standard.**
- *What:* A LEAP risk item says assurance must exclude standard definition (verification only). DQ Ops hands requirements to DGM as auditor requirements, not as "our standard".
- *Source:* memory (architecture mandate).
- *Implication:* Tools for authoring the enterprise rule standard belong to DGM's evaluation, not DQ Ops'.

---

## 6. Capacity and skills (general terms)

**6.1 Small team, demand above supply.**
- *What:* DQ Ops is a small team. A demand-supply view prepared for the project management office in September 2026 showed recorded demand above available hours, with migration assurance work still unhoured.
- *Source:* memory (PMO demand intake).
- *Implication:* A tool that adds standing maintenance (per-dataset model tuning, many jobs to babysit) is a negative. Onboarding a new dataset should be a registration, not a modelling project (this is an explicit ask to LEAP).

**6.2 Skills fit: SQL, dbt, Python, Apps Script, AI-assisted authoring.**
- *What:* The team works in SQL and dbt, Python, Google Apps Script, and Claude Code (seat tier aimed at domain experts building low-code automations). The repo is GitLab; CI is the proposed no-cost orchestrator for the files path.
- *Source:* memory (Claude MTS); research-and-benchmarks.md §4.
- *Implication:* **(inferred)** Tools configured in YAML or SQL and versioned in git fit the team. Tools needing JVM administration, Spark tuning or a dedicated platform admin are a poor fit unless another team runs them.

**6.3 Rule coverage today is presence-heavy; plausibility is missing.**
- *What:* 82% of the 481 catalog rules test presence or population; none test an absolute range. Rolling-window anomaly detection missed an ~83x usage inflation for over 8 months because a persistent error becomes the new median.
- *Source:* memory (presence not plausibility).
- *Implication:* A tool's value is higher if it supports absolute-range checks and per-entity ratio checks alongside anomaly detection. A tool offering only rolling-window anomaly detection repeats a known failure mode.

---

## 7. Access and permission gaps

**7.1 Production integrations depend on access that takes weeks to months.**
- *What:* The org clock items named are a GCP cost centre, a ServiceNow integration user, and Clairvoyance onboarding. A direct ServiceNow connector does not exist yet; incidents go Sheet scrape then BigQuery load until it does.
- *Source:* memory (architecture mandate; BQ is system of record).
- *Implication:* A tool that needs a ServiceNow API integration on day one will wait.

**7.2 ServiceNow access can change underneath us.**
- *What:* An external ServiceNow permission change blocked the incident scraper; it was resolved on 2026-07-28. While blocked, incident status in the Monitor lagged the source.
- *Source:* memory (ServiceNow scraper permission block).
- *Implication:* Scraping-based integrations are fragile. Prefer tools that use sanctioned APIs or receive pushed data, and that show a visible "source stale" state.

**7.3 Google Workspace policy blocks Drive scope for some clients.**
- *What:* Workspace policy drops the Drive scope for the Cloud SDK OAuth client on token refresh, so command-line BigQuery cannot read Sheets-backed external tables; only the console or console-created scheduled queries can. Drive connector exports have a size cap and lossy text reads.
- *Source:* memory (BQ sandbox; Drive extraction reference).
- *Implication:* Tools that rely on reading Google Sheets through OAuth may hit the same block. Another reason to land data in BigQuery tables.

**7.4 Sandbox limits.**
- *What:* The GCP sandbox has BigQuery and Storage rights but lacks Cloud Run deploy roles and service-account impersonation; the Vertex AI API is not enabled; no service account is used while on the sandbox. Decision (2026-09-30): no Cloud Run and no role request in 2026.
- *Source:* memory (BQ sandbox; BQ is system of record).
- *Implication:* Tools needing their own container hosting, a service account, or Vertex AI cannot be piloted in the sandbox without new requests. BigQuery-native or SQL-only tools can.

**7.5 Snowflake to GCS export needs admin grants.**
- *What:* Moving data from the production Snowflake account to GCS needs a new storage integration from the account admin; the self-serve route in another account is missing two grants. That work is on hold.
- *Source:* memory (Snowflake accounts and GCS integrations).
- *Implication:* A tool that assumes easy Snowflake-to-GCP data movement before LEAP is optimistic.

**7.6 Per-user authentication on the agent platform.**
- *What:* Genesis maps OAuth to individual users, so an agent reaches only as far as the person running it, and unattended runs may need a person to start them. Google Workspace connectivity is unconfirmed.
- *Source:* memory (Genesis notes).
- *Implication:* Any AI tool that needs unattended service-account access to data will meet the same question.

---

## 8. Lessons from past tools

**8.1 Elastic anomaly detection (decommissioned: decision 2026-09-25, coverage off from October 2026).**
- *What happened:* Elastic was the only anomaly detection on raw ingestion. Measured over Jan to Aug 2026: its detection rate was statistically similar to Snowflake anomaly detection and rule checks (about 91 to 93%), so the problem was **not accuracy but yield**. Actionable rate was about 6% versus 12% for Snowflake anomaly detection and 60% for rule checks. Elastic produced 48 alerts per confirmed incident; rules produced about 2. In most cited cases another monitor had already ticketed the issue first.
- *Root causes (architectural):* one index fed one job with one threshold, so there was no per-source threshold to tune; evaluation interval was hard-coupled to notification interval (hourly re-alerts, no daily trigger, no suppression window); identical alert subject lines with no category, dimension or lineage, so triage needed opening every email; better precision needed one index and one job per dataset, which is not sustainable for a small team.
- *What could not be proved:* false negatives and the counterfactual were unmeasurable, so the defensible claim is "the configuration did not earn its keep", not "there is no need". 11 of the 13 sources Elastic watched are watched by nothing else.
- *Source:* memory (Elastic AD discontinuation case).
- *Implication for any candidate:* must support per-dataset thresholds from one templated definition; decouple evaluation from notification; offer suppression, dedup and correlation before raising a ticket; attach structured context (dataset, dimension, rule, measured versus expected) to every alert; and be judged on actionable yield, not detection rate alone. Plan a 60 to 90 day parallel bake-off on the same feed before replacing anything.

**8.2 Snowflake Data Metric Functions (piloted, not pursued).**
- *What:* Piloted and not utilised. Not presented as a capability.
- *Source:* memory (showcase framing).
- *Implication:* Warehouse-native DQ features are not automatically adopted; and with Snowflake's exit, this path is closed.

**8.3 DQOps (parked).**
- *What:* Parked because live JDBC is core to it (approval bucket) and dbt covers both the files proof and approved warehouse execution. License is BSL 1.1 (internal use permitted); running it locally on CSV exports was noted as a zero-approval benchmark only.
- *Source:* Plan B benchmark; research-and-benchmarks.md §1.
- *Implication:* Licensing and connection model are evaluated together. Check licenses: Soda Core v4 reportedly moved to Elastic License 2.0 (source-available, not OSI open source); Great Expectations OSS and dbt-duckdb are cleanly open source; the original dbt-expectations package is dead and the maintained fork is the one to use.

**8.4 Over-alerting is a known failure mode for auto-raised tickets.**
- *What:* Alarm storms become incident storms unless the alarm layer correlates by run or pipeline before raising. Auto-raise also makes incident volume non-comparable across a changeover; dual-run the series.
- *Source:* memory (architecture mandate).
- *Implication:* Tools feeding ServiceNow must dedupe by run or pipeline ID before raising.

**8.5 Vendor claims need verification.**
- *What:* Vendor-published statistics and dollar figures are point-in-time and must be re-verified before leadership use. Some research claims were refuted on check (for example, that one AI-authored rule compiles to many enforcement targets).
- *Source:* research-and-benchmarks.md.
- *Implication:* The research should cite primary docs for any capability it relies on, and flag vendor-only claims.

---

## 9. Checklist: a candidate tool must...

| # | A candidate tool must... | Hard / Soft | Why (section) |
|---|---|---|---|
| 1 | Never require AI to query or act on live production databases directly | **Hard** | 1.1 |
| 2 | Execute in-warehouse or via an already-approved path (dbt, warehouse-native), or accept a long InfoSec approval for its own DB connection | **Hard** (approval route is possible but slow) | 1.2 |
| 3 | Not send Restricted rows or samples to a vendor-hosted environment | **Hard** | 1.3 |
| 4 | Run natively on, or integrate natively with, GCP and BigQuery | **Hard** | 4.1, 4.4 |
| 5 | Write results with counts, rule IDs, rule versions and run IDs into our BigQuery tables (not only a proprietary store) | **Hard** | 3.2, 4.2 |
| 6 | Hold thresholds, tiers, SLAs, retention and dimension sets as configuration | **Hard** | 5.1, 5.2, 5.3 |
| 7 | Distinguish "not run" from "passed" and from 0%, and report critical failures separately | **Hard** | 5.4 |
| 8 | Avoid stored long-lived secrets; support workload or federated identity | **Hard** | 1.6 |
| 9 | Not replace the mandated alerting (Clairvoyance) or incident system (ServiceNow); feed them instead | **Hard** | 3.6 |
| 10 | Support per-dataset thresholds from one templated definition, with evaluation decoupled from notification | **Hard** for any anomaly-detection tool | 8.1 |
| 11 | Dedupe and correlate alerts by run or pipeline before raising tickets | **Hard** for any tool that raises tickets | 8.1, 8.4 |
| 12 | Fit the operating model: audit-side for DQ Ops; pipeline enforcement is a recommendation to other teams | **Hard** (scope who owns it) | 3.1, 3.4 |
| 13 | Work on raw landing data in GCS or BigQuery, not only on S3 | Soft (Hard if pitched as the Bronze replacement) | 4.6 |
| 14 | Run file-only (CSV or Parquet) for a no-approval proof | Soft | 1.4 |
| 15 | Be provable in a sandbox with BigQuery and Storage rights only (no Cloud Run, no service account, no Vertex) | Soft | 7.4, 4.3 |
| 16 | Support both Snowflake and BigQuery through the migration | Soft | 4.4 |
| 17 | Read or generate from ODCS data contracts | Soft | 4.7 |
| 18 | Not duplicate Dataplex AutoDQ rule execution; fill the lifecycle gaps around it | Soft | 4.5 |
| 19 | Support absolute-range and ratio checks, not only rolling-window anomaly detection | Soft | 6.3 |
| 20 | Add little standing maintenance; onboarding a dataset is a registration | Soft | 6.1 |
| 21 | Be configured in YAML or SQL and versioned in git (GitLab CI) | Soft | 6.2 |
| 22 | Cost little or nothing, or justify itself against an OSS or GCP-native alternative; state whether it needs in-year or next-cycle budget | Soft | 2.1, 2.2 |
| 23 | Carry a clear license (OSI open source preferred; flag BSL or Elastic License) | Soft | 8.3 |
| 24 | Have a dated exit or replacement path; avoid open-ended lock-in | Soft | 2.5 |
| 25 | Not present inferred owners as accountable owners | Soft | 3.3 |
| 26 | Allow its value to be described in plain operations language for procurement | Soft | 2.4 |
| 27 | Have capabilities verifiable in primary vendor docs | Soft | 8.5 |
