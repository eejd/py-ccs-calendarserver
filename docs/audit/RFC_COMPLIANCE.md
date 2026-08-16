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

# RFC_COMPLIANCE — Per-RFC Scorecard

**Snapshot date**: 2026-08-16
**Target**: August 2026 RFC compliance for a CalDAV/CardDAV/WebDAV server.

This document is the per-RFC scorecard: what's implemented, where, gaps,
and errata. Companion to
[ADR-0014](../adr/0014-segregate-apple-extensions.md) (Apple extensions)
and [ADR-0015](../adr/0015-compliance-testing-architecture.md) (compliance
testing).

---

## 1. Current RFCs implemented/referenced

`doc/RFC/README.md:1-76` is the canonical "Specifications Relevant to
CalendarServer" list. Combined with the grep audit, the RFCs referenced in
code/docs are:

| Area | RFCs explicitly referenced |
|---|---|
| HTTP (legacy) | 7230-7237 (doc only); **code still cites 2068/2616/2617** as live references |
| HTTP misc | 2045-2047, 5789 (PATCH), 7240 (Prefer), 5785 (well-known), 4559 (SPNEGO) |
| WebDAV core | 2518 (NOT 4918 — see §3), 3253 (DeltaV), 3744 (ACL), 4331 (quota) |
| WebDAV ext | 5397, 5689, 5842, 5995, 6578 |
| iCalendar | 5545, 5546, 6047, 6321, 6868, 7265, 7464, 7529, 7953 |
| CalDAV | 4791, 6638, 7808, 7809 |
| vCard / CardDAV | 6350, 6351, 6352, 7095 |
| Obsolete refs in code | 822, 2396, 2426 (vCard 3), 2445 (iCal 2), 2446, 2447, 2518, 2616, 2617, 2965 |
| Drafts | `draft-desruisseaux-ischedule-05` (never RFC), various Apple drafts |

### Compliance tokens advertised by the server

From `twistedcaldav/caldavxml.py:49-69, 1230, 1350, 1369` and
`twistedcaldav/customxml.py:42-90`:

`calendar-access`, `calendar-schedule`, `calendar-auto-schedule`,
`calendar-availability`, `inbox-availability`, `calendar-query-extended`,
`calendar-no-timezone`, `calendar-default-alarms`,
`calendar-managed-attachments`, `timezone-service-set`, `addressbook`,
`calendar-proxy`, `calendarserver-private-events`,
`calendarserver-private-comments`, `calendarserver-principal-property-
search`, `calendarserver-principal-search`, `calendarserver-sharing`,
`calendarserver-sharing-no-scheduling`, `calendarserver-group-sharee`,
`calendarserver-partstat-changes`, `calendarserver-group-attendee`,
`calendarserver-home-sync`, `calendarserver-recurrence-split`.

---

## 2. Core protocol RFCs — implementation status

