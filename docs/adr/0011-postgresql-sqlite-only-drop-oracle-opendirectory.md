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

# ADR-0011: PostgreSQL + SQLite only; drop Oracle/OpenDirectory; ldap3

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

The current storage and directory backend matrix is complex and includes
unmaintained/platform-specific/deprecated components:

### Storage backends

| Backend | Current state | Issues |
|---|---|---|
| PostgreSQL | `pg8000==1.10.6` (`txdav/base/datastore/dbapiclient.py:28`) | Old; pure-Python; slow. Upgrade to 1.31 or migrate to psycopg 3. |
| SQLite | Used for dev/test | Fine; keep. |
| Oracle | `cx_Oracle==5.2` with `lib-patches/cx_Oracle/nclob-fixes-and-prefetch.patch` (52 lines) | `cx_Oracle` renamed to `oracledb` (2022); the patch is obsolete with modern `oracledb`; only needed if Oracle is actually used. `setup.py:353` gates on `ORACLE_HOME` env. |

### Directory backends

| Backend | Current state | Issues |
|---|---|---|
| XML file | `twistedcaldav/directory/` + `twext/who/xml.py` (571 LOC) | Fine; dev/test default. Keep. |
| LDAP | `python-ldap==2.4.28` (`twext/who/ldap/_service.py:29` uses `ldap.async`, removed in Py3) | `python-ldap` needs libldap C deps; `ldap.async` module removed in Py3; rewrite needed. |
| OpenDirectory (macOS) | `pyobjc-framework-OpenDirectory` + `twext/who/opendirectory/_service.py` (1,257 LOC) | macOS-only; PyObjC-coupled; not portable. Drop. |
| Kerberos/SPNEGO | `ccs-pykerberos` C extension (`twistedcaldav/authkerb.py:41`) | Optional; make it an extra; replace with `gssapi` PyPI adapter. See ADR-0012. |

### Other dropped components

| Component | Reason |
|---|---|
| `pySecureTransport` / `OSXFrameworks` | See ADR-0009. |
| `mockldap` | Declared in `requirements-dev.txt:4` but **0 actual imports** in the test code. Dead entry. |
| `lib-patches/cx_Oracle/` | Obsolete with modern `oracledb`; Oracle support dropped. |
| `lib-patches/memcached/` | Patches the memcached **daemon** (not a Python lib) for a 2014-era bug. Run modern memcached instead. |

## Alternatives Considered

### Option A — Keep all backends; modernize each

Keep PostgreSQL, SQLite, Oracle, LDAP, OpenDirectory, Kerberos. Modernize:
`oracledb` replaces `cx_Oracle`; psycopg 3 replaces pg8000; ldap3 replaces
python-ldap; pyobjc-framework-OpenDirectory stays macOS-only; pySecureTransport
dropped.

**Pros**: Maximum backend flexibility; enterprise deployments preserved.
**Cons**: Large maintenance surface; OpenDirectory is macOS-only and not
portable; Oracle is niche and the patch maintenance is a burden; testing
matrix is large.

### Option B — Aggressive simplification: PostgreSQL + SQLite + XML only

Drop Oracle, LDAP, OpenDirectory, Kerberos. Only PostgreSQL + SQLite + XML
directory. Add OAuth/OIDC as primary auth.

**Pros**: Simplest; most modern; smallest testing matrix.
**Cons**: Loses LDAP support, which is still widely used in enterprise
deployments; loses Kerberos/SPNEGO which some macOS/AD environments need.

### Option C — PostgreSQL + SQLite; modernize LDAP; drop Oracle/OpenDirectory

Keep PostgreSQL (primary) and SQLite (dev/test). Keep and modernize LDAP via
`ldap3` (pure-Python, no libldap). Keep XML directory as dev default. Drop
Oracle, OpenDirectory, pySecureTransport/OSXFrameworks, mockldap. Make
Kerberos optional via `gssapi` (see ADR-0012).

**Pros**: Preserves LDAP for enterprise; drops niche/platform-specific
backends; `ldap3` is pure-Python (no C deps); reasonable testing matrix;
aligns with the user's directive.
**Cons**: Loses Oracle (niche) and OpenDirectory (macOS-only); some
enterprise deployments may need Oracle (unlikely for a revival project).

## Decision

We adopt **Option C: PostgreSQL (primary) + SQLite (dev/test); modernize
LDAP via `ldap3`; drop Oracle, OpenDirectory, pySecureTransport/
OSXFrameworks, mockldap; make Kerberos optional via `gssapi`.**

### Storage policy

- **PostgreSQL** is the primary production backend.
  - Phase 1: upgrade `pg8000` to 1.31.x (minimal code change; `dbapiclient.py`
    keeps working; drop `import six` at line 29).
  - Phase 2: migrate to `psycopg[binary]` 3.x (implements DB-API 2.0 PEP 249;
    significantly faster; libpq-based; enables async via `psycopg.AsyncConnection`
    + `twisted.internet.defer Deferred.fromCoroutine` bridge in phase 2).
- **SQLite** is the dev/test backend. Keep as-is.
- **Oracle** is dropped. Remove `cx_Oracle` dependency, `lib-patches/cx_Oracle/`,
  and the `ORACLE_HOME` gate in `setup.py:351-353`.
- The shared connection pool (`config.SharedConnectionPool`,
  `twistedcaldav/stdconfig.py:242`) semantics map cleanly to
  `psycopg_pool.ConnectionPool` in phase 2.

### Directory policy

- **XML file** directory remains the dev/test default (`twext/who/xml.py`).
- **LDAP** is modernized: replace `python-ldap==2.4.28` with `ldap3`
  (pure-Python, no libldap C deps, async-friendly). Rewrite
  `twext/who/ldap/_service.py:29` (`ldap.async` removed in Py3). Use
  `ldap3`'s mock or containerized OpenLDAP in CI for tests.
