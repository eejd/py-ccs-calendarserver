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

# ADR-0014: Segregate Apple extensions behind capability flag

- **Status**: Accepted
- **Date**: 2026-08-16
- **Phase**: 1
- **Decision owner**: Project lead

## Context

The ccs-calendarserver implements ~20 Apple-specific, never-standardized
extensions in the `calendarserver:` XML namespace and `X-CALENDARSERVER-*` /
`X-APPLE-*` iCalendar properties. These are woven throughout the codebase:

- Compliance tokens advertised at `twistedcaldav/caldavxml.py:49-69, 1230,
  1350, 1369` and `twistedcaldav/customxml.py:42-90` and
  `twistedcaldav/carddavxml.py:42`:
  `calendarserver-private-events`, `calendarserver-private-comments`,
  `calendarserver-principal-property-search`, `calendarserver-principal-
  search`, `calendarserver-sharing`, `calendarserver-sharing-no-scheduling`,
  `calendarserver-group-sharee`, `calendarserver-partstat-changes`,
  `calendarserver-group-attendee`, `calendarserver-home-sync`,
  `calendarserver-recurrence-split`, `calendar-proxy`,
  `calendarserver-push-transports`, etc.
- 12 Apple extension drafts in `doc/Extensions/` (`caldav-ctag`,
  `caldav-notifications`, `caldav-privatecomments`, `caldav-privateevents`,
  `caldav-proxy`, `caldav-pubsubdiscovery`, `caldav-recursplit`,
  `caldav-schedulingchanges`, `caldav-sharing`, `calendarserver-bulk-
  change`, `icalendar-maskuids`).

### Research findings (macOS 26/27 client support)

Based on DAVx5/dav4jvm source code analysis (the most widely-deployed
non-Apple client), SOGo/Baikal/Nextcloud compatibility matrices, and
iCloud server behavior stability (full research in
[RFC_COMPLIANCE.md](../audit/RFC_COMPLIANCE.md) §Apple extensions):

**The only Apple-origin extensions DAVx5 actually parses are:**
- `calendarserver:getctag`
- `calendar-proxy` (read-for / write-for)
- `calendarserver:source` / `subscribed`
- `apple.com/ns/ical:calendar-color`

**Everything else is ignored by every major non-Apple client**, but is
still used by Apple Calendar (macOS 26/27) and by iCloud's own server.

### The problem

These extensions are interleaved with standards-compliant code throughout
`twistedcaldav/`, `txdav/`, and `txweb2/dav/`. There is no way to deploy
a "standards-only" server. Some users want a pure standards-compliant
deployment; others need Apple compatibility. The current architecture
forces both.

## Alternatives Considered

### Option A — Keep all extensions in place; no segregation

Leave the extensions where they are. Document them as non-standard.

**Pros**: Zero refactoring effort.
**Cons**: No way to deploy a "standards-only" server; extensions are
entangled with core code; difficult to audit standards compliance; future
removal is painful.

### Option B — Drop all non-standard extensions

Remove all `calendarserver:*` extensions and `X-APPLE-*`/`X-CALENDARSERVER-*`
properties. Be strictly standards-compliant.

**Pros**: Cleanest; pure standards compliance; smallest code surface.
**Cons**: Breaks Apple Calendar compatibility (iCloud sharing, notifications,
private events, push, etc.); many users rely on Apple Calendar; loses
hard-won compatibility work.

### Option C — Segregate into an `apple_extensions/` module behind a capability flag

Move all Apple-specific extensions into a separate `apple_extensions/`
module. Gate them behind a capability flag (`config.EnableAppleExtensions =
True`). When the flag is off, the server advertises only standards-compliant
compliance tokens and does not register Apple XML elements. When the flag is
on, the server advertises Apple extensions and registers Apple XML elements.

**Pros**: Server deployable as "standards-only" or "standards + Apple";
clear separation; easy to audit standards compliance; future removal is
possible; preserves Apple Calendar compatibility for those who need it.
**Cons**: Refactoring effort to move extensions into the module; some
extensions are deeply entangled with core code (e.g. sharing is woven into
the datastore); the `@registerElement` decorator registration needs to be
conditional.

## Decision

