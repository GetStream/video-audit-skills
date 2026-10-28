# Hub: what moving to Stream Video involves, whatever the source

This file holds what every migration shares: how to detect the platform, the nine decision areas a migration plan must cover, the behaviour traps that recur across vendors, the honesty and referral rules, and the Stream facts the plan relies on. Vendor references (`vonage-web.md`, `agora-web.md`) hold only what is specific to that vendor and point here for the rest.

**Pinned to:** `@stream-io/video-react-sdk` 1.44.0, `@stream-io/video-client` 1.62.0, `@stream-io/node-sdk` 0.8.14. Stream facts verified against Stream's public documentation on 2026-10-22. For other SDK versions, mark rows `verify against your SDK version`.

## Detection

Search manifests first, then script tags, then imports. Record every hit with `path:line` and the installed version from the lockfile when there is one.

| Vendor | Web / JS | Server | iOS | Android | Cross-platform | Other signals |
|---|---|---|---|---|---|---|
| **Vonage Video API** (ex-TokBox OpenTok) | `@vonage/client-sdk-video`, `@opentok/client`, `@vonage/video-client`, `@vonage/video-express` (deprecated), `opentok-react` | `@vonage/video`, `@vonage/server-sdk`, `opentok` (Node); `opentok`, `vonage` (Python); `com.opentok.api:opentok-server-sdk`, `com.vonage:server-sdk` (Java); `opentok`, `vonage` (Ruby); `opentok/opentok`, `vonage/client` (PHP) | see `vonage-web.md` Detection | see `vonage-web.md` Detection | `opentok-react-native`, `@vonage/client-sdk-video-react-native` | global `OT.`; `OT.initSession(`; script from `static.opentok.com` or `unpkg.com/@vonage/client-sdk-video` |
| **Agora** | `agora-rtc-sdk-ng` (4.x), `agora-rtc-sdk` (3.x, deprecated), `agora-rtc-react`, `agora-rtm-sdk`, `agora-rtm`, `agora-extension-*`, `agora-chat` | `agora-token`, `agora-access-token` (deprecated) (Node); `agora-token-builder` (Python); AgoraDynamicKey sources (`RtcTokenBuilder2`, `AccessToken2`) in any language; REST calls to `api.agora.io` | see `agora-web.md` Detection | see `agora-web.md` Detection | `react-native-agora`, `agora_rtc_engine` (Flutter) | global `AgoraRTC`; script from `download.agora.io/sdk/release/` |
| **Stream Video** | `@stream-io/video-react-sdk`, `@stream-io/video-client` | `@stream-io/node-sdk` (Node); `getstream` (Python, Go module `github.com/GetStream/getstream-go`); other server SDKs | `StreamVideo`, `StreamVideoSwiftUI`, `StreamVideoUIKit` | `io.getstream:stream-video-android-*` | `@stream-io/video-react-native-sdk`, `stream_video_flutter` | `StreamVideoClient`, `client.call(` |

