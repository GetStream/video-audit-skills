# Android WebView host: what a calling app must do

Companion to `../SKILL.md`. Every item names the failure it prevents and what we observed. The observations come from a minimal native WebView host app on Android 14, 15 and 16 emulators (system WebView 113, 124 and 133), with Stream's React video SDK 1.43.x and 1.44.0 running in the page, and the background, freeze, network-loss and microphone-denial checks repeated on 1.44.1; every Stream API named here exists in 1.44.1. No physical Android phone was tested, so every timing below is an emulator observation from a few runs; verify on your main devices (OEM battery managers add their own restrictions). Snippets are minimal Kotlin sketches of the parts that matter, not complete files: imports, enclosing classes, resources and app helpers (such as `hasRuntimePermissionsFor` or `createWebViewAndLoadCallPage`) are left out. The APIs they use exist at compileSdk 35 with androidx.core 1.13.1, androidx.webkit 1.11.0, AndroidX lifecycle and Kotlin coroutines. Where an item rests on an Android rule, the public documentation page is named; the full list is at the end of `SKILL.md`.

## 1. Hold a foreground service for the whole call

**What we observed.** On Android 15 and 16 emulators, an app without a foreground service lost all network access about 5 to 6 seconds after the user locked the phone or switched apps; Android's network policy marked the app's uid as background-blocked. The same happened when the screen timed out by itself with nobody touching the phone. We did not find the cutoff documented as a numbered behaviour change, so treat the timing as an observation to confirm on your devices, not a platform rule. The call's connections dropped, other participants saw the user leave, and audio took several seconds to come back on return: on React SDK 1.44.1, 6 to 18 s after 32 to 74 s away, and after 95 s or more away 30 s or longer, or not within 40 s of the return (6 of 6 runs). On Android 16 the OS also froze the whole app after about a minute in the background. With a foreground service none of this happened in any case we tested: background, 8 minutes of screen-off, forced deep Doze, battery saver. Android 14 did not block the network in the same tests, but it froze a backgrounded app without a foreground service after about 4 minutes in another app, which also ends the call; the service is needed there too. The service held when the app was in Android's *restricted* standby bucket. It did **not** hold when the user had set the app's battery usage to *Restricted*: Android demoted it at once and stopped it about a minute into a screen-off, and the call died. Check `ActivityManager.isBackgroundRestricted()` when the user joins and, if true, explain that calls may stop with the screen off and offer the battery settings. Public references: "Foreground services", "Foreground service types", "Foreground service types are required" (Android 14), "Power management restrictions", "Cached apps freezer" (AOSP).

**How.** Start the service while the app is eligible to start a foreground service: the call activity visible, the runtime permissions granted. Whoever triggers the start (your native join flow, or a request from the page), check eligibility at that moment, because a request that arrives after the app has gone to the background fails ("Restrictions on starting a foreground service from the background"). `startForeground` can also fail inside the service, where a `try` around `startForegroundService` in the activity does not catch it, so the service handles its own failure, reports it, and stops itself, and the join does not complete as a background-capable call until the service has acknowledged that it is in the foreground, with a timeout. Stop the service when the user leaves the call. `mediaPlayback` plus `microphone` is the recipe we tested; `phoneCall` needs a Telecom `ConnectionService`, and the `camera` and `mediaProjection` types for background camera or screen capture are outside this guide.

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MICROPHONE" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<service android:name=".CallService" android:exported="false"
         android:foregroundServiceType="mediaPlayback|microphone" />
```

```kotlin
class CallService : Service() {
    override fun onBind(intent: Intent?): IBinder? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val nm = getSystemService(NotificationManager::class.java)
        nm.createNotificationChannel(NotificationChannel("call", "Call", NotificationManager.IMPORTANCE_LOW))
        val open = PendingIntent.getActivity(
            this, 0,
            Intent(this, CallActivity::class.java).addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP or Intent.FLAG_ACTIVITY_CLEAR_TOP),   // with singleTop, item 6
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE,
        )
        val n = NotificationCompat.Builder(this, "call")
            .setContentTitle("Call in progress")
            .setSmallIcon(R.drawable.ic_call)
            .setOngoing(true)
            .setContentIntent(open)
            .build()
        // Listen-only: mediaPlayback. A user who can speak needs the microphone
        // type, activated while the activity is visible (Android 14+ checks
        // while-in-use eligibility; adding it later from the background fails).
        val type = if (intent?.getBooleanExtra("mic", false) == true)
            ServiceInfo.FOREGROUND_SERVICE_TYPE_MICROPHONE or ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK
        else ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK
        try {
            ServiceCompat.startForeground(this, 1, n, type)
            CallServiceState.state.value = CallServiceState.Foreground
        } catch (e: Exception) { // ForegroundServiceStartNotAllowedException, SecurityException, InvalidForegroundServiceTypeException
            CallServiceState.state.value = CallServiceState.Failed(e)   // the activity logs it and tells the page (item 9)
            stopSelf()
        }
        return START_NOT_STICKY
    }
}

