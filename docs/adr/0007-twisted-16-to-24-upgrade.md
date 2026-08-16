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

# ADR-0007: Twisted 16.6 → 24.x upgrade; asyncio reactor

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

The codebase is pinned to `Twisted==16.6.0` (2016) — 8 years old, Python 2-only,
pre-typing. This is one of the highest-risk upgrades in the migration:

- `requirements-twisted-default.txt:5` and
  `requirements-twisted-osx.txt:5` pin `Twisted==16.6.0`.
- `setup.py:312` pins `Twisted==16.6.0`.
- The codebase has **3,263 `@inlineCallbacks`** decorators, **1,826
  `returnValue(...)`** calls, and **15,089 `yield`** statements — the entire
  async surface is Twisted-generator-style, not native coroutines.
- **94 `implements(IFoo)`** class-body calls — the deprecated zope.interface
  form removed in zope.interface 5.0.
- `twisted.web2` was removed from Twisted in 12.0 (2012); the project uses
  `txweb2`, a vendored fork (see ADR-0008).
- `twext.enterprise.adbapi2` (`ccs-twistedextensions/twext/enterprise/adbapi2.py`)
  is Apple's fork of `twisted.enterprise.adbapi` and MUST be re-based against
  the modern adbapi.
- `twisted.internet.kqreactor` is imported in twext (now integrated into
  Twisted differently).
- Twisted's typing story begins with Twisted 21.7 (Oct 2021); `Deferred[T]`
  is generic and parametrised; `inlineCallbacks` functions are typed as
  `Callable[..., Deferred[T]]` so `yield` of a `Deferred[X]` infers `X`.

The `lib-patches/Twisted/securetransport.patch` patches Twisted 16.6.0 to
support Apple's `pySecureTransport` SSL backend — this is being dropped (see
ADR-0009) so the patch is irrelevant on modern Twisted.

## Alternatives Considered

### Option A — Twisted 21.7 (minimum for typing)

Upgrade to Twisted 21.7, the first version with inline type hints.

**Pros**: Unblocks `Deferred[T]` typing (ADR-0005); minimal API churn from
16.6.
**Cons**: 21.7 is from 2021; misses 3.12 support improvements (Twisted
22.x+); misses async/await interop improvements (23.x+); still old.

### Option B — Twisted 22.x or 23.x

Upgrade to a recent-but-not-latest Twisted.

**Pros**: Good 3.12 support; `defer.fromCoroutine` / `defer.ensureDeferred`
for async/await interop; `twisted.internet.asyncioreactor` stable.
**Cons**: Not the latest; may miss security fixes and bug fixes in 24.x.

### Option C — Twisted 24.x+ (latest stable) + asyncio reactor

Upgrade to the latest stable Twisted (24.x or newer). Adopt
`twisted.internet.asyncioreactor` as the default reactor for forward asyncio
interop.

**Pros**: Latest security fixes and bug fixes; full 3.12/3.13 support;
best typing support; `asyncioreactor` enables using `aiohttp`/`httpx`/
`aiosqlite` where helpful; `defer.fromCoroutine` / `defer.ensureDeferred`
for modern async/await code.
**Cons**: Most API churn from 16.6; some deprecation warnings to address;
`twisted.web2` is long gone (but we use `txweb2` anyway, ADR-0008).

## Decision

We adopt **Option C: upgrade to Twisted 24.x+ (latest stable) and adopt
`twisted.internet.asyncioreactor` as the default reactor.**

### Policy

- `Twisted>=24.3,<25` (or newer) in `pyproject.toml`. Exact version pinned
  in `uv.lock`.
- The `lib-patches/Twisted/securetransport.patch` MUST be deleted
  (irrelevant on modern Twisted; see ADR-0009).
- `twext.enterprise.adbapi2` MUST be re-based against the modern
  `twisted.enterprise.adbapi` API.
- `twisted.internet.kqreactor` imports in twext MUST be updated or removed.
- The default reactor MUST be `twisted.internet.asyncioreactor` for forward
  asyncio interop. This enables `defer.fromCoroutine(coro)` /
  `defer.ensureDeferred(coro)` for modern async/await code.
- `requirements-twisted-default.txt` and `requirements-twisted-osx.txt`
  MUST be deleted (consolidated into `pyproject.toml`; see ADR-0006).

### Mechanical migration tasks

- **1,826 `returnValue(x)` → `return x`**: modern Twisted accepts `return x`
  inside `@inlineCallbacks`. This is a mechanical sed-style edit.
- **94 `implements(IFoo)` → `@implementer(IFoo)`**: the class-body form was
  removed in zope.interface 5.0. Mechanical edit.
