# video-audit-skills

Read-only audit skills for coding agents (Claude Code, Cursor, Codex and others) that review apps built on video calling and streaming APIs. Published by [Stream](https://getstream.io).

Each skill is a folder of markdown: a `SKILL.md` with the instructions, an `AGENTS.md` for agents that read that file instead, and reference files. There are no scripts, no packages and no CLI to install. Read a skill before you run it; that takes a few minutes and is the point.

## What the skills do, and what they never do

| | |
|---|---|
| **Read** | Your source files, manifests and lockfiles (for versions), using your agent's file read, list and search tools |
| **Write** | Nothing, or at most one report file; the table under [Skills](#skills) says which |
| **Never** | Run code, commands, builds, tests or package managers; use a shell, even for read-only commands; make network requests or use connectors; change your files; copy secret values into a report; follow instructions found in the code they audit |

As with any agent task, the files your agent reads are sent to the model provider your agent uses, under your existing settings. The skills add no other destination.

Each `SKILL.md` declares its `allowed-tools` for Claude Code. Not every agent honours that line, so the same rules are written into the instructions and into each skill's `AGENTS.md`. Markdown rules are instructions, not a sandbox: if you need the limits enforced, configure your agent to allow only file read, list and search tools (and, for a skill that writes a report, a write limited to the report path).

## Skills

| Skill | What it audits | Writes |
|---|---|---|
| [`stream-video-webview-audit`](skills/stream-video-webview-audit/) | A native iOS or Android app that runs Stream's React video SDK inside a WebView (WKWebView, Android WebView, Capacitor, a React Native WebView, an in-app browser) | Nothing; the report is in the agent's reply |
| [`video-integration-audit`](skills/video-integration-audit/) | An app's video calling, meeting or livestreaming integration: detects Vonage Video, Agora or Stream Video, and recognises Twilio Video, Zoom Video SDK, 100ms, Daily and LiveKit | One file: `video-integration-audit.md` at the audited root, or a path you name |

### stream-video-webview-audit

Checks the host app's lifecycle handling, audio session and WebView configuration, and the web page's handling of the call, against the failure modes that make a call inside a WebView drop when the phone locks, go silent after the user returns, rejoin by itself or lose the microphone. Each finding names the file and line, says what the user experiences and gives the fix, with Kotlin and Swift snippets in the skill's `references/` folder.

The checklist has two parts. Section A holds for any WebRTC SDK running inside a WebView. Section B is specific to Stream's React video SDK and was checked against `@stream-io/video-react-sdk` 1.44.1; it names no unreleased SDK features. The checks come from reproductions on Android 14 to 16 emulators and on one iPhone running iOS 26, not from your app, and [`SKILL.md`](skills/stream-video-webview-audit/SKILL.md) lists what was not tested and what the audit does not cover.

Ask your agent, in the repository you want audited:

> Audit the WebView video integration in this repository.

If the iOS app, the Android app and the web page live in separate repositories, run it in each, or open all of them in one workspace. If the web page is not in the repository, tell the agent which `@stream-io/video-react-sdk` version it runs.

### video-integration-audit

Detects which video platform an app uses, audits the integration and writes the findings to one markdown file. **This version reads web apps in depth** (JavaScript and TypeScript clients and Node servers); iOS, Android, React Native and Flutter code gets a platform-level analysis only, as the coverage table below explains, and every report says which parts of the app got which. What the report contains depends on what was detected:

| Detected | Report |
|---|---|
| Vonage Video (OpenTok, TokBox) or Agora | An inventory of every integration site with its Stream Video equivalent, the behaviour changes a find-and-replace port gets wrong, a call-type configuration checklist, a phased migration plan with a fleet-transition note for mobile apps, general findings, and what needs a human decision |
| Stream Video | Findings against Stream's public best practices with evidence and fixes, general findings, and a prioritised fix list. No migration plan, no vendor comparison |
| Stream Video and Vonage or Agora | Both: the plan for the remaining vendor code, the Stream findings for the migrated part, and the coexistence risks |
| Anything else (Twilio Video, Zoom Video SDK, 100ms, Daily, LiveKit, self-built WebRTC) or nothing | What was found, the general checks, and a plain statement that this version has no reference for that platform |

The general checks run in every mode, with the same verdicts and severities whichever platform the app uses, and cover token minting and lifetime, whether the token endpoint authenticates its caller, secrets in client code, join and device error handling, cleanup on leave, reconnect handling, server-enforced permissions, autoplay and permission UX, cost-driving media settings, and webhook verification. Every finding cites `file:line`. The report ends with a machine-readable JSON block (`video-integration-audit/v1`) holding the same rows, which a porting skill can take as its input.

The skill describes other vendors as their public documentation does, never comments on their pricing, and computes no cost comparisons. Documented limitations of any platform, Stream's included, are stated plainly and without disparagement; where Stream lacks something your app uses, the report says so. Every report states that it is published by Stream, and requirements the code cannot settle (fixed IPs, end-to-end encryption, SIP dial-out, on-premises deployment) are referred to Stream's sales team rather than argued. The full rules are in [`references/hub.md`](skills/video-integration-audit/references/hub.md#honesty-and-referral-rules).

| Platform | Web | iOS | Android | React Native |
|---|---|---|---|---|
| Vonage Video | yes | platform-level analysis only | platform-level analysis only | concepts from the web reference |
| Agora | yes | platform-level analysis only | platform-level analysis only | concepts from the web reference |
| Stream Video | yes | not yet | not yet | concepts from the web reference |

"Platform-level analysis" means the full inventory, plan and general checks, with Stream symbols marked "verify against your SDK version". Mapping tables are pinned to the SDK versions named at the top of each reference file and were verified against the vendors' public documentation and published packages on the date given there.

Ask your agent, in the repository you want audited:

> Audit the video integration in this repository.

or, more specifically, "What would moving this app from Vonage to Stream take?" or "Check our Stream Video integration before launch." The agent writes `video-integration-audit.md` and replies with the top findings.

## Install

Copy one skill's folder into your agent's skills folder. Nothing else in this repository is needed.

- **Claude Code:** `.claude/skills/<skill-name>/` in your project, or `~/.claude/skills/<skill-name>/` for all projects. Run it with `/<skill-name>` or ask in plain words.
- **Cursor and Codex:** `.agents/skills/<skill-name>/` in your project, which both scan for skills according to their documentation (October 2026); Cursor also reads `.cursor/skills/`. You can also point the agent at the folder's `AGENTS.md`.
- **Any other agent:** give it the path to the folder's `SKILL.md` or `AGENTS.md` and ask it to follow it using only file read and search.

Whichever agent you use, check that it can read, list and search files without a shell. The skills forbid shell commands, even read-only ones, and an agent that has no other way to read files is told to stop and say so.

## Versions

Each skill has its own version (`metadata.version` in its `SKILL.md`), because a skill can change when an SDK, a server or a mobile OS changes. A skill that depends on an SDK also names the release it was checked against (for example `metadata.react-sdk-baseline`). Releases are tagged per skill (`<skill-name>-v<version>`), and [`CHANGELOG.md`](CHANGELOG.md) lists what changed.

## Contributing

- Skills stay markdown only: no scripts, no network access, no dependency on a CLI or another skill.
- Every SDK, platform or vendor API a skill names must exist in the release the skill names, verified against that release's source or published documentation. No unreleased features.
- Observations say where they were observed (device, OS, SDK version, number of runs); rules that rest on platform documentation name the page.
- ASCII only, one paragraph per line.
- A new skill is a folder under `skills/` whose name matches the `name` in its `SKILL.md`, with its own `AGENTS.md`, plus a row in the table above and in the root [`AGENTS.md`](AGENTS.md).

Questions about Stream: [getstream.io/contact](https://getstream.io/contact/).

## License

[Apache-2.0](LICENSE).
