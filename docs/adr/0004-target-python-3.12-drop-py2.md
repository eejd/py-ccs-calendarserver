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

# ADR-0004: Target Python 3.12+; drop Python 2 entirely

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

The ccs-calendarserver codebase is Python 2.7-only:

- `py-ccs-calendarserver/setup.py:189-190` declares
  `"Programming Language :: Python :: 2.7"` and
  `"Programming Language :: Python :: 2 :: Only"`.
- `HACKING.rst:107` states "We require Python 2.6 or higher."
- 741 .py files, ~324k LOC, with ~277 files (37%) containing at least one
  Python-2-only construct.
- Pinned `Twisted==16.6.0` (2016, pre-Py3).
- `ccs-pycalendar/setup.py` uses `distutils.core.setup` — `distutils` was
  removed in Python 3.12 and is gone in 3.14.

The user's goal is to migrate to modern Python, benefiting from stricter
checks as we target Python 3.14. See
[AUDIT.md](../audit/AUDIT.md) §1 for the full Python-2-idiom inventory.

## Alternatives Considered

### Option A — Python 3.8+ (conservative, maximum ecosystem compat)

Target Python 3.8 as the minimum, since it's still in many distros.

**Pros**: Widest deployment reach; `typing.get_type_hints` works;
`from __future__ import annotations` available.
**Cons**: 3.8 is end-of-life (October 2024); no `zoneinfo` (added 3.9);
no `match` statement (3.10); no `Self` type (3.11); no PEP 695 type
parameters (3.12); misses the user's explicit 3.14 target and the
"stricter checks" benefit.

### Option B — Python 3.11+

Target Python 3.11 as the minimum.

**Pros**: `Self` type, `ExceptionGroup`, `tomllib` in stdlib, fast CPython
(speedups from 3.11).
**Cons**: Still misses PEP 695 (3.12) and deferred annotation evaluation
(PEP 649/749, 3.14); the user explicitly wants 3.14.

### Option C — Python 3.12+ (aiming 3.14)

Target Python 3.12 as the minimum, with CI testing on 3.12, 3.13, and 3.14
(when released). No Python 2 support; no `six` compat layer.

**Pros**: PEP 695 type parameter syntax; PEP 585 generic builtins; PEP 604
`X | Y` union; `zoneinfo` (3.9+); `tomllib` (3.11+); matches the user's
3.14 target; `distutils` already removed so the `ccs-pycalendar` setup.py
fix is forced; `twisted` ≥ 23.10 has full 3.12 support and inline type
hints.
**Cons**: 3.12 is newer than some distros ship; 3.14 is not yet released
(as of Aug 2026, 3.14 is in alpha/beta); `mypy` on 3.14 features may lag.

## Decision

We adopt **Option C: target Python 3.12+ as the minimum, with CI testing on
3.12, 3.13, and 3.14.** No Python 2 support; no `six` compat layer.

### Policy

- The `python_requires` in `pyproject.toml` MUST be `>=3.12`.
- CI MUST test against Python 3.12, 3.13, and 3.14 (when available).
- All `from __future__ import ...` statements MUST be removed (unnecessary
  on 3.12+).
- The `six` library MUST NOT be used; existing `six` usage
  (`txdav/base/datastore/dbapiclient.py:29`) MUST be removed.
- New code SHOULD use modern syntax: `X | Y` unions (PEP 604), generic
  builtins (`list[T]` not `List[T]`, PEP 585), `match` statements where
  appropriate, `type X =` aliases (PEP 695) where they improve readability.
- `setup.py` MUST be replaced by `pyproject.toml` (see
  [ADR-0006](0006-pyproject-toml-uv-ruff.md)).
- `ccs-pycalendar/setup.py` MUST switch from `distutils.core` to
  `setuptools` (or `hatchling`) — `distutils` is removed in 3.12+.

### Migration approach

The Py3 port is a mechanical + manual effort. See
[MIGRATION_PLAN.md](../audit/MIGRATION_PLAN.md) for the step-by-step
sequence. Summary:

