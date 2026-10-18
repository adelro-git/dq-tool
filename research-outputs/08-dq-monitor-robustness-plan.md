# DQ Monitor robustness plan

> **Status: RECOMMENDATIONS, NOT DECISIONS.** Written 2026-10-18 by an AI research session from public sources and the context pack (files 01–06). **Not verified against the Monitor's code.**
>
> **Read this before acting on anything below (people and agents alike):**
> 1. Every item is a *candidate* to map against what DQ Ops has already built ([02](../02-what-we-built.md)) and its constraints ([04](../04-constraints.md)), not an instruction.
> 2. Look the item up in [07-reco-crosswalk.md](07-reco-crosswalk.md) first. It records the existing build piece each item touches, its status per the context pack, and the team's decision.
> 3. Confirm the current state in the code before proposing a change. The crosswalk statuses come from documents and may be stale.
> 4. If an item conflicts with an existing Monitor convention, a constraint in 04, or a recorded team decision, **the existing build wins**. Flag the conflict; do not "fix" the build to match this document.
> 5. Do not implement an item whose crosswalk decision is blank or "Reject".


*As of 2026-10-18. Scope: the stack in production today (Snowflake Silver/Gold, dbt on the approved path, the Apps Script and Sheets intake, the BigQuery sandbox, the Monitor web app, ServiceNow via the export Sheet). The move to GCP under LEAP is not final, so nothing here depends on it.*

*Basis: this plan reviews the architecture as documented in the dq-tool context pack (files 02, 03, 05), set against the vendor and open-source research in [10-dq-tools-and-build-considerations.md](10-dq-tools-and-build-considerations.md). It is not a review of the Monitor's source code. Items marked **(verify)** need a check against the code or the platform before committing.*

---

## 1. The current architecture in one picture

```
Snowflake rule procs (1 per table) ─┐
Snowflake AD model (not ours) ──────┼─► alert EMAILS ─► Gmail scraper (Apps Script, 2h) ─► Sheets ─┐
(Elastic Bronze, off Oct 2026) ─────┘                                                             │
ServiceNow ─► OPx export Sheet (2h) ─► scrapeAllIncs join ────────────────────────────────────────┤
Snowflake AD view ─► dbt on ONE LAPTOP (6h) ─► Drive CSV ─► loader ───────────────────────────────┤
Other teams' result Sheets ─► monthly DQI (by hand) ──────────────────────────────────────────────┤
                                                                                                   ▼
                           BigQuery sandbox (external tables → snapshots → procs → 45 marts, 2h)
                                                                                                   ▼
                           Monitor web app (Apps Script, cached) + Gemini narratives
```

The design rules are sound and already ahead of most vendor tools: honest grading (grey is never green, "No score" is never zero), checks for consistency across surfaces, coverage math that knows about problem tickets, and a verdict model with a replay harness that has 0 mismatches. What follows leaves those rules alone and fixes the **inputs and plumbing** underneath them.

## 2. Where it is fragile, and why

| # | Weakness | Root cause | Evidence in the pack |
|---|---|---|---|
| W1 | Detection arrives as **email prose** | Engines don't write result rows; five mechanisms infer which table an alert is about | 05 §2.1–2.2, called "the largest recurring maintenance cost" |
| W2 | Results are **pass/fail, not measured values** | No results history store (T4 not built) | 01 §4, 06 G11 |
| W3 | **Presence-only** rules; no plausibility | Registry has no range, ratio or cross-field types | 0 of 481; the 83× usage case |
| W4 | AD **absorbs persistent errors** | Rolling 30-day baseline learns the bad level | 02 §3.1 |
| W5 | **Survivorship** in scores | A check that stopped running drops out of the population | Gold DQI flat at 100% since Feb |
| W6 | **Freshness is inferred**, including on our own feeds | No measured load timestamp; pipelines check columns, not dates | Median+MAD mirror "succeeding" on frozen data since July |
| W7 | **Single points of failure** | Laptop export, personal credentials, no service accounts | 05 §2.5 |
| W8 | **Stacked latency** and stale ServiceNow state | 2h export → 2h sweep → 2h refresh; snapshot keeps the oldest row | 05 §2.3–2.4 |
| W9 | **No remediation ledger or recurrence tracking** | INC lifecycle ends at "resolved" | 04 §3.2, 06 G9 |
| W10 | Verdict model **restates history** silently | Stateless re-tag on every sweep; no version stamp | 01 §12 |
| W11 | **Checks don't execute from the registry** | `checks.yml` is a catalogue; one hand-written proc per table runs instead | 02 §5 |

