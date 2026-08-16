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

# ADR-0012: OAuth/OIDC primary auth; legacy Basic/Digest deprecated; drop pycrypto

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

The current authentication stack is outdated and has security liabilities:

### Current auth mechanisms

| Mechanism | Current state | Issues |
|---|---|---|
| Basic auth | `txweb2/auth/basic.py:58` uses `response.decode('base64')` — `decode("base64")` codec removed in Py3. | RFC 2617 only; needs RFC 7617 (UTF-8). |
| Digest auth | `txweb2/auth/digest.py:27` implements **RFC 2617 only** (qop=auth/md5). | No SHA-256 (`algorithm=SHA-256`), no `charset`, no `userhash` per RFC 7616. |
| Kerberos/SPNEGO | `twistedcaldav/authkerb.py:41` (`import kerberos`); `ccs-pykerberos` C extension. | C extension; macOS-coupled; optional. |
| OAuth/Bearer | **Not implemented.** `rg "OAuth|OIDC|Bearer|JWT"` returns nothing. | **Biggest functional gap** for modern clients (iOS/macOS now use OAuth for third-party CalDAV; DAVx5 and most modern clients expect OAuth). |

### Security liabilities

- **`pycrypto==2.6.1`** (`requirements-twisted-default.txt:18`,
  `setup.py:316`): unmaintained since 2013; has CVEs. Used by
  `txdav/caldav/datastore/scheduling/ischedule/dkim.py:31-33`
  (`Crypto.Hash.SHA`, `Crypto.PublicKey.RSA`, `Crypto.Signature.PKCS1_v1_5`),
  `calendarserver/tools/dkimtool.py:20`, plus `Crypto.pct_warnings.
  PowmInsecureWarning` in 4 admin tools. All have direct `cryptography`
  equivalents.
- **No OAuth 2.0 Bearer token support** (RFC 6750/6749): modern clients
  (iOS/macOS Calendar, DAVx5) expect OAuth for third-party CalDAV providers.
  This is the single biggest functional gap.
- **Digest auth is RFC 2617 only**: no SHA-256 per RFC 7616.

### iSchedule and DKIM

- `txdav/caldav/datastore/scheduling/ischedule/dkim.py` implements DKIM
  signing/verifying for the iSchedule HTTP protocol. Algorithms: `rsa-sha1`
  and `rsa-sha256` (`dkim.py:84`). SHA-1 SHOULD be removed; Ed25519
  (RFC 8463) is not supported.
- iSchedule itself was never published as an RFC (`draft-desruisseaux-
  ischedule-05` is the last public draft). It is a dead spec; the
  equivalent standardized path is RFC 6638 + iMIP. See
  [RFC_COMPLIANCE.md](../audit/RFC_COMPLIANCE.md) §5.

## Alternatives Considered

### Option A — Keep current auth; add OAuth as an extra

Keep Basic (RFC 2617), Digest (RFC 2617), Kerberos as-is. Add OAuth/Bearer
as an additional option.

**Pros**: Minimal disruption to existing deployments.
**Cons**: Leaves the `pycrypto` security liability; leaves Digest at RFC 2617
(no SHA-256); leaves `decode("base64")` codec issue in Basic auth; does not
modernize the auth stack.

### Option B — Replace all auth; drop Basic/Digest/Kerberos

Drop Basic, Digest, Kerberos entirely. Implement OAuth 2.0/OIDC as the sole
auth mechanism.

**Pros**: Cleanest; most modern; smallest auth surface.
**Cons**: Breaks all existing deployments that use Basic/Digest; many
clients still use Basic auth (especially for dev/test); Kerberos is needed
in some enterprise AD environments.

### Option C — Add OAuth/OIDC as primary; keep Basic/Digest as deprecated legacy; replace pycrypto; make Kerberos optional

Add OAuth 2.0/OIDC Bearer auth (RFC 6750/6749) as the primary auth
mechanism. Keep Basic (RFC 7617, UTF-8) and Digest (RFC 7616, SHA-256) as
deprecated legacy. Replace `pycrypto` with `cryptography`. Make Kerberos/
SPNEGO optional via `gssapi` PyPI adapter (drop `ccs-pykerberos` C extension
as default).

