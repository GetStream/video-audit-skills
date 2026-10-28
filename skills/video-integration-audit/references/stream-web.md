# Stream Video on the web: best-practice checks

**Pinned to:** `@stream-io/video-react-sdk` 1.44.1 (`@stream-io/video-client` 1.63.0), `@stream-io/node-sdk` 0.8.14. Verified 2026-10-22 against 1.44.0; rechecked against 1.44.1 on 2026-10-28 (no public API changes).

**Source of truth:** Stream's public page "Integration best practices" for React (`https://getstream.io/video/docs/react/advanced/integration-best-practices/`, last changed 2026-09-08). The rows follow it and the read-only audit in Stream's agent-skills pack (`stream-react`, Video reference, "Integration best-practices audit"), last synced 2026-10-22. Where the two disagree, the public page wins, except where the SDK has since changed (noted per row). This version adds server-side and cross-cutting rows the page does not cover (marked "added").

**Mode rules.** Judge against Stream's public documentation only. Where the app works around a Stream limitation, report the limitation as Stream's, not as an app defect. Offer a Stream feature only when the code shows the need.

**How to run it.** Confirm `@stream-io/video-react-sdk` or `@stream-io/video-client` is installed (record the version). Search the app source (call screens, providers, the token route, webhook routes) for each row's anchors. Most rows are absence checks: if the anchor is not found, the row is `FAIL`, or `NEEDS-REVIEW` if it could live elsewhere (a backend, the dashboard). `N/A` only when the feature does not exist in the app, with the reason. Never skip a row.

Plain JavaScript apps (`@stream-io/video-client` without React): apply the same rows, reading React hooks as their `call.state.*$` observables (`useCallCallingState` -> `call.state.callingState$`, `useParticipants` -> `call.state.participants$`). Rows about React components or strict mode are `N/A`.

## Rows reported as general checks

These best-practice items are general checks (`general-checks.md`), reported once in the general findings table, not repeated here: token from a backend route with a `tokenProvider` and short lifetime (G2, G3), `call.join()` in `try/catch` (G4), `camera.enable()` / `microphone.enable()` errors and `onError` on control components (G5), `call.leave()` when the call is no longer needed (G6), reconnects and `setDisconnectionTimeout` (G7), role permissions enforced in the dashboard (G8), autoplay-blocked audio (G9), browser permission prompt UX (G10), webhook signatures (G12).

## Carve-outs (never a finding)

- `<CallControls />` and the other prebuilt components are official; using them is fine. Hand-built controls are fine too when wired through `useCallStateHooks()`.
- `usePersistedDevicePreferences` is deprecated and a no-op; its absence is not a finding (device persistence is the client option `options.devicePersistence`, on by default).
- `call.muteUser()` being a soft mute is the documented behaviour. It is a finding only if the app's own UI promises a mute the user cannot undo (see S25).
- Stream has no fixed IPs and inbound-only SIP: an app that works around either is not defective.

## Checklist

### Client and call setup

| ID | Check | Anchors | Pass condition | Severity |
|---|---|---|---|---|
| S1 | One client, created in `useEffect`/`useState`, disposed on cleanup | `new StreamVideoClient(`, `StreamVideoClient.getOrCreateInstance(`, `disconnectUser(`, `useMemo` | One client per user session; `client.disconnectUser()` in cleanup or on logout; **not** created in `useMemo` (strict mode and re-renders then leave a disconnected instance) | Blocker |
| S2 | Strict-mode safe setup | `useRef` "ran once" guards, `mounted` flags in async effects | No `useRef` run-once guards in setup effects; async setup effects (`call.join`, `getOrCreate`) clean up on unmount. The synchronous client constructor needs only the `disconnectUser()` cleanup | High |
| S3 | One `<StreamVideo>` provider, at the app root | `<StreamVideo` | A single provider above every call screen, not one per screen | High |
| S4 | Media connects through `join()` | `call.join(`, `getOrCreate(` | `call.join({ create: true })` (or `getOrCreate()` followed by `join()`); `getOrCreate()` alone never connects media | Blocker |
| S5 | Components read the call from context | `useCall()`, `client.call(` inside components | Call screens are wrapped in `<StreamCall call={call}>` and read it with `useCall()`; components do not create new `client.call(type, id)` instances (each returns a new, uninitialised object) | Medium |
| S6 | All calling states have UI | `useCallCallingState()`, `CallingState.` | The UI reacts to `JOINING`, `RECONNECTING`, `OFFLINE`, `RECONNECTING_FAILED`, `LEFT`, and (for ringing apps) `RINGING` | High |
| S7 | `connectUser` errors handled | `connectUser(` | Rejections caught and shown (`N/A` when the client is constructed with the user and never calls `connectUser`) | High |

