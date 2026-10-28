# Output contract: the report

The skill writes exactly one file: `video-integration-audit.md` at the root of the audited repository, or the path the user named. Everything the skill found goes into that file; the chat reply only says where it is and lists the top findings.

The report has two readers. A person reads the prose and tables to decide what to do. A coding agent porting the app reads the machine-readable block at the end to change code. Both must find the same facts, so the tables and the block are generated from the same rows.

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
| 4. What changes for this app on Stream | yes | no | yes (remaining vendor code) | no |
| 5. Cost analysis | yes | no | yes (remaining vendor code) | no |
| 6. Integration inventory | yes | no | yes (remaining vendor code) | no |
| 7. Behaviour changes to plan for | yes | no | yes | no |
| 8. Stream best-practice findings | no | yes | yes (migrated part) | no |
| 9. General findings | yes | yes | yes | yes |
| 10. Call-type configuration checklist | yes | if gaps found | yes | no |
| 11. Migration plan | yes | no | yes (finish the transition) | no |
| 12. Flag for a human | yes | yes | yes | yes |
| 13. Questions the code cannot answer | yes | yes | yes | if any |
| 14. Prioritised fix list | yes | yes | yes | yes |
| 15. What this skill cannot do | yes | yes | yes | yes |
| 16. How to continue | yes | yes | yes | yes |
| 17. Machine-readable block | yes | yes | yes | yes |

A section marked `no` keeps its heading with the single line `Not applicable in <mode> mode.`, so section numbers are the same in every report. In Stream mode the report contains no migration plan and no comparison with other vendors.

## 1. Header

```
<!-- video-integration-audit report v1 -->
# Video integration audit: <repository or app name>

- Date: <YYYY-MM-DD>
- Skill: video-integration-audit <version from SKILL.md>
- Mode: <migration | stream | hybrid | neither>
- Scope: <root path audited>; <N> files read; excluded: <dirs> (never stylesheets: they are searched for the vendor's DOM class names)
- Coverage: code-level for <the web platforms found, or "no platform">. <Only when the app has iOS, Android, React Native or Flutter code: "Platform-level only for <those platforms>."> This version has code-level references for web apps only.
- Read-only: this audit read files and wrote this report. It ran no code and made no network calls.
- Publisher: this skill is published by Stream (getstream.io), a video platform vendor. Its checks are the same for every platform; statements about vendors come from their public documentation.
```

The first line is the marker that identifies a report written by this skill. Section 1 is this block: it has no `## 1.` heading of its own.

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

End the dossier with an **Architecture** subsection, the part a porting step works from. Cite `path:line` for every item:

