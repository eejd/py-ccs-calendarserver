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

# ADR-0013: Plist + TOML config with pydantic/TypedDict schema

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

The configuration system is centered on `twistedcaldav/config.py` (381 LOC)
and `twistedcaldav/stdconfig.py` (1,893 LOC):

### Current architecture

- **`ConfigDict(dict)`** (`config.py:36-79`): a dict subclass that supports
  attribute access (`config.ServerRoot`, `config.Authentication.Kerberos.
  service`). Uses `__getattr__`/`__setattr__`/`__delattr__` to route
  non-underscore names through dict items.
- **`Config`** (`config.py:139-181`): a singleton-ish wrapper with
  `__getattr__` that **lazily calls `self.update()` on every attribute
  access** if `_dirty` is set. Reads can trigger side-effects.
- **Global `config = Config()`** at `config.py:381`.
- **`DEFAULT_CONFIG`** (`stdconfig.py:157`): a 1,700-line dict literal — the
  schema of record.
- **`PListConfigProvider`** loads plist files via `plistlib.PlistParser`.
- ~25 `_pre*`/`_post*` hook functions registered via
  `config.addPreUpdateHooks(...)` — config mutations happen as side effects
  of the update cycle.
- **8 `.plist` files** in `conf/` (e.g. `conf/caldavd.plist`,
  `conf/caldavd-test.plist`).

### Python 2 plistlib APIs (all removed in Py3.4+, gone by 3.9)

| File | API | Status in Py3 |
|---|---|---|
| `calendarserver/tap/util.py:106` | `from plistlib import readPlist` | Removed — use `plistlib.load` |
| `calendarserver/tap/test/test_caldav.py:48` | `from plistlib import writePlist` | Removed — use `plistlib.dump` |
| `calendarserver/tools/agent.py:34` | `from plistlib import readPlistFromString, writePlistToString` | Removed — use `loads`/`dumps` |
| `calendarserver/tools/anonymize.py:34` | `from plistlib import readPlistFromString` | Removed |
| `calendarserver/tools/diagnose.py:24` | `from plistlib import readPlist, readPlistFromString` | Removed |
| `twistedcaldav/stdconfig.py:19` | `from plistlib import PlistParser` | Removed (gone by 3.9) |
| `twistedcaldav/dumpconfig.py:20` | `from plistlib import PlistWriter, _escapeAndEncode` | Removed — `_escapeAndEncode` was private; `PlistWriter` is now `_PlistWriter` |

`twistedcaldav/dumpconfig.py` is the **single most py2-entangled plist file**
— it subclasses `PlistWriter` to make an `OrderedPlistWriter`, calls the
private `_escapeAndEncode`, and checks `isinstance(..., unicode)`.

### Why this is painful to type

- `ConfigDict` is a `dict` subclass whose keys are arbitrary strings and
  whose values are themselves `ConfigDict`s (recursive). There is no static
  schema.
- Every consumer writes `config.Authentication.Kerberos.service` — chained
  attribute access on a dict. mypy sees `Config.__getattr__` returning
  `Any`, so the whole config tree is effectively `Any` no matter what.
- `Config.__getattr__` lazily calls `self.update()` — reads can trigger
  side-effects. mypy won't catch that but it's a runtime hazard.

## Alternatives Considered

### Option A — Keep plist, modernize the parser usage

Fix the py2 plistlib API usage (`readPlist`→`load`, `PlistParser`→`load`,
etc.). Keep `ConfigDict` as-is. Author a `TypedDict` overlay alongside for
typed access.

**Pros**: Backwards compatible with existing `.plist` deployments; minimal
disruption.
**Cons**: Plist is Apple-specific and unfamiliar to many users; `ConfigDict.
__getattr__` stays `Any`-typed; no validation.

### Option B — Migrate to TOML with a plist→TOML converter

Ship a converter; deprecate `.plist`; make TOML the default. Use
`pydantic`/`TomlDecoder` for typed access.

**Pros**: TOML is the modern Python config standard (`tomllib` in stdlib
3.11+); cleaner typing story; validation.
**Cons**: Breaks existing deployments unless they run the converter; loses
plist's ordered-dict semantics (but py3.7+ dicts are ordered natively).

### Option C — Support both plist and TOML via a format-agnostic loader

Support both formats via a format-agnostic loader. `ConfigDict` stays a
dict; the `pydantic`/`TypedDict` overlay applies to both. Strict checking
via `pydantic`.

**Pros**: No forced migration; existing `.plist` deployments keep working;
new deployments can use TOML; strict validation via `pydantic` for both.
**Cons**: More code (two parsers); but `plistlib` and `tomllib` are both
stdlib, so the incremental cost is small.

## Decision

We adopt **Option C: support both plist and TOML via a format-agnostic
loader; pydantic/TypedDict strict schema overlay.**

### Policy

- **Format-agnostic loader**: a `load_config(path)` function that detects
  format by file extension (`.plist` or `.xml` → plistlib; `.toml` →
  tomllib) and returns a `dict`.
- **plist modernization**: fix all py2 plistlib API usage:
  - `readPlist` → `plistlib.load`
  - `writePlist` → `plistlib.dump`
  - `readPlistFromString` → `plistlib.loads`
  - `writePlistToString` → `plistlib.dumps`
  - `PlistParser` → `plistlib.load`
  - `PlistWriter` / `OrderedPlistWriter` / `_escapeAndEncode` in
    `dumpconfig.py` → rewrite against modern `plistlib` API (which uses
    `dict` natively ordered since py3.7+; the `OrderedPlistWriter` hack is
    no longer needed).
