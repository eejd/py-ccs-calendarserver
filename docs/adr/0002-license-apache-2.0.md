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

# ADR-0002: License as Apache-2.0; vendored Apache/MIT deps only

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

The original `ccs-calendarserver` is licensed under Apache License 2.0
(`py-ccs-calendarserver/LICENSE.txt:1-295`, copyright "Copyright (c) 2005-2017
Apple Inc."). We are reviving the project and must decide on a license for the
revived codebase.

This decision gates whether we can directly reuse code from the two Rust
reference implementations in the workspace:

- **`rs-rustical`** (RustiCal): AGPL-3.0-or-later
  (`rs-rustical/Cargo.toml:11`). Single maintainer; hobby project.
- **`rs-stalwart`** (Stalwart): AGPL-3.0-only OR Stalwart Enterprise License
  v2 (SELv2) — dual-licensed (`rs-stalwart/README.md:174-181`).
  Commercially backed.

Both are AGPL-licensed, which is a strong copyleft license. If we adopt AGPL,
we could potentially reuse code from these projects. If we stay Apache-2.0, we
can only reference their architecture (not copyrightable in the same way) and
adopt permissively-licensed dependencies.

The key permissively-licensed Rust dependency candidate is **`calcard`**
(from Stalwart Labs, published separately on crates.io), which is
**Apache-2.0 OR MIT** licensed. It is a complete iCalendar/vCard/JSCalendar/
JSContact parser with recurrence expansion and fuzzing. See
`rs-stalwart/crates/dav-proto/Cargo.toml:11` and the `calcard` crate on
crates.io.

## Alternatives Considered

### Option A — Apache-2.0 (keep original license)

Preserve the original ccs-calendarserver Apache-2.0 license. Cannot reuse AGPL
code from RustiCal/Stalwart, but CAN adopt `calcard` (Apache/MIT) via PyO3 and
reference AGPL projects for architecture only (patterns are not
copyrightable).

**Pros**: Maximizes downstream adoption (enterprises, cloud providers, and
proprietary forks can all use it); preserves compatibility with the original
Apple license; `calcard` (the best Rust parser) is available; aligns with
the broader Python ecosystem norm.
**Cons**: Cannot directly copy code from RustiCal or Stalwart's AGPL-licensed
crates; must rely on architecture reference + permissively-licensed deps.

### Option B — AGPL-3.0-only

Adopt AGPL-3.0-only for the revived server. Would allow studying/borrowing
patterns from RustiCal and Stalwart (though pattern study is
license-independent). Could potentially reuse their code directly if we
accept AGPL obligations.

**Pros**: Can directly reuse AGPL code from RustiCal/Stalwart; strong
copyleft ensures all modifications remain open.
**Cons**: Restricts proprietary redistribution; many enterprises and cloud
providers will not touch AGPL; diverges from the original Apple license;
the actual code-reuse benefit is limited because both Rust projects are
tightly coupled to their own internal crates (Stalwart's `dav` depends on
`store`/`trc`/`types`/`rkyv`; RustiCal's `rustical_dav` is coupled to axum
and its own XML derive layer). The reusable surface is small even with
license compatibility.

### Option C — Dual: Apache-2.0 server + vendored Apache deps

Apache-2.0 for the revived server; pull in `calcard` (Apache/MIT) as a PyO3
dependency. Reference AGPL projects for architecture only, no code reuse.

**Pros**: All of Option A's benefits, plus explicit policy that vendored
deps MUST be Apache-2.0 or MIT (or similarly permissive). Maximizes
compatibility; `calcard` available; clear dependency policy.
**Cons**: Same as Option A — no direct code reuse from AGPL projects.

## Decision

We adopt **Option C: Apache-2.0 for the revived server, with a policy that
all vendored dependencies MUST be under Apache-2.0, MIT, BSD, or similarly
permissive licenses.** No AGPL-licensed code may be copied into the
repository.

### Policy

- The revived `py-ccs-calendarserver` is licensed under **Apache License 2.0**,
  preserving the original Apple license.
- All third-party dependencies MUST be under permissive licenses
  (Apache-2.0, MIT, BSD-2/3-Clause, ISC, MPL-2.0). AGPL/GPL dependencies are
  prohibited.
- We MAY reference the architecture and patterns of AGPL-licensed projects
  (RustiCal, Stalwart) for inspiration, since patterns are not copyrightable.
  We MUST NOT copy their source code.
- We MAY adopt `calcard` (Apache-2.0/MIT) as a PyO3 dependency in phase 3,
  subject to conformance parity evaluation (see
  [ADR-0010](0010-port-ccs-pycalendar-to-py3.md)).
- The `LICENSE.txt` file MUST be preserved. New files MUST include the
  standard Apache-2.0 header.

## Consequences

### Positive

- Maximizes downstream adoption: enterprises, cloud providers, and
  proprietary forks can all use the revived server.
- Preserves compatibility with the original Apple ccs-calendarserver license.
- `calcard` (the best Rust iCalendar/vCard parser) is available for future
  PyO3 adoption.
- Clear dependency licensing policy; no AGPL compliance burden.
- Aligns with the broader Python/Twisted ecosystem norm (Twisted, zope.interface,
  cryptography, psycopg are all permissively licensed).

### Negative

- Cannot directly copy code from RustiCal or Stalwart's AGPL-licensed crates.
  Must rely on architecture reference + permissively-licensed deps.
- The reusable surface from RustiCal/Stalwart is small even with license
  compatibility (both are tightly coupled to their internal crates), so this
  is a minor loss.

### Neutral

- The `calendarserver:` Apple extensions and `X-APPLE-*` properties are
  Apple-origin and not separately licensed; they remain under the
  Apache-2.0 umbrella of the original codebase.

### Risks

- **Accidental AGPL contamination**: a contributor copies code from RustiCal
  or Stalwart into the repo. Mitigation: CONTRIBUTING.md MUST state the
  licensing policy; CI SHOULD run a license scanner (`pip-audit` / `reuse`).
- **`calcard` license change**: if Stalwart Labs changes `calcard`'s license
  to AGPL in a future version, we MUST pin to the last Apache/MIT version.
  Mitigation: pin the version in `pyproject.toml`; verify license before each
  upgrade.

### Follow-ups

- Write `CONTRIBUTING.md` with the licensing policy and the Apache-2.0 header
  template.
- Run a license scan of the existing dependency tree (`pip-audit` /
  `reuse lint`) in phase 1.
- Verify `calcard`'s current license on crates.io before phase 3 PyO3
  adoption.

## References

- Original license: `py-ccs-calendarserver/LICENSE.txt:1-295`
- RustiCal license: `rs-rustical/Cargo.toml:11` (AGPL-3.0-or-later)
- Stalwart license: `rs-stalwart/README.md:174-181` (AGPL-3.0-only OR SELv2)
- `calcard` on crates.io: https://crates.io/crates/calcard (Apache-2.0 OR MIT)
- Apache License 2.0: https://www.apache.org/licenses/LICENSE-2.0
- SPDX license identifiers: https://spdx.org/licenses/