### Devices and media UX

| ID | Check | Anchors | Pass condition | Severity |
|---|---|---|---|---|
| S8 | Device switcher | `DeviceSettings`, `DeviceSelectorVideo`, `DeviceSelectorAudioInput`, `useCameraState().devices`, `call.camera.select(` | Users can pick a non-default camera, microphone and speaker | Medium |
| S9 | Devices verified before joining | `AudioVolumeIndicator`, `SpeakerTest`, `MicCaptureErrorNotification` | Users can confirm devices work, not only that they are selected | Medium |
| S10 | Device persistence not disabled | `devicePersistence` in client options | `PASS` unless `options.devicePersistence.enabled === false` without a replacement | Low |
| S11 | Lobby before joining | a lobby or preview screen, `VideoPreview` | Users see their camera and devices before connecting; link joins pass through a user gesture | Medium |
| S12 | Audio connecting indicator | media connecting indicator in custom participant UI | Custom participant UI distinguishes "connecting" from "muted" (built-in `ParticipantView` UI passes) | Low |
| S13 | Speaking while muted | `isSpeakingWhileMuted`, `SpeakingWhileMutedNotification`, `disableSpeakingWhileMutedNotification(` | Surfaced to the user, or explicitly disabled | Medium |
| S14 | Filters can be turned off | `BackgroundFiltersProvider`, `useBackgroundFilters`, `NoiseCancellationProvider`, `useNoiseCancellation` | If filters are used, users have a toggle (background blur is not auto-disabled under CPU load). Background filters support Chrome, Firefox and Edge 124+ (Safari opt-in); noise cancellation is desktop-only | Medium (`N/A` without filters) |
| S15 | Low-bandwidth notice on custom layouts | custom layout components, low-bandwidth UI | Built-in layouts (`SpeakerLayout`, `PaginatedGridLayout`) pass; custom layouts tell the user when video is paused for bandwidth | Medium |
| S16 | Restrictive-network guidance | connection-test link, network help text | Users on corporate networks are told what to allow (domains, TURN over TLS on 443) or pointed at Stream's connection test | Low |
| S17 | SDK styles imported | `@stream-io/video-react-sdk/dist/css/styles.css` | Imported once (`N/A` for fully custom UI) | Low |
| S18 | Quality feedback collected | user rating UI, `call.submitFeedback(` | Some end-of-call feedback path exists | Low |
| S19 | Unsupported browsers advised | supported-platform checks | Users on unsupported browsers are told so | Low |
| S20 | SDK reasonably current | `package.json`, lockfile | `@stream-io/video-react-sdk` not more than about 10 minor versions behind 1.44.1 (the version this file is pinned to, 2026-10-28); `FAIL` below 1.34.0. Point to the SDK's GitHub releases | Low |

### Server and cross-cutting (added)

