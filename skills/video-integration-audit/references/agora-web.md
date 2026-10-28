# Agora on the web -> Stream Video

**Pinned to:** Agora `agora-rtc-sdk-ng` 4.24.9, `agora-rtc-sdk` 3.6.11 (deprecated), `agora-rtc-react` 2.5.1, `agora-rtm-sdk` 2.3.0 (and 1.5.1 for RTM 1.x), `agora-token` 2.0.7, `agora-access-token` 2.0.4 (deprecated). Stream side: `@stream-io/video-react-sdk` 1.44.1, `@stream-io/video-client` 1.63.0, `@stream-io/node-sdk` 0.8.14. Agora names verified against the published package typings and `docs.agora.io` on 2026-10-22; Stream names against Stream's public docs the same day, rechecked against the 1.44.1 release on 2026-10-28. Other versions: mark rows `verify against your SDK version`.

Read `hub.md` first: the decision areas, the cross-vendor traps and the honesty rules live there. This file adds what is specific to Agora.

## The model shift in one paragraph

Agora is an engine plus imperative callbacks: the app creates local tracks, publishes them, subscribes to each remote user's audio and video in a `user-published` handler, plays them into DOM elements, and closes every track itself. Identity is a per-channel `uid` (usually a number the client picks), auth is a per-channel token, and capability is a runtime `setClientRole('host' | 'audience')` switch in `live` mode. Stream is reactive managed state: the app enables devices on a call, the SDK subscribes automatically, the UI renders participants, and `call.leave()` cleans up. Identity is a string user id the backend owns, auth is a per-user token from a `tokenProvider`, and capability comes from the call type's roles and permissions. Much of the client-side media code is deleted rather than renamed; the app's own rules (who may join, record or moderate) stay and move to the server.

## Detection and vintage

| Signal | Means |
|---|---|
| `agora-rtc-sdk-ng` in `package.json`; `import AgoraRTC from 'agora-rtc-sdk-ng'`; `client.join(appid, channel, token, uid)` | Agora Web SDK 4.x (current) |
| `agora-rtc-sdk` in `package.json`; `client.init(appId, ...)` then `client.join(token, channel, uid, ...)`; `AgoraRTC.createStream({ ... })`; events `stream-added`, `stream-subscribed`, `stream-removed`, `peer-leave` | Agora Web SDK 3.x (deprecated by Agora; unsupported since July 2023). See "3.x vintage" below |
| `agora-rtc-react` with `AgoraRTCProvider`, `useJoin`, `useLocalMicrophoneTrack`, `useLocalCameraTrack`, `usePublish`, `useRemoteUsers` | Agora's React wrapper over 4.x. See "agora-rtc-react" below |
| `<script src="https://download.agora.io/sdk/release/AgoraRTC_N-<version>.js">` | 4.x from the CDN (global `AgoraRTC`) |
| `AgoraRTCSDK-<version>.js` from `cdn.agora.io` or `download.agora.io`, or `AgoraRTCSDK.min.js` | 3.x |
| `agora-rtm-sdk` with `new AgoraRTM.RTM(appId, userId)`, `rtm.publish(`, `rtm.subscribe(` | Agora Signaling (RTM) 2.x |
| `agora-rtm-sdk` with `AgoraRTM.createInstance(`, `client.createChannel(`, `channel.sendMessage(`, `client.createLocalInvitation(` | RTM 1.x |
| `agora-chat` | Agora Chat (a separate messaging product) |
| `agora-extension-ai-denoiser`, `agora-extension-virtual-background`, other `agora-extension-*` | Agora extensions (see flag list) |
| Server: `agora-token` (`RtcTokenBuilder.buildTokenWithUid`, `buildTokenWithUserAccount`, `RtmTokenBuilder.buildToken`), `agora-access-token` (deprecated; `buildTokenWithAccount`, absolute expiry), `agora-token-builder` (Python), `RtcTokenBuilder2` / `AccessToken2` sources | Token minting |
| Server: requests to `api.agora.io` or `api.sd-rtn.com` paths `/cloud_recording/`, `/rtmp-converters`, `/dev/v1/kicking-rule` | Cloud Recording, Media Push, banning user privileges |
| `AGORA_APP_ID`, `appCertificate`, `customerKey` / `customerSecret` (REST Basic auth) | Credentials (see G1 in `general-checks.md`) |

**Configuration that is not in the code** (ask; list in "Questions the code cannot answer"): whether the project uses an App Certificate (token auth) or testing mode with App ID only; co-host authentication; Cloud Recording storage targets configured outside the request; Notification Center Service (NCS) callbacks configured in the console; enabled extensions and their licences.

