# Vonage Video API on the web -> Stream Video

**Pinned to:** Vonage `@vonage/client-sdk-video` 2.35.3 (same code as `@opentok/client` 2.35.3), server `@vonage/video` 1.31.1 (used by `@vonage/server-sdk` 3.30.1) and legacy `opentok` 2.24.0, `@vonage/video-express` 1.10.3 (deprecated), React Native `@vonage/client-sdk-video-react-native` / `opentok-react-native` 2.34.1. Stream side: `@stream-io/video-react-sdk` 1.44.0, `@stream-io/video-client` 1.62.0, `@stream-io/node-sdk` 0.8.14. Vonage names verified against the published package typings, the Vonage Video JS reference and `developer.vonage.com` on 2026-10-22; Stream names against Stream's public docs the same day. Other versions: mark rows `verify against your SDK version`.

Read `hub.md` first. This file adds what is specific to Vonage.

## The model shift in one paragraph

Vonage Video (formerly TokBox OpenTok) is a publish/subscribe object model. The app holds a `Session`, creates a `Publisher` that renders into a DOM element, publishes it, and subscribes to each remote `Stream` (usually in a `streamCreated` handler), getting one `Subscriber` per remote stream. A participant who shares their screen publishes a second stream. Tokens are minted per session with a role (`subscriber`, `publisher`, `publisheronly`, `moderator`); sessions are created server-side, have opaque ids and do not expire. Stream is a call and participant model: the app joins a call, enables devices, the SDK receives everyone automatically, and the UI renders participants, each of which can carry camera, microphone and screen-share tracks. Tokens are per user; capabilities come from the call type. The client port is architectural; the server port is mostly renames plus the token model change.

## Detection and vintage

Vonage's unified and legacy web clients are **the same SDK under two names** (`@vonage/client-sdk-video` and `@opentok/client` ship the same code, global `OT`), so client code alone cannot tell the vintages apart. Use the server side and the credential naming.

| Signal | Means |
|---|---|
| `@vonage/client-sdk-video` or `@opentok/client` or `@vonage/video-client` in `package.json`; `OT.initSession(`, `OT.initPublisher(` | Vonage Video web client (either vintage) |
| `<script src="https://static.opentok.com/v2/js/opentok.min.js">` (or old `TB.min.js` / `TB.` global) | Legacy OpenTok CDN |
| `<script src="https://unpkg.com/@vonage/client-sdk-video@.../dist/js/opentok.js">` (or jsDelivr) | Unified CDN |
| Server `@vonage/video` (`new Video({ applicationId, privateKey })`, `generateClientToken(`) or `@vonage/server-sdk` | Unified Vonage server |
| Server `opentok` (`new OpenTok(apiKey, apiSecret)`, `generateToken(`) | Legacy OpenTok server. Note: Since `opentok` 2.24 it can also run with an application id and private key and returns JWT tokens by default, so a JWT or a UUID credential does not by itself prove the unified platform |
| Numeric API key + API secret (`OPENTOK_API_KEY`, `OPENTOK_API_SECRET`, `TOKBOX_*`) | Legacy OpenTok account |
| Application id (UUID) + private key (`VONAGE_APPLICATION_ID`, `VONAGE_PRIVATE_KEY`, a `.key` / `.pem` file) | Unified Vonage application |
| `@vonage/video-express` (`new VideoExpress.Room(`) | Video Express: deprecated by Vonage on 2026-05-12 (Vonage recommends its Video API Reference App instead). Map its room/participant concepts directly to Stream; it is closer to Stream's model than raw opentok.js |
| `opentok-react` (community React wrapper: `OTSession`, `OTPublisher`, `OTStreams`, `OTSubscriber`) | Web React wrapper over opentok.js |
| `opentok-react-native` or `@vonage/client-sdk-video-react-native` (same SDK, two names; `OTSession`, `OTPublisher`, `OTSubscriber`) | React Native: use this file for concepts, mark rows `verify against your SDK version` |
| `nexmo-client` or `@vonage/client-sdk` conversation calls (`newConversation`, `getConversation`, `createConversation`, `joinConversation`, `sendMessageTextEvent`) next to Video | Vonage In-App Messaging: see the flag list |
| `pipecat-ai[vonage-video-connector]` in `requirements*.txt` / `pyproject.toml`; `from pipecat.transports.vonage.video_connector import VonageVideoConnectorTransport` | Vonage Video Connector transport for Pipecat (AI agents in sessions) |
| Server `connectToWebsocket(` (unified) / `websocketConnect(` (legacy), options `bidirectional: true` | Audio Connector (session audio to a WebSocket; two-way since 2025-06-26) |