**Pros**: Modern primary auth (OAuth/OIDC); legacy auth preserved for
compatibility but marked deprecated; `pycrypto` security liability removed;
Kerberos is optional without C extension by default.
**Cons**: Some auth surface complexity (3 mechanisms); OAuth/OIDC
implementation is non-trivial.

## Decision

We adopt **Option C: add OAuth/OIDC as primary; keep Basic/Digest as
deprecated legacy; replace pycrypto with cryptography; make Kerberos optional
via gssapi.**

### Auth policy

- **OAuth 2.0/OIDC Bearer auth** (RFC 6750/6749) is the **primary** auth
  mechanism for phase 1.
  - Implement Bearer token validation per RFC 6750.
  - Support OAuth 2.0 authorization code flow per RFC 6749.
  - Support OIDC discovery per RFC 8414.
  - Token introspection per RFC 7662 (if using external OP).
  - The server MAY act as its own OAuth2/OIDC provider (like Stalwart does
    at `rs-stalwart/crates/http/src/auth/`) or defer to an external provider
    (Keycloak, Auth0, etc.). Phase 1: defer to external provider; phase 2:
    consider built-in provider.
- **Basic auth** (RFC 7617, UTF-8) is **deprecated legacy**. Fix
  `txweb2/auth/basic.py:58` (`decode("base64")` → `base64.b64decode`).
  Update to RFC 7617 (UTF-8 encoding). Mark as deprecated in docs; MAY be
  removed in a future major version.
- **Digest auth** (RFC 7616, SHA-256) is **deprecated legacy**. Update
  `txweb2/auth/digest.py:27` from RFC 2617 to RFC 7616 (add SHA-256
  algorithm, `charset=UTF-8`, `userhash`). Mark as deprecated in docs; MAY
  be removed in a future major version.
- **Kerberos/SPNEGO** (RFC 4559) is **optional** via `gssapi` PyPI adapter.
  - Drop `ccs-pykerberos` C extension as the default.
  - Provide a `gssapi`-based adapter in `twistedcaldav/authkerb.py` (~200 LOC
    rewrite of the `authGSSClientStep`/`authGSSServerStep` state machine).
  - Install via `ccs-calendarserver[kerberos]` extra.
  - Keep the `ccs-pykerberos` C extension as a fallback only if performance
    demands (unlikely).

### pycrypto replacement

- Replace `pycrypto==2.6.1` with `cryptography>=43.0` (already a dependency
  per ADR-0009).
- `txdav/caldav/datastore/scheduling/ischedule/dkim.py:31-33`:
  - `Crypto.Hash.SHA` → `cryptography.hazmat.primitives.hashes.SHA1()`
  - `Crypto.PublicKey.RSA` → `cryptography.hazmat.primitives.asymmetric.rsa`
  - `Crypto.Signature.PKCS1_v1_5` → `cryptography.hazmat.primitives.
    asymmetric.padding.PKCS1v15`
- `calendarserver/tools/dkimtool.py:20`: same replacements.
- Remove `Crypto.pct_warnings.PowmInsecureWarning` from admin tools.
- Remove `pycrypto` from `requirements-twisted-default.txt:18` and
  `setup.py:316`.

### iSchedule and DKIM

- iSchedule is dropped (never standardized; see
  [RFC_COMPLIANCE.md](../audit/RFC_COMPLIANCE.md) §5). The DKIM code in
  `txdav/caldav/datastore/scheduling/ischedule/dkim.py` is removed with it.
- DKIM signing for outbound iMIP email (RFC 6376) is a **phase-2**
  enhancement (currently only iSchedule is signed, not iMIP). Use
  `cryptography` for RSA and Ed25519 (RFC 8463) signatures.

## Consequences

### Positive

- Modern primary auth (OAuth/OIDC) unblocks iOS/macOS/DAVx5 third-party
  CalDAV usage.
- `pycrypto` security liability removed; `cryptography` is maintained, typed,
  and has no known CVEs.
