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

### stream-video-webview-audit

Checks the host app's lifecycle handling, audio session and WebView configuration, and the web page's handling of the call, against the failure modes that make a call inside a WebView drop when the phone locks, go silent after the user returns, rejoin by itself or lose the microphone. Each finding names the file and line, says what the user experiences and gives the fix, with Kotlin and Swift snippets in the skill's `references/` folder.

The checklist has two parts. Section A holds for any WebRTC SDK running inside a WebView. Section B is specific to Stream's React video SDK and was checked against `@stream-io/video-react-sdk` 1.44.1; it names no unreleased SDK features. The checks come from reproductions on Android 14 to 16 emulators and on one iPhone running iOS 26, not from your app, and [`SKILL.md`](skills/stream-video-webview-audit/SKILL.md) lists what was not tested and what the audit does not cover.

Ask your agent, in the repository you want audited:

> Audit the WebView video integration in this repository.

If the iOS app, the Android app and the web page live in separate repositories, run it in each, or open all of them in one workspace. If the web page is not in the repository, tell the agent which `@stream-io/video-react-sdk` version it runs.

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
