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

# ADR-0015: Compliance testing architecture

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

The project has an existing conformance test suite (`ccs-caldavtester`,
1,221 XML test scripts + 995 fixture files) that is the most complete
CalDAV/CardDAV scenario suite in existence. It encodes Apple's actual
conformance requirements including hard-won edge cases (push, sharing,
attachments, alarms, default-alarms, calendaruserproxy, etc.).

However:

- The **driver** (7k LOC Python) is Python 2.7-only: 16 `except X, e:`,
  31 print statements, `cElementTree`, `httplib`, `rfc822`, `plistlib`
  py2 APIs. See [AUDIT.md](../audit/AUDIT.md) §ccs-caldavtester.
- The **1,221 XML test scripts** are data (declarative), not code — they
  are reusable as-is.
- There is no **RFC-anchored** test suite (one suite per RFC, with
  section-level traceability).
- There is no **property-based** or **fuzz** testing.
- There is no **shared test corpus** between the Python server and the
  future Rust implementation (ADR-0003 phase 3).
- The test runner is `twisted.trial` (102 files import `twisted.trial`);
  modern test ergonomics (parametrization, fixtures, reporting) favor
  `pytest`.

### Existing test infrastructure survey

| Tool | Coverage | Status |
|---|---|---|
| `ccs-caldavtester` (Apple) | CalDAV 4791 + extensions + Apple extensions | In repo; py2-only driver |
| CalConnect (TC TECH) test event artifacts | Round-robin interop | Public test materials periodically released |
| IETF conformance tools | **None** — IETF does not run conformance programs | n/a |
| libical validation | 5545 parse + recurrence | Active; `libical/src/test/` |
| `icalendar` (Python) validation | 5545 syntactic + semantic | Active; `Component.validate()` |
| vdirsyncer `tests/system/` | Black-box CalDAV/CardDAV conformance matrix | Active; cross-server |
| sabre/dav `tests/` | White-box + protocol-level | Active |
| Radicale `integ_tests/` | End-to-end | Active |

No formal CalDAV/CardDAV conformance suite exists at the IETF or CalConnect
that we can adopt. We MUST build our own, reusing the `ccs-caldavtester`
corpus as the foundation.

## Alternatives Considered

### Option A — Keep ccs-caldavtester as-is; wrap in a Docker container with py2

Minimal modernization: run the py2 driver in a Docker container.

**Pros**: Fastest to a working test gate.
**Cons**: Defers the conformance architecture; py2 Docker container is a
maintenance liability; no RFC-anchored suite; no property/fuzz testing;
no shared corpus with Rust.

### Option B — Build a new pytest+YAML suite from scratch; port key scenarios

Start fresh with a pytest-based suite organized per-RFC. Port the most
valuable `ccs-caldavtester` scenarios by hand.

**Pros**: Cleaner long-term; modern from day one.
**Cons**: Loses years of accumulated edge-case coverage in the short term;
high effort to re-derive scenarios; risk of missing subtle edge cases.

### Option C — Modernize ccs-caldavtester driver; keep XML corpus; add pytest+YAML Tier-0

Preserve the 1,221 XML scripts as the Tier-1 scenario corpus. Rewrite the
Python driver (7k LOC) from py2/Twisted-16 to Py3.12 + pytest + treq. Add a
new RFC-anchored Tier-0 suite (`tests/rfc/rfcNNNN/`) with YAML data shared
with a future Rust runner. Add Hypothesis property tests for iCal/vCard.

**Pros**: Preserves the accumulated edge-case coverage; modern driver;
RFC-anchored traceability; shared YAML corpus for Rust; property/fuzz
testing; aligns with the user's directive.
**Cons**: Some effort to modernize the driver (7k LOC); two test tiers to
maintain (but they serve different purposes).

## Decision

We adopt **Option C: modernize ccs-caldavtester driver; keep XML corpus;
add RFC-anchored Tier-0 YAML suite; add Hypothesis + atheris.**

### Test pyramid (4 tiers)

#### Tier-0 — RFC-anchored unit suites

- Location: `tests/rfc/rfcNNNN/` (one directory per RFC).
- Each directory contains:
  - The ABNF or normative grammar as a `.txt` reference.
  - Fixture files (valid + invalid + boundary cases).
  - `test_*.py` files annotated with the exact section numbers being
    exercised (e.g. `test_9_6_calendar_query.py` for RFC 4791 §9.6).
