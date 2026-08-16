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

# AUDIT — State of Code

**Snapshot date**: 2026-08-16
**Repository**: `py-ccs-calendarserver` (revived from Apple's archived
`ccs-calendarserver`, last commit 2020-02-12)

This document is a synthesized state-of-code audit covering: codebase
inventory, Python 2 idioms, dependency state, RFC compliance, typing
readiness, sibling repos, and Rust reference implementations. It is the
reference companion to the ADR set (`docs/adr/`).

---

## 1. Codebase inventory

| Metric | Count |
|---|---|
| `.py` files | 741 |
| Total LOC | ~323,724 |
| Test files (`test_*.py`) | 203 |
| Test LOC | ~131,319 (~40% of total) |
| `@inlineCallbacks` decorators | 3,263 |
| `returnValue(...)` calls | 1,826 |
| `yield` statements | 15,089 |
| `async def` usage | 0 |
| C extensions in main repo | 1 (`support/python-wrapper.c`, a launcher) |

### Subsystem breakdown

| Subsystem | .py files | LOC | Py2-idiom density | Notes |
|---|---|---|---|---|
| **txweb2** | 93 | 27,549 | High | Vendored fork of `twisted.web2`. HTTP/bytes layer. Highest risk for Py3 port. |
| **twistedcaldav** | 149 | 79,071 | Medium-high | Core CalDAV logic. Largest files: `storebridge.py` (3,934 LOC), `ical.py` (3,891), `resource.py` (2,954), `stdconfig.py` (1,893), `caldavxml.py` (1,600+). |
| **txdav** | 279 | 139,474 | Medium | Largest subsystem. DAL + CalDAV/CardDAV datastores. `sql.py` (5,031 LOC), `txdav/caldav/datastore/sql.py` (5,547). |
| **calendarserver** | 109 | 46,449 | Medium | Server tap, tools, push, webadmin. `tap/caldav.py` is the main entry point. |
| **contrib** | 103 | 25,959 | High but lower-priority | Performance/loadtest tooling. Candidate for deletion/rewrite. |
| **simplugin** | 3 | 3,291 | Medium | Simulation plugin; small. |
| **conf** | 1 | 1,129 | High | `conf/auth/generate_test_accounts.py`; test-data generator. |
| **bin** | 2 (.py) | small | Low | Shell wrappers need `python2.7` → `python3` updates. |

---

## 2. Python 2 idioms

### Summary counts

| Idiom | Files | Occurrences |
|---|---|---|
| `print` statement | 8 | 17 (+ 7 `print >>`) |
| `iteritems()` | 71 | 152 |
| `itervalues()` | 18 | 31 |
| `iterkeys()` | 11 | 19 |
| `has_key()` | 1 | 1 |
| `xrange` | 32 | 66 |
| `basestring` | 4-5 | ~10 |
| `unicode(...)` builtin | 14 | 23 |
| `isinstance(x, unicode)` | — | 43 |
| `u"..."` literals | 85 | ~930 |
| `except Exception, e:` | 130 | 329 |
| `raise Exception, "msg"` (2-arg) | 4 | ~13 |
| `raise a, b, c` (3-arg) | 2 | 2 |
| `file()` builtin | 30 | 65 |
| `cStringIO` | 28 | 37 |
| `cPickle` | 13 | 27 |
| `import commands` | 2 | — |
| `urllib2` | 13 | — |
| `from urlparse` | 24 | — |
| `StringIO` (py2 module) | 33 | 34 |
| `from cgi import` | 2 | — |
| `md5` module fallback | 1 | — |
| old-style `class Foo:` | 7 | 11 |
| `<>` / backtick repr | 0 | — |
| `.keys()[0]` indexing | 5 | 5 |
| tuple-unpacking in params | 4 | 4 |
| `cmp()` builtin | 3 | 4 |
| `.next()` (generator method) | 15 | 46 |
| `long()` builtin | 1 | 2 |
| `apply()` builtin | 1 | 2 |
| `.sort(cmp=...)` | 1 | 1 |
| `__metaclass__ =` | 0 | — |
| `from __future__ import` (any) | 107 | — |
| `from __future__ import unicode_literals` | 0 | — |

