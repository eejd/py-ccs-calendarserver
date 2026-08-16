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

# MIGRATION_PLAN — Step-by-Step Py3 Migration

**Snapshot date**: 2026-08-16
**Target**: Python 3.12+ (aiming 3.14)

Companion to [ADR-0004](../adr/0004-target-python-3.12-drop-py2.md),
[ADR-0007](../adr/0007-twisted-16-to-24-upgrade.md),
[ADR-0008](../adr/0008-keep-txweb2-forked-phase-1.md),
[ADR-0010](../adr/0010-port-ccs-pycalendar-to-py3.md),
[ADR-0011](../adr/0011-postgresql-sqlite-only-drop-oracle-opendirectory.md).

---

## Overview

- ~3,500-4,500 mechanical line edits across ~277 files (<2% of LOC).
- ~40-60 files needing substantive rewrite.
- **3-4 months** for working Py3.12+ (with sibling repos ported in
  parallel); **4-6 months** for full test-suite green (131k LOC of tests).

### Sequencing rationale

The main repo depends on two critical runtime siblings that MUST be ported
in lockstep:

1. **`ccs-pycalendar`** (81 import sites in main server) — port first;
   it's pure Python with no Twisted coupling.
2. **`ccs-twistedextensions`** (304 import sites) — port in parallel;
   it's deeply Twisted-coupled and is the hardest sibling.

The main repo cannot import without both siblings ported. Therefore, the
sequencing is: **siblings first (or in parallel), then main repo**.

---

## Phase 0 — Prerequisites (week 1-2)

### 0.1 Tooling setup

- [ ] Write `pyproject.toml` for `py-ccs-calendarserver`, `ccs-pycalendar`,
      `ccs-twistedextensions`, `ccs-caldavtester` (ADR-0006).
- [ ] Generate `uv.lock` for each.
- [ ] Configure `ruff` in `pyproject.toml` (lint + format).
- [ ] Set up GitHub Actions CI skeleton (ADR-0015).
- [ ] Install Python 3.12+ via `uv python install 3.12`.

### 0.2 RFC reference library

- [ ] Write `rfc/fetch.sh` + `rfc/manifest.txt` (RFC_COMPLIANCE.md §10).
- [ ] Download ~80 RFCs into `rfc/` folder.
- [ ] Snapshot errata for MUST-tier RFCs (4791, 6352, 5545, 5546, 4918,
      9110, 9112).

### 0.3 Audit verification

- [ ] Verify all `file_path:line_number` references in ADRs and audit docs
      are accurate.
- [ ] Run `rg` to confirm Py2-idiom counts match AUDIT.md §2.

---

## Phase 1 — Sibling repo ports (week 2-8)

### 1.1 ccs-pycalendar (CRITICAL — port first)

**Difficulty**: High (distutils, implicit imports, cStringIO, custom DateTime)
**Estimate**: 2-3 weeks

- [ ] `setup.py:17`: replace `distutils.core.setup` with `setuptools` via
      `pyproject.toml` (ADR-0006, ADR-0010).
- [ ] `src/pycalendar/__init__.py:19-53`: convert implicit relative imports
      (`import binaryvalue`) to absolute (`from . import binaryvalue`).
      Pervasive across 124 files.
- [ ] 13 files: replace `cStringIO as StringIO` with `io.StringIO`/
      `io.BytesIO` (choose bytes-vs-str carefully).
- [ ] Replace `xml.etree.cElementTree` with `xml.etree.ElementTree`.
- [ ] 7 `iteritems`→`items`; 2 `xrange`→`range`; 2 `except X, e:`→`except
      X as e:`; 21 print statements→`print()`.
- [ ] 3 `from __future__` imports → remove.
- [ ] Add `py.typed` marker (ADR-0005).
- [ ] Run the 36-file test suite (9.7k LOC) against Py3; fix failures.
- [ ] Begin strict typing (TYPING_PLAN.md Phase 1).

### 1.2 ccs-twistedextensions (CRITICAL — port in parallel)

**Difficulty**: Very High (Twisted 16→24, zope.interface, sqlparse pin, CFFI)
**Estimate**: 4-6 weeks

- [ ] `setup.py`: replace with `pyproject.toml`; drop `sqlparse==0.2.0` pin
      (audit `parseschema.py` compat with modern sqlparse).
- [ ] 26 `iteritems`→`items`; 6 `xrange`→`range`; 14 `basestring`/`unicode`;
      6 `except X, e:`; 7 `cStringIO`/`cPickle`/`StringIO`; 14 `urllib2`/
      `urlparse`/`commands`; 14 `from __future__`.