| RFC | Status | Where |
|---|---|---|
| **RFC 4791 (CalDAV)** | ✅ Implemented | `twistedcaldav/caldavxml.py:272-690`; method handlers in `twistedcaldav/method/` |
| **RFC 6638 (CalDAV Scheduling)** | ✅ Implemented | `txdav/caldav/datastore/scheduling/` (`scheduler.py`, `implicit.py`, `itip.py`, `delivery.py`) |
| **RFC 6352 (CardDAV)** | ✅ Implemented | `twistedcaldav/carddavxml.py:42`; `twistedcaldav/directory/addressbook.py` |
| **RFC 4918 (WebDAV)** | ⚠️ Code still tracks RFC 2518 | `txweb2/dav/method/*.py` cite 2518; needs 4918 review |
| **RFC 7809 (TZ by reference)** | ✅ Implemented (when enabled) | `caldavxml.py:1369-1380` (`TimezoneServiceSet`); `caldavxml.py:289` (`CalendarTimeZoneID`) |
| **RFC 7529 (RSCALE)** | ✅ Implemented (via pycalendar) | `pycalendar/icalendar/recurrence*.py` |
| **RFC 7953 (VAVAILABILITY)** | ✅ Implemented | `customxml.py:247-280`; `txdav/caldav/datastore/scheduling/freebusy.py:221-264` |
| **RFC 8607 (Managed Attachments)** | ⚠️ Draft-level | Code cites `draft-daboo-caldav-attachments` (`caldavxml.py:1346`); disabled by default (`stdconfig.py:542`) |
| **RFC 6764 (SRV/.well-known)** | ⚠️ Only `.well-known` | `EnableWellKnown: True` (`stdconfig.py:539`); **no SRV record handling** |
| **RFC 5545 (iCalendar)** | ✅ Implemented (via pycalendar) | `ccs-pycalendar/src/pycalendar/icalendar/` |
| **RFC 5546 (iTIP)** | ✅ Implemented | `txdav/caldav/datastore/scheduling/` |
| **RFC 6047 (iMIP)** | ✅ Implemented | `txdav/caldav/datastore/scheduling/imip/` |
| **RFC 6321 (xCal)** | ❌ Not implemented | No `application/calendar+xml` support |
| **RFC 7265 (jCal)** | ✅ Implemented | `EnableJSONData: True` (`stdconfig.py:549`); `application/calendar+json` |
| **RFC 6351 (xCard)** | ❌ Not implemented | No `application/vcard+xml` support |
| **RFC 7095 (jCard)** | ✅ Implemented | `application/vcard+json` (`resource.py:663`) |
| **RFC 6868 (param encoding)** | ✅ Implemented | `ccs-pycalendar/src/pycalendar/utils.py:197, 232` |
| **RFC 7986 (iCal new properties)** | ❌ Not implemented | No NAME/COLOR/IMAGE/CONFERENCE/RESOURCE |
| **RFC 9070 (VVENUE)** | ❌ Not implemented | No VVENUE references |
| **HTTP/1.1 (RFC 9110-9114)** | ⚠️ Code cites 2616/2617 | `txweb2/http_headers.py`, `txweb2/channel/http.py`, `txweb2/auth/digest.py` |
| **HTTP/2 (RFC 9113)** | ❌ Not implemented | txweb2 is HTTP/1.1 only |
| **RFC 7616 (Digest SHA-256)** | ❌ Not implemented | `txweb2/auth/digest.py:27` is RFC 2617 only |
| **OAuth 2.0 Bearer (RFC 6750/6749)** | ❌ Not implemented | No OAuth/Bearer/JWT references |

---

## 3. WebDAV extensions — implementation status

| RFC | Status | Where |
|---|---|---|
| **RFC 5689 (Extended MKCALENDAR)** | ✅ | `twistedcaldav/mkcolxml.py:25,44,57` |
| **RFC 5995 (POST to create)** | ✅ | `txdav/xml/rfc5995.py:22-29`; `twistedcaldav/method/post.py` |
| **RFC 3744 (ACL)** | ✅ | `txdav/xml/rfc3744.py`; `txweb2/dav/method/acl.py:46` |
| **RFC 4437 (Redirect References)** | ❌ | Not implemented |
| **RFC 5842 (BIND)** | ✅ (XML), methods may be partial | `txdav/xml/rfc5842.py:22-29` |
| **RFC 7240 (Prefer header)** | ✅ | `txweb2/http_headers.py:815, 983`; `txweb2/dav/method/propfind.py:115-118` |
| **RFC 6578 (Collection sync)** | ✅ | `txdav/xml/rfc6578.py:24-29`; `twistedcaldav/method/report_sync_collection.py` |

---

## 4. Gaps as of August 2026

### Confirmed gaps (post-2020 RFCs and modernization needed)

1. **HTTP/2 / HTTP/3** — txweb2 is HTTP/1.1 only. RFC 9110/9111/9112/9113/
   9114 entirely absent. **Major gap.** (Phase 2)
2. **HTTP/1.1 (RFC 9110-9114) compliance update** — code cites 2616/2617.
   Needs re-review against 9110 semantics. (Phase 1)
3. **RFC 7616 (HTTP Digest SHA-256)** — `txweb2/auth/digest.py:27` is
   RFC 2617 only. (Phase 1 — ADR-0012)
4. **OAuth 2.0 Bearer (RFC 6750/6749)** — not implemented. **Biggest
   functional gap** for modern clients. (Phase 1 — ADR-0012)