- **OpenDirectory** is dropped. Remove `pyobjc-framework-OpenDirectory`
  dependency and `twext/who/opendirectory/` (1,257 LOC). This is macOS-only
  and not portable. If needed in the future, it can be re-added as a
  macOS-only extra.
- **Kerberos/SPNEGO** is made optional via `gssapi` PyPI adapter. See
  ADR-0012 for auth details.

### Dropped components

| Component | Action | Reason |
|---|---|---|
| `cx_Oracle==5.2` + `lib-patches/cx_Oracle/` | Delete | Oracle support dropped; patch obsolete with modern `oracledb`. |
| `pyobjc-framework-OpenDirectory` + `twext/who/opendirectory/` | Delete | macOS-only; not portable. |
| `pySecureTransport` + `OSXFrameworks` | Delete | See ADR-0009. |
| `mockldap` | Delete | Declared but 0 imports; dead entry. Use `ldap3` mock or containerized slapd. |
| `lib-patches/memcached/` | Delete | Patches memcached daemon for a 2014 bug. Run modern memcached. |
| `python-ldap==2.4.28` | Replace with `ldap3` | `ldap.async` removed in Py3; `ldap3` is pure-Python. |

### Async model

The async model is unchanged: `twext.enterprise.adbapi2` is a thread-pool-based
adapter. The reactor owns a `twisted.python.threadpool.ThreadPool`; each DB
operation runs a **synchronous** DB-API call on a worker thread; results are
marshaled back as `Deferred`s. pg8000/psycopg do not need to be async. This
remains valid in Twisted 24.x. See [AUDIT.md](../audit/AUDIT.md) §7.

Moving to an async driver (asyncpg, async-psycopg) would require a top-to-
bottom `async def` refactor of the datastore — out of scope for phase 1.

## Consequences

### Positive

- Simplified backend matrix; smaller testing surface.
- `ldap3` is pure-Python (no libldap C deps); easier install and CI.
- Drops niche/platform-specific backends (Oracle, OpenDirectory) that are
  maintenance burdens.
- Phase-2 `psycopg 3` migration path is clear and performance-improving.
- Removes obsolete patches (`cx_Oracle`, `memcached`).

### Negative

- Loses Oracle support (niche; unlikely to be needed for a revival project).
- Loses OpenDirectory support (macOS-only; if needed, re-add as extra).
- LDAP rewrite (`twext/who/ldap/_service.py`, 1,341 LOC) is non-trivial;
  `ldap3`'s API differs from `python-ldap`.

### Neutral

- Kerberos/SPNEGO becomes an optional extra; enterprise AD deployments can
  install `ccs-calendarserver[kerberos]` to get the `gssapi` adapter.

### Risks

- **LDAP rewrite breaks enterprise deployments**: `twext/who/ldap/_service.py`
  (1,341 LOC) is a full LDAP directory service. Rewriting against `ldap3` is
  non-trivial. Mitigation: run the LDAP test suite
  (`twext/who/ldap/test/test_service.py`) against the `ldap3`-based
  implementation; use containerized OpenLDAP in CI.
- **`psycopg 3` migration in phase 2 breaks `dbapiclient.py`**: psycopg 3
  implements DB-API 2.0 (PEP 249) so the cursor/execute/fetch API works
  unchanged, but connection pooling and SSL configuration differ.
  Mitigation: phase-2 migration is gated on the DB test suite passing.
- **OpenDirectory drop alienates macOS enterprise users**: if a deployment
  needs OpenDirectory, they MUST stay on the old codebase or re-add the
  backend. Mitigation: document the drop; provide a migration path to LDAP.

### Follow-ups

- Upgrade `pg8000` to 1.31.x; drop `import six` at `dbapiclient.py:29`.
- Replace `python-ldap` with `ldap3`; rewrite `twext/who/ldap/_service.py`.
- Delete `cx_Oracle`, `lib-patches/cx_Oracle/`, `lib-patches/memcached/`.
- Delete `twext/who/opendirectory/` (1,257 LOC).
- Delete `mockldap` from `requirements-dev.txt`.
- Remove `ORACLE_HOME` gate in `setup.py:351-353`.
- Phase 2: migrate `pg8000` → `psycopg[binary]` 3.x.

## References

- `txdav/base/datastore/dbapiclient.py:28` (`import pg8000 as postgres`)
- `txdav/base/datastore/dbapiclient.py:29` (`import six`)
- `txdav/base/datastore/dbapiclient.py:31-54` (`cx_Oracle` guarded import)
- `twext/who/ldap/_service.py:29` (`ldap.async` — removed in Py3)
- `twext/who/opendirectory/_service.py` (1,257 LOC — being dropped)
- `setup.py:351-353` (`ORACLE_HOME` gate)
- `requirements-dev.txt:4` (`mockldap` — dead entry)
- `lib-patches/cx_Oracle/nclob-fixes-and-prefetch.patch` (52 lines — obsolete)
- `lib-patches/memcached/items-assert.patch` (11 lines — obsolete)
- `ldap3` on PyPI: https://pypi.org/project/ldap3/
- `psycopg` 3 docs: https://www.psycopg.org/psycopg3/docs/
- [DEPENDENCIES.md](../audit/DEPENDENCIES.md) — per-dependency replacement table
- [ADR-0009](0009-standardize-tls-pyopenssl-cryptography.md) — TLS (drops pySecureTransport)
- [ADR-0012](0012-oauth-oidc-primary-auth.md) — auth (Kerberos optional)
