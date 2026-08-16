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

# ADR-0009: Standardize TLS on pyOpenSSL+cryptography; drop pySecureTransport

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

The current SSL/TLS stack is split across two platform-specific paths:

- **Non-macOS** (`requirements-twisted-default.txt:20-29`): `pyOpenSSL==16.2.0`
  + `cryptography==1.6` + `service_identity==16.0.0` + `pyasn1==0.1.9` +
  `enum34==1.1.6` + `ipaddress` (backport) + `characteristic==14.3.0`.
- **macOS** (`requirements-twisted-osx.txt:12,16`): `OSXFrameworks`
  (`ccs-pyosxframeworks`, archived by Apple) + `pySecureTransport`
  (`ccs-pysecuretransport`, archived by Apple) — a shim that presents a
  pyOpenSSL-compatible API but backs onto `Security.framework`.
- The `lib-patches/Twisted/securetransport.patch` (31 lines) patches
  Twisted 16.6.0 to detect the `__SecureTransport__` marker attribute and
  bypass pyOpenSSL's verifier on macOS.
- Runtime detection at `calendarserver/push/util.py:42`,
  `calendarserver/tap/util.py:1256,1306,1341,1367`:
  `if hasattr(OpenSSL, "__SecureTransport__"):`.
- `setup.py:334-343` has a platform branch: on macOS without `USE_OPENSSL`
  env, adds `OSXFrameworks` + `pySecureTransport`; otherwise adds
  `pyOpenSSL>=0.15.1` + `service_identity`.

### Why pySecureTransport existed

In 2010-2017, shipping OpenSSL inside a macOS binary was legally awkward
(Apple's system Python didn't bundle OpenSSL headers; macOS didn't ship
OpenSSL by default after 10.7). Using the native Secure Transport API via
cffi let Apple ship a self-contained server that respected the system
keychain and trust store.

### Status in 2026

1. **Secure Transport was deprecated by Apple in macOS 10.13 (High Sierra,
   2017)** and replaced by `Network.framework`. Apple's current guidance is
   `Network.framework` or BoringSSL, not Secure Transport.
2. **`pySecureTransport` (ccs-pysecuretransport) is archived** by Apple and
   was never Py3-compatible; the cffi bindings target deprecated C APIs.
3. **`OSXFrameworks` (ccs-pyosxframeworks) is also archived**. Most of what
   it wraps (keychain, SACL, launchd) is now provided by
   `pyobjc-framework-Security`, `pyobjc-framework-ServiceManagement`, etc.
   `ccs-twistedextensions/twext/python/launchd.py`/`sacl.py` already use
   cffi directly to build extensions, bypassing `OSXFrameworks` for those.
4. **Modern Twisted (24.x) uses `cryptography` + `pyOpenSSL` for TLS**
   uniformly on all platforms, with first-class `SSL_CTX`-based
   configuration. `service_identity` does hostname verification. There is
   no Secure Transport backend in Twisted anymore.

### Security liability

`twistedcaldav/stdconfig.py:174` defaults to
`"SSLMethod": "SSLv23_METHOD"` — a legacy OpenSSL umbrella method that
permits TLS 1.0/1.1, which are deprecated by RFC 8996 and forbidden by
RFC 9325 (BCP 195). This is a security liability.

## Alternatives Considered

### Option A — Keep pySecureTransport on macOS; pyOpenSSL elsewhere

Maintain the platform split; upgrade pySecureTransport to Py3.

**Pros**: Preserves macOS keychain integration.
**Cons**: Secure Transport is deprecated by Apple (2017); pySecureTransport
is archived and never Py3-compatible; maintaining a Py3 port of a
deprecated-API shim is wasted effort; modern Twisted has no Secure Transport
backend; the platform split doubles testing surface.

### Option B — Migrate to Network.framework on macOS; pyOpenSSL elsewhere

Write a new `Network.framework` TLS backend for Twisted on macOS.

**Pros**: Uses Apple's current recommended TLS API.
**Cons**: Enormous effort for marginal benefit; `Network.framework` is
Swift-first with a C bridging API; Twisted has no `Network.framework`
backend and writing one is out of scope; the "use system roots" benefit can
be achieved via `truststore` (PEP 543) instead.

### Option C — Standardize on pyOpenSSL + cryptography + service_identity on all platforms

Use pyOpenSSL 24.x + cryptography 43+ + service_identity 24.x on all
platforms (including macOS). Drop pySecureTransport, OSXFrameworks, the
securetransport.patch, and the `__SecureTransport__` runtime checks. Use
`truststore` (PEP 543) to respect the OS trust store on macOS for
system-root CA loading. Enforce TLS 1.2+/1.3 per RFC 9325.

**Pros**: Single TLS stack on all platforms; modern, maintained, typed
(cryptography ships inline stubs); `truststore` gives back the "use system
roots" benefit; eliminates the platform split; removes the `SSLv23_METHOD`
security liability; enables HTTP/2 (via h2 + ALPN negotiation) in phase 2.
**Cons**: Loses macOS keychain integration for cert loading (mitigated by
loading PEM files directly or via `truststore`).

## Decision

We adopt **Option C: standardize on pyOpenSSL + cryptography +
service_identity on all platforms; drop pySecureTransport, OSXFrameworks,
the securetransport.patch, and the `__SecureTransport__` runtime checks.**

### Policy