We adopt **Option C: segregate Apple-specific extensions into an
`apple_extensions/` module behind a capability flag.**

### Policy

- All Apple-specific extensions MUST be moved into a new
  `apple_extensions/` package (or subpackages within existing packages,
  clearly marked).
- A capability flag `config.EnableAppleExtensions` (default: `True` for
  backwards compatibility) controls whether Apple extensions are registered
  and advertised.
- When `EnableAppleExtensions = False`:
  - The server advertises only standards-compliant compliance tokens
    (CalDAV RFC 4791, CardDAV RFC 6352, WebDAV RFC 4918, scheduling RFC
    6638, sync RFC 6578, ACL RFC 3744, etc.).
  - Apple XML elements are NOT registered with `@registerElement`.
  - Apple DAV properties are NOT returned by PROPFIND.
  - Apple push transports are NOT advertised.
- When `EnableAppleExtensions = True` (default):
  - All Apple extensions are registered and advertised as before.
  - Compliance tokens include the `calendarserver:*` extensions.
- `X-APPLE-*` and `X-CALENDARSERVER-*` iCalendar properties MUST be
  preserved round-trip (Tier 3 below) regardless of the flag, per the
  robustness principle. Unknown X- properties MUST NOT be stripped.

### Tier classification

Based on the macOS 26/27 research (see
[RFC_COMPLIANCE.md](../audit/RFC_COMPLIANCE.md) §Apple extensions):

#### Tier 1 — keep in the standards-compliant core path

(Also consumed by non-Apple clients like DAVx5.)

| Extension | Notes |
|---|---|
| `calendarserver:getctag` | Prefer `sync-token` per RFC 6578 when available; getctag is the cheap "did anything change?" probe. Universally served. |
| `calendar-proxy` (read/write + reverse pointers) | Apple-origin I-D, never RFC; but DAVx5 parses it; wider-than-Apple in practice. |
| `calendarserver:source` / `subscribed` | Subscribed/external calendars; DAVx5 parses both. |
| `apple.com/ns/ical:calendar-color` | Emitted on every iCloud calendar; DAVx5 parses it; de-facto standard. |

#### Tier 2 — keep, but isolate in `apple_extensions/` behind the flag

(Only consumed by Apple Calendar and Apple-compat servers like SOGo/Baikal.)

| Extension | Notes |
|---|---|
| `calendarserver:sharing` (+ sharing-no-scheduling, group-sharee) | Apple's delegated/shared-calendar wire protocol. |
| `calendarserver:notifications` | In-app notification feed. |
| `calendarserver:private-events` | Collection-level per-event visibility in shared calendars. |
| `calendarserver:private-comments` + `X-CALENDARSERVER-PRIVATE-COMMENT` | Per-event private comments. |
| `X-CALENDARSERVER-PRIVATE-EVENT` | Per-event "private to me" flag. |
| `calendarserver:principal-property-search` | Attendee-autocomplete. |
| `calendarserver:principal-search` | Broader principal search. |
| `calendarserver:recurrence-split` | Server-side recurrence splitting for Outlook/Exchange interop. |
| `calendarserver:home-sync` | Bulk sync of entire calendar-home. |
| `calendarserver:bulk-change` | Single POST to modify many resources. |
| `calendarserver:partstat-changes` | Bulk attendee-status update. |
| `calendarserver:group-attendee` | LDAP/server-side group as ATTENDEE. |
| `calendarserver:calendar-availability` | Per-user VAVAILABILITY container property. |
| `calendarserver:calendar-transp` | Per-calendar transparency. |
| `push-transports` / `pushkey` | Apple Push (APNs); clearly label "Apple Push"; add open WebDAV-Push I-D alongside for DAVx5-class clients. |

#### Tier 3 — preserve round-trip only, no semantics