### `__future__` imports breakdown

```
 95  from __future__ import print_function
  9  from __future__ import with_statement
  8  from __future__ import absolute_import
  3  from __future__ import division
  2  from __future__ import generators
  1  from __future__ import nested_scopes
  1  from __future__ import print_function, with_statement
```

**Notably zero** `unicode_literals` — the codebase uses native `str` (bytes
in py2) by default and sprinkles `u"..."` literals (930 occurrences across
85 files) where unicode is needed.

### Representative examples (5-10 per category)

**print statement:**
- `twistedcaldav/test/data/makelargecalendars.py:40`
- `twistedcaldav/test/data/csv2ical.py:113`
- `txweb2/test/simple_client.py:20` (`print >> sys.stderr, ...`)
- `txdav/caldav/datastore/scheduling/test/test_implicit.py:1057`

**iteritems:**
- `calendarserver/accesslog.py:161`
- `calendarserver/tap/caldav.py:851`
- `txdav/base/propertystore/xattr.py:83`
- `txweb2/channel/http.py:1008`
- `txweb2/http_headers.py:518`
- `setup.py:297`

**except X, e:**
- `calendarserver/webcal/resource.py:172`
- `calendarserver/tap/profiling.py:35`
- `calendarserver/tap/caldav.py:408, 1114`
- `contrib/od/setup_directory.py:303`

**3-arg raise (must become `raise b.with_traceback(c)`):**
- `txdav/common/datastore/sql.py:238`
- `txdav/common/datastore/file.py:213`

**tuple-unpacking in params (py2-only syntax):**
- `txweb2/filter/range.py:15` — `def canonicalizeRange((start, end), size):`
- `txweb2/http_headers.py:867` — `def generateCacheControl((k, v)):`
- `txweb2/test/test_server.py:273`
- `txdav/carddav/datastore/index_file.py:124, 890`

**file() builtin:**
- `setup.py:179, 373, 453`
- `conf/auth/generate_test_accounts.py:41`
- `contrib/performance/setbackend.py:28`

**2-arg raise:**
- `txweb2/log.py:142`
- `txweb2/channel/http.py:662, 686`
- `twistedcaldav/memcacheclient.py:1289, 1337`

### Bytes/str boundary (highest-risk area)

- `unicode(...)` builtin: 23 occurrences across 14 files.
- `isinstance(x, unicode)`: 43 occurrences.
- `isinstance(x, str)`: 49 occurrences (in py2 these mean different things).
- `.encode()` / `.decode()`: 77 / 50 files.
- `b"..."` literals: only 35 occurrences — very few places are explicitly
  byte-string aware.
- `txweb2/auth/basic.py:58` — `response.decode('base64')` (codec removed
  in py3).
- `calendarserver/push/applepush.py:302,356,699` — `message.encode("hex")` /
  `token.decode("hex")` (hex codec removed in py3).
- `txweb2/http_headers.py` — `class Token(str)` and many `@type name:
  C{str}` docstrings: in py2 `str` means bytes, in py3 it means text.
  This is the **single most bytes-critical file** in the HTTP stack.

### Twisted deferreds

The codebase is **deeply** Twisted-generator-async:

- 3,263 `@inlineCallbacks` decorators.
- 1,826 `returnValue(...)` calls (all must become `return x` in modern
  Twisted).
- 15,089 `yield` statements (most are deferred-yields).
- `DeferredList`: 14 files.
- `asyncio` interop: 0 references.
- `defer.setDebugging`: 1 file (`contrib/performance/loadtest/sim.py:193-194`).

### setup.py issues

| Line | Construct |
|---|---|
| `setup.py:19` | `from __future__ import print_function` |
| `setup.py:179` | `file()` builtin |
| `setup.py:189-190` | `Python :: 2.7` / `Python :: 2 :: Only` classifiers |
| `setup.py:297` | `iteritems()` |
| `setup.py:312` | `Twisted==16.6.0` pin |
| `setup.py:316` | `pycrypto` (unmaintained) |
| `setup.py:373, 453` | `file()` builtin |