object CallServiceState {
    sealed interface State
    object Idle : State
    object Foreground : State
    data class Failed(val error: Throwable) : State
    val state = MutableStateFlow<State>(Idle)
}

// in the call activity, on join (activity visible), one join attempt at a time; the page's join waits for the result:
suspend fun startCallService(canSpeak: Boolean): Boolean {
    if (!lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) return false   // not eligible right now
    CallServiceState.state.value = CallServiceState.Idle
    try {
        ContextCompat.startForegroundService(this, Intent(this, CallService::class.java).putExtra("mic", canSpeak))
    } catch (e: Exception) {   // the start itself can be refused
        CallServiceState.state.value = CallServiceState.Failed(e)
        return false
    }
    val result = try {
        withTimeoutOrNull(3_000) { CallServiceState.state.first { it != CallServiceState.Idle } }
    } catch (e: CancellationException) {   // the join was abandoned while waiting
        stopService(Intent(this, CallService::class.java))
        throw e
    }
    if (result !is CallServiceState.Foreground) {
        stopService(Intent(this, CallService::class.java))   // a late start must not leave a call notification behind
        return false
    }
    return true
    // false: log it, report it through the bridge (item 9), and tell the user the call works only while the app is on screen
}
// on leave:
stopService(Intent(this, CallService::class.java))

// at join: background-restricted apps lose the service after about a minute of screen-off
val am = getSystemService(ActivityManager::class.java)
if (am.isBackgroundRestricted) showBatteryRestrictedHint()   // e.g. deep link to the app's battery settings
```

Notes:
- **Speakers need the microphone type.** With a playback-only service, a speaker who locked the phone stayed connected, but Android 15 silenced their microphone one second later (`AudioRecordingConfiguration.isClientSilenced()` turned true), so everyone heard silence. With the microphone type the mic was never silenced. Android 14 did not silence it in the same test.
- Declare the foreground-service types you use in the Play Console; the use-case declaration is required for target SDK 34+.
- Do not use the `phoneCall` type unless you integrate with Android's Telecom `ConnectionService`. Background camera and screen capture need their own types and are not covered here.
- If the service does not reach the foreground, the join must not proceed as a background-capable call: report it and offer the foreground-only mode, or the user finds out at the first screen lock.
- Do not toggle service types on every mute. Activate the microphone type once, while visible, for users who can speak.
- `POST_NOTIFICATIONS` keeps the ongoing "call in progress" notification visible on Android 13+, which doubles as the way back into the call.

## 2. Keep the screen on during a call

**What we observed.** The WebView keeps the screen on only while a video element is playing. With cameras off and no screen share, the device's normal 30 s to 2 min timeout applies. Whether that matters is a product choice: most calling apps keep the screen on during a visible call, an audio-only product may prefer to let it lock. It is not a remedy for item 1: without the service, the screen-off that follows the timeout ends the call (that finding belongs to item 1); with the service in place, a locked screen did not end the call in our emulator runs, except under the Restricted battery setting from item 1. Public reference: "Keep the screen on".

```kotlin
// while the call screen is shown and the call is live
window.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)
// on leave
window.clearFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)
```

(In React SDK 1.44.1 only the prebuilt embedded call and livestream components (`@stream-io/video-react-sdk/embedded`) request a Screen Wake Lock while joined, and a request can be refused; a page built from the other components or the client does not. The page can request one itself: `navigator.wakeLock.request('screen')` resolves in WebView 124 and 133. The host flag is the reliable answer.)

## 3. Never reload or recreate the call page as a side effect

**What we observed.** Each of these dropped the call and rejoined it: a few seconds of silence, and a leave and a join for everyone else.

- `webView.reload()` / `loadUrl()` in `onResume` or `onStart` reloads every time the user comes back.
- Reloading or destroying the WebView in `onTrimMemory()`. In our tests `TRIM_MEMORY_UI_HIDDEN` arrived about 1 s after every switch to another app when no foreground service was running. Treat trim callbacks as a hint to drop caches, not to rebuild the call.
- A "page is hung" watchdog that reloads when the page answers a heartbeat late. The page answers late whenever the OS has paused it. If you want a hung-page check, ask the page for the SDK's calling state instead of measuring ping latency.
- `onReceivedError()` without checking `request.isForMainFrame`. On API 23+ this callback reports every failed subresource ("WebViewClient.onReceivedError" reference): a failed image, a blocked analytics beacon, or a failed background `fetch` from the page during a network blip reloaded the whole call.

```kotlin
override fun onReceivedError(view: WebView, request: WebResourceRequest, error: WebResourceError) {
    if (!request.isForMainFrame) return   // subresource failures are the page's business
    // handle a failed page load here
}
```

- Activity recreation on configuration changes ("Handle configuration changes"). Entering and leaving picture-in-picture, dark mode and font scale each rebuilt an activity that did not declare them, along with its WebView and the call page. Either declare the changes the call screen handles itself, and handle them (the same WebView survives `onConfigurationChanged`; no re-creation of the WebView; declaring a change only moves the responsibility for it into the app):

```xml
<activity android:name=".CallActivity"
    android:configChanges="orientation|screenSize|smallestScreenSize|screenLayout|uiMode|fontScale|density|keyboard|keyboardHidden|navigation" />
