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

# RFCCalServ — py-ccs-calendarserver Revival

## Mission

Revive Apple's archived `ccs-calendarserver` (Python 2.7, last touched 2020) as
`py-ccs-calendarserver` under Apache-2.0, bringing it to Python 3.12+ (aiming
3.14) with strict mypy typing, modern dependencies, and current (August 2026)
RFC compliance. The updated, strictly-typed Python codebase then serves as a
**contract and reference source** for a future Rust + PyO3 reimplementation of
core libraries.

The original Apple work was considered one of the most standards-compliant open
source CalDAV/CardDAV implementations. That pedigree is the reason to build on
the stale project rather than start greenfield.

## Three goals, in order

1. **Primary**: Evaluate the state of the code and create an initial audit of
   the needed work, clear initial targets for dependency replacement, and
   Python migration challenges. Migrate all core libraries to strictly-typed
   wherever we can do this immediately to improve development cycle
   reliability, catch problems as we migrate to Python 3.14 benefiting from
   the stricter checks, and prepare the way for the future Rust (strictly
   typed) library migrations.

2. **Secondary**: Expand the existing codebase to fully implement all current
   (Aug 2026) related RFCs to bring the infrastructure up to current standards
   compliance.

3. **Long-term**: Use the updated and validated Python codebase as a contract
   and reference source to implement a Rust + PyO3 implementation of all core
   libraries. At that time, consider refactoring or creating new API seams to
   separate and improve any architectural decisions and facilitate scaling up
   for performance or any other identified needs.

## Project principles

| Area | Decision |
|---|---|
| License | Apache-2.0 server; vendored Apache/MIT deps only. No AGPL code reuse from RustiCal/Stalwart. `calcard` (Apache-2.0/MIT) eligible for future PyO3 adoption. |
| Language | Python 3.12+ (aiming 3.14); drop Python 2 entirely; no `six` compat layer. |
| Typing | mypy strict with per-module overrides; `py.typed` markers for all packages; type the contract layer (PyO3 candidates) in parallel with the Py3 port. |
| Parser | Port `ccs-pycalendar` to Py3 as the canonical contract; preserve lenient fix-up behavior and cached RRULE engine. Defer `calcard` PyO3 evaluation to phase 3 behind a conformance parity gate. |
| Phase-1 scope | Balanced: Py3 port + security fixes + key modern RFCs (OAuth, RFC 7616, RFC 9110 audit, RFC 6764, modern APNs HTTP/2, drop iSchedule). |
| Phase-2 scope (planning only) | Aggressive: HTTP/2 server, Web Push, RFC 8607 managed attachments, RFC 7986/9070 iCal properties, xCal/xCard, DKIM Ed25519, JMAP. |
| Apple extensions | Keep if used in macOS 26/27 (research confirms most are); segregate into an `apple_extensions/` module behind a capability flag so the server is deployable as "standards-only" or "standards + Apple". |
| Storage | PostgreSQL (primary) + SQLite (dev/test); drop Oracle, OpenDirectory. Modernize LDAP via `ldap3` (pure-Python). Keep XML directory as dev default. |
| Auth | Add OAuth 2.0/OIDC Bearer (RFC 6750/6749) as primary; keep Basic (RFC 7617) and Digest (RFC 7616 SHA-256) as deprecated legacy. Make Kerberos optional via `gssapi`. |
| TLS | pyOpenSSL + cryptography + service_identity on all platforms; drop pySecureTransport/OSXFrameworks; enforce TLS 1.2+/1.3 per RFC 9325. |
| Config | Support both plist and TOML via a format-agnostic loader; pydantic/TypedDict strict schema overlay. |
| Tests | Modernize `ccs-caldavtester` driver to Py3.12 + pytest + treq; preserve 1,221 XML scripts; add RFC-anchored Tier-0 YAML suite; add Hypothesis + atheris. |
| Tooling | `pyproject.toml` (PEP 621) + `uv lock`; `ruff` (replaces pyflakes); `pytest` + `pytest-twisted`. |

## Phase roadmap

### Phase 1 — Revival (3-6 months)

**Goal**: A working Python 3.12+ server, strictly-typed contract layer,
modern dependencies, security fixes, key modern RFCs.

