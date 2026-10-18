# 07 · Recommendation crosswalk: research outputs mapped to what DQ Ops already built

**Start here.** This file is the entry point to `research-outputs/`. Every recommendation in files 08, 09 and 10 has one row below. The row says which existing build it touches, what state that build is in, and what the team decided.

## Rules for any agent or reader

1. **The existing build is the baseline.** Files 08–10 are recommendations from an AI research session (2026-10-18). They are not requirements, standards or decisions.
2. **Precedence when sources disagree:** the Monitor's code and live behaviour first, then recorded team decisions, then the context pack (01–06), then these research outputs. A recommendation never overrides a convention in [02](../02-what-we-built.md) or a constraint in [04](../04-constraints.md).
3. **Status values below come from the context pack, not from reading code.** Before proposing work, confirm the status in the code and correct this file if it is wrong.
4. **Only act on rows whose Decision is Adopt or Adapt.** Blank means not yet reviewed. Do not implement it, but you may draft a proposal for the team to review.
5. **Scope is the current stack:** Snowflake, dbt on the approved path, Apps Script and Sheets, the BigQuery sandbox, the Monitor web app, and ServiceNow via the export Sheet. The GCP move under LEAP is **not final**. Items that depend on it are marked *Deferred (LEAP)*.

### Status vocabulary (vs the existing build, per context pack)

| Status | Meaning |
|---|---|
| **Exists** | The pack describes this as built. The recommendation may add nothing or only refine it. |
| **Partial** | Something related is built. The "Existing piece" column names it, and the recommendation extends it. |
| **Planned** | Already in the team's own plan (T1–T9, D-decisions) but not built. The recommendation should align with that plan, not replace it. |
| **Not built** | Nothing comparable in the pack. |
| **Unknown** | The pack doesn't say. Check the code. |
| **Superseded** | Written for a target state that doesn't apply now. Kept for reference only. |

### Decision vocabulary (filled in by the team)

**Adopt** (as written) · **Adapt** (change noted) · **Already have** · **Defer** · **Reject** (reason noted). Add the date and the role that decided.

---

## A. Robustness plan items (file 08)

| ID | Recommendation (short) | Existing piece it touches | Status | Note | Decision |
|---|---|---|---|---|---|
| A1 | Append-only results-readout table: measured value, bounds, counts, rule version, `NOT_RUN` status | BigQuery sandbox (`dqops_landing`); T4 results-readout store | **Planned** (T4) | Align the schema with T4's definition; don't create a second design | |
| A2 | Control coverage register: expected vs observed checks | `checks.yml`; Governance "Tables to add DQ checks to" queue | **Not built** (named as an audit need in 01 §11) | Builds on the registry's existing entries | |
| A3 | Plausibility check types: range, ratio per entity, cross-field, peer comparison | `checks.yml` types (has RECONCILIATION, CUSTOM_SQL) | **Partial** | DFM authors rules (04 §3.1). DQ Ops may only add these as interim monitoring checks or recommendations | |
| A4 | Anomaly baselines that exclude flagged points; anchored reference; one-sided volume bounds | IN Summary Median+MAD mirror (stale); Snowflake AD model (not owned) | **Partial** | Only the mirror is DQ Ops' to change; for the Snowflake model this is a recommendation to its owner | |
| A5 | Freshness heartbeat per table and per internal feed; "source stale" banner | Hourly BigQuery freshness watchdog; Timeliness v3 (three states, alert-based) | **Partial** | Extends existing pieces; check what the watchdog already covers | |
| A6 | Read the latest ServiceNow snapshot row (notes and resolved date too) | `scrapeAllIncs` / All INCs consolidation | **Partial** (state fixed 22 Sep; notes and dates still read oldest row) | Known bug in 02 §2.1 | |
| A7 | Remediation ledger with recurrence window | Incident registry, PRB tab, OPx tagging | **Not built** | Supports OGSM R5; owner naming stays with DPG (04 §3.3) | |
| A8 | Version-stamp verdicts; keep verdict history | Verdict categorizer + replay harness | **Partial** (harness exists; no stamp, history restates) | | |
| A9 | Group alerts into issues; tiered delivery; restart digest | Parser dedup/fan-out; Incident Digest (paused) | **Partial** | Digest is paused pending team feedback (02 §2.5) | |
| A10 | Per-check yield panel; pruning candidates | Engine-yield analysis (Jan–Aug 2026, one-off) | **Partial** | Makes a one-off analysis continuous | |
| B1 | Run `checks.yml` through dbt on Snowflake, writing results to a table | T2 dbt engine design; D6 proposal | **Planned** (dbt build deferred) | Needs an EDO PRD dbt connection | |
| B2 | Schedule via Snowflake tasks or GitLab CI, not a laptop | AD laptop export; D6 | **Planned** (D6 open) | | |
| B3 | AD export via stage → GCS → BigQuery | Laptop AD export; CHM pattern | **Planned** (on hold) | Needs a storage-integration admin grant | |
| B4 | Engine owners write alerts as rows, not just emails | Gmail scraper and parser | **Not built** | Needs the other teams' agreement | |
| B5 | ServiceNow Table API instead of the export Sheet | OPx export Sheet | **Planned** (integration user pending since 2026-09-01) | | |
| B6 | Shared or service identity for scheduled queries and triggers | Personal-credential jobs | **Not built** | Sandbox has no service accounts (04 §7.4) | |