### plistlib py2 APIs (removed in Py3.4+, gone by 3.9)

| File | API |
|---|---|
| `calendarserver/tap/util.py:106` | `from plistlib import readPlist` |
| `calendarserver/tap/test/test_caldav.py:48` | `from plistlib import writePlist` |
| `calendarserver/tools/agent.py:34` | `readPlistFromString, writePlistToString` |
| `calendarserver/tools/anonymize.py:34` | `readPlistFromString` |
| `calendarserver/tools/diagnose.py:24` | `readPlist, readPlistFromString` |
| `twistedcaldav/stdconfig.py:19` | `from plistlib import PlistParser` |
| `twistedcaldav/dumpconfig.py:20` | `PlistWriter, _escapeAndEncode` (private) |

### Migration effort estimate

- **Mechanical edits**: ~3,500-4,500 line changes across ~277 files (<2% of
  LOC). Achievable with `2to3` / `pyupgrade` / `fissix` + manual cleanup.
- **High-risk manual work**: txweb2 HTTP/headers/stream bytes-boundary
  rewrite (~3-5 engineer-weeks); plistlib API migration (~1 week); Twisted
  version upgrade churn across 3,263 inlineCallbacks sites (~1-2 weeks
  mechanical review).
- **Realistic total**: ~15-25% of the 324k LOC will be touched (mostly
  small edits); ~2-3% needs substantive rewrite. **3-4 months** for
  working Py3.12+ (with sibling repos ported in parallel); **4-6 months**
  for full test-suite green.

---

## 3. Sibling repositories

### Criticality matrix

| Repo | Runtime? | Files in main server | Role |
|---|---|---|---|
| **ccs-pycalendar** | YES (critical) | 81 | iCalendar/vCard parse + RRULE expansion |
| **ccs-twistedextensions** | YES (critical) | 304 | DAL, directory, jobs, push, endpoints |
| **ccs-pykerberos** | Optional | 1 | SPNEGO/Kerberos auth |
| **ccs-caldavclientlibrary** | No (dev) | 16 | Load-test/profile client |
| **ccs-caldavtester** | No (dev, subprocess) | 0 imports | Conformance test suite (1,221 XML scripts) |

### ccs-pycalendar (CRITICAL)

- 124 .py files, 25,858 LOC. 36 test files / 9,739 LOC (~38% tests).
- Pure Python, no third-party runtime deps.
- **`distutils.core.setup`** in `setup.py:17` — removed in Py3.12+, gone
  in 3.14. Blocker.
- **Implicit relative imports** throughout `__init__.py:19-53`:
  `import binaryvalue` (py2-only).
- **`cStringIO as StringIO`** in 13 files.
- **`xml.etree.cElementTree`** → `xml.etree.ElementTree`.
- **Custom `DateTime` class** (`datetime.py`, 1,133 LOC) — reimplements
  date arithmetic that stdlib `datetime` + `zoneinfo` does natively.
- **RRULE expansion** (`recurrence.py`, 1,670 LOC) — battle-tested, cached,
  encodes subtle RFC 5545 edge cases. **Hardest part of iCalendar.**
- **Lenient fix-up parser** (`parser.py:11-83`) — `ParserContext` error-
  policy state machine. Load-bearing for real-world client compat.
- **Embedded VTIMEZONE database** (`timezonedb.py`, `zonal/`) — replaceable
  with stdlib `zoneinfo` + `tzdata`.
- **Rust/PyO3 candidacy: HIGH** (pure logic, hot path, no deps).
- See [ADR-0010](../adr/0010-port-ccs-pycalendar-to-py3.md).

### ccs-twistedextensions / twextpy (CRITICAL)