**Configuration that is not in the code** (ask): archive storage targets (S3, Azure) and callbacks configured in the Vonage dashboard; the account's add-ons (allowed IP list, IP proxy, regional media zones, China relay); session-monitoring callback URLs; whether sessions are created with `e2ee` through the REST API.

## Mapping

Stream equivalents are for `@stream-io/video-client` 1.62.0, `@stream-io/video-react-sdk` 1.44.0 and `@stream-io/node-sdk` 0.8.14.

### Server: sessions, tokens and identity

| Vonage construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| unified `new Video({ applicationId, privateKey })` / legacy `new OpenTok(apiKey, apiSecret)` | `new StreamClient(apiKey, apiSecret)` (`@stream-io/node-sdk`) | rename | |
| `createSession({ mediaMode, archiveMode, location })` | no server step needed: node-sdk `client.video.call(type, id)` + `call.getOrCreate({ data: { members, settings_override } })`, or `join({ create: true })` on the client | shape-shift | Stream call ids are chosen by the app (reuse appointment or room ids); Vonage session ids are vendor-generated and stored by the backend: storing the session id can go, but keep whatever else that record carries (who may join, links already sent, history) |
| `mediaMode: 'relayed'` (legacy default) / `MediaMode.RELAYED` | none: Stream always routes through its SFU | gap | Usually invisible; matters if compliance claims rely on media not touching servers. Note: Legacy `opentok` defaults to relayed, `@vonage/video` to routed |
| `archiveMode: 'always'` | call-type recording mode (`auto-on`) | config | |
| `location: '<IP>'` (media server hint) | none needed: edge selection is automatic | delete | |
| `generateClientToken(sessionId, { role, expireTime, data })` / legacy `generateToken(sessionId, { role, expireTime, data })` | `client.generateUserToken({ user_id, validity_in_seconds })`; role moves to the user or call member; `data` moves to call-member `custom` (`call.updateCallMembers({ update_members: [{ user_id, custom }] })`) when it describes this call, or to user `custom` via `upsertUsers` only when it describes the person in every call | shape-shift | Per-session token -> per-user token (`token-scope` in `hub.md`); to pin a token to specific calls, `generateCallToken({ user_id, call_cids, role })`. Connection `data` is per connection: putting call-specific data on the user leaks it into the user's other calls |
| role `moderator` / `publisher` / `publisheronly` / `subscriber` | call-member roles and the call type's permission grid (for example a `host` role with `mute-users`, `kick-user`, `end-call`; an audience role without `send-audio`/`send-video`) | config | `publisheronly` (publish but not subscribe) has no direct equivalent; flag if used |
| `initialLayoutClassList` token option | none (layout classes are a Vonage archive/broadcast layout concept) | gap | See custom layouts below |

### Client: connecting, publishing, subscribing

