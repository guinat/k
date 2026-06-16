# project-one

Clean, consistent scoping documentation for the **storage capacity management & deviation detection** project — an AI / agentic (LangGraph) system that analyzes filesystem and storage configurations across a heterogeneous multi-OS fleet (AIX · Linux, all distributions · Windows Server), at least 500 servers, to serve five objectives: **O1** deviation detection vs standards · **O2** optimization (useless / obsolete / misplaced) · **O3** reliability & full-filesystem prevention · **O4** operational efficiency & forecasting · **O5** risk reduction & compliance.

This folder holds the project's scoping documentation: a single canonical data model, one consistent terminology, English throughout.

## Canonical data model (in one line)

Each collection run produces **one envelope file** — `{ snapshot_id, generated_at, host_count, hosts: [ … ] }` — where every `hosts[]` element is one server's document in a **single, OS-agnostic schema** (`filesystems[]` + `storage_topology` + `paging[]`), all sizes in **exact bytes**, non-applicable fields set to explicit `null`. Run identifiers live on the envelope; each host carries only its `scan_timestamp`.

## Documents (`docs/`)

Read in this order:

1. [Project Overview](docs/project-overview.md) — purpose, scope, the five objectives, approach, process, deliverables, key locked decisions, glossary. **Start here.**
2. [Business Goals](docs/business-goals.md) — B-01…B-13 with areas, value, mapped objectives, and KPIs.
3. [Requirements](docs/requirements.md) — functional (FR-01…FR-20) and non-functional (NFR-01…NFR-17).
4. [Risk Register](docs/risk-register.md) — RSK-01…RSK-23 with mitigations, plus the top-risk responses.
5. [Data Collection Specification](docs/data-collection-spec.md) — the authoritative data contract: the OS-agnostic schema, per-level field tables, objective coverage, excluded derived fields, and worked JSON examples.
6. [Data Collection Request](docs/data-collection-request.md) — the ready-to-send message to the data / collection team (cover note + Appendix A schema + Appendix B field rationale).

## Next steps (out of scope for this set)

Reference standards / golden images · OS-by-OS field-applicability matrix · field-by-field rationale + Ansible collection · LangGraph pipeline architecture · machine-validatable JSON Schema file.

Status: **scoping / pre-implementation** (no application code yet) — 2026-06-15.
