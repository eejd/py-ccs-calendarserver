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

# ADR-0005: mypy strict with per-module rollout; ship py.typed markers

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

The codebase has **zero typing**: 0 occurrences of `from typing import`, 0
type annotations, 0 `# type:` comments, 0 `py.typed` markers, 0 `mypy.ini`
across all three repos (`py-ccs-calendarserver`, `ccs-pycalendar`,
`ccs-twistedextensions`). The code is frozen at the 2017 Apple drop.

The user's goal is to migrate all core libraries to strictly-typed to:

1. Improve development cycle reliability.
2. Catch problems as we migrate to Python 3.14 (stricter checks).
3. Prepare the way for the future Rust (strictly typed) library migrations.

See [TYPING_PLAN.md](../audit/TYPING_PLAN.md) for the full typing-readiness
audit and [ADR-0003](0003-mission-py3-revival-as-typed-contract.md) for the
"typed Python as contract" principle.

### Prerequisite

mypy strict mode on Python 3.14 is not even theoretically possible until a
Python 3 port is done first. The tree has 290 `except X, e:` clauses (syntax
error in Py3), 209 `iteritems`/`itervalues`/`iterkeys`, 187 `unicode(`
builtins, 50 `cStringIO` imports, and is pinned to `Twisted==16.6.0` (2016,
pre-typing). Twisted's typing story begins with Twisted 21.7 (2021).

Therefore, the Py3 port (ADR-0004) and Twisted upgrade (ADR-0007) are hard
prerequisites. However, the typing rollout SHOULD begin in parallel with the
Py3 port for the contract-layer modules, as stated in ADR-0003.

## Alternatives Considered

### Option A — Global `--strict` from day one

Run `mypy --strict` on the entire codebase after the Py3 port.

**Pros**: Maximum type safety; no configuration complexity.
**Cons**: Would produce ~50k errors on day one and demoralize the effort;
some modules (config, twisted.plugins, zope.interface-heavy interfaces) are
structurally hostile to strict typing and would never pass without
`# type: ignore` carpet-bombing.

### Option B — No strict typing; annotations only where convenient

Add type annotations opportunistically; never enforce strict mode.

**Pros**: Low effort; no mypy configuration.
**Cons**: Does not satisfy the user's goal of "strictly-typed core
libraries"; does not catch `None`-where-you-expected-`str` bugs; does not
serve as a reliable Rust contract; annotations drift and become
untrustworthy.

### Option C — Per-module strict overrides with phased rollout

Use mypy's `[[mypy-module]]` per-module strict overrides. Start with
`strict = True` on a small set of contract-layer modules (PyO3 port
candidates); expand the strict set over 11 phases. Ship `py.typed` markers
for all packages. Leave hostile modules (config, plugins, tests) lenient
or `ignore_errors`.

**Pros**: pragmatic; produces clean, well-typed modules one at a time; the
contract layer is typed first, serving the Rust port; demotivating
error floods are avoided.
**Cons**: some configuration complexity; non-strict modules remain
untrustworthy until opted in; requires discipline to expand the strict set
over time.

## Decision

We adopt **Option C: per-module strict overrides with phased rollout.**

### Configuration shape

`pyproject.toml` (or `mypy.ini`) uses per-module `[[tool.mypy.overrides]]`
sections (or `[mypy-module]` sections in `mypy.ini`). See
[TYPING_PLAN.md](../audit/TYPING_PLAN.md) §8 for the full target
configuration.

### `py.typed` markers

Add empty `py.typed` files to:

- `py-ccs-calendarserver/txdav/py.typed`
- `py-ccs-calendarserver/txweb2/py.typed`
- `py-ccs-calendarserver/twistedcaldav/py.typed`
- `py-ccs-calendarserver/calendarserver/py.typed`
- `ccs-twistedextensions/twext/py.typed`
- `ccs-pycalendar/src/pycalendar/py.typed`

And add `package_data={"py.typed"}` to each package's build config.

### Stub dependencies

- `types-psutil`, `types-python-dateutil`, `types-pyOpenSSL`,
  `types-cryptography`, `types-setuptools`, `types-pyasn1`.
- Twisted ships its own inline stubs since 21.7 — no `twisted-stubs` package
  needed after the Twisted upgrade (ADR-0007).
- zope.interface has no upstream stubs. We MUST write a small
  `stubs/zope/interface/__init__.pyi` declaring `Interface`, `Attribute`,
  `implementer`, `implements`, `directlyProvides` as `Any`-typed escape
  hatches, then refine the ~10 high-traffic interfaces into
  `typing.Protocol` mirrors.
- `xattr`, `memcache` — stub locally or use `types-python-memcached`.

### Phased rollout (11 phases)

See [TYPING_PLAN.md](../audit/TYPING_PLAN.md) §8 for the full 11-phase
rollout. Summary:

