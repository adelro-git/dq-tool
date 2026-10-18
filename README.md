# DQ Tools Research Context Pack

**Purpose:** give a deep-research session everything it needs to recommend data quality tools that can actually be adopted at Globe. It explains what Globe's Data Quality Operations team (DQ Ops, part of The Data Group) has already built, the data estate a tool would run on, our integrations today, the constraints a tool must respect, and the open gaps and questions.

**As of:** 2026-10-18. Compiled from the DQ 2026 workspace and its knowledge base. Items marked DRAFT, WIP, proposed or inferred are not settled facts.

**Handling:** internal context. People appear by role only, and there are no credentials, sheet or file IDs, or internal URLs. Check your company's rules before pasting it into any external AI or research service.

## Files

| # | File | Read it for |
|---|---|---|
| 01 | [01-program-org-and-mandate.md](01-program-org-and-mandate.md) | Who DQ Ops is, what we own vs what others own, the 2026 strategy and targets, LEAP (the GCP migration) and our seat in it, the draft policy stack, the DQ framework and its dimensions, key metrics |
| 02 | [02-what-we-built.md](02-what-we-built.md) | Every build (DQ Monitor, alert and incident pipeline, KPI anomaly monitor, check catalog, lineage, control files, data contracts, scorecard, agent specs), each with its status and limits |
| 03 | [03-data-estate-and-architecture.md](03-data-estate-and-architecture.md) | Platforms, the Bronze/Silver/Gold layers, what DQ Ops can and cannot reach, current vs target architecture, the LEAP timeline |
| 04 | [04-constraints.md](04-constraints.md) | InfoSec data path, budget and procurement, operating model, architecture mandates, policy, access gaps, lessons from Elastic, and a hard/soft "a candidate tool must..." checklist |
| 05 | [05-integrations-today-and-possible.md](05-integrations-today-and-possible.md) | The 26-system integration map, current pain points, 12 places a new tool could plug in (with feasibility), and an integration checklist |
| 06 | [06-gaps-prior-research-and-questions.md](06-gaps-prior-research-and-questions.md) | Maturity position, 13 capability gaps, tools already benchmarked (don't redo them), open decisions, 24 prioritised research questions, and a scoring rubric |

## Reading order for a research agent

1. Read **04** (constraints) and **06** (gaps, questions, rubric) first. They define what counts as a good answer.
2. Read **03** and **05** for where a tool must connect.
3. Read **02** for what a tool would replace, complement or duplicate.
4. Read **01** for ownership and governance context when framing recommendations.

## The short version

- **Second-line role.** DQ Ops specifies and audits. Pipeline owners and platform teams build and run. Our team doesn't operate production pipelines.
- **Data path is the hard gate.** AI never runs against live production databases. A third-party tool that opens its own connection to production faces a long InfoSec review. In-warehouse or dbt-native execution is the viable route.
- **The platform is moving.** Silver and Gold sit in Snowflake today. Under LEAP, GCP/BigQuery becomes the system of record over about 15 months, with ServiceNow for incidents and Clairvoyance (Grafana-based) for alerting. Dataplex AutoDQ is already named for running rules. A tool should work across Snowflake and BigQuery, or be BigQuery-native, and fill the gaps Dataplex leaves.
- **Biggest gaps:**
  - No Bronze (raw ingestion) detection since Elastic was decommissioned (coverage off from October 2026).
  - No plausibility or magnitude checks (checks test presence only).
  - No reconciliation.
  - Weak issue lifecycle and remediation tracking.
- **Budget-constrained.** The current stack was built at near-zero license cost on Google Workspace, Apps Script, BigQuery and Snowflake. Any paid tool needs a clear coverage case that gets through procurement.
