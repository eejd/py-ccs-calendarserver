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

# ADR-0003: Mission — Py3 revival as typed contract for Rust+PyO3

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: ongoing
- **Decision owner**: Project lead

## Context

The RFCCalServ project has three goals, in priority order:

1. **Primary**: Evaluate the state of the code and create an initial audit of
   the needed work, any clear initial targets for dependency replacement, and
   any Python migration challenges. Migrate all core libraries to
   strictly-typed wherever we can do this immediately to improve development
   cycle reliability, catch problems as we migrate to Python 3.14 benefiting
   from the stricter checks, and prepare the way for the future Rust (strictly
   typed) library migrations.

2. **Secondary**: Expand the existing codebase to fully implement all current
   (Aug 2026) related RFCs to bring the infrastructure up to current standards
   compliance.

3. **Long-term**: Use the updated and validated Python codebase as a contract
   and reference source to implement a Rust + PyO3 implementation of all core
   libraries. At that time, consider refactoring or creating new API seams to
   separate and improve any architectural decisions, facilitate scaling up for
   performance or any other identified needs.

The original Apple work was considered one of the most standards-compliant
open source CalDAV/CardDAV implementations, so building on the stale project
is preferred over starting greenfield.

This ADR records the three-goal structure and the "typed Python as contract"
principle so that future contributors understand the sequencing rationale and
do not, e.g., skip the typing phase to rush a Rust port.

## Alternatives Considered

### Option A — Greenfield Rust server from the start

Skip the Python revival; build a new Rust CalDAV/CardDAV server from scratch
using RustiCal/Stalwart as references.

**Pros**: No legacy debt; clean architecture from day one; Rust's performance
and safety.
**Cons**: Loses the standards-compliance pedigree of the Apple codebase; the
existing test corpus (1,221 XML scripts in `ccs-caldavtester`) and the
battle-tested lenient iCalendar parser (`ccs-pycalendar`) would need to be
re-derived; much higher risk of protocol-conformance regressions; the user's
explicit goal is to build off the stale project.

### Option B — Python revival only, no Rust end-state

Revive the Python codebase to Py3 + modern deps + RFC compliance and stop
there. No Rust migration.

**Pros**: Lower total effort; no PyO3 complexity.
**Cons**: Leaves performance on the table (the `ccs-pycalendar` RRULE
expansion and DALE DAL are CPU-intensive hot paths); the user's long-term
goal is explicitly a Rust+PyO3 reimplementation; misses the opportunity to
use the typed Python as a contract.

### Option C — Three-goal structure with typed Python as contract

Revive the Python codebase to Py3 + strict typing + RFC compliance, then use
the typed Python as a contract and reference source for a future Rust + PyO3
reimplementation of core libraries.

**Pros**: Preserves the standards-compliance pedigree and test corpus; strict
typing catches bugs during the Py3 port and documents the contract for the
Rust port; phased approach reduces risk; the user's explicit goal.
**Cons**: Higher total effort than Option B; requires maintaining Python and
Rust simultaneously during phase 3; strict typing of a legacy codebase is
non-trivial.

## Decision

We adopt **Option C: the three-goal structure** as described in Context.

### Principles

1. **Phase 1 (Primary)**: Py3 port + strict typing of contract layer + security
   fixes + key modern RFCs. The typed Python codebase is the deliverable.
2. **Phase 2 (Secondary)**: Full 2026 RFC compliance. The typed Python
   codebase remains the deliverable.
3. **Phase 3 (Long-term)**: Rust + PyO3 reimplementation of core libraries,
   using the typed Python as the contract. At that time, refactor or create
   new API seams to separate and improve architectural decisions.

### "Typed Python as contract" principle

Strict typing of the Python codebase serves three purposes:

1. **Documents the contract** a Rust reimplementation MUST satisfy.
2. **Lets us mechanically extract** `dataclass`/`TypedDict` shapes that map
   1:1 to Rust `serde` structs.
3. **Catches `None`-where-you-expected-`str` bugs** that would otherwise
   surface as Rust panics.

Therefore, strict typing of the contract layer (the modules that are PyO3
port candidates) MUST happen in phase 1, in parallel with the Py3 port. See
[ADR-0005](0005-mypy-strict-per-module-rollout.md) for the typing rollout
strategy and [TYPING_PLAN.md](../audit/TYPING_PLAN.md) §10 for the PyO3
preparation analysis.

### Rust port candidate ordering (phase 3)

See [`GOAL.md`](../../GOAL.md) Phase 3 roadmap for the ordered list of Rust
port candidates and the "keep as Python" list.

## Consequences

### Positive

- Preserves the standards-compliance pedigree and the 1,221-script test
  corpus.
- Strict typing catches bugs during the Py3 port and documents the Rust
  contract.
- Phased approach reduces risk; each phase has a clear deliverable.
- Aligns with the user's explicit three-goal structure.

### Negative

- Higher total effort than a Python-only or Rust-only approach.
- Phase 3 requires maintaining Python and Rust simultaneously during the
  transition.
- Strict typing of a legacy codebase (741 .py files, ~324k LOC) is
  non-trivial; see [TYPING_PLAN.md](../audit/TYPING_PLAN.md) for the
  challenge analysis.

### Neutral

- The three-goal structure does not preclude intermediate releases; each
  phase produces a usable server.

### Risks

- **Phase 3 never happens**: if the Python revival consumes all available
  effort, the Rust port is perpetually deferred. Mitigation: the typing
  phase explicitly types the PyO3 port candidates first, so the contract is
  ready even if the port is delayed.
- **Typing becomes a bottleneck**: strict typing of legacy code can be slow.
  Mitigation: per-module strict overrides (ADR-0005); type the contract
  layer first; leave test modules lenient.

### Follow-ups

- ADR-0005: mypy strict rollout strategy.
- ADR-0010: parser strategy (ccs-pycalendar as contract, calcard evaluation
  in phase 3).
- [TYPING_PLAN.md](../audit/TYPING_PLAN.md) §10: PyO3 preparation analysis.
- [MIGRATION_PLAN.md](../audit/MIGRATION_PLAN.md): phase 1 execution plan.

## References

- [GOAL.md](../../GOAL.md) — mission statement and phase roadmap
- [AUDIT.md](../audit/AUDIT.md) — full state-of-code audit
- [TYPING_PLAN.md](../audit/TYPING_PLAN.md) — mypy rollout + PyO3 preparation
- [MIGRATION_PLAN.md](../audit/MIGRATION_PLAN.md) — Py3 migration sequence
