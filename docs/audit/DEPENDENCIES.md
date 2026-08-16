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

# DEPENDENCIES — Replacement Plan

**Snapshot date**: 2026-08-16

Per-dependency replacement table with risk, priority, and modern
alternatives. Companion to [ADR-0006](../adr/0006-pyproject-toml-uv-ruff.md)
(tooling), [ADR-0007](../adr/0007-twisted-16-to-24-upgrade.md) (Twisted),
[ADR-0009](../adr/0009-standardize-tls-pyopenssl-cryptography.md) (TLS),
[ADR-0011](../adr/0011-postgresql-sqlite-only-drop-oracle-opendirectory.md)
(storage/directory), [ADR-0012](../adr/0012-oauth-oidc-primary-auth.md)
(auth/crypto).

---

## Priority legend

- **P0** — Security/blocking. Must be done first.
- **P1** — Blocks Py3 port or modern functionality. Done early in phase 1.
- **P2** — Modernization. Done mid phase 1.
- **P3** — Optional/deferred. Done late phase 1 or phase 2.

---

## Core dependencies

| Dependency | Pinned | Maintained 2026? | Py2-only? | Modern replacement | Risk | Priority |
|---|---|---|---|---|---|---|
| **Twisted** | 16.6.0 | Yes (24.x+) | No | (keep Twisted, upgrade to ≥24.3) | High | P0 |
| **pycrypto** | 2.6.1 | **Unmaintained since 2013; CVEs** | Py2/3 | **`cryptography`** 43+ | Medium | P0 |
| **pySecureTransport** (`ccs-pysecuretransport`) | git | Archived by Apple; ST deprecated | yes | **remove** — pyOpenSSL+cryptography | High | P0 |
| **OSXFrameworks** (`ccs-pyosxframeworks`) | git | Archived by Apple | yes | **remove** | High | P0 |
| **enum34** | 1.1.6 | No (Py3 stdlib) | Py2 backport | **remove** | Low | P0 |
| **ipaddress** | unpinned | No (Py3 stdlib) | Py2 backport | **remove** | Low | P0 |
| **six** | 1.10.0 | Deprecated | — | **remove** | Low | P0 |
| **characteristic** | 14.3.0 | Deprecated (folded into attrs) | No | **remove** (transitive of old service_identity) | Low | P0 |
| **ccs-pycalendar** `distutils.core` | — | `distutils` removed in Py3.12 | — | switch to `setuptools`/`hatchling` | Low | P0 |
| **zope.interface** | 4.3.2 | Yes (7.x) | No | (keep, upgrade) | Low | P1 |
| **setuptools** | 30.2.0 | Yes (70+) | No | (keep, drop pin) | Low | P1 |
| **cffi** | 1.9.1 | Yes (1.17+) | No | (keep) | Low | P1 |
| **pycparser** | 2.17 | Yes (2.22+) | No | (keep, transitive of cffi) | Low | P1 |
| **python-ldap** | 2.4.28 | Yes (3.4.x) | 2.4 was Py2-only | **`ldap3`** (pure-Python, no libldap) | Medium | P1 |
| **sqlparse** | 0.2.0 | Yes (0.5.x) | No | (keep, but unpin; audit `parseschema.py` compat) | Medium | P1 |
| **pg8000** | 1.10.6 | Yes (1.31+) | No | upgrade to 1.31 (phase 1) → `psycopg 3` (phase 2) | Medium | P1 |
| **python-dateutil** | 2.6.0 | Yes (2.9) | No | (keep, combine with stdlib `zoneinfo`) | Low | P1 |
| **pytz** | 2016.7 | Maintenance only | No | **`zoneinfo`** (stdlib 3.9+) + `tzdata` | Medium | P1 |
| **psutil** | 5.0.0 | Yes (6.x) | No | (keep, upgrade) | Low | P1 |
| **setproctitle** | 1.1.10 | Yes (1.3) | No | (keep) | Low | P1 |
| **pyOpenSSL** | 16.2.0 | Yes (24.x) | No | (keep, upgrade; thin wrapper around `cryptography`) | Low | P1 |
| **cryptography** | 1.6 | Yes (43+) | No | (keep, upgrade) | Medium | P1 |
| **service_identity** | 16.0.0 | Yes (24.x) | No | (keep, upgrade; `characteristic` dep dropped) | Low | P1 |
| **pyasn1** | 0.1.9 | Yes (0.6) | No | (keep) | Low | P1 |
| **pyasn1-modules** | 0.0.8 | Yes (0.4) | No | (keep) | Low | P1 |
| **incremental** | 16.10.1 | Yes (24.x) | No | (keep, transitive of Twisted) | Low | P1 |
| **constantly** | 15.1 | Yes (23.x) | No | (keep, transitive of Twisted) | Low | P1 |
| **xattr** | 0.7.5 (commented) | Yes (1.1) | No | (keep, only if upgrade-from-ancient path kept) | Low | P2 |
| **cx_Oracle** | 5.2 (patched) | Renamed to `oracledb` | 5.2 is Py2 | **drop** (Oracle support dropped) | Low | P2 |
| **mockldap** | unpinned | Abandoned (last release 2017) | Py2/3 claim | **remove** (declared but 0 imports); use `ldap3` mock or containerized slapd | Low | P2 |
| **pyflakes** | unpinned | Yes (3.x) | No | **`ruff`** (replaces pyflakes + flake8 + isort + black) | Low | P2 |
| **q** | unpinned | Abandoned | Py2 | **remove** (use `pdbpp`/`icecream`) | Low | P2 |
| **docutils** | unpinned | Yes (0.21) | No | (keep) | Low | P2 |
| **CalDAVClientLibrary** (`ccs-caldavclientlibrary`) | git | Archived, "requires Python 2.5" | yes | fork & port, or replace with `caldav` PyPI for dev/test | High | P3 |
| **CalDAVTester** (`ccs-caldavtester`) | git | Archived | yes | fork & port driver; keep XML corpus | High | P3 |
| **pyobjc-framework-OpenDirectory** | (system) | Yes (10.x) | No | **drop** (OpenDirectory support dropped) | Low | P3 |
| **kerberos** (`ccs-pykerberos`) | 1.3.1 | Apple archived; community forks | Both | make optional via **`gssapi`** PyPI adapter | Medium | P3 |

