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

# TYPING_PLAN — mypy Strict Rollout + PyO3 Preparation

**Snapshot date**: 2026-08-16

Companion to [ADR-0005](../adr/0005-mypy-strict-per-module-rollout.md)
(typing strategy) and [ADR-0003](../adr/0003-mission-py3-revival-as-typed-contract.md)
("typed Python as contract" principle).

---

## 1. Existing typing

**None.** Across all three repos (`py-ccs-calendarserver`,
`ccs-pycalendar`, `ccs-twistedextensions`):

- 0 occurrences of `from typing import`.
- 0 type annotations (`def foo(x: int)`).
- 0 `# type:` comments.
- 0 `py.typed` marker files.
- 0 `mypy.ini` / `setup.cfg [mypy]` / `pyproject.toml [tool.mypy]`.
- 0 `from __future__ import annotations`.

The only forward-looking imports are ~120 `from __future__ import
print_function` / `division` — the minimum needed for py2/py3 compat.

---

## 2. Code structure affecting typing strategy

### txweb2/ — HTTP/WebDAV framework

- 93 files, 27.5k LOC. Vendored fork of `twisted.web2`.
- 8 zope.interface classes in `txweb2/iweb.py` (`IResource`, `IRequest`,
  `IResponse`, `ISite`, `IChanRequest`).
- Big files: `txweb2/http_headers.py` (1,787 LOC), `txweb2/dav/resource.py`
  (2,768 LOC), `txweb2/http.py`, `txweb2/channel/http.py` (has `__getattr__`
  proxies).

### txdav/ — Data-access layer (largest, hardest core)

- 279 files, 139k LOC.
- `txdav/idav.py` (335 LOC): 8 zope.interface Interfaces (`IPropertyName`,
  `IPropertyStore`, `IDataStore`, `IDataStoreObject`, `ITransaction`,
  `INotifier`, `IStoreNotifierFactory`, `IStoreNotifier`).
- DAL in `txdav/common/datastore/` (`sql.py` 5,031 LOC, `file.py` 1,586 LOC,
  `sql_sharing.py` 1,447 LOC) and `txdav/caldav/datastore/sql.py` (5,547)
  and `txdav/carddav/datastore/sql.py` (2,801).
- **115 `@inlineCallbacks`** in `sql.py` alone; project totals 3,263.
- Heavy **class-attribute registration / monkey-patching**:
  `CalendarHome._externalClass = CalendarHomeExternal`,
  `Calendar._objectResourceClass = CalendarObject`,
  `CommonHome._trashClass = TrashCollection`.

### twistedcaldav/ — CalDAV/CardDAV implementation

- 149 files, 79k LOC.
- Largest files: `storebridge.py` (3,934 LOC, 24 classes),
  `ical.py` (3,891 LOC), `resource.py` (2,954 LOC, 11 classes with deep
  multiple inheritance), `stdconfig.py` (1,893 LOC),
  `caldavxml.py` (1,600+ LOC, **71 classes** — XML element-per-class).

### ccs-pycalendar (sibling)

- 124 files, 25.9k LOC. Pure-Python iCalendar/vCard/TZif parser.
- **No Twisted dependency, no zope.interface.** Best PyO3/mypy-stub
  candidate.

### ccs-twistedextensions / twextpy (sibling)

- 101 files, 37.7k LOC.
- `twext/enterprise/dal/syntax.py` (2,142 LOC, ~40 classes modelling SQL
  as a syntax tree). `model.py` (730), `record.py` (739), `parseschema.py`
  (797).

---

## 3. Dynamic patterns that challenge mypy strict mode