| Phase | Modules | Why this order |
|---|---|---|
| 0 | Py3 port + Twisted upgrade | Hard prerequisite |
| 1 | `ccs-pycalendar` (entire) | Pure, no Twisted, no zope; best ROI; first PyO3 candidate |
| 2 | `txdav/xml/base.py` + `txdav/xml/rfc*.py` | Self-contained, uniform pattern |
| 3 | `twistedcaldav/config.py` | Small, high-value |
| 4 | `txdav/common/datastore/sql_tables.py`, `sql_util.py`, `icommondatastore.py`, `idav.py` | Store contracts |
| 5 | `twext/enterprise/dal/{model,parseschema,record,syntax}.py` | DALE; needed before typing the big DAL |
| 6 | `txdav/common/datastore/sql.py` + `txdav/{cal,card}dav/datastore/sql.py` | The big DAL |
| 7 | `txweb2/iweb.py` + `txweb2/http_headers.py` | HTTP primitives |
| 8 | `txweb2/dav/resource.py` + `twistedcaldav/resource.py` + `twistedcaldav/storebridge.py` | DAV resource layer |
| 9 | `twistedcaldav/caldavxml.py` + `carddavxml.py` + `customxml.py` | 98 element classes |
| 10 | `twistedcaldav/ical.py` + `twistedcaldav/stdconfig.py` | Last; wrap external complexity |
| 11 | Opt-in test modules | Only contract tests guarding the Rust port |

### Typing in parallel with the Py3 port

Phase 1 (ccs-pycalendar) and Phase 2 (txdav/xml) typing SHOULD begin in
parallel with the Py3 port, because:

- These modules are PyO3 port candidates; typing them early produces the
  Rust contract.
- They are pure-logic / uniform-pattern modules; typing is straightforward.
- Some annotations will need revision as the port progresses, but the
  structural benefit outweighs the rework cost.

### Hostile-to-typing modules (leave lenient)

- `twistedcaldav/stdconfig.py` — config is a dynamic dict with `__getattr__`;
  structurally impossible to type. Use a `TypedDict` overlay for the typed
  view; leave `ConfigDict.__getattr__` as `Any`. See
  [ADR-0013](0013-plist-and-toml-config-with-pydantic-schema.md).
- `twisted/plugins/caldav.py` — Twisted plugin system uses
  `reflect.namedClass`; mypy cannot statically resolve. `ignore_errors`.
- Test modules — `ignore_errors = True` until opted in (phase 11).

## Consequences

### Positive

- The contract layer (PyO3 port candidates) is typed first, serving the
  Rust port.
- Per-module strict avoids demotivating error floods.
- `py.typed` markers enable downstream type-checking for consumers.
- zope.interface `Protocol` mirrors provide real structural checking for
  the ~10 high-traffic interfaces.
- Consistent always-return-a-`Deferred` discipline (`HACKING.rst:350`) means
  `Deferred[T]` annotations are honest.

### Negative

- Non-strict modules remain untrustworthy until opted in; discipline
  required to expand the strict set.
- zope.interface + mypy has no first-class support; manual `Protocol`
  mirrors are needed for the ~10 high-traffic interfaces.
- Class-registry mutation in the DAL (`CalendarHome._externalClass = ...`)
  will need `ClassVar` declarations or `# type: ignore[attr-defined]`.

### Neutral

- Tests are not typed in phase 1 (131k LOC of tests); the ROI is low and
  the volume is huge. Opt-in in phase 11.

### Risks

- **Typing becomes a bottleneck**: strict typing of legacy code can be slow.
  Mitigation: per-module strict; type the contract layer first; leave
  hostile modules lenient.
- **zope.interface stubs drift**: if zope.interface changes its API, our
  stubs may lag. Mitigation: pin `zope.interface` version; monitor upstream.
- **Twisted stubs incomplete**: Twisted's inline stubs are good for
  `twisted.internet.defer` but less complete for `twisted.mail`,
  `twisted.names`. Mitigation: stub locally where needed.

### Follow-ups

- Write `stubs/zope/interface/__init__.pyi` shim.
- Write `py.typed` markers for all 6 packages.
- Write `mypy.ini` / `pyproject.toml` `[tool.mypy]` configuration.
- Begin Phase 1 (ccs-pycalendar) typing in parallel with the Py3 port.

## References

- [TYPING_PLAN.md](../audit/TYPING_PLAN.md) — full typing-readiness audit +
  mypy.ini target + 11-phase rollout + PyO3 preparation
- [ADR-0003](0003-mission-py3-revival-as-typed-contract.md) — "typed Python
  as contract" principle
- [ADR-0004](0004-target-python-3.12-drop-py2.md) — Py3 prerequisite
- [ADR-0007](0007-twisted-16-to-24-upgrade.md) — Twisted upgrade (unblocks
  `Deferred[T]` typing)
- `HACKING.rst:350` — "If a callable is going to return a Deferred some of
  the time, it should return a deferred all of the time."
- PEP 484 (type hints): https://peps.python.org/pep-0484/
- PEP 544 (Protocols): https://peps.python.org/pep-0544/
- PEP 561 (py.typed): https://peps.python.org/pep-0561/
- mypy strict mode guide: https://mypy.readthedocs.io/en/latest/stable_mypy_code.html
