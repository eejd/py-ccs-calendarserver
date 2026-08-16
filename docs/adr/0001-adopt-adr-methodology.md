<!--
Copyright (c) 2026 Eric DeWitt. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# ADR-0001: Adopt ADR methodology

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: ongoing
- **Decision owner**: Project lead

## Context

The RFCCalServ revival is a multi-year, multi-phase project with ~15
consequential architectural decisions, many of which are only partially
reversible (parser strategy, DALE DAL future, txweb2 fork-vs-replace, storage
backend cuts, Apple extension segregation). Without formal decision records,
future contributors — or future-us in 6 months — will reverse-engineer *why*
from code diffs and stale chat logs, which is the failure mode ADRs exist to
prevent.

The project needs a stable, searchable record of:

- What was decided.
- What alternatives were considered and why they were rejected.
- What consequences and follow-ups were anticipated.
- When the decision was made and in which phase.

## Alternatives Considered

### Option A — No formal decision records

Decisions live in chat logs, commit messages, and contributor memory.

**Pros**: Zero ceremony; no documents to maintain.
**Cons**: Decisions get re-litigated; rationale is lost; onboarding is harder;
architectural drift goes unnoticed. This is the status quo for most stale
projects and is what we are trying to escape.

### Option B — Decisions embedded in GOAL.md only

All decisions recorded as a flat list in the project charter.

**Pros**: Single document to read.
**Cons**: GOAL.md becomes unwieldy; no per-decision history; no supersede
process; alternatives-considered section would bloat the charter.

### Option C — Formal ADRs (Nygard-style) with immutable + supersede model

Each decision gets its own markdown file in `docs/adr/`. Accepted ADRs are
immutable; revisions create a new ADR that supersedes the old one.

**Pros**: Industry-standard practice; tooling ecosystem (`adr-tools`,
`adr-viewer`, `log4brains`) can consume the format; per-decision traceability;
preserves full history; encourages writing alternatives-considered.
**Cons**: Some ceremony; 15 initial ADRs to write; requires discipline to
keep INDEX.md updated.

## Decision

We adopt **formal ADRs** following the Nygard format, located in
[`docs/adr/`](.), with the conventions below.

### Format conventions (tool-friendly)

These conventions match the `adr-tools` / Nygard baseline so that future ADR
tooling (`adr-tools`, `adr-viewer`, `log4brains`) can consume the files without
migration.

- **Filename**: zero-padded numeric prefix + kebab-case title.
  Example: `0001-adopt-adr-methodology.md`.
- **Title line**: `# ADR-NNNN: Title` (H1, matches adr-tools and log4brains).
- **Metadata block**: bold-field format immediately after the H1 (human-readable
  in GitHub, parseable by tools; avoids YAML frontmatter which renders poorly
  in GitHub):
  ```
  - **Status**: Proposed | Accepted | Deprecated | Superseded by [ADR-MMMM](MMMM-...)
  - **Date**: YYYY-MM-DD
  - **Phase**: 1 | 2 | 3 | ongoing
  - **Decision owner**: role/team
  ```
- **Standard section headings** (in order — tools rely on consistent headings):
  1. `## Context`
  2. `## Alternatives Considered` (with `### Option A`, `### Option B`, etc.)
  3. `## Decision`
  4. `## Consequences` (with `### Positive`, `### Negative`, `### Neutral`,
     `### Risks`, `### Follow-ups`)
  5. `## References`
- **RFC 2119 keywords** (MUST/SHOULD/MAY) used in Decision sections for
  unambiguous requirements.
- **Cross-references** via relative markdown links to other ADRs and to
  `file_path:line_number` in the codebase.

### Supersede process

When a decision is revised:

1. Write a new ADR (next number) with full Context/Alternatives/Decision/Consequences.
2. In the new ADR's Context, link back to the superseded ADR.
3. Edit the old ADR's Status line to `Superseded by [ADR-MMMM](MMMM-...)`.
4. Do NOT modify the old ADR's body — it is immutable.
5. Update [`INDEX.md`](INDEX.md) with the new status and the new ADR row.

### Granularity

- **Core ADRs (15)**: high-impact, high-reversibility-cost decisions. Written
  in phase 1.
- **Deferred decisions**: documented as one-liners in
  [`GOAL.md`](../../GOAL.md) with rationale pointers. Promoted to full ADRs
  only if challenged or revisited.

### Location

```
py-ccs-calendarserver/
  docs/
    adr/
      0000-template.md
      INDEX.md
      0001-adopt-adr-methodology.md
      ...
```

### Visibility

ADRs are **public** in the repository under `docs/adr/`. Community can see
the decision history. Operational/security-sensitive details (specific TLS
config values, key handling) stay out of ADRs and live in deployment docs.

### Tooling

Phase 1: plain markdown + hand-maintained `INDEX.md`. No tooling dependency.
Future: MAY adopt `adr-tools`, `adr-viewer`, or `log4brains` for
viewing/searching/managing. The format conventions above are chosen to be
compatible with all three.

## Consequences

### Positive

- Every architectural decision has a stable, citable record.
- Alternatives-considered section prevents re-litigating settled decisions.
- New contributors can understand *why* the architecture is the way it is.
- Tooling-compatible format means we can adopt ADR viewers later without
  migration.
- Supersede process preserves full history while allowing evolution.

### Negative

- Some ceremony: 15 initial ADRs to write; discipline required to keep
  `INDEX.md` updated.
- Risk of ADRs becoming stale if not maintained. Mitigation: the
  deferred-decisions list in `GOAL.md` is the parking lot; promoted ADRs are
  only written when a decision is actively challenged.

### Neutral

- ADRs are documentation, not code; they do not affect runtime behavior.

### Risks

- **ADR rot**: ADRs that do not reflect actual code behavior. Mitigation:
  ADR References section MUST cite `file_path:line_number` so reviewers can
  verify the decision against the code.
- **Over-use**: promoting too many small decisions to full ADRs. Mitigation:
  the 15-core-ADR threshold; deferred-decisions parking lot in `GOAL.md`.

### Follow-ups

- Evaluate ADR tooling (`adr-tools`, `adr-viewer`, `log4brains`) in phase 1
  or 2.
- Consider auto-generating `INDEX.md` from ADR files via a script.

## References

- Nygard, Michael. "Documenting Architecture Decisions." 2016.
  https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
- `adr-tools`: https://github.com/npryce/adr-tools
- `adr-viewer`: https://github.com/mrwilson/adr-viewer
- `log4brains`: https://github.com/thomvaill/log4brains
- Template: [`0000-template.md`](0000-template.md)
- Index: [`INDEX.md`](INDEX.md)