## M. Presentation patterns (file 09)

Panel numbers follow file 09: 1 Executive, 2 IN Summary, 3 Anomaly, 4 Alert Monitoring, 5 DQI, 6 Incident Response, 7 Governance, 8 SRG, 10 About.

| ID | Panel | Recommendation (short) | Existing piece | Status | Decision |
|---|---|---|---|---|---|
| M1 | 1 | Checks-running *n of m* and coverage on report health cards | Report health cards | Partial (no coverage figure) | |
| M2 | 1 | Time window written into every tile label | "Every number names its base" convention | Unknown (base yes, window?) | |
| M3 | 1 | Hatch the current partial month; leave it out of trends | Alerts per month by layer | Unknown | |
| M4 | 1 | Days to close as median + count; time to first update | Days to close tile; MTTR naming rules | Partial | |
| M5 | 1 | Top fold ≤ 6 visuals, Catch → Confirm → Fix → Trust | Executive Dashboard structure | **Exists** (verify count) | |
| M6 | 1 | "Reports healthy over time" line from daily RAG snapshots | Shared RAG helper | Not built | |
| M7 | 2 | Relative + absolute freshness stamp; per-feed stale banner | "Data as of" stamp | Partial | |
| M8 | 2 | Plain-English row title; measured vs expected in its own column | IN Summary rows | Partial (value needs A1) | |
| M9 | 2 | Seven-run history strip per row | — | Not built | |
| M10 | 2 | Pre-publish line as counts; open True Incident blocks "clear" | IN Summary pre-publish use | Partial (rule needs team sign-off) | |
| M11 | 3 | Card header: latest value, "Expected X–Y", last-evaluated marker | Anomaly charts with expected band | Partial (band exists) | |
| M12 | 3 | Grey "Training" and "Not monitored" states | Grey-never-green convention | Unknown | |
| M13 | 3 | Anomalous first; small multiples in "by table" lens (lazy-loaded) | "By table" lens | Partial | |
| M14 | 3 | Hatch points excluded from training and exclusion windows | — | Not built (needs A4) | |
| M15 | 4 | Timeliness strip per table: arrivals vs expected window | Timeliness by table | Partial (needs A5) | |
| M16 | 3 | Segment ranking for KPI anomalies (aggregates only) | — | Not built | |
| M17 | 4 | Facet panel with counts; group-by toggle | Alert Monitoring filters | Unknown | |
| M18 | 4 | Issues, not emails: one row per grouped issue | All Alerts rows (per email/incident) | Partial (needs A9) | |
| M19 | 4 | Shape-coded verdict and Timeliness chips | Verdict mix display | Unknown | |
| M20 | 4 | Reports-impacted count on each issue | Reports Impact tiles | Partial | |
| M21 | 4 | Layer/table health table, most unhealthy first, coverage bars | Layer health | Partial (coverage needs A2) | |
| M22 | 4 | Per-check yield table; pruning candidates | Engine-yield analysis | Partial (needs A10) | |
| M23 | 4 | Ticket SLA as % within + median + breached list | SLA on alert tickets | Partial | |
| M24 | 5 | Method tooltip on composites; "restated by vX" stamp | DQI composites | Unknown (stamp needs A8) | |
| M25 | 5 | "Evaluated n of m expected checks"; "No score" if n = 0 | DQI composites | Not built (needs A1, A2). Would expose Gold survivorship | |
| M26 | 5 | Dimension cards with roll-up pill and trend | DQI layer × dimension | Partial | |
| M27 | 5 | Rule table with records evaluated and failed | Rule-result workbooks (Sheets) | Not built (needs A1) | |
| M28 | 5 | Bands as target + threshold inside the shared RAG helper | Shared RAG helper | Partial | |
| M29 | 6 | Registry row: severity, INC, title, verdict, resolver, age | Incident registry | Partial | |
| M30 | 6 | Read-only status timeline per incident | — | Not built (needs A6, A7) | |
| M31 | 6 | Recurrence badge | — | Not built (needs A7) | |
| M32 | 6 | Narrative opener quoting observed vs expected, labelled AI-drafted | Gemini narrative layer | Partial (needs A1) | |
| M33 | 6 | Root cause as a link to the PRB or upstream job | Linked PRBs | Partial | |
| M34 | 7 | Coverage register view: running / silent / uncatalogued | — | Not built (needs A2) | |
| M35 | 7 | Rank "Tables to add DQ checks to" by impact, with reason | That priority queue | Partial | |
| M36 | 7 | Coverage matrix: tables × check types | — | Not built (needs A2, A3) | |
| M37 | 7 | Saved views per resolver-team lane | Resolver-team lanes | Partial | |
| M38 | 8 | "Not enabled: no score" empty state | SRG placeholder | Partial | |
| M39 | 10 | Methodology page | About: "how we grade" | Partial | |

