<!-- video-integration-audit report v1 -->
# Video integration audit: VideoClaim (ExampleCompany Mutual remote inspections)

- Date: 2026-10-28
- Skill: video-integration-audit 0.2
- Mode: migration
- Scope: `videoclaim/`; 13 files read (`package.json`, `README.md`, `.gitignore`, `server.js`, `lib/vonage.js`, `lib/store.js`, `public/adjuster.html`, `public/join.html`, `public/claim.html`, `public/js/adjuster.js`, `public/js/join.js`, `public/js/claim.js`, `public/css/app.css`); excluded: `.claude/skills/video-integration-audit/` (this skill's own directory). No `node_modules/`, build output or lockfile is present, so versions come from the exact pins in `package.json` and the CDN script URLs. Stylesheets were searched for the vendor's DOM class names (never excluded).
- Coverage: code-level for the web client and the Node server. This version has code-level references for web apps only.
- Read-only: this audit read files and wrote this report. It ran no code and made no network calls.
- Publisher: this skill is published by Stream (getstream.io), a video platform vendor. Its checks are the same for every platform; statements about vendors come from their public documentation.

## 2. What was detected

| Platform | Vendor / product | Vintage | Packages and versions | Evidence |
|---|---|---|---|---|
| server | Vonage Video API, Node server SDK | unified (application id + private key: `new Video({ applicationId, privateKey })`) | `@vonage/video` 1.31.1 | `package.json:13`, `lib/vonage.js:5-8` |
| web client | Vonage Video API web client (global `OT`) | unified CDN (`unpkg.com/@vonage/client-sdk-video`) | `@vonage/client-sdk-video` 2.35.3 | `public/adjuster.html:44`, `public/join.html:27` |

Both versions match the versions this skill's Vonage reference is pinned to. `README.md:3-4` says the integration moved from `opentok` to the Vonage packages in 2024; no legacy `opentok` code remains. Mode is `migration`: Vonage is the only video platform in the code, with no Stream code.

## 3. Usage dossier

**Overview.** VideoClaim is an internal remote-inspection desk for an insurer. A claims adjuster opens a claim file, creates an "inspection" (one Vonage session), and sends the policyholder a link. Both join a 1:1 video session from the browser: the adjuster from a desktop desk page, the policyholder from a phone page that starts on the rear camera. During the session the adjuster can grab still photos of the policyholder's video for the claim file, both sides can text-chat, and the server records the session as one composed 720p archive. Ending the inspection disconnects everyone and stops the archive; the adjuster later downloads the recording into the claim file.

### S1 - Inspection session (`meeting`)

- **Where it starts.** The desk page `public/adjuster.html` (opened from the claim file, `public/js/claim.js:11,16`). "Create inspection" calls `POST /api/claims/:claimId/inspections` (`public/js/adjuster.js:49-55`), which creates a routed Vonage session with manual archiving (`server.js:62-65`), mints a 30-day policyholder token and builds the join link with the token in the query string (`server.js:70-74`). The adjuster copies the link (`public/js/adjuster.js:56-61`) and sends it outside the app.
- **Adjuster joins.** "Join inspection" fetches `GET .../token?role=adjuster` (`public/js/adjuster.js:62-68`, `server.js:82-89`), then `connect()` builds the session, publisher and handlers and calls `session.connect` (`public/js/adjuster.js:122-164`). On `sessionConnected` the page asks the server to start recording (`public/js/adjuster.js:130-137`).
- **Policyholder joins.** `public/join.html` reads `claim`, `inspection` and `token` from the URL (`public/js/join.js:30-34`), fetches the inspection, refuses on the client if `endedAt` is set (`public/js/join.js:35-36`), connects with the URL token and publishes with `facingMode: 'environment'` (`public/js/join.js:6-28,53-58`). A "Switch camera" button tears down and re-creates the publisher on the next camera (`public/js/join.js:63-70`).
- **Photos.** "Take photo" calls `subscriber.getImgData()` on the policyholder's stream and posts the PNG to `POST /api/inspections/:id/photos` (`public/js/adjuster.js:79-87`, `server.js:91-105`). This is a core claim-evidence feature.
- **Chat.** Both pages send `session.signal({ type: 'chat' })` to the whole session (no `to:`) and render `signal:chat` events (`public/js/adjuster.js:88-96,149-152`, `public/js/join.js:45-48,81-89`). Messages are not stored anywhere.
- **Ending.** "End inspection" force-disconnects every known connection, the desk's own last (`public/js/adjuster.js:97-105`), then calls `POST /api/inspections/:id/end`, which sets `endedAt` and stops the archive (`server.js:139-151`). Both pages reload two seconds after any `sessionDisconnected` (`public/js/adjuster.js:153-156`, `public/js/join.js:49-52`).
- **State.** Page-level globals (`session`, `publisher`, `subscriber`, `micOn`, `cameraOn`, `connections`) in each page script; inspection records in `data/claims.json`.

### S2 - Inspection recording (`recording`)

- Started by the server when the desk connects (`server.js:107-133`): one composed archive, `1280x720`, `bestFit` layout, audio and video (`server.js:114-121`). A start already in flight is shared through the in-memory `starts` map (`server.js:9,111-127`); once an `archiveId` exists the route returns that archive whatever its status and never starts another (`server.js:110`).
- Status is polled every 5 seconds by the desk (`public/js/adjuster.js:15-29`) through `GET /api/inspections/:id/recording` (`server.js:134-138`).
- Stopped by the end route (`server.js:144-148`).
- Saved on an explicit click: `POST /api/inspections/:id/recording/save` checks the archive is `available` with a URL, downloads the MP4 and writes it to `data/recordings/` (`server.js:152-171`). `README.md:24-27` states the Vonage application uses Vonage-hosted archive storage and that downloads happen only on that click.

### Architecture

- **Client:** plain JavaScript with jQuery 3.7.1 from a CDN (`public/adjuster.html:43`, `public/join.html:26`); multi-page static HTML served by `express.static` (`server.js:13`); no router, no build tool (`package.json:6-8` has only `start`). The Vonage session object is a page-level global created in `connect()` (`public/js/adjuster.js:1,123`) and after the inspection fetch (`public/js/join.js:1,37`); its lifetime is the page.
- **Server routes** (Express, `server.js`; none has authentication, and there is no auth middleware anywhere):
  - `POST /api/claims/:claimId/inspections` (`server.js:60-78`): `createSession`, `generateClientToken` (policyholder, 30 days). Request: none. Response: `{ applicationId, inspection }` including `sessionId` and `joinUrl` with the token.
  - `GET /api/claims/:claimId/inspections/:id` (`server.js:79-81`): returns `{ applicationId, inspection }`, including the stored `joinUrl` and therefore the policyholder token. Called by both pages.
  - `GET /api/claims/:claimId/inspections/:id/token` (`server.js:82-89`): `generateClientToken`. Request: query `role` (free text, default `policyholder`). Response: `{ applicationId, inspection, token }`. Refuses ended inspections (`server.js:84`).
  - `GET /api/claims` (`server.js:45`) and `GET /api/claims/:id` (`server.js:56-58`): return claim records with every inspection's `joinUrl` (token), `sessionId`, `archiveId`, photo and recording URLs.
  - `POST /api/inspections/:id/recording/start` (`server.js:107-133`): `startArchive`, `getArchive`.
  - `GET /api/inspections/:id/recording` (`server.js:134-138`): `getArchive`.
  - `POST /api/inspections/:id/end` (`server.js:139-151`): `getArchive`, `stopArchive`.
  - `POST /api/inspections/:id/recording/save` (`server.js:152-171`): `getArchive`, then `fetch(archive.url)`.
  - `POST /api/inspections/:id/photos` (`server.js:91-105`): stores the `getImgData()` PNG.
  - Static `/files/photos` and `/files/recordings` (`server.js:14-15`).
  - No webhook or callback routes.
- **Identity:** there are no user accounts. The only identity is a role label inside the Vonage token's `data` (`server.js:36`), taken from the `role` query parameter (`server.js:87`, sent as `adjuster` by `public/js/adjuster.js:64`) or fixed to `policyholder` (`server.js:71`). Display names are hard-coded publisher names `Adjuster` and `Policyholder` (`public/js/adjuster.js:158`, `public/js/join.js:9`). Chat labels "You" by comparing the sender's `connectionId` with the local one (`public/js/adjuster.js:150`, `public/js/join.js:46`). The client chooses the role label.
- **Stored vendor ids:** `inspection.sessionId` (`server.js:67`), `inspection.archiveId` (`server.js:68,122`), `inspection.joinUrl` containing the policyholder token value (`server.js:74`; value not copied: `<redacted>`), `claim.recordings[].archiveId` (`server.js:165`), all in `data/claims.json` (`lib/store.js:7,15-17`); in-memory `starts` map of pending archive starts keyed by inspection id (`server.js:9,126`).
- **Recordings and other artifacts:** composed MP4 archives in Vonage-hosted storage (`README.md:24-26`), copied on click to `data/recordings/<inspection>-<archive>.mp4` (`server.js:162-163`) and served without authentication at `/files/recordings` (`server.js:15`). Photos as PNG in `data/photos/` (`server.js:95-97`), served at `/files/photos` (`server.js:14`). Whether a storage target or archive callback is configured in the Vonage dashboard is not in the code (section 13).
- **Video UI:**
  - Remote tile `#remote` with Vonage's default subscriber UI (no `insertDefaultUI: false`) on both pages (`public/adjuster.html:23`, `public/join.html:14`, `public/js/adjuster.js:139-141`, `public/js/join.js:39-41`): plain JS port -> keep custom, built on `call.bindVideoElement(el, sessionId, 'videoTrack')` + `call.bindAudioElement(el, sessionId)` (`<ParticipantView>` only if the app moves to React).
  - Local picture-in-picture `#local` with Vonage's default publisher UI (`public/js/adjuster.js:157-159`, `public/js/join.js:13`, `public/css/app.css:18`): keep custom, same binding.
  - Elements the default UI shows: name bar and mute button (both hidden by `public/css/app.css:31-32`), audio-level meter, video-off indicator, the blocked-audio control on subscribers (`audioBlockedDisplayMode` not set), and the "Archiving on" light on each page's own publisher tile while the archive runs (`archiveStatusDisplayMode` not set, and no CSS hides it). On `join.html` that light is the only recording notice the policyholder gets. Keep custom: rebuild a recording indicator from `call.state.recording$` and a blocked-audio control on both pages.
  - Controls (mic, camera, photo, end; switch camera on the phone) (`public/adjuster.html:24-29`, `public/join.html:15-19`): keep custom (`CallControls` is a React component).
  - Recording indicator and status (`public/adjuster.html:31-33`): keep custom, driven by `call.state.recording$` and `call.recording_ready`.
  - Chat panel (`public/adjuster.html:36-40`, `public/join.html:20-24`): keep custom.
- **Environment:** `VONAGE_APPLICATION_ID` (`lib/vonage.js:6`, `server.js:42`), `VONAGE_PRIVATE_KEY` (a file path, `lib/vonage.js:7`), `PUBLIC_BASE_URL` (join links, `server.js:72`). `.gitignore:3-4` lists `private.key` and `.env`; neither file is present in the tree.
- **Tests:** none.

## 4. What changes for this app on Stream

### What the app gains

- **Code that goes away:** 6 `delete` rows in 3 files (`public/js/adjuster.js`, `public/js/join.js`, `public/css/app.css`): the two subscribe loops (INV-32, INV-33), the two explicit publish calls (INV-26, INV-27), the connection map kept only to force-disconnect everyone (INV-10), and the stylesheet rules on Vonage's DOM classes (INV-37). As an option, none of the app's Video UI is replaceable by a prebuilt component while it stays plain JavaScript: Stream's prebuilt components are React.
- **Built-in features for hand-built ones:**
  - Ending an inspection (INV-09): the app emulates "end for everyone" with a client-side force-disconnect loop, a server `endedAt` flag and a client-side check on the join page, while the 30-day policyholder token keeps working against the session. On Stream, `call.end()` (node-sdk) ends the call for everyone, and removing `join-ended-call` from the call type makes the platform refuse rejoins, so an ended inspection stays ended without trusting the client.
  - Noise cancellation (INV-38): the desk has no noise filter beyond the browser's own. Stream's noise cancellation (`@stream-io/audio-filters-web`) is available for the desk; Stream documents it for desktop browsers only, so the policyholder's phone would not get it. Offered, not assumed.

### What the app must change

- Photo capture (INV-34): `subscriber.getImgData()` has no mapping in this skill. On a plain JS port the app owns the `<video>` element it binds with `call.bindVideoElement`, so a still can be captured from that element with browser canvas code; the resulting resolution depends on what the SDK delivers to that element (verify against your SDK version). Flagged FL-01.
- Default Vonage tile UI (INV-35, INV-36): in plain JavaScript Stream renders no tile chrome. The app must build its own recording indicator on both pages (the join page's only recording notice today is Vonage's archiving light), a blocked-audio control (G9 passes today only because of Vonage's default UI), and any video-off state it wants to show. Flagged FL-05.
- Loading the SDK (INV-18, INV-19): the pages load Vonage from a CDN script tag and the app has no build step. `@stream-io/video-client` needs either a bundling step or a module CDN import (verify against your SDK version).
- Identity (INV-02, INV-03, INV-04): Stream tokens are per user, so the app needs user ids it does not have today: one per adjuster and one per policyholder per inspection, minted by the server. The 30-day token in the link must become an invite the server exchanges for a short-lived, call-scoped token. Flagged FL-09.
- Chat (INV-14 to INV-17): custom events are limited to 5 KB (Vonage signals 8 kB, and the input has no length limit), have no reconnect buffering, and do not document whether the sender receives its own event, so the app must render its own messages locally. Flagged FL-07.
- No `gap` rows and no blocker-class requirement (end-to-end encryption, fixed IPs, SIP dial-out) were found in the code.

## 5. Cost analysis

**What drives cost.**

| Feature | What the code does | Rows | What to measure |
|---|---|---|---|
| Participant minutes | 1:1 inspections: the adjuster and the policyholder each publish camera and microphone and receive the other's video, full-stage on the desk and in a stage of up to 55vh on the phone (`public/css/app.css:16,36`). No published resolution or frame rate is set, so SDK defaults apply | INV-24, INV-25 | participant minutes per inspection (about 2 x inspection length) and the resolution each participant receives; set the call type's target resolution deliberately (section 10) |
| Recording | one composite 1280x720 recording of every inspection, from the adjuster joining until the inspection ends | INV-39, INV-40, INV-41 | recorded call minutes at 720p |
| Noise cancellation (only if the upgrade is adopted) | none today; would apply to the adjuster's desktop microphone | INV-38 | adjuster participant minutes with noise cancellation on |

Not used: HLS, RTMP, transcription, closed captions, SIP, individual or raw recording.

**Usage volume:** `none`. The code gives no evidence of how many inspections run: no scaling configuration, load targets or configured limits. `README.md:30` asks for a single server process against a local JSON file, which is a deployment constraint, not a volume figure. The volume questions are in section 13.

**Pricing.** Stream's pricing page (`https://getstream.io/video/pricing/`) bills participant minutes by the aggregated resolution of the video each participant receives, and lists recording separately, by quality from audio to Full HD, per 1,000 call minutes, and noise cancellation per 1,000 participant minutes. Vonage publishes its own prices for sessions and archiving on its pricing page. Every feature this app uses is listed on Stream's page. Stream's listed prices are pay-as-you-go, and its Enterprise plan lists volume discounts behind Contact Sales, so ask Stream sales (`https://getstream.io/contact/`) for a quote at your real inspection volumes: a volume quote, not the list prices, is the right basis for comparing the two vendors.

## 6. Integration inventory

All Stream equivalents are for `@stream-io/video-client` 1.63.0 (client) and `@stream-io/node-sdk` 0.8.14 (server) unless marked otherwise. Surface S1 = inspection session, S2 = recording.

### tokens-identity

| ID | Location | Vendor construct | Decision area | Stream equivalent (SDK version) | Class | Notes |
|---|---|---|---|---|---|---|
| INV-01 | `lib/vonage.js:5-8` | `new Video({ applicationId, privateKey })` | tokens-identity | `new StreamClient(apiKey, apiSecret)` (`@stream-io/node-sdk` 0.8.14) | rename | Read the API secret from the server environment, as the private key path is today |
| INV-02 | `server.js:33-40` | `generateClientToken(sessionId, { role: 'moderator', data: { role, inspection }, expireTime })` | tokens-identity | `client.generateUserToken({ user_id, validity_in_seconds })`; role moves to the call member; `data` (role label, inspection id) moves to call-member `custom` via `call.updateCallMembers({ update_members: [{ user_id, custom }] })` | shape-shift | Per-session token -> per-user token (`token-scope`). The app has no user ids yet (FL-09) |
| INV-03 | `server.js:70-74` | 30-day policyholder token from `generateClientToken` placed in the join URL and stored in `joinUrl` | tokens-identity | `generateCallToken({ user_id, call_cids, role })` for the policyholder, scoped to the inspection's call, minted when the link is opened | shape-shift | Put an opaque invite id in the link, not a token; stop storing tokens in `data/claims.json` |
| INV-04 | `server.js:82-89` | token route taking `role` from `req.query.role` | tokens-identity | token route returning `{ vendor, apiKey, token, callId }`, role decided on the server; client `tokenProvider` | shape-shift | `vendor` field is the server-decided switch for the transition (section 11) |
| INV-05 | `server.js:41-43` | `connectionInfo` returns `VONAGE_APPLICATION_ID` to the client | tokens-identity | return the Stream API key (public) | rename | |

### room-call-type

| ID | Location | Vendor construct | Decision area | Stream equivalent (SDK version) | Class | Notes |
|---|---|---|---|---|---|---|
| INV-06 | `server.js:62-67` | `createSession({ mediaMode: MediaMode.ROUTED, archiveMode: ArchiveMode.MANUAL })`; `sessionId` stored | room-call-type | `client.video.call(type, id)` + `call.getOrCreate({ data: { members } })` with call id = inspection id and both parties as members | shape-shift | Routed media matches Stream's SFU model. `sessionId` storage goes; the inspection record and its link stay |
| INV-07 | `server.js:64` | `archiveMode: ArchiveMode.MANUAL` | room-call-type | call-type recording mode: app-started, or `auto-on` (see INV-41); exact mode names verify against your SDK version | config | |

### permissions

| ID | Location | Vendor construct | Decision area | Stream equivalent (SDK version) | Class | Notes |
|---|---|---|---|---|---|---|
| INV-08 | `server.js:35` | token role `moderator` for every caller, policyholder included | permissions | call-member roles and the call type's grid: an adjuster role with `send-audio`, `send-video`, `end-call`; a policyholder role with `send-audio`, `send-video` only | config | Role mapping in section 10; today the policyholder can force-disconnect and force-mute the adjuster (G8) |

### call-lifecycle

| ID | Location | Vendor construct | Decision area | Stream equivalent (SDK version) | Class | Notes |
|---|---|---|---|---|---|---|
| INV-09 | `public/js/adjuster.js:97-105`, `server.js:139-151`, `public/js/join.js:36` | "end for everyone" built from `session.forceDisconnect(connection)` per connection + server `endedAt` + client-side ended check | call-lifecycle | `call.end()` (node-sdk) from the end route, or `call.endCall()` (client, needs `end-call`); remove `join-ended-call` from the call type | upgrade | Vonage sessions never end, so today an ended inspection can still be joined with the link token. `public/js/adjuster.js:101` compares against `session.connection.id` while the rest of the file uses `connectionId` (`:125,:150`); if `id` is not a `Connection` property the sort does nothing and the desk may disconnect itself first (verify) |
| INV-10 | `public/js/adjuster.js:124-129` | `connectionCreated` / `connectionDestroyed` map of connections | call-lifecycle | none needed: its only use is INV-09 | delete | `identity-keying`: Stream acts on users, not connections |

### events-reconnect

| ID | Location | Vendor construct | Decision area | Stream equivalent (SDK version) | Class | Notes |
|---|---|---|---|---|---|---|
| INV-11 | `public/js/adjuster.js:130-137` | `sessionConnected` (enables controls, starts recording) | events-reconnect | `call.join()` resolves; calling state `JOINED` | rename | Recording start moves to the server or the call type (INV-41) |
| INV-12 | `public/js/adjuster.js:153-156` | `sessionDisconnected` -> reload after 2 s, any reason | events-reconnect | calling state `LEFT` / `OFFLINE` / `RECONNECTING_FAILED` from `call.state.callingState$`; show `RECONNECTING` | shape-shift | G7 |
| INV-13 | `public/js/join.js:49-52` | `sessionDisconnected` -> reload after 2 s, any reason | events-reconnect | as INV-12 | shape-shift | G7 |
| INV-14 | `public/js/adjuster.js:92` | `session.signal({ type: 'chat', data })` | events-reconnect | `call.sendCustomEvent({ type, ... })` | behavioral-diff | 5 KB limit (8 kB today, no `maxlength` on `public/adjuster.html:39`); no reconnect buffering; render own message locally |
| INV-15 | `public/js/join.js:85` | `session.signal({ type: 'chat', data })` | events-reconnect | `call.sendCustomEvent({ type, ... })` | behavioral-diff | as INV-14 (`public/join.html:23`) |
| INV-16 | `public/js/adjuster.js:149-152` | `session.on('signal:chat')`, sender from `event.from.connectionId` | events-reconnect | `call.on('custom', e => ...)` filtered on a payload `type`; sender from `event.user.id` | shape-shift | `event.data` is inserted as HTML (FL-03); fix before or during the port |
| INV-17 | `public/js/join.js:45-48` | `session.on('signal:chat')`, sender from `event.from.connectionId` | events-reconnect | as INV-16 | shape-shift | as INV-16 |

### track-device-lifecycle

| ID | Location | Vendor construct | Decision area | Stream equivalent (SDK version) | Class | Notes |
|---|---|---|---|---|---|---|
| INV-18 | `public/adjuster.html:44` | `<script src=".../@vonage/client-sdk-video@2.35.3/dist/js/opentok.min.js">` | track-device-lifecycle | `@stream-io/video-client` 1.63.0 | shape-shift | No build step in the app: bundling or a module CDN import is needed (verify against your SDK version) |
| INV-19 | `public/join.html:27` | same script tag | track-device-lifecycle | `@stream-io/video-client` 1.63.0 | shape-shift | as INV-18 |
| INV-20 | `public/js/adjuster.js:123` | `OT.initSession(applicationId, sessionId)` | track-device-lifecycle | `new StreamVideoClient({ apiKey, user, tokenProvider })` + `client.call(type, id)` | shape-shift | |
| INV-21 | `public/js/join.js:37` | `OT.initSession(applicationId, sessionId)` | track-device-lifecycle | as INV-20 | shape-shift | |
| INV-22 | `public/js/adjuster.js:160-163` | `session.connect(token, callback)` | track-device-lifecycle | `await call.join()` in `try/catch` | shape-shift | G4 |
| INV-23 | `public/js/join.js:53-58` | `session.connect(token, callback)` | track-device-lifecycle | `await call.join()` in `try/catch` | shape-shift | G4 |
| INV-24 | `public/js/adjuster.js:157-159` | `OT.initPublisher('local', { name: 'Adjuster', ... })` | track-device-lifecycle | `call.camera.enable()` + `call.microphone.enable()`; `name` -> user name; render local with `call.bindVideoElement` | shape-shift | `publish-on-join`: set device state before `join()`; no `resolution` set today (G11) |
| INV-25 | `public/js/join.js:6-15` | `OT.initPublisher('local', { facingMode: 'environment', videoSource, publishAudio, publishVideo, name: 'Policyholder' })` | track-device-lifecycle | `call.camera.selectDirection('back', { enableCamera: false })` before join, then `call.camera.enable()` / `call.microphone.enable()` | shape-shift | |
| INV-26 | `public/js/adjuster.js:162` | `session.publish(publisher, callback)` | track-device-lifecycle | implicit once devices are enabled and the call is joined | delete | |
| INV-27 | `public/js/join.js:16-17` | `session.publish(publisher, callback)` | track-device-lifecycle | implicit | delete | |
| INV-28 | `public/js/join.js:63-70` | camera switch: `session.unpublish(publisher)` + `publisher.destroy()` + new publisher with `videoSource: deviceId` | track-device-lifecycle | `call.camera.flip()` or `call.camera.select(deviceId)` | shape-shift | Today the switch removes the stream: the desk sees `streamDestroyed`, its photo button disables, the composed archive loses the tile until republish. On Stream the participant stays |
| INV-29 | `public/js/join.js:19-26` | `OT.getDevices()` + `publisher.getVideoSource()` to enable the switch button | track-device-lifecycle | camera device list from the camera manager's state (`useCameraState().devices` is the React form; plain JS name verify against your SDK version) | shape-shift | Not needed if the button simply calls `call.camera.flip()` |
| INV-30 | `public/js/adjuster.js:69-78` | `publisher.publishAudio(bool)` / `publishVideo(bool)` | track-device-lifecycle | `call.microphone.toggle()` / `call.camera.toggle()` | rename | Drive the labels from device state (G13) |
| INV-31 | `public/js/join.js:71-80` | `publisher.publishAudio(bool)` / `publishVideo(bool)` | track-device-lifecycle | `call.microphone.toggle()` / `call.camera.toggle()` | rename | as INV-30 |
| INV-32 | `public/js/adjuster.js:138-148` | `streamCreated` -> `session.subscribe(event.stream, 'remote', ...)`; `streamDestroyed` | track-device-lifecycle | none: automatic; render from `call.state.participants$` with `call.bindVideoElement(el, sessionId, 'videoTrack')` + `call.bindAudioElement(el, sessionId)` | delete | `subscribe-loop`, `streams-vs-participants`: the photo button and "Policyholder connected" are keyed on the stream |
| INV-33 | `public/js/join.js:38-44` | `streamCreated` -> `session.subscribe(...)`; `streamDestroyed` | track-device-lifecycle | as INV-32 | delete | |
| INV-34 | `public/js/adjuster.js:82` | `subscriber.getImgData()` (photo for the claim file) | track-device-lifecycle | no mapping in this skill | no-mapping | FL-01 |
| INV-35 | `public/js/adjuster.js:139-141,157-159` | Vonage default tile UI on the desk (name bar and mute hidden by CSS; audio-level meter, video-off indicator, archiving light on the publisher, blocked-audio control on the subscriber) | track-device-lifecycle | plain JS: none; build tiles with `call.bindVideoElement` / `call.bindAudioElement`, a recording indicator from `call.state.recording$`, a blocked-audio control with `call.resumeAudio()` | shape-shift | The desk already has its own REC indicator (`public/adjuster.html:31`) |
| INV-36 | `public/js/join.js:8-13,39-41` | Vonage default tile UI on the policyholder page, including the "Archiving on" light on the policyholder's own tile | track-device-lifecycle | as INV-35 | shape-shift | The archiving light is the policyholder's only recording notice: rebuild it (FL-05) |
| INV-37 | `public/css/app.css:31-32` | rules on `.OT_root .OT_name`, `.OT_name_bar`, `.OT_mute` | track-device-lifecycle | none: rules on the app's own tile elements | delete | Carry the product choice over: no name bar, no per-tile mute button |
| INV-38 | `public/js/adjuster.js:157` | `OT.initPublisher` with no noise filter on a speaking surface | track-device-lifecycle | `@stream-io/audio-filters-web` (`NoiseCancellation`) with `call.microphone.enableNoiseCancellation(...)` | upgrade | Desktop browsers only: the desk benefits, the policyholder's phone does not. Offered, not assumed |

### recording-livestream

| ID | Location | Vendor construct | Decision area | Stream equivalent (SDK version) | Class | Notes |
|---|---|---|---|---|---|---|
| INV-39 | `server.js:114-121` | `startArchive(sessionId, { name, outputMode: COMPOSED, hasAudio, hasVideo, ... })` | recording-livestream | node-sdk `call.startRecording({ recording_type: 'composite' })` | shape-shift | Archive `name` (claim number / inspection id) has no stated equivalent: keep it in the app's own record |
| INV-40 | `server.js:117-118` | archive `resolution: '1280x720'`, `layout: { type: 'bestFit' }` | recording-livestream | call-type recording layout `grid` (`settings.recording.layout`) and quality 720p (setting name verify against your SDK version) | config | Check `grid` against `bestFit` visually for a 1:1 call |
| INV-41 | `server.js:107-133`, `public/js/adjuster.js:133-136` | start-on-desk-connect orchestration: `starts` dedupe map, `archiveId` reuse | recording-livestream | call-type recording mode `auto-on`, or keep a single server start (INV-39) if recording must begin at the adjuster's join | config | Product decision FL-06; `archive-auto-timers`: today a stopped archive is never restarted (`server.js:110`) |
| INV-42 | `server.js:134-138` | `getArchive(archiveId)` status route | recording-livestream | `call.recording_ready` (WebSocket and webhook); `call.listRecordings()` | shape-shift | |
| INV-43 | `public/js/adjuster.js:15-29` | client polls archive status every 5 s | recording-livestream | recording state from `call.state.recording$`; readiness from `call.recording_ready` | shape-shift | |
| INV-44 | `server.js:144-148` | `getArchive` then `stopArchive(archiveId)` | recording-livestream | `call.stopRecording()` (node-sdk form verify against your SDK version); `call.end()` (INV-09) | rename | |
| INV-45 | `server.js:152-171` | download `archive.url` after status `available`, write MP4 to `data/recordings/` | recording-livestream | recording URL from `call.listRecordings()` or the `call.recording_ready` payload; keep the app's own download into the claim file | shape-shift | |
| INV-46 | `README.md:24-27` | Vonage-hosted archive storage (dashboard, not code) | recording-livestream | Stream-managed storage by default, or Amazon S3, Google Cloud Storage or Azure Blob | config | Vonage's archiving guide: archives without a storage target can be downloaded for 72 hours (FL-04) |

## 7. Behaviour changes to plan for

| ID | Trap | Evidence | Rows | What the port must do instead |
|---|---|---|---|---|
| BC-01 | `subscribe-loop` | `public/js/adjuster.js:138-148`, `public/js/join.js:38-44` | INV-32, INV-33 | Delete both loops; bind elements per participant from `call.state.participants$`. The photo feature depends on the subscriber object today: rebuild it on the app-owned video element (INV-34) |
| BC-02 | `streams-vs-participants` | photo button enabled on `streamCreated`, disabled on `streamDestroyed` (`public/js/adjuster.js:142,147`) | INV-32 | Enable "Take photo" when the policyholder participant has a published video track (`publishedTracks`); a policyholder who turns the camera off stays a participant on Stream |
| BC-03 | `publish-on-join` | publishing happens in the connect callback (`public/js/adjuster.js:162`, `public/js/join.js:57`) | INV-24, INV-25, INV-26, INV-27 | Set camera direction and camera/microphone state before `call.join()`; do not rely on call-type defaults |
| BC-04 | `targeted-signals` / `signal-targeting` | `public/js/adjuster.js:92,150`, `public/js/join.js:46,85` | INV-14, INV-15, INV-16, INV-17 | No `to:` is used, so the audience is unchanged (all call members watching the call), but: enforce 5 KB; nothing critical across reconnects; render the sender's own message locally and filter echoes on `event.user.id` (Stream does not document whether the sender receives its own event) |
| BC-05 | `identity-keying` | `forceDisconnect(connections[id])` (`public/js/adjuster.js:104`), map keyed on `connectionId` (`:125`), "You" decided by `connectionId` (`:150`, `public/js/join.js:46`) | INV-09, INV-10, INV-16, INV-17 | Key on Stream user ids; if a policyholder opens the link on two devices both are the same user |
| BC-06 | `session-immortality` | ended state is an app flag (`server.js:141`) checked on the client (`public/js/join.js:36`) while the session and 30-day token stay valid | INV-06, INV-09 | Fresh call per inspection (call id = inspection id); `call.end()` on end; remove `join-ended-call` so the platform refuses rejoins |
| BC-07 | `server-token-per-session` / `token-scope` | `generateClientToken(inspection.sessionId, ...)` per inspection (`server.js:39`) | INV-02, INV-03, INV-04 | One user token per user session; access to the inspection from call membership and the call type; call-scoped token for policyholders |
| BC-08 | `archive-auto-timers` | start route returns the existing archive whatever its status and never starts another (`server.js:110`) | INV-39, INV-41 | Re-baseline recording continuity on Stream (no auto-stop timing is stated in this skill's references); decide what happens when a recording stops while the inspection continues |

## 8. Stream best-practice findings

Not applicable in migration mode.

## 9. General findings

| Check | Verdict | Severity | Evidence | Fix |
|---|---|---|---|---|
| G1. Secrets stay on the server | PASS | Blocker | The private key is read on the server from the path in `VONAGE_PRIVATE_KEY`, default `./private.key` (`lib/vonage.js:7`); `.gitignore:3-4` excludes `private.key` and `.env`; neither is present in the tree; no key, secret or server SDK in `public/` | Keep it that way; serve only `public/` (already `server.js:13`) |
| G2. Tokens minted per user, expire, renewed | FAIL | High | Policyholder token expires after 30 days (`server.js:70-71`), travels in the link URL (`server.js:73-74`), is stored in `data/claims.json` and returned by `GET /api/claims` and `GET .../inspections/:id` (`server.js:45,79-81`). Adjuster token uses the SDK default expiry (24 hours per this skill's reference, `server.js:87`) and is fetched per connection, which is the Vonage renewal path | Mint the policyholder token when the link is opened, with an expiry of hours, not days; put an invite id in the link; stop storing and returning tokens |
| G3. Token endpoint authenticates and binds the caller | FAIL | Blocker | No authentication on any route (`server.js:45-171`). The token route mints a `moderator` token for anyone who knows a claim and inspection id, with the role label from `req.query.role` (`server.js:82-89`); `GET /api/claims` lists every claim and inspection id, and the stored link tokens (`server.js:45`). Whether a proxy restricts the desk routes is not in the code (`server.js:10` trusts a proxy) | Put the desk API, the token route and `/files` behind the app's authentication; decide the role on the server from the authenticated user; give the policyholder flow its own invite-based route that returns only that inspection's token |
| G4. Joining is error-handled | FAIL | Blocker | `session.connect` errors show an `alert` only (`public/js/adjuster.js:161`, `public/js/join.js:54`); the desk's Join button stays disabled (`public/js/adjuster.js:63`) and the phone page has no retry; `session.publish` errors are only logged (`public/js/adjuster.js:162`, `public/js/join.js:16-17`) | Show the failure in the page with Retry and Leave; re-enable Join; surface publish failures |
| G5. Camera and microphone failures are handled | FAIL | High | `OT.initPublisher` completion callbacks only log (`public/js/adjuster.js:159`, `public/js/join.js:13-15`); no `accessDenied` handler; a failed camera switch leaves the phone with no publisher and the switch button disabled (`public/js/join.js:65-69`) | Handle the publisher error and `accessDenied`; tell the user which device failed; keep the old publisher until the new one publishes |
| G6. Leaving releases the call and the devices | FAIL | High | No `session.disconnect(` anywhere; `publisher.destroy()` only in the camera switch (`public/js/join.js:68`); no `pagehide` / `beforeunload`; the policyholder page has no Leave control; the desk's "Claim file" header link leaves mid-session without disconnecting (`public/adjuster.html:10`) | Add Leave on the phone page; call `session.disconnect()` and `publisher.destroy()` on leave and on `pagehide` |
| G7. Reconnects are shown | FAIL | Medium | No `sessionReconnecting` / `sessionReconnected` handlers; every `sessionDisconnected` reloads the page after 2 s whatever its reason (`public/js/adjuster.js:153-156`, `public/js/join.js:49-52`); after a reload the desk's recording start returns the old archive even if it stopped (`server.js:110`) | Show "Reconnecting" on `sessionReconnecting`; on `sessionDisconnected` branch on the reason and offer Rejoin instead of reloading; restart recording if the archive is no longer `started` |
| G8. Permissions enforced on the server | FAIL | High | Every token is `moderator` (`server.js:35`), so the policyholder, or anyone holding the link, can force-disconnect and force-mute the adjuster; "End inspection" is hidden only by being absent from `join.html`; the end, recording and photo routes accept anyone (`server.js:91,107,139,152`) | Give the policyholder `role: 'publisher'` and the adjuster `moderator`; authenticate the end, recording and photo routes |
| G9. Autoplay-blocked audio is handled | PASS | High | Subscribers use Vonage's default UI with `audioBlockedDisplayMode` unset (`public/js/adjuster.js:139-141`, `public/js/join.js:39-41`); `public/css/app.css:31-32` hides `.OT_name` and `.OT_mute`, not the blocked-audio control named in this skill's references (check on a phone) | Keep the default UI; on a plain JS Stream port, rebuild the control with `call.resumeAudio()` in a click handler (detection API for plain JS: verify against your SDK version) |
| G10. Device permission prompts are explained | FAIL | High | The phone page shows a static "Allow your camera and microphone" line before publishing (`public/js/join.js:55`); no `accessDialogOpened` / `accessDenied` handlers on either page, so a denial is never explained; the desk says nothing | Handle `accessDialogOpened`, `accessAllowed`, `accessDenied`; explain how to re-enable the camera in the browser settings |
| G11. Cost-driving media settings are explicit | FAIL | Low | No publisher `resolution` or `frameRate` (`public/js/adjuster.js:157-158`, `public/js/join.js:7-12`); archive `resolution: '1280x720'` explicit (`server.js:117`). Photo stills are taken from the received policyholder video, so its resolution is also evidence quality | Set the policyholder's published resolution deliberately for photo quality, and the desk's lower; record the choices for the call type's target resolution |
| G12. Webhook and callback handlers verify the sender | N/A | High | The app receives no callbacks: archive status is polled (`server.js:134-138`); no callback route exists. If an archive callback URL is configured in the Vonage dashboard, nothing handles it | None now; verify any future recording webhook (`call.recording_ready`) before acting on it |
| G13. Media state shown follows the SDK | FAIL | Medium | Mic and camera labels come from local flags flipped by the app's buttons (`public/js/adjuster.js:2,69-78`, `public/js/join.js:2,71-80`); no Publisher `muteForced`, `mediaStopped` or `audioInputDeviceChanged` handlers, while every token can force-mute (`server.js:35`) | Update labels on the Publisher's `muteForced` and `mediaStopped`, or read `publisher.stream.hasAudio` / `hasVideo` |

## 10. Call-type configuration checklist

| Setting | Current source | Stream call-type setting | Value the code implies |
|---|---|---|---|
| Call type | one session per inspection (`server.js:62-65`) | a custom call type for inspections (built-in `default` with overrides also works) | one type, `inspection`, 1:1 |
| Target resolution | no publisher resolution (`public/js/adjuster.js:157`, `public/js/join.js:7-12`); remote video full-stage (`public/adjuster.html:23`, `public/css/app.css:16`) | `settings.video.target_resolution` | not set in code; choose from the photo-evidence need and the 470px desk stage. Stream bills per participant minute by the aggregated resolution each participant receives |
| Camera on at join | both pages publish video at join (`public/js/adjuster.js:157`, `public/js/join.js:9`) | `settings.video.camera_default_on` | true (also set explicitly on the client, BC-03) |
| Microphone on at join | both pages publish audio at join | `settings.audio.mic_default_on` | true |
| Recording mode | `archiveMode: MANUAL` (`server.js:64`), started when the desk connects (`public/js/adjuster.js:133`) | call-type recording mode (verify against your SDK version) | app-started at the adjuster's join, or `auto-on` (FL-06) |
| Recording layout | `layout: { type: 'bestFit' }` (`server.js:118`) | `settings.recording.layout` | `grid` |
| Recording quality | `resolution: '1280x720'` (`server.js:117`) | recording quality (setting name verify against your SDK version) | 720p |
| Audio-only recording | `hasAudio: true, hasVideo: true` (`server.js:119-120`) | `settings.recording.audio_only` | false |
| Recording storage | Vonage-hosted (`README.md:24-26`) | Stream-managed, or Amazon S3, Google Cloud Storage, Azure Blob | decide with FL-04 |
| Rejoining ended calls | `endedAt` flag and client check (`server.js:141`, `public/js/join.js:36`) | remove `join-ended-call` from both roles | an ended inspection stays ended |
| Backstage, join-ahead | none | n/a | not a livestream |
| Ring timeouts | none | n/a | link-join meeting, no ringing |

**Role mapping**

| Vendor role or flag | Source | Stream role | Capabilities needed | Decided on |
|---|---|---|---|---|
| token role `moderator` issued to the adjuster | `server.js:35`, requested with `?role=adjuster` (`public/js/adjuster.js:64`) | call-member role for adjusters (for example `host`) | `send-audio`, `send-video`, `end-call` (if the desk ends the call from the client); `start-record-call` only if recording starts from the client | client today (query parameter); server on Stream |
| token role `moderator` issued to the policyholder | `server.js:35,71` | call-member role for policyholders | `send-audio`, `send-video` | server (fixed), but over-privileged today |
| `data.role` label (`adjuster` / `policyholder`) | `server.js:36,87` | call-member `custom`, or simply the member role | none | client today |

## 11. Migration plan

**Size of the work**

| Decision area | Rows | delete | upgrade | gap | no-mapping | Files |
|---|---|---|---|---|---|---|
| tokens-identity | 5 | 0 | 0 | 0 | 0 | 2 |
| room-call-type | 2 | 0 | 0 | 0 | 0 | 1 |
| permissions | 1 | 0 | 0 | 0 | 0 | 1 |
| call-lifecycle | 2 | 1 | 1 | 0 | 0 | 3 |
| events-reconnect | 7 | 0 | 0 | 0 | 0 | 2 |
| track-device-lifecycle | 21 | 5 | 1 | 0 | 1 | 5 |
| ringing | 0 | 0 | 0 | 0 | 0 | 0 |
| recording-livestream | 8 | 0 | 0 | 0 | 0 | 3 |
| fleet-transition | 0 | 0 | 0 | 0 | 0 | 0 |
| integrations | 0 | 0 | 0 | 0 | 0 | 0 |
| **Total** | 46 | 6 | 2 | 0 | 1 | 9 distinct |

**Phase 1 - Server first** (INV-01 to INV-08, INV-39 to INV-46, server half of INV-09). Add the Stream server client next to the Vonage one; create a user per adjuster and per policyholder; add a `vendor` field to each inspection, decided at creation; for Stream inspections create the call with both members, mint per-user and call-scoped tokens through a route that returns `{ vendor, apiKey, token, callId }`; configure the inspection call type (section 10); port recording start, stop, listing and save. Fix G3 and G8 in the same step: the new routes should not repeat the unauthenticated, everyone-is-moderator model. Verify: a policyholder token cannot end the call or start recordings; an ended call refuses joins; a recording appears in `call.listRecordings()` and is saved to the claim file.

**Phase 2 - Client surfaces** (INV-10 to INV-37). Both participants of an inspection must be on the same platform, so the desk and the join page switch together per inspection: each page loads the SDK for `inspection.vendor`. Build the desk page first (more features: photo, end, recording state), then the join page, and release them together. Rebuild the recording indicator and blocked-audio control on both pages (INV-35, INV-36), the photo capture (INV-34), and chat with escaped rendering (INV-14 to INV-17). Verify on a phone: rear camera at join, camera switch, photo resolution, recording notice visible to the policyholder, reconnect after a network drop.

**Phase 3 - Upgrades** (INV-09, INV-38). Server-enforced end with `call.end()` and no `join-ended-call` (strongly suggested, since it fixes today's reusable links); noise cancellation on the desk, offered, not assumed.

**Phase 4 - Transition and cleanup.** After the last Vonage inspection has ended and its recording is saved, remove `@vonage/video`, the CDN script tags, `VONAGE_*` variables, `sessionId` and `archiveId` handling and the `.OT_` rules; keep historical recordings where they are stored.

**Fleet-transition note.** One surface (S1, inspection meeting) on one platform (mobile and desktop browsers), plus server recording (S2). There are no native apps, so users switch when they load the new pages. Rendezvous strategy: pin the vendor per inspection at creation (the meeting rule). Recommended option: B, per-session routing, scoped by inspection: new inspections on Stream, open Vonage inspections finish on Vonage, because policyholder links already sent stay valid for up to 30 days (`server.js:70`) and recordings in flight still need saving (and, with Vonage-hosted storage, within its 72-hour download window). The Vonage account stays active until then. What still needs coordination: desk tabs open on the old build (the desk reloads on every disconnect, which helps), links already sent, unsaved recordings. Early-prep checklist against the app: server-decided vendor in the join path - missing (`server.js:41-43` returns only the application id); forced-update gate - not needed on web (a reload loads the new build); client capability reporting at login - not applicable (no login); push pre-registration - not applicable (no ringing); version-adoption telemetry - none; store review dates - not applicable.

## 12. Flag for a human

| ID | What | Where | Why it needs a person | Owner |
|---|---|---|---|---|
| FL-01 | Photo capture from the remote video has no Stream mapping in this skill (`no-mapping`, INV-34) | `public/js/adjuster.js:82` | A core claim-evidence feature: decide the capture method and the minimum still resolution, then test it on the phones policyholders use | app-team |
| FL-02 | Whole API and stored files are unauthenticated: claims, inspection ids, stored link tokens, photo and recording URLs are listed by `GET /api/claims`; `/files/photos` and `/files/recordings` are public | `server.js:14-15,45,56-58,79-81` | Recordings and photos of policyholders' homes are reachable by anyone who can reach the server; needs an authentication design for the desk and an invite design for policyholders (part of G3) | app-team |
| FL-03 | Chat messages are inserted as HTML: a message from the other side (or anyone with a link token) can inject markup and script into the adjuster's desk | `public/js/adjuster.js:151`, `public/js/join.js:47` | Security defect outside G1-G13; render with `.text()` and carry the fix into the port | app-team |
| FL-04 | Recordings depend on Vonage-hosted storage and are copied only when someone clicks Save; Vonage's archiving guide states archives without a storage target can be downloaded for 72 hours | `README.md:24-27`, `server.js:152-171` | Unsaved inspection recordings may be lost; decide automatic saving or a storage target now, and the Stream storage choice for the port | app-team |
| FL-05 | The policyholder's only recording notice is Vonage's default "Archiving on" light on their own tile; recording starts when the desk connects | `public/js/join.js:8-13`, `public/js/adjuster.js:133` | Consent and notice are a legal and product decision; the port must rebuild a notice on the join page in any case | app-team |
| FL-06 | Recording start policy: at the adjuster's join (today) or for the whole call (`auto-on`) | `server.js:107-133`, `public/js/adjuster.js:133` | Product decision; also decide what happens if a recording stops mid-inspection (BC-08) | app-team |
| FL-07 | Chat is not kept in the claim file; on Stream, custom events are not stored either | `public/js/adjuster.js:88-96`, `public/js/join.js:81-89` | Decide whether chat belongs in the claim file (the app's own store, or a chat product such as Stream Chat); keep call membership to the two parties, since custom events reach every member watching the call | app-team |
| FL-08 | Role mapping: every token is `moderator` today | `server.js:35` | Decide adjuster and policyholder capabilities (section 10) | app-team |
| FL-09 | No user identity: Stream needs a user id per adjuster and per policyholder | `server.js:36,71,87` | Decide the user model (staff accounts, per-inspection policyholder users) and the link-to-token exchange | app-team |
| FL-10 | Vonage dashboard configuration the code assumes: routed sessions and archiving enabled, Vonage-hosted storage, no callback URL | `README.md:24-26` | Not visible in code; confirm before the transition | app-team |

No text in the repository tried to steer this audit.

## 13. Questions the code cannot answer

- How many inspections run per day or month, how long they last, and how many adjusters work at once?
- What is the Vonage contract end date?
- Is the desk deployed behind an SSO or VPN proxy (`server.js:10` trusts one)? Which paths does it expose publicly for policyholders?
- Is a storage target or an archive callback configured in the Vonage dashboard? Where do past recordings live, and must any move?
- What retention and consent rules apply to inspection recordings, photos and chat?
- How are links sent to policyholders (SMS, email), and how long must they stay valid?
- What phones and browsers do policyholders use? What still-photo resolution do adjusters need?
- Will the client stay plain JavaScript, or move to React (which would allow Stream's prebuilt components)?
- Do customers' or adjusters' networks need firewall or allow-list configuration?

## 14. Prioritised fix list

- **Blocker - G3:** authenticate the desk API, token route and files; decide roles on the server; give policyholders an invite route scoped to one inspection (`server.js:45,82-89`).
- **Blocker - G4:** show join and publish failures in the page with Retry and Leave; re-enable Join (`public/js/adjuster.js:63,161-162`, `public/js/join.js:16-17,54`).
- **High - G2:** mint policyholder tokens on demand with an expiry of hours; stop storing and returning tokens (`server.js:70-74`).
- **High - G8:** policyholder `role: 'publisher'`; authenticate end, recording and photo routes (`server.js:35,91,107,139,152`).
- **High - G5:** handle publisher errors and `accessDenied`; keep the old camera until the new one publishes (`public/js/adjuster.js:159`, `public/js/join.js:13-15,63-70`).
- **High - G6:** add Leave on the phone; `session.disconnect()` and `publisher.destroy()` on leave and `pagehide` (`public/js/join.js`, `public/js/adjuster.js`).
- **High - G10:** handle `accessDialogOpened` / `accessDenied` and explain recovery (`public/js/join.js:55`, `public/js/adjuster.js:157`).
- **Medium - G7:** show reconnecting state; replace the unconditional reload; restart a stopped archive (`public/js/adjuster.js:153-156`, `public/js/join.js:49-52`, `server.js:110`).
- **Medium - G13:** drive mic and camera labels from Publisher `muteForced` / `mediaStopped` (`public/js/adjuster.js:69-78`, `public/js/join.js:71-80`).
- **Low - G11:** set published resolutions deliberately, with photo quality in mind (`public/js/join.js:7-12`, `public/js/adjuster.js:157`).

Section 12 also lists FL-02 (unauthenticated recordings and photos), FL-03 (chat HTML injection) and FL-04 (recordings at risk of expiring unsaved): security and data-retention items outside G1-G13 that deserve the same priority as the Blockers.

## 15. What this skill cannot do

- It read the code and nothing else: no Vonage dashboard, no account settings, no production traffic, no runtime behaviour. Storage targets, callbacks and any proxy in front of the server are unknown.
- It changed nothing and proposes diffs only when asked.
- It names what drives cost but quotes no prices and compares none.
- Mapping rows marked `verify against your SDK version` (plain JS device list and autoplay detection, recording mode and quality setting names, node-sdk `stopRecording`, loading the client without a build step) were not confirmed for the versions this app would pin.
- The photo capture (INV-34) has no mapping in this skill.
- Every platform found (web client, Node server) has a reference in this version; there is no iOS, Android, React Native or Flutter code.

## 16. How to continue

To port the code, give this report to your coding agent together with Stream's Video docs (`https://getstream.io/video/docs/`). The app is plain JavaScript, not React, so no platform skill from Stream's agent-skills pack applies as it stands. The JSON block below is written as the porting input. For commercial or roadmap questions, contact Stream at `https://getstream.io/contact/`.

## 17. Machine-readable block

```json
{
  "schema": "video-integration-audit/v1",
  "skill_version": "0.2",
  "date": "2026-10-28",
  "mode": "migration",
  "detected": [
    { "platform": "server", "vendor": "vonage", "product": "Video API Node server SDK", "vintage": "unified", "packages": { "@vonage/video": "1.31.1" }, "evidence": ["package.json:13", "lib/vonage.js:5"] },
    { "platform": "web", "vendor": "vonage", "product": "Video API web client", "vintage": "unified CDN", "packages": { "@vonage/client-sdk-video": "2.35.3" }, "evidence": ["public/adjuster.html:44", "public/join.html:27"] }
  ],
  "surfaces": [
    { "id": "S1", "kind": "meeting", "platforms": ["web"], "entry": "public/js/adjuster.js:62" },
    { "id": "S2", "kind": "recording", "platforms": ["web", "server"], "entry": "server.js:107" }
  ],
  "app": {
    "client": { "framework": "plain javascript (jQuery 3.7.1)", "router": "none (multi-page static HTML)", "build": "none", "client_owner": "public/js/adjuster.js:1" },
    "server": { "language": "javascript", "framework": "express", "entry": "server.js:1" }
  },
  "server_routes": [
    { "id": "R1", "purpose": "token", "method": "POST", "path": "/api/claims/:claimId/inspections", "location": "server.js:60", "request": [], "response": ["applicationId", "inspection"], "auth": "none", "vendor_calls": ["createSession", "generateClientToken"] },
    { "id": "R2", "purpose": "token", "method": "GET", "path": "/api/claims/:claimId/inspections/:id", "location": "server.js:79", "request": [], "response": ["applicationId", "inspection"], "auth": "none", "vendor_calls": [] },
    { "id": "R3", "purpose": "token", "method": "GET", "path": "/api/claims/:claimId/inspections/:id/token", "location": "server.js:82", "request": ["role"], "response": ["applicationId", "inspection", "token"], "auth": "none", "vendor_calls": ["generateClientToken"] },
    { "id": "R4", "purpose": "other", "method": "GET", "path": "/api/claims", "location": "server.js:45", "request": [], "response": ["claims"], "auth": "none", "vendor_calls": [] },
    { "id": "R5", "purpose": "recording", "method": "POST", "path": "/api/inspections/:id/recording/start", "location": "server.js:107", "request": [], "response": ["archive"], "auth": "none", "vendor_calls": ["startArchive", "getArchive"] },
    { "id": "R6", "purpose": "recording", "method": "GET", "path": "/api/inspections/:id/recording", "location": "server.js:134", "request": [], "response": ["archive"], "auth": "none", "vendor_calls": ["getArchive"] },
    { "id": "R7", "purpose": "recording", "method": "POST", "path": "/api/inspections/:id/end", "location": "server.js:139", "request": [], "response": ["inspection"], "auth": "none", "vendor_calls": ["getArchive", "stopArchive"] },
    { "id": "R8", "purpose": "recording", "method": "POST", "path": "/api/inspections/:id/recording/save", "location": "server.js:152", "request": [], "response": ["recording"], "auth": "none", "vendor_calls": ["getArchive", "archive.url download"] },
    { "id": "R9", "purpose": "other", "method": "POST", "path": "/api/inspections/:id/photos", "location": "server.js:91", "request": ["image"], "response": ["photo"], "auth": "none", "vendor_calls": [] }
  ],
  "identity": { "user_id_source": "none: no user accounts; a role label in token data", "user_id_type": "role label inside Vonage token data", "display_name_source": "hard-coded publisher names Adjuster and Policyholder", "client_chosen": true, "evidence": ["server.js:36", "server.js:87", "public/js/adjuster.js:64", "public/js/adjuster.js:158", "public/js/join.js:9"] },
  "roles": [
    { "vendor_role": "moderator (adjuster)", "source": "server.js:35", "stream_role": "adjuster call-member role (for example host)", "capabilities": ["send-audio", "send-video", "end-call"], "decided_on": "client" },
    { "vendor_role": "moderator (policyholder)", "source": "server.js:71", "stream_role": "policyholder call-member role", "capabilities": ["send-audio", "send-video"], "decided_on": "server" },
    { "vendor_role": "data.role label", "source": "server.js:87", "stream_role": "call-member custom or member role", "capabilities": [], "decided_on": "client" }
  ],
  "persisted_vendor_ids": [
    { "what": "Vonage session id per inspection", "store": "data/claims.json inspections[].sessionId", "location": "server.js:67" },
    { "what": "archive id per inspection", "store": "data/claims.json inspections[].archiveId", "location": "server.js:122" },
    { "what": "policyholder token inside the join link", "store": "data/claims.json inspections[].joinUrl", "location": "server.js:74" },
    { "what": "archive id per saved recording", "store": "data/claims.json recordings[].archiveId", "location": "server.js:165" },
    { "what": "pending archive starts", "store": "in-memory starts map", "location": "server.js:9" }
  ],
  "artifacts": [
    { "kind": "recording", "output": "composite 1280x720 MP4", "storage": "Vonage-hosted, copied on click to data/recordings, served unauthenticated at /files/recordings", "location": "server.js:152" },
    { "kind": "photo", "output": "PNG still from subscriber.getImgData()", "storage": "data/photos, served unauthenticated at /files/photos", "location": "server.js:91" }
  ],
  "ui_components": [
    { "what": "remote video tile with Vonage default subscriber UI", "location": "public/js/adjuster.js:139", "stream_component": "none (plain JS): call.bindVideoElement + call.bindAudioElement", "action": "keep-custom" },
    { "what": "remote video tile on the policyholder page", "location": "public/js/join.js:39", "stream_component": "none (plain JS): call.bindVideoElement + call.bindAudioElement", "action": "keep-custom" },
    { "what": "local picture-in-picture tile", "location": "public/css/app.css:18", "stream_component": "none (plain JS): call.bindVideoElement", "action": "keep-custom" },
    { "what": "Vonage archiving light on the policyholder's own tile (only recording notice)", "location": "public/js/join.js:13", "stream_component": "none (plain JS): indicator from call.state.recording$", "action": "keep-custom" },
    { "what": "blocked-audio control from the default subscriber UI", "location": "public/js/join.js:39", "stream_component": "none (plain JS): button calling call.resumeAudio()", "action": "keep-custom" },
    { "what": "call controls (mic, camera, photo, end, switch camera)", "location": "public/adjuster.html:24", "stream_component": "none (plain JS)", "action": "keep-custom" },
    { "what": "REC indicator and recording status", "location": "public/adjuster.html:31", "stream_component": "none (plain JS): call.state.recording$", "action": "keep-custom" },
    { "what": "chat panel", "location": "public/adjuster.html:36", "stream_component": "none", "action": "keep-custom" }
  ],
  "env": ["VONAGE_APPLICATION_ID", "VONAGE_PRIVATE_KEY", "PUBLIC_BASE_URL"],
  "tests": [],
  "inventory": [
    { "id": "INV-01", "location": "lib/vonage.js:5", "vendor": "vonage", "construct": "new Video({ applicationId, privateKey })", "decision_area": "tokens-identity", "stream_equivalent": "new StreamClient(apiKey, apiSecret)", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "rename", "verify": false, "surface": "S1", "notes": "read the API secret from the server environment" },
    { "id": "INV-02", "location": "server.js:33", "vendor": "vonage", "construct": "generateClientToken(sessionId, { role: 'moderator', data, expireTime })", "decision_area": "tokens-identity", "stream_equivalent": "client.generateUserToken({ user_id, validity_in_seconds }); data to call-member custom via call.updateCallMembers", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "per-session to per-user token; app has no user ids (FL-09)" },
    { "id": "INV-03", "location": "server.js:70", "vendor": "vonage", "construct": "30-day policyholder token in the join URL, stored in joinUrl", "decision_area": "tokens-identity", "stream_equivalent": "generateCallToken({ user_id, call_cids, role }) minted when the link is opened", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "link carries an invite id, not a token; stop storing tokens" },
    { "id": "INV-04", "location": "server.js:82", "vendor": "vonage", "construct": "token route with role from req.query.role", "decision_area": "tokens-identity", "stream_equivalent": "token route returning { vendor, apiKey, token, callId }; client tokenProvider", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "role decided on the server" },
    { "id": "INV-05", "location": "server.js:41", "vendor": "vonage", "construct": "connectionInfo returns VONAGE_APPLICATION_ID", "decision_area": "tokens-identity", "stream_equivalent": "return the Stream API key", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "rename", "verify": false, "surface": "S1", "notes": "" },
    { "id": "INV-06", "location": "server.js:62", "vendor": "vonage", "construct": "createSession({ mediaMode: ROUTED, archiveMode: MANUAL }); sessionId stored", "decision_area": "room-call-type", "stream_equivalent": "client.video.call(type, id) + call.getOrCreate({ data: { members } }), call id = inspection id", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "sessionId storage goes; inspection record stays" },
    { "id": "INV-07", "location": "server.js:64", "vendor": "vonage", "construct": "archiveMode: ArchiveMode.MANUAL", "decision_area": "room-call-type", "stream_equivalent": "call-type recording mode: app-started or auto-on", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "config", "verify": true, "surface": "S2", "notes": "mode names verify against your SDK version" },
    { "id": "INV-08", "location": "server.js:35", "vendor": "vonage", "construct": "token role moderator for every caller", "decision_area": "permissions", "stream_equivalent": "call-member roles and call-type grid (adjuster: send-audio, send-video, end-call; policyholder: send-audio, send-video)", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "config", "verify": false, "surface": "S1", "notes": "policyholder can force-disconnect and force-mute today" },
    { "id": "INV-09", "location": "public/js/adjuster.js:97", "vendor": "vonage", "construct": "end for everyone via session.forceDisconnect loop + server endedAt (server.js:139) + client ended check (public/js/join.js:36)", "decision_area": "call-lifecycle", "stream_equivalent": "call.end() (node-sdk) or call.endCall(); remove join-ended-call from the call type", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "upgrade", "verify": false, "surface": "S1", "notes": "platform-enforced end; session.connection.id at adjuster.js:101 vs connectionId elsewhere (verify)" },
    { "id": "INV-10", "location": "public/js/adjuster.js:124", "vendor": "vonage", "construct": "connectionCreated / connectionDestroyed connection map", "decision_area": "call-lifecycle", "stream_equivalent": "none needed: only feeds INV-09", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "delete", "verify": false, "surface": "S1", "notes": "identity-keying" },
    { "id": "INV-11", "location": "public/js/adjuster.js:130", "vendor": "vonage", "construct": "sessionConnected", "decision_area": "events-reconnect", "stream_equivalent": "call.join() resolves; calling state JOINED", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "rename", "verify": false, "surface": "S1", "notes": "recording start moves (INV-41)" },
    { "id": "INV-12", "location": "public/js/adjuster.js:153", "vendor": "vonage", "construct": "sessionDisconnected -> reload", "decision_area": "events-reconnect", "stream_equivalent": "call.state.callingState$ LEFT / OFFLINE / RECONNECTING_FAILED; show RECONNECTING", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "G7" },
    { "id": "INV-13", "location": "public/js/join.js:49", "vendor": "vonage", "construct": "sessionDisconnected -> reload", "decision_area": "events-reconnect", "stream_equivalent": "call.state.callingState$ LEFT / OFFLINE / RECONNECTING_FAILED; show RECONNECTING", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "G7" },
    { "id": "INV-14", "location": "public/js/adjuster.js:92", "vendor": "vonage", "construct": "session.signal({ type: 'chat', data })", "decision_area": "events-reconnect", "stream_equivalent": "call.sendCustomEvent({ type, ... })", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "behavioral-diff", "verify": false, "surface": "S1", "notes": "5 KB limit; no reconnect buffering; render own message locally" },
    { "id": "INV-15", "location": "public/js/join.js:85", "vendor": "vonage", "construct": "session.signal({ type: 'chat', data })", "decision_area": "events-reconnect", "stream_equivalent": "call.sendCustomEvent({ type, ... })", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "behavioral-diff", "verify": false, "surface": "S1", "notes": "as INV-14" },
    { "id": "INV-16", "location": "public/js/adjuster.js:149", "vendor": "vonage", "construct": "session.on('signal:chat'), sender from event.from.connectionId", "decision_area": "events-reconnect", "stream_equivalent": "call.on('custom', ...) filtered on payload type; sender from event.user.id", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "event.data inserted as HTML (FL-03)" },
    { "id": "INV-17", "location": "public/js/join.js:45", "vendor": "vonage", "construct": "session.on('signal:chat'), sender from event.from.connectionId", "decision_area": "events-reconnect", "stream_equivalent": "call.on('custom', ...) filtered on payload type; sender from event.user.id", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "event.data inserted as HTML (FL-03)" },
    { "id": "INV-18", "location": "public/adjuster.html:44", "vendor": "vonage", "construct": "script tag @vonage/client-sdk-video@2.35.3 opentok.min.js", "decision_area": "track-device-lifecycle", "stream_equivalent": "@stream-io/video-client via bundling or a module CDN import", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": true, "surface": "S1", "notes": "app has no build step" },
    { "id": "INV-19", "location": "public/join.html:27", "vendor": "vonage", "construct": "script tag @vonage/client-sdk-video@2.35.3 opentok.min.js", "decision_area": "track-device-lifecycle", "stream_equivalent": "@stream-io/video-client via bundling or a module CDN import", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": true, "surface": "S1", "notes": "app has no build step" },
    { "id": "INV-20", "location": "public/js/adjuster.js:123", "vendor": "vonage", "construct": "OT.initSession(applicationId, sessionId)", "decision_area": "track-device-lifecycle", "stream_equivalent": "new StreamVideoClient({ apiKey, user, tokenProvider }) + client.call(type, id)", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "" },
    { "id": "INV-21", "location": "public/js/join.js:37", "vendor": "vonage", "construct": "OT.initSession(applicationId, sessionId)", "decision_area": "track-device-lifecycle", "stream_equivalent": "new StreamVideoClient({ apiKey, user, tokenProvider }) + client.call(type, id)", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "" },
    { "id": "INV-22", "location": "public/js/adjuster.js:160", "vendor": "vonage", "construct": "session.connect(token, callback)", "decision_area": "track-device-lifecycle", "stream_equivalent": "await call.join() in try/catch", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "G4" },
    { "id": "INV-23", "location": "public/js/join.js:53", "vendor": "vonage", "construct": "session.connect(token, callback)", "decision_area": "track-device-lifecycle", "stream_equivalent": "await call.join() in try/catch", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "G4" },
    { "id": "INV-24", "location": "public/js/adjuster.js:157", "vendor": "vonage", "construct": "OT.initPublisher('local', { name: 'Adjuster' })", "decision_area": "track-device-lifecycle", "stream_equivalent": "call.camera.enable() + call.microphone.enable(); render with call.bindVideoElement", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "publish-on-join; no resolution set (G11)" },
    { "id": "INV-25", "location": "public/js/join.js:13", "vendor": "vonage", "construct": "OT.initPublisher('local', { facingMode: 'environment', videoSource, publishAudio, publishVideo })", "decision_area": "track-device-lifecycle", "stream_equivalent": "call.camera.selectDirection('back', { enableCamera: false }) before join, then call.camera.enable() / call.microphone.enable()", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "" },
    { "id": "INV-26", "location": "public/js/adjuster.js:162", "vendor": "vonage", "construct": "session.publish(publisher, callback)", "decision_area": "track-device-lifecycle", "stream_equivalent": "implicit once devices are enabled and the call is joined", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "delete", "verify": false, "surface": "S1", "notes": "" },
    { "id": "INV-27", "location": "public/js/join.js:16", "vendor": "vonage", "construct": "session.publish(publisher, callback)", "decision_area": "track-device-lifecycle", "stream_equivalent": "implicit once devices are enabled and the call is joined", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "delete", "verify": false, "surface": "S1", "notes": "" },
    { "id": "INV-28", "location": "public/js/join.js:63", "vendor": "vonage", "construct": "camera switch via session.unpublish + publisher.destroy + new publisher with videoSource", "decision_area": "track-device-lifecycle", "stream_equivalent": "call.camera.flip() or call.camera.select(deviceId)", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "today the stream disappears during the switch" },
    { "id": "INV-29", "location": "public/js/join.js:19", "vendor": "vonage", "construct": "OT.getDevices() + publisher.getVideoSource()", "decision_area": "track-device-lifecycle", "stream_equivalent": "camera device list from the camera manager state", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": true, "surface": "S1", "notes": "not needed if the button calls call.camera.flip()" },
    { "id": "INV-30", "location": "public/js/adjuster.js:69", "vendor": "vonage", "construct": "publisher.publishAudio(bool) / publishVideo(bool)", "decision_area": "track-device-lifecycle", "stream_equivalent": "call.microphone.toggle() / call.camera.toggle()", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "rename", "verify": false, "surface": "S1", "notes": "labels from device state (G13)" },
    { "id": "INV-31", "location": "public/js/join.js:71", "vendor": "vonage", "construct": "publisher.publishAudio(bool) / publishVideo(bool)", "decision_area": "track-device-lifecycle", "stream_equivalent": "call.microphone.toggle() / call.camera.toggle()", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "rename", "verify": false, "surface": "S1", "notes": "labels from device state (G13)" },
    { "id": "INV-32", "location": "public/js/adjuster.js:138", "vendor": "vonage", "construct": "streamCreated -> session.subscribe; streamDestroyed", "decision_area": "track-device-lifecycle", "stream_equivalent": "automatic; render call.state.participants$ with call.bindVideoElement + call.bindAudioElement", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "delete", "verify": false, "surface": "S1", "notes": "subscribe-loop; photo button keyed on stream" },
    { "id": "INV-33", "location": "public/js/join.js:38", "vendor": "vonage", "construct": "streamCreated -> session.subscribe; streamDestroyed", "decision_area": "track-device-lifecycle", "stream_equivalent": "automatic; render call.state.participants$ with call.bindVideoElement + call.bindAudioElement", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "delete", "verify": false, "surface": "S1", "notes": "subscribe-loop" },
    { "id": "INV-34", "location": "public/js/adjuster.js:82", "vendor": "vonage", "construct": "subscriber.getImgData()", "decision_area": "track-device-lifecycle", "stream_equivalent": "no mapping in this skill", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "no-mapping", "verify": false, "surface": "S1", "notes": "FL-01; capture from the app-owned video element with browser code (verify)" },
    { "id": "INV-35", "location": "public/js/adjuster.js:139", "vendor": "vonage", "construct": "Vonage default tile UI on the desk", "decision_area": "track-device-lifecycle", "stream_equivalent": "own tiles with call.bindVideoElement / call.bindAudioElement, indicator from call.state.recording$, blocked-audio button with call.resumeAudio()", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "" },
    { "id": "INV-36", "location": "public/js/join.js:8", "vendor": "vonage", "construct": "Vonage default tile UI on the policyholder page, including the archiving light", "decision_area": "track-device-lifecycle", "stream_equivalent": "own tiles with call.bindVideoElement / call.bindAudioElement, indicator from call.state.recording$, blocked-audio button with call.resumeAudio()", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S1", "notes": "only recording notice for the policyholder (FL-05)" },
    { "id": "INV-37", "location": "public/css/app.css:31", "vendor": "vonage", "construct": "CSS rules on .OT_root .OT_name, .OT_name_bar, .OT_mute", "decision_area": "track-device-lifecycle", "stream_equivalent": "none: rules on the app's own tile elements", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "delete", "verify": false, "surface": "S1", "notes": "keep the choice: no name bar, no per-tile mute" },
    { "id": "INV-38", "location": "public/js/adjuster.js:157", "vendor": "vonage", "construct": "OT.initPublisher with no noise filter", "decision_area": "track-device-lifecycle", "stream_equivalent": "@stream-io/audio-filters-web NoiseCancellation with call.microphone.enableNoiseCancellation(...)", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "upgrade", "verify": false, "surface": "S1", "notes": "desktop browsers only; offered, not assumed" },
    { "id": "INV-39", "location": "server.js:114", "vendor": "vonage", "construct": "startArchive(sessionId, { name, outputMode: COMPOSED, hasAudio, hasVideo })", "decision_area": "recording-livestream", "stream_equivalent": "call.startRecording({ recording_type: 'composite' })", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "shape-shift", "verify": false, "surface": "S2", "notes": "keep the archive name in the app's own record" },
    { "id": "INV-40", "location": "server.js:117", "vendor": "vonage", "construct": "archive resolution '1280x720', layout bestFit", "decision_area": "recording-livestream", "stream_equivalent": "settings.recording.layout grid; recording quality 720p", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "config", "verify": true, "surface": "S2", "notes": "quality setting name verify against your SDK version" },
    { "id": "INV-41", "location": "server.js:107", "vendor": "vonage", "construct": "start-on-desk-connect orchestration: starts map, archiveId reuse (public/js/adjuster.js:133)", "decision_area": "recording-livestream", "stream_equivalent": "call-type recording mode auto-on, or a single server start (INV-39)", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "config", "verify": true, "surface": "S2", "notes": "FL-06; stopped archive never restarted today" },
    { "id": "INV-42", "location": "server.js:134", "vendor": "vonage", "construct": "getArchive(archiveId) status route", "decision_area": "recording-livestream", "stream_equivalent": "call.recording_ready (WebSocket and webhook); call.listRecordings()", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "shape-shift", "verify": false, "surface": "S2", "notes": "" },
    { "id": "INV-43", "location": "public/js/adjuster.js:15", "vendor": "vonage", "construct": "client polls archive status every 5 s", "decision_area": "recording-livestream", "stream_equivalent": "call.state.recording$; call.recording_ready", "stream_sdk": "@stream-io/video-client@1.63.0", "class": "shape-shift", "verify": false, "surface": "S2", "notes": "" },
    { "id": "INV-44", "location": "server.js:147", "vendor": "vonage", "construct": "stopArchive(archiveId)", "decision_area": "recording-livestream", "stream_equivalent": "call.stopRecording(); call.end() (INV-09)", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "rename", "verify": true, "surface": "S2", "notes": "node-sdk form verify against your SDK version" },
    { "id": "INV-45", "location": "server.js:152", "vendor": "vonage", "construct": "download archive.url when available, write MP4 to data/recordings", "decision_area": "recording-livestream", "stream_equivalent": "recording URL from call.listRecordings() or the call.recording_ready payload; keep the app's own download", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "shape-shift", "verify": false, "surface": "S2", "notes": "" },
    { "id": "INV-46", "location": "README.md:24", "vendor": "vonage", "construct": "Vonage-hosted archive storage (dashboard)", "decision_area": "recording-livestream", "stream_equivalent": "Stream-managed storage, or Amazon S3, Google Cloud Storage, Azure Blob", "stream_sdk": "@stream-io/node-sdk@0.8.14", "class": "config", "verify": false, "surface": "S2", "notes": "72-hour Vonage download window without a storage target (FL-04)" }
  ],
  "behaviour_changes": [
    { "id": "BC-01", "trap": "subscribe-loop", "rows": ["INV-32", "INV-33"], "action": "delete both loops; bind elements per participant; rebuild photo capture on the app-owned video element" },
    { "id": "BC-02", "trap": "streams-vs-participants", "rows": ["INV-32"], "action": "enable Take photo when the policyholder participant has a published video track" },
    { "id": "BC-03", "trap": "publish-on-join", "rows": ["INV-24", "INV-25", "INV-26", "INV-27"], "action": "set camera direction and device state before call.join()" },
    { "id": "BC-04", "trap": "targeted-signals", "rows": ["INV-14", "INV-15", "INV-16", "INV-17"], "action": "enforce 5 KB; nothing critical across reconnects; render own messages locally and filter echoes on event.user.id" },
    { "id": "BC-05", "trap": "identity-keying", "rows": ["INV-09", "INV-10", "INV-16", "INV-17"], "action": "key on Stream user ids instead of connection ids" },
    { "id": "BC-06", "trap": "session-immortality", "rows": ["INV-06", "INV-09"], "action": "fresh call per inspection; call.end() on end; remove join-ended-call" },
    { "id": "BC-07", "trap": "server-token-per-session", "rows": ["INV-02", "INV-03", "INV-04"], "action": "per-user tokens; call-scoped token for policyholders; access from membership and call type" },
    { "id": "BC-08", "trap": "archive-auto-timers", "rows": ["INV-39", "INV-41"], "action": "re-baseline recording continuity; decide behaviour when a recording stops mid-inspection" }
  ],
  "findings": [
    { "id": "G1", "set": "general", "verdict": "PASS", "severity": "Blocker", "evidence": ["lib/vonage.js:7", ".gitignore:3"], "fix": "keep the private key server-side" },
    { "id": "G2", "set": "general", "verdict": "FAIL", "severity": "High", "evidence": ["server.js:70", "server.js:74", "server.js:45", "server.js:79"], "fix": "mint policyholder tokens on demand with an expiry of hours; stop storing and returning tokens" },
    { "id": "G3", "set": "general", "verdict": "FAIL", "severity": "Blocker", "evidence": ["server.js:45", "server.js:82", "server.js:87", "server.js:35"], "fix": "authenticate desk API, token route and files; decide role on the server; invite route for policyholders" },
    { "id": "G4", "set": "general", "verdict": "FAIL", "severity": "Blocker", "evidence": ["public/js/adjuster.js:161", "public/js/adjuster.js:63", "public/js/join.js:54", "public/js/join.js:16"], "fix": "show join and publish failures with Retry and Leave; re-enable Join" },
    { "id": "G5", "set": "general", "verdict": "FAIL", "severity": "High", "evidence": ["public/js/adjuster.js:159", "public/js/join.js:13", "public/js/join.js:65"], "fix": "handle publisher errors and accessDenied; keep the old camera until the new one publishes" },
    { "id": "G6", "set": "general", "verdict": "FAIL", "severity": "High", "evidence": ["public/js/join.js:68", "public/adjuster.html:10"], "fix": "add Leave; session.disconnect() and publisher.destroy() on leave and pagehide" },
    { "id": "G7", "set": "general", "verdict": "FAIL", "severity": "Medium", "evidence": ["public/js/adjuster.js:153", "public/js/join.js:49", "server.js:110"], "fix": "show reconnecting state; replace unconditional reload; restart a stopped archive" },
    { "id": "G8", "set": "general", "verdict": "FAIL", "severity": "High", "evidence": ["server.js:35", "server.js:91", "server.js:107", "server.js:139", "server.js:152"], "fix": "policyholder role publisher; authenticate end, recording and photo routes" },
    { "id": "G9", "set": "general", "verdict": "PASS", "severity": "High", "evidence": ["public/js/adjuster.js:139", "public/js/join.js:39", "public/css/app.css:31"], "fix": "keep default UI; rebuild the blocked-audio control with call.resumeAudio() on a plain JS port" },
    { "id": "G10", "set": "general", "verdict": "FAIL", "severity": "High", "evidence": ["public/js/join.js:55", "public/js/adjuster.js:157"], "fix": "handle accessDialogOpened, accessAllowed, accessDenied and explain recovery" },
    { "id": "G11", "set": "general", "verdict": "FAIL", "severity": "Low", "evidence": ["public/js/adjuster.js:157", "public/js/join.js:7", "server.js:117"], "fix": "set published resolutions deliberately, with photo quality in mind" },
    { "id": "G12", "set": "general", "verdict": "N/A", "severity": "High", "evidence": ["server.js:134"], "fix": "none now; verify any future recording webhook" },
    { "id": "G13", "set": "general", "verdict": "FAIL", "severity": "Medium", "evidence": ["public/js/adjuster.js:69", "public/js/join.js:71", "server.js:35"], "fix": "update labels on Publisher muteForced and mediaStopped" }
  ],
  "call_type_config": [
    { "setting": "call_type", "source": "server.js:62", "value": "custom inspection type, 1:1", "stream_setting": "custom call type" },
    { "setting": "target_resolution", "source": "public/js/join.js:7", "value": "not set in code; choose for photo evidence and the 470px desk stage", "stream_setting": "settings.video.target_resolution" },
    { "setting": "camera_default_on", "source": "public/js/join.js:9", "value": "true", "stream_setting": "settings.video.camera_default_on" },
    { "setting": "mic_default_on", "source": "public/js/adjuster.js:157", "value": "true", "stream_setting": "settings.audio.mic_default_on" },
    { "setting": "recording_mode", "source": "server.js:64", "value": "app-started at adjuster join, or auto-on", "stream_setting": "call-type recording mode (verify against your SDK version)" },
    { "setting": "recording_layout", "source": "server.js:118", "value": "grid", "stream_setting": "settings.recording.layout" },
    { "setting": "recording_quality", "source": "server.js:117", "value": "720p", "stream_setting": "recording quality (verify against your SDK version)" },
    { "setting": "recording_audio_only", "source": "server.js:119", "value": "false", "stream_setting": "settings.recording.audio_only" },
    { "setting": "recording_storage", "source": "README.md:24", "value": "decide with FL-04", "stream_setting": "Stream-managed, Amazon S3, Google Cloud Storage or Azure Blob" },
    { "setting": "join_ended_call", "source": "server.js:141", "value": "revoked for both roles", "stream_setting": "role grid: remove join-ended-call" }
  ],
  "flags": [
    { "id": "FL-01", "what": "photo capture via subscriber.getImgData() has no mapping", "location": "public/js/adjuster.js:82", "owner": "app-team" },
    { "id": "FL-02", "what": "unauthenticated API and public recording and photo files", "location": "server.js:45", "owner": "app-team" },
    { "id": "FL-03", "what": "chat messages inserted as HTML (script injection into the desk)", "location": "public/js/adjuster.js:151", "owner": "app-team" },
    { "id": "FL-04", "what": "recordings in Vonage-hosted storage copied only on click; 72-hour download window", "location": "server.js:152", "owner": "app-team" },
    { "id": "FL-05", "what": "policyholder's only recording notice is Vonage's default archiving light", "location": "public/js/join.js:13", "owner": "app-team" },
    { "id": "FL-06", "what": "recording start policy and behaviour when a recording stops mid-inspection", "location": "server.js:107", "owner": "app-team" },
    { "id": "FL-07", "what": "chat not kept in the claim file; decide retention and keep call membership to two parties", "location": "public/js/adjuster.js:88", "owner": "app-team" },
    { "id": "FL-08", "what": "role mapping: every token is moderator today", "location": "server.js:35", "owner": "app-team" },
    { "id": "FL-09", "what": "no user identity; Stream needs user ids for adjusters and policyholders", "location": "server.js:87", "owner": "app-team" },
    { "id": "FL-10", "what": "Vonage dashboard configuration assumed by the code", "location": "README.md:24", "owner": "app-team" }
  ],
  "questions": [
    "How many inspections run per day or month, how long are they, and how many adjusters work at once?",
    "What is the Vonage contract end date?",
    "Is the desk behind an SSO or VPN proxy, and which paths are public for policyholders?",
    "Is a storage target or archive callback configured in the Vonage dashboard, where do past recordings live, and must any move?",
    "What retention and consent rules apply to recordings, photos and chat?",
    "How are links sent to policyholders and how long must they stay valid?",
    "Which phones and browsers do policyholders use, and what still-photo resolution do adjusters need?",
    "Will the client stay plain JavaScript or move to React?",
    "Do customers' or adjusters' networks need firewall or allow-list configuration?"
  ],
  "effort": [
    { "decision_area": "tokens-identity", "rows": 5, "delete": 0, "upgrade": 0, "gap": 0, "no_mapping": 0, "files": 2 },
    { "decision_area": "room-call-type", "rows": 2, "delete": 0, "upgrade": 0, "gap": 0, "no_mapping": 0, "files": 1 },
    { "decision_area": "permissions", "rows": 1, "delete": 0, "upgrade": 0, "gap": 0, "no_mapping": 0, "files": 1 },
    { "decision_area": "call-lifecycle", "rows": 2, "delete": 1, "upgrade": 1, "gap": 0, "no_mapping": 0, "files": 3 },
    { "decision_area": "events-reconnect", "rows": 7, "delete": 0, "upgrade": 0, "gap": 0, "no_mapping": 0, "files": 2 },
    { "decision_area": "track-device-lifecycle", "rows": 21, "delete": 5, "upgrade": 1, "gap": 0, "no_mapping": 1, "files": 5 },
    { "decision_area": "ringing", "rows": 0, "delete": 0, "upgrade": 0, "gap": 0, "no_mapping": 0, "files": 0 },
    { "decision_area": "recording-livestream", "rows": 8, "delete": 0, "upgrade": 0, "gap": 0, "no_mapping": 0, "files": 3 },
    { "decision_area": "fleet-transition", "rows": 0, "delete": 0, "upgrade": 0, "gap": 0, "no_mapping": 0, "files": 0 },
    { "decision_area": "integrations", "rows": 0, "delete": 0, "upgrade": 0, "gap": 0, "no_mapping": 0, "files": 0 }
  ],
  "changes": {
    "gains": [
      { "kind": "code-goes-away", "rows": ["INV-10", "INV-26", "INV-27", "INV-32", "INV-33", "INV-37"], "summary": "subscribe loops, explicit publish calls, connection map for end-for-everyone, OT_ stylesheet rules; 3 files" },
      { "kind": "built-in-feature", "rows": ["INV-09"], "summary": "platform-enforced end with call.end() and no join-ended-call replaces force-disconnect loop and client-side ended check" },
      { "kind": "built-in-feature", "rows": ["INV-38"], "summary": "noise cancellation on the desk (desktop browsers only), offered" }
    ],
    "must_change": [
      { "rows": ["INV-34"], "summary": "no mapping for subscriber.getImgData(): rebuild photo capture on the app-owned video element" },
      { "rows": ["INV-35", "INV-36"], "summary": "rebuild recording indicator and blocked-audio control on both pages; plain JS has no tile chrome" },
      { "rows": ["INV-18", "INV-19"], "summary": "load @stream-io/video-client without the current CDN script tag: bundling or module import" },
      { "rows": ["INV-02", "INV-03", "INV-04"], "summary": "introduce user ids and an invite-to-token exchange for policyholders" },
      { "rows": ["INV-14", "INV-15", "INV-16", "INV-17"], "summary": "chat on custom events: 5 KB limit, no reconnect buffering, render own messages locally" }
    ]
  },
  "cost_drivers": [
    { "feature": "participant minutes", "what": "1:1 inspections, both sides publish camera and microphone at SDK default resolution", "rows": ["INV-24", "INV-25"], "measure": "participant minutes per inspection and the resolution each participant receives", "listed_on_stream_pricing_page": true },
    { "feature": "recording", "what": "composite 720p recording of every inspection from adjuster join to end", "rows": ["INV-39", "INV-40", "INV-41"], "measure": "recorded call minutes at 720p", "listed_on_stream_pricing_page": true },
    { "feature": "noise cancellation", "what": "not used today; would apply to the adjuster desk if the upgrade is adopted", "rows": ["INV-38"], "measure": "adjuster participant minutes with noise cancellation on", "listed_on_stream_pricing_page": true }
  ],
  "usage_volume": { "level": "none", "evidence": [], "summary": "the code gives no evidence of usage volume" }
}
```
