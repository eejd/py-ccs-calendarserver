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

# ADR-0010: Port ccs-pycalendar to Py3 as canonical contract; defer calcard

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

`ccs-pycalendar` (124 .py files, 25.9k LOC) is the iCalendar/vCard parsing
and recurrence engine for the entire server. It is a **critical runtime
dependency** — 81 files in `py-ccs-calendarserver` import `pycalendar`,
spanning `twistedcaldav/ical.py`, `twistedcaldav/dateops.py`,
`twistedcaldav/timezones.py`, `twistedcaldav/method/report_freebusy.py`,
`calendarserver/tools/{calverify,managetimezones,anonymize,purge}.py`, and
more. See [`ccs-pycalendar`](../../GOAL.md) and
[AUDIT.md](../audit/AUDIT.md) §3 for the full inventory.

### Why ccs-pycalendar is bespoke and valuable

- **Lenient fix-up parser** (`parser.py:11-83`): `ParserContext` is an
  error-policy configuration (allow/ignore/fix/raise) for malformed input.
  Apple's clients produce a lot of malformed iCalendar, so the parser is
  deliberately lenient and has Fix-modes for: escaped colons, blank lines,
  vCard 2.x BASE64, invalid DATE-TIME values, leading-space years, invalid
  DURATION, over-long ADR/N values, malformed REQUEST-STATUS, backslash-
  escaping in URI/GEO values. **This lenient behavior is load-bearing** —
  any replacement MUST preserve the fix-up behaviors or existing clients
  will break.
- **Cached RRULE expansion** (`recurrence.py:838-887`): `Recurrence.expand()`
  caches results across calls — important for free/busy queries that
  re-query the same event. The 1,670 LOC of `recurrence.py` encode subtle
  RFC 5545 edge cases (negative BYMONTHDAY, BYDAY with ordinals like -1MO,
  WKST interactions, BYSETPOS over BYxxx combinations). **This is the
  hardest part of iCalendar and this implementation is battle-tested.**
- **Custom DateTime class** (`datetime.py`, 1,133 LOC): reimplements date
  arithmetic that py3's stdlib `datetime` + `zoneinfo` does natively.
- **Embedded VTIMEZONE database** (`timezonedb.py`, `zonal/`): packages
  Olson tzdata as VTIMEZONE components. Modern replacement: stdlib
  `zoneinfo` (Py3.9+) + `tzdata` package.
- **`distutils.core.setup`** in `setup.py:17` — removed in Py3.12+, gone in
  3.14. This is a blocker.
- **Implicit relative imports** throughout `__init__.py:19-53`:
  `import binaryvalue`, `import icalendar.recurrencevalue` — py2-only,
  break completely on Py3 (need `from . import binaryvalue`).

### The calcard alternative

**`calcard`** (from Stalwart Labs, published separately on crates.io) is a
Rust crate (v0.3.9) licensed under **Apache-2.0 OR MIT** (permissively
licensed, safe for our Apache-2.0 project per ADR-0002). It provides:

- iCalendar, vCard, JSCalendar, JSContact parsing.
- Bidirectional conversion (iCalendar↔JSCalendar, vCard↔JSContact).
- Recurrence-rule expansion.
- IANA timezone detection.
- Fuzzing harness (`cargo-fuzz`, MIRI-tested).
- Actively maintained (Stalwart Labs, commercial backing).

See `rs-stalwart/crates/dav-proto/Cargo.toml:11` and
`rs-stalwart/crates/groupware/src/calendar/mod.rs:36-49`.

**Risk**: `calcard`'s leniency profile differs from `ccs-pycalendar`'s.
Real-world Apple/DAVx5 clients produce malformed data that ccs-pycalendar
currently fixes up. `calcard` follows Postel's law ("best effort to parse
non-conformant objects") but the specific fix-up behaviors may not match.
A conformance parity gate is needed before any switch.

## Alternatives Considered

### Option A — Port ccs-pycalendar to Py3 first; evaluate calcard later

Mechanical Py3 port of ccs-pycalendar (distutils→setuptools, implicit→absolute
imports, cStringIO→io, cElementTree→ElementTree). Preserve the lenient
fix-up parser and cached RRULE engine exactly. Later, port `recurrence.py`
to Rust. Lowest risk to existing client compat.