- Python 2 → 3.12+ port of `py-ccs-calendarserver`, `ccs-pycalendar`,
  `ccs-twistedextensions`, `ccs-caldavtester`.
- Twisted 16.6 → 24.x upgrade; `returnValue`→`return`, `implements`→`@implementer`.
- Replace `pycrypto` → `cryptography`; drop `pySecureTransport`/`OSXFrameworks`.
- Replace `python-ldap` → `ldap3`; drop Oracle/OpenDirectory.
- Add OAuth 2.0/OIDC Bearer auth; add RFC 7616 Digest SHA-256.
- RFC 9110 audit; RFC 6764 SRV; modern APNs HTTP/2; drop iSchedule.
- Strict typing of contract layer: `ccs-pycalendar`, `txdav/xml/*`,
  `twistedcaldav/config.py`, `txdav/common/datastore/sql_tables.py`,
  `txdav/idav.py`.
- Modernize `ccs-caldavtester` driver; RFC reference library (`rfc/`).
- Segregate Apple extensions into `apple_extensions/` module.
- `pyproject.toml` + `uv lock` + `ruff` + GitHub Actions CI.

### Phase 2 — Modernization (planning, TBD)

**Goal**: Full 2026 RFC compliance.

- HTTP/2 server (RFC 9113); possibly HTTP/3 (RFC 9114).
- Web Push (RFC 8030/8291/8292); WebDAV-Push I-D for DAVx5-class clients.
- RFC 8607 managed attachments.
- RFC 7986 iCalendar properties; RFC 9070 VVENUE.
- xCal (RFC 6321) / xCard (RFC 6351) XML content-type support.
- DKIM Ed25519 (RFC 8463) for iMIP.
- `pg8000` → `psycopg 3` migration.
- `txweb2` HTTP channel → `twisted.web` migration.
- Evaluate JMAP for Calendars (draft-ietf-jmap-calendars).

### Phase 3 — Rust + PyO3 (long-term)

**Goal**: Reimplement core libraries in Rust, using the typed Python
codebase as the contract.

**Rust port candidates (in order)**:
1. `ccs-pycalendar.recurrence` / `datetime` — pure logic, hot path, no deps.
2. `twext.enterprise.dal` (syntax + model + parseschema + record) — 4.4k LOC SQL DSL.
3. `txdav/xml/base.py` + `txdav/xml/rfc*.py` — 506 element classes.
4. `twistedcaldav/caldavxml.py` / `carddavxml.py` / `customxml.py` — 98 element classes.
5. `txdav/common/datastore/sql_tables.py` + `sql_schema/*.sql` — schema.
6. `twistedcaldav/dateops.py` — date arithmetic.

**Keep as Python (too Twisted-entangled)**:
- `txweb2/channel/http.py`, `txweb2/http.py`, `txweb2/server.py` — replace wholesale with Rust `hyper`/`axum`.
- `twistedcaldav/resource.py`, `txweb2/dav/resource.py` — DAV resource tree.
- `twistedcaldav/storebridge.py` (3,934 LOC) — adapter; disappears in unified Rust server.
- `twisted/plugins/caldav.py`, `calendarserver/tap/caldav.py` — Twisted plumbing.
- `twistedcaldav/stdconfig.py` + `config.py` — replace with `serde`-deserialised TOML in Rust.

## Architecture decisions

Architectural decisions are recorded as **ADRs** (Architecture Decision
Records) in [`docs/adr/`](docs/adr/). See [`docs/adr/INDEX.md`](docs/adr/INDEX.md)
for the decision log.

**Core ADRs (15)** — the high-impact, high-reversibility-cost decisions:

