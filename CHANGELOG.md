# Changelog

Entries are per skill, newest first. Each skill is versioned on its own; see "Versions" in [`README.md`](README.md).

## video-integration-audit 0.1 (2026-10-28)

First public version. Detects the video platform in the code (Vonage Video API, OpenTok, TokBox, Agora, Stream Video; recognises Twilio Video, Zoom Video SDK, 100ms, Daily and LiveKit) and reports in one of four modes: migration, stream, hybrid or neither.

- General checks G1 to G12 (secrets, token minting and the token endpoint, join and device error handling, cleanup on leave, reconnects, server-enforced permissions, autoplay and permission prompts, cost-driving media settings, webhook verification), with the same verdict and severity for every vendor.
- Vonage Video (OpenTok and unified) and Agora (3.x and 4.x) web references, with the Stream equivalents pinned to the SDK versions named in each file.
- Stream Video web best-practice checks S1 to S30.
- Output contract: fifteen sections by mode plus a machine-readable `video-integration-audit/v1` JSON block.
- Honesty and referral rules: other vendors described as their documentation does, no pricing or cost comparisons, Stream's limitations stated plainly, requirements the code cannot settle referred to sales.

The test apps and recorded runs the skill is evaluated against are kept outside this repository.

## stream-video-webview-audit 0.5 (2026-10-28)

First public version. Checked against `@stream-io/video-react-sdk` 1.44.1 (`@stream-io/video-client` 1.63.0).

- Section A: vendor-independent checks for any WebRTC SDK inside a WebView (Android foreground service, keeping the screen on, no reloads or recreated pages, configuration changes, WebView pause and timers, one call-screen instance, renderer or content-process death, autoplay policy, microphone permission, edge-to-edge insets, the iOS audio session, origin checks on the native bridge).
- Section B: checks specific to Stream's React video SDK (iOS background media handling, client and call lifetime, page lifecycle events, autoplay-blocked audio, host event logging, screen-share availability in WebViews, minimum SDK version).
- References with Kotlin and Swift snippets for the Android and iOS hosts.