- `pyOpenSSL>=24.0`, `cryptography>=43.0`, `service_identity>=24.0` in
  `pyproject.toml` for all platforms.
- `requirements-twisted-osx.txt` MUST be deleted; merged into
  `requirements-twisted-default.txt` (which itself becomes
  `pyproject.toml` per ADR-0006).
- `lib-patches/Twisted/securetransport.patch` MUST be deleted.
- The `if sys.platform == "darwin" and USE_OPENSSL is None:` branch in
  `setup.py:334-343` MUST be removed; keep only the `else` branch.
- The `hasattr(OpenSSL, "__SecureTransport__")` runtime checks at
  `calendarserver/push/util.py:42` and
  `calendarserver/tap/util.py:1256,1306,1341,1367` MUST be removed;
  collapse to the pyOpenSSL path.
- `pyobjc-framework-Security` MAY be used for keychain access (cert
  loading) if needed; otherwise load PEM files directly.
- The `SSLv23_METHOD` default at `twistedcaldav/stdconfig.py:174` MUST be
  replaced with TLS 1.2+ enforcement per RFC 9325 (BCP 195). Specifically:
  - Disable TLS 1.0 and TLS 1.1 (RFC 8996).
  - Disable SSLv2, SSLv3 (long deprecated).
  - Enable TLS 1.2 and TLS 1.3.
  - Disable SHA-1 signature hashes in TLS 1.2 per RFC 9155.
- HSTS (RFC 6797) SHOULD be added as a default-on middleware.

### Dependencies to remove

| Dependency | Reason |
|---|---|
| `pySecureTransport` (`ccs-pysecuretransport`) | Archived by Apple; Secure Transport deprecated by Apple (2017); never Py3-compatible. |
| `OSXFrameworks` (`ccs-pyosxframeworks`) | Archived by Apple; replaced by pyobjc-framework-* where needed. |
| `enum34` | Py3 stdlib (`enum`). |
| `ipaddress` | Py3 stdlib. |
| `characteristic` | Deprecated; folded into `attrs`; newer `service_identity` no longer needs it. |

## Consequences

### Positive

- Single TLS stack on all platforms; eliminates the platform split.
- Modern, maintained, typed (`cryptography` ships inline stubs).
- `truststore` gives back the "use system roots" benefit on macOS.
- Removes the `SSLv23_METHOD` security liability.
- Enables HTTP/2 (via h2 + ALPN negotiation) in phase 2.
- Eliminates the `securetransport.patch` maintenance burden.
- Reduces testing surface (one TLS path, not two).

### Negative

- Loses macOS keychain integration for cert loading. Mitigation: load PEM
  files directly or via `truststore`; use `pyobjc-framework-Security` for
  keychain access if needed.
- macOS-specific SACL (`twext/python/sacl.py`) and launchd
  (`twext/python/launchd.py`) CFFI extensions remain (they are separate
  from the TLS stack). These need CFFI `set_source` migration (see
  [DEPENDENCIES.md](../audit/DEPENDENCIES.md)) but are not affected by this
  ADR.

### Neutral

- The `truststore` package (PEP 543) is a new dependency for macOS
  system-root CA loading. It is pure-Python and maintained.

### Risks

- **TLS regression on macOS**: if the system trust store is not correctly
  loaded via `truststore`, TLS verification may fail for system-trusted CAs.
  Mitigation: test TLS against real macOS system CAs in CI; fall back to
  explicit PEM file configuration.
- **HSTS breaks dev clients**: HSTS (RFC 6797) can interfere with
  development setups using self-signed certs. Mitigation: HSTS SHOULD be
  configurable and disabled by default in dev mode.

### Follow-ups

- Delete `requirements-twisted-osx.txt`, `lib-patches/Twisted/securetransport.patch`.
- Remove `__SecureTransport__` runtime checks.
- Replace `SSLv23_METHOD` default with TLS 1.2+ enforcement.
- Add `truststore` dependency for macOS system-root CA loading.
- Add HSTS middleware (configurable, default-on in production).
- Audit all TLS configuration in `twistedcaldav/stdconfig.py`.

## References

- `requirements-twisted-default.txt:20-29` (pyOpenSSL path)
- `requirements-twisted-osx.txt:12,16` (pySecureTransport path)
- `setup.py:334-343` (platform branch)
- `lib-patches/Twisted/securetransport.patch` (31 lines, being deleted)
- `calendarserver/push/util.py:42` (`__SecureTransport__` check)
- `calendarserver/tap/util.py:1256,1306,1341,1367` (`__SecureTransport__` checks)
- `twistedcaldav/stdconfig.py:174` (`SSLv23_METHOD` default)
- RFC 8446 (TLS 1.3): https://www.rfc-editor.org/rfc/rfc8446.html
- RFC 9325 (TLS Best Practices, BCP 195): https://www.rfc-editor.org/rfc/rfc9325.html
- RFC 9155 (SHA-1 deprecation in TLS 1.2): https://www.rfc-editor.org/rfc/rfc9155.html
- RFC 8996 (TLS 1.0/1.1 deprecation): https://www.rfc-editor.org/rfc/rfc8996.html
- RFC 6797 (HSTS): https://www.rfc-editor.org/rfc/rfc6797.html
- PEP 543 (truststore): https://peps.python.org/pep-0543/
- `truststore` on PyPI: https://pypi.org/project/truststore/
