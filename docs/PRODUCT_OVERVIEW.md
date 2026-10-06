# Product Overview

Volume Booster is a sound-tuning app for Android and iOS. It combines a boost dial, a 10-band equalizer with presets, bass and virtualizer controls, and a built-in music player so the user can hear the effect of each change.

What the app does differs by platform, and this document says so wherever it matters. The short version: on Android the effects go through the platform audio-effect engine; on iOS they apply to the app's own player only, because iOS does not let an app process other apps' audio.

## Who it is for

People who want to adjust the tone or output gain of their phone's audio, preview the change on their own music, and keep the setting in place while they use other apps (Android) or while the app plays in the background (iOS).

## Navigation

The home screen has a fixed bottom bar. Each tab keeps its state when the user switches away.

| Tab | Android | iOS |
|---|---|---|
| Home | Boost dial, system volume slider, boost presets | Same |
| Edge Lighting | Overlay settings | Hidden |
| Equalizer | Dials, presets, band sliders | Same |
| Songs | Playlists, search, device-wide library, imported files | Playlists, search, imported files |
| Favorites | Favourited songs | Same |
| Settings | Preferences, language, tutorial | Same |

## Features

### Boost dial
A dial from 100 to 200 percent (default 140) with 125, 150, 175 and Max shortcuts. 100 percent means no boost. The dial is a user-friendly scale over a gain setting, not a measured loudness: on Android it becomes a loudness-enhancer target gain of up to 2000 mB, on iOS a mixer gain of up to 2.0 on the in-app player. A separate System slider is kept in step with the device media volume and the hardware volume buttons.

### Equalizer
- Ten bands from 31 Hz to 16 kHz, with a vertical slider per band.
- Twenty presets: Default, Normal, Flat, Bass, Treble Boost, Vocal Boost, Loud, Headphones, Pop, Rock, Hip Hop, R&B, Jazz, Classical, Dance, Electronic, Acoustic, Latin, Piano and Country.
- Moving a band deselects the preset and shows Custom.
- Two dials for Bass Boost and Virtualizer.
- Sub Bass, a one-tap deep-bass mode. It is mutually exclusive with the equalizer.

### Music player
- Imported files (file picker on both platforms) and, on Android, a scan of the device's audio library.
- Three demo tracks hosted remotely and cached after the first play; two of them sit behind a rewarded ad for non-Pro users.
- Playlists, favourites, queue with reorder and swipe-to-remove, shuffle, repeat with a bounded repeat count.
- A mini player on every tab and a full Now Playing screen.

### Edge Lighting (Android only)
A glow along the screen edges drawn in an overlay window. Colour, speed, thickness, corner radius, notch style and a link-to-bottom-corner option are adjustable. When the microphone permission is granted, the overlay pulses with the captured output level.

### External media control (Android only)
After the user grants Notification access, the mini player can show and control another app's current playback (play or pause, next, previous, seek).

### Settings
Preferred music player, vibration intensity (off, default, medium, high), tap sounds, a music-spectrum toggle for the in-app waveform animation, Booster Memory, language, tutorial replay and a reset option.

### Pro subscription
A monthly and a yearly plan with a seven-day trial offer; the paywall lists no ads, uninterrupted playback and support for future updates.

### Onboarding and tutorial
A language chooser, a five-screen onboarding, and a coach-mark tutorial that can be replayed from Settings.

## Languages

English, Arabic, German, French, Hindi, Italian, Japanese, Portuguese, Urdu and Chinese.

## What the app does not do

- It does not change what a speaker or headphone can physically output, and it makes no claim about a safe maximum volume.
- It does not apply effects to other apps' audio on iOS.
- It does not guarantee that effects on the global output session reach every app on every Android device; support varies by device and OEM, which is why the app probes each effect before using it.
- The app has no sign-in or user account.

## Screens

Real Android screenshots are in the [README](../README.md). The iOS build was not captured for this showcase.
