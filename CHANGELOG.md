# Changelog

Entries are per skill, newest first. Each skill is versioned on its own; see "Versions" in [`README.md`](README.md).

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
