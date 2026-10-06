# Engineering Challenges

Seven problems that stand out in the production code. Outcomes describe observable behaviour in the code; no metrics are claimed.

---

## 1. One control surface over two unequal audio platforms

**Problem**
Android lets an app attach effects to the output mix. iOS does not let an app touch other apps' audio at all. The product shows the same Equalizer and Boost screens on both, so the shared code must not imply the same reach.

**Context**
The shared layer is Flutter; the audio work has to be native on both sides.

**Engineering approach**
- A single method-channel contract (capabilities, equalizer, bass, virtualizer, loudness) is implemented in Kotlin and in Swift.
- Android maps it to `Equalizer`, `BassBoost`, `Virtualizer` and `LoudnessEnhancer`.
- iOS builds an `AVAudioEngine` graph and maps the same values onto EQ bands, a low-shelf, a reverb mix and mixer gain, for the in-app player only.
- Android-only calls return harmless values on iOS, and platform-only UI (the Edge Lighting tab, device-wide library, external media control) is hidden or skipped instead of faked.

**Trade-offs**
Two implementations double the native surface, and the iOS virtualizer is an approximation (reverb), not the same effect. The benefit is that the Dart controllers have no per-platform audio branches.

**Outcome**
One set of controllers and screens, with every difference living in native code and written down in the Android vs iOS table.

---

## 2. Keeping Android effects alive without crashing the process

**Problem**
Effect objects belong to the app process. If the process is killed, the effects disappear, and the user hears the change vanish. Keeping the process alive needs a foreground service, and Android restricts when one may start: from the background on Android 12 and later, and from a boot broadcast on Android 15.

**Context**
A sticky service is revived by the system in the background, which is exactly where promotion is refused. Promoting in `onCreate` would turn that refusal into an uncaught exception that kills the process, so the code deliberately avoids it.

**Engineering approach**
- A singleton engine owns the effects; a `mediaPlayback` foreground service keeps the process alive and exposes a Stop action.
- Promotion moved to `onStartCommand`, where a refusal returns a not-sticky stop and records a non-fatal instead of throwing.
- Every parameter change is mirrored to native preferences, so a fresh process can rebuild the chain before Flutter starts.
- The boot receiver restarts the booster only if saved state says it was active, and skips Android 15 and later, where the app restores state on the next launch.
- A "stopped by user" flag stops Flutter from re-enabling something the user just turned off in the notification.

**Trade-offs**
On Android 15 and later the booster does not come back after a reboot until the user opens the app. A persistent notification is the price of staying alive.

**Outcome**
A refused service start degrades to a stopped service with preserved state, not a crash.

---

## 3. Ten bands in the UI, a handful in the hardware

**Problem**
The interface presents ten bands, but Android's equalizer reports its own band count and centre frequencies. A developer-facing note in the presets file records client feedback that presets sounded the same.

**Context**
Devices commonly expose about five bands, so subtle multi-band shapes get flattened.

**Engineering approach**
- The facade interpolates the ten-band curve in log-frequency space and samples it at each centre frequency the device reports, then scales into the device's own level range.
- Preset gains were widened and placed so that their character survives at the resampled bands.
- Because the effects reduce headroom, a compensation term (a base cost for the equalizer plus shares tied to bass and virtualizer strength) is added to the loudness gain, and the total is capped.

**Trade-offs**
Wider presets trade subtlety for audibility. Compensation is tuned by ear and by constants in code, not measured, and the cap is a blunt guard against distortion rather than a limiter.

**Outcome**
Presets are designed to stay distinguishable across devices with different band layouts, and the compensation term is meant to keep loudness from dropping when effects are switched on. Neither was measured for this write-up.

---

## 4. A fatal audio-engine exception from a valid file (iOS)

**Problem**
Files chosen from Files or iCloud can be 22.05 or 32 kHz MP3s. A parametric EQ band at or above the Nyquist frequency makes `AVAudioEngine` raise a fatal CoreAudio exception.

**Context**
The graph is rebuilt at each file's own format so the player node and EQ agree on sample rate.

**Engineering approach**
When connections are rebuilt, each band frequency is clamped to 45 percent of the file's sample rate before the engine restarts. Seek and pause record the position as a frame offset so playback can be rescheduled from any point, and a generation counter discards stale completion callbacks.

**Trade-offs**
Clamping moves the highest bands on low-rate files. Hand-rolled position tracking replaces a ready-made player, in exchange for hosting the EQ.

**Outcome**
Low-rate files are routed through the effect graph with safe band frequencies, and stale completion callbacks from earlier schedules are ignored.

---

## 5. Interruptions, routes and background on iOS

**Problem**
With an engine graph instead of a high-level player, the app owns what happens when a call arrives, headphones change, or the app goes to the background.

**Context**
The session category is `.playback` with the audio background mode declared.

**Engineering approach**
The engine observes interruptions, route changes, backgrounding and activation. On an interruption it remembers whether it was playing, pauses and, when the interruption ends, reactivates the session and resumes. On other events it checks the engine is running and restarts it, then reschedules playback if it was meant to be playing.

**Trade-offs**
There are no lock-screen or Control Center controls, so users must return to the app to control playback. Route changes only verify liveness; they do not pause on unplug.

**Outcome**
Playback survives backgrounding and interruptions without leaving a dead engine.

---

## 6. Ads that cannot appear by accident

**Problem**
Ads must be switchable per placement without shipping a release, and a failed config fetch must not turn them on.

**Context**
Ads cover app-open, banner, native, interstitial and rewarded formats across many placements, plus a Pro entitlement.

**Engineering approach**
- Placement flags live in one remote document with a global kill switch.
- The service is fail-closed: nothing shows until a read succeeds, and then only placements whose field is literally `true`. Missing, false or non-boolean values mean off.
- A live listener applies changes mid-session, and a connectivity retry covers offline starts.
- A local fork of the mediation plugin adds a preload cache so banner and native ads can be warmed before their widgets exist.
- Request throttling and cooldowns are covered by unit tests.

**Trade-offs**
Fail-closed means a user on a flaky connection sees fewer ads. A new placement needs an explicit remote flag before it ever shows. Maintaining a plugin fork adds upgrade cost.

**Outcome**
Offline, broken or missing configuration results in no ads.

---

## 7. A permission prompt that never appears (iOS)

**Problem**
Requesting App Tracking Transparency from Dart on a cold launch can reach iOS before the app is active, and iOS then resolves the request as not determined without showing a dialog.

**Context**
Analytics collection waits for the result, so a silent non-prompt also affects data handling.

**Engineering approach**
The request is fired natively when the app becomes active, independent of the Flutter engine, and also callable from Dart. Both paths share one in-flight request, so callers receive the same resolved status. The notification-based trigger is used because a scene-based app does not receive the app-delegate activation callbacks.

**Trade-offs**
Two entry points must share one source of truth. Failures fall back to non-personalised ads rather than blocking startup.

**Outcome**
The system prompt is raised reliably once per launch regardless of which path arrives first.