```

or keep the WebView outside the recreated view hierarchy. The audit passes either approach, reports `supportsPictureInPicture` declared without these declarations and without a WebView held outside the hierarchy as FAIL, and other missing declarations as NEEDS-REVIEW; test PiP, split-screen and foldable posture changes on your devices.

## 4. Handle the renderer process dying

**What we observed.** When the WebView renderer was killed, a host without an `onRenderProcessGone` override crashed entirely (the platform default; "Managing WebView objects: Termination Handling API"). A host that returned `true` without rebuilding showed a white screen with no audio. Rebuilding and reloading rejoined the call in about a second on the emulator.

Gate the rebuild on native call intent (the user has not hung up), bound it to 3 attempts with backoff, pass the microphone and camera state native code last knew to the new page before it joins (a state native code does not know stays off until the user turns it on), and show an error after the last attempt. An unconditional rebuild loops on a page that crashes while loading and rejoins a call the user has left. Reset the counter when the page reports a successful join.

```kotlin
private var rendererRestarts = 0

override fun onRenderProcessGone(view: WebView, detail: RenderProcessGoneDetail): Boolean {
    pageReady = false   // item 9: nothing is delivered to a destroyed view
    container.removeView(view)
    view.destroy()
    hostEvent("renderer.gone", mapOf("crash" to detail.didCrash(), "restarts" to rendererRestarts))   // buffered, item 9
    if (!callIntendedLive || rendererRestarts >= 3) { showCallEndedWithError(); return true }
    rendererRestarts += 1
    val delayMs = 1_000L shl (rendererRestarts - 1)   // 1 s, 2 s, 4 s
    container.postDelayed({ if (callIntendedLive) createWebViewAndLoadCallPage(micOn = nativeMicOn, cameraOn = nativeCameraOn) }, delayMs)
    return true
}
```

## 5. Do not pause the WebView's timers during a call

**What we observed.** `WebView.pauseTimers()` (often copied into `onPause`) is process-wide: it pauses JavaScript timers for every WebView in the app (`WebView.pauseTimers()` reference), so audit every WebView in the app for it, not only the call page's. With it in `onPause` and the screen off, media kept arriving, but the call's signalling went quiet. The server dropped the session after about 30 seconds and audio stopped; the rejoin on screen-on took about 6 s.

`WebView.onPause()` is a different call. Per its reference it pauses extra processing (animations, geolocation) on a best-effort basis and does not pause JavaScript; we did not reproduce it as a call-killer in our runs, so the audit reports it at Medium rather than Blocker. Keep it off a live call page anyway: the guard below covers both.

```kotlin
override fun onPause() {
    super.onPause()
    if (!callInProgress) { webView.onPause(); webView.pauseTimers() }
}
override fun onResume() {
    super.onResume()
    webView.resumeTimers(); webView.onResume()
}
```

## 6. One call screen, one page

**What we observed.** When a notification opened a second instance of the call activity, both pages stayed in the call as the same user. The hidden one kept playing, and the user heard every speaker twice (it sounds like echo); everyone else saw them twice in the participant list.

Route notification and deep-link taps to the existing call screen: `launchMode="singleTop"` plus `FLAG_ACTIVITY_SINGLE_TOP` / `FLAG_ACTIVITY_CLEAR_TOP` on the `PendingIntent`, handled in `onNewIntent` ("Tasks and the back stack"). Avoid `FLAG_ACTIVITY_MULTIPLE_TASK` and `documentLaunchMode` for the call screen, and make sure a call screen that is finished also leaves the call, unless the product keeps the call alive in a minimised state by design; then one coordinator (a service or an app-level object) owns the call and every screen attaches to it instead of joining again. `singleTask` works too, but changes back-stack behaviour, so check your navigation.

```xml
<activity android:name=".CallActivity" android:launchMode="singleTop" ... />
```

```kotlin
override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    setIntent(intent)
    // already in the call: do not load the page again; update UI state if the intent carries something new
}
```

## 7. Microphone permission: ask natively first, then grant only what was asked

**What we observed.** If the page's microphone request was denied once (for example while the app was still showing Android's own permission dialog), Stream's SDK, on every version we tested up to and including 1.44.1, did not ask again in that page, and unmute did nothing until a reload. The host side:

```kotlin
// WebView settings for a call page
webView.settings.apply {
    javaScriptEnabled = true
    domStorageEnabled = true
    mediaPlaybackRequiresUserGesture = false   // default is true: remote audio would wait for a tap
}