5. **RFC 7986 (iCalendar properties)** — IMAGE, COLOR, NAME, CONFERENCE.
   Not implemented. Needs pycalendar update first. (Phase 2)
6. **RFC 9070 (VVENUE)** — not implemented. (Phase 2)
7. **xCal (RFC 6321) / xCard (RFC 6351)** — XML representations not
   supported. (Phase 2)
8. **RFC 6764 SRV records** — discovery only via `.well-known`; SRV record
   side absent. (Phase 1)
9. **Push notifications (modern)** — APNs binary protocol (dead 2021).
   Needs HTTP/2 APNs + WebDAV-Push. (Phase 1 for APNs; Phase 2 for Web Push)
10. **TLS configuration defaults unsafe** — `stdconfig.py:174` defaults to
    `SSLv23_METHOD`. (Phase 1 — ADR-0009)
11. **WebDAV BIND (RFC 4437)** — not implemented. (Phase 2, low priority)
12. **DKIM Ed25519 (RFC 8463)** — only rsa-sha1/sha256. SHA-1 should be
    removed. (Phase 2)
13. **DKIM for iMIP** — DKIM is only for iSchedule (being dropped); iMIP
    email is not signed. (Phase 2)

### Phase-1 RFC scope (per ADR-0012, ADR-0009)

- RFC 9110 audit (update code references from 2616/2617 to 9110).
- RFC 7616 Digest SHA-256 (upgrade `txweb2/auth/digest.py`).
- RFC 6750/6749 OAuth 2.0 Bearer auth (new implementation).
- RFC 6764 SRV records (add SRV record handling).
- RFC 9325 TLS best practices (enforce TLS 1.2+/1.3; drop `SSLv23_METHOD`).
- RFC 6797 HSTS.
- Modern APNs HTTP/2 (replace legacy binary protocol).
- Drop iSchedule (never standardized; rely on RFC 6638 + iMIP).

### Phase-2 RFC deferrals

- HTTP/2 server (RFC 9113); possibly HTTP/3 (RFC 9114).
- Web Push (RFC 8030/8291/8292); WebDAV-Push I-D for DAVx5-class clients.
- RFC 8607 managed attachments (align code from draft to RFC).
- RFC 7986 iCalendar properties; RFC 9070 VVENUE.
- xCal (RFC 6321) / xCard (RFC 6351).
- DKIM Ed25519 (RFC 8463) for iMIP.
- Evaluate JMAP for Calendars (draft-ietf-jmap-calendars).

---

## 5. iSchedule and DKIM

- **iSchedule** (`txdav/caldav/datastore/scheduling/ischedule/`): never
  published as an RFC (`draft-desruisseaux-ischedule-05` is the last public
  draft). Dead spec. **Being dropped** in phase 1. Rely on RFC 6638 + iMIP.
- **DKIM** (`dkim.py`): only for iSchedule. Algorithms: `rsa-sha1` and
  `rsa-sha256` (`dkim.py:84`). SHA-1 should be removed; Ed25519 (RFC 8463)
  not supported. DKIM for outbound iMIP email is a phase-2 enhancement.
- `calendarserver/tools/dkimtool.py` is a CLI for generating RSA DKIM keys
  for iSchedule. Being removed with iSchedule.

---

## 6. Push notifications

### Current mechanisms (all problematic)

1. **APNs (Apple Push Notification Service)** — `calendarserver/push/
   applepush.py`: uses raw binary framing (`struct`) over TLS via
   `OpenSSL`. **Dead** — Apple deprecated the legacy binary APNs interface
   on March 31, 2021. Current APNs uses HTTP/2 with JWT auth. (Phase 1
   modernization — ADR-0012)
2. **AMP push** — `calendarserver/push/amppush.py`: Twisted AMP protocol
   over a local control socket for inter-process push distribution. Not
   network-facing. Keep.
3. **pubsub discovery** — `doc/Extensions/caldav-pubsubdiscovery.txt`:
   Apple extension for XMPP-based pubsub. Compliance token `pubsubnodes`
   is commented out (`calendarserver/tap/caldav.py:1036`). Effectively
   disabled.