| Vonage construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| `OT.initSession(apiKeyOrApplicationId, sessionId, options)` | `new StreamVideoClient({ apiKey, user, tokenProvider })` + `client.call(type, id)` | shape-shift | |
| `session.connect(token, callback)` | `await call.join({ create: true })` in `try/catch` | shape-shift | The callback's error argument becomes a rejection (G4) |
| `session.disconnect()` | `await call.leave()` | rename | |
| `OT.initPublisher(targetElement, { publishAudio, publishVideo, resolution, frameRate, name, audioSource, videoSource })` | no publisher object: `call.camera.enable()` / `call.microphone.enable()`; `name` -> user name; `resolution` / `frameRate` -> call type `target_resolution` (record it, G11) | shape-shift | The DOM-injecting publisher goes; render the local participant with `<ParticipantView>` |
| `session.publish(publisher, callback)` | implicit once devices are enabled and the call is joined | delete | `publish-on-join` trap: set device state before `join()`; call-type defaults are `settings.audio.mic_default_on` and `settings.video.camera_default_on` |
| `session.unpublish(publisher)` / `publisher.destroy()` | `call.camera.disable()` + `call.microphone.disable()` | behavioral-diff | Vonage removes the stream (`streamDestroyed` remotely); on Stream the participant stays and loses tracks |
| `publisher.publishAudio(bool)` / `publishVideo(bool)` | `call.microphone.enable()` / `disable()` / `toggle()`, same for `call.camera` | rename | |
| `publisher.cycleVideo()` / `setVideoSource(deviceId)` / `setAudioSource(...)` | `call.camera.flip()` / `call.camera.select(deviceId)` / `call.microphone.select(deviceId)` | rename | |
| `OT.getDevices()`, `OT.getAudioOutputDevices()` | `useCameraState().devices`, `useMicrophoneState().devices`, `call.speaker` device selection | shape-shift | |
| `publisher.applyVideoFilter(...)` / `applyAudioFilter(...)` | `BackgroundFiltersProvider` / `useBackgroundFilters()`; `NoiseCancellationProvider` with `@stream-io/audio-filters-web` | shape-shift | Browser support differs: background filters Chrome, Firefox, Edge 124+ (Safari opt-in); noise cancellation desktop only |
| `session.on('streamCreated', e => session.subscribe(e.stream, el, props))` | none: automatic; render `useParticipants()` / `call.state.participants$` with `<ParticipantView>` (React) or `call.bindVideoElement(el, sessionId, 'videoTrack')` + `call.bindAudioElement(el, sessionId)` (plain JS) | delete | `subscribe-loop` trap |
| `session.subscribe(stream, el, { subscribeToAudio: false })`, `subscriber.subscribeToVideo(false)` | video: `call.setIncomingVideoEnabled(false)` (stops receiving video); audio: `call.speaker.setParticipantVolume(sessionId, 0)` silences playback but the audio is still received | behavioral-diff | Selective subscription is a product behaviour: rebuild deliberately. Where the app relies on not receiving a participant's audio (bandwidth, privacy), there is no mapping in this skill: flag it |
| `subscriber.setPreferredResolution({ width, height })` / `setPreferredFrameRate(fps)` | automatic size-aware selection; override `call.setPreferredIncomingVideoResolution({ width, height }, [sessionId])` | delete or shape-shift | |
| `subscriber.setAudioVolume(0..100)` | `call.speaker.setParticipantVolume(sessionId, 0..1)` | behavioral-diff | `volume-scale` trap (Stream throws outside 0-1) |
| `audioLevelUpdated` (publisher, subscriber) | participant `audioLevel`, `isSpeaking`; `useDominantSpeaker()` | shape-shift | |
| subscriber `videoDisabled` / `videoEnabled` (reason `quality`) | participant `connectionQuality`; built-in layouts show low-bandwidth state | shape-shift | Custom UIs rebuild the "video paused for quality" notice (S15) |
| subscriber `audioBlocked` / `audioUnblocked`, `OT.unblockAudio()` | `useIsAutoplayBlocked()` + `call.resumeAudio()` | rename | G9 |
| publisher `accessDialogOpened`, `accessAllowed`, `accessDenied` | `useCameraState()` / `useMicrophoneState()` `hasBrowserPermission`, `isPromptingPermission` | shape-shift | G10 |
| `getStats()` / `getRtcStatsReport()` on publisher or subscriber | call stats (`useCallStatsReport()`) | shape-shift | |
| `OT.checkSystemRequirements()`, `OT.checkScreenSharingCapability()` | supported-platform checks in the docs | delete | |
| `session.getPublisherForStream()` / `getSubscribersForStream()` | not needed | delete | |

