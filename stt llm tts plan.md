# Master Implementation Plan: RAAH Codebase Refactoring

## Executive Summary
This document dictates the precise code-level modifications required to transform the current RAAH codebase from a monolithic, battery-heavy WebSocket prototype into a production-grade, background-compliant **LiveKit WebRTC Application**. 

All logic within this plan corresponds 1:1 with the verified `ultimate_system_architecture.md`.

---

## Phase 1: The Core Voice Engine (LiveKit Migration)
- **[MODIFY]** `ElevenLabsRealtimeService.swift`
  - Strip out all custom WebSocket (`URLSessionWebSocketTask`) configuration.
  - Implement the official **LiveKit WebRTC Swift SDK**.
  - Fetch the ElevenLabs ephemeral token and pass the `signed_url` directly into the `Room.connect(url)` method.
  - Expose native `audioLevel` publishers from the LiveKit `RemoteAudioTrack` so `VoiceOrbView` can perfectly sync its animations to the AI's voice.

## Phase 2: Resilience & Network Handoffs
- **[MODIFY]** `ElevenLabsRealtimeService.swift` & `VoiceStateManager`
  - WebRTC natively handles IP handovers (e.g., switching from WiFi to LTE instantly). However, for profound signal drops (e.g., walking into subway tunnels), configure the LiveKit `Room` delegate to actively buffer.
  - Increase `maxReconnectAttempts` to allow for at least **30-45 seconds of offline grace-period** before terminating the session.
  - *Fallback UX:* While `Room.state == .reconnecting`, intercept the state change and play an offline `.mp3` file (*"I'm losing your signal, give me a sec..."*) via Apple's native `AVAudioPlayer` to prevent confusing silence.

## Phase 3: Background Audio Resumption Protocol
- **[MODIFY]** `AppCoordinator` & `AudioSessionManager.swift`
  - Ensure the audio session correctly asks iOS for indefinite `playback` + `record` background privileges. 
- **[NEW LOGIC]** **The Geofence Wake-Up Trigger**
  - When iOS fires a `CoreLocation` Geofence in the deep background (e.g., `didExitRegion` after dwelling at a POI), the app only has ~10 seconds of background execution time.
  - `AudioSessionManager` must aggressively reclaim `AVAudioSession` priority (`.setActive(true)`).
  - It rapidly re-initializes the LiveKit WebRTC socket (if dormant) and uses the ElevenLabs `Client Event / Add Context` JSON payload API to inject an invisible system prompt (e.g., *"The user just spent 15 minutes at the Louvre. Ask them how it was."*). 
  - This violently forces the AI to output TTS and speak the narration directly into the user's AirPods without the user ever touching their phone.

## Phase 4: Graceful Suspension (Battery Protection)
- **[MODIFY]** `ElevenLabsRealtimeService.swift`
  - Leaving an open microphone running in the user's pocket for 2 hours will rapidly drain the battery and violate App Store privacy limits.
  - Implement a `TokenBucket` or Silence Timer based on Voice Activity Detection (VAD).
  - If 5 continuous minutes pass with absolute silence from both the user and the AI, safely disconnect the LiveKit `Room` and deactivate the `AVAudioSession`.
  - Use `didEnterRegion` Geofences as the global trigger to re-establish the connection transparently.

## Phase 5: Deprecation of Legacy Mosaics (Fixing the Split-Brain)
To conform to the Master Architecture and prevent uncatchable memory leaks, parallel sensory feeds are being eliminated.

- **[DELETE]** `OpenAIVisionService.swift`
  - Completely obliterate this file. The legacy "Snap and Ask" feature is officially descoped from the MVP. Attempting to invoke `AVCaptureSession` while the iPhone screen is locked results in fatal iOS background crashes. The AI is pure-audio.
- **[DELETE]** `OpenAIRealtimeService.swift`
  - We have standardized exclusively on ElevenLabs for speech, LLM inference, and tool calling. Delete this file to prevent architectural drift, UI confusion, and duplicate token billing.
- **[DELETE]** Manual `CLLocationManager` polling loops inside standard Views. All coordinates must flow strictly outward from the single Mapbox `EnhancedLocationManager`.