| ID | Check | Anchors | Pass condition | Severity |
|---|---|---|---|---|
| S21 | Call types fit the surfaces | `client.call('<type>'`, call-type names | Each kind of surface (meeting, calling, livestream) uses a call type configured for it; `development` is never used in production | Medium |
| S22 | Ringing uses Stream ringing | `sendCustomEvent(` with invite-like types, invite timers, push-sent invites, `ring: true`, `call.ring`, `<RingingCall` | Apps that ring users use `getOrCreate({ ring: true, data: { members } })` and the ring events. Hand-rolled invites on custom events or an external push service are not a defect in themselves: `PASS` with the upgrade offer as a note (server-owned timeouts, cancel on every device, `call.missed`), offered, not assumed. `FAIL` only when the hand-rolled flow has a defect the code shows: no timeout, no cancel when the caller hangs up, or invites anyone can send | Medium (`N/A` without calling) |
| S23 | Livestreams have a go-live path | `goLive(`, `stopLive(`, backstage settings, `join_ahead_time_seconds`, `livestream` call type | Every `livestream` or `audio_room` surface calls `goLive()` (or the call type disables backstage); hosts have `join-backstage`; join-ahead is a deliberate choice | High (`N/A` without livestreams) |
| S24 | Ended calls stay ended where the product needs it | `endCall(`, `call.end(`, `join-ended-call` | Where ending a call should be final (1:1 calls, appointments), the call type does not grant `join-ended-call` to ordinary users (most built-in types grant it by default) | Medium (`N/A` if the app never ends calls) |
| S25 | Mute semantics match the UI's promise | `muteUser(`, `muteAllUsers(`, `muteOthers(`, `revokePermissions(` | If the UI offers a mute the participant should not undo ("mute and lock", webinar attendees), the app also revokes `send-audio`; a plain soft mute passes otherwise | Medium (`N/A` without moderation) |
| S26 | Custom events used within their limits | `sendCustomEvent(`, `.on('custom'` | Payloads stay under 5 KB; nothing that must survive a reconnect relies on a custom event; handlers ignore the sender's own events (filter on `event.user.id`; self-delivery is not documented); private messages are not sent as call-wide custom events | Medium (`N/A` without custom events) |
| S27 | Counts use the server count | `participants.length`, `useParticipants().length`, `useParticipantCount()` | Displayed participant or viewer counts use `useParticipantCount()` (server-computed, includes anonymous viewers); `useParticipants()` is truncated at 250 in large calls | Low (Medium for livestreams) |
| S28 | Live captions use closed captions | `startTranscription(`, `startClosedCaptions(`, `call.closed_caption` | Live captions come from `startClosedCaptions()`; `startTranscription()` is for the post-call transcript | Medium (`N/A` without captions) |
| S29 | RTMP ingest keys last long enough | `ingress.rtmp`, `useCallIngress`, `generateUserToken(` used as a stream key | The user token used as an RTMP stream key is valid for the broadcast's length (the default token lifetime is 1 hour) | Medium (`N/A` without RTMP ingest) |
| S30 | Target resolution fits the layouts | `target_resolution`, call-type settings, layout components | The call type's `settings.video.target_resolution` fits the layouts the app renders; Stream bills per participant minute by the aggregated resolution of the video each participant receives, so a grid of small tiles rarely needs the default type's high target. Link the pricing page; quote no prices | Low |
| S31 | Noise cancellation considered where people speak | `NoiseCancellationProvider`, `useNoiseCancellation`, `NoiseCancellation` from `@stream-io/audio-filters-web`, `call.microphone.enableNoiseCancellation(` | `meeting` and `calling` surfaces where people speak offer Stream's noise cancellation (desktop browsers), or the report notes it as an option. A missing filter is `NEEDS-REVIEW`, never `FAIL`: it is a product choice, metered separately on the pricing page | Low (`N/A` for listen-only and livestream-viewer surfaces) |

## Fix list

Carry every `FAIL` and `NEEDS-REVIEW` into the report's prioritised fix list. The report's "How to continue" points to the agent-skills pack's `stream-react` skill (or `stream-react-native` for React Native), whose enhance track can apply these fixes.