## R. First-report design recommendations (file 10). Superseded framing

File 10's R-list assumed a new build on a target platform. Each row says what still applies to the current stack, and where.

| ID | Recommendation (short) | Existing piece | Status | Current-stack equivalent | Decision |
|---|---|---|---|---|---|
| R1 | Control plane / executor split | Checks run in Snowflake; Monitor reads results | Partial | B1, B2 | |
| R2 | Metadata tier on all tables + query tier on critical | — | Not built | A5 (freshness); Snowflake DMFs were piloted and not pursued (04 §8.2) | |
| R3 | Versioned check definitions in Git with plan/dry-run | `checks.yml` in GitLab; T3 CI gate | Partial / Planned | B1, T3 | |
| R4 | Sensor/rule separation | — | Not built | A1, B1 | |
| R5 | Append-only partitioned results history | BigQuery snapshots and restore points | Partial | A1 | |
| R6 | Partition-level incremental checks + freshness gate | Silver checks evaluate new records | Partial | A5 | |
| R7 | Seasonal baseline, training exclusion, mark-as-normal | Snowflake AD; Median+MAD same-weekday | Partial | A4 | |
| R8 | Warning/error/fatal severity tiers | Binary pass/fail; T3 bands | Planned | T3 | |
| R9 | Incident model: grouping, SEV, TTA/TTR, pruning | Verdict taxonomy, INC/PRB, OPx | Partial | A7, A9, A10 | |
| R10 | Mandatory owner and tier per asset; routing | Owner naming is DPG's (04 §3.3) | Partial | Show missing owners; don't assign | |
| R11 | Lineage from query history | Document-built lineage explorer; DataHub official | Partial | Query-log lineage was excluded by constraints (06 §3.2) | |
| R12 | Aggregates-first privacy, masked samples | MRD failing-record cap (1,000) | Partial | Keep as is | |
| R13 | SSO, RBAC, audit log, secrets management, telemetry off | Domain-gated web app; no service accounts | Partial | B6 | |
| R14 | Host in Globe AWS, or UI on Vercel | — | **Superseded** | Conflicts with 04 §1.3 and 05 §0. Suggested: Reject | |
| R15 | Integrate dbt/GX results; embed only permissive licences | dbt approved; prior licence review (06 §3.1) | Exists (principle) | — | |
| R16 | Telco check library (CDR counts, MSISDN validity, reconciliation) | Registry check types | Partial | A3 | |
| R17 | AI as a thin layer; deterministic detection | "AI authors, engines execute"; Gemini; Cerberus | **Exists** (principle) | — | |

