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

# ADR-0008: Keep txweb2 forked for phase 1; plan staged migration in phase 2

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

`txweb2` is a vendored fork of `twisted.web2` (93 .py files, 27.5k LOC). Per
`txweb2/__init__.py:27-37`:

> "txweb2: a transitional package for Calendar Server to move from a dependence
> on twisted.web2 to twisted.web. This is a copy of (most of) twisted.web2, but
> the intention is for this package to disappear and gradually get replaced
> with twisted.web. Features from this package are being merged into
> twisted.web. Once that is complete, this package will be removed."

The transition never happened before Apple archived the project. `txweb2`
contains: `http.py`, `http_headers.py` (1,787 LOC), `stream.py`, `server.py`,
`channel/http.py`, `auth/{basic,digest,tls,wrapper}.py`, `dav/*` (full WebDAV
method handlers: PROPFIND, PROPPATCH, MKCOL, COPY, MOVE, LOCK, ACL, reports),
`filter/{gzip,range,location}.py`, `client/http.py`, `metafd.py`,
`static.py`, `fileupload.py`.

It is imported by 100+ sites across `calendarserver/`, `twistedcaldav/`,
`txdav/`. Key entry points:

- `calendarserver/tap/caldav.py:79` imports `HTTPFactory`, `Site`, `Request`
  from `txweb2.channel.http` / `txweb2.server` — the server's main HTTP
  listener.
- Every CalDAV method module returns `txweb2.http.Response` /
  `JSONResponse` / `StatusResponse` / `HTTPError`.

`twisted.web` (the modern one) is NOT a drop-in replacement:

- `twisted.web` has no WebDAV support (`PROPFIND`, `MKCOL`, `LOCK`, `ACL`,
  reports). txweb2's `dav/` module is the only WebDAV layer and MUST stay.
- `txweb2.http_headers.Headers` is more RFC-compliant than
  `twisted.web.http_headers` was historically.
- `txweb2.channel/http.py`'s HTTP/1.1 parser with `metafd.py`
  (`ConnectionLimiter`, `ReportingHTTPService`) has no modern `twisted.web`
  equivalent.

txweb2 is also the **highest-risk area for the Py3 port** (per
[MIGRATION_PLAN.md](../audit/MIGRATION_PLAN.md) §3): native py2 `str`=bytes
assumption throughout the HTTP parser, header tokenizer, stream module.
Tuple-unpacking in params, `iteritems`, `cPickle`, `cStringIO`, old-style
classes, 2-arg raises.

## Alternatives Considered

### Option A — Replace txweb2 with twisted.web + a new WebDAV layer

Write a new WebDAV layer on top of `twisted.web.resource.Resource`; replace
txweb2's HTTP channel with `twisted.web.server.Site`.

**Pros**: Eliminates the txweb2 fork; uses maintained `twisted.web` HTTP
stack; sets up HTTP/2 support (via `twisted.web` + h2).
**Cons**: Enormous effort — 100+ import sites, 27.5k LOC of txweb2, plus a
new WebDAV layer; high risk of protocol-conformance regressions; the WebDAV
method handlers in `txweb2/dav/` are battle-tested and would need
reimplementation; blocks the phase-1 Py3 port timeline.

### Option B — Replace txweb2's HTTP channel with twisted.web; keep txweb2/dav

Use `twisted.web.server.Site` for the HTTP transport; keep `txweb2/dav/*`
(WebDAV method handlers) and `txweb2/http_headers.py` as a library.

**Pros**: Gets modern HTTP transport without re-writing WebDAV.
**Cons**: `txweb2.dav.*` expects `txweb2.http.Request`/`Response` types,
not `twisted.web.http.Request`; the adapter layer would be non-trivial;
still blocks the phase-1 Py3 port timeline.

### Option C — Keep txweb2 forked for phase 1; plan staged migration in phase 2

Port txweb2 to Python 3 in-place (fix the bytes/str boundary, remove py2
idioms, fix internal Twisted imports). Keep it as a forked, vendored
package. Plan a staged migration to `twisted.web` in phase 2 (or phase 3 as
part of the Rust port).

**Pros**: Lowest risk for phase 1; preserves the battle-tested WebDAV layer;
fastest path to a working Py3 server; the fork can be maintained
independently of upstream Twisted.
**Cons**: txweb2 remains a fork that must be maintained; no HTTP/2 support
in phase 1 (deferred to phase 2); the bytes/str boundary rewrite is still
high-risk engineer-weeks.

## Decision

We adopt **Option C: keep txweb2 forked for phase 1; plan staged migration
in phase 2.**