- **`twisted.python.log` → `twisted.logger`**: the old logging module is
  deprecated. Mechanical but voluminous.
- **`twisted.python.constants.Values` / `ValueConstant`**: the pre-`enum.Enum`
  pattern (256 uses). SHOULD be migrated to `enum.Enum` over time, but not
  blocking.
- **SSL API changes**: `twisted.internet._sslverify` internal imports in
  `txweb2` MUST be replaced with the public `twisted.internet.ssl` API.

### asyncio reactor adoption

`twisted.internet.asyncioreactor` runs Twisted on top of the asyncio event
loop. This enables:

- Using `aiohttp`/`httpx`/`aiosqlite` where helpful (e.g., in tests or for
  HTTP/2 client support).
- `defer.fromCoroutine(coro)` / `defer.ensureDeferred(coro)` for modern
  async/await code.
- Future migration of `@inlineCallbacks` functions to `async def` + `await`
  (optional, not required in phase 1).

**Caveat**: The `asyncioreactor` has some limitations (e.g., signal handling
differences). If it causes issues, fall back to the default reactor. The
asyncio adoption is a SHOULD, not a MUST.

## Consequences

### Positive

- Unblocks `Deferred[T]` typing (ADR-0005).
- Latest security fixes and bug fixes.
- Full Python 3.12/3.13/3.14 support.
- `asyncioreactor` enables modern async/await interop.
- `defer.fromCoroutine` / `defer.ensureDeferred` bridge for future `async
  def` code.
- `returnValue(x)` → `return x` is cleaner and matches modern Twisted style.

### Negative

- Most API churn from 16.6; some deprecation warnings to address.
- 3,263 `@inlineCallbacks` sites need review (mostly mechanical).
- `twext.enterprise.adbapi2` re-base is non-trivial.
- `txweb2` may need fixes for internal `twisted.internet._sslverify` imports
  that moved.

### Neutral

- The `@inlineCallbacks` + `yield` pattern continues to work on modern
  Twisted; migration to `async def` + `await` is optional and deferred.

### Risks

- **`adbapi2` re-base breaks the DB layer**: `twext.enterprise.adbapi2` is
  Apple's fork with hooks, failover (`ConnectionPoolClient`), and
  `Commit`/`Rollback` primitives. The modern `adbapi` API may differ in
  subtle ways. Mitigation: re-base carefully; run the DB test suite
  (`txdav/common/datastore/test/`) against the re-based `adbapi2`.
- **`txweb2` internal imports break**: `txweb2/channel/http.py` and
  `txweb2/stream.py` may reach into `twisted.internet` internals that moved.
  Mitigation: dedicated engineer-weeks for txweb2 reconciliation (see
  ADR-0008).
- **`asyncioreactor` signal handling**: may differ from the default reactor
  on SIGINT/SIGTERM. Mitigation: test the server's signal handling; fall
  back to default reactor if issues arise.

### Follow-ups

- Re-base `twext.enterprise.adbapi2` against modern `twisted.enterprise.adbapi`.
- Fix `txweb2` internal `twisted.internet._sslverify` imports (see ADR-0008).
- Migrate 94 `implements()` → `@implementer` (mechanical).
- Migrate 1,826 `returnValue(x)` → `return x` (mechanical).
- Delete `lib-patches/Twisted/securetransport.patch` (ADR-0009).
- Delete `requirements-twisted-*.txt` (ADR-0006).
- Test `asyncioreactor` with the full server; fall back to default if
  issues arise.

## References

- `requirements-twisted-default.txt:5` (`Twisted==16.6.0`)
- `requirements-twisted-osx.txt:5` (`Twisted==16.6.0`)
- `setup.py:312` (`Twisted==16.6.0`)
- `lib-patches/Twisted/securetransport.patch` (31 lines, being deleted)
- `ccs-twistedextensions/twext/enterprise/adbapi2.py` (Apple's adbapi fork)
- Twisted typing (since 21.7): https://docs.twisted.org/en/stable/api/twisted.internet.defer.html
- `twisted.internet.asyncioreactor`: https://docs.twisted.org/en/stable/api/twisted.internet.asyncioreactor.html
- `defer.fromCoroutine`: https://docs.twisted.org/en/stable/api/twisted.internet.defer.html#fromCoroutine
- `defer.ensureDeferred`: https://docs.twisted.org/en/stable/api/twisted.internet.defer.html#ensureDeferred
- Twisted + asyncio interop guide: https://docs.twisted.org/en/stable/core/howto/asyncio-integration.html
