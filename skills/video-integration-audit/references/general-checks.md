# General checks (every mode, every vendor)

These checks apply to any video integration, whichever platform it uses. **The check, its pass condition and its severity are the same for every vendor.** Only the anchors you search for differ, and they are listed per vendor under each check. That is what makes the results comparable: the same defect in a Vonage app, an Agora app and a Stream app gets the same verdict and the same severity.

Run every check in every mode. In the "neither" mode (no supported video SDK found) most checks are `N/A`; say why on each row.

## Verdicts and severity

- **Verdict:** `PASS`, `FAIL`, `N/A` (the feature does not exist in this app; say why), `NEEDS-REVIEW` (the answer lives somewhere the code does not show, such as a dashboard setting or a service outside the repository; say where to look).
- **Severity:** `Blocker` (calls will not reliably connect, media keeps flowing after the user left, or security is broken), `High` (a real production stability, security or UX gap), `Medium` and `Low` (hardening and polish).
- Severity belongs to the check, not to the verdict: a `PASS` row still shows the check's severity so readers can see what was at stake.
- Most checks are **absence checks**: the defect is a missing handler, not visibly wrong code. Before calling an anchor absent, follow the app's own wrappers (a shared hook, a utility module, a wrapper library such as `agora-rtc-react`) that may implement it under another name. If it is still not found anywhere in the code you were given, the verdict is `FAIL`, unless the code could plausibly live outside the repository (a separate backend, a dashboard setting), in which case it is `NEEDS-REVIEW` and the row says what to check.
- Cite evidence as `path:line` for every verdict other than `N/A`. For an absence, say which anchors you searched for and where.
- Never quote a secret value. Cite the file and line and write `<redacted>`.

## The checks

### G1. Secrets stay on the server - Blocker

**Why:** the signing secret (Agora App Certificate, Vonage API secret or private key, Stream API secret, a vendor REST key) lets anyone mint tokens for any user and any room. In a client bundle or a committed file it is public.

**Pass:** the secret is read from the server environment or a secret store, never from client code, mobile app code, a client-side env prefix (`NEXT_PUBLIC_`, `VITE_`, `EXPO_PUBLIC_`, `REACT_APP_`), or a committed config file.

**FAIL if** a secret is assigned in client code, read through a client-side env prefix, present in a committed `.env`/config file, or a token builder library is imported into client code.

| Vendor | Anchors |
|---|---|
| Any | `-----BEGIN PRIVATE KEY-----`, `-----BEGIN RSA PRIVATE KEY-----`, `.env` files tracked in the repository, client env prefixes next to `SECRET`, `CERT`, `PRIVATE` |
| Agora | `appCertificate`, `APP_CERTIFICATE`, `AGORA_APP_CERT`, `RtcTokenBuilder` / `RtmTokenBuilder` / `AccessToken2` imported in client code, `customerSecret`, `CUSTOMER_SECRET` |
| Vonage | `apiSecret`, `OPENTOK_API_SECRET`, `VONAGE_API_SECRET`, `privateKey`, `VONAGE_PRIVATE_KEY`, `.key` / `.pem` files, `opentok` or `@vonage/video` imported in client code |
| Stream | `STREAM_API_SECRET`, `apiSecret`, `api_secret`, `@stream-io/node-sdk` or `getstream` server SDK imported in client code |

### G2. Tokens are minted per user on a server, expire, and are renewed - High (Blocker if a static token ships in client code)

**Why:** a token that never expires, or lives for days, cannot be revoked in practice. A static token in client code is a shared credential.

**Pass:** the client fetches its token from the app's backend and the token expires within 24 hours (this audit's threshold, not a vendor limit). Where the platform checks the token during the session (Agora, Stream), the client also renews it before expiry (a renewal event handler or a token provider callback). Where it checks the token only when connecting (Vonage), a fresh token fetched for each new connection is the renewal path.

**FAIL if** the token is hard-coded or bundled (Blocker), has no expiry or an expiry over 24 hours, or, on a platform that checks tokens during the session, the client has no renewal path for calls that can outlast the token.