- **TOML support**: add `tomllib` (stdlib 3.11+) for reading TOML config
  files. Add a sample `conf/caldavd-test.toml` alongside the existing
  `conf/caldavd-test.plist`.
- **pydantic schema overlay**: author a `pydantic` model (or `TypedDict`
  with `total=False`) that mirrors `DEFAULT_CONFIG` (`stdconfig.py:157`).
  This becomes the *typed* view. Provide a typed accessor
  `def cfg() -> CalendarServerConfig` (the pydantic model). Migrate call
  sites to it over time. The legacy `config.Foo.Bar` chain keeps working
  but stays `Any`-typed.
- **Validation**: `pydantic` validates the config dict at load time,
  catching type errors, missing required fields, and invalid values. This
  is a major improvement over the current silent-accept behavior.
- **`ConfigDict.__getattr__`**: stays `Any`-typed. It is structurally
  impossible to type chained attribute access on a dict. This is one of the
  few places where `Any` is the correct answer. See
  [TYPING_PLAN.md](../audit/TYPING_PLAN.md) §5.
- **Migration path**: existing `.plist` deployments keep working; new
  deployments SHOULD use TOML. A `calendarserver_config_convert` tool
  SHOULD be provided to convert `.plist` → `.toml`.

### `dumpconfig.py` rewrite

`twistedcaldav/dumpconfig.py` is the most py2-entangled plist file. It:

- Subclasses `PlistWriter` to make `OrderedPlistWriter`.
- Calls the private `_escapeAndEncode`.
- Checks `isinstance(..., unicode)`.

On Py3, `plistlib` uses `dict` natively ordered (since py3.7+), so the
`OrderedPlistWriter` hack is no longer needed. The rewrite:

- Use `plistlib.dump(config_dict, fp)` for plist output.
- Use `tomllib`/`tomli_w` for TOML output.
- Remove `isinstance(..., unicode)` checks (use `str`).
- Remove `_escapeAndEncode` usage (private API, no equivalent needed).

## Consequences

### Positive

- No forced migration; existing `.plist` deployments keep working.
- New deployments can use TOML (modern standard).
- `pydantic` validation catches config errors at load time.
- The `pydantic` model serves as a typed schema documentation.
- `dumpconfig.py` rewrite is cleaner (no `OrderedPlistWriter` hack).

### Negative

- Two config formats to support (but both are stdlib).
- `ConfigDict.__getattr__` stays `Any`-typed (structurally impossible to
  type; accepted trade-off).
- `pydantic` is a new dependency (but it is pure-Python, maintained, and
  widely used).

### Neutral

- The `~25 _pre*`/`_post*` hook functions in `stdconfig.py` remain; they
  are side-effect-driven config mutations. `pydantic` validation happens
  after hooks run, so hooks can still mutate the config before validation.

### Risks

- **pydantic model drift**: if `DEFAULT_CONFIG` changes but the pydantic
  model is not updated, validation may reject valid configs. Mitigation:
  generate the pydantic model from `DEFAULT_CONFIG` via a script, or keep
  them in lockstep in code review.
- **TOML limitations**: TOML does not support some plist constructs (e.g.
  nested arrays of dicts are verbose in TOML). Mitigation: most ccs-
  calendarserver config is flat enough for TOML; complex nested structures
  can use TOML tables.
- **`Config.__getattr__` side-effects**: the lazy `self.update()` on every
  attribute access is a runtime hazard. Mitigation: document this; consider
  refactoring to explicit `config.reload()` in phase 2.

### Follow-ups

- Fix all py2 plistlib API usage (7 files; see Context table).
- Rewrite `twistedcaldav/dumpconfig.py` against modern `plistlib`.
- Add `tomllib` support and a sample `conf/caldavd-test.toml`.
- Author `pydantic` model mirroring `DEFAULT_CONFIG`.
- Provide `calendarserver_config_convert` tool (plist → TOML).
- Add `pydantic` to `pyproject.toml` dependencies.

## References

- `twistedcaldav/config.py:36-79` (`ConfigDict` with `__getattr__`/`__setattr__`)
- `twistedcaldav/config.py:139-181` (`Config` with lazy-update `__getattr__`)
- `twistedcaldav/config.py:381` (`config = Config()` global)
- `twistedcaldav/stdconfig.py:157` (`DEFAULT_CONFIG = {` 1.7k-line dict literal)
- `twistedcaldav/stdconfig.py:19` (`from plistlib import PlistParser`)
- `twistedcaldav/dumpconfig.py:20` (`from plistlib import PlistWriter, _escapeAndEncode`)
- `calendarserver/tap/util.py:106` (`from plistlib import readPlist`)
- `calendarserver/tools/agent.py:34` (`readPlistFromString, writePlistToString`)
- PEP 519 (tomllib): https://peps.python.org/pep-0519/ (no — tomllib is PEP 680)
- PEP 680 (tomllib in stdlib): https://peps.python.org/pep-0680/
- `pydantic` docs: https://docs.pydantic.dev/
- [TYPING_PLAN.md](../audit/TYPING_PLAN.md) §5 — config typing analysis
- [MIGRATION_PLAN.md](../audit/MIGRATION_PLAN.md) §4 — plistlib migration