### What's missing

- **HTTP/2 APNs** — modern APNs provider API. (Phase 1)
- **WebDAV-Push** (open I-D from bitfireAT/DAVx5) — for DAVx5-class
  clients. (Phase 2)
- **Web Push (RFC 8030/8291/8292)** — for browser-based notifications.
  (Phase 2)
- **XMPP/XEP-0060** — not implemented. (Not planned)

---

## 7. Timezone service

Two implementations:

- **Legacy Apple timezone service** (`twistedcaldav/timezoneservice.py`):
  `TimezoneServiceResource`. Apple-proprietary. Default off
  (`stdconfig.py:556`).
- **Standards-track TZDIST** (`twistedcaldav/timezonestdservice.py`):
  implements **RFC 7808**. `TimezoneStdServiceResource`. Supports JSON
  output. Uses `/.well-known/timezone` URIs.

**RFC 7809 (CalDAV TZ by Reference)** conformance:

- Compliance token `calendar-no-timezone` (`caldavxml.py:68`) advertised
  when `config.EnableTimezonesByReference` is on.
- `TimezoneServiceSet` WebDAV property (`caldavxml.py:1369-1380`) lets
  clients discover the TZDIST service URL (RFC 7809 §5.1).
- `CalendarTimeZoneID` element (`caldavxml.py:289`) implements
  `calendar-timezone-id` (RFC 7809 §5.2). Doc comment still says
  "draft-ietf-tzdist-caldav-timezone-ref-01" — should be updated to cite
  RFC 7809.

**Gap**: both `EnableTimezoneService` (legacy) and
`EnableTimezonesByReference` (RFC 7809) default off. Out-of-the-box install
advertises nothing.

---

## 8. Apple extensions — macOS 26/27 client support

Based on DAVx5/dav4jvm source code analysis, SOGo/Baikal/Nextcloud
compatibility matrices, and iCloud server behavior stability:

### Tier 1 — keep in standards-compliant core (also parsed by DAVx5)

| Extension | DAVx5? | Other clients | Superseded by RFC? |
|---|---|---|---|
| `calendarserver:getctag` | ✅ parses | Universal | RFC 6578 (but getctag is cheaper probe) |
| `calendar-proxy` (read/write + reverse) | ✅ parses | SOGo | No RFC (draft expired) |
| `calendarserver:source` / `subscribed` | ✅ parses | SOGo | No RFC |
| `apple.com/ns/ical:calendar-color` | ✅ parses | Most display it | No RFC (de-facto) |

### Tier 2 — keep, segregate behind capability flag (only Apple Calendar)

| Extension | macOS 26/27? | Other clients | Notes |
|---|---|---|---|
| `calendarserver:sharing` (+ variants) | Likely Yes | None | Apple's shared-calendar wire protocol |
| `calendarserver:notifications` | Likely Yes | None | In-app notification feed |
| `calendarserver:private-events` | Likely Yes | None | Per-event visibility in shared calendars |
| `calendarserver:private-comments` / `X-CALENDARSERVER-PRIVATE-COMMENT` | Likely Yes | None (preserved round-trip) | Private comments |
| `X-CALENDARSERVER-PRIVATE-EVENT` | Likely Yes | None (preserved round-trip) | Per-event private flag |
| `calendarserver:principal-property-search` | Likely Yes | None | Attendee autocomplete |
| `calendarserver:principal-search` | Likely Yes | None | Broader principal search |
| `calendarserver:recurrence-split` | Likely Yes | None | Outlook/Exchange interop |
| `calendarserver:home-sync` | Likely Yes | None | Bulk calendar-home sync |
| `calendarserver:bulk-change` | Likely Yes | None | Batch modifications |
| `calendarserver:partstat-changes` | Likely Yes | None | Bulk attendee-status update |
| `calendarserver:group-attendee` | Likely Yes | None | LDAP group as ATTENDEE |
| `calendarserver:calendar-availability` | Likely Yes | None | Per-user VAVAILABILITY |
| `calendarserver:calendar-transp` | Likely Yes | None | Per-calendar transparency |
| `push-transports` / `pushkey` | Yes | None | Apple Push (APNs); needs HTTP/2 modernization |