| Pattern | Count | Notes |
|---|---|---|
| `getattr(` dynamic | 236 | Plugin/reflect dispatch. |
| `setattr(` | 74 | Runtime class-attribute mutation. |
| `def __getattr__` | 5 | `twistedcaldav/config.py` (×4), `txweb2/channel/http.py` (×2). |
| `def __setattr__` | 2 | `twistedcaldav/config.py`. |
| Module-global mutation (`Foo.bar = …`) | 57 | Class-registry wiring in DAL. |
| `@implementer` (modern) | 16 | Good. |
| `implements(` (deprecated) | **94** | Must migrate to `@implementer`. |
| `Interface` subclasses | 32 | zope.interface interfaces. |
| `from zope.interface import` | 71 imports | |
| `@inlineCallbacks` | 3,263 | |
| `returnValue(` | 1,826 | Must become `return x`. |
| Twisted `Values`/`ValueConstant` enums | 256 uses | Pre-`enum.Enum` pattern. |

### zope.interface + mypy

zope.interface has **no first-class mypy support**:

- `Interface` subclasses are not real classes; `Attribute("…")` is not a
  typed slot.
- `implements(IFoo)` / `@implementer(IFoo)` does not make `IFoo` a
  structural type mypy can verify.
- The 32 interfaces need a **manual Protocol mirror** if you want real
  checking, OR accept `Any` for anything flowing through an interface.

**Recommendation**: adopt the `zope.interface` → `typing.Protocol`
*dual-definition* pattern for the ~10 high-traffic "leaf" interfaces
(`IPropertyStore`, `ITransaction`, `IStoreNotifier`) and leave the rest as
`Any`-typed boundaries.

### Monkeypatching / class-registry wiring

The class-attribute mutation in `txdav/{cal,card}dav/datastore/sql*.py`
(`CalendarHome._externalClass = …`, `Calendar._objectResourceClass = …`)
is a runtime polymorphism mechanism. mypy will report "Class has no
attribute `_externalClass`" on the parent. Strategies:

1. Declare as `ClassVar[Optional[type[...]]] = None` on the parent.
2. Use `# type: ignore[attr-defined]` on each mutation line.
3. Refactor to constructor injection (best for Rust port; biggest diff).

---

## 4. Custom XML element system

`txdav/xml/base.py:118` — `WebDAVElement(object)` with class variables
`namespace`, `name`, `allowed_children`, `allowed_attributes`.

- `allowed_children` is a `dict` mapping **either a class or a
  (namespace, name) qname tuple** → `(min_count, max_count)` tuple.
  Heterogeneous keys — hostile to strict typing.
- `_elements_by_qname = {}` registry (`base.py:62`) populated by
  `@registerElement` (506 decorators across 18 modules).
- `WebDAVEmptyElement` (`base.py:560`) uses `__new__`-based singleton
  interning.
- `WebDAVElement.__init__(self, *children, **attributes)` — `**attributes`
  has no fixed keys; can only be typed as `**kwargs: object`.

**Typing strategy**: type `children` as `tuple[WebDAVElement | PCDATAElement]`,
`attributes` as `dict[str, object]`; don't try to type `allowed_children`
precisely (annotate as `ClassVar[dict[object, tuple[Optional[int],
Optional[int]]]]`). The `@registerElement` decorator typed as
`Callable[[type[_T]], type[_T]]` so it is transparent to mypy. ~3-5 days
of focused work; the subclass pattern is uniform.

---

## 5. Configuration system

`twistedcaldav/config.py` (381 LOC) + `twistedcaldav/stdconfig.py` (1,893
LOC):

- `ConfigDict(dict)` with `__getattr__`/`__setattr__` (`config.py:36-79`).
- `Config` singleton with lazy `self.update()` on every attribute access
  (`config.py:139-181`).
- `DEFAULT_CONFIG` is a 1,700-line dict literal (`stdconfig.py:157`).

**Why painful**: `ConfigDict` is a `dict` subclass whose keys are arbitrary
strings and values are themselves `ConfigDict`s (recursive). Every consumer
writes `config.Authentication.Kerberos.service` — chained attribute access
on a dict. mypy sees `Config.__getattr__` returning `Any`.