### Events

| Vonage event | Stream equivalent | Class | Notes |
|---|---|---|---|
| `sessionConnected` | `call.join()` resolves; `useCallCallingState()` `JOINED` | rename | |
| `sessionDisconnected` (reasons `clientDisconnected`, `forceDisconnected`, `networkDisconnected`) | calling state `LEFT` / `OFFLINE` / `RECONNECTING_FAILED`; kicks arrive as a `LEFT` state after `call.kickUser` | shape-shift | One event with a reason fans out to several Stream signals |
| `sessionReconnecting` / `sessionReconnected` | calling state `RECONNECTING` -> `JOINED` | rename | G7 |
| `connectionCreated` / `connectionDestroyed` | `call.session_participant_joined` / `call.session_participant_left`; counts from `useParticipantCount()` | behavioral-diff | Vonage fires for every connection, including subscribe-only clients; anonymous Stream viewers receive no events and are counted only in the server count |
| `streamCreated` / `streamDestroyed` | participant `publishedTracks` changes (reactive) | behavioral-diff | `streams-vs-participants` trap |
| `streamPropertyChanged` (`hasAudio`, `hasVideo`, `videoDimensions`) | participant state | delete | |
| `signal` / `signal:<type>` | `call.on('custom', e => ...)`, filtered on a payload field | shape-shift | Per-type listeners become in-handler filtering |
| `archiveStarted` / `archiveStopped` | `call.recording_started` / `call.recording_stopped` (plus `call.recording_ready`, `call.recording_failed`) | rename | |
| `muteForced` | `call.user_muted`; microphone state changes | shape-shift | |
| `OT.on('exception')` | promise rejections; `onError` on controls | shape-shift | |

### Signalling

| Vonage construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| `session.signal({ type, data, to, retryAfterReconnect })` (data up to 8 kB, type up to 128 characters) | `call.sendCustomEvent({ type, ... })` (up to 5 KB) | behavioral-diff | `targeted-signals` trap: no `to:` targeting, and a recipient field in the payload does not keep the message from other members; a private one-recipient signal moves to Stream Chat or a server relay, never to custom events; no `retryAfterReconnect` buffering; payloads between 5 and 8 kB fail only after migration |
| server `sendSignal(signal, sessionId, connectionId?)` / legacy `signal(sessionId, connectionId, payload)` | server-side custom event: node-sdk `call.sendCallEvent({ user_id, custom })` | shape-shift | Note the argument order change between Vonage's own SDKs |

### Moderation

| Vonage construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| `session.forceMuteStream(stream)` (client, moderator token) / server `muteStream(sessionId, streamId)` (unified) / `forceMuteStream` (legacy) | `call.muteUser(userId, 'audio')`; for a mute the user cannot undo, add `call.revokePermissions(userId, ['send-audio'])` | behavioral-diff | `moderator-mute` trap (see `hub.md`) |
| `session.forceMuteAll(excludedStreams)` / server `forceMuteAll` | `call.muteAllUsers('audio')`; to keep later joiners muted, take `send-audio` away from the role in the call type and grant it to speakers | behavioral-diff | Vonage documents that streams published after `forceMuteAll` are muted until `disableForceMute()`; Stream's mute-all is one-shot |
| `session.disableForceMute()` | `call.grantPermissions(userId, ['send-audio'])` or restore the role grant | shape-shift | |
| `session.forceDisconnect(connection)` / server `disconnectClient(sessionId, connectionId)` (unified) / `forceDisconnect` (legacy) | `call.kickUser({ user_id, block })` | behavioral-diff | `identity-keying`: Vonage removes one connection, Stream acts on the user (all devices) |
| `session.forceUnpublish(stream)` | one-off: `call.muteUser(userId, 'video' \| 'audio' \| 'screenshare')`; lasting: `call.revokePermissions(userId, ['send-video', 'send-audio'])` (or `'screenshare'`) | behavioral-diff | `forceUnpublish` acts on one stream once; a revoked permission lasts until granted back and covers all of that user's devices. Pick per moderation action |
| `session.capabilities.forceMute` / `forceDisconnect` checks | `useHasPermissions(OwnCapability.MUTE_USERS)` (React), `call.permissionsContext` | shape-shift | UI checks are fine if the call-type grid matches (G8) |
| no "end for everyone" (sessions never end) | `call.endCall()` (client, `end-call`) / `call.end()` (Node) | upgrade | Changes re-entry: see `call-lifecycle` in `hub.md` and S24 |

