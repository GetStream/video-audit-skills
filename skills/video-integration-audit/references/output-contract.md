# Output contract: the report

The skill writes exactly one file: `video-integration-audit.md` at the root of the audited repository, or the path the user named. Everything the skill found goes into that file; the chat reply only says where it is and lists the top findings.

The report has two readers. A person reads the prose and tables to decide what to do. The porting skill in Stream's agent-skills pack reads the machine-readable block at the end to change code. Both must find the same facts, so the tables and the block are generated from the same rows.

## Rules for the whole report

- **ASCII only.** Plain `-`, `->`, `"`, `'`; no typographic dashes, arrows or quotes.
- **Evidence is `path:line`**, relative to the audited root. Every inventory row, finding and plan step that rests on code cites at least one location.
- **No secrets.** Never copy a key, secret, certificate, private key or token value. Cite the location and write `<redacted>`.
- **No customer data.** Do not copy user names, emails, phone numbers or other personal data found in fixtures, seeds or logs.
- **Pinned versions.** Every Stream equivalent names the SDK version the mapping was written against (from the reference file's header). Write `verify against your SDK version` where the reference says so.
- **Code facts and vendor facts.** Describe what the app's code does from the code. Statements about what a vendor's platform does (token revocation or expiry, what a restart breaks, recording outputs, limits) come from a reference file in this skill; if no reference says it, describe only the code's behaviour or mark the point `NEEDS-REVIEW`.
- **No guessing.** If a construct has no row in the vendor reference, say `no mapping in this skill` and add it to the flag-for-human list. Never invent an API name: every Stream or vendor symbol anywhere in the report, Fix columns included, comes from a reference file in this skill or carries `verify against your SDK version`. Before writing `no mapping`, search the references (including the Stream facts table in `hub.md`) for the construct.
- **No commercial claims.** No prices, no vendor cost comparisons, no claims about either vendor beyond their public documentation. See the honesty rules in `hub.md`.

## Sections by mode

| Section | Migration (Vonage / Agora) | Stream | Hybrid | Neither / unsupported |
|---|---|---|---|---|
| 1. Header | yes | yes | yes | yes |
| 2. What was detected | yes | yes | yes | yes |
| 3. Usage dossier | yes | short | yes | no |
| 4. Integration inventory | yes | no | yes (remaining vendor code) | no |
| 5. Behaviour changes to plan for | yes | no | yes | no |
| 6. Stream best-practice findings | no | yes | yes (migrated part) | no |
| 7. General findings | yes | yes | yes | yes |
| 8. Call-type configuration checklist | yes | if gaps found | yes | no |
| 9. Migration plan | yes | no | yes (finish the transition) | no |
| 10. Flag for a human | yes | yes | yes | yes |
| 11. Questions the code cannot answer | yes | yes | yes | if any |
| 12. Prioritised fix list | yes | yes | yes | yes |
| 13. What this skill cannot do | yes | yes | yes | yes |
| 14. How to continue | yes | yes | yes | yes |
| 15. Machine-readable block | yes | yes | yes | yes |

A section marked `no` keeps its heading with the single line `Not applicable in <mode> mode.`, so section numbers are the same in every report. In Stream mode the report contains no migration plan and no comparison with other vendors.

## 1. Header

```
<!-- video-integration-audit report v1 -->
# Video integration audit: <repository or app name>

- Date: <YYYY-MM-DD>
- Skill: video-integration-audit <version from SKILL.md>
- Mode: <migration | stream | hybrid | neither>
- Scope: <root path audited>; <N> files read; excluded: <dirs>
- Read-only: this audit read files and wrote this report. It ran no code and made no network calls.
- Publisher: this skill is published by Stream (getstream.io), a video platform vendor. Its checks are the same for every platform; statements about vendors come from their public documentation.
```

The first line is the marker that identifies a report written by this skill.

## 2. What was detected

| Platform | Vendor / product | Vintage | Packages and versions | Evidence |
|---|---|---|---|---|
| web client | Agora Video SDK for Web | 4.x (`agora-rtc-sdk-ng`) | `agora-rtc-sdk-ng` 4.24.9 | `package.json:12` |

One row per platform per vendor (web client, iOS, Android, React Native, Flutter, server). Include server SDKs and token libraries. Then one sentence stating the mode and why.

## 3. Usage dossier

A standalone description a reader can check against their own understanding of the app ("did we understand your app?"). First a short overview of the video features the app uses, then one subsection per feature describing **how it is glued into the rest of the app**: where it starts, which screens and services touch it, what state it keeps, what server endpoints back it. Classify each call surface:

- `calling` - one user rings another (1:1 or small group, invite and answer)
- `meeting` - scheduled or link-join rooms
- `livestream` - hosts and an audience
- `recording` - recordings or other post-call artifacts

The classification is per surface, not per app: one codebase can have all four.

## 4. Integration inventory

One row per **integration site**: a place in the code that uses a vendor construct. Group rows by decision area (the nine areas in `hub.md`).

| ID | Location | Vendor construct | Decision area | Stream equivalent (SDK version) | Class | Notes |
|---|---|---|---|---|---|---|
| INV-01 | `web/src/room.js:40` | `client.on('user-published')` + `client.subscribe(user, mediaType)` | track-device-lifecycle | none needed: automatic subscription; render `call.state.participants$` (`@stream-io/video-client` 1.62.0) | delete | delete the subscribe loop |

`Class` is one of:

- `rename` - same behaviour, different name
- `shape-shift` - same intent, different call shape or arguments
- `behavioral-diff` - looks equivalent, behaves differently (these also go into section 5)
- `delete` - the Stream SDK does this itself; the code goes away
- `config` - moves from code to call-type or dashboard configuration
- `upgrade` - Stream offers a first-class feature for hand-rolled code (ringing, server-side moderation, native HLS)
- `gap` - no Stream equivalent today (also goes into section 10)
- `no-mapping` - this skill has no mapping for it (also goes into section 10)

The vendor references' Class columns use three more forms, which never appear in the report as such. `flag`: the row also goes into section 10; give it `gap` when Stream has nothing for it, `no-mapping` when the reference gives only a partial mapping, otherwise the class of the Stream equivalent it names. Two classes joined (`shape-shift + config`, `delete (orchestration), config (storage, layout)`): write one inventory row per class. `X or Y`: the row's notes say which applies.

## 5. Behaviour changes to plan for

The constructs that a find-and-replace port gets wrong without any error: one row per kill-list trap from the vendor reference that this app actually hits, with the inventory rows it affects and what the port must do instead. List only traps with evidence in this codebase.

## 6. Stream best-practice findings

Stream and hybrid modes only. One row per check in `stream-web.md` (or the platform's Stream reference), in its order:

| Check | Verdict | Severity | Evidence | Fix |
|---|---|---|---|---|

## 7. General findings

One row per check in `general-checks.md`, G1 to G12, in order, same columns as section 6. Never skip a row. When the app has several vendors or several backends (hybrid mode, or Vonage on web and Agora on mobile), judge each check per vendor: any `FAIL` makes the row `FAIL`; otherwise any `NEEDS-REVIEW` makes it `NEEDS-REVIEW`. The Evidence column names the vendor or surface for each part.

## 8. Call-type configuration checklist

Settings that move from code (session flags, token claims, per-room REST options) to Stream call-type configuration, one row each: the setting, its current source location, the Stream call-type setting it becomes, the value the code implies. Include target resolution (Stream bills per participant minute by the aggregated resolution of the video each participant receives; check the call type's target resolution against the layouts the app uses), recording mode and layout, backstage and join-ahead for livestreams, ring timeouts for calling surfaces, and the role and permission grid.

## 9. Migration plan

Phases, each with its goal, the inventory rows it covers, what changes, and how to verify it. The default shape:

1. **Server first** - token minting, user identity, call types and permissions, recording and webhook handlers; both vendors' backends side by side.
2. **One client surface at a time** - in the order the dossier suggests (usually the simplest surface first), behind a server-decided vendor switch.
3. **Upgrades** - the `upgrade` rows (ringing, server-enforced moderation, native livestream), offered, not assumed.
4. **Transition and cleanup** - the fleet-transition plan (see `hub.md`), then removal of the old vendor's SDKs, token routes and REST calls.

Then the **fleet-transition note**: which surfaces exist on which platforms, the rendezvous strategy per surface, the recommended transition option with its reason, and the early-prep checklist diffed against what the app already has. For web-only apps state that users switch as they load the new build, and say what still needs coordination (open tabs on the old build, recordings in flight, links already sent).

In hybrid mode this section is "Finishing the transition": the remaining vendor code from section 4, plus the coexistence risks found (two token systems, two sets of webhooks, mobile installs that still point at the old vendor).

## 10. Flag for a human

Everything the skill could not decide: `gap` and `no-mapping` rows, product decisions (how to map roles, what to do with a feature Stream lacks), and blocker-class requirements (end-to-end encryption, fixed IPs, SIP dial-out, data residency). One row per item: what, where (`path:line`), why it needs a person, and who is best placed to answer (the app team, or Stream sales where the honesty rules route it there).

## 11. Questions the code cannot answer

Facts a migration plan needs that are not in the code: scale (peak concurrent participants per call, calls per day, largest livestream audience), the old vendor's contract end date, firewall or allow-list requirements in customers' networks, dashboard-side configuration of the current vendor, and anything the dossier marked as uncertain. Batch them; do not stop the audit to ask.

## 12. Prioritised fix list

Every `FAIL` and `NEEDS-REVIEW` from sections 6 and 7, Blockers first, then High, Medium, Low. One line each: check id, one-line fix, evidence. These fixes help whether or not the app migrates.

## 13. What this skill cannot do

State plainly, adapted to the app:

- It read the code and nothing else: no dashboards, no vendor account settings, no production traffic, no runtime behaviour.
- It changed nothing and proposes diffs only when asked.
- It does not estimate cost or compare prices.
- Mapping rows marked `verify against your SDK version` were not confirmed for the exact versions this app pins.
- Platforms without a reference file in this skill version (name them) got a platform-level analysis only.

## 14. How to continue

- **Migration and hybrid mode:** to port the code, install the porting skill from Stream's agent-skills pack (`https://github.com/GetStream/agent-skills`) and give it this report. The Stream docs page "Migrate from <vendor> to Stream" covers the same ground for people. For commercial or roadmap questions, contact Stream at `https://getstream.io/contact/`.
- **Stream mode:** to apply the fixes, install the pack's platform skill for this stack (for example `stream-react`) and ask it to fix the findings in this report.
- **Neither mode:** nothing to continue with in this skill version; name what would be needed.

## 15. Machine-readable block

The last section of the report is one fenced `json` block holding the same rows as the tables. The porting skill reads only this block, so it must be complete and valid JSON.

```json
{
  "schema": "video-integration-audit/v1",
  "skill_version": "0.1.0",
  "date": "YYYY-MM-DD",
  "mode": "migration",
  "detected": [
    { "platform": "web", "vendor": "agora", "product": "Video SDK for Web", "vintage": "4.x", "packages": { "agora-rtc-sdk-ng": "4.24.9" }, "evidence": ["package.json:12"] }
  ],
  "surfaces": [
    { "id": "S1", "kind": "meeting", "platforms": ["web"], "entry": "web/src/main.js:21" }
  ],
  "inventory": [
    {
      "id": "INV-01",
      "location": "web/src/room.js:40",
      "vendor": "agora",
      "construct": "client.on('user-published') + client.subscribe(user, mediaType)",
      "decision_area": "track-device-lifecycle",
      "stream_equivalent": "automatic subscription; render call.state.participants$",
      "stream_sdk": "@stream-io/video-client@1.62.0",
      "class": "delete",
      "verify": false,
      "surface": "S1",
      "notes": "delete the subscribe loop"
    }
  ],
  "behaviour_changes": [
    { "id": "BC-01", "trap": "subscribe-loop", "rows": ["INV-01"], "action": "delete the loop; rebuild any selective subscription with setIncomingVideoEnabled" }
  ],
  "findings": [
    { "id": "G4", "set": "general", "verdict": "FAIL", "severity": "Blocker", "evidence": ["web/src/main.js:35"], "fix": "await join inside try/catch and show the error" }
  ],
  "call_type_config": [
    { "setting": "target_resolution", "source": "web/src/room.js:88", "value": "1920x1080", "stream_setting": "settings.video.target_resolution" }
  ],
  "flags": [
    { "id": "FL-01", "what": "end-to-end encryption", "location": "web/src/room.js:30", "owner": "stream-sales" }
  ],
  "questions": ["Peak participants per call?"]
}
```

`decision_area` takes one of: `tokens-identity`, `room-call-type`, `permissions`, `call-lifecycle`, `events-reconnect`, `track-device-lifecycle`, `ringing`, `recording-livestream`, `fleet-transition`, or `integrations` (SIP, AI agents and connectors, chat products; see `hub.md`). `findings[].set` is `general` (G1-G12) or `stream` (S-rows). `verify` is `true` when the row carries `verify against your SDK version`, otherwise `false`. `owner` is `app-team` or `stream-sales`. Omit keys that do not apply to the mode, but keep the array keys present (empty arrays are fine).

## Self-check before writing

- The header has the marker line and the publisher line.
- All fifteen section headings are present, numbered 1 to 15, in order; the ones the mode does not use carry the single line `Not applicable in <mode> mode.` No extra sections.
- Every general check G1-G12 has a row.
- Every inventory row has a location, a decision area and a class; every `gap` and `no-mapping` row is also in section 10.
- SIP, DTMF, AI agents and audio connectors, and chat or messaging products are tagged `integrations`.
- Every API name in the report appears in a reference file or is marked `verify against your SDK version`.
- No secret values, no personal data, no prices, no claims about either vendor beyond the references.
- No private or one-recipient message is mapped to custom events, and no option suggests making it public (`targeted-signals`).
- The JSON block parses (balanced braces, double-quoted keys, no trailing commas) and its row counts match the tables.