- **Client:** framework and router (React, Next.js, Vue, plain JavaScript), build tool, and where the vendor client or session object is created and what owns its lifetime (a module singleton, a React context, a store, component state).
- **Server routes:** every route, job or handler that touches the vendor: the token endpoint (method, path, request fields, response fields, and the authentication in front of it), webhook and callback endpoints, and each vendor REST call.
- **Identity:** where the user id and the display name come from and what type the id is (an Agora numeric uid, a name inside a Vonage token's `data`, a Stream user id); whether the client chooses it.
- **Stored vendor ids:** database columns, caches or in-memory maps that hold session ids, channel names, archive or recording ids, or recording URLs.
- **Recordings and other artifacts:** where recordings, transcripts and broadcast outputs end up, as far as the code shows (storage configuration, upload or download code). Storage set in a vendor dashboard is a question for section 13.
- **Video UI:** the app's own video components (tiles, grids, controls, device pickers, lobby, ring screens), each with the Stream prebuilt component that could replace it, named in a reference file, or `keep custom`.
- **Environment:** the names of the environment variables the video code reads. Never their values.
- **Tests:** test files and mocks that reference the vendor SDK.

In Stream mode, write only Client, Server routes and Video UI.

## 4. What changes for this app on Stream

Migration and hybrid modes. A short summary, built only from rows already in the report, of what moving changes for this app. It has two halves of equal standing, and every line cites the row ids behind it (`INV-`, `BC-`, `FL-`), which carry the `path:line`. Never state a benefit that has no row behind it, and never offer a feature the app does not need (`hub.md` honesty rule 6).

**What the app gains.** Under these headings, leaving out any that would be empty:

- **Code that goes away:** the number of `delete` rows (only `delete`: `rename` and `shape-shift` rows are code that changes, not code that goes away) and the files they touch, grouped by what the code did (for example subscription loops, layer selection, token renewal plumbing, reconnect bookkeeping, archive status polling). Then, as an option, the Video UI items from section 3 that a Stream prebuilt component could replace.
- **Built-in features for hand-built ones:** each `upgrade` row: what the app built itself, the Stream feature that replaces it, and what the app gains, in one clause taken from the reference (for example server-owned ring timeouts and missed-call events).
- **Rework needed on any platform:** code that depends on a product its vendor has deprecated or discontinued, quoting the vendor's notice as the reference gives it (Vonage In-App Messaging, Video Express, Agora Web SDK 3.x, `agora-access-token`). Say plainly that this work is needed whether or not the app moves.

**What the app must change.** Every `gap` and `no-mapping` row, one line each with what the app has to do differently, and every blocker-class flag (end-to-end encryption, fixed IPs, SIP dial-out). For a `gap` row say plainly that Stream has no equivalent today and what that means for the feature (drop it, build it, or keep a vendor for it). Same weight and tone as the gains; do not soften.

In hybrid mode, cover only the remaining vendor code.

## 5. Cost analysis

Migration and hybrid modes. Three parts, in this order. No prices, no totals and no "cheaper" anywhere in the section: the reader does the arithmetic with their own numbers.

**What drives cost.** One row per metered feature the code uses: participant minutes and the published resolution, recording, broadcast and HLS, RTMP, transcription and captions, noise cancellation, SIP and connectors.

| Feature | What the code does | Rows | What to measure |
|---|---|---|---|
| Recording | <what is recorded, when it starts, composite or per participant, resolution> | <row ids> | <the volume to measure, for example recorded call minutes, or recorded participant minutes for per-participant files> |

**Usage volume.** Say whether the code shows how much the app is used, because Stream's price for an app depends on its volume (see Pricing below). Look for what the code states, never for what it might imply: deployment and scaling configuration (replica counts, autoscaling limits, regions), load-test or capacity targets, configured limits (participants per call, rooms per tenant), the number of tenants or customers configured, scheduled jobs that create calls, and statements in READMEs or docs. Write one of:

- `none`: the code gives no evidence of volume. This is the usual case; say so in one sentence and put the volume questions in section 13.
- `indications`: the code shows signs of scale but not enough to estimate (for example autoscaling to many instances, several regions, many tenants). List each sign with `path:line`.
- `estimate`: the code states numbers that give an order of magnitude (for example a load-test target of concurrent calls, or a README stating sessions per day). Show the arithmetic from those numbers only, label the result as an estimate from the code, and never add numbers the code does not state.

**Pricing.** One paragraph: Stream's pricing page (`https://getstream.io/video/pricing/`) lists these features separately from participant minutes, in the units the Stream facts table in `hub.md` gives; <vendor> prices them on its own pricing page. Where Stream's pricing page does not list a feature the app uses (the facts table names them), say so. Then always recommend contacting Stream sales (`https://getstream.io/contact/`) for a quote at the app's real volumes, because the pricing page lists pay-as-you-go prices and Stream's Enterprise plan adds volume discounts (the facts table has the page's wording). When the usage volume is `indications` or `estimate`, say so in the recommendation and name the evidence: a volume quote is the right basis for comparing the two vendors, not the list prices. Never state the size of a discount.

In hybrid mode, cover only the remaining vendor code.

## 6. Integration inventory

One row per **integration site**: a place in the code that uses a vendor construct. Group rows by decision area (the nine areas in `hub.md`).

| ID | Location | Vendor construct | Decision area | Stream equivalent (SDK version) | Class | Notes |
|---|---|---|---|---|---|---|
| INV-01 | `web/src/room.js:40` | `client.on('user-published')` + `client.subscribe(user, mediaType)` | track-device-lifecycle | none needed: automatic subscription; render `call.state.participants$` (`@stream-io/video-client` 1.63.0) | delete | delete the subscribe loop |

`Class` is one of:

- `rename` - same behaviour, different name
- `shape-shift` - same intent, different call shape or arguments
- `behavioral-diff` - looks equivalent, behaves differently (these also go into section 7)
- `delete` - the Stream SDK does this itself; the code goes away
- `config` - moves from code to call-type or dashboard configuration
- `upgrade` - Stream offers a first-class feature for hand-rolled code (ringing, server-side moderation, native HLS)
- `gap` - no Stream equivalent today (also goes into section 12)
- `no-mapping` - this skill has no mapping for it (also goes into section 12)

The vendor references' Class columns use three more forms, which never appear in the report as such. `flag`: the row also goes into section 12; give it `gap` when Stream has nothing for it, `no-mapping` when the reference gives only a partial mapping, otherwise the class of the Stream equivalent it names. Two classes joined (`shape-shift + config`, `delete (orchestration), config (storage, layout)`): write one inventory row per class. `X or Y`: the row's notes say which applies.

## 7. Behaviour changes to plan for

The constructs that a find-and-replace port gets wrong without any error: one row per kill-list trap from the vendor reference that this app actually hits, with the inventory rows it affects and what the port must do instead. List only traps with evidence in this codebase.

## 8. Stream best-practice findings

Stream and hybrid modes only. One row per check in `stream-web.md` (or the platform's Stream reference), in its order:

| Check | Verdict | Severity | Evidence | Fix |
|---|---|---|---|---|

## 9. General findings

One row per check in `general-checks.md`, G1 to G13, in order, same columns as section 8. Never skip a row. When the app has several vendors or several backends (hybrid mode, or Vonage on web and Agora on mobile), judge each check per vendor: any `FAIL` makes the row `FAIL`; otherwise any `NEEDS-REVIEW` makes it `NEEDS-REVIEW`. The Evidence column names the vendor or surface for each part.

## 10. Call-type configuration checklist

Settings that move from code (session flags, token claims, per-room REST options) to Stream call-type configuration, one row each: the setting, its current source location, the Stream call-type setting it becomes, the value the code implies. Include target resolution (Stream bills per participant minute by the aggregated resolution of the video each participant receives; check the call type's target resolution against the layouts the app uses), recording mode and layout, backstage and join-ahead for livestreams, ring timeouts for calling surfaces, and the role and permission grid.

Then a **role mapping** table: one row per vendor role, token role or client-side role flag the code uses, with its source location, the Stream role it becomes, and the call-type capabilities that role needs (capability names from `hub.md`, for example `send-audio`, `send-video`, `mute-users`, `start-record-call`). Where the code decides roles on the client, say so: on Stream the server decides.

## 11. Migration plan

Start with the **size of the work**: one row per decision area, with the number of inventory rows, how many of them are `delete`, `upgrade`, `gap` and `no-mapping`, and the number of files they touch. Counts only; never estimate time or effort in days.

Then phases, each with its goal, the inventory rows it covers, what changes, and how to verify it. The default shape:

1. **Server first** - token minting, user identity, call types and permissions, recording and webhook handlers; both vendors' backends side by side.
2. **One client surface at a time** - in the order the dossier suggests (usually the simplest surface first), behind a server-decided vendor switch.
3. **Upgrades** - the `upgrade` rows (ringing, server-enforced moderation, native livestream), offered, not assumed.
4. **Transition and cleanup** - the fleet-transition plan (see `hub.md`), then removal of the old vendor's SDKs, token routes and REST calls.

Then the **fleet-transition note**: which surfaces exist on which platforms, the rendezvous strategy per surface, the recommended transition option with its reason, and the early-prep checklist diffed against what the app already has. For web-only apps state that users switch as they load the new build, and say what still needs coordination (open tabs on the old build, recordings in flight, links already sent).

In hybrid mode this section is "Finishing the transition": the remaining vendor code from section 6, plus the coexistence risks found (two token systems, two sets of webhooks, mobile installs that still point at the old vendor).

## 12. Flag for a human

Everything the skill could not decide: `gap` and `no-mapping` rows, product decisions (how to map roles, what to do with a feature Stream lacks), and blocker-class requirements (end-to-end encryption, fixed IPs, SIP dial-out, data residency). One row per item, with its `FL-` id (the same ids as the JSON block): what, where (`path:line`), why it needs a person, and who is best placed to answer (the app team, or Stream sales where the honesty rules route it there).

## 13. Questions the code cannot answer

Facts a migration plan needs that are not in the code: scale (peak concurrent participants per call, calls per day, largest livestream audience), the old vendor's contract end date, firewall or allow-list requirements in customers' networks, dashboard-side configuration of the current vendor, and anything the dossier marked as uncertain. Batch them; do not stop the audit to ask.

## 14. Prioritised fix list

Every `FAIL` and `NEEDS-REVIEW` from sections 8 and 9, Blockers first, then High, Medium, Low. One line each: check id, one-line fix, evidence. These fixes help whether or not the app migrates.

## 15. What this skill cannot do

State plainly, adapted to the app:

- It read the code and nothing else: no dashboards, no vendor account settings, no production traffic, no runtime behaviour.
- It changed nothing and proposes diffs only when asked.
- It names what drives cost but quotes no prices and compares none.
- Mapping rows marked `verify against your SDK version` were not confirmed for the exact versions this app pins.
- Platforms without a reference file in this skill version (name them) got a platform-level analysis only.

## 16. How to continue

- **Migration and hybrid mode:** to port the code, give this report to your coding agent together with Stream's Video docs (`https://getstream.io/video/docs/`) and, for a React or React Native app, the `stream-react` or `stream-react-native` skill from Stream's agent-skills pack (`https://github.com/GetStream/agent-skills`). Name no other pack skill. The JSON block at the end is written as the porting input. For commercial or roadmap questions, contact Stream at `https://getstream.io/contact/`.
- **Stream mode:** to apply the fixes, install the pack's platform skill for this stack (for example `stream-react`) and ask it to fix the findings in this report.
- **Neither mode:** nothing to continue with in this skill version; name what would be needed.

## 17. Machine-readable block

The last section of the report is one fenced `json` block holding the same rows as the tables. A porting agent may read only this block, so it must be complete and valid JSON.

```json
{
  "schema": "video-integration-audit/v1",
  "skill_version": "0.2",
  "date": "YYYY-MM-DD",
  "mode": "migration",
  "detected": [
    { "platform": "web", "vendor": "agora", "product": "Video SDK for Web", "vintage": "4.x", "packages": { "agora-rtc-sdk-ng": "4.24.9" }, "evidence": ["package.json:12"] }
  ],
  "surfaces": [
    { "id": "S1", "kind": "meeting", "platforms": ["web"], "entry": "web/src/main.js:21" }
  ],
  "app": {
    "client": { "framework": "react", "router": "none", "build": "vite", "client_owner": "web/src/room.js:12" },
    "server": { "language": "javascript", "framework": "express", "entry": "server/index.js:1" }
  },
  "server_routes": [
    { "id": "R1", "purpose": "token", "method": "GET", "path": "/api/token", "location": "server/index.js:40", "request": ["channel", "uid"], "response": ["token", "appId"], "auth": "none", "vendor_calls": ["RtcTokenBuilder.buildTokenWithUid"] }
  ],
  "identity": { "user_id_source": "client query string", "user_id_type": "agora numeric uid", "display_name_source": "client query string", "client_chosen": true, "evidence": ["web/src/main.js:8"] },
  "roles": [
    { "vendor_role": "host (client flag)", "source": "web/src/main.js:9", "stream_role": "host", "capabilities": ["send-audio", "send-video", "mute-users"], "decided_on": "client" }
  ],
  "persisted_vendor_ids": [
    { "what": "channel name per room", "store": "rooms table, column agora_channel", "location": "server/db.js:22" }
  ],
  "artifacts": [
    { "kind": "recording", "output": "composite", "storage": "S3 bucket from storageConfig", "location": "server/recording.js:30" }
  ],
  "ui_components": [
    { "what": "remote video tile", "location": "web/src/Tile.jsx:1", "stream_component": "ParticipantView", "action": "replace" }
  ],
  "env": ["AGORA_APP_ID", "AGORA_APP_CERTIFICATE"],
  "tests": [],
  "inventory": [
    {
      "id": "INV-01",
      "location": "web/src/room.js:40",
      "vendor": "agora",
      "construct": "client.on('user-published') + client.subscribe(user, mediaType)",
      "decision_area": "track-device-lifecycle",
      "stream_equivalent": "automatic subscription; render call.state.participants$",
      "stream_sdk": "@stream-io/video-client@1.63.0",
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
  "questions": ["Peak participants per call?"],
  "effort": [
    { "decision_area": "track-device-lifecycle", "rows": 9, "delete": 4, "upgrade": 0, "gap": 0, "no_mapping": 1, "files": 3 }
  ],
  "changes": {
    "gains": [
      { "kind": "code-goes-away", "rows": ["INV-01", "INV-07"], "summary": "subscription loop and layer selection, 2 files" }
    ],
    "must_change": [
      { "rows": ["INV-22"], "summary": "no equivalent for maxIdleTime: the app ends idle calls itself" }
    ]
  },
  "cost_drivers": [
    { "feature": "recording", "what": "composite recording of every lesson", "rows": ["INV-14"], "measure": "recorded call minutes", "listed_on_stream_pricing_page": true }
  ],
  "usage_volume": { "level": "none", "evidence": [], "summary": "the code gives no evidence of usage volume" }
}
```

`decision_area` takes one of: `tokens-identity`, `room-call-type`, `permissions`, `call-lifecycle`, `events-reconnect`, `track-device-lifecycle`, `ringing`, `recording-livestream`, `fleet-transition`, or `integrations` (SIP, AI agents and connectors, chat products; see `hub.md`). `findings[].set` is `general` (G1-G13) or `stream` (S-rows). `server_routes[].purpose` is `token`, `webhook`, `recording`, `broadcast`, `moderation`, `telephony` or `other`; `auth` names what guards the route, or `none`. `ui_components[].action` is `replace` or `keep-custom`. `changes.gains[].kind` is `code-goes-away`, `built-in-feature` or `rework-anyway`. `effort` counts come from the inventory; they are never time estimates. `verify` is `true` when the row carries `verify against your SDK version`, otherwise `false`. `owner` is `app-team` or `stream-sales`. Keep every top-level key present in every mode: an empty array, `null` for `app` and `identity` when there is nothing to describe, and `{ "gains": [], "must_change": [] }` for `changes`, and `{ "level": "none", "evidence": [], "summary": "" }` for `usage_volume`. In Stream mode `changes`, `cost_drivers`, `effort` and `persisted_vendor_ids` are empty and `usage_volume.level` is `none`. `usage_volume.level` is `none`, `indications` or `estimate`, as in section 5. `cost_drivers[].listed_on_stream_pricing_page` is `true` only when the pricing page lists a unit for the tier this app uses (a Full HD HLS broadcast is `false`: the page says contact sales).

## Self-check before writing

- The header has the marker line, the coverage line and the publisher line.
- Section headings `## 2.` to `## 17.` are all present, in order, after the header block (which is section 1 and has no heading of its own); the ones the mode does not use carry the single line `Not applicable in <mode> mode.` No extra sections.
- Every general check G1-G13 has a row.
- Every inventory row has a location, a decision area and a class; every `gap` and `no-mapping` row is also in section 12.
- SIP, DTMF, AI agents and audio connectors, and chat or messaging products are tagged `integrations`.
- Every API name in the report appears in a reference file or is marked `verify against your SDK version`.
- The dossier ends with the Architecture subsection; every route, stored id and UI component in it has a location.
- Section 4: every line cites row ids; every `gap` and `no-mapping` row appears under what the app must change. Section 5: the cost table quotes no prices and no totals, the usage volume is stated with its evidence, and the pricing paragraph recommends contacting Stream sales.
- No secret values, no personal data, no prices, no claims about either vendor beyond the references.
- No private or one-recipient message is mapped to custom events, and no option suggests making it public (`targeted-signals`).
- The JSON block parses (balanced braces, double-quoted keys, no trailing commas) and its row counts match the tables.