(Matches DAVx5's "unknown properties are retained" rule.)

| Extension | Notes |
|---|---|
| `X-APPLE-STRUCTURED-LOCATION` | Emitted by current Calendar.app on geocoded locations. Optionally map to/from RFC 9254 `VLOCATION` if adopted. |
| All other `X-APPLE-*` / `X-CALENDARSERVER-*` iCalendar properties | Do not implement semantics; never strip. |

#### Tier 4 — candidates for dropping (if confirmed unused)

None of the listed extensions can be definitively dropped on the evidence
available. The most fragile are `recurrence-split`, `bulk-change`,
`partstat-changes`, and `group-attendee` (niche even within Apple's flow;
closest to having RFC 6638 replacements). If surface area must shrink, drop
those four first behind the Tier-2 flag and verify against a live iCloud
account.

### Implementation approach

- Create `apple_extensions/` package with submodules for each Tier-2
  extension group (sharing, notifications, private-events, push, etc.).
- Use conditional `@registerElement` registration: the Apple XML element
  classes are registered only when `config.EnableAppleExtensions` is True.
  This likely requires a registration deferral mechanism (register elements
  at config-load time rather than import time).
- Move Tier-2 compliance token advertisement from `caldavxml.py:49-69` into
  `apple_extensions/` and conditionally append to the compliance list.
- Tier-1 extensions stay in core code (they are consumed by non-Apple
  clients).
- Tier-3 (X- properties) are handled by the parser's "preserve unknown"
  behavior — no code change needed beyond ensuring the parser does not
  strip unknown X- properties.

## Consequences

### Positive

- Server deployable as "standards-only" or "standards + Apple".
- Clear separation of standards-compliant code from Apple extensions.
- Easy to audit standards compliance (when the flag is off).
- Future removal of Apple extensions is possible (if Apple deprecates them).
- Preserves Apple Calendar compatibility for those who need it.
- Tier-1 extensions (used by DAVx5) stay in core.
- Tier-3 (X- properties) are preserved round-trip regardless of the flag.

### Negative

- Refactoring effort to move Tier-2 extensions into `apple_extensions/`.
- Some extensions are deeply entangled with core code (e.g. sharing is
  woven into the datastore at `txdav/common/datastore/sql_sharing.py`;
  push is woven into `calendarserver/push/`). Full segregation may require
  significant refactoring.
- Conditional `@registerElement` registration needs a deferral mechanism.

### Neutral

- The `apple_extensions/` module is Apache-2.0 licensed like the rest of
  the codebase (ADR-0002).

### Risks

- **Entanglement**: some Tier-2 extensions (sharing, push) are deeply
  woven into core code and may not be cleanly separable without significant
  refactoring. Mitigation: phase 1 segregation is best-effort; the flag
  controls advertisement and registration, not necessarily full code
  physical separation. Full physical separation may wait for phase 2/3.
- **Conditional registration complexity**: the `@registerElement` decorator
  runs at import time; conditional registration requires deferral.
  Mitigation: implement a registration deferral mechanism (register elements
  at config-load time).
- **False "standards-only" claim**: if Tier-1 extensions (getctag,
  calendar-proxy, etc.) are considered non-standard by some strict
  interpretation. Mitigation: Tier-1 extensions are widely deployed and
  parsed by DAVx5; they are de-facto standards even if not IETF standards.
  Document this clearly.

### Follow-ups

- Create `apple_extensions/` package structure.
- Implement conditional `@registerElement` registration deferral.
- Move Tier-2 compliance token advertisement into `apple_extensions/`.
- Add `config.EnableAppleExtensions` flag (default: True).
- Document the tier classification in `apple_extensions/README.md`.
- Phase 2: consider full physical separation of entangled extensions
  (sharing, push).

## References

- `twistedcaldav/caldavxml.py:49-69, 1230, 1350, 1369` (compliance tokens)
- `twistedcaldav/customxml.py:42-90` (Apple extension XML elements)
- `twistedcaldav/carddavxml.py:42` (CardDAV compliance)
- `doc/Extensions/` (12 Apple extension drafts)
- `txdav/common/datastore/sql_sharing.py` (sharing entanglement)
- `calendarserver/push/` (push entanglement)
- DAVx5/dav4jvm source: `https://github.com/bitfireAT/dav4jvm`
- [RFC_COMPLIANCE.md](../audit/RFC_COMPLIANCE.md) §Apple extensions — full
  macOS 26/27 research with per-extension status table
- [AUDIT.md](../audit/AUDIT.md) §RFC compliance — extension inventory