- [ ] **94 `implements(IFoo)` → `@implementer(IFoo)`** across `who/` and
      `enterprise/` (mechanical but pervasive).
- [ ] `inspect.getargspec` → `getfullargspec` (removed in Py3.11).
- [ ] `Queue` → `queue`; `md5`/`sha` → `hashlib`; `cPickle` → `pickle`.
- [ ] CFFI `ffi.verify()` → `ffi.set_source()` in `twext/python/launchd.py`
      and `twext/python/sacl.py` (macOS-only; conditional build).
- [ ] Re-base `twext/enterprise/adbapi2.py` against modern
      `twisted.enterprise.adbapi` (ADR-0007).
- [ ] `twisted.internet.kqreactor` imports → update or remove.
- [ ] `twext/who/ldap/_service.py:29`: rewrite `ldap.async` usage (removed
      in Py3) against `ldap3` (ADR-0011).
- [ ] Drop `twext/who/opendirectory/` (1,257 LOC; ADR-0011).
- [ ] Add `py.typed` marker.
- [ ] Run the 37-file test suite (15k LOC) against Py3; fix failures.

### 1.3 ccs-caldavtester (port for conformance testing)

**Difficulty**: High (py2 stdlib, plistlib) but small driver (7k LOC)
**Estimate**: 1-2 weeks

- [ ] `setup.py`: replace with `pyproject.toml`.
- [ ] 6 `iteritems`; 16 `except X, e:`; 31 print statements; 31
      `cStringIO`/`StringIO`; 11 `urllib2`/`urlparse`/`commands`.
- [ ] `xml.etree.cElementTree` → `xml.etree.ElementTree`.
- [ ] `httplib` → `http.client`; `urlparse` → `urllib.parse`; `rfc822` →
      `email.utils`; `plistlib.readPlistFromString` → `plistlib.loads`.
- [ ] Rewrite driver from py2/Twisted-16 to Py3.12 + pytest + treq
      (ADR-0015). Keep 1,221 XML scripts unchanged.
- [ ] Run the full XML corpus against the modernized driver; fix
      regressions.

### 1.4 ccs-pykerberos (optional)

**Difficulty**: Low-Medium
**Estimate**: 1 week (if keeping); or replace with `gssapi` (ADR-0012)

- [ ] `setup.py:14-16`: `commands.getoutput` → `subprocess.getoutput`.
- [ ] Drop `#if PY_MAJOR_VERSION < 3` branches in C code.
- [ ] Audit `kerberosgss.c` for `PyUnicode_GET_SIZE`/`PyString_AS_STRING`
      (removed in Py3.12).
- [ ] OR: replace with `gssapi` PyPI adapter; rewrite
      `twistedcaldav/authkerb.py` (~200 LOC).

### 1.5 ccs-caldavclientlibrary (dev-only)

**Difficulty**: Medium
**Estimate**: 2 weeks (or replace with `caldav` PyPI)

- [ ] 300 print statements → `print()`.
- [ ] `httplib` → `http.client`; `urlparse` → `urllib.parse`; `urllib` →
      `urllib.request`; `ssl` updates.
- [ ] Drop PyObjC/AppKit UI (macOS-only).
- [ ] OR: replace with modern `caldav` PyPI for load-testing.

---

## Phase 2 — Main repo Py3 port (week 4-12)

### 2.1 setup.py + requirements

- [ ] Replace `setup.py` with `pyproject.toml` (ADR-0006).
- [ ] Delete all 7 `requirements-*.txt` files.
- [ ] `setup.py:179, 373, 453`: `file()` → `open()`.
- [ ] `setup.py:297`: `iteritems()` → `items()`.
- [ ] `setup.py:189-190`: remove py2 classifiers.
- [ ] `setup.py:312`: unpin `Twisted==16.6.0`.
- [ ] `setup.py:316`: remove `pycrypto` (ADR-0012).
- [ ] `setup.py:334-343`: remove platform branch (ADR-0009).
- [ ] `setup.py:351-353`: remove `ORACLE_HOME` gate (ADR-0011).

### 2.2 Mechanical Py2→Py3 edits (across ~277 files)

- [ ] 152 `iteritems()` → `items()`.
- [ ] 31 `itervalues()` → `values()`.
- [ ] 19 `iterkeys()` → `keys()`.
- [ ] 1 `has_key()` → `in`.
- [ ] 66 `xrange` → `range`.
- [ ] ~10 `basestring` → `str`.
- [ ] 23 `unicode(...)` → `str(...)`.
- [ ] 43 `isinstance(x, unicode)` → `isinstance(x, str)`.
- [ ] 329 `except X, e:` → `except X as e:`.
- [ ] ~13 `raise X, msg` → `raise X(msg)`.
- [ ] 2 `raise a, b, c` → `raise b.with_traceback(c)`:
      - `txdav/common/datastore/sql.py:238`
      - `txdav/common/datastore/file.py:213`