## 3. What to borrow from the research

Each item names its source pattern, the weakness it fixes, and what it needs. **Tier A** is within DQ Ops' own control today. **Tier B** needs one known, bounded ask. **Tier C** depends on other teams.

### Tier A: within your control now

**A1. A results-readout table (fixes W2, W5, W10; DQOps pattern)**
Add one append-only table to the BigQuery sandbox landing dataset (Apps Script writes there are already approved). One row per check per run:

```
run_id, check_id, check_version, layer, table, column, dimension,
actual_value, expected_value, lower_bound, upper_bound,
records_evaluated, records_failed, severity (warning|error|fatal),
status (PASSED|FAILED|NOT_RUN|ERROR), source_engine, observed_at, ingested_at
row_id = hash(check_id, partition/date, run grain)   -- reruns overwrite, never duplicate
```

- `NOT_RUN` as its own status makes the draft Standard's "no score when a test did not run" mechanical, and it removes the Gold survivorship 100%.
- Feed it from whatever exists today: parse measured values out of the alert emails where they are present, and backfill from the rule-result Sheets. It becomes the T4 audit trail even before execution moves.
- Keep at least 90 days (seasonal models need it). Plan for 3 years, since the draft Contracts Policy asks for it.

**A2. A control coverage register (fixes W5; Monte Carlo "data products", draft Standard DQ-13)**
A mart of *expected* checks (from `checks.yml`) joined to *observed* runs (A1). Every check gets one of: running, silent (expected but no result in N days), or uncatalogued (results with no registry entry). Surface "silent" checks on Governance & Risk. A table with no rules then reads as "uncovered", not as perfect.

**A3. Plausibility check types in the registry (fixes W3)**
Add these types to `checks.yml`, with dimension tags:

| Type | Example | dbt route on Snowflake (verify package versions) |
|---|---|---|
| `RANGE` | daily broadband usage GB per subscriber between 0 and X | `dbt_expectations.expect_column_values_to_be_between` (maintained fork) or a singular test |
| `RATIO_PER_ENTITY` | usage ÷ active subscribers within band, per brand | singular SQL test returning offending rows |
| `CROSS_FIELD` | reload amount > 0 when reload count > 0 | `dbt_utils.expression_is_true` |
| `PEER_COMPARISON` | brand share of total within its historical band | singular SQL test |
| `RECONCILIATION` (exists, extend) | Gold KPI vs Silver aggregate within 0.3% | singular SQL test with a tolerance parameter |

Start with peso metrics (revenue, ARPU) and the broadband usage family. Coverage today runs opposite to financial materiality.