| Vendor | Anchors |
|---|---|
| Agora | `buildTokenWithUid(` / `buildTokenWithUserAccount(` (`agora-token`: expiry in seconds from now, maximum 24 hours) or `buildTokenWithUid(` / `buildTokenWithAccount(` (deprecated `agora-access-token`: absolute epoch expiry); client `renewToken(` inside a `token-privilege-will-expire` handler |
| Vonage | `generateToken(` (legacy `opentok`) / `generateClientToken(` (`@vonage/video`) with `expireTime` in epoch seconds (default 24 hours; Vonage's current token guide states a 24-hour maximum, the legacy SDK documents 30 days); a fresh token fetched for each new connection |
| Stream | `generateUserToken({ user_id, validity_in_seconds })`, `createToken(`; client `tokenProvider` (renews automatically) versus a static `token:` |

### G3. The token endpoint authenticates the caller and binds the token to that caller - Blocker

**Why:** if the endpoint mints a token for whatever user id, uid, role or room the request names, anyone can join any room as anyone, including as a moderator.

**Pass:** the token route sits behind the app's own authentication, takes the identity from the authenticated session (not from a query or body parameter), and decides room access and role on the server.

**FAIL if** the route reads the identity, role or room from the request with no check against an authenticated session. `NEEDS-REVIEW` if the route is behind middleware you cannot see.

| Vendor | Anchors |
|---|---|
| Any | the route that calls the token builder: `/token`, `/api/token`, `/session`, `/credentials`, `/join`; `req.query`, `req.body`, `searchParams.get(`, `request.args` feeding the builder |
| Agora | client-chosen numeric uid (`Math.random` uid, uid from user input) passed to the token route |
| Vonage | `role: 'moderator'` or `role: 'publisher'` granted to every caller; session id taken from the request |
| Stream | `user_id` taken from the request; `role: 'admin'` set on user upsert for every caller |

### G4. Joining is error-handled - Blocker

**Why:** join fails for ordinary reasons (expired token, network block, denied permission, full room). An unhandled rejection leaves the user on a spinner with no way out.

**Pass:** the join call is awaited inside `try/catch` (or `.catch`, or the platform's error callback), and the failure is shown to the user with a way to retry or leave.

| Vendor | Anchors |
|---|---|
| Agora | `client.join(` (web), `joinChannel(` (native) |
| Vonage | `session.connect(token, callback)` (error is the callback's first argument), `session.publish(publisher, callback)` |
| Stream | `call.join(` |

### G5. Camera and microphone failures are handled - High

**Why:** devices fail (in use by another app, unplugged, blocked by OS policy). An uncaught failure strands the UI in a half-joined state.

**Pass:** device acquisition and enable/disable calls are error-handled, and the UI tells the user what failed.

| Vendor | Anchors |
|---|---|
| Agora | `createMicrophoneAudioTrack(`, `createCameraVideoTrack(`, `createMicrophoneAndCameraTracks(`, `createScreenVideoTrack(` |
| Vonage | `OT.initPublisher(` completion callback, publisher `accessDenied` event |
| Stream | `call.camera.enable(`, `call.microphone.enable(`, `call.screenShare.enable(`, `onError` props on control components |

### G6. Leaving releases the call and the devices - High

**Why:** a user who navigates away or closes the tab while the call object lives on keeps publishing, keeps the camera light on, or appears as a ghost participant.

**Pass:** the app leaves the call and releases local media when the user leaves, when the call screen unmounts or the route changes, and on page unload where the platform allows it.

| Vendor | Anchors |
|---|---|
| Agora | `client.leave(`, `track.close(` for every local track, unmount/route-change cleanup, `pagehide` / `beforeunload` |
| Vonage | `session.disconnect(`, `publisher.destroy(`, unmount cleanup, `pagehide` / `beforeunload` |
| Stream | `call.leave(`, `client.disconnectUser(` on logout or unmount |

### G7. Reconnects are shown, not treated as the end of the call - Medium

**Why:** networks drop for seconds at a time. The SDK reconnects on its own; an app that tears the call down on the first disconnect event turns a blip into a dropped call.

**Pass:** the UI reflects the reconnecting state and the app does not leave or rebuild the call on a transient disconnect.

| Vendor | Anchors |
|---|---|
| Agora | `connection-state-change` handler (`RECONNECTING`, `DISCONNECTED` with reason) |
| Vonage | `sessionReconnecting`, `sessionReconnected`, `sessionDisconnected` (reason `networkDisconnected`) |
| Stream | `useCallCallingState()` / `call.state.callingState$` (`RECONNECTING`, `OFFLINE`), `setDisconnectionTimeout` |

### G8. Permissions are enforced on the server, not only hidden in the UI - High

**Why:** host-only buttons hidden in the UI stop nobody: the API calls behind them can be made directly. If every token grants full rights, or moderation is a message the client is trusted to obey, any participant can act as host.

**Pass:** what a participant may do (publish, screen share, mute others, remove others, record) is decided on the server, in the token's role or the platform's role and permission configuration, and moderation actions are enforced by the platform.

**FAIL if** host status comes from client state (a checkbox, a query parameter, local storage) while tokens grant every user the same broad role, or if "mute" and "remove" are messages each client chooses to obey. `NEEDS-REVIEW` when the role configuration lives in a dashboard you cannot see.

| Vendor | Anchors |
|---|---|
| Agora | `setClientRole(` driven by client state; host actions sent over RTM / Signaling (`publish(` of a mute or kick message); server-side enforcement is the banning-user-privileges REST API (`kicking-rule`) |
| Vonage | token `role` (`publisher`, `moderator`, `subscriber`); `forceMuteStream(`, `forceMuteAll(`, `forceDisconnect(` (enforced by the platform) versus signals asking clients to mute |
| Stream | role and permission grid on the call type (dashboard or API), `call.muteUser(`, `call.blockUser(`, `call.kickUser(`, `revokePermissions(`; `ownCapabilities` checks in the UI are fine as long as the dashboard grid matches |

### G9. Autoplay-blocked audio is handled - High

**Why:** browsers block audio playback until the user interacts with the page. A participant who joins from a link without clicking hears nothing and there is no error.

**Pass:** the app detects blocked playback and shows a control that resumes audio on a user gesture, or it uses the platform's default UI that does this (Vonage's default subscriber UI unless `audioBlockedDisplayMode: 'off'`; Stream's default `ParticipantView` UI). `N/A` for native mobile apps.

| Vendor | Anchors |
|---|---|
| Agora | `AgoraRTC.onAutoplayFailed =` (there is no default UI: without this callback blocked audio is silent) |
| Vonage | subscriber `audioBlocked` / `audioUnblocked` events, `OT.unblockAudio()`, subscriber property `audioBlockedDisplayMode` |
| Stream | `useIsAutoplayBlocked()` (React), `call.resumeAudio()` inside a user gesture; the default `ParticipantView` UI shows a blocked-audio overlay. From `@stream-io/video-client` 1.63.0 (React SDK 1.44.1) any click or key press on the page also resumes blocked audio: that ends the silence at the first interaction but tells a user who has not interacted nothing, so it does not pass G9 on its own. Below that version, name the upgrade in the fix |

### G10. Device permission prompts are explained - High

**Why:** browsers and mobile OSes show the camera/microphone prompt once. A user who denies it, or does not notice it, needs to be told why the call has no camera and how to fix it.

**Pass:** the UI explains a pending or denied camera/microphone permission.

| Vendor | Anchors |
|---|---|
| Agora | `PERMISSION_DENIED`, `NOT_READABLE`, `DEVICE_NOT_FOUND` error codes from track creation |
| Vonage | publisher `accessDialogOpened`, `accessDialogClosed`, `accessAllowed`, `accessDenied` events |
| Stream | `useCameraState().hasBrowserPermission`, `isPromptingPermission` (React) |

### G11. Cost-driving media settings are explicit - Low

**Why:** published resolution, frame rate and recording resolution drive bandwidth and, on platforms that bill by resolution, cost. Settings left at a high default or set higher than the layout displays waste both.

**Pass:** the app sets camera and screen-share resolution deliberately, and the published resolution fits the layouts it uses (a 1080p camera feed into a grid of small tiles is a `NEEDS-REVIEW`). Record every explicit setting in the report; the migration plan uses them.

| Vendor | Anchors |
|---|---|
| Agora | `encoderConfig` (`'720p_1'`, `'1080p_1'`, objects with `width`/`height`), `setLowStreamParameter(`, Cloud Recording `transcodingConfig` width and height |
| Vonage | `OT.initPublisher` `resolution` (`'1280x720'`, `'1920x1080'`), `frameRate`; archive `resolution` |
| Stream | call type `settings.video.target_resolution`, `call.screenShare.setSettings(`, recording `quality` and HLS `quality_tracks` settings |

### G12. Webhook and callback handlers verify the sender - High

**Why:** endpoints that receive recording-ready, session or participant events drive billing, storage and access decisions. Unverified, they accept forged events from anyone who finds the URL.

**Pass:** each handler verifies the platform's signature before acting. `N/A` if the app receives no callbacks.

| Vendor | Anchors |
|---|---|
| Agora | Notification Center Service (NCS) callback handler and its signature check (verify the header name against Agora's NCS docs) |
| Vonage | callback URL handlers for archive, broadcast, render, captions and session monitoring (`statusCallbackUrl`); a signature or shared-secret check (verify against your Vonage account settings) |
| Stream | webhook route; `client.verifyAndParseWebhook(rawBody, signature)` or `client.verifyWebhook(body, signature)` with the `x-signature` header, on the raw request body |

## Reporting

One row per check, in this order, in the report's "General findings" table (see `output-contract.md`). Never skip a row. Carry each `FAIL` and `NEEDS-REVIEW` into the prioritised fix list, Blockers first.
