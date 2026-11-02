# Changelog

Entries are per skill, newest first. Each skill is versioned on its own; see "Versions" in [`README.md`](README.md).

## stream-video-webview-audit 0.7 (2026-11-02)

Checked against `@stream-io/video-react-sdk` 1.45.1 (`@stream-io/video-client` 1.64.1).

- A9: from 1.45.1 the SDK asks the browser again when the user turns the microphone or camera on after a denial, so a deny-first host costs the user a second tap and a muted start instead of a dead microphone. Severity is High below 1.45.1 and Medium from it.
- B7 now asks for 1.45.1 or later.

## video-integration-audit 0.2.2 (2026-11-02)

- Pinned to `@stream-io/video-react-sdk` 1.45.1 (`@stream-io/video-client` 1.64.1, `@stream-io/audio-filters-web` 0.11.1).
- Noise-cancellation rows: from audio-filters-web 0.11.1, Safari and other WebKit browsers report noise cancellation unsupported unless the app passes `withSafariSupport: true`.
- G10: `isBrowserPermissionBlocked` (1.45.1) separates a browser block from a denial the SDK retries.

## stream-video-webview-audit 0.6 (2026-11-02)

Checked against `@stream-io/video-react-sdk` 1.45.0 (`@stream-io/video-client` 1.64.0). What 1.45.0 changes comes from its source and release notes; the device runs behind the observations were not repeated on it.

- B7 now asks for 1.45.0 or later. 1.45.0 no longer leaves a call offline after a stale browser `offline` event on a working network, and `call.leave()` no longer hangs on a pending quality change. It restarts remote audio after an iOS audio interruption, and it retries a token refresh that fails during a reconnect. The stale `offline` event is no longer listed as having no host fix.
- A11, B3, B5 and the iOS reference: the SDK now notices an iOS audio interruption and restarts remote audio when it ends, but does not tell the page, so the host's log is still the only record of the cause.
- B6: from 1.45.0 a screen share's `contentHint` sets what gives way under load (`motion` keeps the frame rate, `detail` and `text` keep the resolution), and the share's bitrate cap is no longer scaled down to its starting capture size.
- A9 and the Android reference: 1.45.0 still does not ask for the microphone again after one denial.

## video-integration-audit 0.2.1 (2026-11-02)

- Stream equivalents pinned to `@stream-io/video-react-sdk` 1.45.0 (`@stream-io/video-client` 1.64.0); no API the references name was removed or renamed. S20 measures against 1.45.0.
- S31: from 1.45.0, noise cancellation turns itself off when the device cannot keep up and reports `suspended` or `overloaded`; a toggle that shows only on and off misreads that.
- Microphone-processing rows (Vonage, Agora): from 1.45.0 the browser's noise suppression is off while Stream's noise cancellation is registered.
- Screen-share rows (Vonage, Agora): from 1.45.0 `contentHint` sets what gives way under load; Agora's `optimizationMode` maps onto it.

## video-integration-audit 0.2 (2026-10-28)

First public version (0.1 was internal only). Detects the video platform in the code (Vonage Video API, OpenTok, TokBox, Agora, Stream Video; recognises Twilio Video, Zoom Video SDK, 100ms, Daily and LiveKit) and reports in one of four modes: migration, stream, hybrid or neither. Code-level references cover web apps; iOS, Android, React Native and Flutter code gets a platform-level analysis.

- General checks G1 to G13 (secrets, token minting and the token endpoint, join and device error handling, cleanup on leave, reconnects, server-enforced permissions, autoplay and permission prompts, cost-driving media settings, webhook verification, media state shown to the user), with the same verdict and severity for every vendor.
- Vonage Video (OpenTok and unified) and Agora (3.x and 4.x) web references, with the Stream equivalents pinned to `@stream-io/video-react-sdk` 1.44.1 (`@stream-io/video-client` 1.63.0) and `@stream-io/node-sdk` 0.8.14.
- Stream Video web best-practice checks S1 to S31.
- Output contract: seventeen sections by mode plus a machine-readable `video-integration-audit/v1` JSON block. Section 4, "What changes for this app on Stream", lists what the app gains and what it must change, each line tied to evidence in the code. Section 5, "Cost analysis", names what drives the app's cost with no prices, states what the code shows about usage volume, and recommends asking Stream sales for a volume quote. The dossier's Architecture subsection and the JSON block carry what a porting step needs: client and server structure, server routes, identity, role mapping, stored vendor ids, recordings, video UI components, environment variable names, tests and the size of the work per area.
- Honesty and referral rules: other vendors described as their documentation does, no prices or cost comparisons, no measured quality claims, Stream's limitations stated plainly, requirements the code cannot settle referred to sales.

The test apps and recorded runs the skill is evaluated against are kept outside this repository.

## stream-video-webview-audit 0.5 (2026-10-28)

First public version. Checked against `@stream-io/video-react-sdk` 1.44.1 (`@stream-io/video-client` 1.63.0).

- Section A: vendor-independent checks for any WebRTC SDK inside a WebView (Android foreground service, keeping the screen on, no reloads or recreated pages, configuration changes, WebView pause and timers, one call-screen instance, renderer or content-process death, autoplay policy, microphone permission, edge-to-edge insets, the iOS audio session, origin checks on the native bridge).
- Section B: checks specific to Stream's React video SDK (iOS background media handling, client and call lifetime, page lifecycle events, autoplay-blocked audio, host event logging, screen-share availability in WebViews, minimum SDK version).
- References with Kotlin and Swift snippets for the Android and iOS hosts.
