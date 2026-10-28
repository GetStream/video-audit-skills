# iOS WKWebView host: what a calling app must do

Companion to `../SKILL.md`. Every item names the failure it prevents and what we observed. The observations come from a minimal WKWebView host app on one real iPhone (an iPhone 12 mini running iOS 26, loudspeaker route), with Stream's React video SDK 1.43.x and 1.44.0 running in the page and the background-handling checks (section 2) repeated on 1.44.1; a few checks ran on the iOS Simulator and are marked. Every Stream API named here exists in React SDK 1.44.1. "The user hears it" was judged by a microphone next to the phone, so nothing below is validated on headphones, Bluetooth or the receiver, and the counts are correlated samples from a few runs, not independent validations. Snippets are minimal Swift sketches, not complete files (enclosing classes and app helpers are left out), matching the host we verified (UIKit; the same calls apply from SwiftUI through a `UIViewRepresentable`). Where an item rests on an iOS rule, the public documentation page is named; the full list is at the end of `SKILL.md`.

## 1. WebView configuration for a call page

**What we observed.** A host that left `mediaTypesRequiringUserActionForPlayback` at its default (`.all` on iOS, per the `WKWebViewConfiguration` reference) got every remote audio element and the shared video paused until a tap (Simulator). `allowsInlineMediaPlayback` defaults to false on iPhone, which sends video full-screen. Set both before creating the WebView; they cannot be changed afterwards.

```swift
let config = WKWebViewConfiguration()
config.allowsInlineMediaPlayback = true
config.mediaTypesRequiringUserActionForPlayback = []
let webView = WKWebView(frame: .zero, configuration: config)
webView.navigationDelegate = self
webView.uiDelegate = self
```

Keep one WebView per call. If a deep link or a notification can push the call screen again while a call is live, route it to the existing instance; two live pages put the same user in the call twice and the user hears every speaker twice.

## 2. Background handling: do nothing; never `pauseAllMediaPlayback` on a call page

