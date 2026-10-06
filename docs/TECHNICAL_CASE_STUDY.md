# Technical Case Study

This document describes how Volume Booster handles audio on each platform. Statements come from reading the production project's Dart, Kotlin and Swift code and its manifests; nothing was run or measured for this write-up, and no performance or loudness figures are claimed.

## 1. The problem

A phone's volume ceiling is set by the OS and the hardware. An app can do three honest things about it: reshape the frequency response, change digital gain before the output stage, and attach platform effects where the OS allows it. What it cannot do is make a speaker physically louder or work around a platform's sandbox. The app is built around those limits, and the two platforms allow very different things.

## 2. Shape of the app

- **Shared Flutter layer.** Screens, `provider` controllers, persistence, ads, billing and analytics are written once in Dart (about 27k lines excluding generated localisation).
- **Native audio layers.** Kotlin (about 1.8k lines) and Swift (about 950 lines) implement the audio behaviour behind method channels.
- **One channel contract.** The Dart `AudioEngine` facade calls the same methods on both platforms: capability query, equalizer on and bands, bass boost, virtualizer, loudness gain. Each platform answers in its own way.

## 3. Android audio

### What is used
The native engine uses four classes from `android.media.audiofx`:

| Class | Driven by |
|---|---|
| `Equalizer` | Band levels from the 10-band curve, resampled onto the device's bands |
| `BassBoost` | Bass dial or Sub Bass, strength 0 to 1000 |
| `Virtualizer` | Virtualizer dial, strength 0 to 1000 |
| `LoudnessEnhancer` | Boost dial plus automatic compensation, target gain in milliBel |

All four are created on audio session 0, the global output mix, rather than on a session owned by the app's own player. The in-app player (`just_audio`) is one source that passes through that mix; other apps' output passes through it too, but whether an effect on session 0 is honoured depends on the Android version, the OEM audio stack and the device, so the app does not assume it. That is why every effect is probed first.

### Capability probing
`getCapabilities` creates each effect briefly inside a try block and reports whether it exists, along with the equalizer's band count, level range and centre frequencies. If creation throws (emulators and some OEM builds), the feature reports unavailable instead of crashing. Dart then skips the calls it cannot make.

### From 10 bands to the device's bands
The UI curve has ten fixed centre frequencies. The facade interpolates it in log-frequency space at each frequency the device reports, converts to the device's milliBel range and sends the levels down. The comments in the presets file explain that devices commonly expose about five bands, so preset gains were widened and placed where the resampled bands land.

### The boost dial and gain compensation
The dial runs 100 to 200 percent. 100 maps to zero gain; 200 maps to a 2000 mB target on the loudness enhancer. Enabling the equalizer, bass boost or virtualizer attenuates the signal to make headroom, so the equalizer controller adds a compensation term (a base cost for the EQ plus a share proportional to bass and virtualizer strength). The two terms are summed and clamped to 3000 mB total. Beyond that clamp and per-effect strength clamping, there is no limiter, normaliser or clipping detector in the app code, so pushing gain high can distort.

### Staying alive
Effects are native objects in the app process. A singleton `AudioEffectEngine` owns them, and `AudioEffectService`, a foreground service of type `mediaPlayback`, keeps the process alive while at least one effect is enabled. It shows an ongoing notification with a Stop action. Stopping releases every effect and sets a "stopped by user" flag so the Flutter UI reconciles itself on the next launch instead of silently re-enabling the booster.

Hard-won details in that service:
- `startForeground()` is not called in `onCreate`. When the system revives a sticky service in the background on Android 12 and later, promotion throws and used to kill the process within seconds. Promotion happens in `onStartCommand`, where a failure stops the service quietly and records a non-fatal.
- The boot receiver restarts the booster only if the persisted state says an effect was active, and not at all on Android 15 and later, where starting that service type from a boot broadcast is disallowed.
- Effect state is mirrored to native `SharedPreferences` on every change, so `restore()` can rebuild the chain in a fresh process before Flutter is running.