- [ ] 65 `file()` → `open()`.
- [ ] 37 `cStringIO` → `io.BytesIO`/`io.StringIO`.
- [ ] 27 `cPickle` → `pickle`.
- [ ] 2 `import commands` → `subprocess`.
- [ ] 13 `urllib2` → `urllib.request`/`urllib.error`/`urllib.parse`.
- [ ] 24 `from urlparse` → `from urllib.parse`.
- [ ] 34 `StringIO` (py2 module) → `io.BytesIO`/`io.StringIO`.
- [ ] 2 `from cgi import` → alternative (cgi removed in Py3.13).
- [ ] 1 `md5` module fallback → `hashlib.md5`.
- [ ] 11 old-style `class Foo:` → `class Foo(object):` (or modernize).
- [ ] 4 tuple-unpacking in params → rewrite (py2-only syntax):
      - `txweb2/filter/range.py:15`
      - `txweb2/http_headers.py:867`
      - `txweb2/test/test_server.py:273`
      - `txdav/carddav/datastore/index_file.py:124, 890`
- [ ] 5 `.keys()[0]` / `.values()[0]` → `list(...)[0]` or `next(iter(...))`.
- [ ] 4 `cmp()` → `key=` or `(a > b) - (a < b)`.
- [ ] 1 `.sort(cmp=...)` → `.sort(key=...)`: `twistedcaldav/dateops.py:210`.
- [ ] 46 `.next()` → `next(...)`.
- [ ] 2 `long()` → `int()`.
- [ ] 2 `apply()` → `func(*args, **kwargs)`.
- [ ] 107 `from __future__ import ...` → remove.
- [ ] 930 `u"..."` literals → keep (valid in Py3) or remove prefix.
- [ ] `txweb2/auth/basic.py:58`: `response.decode('base64')` →
      `base64.b64decode(response)`.
- [ ] `calendarserver/push/applepush.py:302,356,699`:
      `message.encode("hex")` → `message.hex()`;
      `token.decode("hex")` → `bytes.fromhex(token)`.

### 2.3 plistlib API migration (7 files)

See [ADR-0013](../adr/0013-plist-and-toml-config-with-pydantic-schema.md).

- [ ] `calendarserver/tap/util.py:106`: `readPlist` → `plistlib.load`.
- [ ] `calendarserver/tap/test/test_caldav.py:48`: `writePlist` →
      `plistlib.dump`.
- [ ] `calendarserver/tools/agent.py:34`: `readPlistFromString,
      writePlistToString` → `plistlib.loads, plistlib.dumps`.
- [ ] `calendarserver/tools/anonymize.py:34`: `readPlistFromString` →
      `plistlib.loads`.
- [ ] `calendarserver/tools/diagnose.py:24`: `readPlist,
      readPlistFromString` → `plistlib.load, plistlib.loads`.
- [ ] `twistedcaldav/stdconfig.py:19`: `PlistParser` → `plistlib.load`.
- [ ] `twistedcaldav/dumpconfig.py:20`: rewrite `PlistWriter`/
      `OrderedPlistWriter`/`_escapeAndEncode` against modern `plistlib`
      (dict natively ordered since py3.7+; the `OrderedPlistWriter` hack is
      no longer needed).
- [ ] Add TOML support (`tomllib`); add `conf/caldavd-test.toml` sample.
- [ ] Author `pydantic` model mirroring `DEFAULT_CONFIG`.
- [ ] Provide `calendarserver_config_convert` tool (plist → TOML).

### 2.4 Twisted 16→24 upgrade (ADR-0007)

**Estimate**: 1-2 weeks (mechanical review)

- [ ] Upgrade `Twisted` to ≥24.3 in `pyproject.toml`.
- [ ] 1,826 `returnValue(x)` → `return x` (mechanical sed).
- [ ] 94 `implements(IFoo)` → `@implementer(IFoo)` (mechanical).
- [ ] `twisted.python.log` → `twisted.logger` (voluminous).
- [ ] Delete `lib-patches/Twisted/securetransport.patch` (ADR-0009).
- [ ] Delete `requirements-twisted-*.txt` (ADR-0006).
- [ ] Test `asyncioreactor` with the full server; fall back to default
      reactor if issues (ADR-0007).

### 2.5 txweb2 bytes/str boundary rewrite (HIGHEST RISK)

**Estimate**: 3-5 engineer-weeks (ADR-0008)