| # | Title | Status |
|---|---|---|
| [0001](docs/adr/0001-adopt-adr-methodology.md) | Adopt ADR methodology | Accepted |
| [0002](docs/adr/0002-license-apache-2.0.md) | License: Apache-2.0 | Accepted |
| [0003](docs/adr/0003-mission-py3-revival-as-typed-contract.md) | Mission: Py3 revival as typed contract for Rust+PyO3 | Accepted |
| [0004](docs/adr/0004-target-python-3.12-drop-py2.md) | Target Python 3.12+; drop Python 2 | Accepted |
| [0005](docs/adr/0005-mypy-strict-per-module-rollout.md) | mypy strict with per-module rollout | Accepted |
| [0006](docs/adr/0006-pyproject-toml-uv-ruff.md) | pyproject.toml + uv + ruff + pytest | Accepted |
| [0007](docs/adr/0007-twisted-16-to-24-upgrade.md) | Twisted 16.6 → 24.x upgrade | Accepted |
| [0008](docs/adr/0008-keep-txweb2-forked-phase-1.md) | Keep txweb2 forked for phase 1 | Accepted |
| [0009](docs/adr/0009-standardize-tls-pyopenssl-cryptography.md) | Standardize TLS on pyOpenSSL+cryptography | Accepted |
| [0010](docs/adr/0010-port-ccs-pycalendar-to-py3.md) | Port ccs-pycalendar to Py3 as canonical contract | Accepted |
| [0011](docs/adr/0011-postgresql-sqlite-only-drop-oracle-opendirectory.md) | PostgreSQL + SQLite only; drop Oracle/OpenDirectory | Accepted |
| [0012](docs/adr/0012-oauth-oidc-primary-auth.md) | OAuth/OIDC primary auth; legacy Basic/Digest deprecated | Accepted |
| [0013](docs/adr/0013-plist-and-toml-config-with-pydantic-schema.md) | Plist + TOML config with pydantic schema | Accepted |
| [0014](docs/adr/0014-segregate-apple-extensions.md) | Segregate Apple extensions behind capability flag | Accepted |
| [0015](docs/adr/0015-compliance-testing-architecture.md) | Compliance testing architecture | Accepted |

## Deferred decisions

The following decisions are documented here as one-liners with rationale
pointers. They are **not** elevated to full ADRs yet. Any challenged or
revisited decision SHOULD be promoted to a full ADR via the supersede
process described in [ADR-0001](docs/adr/0001-adopt-adr-methodology.md).

### Language & tooling

- **asyncio reactor**: Adopt `twisted.internet.asyncioreactor` as the default
  for forward asyncio interop (PEP 3156). Covered by ADR-0007.
- **ruff/uv/pytest-twisted**: Specific tool choices. Covered by ADR-0006.

### Framework & I/O

- **pg8000 → psycopg 3 sequencing**: Upgrade `pg8000` to 1.31 for phase 1
  port; migrate to `psycopg 3` in phase 2 for performance. See
  [DEPENDENCIES.md](docs/audit/DEPENDENCIES.md).
- **pytz → stdlib zoneinfo**: Replace `pytz 2016.7` with stdlib `zoneinfo`
  + `tzdata`; retire `ccs-pycalendar/zonal/` packaging tooling. Mechanical;
  noted in [DEPENDENCIES.md](docs/audit/DEPENDENCIES.md).
- **DALE DAL keep-vs-replace**: Keep DALE DAL (`twext.enterprise.dal`) for
  phase 1; evaluate replacement with SQLAlchemy Core or Rust `sqlx` in phase
  3. See [ADR-0010](docs/adr/0010-port-ccs-pycalendar-to-py3.md) and
  [TYPING_PLAN.md](docs/audit/TYPING_PLAN.md).
- **Kerberos via `gssapi`**: Make Kerberos/SPNEGO optional via `gssapi` PyPI
  adapter; drop `ccs-pykerberos` C extension as default. Covered by ADR-0012.
- **TLS best practices / HSTS**: Enforce TLS 1.2+/1.3 per RFC 9325; add HSTS
  (RFC 6797). Covered by ADR-0009.

### Protocol & RFCs

- **Phase-1 RFC scope details**: RFC 9110 audit, RFC 7616 Digest SHA-256,
  RFC 6764 SRV, modern APNs HTTP/2, drop iSchedule. See
  [RFC_COMPLIANCE.md](docs/audit/RFC_COMPLIANCE.md).
- **Phase-2 RFC deferrals**: HTTP/2 server (RFC 9113), Web Push (RFC
  8030/8291/8292), RFC 8607 managed attachments, RFC 7986/9070 iCal
  properties, xCal/xCard (RFC 6321/6351), DKIM Ed25519 (RFC 8463), JMAP for
  Calendars. See [RFC_COMPLIANCE.md](docs/audit/RFC_COMPLIANCE.md).