**Recommended approach**: keep `ConfigDict` typed as `dict[str, object]`
internally; author a parallel `TypedDict`/pydantic model that mirrors
`DEFAULT_CONFIG`; provide a typed accessor `def cfg() -> ConfigProtocol`.
Do NOT attempt to make `Config.__getattr__` type-safe — structurally
impossible. See [ADR-0013](../adr/0013-plist-and-toml-config-with-pydantic-schema.md).

---

## 6. Datastore / DAL

- **`twext.enterprise.dal`**: DALE SQL DSL. `syntax.py` (2,142 LOC, ~40
  classes) models SQL as an AST. `ColumnSyntax.__eq__`/`__gt__` are
  overloaded to return `Comparison` objects, not booleans — needs
  `@overload`.
- **`txdav/common/datastore/`**: CalDAV/CardDAV store built on DALE.
  `CommonHome` composed with many Mixins (`SharingHomeMixIn`,
  `DelegatesAPIMixin`, `GroupsAPIMixin`, `GroupCacherAPIMixin`,
  `imipAPIMixin`, `APNSubscriptionsMixin`).
- **Async**: uniformly `Deferred[T]`. 115 `@inlineCallbacks` in `sql.py`
  alone.
- **Schema**: `txdav/common/datastore/sql_schema/` contains `.sql` files
  parsed at startup. The schema is data, not code — a Rust reimplementation
  can ingest the same `.sql` files.

---

## 7. Test suite typing

- 203 test files, 131k LOC. `twisted.trial.unittest.TestCase`.
- Zero typing.

**Recommendation**: do NOT type the test suite in the first phase. Tests
are checked at runtime; ROI is low and volume is huge. Add per-module
override `ignore_errors = True` for test modules. In phase 11, opt-in
the large serialization tests (`test_icalendar.py` 12,213 LOC,
`test_sql.py` 10,225 LOC) which exercise Rust port contracts.

---

## 8. Recommended mypy configuration

### Prerequisite

A Python 3 port MUST come first. The tree has 290 `except X, e:` clauses
(syntax error in Py3), 209 `iteritems`, 187 `unicode(`, 50 `cStringIO`,
pinned `Twisted==16.6.0` (pre-typing). Twisted's typing begins with 21.7.

### Target `pyproject.toml` `[tool.mypy]` shape

```toml
[tool.mypy]
python_version = "3.12"
namespace_packages = true
explicit_package_bases = true
mypy_path = "$MYPY_CONFIG_FILE_DIR/stubs"
strict_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_return_any = true
disallow_untyped_defs = false   # gate via per-module
disallow_incomplete_defs = false

# Phase-1 strictly-typed modules:
[[tool.mypy.overrides]]
module = ["txdav.xml.*"]
disallow_untyped_defs = true
disallow_incomplete_defs = true
disallow_untyped_decorators = true
check_untyped_defs = true

[[tool.mypy.overrides]]
module = ["pycalendar.*"]
strict = true

[[tool.mypy.overrides]]
module = ["twistedcaldav.config"]
disallow_untyped_defs = true

# Phase-2:
[[tool.mypy.overrides]]
module = ["txdav.common.datastore.sql_tables", "txdav.common.datastore.sql_util"]
strict = true

[[tool.mypy.overrides]]
module = ["twext.enterprise.dal.*"]
strict = true

# Hostile-to-typing modules — leave lenient:
[[tool.mypy.overrides]]
module = ["twistedcaldav.stdconfig"]
ignore_errors = true

[[tool.mypy.overrides]]
module = ["twisted.plugins.caldav"]
ignore_errors = true

# Tests — opt-in only:
[[tool.mypy.overrides]]
module = ["*test*"]
ignore_errors = true

# Sibling packages — must ship py.typed markers:
[[tool.mypy.overrides]]
module = ["twext.*"]
ignore_missing_imports = false

[[tool.mypy.overrides]]
module = ["pycalendar.*"]
ignore_missing_imports = false
```