**A4. Anomaly baselines that can't learn a bad level (fixes W4; Uber, Bigeye, Monte Carlo, Snowflake/Databricks)**
For the model you own (the IN Summary Median+MAD), and as a written recommendation to the owner of the Snowflake AD model:
- **Exclude flagged points from training** until someone marks them normal (Monte Carlo's "mark as normal"). This alone would have stopped the 83× level becoming the baseline.
- Add an **anchored reference** alongside the rolling one, such as the same weekday 52 weeks back or a certified baseline period. Alert when the level departs from the anchor even if it is stable against the last 30 days.
- Use **one-sided bounds** on volumes by default.
- Use **exclusion windows** for PH holidays and test the 15th/30th payday cycle (an inference to check against your data).
- Keep severity at **table level** (Uber's lesson against alert fatigue).

**A5. Freshness heartbeat for every feed, including your own (fixes W6; Monte Carlo table-monitor tier)**
- Read `LAST_ALTERED` / `MAX(load_ts)` per watched Snowflake table through the approved read path, near-zero cost. That gives the measured Fresh / Overdue / Unverified state the panel already models.
- Extend the hourly BigQuery watchdog to every *internal* input: Drive CSV age, mirror sheet max date, the OPx export timestamp, the narrative timestamp. Show a "source stale" banner per panel, not just the "Data as of" stamp.
- Check the date, not the columns. That is the exact failure the mirror had.

**A6. Fix the ServiceNow snapshot read (fixes W8, quick win)**
Choose the **latest** row per ticket (by updated timestamp, or the last appended row), not the first. Do this for resolution notes and resolved date as well as state. About 11 tickets lost notes and dates in one sample. Add a parity check between the two export tabs.

**A7. Remediation ledger and recurrence (fixes W9; Monte Carlo incident model)**
Built in the sandbox from the incident data you already have:
- Per issue: owner as named by DPG (or "unnamed"), expected resolution date, status history with timestamps, verified-closure date, and a **recurrence window**: did the same check/table fail again within N days of closure?
- Keep **status separate from model feedback**. Monte Carlo deliberately doesn't let triage clicks loosen thresholds, and the same logic protects your verdict model.
- This directly feeds DAQO OGSM R5 (deviation register, % with named owner and date, repeat rate).

**A8. Version the verdict model (fixes W10)**
Stamp every verdict with `classifier_version` and `tagged_at`, and keep prior verdicts in a history table rather than overwriting. Months still restate when rules change, but the Monitor can show "restated by vX" and audit can recompute any past figure.

**A9. Group alerts before they become tickets (Elastic lesson, Monte Carlo, Uber)**
- In the Monitor's alert model, group alerts by table and run within a time window into one "issue". Count yield per issue, not per email.
- Re-enable the Incident Digest as the delivery tier for low-severity items: real-time for critical tables, digest for the rest.
- Put the same grouping rule into the INC field-mapping request, so auto-raised tickets carry a correlation ID.

**A10. Monitor health: engine and check yield as a standing panel**
You already measured alerts per confirmed incident per engine (rules 2.3:1, AD 15:1, Elastic 48:1). Compute it continuously **per check** from A1 and A7, and list checks with zero actionable alerts in 90 days as pruning candidates. Monte Carlo's guidance that every monitor maps to an action is the same principle.

### Tier B: one bounded ask each

| # | Change | Fixes | The ask |
|---|---|---|---|
| B1 | Run `checks.yml` through **dbt on Snowflake** (registry → generic and singular tests), and write results to a results table in the DQ group schema on EDO PRD; tests return `actual_value` and failing-row counts, not just pass/fail | W11, W2, W3 | An EDO PRD dbt connection (today's profile reaches CSSG only) and a non-interactive credential |
| B2 | Schedule with **Snowflake tasks** in the DQ group schema (your role can own tasks there) or a **GitLab CI** runner, not a laptop | W7 | Same non-interactive credential; for CI, a governed runner |
| B3 | Move the AD export off the laptop: a Snowflake task writes to a stage, then GCS, then BigQuery (the CHM pattern) | W7, W8 | A storage integration on EDO PRD (account-admin ask); interim fallback is B2's runner |
| B4 | **Structured alerts**: ask the rule-check and AD owners to also write each alert as a row (table, rule, measured vs expected, severity, run ID) to a shared table, with email kept only as notification | W1 | Owner agreement; no new approval class |
| B5 | **ServiceNow Table API** read instead of the export Sheet | W8 | Integration user (pending since 2026-09-01) |
| B6 | Shared or service identity for the BigQuery scheduled queries and Apps Script triggers | W7 | DPE or sandbox owner |

B4 is the biggest structural win. Once alerts arrive as rows, the scraper, parser and table-inference mechanisms shrink to a fallback.

### Tier C: depends on others, so specify rather than build

- **Bronze:** DQ Ops can't reach S3 or DataOne. The practical step is auditing **control-file results** (expected vs loaded counts, the 99.7% target) once DataOps lands them in a table, through the A1 schema. Specify that shape now.
- **Lineage:** keep the explorer for incident tracing. Feed impact and grouping from it where one report is onboarded, and register with DataHub rather than compete with it.
- **Network (NTG) results:** ingest via Sheet → CSV into the A1 schema as a separate domain, keeping "No score" until it is graded.

## 4. What not to borrow

- **Vendor SaaS or hosted UIs (Monte Carlo, Anomalo SaaS, Vercel).** These fail the InfoSec rules: their own production connections, and Restricted data on a third party. Their *patterns* are what's worth taking.
- **Row sampling into the app.** Monte Carlo's AI features depend on it, and that conflicts with your data-path rule. Keep failing-row evidence capped and in-warehouse (the MRD pilot's 1,000-row cap is the right shape).
- **One ML model per dataset to tune.** This is the Elastic failure mode and too heavy for the team. One templated definition, with thresholds as parameters per dataset.
- **AI in the detection path.** Keep Gemini (and later Cerberus on Genesis) on rule drafting, narratives and RCA screening, reading A1 and A7. Detection stays deterministic and reproducible.
- **DQOps as a dependency.** BSL licence and live JDBC. Its sensor/rule/readout design is already reflected in A1 and B1.

## 5. Suggested sequence

| When | Do | Why first |
|---|---|---|
| **Weeks 1–2** | A6 (ServiceNow latest row), A5 (freshness on internal feeds), A8 (verdict versioning) | Small, fixes known data bugs, no dependencies |
| **Weeks 2–6** | A1 (results table + backfill), A2 (coverage register), A9 (grouping + digest), A10 (yield panel) | Gives audit evidence and kills survivorship; uses only the sandbox |
| **Weeks 4–8** | A3 (plausibility types, peso metrics first), A4 (anchored baselines on your own model + written ask to the AD owner) | Closes the 83× class of miss |
| **In parallel** | File the Tier B asks: EDO PRD dbt connection and non-interactive credential (B1/B2), storage integration (B3), structured-alert agreement (B4), chase the ServiceNow integration user (B5) | Org clock is the bottleneck, so start now |
| **When B1/B2 land** | Retire one-proc-per-table; dbt runs the registry on a schedule and writes to A1 | Registry becomes execution |
| **Then** | A7 remediation ledger + recurrence on top of A1 and clean incident data | Needs A6 and A9 first to be trustworthy |

## 6. Mapping to the context pack's checklist

| File 04 checklist item | Covered by |
|---|---|
| 5 Results with counts, rule IDs, versions, run IDs | A1, B1 |
| 6 Thresholds, tiers, dimension sets as configuration | A1 (severity, dimension columns), A3 |
| 7 "Not run" ≠ passed ≠ 0% | A1 `NOT_RUN`, A2 |
| 8 No long-lived secrets | B2, B6 |
| 10 Per-dataset thresholds from one template; evaluation decoupled from notification | A4, A9 |
| 11 Dedupe/correlate before tickets | A9 |
| 19 Absolute-range and ratio checks | A3 |
| 20 Onboarding is a registration | A3 + B1 (one `checks.yml` entry per check) |
| 21 YAML/SQL in Git | A3, B1 |

Items 4, 13, 15 and 16 (GCP-native execution, Bronze on GCS, sandbox-only proof, Snowflake + BigQuery portability) are left out because the platform move isn't final.

## 7. To firm this up

- A code-level review of the Monitor and Apps Script projects (the GitLab repo) to confirm W1–W11 and size each item.
- Confirm which alert emails already carry a measured value (this decides how much of A1 can be backfilled).
- Confirm dbt package versions available on the approved path (the dbt-expectations fork, dbt-utils) **(verify)**.