- Digest auth upgraded to RFC 7616 (SHA-256).
- Basic auth fixed for Py3 (`base64.b64decode`).
- Kerberos is optional without C extension by default; `gssapi` is
  pure-Python (CFFI to libgssapi).

### Negative

- OAuth/OIDC implementation is non-trivial; phase 1 defers to external
  provider (Keycloak, Auth0, etc.).
- Some auth surface complexity (3 mechanisms: OAuth, Basic, Digest).
- `gssapi` PyPI package needs libgssapi system library (but no C extension
  to build).

### Neutral

- iSchedule is dropped; its DKIM code is removed. DKIM for iMIP is a
  phase-2 enhancement.

### Risks

- **OAuth/OIDC complexity**: implementing a full OAuth2 server is out of
  scope for phase 1. Mitigation: defer to external provider; implement
  Bearer token validation only (RFC 6750); the server validates tokens
  against an external OP's introspection endpoint (RFC 7662) or JWKS.
- **`gssapi` API differs from `ccs-pykerberos`**: the
  `authGSSClientStep`/`authGSSServerStep` state machine does not map 1:1 to
  `gssapi`'s `SecurityContext` API. Mitigation: ~200 LOC rewrite of
  `twistedcaldav/authkerb.py`; run the Kerberos test suite
  (`ccs-pykerberos/tests/test_kerberos.py`) against the new adapter.
- **Digest SHA-256 client compat**: some older clients may not support
  SHA-256 Digest. Mitigation: offer both MD5 and SHA-256; prefer SHA-256
  when client supports it.

### Follow-ups

- Implement OAuth 2.0 Bearer token validation (RFC 6750) in a new
  `txweb2/auth/oauth.py` or `twistedcaldav/auth_oauth.py`.
- Fix `txweb2/auth/basic.py:58` (`decode("base64")` → `base64.b64decode`);
  update to RFC 7617.
- Update `txweb2/auth/digest.py:27` to RFC 7616 (SHA-256, charset, userhash).
- Replace `pycrypto` with `cryptography` in dkim.py and dkimtool.py.
- Rewrite `twistedcaldav/authkerb.py` to use `gssapi` PyPI adapter.
- Drop iSchedule (remove `txdav/caldav/datastore/scheduling/ischedule/`).
- Phase 2: DKIM signing for outbound iMIP email (RFC 6376, Ed25519 RFC 8463).

## References

- `txweb2/auth/basic.py:58` (`decode("base64")` codec removed in Py3)
- `txweb2/auth/digest.py:27` (RFC 2617 only)
- `twistedcaldav/authkerb.py:41` (`import kerberos`)
- `txdav/caldav/datastore/scheduling/ischedule/dkim.py:31-33` (pycrypto)
- `calendarserver/tools/dkimtool.py:20` (pycrypto)
- `requirements-twisted-default.txt:18` (`pycrypto==2.6.1`)
- `setup.py:316` (`pycrypto`)
- RFC 6750 (Bearer): https://www.rfc-editor.org/rfc/rfc6750.html
- RFC 6749 (OAuth 2.0): https://www.rfc-editor.org/rfc/rfc6749.html
- RFC 7617 (Basic UTF-8): https://www.rfc-editor.org/rfc/rfc7617.html
- RFC 7616 (Digest SHA-256): https://www.rfc-editor.org/rfc/rfc7616.html
- RFC 4559 (SPNEGO): https://www.rfc-editor.org/rfc/rfc4559.html
- RFC 7662 (Token Introspection): https://www.rfc-editor.org/rfc/rfc7662.html
- RFC 8414 (OIDC Discovery): https://www.rfc-editor.org/rfc/rfc8414.html
- `gssapi` on PyPI: https://pypi.org/project/gssapi/
- `cryptography` docs: https://cryptography.io/
- Stalwart OAuth2 implementation: `rs-stalwart/crates/http/src/auth/`
- [RFC_COMPLIANCE.md](../audit/RFC_COMPLIANCE.md) §5 — iSchedule/DKIM
- [ADR-0009](0009-standardize-tls-pyopenssl-cryptography.md) — TLS (cryptography dep)