- ~3,500-4,500 mechanical line edits across ~277 files (`iteritems`→`items`,
  `xrange`→`range`, `print`→`print()`, `except X, e:`→`except X as e:`,
  `raise X, msg`→`raise X(msg)`, `file()`→`open()`, `cPickle`→`pickle`,
  `cStringIO`→`io`, `unicode`→`str`, `basestring`→`str`,
  `urlparse`→`urllib.parse`, `urllib2`→`urllib.request/error/parse`).
- ~40-60 files needing substantive rewrite (txweb2 HTTP/bytes layer, plist
  tooling, Twisted upgrade churn across 3,263 `@inlineCallbacks` sites).
- Realistic estimate: 3-4 months for working Py3.12+ (with sibling repos
  ported in parallel); 4-6 months for full test-suite green.

## Consequences

### Positive

- Access to modern Python features: `zoneinfo`, `tomllib`, PEP 695 type
  parameters, PEP 604 unions, `match` statements, `Self` type.
- `distutils` removal forces the `ccs-pycalendar` setup.py fix (a blocker
  for 3.14).
- Twisted ≥ 23.10 has full 3.12 support and inline type hints, unblocking
  the typing strategy (ADR-0005).
- No `six` compat layer means cleaner code and one less dependency.
- Stricter checks (3.14 deferred annotations, improved error messages)
  catch bugs earlier.

### Negative

- 3.12 is newer than some LTS distros ship (Ubuntu 22.04 ships 3.10; Ubuntu
  24.04 ships 3.12). Users on older distros MUST use pyenv/uv to install
  3.12+.
- 3.14 is not yet released (as of Aug 2026); CI on 3.14 alpha/beta may have
  flaky failures. Mitigation: 3.12 is the CI gate; 3.13/3.14 are
  best-effort.
- The port is a substantial effort (see MIGRATION_PLAN.md).

### Neutral

- The `from __future__ import print_function` statements (95 files) become
  no-ops and are removed.

### Risks

- **Sibling repo ports block the main repo**: `ccs-pycalendar` and
  `ccs-twistedextensions` MUST be ported in lockstep. If they lag, the main
  server cannot import. Mitigation: port siblings first or in parallel;
  see MIGRATION_PLAN.md sequencing.
- **bytes/str boundary bugs in txweb2**: the HTTP/headers/stream layer
  assumes py2 `str`=`bytes`. This is the highest-risk area. Mitigation:
  dedicated engineer-weeks for the txweb2 bytes-boundary rewrite; see
  MIGRATION_PLAN.md §3.
- **Twisted 16→24 API churn**: 3,263 `@inlineCallbacks` sites; 1,826
  `returnValue()` calls; 94 `implements()` calls. Mitigation: mechanical
  review pass; see ADR-0007.

### Follow-ups

- ADR-0005: mypy strict typing (depends on Py3).
- ADR-0006: pyproject.toml + uv + ruff.
- ADR-0007: Twisted 16→24 upgrade.
- [MIGRATION_PLAN.md](../audit/MIGRATION_PLAN.md): step-by-step port.
- [DEPENDENCIES.md](../audit/DEPENDENCIES.md): dependency replacements.

## References

- Python 2 declaration: `py-ccs-calendarserver/setup.py:189-190`
- `HACKING.rst:107` ("We require Python 2.6 or higher")
- `ccs-pycalendar/setup.py:17` (`distutils.core.setup`)
- Python 3.12 What's New: https://docs.python.org/3/whatsnew/3.12.html
- Python 3.13 What's New: https://docs.python.org/3/whatsnew/3.13.html
- Python 3.14 What's New (draft): https://docs.python.org/3/whatsnew/3.14.html
- PEP 585 (generic builtins): https://peps.python.org/pep-0585/
- PEP 604 (`X | Y`): https://peps.python.org/pep-0604/
- PEP 695 (type parameters): https://peps.python.org/pep-0695/
- PEP 649/749 (deferred annotations): https://peps.python.org/pep-0649/