### Other Android-only features
- **Media sessions.** `MediaSessionManager` through a notification-listener component reads title, artist, position and state from another app's session and sends play, pause, next, previous and seek. It works only after the user grants Notification access, and the code rate-limits rebind attempts.
- **Device library.** A MediaStore scan, off the main thread, feeds the All Songs list.
- **Edge Lighting.** A foreground service of type `specialUse` hosts an overlay window. A `Visualizer` on session 0 captures the waveform, the service reduces it to an RMS level and the view pulses with it. Capture pauses when the screen turns off, and without microphone permission the glow animates without the pulse.

## 4. iOS audio

iOS does not let an app apply effects to other apps' audio, and the iOS code is written around that. Everything the user hears through the effects is the app's own player.

### The graph
`AudioEngineIos` owns an `AVAudioEngine` with this chain:

```
player node -> EQ (10 parametric bands + 1 low shelf) -> reverb -> main mixer -> output
```

| Control | iOS implementation |
|---|---|
| Equalizer | `AVAudioUnitEQ` bands, gain from the millibel value divided by 100 |
| Bass boost | Low-shelf band at 90 Hz, up to 12 dB |
| Virtualizer | Room reverb, wet mix 0 to 60 percent (an approximation of spatial widening) |
| Boost dial | Mixer output volume, 1.0 at 100 percent up to 2.0 at the top; a code comment notes the top end can clip |

Capabilities are reported as always available, in the same shape as Android's reply, so the shared Dart code needs no branches.

### The player
Playback runs through `AVAudioPlayerNode` rather than `just_audio`, because only an engine graph can host the EQ. Dart polls state every 400 ms over a second channel. Detail worth noting: EQ band frequencies are clamped below 45 percent of the file's sample rate, because a parametric band at or above Nyquist makes `AVAudioEngine` raise a fatal exception, and some picked MP3s use low sample rates.

### Session, background and interruptions
The session category is `.playback`, with the audio background mode declared in the app's configuration. The engine listens for interruptions, route changes, backgrounding and activation. On an interruption it records whether playback was running, pauses, and resumes when the interruption ends. On route change and lifecycle events it checks the engine is running and restarts it if needed. There are no lock-screen or Control Center transport controls.

### Not available
Effects on other apps' audio, other apps' media control, the overlay, the visualizer and the device-wide library scan do not exist on iOS. The Edge Lighting tab is hidden; the in-app "spectrum" is a drawn animation, not analysis of the audio.

## 5. Shared services

### Persistence
`shared_preferences` holds equalizer state, presets, playlists, user tracks, favourites and settings. The boost level restores on launch only when Booster Memory is on.

### Ads
A mediation SDK serves every format (app-open, banner, native, interstitial, rewarded). A local fork of its Flutter plugin adds a preload cache so banner and native ads can be warmed before their widgets mount. Placement keys are read from a remote Firestore document. The service fails closed: until one read succeeds, every placement is off, and afterwards a placement shows only if its field is literally `true`. A global kill switch and a live listener apply changes without a restart. A Pro entitlement turns all formats off. Two of three demo tracks are gated behind a rewarded ad.

### Billing
`in_app_purchase` on both stores, with a monthly and a yearly product and a trial offer. On Android a model collapses the base-plan and trial entries so the paywall shows the renewal price. The entitlement is cached locally.

### Analytics and diagnostics
An event facade validates events, queues them and tags them with session context. Every audio, media and overlay channel call reports failures to Crashlytics as tagged non-fatals; the media-session poll, which runs about once a second, is rate-limited to one report per five minutes. Performance traces and messaging are wired through Firebase.

### Consent
On iOS the App Tracking Transparency request is triggered natively when the app becomes active, because a Dart-only request could resolve without showing a prompt on a cold launch. Analytics collection waits for it to resolve.

## 6. Testing and build

Six test files with 25 test cases cover ad request throttling and cooldowns, analytics event validation, ad event naming, crash-reporting initialisation and a widget smoke test. There are no automated tests for the audio paths. The Android release build enables R8 code and resource shrinking.

## 7. Limits stated plainly

- iOS effects touch the in-app player only.
- Android effects target the global output session; reach into other apps is device-dependent and was not verified for this write-up.
- No limiter or clipping protection beyond the gain cap.
- Neither platform changes speaker hardware output.
- No claim about hearing, loudness measurements or safe listening levels.