### Tier 3 — preserve round-trip only, no semantics

| Extension | Notes |
|---|---|
| `X-APPLE-STRUCTURED-LOCATION` | Emitted by current Calendar.app. Optionally map to RFC 9254 `VLOCATION`. |
| All other `X-APPLE-*` / `X-CALENDARSERVER-*` | Do not implement semantics; never strip. |

See [ADR-0014](../adr/0014-segregate-apple-extensions.md) for the full
segregation plan.

---

## 9. Active IETF Internet-Drafts to track (Aug 2026)

| Draft | Title | Status |
|---|---|---|
| `draft-ietf-calext-ical-tasks-17` | Task Extensions to iCalendar | RFC Ed Queue |
| `draft-ietf-calext-jscalendarbis-18` | JSCalendar 2.0 | AD Evaluation |
| `draft-ietf-jmap-calendars-28` | JMAP for Calendars | RFC Ed Queue (blocked) |
| `draft-ietf-calext-icalendar-jscalendar-extensions-06` | iCal Format Extensions for JSCalendar | WG Last Call |
| `draft-ietf-calext-jscalendar-icalendar-25` | JSCalendar ↔ iCalendar conversion | WG Last Call |
| `draft-ietf-mailmaint-pacc-03` | Auto-config of email/calendar/contact servers | WG Document |
| `draft-kashyap-calext-ical-property-deps-00` | Machine-Readable Property Dependencies for RFC 5545 | Individual |