### `py.typed` markers

Add empty `py.typed` files to:

- `py-ccs-calendarserver/txdav/py.typed`
- `py-ccs-calendarserver/txweb2/py.typed`
- `py-ccs-calendarserver/twistedcaldav/py.typed`
- `py-ccs-calendarserver/calendarserver/py.typed`
- `ccs-twistedextensions/twext/py.typed`
- `ccs-pycalendar/src/pycalendar/py.typed`

### Stub dependencies

- `types-psutil`, `types-python-dateutil`, `types-pyOpenSSL`,
  `types-cryptography`, `types-setuptools`, `types-pyasn1`.
- Twisted ships its own inline stubs since 21.7 — no `twisted-stubs` needed.
- zope.interface: write `stubs/zope/interface/__init__.pyi` shim
  (`Interface`, `Attribute`, `implementer`, `implements`,
  `directlyProvides` as `Any`-typed).
- `xattr`, `memcache` — stub locally or use `types-python-memcached`.

---

## 9. Phased rollout (11 phases)

| Phase | Modules | Why this order |
|---|---|---|
| 0 | Py3 port + Twisted upgrade | Hard prerequisite |
| 1 | `ccs-pycalendar` (entire) | Pure, no Twisted, no zope; best ROI; first PyO3 candidate |
| 2 | `txdav/xml/base.py` + `txdav/xml/rfc*.py` | Self-contained, uniform pattern (§4) |
| 3 | `twistedcaldav/config.py` | Small (381 LOC), high-value (with TypedDict overlay) |
| 4 | `txdav/common/datastore/sql_tables.py`, `sql_util.py`, `icommondatastore.py`, `idav.py` (with Protocol mirrors) | Store contracts |
| 5 | `twext/enterprise/dal/{model,parseschema,record,syntax}.py` | DALE; needed before typing the big DAL |
| 6 | `txdav/common/datastore/sql.py` (5k LOC) + the two `txdav/{cal,card}dav/datastore/sql.py` | The big DAL |
| 7 | `txweb2/iweb.py` (Protocol mirrors) + `txweb2/http_headers.py` | HTTP primitives |
| 8 | `txweb2/dav/resource.py` + `twistedcaldav/resource.py` + `twistedcaldav/storebridge.py` | DAV resource layer |
| 9 | `twistedcaldav/caldavxml.py` + `carddavxml.py` + `customxml.py` | 98 element classes |
| 10 | `twistedcaldav/ical.py` + `twistedcaldav/stdconfig.py` | Last; wrap external complexity |
| 11 | Opt-in test modules | Only contract tests guarding the Rust port |

### Typing in parallel with the Py3 port

Phase 1 (ccs-pycalendar) and Phase 2 (txdav/xml) typing SHOULD begin in
parallel with the Py3 port. These modules are PyO3 port candidates; typing
them early produces the Rust contract. Some annotations will need revision
as the port progresses, but the structural benefit outweighs the rework
cost.

---

## 10. PyO3 / Rust migration preparation

Strict typing pays off for the Rust port in three ways: (a) documents the
contract; (b) lets us mechanically extract `dataclass`/`TypedDict` shapes
that map 1:1 to Rust `serde` structs; (c) catches `None`-where-you-expected-
`str` bugs that would otherwise surface as Rust panics.

### Tier 1 — Rust port candidates (pure data / parsing / protocol logic, no Twisted I/O)