Recognised but **without a reference in this version** (mode `neither` unless Stream, Vonage or Agora is also present): Twilio Video (`twilio-video`; server `twilio` with `VideoGrant`), Zoom Video SDK (`@zoom/videosdk`) and Zoom Meeting SDK (`@zoom/meetingsdk`, a different product: an embedded Zoom meeting, not an SDK port), 100ms (`@100mslive/*`), Daily (`@daily-co/daily-js`, `@daily-co/daily-react`), LiveKit (`livekit-client`, `livekit-server-sdk`, `@livekit/components-react`), Amazon Chime SDK (`amazon-chime-sdk-js`), Jitsi (`lib-jitsi-meet`, `@jitsi/react-sdk`), and self-built WebRTC (`new RTCPeerConnection(` with the app's own signalling).

Traps:

- `getUserMedia` or a `<video>` element alone is not a video platform (a camera preview, a photo capture). Report "no video platform".
- npm `getstream` and `stream-chat` are Stream Activity Feeds and Stream Chat, not Stream Video. Python `getstream` is Stream's unified server SDK and counts as Stream Video only if the code calls video endpoints (`video.call(`, `call.create(`, `generate_token`) for calls.
- Vonage's unified and legacy web clients share one code base and one global (`OT`); the client code can look identical across vintages. See `vonage-web.md` for vintage rules.

## The nine decision areas

Every inventory row belongs to one area (`decision_area` in the JSON block). Each area says what changes, the failure it prevents, what to look for, and what the plan must decide.

### 1. Tokens and identity (`tokens-identity`)

**What changes.** Both Vonage and Agora mint tokens per room (session or channel) and carry the role in the token. Stream mints a token **per user**, signed with the app's API secret, valid across calls; what the user may do in a given call comes from the call type's role and permission grid and the user's call-member role, not from the token. A call-scoped token (`generateCallToken({ user_id, call_cids, role })`) exists for restricting a token to specific calls.

**Identity.** Stream user ids are strings the backend owns. Vendor ids that are numeric, client-chosen or per-connection (Agora `uid`, Vonage `connectionId`) must be mapped to a stable server-side user id. Per-device presence is Stream's `sessionId` on each participant; one user can have several sessions.

**Failure it prevents.** A port that keeps minting "one token per room" for any requested identity silently widens access: a Stream user token works for every call the user's role allows.

**Look for.** The token route, the identity it uses, the role it grants, its lifetime, and where the client keeps per-user state keyed on the vendor id.

**Plan.** One token route per user session (`generateUserToken({ user_id, validity_in_seconds })`, default lifetime 1 hour), a `tokenProvider` on the client, and a mapping table from vendor ids to user ids wherever stored data or analytics are keyed on them.

### 2. Rooms become calls with a call type (`room-call-type`)

**What changes.** A Vonage session id or an Agora channel name becomes a Stream call id; the call's full identifier is `type:id` (for example `default:standup`). The call **type** carries configuration that the vendor kept in code, token claims or per-request REST options: audio and video defaults, backstage, recording, transcription, ringing timeouts, the role and permission grid. Built-in types are `default`, `audio_room`, `livestream` and `development` (testing only); apps can create their own.

**Failure it prevents.** Everything on `default`, with settings scattered in code: the most common source of later configuration drift.

**Look for.** Session or channel creation options, room naming schemes, per-room flags passed to the server, any room-reuse rule.

**Plan.** One call type per kind of surface (meeting, calling, livestream), the call-type configuration checklist (output contract section 8), and the call-id scheme (Stream call ids are chosen by the app, so appointment or room ids can be reused directly).

### 3. Permissions are additive roles on the server (`permissions`)

**What changes.** Stream decides permissions from the user's role and the call-member role against the call type's grid of capabilities (`send-audio`, `send-video`, `screenshare`, `mute-users`, `kick-user`, `block-users`, `end-call`, `start-record-call`, `join-backstage`, `join-ended-call`, and others). Capabilities can be granted and revoked per user at runtime (`call.grantPermissions(userId, caps)`, `call.revokePermissions(userId, caps)`), and a user can ask for one (`call.requestPermissions({ permissions })`, which hosts see as `call.permission_request`). A user's effective capabilities are the union of what their roles grant.

**Failure it prevents.** Porting "host" as a client-side flag. Or mapping a vendor's audience/publisher role switch to a toggle that does not exist: on Stream, promotion is a permission grant.

**Look for.** Where the app decides who is host, moderator, speaker, audience; every moderation action and how it is enforced.

**Plan.** The role mapping (vendor role -> Stream call-member role), the grid for each call type, and which moderation actions move from client messages to platform calls.

### 4. Call lifecycle: a call is a place, a session is an event (`call-lifecycle`)

**What changes.** A Stream call is a durable server-side entity: it exists before anyone joins, holds members, settings and history, and can host many sessions. A session starts when the first participant joins and ends after the last one leaves (plus an inactivity timeout, 30 seconds by default). `call.endCall()` (client, needs `end-call`) or `call.end()` (Node) ends the call for everyone. Whether an ended call can be rejoined depends on the `join-ended-call` capability, which most built-in types grant by default: revoke it where an ended call must stay ended.

**Failure it prevents.** Two opposite ports. A source room that "never ends" (Vonage sessions do not expire) ported to Stream with `endCall` as the moderator's "close room" changes re-entry for recurring rooms. A source room that dies when empty ported as a reusable call id brings back stale state (old members, old settings, an ended call that still accepts joins).

**Look for.** Room reuse, "end meeting for all" actions, recurring-room products, cleanup jobs, and 1:1 calling flows that end when one side hangs up.

**Plan.** Per surface: a reusable call (meetings, recurring rooms) or a fresh call id per conversation (1:1 calls, ringing: ringing works once per call id). For 1:1 calls, end the call when either side hangs up and revoke `join-ended-call` on that call type, so a hung-up call cannot be rejoined.

### 5. Events and reconnects (`events-reconnect`)

**What changes.** Vendor SDKs push imperative events (user joined, stream created, track published) that apps turn into UI state by hand. Stream exposes reactive state (`call.state.participants$`, React `useCallStateHooks()`), and keeps discrete events for things that are not state (`custom`, `call.reaction_new`, `call.recording_ready`, `call.ended`, `call.session_participant_joined`). Stream reconnects on its own; the calling state shows `RECONNECTING`, `OFFLINE`, `RECONNECTING_FAILED`; `call.setDisconnectionTimeout(seconds)` bounds how long it keeps trying (default 0: indefinitely).

**Failure it prevents.** Hand-rolled participant lists that drift from the SDK's state; UI that treats a reconnect as a leave.

**Look for.** Every event handler, what state it mutates, and what business logic it carries beyond UI (analytics, billing, custom signalling).

**Plan.** Delete handlers that only mirror state; move business logic to state subscriptions or to server webhooks; keep custom signalling on `call.sendCustomEvent` (payload up to 5 KB, delivered to call members who watch the call, no per-recipient targeting) or Stream Chat. Messages meant for one recipient only follow the `targeted-signals` trap below: they never move to custom events.

### 6. Tracks, devices and publishing (`track-device-lifecycle`)

**What changes.** Vendors make the app create local tracks or publishers, publish them, subscribe to each remote stream and attach media to DOM elements. On Stream the app enables devices on the call (`call.camera.enable()`, `call.microphone.enable()`, `call.screenShare.enable()`), the SDK subscribes automatically, and rendering binds to participants (`<ParticipantView participant={p} />`). Video quality adapts to bandwidth and to the size each tile is rendered at, with no app code; manual overrides exist (`call.setPreferredIncomingVideoResolution`, `call.setIncomingVideoEnabled`). `call.leave()` releases everything.

**Failure it prevents.** Porting the subscribe loop, manual layer selection or manual track teardown, which then fights the SDK.

**Look for.** Track creation, publish and subscribe calls, per-remote quality selection, DOM attach and detach, track close and destroy calls.

**Plan.** These rows are mostly `delete` or `shape-shift`. Set the initial camera and microphone state explicitly before joining (call-type defaults can otherwise decide it), and decide per surface whether audience members publish at all.

### 7. Ringing is an upgrade offer, not a mapping (`ringing`)

**What changes.** Neither Vonage nor Agora has a first-class ringing feature in its current video SDKs (Agora's older Signaling SDK, RTM 1.x, has call-invitation objects; see `agora-web.md`); apps that ring users build it themselves or on those invitations: invites over signalling or push, accept and decline messages, timeouts, missed-call bookkeeping, CallKit or ConnectionService wiring. Stream rings server-side: `call.getOrCreate({ ring: true, data: { members: [caller, ...callees] } })` (top-level `ring`; include the caller in members; use a fresh call id per ring), events `call.ring`, `call.accepted`, `call.rejected` (reason `timeout`, `busy`, `decline` or `cancel`) and `call.missed`, accept by `call.join()`, decline by `call.leave({ reject: true, reason: 'decline' })`, timeouts in the call type's `settings.ring` (`incoming_call_timeout_ms`, `auto_cancel_timeout_ms`, `missed_call_timeout_ms`), push through the call type's notification settings, and the React `<RingingCall />` component.

**Failure it prevents.** A line-by-line port of the hand-rolled machinery onto custom events, which then lacks multi-device cancel, missed-call records and the mobile push path.

**Look for.** Invite messages, accept/decline handlers, invite timers, push-sent invites, "busy" logic, CallKit or ConnectionService code.

**Plan.** Mark these rows `upgrade`: offer to delete the machinery and use Stream ringing, and say what the app gains (server-owned timeouts, cancel on every device, missed-call events). Offered, not assumed: the user decides.

### 8. Recording, egress and livestream (`recording-livestream`)

**What changes.** Stream starts recordings from the client (with `start-record-call`) or the server: `call.startRecording()` / `call.stopRecording()` (composite by default), typed `call.startRecording('composite' | 'individual' | 'raw')` on the client, `call.startRecording({ recording_type })` in node-sdk 0.8.14. Composite layouts are `grid`, `spotlight` and `single-participant`; a custom recording web app (`settings.recording.layout = { name: 'custom', external_app_url }`) exists, and Stream's docs ask you to contact support before using it. Individual recording writes up to six files per participant session. Storage is Stream-managed by default, or Amazon S3, Google Cloud Storage or Azure Blob. Recording quality goes up to 1440p. Readiness arrives as `call.recording_ready` (a webhook too).

Livestream: `livestream` and `audio_room` calls start in **backstage**; viewers wait until `call.goLive()` (options include `start_hls`, `start_recording`) or join early with `settings.backstage.join_ahead_time_seconds`. HLS: `call.startHLS()` / `call.stopHLS()` on the client, `startHLSBroadcasting()` on Node, playlist at `call.state.egress.hls.playlist_url`, up to three quality tracks, maximum 1440p. RTMP out: `call.startRTMPBroadcasts({ broadcasts: [{ name, stream_url, stream_key }] })`, `call.stopRTMPBroadcast(name)`, `call.stopAllRTMPBroadcasts()`. RTMP in: `call.state.ingress.rtmp.address`, with a user token as the stream key. Viewers can watch over WebRTC (`<LivestreamPlayer>`) or HLS.

Transcription is a **false friend**: `call.startTranscription()` writes a post-call file; live captions are `call.startClosedCaptions()` with `call.closed_caption` events.

**Failure it prevents.** A ported livestream that never calls `goLive()` leaves viewers waiting with no error. Recording orchestration ported verbatim instead of deleted. Live captions ported to `startTranscription` show nothing.

**Look for.** Recording start and stop, storage configuration, recording-ready handling, broadcast and CDN push, HLS players, captions.

**Plan.** Re-express storage and layout as call-type settings, delete the vendor-specific start/stop bookkeeping (resource ids, polling) while keeping the app's own rules (who may record, consent, retention, delivery of the files), add an explicit go-live decision for every livestream surface (call `goLive()`, or disable backstage on the call type).

### Integrations outside the call model (`integrations`)

Telephony (SIP dial-in and dial-out, DTMF), AI agents and audio connectors (an agent or a media stream leaving the call for an AI service), and chat or messaging products used next to video. They are not one of the nine areas because they rarely port line by line: each is either a supported Stream feature (inbound SIP, `call.dtmf`, Stream Chat), Stream's agent path (Vision Agents: the agent joins the call as a server-side participant), or a gap. Tag their inventory rows `integrations`, and list every one in "Flag for a human" with the right owner.

### 9. Fleet transition and coexistence (`fleet-transition`)

**What changes.** Every participant of a live session must be on the same platform: there is no bridging a Vonage or Agora participant into a Stream call. Web apps switch when users load the new build, which is fast but not instant: open tabs and cached bundles keep running the old SDK, so pin the vendor per call or room and keep the old credential route until those clients are gone. Mobile apps cannot be repointed: old installs keep running the old SDK for weeks or months.

**Plan, per surface:**

- **Calling (ringing):** choose the vendor per call at ring time; ring on Stream when both sides run a Stream-capable build, otherwise on the old vendor. Needs both SDKs in the transition build and push tokens registered with Stream before the switch.
- **Meetings:** pin the vendor per room when the room is created; old installs joining a Stream-pinned room see an "update to join" screen.
- **Livestreams with HLS viewers:** move hosts first; viewers keep watching an HLS URL and do not notice. WebRTC viewers behave like meetings.
- **Recordings and webhooks:** backend consumers handle both formats until the old vendor is switched off.

**Transition options** to present: A, hard cutover with a forced update; B, per-session vendor routing (the usual recommendation for mobile); C, scoped by cohort or feature (new surfaces on Stream only). The old vendor account must stay active through B and C, so its contract end date constrains the timeline.

**Early-prep checklist** (useful even if the app never migrates; diff it against what the app has): a server-decided vendor in the join path (credentials come back as `{ vendor, ...credentials }`), a forced-update gate on the call path, client capability or version reporting at login, push pre-registration for ringing apps, version-adoption telemetry, and store review and staged-rollout dates on the plan.

## Behaviour traps across vendors (the kill list)

Each vendor reference lists its own traps with anchors. These recur:

| Trap | What a literal port gets wrong |
|---|---|
| `subscribe-loop` | Explicit per-remote subscribe code is unnecessary on Stream; ported, it duplicates or fights automatic subscription, and any selective-subscribe product behaviour (audio-only mode, capacity guards) silently disappears unless rebuilt with `setIncomingVideoEnabled` / `setPreferredIncomingVideoResolution`. |
| `streams-vs-participants` | Vendors model a screen share as a separate stream or a separate client. On Stream it is another track on the same participant (`hasScreenShare(p)`, `trackType="screenShareTrack"`). Counts, grid sizing and "N in call" logic keyed on streams or clients change. |
| `volume-scale` | Vendor per-participant volume is 0-100 (both Vonage and Agora); Stream's `call.speaker.setParticipantVolume(sessionId, volume)` takes 0-1 and throws outside that range. |
| `moderator-mute` | Vendor moderation mutes (Vonage `forceMuteStream` / `forceMuteAll`, where `forceMuteAll` also mutes streams published later until `disableForceMute`) and self-enforced mute messages (Agora over RTM) map to `call.muteUser(userId, 'audio')` / `call.muteAllUsers('audio')`, which are one-shot soft mutes: Stream documents that a muted user can unmute themselves, and later joiners are not muted. For a mute the user cannot undo, also `call.revokePermissions(userId, ['send-audio'])`; for 'everyone who joins stays muted', remove `send-audio` from the role in the call type and grant it to speakers. |
| `publish-on-join` | Vendor SDKs publish only when told. On Stream initial device state can come from call-type defaults; set camera and microphone state explicitly before `join()`. |
| `identity-keying` | Moderation, UI and analytics keyed on per-connection or numeric vendor ids behave differently when one user has two devices; Stream kicks and blocks act on the user. |
| `token-scope` | Per-room tokens become per-user tokens (tokens-identity area). |
| `backstage` | Livestreams start in backstage; no `goLive()`, no stream. |
| `captions-false-friend` | `startTranscription` is post-call; live captions are `startClosedCaptions`. |
| `targeted-signals` | Vendor signals can target one connection or user; Stream custom events reach all call members who watch the call, and a recipient field in the payload only decides who displays it: everyone receives it. Non-sensitive routing ("you are next", UI hints) may move to custom events with a recipient field. Anything private (a question to the host, personal or health data, anything the app shows to one person only) must not: move it to Stream Chat (a direct channel) or a server relay the app owns, and flag it for a human. Never offer making the message public as an option. Payload limit 5 KB. Whether the sender receives its own custom event is not documented: filter on `event.user.id`. |
| `participant-cap` | `useParticipants()` / `call.state.participants` is truncated at 250 in large calls; use `useParticipantCount()` (server-computed, includes anonymous viewers) for counts. |

## Honesty and referral rules

These apply to every sentence the report writes about either platform.

1. **Describe other vendors as their public documentation does.** No characterisation of their quality, roadmap, company or customers, and no comment on their pricing models. A limitation that a vendor's own documentation states, and that matters to the app, is reported plainly and without disparagement, exactly as Stream's own limitations are.
2. **Say where Stream lacks something.** A `gap` row is stated plainly with what the app loses. Do not soften it and do not promise roadmap.
3. **No prices, no cost comparisons.** State what drives cost on Stream: per participant minute, by the aggregated resolution of the video tracks each participant receives (see `https://getstream.io/video/pricing/`). Then point at the app's own settings that move it (published resolution, layouts, target resolution). Recommend checking the call type's target resolution against the layouts the app uses.
4. **Refer to Stream sales** (`https://getstream.io/contact/`) for anything commercial or roadmap-dependent, and say it is a referral, not an answer:
   - **Fixed IPs and strict firewalls.** Stream publishes no fixed or static IP list; servers are a dynamically managed set. It works with a domain allowlist (`*.stream-io-video.com`, `*.stream-io-api.com`, `*.getstream.io`) and, at minimum, TURN over TLS on TCP 443. If the code shows a restricted-network setup (forced relay, custom TURN, a vendor IP allowlist or proxy add-on), name the file and line, state this, and refer the user to sales for their network's requirements.
   - **End-to-end encryption.** See the Stream facts below; always a flag for a human with a sales referral.
   - **SIP dial-out**, **on-premises deployment**, **dedicated infrastructure**, **data residency beyond what the docs state**, and **scale beyond the app's evident needs**: ask for the user's numbers, and refer.
   - **Mainland China.** Do not try to solve China-market delivery; say Stream's docs do not cover it and refer.
5. **Never argue jurisdiction, ownership or commercials.** The report is technical.
6. **Upgrade offers only where the code shows the need** (hand-rolled ringing -> Stream ringing; client-obeyed moderation -> server-enforced permissions; an external CDN or packager for HLS -> native HLS; a separate chat or signalling product -> Stream Chat). Never pitch features the app does not need.

## Stream facts the plan relies on

Verified against Stream's public documentation on 2026-10-22. Quote these, not memory.

| Topic | Fact |
|---|---|
| Built-in call types | `default`, `audio_room`, `livestream`, `development` (testing only). Apps define more. |
| Server calls | node-sdk: `client.video.call(type, id)`, then `call.getOrCreate(...)`, `call.startRecording({ recording_type })`, `call.sendCallEvent({ user_id, custom })`, `call.end()`. |
| Plain JS rendering | `call.bindVideoElement(videoEl, sessionId, 'videoTrack' \| 'screenShareTrack')` and `call.bindAudioElement(audioEl, sessionId)` return an unbind function; React apps use `<ParticipantView>`. |
| Join-time device defaults | call-type `settings.audio.mic_default_on`, `settings.audio.speaker_default_on`, `settings.video.camera_default_on`. |
| Token lifetime | `generateUserToken` defaults to 1 hour; the docs recommend a `tokenProvider` with short-lived tokens (about 4 hours). |
| Anonymous and guest users | Anonymous users (`{ type: 'anonymous' }`, token optional; a call-scoped token can restrict them) open no WebSocket and receive no events, and can only watch livestreams or join calls. Guest users (`{ id, type: 'guest' }`, no token) are created on connect. Every other user needs a token from the app's backend. |
| Ending calls | `call.endCall()` (client), `call.end()` (Node); `join-ended-call` controls rejoining an ended call and is granted by default on most types. |
| Mute | `call.muteUser` is soft (self-unmute allowed); revoking `send-audio` stops publishing. |
| Custom events | `call.sendCustomEvent({ type, ... })`, received as `custom`; 5 KB; to call members watching the call; no per-recipient targeting. |
| Recording | composite, individual (up to six files per participant session), raw; up to 1440p; custom recording app via support. |
| HLS and RTMP | HLS up to 1440p with up to three quality tracks; RTMP out and RTMP in supported; WebRTC livestream viewing. |
| 4K | No preset: set `settings.video.target_resolution`; Stream's docs ask you to contact support. Recording and HLS stop at 1440p. |
| AV1 | Selected automatically where supported (recent iPhones and Galaxy devices, Chrome, Firefox 136+, Safari 17+); the codec can switch during a call. |
| SIP | Inbound SIP trunks only; outbound dialling is in development. DTMF arrives as `call.dtmf`. |
| End-to-end encryption | Stream documents E2EE for React/JS, Android, iOS and Flutter (no React Native page), with keys distributed by the app; it cannot be combined with recording, transcription or HLS. Confirm availability for your plan with Stream before relying on it. |
| Networking | No fixed IP list; domain allowlist; TURN-TLS on TCP 443 is the minimum workable path. |
| Billing driver | Per participant minute, by the aggregated resolution of the video tracks each participant receives. |