**Pros**: Lowest risk; preserves battle-tested behavior; the parser is the
contract for the Rust port; ccs-pycalendar has excellent test coverage
(36 test files, 9.7k LOC) that serves as the conformance gate.
**Cons**: Does not benefit from `calcard`'s fuzzing and JSCalendar/JSContact
support in the short term; the custom `DateTime` class and `zonal/` tooling
are redundant with stdlib `zoneinfo`.

### Option B — Adopt calcard via PyO3 early; retire ccs-pycalendar

Write a PyO3 wrapper around `calcard`; expose `ICalendar::parse`,
`VCard::parse`, `into_jscalendar`, `into_jscontact`, `to_string` to Python.

**Pros**: Modern, fuzzed parser; JSCalendar/JSContact support; Rust
performance; eliminates 25.9k LOC of Python to maintain.
**Cons**: `calcard`'s leniency profile differs; real-world clients produce
malformed data ccs-pycalendar currently fixes up; need a compat shim; high
risk of breaking existing clients; the ccs-pycalendar test suite (9.7k LOC)
would need to pass against the `calcard` wrapper, which is non-trivial.

### Option C — Hybrid: port ccs-pycalendar to Py3 as contract; wrap calcard in parallel

Run both during migration: ccs-pycalendar remains source of truth; add a
calcard-backed implementation behind a flag; use the conformance suite to
verify behavioral parity before switching.

**Pros**: Safest cutover; preserves ccs-pycalendar as the contract; calcard
evaluated against real test corpus; can fall back if parity is not reached.
**Cons**: Most work; two parsers to maintain during the evaluation period.

## Decision

We adopt **Option A for phase 1: port ccs-pycalendar to Py3 as the canonical
contract. Defer Option C (hybrid calcard evaluation) to phase 3, after the
Py3 port is stable and the conformance suite is modernized.**

### Phase 1 policy

- ccs-pycalendar MUST be ported to Python 3.12+ as the canonical iCalendar/
  vCard parser contract.
- The lenient fix-up parser behavior MUST be preserved exactly. The
  `ParserContext` error-policy state machine (`parser.py:11-83`) is
  load-bearing.
- The cached RRULE expansion behavior MUST be preserved. The
  `Recurrence.expand()` caching (`recurrence.py:838-887`) is load-bearing.
- `setup.py:17` (`distutils.core.setup`) MUST be replaced with `setuptools`
  or `hatchling` via `pyproject.toml` (ADR-0006).
- Implicit relative imports (`__init__.py:19-53`) MUST be converted to
  absolute imports (`from . import binaryvalue`).
- `cStringIO` (13 files) MUST be replaced with `io.StringIO`/`io.BytesIO`
  (bytes-vs-str must be chosen carefully).
- `xml.etree.cElementTree` MUST be replaced with `xml.etree.ElementTree`.
- The `zonal/` package and `timezonedb.py` MAY be replaced with stdlib
  `zoneinfo` + `tzdata` package (deferred decision; see GOAL.md).
- The custom `DateTime` class (`datetime.py`, 1,133 LOC) MAY be partially
  replaced with stdlib `datetime` + `zoneinfo` in phase 2 (deferred; high
  risk).
- A `py.typed` marker MUST be added (ADR-0005); ccs-pycalendar is the first
  module to be strictly typed (TYPING_PLAN.md Phase 1).

### Phase 3 evaluation (deferred)

In phase 3, evaluate `calcard` (Apache-2.0/MIT) as a PyO3 replacement for
ccs-pycalendar's hot paths:

1. Write a PyO3 wrapper around `calcard` exposing `ICalendar::parse`,
   `VCard::parse`, `to_string`.
2. Run both parsers (ccs-pycalendar and calcard) against the full
   conformance suite (1,221 XML scripts + 9.7k LOC of ccs-pycalendar tests).
3. Identify leniency-parity gaps; write compat shims if needed.
4. If parity is reached, switch the hot paths (parse, serialize, recurrence
   expansion) to the calcard wrapper; keep ccs-pycalendar as a fallback
   behind a flag.