**What we observed.** The primary rule is to do nothing when the app goes to the background. With the app backgrounded for 5 s, 30 s, 2 min and 10 min, with or without the `audio` background mode in `UIBackgroundModes`, the call stayed audible throughout; WebKit kept the app alive while its media played. (The mode is still Apple's documented contract for background audio, so the audit treats a missing mode as NEEDS-REVIEW rather than irrelevant; see `SKILL.md` A11.) A host that called `webView.pauseAllMediaPlayback` when the app went to the background (a typical "save battery" line) left the call **silent after the user came back** on 1.43.x and 1.44.0. After a 15 s background it stayed silent in every run (46 s observed after the return), and the page looked healthy throughout: every audio element playing, `currentTime` advancing, packets arriving, audio levels normal. After 2 minutes in the background, audio came back about 19 s after the return, only because the SDK rejoined the call. On 1.44.1 the same host kept the call audible during and after the background in every run (iPhone 12 mini, iOS 26.6, 15 s and 2 min, three runs each), with no reconnect. So on 1.44.1 `pauseAllMediaPlayback` does not silence the call, and a host that calls it to get silence does not get it; on earlier releases it breaks the call. Neither is a reason to call it. The two APIs differ in the `WKWebView` reference: `pauseAllMediaPlayback(completionHandler:)` pauses the media and leaves it to the page to resume; `setAllMediaPlaybackSuspended(_:completionHandler:)` suspends playback and, when lifted, lets it continue.

Only a product that intends silence in the background (for example to let another app's audio through) should touch page media at all, and then with the suspend pair below, which was audible after return in every run with a sub-second reconnect (on 1.44.1: silent in the background, audible about 2 s after the return). A host that does nothing keeps other apps quiet: a video started in Safari while the call was in the background could not play, because the SDK took the audio back within about a second each time (1.44.0 and 1.44.1); with the suspend pair it played. Lift the suspension in `willEnterForeground` before anything else touches the WebView, and test on the device that audio returns. If the call screen goes away while the app is in the background, lift the suspension before removing the observers when the WebView will be reused, or destroy the WebView. A product that wants the call audible with the phone in a pocket must not use the pair either: it silences the background call by design.

```swift
private var suspendedByHost = false
private var lifecycleObservers: [NSObjectProtocol] = []   // remove them when the call screen goes away

func observeBackgroundForSilence() {   // once per call screen
    let nc = NotificationCenter.default
    lifecycleObservers.append(nc.addObserver(forName: UIApplication.didEnterBackgroundNotification, object: nil, queue: .main) { [weak self] _ in
        guard let self, self.callIsLive else { return }
        self.suspendedByHost = true
        self.webView.setAllMediaPlaybackSuspended(true)
    })
    lifecycleObservers.append(nc.addObserver(forName: UIApplication.willEnterForegroundNotification, object: nil, queue: .main) { [weak self] _ in
        guard let self, self.suspendedByHost else { return }   // lift it even if the call ended while in the background
        self.suspendedByHost = false
        self.webView.setAllMediaPlaybackSuspended(false)
    })
}
```

Also avoid recreating the WebView on background, which drops the user from the call, and `closeAllMediaPresentations` on a call page (we did not test it; it acts on the page's media without the page asking).

## 3. Keep the screen on during a call

A WebView keeps the screen awake only while a video element is playing. In an audio-only stretch the device's auto-lock timer applies. Set `UIApplication.isIdleTimerDisabled` (`UIApplication` reference) while the call screen is shown and clear it on leave. On iOS a locked screen did not end the call in our runs (lock for 30 to 45 s by hand: audible throughout), so this is a product choice here, unlike Android without a foreground service.

```swift
UIApplication.shared.isIdleTimerDisabled = true    // on call start
UIApplication.shared.isIdleTimerDisabled = false   // on leave
```

## 4. Rebuild on WebContent process death; never reload as a side effect

**What we observed.** Killing the WebView's content process (what jetsam does under memory pressure) fired `webViewWebContentProcessDidTerminate` within about a second; the view went blank (white in a default host) and audio stopped. The page cannot emit anything, it is gone. A host that reloads there was back in the call in about 2 s; a host that does nothing leaves the user on a blank screen while the other participants see a ghost for about 15 s (in our runs, until the server removed the participant). Reference: `WKNavigationDelegate.webViewWebContentProcessDidTerminate(_:)`.

Gate the reload on native call intent (`callIsLive` is false after a hang-up), bound it to 3 attempts with backoff, give the new page the microphone and camera state the host last knew when it sends `page.ready`, before it joins (a state the host does not know stays off until the user turns it on), and show an error after the last attempt. An unconditional reload loops on a page that crashes while loading and rejoins a call the user has left. Reset the counter when the page reports a successful join.

```swift
private var contentRestarts = 0

func webViewWebContentProcessDidTerminate(_ webView: WKWebView) {
    pageReady = false   // item 6: nothing is delivered until the reloaded page sends page.ready
    hostEvent("webcontent.terminated", ["restarts": contentRestarts])   // buffered (item 6): the page is gone, the next one gets it
    guard callIsLive, contentRestarts < 3 else { showCallEndedWithError(); return }
    contentRestarts += 1
    let delay = pow(2.0, Double(contentRestarts - 1))   // 1 s, 2 s, 4 s
    DispatchQueue.main.asyncAfter(deadline: .now() + delay) { [weak self] in
        guard let self, self.callIsLive else { return }
        webView.reload()   // the page rejoins after page.ready returns the microphone and camera state
    }
}
```

Reload **only** here and on an explicit user action. Do not reload in `viewWillAppear`, on `didReceiveMemoryWarning` or from a heartbeat watchdog; each of those drops and rejoins the call. `didFail navigation` and `didFailProvisionalNavigation` are main-frame navigation callbacks (there is no per-frame flag as on Android), so the rule there is: do not reload on `NSURLErrorCancelled` (a navigation the host or the page replaced), do not reload a navigation the host itself started (compare the `WKNavigation` returned by `load` with the one in the callback), and never in a loop; bound retries as above.

## 5. Microphone permission

Declare `NSMicrophoneUsageDescription` (and `NSCameraUsageDescription` if the page publishes video). If you implement `WKUIDelegate.webView(_:requestMediaCapturePermissionFor:initiatedByFrame:type:decisionHandler:)`, grant for the call page's origin so the user is not asked twice; without the delegate method WebKit shows its own per-origin prompt. Prompt for the system microphone permission natively before the page loads (`AVAudioApplication.requestRecordPermission` on iOS 17+), so the page's first request succeeds. This item is stated from the `WKUIDelegate` reference; the deny-then-grant failure was reproduced on Android, not on iOS.

```swift
func webView(_ webView: WKWebView, requestMediaCapturePermissionFor origin: WKSecurityOrigin, initiatedByFrame frame: WKFrameInfo,
             type: WKMediaCaptureType, decisionHandler: @escaping (WKPermissionDecision) -> Void) {
    // compare the full origin (protocol, host, port) and the frame, not the hostname alone
    let isCallPage = origin.protocol == "https" && origin.host == callPageHost
        && (origin.port == 0 || origin.port == 443) && frame.isMainFrame
    decisionHandler(isCallPage ? .grant : .deny)
}
```

## 6. Audio session: configure once, observe interruptions, tell the page

**What we observed.** WebKit manages the audio session for page media. A host that changed the category mid-call (`.playback`, `.playAndRecord` with `.voiceChat`, `.soloAmbient`) made WebKit pause every media element, and Stream's SDK (1.43.x and 1.44.0) resumed them within a second in all 21 runs; `.ambient` and `setActive(false)` had no effect. So mid-call changes are a smell rather than a failure with this SDK, with two side effects worth knowing: `.playAndRecord` + `.voiceChat` without `.defaultToSpeaker` routes the call to the earpiece, and `.mixWithOthers` changes how other apps' audio interacts with the call. If you set a category, do it once before the page loads.

What the page cannot reliably see, and the host can: an interruption that leaves the page visible (a phone call, Siri, an alarm; "Responding to audio session interruptions"; `navigator.audioSession` reported no state in our iOS 26 WKWebView), a route change (headphones, Bluetooth), a media-services reset, the reload from item 4, and the real OS version (a WKWebView on iOS 26 reports `iPhone OS 18_7` in its user agent). Forward them to the page with timestamps, so the page can write them to the app's own logs together with the call id and the local participant's session id. React SDK 1.44.1 already records page visibility in the call's traces by itself after a join (when stats reporting is enabled for the call); it has no API meant for events reported by the host, so these stay in your logs.

Send each event as one JSON object (`name`, `payload`, `timestamp` in Unix milliseconds, `os`), passed to the page as a string literal and parsed there; never splice the name or the payload into JavaScript source as text. The Android host in the companion reference uses the same shape. An event evaluated into a page that is suspended, not yet loaded or gone is lost, so keep the last 50 natively, deliver only after the current page has sent `page.ready`, and replay the buffer then. A new navigation (`didStartProvisionalNavigation`), a WebContent termination or replacing the WebView resets that; the sketch below already moves WebView calls to the main thread. Keep the observer tokens and remove them when the call screen goes away.

```swift
private var hostEventBuffer: [[String: Any]] = []
private var pageReady = false   // true after the current page's page.ready; false on navigation or WebContent termination

func hostEvent(_ name: String, _ detail: [String: Any] = [:]) {
    let event: [String: Any] = [
        "name": name,
        "payload": detail,
        "timestamp": Int(Date().timeIntervalSince1970 * 1000),
        "os": UIDevice.current.systemVersion,
    ]
    hostEventBuffer.append(event)
    if hostEventBuffer.count > 50 { hostEventBuffer.removeFirst() }
    if pageReady { deliver(event, to: "hostEvent") }   // otherwise buffered; replayed on the next page.ready
}

func replayHostEvents() { deliver(hostEventBuffer, to: "hostEventReplay") }   // on the page's "page.ready" message, after setting pageReady = true

private func deliver(_ value: Any, to function: String) {   // one event object, or the buffer as an array
    guard let data = try? JSONSerialization.data(withJSONObject: value),
          let json = String(data: data, encoding: .utf8),
          let literalData = try? JSONSerialization.data(withJSONObject: json, options: .fragmentsAllowed),
          let literal = String(data: literalData, encoding: .utf8) else { return }   // a quoted JS string literal
    DispatchQueue.main.async { [weak self] in   // WKWebView calls must happen on the main thread
        self?.webView.evaluateJavaScript("window.\(function) && window.\(function)(\(literal))")   // the page does JSON.parse
    }
}

func observeAudioSession() {
    let nc = NotificationCenter.default
    nc.addObserver(forName: AVAudioSession.interruptionNotification, object: nil, queue: .main) { [weak self] n in
        let raw = n.userInfo?[AVAudioSessionInterruptionTypeKey] as? UInt
        let began = raw.flatMap(AVAudioSession.InterruptionType.init) == .began
        self?.hostEvent(began ? "audio.interruption.began" : "audio.interruption.ended")
    }
    nc.addObserver(forName: AVAudioSession.routeChangeNotification, object: nil, queue: .main) { [weak self] _ in
        let outputs = AVAudioSession.sharedInstance().currentRoute.outputs.map { $0.portType.rawValue }
        self?.hostEvent("audio.route", ["outputs": outputs])
    }
    nc.addObserver(forName: AVAudioSession.mediaServicesWereResetNotification, object: nil, queue: .main) { [weak self] _ in
        self?.hostEvent("audio.mediaServicesReset")
    }
    nc.addObserver(forName: UIApplication.didEnterBackgroundNotification, object: nil, queue: .main) { [weak self] _ in
        self?.hostEvent("app.background")
    }
    nc.addObserver(forName: UIApplication.didBecomeActiveNotification, object: nil, queue: .main) { [weak self] _ in
        self?.hostEvent("app.active")
    }
}
```

If the page also sends messages to the host (`window.webkit.messageHandlers.callHost.postMessage(...)`), check `message.frameInfo.isMainFrame` and the full `message.frameInfo.securityOrigin` (`protocol`, `host`, `port`) against the call page's origin in `userContentController(_:didReceive:)` before acting on them, and keep native actions driven by native call state: a message is a request, not a command. Answer `page.ready` by setting `pageReady = true`, then `replayHostEvents()` and the current microphone and camera state.

## What did not break on iOS 26 in our runs

So you do not spend time on it: backgrounding for up to 10 minutes with or without the `audio` background mode (an observation on one device; the mode remains Apple's documented contract, see item 2); a system alert covering the app during the join; the device's thermal state forced to serious or critical; unmute / mute / unmute on the loudspeaker; a 90-minute call with speakers muting and shares restarting, including 10 minutes in the background; a hand lock of 30 to 45 s; Wi-Fi off for 15 to 20 s (audible about 2 s after the network returned). What was not tested and what this guide does not cover (Siri, phone calls, alarms, route changes, network handoff, authentication) is listed in `SKILL.md` section 1.