### Screen share

| Vonage construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| a second `OT.initPublisher(el, { videoSource: 'screen' })` + `session.publish(...)` | `call.screenShare.enable()` / `disable()` / `toggle()`; `enableScreenShareAudio()`; `setSettings({ maxFramerate, maxBitrate, contentHint })` | behavioral-diff | `streams-vs-participants` trap: the share is a track on the same participant; render it with `<ParticipantView participant={p} trackType="screenShareTrack" />` where `hasScreenShare(p)` |
| `stream.videoType === 'screen'` checks to place screen tiles | `hasScreenShare(p)`; `useHasOngoingScreenShare()` | shape-shift | |

### Archiving, broadcast, composer, captions

| Vonage construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| `startArchive(sessionId, { name, hasAudio, hasVideo, outputMode: 'composed', layout, resolution, streamMode })` | `call.startRecording()` (client) / node-sdk `call.startRecording({ recording_type: 'composite' })`; layout (`grid`, `spotlight`, `single-participant`), quality (up to 1440p) and storage on the call type | shape-shift + config | Stream clients with `start-record-call` can start recordings; Vonage archives are server-only |
| `outputMode: 'individual'` | individual recording: client `call.startRecording('individual')`, node-sdk `call.startRecording({ recording_type: 'individual' })`; up to six files per participant session | shape-shift | |
| `hasVideo: false` archives | audio-only recording: call-type `settings.recording.audio_only: true` | config | |
| `stopArchive`, `searchArchives` (legacy `listArchives`), `deleteArchive` | `call.stopRecording()`, `call.listRecordings()`, delete via the API | rename | |
| `updateArchiveLayout` (legacy `setArchiveLayout`) with a custom `stylesheet` | `settings.recording.layout` options; custom CSS via `layout.external_css_url`; a custom recording app via `external_app_url` (contact support first, per Stream's docs) | shape-shift | |
| archive status callbacks; polling for availability | `call.recording_ready` (WebSocket and webhook) | shape-shift | |
| `startBroadcast(sessionId, { outputs: { hls: { lowLatency, dvr }, rtmp: [...] }, layout, maxDuration, resolution })` | HLS: `call.goLive({ start_hls: true })` or `call.startHLS()` (Node `startHLSBroadcasting()`), playlist `call.state.egress.hls.playlist_url`; RTMP: `call.startRTMPBroadcasts({ broadcasts: [{ name, stream_url, stream_key }] })` | shape-shift | One Vonage broadcast object splits into HLS and RTMP on Stream. LL-HLS and DVR options: not in Stream's docs (flag). Also consider WebRTC viewing (`<LivestreamPlayer>`) for low latency |
| `stopBroadcast` | `call.stopHLS()`, `call.stopRTMPBroadcast(name)`, `call.stopAllRTMPBroadcasts()` | shape-shift | |
| `searchBroadcasts` / `getBroadcast` | the call's egress state: `call.state.egress` (`broadcasting`, `hls`, `rtmps`) | shape-shift | Per call, not a cross-session search |
| `addStreamToBroadcast` / `removeStreamFromBroadcast` (choosing which streams the broadcast shows) | none in this skill | no-mapping | Flag for a human |
| Experience Composer `startExperienceComposerRender(sessionId, token, { url, ... })` (legacy `startRender`) used for custom archive or broadcast layouts | composite recording or broadcast with a custom recording app (`settings.recording.layout = { name: 'custom', external_app_url }`), with Stream support's help per its docs | flag | Partial mapping. Composer used to fan one session out into another (chaining) has no equivalent: `gap` |
| `enableCaptions(sessionId, token, { languageCode, partialCaptions })` / `disableCaptions(captionId)` (legacy `startCaptions` / `stopCaptions`) | `call.startClosedCaptions({ language })` / `stopClosedCaptions()`, events `call.closed_caption` | shape-shift | `captions-false-friend`: not `startTranscription` |
| archive `hasTranscription` / `transcriptionProperties` | `call.startTranscription()` (post-call transcript, `call.transcription_ready`) | shape-shift | |

### AI, telephony, connectors (decision area `integrations`)

| Vonage construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| Audio Connector `connectToWebsocket(sessionId, token, { uri, streams, audioRate, bidirectional })` | Stream's agent path: Vision Agents, where an agent joins the call as a server-side participant | flag | Report as its own inventory class ("AI integration"), not as plain audio streaming. A two-way connector (`bidirectional: true`) is an AI agent in practice |
| Pipecat `VonageVideoConnectorTransport` (`pipecat.transports.vonage.video_connector`) / the Video Connector server SDK | Stream's agent path (Vision Agents) | flag | AI integration class; the agent code moves to a Stream transport. Re-architecture, not a port |
| SIP Interconnect dial-out: `initiateSIPCall(sessionId, { token, sip: { uri, from, headers, auth, secure, video } })` (legacy `dial`) | none today: Stream supports inbound SIP trunks only; outbound is in development | gap | Blocker-class for apps that dial phones: say so, refer to sales |
| inbound phone-to-video bridging (SIP into a session) | Stream inbound SIP trunks | shape-shift | |
| `playDTMF(sessionId, digits, connectionId?)` (legacy `playDTMF(sessionId, connectionId, digits)`) | receiving DTMF: `call.dtmf` event (`digit`, `seq_number`); sending DTMF from the call: not in Stream's docs | flag | |

### Network and security options

| Vonage construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| `OT.initSession(..., { iceConfig: { includeServers: 'custom', transportPolicy: 'relay', customServers: [...] } })`, `session.setIceConfig(...)` | Stream's TURN (UDP and TLS on 443); domain allowlist; no custom TURN, no fixed IPs | flag | Restricted-network rule in `hub.md`: name the file and line, state Stream's position, refer to sales |
| `ipWhitelist: true` (Vonage's allowed IP list add-on) | none: no fixed IPs | flag | Same rule |
| `OT.setProxyUrl(url)` (IP proxy add-on; a global call made before any other `OT` call) | none documented | flag | Same rule |
| `encryptionSecret` option on `OT.initSession`, `session.setEncryptionSecret(secret)`, subscriber `encryptionSecretMismatch`; server `createSession({ e2ee: true })` (legacy `opentok` or REST; `@vonage/video` has no `e2ee` option); React Native `OTSession encryptionSecret` prop | Stream E2EE (see `hub.md` Stream facts) | flag | Blocker-class requirement, common in telehealth: state what Stream documents (incompatible with recording, transcription and HLS) and refer to sales |
| `sessionMigration: true` | none needed | delete | |
| `singlePeerConnection`, `connectionEventsSuppressed` | none | delete | |

### In-App Messaging next to Video (decision area `integrations`)

Vonage's notice: "Effective April 30th, 2026, Vonage In-App Messaging will no longer be available." The notice names no packages. If the app uses `nexmo-client` or `@vonage/client-sdk` conversation methods (`newConversation`, `getConversation`, `getConversations`, `createConversation`, `joinConversation`, `sendMessageTextEvent` and other `sendMessage*Event`) for chat alongside Video, flag it: "Vonage's notice says In-App Messaging was discontinued on 2026-04-30; this code calls its conversation APIs (`path:line`)." Pair it with the Stream Chat upgrade offer (same API key, users and tokens as Stream Video). State only what the notice says; do not speculate about what still works.

## Behaviour traps specific to Vonage

| Trap | Anchors | What to plan |
|---|---|---|
| `subscribe-loop` | `streamCreated`, `session.subscribe(` | Delete. Selective subscription (audio-only mode, capacity guards, "show N videos") must be rebuilt deliberately |
| `streams-vs-participants` | counters incremented in `streamCreated`, `videoType === 'screen'`, grid sizing by stream count | One participant can have a camera and a screen share; counts and layouts move to participants plus `hasScreenShare(p)` |
| `volume-scale` | `setAudioVolume(`, sliders `max="100"` | Divide by 100 |
| `moderator-mute` | `forceMuteStream(`, `forceMuteAll(`, `muteStream(`, `disableForceMute(` | Add `revokePermissions` or a call-type role design; see `hub.md` |
| `identity-keying` | `forceDisconnect(connection)`, maps keyed on `connectionId`, `connection.data` parsing | Key on the user id; per-device presence is `sessionId` |
| `connection-counting` | viewer counts from `connectionCreated` | `useParticipantCount()` (includes anonymous viewers) |
| `publish-on-join` | apps that join without publishing until a "go on camera" button | Set `call.camera` / `call.microphone` state before `join()`; check the call type's defaults |
| `session-immortality` | stored session ids reused for recurring rooms; "close room" actions | Decide per surface: reusable call ids, or fresh ones; `endCall` is final unless `join-ended-call` is granted (see `hub.md`) |
| `signal-targeting` | `signal({ to: ... })`, `retryAfterReconnect: true`, payloads near 8 kB | Re-model 1:1 messages: private ones to Stream Chat or a server relay, never custom events (`targeted-signals` in `hub.md`); nothing critical on custom events across reconnects; 5 KB |
| `server-token-per-session` | `generateClientToken(sessionId, ...)` called per room join | One user token per user session; room access from membership and the call type |
| `archive-auto-timers` | logic assuming Vonage's archive or broadcast auto-stop timings | Re-baseline on Stream (no equivalent stated in Stream's docs; verify) |

## Flag for a human (always)

End-to-end encryption; restricted-network options (`iceConfig` relay or custom TURN, `ipWhitelist`, `OT.setProxyUrl`); SIP dial-out and DTMF sending; Experience Composer; Audio Connector and the Pipecat / Video Connector agent path; LL-HLS or DVR broadcast options; choosing which streams a broadcast shows; private one-recipient signals; audio subscription switched off per participant; relayed media mode where compliance depends on it; `publisheronly` roles; In-App Messaging dependencies; anything configured in the Vonage dashboard that the code assumes.

## Server SDK renames (legacy `opentok` -> unified `@vonage/video`)

Useful for vintage detection and for reading older code. Both map to the Stream rows above.

| Legacy `opentok` | Unified `@vonage/video` |
|---|---|
| `generateToken(sessionId, opts)` | `generateClientToken(sessionId, opts)` |
| `listArchives` | `searchArchives` |
| `setArchiveLayout(archiveId, type, stylesheet, screenshareType)` | `updateArchiveLayout(archiveId, layout)` |
| `signal(sessionId, connectionId, payload)` | `sendSignal(signal, sessionId, connectionId?)` (argument order changes) |
| `forceDisconnect(sessionId, connectionId)` | `disconnectClient(sessionId, connectionId)` |
| `getStream` / `listStreams` | `getStreamInfo(sessionId, streamId?)` |
| `forceMuteStream(sessionId, streamId)` | `muteStream(sessionId, streamId)` |
| `forceMuteAll`, `disableForceMute` | same names |
| `listBroadcasts` | `searchBroadcasts` |
| `addBroadcastStream` / `removeBroadcastStream` | `addStreamToBroadcast` / `removeStreamFromBroadcast` |
| `setBroadcastLayout` | no unified equivalent |
| `startRender` / `stopRender` | `startExperienceComposerRender` / `stopExperienceComposerRender` |
| `websocketConnect` | `connectToWebsocket` |
| `dial` | `initiateSIPCall` |
| `playDTMF(sessionId, connectionId, digits)` | `playDTMF(sessionId, digits, connectionId?)` |
| `startCaptions` / `stopCaptions` | `enableCaptions` / `disableCaptions` |
| callbacks | promises only |