## C. First-report capability checklist (file 10), condensed

The C-list is a generic target-state checklist. Rows are grouped where the mapping is the same.

| IDs | Capability | Status vs existing build | Pointer |
|---|---|---|---|
| C-01 | Source connectors | Partial: Snowflake via dbt only; Bronze unreachable | 03 §7.3 |
| C-02 | Asset inventory with owner/domain/tier | Partial: registry tags; owners are DPG's | A2, R10 |
| C-03 | Freshness/volume/schema monitors on all tables | Partial: timeliness is alert-based; schema drift paused | A5 |
| C-04, C-05 | Column metrics; custom SQL rules mapped to dimensions | Partial: presence-heavy registry | A3 |
| C-06 | Checks-as-code with CI | Planned (T3) | B1 |
| C-07, C-08 | Scheduler with triggers; partition-level execution | Partial: Snowflake procs per table | B1, B2, A5 |
| C-09 | Append-only result history | Planned (T4) | A1 |
| C-10 | Seasonal thresholds, exclusions, mark-as-normal | Partial | A4 |
| C-11 | Severity levels; blocking gate | Planned (T3); blocking is producers' job (T6) | — |
| C-12 | Alert routing | Partial: digest paused; Clairvoyance future | A9 |
| C-13, C-14, C-16 | SSO/RBAC; immutable audit log; secrets | Partial | B6, A1 |
| C-15 | Sampling off by default; masking | Exists in spirit (MRD cap) | — |
| C-17, C-18 | Incident lifecycle; TTA/TTR SLOs | Partial: SLA on alert tickets exists | A7 |
| C-19 | Source-to-target reconciliation | Partial: 87 validity/recon checks; no recon tooling | A3 |
| C-20 | Segmented metrics, test-before-save | Unknown | — |
| C-21, C-22 | Lineage; impact-aware grouping | Partial: one report onboarded | R11 |
| C-23 | Scorecards per domain/dimension | Exists: Enterprise DQ Scorecard v3, DQI | M24–M28 |
| C-24 | Monitor-health and alert feedback | Partial: verdicts + yield analysis | A10 |
| C-25 | Cost meter per check | Not built (~230 credits/month known in total) | — |
| C-26 | Ingest dbt/GX results | Planned with B1 | B1 |
| C-27 | API/MCP for status | Partial: Genesis/Cerberus reads results (planned) | — |
| C-28 | Telco check library | Not built | A3 |
| C-29 | Data products with inherited coverage | Not built | A2 |
| C-30, C-31 | Multivariate drift; triage scoring | Not built | Defer |
| C-32 | LLM rule suggestion with human approval | Planned: Cerberus drafts checks for approval | — |
| C-33 | Unstructured / AI-output monitoring | Not built; out of current scope | Defer |
| C-34 | Compliance artefacts (PIA, NPC registration) | Unknown | Ask DPO |

---

## How to update this file

- When the team decides on an item, fill in **Decision** with the date and role, e.g. "Adapt: schema follows T4, 2026-10-25, DQ Ops head".
- When code review shows a status is wrong, fix the status and add a short note.
- Don't edit files 08–10 to record decisions. They stay as the original recommendations, and this file is where they meet the real build.