| Module | Why |
|---|---|
| `ccs-pycalendar/src/pycalendar/**` | iCalendar/vCard/TZif parser, pure, 25.9k LOC. **Best first PyO3 target.** |
| `txdav/xml/base.py` + `txdav/xml/rfc*.py` | XML element model + 506 element classes. Maps to Rust `enum WebDavElement`. |
| `twistedcaldav/caldavxml.py` / `carddavxml.py` | CalDAV/CardDAV-specific element classes. |
| `txdav/common/datastore/sql_tables.py` + `sql_schema/*.sql` | Schema is declarative SQL — Rust can use `diesel`/`sqlx`. |
| `twext/enterprise/dal/{model,parseschema}.py` | Schema parser; small, pure. |
| `twistedcaldav/dateops.py` | Date arithmetic, pure given a `DateTime` type. |

### Tier 2 — Logic ports (typed but coupled to Twisted's `Deferred`)

| Module | Notes |
|---|---|
| `txdav/common/datastore/sql.py` + `txdav/{cal,card}dav/datastore/sql.py` | DAL business logic. Port to Rust + `tokio` once contracts typed. |
| `twext/enterprise/dal/syntax.py` + `record.py` | DALE SQL AST. Could become Rust macro/DSL; or replace with `sqlx`. |
| `twistedcaldav/ical.py` | Calendar object wrapper; mostly delegates. |
| `txdav/common/datastore/sql_sharing.py`, `sql_directory.py`, `sql_notification.py` | Sharing/delegation/notification logic. |

### Tier 3 — Keep as Python (too Twisted-entangled)

| Module | Why it stays |
|---|---|
| `txweb2/channel/http.py`, `txweb2/http.py`, `txweb2/server.py` | HTTP I/O loop — replace wholesale with Rust `hyper`/`axum`. |
| `twistedcaldav/resource.py`, `txweb2/dav/resource.py` | DAV resource tree tied to Twisted request/response. |
| `twistedcaldav/storebridge.py` (3,934 LOC) | Adapter; disappears in unified Rust server. |
| `twisted/plugins/caldav.py`, `calendarserver/tap/caldav.py` | Twisted plumbing; replaced by Rust `clap`+`tokio` main. |
| `twistedcaldav/stdconfig.py` + `config.py` | Replace with `serde`-deserialised TOML in Rust. |
| `twext/who/{ldap,opendirectory}` | Directory backends; port only if keeping same deps. |

### How strict typing accelerates the port

1. **Protocol extraction**: every `Interface`/`Protocol` becomes a Rust
   `trait` definition.
2. **`TypedDict` → `serde::Deserialize` struct**: config overlay and DALE
   row types translate mechanically.
3. **`Optional[T]` → `Option<T>`**: forces `None` handling that Python
   hides and Rust requires.
4. **`Final` / `Literal`**: the 256 `Values`/`ValueConstant` enum sites
   become Rust `enum`s with exhaustive `match`.
5. **`@overload`**: overloaded entry points document the exact Rust
   `From<X> for …` impls needed.

---

## References

- `HACKING.rst:224` (`__all__` convention)
- `HACKING.rst:243` (inlineCallbacks convention)
- `HACKING.rst:269` (camelCase convention)
- `HACKING.rst:350` (always-Deferred convention — gold for typing)
- `txdav/idav.py:34, 65-100` (zope.interface definitions)
- `txdav/xml/base.py:62, 118, 130, 193-208, 560-572` (XML element system)
- `twistedcaldav/config.py:36-79, 139-181, 381` (ConfigDict/Config)
- `twistedcaldav/stdconfig.py:157` (DEFAULT_CONFIG)
- `txdav/common/datastore/sql.py:32-72, 87` (DALE imports, implements)
- `ccs-twistedextensions/twext/enterprise/dal/syntax.py` (2,142 LOC DALE)
- [ADR-0003](../adr/0003-mission-py3-revival-as-typed-contract.md) — typed Python as contract
- [ADR-0005](../adr/0005-mypy-strict-per-module-rollout.md) — typing strategy
- PEP 484: https://peps.python.org/pep-0484/
- PEP 544 (Protocols): https://peps.python.org/pep-0544/
- PEP 561 (py.typed): https://peps.python.org/pep-0561/