5. If parity is not reached, keep ccs-pycalendar and port `recurrence.py` +
   `datetime.py` to Rust directly (without calcard).

### What NOT to do

- Do NOT do a wholesale library swap to `icalendar` (PyPI) or `vobject` in
  phase 1. The combination of (a) lenient fix-up parser, (b) cached
  recurrence expansion, (c) embedded VTIMEZONE db, and (d) full
  read/write/round-trip serialization is unique to ccs-pycalendar. See
  [AUDIT.md](../audit/AUDIT.md) §3 "replacement library comparison".

## Consequences

### Positive

- Lowest risk to existing client compatibility.
- Preserves the battle-tested lenient parser and cached RRULE engine.
- ccs-pycalendar's excellent test coverage (36 files, 9.7k LOC) serves as
  the conformance gate.
- ccs-pycalendar becomes the typed contract for the Rust port (ADR-0005
  Phase 1 typing).
- `calcard` evaluation is deferred to phase 3, when the conformance suite
  is modernized and can serve as a parity gate.

### Negative

- Does not benefit from `calcard`'s fuzzing and JSCalendar/JSContact
  support in the short term.
- The custom `DateTime` class and `zonal/` tooling are redundant with
  stdlib `zoneinfo` (but replacing them is high-risk and deferred).
- 25.9k LOC of Python to maintain until phase 3.

### Neutral

- The phase-3 `calcard` evaluation may result in a full switch, a hybrid,
  or rejection. The decision is deferred with a clear evaluation process.

### Risks

- **Leniency behavior drift during Py3 port**: the bytes/str boundary
  changes in Py3 could subtly alter the parser's lenient behavior.
  Mitigation: run the full ccs-pycalendar test suite (9.7k LOC) against the
  Py3 port; add property-based round-trip tests (Hypothesis) per ADR-0015.
- **`calcard` license change**: if Stalwart Labs changes `calcard`'s
  license to AGPL in a future version, the phase-3 evaluation is blocked.
  Mitigation: pin the version; verify license before evaluation.
- **`calcard` API instability**: `calcard` is at v0.3.9; API may change
  before v1.0. Mitigation: pin the version; wrapper isolates Python from
  API changes.

### Follow-ups

- Port ccs-pycalendar to Py3 (distutils→setuptools, implicit→absolute,
  cStringIO→io, cElementTree→ElementTree). See
  [MIGRATION_PLAN.md](../audit/MIGRATION_PLAN.md).
- Add `py.typed` marker; begin strict typing (TYPING_PLAN.md Phase 1).
- Add property-based round-trip tests (Hypothesis) per ADR-0015.
- Phase 3: write `calcard` PyO3 wrapper; run conformance parity evaluation.

## References

- `ccs-pycalendar/setup.py:17` (`distutils.core.setup`)
- `ccs-pycalendar/src/pycalendar/__init__.py:19-53` (implicit relative imports)
- `ccs-pycalendar/src/pycalendar/parser.py:11-83` (ParserContext lenient parser)
- `ccs-pycalendar/src/pycalendar/icalendar/recurrence.py:838-887` (cached RRULE expansion)
- `ccs-pycalendar/src/pycalendar/datetime.py` (1,133 LOC custom DateTime)
- `ccs-pycalendar/src/pycalendar/timezonedb.py` (VTIMEZONE database)
- `calcard` on crates.io: https://crates.io/crates/calcard (Apache-2.0 OR MIT)
- `rs-stalwart/crates/dav-proto/Cargo.toml:11` (calcard dependency)
- `rs-stalwart/crates/groupware/src/calendar/mod.rs:36-49` (iCalendar coverage)
- [AUDIT.md](../audit/AUDIT.md) §3 — ccs-pycalendar deep audit
- [TYPING_PLAN.md](../audit/TYPING_PLAN.md) Phase 1 — ccs-pycalendar typing
- [ADR-0002](0002-license-apache-2.0.md) — Apache-2.0; calcard eligible
- [ADR-0005](0005-mypy-strict-per-module-rollout.md) — typing strategy