Monitor: CALEXT WG (https://datatracker.ietf.org/wg/calext/about/), JMAP WG
(https://datatracker.ietf.org/wg/jmap/about/), MAILMAINT WG
(https://datatracker.ietf.org/wg/mailmaint/about/).

---

## 10. RFC reference library (`rfc/` folder)

A `rfc/` folder in the repo root will contain downloaded RFCs organized by
category. A `rfc/fetch.sh` script downloads ~80 RFCs from `rfc-editor.org`
plus errata snapshots for MUST-tier RFCs and active IETF drafts.

### Suggested folder layout

```
rfc/
  00-INDEX.txt                  # generated manifest
  http/                         # 9110-9114, 7541, 9204, 9000, 7230-7237, 2616, 2818
  tls/                          # 8446, 9325, 9155, 5246, 6066
  auth/                         # 7617, 7616, 2617, 6750, 6749, 8705, 4559, 8053
  webdav/                       # 4918, 2518, 3253, 3744, 4331, 5397, 5689, 5842, 4437, 5995, 6578, 7240, 5789, 5785, 8615
  caldav/                       # 4791, 6638, 7809, 8607, 6764, 7529, 7953
  carddav/                      # 6352, 6350, 6351, 7095, 6868
  icalendar/                    # 5545, 5546, 6047, 6321, 7265, 7986, 7464, 9073, 9074, 9253, 9070, 8984
  tzdist/                       # 7808, 7809, 8536, 9636
  push/                         # 8030, 8291, 8292, 8187, 8188
  dkim-mail/                    # 6376, 8463, 5321, 5322, 2045-2049, 2231, 6857
  security/                     # 9116, 6797
  dns-discovery/                # 2782, 6764, 6763, 8499
  url-iri-abnf/                 # 3986, 3987, 5234, 7405, 7303, 8259, 4648
  i18n/                         # 5646, 4647
  errata/                       # snapshots for MUST-tier RFCs
  drafts/active/                # current IETF drafts (§9)
  drafts/historical/            # Apple/CalConnect drafts
```

**Follow-up**: write `rfc/fetch.sh` + `rfc/manifest.txt` (noted in
[MIGRATION_PLAN.md](MIGRATION_PLAN.md) as a phase-1 follow-up).

---

## 11. Compliance testing

See [ADR-0015](../adr/0015-compliance-testing-architecture.md) for the full
testing architecture (Tier-0 RFC-anchored, Tier-1 ccs-caldavtester, Tier-2
interop, Tier-3 property/fuzz).

### Test/validation infrastructure survey

| Tool | Coverage | Status |
|---|---|---|
| `ccs-caldavtester` (Apple) | CalDAV 4791 + extensions + Apple extensions | In repo; py2-only driver |
| CalConnect (TC TECH) | Round-robin interop | Public test materials periodically |
| IETF conformance tools | **None** | IETF does not run conformance |
| libical validation | 5545 parse + recurrence | Active |
| `icalendar` (Python) validation | 5545 syntactic + semantic | Active |
| vdirsyncer `tests/system/` | Black-box cross-server conformance | Active |
| sabre/dav `tests/` | White-box + protocol-level | Active |
| Radicale `integ_tests/` | End-to-end | Active |

---

## 12. Scorecard summary

| Area | Compliant as of 2026? | Phase |
|---|---|---|
| CalDAV RFC 4791 | ✅ Yes | — |
| CardDAV RFC 6352 | ✅ Yes | — |
| CalDAV Scheduling RFC 6638 | ✅ Yes | — |
| VAVAILABILITY RFC 7953 | ✅ Yes | — |
| RSCALE RFC 7529 | ✅ Yes | — |
| Managed Attachments RFC 8607 | ⚠️ Draft-level | Phase 2 |
| TZ by Reference RFC 7809 | ✅ Yes (when enabled) | — |
| TZDIST RFC 7808 | ✅ Yes | — |
| WebDAV RFC 4918 | ⚠️ Code tracks 2518 | Phase 1 |
| ACL RFC 3744 | ✅ Yes | — |
| Sync RFC 6578 | ✅ Yes | — |
| Prefer RFC 7240 | ✅ Yes | — |
| Extended MKCOL RFC 5689 | ✅ Yes | — |
| POST-to-create RFC 5995 | ✅ Yes | — |
| BIND RFC 5842 | ✅ (partial) | — |
| BIND RFC 4437 | ❌ No | Phase 2 (low) |
| SRV Discovery RFC 6764 | ⚠️ Only `.well-known` | Phase 1 |
| iCalendar RFC 5545 | ✅ Yes | — |
| iTIP/iMIP RFC 5546/6047 | ✅ Yes | — |
| jCal RFC 7265 / jCard RFC 7095 | ✅ Yes | — |
| xCal RFC 6321 / xCard RFC 6351 | ❌ No | Phase 2 |
| RFC 6868 param encoding | ✅ Yes | — |
| RFC 7986 iCal properties | ❌ No | Phase 2 |
| RFC 9070 VVENUE | ❌ No | Phase 2 |
| HTTP/1.1 RFC 9110-9114 | ⚠️ Code cites 2616/2617 | Phase 1 |
| HTTP/2 / HTTP/3 | ❌ No | Phase 2 |
| HTTP Digest SHA-256 RFC 7616 | ❌ No (RFC 2617) | Phase 1 |
| OAuth 2.0 Bearer RFC 6750/6749 | ❌ No | Phase 1 |
| TLS 1.2/1.3 RFC 9325 | ❌ `SSLv23_METHOD` default | Phase 1 |
| Push (APNs legacy binary) | ❌ Dead (2021) | Phase 1 |
| Web Push RFC 8030/8291/8292 | ❌ No | Phase 2 |
| DKIM Ed25519 RFC 8463 | ❌ No (rsa-sha1/sha256) | Phase 2 |
| iSchedule | ⚠️ Draft never standardized | Drop (Phase 1) |
| Python version | ❌ Py2 only | Phase 1 |

---

## References

- `doc/RFC/README.md:1-76` (canonical RFC list)
- `twistedcaldav/caldavxml.py:49-69, 1230, 1350, 1369` (compliance tokens)
- `twistedcaldav/customxml.py:42-90` (Apple extension XML elements)
- `doc/Extensions/` (12 Apple extension drafts)
- [ADR-0014](../adr/0014-segregate-apple-extensions.md) — Apple extension segregation
- [ADR-0015](../adr/0015-compliance-testing-architecture.md) — testing architecture
- [ADR-0012](../adr/0012-oauth-oidc-primary-auth.md) — auth modernization
- [ADR-0009](../adr/0009-standardize-tls-pyopenssl-cryptography.md) — TLS