- [ ] `txweb2/http_headers.py` (1,787 LOC): rewrite `class Token(str)` and
      the header tokenizer for py3 bytes/text separation. HTTP headers are
      text-after-decoding; header values are bytes on the wire.
- [ ] `txweb2/channel/http.py`: rewrite the HTTP/1.1 parser for py3
      bytes/text. Fix `iteritems`, `cPickle`, `cStringIO`, old-style
      classes, 2-arg raises.
- [ ] `txweb2/stream.py`: rewrite the byte stream module. Ensure every
      producer produces bytes.
- [ ] Fix internal `twisted.internet._sslverify` imports → use public
      `twisted.internet.ssl` API (ADR-0007, ADR-0009).
- [ ] Remove `hasattr(OpenSSL, "__SecureTransport__")` runtime checks at
      `calendarserver/push/util.py:42` and
      `calendarserver/tap/util.py:1256,1306,1341,1367` (ADR-0009).
- [ ] Fix `txweb2/metafd.py` `twisted.internet.abstract.FileDescriptor`
      internal imports.
- [ ] 4 tuple-unpacking in params (see §2.2).
- [ ] Comprehensive test coverage for the HTTP layer; property-based
      testing for header parsing (ADR-0015 Tier-3).

### 2.6 Security fixes (ADR-0009, ADR-0012)

- [ ] Replace `pycrypto` with `cryptography` in
      `txdav/caldav/datastore/scheduling/ischedule/dkim.py:31-33` and
      `calendarserver/tools/dkimtool.py:20`.
- [ ] Drop iSchedule (remove `txdav/caldav/datastore/scheduling/ischedule/`).
- [ ] Replace `SSLv23_METHOD` default at `twistedcaldav/stdconfig.py:174`
      with TLS 1.2+ enforcement per RFC 9325.
- [ ] Add HSTS middleware (RFC 6797; configurable, default-on in prod).
- [ ] Add `truststore` dependency for macOS system-root CA loading.
- [ ] Fix `txweb2/auth/basic.py:58` (`base64.b64decode`); update to RFC
      7617 (UTF-8).
- [ ] Update `txweb2/auth/digest.py:27` to RFC 7616 (SHA-256, charset,
      userhash).
- [ ] Implement OAuth 2.0 Bearer token validation (RFC 6750) in a new
      `txweb2/auth/oauth.py` or `twistedcaldav/auth_oauth.py`.
- [ ] Rewrite `twistedcaldav/authkerb.py` to use `gssapi` (optional extra).

### 2.7 Storage/directory modernization (ADR-0011)

- [ ] Upgrade `pg8000` to 1.31.x; drop `import six` at
      `txdav/base/datastore/dbapiclient.py:29`.
- [ ] Delete `cx_Oracle`, `lib-patches/cx_Oracle/`,
      `lib-patches/memcached/`.
- [ ] Delete `twext/who/opendirectory/` (1,257 LOC).
- [ ] Delete `mockldap` from `requirements-dev.txt`.
- [ ] Replace `python-ldap` with `ldap3` in `twext/who/ldap/_service.py`.
- [ ] Drop `pySecureTransport` / `OSXFrameworks` dependencies.
- [ ] Drop `pyobjc-framework-OpenDirectory` dependency.

### 2.8 Apple extension segregation (ADR-0014)

**Estimate**: 2-3 weeks (best-effort in phase 1; full physical separation
may wait for phase 2/3)

- [ ] Create `apple_extensions/` package structure.
- [ ] Implement conditional `@registerElement` registration deferral.
- [ ] Move Tier-2 compliance token advertisement into `apple_extensions/`.
- [ ] Add `config.EnableAppleExtensions` flag (default: True).
- [ ] Document the tier classification in `apple_extensions/README.md`.

### 2.9 bin/ scripts

- [ ] Update `bin/_build.sh:756-759`: `lib/python2.7/site-packages/...` →
      `lib/python3.12/site-packages/...`.
- [ ] Update `bin/_py.sh:64`: `python2.7` → `python3`.
- [ ] Update shell wrappers (`bin/develop`, `bin/run`, `bin/test`) to use
      `uv` (ADR-0006).

---

## Phase 3 — Test suite + CI (week 8-16)

### 3.1 Test suite green

- [ ] Run the 203-file test suite (131k LOC) against Py3; fix failures.
- [ ] Run the modernized `ccs-caldavtester` against the Py3 server; fix
      conformance regressions.
- [ ] Add Hypothesis property-based round-trip tests for iCalendar/vCard
      (ADR-0015 Tier-3).
- [ ] Add `atheris` fuzzing job (nightly CI).