- Data format: YAML (request, response, preconditions, postconditions).
- Shared with a future Rust runner: the YAML corpus is consumed by both a
  Python pytest runner and a Rust `conformance-runner` crate.
- Makes it trivial to answer "are we compliant with §9.6 of 4791?" by
  pointing at `tests/rfc/rfc4791/test_9_6_calendar_query.py`.

#### Tier-1 — Protocol-level suites (cross-RFC scenarios)

- The `ccs-caldavtester` XML scripts (1,221 files) are the canonical
  Tier-1 corpus.
- The **driver** is modernized from py2/Twisted-16 to Py3.12 + pytest +
  treq. The XML scripts are unchanged (they are data).
- Wraps with `pytest` + `pytest-twisted` so it drops into modern CI.
- Preserves years of accumulated scenario knowledge (Apple push, sharing,
  attachments, etc.) without re-deriving it.

#### Tier-2 — End-to-end interop suites

- Run the server under Twisted in-process for fast feedback.
- Spin up a real HTTP server (in a Docker container or a `treq`-against-
  real-port test) for transport-level reality (HTTP/1.1, HTTP/2, TLS).
- Borrow **vdirsyncer's** test matrix and run it as a nightly CI job
  against the running server — it's the de-facto cross-server conformance
  check. Clone `https://github.com/pimutils/vdirsyncer` `tests/system/`.
- Borrow **`icalendar`** (Python PyPI) library's test corpus of edge-case
  `.ics` files for parser golden tests.

#### Tier-3 — Property-based / fuzz testing

