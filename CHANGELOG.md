# Changelog

Entries are per skill, newest first. Each skill is versioned on its own; see "Versions" in [`README.md`](README.md).

## stream-video-webview-audit 0.5 (2026-10-28)

First public version. Checked against `@stream-io/video-react-sdk` 1.44.1 (`@stream-io/video-client` 1.63.0).

- Section A: vendor-independent checks for any WebRTC SDK inside a WebView (Android foreground service, keeping the screen on, no reloads or recreated pages, configuration changes, WebView pause and timers, one call-screen instance, renderer or content-process death, autoplay policy, microphone permission, edge-to-edge insets, the iOS audio session, origin checks on the native bridge).
- Section B: checks specific to Stream's React video SDK (iOS background media handling, client and call lifetime, page lifecycle events, autoplay-blocked audio, host event logging, screen-share availability in WebViews, minimum SDK version).
- References with Kotlin and Swift snippets for the Android and iOS hosts.
