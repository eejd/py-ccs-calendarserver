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

# ADR-0006: pyproject.toml + uv + ruff + pytest

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

The current build/dependency/lint/test tooling is outdated:

- `py-ccs-calendarserver/setup.py` (502 lines) uses `file()` builtin, `iteritems()`,
  py2 classifiers, and pins `Twisted==16.6.0`. See `setup.py:179,297,373,453,189-190`.
- 7 separate `requirements-*.txt` files with hand-maintained nested-indentation
  dependency trees (not standard pip syntax — it's a visualization that pip
  flattens). Versions are from 2016-2017.
- `pyflakes` is the linter (`requirements-dev.txt:2`); no formatter configured.
- Tests use `twisted.trial` exclusively (102 files import `twisted.trial`).
- `ccs-pycalendar/setup.py:17` uses `distutils.core.setup` — removed in Py3.12+.
- `ccs-caldavtester` and `ccs-caldavclientlibrary` also use `setup.py` with
  py2 constructs.

The Python packaging ecosystem has standardized on `pyproject.toml` (PEP 517/
518/621) and modern tooling (`uv`, `ruff`, `pytest`).

## Alternatives Considered

### Option A — Keep setup.py + requirements-*.txt + pyflakes + trial

Minimal modernization: fix the py2 constructs in setup.py, bump versions,
keep the existing structure.

**Pros**: Smallest diff; no new tooling to learn.
**Cons**: `setup.py` is deprecated (PEP 517/621 recommend `pyproject.toml`);
`distutils` is removed in Py3.12+ (blocks `ccs-pycalendar`); `pyflakes` is
slow and limited (no formatting, no import sorting); `trial` has worse
ergonomics than `pytest` (parametrization, fixtures, reporting); the
nested `requirements-*.txt` files are non-standard and fragile.

### Option B — Poetry + black + flake8 + pytest

Adopt Poetry as the dependency manager and build backend; black for
formatting; flake8 for linting; pytest for tests.

**Pros**: Mature ecosystem; Poetry is widely used.
**Cons**: Poetry is slow for large dependency trees; `black` + `flake8` +
`isort` is three separate tools where `ruff` does all three in one
Rust-based tool; Poetry's lockfile format is non-standard (not PEP 660
`uv.lock`-style).

### Option C — pyproject.toml (PEP 621) + uv + ruff + pytest

Adopt `pyproject.toml` as the single source of project metadata and
dependencies; `uv` (astral, Rust-based) for environment/dependency/build
frontend; `ruff` (astral, Rust-based) for linting + formatting (replaces
pyflakes + flake8 + isort + black); `pytest` + `pytest-twisted` for tests
(with `twisted.trial` underneath for async Deferred support).

**Pros**: `uv` is dramatically faster than pip/poetry; `ruff` replaces
4 tools with one; `pyproject.toml` is the standard; `pytest` + `pytest-twisted`
gives modern test ergonomics while preserving `twisted.trial` async support;
`uv lock` produces a PEP 660-style lockfile for reproducible CI.
**Cons**: `uv` is newer (but backed by astral, the same team as `ruff`);
`pytest-twisted` has some quirks but is the de-facto standard for Twisted
testing in 2026.

## Decision

We adopt **Option C: `pyproject.toml` + `uv` + `ruff` + `pytest`.**

### Policy

- `setup.py` MUST be replaced by `pyproject.toml` (PEP 621 metadata). A
  minimal `setup.py` MAY be kept as a shim for legacy `pip install -e .`
  workflows, but the source of truth is `pyproject.toml`.
- All 7 `requirements-*.txt` files MUST be replaced by dependency groups in
  `pyproject.toml` (`[project.optional-dependencies]` for extras like
  `ldap`, `postgres`, `kerberos`, `dev`) and a `uv.lock` lockfile for
  reproducible CI.
- Dependency versions MUST use unpinned upper bounds
  (e.g. `twisted>=24.3,<25`, `cryptography>=43,<45`) rather than exact pins.
  The lockfile pins exact versions for reproducibility.