private var pendingWebPermission: PermissionRequest? = null

override fun onPermissionRequest(request: PermissionRequest) {
    val origin = request.origin
    val fromCallPage = origin.scheme == "https" && origin.host == CALL_PAGE_HOST && origin.port == -1   // exact origin
    val wanted = request.resources.filter {
        it == PermissionRequest.RESOURCE_AUDIO_CAPTURE || it == PermissionRequest.RESOURCE_VIDEO_CAPTURE
    }.toTypedArray()
    if (!fromCallPage || wanted.isEmpty()) { request.deny(); return }
    // the matching runtime permissions (RECORD_AUDIO / CAMERA) were requested in the lobby;
    // if not granted yet, ask now and answer the web request after the result - never deny first and ask later
    if (hasRuntimePermissionsFor(wanted)) { request.grant(wanted); return }
    pendingWebPermission = request
    requestRuntimePermissions(wanted) { granted ->
        if (pendingWebPermission !== request) return@requestRuntimePermissions   // cancelled, replaced, or the page navigated: answer nothing
        pendingWebPermission = null
        if (granted && hasRuntimePermissionsFor(wanted)) request.grant(wanted) else request.deny()
    }
}

override fun onPermissionRequestCanceled(request: PermissionRequest) {
    if (pendingWebPermission == request) pendingWebPermission = null
}

// in the WebViewClient's onPageStarted for the call WebView: a stale request must not be answered after navigation
override fun onPageStarted(view: WebView, url: String, favicon: Bitmap?) { pendingWebPermission = null; pageReady = false }   // pageReady: item 9
```

Keep `MODIFY_AUDIO_SETTINGS` in the manifest. In our test host, removing it made the WebView's microphone fail with `NotReadableError: Could not start audio source`.

## 8. Edge-to-edge (apps targeting Android 15)

**What we observed.** At target SDK 35, Android 15 draws the app edge-to-edge ("Android 15 behavior changes: edge-to-edge enforcement"), so the top of the call page sat under the status bar and taps there went to the system. Apply the system-bar insets to the WebView's container so page controls at the top (the "tap to enable audio" prompt) and at the bottom stay tappable. If the page should draw under the bars instead, the page must use `env(safe-area-inset-*)` and the WebView must receive the insets.

```kotlin
ViewCompat.setOnApplyWindowInsetsListener(webViewContainer) { v, insets ->
    val bars = insets.getInsets(WindowInsetsCompat.Type.systemBars() or WindowInsetsCompat.Type.displayCutout())
    v.setPadding(bars.left, bars.top, bars.right, bars.bottom)
    WindowInsetsCompat.CONSUMED
}
```

## 9. Bridge: origin-checked, and the host reports what the page cannot see

If the page controls native features (starting the call service, keep-screen-on) through a JavaScript bridge, accept those calls only from the call page's origin and main frame. `addJavascriptInterface` cannot do that: the injected object is callable from every frame in the WebView, the receiving method cannot tell which frame called it, and reading the WebView's current URL does not authenticate the caller. Use `WebViewCompat.addWebMessageListener` with an exact origin rule (scheme, host and port) and check `isMainFrame` on each message. Keep the actual service start and stop in native code driven by native call state: a message from the page is a request, and the activity acts on it only if its own state agrees.

```kotlin
const val CALL_PAGE_ORIGIN = "https://call.example.com"   // scheme + host (+ port if not the default), no path

