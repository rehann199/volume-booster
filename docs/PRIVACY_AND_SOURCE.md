# Privacy and Source

## What this repository is

A portfolio showcase for a production application. It describes the engineering of Volume Booster; it is not a source distribution.

## What is published

- Written descriptions of architecture, design decisions and trade-offs
- Original diagrams created for this showcase
- Real screenshots from the Android app
- The app icon, resized
- Public store links for the Android and iOS listings

## What is intentionally not published

- Production source code, in whole or in part
- Build and signing configuration, certificates and keystores
- Firebase and Google service configuration, API keys and tokens
- Ad network account, application and zone identifiers
- Billing product identifiers
- Remote-config and security-rule contents, and internal endpoints
- Version control history of the production repository
- Analytics exports or any user data

## What the app handles, as seen in the code

This section describes behaviour visible in the production project. It is not the app's privacy policy; the policy linked from the store listings is the authoritative statement.

- **No accounts.** There is no sign-in. Settings, equalizer state, playlists and favourites are stored on the device.
- **Local audio.** Files the user imports, and on Android the device's audio library, are read to play them. The app's audio processing happens on the device.
- **Permissions.** Android requests notification and audio-library access at first use, microphone access only when Edge Lighting is turned on (the overlay reads the output level through the platform visualizer), Notification access only if the user opts into controlling another app's playback, and "display over other apps" for the overlay. iOS declares usage descriptions for the music library, files, microphone and tracking.
- **Analytics and diagnostics.** A Firebase-based event layer, crash and non-fatal reporting, and performance traces. On iOS, analytics collection is held off until the App Tracking Transparency prompt has been answered.
- **Ads.** Ads are served through a mediation SDK and can be switched off per placement from a remote configuration. A Pro subscription removes them.
- **Network use.** Remote ad configuration, ad requests, store billing, push messaging, a store version check, and downloading demo tracks that are then cached on the device.

## How the descriptions were produced

The production project was inspected read-only. Statements describe behaviour observed in code, manifests and project documents; nothing was built or run for this write-up, and no performance, loudness or usage figures are claimed. Where behaviour could not be confirmed it is worded conservatively or left out. Diagram labels are conceptual and do not match production identifiers.

## Images

Screenshots are real captures from the Android app. Only screens without personal data were used; Android status and navigation bars were removed. Images were re-encoded from raw pixels with metadata stripped. The iOS build is not pictured.

## Publication checks

Before publishing, every file was checked for source code, credentials, signing material, personal data, email addresses, local file-system paths, account references and embedded image metadata. None remain.

## Access

Source access for hiring or due-diligence purposes can be discussed at the owner's discretion.