- 101 .py files, 37,672 LOC. 37 test files / 15,186 LOC (~40% tests).
- Import name `twext`; PyPI name `twextpy`.
- `install_requires=["cffi", "twisted>=16.6"]`. Extras: `dal`
  (`sqlparse==0.2.0`), `ldap` (`python-ldap`), `opendirectory`
  (`pyobjc-framework-OpenDirectory`), `oracle` (`cx_Oracle`), `postgres`
  (`[]`).
- **`zope.interface.implements`** (py2-only) → `@implementer`. Pervasive.
- **`sqlparse==0.2.0`** pin incompatible with modern `sqlparse` (0.4+).
- **CFFI `ffi.verify()`** extensions (deprecated; replaced by
  `ffi.set_source()`): `twext/python/launchd.py` (macOS launchd socket
  activation), `twext/python/sacl.py` (macOS SACL).
- **Twisted-coupled**: `twext.enterprise.adbapi2` (Apple's adbapi fork);
  `twext.who.ldap` (deferToThread); `twext.who.opendirectory`
  (NSAutoreleasePool).
- **DALE DAL** (`twext/enterprise/dal/`): `syntax.py` (2,142 LOC, ~40
  classes), `model.py` (730), `parseschema.py` (797), `record.py` (739).
  Bespoke SQL DSL. See [ADR-0011](../adr/0011-postgresql-sqlite-only-drop-oracle-opendirectory.md).
- **Rust/PyO3 candidacy**: DALE DAL (HIGH, pure logic); `twext.who.xml`
  (Medium); everything else stays Python.
- Migration difficulty: **Very High**.

### ccs-caldavtester

- 45 .py files, 7,384 LOC. **1,221 XML files** (995 fixtures + 226 test
  scripts).
- `install_requires=["pycalendar"]`. Uses `httplib`, `urlparse`, `rfc822`,
  `plistlib.readPlistFromString`, `xml.etree.cElementTree`.
- **Dev/test only**; invoked via subprocess (`bin/testserver:29`).
- The XML scripts are the most valuable asset — they encode Apple's actual
  conformance requirements.
- See [ADR-0015](../adr/0015-compliance-testing-architecture.md).

### ccs-caldavclientlibrary

- 181 .py files, 15,552 LOC. 300 print statements (no `print_function`).
- Uses `httplib`, `urlparse`, `urllib`, `ssl`, PyObjC/AppKit (macOS GUI).
- **Dev-only**; 16 files in `py-ccs-calendarserver` import it, all under
  `contrib/performance/` and `simplugin/`.
- Migration difficulty: Medium. Replacement candidate: modern `caldav` PyPI.

### ccs-pykerberos

- 4 Python files + 5 C source files (2,557 LOC C).
- C extension wrapping GSSAPI/Kerberos. Already has Py3 macros.
- 1 import in main server: `twistedcaldav/authkerb.py:41`.
- Optional. See [ADR-0012](../adr/0012-oauth-oidc-primary-auth.md).

---

## 4. Rust reference implementations

### rs-rustical (RustiCal)

- CalDAV/CardDAV server. AGPL-3.0-or-later. Single maintainer. Very active
  (245 commits in 2026; latest 2026-08-09).
- Workspace: `rustical_dav` (reusable WebDAV trait framework),
  `rustical_caldav`, `rustical_carddav`, `rustical_xml` (custom XML derive
  on quick-xml), `rustical_store_sqlite` (sqlx, clean relational schema).
- HTTP: axum 0.8 + tower. DB: sqlx + SQLite (WAL). Parser: `caldata` crate.
- **Protocol coverage**: CalDAV 4791 ✅, CardDAV 6352 ✅, WebDAV 4918 ✅,
  sync 6578 ✅, ACL 3744 ⚠️ (partial, no ACL method), scheduling 6638 ❌
  (not implemented), TZ 7809 partial, managed attachments 8607 ❌, BIND
  5842/4437 ❌, OAuth ❌ (OIDC client only), HTTP/2 ❌.
- **Adoptability**: `caldata` parser is AGPL (verify license); the trait-
  based `Resource` WebDAV architecture is an excellent **reference
  template** for a future typed Rust WebDAV layer. The SQLite schema is
  clean and small.
- See [ADR-0002](../adr/0002-license-apache-2.0.md) (license constraints)
  and [GOAL.md](../../GOAL.md) Phase 3 (Rust migration runway).

### rs-stalwart (Stalwart)

- Full mail + JMAP + CalDAV/CardDAV/WebDAV server. AGPL-3.0-only OR SELv2.
  Commercially backed. Extremely active (465 commits in 2026; latest
  2026-08-15).
- 27 crates. HTTP: hyper 1.11 (HTTP/1 + HTTP/2). DB: pluggable (RocksDB,
  FoundationDB, PostgreSQL, MySQL, SQLite, Redis, S3). TLS: tokio-rustls +
  ACME.
- **Protocol coverage**: CalDAV 4791 ✅, CardDAV 6352 ✅, WebDAV 4918 ✅,
  scheduling 6638 ✅ (full iTIP/iMIP), ACL 3744 ✅ (full, ACL method),
  sync 6578 ✅, TZ 7809 ✅, OAuth ✅ (full OAuth2 server + OIDC provider
  + Bearer), HTTP/2 ✅, CalendarServer namespace tokenized.
- **`calcard`** crate (Apache-2.0/MIT): iCalendar/vCard/JSCalendar/
  JSContact parser with recurrence + fuzzing. **The single best PyO3
  candidate.** See [ADR-0010](../adr/0010-port-ccs-pycalendar-to-py3.md).
- **`dav-proto/src/schema/mod.rs`** (1,484 lines): exhaustive WebDAV/
  CalDAV/CardDAV/CalendarServer XML element catalog. **Invaluable as a
  typed contract checklist.**
- **`groupware/src/scheduling/`**: complete RFC 6638 iTIP implementation
  in Rust. Reference for organizer/attendee diffing, `ItipSnapshot`.
- **Adoptability**: `calcard` (Apache/MIT) is adoptable; everything else
  DAV-related is AGPL/SEL-licensed and tightly entangled with Stalwart
  internals. Reference only.
- See [ADR-0002](../adr/0002-license-apache-2.0.md) and
  [GOAL.md](../../GOAL.md) Phase 3.

### Comparative summary

| Dimension | rs-rustical | rs-stalwart |
|---|---|---|
| Scope | CalDAV/CardDAV only | Full mail + JMAP + CalDAV/CardDAV |
| License | AGPL-3.0-or-later | AGPL-3.0-only + SELv2 |
| Parser crate | `caldata` (license unverified) | `calcard` (Apache-2.0/MIT) ⭐ |
| HTTP stack | axum 0.8 | hyper 1.11 (HTTP/1+2) |
| TLS/HTTP2 server | None | Yes (rustls + ACME) |
| Storage | SQLite (sqlx, clean schema) | Pluggable KV (RocksDB/PG/MySQL/...) |
| WebDAV reusable layer | Yes (trait framework, decoupled) | Yes in spirit (schema + handlers) but coupled |
| Scheduling (6638) | ❌ | ✅ Full |
| ACL (3744) | Partial | Full |
| OAuth/Bearer | OIDC client only | Full OAuth2 server + OIDC provider |
| CalendarServer namespace | No | Yes (tokenized in schema) |

**Recommendation**: Study Stalwart's protocol crates for the contract
(scheduling, ACL, XML vocabulary, OAuth); study RustiCal's architecture for
the shape (trait-based `Resource`, small crates, sqlx migrations). Adopt
`calcard` (Apache/MIT) as the first PyO3 dependency in phase 3.

---

## 5. References

- [GOAL.md](../../GOAL.md) — mission, principles, phase roadmap
- [DEPENDENCIES.md](DEPENDENCIES.md) — per-dependency replacement table
- [RFC_COMPLIANCE.md](RFC_COMPLIANCE.md) — per-RFC scorecard
- [TYPING_PLAN.md](TYPING_PLAN.md) — mypy rollout + PyO3 preparation
- [MIGRATION_PLAN.md](MIGRATION_PLAN.md) — step-by-step Py3 migration
- [ADR index](../adr/INDEX.md) — architectural decisions