---

## lib-patches/

| Patch | Target | Action | Reason |
|---|---|---|---|
| `lib-patches/Twisted/securetransport.patch` (31 lines) | Twisted 16.6.0 | **Delete** | Irrelevant on modern Twisted; pySecureTransport dropped (ADR-0009). |
| `lib-patches/cx_Oracle/nclob-fixes-and-prefetch.patch` (52 lines) | cx_Oracle 5.2 C sources | **Delete** | Oracle support dropped (ADR-0011); patch obsolete with modern `oracledb`. |
| `lib-patches/memcached/items-assert.patch` (11 lines) | memcached 1.4.24 daemon | **Delete** | Patches the memcached daemon for a 2014 bug. Run modern memcached 1.6.x. |

---

## requirements-*.txt files (all to be replaced by `pyproject.toml` + `uv.lock`)

| File | Action |
|---|---|
| `requirements-cs.txt` | Replace with `pyproject.toml` `[project.dependencies]` |
| `requirements-default.txt` | Delete (aggregator) |
| `requirements-dev.txt` | Replace with `pyproject.toml` `[project.optional-dependencies.dev]` |
| `requirements-twisted-default.txt` | Delete (merged into main deps) |
| `requirements-twisted-osx.txt` | Delete (platform split removed; ADR-0009) |
| `requirements-osx.txt` | Delete (aggregator) |
| `requirements-ignore-installed.txt` | Delete (patched-Twisted workaround no longer needed) |

---

## Stub dependencies (for mypy)

| Stub | For | Source |
|---|---|---|
| `types-psutil` | psutil | PyPI |
| `types-python-dateutil` | python-dateutil | PyPI |
| `types-pyOpenSSL` | pyOpenSSL | PyPI |
| `types-cryptography` | cryptography | PyPI (or inline stubs) |
| `types-setuptools` | setuptools | PyPI |
| `types-pyasn1` | pyasn1 | PyPI |
| Twisted stubs | Twisted | Inline (since 21.7; no `twisted-stubs` package needed) |
| zope.interface stubs | zope.interface | **Write locally** (`stubs/zope/interface/__init__.pyi`) — no upstream |
| `types-python-memcached` | memcache (if used) | PyPI |

---

## New dependencies (phase 1)

| Dependency | Purpose | Priority |
|---|---|---|
| `pydantic` | Config schema validation (ADR-0013) | P1 |
| `truststore` | macOS system-root CA loading (ADR-0009) | P1 |
| `ldap3` | Pure-Python LDAP (replaces python-ldap; ADR-0011) | P1 |
| `gssapi` | Optional Kerberos/SPNEGO (ADR-0012) | P3 (optional extra) |
| `hypothesis` | Property-based testing (ADR-0015) | P2 |
| `atheris` | Fuzz testing (ADR-0015) | P2 |
| `pytest-twisted` | Twisted test adapter for pytest (ADR-0015) | P1 |
| `pytest-cov` | Coverage | P1 |
| `pytest-xdist` | Parallel test execution | P2 |
| `pytest-hypothesis` | Hypothesis pytest plugin | P2 |
| `mypy` | Static type checker (ADR-0005) | P1 |
| `ruff` | Linter + formatter (ADR-0006) | P1 |
| `uv` | Environment/dependency manager (ADR-0006) | P1 |

---

## New dependencies (phase 2, deferred)

| Dependency | Purpose | Priority |
|---|---|---|
| `psycopg[binary]` 3.x | PostgreSQL driver (replaces pg8000) | P2 (phase 2) |
| `h2` | HTTP/2 support (if using twisted.web + h2) | P2 (phase 2) |
| `tomli_w` | TOML writing (for config converter; ADR-0013) | P2 |

---

## References

- `py-ccs-calendarserver/requirements-*.txt` (7 files, 2016-era pins)
- `py-ccs-calendarserver/setup.py:309-349` (install_requires, extras_require)
- `ccs-twistedextensions/setup.py:219-236` (twextpy extras)
- `ccs-pycalendar/setup.py:17` (distutils.core.setup)
- `lib-patches/` (3 patches, all being deleted)
- [ADR-0006](../adr/0006-pyproject-toml-uv-ruff.md) — pyproject.toml + uv
- [ADR-0007](../adr/0007-twisted-16-to-24-upgrade.md) — Twisted upgrade
- [ADR-0009](../adr/0009-standardize-tls-pyopenssl-cryptography.md) — TLS
- [ADR-0011](../adr/0011-postgresql-sqlite-only-drop-oracle-opendirectory.md) — storage/directory
- [ADR-0012](../adr/0012-oauth-oidc-primary-auth.md) — auth/crypto