### Phase 1 policy

- txweb2 MUST be ported to Python 3 in-place. This is the highest-risk area
  of the Py3 port (see [MIGRATION_PLAN.md](../audit/MIGRATION_PLAN.md) §3).
- The bytes/str boundary in `txweb2/http_headers.py`, `txweb2/channel/http.py`,
  `txweb2/stream.py` MUST be carefully rewritten. In py2, `str`=`bytes`; in
  py3, `str`=`text` and `bytes`=`bytes`. HTTP headers and body content are
  bytes; header tokenization is text-after-decoding.
- Internal `twisted.internet._sslverify` imports MUST be replaced with the
  public `twisted.internet.ssl` API (see ADR-0007, ADR-0009).
- The `lib-patches/Twisted/securetransport.patch` logic in txweb2 (runtime
  `hasattr(OpenSSL, "__SecureTransport__")` checks at
  `calendarserver/push/util.py:42`, `calendarserver/tap/util.py:1256,1306,
  1341,1367`) MUST be removed (see ADR-0009).
- `metafd.py`'s use of `twisted.internet.abstract.FileDescriptor` internals
  that moved MUST be fixed.

### Phase 2 plan (deferred)

Staged migration to `twisted.web`:

1. Replace `txweb2.http.Response`/`StatusResponse`/`JSONResponse` with
   `twisted.web.http.Request`-based rendering.
2. Move resources to subclass `twisted.web.resource.Resource`.
3. Drop txweb2's HTTP channel/transport layer.
4. Keep `txweb2.dav.*` as a separate `txdav-webdav` library (since
   `twisted.web` has no WebDAV support).

This is a large mechanical change but each module is independent. Defer to
phase 2 or phase 3 (Rust port may supersede).

## Consequences

### Positive

- Lowest risk for phase 1; preserves the battle-tested WebDAV layer.
- Fastest path to a working Py3 server.
- The fork can be maintained independently of upstream Twisted.
- WebDAV method handlers (`txweb2/dav/*`) remain intact.

### Negative

- txweb2 remains a fork that must be maintained (27.5k LOC).
- No HTTP/2 support in phase 1 (deferred to phase 2).
- The bytes/str boundary rewrite is high-risk engineer-weeks (~3-5
  engineer-weeks per MIGRATION_PLAN.md).

### Neutral

- The phase-2 migration to `twisted.web` may be superseded by the phase-3
  Rust port (which would replace txweb2 wholesale with a Rust HTTP/DAV
  layer).

### Risks

- **bytes/str boundary bugs**: the HTTP parser, header tokenizer, and stream
  module assume py2 `str`=`bytes`. Incorrect py3 porting could cause
  subtle data corruption (e.g., non-ASCII header values, iCalendar body
  content). Mitigation: dedicated engineer-weeks; comprehensive test
  coverage for the HTTP layer; property-based testing for header parsing.
- **txweb2 fork rot**: if we keep txweb2 forked for too long, it drifts
  further from modern Twisted internals. Mitigation: phase-2 migration
  plan; the fork is a known temporary state.
- **`metafd.py` breakage**: `metafd.py` reaches into
  `twisted.internet.abstract.FileDescriptor` internals that may have moved.
  Mitigation: audit and fix during the Py3 port.

### Follow-ups

- Port txweb2 to Python 3 (bytes/str boundary, py2 idioms, internal Twisted
  imports). See [MIGRATION_PLAN.md](../audit/MIGRATION_PLAN.md) §3.
- Remove `__SecureTransport__` runtime checks (ADR-0009).
- Fix `metafd.py` `FileDescriptor` imports.
- Phase 2: staged migration to `twisted.web` (deferred).

## References

- `txweb2/__init__.py:27-37` (transitional package docstring)
- `txweb2/http_headers.py` (1,787 LOC — most bytes-critical file)
- `txweb2/channel/http.py` (HTTP/1.1 parser)
- `txweb2/stream.py` (byte stream module)
- `txweb2/dav/` (WebDAV method handlers)
- `txweb2/metafd.py` (ConnectionLimiter, ReportingHTTPService)
- `calendarserver/tap/caldav.py:79` (main HTTP listener entry point)
- [MIGRATION_PLAN.md](../audit/MIGRATION_PLAN.md) §3 — txweb2 bytes/str
  boundary rewrite
- [ADR-0007](0007-twisted-16-to-24-upgrade.md) — Twisted upgrade
- [ADR-0009](0009-standardize-tls-pyopenssl-cryptography.md) — TLS
  standardization (removes `__SecureTransport__` checks)