- Library: **Hypothesis** (https://hypothesis.readthedocs.io/).
- Strategy: build a `Calendar` generator from the RFC 5545 ABNF using
  Hypothesis composites. Reuse the `icalendar` library's internal builders
  if helpful.
- Invariants to check:
  - parse → serialize → parse identity (modulo property ordering).
  - DTSTART/DTEND/DTEND-with-DURATION normalization.
  - RRULE expansion determinism for bounded ranges.
  - TZID resolution consistency.
  - xCal (RFC 6321) and jCal (RFC 7265) round-trips produce semantically
    equivalent objects.
- Same approach for vCard 4.0 against RFC 6350 + jCard (RFC 7095).
- Fuzzing pass with **`atheris`** (Google, https://github.com/google/atheris)
  on the byte-level parser to find panics/exceptions. Nightly CI job with
  corpus caching.

### Shared tests across Python and future Rust implementations

- Express each RFC test as **data + expectation** (a YAML/JSON file:
  request, response, precondition, postcondition). This is exactly what
  `ccs-caldavtester` already does for Tier-1.
- Build two thin runners: one Python (pytest), one Rust (a
  `conformance-runner` crate in phase 3). Both consume the same YAML
  corpus.
- For parser-level tests (iCalendar/vCard), produce a shared golden corpus
  of `.ics`/`.vcf` files with expected parse trees (as JSON), and have each
  implementation compare against the JSON.

### pytest + twisted.trial, or pure trial?

**pytest as the entrypoint, with `pytest-twisted` (which uses
`twisted.trial` underneath) for async.** Specifically:

- Use `@pytest_twisted.inlineCallbacks` for tests that need Deferreds.
- For pure-asyncio code paths, use `defer.ensureDeferred(coro)` inside a
  `@inlineCallbacks` test, or `twisted.internet.defer.fromCoroutine`
  (PEP 492 / 3156 interop; see ADR-0007).
- Pure `trial` is fine but gives less ergonomic parametrization, fixtures,
  and reporting than pytest. Since the migration target includes
  Hypothesis + JSON test data, pytest is the better host.
- This matches what Radicale, vdirsyncer, and most modern Twisted-using
  projects do.

### CI integration (GitHub Actions)

7 jobs:

| Job | Purpose | Trigger |
|---|---|---|
| `lint` | `ruff check` + `ruff format --check` + `mypy` | every push |
| `unit-py-3.12` | Tier-0 RFC tests on Python 3.12 | every push |
| `unit-py-3.13` | Tier-0 RFC tests on Python 3.13 | every push |
| `unit-py-3.14` | Tier-0 RFC tests on Python 3.14 (best-effort) | every push |
| `protocol` | Tier-1 ccs-caldavtester modernized suite, against in-process Twisted server | every push |
| `interop` | Tier-2: spin up server in container, run vdirsyncer matrix | nightly |
| `property` | Tier-3: Hypothesis round-trip tests + nightly `atheris` fuzz | nightly + on tag |

Use `uv` for environment management and `ruff` for formatting/linting
(ADR-0006). Cache the IETF RFC downloads and errata snapshots as a CI
cache key so the `rfc/` folder is reproducible.

### RFC reference library

A `rfc/` folder in the repo root contains downloaded RFCs organized by
category. A `rfc/fetch.sh` script downloads ~80 RFCs from
`rfc-editor.org` plus errata snapshots for MUST-tier RFCs and active IETF
drafts. See [RFC_COMPLIANCE.md](../audit/RFC_COMPLIANCE.md) §1.19 for the
full folder layout and manifest.

**Follow-up**: write `rfc/fetch.sh` + `rfc/manifest.txt` (noted in
[MIGRATION_PLAN.md](../audit/MIGRATION_PLAN.md) as a phase-1 follow-up).

## Consequences

### Positive

- Preserves the 1,221-script scenario corpus (years of edge-case coverage).
- Modern driver (Py3.12 + pytest + treq).
- RFC-anchored traceability (Tier-0: `tests/rfc/rfcNNNN/`).
- Shared YAML corpus for future Rust runner (phase 3).
- Property-based testing catches round-trip bugs.
- Fuzz testing catches parser panics.
- vdirsyncer interop matrix provides cross-server validation.
- pytest ergonomics (parametrization, fixtures, reporting).

### Negative

- Some effort to modernize the ccs-caldavtester driver (7k LOC).
- Two test tiers to maintain (Tier-0 and Tier-1 serve different purposes:
  Tier-0 is RFC-section-anchored; Tier-1 is cross-RFC scenarios).
- Hypothesis tests can be flaky if not seeded. Mitigation: use
  `--hypothesis-seed` in CI for replay; cache failing examples in
  `.hypothesis/examples/`.

### Neutral

- The `ccs-caldavtester` XML scripts remain the canonical Tier-1 corpus;
  the driver is the only part that changes.

### Risks

- **ccs-caldavtester driver modernization breaks the corpus**: the XML
  scripts expect specific driver behavior. Mitigation: modernize the driver
  incrementally; run the full corpus against the modernized driver; fix
  any regressions before merging.
- **vdirsyncer matrix is slow**: running the full vdirsyncer test suite
  against the server may take 30+ minutes. Mitigation: run nightly, not on
  every push; parallelize with `pytest-xdist`.
- **Hypothesis flakiness**: property-based tests may find new failing
  examples over time. Mitigation: commit `.hypothesis/examples/` to the
  repo so failing examples are replayed in every CI run.

### Follow-ups

- Modernize `ccs-caldavtester` driver to Py3.12 + pytest + treq.
- Create `tests/rfc/` skeleton with `rfcNNNN/` directories.
- Write `rfc/fetch.sh` + `rfc/manifest.txt`.
- Clone vdirsyncer `tests/system/` as Tier-2 interop matrix.
- Set up GitHub Actions CI with 7 jobs.
- Add Hypothesis strategies for iCalendar/vCard.
- Add `atheris` fuzzing job (nightly).
- Write a thin Rust `conformance-runner` crate in phase 3 that consumes
  the same YAML corpus.

## References

- `ccs-caldavtester/` (1,221 XML scripts + 995 fixtures + 7k LOC py2 driver)
- [RFC_COMPLIANCE.md](../audit/RFC_COMPLIANCE.md) §1.19 — RFC folder layout
- [RFC_COMPLIANCE.md](../audit/RFC_COMPLIANCE.md) §3 — compliance testing
  approach (full research)
- Hypothesis: https://hypothesis.readthedocs.io/
- `atheris`: https://github.com/google/atheris
- vdirsyncer: https://github.com/pimutils/vdirsyncer
- `icalendar` (Python): https://github.com/collective/icalendar
- `pytest-twisted`: https://github.com/pytest-dev/pytest-twisted
- [ADR-0003](0003-mission-py3-revival-as-typed-contract.md) — phase 3 Rust
  port (shared conformance corpus)
- [ADR-0006](0006-pyproject-toml-uv-ruff.md) — pytest + pytest-twisted
- [ADR-0007](0007-twisted-16-to-24-upgrade.md) — asyncio reactor for
  `defer.fromCoroutine` in tests