### 3.2 CI (GitHub Actions, 7 jobs — ADR-0015)

- [ ] `lint`: `ruff check` + `ruff format --check` + `mypy`.
- [ ] `unit-py-3.12`, `unit-py-3.13`, `unit-py-3.14`: Tier-0 RFC tests.
- [ ] `protocol`: Tier-1 ccs-caldavtester suite, in-process Twisted server.
- [ ] `interop`: Tier-2 vdirsyncer matrix (nightly).
- [ ] `property`: Tier-3 Hypothesis + atheris (nightly).
- [ ] Cache `uv.lock` and `rfc/` folder as CI cache keys.

### 3.3 Typing (in parallel — TYPING_PLAN.md)

- [ ] Write `stubs/zope/interface/__init__.pyi` shim.
- [ ] Write `py.typed` markers for all 6 packages.
- [ ] Write `mypy.ini` / `pyproject.toml [tool.mypy]` configuration.
- [ ] Phase 1: type `ccs-pycalendar` (entire).
- [ ] Phase 2: type `txdav/xml/base.py` + `txdav/xml/rfc*.py`.
- [ ] Phase 3: type `twistedcaldav/config.py`.
- [ ] Phase 4: type `txdav/common/datastore/sql_tables.py`, `sql_util.py`,
      `icommondatastore.py`, `idav.py`.

---

## Phase 4 — RFC compliance (week 12-20)

### Phase-1 RFC scope (per ADR-0012, ADR-0009, RFC_COMPLIANCE.md §4)

- [ ] RFC 9110 audit: update code references from 2616/2617 to 9110 in
      `txweb2/http_headers.py`, `txweb2/channel/http.py`,
      `txweb2/auth/digest.py`, `txweb2/dav/method/*.py`.
- [ ] RFC 7616 Digest SHA-256 (§2.6).
- [ ] RFC 6750/6749 OAuth 2.0 Bearer auth (§2.6).
- [ ] RFC 6764 SRV records: add SRV record handling (server-side
      discovery).
- [ ] RFC 9325 TLS best practices (§2.6).
- [ ] RFC 6797 HSTS (§2.6).
- [ ] Modern APNs HTTP/2: replace legacy binary protocol in
      `calendarserver/push/applepush.py`.
- [ ] Drop iSchedule (§2.6).

---

## Risk register

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| txweb2 bytes/str boundary bugs | High (data corruption) | High | Dedicated engineer-weeks; comprehensive tests; property-based header parsing |
| Sibling repo ports block main repo | High (cannot import) | Medium | Port siblings first/in parallel (Phase 1) |
| Twisted 16→24 API churn | Medium | High | Mechanical review; 3,263 inlineCallbacks sites |
| adbapi2 re-base breaks DB layer | High (datastore) | Medium | Re-base carefully; run DB test suite |
| LDAP rewrite breaks enterprise | Medium | Medium | Run LDAP test suite; containerized OpenLDAP in CI |
| ccs-pycalendar leniency drift | Medium (client compat) | Low | Run full test suite; property-based round-trip tests |
| Config pydantic model drift | Low | Medium | Generate model from DEFAULT_CONFIG or lockstep in review |
| OAuth/OIDC complexity | Medium | Medium | Defer to external provider; Bearer validation only in phase 1 |

---

## References

- [AUDIT.md](AUDIT.md) — full state-of-code audit
- [DEPENDENCIES.md](DEPENDENCIES.md) — dependency replacement table
- [RFC_COMPLIANCE.md](RFC_COMPLIANCE.md) — per-RFC scorecard
- [TYPING_PLAN.md](TYPING_PLAN.md) — mypy rollout
- [ADR-0004](../adr/0004-target-python-3.12-drop-py2.md) — Py3 target
- [ADR-0007](../adr/0007-twisted-16-to-24-upgrade.md) — Twisted upgrade
- [ADR-0008](../adr/0008-keep-txweb2-forked-phase-1.md) — txweb2
- [ADR-0009](../adr/0009-standardize-tls-pyopenssl-cryptography.md) — TLS
- [ADR-0010](../adr/0010-port-ccs-pycalendar-to-py3.md) — parser
- [ADR-0011](../adr/0011-postgresql-sqlite-only-drop-oracle-opendirectory.md) — storage
- [ADR-0012](../adr/0012-oauth-oidc-primary-auth.md) — auth
- [ADR-0013](../adr/0013-plist-and-toml-config-with-pydantic-schema.md) — config
- [ADR-0014](../adr/0014-segregate-apple-extensions.md) — Apple extensions
- [ADR-0015](../adr/0015-compliance-testing-architecture.md) — testing