if (WebViewFeature.isFeatureSupported(WebViewFeature.WEB_MESSAGE_LISTENER)) {
    WebViewCompat.addWebMessageListener(webView, "callHost", setOf(CALL_PAGE_ORIGIN)) { _, message, sourceOrigin, isMainFrame, _ ->
        if (!isMainFrame || sourceOrigin.toString() != CALL_PAGE_ORIGIN) return@addWebMessageListener
        when (message.data) {
            "call.active" -> if (callInProgress) ensureCallServiceStarted()
            "call.ended" -> if (callInProgress) endCallFromPage()   // native state decides what "ended" means
            "page.ready" -> { pageReady = true; replayHostEvents() }
        }
    }
}
```

In the other direction, tell the page what only native code sees, so that a complaint can be matched to a cause later: app background / foreground, the renderer restart from item 4, the service failing to reach the foreground from item 1, audio focus loss, and the real OS version. Send each event as one JSON object (`name`, `payload`, `timestamp` in Unix milliseconds, `os`) passed as a string literal, never spliced into JavaScript source as text; the iOS host in the companion reference uses the same shape, so one page handler serves both. An event evaluated into a page that is suspended, not yet loaded or gone is lost, so keep the last 50 natively, deliver only after the current page has sent `page.ready`, and replay the buffer then (above). A new navigation, a renderer death or replacing the WebView resets that. Do all of this on the main thread. The page writes them to the app's own logs with timestamps, the call id and the local participant's session id. After a join, React SDK 1.44.1 records page visibility and `freeze` / `resume` in the call's traces by itself (when stats reporting is enabled for the call), but has no API meant for events reported by the host.

```kotlin
private val hostEvents = ArrayDeque<JSONObject>()
private var pageReady = false   // true after the current page's page.ready; false on navigation or renderer death

fun hostEvent(name: String, detail: Map<String, Any?> = emptyMap()) {
    val event = JSONObject()
        .put("name", name)
        .put("payload", JSONObject(detail))
        .put("timestamp", System.currentTimeMillis())
        .put("os", Build.VERSION.RELEASE)
    if (hostEvents.size >= 50) hostEvents.removeFirst()
    hostEvents.addLast(event)
    if (!pageReady) return   // buffered; replayed on the next page.ready
    // the page's window.hostEvent takes one JSON string and parses it
    webView.evaluateJavascript("window.hostEvent && window.hostEvent(${JSONObject.quote(event.toString())})", null)
}

private fun replayHostEvents() {
    val batch = JSONArray(hostEvents.toList()).toString()
    webView.evaluateJavascript("window.hostEventReplay && window.hostEventReplay(${JSONObject.quote(batch)})", null)
}
```

## Call-page web app: keep the call alive across in-app navigation

Not a host item, but the host engineer is often the one who finds it: if the page's video client or call object is owned by a component that unmounts during a call (the call route itself, a conditionally rendered wrapper), an unmount whose cleanup disconnects or leaves (the usual effect pattern) tears it down and a remount recreates it inside the same page, which is a leave and a rejoin for everyone; without such a cleanup, a remount can join a second time or leave a call that no component owns. Keep the client and the call object above the route that renders the call (an app-level store or provider, scoped to the user session), and leave only on an explicit hang-up or a navigation away that the product means to end the call. This applies only while the same page and WebView stay alive; a destroyed WebView (items 3 and 4) always drops the user from the call, and the new page is back in it only if it runs the app's join flow.
