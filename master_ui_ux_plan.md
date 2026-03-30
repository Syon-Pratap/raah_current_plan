# Master UI/UX Visual Animation Plan

## 1. The Audio-Reactive Orb (Piping LiveKit to SwiftUI)
The most critical UI upgrade is transitioning the main avatar (`OrbView`) from a "Fake Loop Animation" to "Real-Time Audio Reactivity".

**The Implementation Blueprint:**
1. **The LiveKit Decibel Listener:** In `ElevenLabsRealtimeService.swift`, the LiveKit WebRTC room emits native audio frames. LiveKit has a built-in `audioLevel` observer that continuously returns the RMS (Root Mean Square) volume of the incoming AI voice.
2. **The VoiceStateManager Bridge:** The `VoiceStateManager` (our MVVM-C coordinator) must expose an `@Published var currentAudioLevel: CGFloat` (normalized between 0.0 and 1.0). This variable updates at 60fps while the AI is speaking.
3. **The `OrbView.swift` Refactor:**
   - Remove the generic `Task.sleep` async loops in `startRingLoop()`.
   - Bind the `ringPhase`, `scaleEffect`, and `glowRadius` directly to `voiceState.currentAudioLevel`. 
   - **The Math:** `scaleEffect(1.0 + (voiceState.currentAudioLevel * 0.4))`
   - *Result:* When the AI takes a breath, the Orb organically shrinks. When the AI emphasizes a loud syllable, the Orb flares instantly.

## 2. The Mapbox Heads-Up Display (HUD)
The AI is meant to be a walking companion, meaning the `OrbView` and `JournalView` overlays must sit cleanly on top of the 3D Mapbox `NavigationMapView`.

**The Implementation Blueprint:**
1. **Z-Stack Overlay:** Mapbox uses a `UIViewRepresentable` to render the 3D map engine. This must sit at the absolute bottom `#0` index of the Main SwiftUI ZStack.
2. **The Liquid Glass Layer:** Use the existing `LiquidGlassModifier` on all floating interface cards. Because `LiquidGlass.swift` uses iOS `.ultraThinMaterial`, it performs hardware-accelerated background blurring. This ensures the 3D buildings on the Mapbox layer dynamically blur behind the UI text.
3. **Hit Testing (`allowsHitTesting(false)`):** Critical for UX. The invisible wrappers holding the Orb and the Top Navigation Bar must not intercept touch events, allowing the user to seamlessly pan and rotate the Mapbox map *beneath* the floating UI elements.

## 3. Contextual Spatial Haptics (Fixing the Ghost Vibration)
You have `HapticEngine.swift` providing standard OS haptics. In a walking tour, users often have their phones in their pockets. Haptics are your primary UX mechanism to demand attention safely without a screen.

**The Implementation Blueprint:**
1. **The POI Geofence Trigger (`HapticEngine.success()`):** As dictated in the Auto-Narration pagination rules, when a `POI CoreLocation Geofence` trips (meaning the user dwelled near a highly scored POI that the AI is about to narrate), trigger a double-pulse haptic *immediately before* the AI claims background audio priority. This physical vibration cues the user to listen.
2. **The Bounding Box Exemption:** You must explicitly silence haptics when the 2-kilometer `Bounding Box Geofence` exits. This outer geofence silently triggers a background fetch to Supabase for the next chunk of 19 POIs. If haptics fire here, the user's phone will vibrate randomly without the AI ever speaking, creating a confusing "Ghost Vibration" UX.
3. **The State Change Trigger (`HapticEngine.light()`):** When the `OrbView` transitions state over WebRTC (e.g., from `.listening` to `.thinking`), fire a highly subtle light haptic tap. This confirms to the user that the AI registered their voice request even if they aren't looking at the screen.

---

### Conclusion
By ripping out the hardcoded 3-second math loops in `OrbView` and explicitly binding the `scaleEffect` modifiers to the live WebRTC `audioLevel` data stream, RAAH achieves the hyper-realistic, biologically reactive AI aesthetic that separates basic apps from elite, sensory-rich products.