- `ruff` MUST be configured in `pyproject.toml` `[tool.ruff]` for linting
  (replaces `pyflakes` + `flake8` + `isort`) and formatting (replaces
  `black`).
- `pytest` + `pytest-twisted` MUST be the test runner. Existing
  `twisted.trial.unittest.TestCase` subclasses work unchanged under
  `pytest-twisted`.
- `ccs-pycalendar/setup.py` MUST switch from `distutils.core` to
  `setuptools` or `hatchling` backend, configured via `pyproject.toml`.

### Tooling versions (phase 1)

- `uv` (latest stable)
- `ruff` (latest stable, ≥0.6)
- `pytest` ≥8.0
- `pytest-twisted` ≥1.14
- `pytest-cov`, `pytest-xdist` (parallel), `pytest-hypothesis`
- Python ≥3.12 (per ADR-0004)

### CI integration

GitHub Actions jobs use `uv` to install dependencies from `uv.lock` and run
`ruff check`, `ruff format --check`, `mypy`, and `pytest`. See ADR-0015 for
the full CI job matrix.

## Consequences

### Positive

- `uv` is dramatically faster than pip/poetry (Rust-based); CI install times
  drop significantly.
- `ruff` replaces 4 tools (pyflakes, flake8, isort, black) with one;
  lint+format runs in milliseconds.
- `pyproject.toml` is the standard; downstream tooling (mypy, ruff, uv,
  IDEs) all read it natively.
- `uv lock` produces reproducible CI builds.
- `pytest` + `pytest-twisted` gives modern test ergonomics (parametrization,
  fixtures, reporting) while preserving `twisted.trial` async support.
- `ccs-pycalendar` `distutils` blocker is resolved.

### Negative

- Contributors must learn `uv` and `ruff` (low barrier; both are
  pip/flake8-like in UX).
- `uv.lock` is a new file to maintain (but `uv lock --upgrade` handles it).
- Some legacy `bin/` shell scripts (`bin/develop`, `bin/run`) reference the
  old `requirements-*.txt` structure and MUST be updated.

### Neutral

- `twisted.trial` remains the underlying async test infrastructure;
  `pytest-twisted` is a thin adapter.

### Risks

- **`uv` maturity**: `uv` is newer than pip/poetry but is backed by astral
  (the `ruff` team) and has broad adoption as of 2026. Low risk.
- **`pytest-twisted` quirks**: `@pytest_twisted.inlineCallbacks` has some
  edge cases with fixtures. Mitigation: use `twisted.trial.unittest.TestCase`
  for complex async tests; `pytest` for simple ones.
- **Lockfile drift**: if contributors forget to run `uv lock` after
  changing dependencies. Mitigation: CI checks that `uv.lock` is up to date
  (`uv lock --check`).

### Follow-ups

- Write `pyproject.toml` for `py-ccs-calendarserver`, `ccs-pycalendar`,
  `ccs-twistedextensions`, `ccs-caldavtester`.
- Generate `uv.lock` for each.
- Configure `ruff` in `pyproject.toml` (lint rules + format settings).
- Update `bin/develop`, `bin/run`, `bin/test` shell scripts to use `uv`.
- Add CI job that verifies `uv.lock` is up to date.

## References

- `py-ccs-calendarserver/setup.py:179,297,373,453` (py2 constructs)
- `py-ccs-calendarserver/setup.py:189-190` (py2 classifiers)
- `ccs-pycalendar/setup.py:17` (`distutils.core.setup`)
- `py-ccs-calendarserver/requirements-*.txt` (7 files, 2016-era pins)
- PEP 517: https://peps.python.org/pep-0517/
- PEP 518: https://peps.python.org/pep-0518/
- PEP 621: https://peps.python.org/pep-0621/
- PEP 660: https://peps.python.org/pep-0660/
- `uv` docs: https://docs.astral.sh/uv/
- `ruff` docs: https://docs.astral.sh/ruff/
- `pytest-twisted`: https://github.com/pytest-dev/pytest-twisted
