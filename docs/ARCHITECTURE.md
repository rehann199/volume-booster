# Architecture

All diagrams are original and conceptual. They describe responsibilities, not production class or file names.

## 1. Layers

```mermaid
flowchart TD
    UI[Screens and widgets]
    CT[Controllers<br/>volume, equalizer, player,<br/>edge lighting, subscription]
    AE[Audio engine facade]
    PL[Player]
    PR[(Local preferences)]
    SV[Services<br/>ads, billing, analytics,<br/>messaging, updates, consent]
    NT[Native layers<br/>Kotlin + Swift]
    FB[Firebase]
    UI --> CT
    CT --> PR
    CT --> AE
    CT --> PL
    AE --> NT
    PL --> NT
    UI --> SV
    SV --> FB
    SV --> NT
```

Screens never talk to a native API or an SDK directly. Controllers hold state and persist it; the audio facade and the services are the only code that crosses into native layers or the network.

## 2. Platform abstraction

The shared code sees one contract. Each platform implements it differently.

```mermaid
flowchart LR
    subgraph Dart[Shared Dart]
        EQC[Equalizer controller]
        VOL[Volume controller]
        FAC[Audio engine facade]
        PLC[Player controller]
    end
    EQC --> FAC
    VOL --> FAC
    FAC -->|audio_engine channel| ANDN
    FAC -->|audio_engine channel| IOSN
    subgraph ANDR[Android]
        ANDN[Channel handler] --> ENG[Effect engine<br/>global session 0]
        ENG --> E1[Equalizer]
        ENG --> E2[BassBoost]
        ENG --> E3[Virtualizer]
        ENG --> E4[LoudnessEnhancer]
        ANDN --> SVC[Foreground service]
        PLC --> JA[just_audio]
    end
    subgraph IOSR[iOS]
        IOSN[Channel handler] --> GR[AVAudioEngine graph]
        PLC -->|ios_player channel| GR
        GR --> N1[Player node]
        N1 --> N2[EQ + low shelf]
        N2 --> N3[Reverb]
        N3 --> N4[Main mixer]
    end
```

Points to note:
- The channel handlers answer the same method names. Android-only methods (the "stopped by user" flags) are no-ops on iOS.
- The capability reply has the same shape on both platforms, so Dart does not branch on platform to decide which controls to show.
- Playback is the one place the platforms diverge in Dart: `just_audio` on Android, a native player behind a second channel on iOS, because only an engine graph can host the iOS EQ.

## 3. Applying an equalizer change

```mermaid
sequenceDiagram
    participant U as User
    participant C as Equalizer controller
    participant F as Audio engine facade
    participant N as Native layer
    U->>C: Moves a band or picks a preset
    C->>C: Persist bands, mark preset as Custom or selected
    C->>F: Apply(enabled, 10-band curve)
    F->>F: Resample curve to device band centres
    F->>N: setEqualizerEnabled, setEqualizerBands
    F->>N: setBassBoost, setVirtualizer
    F->>F: Recompute gain compensation
    F->>N: setLoudnessGain(boost + compensation, capped)
    N-->>F: success flag, or an error reported as a non-fatal
```

## 4. Android effect lifetime

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Active: Effect enabled, service started
    Active --> Active: Parameter change, mirrored to native preferences
    Active --> Idle: All effects off, release effects, stop service
    Active --> Idle: Stop tapped in notification, release, set stopped-by-user flag
    Active --> Dead: OS kills process
    Dead --> Active: Sticky restart succeeds, restore from native preferences
    Dead --> Idle: Foreground start refused, stop quietly, restore on next app launch
```

## 5. iOS player lifecycle

```mermaid
flowchart TD
    A[Pick a file] --> B[Load file, rebuild graph at its sample rate<br/>clamp EQ bands below Nyquist]
    B --> C[Schedule segment and play]
    C --> D{Event}
    D -->|Pause or seek| E[Record position, stop node,<br/>reschedule from new frame]
    D -->|Interruption began| F[Remember playing state, pause]
    F -->|Interruption ended| G[Reactivate session, resume]
    D -->|Route change or foreground| H[Ensure engine is running]
    D -->|File finished| I[Report completion once]
```

## 6. Ad gating

```mermaid
flowchart TD
    R[Ad slot requests a format] --> P{Pro entitlement?}
    P -->|yes| X[No ad]
    P -->|no| F{Remote config read succeeded?}
    F -->|no: offline, error, pending| X
    F -->|yes| G{Global switch on?}
    G -->|no| X
    G -->|yes| K{Placement field is true?}
    K -->|missing or false| X
    K -->|true| S[Request through mediation SDK]
```

The remote document is watched live, so flipping a flag takes effect in a running session.

## 7. Cross-cutting

- **Failure reporting.** Every audio, media and overlay channel call reports its failure as a tagged non-fatal; the polled call is rate-limited.
- **Persistence.** Dart state lives in `shared_preferences`. Android additionally keeps a native copy of effect state, because the service may run when Flutter does not.
- **Consent.** On iOS, App Tracking Transparency resolves before ads and analytics start; on Android the step is skipped.