**Out of scope for a port:** apps that serve users inside mainland China. Say Stream's docs do not cover mainland China and refer to sales (see `hub.md`).

## Mapping (4.x)

Stream equivalents are for `@stream-io/video-client` 1.63.0 and `@stream-io/video-react-sdk` 1.44.1.

### Tokens and identity

| Agora construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| `RtcTokenBuilder.buildTokenWithUid(appId, cert, channel, uid, role, tokenExpire, privilegeExpire)` / `buildTokenWithUserAccount(...)` | `client.generateUserToken({ user_id, validity_in_seconds })` (`@stream-io/node-sdk`) | shape-shift | Per-channel token -> per-user token; channel and role leave the token. Default lifetime 1 hour |
| `RtmTokenBuilder.buildToken(appId, cert, userId, expire)` | none needed for in-call signalling (custom events use the same user token); Stream Chat uses the same user token if adopted | delete | |
| App ID + App Certificate | API key (public) + API secret (server only) | rename | Agora App ID with no certificate (testing mode) has no Stream analogue: Stream users get a token from the app's backend; only guest and anonymous users connect without one, with the limits in `hub.md` (Stream facts) |
| `uid` (number, often `Math.random()` on the client; or `null` for server-assigned) / user account string | Stream user id (string, from the app's backend) | shape-shift | Rekey every map, DOM id and analytics field keyed on `uid` (see `uid-identity` below) |
| `client.renewToken(token)` in `client.on('token-privilege-will-expire')` | `tokenProvider: async () => token` on `new StreamVideoClient(...)`; the SDK renews | delete | Also delete `token-privilege-did-expire` handling |

### Channels, modes and roles

| Agora construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| `AgoraRTC.createClient({ mode: 'rtc', codec })` | `new StreamVideoClient({ apiKey, user, tokenProvider })`; React `<StreamVideo client={client}>` | shape-shift | One client per user session, not per channel. Codec choice is the SDK's (AV1, VP9, H.264, VP8 negotiated); drop it |
| `mode: 'live'` + `setClientRole('host' \| 'audience', { level })` | a `livestream` (or `audio_room`) call type; hosts have `send-audio`/`send-video`; promotion is `call.grantPermissions(userId, ['send-audio', 'send-video'])` or `call.requestPermissions({ permissions })` | config | No role toggle; see `permissions` in `hub.md`. Audience latency levels have no equivalent: WebRTC viewers are realtime; HLS viewers are the scale path |
| `mode: 'p2p'` (4.24+) | none: Stream is always routed through its SFU | gap | Flag if the app relies on media never transiting servers |
| channel name | call id; full id `type:id` | rename | Choose a call type per surface |
| `client.join(appid, channel, token, uid)` | `client.call(type, id)` then `await call.join({ create: true })` | shape-shift | Wrap in `try/catch` (G4) |
| `client.leave()` + `track.stop()` / `track.close()` per local track | `await call.leave()` | delete | Manual track teardown goes away |

### Tracks, devices and rendering

| Agora construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| `createMicrophoneAudioTrack()`, `createCameraVideoTrack()`, `createMicrophoneAndCameraTracks()` + `client.publish([...])` | `call.microphone.enable()`, `call.camera.enable()` (before or after join) | shape-shift | Two steps become one. Set device state explicitly before `join()` (`publish-on-join` in `hub.md`) |
| `encoderConfig: '720p_1'`, `'1080p_1'`, `{ width, height, frameRate, bitrate }` | call type `settings.video.target_resolution`; the SDK publishes simulcast layers | config | Record the value (G11); see the billing note in `hub.md` |
| `client.on('user-published', (user, mediaType) => client.subscribe(user, mediaType))` + `user.videoTrack.play(el)` / `user.audioTrack.play()` | none: automatic subscription; render `useParticipants()` / `call.state.participants$` with `<ParticipantView participant={p} />` (React) or `call.bindVideoElement(el, sessionId, 'videoTrack')` + `call.bindAudioElement(el, sessionId)` (plain JS) | delete | `subscribe-loop` trap |
| `user-unpublished`, `user-joined`, `user-left` handlers that edit the DOM | participant state; discrete events `call.session_participant_joined` / `call.session_participant_left` if business logic needs them | delete | Keep only the business logic, moved to state or webhooks |
| `client.remoteUsers` | `useRemoteParticipants()` / `call.state.remoteParticipants` | rename | |
| `track.setEnabled(bool)` (stops capture) / `track.setMuted(bool)` (keeps capture; remote sees `user-unpublished`) | `call.camera.enable()` / `disable()` / `toggle()`, same for `call.microphone` | rename | Both Agora variants map to the same Stream calls |
| `remoteAudioTrack.setVolume(0..100)` | `call.speaker.setParticipantVolume(sessionId, 0..1)` | behavioral-diff | `volume-scale` trap. Local `localAudioTrack.setVolume(0..1000)` (microphone gain) has no direct Stream equivalent: flag |
| `client.enableDualStream()`, `setLowStreamParameter()`, `setRemoteVideoStreamType(uid, 0 \| 1)` | none needed: simulcast and size-aware layer selection are automatic; overrides `call.setPreferredIncomingVideoResolution({ width, height }, sessionIds)` | delete | Ported layer selection fights the SDK |
| `client.enableAudioVolumeIndicator()` + `volume-indicator` | participant `isSpeaking`, `isDominantSpeaker`, `audioLevel`; `useDominantSpeaker()` | delete | No enable call |
| `AgoraRTC.getDevices()`, `getCameras()`, `getMicrophones()`, `track.setDevice(id)` | `useCameraState().devices`, `call.camera.select(deviceId)`; `call.camera.listDevices()` (an observable) | shape-shift | |
| `AgoraRTC.onAutoplayFailed = () => ...` | `useIsAutoplayBlocked()` + `call.resumeAudio()` in a user gesture; default `ParticipantView` overlay | shape-shift | G9 |
| `AgoraRTC.checkSystemRequirements()` | supported-platform checks in the docs | delete | |
| `connection-state-change` (`RECONNECTING`, `DISCONNECTED`) | `useCallCallingState()` (`RECONNECTING`, `OFFLINE`, `RECONNECTING_FAILED`) | shape-shift | G7 |
| `network-quality` | participant `connectionQuality` | rename | |

### Screen share

| Agora construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| `AgoraRTC.createScreenVideoTrack(config, withAudio)` published on a **second client** joined with its own uid | `call.screenShare.enable()` / `disable()` / `toggle()` on the same call; `call.screenShare.enableScreenShareAudio()`; render with `<ParticipantView participant={p} trackType="screenShareTrack" />` where `hasScreenShare(p)` | behavioral-diff | `screen-share-client` trap: delete the second client, its token request and every uid-offset convention |
| `encoderConfig` / `optimizationMode: 'detail' \| 'motion'` on the screen track | `call.screenShare.setSettings({ maxFramerate, maxBitrate, contentHint })` before enabling | shape-shift | |
| `track-ended` on the screen track | screen-share state (`useScreenShareState()`) | delete | |

### Signalling, host controls and ringing

| Agora construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| RTM 2.x `rtm.publish(channel, message)` / RTM 1.x `channel.sendMessage({ text })` for in-call app messages | `call.sendCustomEvent({ type, ... })` + `call.on('custom', ...)`; or Stream Chat for durable messages | shape-shift | 5 KB limit; reaches call members watching the call; no per-recipient targeting (`targeted-signals` in `hub.md`) |
| RTM messages telling a client to mute itself or leave (host controls) | `call.muteUser(userId, 'audio')`, `call.muteAllUsers('audio')`, `call.kickUser({ user_id, block })`, `call.blockUser(userId)`, `call.revokePermissions(userId, caps)`; enforced by the platform | upgrade | Client-obeyed moderation becomes server-enforced (G8) |
| REST `POST /dev/v1/kicking-rule` with `privileges: ['publish_audio' \| 'publish_video' \| 'join_channel']` | `call.revokePermissions(userId, ['send-audio' \| 'send-video'])`, `call.blockUser(userId)` | shape-shift | Stream revocation lasts until granted back, not for N minutes |
| RTM peer or inbox messages carrying invites, accept/decline replies, invite timeouts | Stream ringing (`hub.md`, area 7) | upgrade | `calling` surface |
| RTM 1.x call invitations: `client.createLocalInvitation(calleeId)`, `localInvitation.send()` / `cancel()`, `RemoteInvitationReceived`, `remoteInvitation.accept()` / `refuse()`, events `LocalInvitationAccepted`, `LocalInvitationRefused`, `LocalInvitationCanceled`, `LocalInvitationFailure`, `RemoteInvitationCanceled` | Stream ringing: `call.getOrCreate({ ring: true, data: { members } })`, accept with `call.join()`, decline with `call.leave({ reject: true, reason: 'decline' })`, events `call.ring`, `call.accepted`, `call.rejected`, `call.missed` (`hub.md`, area 7) | shape-shift | `calling` surface. Agora keeps invitation state per invitation object; Stream keeps it on the call, so use a fresh call id per ring. Push, multi-device cancel and missed-call bookkeeping built around the invitations become `upgrade` rows |
| `client.sendStreamMessage(...)` / `stream-message` (data stream; declared in the ESM typings, not in Agora's API reference) | `call.sendCustomEvent(...)` | shape-shift | Data-stream reliability options have no equivalent |
| `agora-chat` | Stream Chat (same API key, users and tokens) | upgrade | A re-integration, not a rename; decision area `integrations` |

### Recording, broadcast and ingest

| Agora construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| Cloud Recording REST `acquire` -> `start` (`mode/mix`) -> `stop`, with `storageConfig` | `call.startRecording()` / `call.stopRecording()` (client, with `start-record-call`) or node-sdk `call.startRecording({ recording_type: 'composite' })`; layout and storage on the call type | delete (orchestration), config (storage, layout) | The resource id / sid bookkeeping goes away. `call.recording_ready` replaces polling `query` |
| Cloud Recording `mode/individual` | individual recording: client `call.startRecording('individual')`, node-sdk `call.startRecording({ recording_type: 'individual' })`; up to six files per participant session | shape-shift | |
| Cloud Recording `mode/web` (records a web page) | composite recording with a custom recording app (`settings.recording.layout = { name: 'custom', external_app_url }`); Stream's docs ask you to contact support first | flag | Partial mapping: flag for a human |
| `transcodingConfig` (`width`, `height`, `mixedVideoLayout`) | recording `quality` (up to 1440p) and `layout` (`grid`, `spotlight`, `single-participant`) on the call type | config | |
| Media Push REST `rtmp-converters` / client `setLiveTranscoding()` + `startLiveStreaming(url)` | RTMP out: `call.startRTMPBroadcasts({ broadcasts: [{ name, stream_url, stream_key }] })`, `call.stopRTMPBroadcast(name)`; or native HLS: `call.goLive({ start_hls: true })` / `call.startHLS()`, playlist `call.state.egress.hls.playlist_url` | shape-shift or upgrade | Where Media Push feeds an external packager or CDN only to produce HLS, native HLS removes that hop: offer it (`upgrade`). Ask about the CDN's other duties first |
| Media Gateway (RTMP/SRT ingest into a channel) | RTMP ingest: `call.state.ingress.rtmp.address` with a user token as the stream key; WHIP ingest | shape-shift | SRT: verify against current Stream docs |
| NCS callbacks (recording, channel events) | Stream webhooks (configure in the dashboard; verify with `client.verifyAndParseWebhook`) | shape-shift | Regenerate handlers from Stream's event payloads; do not translate |

### Other

| Agora construct | Stream equivalent | Class | Notes |
|---|---|---|---|
| `client.setEncryptionConfig(mode, secret, salt)` (built-in media encryption, e.g. `aes-128-gcm2`) | Stream E2EE (see `hub.md` Stream facts) | flag | Blocker-class requirement: state what Stream documents and refer to sales |
| `client.startProxyServer(3 \| 5)` (cloud proxy: UDP, or TCP/TLS 443) | TURN over TLS on TCP 443 and a domain allowlist; no fixed IPs | flag | Restricted-network rule in `hub.md`: name the file and line, refer to sales |
| `agora-extension-ai-denoiser` | `@stream-io/audio-filters-web` with `NoiseCancellationProvider` | shape-shift | Desktop browsers only; flag mobile-web use |
| `agora-extension-virtual-background` | `BackgroundFiltersProvider` / `useBackgroundFilters()` | shape-shift | Chrome, Firefox, Edge 124+; Safari opt-in; flag older or mobile browsers |
| Agora Conversational AI Engine, or an AI agent joining a channel as a uid | Stream's agent path: Vision Agents, where an agent joins the call as a server-side participant | flag | Decision area `integrations`; a re-architecture, not a port |
| Agora on-premises recording | none in Stream's docs | flag | Refer to sales |

## Behaviour traps specific to Agora

| Trap | Anchors | What to plan |
|---|---|---|
| `subscribe-loop` | `user-published`, `client.subscribe(`, `.play(` | Delete. If the app subscribes selectively (audio-only mode, a cap on rendered videos), rebuild that with `call.setIncomingVideoEnabled(false)`, which stops receiving video, and say so. Not rendering a tile is not the same as not receiving it; selective audio subscription has no mapping in this skill: flag it |
| `uid-identity` | `Math.random` uids, `uid` from user input, maps keyed on `uid`, `uid === 0` checks (the local user in `volume-indicator`) | Server-owned string user ids; one user can have several sessions (`sessionId`); `uid === 0` means "me" only in Agora |
| `screen-share-client` | a second `createClient(`, uid offsets (`uid + 100000`, `uid >= OFFSET`), filtering remote users by uid range | Delete the second client. Counts that subtract screen clients become plain participant counts; tiles come from `hasScreenShare(p)` (`streams-vs-participants` in `hub.md`) |
| `volume-scale` | remote `setVolume(`, sliders `max="100"` | Divide by 100 for `setParticipantVolume`; values above 100 (Agora local tracks accept up to 1000) have no equivalent |
| `mute-variants` | `setMuted(` vs `setEnabled(` | Agora `setMuted(true)` keeps the camera on and fires `user-unpublished` remotely; `setEnabled(false)` releases the device. Stream `disable()` covers both; check camera-light expectations |
| `client-enforced-host` | `setClientRole(` driven by UI state, RTM host commands | Server-enforced permissions (G8); hosts decided by the backend and call-member roles |
| `token-renew-loop` | `token-privilege-will-expire`, `renewToken(` | Delete; `tokenProvider` |
| `manual-teardown` | `track.close()`, `track.stop()`, `client.leave()` sequences, `RtcEngine.destroy` equivalents | Delete; `call.leave()` |
| `autoplay-no-default` | absence of `onAutoplayFailed` | Agora has no default blocked-audio UI; a ported Stream app gets the default `ParticipantView` overlay, so this improves, but custom UIs need `useIsAutoplayBlocked()` |
| `live-latency-levels` | `setClientRole('audience', { level })` | No equivalent: WebRTC viewers are realtime; HLS for scale |

## Flag for a human (always)

RTM-based host controls and invites, including RTM 1.x call invitations (product decisions: which become permissions, which ringing); any one-recipient RTM message carrying private content (`targeted-signals` in `hub.md`); Cloud Recording `web` mode; Media Push and the CDN behind it; built-in encryption; cloud proxy; extensions on mobile browsers; Conversational AI; on-premises recording; `p2p` mode; microphone gain above 100; any data-stream use; apps serving mainland China.

## 3.x vintage

Map 3.x to the 4.x rows above first; the Stream side is the same.

| 3.x | 4.x concept | Notes |
|---|---|---|
| `client.init(appId, onSuccess, onFailure)` then `client.join(token, channel, uid, info, onSuccess, onFailure)` | `client.join(appid, channel, token, uid)` | Argument order differs; callbacks instead of promises |
| `AgoraRTC.createStream({ streamID, audio, video, screen })` + `stream.init()` + `client.publish(stream)` | local tracks + `publish` | One stream object holds audio and video |
| `stream-added` -> `client.subscribe(stream)`; `stream-subscribed` -> `stream.play(elementId)` | `user-published` -> `subscribe` -> `play` | Same `subscribe-loop` trap |
| `stream-removed`, `peer-leave` | `user-unpublished`, `user-left` | |
| `stream.setAudioVolume(0..100)` | remote `setVolume(0..100)` | Same `volume-scale` trap |
| `stream.setVideoProfile('720p_3')` | `encoderConfig` | G11 |
| `client.setClientRole('host' \| 'audience', cb)` in `mode: 'live'` | same | |

A 3.x app is also an upgrade candidate on Agora itself (Agora no longer supports 3.x); say so neutrally, as Agora's own notice does.

## agora-rtc-react 2.x

| Hook or component | Stream React equivalent |
|---|---|
| `AgoraRTCProvider` | `<StreamVideo client={client}>` + `<StreamCall call={call}>` |
| `useJoin({ appid, channel, token, uid })` | `call.join({ create: true })` in an effect, with cleanup `call.leave()` |
| `useLocalMicrophoneTrack()`, `useLocalCameraTrack()`, `usePublish(tracks)` | `call.microphone.enable()`, `call.camera.enable()`; state from `useMicrophoneState()`, `useCameraState()` |
| `useRemoteUsers()`, `useRemoteAudioTracks(users)` | `useRemoteParticipants()`; audio plays automatically |
| `RemoteUser`, `LocalVideoTrack`, `LocalUser` | `<ParticipantView participant={p} />` |
| `useLocalScreenTrack()`, `AgoraRTCScreenShareProvider` | `call.screenShare.enable()`, `useScreenShareState()` |
| `useConnectionState()`, `useIsConnected()` | `useCallCallingState()` |
| `useNetworkQuality()` | participant `connectionQuality` |
| `useVolumeLevel()` | participant `audioLevel`, `isSpeaking` |
| `useClientEvent(client, event, handler)` | `call.on(event, handler)` for discrete events; state hooks otherwise |
