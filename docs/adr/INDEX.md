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

# ADR Index

| Number | Title | Status | Date | Phase |
|--------|-------|--------|------|-------|
| [0001](0001-adopt-adr-methodology.md) | Adopt ADR methodology | Accepted | 2026-08-16 | ongoing |
| [0002](0002-license-apache-2.0.md) | License: Apache-2.0; vendored Apache/MIT deps only | Accepted | 2026-08-16 | 1 |
| [0003](0003-mission-py3-revival-as-typed-contract.md) | Mission: Py3 revival as typed contract for Rust+PyO3 | Accepted | 2026-08-16 | ongoing |
| [0004](0004-target-python-3.12-drop-py2.md) | Target Python 3.12+; drop Python 2 entirely | Accepted | 2026-08-16 | 1 |
| [0005](0005-mypy-strict-per-module-rollout.md) | mypy strict with per-module rollout; ship py.typed markers | Accepted | 2026-08-16 | 1 |
| [0006](0006-pyproject-toml-uv-ruff.md) | pyproject.toml + uv + ruff + pytest | Accepted | 2026-08-16 | 1 |
| [0007](0007-twisted-16-to-24-upgrade.md) | Twisted 16.6 → 24.x upgrade; asyncio reactor | Accepted | 2026-08-16 | 1 |
| [0008](0008-keep-txweb2-forked-phase-1.md) | Keep txweb2 forked for phase 1; plan staged migration in phase 2 | Accepted | 2026-08-16 | 1 |
| [0009](0009-standardize-tls-pyopenssl-cryptography.md) | Standardize TLS on pyOpenSSL+cryptography; drop pySecureTransport | Accepted | 2026-08-16 | 1 |
| [0010](0010-port-ccs-pycalendar-to-py3.md) | Port ccs-pycalendar to Py3 as canonical contract; defer calcard | Accepted | 2026-08-16 | 1 |
| [0011](0011-postgresql-sqlite-only-drop-oracle-opendirectory.md) | PostgreSQL + SQLite only; drop Oracle/OpenDirectory; ldap3 | Accepted | 2026-08-16 | 1 |
| [0012](0012-oauth-oidc-primary-auth.md) | OAuth/OIDC primary auth; legacy Basic/Digest deprecated; drop pycrypto | Accepted | 2026-08-16 | 1 |
| [0013](0013-plist-and-toml-config-with-pydantic-schema.md) | Plist + TOML config with pydantic/TypedDict schema | Accepted | 2026-08-16 | 1 |
| [0014](0014-segregate-apple-extensions.md) | Segregate Apple extensions behind capability flag | Accepted | 2026-08-16 | 1 |
| [0015](0015-compliance-testing-architecture.md) | Compliance testing architecture (Tier-0/1/2/3) | Accepted | 2026-08-16 | 1 |

## Status legend

- **Proposed** — drafted, not yet ratified.
- **Accepted** — ratified; in effect.
- **Deprecated** — still in effect but slated for removal; do not extend.
- **Superseded** — replaced by a later ADR; do not follow. The superseding ADR is linked in the ADR's Status line.
- **Rejected** — considered and rejected; not in effect.

## Supersede process

When a decision is revised:

1. Write a new ADR (next number) with full Context/Alternatives/Decision/Consequences.
2. In the new ADR's Context, link back to the superseded ADR.
3. Edit the old ADR's Status line to `Superseded by [ADR-MMMM](MMMM-...)`.
4. Do NOT modify the old ADR's body — it is immutable.
5. Update this INDEX.md with the new status and the new ADR row.

See [ADR-0001](0001-adopt-adr-methodology.md) for full process details.