- **iSchedule drop**: Drop iSchedule (never standardized; last draft
  `draft-desruisseaux-ischedule-05`). Rely on RFC 6638 + iMIP for
  server-to-server scheduling. See
  [RFC_COMPLIANCE.md](docs/audit/RFC_COMPLIANCE.md).
- **APNs HTTP/2 modernization**: Migrate APNs from legacy binary protocol
  (dead 2021) to HTTP/2 provider API; add WebDAV-Push I-D for DAVx5-class
  clients. See [RFC_COMPLIANCE.md](docs/audit/RFC_COMPLIANCE.md).
- **Apple extension tier classification details**: Tier 1 core / Tier 2
  segregated / Tier 3 round-trip-only. See
  [ADR-0014](docs/adr/0014-segregate-apple-extensions.md) appendix.

### Testing

- **Hypothesis/atheris specifics**: Hypothesis strategies from RFC 5545/6350
  ABNF; `atheris` fuzz on byte-level parser. Covered by ADR-0015.
- **GitHub Actions CI**: 7 jobs — lint (ruff+mypy), unit-py-3.12/3.13/3.14,
  protocol (Tier-1), interop (Tier-2), property (Tier-3), nightly-fuzz.
  Covered by ADR-0015.

### Rust migration runway

- **`calcard` as Rust reference**: Use `calcard` (Apache-2.0/MIT, from
  Stalwart) as the reference Rust iCalendar/vCard parser if/when PyO3
  adoption is evaluated. See
  [ADR-0010](docs/adr/0010-port-ccs-pycalendar-to-py3.md).
- **RustiCal architecture template**: Use RustiCal's trait-based `Resource`
  WebDAV architecture as the template for a future typed Rust WebDAV layer.
  See `rs-rustical/crates/dav/src/resource/mod.rs:39`.
- **Stalwart `dav-proto/schema` as XML vocabulary checklist**: Use
  Stalwart's `crates/dav-proto/src/schema/mod.rs` (1484 lines enumerating
  every DAV/CalDAV/CardDAV/CalendarServer XML element) as the canonical
  checklist for the typed contract layer. See
  `rs-stalwart/crates/dav-proto/src/schema/mod.rs`.
- **Rust port candidate ordering**: See Phase 3 roadmap above and
  [TYPING_PLAN.md](docs/audit/TYPING_PLAN.md) §10.

### Process

- **IETF WG mailing list subscriptions**: Subscribe to CALEXT, JMAP,
  MAILMAINT IETF WG mailing lists; track 7 active drafts. See
  [RFC_COMPLIANCE.md](docs/audit/RFC_COMPLIANCE.md) §1.18.

## Reference documents

| Document | Purpose |
|---|---|
| [AUDIT.md](docs/audit/AUDIT.md) | Full state-of-code audit (codebase inventory, Py2 idioms, dependency state, RFC compliance, typing readiness, sibling repos, Rust references). |
| [DEPENDENCIES.md](docs/audit/DEPENDENCIES.md) | Per-dependency replacement table with risk/priority/modern alternatives. |
| [RFC_COMPLIANCE.md](docs/audit/RFC_COMPLIANCE.md) | Per-RFC scorecard (implemented/where/gaps/errata). |
| [TYPING_PLAN.md](docs/audit/TYPING_PLAN.md) | mypy.ini target shape, phased rollout, stub strategy. |
| [MIGRATION_PLAN.md](docs/audit/MIGRATION_PLAN.md) | Step-by-step Py3 migration sequence with file/LOC estimates. |

## Origin

This project is a revival of Apple's archived `ccs-calendarserver`:
- Original repo: https://github.com/apple/ccs-calendarserver (archived 2020)
- License: Apache License 2.0 (preserved)
- Last Apple commit: 2020-02-12
- Original copyright: Copyright (c) 2005-2017 Apple Inc.

The Apple-specific `calendarserver:` extensions and `X-APPLE-*` / `X-CALENDARSERVER-*`
iCalendar properties are preserved for client compatibility (see
[ADR-0014](docs/adr/0014-segregate-apple-extensions.md)) but segregated so the
server can be deployed in a "standards-only" configuration.
